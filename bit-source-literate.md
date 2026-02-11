# bit — Literate Programming Document

This document contains all Haskell source files and the cabal
file for the bit project, presented in literate-programming style.

---

## Bit.hs

**Path:** `Bit.hs`

*Source file.*

```haskell
{-# LANGUAGE ScopedTypeVariables #-}

import Bit.Commands
import GHC.IO.Encoding (setLocaleEncoding, utf8)
import System.IO (hSetEncoding, stdout, stderr, hIsTerminalDevice)
import System.Info (os)
import System.Process (callCommand)
import Control.Exception (catch, SomeException)
import Control.Monad (when)

main :: IO ()
main = do
    -- Set console to UTF-8 on Windows (only when interactive)
    -- Skip during automated tests to avoid git binary file issues
    isTerminal <- hIsTerminalDevice stdout
    when (os == "mingw32" && isTerminal) $
        callCommand "chcp 65001 > nul 2>&1" `catch` \(_ :: SomeException) -> pure ()
    
    -- Set UTF-8 for all IO (stdout, stderr, and subprocess pipes)
    -- This ensures readProcessWithExitCode decodes git output as UTF-8
    setLocaleEncoding utf8
    hSetEncoding stdout utf8
    hSetEncoding stderr utf8
    Bit.Commands.run
```

---

## Bit/AtomicWrite.hs

**Path:** `Bit/AtomicWrite.hs`

*Source file.*

```haskell
{-# LANGUAGE ScopedTypeVariables #-}

-- | Atomic file writes with Windows retry logic.
--
-- This module provides:
-- 
-- * 'atomicWriteFile' - Atomic write using temp file + rename pattern
-- * 'DirWriteLock' - Directory-level locking for coordinating concurrent writes
-- * 'LockRegistry' - Global registry of directory locks for process-wide coordination
--
-- On Windows, atomic writes use retry logic to handle transient "permission denied"
-- errors caused by antivirus, Windows Search, or other processes holding handles.
--
-- This module has no dependencies on other Bit modules to avoid circular imports.
module Bit.AtomicWrite
  ( -- * Atomic file writes
    atomicWriteFile
  , atomicWriteFileStr
  , atomicWriteFileWithLock
  
    -- * Directory locking (for coordinated concurrent writes)
  , DirWriteLock
  , newDirWriteLock
  , withDirWriteLock
  
    -- * Lock registry (process-wide lock coordination)
  , LockRegistry
  , newLockRegistry
  , withLockedDir
  ) where

import System.Directory (renameFile, removeFile)
import System.IO (openTempFile, hClose)
import System.FilePath (takeDirectory, (</>))
import qualified Data.ByteString as BS
import Data.Text.Encoding (encodeUtf8)
import Data.Text (pack)
import Control.Exception (bracketOnError, catch, IOException, try)
import Control.Monad (void)
import Control.Concurrent (threadDelay)
import Control.Concurrent.MVar (MVar, newMVar, withMVar, modifyMVar)
import qualified Data.Map.Strict as Map

-- ============================================================================
-- Atomic File Writes
-- ============================================================================

-- | Write content to target atomically (temp file + rename). Spec § Atomic Operations.
-- Uses bracketOnError to clean up temp file if an exception occurs.
-- On Windows, retries the rename a few times if the target is locked.
atomicWriteFile :: FilePath -> BS.ByteString -> IO ()
atomicWriteFile target content = do
  let tempDir = takeDirectory target
  bracketOnError
    (openTempFile tempDir ".bit-tmp")
    (\(tempPath, handle) -> do
        hClose handle
        removeFile tempPath `catch` \(_ :: IOException) -> pure ())
    (\(tempPath, handle) -> do
        BS.hPut handle content
        hClose handle
        -- Small delay to ensure handle is fully released on Windows
        threadDelay 10000  -- 10ms
        renameWithRetry tempPath target 5)

-- | Retry rename up to n times with increasing delays (for Windows file locking).
-- If target file exists and is locked, try to delete it first.
renameWithRetry :: FilePath -> FilePath -> Int -> IO ()
renameWithRetry src dest 0 = do
  -- Final attempt: try to remove target first, then rename
  removeFile dest `catch` \(_ :: IOException) -> pure ()
  renameFile src dest
renameWithRetry src dest n = do
  result <- try (renameFile src dest)
  case result of
    Right () -> pure ()
    Left (_ :: IOException) -> do
      -- Try to remove the locked target file
      void (try (removeFile dest) :: IO (Either IOException ()))
      threadDelay (50000 * (6 - n))  -- 50ms, 100ms, 150ms, 200ms, 250ms
      renameWithRetry src dest (n - 1)

-- | Atomic write of a String (UTF-8).
atomicWriteFileStr :: FilePath -> String -> IO ()
atomicWriteFileStr path str = atomicWriteFile path (encodeUtf8 (pack str))

-- ============================================================================
-- Directory Write Locks
-- ============================================================================

-- | Lock for coordinating writes to files in a directory.
--
-- Combines MVar (thread-level coordination within this process) with
-- retry logic for Windows OS-level conflicts. For process-level coordination,
-- use 'LockRegistry'.
--
-- The temp-file-rename pattern has contention at the directory level when
-- creating temp files and calling 'renameFile', so we lock at the directory
-- level rather than the file level.
data DirWriteLock = DirWriteLock
  { dwlMVar    :: MVar ()     -- ^ Thread-level coordination
  , _dwlDir     :: FilePath    -- ^ Directory being protected
  , _dwlLockFile :: FilePath   -- ^ Path to .lock file (for future file locking)
  }

-- | Create a new directory write lock.
--
-- The lock file path is @dir </> ".bit-write.lock"@.
newDirWriteLock :: FilePath -> IO DirWriteLock
newDirWriteLock dir = do
  mvar <- newMVar ()
  let lockFile = dir </> ".bit-write.lock"
  pure $ DirWriteLock mvar dir lockFile

-- | Execute action while holding the directory lock.
--
-- This coordinates Haskell threads within the same process.
-- For OS-level coordination (multiple processes), the retry logic in
-- 'atomicWriteFile' handles transient conflicts.
withDirWriteLock :: DirWriteLock -> IO a -> IO a
withDirWriteLock dwl action = withMVar (dwlMVar dwl) $ \() -> action

-- | Atomically write a file using temp + rename pattern with directory locking.
--
-- Thread-safe within this process and handles Windows quirks with retry logic.
atomicWriteFileWithLock :: DirWriteLock -> FilePath -> BS.ByteString -> IO ()
atomicWriteFileWithLock dwl destPath content =
  withDirWriteLock dwl $ atomicWriteFile destPath content

-- ============================================================================
-- Lock Registry (process-wide)
-- ============================================================================

-- | Registry of directory locks for process-wide coordination.
--
-- Use this when you have multiple workers that may write to different
-- directories and need to coordinate their writes:
--
-- @
-- registry <- newLockRegistry
-- forConcurrently_ files $ \file ->
--   withLockedDir registry file $
--     atomicWriteFile file content
-- @
newtype LockRegistry = LockRegistry (MVar (Map.Map FilePath DirWriteLock))

-- | Create a new lock registry.
newLockRegistry :: IO LockRegistry
newLockRegistry = LockRegistry <$> newMVar Map.empty

-- | Execute action while holding the lock for a file's directory.
--
-- Gets or creates a lock for the directory containing the file,
-- then runs the action while holding that lock.
withLockedDir :: LockRegistry -> FilePath -> IO a -> IO a
withLockedDir (LockRegistry mvar) path action = do
  let dir = takeDirectory path
  lock <- modifyMVar mvar $ \locks ->
    case Map.lookup dir locks of
      Just lock -> pure (locks, lock)
      Nothing -> do
        lock <- newDirWriteLock dir
        pure (Map.insert dir lock locks, lock)
  withDirWriteLock lock action
```

---

## Bit/Commands.hs

**Path:** `Bit/Commands.hs`

*Source file.*

```haskell
{-# LANGUAGE LambdaCase #-}
{-# LANGUAGE ScopedTypeVariables #-}

module Bit.Commands (run) where

import qualified Bit.Core as Bit
import Bit.Types (BitEnv(..), ForceMode(..), runBitM)
import qualified Bit.Scan as Scan  -- Only for the pre-scan in runCommand
import Bit.Remote (getDefaultRemote, resolveRemote)
import Bit.Utils (atomicWriteFileStr)
import Bit.Concurrency (Concurrency(..))
import qualified Bit.RemoteWorkspace as RemoteWorkspace
import System.Environment (getArgs)
import Bit.Help (printMainHelp, printTerseHelp, printCommandHelp)
import System.Exit (ExitCode(..), exitWith, exitSuccess)
import System.FilePath ((</>))
import System.IO (hPutStrLn, stderr)
import Control.Monad (when, unless, void)
import qualified System.Directory as Dir
import qualified Internal.Git as Git
import Data.List (dropWhileEnd)
-- Strict IO imports to avoid Windows file locking issues
import qualified Data.ByteString as BS
import qualified Data.Text as T
import Data.Text.Encoding (decodeUtf8')

run :: IO ()
run = do
    args <- getArgs
    case args of
        []               -> printMainHelp >> exitSuccess
        ["help"]         -> printMainHelp >> exitSuccess
        ["help", cmd]    -> printCommandHelp cmd >> exitSuccess
        ["help", c1, c2] -> printCommandHelp (c1 ++ " " ++ c2) >> exitSuccess
        ["-h"]           -> printMainHelp >> exitSuccess
        ["--help"]       -> printMainHelp >> exitSuccess
        _  -> case extractRemoteTarget args of
            RemoteError msg -> do
                hPutStrLn stderr $ "fatal: " ++ msg
                exitWith (ExitFailure 1)
            NoRemote remaining -> runCommand remaining
            RemoteFound remoteName remaining ->
                runRemoteCommand remoteName remaining

-- | Result of extracting a remote target from CLI args.
data RemoteExtract
    = NoRemote [String]            -- ^ No remote specified; remaining args
    | RemoteFound String [String]  -- ^ Remote name + remaining args
    | RemoteError String           -- ^ Error message

-- | Extract @<remote> or --remote <name> from the leading args.
-- Both forms must appear at the start of the arg list, consistent with
-- each other and avoiding conflicts with subcommand flags (e.g. verify --remote).
extractRemoteTarget :: [String] -> RemoteExtract
extractRemoteTarget [] = NoRemote []
extractRemoteTarget (('@':name@(_:_)):rest)
    | "--remote" `elem` rest = RemoteError
        "Cannot use both @<remote> and --remote <name>."
    | otherwise = RemoteFound name rest
extractRemoteTarget ("--remote":name:rest) = RemoteFound name rest
extractRemoteTarget ["--remote"] = RemoteError
    "'--remote' requires a remote name argument."
extractRemoteTarget args = NoRemote args

-- | Execute a command in the context of an ephemeral remote workspace.
-- Each command fetches the bundle from remote, inflates into a temp workspace,
-- operates, re-bundles if needed, and pushes back. No persistent workspace.
runRemoteCommand :: String -> [String] -> IO ()
runRemoteCommand remoteName args = do
    cwd <- Dir.getCurrentDirectory
    bitExists <- Dir.doesDirectoryExist (cwd </> ".bit")
    unless bitExists $ do
        hPutStrLn stderr "fatal: not a bit repository (or any of the parent directories): .bit"
        exitWith (ExitFailure 1)

    mRemote <- resolveRemote cwd remoteName
    case mRemote of
        Nothing -> do
            hPutStrLn stderr $ "fatal: remote '" ++ remoteName ++ "' not found."
            exitWith (ExitFailure 1)
        Just remote -> case args of
            ["init"] ->
                RemoteWorkspace.initRemote remote remoteName
            ("add":paths) ->
                RemoteWorkspace.addRemote remote paths >>= exitWith
            ("commit":commitArgs) ->
                RemoteWorkspace.commitRemote remote commitArgs >>= exitWith
            ("status":rest) ->
                RemoteWorkspace.statusRemote remote rest >>= exitWith
            ("log":rest) ->
                RemoteWorkspace.logRemote remote rest >>= exitWith
            ("ls-files":rest) ->
                RemoteWorkspace.lsFilesRemote remote rest >>= exitWith
            _ -> do
                hPutStrLn stderr $ "error: command not supported in remote context: " ++ unwords args
                hPutStrLn stderr "Supported: init, add, commit, status, log, ls-files"
                exitWith (ExitFailure 1)

-- | Helper function to push with upstream tracking
pushWithUpstream :: BitEnv -> FilePath -> String -> IO ()
pushWithUpstream env cwd name = do
    mNamedRemote <- resolveRemote cwd name
    let envWithRemote = env { envRemote = mNamedRemote }
    runBitM envWithRemote Bit.push
    -- After successful push, set upstream tracking
    void $ Git.setupBranchTrackingFor name
    putStrLn $ "branch 'main' set up to track '" ++ name ++ "/main'."

-- | Sync .bitignore to .bit/index/.gitignore with normalization
syncBitignoreToIndex :: FilePath -> IO ()
syncBitignoreToIndex cwd = do
    let bitignoreSrc = cwd </> ".bitignore"
        bitignoreDest = cwd </> ".bit" </> "index" </> ".gitignore"
    bitignoreExists <- Dir.doesFileExist bitignoreSrc
    if bitignoreExists
        then writeBitignore bitignoreSrc bitignoreDest
        else removeStaleGitignore bitignoreDest
  where
    writeBitignore :: FilePath -> FilePath -> IO ()
    writeBitignore src dest = do
        bs <- BS.readFile src
        let content = either (const "") T.unpack (decodeUtf8' bs)
            normalizedLines = filter (not . null) $
              map (trim . filter (/= '\r')) (lines content)
        atomicWriteFileStr dest (unlines normalizedLines)
    
    removeStaleGitignore :: FilePath -> IO ()
    removeStaleGitignore dest = do
        destExists <- Dir.doesFileExist dest
        when destExists $ Dir.removeFile dest
    
    trim :: String -> String
    trim = dropWhile (== ' ') . dropWhileEnd (== ' ')

-- | Extract the command key from args for help lookup.
-- Handles multi-word commands like "remote add", "merge --continue", etc.
commandKey :: [String] -> String
commandKey ("remote":sub:_)
    | sub `elem` ["add", "show", "repair"] = "remote " ++ sub
commandKey ("merge":sub:_)
    | sub `elem` ["--continue", "--abort"] = "merge " ++ sub
commandKey ("branch":sub:_)
    | sub `elem` ["--unset-upstream"] = "branch " ++ sub
commandKey (cmd:_) = cmd
commandKey [] = ""

runCommand :: [String] -> IO ()
runCommand args = do
    let hasHelp = "--help" `elem` args
    let hasTerseHelp = "-h" `elem` args
    let hasForce = "--force" `elem` args || "-f" `elem` args
    let hasForceWithLease = "--force-with-lease" `elem` args
    let isSequential = "--sequential" `elem` args
    when (hasForce && hasForceWithLease) $ do
        hPutStrLn stderr "fatal: Cannot use both --force and --force-with-lease"
        exitWith (ExitFailure 1)
    let forceMode
          | hasForce          = Force
          | hasForceWithLease = ForceWithLease
          | otherwise         = NoForce
    let cmd = filter (`notElem` ["--force", "-f", "--force-with-lease", "--sequential", "-h", "--help"]) args

    -- Help intercept (before repo check — help works without a repo)
    when (hasHelp || hasTerseHelp) $ do
        let key = commandKey cmd
        if null key
            then printMainHelp >> exitSuccess
            else if hasTerseHelp
                then printTerseHelp key >> exitSuccess
                else printCommandHelp key >> exitSuccess

    cwd <- Dir.getCurrentDirectory
    bitExists <- Dir.doesDirectoryExist (cwd </> ".bit")

    -- Lightweight env (no scan) — for read-only commands
    let baseEnv = do
            mRemote <- getDefaultRemote cwd
            pure $ BitEnv cwd [] mRemote forceMode

    -- Full env (scan + bitignore sync + metadata write) — for write commands
    let scannedEnv = do
            syncBitignoreToIndex cwd
            localFiles <- Scan.scanWorkingDir cwd
            Scan.writeMetadataFiles cwd localFiles
            mRemote <- getDefaultRemote cwd
            pure $ BitEnv cwd localFiles mRemote forceMode

    -- Repo existence check (skip for init)
    let needsRepo = cmd /= ["init"]
    when (needsRepo && not bitExists) $ do
        hPutStrLn stderr "fatal: not a bit repository (or any of the parent directories): .bit"
        exitWith (ExitFailure 1)

    -- Helper functions for running commands
    let runScanned action = scannedEnv >>= \env -> runBitM env action
    let runBase action = baseEnv >>= \env -> runBitM env action
    let runScannedWithRemote name action = do
            env <- scannedEnv
            mNamedRemote <- resolveRemote cwd name
            runBitM env { envRemote = mNamedRemote } action
    let runBaseWithRemote name action = do
            env <- baseEnv
            mNamedRemote <- resolveRemote cwd name
            runBitM env { envRemote = mNamedRemote } action

    case cmd of
        -- ── No env needed ────────────────────────────────────
        ["init"]                        -> Bit.init
        ["remote", "add", name, url]    -> Bit.remoteAdd name url
        ["fsck"]                        -> Bit.fsck cwd
        ["merge", "--abort"]            -> Bit.mergeAbort
        ["branch", "--unset-upstream"]  -> Bit.unsetUpstream

        -- ── Lightweight env (no scan) ────────────────────────
        ("log":rest)                    -> Bit.log rest >>= exitWith
        ("ls-files":rest)               -> Bit.lsFiles rest >>= exitWith
        ["remote", "show"]              -> runBase $ Bit.remoteShow Nothing
        ["remote", "show", name]        -> runBaseWithRemote name $ Bit.remoteShow (Just name)
        ["remote", "repair"]             -> runBase $ Bit.remoteRepair Nothing (if isSequential then Sequential else Parallel 0)
        ["remote", "repair", name]      -> runBaseWithRemote name $ Bit.remoteRepair (Just name) (if isSequential then Sequential else Parallel 0)
        ["verify"]                      -> runBase $ Bit.verify Bit.VerifyLocal (if isSequential then Sequential else Parallel 0)
        ["verify", "--remote"]          -> runBase $ Bit.verify Bit.VerifyRemote (if isSequential then Sequential else Parallel 0)

        ("rm":rest)                     -> runBase (Bit.rm rest) >>= exitWith

        -- ── Full scanned env (needs working directory state) ─
        ("add":rest)                    -> do
            void scannedEnv
            Bit.add rest >>= exitWith
        ("commit":rest)                 -> do
            void scannedEnv
            Bit.commit rest >>= exitWith
        ("diff":rest)                   -> do
            void scannedEnv
            Bit.diff rest >>= exitWith
        ("status":rest)                 -> runScanned (Bit.status rest) >>= exitWith
        ("restore":rest)                -> runScanned (Bit.restore rest) >>= exitWith
        ("checkout":rest)               -> runScanned (Bit.checkout rest) >>= exitWith
        
        -- push
        ["push"]                        -> runScanned Bit.push
        ["push", "-u", name]            -> scannedEnv >>= \env -> pushWithUpstream env cwd name
        ["push", "--set-upstream", name] -> scannedEnv >>= \env -> pushWithUpstream env cwd name
        ["push", name]                  -> runScannedWithRemote name Bit.push
        
        -- pull
        ["pull"]                        -> runScanned $ Bit.pull Bit.defaultPullOptions
        ["pull", name]                  -> runScannedWithRemote name $ Bit.pull Bit.defaultPullOptions
        ["pull", "--accept-remote"]     -> runScanned $ Bit.pull (Bit.PullOptions Bit.PullAcceptRemote)
        ["pull", "--manual-merge"]      -> runScanned $ Bit.pull (Bit.PullOptions Bit.PullManualMerge)
        ["pull", name, "--accept-remote"] -> runScannedWithRemote name $ Bit.pull (Bit.PullOptions Bit.PullAcceptRemote)
        ["pull", "--accept-remote", name] -> runScannedWithRemote name $ Bit.pull (Bit.PullOptions Bit.PullAcceptRemote)
        ["pull", name, "--manual-merge"] -> runScannedWithRemote name $ Bit.pull (Bit.PullOptions Bit.PullManualMerge)
        ["pull", "--manual-merge", name] -> runScannedWithRemote name $ Bit.pull (Bit.PullOptions Bit.PullManualMerge)
        
        -- fetch
        ["fetch"]                       -> runScanned Bit.fetch
        ["fetch", name]                 -> runScannedWithRemote name Bit.fetch
        
        ["merge", "--continue"]         -> runScanned Bit.mergeContinue
        _                               -> do
            hPutStrLn stderr $ "bit: '" ++ unwords cmd ++ "' is not a bit command. See 'bit help'."
            exitWith (ExitFailure 1)
```

---

## Bit/Concurrency.hs

**Path:** `Bit/Concurrency.hs`

*Source file.*

```haskell
{-# LANGUAGE BangPatterns #-}

-- | Concurrency utilities for bounded parallel execution.
--
-- This module provides helpers for running IO-bound operations in parallel
-- with bounded concurrency to avoid backpressure (too many concurrent IO
-- operations overwhelming the OS).
--
-- The concurrency levels are based on the number of available CPU cores
-- (from 'getNumCapabilities') and scaled appropriately for different workload
-- types (file IO, network IO, etc.).
module Bit.Concurrency
  ( -- * Concurrency level calculation
    ioConcurrency
  , networkConcurrency
  
    -- * Concurrency control
  , Concurrency(..)
  , runConcurrently
  , runConcurrentlyBounded
  
    -- * Re-exports from Scan.hs for compatibility
  , mapConcurrentlyBounded
  ) where

import Control.Concurrent (getNumCapabilities)
import Control.Concurrent.QSem (newQSem, waitQSem, signalQSem)
import Control.Concurrent.Async (mapConcurrently)
import Control.Exception (bracket_)

-- | Concurrency mode for running operations.
data Concurrency
  = Parallel Int  -- ^ Run up to N operations in parallel
  | Sequential    -- ^ Run operations one at a time (for debugging or benchmarking)
  deriving (Show, Eq)

-- | Standard concurrency level for IO-bound file operations.
-- Scales with available cores, bounded to avoid backpressure.
--
-- For IO-bound work (file reads, hashing), threads spend most time waiting
-- on IO, so we use a multiplier of 4× cores. This provides good parallelism
-- without overwhelming the disk subsystem.
--
-- Minimum of 4 ensures reasonable performance even on single-core systems.
ioConcurrency :: IO Int
ioConcurrency = max 4 . (* 4) <$> getNumCapabilities

-- | Lower concurrency for network/subprocess operations.
--
-- Network and subprocess operations have additional constraints:
-- - Each subprocess has OS overhead
-- - Cloud APIs may have rate limits
-- - Network bandwidth is shared
--
-- We use a lower multiplier (2× cores) and cap at 8 to avoid overwhelming
-- remote services or the network stack.
networkConcurrency :: IO Int
networkConcurrency = min 8 . max 2 . (* 2) <$> getNumCapabilities

-- | Run an action over a list with the specified concurrency level.
--
-- When 'Sequential', operations run one at a time with 'mapM'.
-- When 'Parallel n', operations run with bounded parallelism using 'mapConcurrentlyBounded'.
runConcurrently :: Concurrency -> (a -> IO b) -> [a] -> IO [b]
runConcurrently Sequential f xs = mapM f xs
runConcurrently (Parallel n) f xs = mapConcurrentlyBounded n f xs

-- | Like 'runConcurrently' but takes the bound as a direct Int parameter.
-- Useful when you already have a computed concurrency level.
runConcurrentlyBounded :: Int -> (a -> IO b) -> [a] -> IO [b]
runConcurrentlyBounded = mapConcurrentlyBounded

-- | Bounded parallel map: runs up to @bound@ actions concurrently.
--
-- Uses a 'QSem' (quantity semaphore) to limit the number of concurrent
-- operations. Each action acquires the semaphore before running and
-- releases it after completion (even on exception).
--
-- This is the same implementation used in Bit.Scan and is re-exported
-- here for consistency.
mapConcurrentlyBounded :: Int -> (a -> IO b) -> [a] -> IO [b]
mapConcurrentlyBounded bound f xs = do
    sem <- newQSem bound
    mapConcurrently (\x -> bracket_ (waitQSem sem) (signalQSem sem) (f x)) xs
```

---

## Bit/ConcurrentFileIO.hs

**Path:** `Bit/ConcurrentFileIO.hs`

*Source file.*

```haskell
{-# LANGUAGE NoImplicitPrelude #-}
{-# LANGUAGE ScopedTypeVariables #-}

-- | Strict, concurrent-safe file IO operations.
--
-- This module intentionally does NOT export lazy IO functions.
-- Use 'readFileBinaryStrict' instead of 'Prelude.readFile'.
--
-- All operations:
-- * Use strict 'ByteString' to ensure file handles close immediately
-- * Are safe for concurrent access (no lazy IO handle retention)
-- * Work correctly on Windows (no "file is locked" errors)
--
-- Import this module instead of 'Prelude' for file operations:
--
-- @
-- import Bit.ConcurrentFileIO (readFileBinaryStrict, writeFileBinaryStrict)
-- @
module Bit.ConcurrentFileIO
  ( -- * Reading (strict)
    readFileBinaryStrict
  , readFileUtf8Strict
  , readFileMaybe
  , readFileUtf8Maybe
  
    -- * Writing (strict)
  , writeFileBinaryStrict
  , writeFileUtf8Strict
  
    -- * Re-exports (safe operations only)
  , BS.ByteString
  , T.Text
  ) where

import Prelude (FilePath, Maybe(..), Either(..), pure, ($), (.), const, either)
import Control.Exception (try, SomeException, throwIO)
import Control.Monad.IO.Class (MonadIO, liftIO)
import qualified Data.ByteString as BS
import qualified Data.Text as T
import qualified Data.Text.Encoding as T

-- ============================================================================
-- Reading (strict)
-- ============================================================================

-- | Read entire file strictly into memory as 'ByteString'.
-- File handle is closed before returning.
--
-- This is the preferred way to read files in concurrent code.
-- Unlike 'Prelude.readFile', this does not use lazy IO.
readFileBinaryStrict :: MonadIO m => FilePath -> m BS.ByteString
readFileBinaryStrict = liftIO . BS.readFile

-- | Read entire file as strict UTF-8 'Text'.
-- File handle is closed before returning.
--
-- Throws 'T.UnicodeException' on invalid UTF-8.
readFileUtf8Strict :: MonadIO m => FilePath -> m T.Text
readFileUtf8Strict path = liftIO $ do
  bs <- BS.readFile path
  case T.decodeUtf8' bs of
    Left err -> throwIO err
    Right t  -> pure t

-- | Read file, returning 'Nothing' on any error.
-- Useful for "check if exists and read" patterns.
--
-- This combines file existence check and read into one atomic operation,
-- avoiding TOCTOU race conditions.
readFileMaybe :: MonadIO m => FilePath -> m (Maybe BS.ByteString)
readFileMaybe path = liftIO $ do
  (result :: Either SomeException BS.ByteString) <- try (BS.readFile path)
  pure $ either (const Nothing) Just result

-- | Read file as UTF-8, returning 'Nothing' on any error (including invalid UTF-8).
readFileUtf8Maybe :: MonadIO m => FilePath -> m (Maybe T.Text)
readFileUtf8Maybe path = liftIO $ do
  (result :: Either SomeException BS.ByteString) <- try (BS.readFile path)
  pure $ either (const Nothing) (either (const Nothing) Just . T.decodeUtf8') result

-- ============================================================================
-- Writing (strict)
-- ============================================================================

-- | Write 'ByteString' to file strictly.
-- File handle is closed before returning.
--
-- NOTE: This is NOT atomic. For atomic writes, use 'Bit.Utils.atomicWriteFile'.
writeFileBinaryStrict :: MonadIO m => FilePath -> BS.ByteString -> m ()
writeFileBinaryStrict path = liftIO . BS.writeFile path

-- | Write 'Text' to file as UTF-8.
-- File handle is closed before returning.
--
-- NOTE: This is NOT atomic. For atomic writes, use 'Bit.Utils.atomicWriteFile'.
writeFileUtf8Strict :: MonadIO m => FilePath -> T.Text -> m ()
writeFileUtf8Strict path = liftIO . BS.writeFile path . T.encodeUtf8
```

---

## Bit/ConcurrentIO.hs

**Path:** `Bit/ConcurrentIO.hs`

*Source file.*

```haskell
{-# LANGUAGE GeneralizedNewtypeDeriving #-}
{-# LANGUAGE RankNTypes #-}
{-# LANGUAGE ScopedTypeVariables #-}

-- | Type-safe concurrent IO operations.
--
-- This module provides a 'ConcurrentIO' newtype that restricts IO to
-- strict, concurrent-safe operations. The constructor is NOT exported,
-- preventing smuggling of arbitrary lazy IO via 'liftIO'.
--
-- All file operations use strict 'ByteString' to ensure file handles
-- are closed before returning, eliminating Windows "file is locked" errors.
--
-- Usage:
--
-- @
-- import Bit.ConcurrentIO
--
-- scanFiles :: [FilePath] -> ConcurrentIO [FileHash]
-- scanFiles paths = do
--   sem <- newQSemC 8
--   mapConcurrentlyBoundedC sem hashFile paths
--
-- hashFile :: FilePath -> ConcurrentIO FileHash
-- hashFile path = do
--   contents <- readFileStrict path
--   pure $ FileHash path (sha256 contents)
-- @
module Bit.ConcurrentIO
  ( -- * The ConcurrentIO monad
    ConcurrentIO  -- Type exported, constructor hidden
  , runConcurrentIO
  
    -- * Strict ByteString file operations
  , readFileStrict
  , writeFileStrict
  , readFileMaybeC
  
    -- * Concurrency primitives
  , mapConcurrentlyBoundedC
  , newQSemC
  , forkIOC
  , threadDelayC
  
    -- * IORef operations (strict)
  , newIORefC
  , readIORefC
  , atomicModifyIORefC'
  
    -- * Exception handling
  , tryC
  , catchC
  , bracketC
  , finallyC
  ) where

import qualified Data.ByteString as BS
import Control.Concurrent (ThreadId, forkIO, threadDelay)
import Control.Concurrent.Async (mapConcurrently)
import Control.Concurrent.QSem (QSem, newQSem, waitQSem, signalQSem)
import Control.Exception (Exception, SomeException, try, catch, bracket, bracket_, finally)
import Data.IORef (IORef, newIORef, readIORef, atomicModifyIORef')

-- | A restricted IO monad that only permits strict, concurrent-safe operations.
--
-- The constructor 'UnsafeConcurrentIO' is NOT exported. This prevents users
-- from lifting arbitrary lazy IO operations into 'ConcurrentIO'.
--
-- To add new operations to 'ConcurrentIO', add them to this module with
-- explicit wrappers around the safe IO functions.
newtype ConcurrentIO a = UnsafeConcurrentIO { runConcurrentIO :: IO a }
  deriving (Functor, Applicative, Monad)
  -- NOTE: No MonadIO instance! This is intentional.
  -- Deriving MonadIO would allow 'liftIO' to smuggle arbitrary lazy IO.

-- ============================================================================
-- Strict ByteString file operations
-- ============================================================================

-- | Read entire file strictly into memory as 'ByteString'.
-- File handle is closed before returning.
readFileStrict :: FilePath -> ConcurrentIO BS.ByteString
readFileStrict = UnsafeConcurrentIO . BS.readFile

-- | Write 'ByteString' to file strictly.
-- File handle is closed before returning.
writeFileStrict :: FilePath -> BS.ByteString -> ConcurrentIO ()
writeFileStrict path = UnsafeConcurrentIO . BS.writeFile path

-- | Read file, returning 'Nothing' on any error.
-- Useful for "check if exists and read" patterns.
readFileMaybeC :: FilePath -> ConcurrentIO (Maybe BS.ByteString)
readFileMaybeC path = UnsafeConcurrentIO $
  either (const Nothing) Just <$> (try (BS.readFile path) :: IO (Either SomeException BS.ByteString))

-- ============================================================================
-- Concurrency primitives
-- ============================================================================

-- | Create a new quantity semaphore for bounding parallelism.
newQSemC :: Int -> ConcurrentIO QSem
newQSemC = UnsafeConcurrentIO . newQSem

-- | Bounded parallel map: runs up to the semaphore's bound concurrently.
--
-- Each action acquires the semaphore before running and releases after.
-- Uses 'mapConcurrently' from the async package internally.
mapConcurrentlyBoundedC 
  :: Traversable t 
  => QSem 
  -> (a -> ConcurrentIO b) 
  -> t a 
  -> ConcurrentIO (t b)
mapConcurrentlyBoundedC sem f xs = UnsafeConcurrentIO $
  mapConcurrently (withSem . runConcurrentIO . f) xs
  where
    withSem action = bracket_ (waitQSem sem) (signalQSem sem) action

-- | Fork a new thread.
forkIOC :: ConcurrentIO () -> ConcurrentIO ThreadId
forkIOC action = UnsafeConcurrentIO $ forkIO (runConcurrentIO action)

-- | Delay for the specified number of microseconds.
threadDelayC :: Int -> ConcurrentIO ()
threadDelayC = UnsafeConcurrentIO . threadDelay

-- ============================================================================
-- IORef operations (strict)
-- ============================================================================

-- | Create a new 'IORef'.
newIORefC :: a -> ConcurrentIO (IORef a)
newIORefC = UnsafeConcurrentIO . newIORef

-- | Read an 'IORef'.
readIORefC :: IORef a -> ConcurrentIO a
readIORefC = UnsafeConcurrentIO . readIORef

-- | Strictly modify an 'IORef'. The function is applied strictly.
atomicModifyIORefC' :: IORef a -> (a -> (a, b)) -> ConcurrentIO b
atomicModifyIORefC' ref f = UnsafeConcurrentIO $ atomicModifyIORef' ref f

-- ============================================================================
-- Exception handling
-- ============================================================================

-- | Try an action, catching exceptions.
tryC :: Exception e => ConcurrentIO a -> ConcurrentIO (Either e a)
tryC action = UnsafeConcurrentIO $ try (runConcurrentIO action)

-- | Catch exceptions from an action.
catchC :: Exception e => ConcurrentIO a -> (e -> ConcurrentIO a) -> ConcurrentIO a
catchC action handler = UnsafeConcurrentIO $
  runConcurrentIO action `catch` (runConcurrentIO . handler)

-- | Bracket pattern for resource management.
bracketC :: ConcurrentIO a -> (a -> ConcurrentIO b) -> (a -> ConcurrentIO c) -> ConcurrentIO c
bracketC acquire release use = UnsafeConcurrentIO $
  bracket (runConcurrentIO acquire) (runConcurrentIO . release) (runConcurrentIO . use)

-- | Run cleanup action regardless of success or failure.
finallyC :: ConcurrentIO a -> ConcurrentIO b -> ConcurrentIO a
finallyC action cleanup = UnsafeConcurrentIO $
  runConcurrentIO action `finally` runConcurrentIO cleanup
```

---

## Bit/Conflict.hs

**Path:** `Bit/Conflict.hs`

*Source file.*

```haskell
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE MultiWayIf #-}

module Bit.Conflict
  ( Resolution(..)
  , DeletedSide(..)  -- re-exported from Internal.Git
  , ConflictInfo(..)
  , resolveConflict
  , resolveAll
  , getConflictedFilesE
  , parseConflictInfo
  ) where

import Data.Char (toLower, isSpace)
import Data.List (isPrefixOf, dropWhileEnd)
import Data.Maybe (mapMaybe)
import Text.Read (readMaybe)
import Control.Monad (void, when)
import System.Exit (ExitCode(..))
import qualified Internal.Git as Git
import Internal.Git (DeletedSide(..))
import System.IO (hPutStrLn, stderr, hFlush, stdout)
import Bit.Internal.Metadata (MetaContent(..), parseMetadata, displayHash)

-- | A conflict resolution choice: keep local or take remote.
data Resolution = KeepLocal | TakeRemote
  deriving (Show, Eq)

-- | Conflict type (mirrors Internal.Git.ConflictType but doesn't depend on IO module).
data ConflictInfo
  = ContentConflict FilePath
  | ModifyDelete FilePath DeletedSide
  | AddAdd FilePath
  deriving (Show, Eq)

-- | Get list of conflicted files
getConflictedFilesE :: IO [FilePath]
getConflictedFilesE = do
  (code, out, _) <- Git.runGitWithOutput ["diff", "--name-only", "--diff-filter=U"]
  pure $ case code of
    ExitSuccess -> filter (not . null) (lines out)
    _ -> []

-- | Detect conflict type
getConflictInfoE :: FilePath -> IO ConflictInfo
getConflictInfoE path = do
  (_, out, _) <- Git.runGitWithOutput ["ls-files", "-u", "--", path]
  pure (parseConflictInfo path out)

-- | Pure parsing of `git ls-files -u` output into ConflictInfo.
parseConflictInfo :: FilePath -> String -> ConflictInfo
parseConflictInfo path out =
  let stageNum line = case reverse (words (takeWhile (/= '\t') line)) of
        (s:_) | s `elem` ["1","2","3"] -> readMaybe s :: Maybe Int
        _ -> Nothing
      stageNums = mapMaybe stageNum (lines out)
      has1 = 1 `elem` stageNums
      has2 = 2 `elem` stageNums
      has3 = 3 `elem` stageNums
  in if | has2 && has3 && has1     -> ContentConflict path
        | has2 && has3 && not has1 -> AddAdd path
        | has2 && not has3         -> ModifyDelete path DeletedInTheirs
        | has3 && not has2         -> ModifyDelete path DeletedInOurs
        | otherwise                -> ContentConflict path

-- | Print a conflict type announcement (git-style message).
announceConflict :: ConflictInfo -> IO ()
announceConflict (ContentConflict path) =
  putStrLn $ "CONFLICT (content): Merge conflict in " ++ path
announceConflict (ModifyDelete path DeletedInOurs) =
  putStrLn $ "CONFLICT (modify/delete): " ++ path ++ " deleted in HEAD and modified in origin/main"
announceConflict (ModifyDelete path DeletedInTheirs) =
  putStrLn $ "CONFLICT (modify/delete): " ++ path ++ " deleted in origin/main and modified in HEAD"
announceConflict (AddAdd path) =
  putStrLn $ "CONFLICT (add/add): Merge conflict in " ++ path

-- | Format side info (hash + size) from metadata content retrieved via `git show :N:path`.
formatSideInfo :: ExitCode -> String -> (String, String)
formatSideInfo code content = case parseMetadata content of
  Just mc -> (displayHash (metaHash mc), show (metaSize mc))
  Nothing | code /= ExitSuccess || null content -> ("(deleted)", "-")
          | otherwise -> ("(text file)", "-")

-- | Resolve a single conflict: announce type, display info, ask user, apply choice.
resolveConflict :: Int -> Int -> FilePath -> IO Resolution
resolveConflict idx total path = do
  -- 1. Detect and announce conflict type
  cinfo <- getConflictInfoE path
  announceConflict cinfo

  -- 2. Display path with progress counter
  let displayPath = if "index/" `isPrefixOf` path then drop 6 path else path
  putStrLn $ "Conflict [" ++ show idx ++ "/" ++ show total ++ "]: " ++ displayPath

  -- 3. Show local (ours, stage 2) and remote (theirs, stage 3) metadata
  (codeO, oursOut, _)   <- Git.runGitWithOutput ["show", ":2:" ++ path]
  (codeR, theirsOut, _) <- Git.runGitWithOutput ["show", ":3:" ++ path]
  let (hashO, sizeO) = formatSideInfo codeO oursOut
  let (hashR, sizeR) = formatSideInfo codeR theirsOut
  putStrLn $ "  Local:  " ++ hashO ++ " (" ++ sizeO ++ " bytes)"
  putStrLn $ "  Remote: " ++ hashR ++ " (" ++ sizeR ++ " bytes)"

  -- 4. Ask user for choice
  putStr "  Use (l)ocal or (r)emote version? "
  hFlush stdout
  choice <- getLine
  let res = if normalize choice `elem` ["r", "remote"] then TakeRemote else KeepLocal

  -- 5. Apply resolution
  applyResolution path res
  pure res

-- | Apply a resolution to one file: git checkout ours/theirs + git add.
applyResolution :: FilePath -> Resolution -> IO ()
applyResolution path res = do
  let checkoutFlag = case res of
        KeepLocal  -> "--ours"
        TakeRemote -> "--theirs"
  code <- Git.runGitRaw ["checkout", checkoutFlag, "--", path]
  when (code /= ExitSuccess) $ hPutStrLn stderr "Warning: checkout failed."
  void $ Git.runGitRaw ["add", path]

-- | Normalize user input: trim whitespace, lowercase.
normalize :: String -> String
normalize = map toLower . dropWhileEnd isSpace . dropWhile isSpace

-- | Resolve all conflicts as a structured traversal.
-- Each conflict is visited exactly once, in order, with correct numbering.
-- Returns the list of resolutions applied.
resolveAll :: [FilePath] -> IO [Resolution]
resolveAll conflicts = do
  when (null conflicts) $ putStrLn "No unmerged paths."
  let total = length conflicts
  mapM (\(i, p) -> resolveConflict i total p) (zip [1..] conflicts)
```

---

## Bit/CopyProgress.hs

**Path:** `Bit/CopyProgress.hs`

*Source file.*

```haskell
{-# LANGUAGE BangPatterns #-}

-- | Progress reporting for file copy operations.
-- Provides chunked binary copy with byte-level progress tracking.
module Bit.CopyProgress
    ( SyncProgress(..)
    , newSyncProgress
    , copyFileWithProgress
    , withSyncProgressReporter
    , incrementFilesComplete
    ) where

import System.IO
    ( withBinaryFile, IOMode(ReadMode, WriteMode)
    , hGetBuf, hPutBuf
    , hIsTerminalDevice, hPutStrLn, stderr
    )
import Bit.Progress (reportProgress, clearProgress)
import System.Directory (createDirectoryIfMissing, copyFile)
import System.FilePath (takeDirectory)
import Data.IORef (IORef, newIORef, readIORef, atomicModifyIORef')
import Control.Concurrent (forkIO, threadDelay, killThread)
import Control.Exception (finally)
import Control.Monad (when)
import Foreign.Marshal.Alloc (allocaBytes)
import Bit.Utils (formatBytes)

-- | Shared progress state for sync operations (push/pull).
data SyncProgress = SyncProgress
    { spFilesTotal     :: !Int              -- Total files to sync
    , spFilesComplete  :: !(IORef Int)      -- Files completed so far
    , spBytesTotal     :: !(IORef Integer)  -- Total bytes to sync (sum of known file sizes)
    , spBytesCopied    :: !(IORef Integer)  -- Bytes copied so far
    , spCurrentFile    :: !(IORef String)   -- Currently copying file name
    }

-- | Create a new sync progress tracker.
newSyncProgress :: Int -> IO SyncProgress
newSyncProgress total = do
    complete <- newIORef 0
    bytesTotal <- newIORef 0
    bytesCopied <- newIORef 0
    currentFile <- newIORef ""
    pure SyncProgress
        { spFilesTotal = total
        , spFilesComplete = complete
        , spBytesTotal = bytesTotal
        , spBytesCopied = bytesCopied
        , spCurrentFile = currentFile
        }

-- | Increment the files complete counter.
incrementFilesComplete :: SyncProgress -> IO ()
incrementFilesComplete progress = 
    atomicModifyIORef' (spFilesComplete progress) (\n -> (n + 1, ()))

-- | Copy a file with progress reporting. Uses chunked binary copy for large files.
-- For files smaller than the threshold, uses plain copyFile (no progress overhead).
-- Updates the progress counters during copy.
copyFileWithProgress :: FilePath -> FilePath -> Integer -> SyncProgress -> IO ()
copyFileWithProgress src dest fileSize progress = do
    let sizeThreshold = 1024 * 1024  -- 1MB threshold
    if fileSize < sizeThreshold
        then do
            -- Small file: use plain copyFile, no chunked progress
            createDirectoryIfMissing True (takeDirectory dest)
            copyFile src dest
            -- Still update byte counter for aggregate progress
            atomicModifyIORef' (spBytesCopied progress) (\n -> (n + fileSize, ()))
        else do
            -- Large file: chunked copy with progress
            createDirectoryIfMissing True (takeDirectory dest)
            copyFileChunked src dest fileSize (spBytesCopied progress)

-- | Chunked binary copy with progress updates. Uses strict IO (no lazy ByteString).
-- Chunk size: 64KB (good balance between IO syscalls and memory usage).
copyFileChunked :: FilePath -> FilePath -> Integer -> IORef Integer -> IO ()
copyFileChunked src dest _expectedSize bytesRef = do
    let chunkSize = 64 * 1024  -- 64KB chunks
    allocaBytes chunkSize $ \buffer ->
        withBinaryFile src ReadMode $ \hIn ->
            withBinaryFile dest WriteMode $ \hOut -> do
                let loop !bytesSoFar = do
                        count <- hGetBuf hIn buffer chunkSize
                        if count == 0
                            then pure ()  -- EOF
                            else do
                                hPutBuf hOut buffer count
                                let newTotal = bytesSoFar + fromIntegral count
                                -- Update progress counter atomically
                                atomicModifyIORef' bytesRef (\n -> (n + fromIntegral count, ()))
                                loop newTotal
                loop (0 :: Integer)

-- | Start a progress reporter thread and run an action.
-- Automatically stops the reporter when the action completes.
-- Only shows progress if on a TTY and total files > threshold.
withSyncProgressReporter :: SyncProgress -> IO a -> IO a
withSyncProgressReporter progress action = do
    isTTY <- hIsTerminalDevice stderr
    let shouldShowProgress = isTTY && spFilesTotal progress > 0
    if shouldShowProgress
        then do
            reporterThread <- forkIO (syncProgressLoop progress)
            finally action $ do
                killThread reporterThread
                -- Final summary line
                filesCompleted <- readIORef (spFilesComplete progress)
                totalBytes <- readIORef (spBytesCopied progress)
                clearProgress
                hPutStrLn stderr $ "Synced " ++ show filesCompleted ++ " files (" ++ formatBytes totalBytes ++ ")."
        else do
            -- Non-TTY: print one line per file as it completes
            if spFilesTotal progress > 0
                then actionWithPerFilePrint progress action
                else action

-- | Progress reporter loop: displays aggregate and per-file progress.
syncProgressLoop :: SyncProgress -> IO ()
syncProgressLoop progress = go
  where
    go = do
        filesCompleted <- readIORef (spFilesComplete progress)
        bytesCopied <- readIORef (spBytesCopied progress)
        totalBytes <- readIORef (spBytesTotal progress)
        _currentFile <- readIORef (spCurrentFile progress)
        
        -- Use bytes for percentage when available, fall back to file count
        let pct = if totalBytes > 0
                  then fromIntegral ((bytesCopied * 100) `div` totalBytes) :: Int
                  else if spFilesTotal progress > 0
                       then (filesCompleted * 100) `div` spFilesTotal progress
                       else 0

        -- Show aggregate progress
        let progressLine = "Syncing files: " ++ show filesCompleted ++ "/" ++ show (spFilesTotal progress)
                         ++ " files, " ++ formatBytes bytesCopied
                         ++ if totalBytes > 0
                            then " / " ++ formatBytes totalBytes ++ " (" ++ show pct ++ "%)"
                            else ""
        
        reportProgress progressLine
        
        threadDelay 100000  -- 100ms update interval
        
        when (filesCompleted < spFilesTotal progress) go

-- | Run action with per-file print for non-TTY environments.
actionWithPerFilePrint :: SyncProgress -> IO a -> IO a
actionWithPerFilePrint _progress action = action
-- For non-TTY, we'd print each file as it completes, but that requires
-- hooking into each copyFileWithProgress call site. For now, just run the action.
-- The caller can print messages manually if needed.
```

---

## Bit/Core.hs

**Path:** `Bit/Core.hs`

*Source file.*

```haskell
{-# LANGUAGE ScopedTypeVariables #-}
{-# LANGUAGE OverloadedStrings #-}

module Bit.Core
    ( -- Repo initialization
      init
    , initializeRepoAt

      -- Git passthrough (these take args from the CLI)
    , add
    , commit
    , diff
    , log
    , lsFiles
    , restore
    , checkout
    , status
    , reset
    , rm
    , mv
    , branch
    , merge

      -- Core sync operations
    , push
    , pull
    , fetch

      -- Verification
    , VerifyTarget(..)
    , verify
    , fsck

      -- Remote management
    , remoteAdd
    , remoteShow
    , remoteRepair

      -- Merge management
    , mergeContinue
    , mergeAbort

      -- Branch management
    , unsetUpstream

      -- Types re-exported for Commands.hs
    , PullMode(..)
    , PullOptions(..)
    , defaultPullOptions
    ) where

import Prelude hiding (init, log)
import Bit.Core.Helpers (PullMode(..), PullOptions(..), defaultPullOptions)
import Bit.Core.Init (init, initializeRepoAt)
import Bit.Core.GitPassthrough
    ( add
    , commit
    , diff
    , log
    , lsFiles
    , restore
    , checkout
    , status
    , reset
    , rm
    , mv
    , branch
    , merge
    , mergeContinue
    , mergeAbort
    , unsetUpstream
    )
import Bit.Core.Push (push)
import Bit.Core.Pull (pull)
import Bit.Core.Fetch (fetch)
import Bit.Core.RemoteManagement (remoteAdd, remoteShow, remoteRepair)
import Bit.Core.Verify (VerifyTarget(..), verify, fsck)

```

---

## Bit/Core/Fetch.hs

**Path:** `Bit/Core/Fetch.hs`

*Source file.*

```haskell
{-# LANGUAGE ScopedTypeVariables #-}
{-# LANGUAGE OverloadedStrings #-}

module Bit.Core.Fetch
    ( fetch
    , cloudFetch
    , filesystemFetch
    , fetchRemoteBundle
    , fetchBundle
    , saveFetchedBundle
    , classifyRemoteState
    , interpretRemoteItems
    , FetchOutcome(..)
    ) where

import qualified System.Directory as Dir
import System.FilePath ((</>))
import Control.Monad (when, void)
import System.Exit (ExitCode(..), exitWith)
import qualified Internal.Git as Git
import qualified Internal.Transport as Transport
import System.IO (stderr, hPutStrLn)
import Bit.Remote (Remote, remoteName, remoteUrl, RemoteState(..), FetchResult(..))
import Bit.Types (BitM, BitEnv(..))
import Control.Monad.Trans.Reader (asks)
import Control.Monad.IO.Class (liftIO)
import Internal.Config (fromCwdPath, bundleCwdPath, fetchedBundle, bundleGitRelPath, fromGitRelPath)
import Bit.Core.Helpers (getRemoteType, withRemote, safeRemove, checkFilesystemRemoteIsRepo)
import qualified Bit.Device as Device
import System.Directory (copyFile)
import Bit.Utils (trimGitOutput)

-- ============================================================================
-- Types
-- ============================================================================

data FetchOutcome
    = UpToDate
    | Updated { foOldHash :: String, foNewHash :: String }
    | FetchedFirst String    -- new hash
    | FetchError String
    deriving (Show, Eq)

-- ============================================================================
-- Fetch operations
-- ============================================================================

fetch :: BitM ()
fetch = withRemote $ \remote -> do
    cwd <- asks envCwd

    -- Determine if this is a filesystem or cloud remote
    mType <- liftIO $ getRemoteType cwd (remoteName remote)
    case mType of
        Just t | Device.isFilesystemType t -> liftIO $ filesystemFetch cwd remote
        _ -> cloudFetch remote  -- Cloud remote or no target info (use cloud flow)

-- | Fetch from a cloud remote (original flow, unchanged).
cloudFetch :: Remote -> BitM ()
cloudFetch remote = do
    mb <- liftIO $ fetchRemoteBundle remote
    outcome <- liftIO $ saveFetchedBundle remote mb
    liftIO $ renderFetchOutcome remote outcome

-- | Fetch from a filesystem remote using named git remote.
filesystemFetch :: FilePath -> Remote -> IO ()
filesystemFetch _cwd remote = do
    let name = remoteName remote
        remotePath = remoteUrl remote
    putStrLn $ "Fetching from filesystem remote: " ++ remotePath

    -- Check if remote has .bit/ directory
    checkFilesystemRemoteIsRepo remotePath

    -- Ensure git remote URL is current (device may have moved)
    void $ Git.addRemote name (remotePath </> ".bit" </> "index")

    -- Native git fetch — handles refspec automatically
    putStrLn "Fetching remote commits..."
    (fetchCode, _fetchOut, fetchErr) <- Git.runGitWithOutput ["fetch", name]

    when (fetchCode /= ExitSuccess) $ do
        hPutStrLn stderr $ "Error fetching from remote: " ++ fetchErr
        exitWith fetchCode

    hPutStrLn stderr $ "From " ++ name
    hPutStrLn stderr $ " * [new branch]      main       -> " ++ name ++ "/main"

    putStrLn "Fetch complete."

-- ============================================================================
-- Helper functions for fetch operations
-- ============================================================================

-- | Classify remote state (empty, valid bit, non-bit, corrupted, network error)
-- This is domain logic: it knows what .bit/ means and interprets remote contents
classifyRemoteState :: Remote -> IO RemoteState
classifyRemoteState remote =
    either StateNetworkError interpretRemoteItems <$> Transport.listRemoteItems remote 1

-- | Pure interpretation of remote items into domain state
interpretRemoteItems :: [Transport.TransportItem] -> RemoteState
interpretRemoteItems items
    | null items = StateEmpty
    | ".bit" `elem` map Transport.tiName items = StateValidRgit
    | otherwise = StateNonRgitOccupied (take 3 (map Transport.tiName items))

-- | Download the remote bundle for comparison. Returns temp bundle path or error.
-- This is domain logic: it knows about .bit/ layout and bundle files
fetchBundle :: Remote -> IO FetchResult
fetchBundle remote = do
    let localDest = ".bit/temp_remote.bundle"
    
    result <- Transport.copyFromRemoteDetailed remote ".bit/bit.bundle" localDest
    case result of
        Transport.CopySuccess -> pure (BundleFound localDest)
        Transport.CopyNotFound -> pure RemoteEmpty
        Transport.CopyNetworkError _ -> 
            pure (NetworkError "Network unreachable: Check your internet connection or remote name.")
        Transport.CopyOtherError err -> pure (NetworkError err)

-- | Fetch the remote bundle, classify its state, and return path or Nothing on error.
fetchRemoteBundle :: Remote -> IO (Maybe FilePath)
fetchRemoteBundle remote = do
    -- First check remote state (this also determines if remote exists/is valid)
    remoteState <- classifyRemoteState remote
    
    case remoteState of
        StateEmpty -> do
            hPutStrLn stderr "Aborting: Remote is empty. Run 'bit push' first."
            pure Nothing
        
        StateNonRgitOccupied items -> do
            let itemList = unlines $ map ("    " ++) items
            hPutStrLn stderr $ unlines
                [ "fatal: The remote path is not empty and not a bit repository."
                , ""
                , "Found files/directories:"
                , itemList
                , ""
                , "To use a new bit remote, either choose an empty location or push"
                , "to initialize a bit repository at the remote location first."
                ]
            pure Nothing

        StateValidRgit -> do
            fetchResult <- fetchBundle remote
            case fetchResult of
                BundleFound bPath -> pure $ Just bPath
                _ -> do
                    hPutStrLn stderr $ unlines
                        [ "fatal: Could not read from remote repository."
                        , ""
                        , "Please make sure you have the correct access rights"
                        , "and the repository exists."
                        ]
                    pure Nothing

        StateNetworkError err ->
            do
                hPutStrLn stderr $ "Aborting: Network error -> " ++ err
                pure Nothing

        StateCorruptedRgit msg ->
            do
                hPutStrLn stderr $ "Aborting: [X] Corrupted remote -> " ++ msg
                pure Nothing

saveFetchedBundle :: Remote -> Maybe FilePath -> IO FetchOutcome
saveFetchedBundle _remote Nothing = pure (FetchError "No bundle to save")
saveFetchedBundle remote (Just bPath) = do
    let name = remoteName remote
        fetchedPath = fromCwdPath (bundleCwdPath fetchedBundle)
        bundleGitPath = fromGitRelPath (bundleGitRelPath fetchedBundle)

    -- Read old tracking ref hash (before overwriting)
    maybeOldHash <- revParseTrackingRef name

    -- Copy bundle to local path
    copyFile bPath fetchedPath
    safeRemove bPath

    -- Register bundle as named git remote and fetch objects + refs
    void $ Git.addRemote name bundleGitPath
    (fetchCode, _, _) <- Git.runGitWithOutput ["fetch", name]

    -- Read new tracking ref hash (populated by git fetch)
    maybeNewHash <- if fetchCode == ExitSuccess
        then revParseTrackingRef name
        else Git.getHashFromBundle fetchedBundle  -- fallback

    -- If git fetch didn't update the ref (e.g. bundle has no matching refspec),
    -- manually set it from the bundle hash
    when (fetchCode /= ExitSuccess || maybeNewHash == Nothing) $ do
        mHash <- Git.getHashFromBundle fetchedBundle
        case mHash of
            Just h  -> void $ Git.updateRemoteTrackingBranchToHash name h
            Nothing -> pure ()

    -- Determine outcome
    let effectiveNewHash = case maybeNewHash of
            Just h  -> Just h
            Nothing -> Nothing  -- will be caught below
    case (maybeOldHash, effectiveNewHash) of
        (Just oldHash, Just newHash) | oldHash == newHash -> pure UpToDate
        (Just oldHash, Just newHash) -> pure (Updated { foOldHash = oldHash, foNewHash = newHash })
        (Nothing, Just newHash) -> pure (FetchedFirst newHash)
        _ -> pure (FetchError "Could not extract hash from bundle")

-- | Read the tracking ref for a named remote. Returns Nothing if ref doesn't exist.
revParseTrackingRef :: String -> IO (Maybe String)
revParseTrackingRef name = do
    (code, out, _) <- Git.runGitWithOutput ["rev-parse", Git.remoteTrackingRef name]
    pure $ case code of
        ExitSuccess -> Just (trimGitOutput out)
        _ -> Nothing

-- | Render fetch outcome to stdout/stderr.
renderFetchOutcome :: Remote -> FetchOutcome -> IO ()
renderFetchOutcome _remote UpToDate = pure ()  -- Silent on up-to-date
renderFetchOutcome _remote (Updated { foOldHash = old, foNewHash = new }) = do
    putStrLn "Scanning remote..."
    putStrLn $ "Updated: " ++ old ++ " -> " ++ new
    putStrLn "Fetch complete."
renderFetchOutcome remote (FetchedFirst newHash) = do
    putStrLn "Scanning remote..."
    hPutStrLn stderr $ "From " ++ remoteName remote
    hPutStrLn stderr $ " * [new branch]      main       -> origin/main"
    putStrLn $ "Fetched: " ++ newHash
    putStrLn "Fetch complete."
renderFetchOutcome _remote (FetchError err) = do
    hPutStrLn stderr $ "Warning: " ++ err
```

---

## Bit/Core/GitPassthrough.hs

**Path:** `Bit/Core/GitPassthrough.hs`

*Source file.*

```haskell
{-# LANGUAGE ScopedTypeVariables #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE MultiWayIf #-}

module Bit.Core.GitPassthrough
    ( -- Git passthrough
      add
    , commit
    , diff
    , log
    , lsFiles
    , restore
    , checkout
    , status
    , reset
    , rm
    , mv
    , branch
    , merge
      -- Merge/branch management  
    , mergeContinue
    , mergeAbort
    , doMergeAbort
    , unsetUpstream
      -- Restore/checkout helpers
    , doRestore
    , doCheckout
    ) where

import Prelude hiding (log)
import qualified System.Directory as Dir
import System.FilePath ((</>), takeDirectory, equalFilePath)
import Control.Monad (when, unless, void, forM_)
import Data.Foldable (traverse_)
import Control.Monad.Trans.Class (lift)
import System.Exit (ExitCode(..), exitWith)
import Control.Exception (throwIO, IOException, catch)
import qualified Internal.Git as Git
import Internal.Config (bitIndexPath)
import System.IO (stderr, hPutStrLn, hPutStr)
import Bit.Types (BitM, BitEnv(..))
import Control.Monad.Trans.Reader (asks)
import qualified Data.List
import Control.Monad.IO.Class (liftIO)
import Data.List (isPrefixOf, foldl')
import qualified Bit.Conflict as Conflict
import qualified Bit.Device as Device
import qualified Bit.Internal.Metadata as Metadata
import Bit.Remote (remoteName, remoteUrl)
import qualified Bit.Core.Transport as Transport
import Bit.Core.Helpers
    ( fileExistsE
    , createDirE
    , copyFileE
    , readFileE
    , removeDirectoryRecursive
    , restoreCheckoutPaths
    , expandPathsToFiles
    , getLocalHeadE
    , getRemoteType
    )

-- ============================================================================
-- Git passthrough (thin wrappers)
-- ============================================================================

add :: [String] -> IO ExitCode
add args = Git.runGitRaw ("add" : args)

commit :: [String] -> IO ExitCode
commit args = Git.runGitRaw ("commit" : args)

diff :: [String] -> IO ExitCode
diff args = Git.runGitRaw ("diff" : args)

log :: [String] -> IO ExitCode
log args = Git.runGitRaw ("log" : args)

lsFiles :: [String] -> IO ExitCode
lsFiles args = Git.runGitRaw ("ls-files" : args)

reset :: [String] -> IO ExitCode
reset args = Git.runGitRaw ("reset" : args)

rm :: [String] -> BitM ExitCode
rm args = do
    cwd <- asks envCwd
    let flags = parseRmFlags args
        -- Strip quiet flags so git always outputs the file list for parsing
        gitArgs = stripQuiet args
    (code, out, err) <- liftIO $ Git.runGitWithOutput ("rm" : gitArgs)

    -- Remove actual files from working directory
    when (code == ExitSuccess && not (rmCached flags) && not (rmDryRun flags)) $ liftIO $ do
        let paths = parseRmOutput out
        forM_ paths $ \path -> do
            let actualPath = cwd </> path
            exists <- Dir.doesFileExist actualPath
            when exists $
                Dir.removeFile actualPath `catch` \(e :: IOException) ->
                    hPutStrLn stderr $ "warning: failed to remove " ++ path ++ ": " ++ show e
        -- Clean up empty parent directories
        forM_ paths $ \path ->
            removeEmptyParents cwd (takeDirectory (cwd </> path))

    -- Print output
    liftIO $ do
        unless (rmQuiet flags) $
            putStr (Git.rewriteGitHints out)
        hPutStr stderr (Git.rewriteGitHints err)
        case code of
            ExitSuccess -> pure ()
            ExitFailure n -> hPutStrLn stderr ("bit: git exited with code " ++ show n)

    pure code

mv :: [String] -> IO ExitCode
mv args = Git.runGitRaw ("mv" : args)

branch :: [String] -> IO ExitCode
branch args = Git.runGitRaw ("branch" : args)

merge :: [String] -> IO ExitCode
merge args = Git.runGitRaw ("merge" : args)

-- ============================================================================
-- Stateful passthrough (needs BitEnv)
-- ============================================================================

status :: [String] -> BitM ExitCode
status args = do
    liftIO $ Git.runGitRaw ("status" : args)

restore :: [String] -> BitM ExitCode
restore = doRestore

checkout :: [String] -> BitM ExitCode
checkout = doCheckout

-- ============================================================================
-- Merge/branch management
-- ============================================================================

mergeContinue :: BitM ()
mergeContinue = do
    cwd <- asks envCwd
    mRemote <- asks envRemote
    let conflictsDir = cwd </> ".bit" </> "conflicts"
    conflictsExist <- liftIO $ Dir.doesDirectoryExist conflictsDir

    gitConflicts <- liftIO Conflict.getConflictedFilesE

    if | not (null gitConflicts) ->
            liftIO $ hPutStrLn stderr "error: you have not resolved your conflicts yet."
       | not conflictsExist -> do
            (code, _, _) <- liftIO $ Git.runGitWithOutput ["rev-parse", "--verify", "MERGE_HEAD"]
            case code of
                ExitSuccess -> do
                    oldHead <- liftIO getLocalHeadE
                    liftIO $ do
                        void $ Git.runGitRaw ["commit", "-m", "Merge remote"]
                        putStrLn "Merge complete."
                    traverse_ (\remote -> do
                            mType <- liftIO $ getRemoteType cwd (remoteName remote)
                            let transport = case mType of
                                  Just t | Device.isFilesystemType t -> Transport.mkFilesystemTransport (remoteUrl remote)
                                  _ -> Transport.mkCloudTransport remote
                            Transport.syncBinariesAfterMerge transport remote oldHead) mRemote
                _ -> liftIO $ do
                    hPutStrLn stderr "error: no merge in progress."
                    exitWith (ExitFailure 1)
       | otherwise -> do
                invalid <- liftIO $ Metadata.validateMetadataDir (cwd </> bitIndexPath)
                unless (null invalid) $ liftIO $ do
                    hPutStrLn stderr "fatal: Metadata files contain conflict markers. Merge aborted."
                    throwIO (userError "Invalid metadata")

                oldHead <- liftIO getLocalHeadE
                (code, _, _) <- liftIO $ Git.runGitWithOutput ["rev-parse", "--verify", "MERGE_HEAD"]
                when (code /= ExitSuccess) $ do
                    -- Use actual remote name if available, fall back to "origin"
                    let remName = maybe "origin" remoteName mRemote
                    (mergeCode, _, _) <- liftIO $ Git.runGitWithOutput ["merge", "--no-commit", "--no-ff", Git.remoteTrackingRef remName]
                    when (mergeCode /= ExitSuccess) $
                        liftIO $ hPutStrLn stderr "warning: Could not start merge. Proceeding anyway."

                liftIO $ do
                    void $ Git.runGitRaw ["commit", "-m", "Merge remote (manual merge resolved)"]
                    putStrLn "Merge complete."
                    removeDirectoryRecursive conflictsDir
                    putStrLn "Conflict directories cleaned up."

                traverse_ (\remote -> do
                        mType <- liftIO $ getRemoteType cwd (remoteName remote)
                        let transport = case mType of
                              Just t | Device.isFilesystemType t -> Transport.mkFilesystemTransport (remoteUrl remote)
                              _ -> Transport.mkCloudTransport remote
                        Transport.syncBinariesAfterMerge transport remote oldHead) mRemote

mergeAbort :: IO ()
mergeAbort = doMergeAbort

doMergeAbort :: IO ()
doMergeAbort = do
    cwd <- Dir.getCurrentDirectory
    let conflictsDir = cwd </> ".bit" </> "conflicts"
    
    -- Abort git merge
    code <- Git.mergeAbort
    case code of
        ExitSuccess -> do
            putStrLn "Merge aborted. Your working tree is unchanged."
            
            -- Clean up conflict directories
            conflictsExist <- Dir.doesDirectoryExist conflictsDir
            when conflictsExist $ do
                removeDirectoryRecursive conflictsDir
                putStrLn "Conflict directories cleaned up."
        _ -> do
            hPutStrLn stderr "error: no merge in progress."
            exitWith (ExitFailure 1)

unsetUpstream :: IO ()
unsetUpstream = void Git.unsetBranchUpstream

-- ============================================================================
-- Restore and Checkout implementations
-- ============================================================================

doRestore :: [String] -> BitM ExitCode
doRestore args = do
    cwd <- asks envCwd
    code <- lift $ Git.runGitRaw ("restore" : args)
    when (code == ExitSuccess) $ do
        let stagedOnly = ("--staged" `elem` args || "-S" `elem` args) &&
                         not ("--worktree" `elem` args || "-W" `elem` args)
        unless stagedOnly $
            syncTextFilesFromIndex cwd (restoreCheckoutPaths args)
    pure code

doCheckout :: [String] -> BitM ExitCode
doCheckout args = do
    let args' = case Data.List.elemIndex "--" args of
          Just _ -> args
          Nothing -> let (opts, paths) = Data.List.span (\a -> a == "--" || "-" `isPrefixOf` a) args
                     in opts ++ ["--"] ++ paths
    code <- lift $ Git.runGitRaw ("checkout" : args')
    when (code == ExitSuccess) $ do
        cwd <- asks envCwd
        syncTextFilesFromIndex cwd (restoreCheckoutPaths args')
    pure code

-- | After a successful restore/checkout, copy text files from .bit/index/ back
-- to the working directory. Binary metadata files are left alone.
syncTextFilesFromIndex :: FilePath -> [FilePath] -> BitM ()
syncTextFilesFromIndex cwd rawPaths = do
    paths <- lift $ expandPathsToFiles cwd rawPaths
    forM_ paths $ \filePath -> do
        let metaPath = cwd </> bitIndexPath </> filePath
        let workPath = cwd </> filePath
        metaExists <- lift $ fileExistsE metaPath
        when metaExists $ do
            mcontent <- lift $ readFileE metaPath
            let isBinaryMetadata = maybe True (\content -> any ("hash: " `isPrefixOf`) (lines content)) mcontent
            unless isBinaryMetadata $ lift $ do
                createDirE (takeDirectory workPath)
                copyFileE metaPath workPath

-- ============================================================================
-- rm helpers
-- ============================================================================

data RmFlags = RmFlags
    { rmCached :: Bool
    , rmDryRun :: Bool
    , rmQuiet  :: Bool
    }

-- | Parse rm-relevant flags from args, respecting @--@ separator.
parseRmFlags :: [String] -> RmFlags
parseRmFlags args =
    let (flagPart, _) = break (== "--") args
    in foldl' checkArg (RmFlags False False False) flagPart
  where
    checkArg flags "--cached"  = flags { rmCached = True }
    checkArg flags "--dry-run" = flags { rmDryRun = True }
    checkArg flags "--quiet"   = flags { rmQuiet = True }
    checkArg flags arg
        | "-" `isPrefixOf` arg && not ("--" `isPrefixOf` arg) =
            flags { rmDryRun = rmDryRun flags || 'n' `elem` drop 1 arg
                  , rmQuiet  = rmQuiet flags || 'q' `elem` drop 1 arg
                  }
    checkArg flags _ = flags

-- | Strip @-q@/@--quiet@ from args (before @--@) so git always outputs file list.
stripQuiet :: [String] -> [String]
stripQuiet args =
    let (before, after) = break (== "--") args
    in concatMap stripArg before ++ after
  where
    stripArg "--quiet" = []
    stripArg arg
        | "-" `isPrefixOf` arg && not ("--" `isPrefixOf` arg) =
            let stripped = filter (/= 'q') (drop 1 arg)
            in if null stripped then [] else ['-' : stripped]
    stripArg arg = [arg]

-- | Parse @rm 'path'@ lines from git rm output.
parseRmOutput :: String -> [FilePath]
parseRmOutput out =
    [ take (length path - 1) path
    | line <- lines out
    , "rm '" `isPrefixOf` line
    , let path = drop 4 line
    , not (null path)
    ]

-- | Remove empty parent directories up to (but not including) @stopAt@.
removeEmptyParents :: FilePath -> FilePath -> IO ()
removeEmptyParents stopAt dir = go dir `catch` \(_ :: IOException) -> pure ()
  where
    go d | equalFilePath d stopAt = pure ()
         | equalFilePath d (takeDirectory d) = pure ()
         | otherwise = do
              contents <- Dir.listDirectory d
              when (null contents) $ do
                  Dir.removeDirectory d
                  go (takeDirectory d)
```

---

## Bit/Core/Helpers.hs

**Path:** `Bit/Core/Helpers.hs`

*Source file.*

```haskell
{-# LANGUAGE ScopedTypeVariables #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE LambdaCase #-}
{-# LANGUAGE MultiWayIf #-}

module Bit.Core.Helpers
    ( -- Types
      PullMode(..)
    , PullOptions(..)
    , defaultPullOptions
      -- Git helpers
    , AncestorQuery(..)
    , getLocalHeadE
    , checkIsAheadE
    , hasStagedChangesE
    , getRemoteType
    , getRemoteTargetType
    , checkFilesystemRemoteIsRepo
      -- Monadic helpers
    , withRemote
    , gitQuery
    , gitRaw
    , tell
    , tellErr
    , fileExistsE
    , createDirE
    , readFileE
    , writeFileAtomicE
    , copyFileE
      -- Utility functions
    , safeRemove
    , formatPathList
    , printVerifyIssue
    , readFileMaybe
    , removeDirectoryRecursive
    , restoreCheckoutPaths
    , expandPathsToFiles
    ) where

import qualified System.Directory as Dir
import System.Directory (copyFile, removeFile, createDirectoryIfMissing, removeDirectory, listDirectory, doesDirectoryExist)
import System.FilePath ((</>), normalise)
import Control.Monad (when, unless, forM_)
import System.Exit (ExitCode(..), exitWith)
import Internal.Git (AncestorQuery(..))
import qualified Internal.Git as Git
import qualified Bit.Device as Device
import Bit.Remote (Remote)
import Data.List (isPrefixOf, foldl')
import System.IO (stderr, hPutStrLn)
import Bit.Types (BitM, BitEnv(..), unPath)
import Control.Monad.Trans.Reader (asks)
import Control.Monad.IO.Class (liftIO)
import Bit.Utils (toPosix, atomicWriteFileStr, trimGitOutput)
import qualified Bit.Verify as Verify
import qualified Bit.Scan as Scan
import Internal.Config (bitIndexPath)
-- Strict IO imports to avoid Windows file locking issues
import qualified Data.ByteString as BS
import qualified Data.Text as T
import Data.Text.Encoding (decodeUtf8')

-- ============================================================================
-- Types
-- ============================================================================

-- | How to handle the merge during pull.
data PullMode
    = PullNormal        -- ^ Normal merge (fast-forward or three-way)
    | PullAcceptRemote  -- ^ Force-checkout remote branch
    | PullManualMerge   -- ^ Interactive per-file conflict resolution
    deriving (Show, Eq)

newtype PullOptions = PullOptions
    { pullMode       :: PullMode
    } deriving (Show)

defaultPullOptions :: PullOptions
defaultPullOptions = PullOptions PullNormal

-- ============================================================================
-- Git helpers via effect layer
-- ============================================================================

getLocalHeadE :: IO (Maybe String)
getLocalHeadE = do
    (code, out, _) <- Git.runGitWithOutput ["rev-parse", "HEAD"]
    pure $ case code of
        ExitSuccess -> Just (trimGitOutput out)
        _ -> Nothing

-- | Check if aqDescendant is ahead of aqAncestor (i.e., aqAncestor is an ancestor of aqDescendant).
checkIsAheadE :: AncestorQuery -> IO Bool
checkIsAheadE (AncestorQuery ancestor descendant) =
    (\(code, _, _) -> code == ExitSuccess) <$>
    Git.runGitWithOutput ["merge-base", "--is-ancestor", ancestor, descendant]

hasStagedChangesE :: IO Bool
hasStagedChangesE =
    (\(code, _, _) -> code == ExitFailure 1) <$>
    Git.runGitWithOutput ["diff", "--cached", "--quiet"]

-- | Determine the remote type from a remote name.
-- Returns the RemoteType if the remote is configured, Nothing otherwise.
getRemoteType :: FilePath -> String -> IO (Maybe Device.RemoteType)
getRemoteType cwd remName = Device.readRemoteType cwd remName

-- | Determine the remote target type from a remote name (legacy, for backward compat).
-- Returns the RemoteTarget if the remote is configured, Nothing otherwise.
getRemoteTargetType :: FilePath -> String -> IO (Maybe Device.RemoteTarget)
getRemoteTargetType cwd remName = Device.readRemoteFile cwd remName

-- | Check if a filesystem path is a bit repository. Exits with error if not.
-- Used by filesystem push, pull, and fetch operations to validate the remote.
checkFilesystemRemoteIsRepo :: FilePath -> IO ()
checkFilesystemRemoteIsRepo remotePath = do
    let remoteBitDir = remotePath </> ".bit"
    remoteHasBit <- Dir.doesDirectoryExist remoteBitDir
    unless remoteHasBit $ do
        hPutStrLn stderr "error: Remote is not a bit repository."
        exitWith (ExitFailure 1)

-- ============================================================================
-- Compatibility helpers (effect system removal)
-- ============================================================================

tell :: String -> IO ()
tell = putStrLn

tellErr :: String -> IO ()
tellErr = hPutStrLn stderr

gitRaw :: [String] -> IO ExitCode
gitRaw = Git.runGitRaw

gitQuery :: [String] -> IO (ExitCode, String, String)
gitQuery = Git.runGitWithOutput

readFileE :: FilePath -> IO (Maybe String)
readFileE = readFileMaybe

writeFileAtomicE :: FilePath -> String -> IO ()
writeFileAtomicE = atomicWriteFileStr

copyFileE :: FilePath -> FilePath -> IO ()
copyFileE = copyFile

fileExistsE :: FilePath -> IO Bool
fileExistsE = Dir.doesFileExist

createDirE :: FilePath -> IO ()
createDirE = createDirectoryIfMissing True

-- | Run an action with the remote, or print error if not configured.
withRemote :: (Remote -> BitM ()) -> BitM ()
withRemote action = do
  mRemote <- asks envRemote
  case mRemote of
    Nothing -> liftIO $ do
        hPutStrLn stderr "fatal: No upstream configured and no remote specified."
        hPutStrLn stderr "hint: bit push <remote>"
        hPutStrLn stderr "hint: bit push -u <remote>    (to set default upstream)"
        exitWith (ExitFailure 1)
    Just remote -> action remote

-- ============================================================================
-- Utility functions
-- ============================================================================

safeRemove :: FilePath -> IO ()
safeRemove filePath = do
    exists <- Dir.doesFileExist filePath
    when exists (Dir.removeFile filePath)

formatPathList :: [FilePath] -> [String]
formatPathList paths
  | length paths <= 20 = map (\p -> "        " ++ toPosix p) paths
  | otherwise         = map (\p -> "        " ++ toPosix p) (take 10 paths)
                        ++ ["        ... and " ++ show (length paths - 10) ++ " more"]

printVerifyIssue :: (String -> String) -> Verify.VerifyIssue -> IO ()
printVerifyIssue fmtHash = \case
  Verify.HashMismatch filePath expectedHash actualHash _expectedSize _actualSize -> do
    hPutStrLn stderr $ "[ERROR] Hash mismatch: " ++ toPosix (unPath filePath)
    hPutStrLn stderr $ "  Expected: " ++ fmtHash expectedHash
    hPutStrLn stderr $ "  Actual:   " ++ fmtHash actualHash
  Verify.Missing filePath ->
    hPutStrLn stderr $ "[ERROR] Missing: " ++ toPosix (unPath filePath)

readFileMaybe :: FilePath -> IO (Maybe String)
readFileMaybe filePath = do
    exists <- Dir.doesFileExist filePath
    if exists
        then do
            bs <- BS.readFile filePath
            pure $ either (const Nothing) (Just . T.unpack) (decodeUtf8' bs)
        else pure Nothing

removeDirectoryRecursive :: FilePath -> IO ()
removeDirectoryRecursive dir = do
    exists <- doesDirectoryExist dir
    when exists $ do
        contents <- listDirectory dir
        forM_ contents $ \item -> do
            let itemPath = dir </> item
            isDir <- doesDirectoryExist itemPath
            if isDir
                then removeDirectoryRecursive itemPath
                else removeFile itemPath
        removeDirectory dir

restoreCheckoutPaths :: [String] -> [String]
restoreCheckoutPaths args =
    let restoreFlags = ["--staged", "-S", "--worktree", "-W",
                        "--patch", "-p", "--quiet", "-q",
                        "--ours", "--theirs", "--merge", "-m",
                        "--pathspec-file-nul", "--overlay", "--no-overlay",
                        "--ignore-unmerged", "--recurse-submodules", "--no-recurse-submodules"]
        isFlag arg = arg `elem` restoreFlags ||
                     arg == "--" ||
                     "--source=" `isPrefixOf` arg ||
                     "-s" `isPrefixOf` arg ||
                     "--pathspec-from-file=" `isPrefixOf` arg ||
                     "--conflict=" `isPrefixOf` arg ||
                     "--inter-hunk-context=" `isPrefixOf` arg ||
                     "--unified=" `isPrefixOf` arg ||
                     "-U" `isPrefixOf` arg
        (_, paths) = foldl' (\(afterDash, acc) arg ->
            if | arg == "--" -> (True, acc)
               | afterDash   -> (True, arg:acc)
               | isFlag arg  -> (False, acc)
               | otherwise   -> (False, arg:acc)
            ) (False, []) args
    in reverse paths

expandPathsToFiles :: FilePath -> [String] -> IO [FilePath]
expandPathsToFiles cwd paths = do
    let indexRoot = cwd </> bitIndexPath
    allFiles <- Scan.listMetadataPaths indexRoot
    pure $ concatMap (\p ->
        if p == "." || p == "./"
        then allFiles
        else let p' = normalise p
                 pPrefix = p' ++ "/"
                 matches = filter (\f -> let f' = normalise f
                                         in f' == p' || pPrefix `isPrefixOf` (f' ++ "/")) allFiles
             in if null matches then [p] else matches
        ) paths
```

---

## Bit/Core/Init.hs

**Path:** `Bit/Core/Init.hs`

*Source file.*

```haskell
{-# LANGUAGE ScopedTypeVariables #-}
{-# LANGUAGE OverloadedStrings #-}

module Bit.Core.Init
    ( init
    , initializeRepoAt
    ) where

import Prelude hiding (init)
import qualified System.Directory as Dir
import System.FilePath ((</>))
import Control.Monad (unless, void)
import qualified Internal.Git as Git
import System.Process (readProcessWithExitCode)
import Bit.Utils (atomicWriteFileStr, toPosix)

init :: IO ()
init = initializeRepo

initializeRepo :: IO ()
initializeRepo = do
    cwd <- Dir.getCurrentDirectory
    putStrLn $ "Initializing bit in: " ++ cwd
    initializeRepoAt cwd
    putStrLn "bit initialized successfully!"

-- | Initialize a bit repository at the specified target directory.
-- This is used both for local `bit init` and for creating filesystem remotes.
initializeRepoAt :: FilePath -> IO ()
initializeRepoAt targetDir = do
    let targetBitDir = targetDir </> ".bit"
    let targetBitIndexPath = targetBitDir </> "index"
    let targetBitGitDir = targetBitIndexPath </> ".git"
    let targetBitDevicesDir = targetBitDir </> "devices"
    let targetBitRemotesDir = targetBitDir </> "remotes"

    -- 1. Create .bit directory
    Dir.createDirectoryIfMissing True targetBitDir

    -- 2. Create .bit/index directory (needed before git init)
    Dir.createDirectoryIfMissing True targetBitIndexPath

    -- 3. Init Git in the index directory
    hasGit <- Dir.doesDirectoryExist targetBitGitDir
    unless hasGit $ do
        -- Initialize git in .bit/index, which will create .bit/index/.git
        void $ Git.runGitAt targetBitIndexPath ["init"]
        
        -- Fix for Windows external/USB drives: add to safe.directory
        -- git 2.35.2+ rejects directories with different ownership
        absIndex <- Dir.makeAbsolute targetBitIndexPath
        let safePath = toPosix absIndex
        void $ readProcessWithExitCode "git" ["config", "--global", "--add", "safe.directory", safePath] ""

    -- 3a. Create .git/bundles directory for storing bundle files
    Dir.createDirectoryIfMissing True (targetBitGitDir </> "bundles")

    -- 4. Configure default branch name to "main" (for the repo we just created)
    void $ Git.runGitAt targetBitIndexPath ["config", "init.defaultBranch", "main"]
    
    -- 4a. Configure core.quotePath to false (display Unicode filenames properly)
    void $ Git.runGitAt targetBitIndexPath ["config", "core.quotePath", "false"]
    
    -- 5. Rename the initial branch to "main" if it's "master"
    void $ Git.runGitAt targetBitIndexPath ["branch", "-m", "master", "main"]

    -- 6. Create other .bit subdirectories
    Dir.createDirectoryIfMissing True targetBitDevicesDir
    Dir.createDirectoryIfMissing True targetBitRemotesDir

    -- 5a. Create config file with default values
    let configPath = targetBitDir </> "config"
    configExists <- Dir.doesFileExist configPath
    unless configExists $ do
        let defaultConfig = unlines
                [ "[text]"
                , "    size-limit = 1048576  # 1MB, files larger are always binary"
                , "    extensions = .txt,.md,.yaml,.yml,.json,.xml,.html,.css,.js,.py,.hs,.rs"
                ]
        atomicWriteFileStr configPath defaultConfig

    -- 5b. Merge driver: prevent Git from writing conflict markers
    void $ Git.runGitAt targetBitIndexPath ["config", "merge.bit-metadata.name", "bit metadata"]
    void $ Git.runGitAt targetBitIndexPath ["config", "merge.bit-metadata.driver", "false"]
    Dir.createDirectoryIfMissing True (targetBitGitDir </> "info")
    atomicWriteFileStr (targetBitGitDir </> "info" </> "attributes") "* merge=bit-metadata -text\n"
```

---

## Bit/Core/Pull.hs

**Path:** `Bit/Core/Pull.hs`

*Source file.*

```haskell
{-# LANGUAGE ScopedTypeVariables #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE LambdaCase #-}
{-# LANGUAGE TypeApplications #-}
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE DuplicateRecordFields #-}
{-# LANGUAGE OverloadedRecordDot #-}

module Bit.Core.Pull
    ( pull
    , cloudPull
    , filesystemPull
    , filesystemPullLogicImpl
    , filesystemPullAcceptRemoteImpl
    , pullAcceptRemoteImpl
    , pullManualMergeImpl
    , pullWithCleanup
    , pullLogic
    , DivergentFile(..)
    , findDivergentFiles
    , createConflictDirectories
    , printConflictList
    ) where

import Prelude hiding (log)
import System.FilePath ((</>), normalise, takeDirectory)
import Control.Monad (when, unless, void, forM_)
import Data.Foldable (traverse_)
import System.Exit (ExitCode(..), exitWith)
import qualified Internal.Git as Git
import qualified Internal.Transport as Transport
import Internal.Config (bitIndexPath, fetchedBundle)
import qualified Bit.Scan as Scan
import qualified Bit.Verify as Verify
import qualified Bit.Conflict as Conflict
import qualified Bit.Remote.Scan as Remote.Scan
import qualified Data.List as List
import qualified Data.Map as Map
import System.IO (stderr, hPutStrLn)
import Control.Exception (try, SomeException, throwIO)
import Bit.Utils (toPosix, filterOutBitPaths, trimGitOutput)
import Data.Maybe (maybeToList)
import Bit.Remote (Remote, remoteName, remoteUrl)
import Bit.Types (BitM, BitEnv(..), ForceMode(..), Hash, HashAlgo(..), EntryKind(..), syncHash, runBitM, unPath)
import Control.Monad.Trans.Reader (asks)
import Control.Monad.IO.Class (liftIO)
import Control.Monad.Trans.Class (lift)
import Bit.Internal.Metadata (MetaContent(..), serializeMetadata, displayHash, validateMetadataDir)
import Bit.Concurrency (Concurrency(..))
import qualified Bit.Device as Device
import Bit.Core.Helpers
    ( PullMode(..)
    , PullOptions(..)
    , defaultPullOptions
    , getRemoteType
    , withRemote
    , getLocalHeadE
    , hasStagedChangesE
    , gitQuery
    , gitRaw
    , tell
    , tellErr
    , fileExistsE
    , createDirE
    , copyFileE
    , writeFileAtomicE
    , printVerifyIssue
    , checkFilesystemRemoteIsRepo
    )
import Bit.Core.Transport
    ( FileTransport
    , mkCloudTransport
    , mkFilesystemTransport
    , applyMergeToWorkingDir
    , transportSyncAllFiles
    )
import Bit.Core.Fetch (fetchRemoteBundle, saveFetchedBundle, FetchOutcome(..))

-- ============================================================================
-- Pull operations
-- ============================================================================

pull :: PullOptions -> BitM ()
pull opts = withRemote $ \remote -> do
    cwd <- asks envCwd
    
    -- Determine if this is a filesystem or cloud remote
    mType <- liftIO $ getRemoteType cwd (remoteName remote)
    case mType of
        Just t | Device.isFilesystemType t -> liftIO $ filesystemPull cwd remote opts
        _ -> cloudPull remote opts  -- Cloud remote or no target info (use cloud flow)

-- | Pull from a cloud remote (uses unified transport abstraction).
cloudPull :: Remote -> PullOptions -> BitM ()
cloudPull remote opts =
    let transport = mkCloudTransport remote
    in case pullMode opts of
        PullAcceptRemote -> pullAcceptRemoteImpl transport remote
        PullManualMerge  -> pullManualMergeImpl remote
        PullNormal       -> pullWithCleanup transport remote opts

-- | Pull from a filesystem remote using named git remote.
filesystemPull :: FilePath -> Remote -> PullOptions -> IO ()
filesystemPull cwd remote opts = do
    let name = remoteName remote
        remotePath = remoteUrl remote
    putStrLn $ "Pulling from filesystem remote: " ++ remotePath

    -- Check if remote has .bit/ directory
    checkFilesystemRemoteIsRepo remotePath

    -- 1. Ensure git remote URL is current and fetch
    void $ Git.addRemote name (remotePath </> ".bit" </> "index")

    putStrLn "Fetching remote commits..."
    (fetchCode, _fetchOut, fetchErr) <- Git.runGitWithOutput ["fetch", name]

    when (fetchCode /= ExitSuccess) $ do
        hPutStrLn stderr $ "Error fetching from remote: " ++ fetchErr
        exitWith fetchCode

    hPutStrLn stderr $ "From " ++ name
    hPutStrLn stderr $ " * [new branch]      main       -> " ++ name ++ "/main"

    -- 3. Get remote HEAD hash
    (remoteHeadCode, remoteHeadOut, _) <- Git.runGitWithOutput ["rev-parse", Git.remoteTrackingRef name]
    when (remoteHeadCode /= ExitSuccess) $ do
        hPutStrLn stderr "Error: Could not get remote HEAD"
        exitWith (ExitFailure 1)

    let remoteHash = trimGitOutput remoteHeadOut
    
    -- Proof of possession — always verify filesystem remote before pulling
    unless (pullMode opts == PullAcceptRemote) $ do
        putStrLn "Verifying remote repository..."
        result <- Verify.verifyLocalAt remotePath Nothing (Parallel 0)
        if null result.vrIssues
            then putStrLn $ "Verified " ++ show result.vrCount ++ " remote files."
            else do
                hPutStrLn stderr $ "error: Remote working tree does not match remote metadata (" ++ show (length result.vrIssues) ++ " issues)."
                mapM_ (printVerifyIssue id) result.vrIssues
                hPutStrLn stderr "hint: Run 'bit verify' in the remote repo to see all mismatches."
                hPutStrLn stderr "hint: Run 'bit pull --accept-remote' to accept the remote's actual state."
                exitWith (ExitFailure 1)
    
    -- 4. Build transport and delegate to unified pull logic
    let transport = mkFilesystemTransport remotePath
    
    -- Create a minimal BitEnv to call the shared logic
    localFiles <- Scan.scanWorkingDir cwd
    let env = BitEnv cwd localFiles (Just remote) NoForce
    
    -- Delegate to the unified path
    case pullMode opts of
        PullAcceptRemote -> runBitM env (filesystemPullAcceptRemoteImpl transport name remoteHash)
        _                -> runBitM env (filesystemPullLogicImpl transport remote remoteHash)

-- | Filesystem pull logic (simplified - no bundle fetching, just merge + sync)
filesystemPullLogicImpl :: FileTransport -> Remote -> String -> BitM ()
filesystemPullLogicImpl transport remote remoteHash = do
    cwd <- asks envCwd
    oldHash <- lift getLocalHeadE
    let name = remoteName remote

    case oldHash of
        Nothing -> do
            lift $ putStrLn $ "Checking out " ++ take 7 remoteHash ++ " (first pull)"
            checkoutCode <- lift $ Git.checkoutRemoteAsMain name
            case checkoutCode of
                ExitSuccess -> lift $ do
                    transportSyncAllFiles transport cwd
                    putStrLn "Syncing binaries... done."
                    void $ Git.updateRemoteTrackingBranchToHash name remoteHash
                _ -> lift $ hPutStrLn stderr "Error: Failed to checkout remote branch."

        Just localHash -> do
            (mergeCode, mergeOut, mergeErr) <- lift $ Git.runGitWithOutput
                ["merge", "--no-commit", "--no-ff", Git.remoteTrackingRef name]

            (finalMergeCode, finalMergeOut, finalMergeErr) <-
                lift $ if mergeCode /= ExitSuccess && "refusing to merge unrelated histories" `List.isInfixOf` (mergeOut ++ mergeErr)
                then do
                    putStrLn "Merging unrelated histories..."
                    Git.runGitWithOutput ["merge", "--no-commit", "--no-ff", "--allow-unrelated-histories", Git.remoteTrackingRef name]
                else pure (mergeCode, mergeOut, mergeErr)

            case finalMergeCode of
                ExitSuccess -> do
                    lift $ putStrLn $ "Updating " ++ take 7 localHash ++ ".." ++ take 7 remoteHash
                    lift $ putStrLn "Merge made by the 'recursive' strategy."
                    hasChanges <- lift hasStagedChangesE
                    when hasChanges $ lift $ void $ Git.runGitRaw ["commit", "-m", "Merge remote"]
                    -- CRITICAL: Always read actual HEAD after merge, never use remoteHash
                    lift $ applyMergeToWorkingDir transport cwd localHash
                    lift $ putStrLn "Syncing binaries... done."
                    lift $ void $ Git.updateRemoteTrackingBranchToHash name remoteHash
                _ -> do
                    lift $ do
                        putStrLn finalMergeOut
                        hPutStrLn stderr finalMergeErr
                        putStrLn "Automatic merge failed."
                        putStrLn "bit requires you to pick a version for each conflict."
                        putStrLn ""
                        putStrLn "Resolving conflicts..."

                    conflicts <- lift Conflict.getConflictedFilesE
                    resolutions <- lift $ Conflict.resolveAll conflicts
                    let total = length resolutions

                    invalid <- lift $ validateMetadataDir (cwd </> bitIndexPath)
                    unless (null invalid) $ lift $ do
                        void $ Git.runGitRaw ["merge", "--abort"]
                        hPutStrLn stderr "fatal: Metadata files contain conflict markers. Merge aborted."
                        throwIO (userError "Invalid metadata")

                    conflictsNow <- lift Conflict.getConflictedFilesE
                    when (null conflictsNow) $ lift $ do
                        void $ Git.runGitRaw ["commit", "-m", "Merge remote (resolved " ++ show total ++ " conflict(s))"]
                        putStrLn $ "Merge complete. " ++ show total ++ " conflict(s) resolved."
                        -- CRITICAL: Always read actual HEAD after merge, never use remoteHash
                        applyMergeToWorkingDir transport cwd localHash
                        putStrLn "Syncing binaries... done."
                        void $ Git.updateRemoteTrackingBranchToHash name remoteHash

-- | Filesystem pull --accept-remote implementation
filesystemPullAcceptRemoteImpl :: FileTransport -> String -> String -> BitM ()
filesystemPullAcceptRemoteImpl transport name remoteHash = do
    cwd <- asks envCwd
    lift $ putStrLn "Accepting remote file state as truth..."

    -- Record current HEAD before checkout
    oldHead <- lift getLocalHeadE

    -- Force-checkout the remote branch
    checkoutCode <- lift $ Git.checkoutRemoteAsMain name
    case checkoutCode of
        ExitSuccess -> do
            -- Sync actual files based on what changed
            maybe (lift $ transportSyncAllFiles transport cwd)
                  (\oh -> lift $ applyMergeToWorkingDir transport cwd oh) oldHead

            -- Update tracking ref
            lift $ do
                void $ Git.updateRemoteTrackingBranchToHash name remoteHash
                putStrLn "Pull with --accept-remote completed."
        _ -> lift $ hPutStrLn stderr "Error: Failed to checkout remote state."

-- | Pull with --accept-remote: force-checkout the remote branch, then sync files.
-- Git manages .bit/index/ (the metadata); we only sync actual files to the working tree.
pullAcceptRemoteImpl :: FileTransport -> Remote -> BitM ()
pullAcceptRemoteImpl transport remote = do
    cwd <- asks envCwd
    let name = remoteName remote
    lift $ tell "Accepting remote file state as truth..."

    -- 1. Fetch the remote bundle so git has the remote's history
    maybeBundlePath <- lift $ fetchRemoteBundle remote
    case maybeBundlePath of
        Nothing -> lift $ tellErr "Error: Could not fetch remote bundle."
        Just bPath -> do
            outcome <- lift $ saveFetchedBundle remote (Just bPath)
            case outcome of
                FetchError err -> lift $ tellErr $ "Error: " ++ err
                _ -> pure ()  -- No need to render fetch output during pull

            -- 2. Record current HEAD before checkout (for diff-based sync)
            oldHead <- lift getLocalHeadE

            -- 3. Force-checkout the remote branch.
            lift $ tell "Scanning remote files..."
            checkoutCode <- lift $ Git.checkoutRemoteAsMain name
            case checkoutCode of
                ExitSuccess -> do
                    -- 4. Sync actual files to working tree based on what changed in git
                    (_remoteCode, remoteOut, _) <- lift $ gitQuery ["rev-parse", Git.remoteTrackingRef name]
                    let _newHash = takeWhile (/= '\n') remoteOut
                    maybe (lift $ transportSyncAllFiles transport cwd)
                          (\oh -> lift $ applyMergeToWorkingDir transport cwd oh) oldHead

                    -- 5. Update tracking ref
                    maybeRemoteHash <- lift $ Git.getHashFromBundle fetchedBundle
                    lift $ traverse_ (void . Git.updateRemoteTrackingBranchToHash name) maybeRemoteHash

                    lift $ tell "Pull with --accept-remote completed."
                _ -> lift $ tellErr "Error: Failed to checkout remote state."

-- | Pull with --manual-merge: detect remote divergence and create conflict directories.
pullManualMergeImpl :: Remote -> BitM ()
pullManualMergeImpl remote = do
    cwd <- asks envCwd
    let name = remoteName remote
    lift $ tell "Fetching remote metadata... done."

    maybeBundlePath <- lift $ fetchRemoteBundle remote
    case maybeBundlePath of
        Nothing -> lift $ tellErr "Error: Could not fetch remote bundle."
        Just bPath -> do
            outcome <- lift $ saveFetchedBundle remote (Just bPath)
            case outcome of
                FetchError err -> lift $ tellErr $ "Error: " ++ err
                _ -> pure ()  -- No need to render fetch output during pull

            entries <- lift $ Verify.loadMetadataFromBundle fetchedBundle
            let remoteMeta = Verify.binaryEntries entries
            lift $ tell "Scanning remote files... done."
            result <- lift $ Remote.Scan.fetchRemoteFiles remote
            case result of
                Left _ -> lift $ tellErr "Error: Could not fetch remote file list."
                Right remoteFiles -> do
                    let filteredRemoteFiles = filterOutBitPaths remoteFiles
                    localMeta <- lift $ Verify.loadBinaryMetadata (cwd </> bitIndexPath) (Parallel 0)

                    let remoteFileMap = Map.fromList
                          [ (normalise (unPath e.path), (h, e.kind))
                          | e <- filteredRemoteFiles
                          , h <- maybeToList (syncHash e.kind)
                          ]
                        remoteMetaMap = Map.fromList [(normalise (unPath m.bfmPath), (m.bfmHash, m.bfmSize)) | m <- remoteMeta]
                        localMetaMap = Map.fromList [(normalise (unPath m.bfmPath), (m.bfmHash, m.bfmSize)) | m <- localMeta]

                    lift $ tell "Comparing..."
                    let divergentFiles = findDivergentFiles remoteFileMap remoteMetaMap

                    if null divergentFiles
                        then do
                            lift $ tell "No remote divergence detected. Proceeding with normal pull..."
                            let transport = mkCloudTransport remote
                            pullWithCleanup transport remote defaultPullOptions
                        else do
                            _oldHash <- lift getLocalHeadE
                            (_remoteCode, remoteOut, _) <- lift $ gitQuery ["rev-parse", Git.remoteTrackingRef name]
                            let _newHash = takeWhile (/= '\n') remoteOut

                            (mergeCode, mergeOut, mergeErr) <- lift $ gitQuery ["merge", "--no-commit", "--no-ff", Git.remoteTrackingRef name]
                            (_finalMergeCode, _, _) <- lift $ if mergeCode /= ExitSuccess && "refusing to merge unrelated histories" `List.isInfixOf` (mergeOut ++ mergeErr)
                                then do tell "Merging unrelated histories (e.g. first pull)..."; gitQuery ["merge", "--no-commit", "--no-ff", "--allow-unrelated-histories", Git.remoteTrackingRef name]
                                else pure (mergeCode, mergeOut, mergeErr)

                            createConflictDirectories remote divergentFiles localMetaMap

                            lift $ printConflictList divergentFiles localMetaMap
                            lift $ do
                                tell ""
                                tell "To resolve:"
                                tell "  1. Examine files in .bit/conflicts/<path>/"
                                tell "  2. Copy your chosen version to <path>"
                                tell "  3. Run 'bit add <path>'"
                                tell "  4. Run 'bit merge --continue'"
                                tell ""
                                tell "Or abort: 'bit merge --abort'"

-- | Filesystem pull logic (simplified - no bundle fetching, just merge + sync)
pullWithCleanup :: FileTransport -> Remote -> PullOptions -> BitM ()
pullWithCleanup transport remote opts = do
    env <- asks id
    result <- liftIO $ try @SomeException (runBitM env (pullLogic transport remote opts))
    either (\ex -> do
            inProgress <- lift $ Git.isMergeInProgress
            if inProgress
                then lift $ do
                    void $ gitRaw ["merge", "--abort"]
                    tell "Merge aborted. Your working tree is unchanged."
                else lift $ throwIO ex)
        (const $ pure ()) result

pullLogic :: FileTransport -> Remote -> PullOptions -> BitM ()
pullLogic transport remote _opts = do
    cwd <- asks envCwd
    let name = remoteName remote
    maybeBundlePath <- lift $ fetchRemoteBundle remote
    case maybeBundlePath of
        Nothing -> pure ()
        Just bPath -> do
            outcome <- lift $ saveFetchedBundle remote (Just bPath)
            case outcome of
                FetchError err -> lift $ tellErr $ "Error: " ++ err
                _ -> pure ()  -- No need to render fetch output during pull
            (_, countOut, _) <- lift $ gitQuery ["rev-list", "--count", Git.remoteTrackingRef name]
            let n = takeWhile (`elem` ['0'..'9']) (filter (/= '\n') countOut)
            lift $ tell $ "remote: Counting objects: " ++ (if null n then "0" else n) ++ ", done."

            -- Proof of possession — always verify remote before pulling
            lift $ putStrLn "Verifying remote files..."
            result <- lift $ Verify.verifyRemote cwd remote Nothing (Parallel 0)
            if null result.vrIssues
                then lift $ putStrLn $ "Verified " ++ show result.vrCount ++ " remote files."
                else lift $ do
                    hPutStrLn stderr $ "error: Remote files do not match remote metadata (" ++ show (length result.vrIssues) ++ " issues)."
                    mapM_ (printVerifyIssue id) result.vrIssues
                    hPutStrLn stderr "hint: Run 'bit verify --remote' to see all mismatches."
                    hPutStrLn stderr "hint: Run 'bit pull --accept-remote' to accept the remote's actual state."
                    exitWith (ExitFailure 1)

            oldHash <- lift getLocalHeadE
            (_remoteCode, remoteOut, _) <- lift $ gitQuery ["rev-parse", Git.remoteTrackingRef name]
            let newHash = takeWhile (/= '\n') remoteOut

            case oldHash of
                Nothing -> do
                    lift $ tell $ "Checking out " ++ take 7 newHash ++ " (first pull)"
                    checkoutCode <- lift $ Git.checkoutRemoteAsMain name
                    case checkoutCode of
                        ExitSuccess -> lift $ do
                            transportSyncAllFiles transport cwd
                            tell "Syncing binaries... done."
                        _ -> lift $ tellErr "Error: Failed to checkout remote branch."

                Just localHead -> do
                    (mergeCode, mergeOut, mergeErr) <- lift $ gitQuery ["merge", "--no-commit", "--no-ff", Git.remoteTrackingRef name]

                    (finalMergeCode, finalMergeOut, finalMergeErr) <-
                        lift $ if mergeCode /= ExitSuccess && "refusing to merge unrelated histories" `List.isInfixOf` (mergeOut ++ mergeErr)
                        then do tell "Merging unrelated histories..."; gitQuery ["merge", "--no-commit", "--no-ff", "--allow-unrelated-histories", Git.remoteTrackingRef name]
                        else pure (mergeCode, mergeOut, mergeErr)

                    case finalMergeCode of
                      ExitSuccess -> do
                        lift $ do
                            tell $ "Updating " ++ take 7 localHead ++ ".." ++ take 7 newHash
                            tell "Merge made by the 'recursive' strategy."
                        hasChanges <- lift hasStagedChangesE
                        when hasChanges $ lift $ void $ gitRaw ["commit", "-m", "Merge remote"]
                        lift $ do
                            applyMergeToWorkingDir transport cwd localHead
                            tell "Syncing binaries... done."
                        maybeRemoteHash <- lift $ Git.getHashFromBundle fetchedBundle
                        lift $ traverse_ (void . Git.updateRemoteTrackingBranchToHash name) maybeRemoteHash
                      _ -> do
                        lift $ do
                            tell finalMergeOut
                            tellErr finalMergeErr
                            tell "Automatic merge failed."
                            tell "bit requires you to pick a version for each conflict."
                            tell ""
                            tell "Resolving conflicts..."

                        conflicts <- lift Conflict.getConflictedFilesE
                        resolutions <- lift $ Conflict.resolveAll conflicts
                        let total = length resolutions

                        invalid <- lift $ validateMetadataDir (cwd </> bitIndexPath)
                        unless (null invalid) $ lift $ do
                            void $ gitRaw ["merge", "--abort"]
                            tellErr "fatal: Metadata files contain conflict markers. Merge aborted."
                            throwIO (userError "Invalid metadata")

                        conflictsNow <- lift Conflict.getConflictedFilesE
                        when (null conflictsNow) $ lift $ do
                            void $ gitRaw ["commit", "-m", "Merge remote (resolved " ++ show total ++ " conflict(s))"]
                            tell $ "Merge complete. " ++ show total ++ " conflict(s) resolved."
                            applyMergeToWorkingDir transport cwd localHead
                            tell "Syncing binaries... done."
                            void $ Git.updateRemoteTrackingBranchToHash name newHash

-- ============================================================================
-- Helper types and functions
-- ============================================================================

-- | A file where remote actual content doesn't match remote metadata.
-- Prevents transposition bugs vs bare (FilePath, Hash, Hash, Integer, Integer) tuple.
data DivergentFile = DivergentFile
    { dfPath         :: FilePath
    , dfExpectedHash :: Hash 'MD5
    , dfActualHash   :: Hash 'MD5
    , dfExpectedSize :: Integer
    , dfActualSize   :: Integer
    }
    deriving (Show, Eq)

-- | Find files where remote actual files don't match remote metadata.
findDivergentFiles :: Map.Map FilePath (Hash 'MD5, EntryKind) -> Map.Map FilePath (Hash 'MD5, Integer) -> [DivergentFile]
findDivergentFiles remoteFileMap remoteMetaMap =
    Map.foldlWithKey (\acc filePath (expectedHash, expectedSize) ->
        let normalizedPath = normalise filePath
        in case Map.lookup normalizedPath remoteFileMap of
            Nothing -> acc
            Just (actualHash, entryKind) ->
                case entryKind of
                    File _ actualSize _ ->
                        if actualHash == expectedHash && actualSize == expectedSize
                            then acc
                            else DivergentFile filePath expectedHash actualHash expectedSize actualSize : acc
                    _ -> acc
        ) [] remoteMetaMap

-- | Create conflict directories for divergent files.
createConflictDirectories :: Remote -> [DivergentFile] -> Map.Map FilePath (Hash 'MD5, Integer) -> BitM ()
createConflictDirectories remote divergentFiles localMetaMap = do
    cwd <- asks envCwd
    let conflictsDir = cwd </> ".bit" </> "conflicts"
    lift $ createDirE conflictsDir

    forM_ divergentFiles $ \df -> do
        let conflictDir = conflictsDir </> df.dfPath
        lift $ createDirE (takeDirectory conflictDir)

        let localPath = cwd </> df.dfPath
        localExists <- lift $ fileExistsE localPath
        when localExists $ lift $ copyFileE localPath (conflictDir </> "LOCAL")

        code <- liftIO $ Transport.copyFromRemote remote (toPosix df.dfPath) (conflictDir </> "REMOTE")
        when (code /= ExitSuccess) $ lift $ tellErr $ "Warning: Could not download remote file: " ++ df.dfPath

        lift $ case Map.lookup (normalise df.dfPath) localMetaMap of
            Just (localHash, localSize) ->
                writeFileAtomicE (conflictDir </> "METADATA_LOCAL") $
                    serializeMetadata (MetaContent localHash localSize)
            Nothing -> writeFileAtomicE (conflictDir </> "METADATA_LOCAL") "hash: (not tracked)\nsize: 0\n"

        lift $ writeFileAtomicE (conflictDir </> "METADATA_REMOTE") $
            serializeMetadata (MetaContent df.dfActualHash df.dfActualSize)

-- | Print conflict list in spec format.
printConflictList :: [DivergentFile] -> Map.Map FilePath (Hash 'MD5, Integer) -> IO ()
printConflictList divergentFiles localMetaMap = do
    putStrLn ""
    putStrLn "✗ Remote divergence detected:"
    putStrLn ""

    forM_ divergentFiles $ \df -> do
        putStrLn $ "  " ++ toPosix df.dfPath ++ ":"

        let localInfo = case Map.lookup (normalise df.dfPath) localMetaMap of
                Just (localHash, localSize) -> (displayHash localHash, show localSize)
                Nothing -> ("(not tracked)", "0")

        putStrLn $ "    Local:           " ++ fst localInfo ++ " (" ++ snd localInfo ++ " bytes)"
        putStrLn $ "    Remote actual:   " ++ displayHash df.dfActualHash ++ " (" ++ show df.dfActualSize ++ " bytes)"
        putStrLn $ "    Remote metadata: " ++ displayHash df.dfExpectedHash ++ " (" ++ show df.dfExpectedSize ++ " bytes)"
        putStrLn $ ""
        putStrLn $ "    Files saved to: .bit/conflicts/" ++ toPosix df.dfPath ++ "/"
        putStrLn ""

    putStrLn "This can happen when:"
    putStrLn "  - Files were modified directly on the remote (not via bit)"
    putStrLn "  - A partial push from another client"
    putStrLn "  - Remote storage corruption"
```

---

## Bit/Core/Push.hs

**Path:** `Bit/Core/Push.hs`

*Source file.*

```haskell
{-# LANGUAGE ScopedTypeVariables #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE LambdaCase #-}
{-# LANGUAGE OverloadedRecordDot #-}

module Bit.Core.Push
    ( push
    , cloudPush
    , filesystemPush
    , pushBundle
    , uploadToRemote
    , cleanupTemp
    , pushToRemote
    , updateLocalBundleAfterPush
    , syncRemoteFiles
    , processExistingRemote
    ) where

import qualified System.Directory as Dir
import System.FilePath ((</>))
import Control.Monad (when, unless, void)
import Data.Foldable (traverse_)
import System.Exit (ExitCode(..), exitWith)
import qualified Internal.Git as Git
import qualified Internal.Transport as Transport
import Internal.Config (fetchedBundle, BundleName(..), bundleCwdPath, fromCwdPath)
import Bit.Utils (trimGitOutput)
import qualified Bit.Pipeline as Pipeline
import qualified Bit.Remote.Scan as Remote.Scan
import qualified Data.List as List
import Bit.Concurrency (runConcurrentlyBounded)
import Control.Concurrent (getNumCapabilities)
import System.IO (stderr, hPutStrLn)
import Control.Exception (bracket)
import Bit.Remote (Remote, remoteName, remoteUrl, RemoteState(..), FetchResult(..), displayRemote)
import Bit.Types (BitM, BitEnv(..), ForceMode(..))
import Control.Monad.Trans.Reader (asks)
import Control.Monad.IO.Class (liftIO)
import Control.Monad.Trans.Class (lift)
import qualified Bit.CopyProgress as CopyProgress
import qualified Bit.Verify as Verify
import Bit.Concurrency (Concurrency(..))
import qualified Bit.Device as Device
import System.Directory (copyFile)
import Bit.Core.Helpers
    ( AncestorQuery(..)
    , getRemoteType
    , withRemote
    , getLocalHeadE
    , checkIsAheadE
    , fileExistsE
    , tell
    , tellErr
    , printVerifyIssue
    , safeRemove
    )
import Bit.Core.Init (initializeRepoAt)
import Bit.Core.Transport (executeCommand, filesystemSyncAllFiles, filesystemSyncChangedFiles)
import Bit.Core.Fetch (classifyRemoteState, fetchBundle)

-- ============================================================================
-- Push operations
-- ============================================================================

push :: BitM ()
push = withRemote $ \remote -> do
    cwd <- asks envCwd

    -- Proof of possession — always verify local before pushing
    liftIO $ putStrLn "Verifying local files..."
    result <- liftIO $ Verify.verifyLocal cwd Nothing (Parallel 0)
    if null result.vrIssues
        then liftIO $ putStrLn $ "Verified " ++ show result.vrCount ++ " files. All match metadata."
        else liftIO $ do
            hPutStrLn stderr $ "error: Working tree does not match metadata (" ++ show (length result.vrIssues) ++ " issues)."
            mapM_ (printVerifyIssue id) result.vrIssues  -- full hash, no truncation
            hPutStrLn stderr "hint: Run 'bit verify' to see all mismatches."
            hPutStrLn stderr "hint: Run 'bit add' to update metadata, or 'bit restore' to restore files."
            exitWith (ExitFailure 1)

    -- Determine if this is a filesystem or cloud remote
    mType <- liftIO $ getRemoteType cwd (remoteName remote)
    case mType of
        Just t | Device.isFilesystemType t -> liftIO $ filesystemPush cwd remote
        _ -> cloudPush remote  -- Cloud remote or no target info (use cloud flow)

-- | Push to a cloud remote (original flow, unchanged).
cloudPush :: Remote -> BitM ()
cloudPush remote = do
    fMode <- asks envForceMode
    state <- liftIO $ do
        putStrLn $ "Inspecting remote: " ++ displayRemote remote
        classifyRemoteState remote

    case state of
        StateEmpty -> do
            liftIO $ putStrLn "Remote is empty. Initializing..."
            syncRemoteFiles
            liftIO $ pushBundle remote
            updateLocalBundleAfterPush

        StateValidRgit -> do
            fetchResult <- liftIO $ do
                putStrLn "Remote is a bit repo. Checking history..."
                fetchBundle remote
            case fetchResult of
                BundleFound bPath -> do
                    let fetchedPath = fromCwdPath (bundleCwdPath fetchedBundle)
                    liftIO $ do
                        copyFile bPath fetchedPath
                        safeRemove bPath
                    processExistingRemote
                _ -> liftIO $ hPutStrLn stderr "Error: Remote .bit found but metadata is missing."

        StateNonRgitOccupied samples ->
            case fMode of
                Force -> do
                    liftIO $ hPutStrLn stderr "Warning: --force used. Overwriting non-bit remote..."
                    syncRemoteFiles
                    liftIO $ pushBundle remote
                    updateLocalBundleAfterPush
                _ -> liftIO $ do
                    hPutStrLn stderr "-------------------------------------------------------"
                    hPutStrLn stderr "[!] STOP: Remote is NOT a bit repository!"
                    hPutStrLn stderr $ "Found existing files: " ++ List.intercalate ", " samples
                    hPutStrLn stderr "To initialize anyway (destructive): bit init --force"
                    hPutStrLn stderr "-------------------------------------------------------"

        StateNetworkError err ->
            liftIO $ hPutStrLn stderr $ "Aborting: Network error -> " ++ err

        StateCorruptedRgit msg ->
            liftIO $ hPutStrLn stderr $ "Aborting: [X] Corrupted remote -> " ++ msg

-- | Push to a filesystem remote. Creates a full bit repo at the remote location.
filesystemPush :: FilePath -> Remote -> IO ()
filesystemPush cwd remote = do
    let remotePath = remoteUrl remote
    putStrLn $ "Pushing to filesystem remote: " ++ remotePath
    
    -- 1. Check if remote has .bit/ directory (first push vs subsequent)
    let remoteBitDir = remotePath </> ".bit"
    remoteHasBit <- Dir.doesDirectoryExist remoteBitDir
    
    unless remoteHasBit $ do
        putStrLn "First push: initializing bit repo at remote..."
        initializeRepoAt remotePath
    
    -- 2. Fetch local into remote (at the remote, "origin" is the local side)
    let localIndexGit = cwd </> ".bit" </> "index" </> ".git"
    let remoteIndex = remotePath </> ".bit" </> "index"

    putStrLn "Fetching local commits into remote..."
    (fetchCode, _fetchOut, fetchErr) <- Git.runGitAt remoteIndex
        ["fetch", localIndexGit, "main:" ++ Git.remoteTrackingRef "origin"]

    when (fetchCode /= ExitSuccess) $ do
        hPutStrLn stderr $ "Error fetching into remote: " ++ fetchErr
        exitWith fetchCode

    -- 3. Capture remote HEAD before merge
    (oldHeadCode, oldHeadOut, _) <- Git.runGitAt remoteIndex ["rev-parse", "HEAD"]
    let mOldHead = case oldHeadCode of
            ExitSuccess -> Just (trimGitOutput oldHeadOut)
            _ -> Nothing

    -- 4. Check if remote HEAD is ancestor of what we're pushing (fast-forward check)
    traverse_ (const $ do
        (checkCode, _, _) <- Git.runGitAt remoteIndex
            ["merge-base", "--is-ancestor", "HEAD", Git.remoteTrackingRef "origin"]
        when (checkCode /= ExitSuccess) $ do
            hPutStrLn stderr "error: Remote has local commits that you don't have."
            hPutStrLn stderr "hint: Run 'bit pull' to merge remote changes first, then push again."
            exitWith (ExitFailure 1)
        ) mOldHead

    -- 5. Merge at remote (ff-only)
    putStrLn "Merging at remote (fast-forward only)..."
    (mergeCode, _mergeOut, mergeErr) <- Git.runGitAt remoteIndex
        ["merge", "--ff-only", Git.remoteTrackingRef "origin"]
    
    case mergeCode of
        ExitSuccess -> do
            -- 6. Get new HEAD at remote
            (newHeadCode, newHeadOut, _) <- Git.runGitAt remoteIndex ["rev-parse", "HEAD"]
            when (newHeadCode /= ExitSuccess) $ do
                hPutStrLn stderr "Error: Could not get remote HEAD after merge"
                exitWith (ExitFailure 1)
            
            let newHead = trimGitOutput newHeadOut
            
            -- 7. Sync actual files based on what changed
            case mOldHead of
                Nothing -> do
                    -- First push: sync all files from new HEAD
                    putStrLn "First push: syncing all files to remote..."
                    filesystemSyncAllFiles cwd remotePath newHead
                Just oldHead -> do
                    -- Subsequent push: sync only changed files
                    putStrLn "Syncing changed files to remote..."
                    filesystemSyncChangedFiles cwd remotePath oldHead newHead
            
            -- 8. Update local tracking ref
            putStrLn "Updating local tracking ref..."
            void $ Git.updateRemoteTrackingBranchToHead (remoteName remote)

            putStrLn "Push complete."
        _ -> do
            hPutStrLn stderr $ "error: Failed to merge at remote: " ++ mergeErr
            exitWith (ExitFailure 1)

-- ============================================================================
-- Push helper functions
-- ============================================================================

-- | Push the git bundle to remote. Uses bracket to ensure temp bundle cleanup.
pushBundle :: Remote -> IO ()
pushBundle remote = do
    let tempBundle = BundleName "bit"
        tempBundleCwdPath = fromCwdPath (bundleCwdPath tempBundle)

    -- bracket <setup> <cleanup> <action>
    bracket
        (Git.createBundle tempBundle)              -- 1. Acquire
        (const $ cleanupTemp tempBundleCwdPath)    -- 2. Release (Always runs)
        (\case                                     -- 3. Work
            ExitSuccess -> uploadToRemote tempBundleCwdPath remote
            _ -> hPutStrLn stderr "Error creating bundle"
        )

-- Helper for the upload logic to keep the bracket clean
uploadToRemote :: FilePath -> Remote -> IO ()
uploadToRemote src remote = do
    putStrLn "Uploading bundle to remote..."
    rCode <- Transport.copyToRemote src remote ".bit/bit.bundle"
    case rCode of
        ExitSuccess -> putStrLn "Metadata push complete."
        _ -> hPutStrLn stderr "Error uploading bundle."

-- Helper for cleanup that doesn't crash if the file was never made
cleanupTemp :: FilePath -> IO ()
cleanupTemp filePath = do
    exists <- Dir.doesFileExist filePath
    when exists (Dir.removeFile filePath)

-- | Sync files, push bundle, and update local tracking. Used after remote checks pass.
pushToRemote :: Remote -> BitM ()
pushToRemote remote = do
  syncRemoteFiles
  liftIO $ pushBundle remote
  updateLocalBundleAfterPush

-- | After a successful push, update the local fetched_remote.bundle to current HEAD
-- so bit status shows up to date instead of "ahead of remote".
updateLocalBundleAfterPush :: BitM ()
updateLocalBundleAfterPush = do
    mRemote <- asks envRemote
    code <- liftIO $ Git.createBundle fetchedBundle
    case code of
        ExitSuccess -> case mRemote of
            Just remote -> void $ liftIO $ Git.updateRemoteTrackingBranchToHead (remoteName remote)
            Nothing     -> pure ()
        _ -> pure ()

syncRemoteFiles :: BitM ()
syncRemoteFiles = withRemote $ \remote -> do
    cwd <- asks envCwd
    localFiles <- asks envLocalFiles
    remoteResult <- liftIO $ Remote.Scan.fetchRemoteFiles remote
    either
        (const $ liftIO $ hPutStrLn stderr "Error: Failed to fetch remote file list.")
        (\remoteFiles -> do
            let actions = Pipeline.pushSyncFiles localFiles remoteFiles
            liftIO $ putStrLn "--- Pushing Changes to Remote ---"
            if null actions
                then liftIO $ putStrLn "Remote is already up to date."
                else do
                    -- Create progress tracker for cloud operations (file-count only)
                    progress <- liftIO $ CopyProgress.newSyncProgress (length actions)
                    liftIO $ CopyProgress.withSyncProgressReporter progress $ do
                        -- Use lower concurrency for network/subprocess operations
                        caps <- getNumCapabilities
                        let concurrency = min 8 (max 2 (caps * 2))
                        void $ runConcurrentlyBounded concurrency (\a -> do
                            executeCommand cwd remote a
                            CopyProgress.incrementFilesComplete progress
                            ) actions)
        remoteResult

processExistingRemote :: BitM ()
processExistingRemote = do
    fMode <- asks envForceMode
    mRemote <- asks envRemote
    case fMode of
      Force -> do
            lift $ tellErr "Warning: --force used. Overwriting remote history..."
            maybe (lift $ tellErr "Error: No remote configured.") pushToRemote mRemote
      ForceWithLease -> do
                    maybeRemoteHash <- liftIO $ Git.getHashFromBundle fetchedBundle
                    let fetchedPath = fromCwdPath (bundleCwdPath fetchedBundle)
                    hasFetchedBundle <- lift $ fileExistsE fetchedPath

                    case (maybeRemoteHash, hasFetchedBundle) of
                        (Just rHash, True) -> do
                            maybeFetchedHash <- liftIO $ Git.getHashFromBundle fetchedBundle
                            case maybeFetchedHash of
                                Just fileHash | rHash == fileHash -> do
                                    lift $ tell "Remote check passed (--force-with-lease). Proceeding with push..."
                                    maybe (lift $ tellErr "Error: No remote configured.") pushToRemote mRemote
                                Just _fHash -> lift $ do
                                    tellErr "---------------------------------------------------"
                                    tellErr "ERROR: Remote has changed since last fetch!"
                                    tellErr "Someone else pushed to the remote."
                                    tellErr "Run 'bit fetch' to update your local view of the remote."
                                    tellErr "---------------------------------------------------"
                                Nothing -> lift $ tellErr "Error: Could not extract hash from fetched bundle."
                        (Just _, False) -> do
                            lift $ tellErr "Warning: No local fetched bundle found. Proceeding with push (--force-with-lease)..."
                            maybe (lift $ tellErr "Error: No remote configured.") pushToRemote mRemote
                        (Nothing, _) -> lift $ tellErr "Error: Could not extract hash from remote bundle."
      NoForce -> do
                    maybeRemoteHash <- liftIO $ Git.getHashFromBundle fetchedBundle
                    maybeLocalHash <- lift getLocalHeadE

                    case (maybeLocalHash, maybeRemoteHash) of
                        (Just lHash, Just rHash) -> do
                            isAhead <- lift $ checkIsAheadE (AncestorQuery { aqAncestor = rHash, aqDescendant = lHash })

                            if isAhead
                                then do
                                    lift $ tell "Remote check passed. Proceeding with push..."
                                    maybe (lift $ tellErr "Error: No remote configured.") pushToRemote mRemote
                                else lift $ do
                                    tellErr "---------------------------------------------------"
                                    tellErr "ERROR: Remote history has diverged or is ahead!"
                                    tellErr "Please run 'bit pull' before pushing."
                                    tellErr "---------------------------------------------------"

                        _ -> lift $ tellErr "Error: Could not extract hashes for comparison."
```

---

## Bit/Core/RemoteManagement.hs

**Path:** `Bit/Core/RemoteManagement.hs`

*Source file.*

```haskell
{-# LANGUAGE ScopedTypeVariables #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TypeApplications #-}
{-# LANGUAGE LambdaCase #-}
{-# LANGUAGE MultiWayIf #-}
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE OverloadedRecordDot #-}

module Bit.Core.RemoteManagement
    ( remoteAdd
    , addRemote
    , addRemoteFilesystem
    , promptDeviceName
    , remoteShow
    , remoteRepair
    , formatRemoteDisplay
    , showRemoteStatusFromBundle
    ) where

import qualified System.Directory as Dir
import System.FilePath ((</>), takeDirectory)
import Control.Monad (unless, void, when, forM_)
import System.Exit (ExitCode(..), exitWith)
import Internal.Git (AncestorQuery(..))
import qualified Internal.Git as Git
import qualified Internal.Transport as Transport
import qualified Bit.Device as Device
import qualified Bit.DevicePrompt as DevicePrompt
import Data.UUID (UUID)
import System.IO (stderr, hPutStrLn)
import Control.Exception (try, IOException)
import Data.Maybe (fromMaybe)
import qualified Data.Map.Strict as Map
import qualified Data.Set as Set
import qualified Data.Text as T
import Internal.Config (bitDevicesDir, bitRemotesDir, fetchedBundle, bundleCwdPath, fromCwdPath, BundleName(..), bitIndexPath, bundleGitRelPath, fromGitRelPath)
import Bit.Types (BitM, BitEnv(..), Path(..), Hash(..), HashAlgo(..), hashToText)
import Control.Monad.Trans.Reader (asks)
import Control.Monad.IO.Class (liftIO)
import Bit.Remote (Remote, remoteUrl, remoteName, displayRemote, resolveRemote)
import Bit.Core.Helpers (getRemoteType)
import qualified Bit.Core.Fetch as Fetch
import qualified Bit.Verify as Verify
import Bit.Concurrency (Concurrency(..))
import Bit.Utils (toPosix, atomicWriteFileStr)
import Bit.Internal.Metadata (MetaContent(..), serializeMetadata)

-- ============================================================================
-- Remote Management
-- ============================================================================

remoteAdd :: String -> String -> IO ()
remoteAdd = addRemote

addRemote :: String -> String -> IO ()
addRemote name pathOrUrl = do
    cwd <- Dir.getCurrentDirectory
    Dir.createDirectoryIfMissing True bitDevicesDir
    Dir.createDirectoryIfMissing True bitRemotesDir
    pathType <- Device.classifyRemotePath pathOrUrl
    case pathType of
        Device.CloudRemote url -> do
            Device.writeRemoteFile cwd name Device.RemoteCloud (Just url)
            -- Register bundle path as named git remote (bundle may not exist yet)
            let bundleGitPath = fromGitRelPath (bundleGitRelPath fetchedBundle)
            void $ Git.addRemote name bundleGitPath
            putStrLn $ "Remote '" ++ name ++ "' added (" ++ url ++ ")."
        Device.FilesystemPath filePath -> addRemoteFilesystem cwd name filePath

promptDeviceName :: FilePath -> FilePath -> Maybe String -> IO String
promptDeviceName cwd _volRoot mLabel =
    DevicePrompt.acquireDeviceNameAuto mLabel $ \name -> (name `elem`) <$> Device.listDeviceNames cwd

addRemoteFilesystem :: FilePath -> String -> FilePath -> IO ()
addRemoteFilesystem cwd name filePath = do
    absPath <- Dir.makeAbsolute filePath
    exists <- Dir.doesDirectoryExist absPath
    unless exists $ do
        hPutStrLn stderr ("fatal: Path does not exist or is not accessible: " ++ filePath)
        case filePath of
            ('\\':_) -> uncHint
            ('/':'/':_) -> uncHint
            _ -> pure ()
        exitWith (ExitFailure 1)
    volRoot <- Device.getVolumeRoot absPath
    isFixed <- Device.isFixedDrive volRoot
    if isFixed
        then addRemoteFixed cwd name absPath
        else addRemoteDevice cwd name absPath volRoot
  where
    uncHint = hPutStrLn stderr "hint: For UNC paths under Git Bash / MINGW, use forward slashes: //server/share/path"

-- | Add a fixed-drive filesystem remote. Path stored only in git config.
addRemoteFixed :: FilePath -> String -> FilePath -> IO ()
addRemoteFixed cwd name absPath = do
    void $ Git.addRemote name (absPath </> ".bit" </> "index")
    Device.writeRemoteFile cwd name Device.RemoteFilesystem Nothing
    putStrLn $ "Remote '" ++ name ++ "' added (" ++ absPath ++ ")."

-- | Add a removable/network device remote. Uses device UUID tracking.
addRemoteDevice :: FilePath -> String -> FilePath -> FilePath -> IO ()
addRemoteDevice cwd name absPath volRoot = do
    let relPath = Device.getRelativePath volRoot absPath
    mStoreUuid <- Device.readBitStore volRoot
    mExistingDevice <- maybe (pure Nothing) (Device.findDeviceByUuid cwd) mStoreUuid
    result <- try @IOException $ case (mStoreUuid, mExistingDevice) of
        (Just _u, Just dev) -> do
            putStrLn $ "Using existing device '" ++ dev ++ "'."
            _mInfo <- Device.readDeviceFile cwd dev
            Device.writeRemoteFile cwd name Device.RemoteDevice (Just (dev ++ ":" ++ relPath))
            void $ Git.addRemote name (absPath </> ".bit" </> "index")
            putStrLn $ "Remote '" ++ name ++ "' → " ++ dev ++ ":" ++ relPath
            putStrLn $ "(using existing device '" ++ dev ++ "')"
            pure ()
        (Just u, Nothing) -> do
            mLabel <- Device.getVolumeLabel volRoot
            deviceName' <- promptDeviceName cwd volRoot mLabel
            registerDevice cwd name volRoot deviceName' relPath u
            void $ Git.addRemote name (absPath </> ".bit" </> "index")
        (Nothing, _) -> do
            mLabel <- Device.getVolumeLabel volRoot
            deviceName' <- promptDeviceName cwd volRoot mLabel
            u <- Device.generateStoreUuid
            Device.writeBitStore volRoot u
            registerDevice cwd name volRoot deviceName' relPath u
            void $ Git.addRemote name (absPath </> ".bit" </> "index")
    case result of
        Right () -> pure ()
        Left _err -> do
            -- Cannot create .bit-store at volume root (e.g. permission denied on C:\)
            -- Fall back to path-based storage for local directories
            Device.writeRemoteFile cwd name Device.RemoteFilesystem Nothing
            void $ Git.addRemote name (absPath </> ".bit" </> "index")
            putStrLn $ "Remote '" ++ name ++ "' added (" ++ absPath ++ ")."

-- | Detect storage type, get serial, write device + remote files, and print confirmation.
-- Shared between the "existing UUID / no device" and "no UUID" branches.
registerDevice :: FilePath -> String -> FilePath -> String -> FilePath -> UUID -> IO ()
registerDevice cwd name volRoot deviceName' relPath u = do
    storeType' <- Device.detectStorageType volRoot
    mSerial <- case storeType' of
        Device.Physical -> Device.getHardwareSerial volRoot
        Device.Network -> pure Nothing
    Device.writeDeviceFile cwd deviceName' (Device.DeviceInfo u storeType' mSerial)
    Device.writeRemoteFile cwd name Device.RemoteDevice (Just (deviceName' ++ ":" ++ relPath))
    putStrLn $ "Remote '" ++ name ++ "' → " ++ deviceName' ++ ":" ++ relPath
    putStrLn $ "Device '" ++ deviceName' ++ "' registered (" ++ displayStorageType storeType' ++ ")."

displayStorageType :: Device.StorageType -> String
displayStorageType Device.Physical = "physical"
displayStorageType Device.Network  = "network"

-- ============================================================================
-- Remote show / repair
-- ============================================================================

remoteShow :: Maybe String -> BitM ()
remoteShow mRemoteName = do
    cwd <- asks envCwd
    case mRemoteName of
        Nothing -> do
            let remotesDir = cwd </> bitRemotesDir
            dirExists <- liftIO $ Dir.doesDirectoryExist remotesDir
            if not dirExists
                then liftIO $ putStrLn "No remotes configured. Use 'bit remote add <name> <url>' to add one."
                else do
                    remoteNames <- liftIO $ Dir.listDirectory remotesDir
                    if null remoteNames
                        then liftIO $ putStrLn "No remotes configured. Use 'bit remote add <name> <url>' to add one."
                        else liftIO $ forM_ remoteNames $ \rName -> do
                            mRemote' <- resolveRemote cwd rName
                            mType' <- Device.readRemoteType cwd rName
                            display <- case mRemote' of
                                Just r  -> formatRemoteDisplayByType cwd rName mType' r
                                Nothing -> do
                                    mTarget' <- Device.readRemoteFile cwd rName
                                    formatRemoteDisplay cwd rName mTarget'
                            putStrLn display
        Just name -> do
            (mRemote, mType) <- liftIO $ (,) <$> resolveRemote cwd name <*> Device.readRemoteType cwd name
            case mRemote of
                Nothing -> liftIO $ putStrLn "No remotes configured. Use 'bit remote add <name> <url>' to add one."
                Just remote -> do
                    -- Display the remote line
                    display <- liftIO $ formatRemoteDisplayByType cwd name mType remote
                    liftIO $ do
                        putStrLn display
                        putStrLn ""
                    -- Show status: ref-based for filesystem/device, bundle-based for cloud
                    let isFs = maybe False Device.isFilesystemType mType
                    if isFs
                        then liftIO $ showRefBasedRemoteStatus name (remoteUrl remote)
                        else do
                            let fetchedPath = fromCwdPath (bundleCwdPath fetchedBundle)
                            hasBundle <- liftIO $ Dir.doesFileExist fetchedPath
                            if hasBundle
                                then liftIO $ showRemoteStatusFromBundle name (Just (remoteUrl remote))
                                else do
                                    maybeBundlePath <- liftIO $ Fetch.fetchRemoteBundle remote
                                    case maybeBundlePath of
                                        Just bPath -> do
                                            outcome <- liftIO $ Fetch.saveFetchedBundle remote (Just bPath)
                                            case outcome of
                                                Fetch.FetchError err -> liftIO $ hPutStrLn stderr $ "Warning: " ++ err
                                                _ -> pure ()
                                            liftIO $ showRemoteStatusFromBundle name (Just (remoteUrl remote))
                                        Nothing -> liftIO $ do
                                            putStrLn $ "  Fetch URL: " ++ remoteUrl remote
                                            putStrLn $ "  Push  URL: " ++ remoteUrl remote
                                            putStrLn ""
                                            putStrLn "  HEAD branch: (unknown)"
                                            putStrLn ""
                                            putStrLn "  Local branch configured for 'bit pull':"
                                            putStrLn "    main merges with remote (unknown)"
                                            putStrLn ""
                                            putStrLn "  Local refs configured for 'bit push':"
                                            putStrLn "    main pushes to main (unknown)"

-- ============================================================================
-- Remote repair
-- ============================================================================

-- | Repair action: copy a file from one side to repair the other.
-- Carries the expected hash+size so metadata can be restored.
data RepairAction
    = RepairLocal  Path Path (Hash 'MD5) Integer  -- ^ Copy from remote sourcePath to fix local destPath, with expected hash+size
    | RepairRemote Path Path (Hash 'MD5) Integer  -- ^ Copy from local sourcePath to fix remote destPath, with expected hash+size
    deriving (Show)

-- | Result of executing a single repair action.
data RepairResult = Repaired Path | RepairFailed Path String
    deriving (Show)

remoteRepair :: Maybe String -> Concurrency -> BitM ()
remoteRepair mName concurrency = do
    cwd <- asks envCwd
    -- Resolve remote
    (mRemote, _resolvedName) <- liftIO $ do
        name <- maybe Git.getTrackedRemoteName pure mName
        mRemote <- resolveRemote cwd name
        pure (mRemote, name)
    case mRemote of
        Nothing -> liftIO $ do
            maybe
                (hPutStrLn stderr "fatal: No remote configured.")
                (\n -> hPutStrLn stderr $ "fatal: '" ++ n ++ "' does not appear to be a git remote.")
                mName
            hPutStrLn stderr "hint: Set remote with 'bit remote add <name> <url>'"
            exitWith (ExitFailure 1)
        Just remote -> liftIO $ do
            putStrLn $ "Repairing against remote: " ++ displayRemote remote
            putStrLn ""

            -- Determine if filesystem or cloud remote
            mType <- getRemoteType cwd (remoteName remote)
            let isFilesystem = maybe False Device.isFilesystemType mType
                remotePath = remoteUrl remote

            if isFilesystem
                then repairFilesystem cwd remote remotePath concurrency
                else repairCloud cwd remote concurrency

-- | Repair against a filesystem remote (direct file access, no rclone).
repairFilesystem :: FilePath -> Remote -> FilePath -> Concurrency -> IO ()
repairFilesystem cwd _remote remotePath concurrency = do
    -- Load committed metadata from both sides (immune to scan updates)
    let localIndexDir = cwd </> bitIndexPath
        remoteIndexDir = remotePath </> bitIndexPath
    localMeta <- Verify.loadCommittedBinaryMetadata localIndexDir
    remoteMeta <- Verify.loadCommittedBinaryMetadata remoteIndexDir

    -- Verify both sides
    putStrLn "Verifying local files..."
    localResult <- Verify.verifyLocal cwd Nothing concurrency
    putStrLn $ "  " ++ show localResult.vrCount ++ " files checked, " ++ show (length localResult.vrIssues) ++ " issues"

    putStrLn "Verifying remote files..."
    remoteResult <- Verify.verifyLocalAt remotePath Nothing concurrency
    putStrLn $ "  " ++ show remoteResult.vrCount ++ " files checked, " ++ show (length remoteResult.vrIssues) ++ " issues"

    runRepairLogic localMeta remoteMeta localResult.vrIssues remoteResult.vrIssues
        (executeFilesystemRepair cwd remotePath)

-- | Repair against a cloud remote (bundle-based, uses rclone).
repairCloud :: FilePath -> Remote -> Concurrency -> IO ()
repairCloud cwd remote concurrency = do
    -- Fetch bundle
    maybeBundlePath <- Fetch.fetchRemoteBundle remote
    case maybeBundlePath of
        Nothing -> do
            hPutStrLn stderr "fatal: Could not fetch remote bundle."
            exitWith (ExitFailure 1)
        Just bPath -> do
            outcome <- Fetch.saveFetchedBundle remote (Just bPath)
            case outcome of
                Fetch.FetchError err -> do
                    hPutStrLn stderr $ "fatal: " ++ err
                    exitWith (ExitFailure 1)
                _ -> pure ()

    -- Load committed metadata (immune to scan updates)
    let localIndexDir = cwd </> bitIndexPath
    localMeta <- Verify.loadCommittedBinaryMetadata localIndexDir
    entries <- Verify.loadMetadataFromBundle fetchedBundle
    let remoteMeta = Verify.binaryEntries entries

    -- Verify both sides
    putStrLn "Verifying local files..."
    localResult <- Verify.verifyLocal cwd Nothing concurrency
    putStrLn $ "  " ++ show localResult.vrCount ++ " files checked, " ++ show (length localResult.vrIssues) ++ " issues"

    putStrLn "Verifying remote files..."
    remoteResult <- Verify.verifyRemote cwd remote Nothing concurrency
    putStrLn $ "  " ++ show remoteResult.vrCount ++ " files checked, " ++ show (length remoteResult.vrIssues) ++ " issues"

    runRepairLogic localMeta remoteMeta localResult.vrIssues remoteResult.vrIssues
        (executeRepair cwd remote)

-- | Common repair logic shared between filesystem and cloud remotes.
runRepairLogic :: [Verify.BinaryFileMeta]    -- local metadata
              -> [Verify.BinaryFileMeta]    -- remote metadata
              -> [Verify.VerifyIssue]             -- local issues
              -> [Verify.VerifyIssue]             -- remote issues
              -> (RepairAction -> IO RepairResult) -- repair executor
              -> IO ()
runRepairLogic localMeta remoteMeta localIssues remoteIssues executeAction =
    if null localIssues && null remoteIssues
        then putStrLn "\nAll files verified. Nothing to repair."
        else do
            putStrLn ""

            -- Build sets of broken paths
            let localIssueSet = Set.fromList (map issuePath localIssues)
                remoteIssueSet = Set.fromList (map issuePath remoteIssues)

            -- Build content indexes from VERIFIED files
            let remoteVerified = buildContentIndex remoteMeta remoteIssueSet
                localVerified  = buildContentIndex localMeta localIssueSet

            -- Build metadata maps for lookup
            let localMetaMap = Map.fromList [(m.bfmPath, (m.bfmHash, m.bfmSize)) | m <- localMeta]
                remoteMetaMap = Map.fromList [(m.bfmPath, (m.bfmHash, m.bfmSize)) | m <- remoteMeta]

            -- Plan repairs: use the OTHER side's metadata as source of truth
            -- (local metadata may reflect corrupted state after a scan)
            let (localRepairs, localUnrepairable) =
                    planRepairs localIssues remoteMetaMap remoteVerified RepairLocal
                (remoteRepairs, remoteUnrepairable) =
                    planRepairs remoteIssues localMetaMap localVerified RepairRemote
                allRepairs = localRepairs ++ remoteRepairs
                allUnrepairable = localUnrepairable ++ remoteUnrepairable

            when (null allRepairs && null allUnrepairable) $
                putStrLn "No repairable issues found (issues may be in text files or untracked files)."

            -- Execute repairs
            unless (null allRepairs) $ do
                putStrLn $ "Repairing " ++ show (length allRepairs) ++ " file(s)..."
                results <- mapM executeAction allRepairs

                let repaired = [p | Repaired p <- results]
                    failed   = [(p, e) | RepairFailed p e <- results]

                forM_ repaired $ \p ->
                    putStrLn $ "  [REPAIRED] " ++ toPosix (unPath p)
                forM_ failed $ \(p, e) ->
                    hPutStrLn stderr $ "  [FAILED]   " ++ toPosix (unPath p) ++ " (" ++ e ++ ")"

                -- Summary
                putStrLn ""
                putStrLn $ show (length repaired) ++ " repaired, "
                    ++ show (length failed) ++ " failed, "
                    ++ show (length allUnrepairable) ++ " unrepairable."

                unless (null failed && null allUnrepairable) $
                    exitWith (ExitFailure 1)

            unless (null allUnrepairable) $ do
                when (null allRepairs) $ do
                    forM_ allUnrepairable $ \p ->
                        hPutStrLn stderr $ "  [UNREPAIRABLE] " ++ toPosix (unPath p)
                    putStrLn ""
                    putStrLn $ "0 repaired, 0 failed, " ++ show (length allUnrepairable) ++ " unrepairable."
                    exitWith (ExitFailure 1)

-- | Extract the path from a VerifyIssue.
issuePath :: Verify.VerifyIssue -> Path
issuePath (Verify.HashMismatch p _ _ _ _) = p
issuePath (Verify.Missing p) = p

-- | Build a content-addressable index from verified metadata entries.
-- Maps (hashString, size) to a Path that is known to be good.
-- Excludes any paths that are in the issue set (those are broken).
buildContentIndex :: [Verify.BinaryFileMeta] -> Set.Set Path -> Map.Map (String, Integer) Path
buildContentIndex entries issueSet =
    Map.fromList
        [ ((T.unpack (hashToText m.bfmHash), m.bfmSize), m.bfmPath)
        | m <- entries
        , not (Set.member m.bfmPath issueSet)
        ]

-- | Plan repair actions for a list of issues.
-- For each issue, looks up the expected (hash, size) in the opposite side's content index.
-- Returns (repair actions, unrepairable paths).
planRepairs :: [Verify.VerifyIssue]
            -> Map.Map Path (Hash 'MD5, Integer)
            -> Map.Map (String, Integer) Path
            -> (Path -> Path -> Hash 'MD5 -> Integer -> RepairAction)
            -> ([RepairAction], [Path])
planRepairs issues metaMap contentIndex mkAction = foldr go ([], []) issues
  where
    go issue (repairs, unrepairables) =
        let p = issuePath issue
        in case Map.lookup p metaMap of
            Nothing -> (repairs, unrepairables)  -- not in binary metadata, skip
            Just (expectedHash, expectedSize) ->
                let key = (T.unpack (hashToText expectedHash), expectedSize)
                in case Map.lookup key contentIndex of
                    Just sourcePath -> (mkAction sourcePath p expectedHash expectedSize : repairs, unrepairables)
                    Nothing -> (repairs, p : unrepairables)

-- | Restore the metadata file in .bit/index/ to match the expected hash+size.
restoreLocalMetadata :: FilePath -> Path -> Hash 'MD5 -> Integer -> IO ()
restoreLocalMetadata cwd destPath expectedHash expectedSize = do
    let metaPath = cwd </> bitIndexPath </> unPath destPath
    Dir.createDirectoryIfMissing True (takeDirectory metaPath)
    atomicWriteFileStr metaPath (serializeMetadata (MetaContent expectedHash expectedSize))

-- | Execute a single repair action for cloud remotes (uses rclone).
executeRepair :: FilePath -> Remote -> RepairAction -> IO RepairResult
executeRepair cwd remote (RepairLocal sourcePath destPath expectedHash expectedSize) = do
    -- Copy from remote to fix local
    let remoteRelPath = toPosix (unPath sourcePath)
        localFullPath = cwd </> unPath destPath
    Dir.createDirectoryIfMissing True (takeDirectory localFullPath)
    code <- Transport.copyFromRemote remote remoteRelPath localFullPath
    case code of
        ExitSuccess -> do
            restoreLocalMetadata cwd destPath expectedHash expectedSize
            pure (Repaired destPath)
        _ -> pure (RepairFailed destPath "copy from remote failed")
executeRepair cwd remote (RepairRemote sourcePath destPath _ _) = do
    -- Copy from local to fix remote
    let localFullPath = cwd </> unPath sourcePath
        remoteRelPath = toPosix (unPath destPath)
    code <- Transport.copyToRemote localFullPath remote remoteRelPath
    pure $ case code of
        ExitSuccess -> Repaired destPath
        _ -> RepairFailed destPath "copy to remote failed"

-- | Execute a single repair action for filesystem remotes (direct file copy).
executeFilesystemRepair :: FilePath -> FilePath -> RepairAction -> IO RepairResult
executeFilesystemRepair cwd remotePath (RepairLocal sourcePath destPath expectedHash expectedSize) = do
    -- Copy from remote filesystem to fix local
    let srcFullPath = remotePath </> unPath sourcePath
        dstFullPath = cwd </> unPath destPath
    Dir.createDirectoryIfMissing True (takeDirectory dstFullPath)
    result <- try @IOException $ Dir.copyFile srcFullPath dstFullPath
    case result of
        Right () -> do
            restoreLocalMetadata cwd destPath expectedHash expectedSize
            pure (Repaired destPath)
        Left e -> pure (RepairFailed destPath (show e))
executeFilesystemRepair cwd remotePath (RepairRemote sourcePath destPath expectedHash expectedSize) = do
    -- Copy from local to fix remote filesystem, also restore remote metadata
    let srcFullPath = cwd </> unPath sourcePath
        dstFullPath = remotePath </> unPath destPath
        remoteMetaPath = remotePath </> bitIndexPath </> unPath destPath
    Dir.createDirectoryIfMissing True (takeDirectory dstFullPath)
    Dir.createDirectoryIfMissing True (takeDirectory remoteMetaPath)
    result <- try @IOException $ do
        Dir.copyFile srcFullPath dstFullPath
        atomicWriteFileStr remoteMetaPath (serializeMetadata (MetaContent expectedHash expectedSize))
    pure $ case result of
        Right () -> Repaired destPath
        Left e -> RepairFailed destPath (show e)

-- | Format remote display line (e.g. "origin → black_usb:Backup (physical, connected at E:\)")
formatRemoteDisplay :: FilePath -> String -> Maybe Device.RemoteTarget -> IO String
formatRemoteDisplay cwd name = maybe (pure (name ++ " → (no target)")) $ \case
    Device.TargetLocalPath p -> pure (name ++ " → " ++ p ++ " (local path)")
    Device.TargetDevice dev devPath -> do
        res <- Device.resolveRemoteTarget cwd (Device.TargetDevice dev devPath)
        mInfo <- Device.readDeviceFile cwd dev
        let typ = maybe "unknown" (displayStorageType . Device.deviceType) mInfo
        case res of
            Device.Resolved mount -> pure (name ++ " → " ++ dev ++ ":" ++ devPath ++ " (" ++ typ ++ ", connected at " ++ mount ++ ")")
            Device.NotConnected _ -> pure (name ++ " → " ++ dev ++ ":" ++ devPath ++ " (" ++ typ ++ ", NOT CONNECTED)")
    Device.TargetCloud u -> pure (name ++ " → " ++ u ++ " (cloud)")

-- | Format remote display line using RemoteType instead of RemoteTarget.
formatRemoteDisplayByType :: FilePath -> String -> Maybe Device.RemoteType -> Remote -> IO String
formatRemoteDisplayByType cwd name mType remote = case mType of
    Just Device.RemoteFilesystem -> pure (name ++ " → " ++ remoteUrl remote ++ " (filesystem)")
    Just Device.RemoteDevice -> do
        -- Read the target to get device info
        mTarget <- Device.readRemoteFile cwd name
        case mTarget of
            Just (Device.TargetDevice dev devPath) -> do
                mInfo <- Device.readDeviceFile cwd dev
                let typ = maybe "unknown" (displayStorageType . Device.deviceType) mInfo
                pure (name ++ " → " ++ dev ++ ":" ++ devPath ++ " (" ++ typ ++ ", connected at " ++ remoteUrl remote ++ ")")
            _ -> pure (name ++ " → " ++ displayRemote remote ++ " (device)")
    Just Device.RemoteCloud -> pure (name ++ " → " ++ remoteUrl remote ++ " (cloud)")
    Nothing -> pure (name ++ " → " ++ displayRemote remote)

-- | Show remote status using git tracking refs (for filesystem/device remotes).
-- No bundle needed — reads directly from refs/remotes/<name>/main.
showRefBasedRemoteStatus :: String -> String -> IO ()
showRefBasedRemoteStatus name url = do
    maybeLocal <- Git.getLocalHead
    (refCode, refOut, _) <- Git.runGitWithOutput ["rev-parse", Git.remoteTrackingRef name]
    let maybeRemote = case refCode of
            ExitSuccess -> Just (filter (/= '\n') refOut)
            _ -> Nothing
    putStrLn $ "* remote " ++ name
    putStrLn $ "  Fetch URL: " ++ url
    putStrLn $ "  Push  URL: " ++ url
    putStrLn ""
    case (maybeLocal, maybeRemote) of
        (Nothing, Just _) -> do
            putStrLn "  HEAD branch: (unknown)"
            putStrLn ""
            putStrLn "  Local branch configured for 'bit pull':"
            putStrLn "    main merges with remote (unknown)"
            putStrLn ""
            putStrLn "  Local refs configured for 'bit push':"
            putStrLn "    main pushes to main (local out of date)"
        (Just lHash, Just rHash) -> do
            putStrLn "  HEAD branch: main"
            putStrLn ""
            if lHash == rHash
                then do
                    putStrLn "  Local branch configured for 'bit pull':"
                    putStrLn "    main merges with remote main"
                    putStrLn ""
                    putStrLn "  Local refs configured for 'bit push':"
                    putStrLn "    main pushes to main (up to date)"
                else do
                    status <- classifyPushStatus lHash rHash
                    putStrLn "  Local branch configured for 'bit pull':"
                    putStrLn "    main merges with remote main"
                    putStrLn ""
                    putStrLn "  Local refs configured for 'bit push':"
                    case status of
                        PushRefFastForwardable -> putStrLn "    main pushes to main (fast-forwardable)"
                        PushRefLocalOutOfDate  -> putStrLn "    main pushes to main (local out of date)"
                        PushRefDiverged       -> putStrLn "    main pushes to main (diverged)"
                        PushRefUpToDate       -> putStrLn "    main pushes to main (up to date)"
        _ -> do
            putStrLn "  HEAD branch: (unknown)"
            putStrLn ""
            putStrLn "  Local branch configured for 'bit pull':"
            putStrLn "    main merges with remote (unknown)"
            putStrLn ""
            putStrLn "  Local refs configured for 'bit push':"
            putStrLn "    main pushes to main (unknown)"

showRemoteStatusFromBundle :: String -> Maybe String -> IO ()
showRemoteStatusFromBundle name mUrl = do
    maybeLocal <- Git.getLocalHead
    let url = fromMaybe "?" mUrl
    putStrLn $ "* remote " ++ name
    putStrLn $ "  Fetch URL: " ++ url
    putStrLn $ "  Push  URL: " ++ url
    putStrLn ""
    compareHistory maybeLocal fetchedBundle

-- | Status of local ref relative to remote (for 'bit remote show' push message).
data PushRefStatus
  = PushRefUpToDate        -- ^ Same commit
  | PushRefFastForwardable -- ^ Local ahead of remote
  | PushRefLocalOutOfDate  -- ^ Remote ahead of local
  | PushRefDiverged        -- ^ Both have commits the other doesn't; merge/rebase needed
  deriving (Show, Eq)

-- | Classify push status from local and remote commit hashes. Calls git internally;
-- callers never see raw boolean ancestry flags.
classifyPushStatus :: String -> String -> IO PushRefStatus
classifyPushStatus localHash remoteHash = do
  localAhead  <- Git.checkIsAhead (AncestorQuery { aqAncestor = remoteHash, aqDescendant = localHash })
  remoteAhead <- Git.checkIsAhead (AncestorQuery { aqAncestor = localHash, aqDescendant = remoteHash })
  pure $ case (localAhead, remoteAhead) of
    (True, False) -> PushRefFastForwardable
    (False, True) -> PushRefLocalOutOfDate
    (False, False) -> PushRefDiverged
    (True, True)   -> PushRefUpToDate

compareHistory :: Maybe String -> BundleName -> IO ()
compareHistory maybeLocal bundleName = do
    maybeRemote <- Git.getHashFromBundle bundleName
    case (maybeLocal, maybeRemote) of
        (Nothing, Just _) -> do
            putStrLn "  HEAD branch: (unknown)"
            putStrLn ""
            putStrLn "  Local branch configured for 'bit pull':"
            putStrLn "    main merges with remote (unknown)"
            putStrLn ""
            putStrLn "  Local refs configured for 'bit push':"
            putStrLn "    main pushes to main (local out of date)"

        (Just lHash, Just rHash) -> do
            putStrLn "  HEAD branch: main"
            putStrLn ""
            if lHash == rHash
                then do
                    putStrLn "  Local branch configured for 'bit pull':"
                    putStrLn "    main merges with remote main"
                    putStrLn ""
                    putStrLn "  Local refs configured for 'bit push':"
                    putStrLn "    main pushes to main (up to date)"
                else do
                    status <- classifyPushStatus lHash rHash

                    putStrLn "  Local branch configured for 'bit pull':"
                    putStrLn "    main merges with remote main"
                    putStrLn ""
                    putStrLn "  Local refs configured for 'bit push':"
                    case status of
                        PushRefFastForwardable -> putStrLn "    main pushes to main (fast-forwardable)"
                        PushRefLocalOutOfDate  -> putStrLn "    main pushes to main (local out of date)"
                        PushRefDiverged       -> putStrLn "    main pushes to main (diverged)"
                        PushRefUpToDate       -> putStrLn "    main pushes to main (up to date)"
        _ -> pure ()
```

---

## Bit/Core/Transport.hs

**Path:** `Bit/Core/Transport.hs`

*Source file.*

```haskell
{-# LANGUAGE ScopedTypeVariables #-}
{-# LANGUAGE OverloadedStrings #-}

module Bit.Core.Transport
    ( -- File transport abstraction
      FileTransport(..)
    , mkCloudTransport
    , mkFilesystemTransport
      -- Working directory sync operations
    , applyMergeToWorkingDir
    , downloadOrCopyFromIndex
    , filesystemDownloadOrCopyFromIndex
    , filesystemSyncRemoteFilesToLocalFromHEAD
    , filesystemCopyFileToRemote
    , filesystemDeleteFileAtRemote
    , filesystemSyncAllFiles
    , filesystemSyncChangedFiles
    , safeDeleteWorkFile
    , copyFromIndexToWorkTree
    , isTextFileInIndex
    , isTextMetadataFile
    , syncBinariesAfterMerge
    , executeCommand
    , executePullCommand
    ) where

import qualified System.Directory as Dir
import System.Directory (copyFile, createDirectoryIfMissing)
import System.FilePath ((</>), takeDirectory)
import Control.Monad (when, void, forM)
import Data.Foldable (traverse_)
import System.Exit (ExitCode(..))
import qualified Internal.Git as Git
import Internal.Git (NameStatusChange(Added, Deleted, Modified, Renamed, Copied))
import qualified Internal.Transport as Transport
import Internal.Config (bitIndexPath, fetchedBundle)
import qualified Bit.Scan as Scan
import qualified Bit.Pipeline as Pipeline
import qualified Bit.Remote.Scan as Remote.Scan
import Data.List (isPrefixOf)
import Bit.Concurrency (runConcurrentlyBounded)
import Control.Concurrent (getNumCapabilities)
import System.IO (stderr, hPutStrLn)
import Bit.Utils (toPosix)
import Bit.Plan (RcloneAction(..))
import Bit.Remote (Remote, remoteName)
import Bit.Types (BitM, BitEnv(..), unPath)
import Control.Monad.Trans.Reader (asks)
import Control.Monad.IO.Class (liftIO)
import qualified Bit.CopyProgress as CopyProgress
import Bit.CopyProgress (SyncProgress)
import Data.IORef (writeIORef, atomicModifyIORef')
import qualified Bit.Internal.Metadata as Metadata
-- Strict IO imports to avoid Windows file locking issues
import qualified Data.ByteString as BS
import qualified Data.Text as T
import Data.Text.Encoding (decodeUtf8')
import Bit.Core.Helpers
    ( getLocalHeadE
    , readFileMaybe
    )

-- ============================================================================
-- Internal types
-- ============================================================================

-- | File classified for sync. Text has no size; binary carries byte count for progress.
data FileToSync
    = TextToSync   FilePath
    | BinaryToSync FilePath Integer  -- ^ byte count for progress tracking
    deriving (Show, Eq)

-- | Extract binary files with size > 0 for progress tracking (excludes zero-size).
binaryFiles :: [FileToSync] -> [(FilePath, Integer)]
binaryFiles fs = [(p, s) | BinaryToSync p s <- fs, s > 0]

-- ============================================================================
-- FILE TRANSPORT ABSTRACTION
-- ============================================================================

-- | Abstracts how files are transferred during pull.
-- Cloud remotes use rclone; filesystem remotes use direct file copy.
data FileTransport = FileTransport
  { -- | Copy/download a single file to the working directory.
    -- Args: cwd, relative path, progress tracker
    transportDownloadFile :: FilePath -> FilePath -> SyncProgress -> IO ()
    -- | Sync ALL files from remote to local (used for first pull when there's no oldHead to diff against).
    -- Called after git checkout has updated the index, so files are listed from current HEAD.
    -- Args: cwd
  , transportSyncAllFiles :: FilePath -> IO ()
  }

-- | Build a cloud transport that uses rclone to copy files.
mkCloudTransport :: Remote -> FileTransport
mkCloudTransport remote = FileTransport
  { transportDownloadFile = \cwd filePath progress -> downloadOrCopyFromIndex cwd remote filePath progress
  , transportSyncAllFiles = \cwd -> do
      -- Cloud path uses the existing syncRemoteFilesToLocal logic
      -- which scans remote via rclone and syncs to local.
      localFiles <- Scan.scanWorkingDir cwd
      remoteResult <- Remote.Scan.fetchRemoteFiles remote
      either (const $ hPutStrLn stderr "Error: Failed to fetch remote file list.") (\remoteFiles -> do
          let actions = Pipeline.pullSyncFiles localFiles remoteFiles
          putStrLn "--- Pulling changes from remote ---"
          if null actions
            then putStrLn "Working tree already up to date with remote."
            else do
              -- Gather file sizes from index metadata for byte-level progress
              sizes <- forM actions $ \a -> case a of
                  Copy _ dest -> getFileSizeFromIndex cwd (unPath dest)
                  Move src _  -> getFileSizeFromIndex cwd (unPath src)
                  Swap _ src dest -> (+) <$> getFileSizeFromIndex cwd (unPath src)
                                        <*> getFileSizeFromIndex cwd (unPath dest)
                  Delete _    -> pure 0
              let totalBytes = sum sizes

              -- Create progress tracker with byte totals
              progress <- CopyProgress.newSyncProgress (length actions)
              writeIORef (CopyProgress.spBytesTotal progress) totalBytes
              CopyProgress.withSyncProgressReporter progress $ do
                -- Use lower concurrency for network/subprocess operations
                caps <- getNumCapabilities
                let concurrency = min 8 (max 2 (caps * 2))
                void $ runConcurrentlyBounded concurrency
                  (executePullCommand cwd remote progress) actions
          ) remoteResult
  }

-- | Build a filesystem transport that uses direct file copy.
mkFilesystemTransport :: FilePath -> FileTransport
mkFilesystemTransport remotePath = FileTransport
  { transportDownloadFile = filesystemDownloadOrCopyFromIndex' remotePath
  , transportSyncAllFiles = filesystemSyncRemoteFilesToLocal' remotePath
  }
  where
    -- Wrapper that matches the signature expected by FileTransport
    filesystemDownloadOrCopyFromIndex' remPath cwd filePath progress =
        filesystemDownloadOrCopyFromIndex cwd remPath filePath progress
    filesystemSyncRemoteFilesToLocal' _remPath cwd =
      filesystemSyncRemoteFilesToLocalFromHEAD cwd remotePath

-- ============================================================================
-- Working tree synchronization
-- ============================================================================

-- | After a merge, mirror git's metadata changes onto the actual working directory.
-- Uses `git diff --name-status oldHead newHead` to determine what changed,
-- then downloads/deletes/moves actual files accordingly.
-- This replaces syncRemoteFilesToLocal for merge pulls.
--
-- CRITICAL: Always reads the actual HEAD after merge from git (via getLocalHeadE).
-- Never accepts newHead as a parameter - this prevents the bug where remoteHash
-- was passed instead of the merged HEAD, causing local-only files to appear deleted.
applyMergeToWorkingDir :: FileTransport -> FilePath -> String -> IO ()
applyMergeToWorkingDir transport cwd oldHead = do
    newHead <- getLocalHeadE
    traverse_ (\newH -> do
        changes <- Git.getDiffNameStatus oldHead newH
        putStrLn "--- Pulling changes from remote ---"
        if null changes
            then putStrLn "Working tree already up to date with remote."
            else do
                -- First pass: collect paths that will be copied and their sizes
                filesToCopy <- fmap concat $ forM changes $ \change -> case change of
                    Added p -> pure [p]
                    Modified p -> pure [p]
                    Renamed _ newPath -> pure [newPath]
                    Copied _ newPath -> pure [newPath]
                    Deleted _ -> pure []
                
                -- Gather file sizes for binary files (for progress tracking)
                fileInfo <- forM filesToCopy $ \filePath -> do
                    fromIndex <- isTextFileInIndex cwd filePath
                    if fromIndex
                        then pure (TextToSync filePath)
                        else BinaryToSync filePath <$> getFileSizeFromIndex cwd filePath

                let bins = binaryFiles fileInfo
                    totalFiles = length bins
                    totalBytes = sum [s | (_, s) <- bins]

                -- Create progress tracker
                progress <- CopyProgress.newSyncProgress totalFiles
                writeIORef (CopyProgress.spBytesTotal progress) totalBytes
                
                -- Second pass: apply changes with progress (parallelized)
                CopyProgress.withSyncProgressReporter progress $ do
                    -- Use lower concurrency for file operations to avoid thrashing
                    caps <- getNumCapabilities
                    let concurrency = max 2 (caps * 2)
                    void $ runConcurrentlyBounded concurrency (\change -> case change of
                        Added p -> (transportDownloadFile transport) cwd p progress
                        Modified p -> (transportDownloadFile transport) cwd p progress
                        Deleted p -> safeDeleteWorkFile cwd p
                        Renamed oldPath newPath -> do
                            safeDeleteWorkFile cwd oldPath
                            (transportDownloadFile transport) cwd newPath progress
                        Copied _ newPath -> (transportDownloadFile transport) cwd newPath progress
                        ) changes
        ) newHead

-- | Download a file from remote, or copy from index if it's a text file.
-- Used by cloud transport. Updates progress counters after download.
downloadOrCopyFromIndex :: FilePath -> Remote -> FilePath -> SyncProgress -> IO ()
downloadOrCopyFromIndex cwd remote filePath progress = do
    fromIndex <- isTextFileInIndex cwd filePath
    if fromIndex
        then copyFromIndexToWorkTree cwd filePath
        else do
            let localPath = cwd </> filePath
            createDirectoryIfMissing True (takeDirectory localPath)
            void $ Transport.copyFromRemote remote (toPosix filePath) (toPosix localPath)
            exists <- Dir.doesFileExist localPath
            when exists $ do
                size <- Dir.getFileSize localPath
                atomicModifyIORef' (CopyProgress.spBytesCopied progress) (\n -> (n + fromIntegral size, ()))
            CopyProgress.incrementFilesComplete progress

-- | Download a file from remote or copy from index for filesystem pull.
-- Parameter order: localRoot, remotePath, filePath (relative), progress.
filesystemDownloadOrCopyFromIndex :: FilePath -> FilePath -> FilePath -> SyncProgress -> IO ()
filesystemDownloadOrCopyFromIndex localRoot remotePath filePath progress = do
    fromIndex <- isTextFileInIndex localRoot filePath
    if fromIndex
        then copyFromIndexToWorkTree localRoot filePath
        else do
            -- Binary file: copy from remote working tree
            let srcPath = remotePath </> filePath
            let destPath = localRoot </> filePath
            srcExists <- Dir.doesFileExist srcPath
            when srcExists $ do
                size <- Dir.getFileSize srcPath
                writeIORef (CopyProgress.spCurrentFile progress) filePath
                CopyProgress.copyFileWithProgress srcPath destPath (fromIntegral size) progress
                CopyProgress.incrementFilesComplete progress

-- | Sync all files from remote to local using current HEAD (after checkout).
-- This is called after git checkout has updated the index, so we list files from HEAD.
filesystemSyncRemoteFilesToLocalFromHEAD :: FilePath -> FilePath -> IO ()
filesystemSyncRemoteFilesToLocalFromHEAD localRoot remotePath = do
    let localIndex = localRoot </> ".bit" </> "index"
    -- Use HEAD to list files (after checkout, HEAD points to the remote branch)
    (code, out, _) <- Git.runGitWithOutput ["ls-tree", "-r", "--name-only", "HEAD"]
    when (code == ExitSuccess) $ do
        let paths = filter (not . null) (lines out)
        
        -- First pass: classify files and gather sizes for binary files
        fileInfo <- forM paths $ \filePath -> do
            let metaPath = localIndex </> filePath
            isText <- isTextMetadataFile metaPath
            if isText
                then pure (TextToSync filePath)
                else do
                    let srcPath = remotePath </> filePath
                    srcExists <- Dir.doesFileExist srcPath
                    if srcExists
                        then BinaryToSync filePath . fromIntegral <$> Dir.getFileSize srcPath
                        else pure (BinaryToSync filePath 0)

        let bins = binaryFiles fileInfo
            totalBytes = sum [s | (_, s) <- bins]

        -- Create progress tracker
        progress <- CopyProgress.newSyncProgress (length bins)
        writeIORef (CopyProgress.spBytesTotal progress) totalBytes
        
        -- Second pass: copy files with progress (parallelized)
        CopyProgress.withSyncProgressReporter progress $ do
            -- Use lower concurrency for file copies to avoid disk thrashing
            caps <- getNumCapabilities
            let concurrency = max 2 (caps * 2)
            void $ runConcurrentlyBounded concurrency (\ft -> case ft of
                TextToSync filePath -> do
                    let metaPath = localIndex </> filePath
                        workPath = localRoot </> filePath
                    createDirectoryIfMissing True (takeDirectory workPath)
                    copyFile metaPath workPath
                BinaryToSync filePath size -> do
                    let srcPath = remotePath </> filePath
                        destPath = localRoot </> filePath
                    srcExists <- Dir.doesFileExist srcPath
                    when srcExists $ do
                        writeIORef (CopyProgress.spCurrentFile progress) filePath
                        CopyProgress.copyFileWithProgress srcPath destPath size progress
                        CopyProgress.incrementFilesComplete progress
                ) fileInfo

-- | Copy a file from local to remote (handles both text and binary).
-- Parameter order: localRoot, remotePath, remoteIndex, filePath (relative).
-- TRANSPOSITION NOTE: 4 FilePaths — rely on naming conventions.
filesystemCopyFileToRemote :: FilePath -> FilePath -> FilePath -> FilePath -> SyncProgress -> IO ()
filesystemCopyFileToRemote localRoot remotePath remoteIndex filePath progress = do
    -- Check if it's a text file (content in index) or binary (hash/size in index)
    let metaPath = remoteIndex </> filePath
    isText <- isTextMetadataFile metaPath
    if isText
        then do
            -- Text file: metadata IS the content, copy from remote index to working tree
            let workPath = remotePath </> filePath
            createDirectoryIfMissing True (takeDirectory workPath)
            copyFile metaPath workPath
        else do
            -- Binary file: metadata is hash/size, copy actual file from local working tree
            let srcPath = localRoot </> filePath
            let destPath = remotePath </> filePath
            srcExists <- Dir.doesFileExist srcPath
            when srcExists $ do
                size <- Dir.getFileSize srcPath
                writeIORef (CopyProgress.spCurrentFile progress) filePath
                CopyProgress.copyFileWithProgress srcPath destPath (fromIntegral size) progress
                CopyProgress.incrementFilesComplete progress

-- | Delete a file at the remote working tree.
filesystemDeleteFileAtRemote :: FilePath -> FilePath -> IO ()
filesystemDeleteFileAtRemote remotePath filePath = do
    let fullPath = remotePath </> filePath
    exists <- Dir.doesFileExist fullPath
    when exists $ Dir.removeFile fullPath

-- | Sync all files from a commit to the filesystem remote (first push).
filesystemSyncAllFiles :: FilePath -> FilePath -> String -> IO ()
filesystemSyncAllFiles localRoot remotePath commitHash = do
    let remoteIndex = remotePath </> ".bit" </> "index"
    files <- Git.runGitAt remoteIndex ["ls-tree", "-r", "--name-only", commitHash]
    case files of
        (ExitSuccess, out, _) -> do
            let paths = filter (not . null) (lines out)
            
            -- First pass: classify files and gather sizes for binary files
            fileInfo <- forM paths $ \filePath -> do
                let metaPath = remoteIndex </> filePath
                isText <- isTextMetadataFile metaPath
                if isText
                    then pure (TextToSync filePath)
                    else do
                        let srcPath = localRoot </> filePath
                        srcExists <- Dir.doesFileExist srcPath
                        if srcExists
                            then BinaryToSync filePath . fromIntegral <$> Dir.getFileSize srcPath
                            else pure (BinaryToSync filePath 0)

            let bins = binaryFiles fileInfo
                totalBytes = sum [s | (_, s) <- bins]

            -- Create progress tracker
            progress <- CopyProgress.newSyncProgress (length bins)
            writeIORef (CopyProgress.spBytesTotal progress) totalBytes

            -- Second pass: copy files with progress (parallelized)
            CopyProgress.withSyncProgressReporter progress $ do
                caps <- getNumCapabilities
                let concurrency = max 2 (caps * 2)
                void $ runConcurrentlyBounded concurrency (\ft -> case ft of
                    TextToSync filePath -> do
                        let metaPath = remoteIndex </> filePath
                            workPath = remotePath </> filePath
                        createDirectoryIfMissing True (takeDirectory workPath)
                        copyFile metaPath workPath
                    BinaryToSync filePath size -> do
                        let srcPath = localRoot </> filePath
                            destPath = remotePath </> filePath
                        srcExists <- Dir.doesFileExist srcPath
                        when srcExists $ do
                            writeIORef (CopyProgress.spCurrentFile progress) filePath
                            CopyProgress.copyFileWithProgress srcPath destPath size progress
                            CopyProgress.incrementFilesComplete progress
                    ) fileInfo
        _ -> pure ()

-- | Sync only changed files between two commits.
-- Parameter order: localRoot, remotePath, oldHead (commit hash), newHead (commit hash).
filesystemSyncChangedFiles :: FilePath -> FilePath -> String -> String -> IO ()
filesystemSyncChangedFiles localRoot remotePath oldHead newHead = do
    let remoteIndex = remotePath </> ".bit" </> "index"
    changes <- Git.runGitAt remoteIndex ["diff", "--name-status", oldHead, newHead]
    case changes of
        (ExitSuccess, out, _) -> do
            let parsedChanges = Git.parseNameStatusOutput out
            
            -- First pass: collect paths that will be copied and their sizes
            filesToCopy <- fmap concat $ forM parsedChanges $ \change -> case change of
                Added p -> pure [p]
                Modified p -> pure [p]
                Renamed _ newPath -> pure [newPath]
                Copied _ newPath -> pure [newPath]
                Deleted _ -> pure []
            
            -- Gather file sizes for binary files
            fileInfo <- forM filesToCopy $ \p -> do
                let metaPath = remoteIndex </> p
                isText <- isTextMetadataFile metaPath
                if isText
                    then pure (TextToSync p)
                    else do
                        let srcPath = localRoot </> p
                        srcExists <- Dir.doesFileExist srcPath
                        if srcExists
                            then BinaryToSync p . fromIntegral <$> Dir.getFileSize srcPath
                            else pure (BinaryToSync p 0)

            let bins = binaryFiles fileInfo
                totalBytes = sum [s | (_, s) <- bins]

            -- Create progress tracker
            progress <- CopyProgress.newSyncProgress (length bins)
            writeIORef (CopyProgress.spBytesTotal progress) totalBytes

            -- Second pass: apply changes with progress (parallelized)
            CopyProgress.withSyncProgressReporter progress $ do
                caps <- getNumCapabilities
                let concurrency = max 2 (caps * 2)
                void $ runConcurrentlyBounded concurrency (\change -> case change of
                    Added p -> filesystemCopyFileToRemote localRoot remotePath remoteIndex p progress
                    Modified p -> filesystemCopyFileToRemote localRoot remotePath remoteIndex p progress
                    Deleted p -> filesystemDeleteFileAtRemote remotePath p
                    Renamed oldPath newPath -> do
                        filesystemDeleteFileAtRemote remotePath oldPath
                        filesystemCopyFileToRemote localRoot remotePath remoteIndex newPath progress
                    Copied _ newPath -> filesystemCopyFileToRemote localRoot remotePath remoteIndex newPath progress
                    ) parsedChanges
        _ -> pure ()

-- | Safely delete a file from the working directory.
safeDeleteWorkFile :: FilePath -> FilePath -> IO ()
safeDeleteWorkFile cwd filePath = do
    let fullPath = cwd </> filePath
    exists <- Dir.doesFileExist fullPath
    when exists $ Dir.removeFile fullPath

-- ============================================================================
-- Helper functions
-- ============================================================================

-- | True if the path is a text file in the index (content stored in metadata, not hash/size).
-- Used during pull to avoid re-downloading from rclone when content is already in the bundle.
isTextFileInIndex :: FilePath -> FilePath -> IO Bool
isTextFileInIndex localRoot filePath = do
    let metaPath = localRoot </> bitIndexPath </> filePath
    exists <- Dir.doesFileExist metaPath
    if not exists then pure False
    else do
        mcontent <- readFileMaybe metaPath
        pure $ maybe False (\content -> not (any ("hash: " `isPrefixOf`) (lines content))) mcontent

-- | Copy a file from the index to the working tree. Call only when the path
-- is a text file (content in index). Creates parent dirs as needed.
copyFromIndexToWorkTree :: FilePath -> FilePath -> IO ()
copyFromIndexToWorkTree localRoot filePath = do
    let metaPath = localRoot </> bitIndexPath </> filePath
        workPath = localRoot </> filePath
    createDirectoryIfMissing True (takeDirectory workPath)
    copyFile metaPath workPath

-- | Check if a metadata file is a text file (content stored directly) or binary (hash/size stored).
-- Text files don't have "hash:" lines, binary files do.
isTextMetadataFile :: FilePath -> IO Bool
isTextMetadataFile metaPath = do
    exists <- Dir.doesFileExist metaPath
    if not exists then pure False
    else do
        -- Use strict ByteString reading to avoid Windows file locking issues
        bs <- BS.readFile metaPath
        let content = either (const "") T.unpack (decodeUtf8' bs)
        pure $ not (any ("hash: " `isPrefixOf`) (lines content))

-- | Read a binary file's size from its index metadata.
-- Returns 0 for text files or if metadata is missing/unparseable.
getFileSizeFromIndex :: FilePath -> FilePath -> IO Integer
getFileSizeFromIndex localRoot filePath = do
    let metaPath = localRoot </> bitIndexPath </> filePath
    result <- Metadata.parseMetadataFile metaPath
    pure $ maybe 0 Metadata.metaSize result

-- | Sync binaries after a successful merge commit
syncBinariesAfterMerge :: FileTransport -> Remote -> Maybe String -> BitM ()
syncBinariesAfterMerge transport remote oldHead = do
    cwd <- asks envCwd
    let name = remoteName remote
    liftIO $ putStrLn "Syncing binaries... done."
    -- Apply diff-based sync or full sync depending on whether we have an old HEAD
    liftIO $ maybe (transportSyncAllFiles transport cwd) (applyMergeToWorkingDir transport cwd) oldHead
    maybeRemoteHash <- liftIO $ Git.getHashFromBundle fetchedBundle
    liftIO $ traverse_ (void . Git.updateRemoteTrackingBranchToHash name) maybeRemoteHash

-- | Executes/Prints the command to be run in the shell (push: local -> remote).
executeCommand :: FilePath -> Remote -> RcloneAction -> IO ()
executeCommand localRoot remote action = case action of
        Copy src dest -> do
            let localPath = toPosix (localRoot </> unPath src)
            void $ Transport.copyToRemote localPath remote (toPosix (unPath dest))

        Move src dest ->
            void $ Transport.moveRemote remote (toPosix (unPath src)) (toPosix (unPath dest))

        Delete p ->
            void $ Transport.deleteRemote remote (toPosix (unPath p))

        Swap tmp src dest -> do
            void $ Transport.moveRemote remote (toPosix (unPath src)) (toPosix (unPath tmp))
            void $ Transport.moveRemote remote (toPosix (unPath dest)) (toPosix (unPath src))
            void $ Transport.moveRemote remote (toPosix (unPath tmp)) (toPosix (unPath dest))

-- | Execute a single pull action: copy from remote to local or delete local file.
-- Text files are already in the git bundle (index); copy from index to work dir instead of rclone.
-- Updates progress counters (bytes + files) after each copy.
executePullCommand :: FilePath -> Remote -> SyncProgress -> RcloneAction -> IO ()
executePullCommand localRoot remote progress action = case action of
        Copy _src dest -> do
            fromIndex <- isTextFileInIndex localRoot (unPath dest)
            if fromIndex
            then copyFromIndexToWorkTree localRoot (unPath dest)
            else do
                let localPath = toPosix (localRoot </> unPath dest)
                createDirectoryIfMissing True (takeDirectory (localRoot </> unPath dest))
                void $ Transport.copyFromRemote remote (toPosix (unPath dest)) localPath
                exists <- Dir.doesFileExist (localRoot </> unPath dest)
                when exists $ do
                    size <- Dir.getFileSize (localRoot </> unPath dest)
                    atomicModifyIORef' (CopyProgress.spBytesCopied progress) (\n -> (n + fromIntegral size, ()))
            CopyProgress.incrementFilesComplete progress
        Move src dest -> do
            fromIndex <- isTextFileInIndex localRoot (unPath src)
            if fromIndex
            then copyFromIndexToWorkTree localRoot (unPath src)
            else do
                let localSrcPath = localRoot </> unPath src
                createDirectoryIfMissing True (takeDirectory localSrcPath)
                void $ Transport.copyFromRemote remote (toPosix (unPath src)) (toPosix localSrcPath)
                exists <- Dir.doesFileExist localSrcPath
                when exists $ do
                    size <- Dir.getFileSize localSrcPath
                    atomicModifyIORef' (CopyProgress.spBytesCopied progress) (\n -> (n + fromIntegral size, ()))
            let localDestPath = localRoot </> unPath dest
            destExists <- Dir.doesFileExist localDestPath
            when destExists $ Dir.removeFile localDestPath
            CopyProgress.incrementFilesComplete progress
        Delete filePath -> do
            let localPath = localRoot </> unPath filePath
            exists <- Dir.doesFileExist localPath
            when exists $ Dir.removeFile localPath
        Swap tmp src dest -> do
            -- For pull, swap files on the local filesystem.
            -- Text files: copy fresh from index (index has correct post-merge content).
            -- Binary files: three-step rename via temp file.
            srcIsText <- isTextFileInIndex localRoot (unPath src)
            destIsText <- isTextFileInIndex localRoot (unPath dest)
            case (srcIsText, destIsText) of
                (True, True) -> do
                    copyFromIndexToWorkTree localRoot (unPath src)
                    copyFromIndexToWorkTree localRoot (unPath dest)
                (False, False) -> do
                    let tmpPath = localRoot </> unPath tmp
                        srcPath = localRoot </> unPath src
                        destPath = localRoot </> unPath dest
                    Dir.renameFile srcPath tmpPath
                    Dir.renameFile destPath srcPath
                    Dir.renameFile tmpPath destPath
                (True, False) -> do
                    -- src is text (from index), dest is binary (swap via temp)
                    let tmpPath = localRoot </> unPath tmp
                        destPath = localRoot </> unPath dest
                    Dir.renameFile destPath tmpPath
                    copyFromIndexToWorkTree localRoot (unPath src)
                    Dir.renameFile tmpPath destPath
                (False, True) -> do
                    let tmpPath = localRoot </> unPath tmp
                        srcPath = localRoot </> unPath src
                    Dir.renameFile srcPath tmpPath
                    copyFromIndexToWorkTree localRoot (unPath dest)
                    Dir.renameFile tmpPath srcPath
```

---

## Bit/Core/Verify.hs

**Path:** `Bit/Core/Verify.hs`

*Source file.*

```haskell
{-# LANGUAGE ScopedTypeVariables #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE OverloadedRecordDot #-}

module Bit.Core.Verify
    ( VerifyTarget(..)
    , verify
    , fsck
    ) where

import System.FilePath ((</>))
import Control.Monad (when)
import Data.Foldable (traverse_)
import Control.Monad.IO.Class (liftIO)
import Control.Monad.Trans.Reader (asks)
import Control.Exception (finally)
import Control.Concurrent (forkIO, threadDelay, killThread)
import Data.IORef (IORef, newIORef, readIORef)
import System.IO (stderr, hIsTerminalDevice)

import Bit.Types (BitM, BitEnv(..))
import Bit.Concurrency (Concurrency)
import qualified Bit.Verify as Verify
import qualified Bit.Fsck as Fsck
import Internal.Config (fetchedBundle)
import Bit.Progress (reportProgress, clearProgress)

import qualified Bit.Device as Device
import Bit.Core.Helpers (withRemote, printVerifyIssue, getRemoteType)
import Bit.Remote (Remote, remoteName, remoteUrl)

-- | Whether to verify local working tree or remote.
data VerifyTarget = VerifyLocal | VerifyRemote
  deriving (Show, Eq)

verify :: VerifyTarget -> Concurrency -> BitM ()
verify target concurrency = case target of
  VerifyRemote -> withRemote $ \remote -> do
      cwd <- asks envCwd
      mType <- liftIO $ getRemoteType cwd (remoteName remote)
      let isFilesystem = maybe False Device.isFilesystemType mType
      if isFilesystem
        then verifyFilesystemRemote (remoteUrl remote) concurrency
        else verifyCloudRemote cwd remote concurrency

  VerifyLocal -> do
      cwd <- asks envCwd
      let indexDir = cwd </> ".bit/index"
      meta <- liftIO $ Verify.loadBinaryMetadata indexDir concurrency
      let fileCount = length meta

      if fileCount > 5
        then liftIO $ do
          isTTY <- hIsTerminalDevice stderr
          counter <- newIORef (0 :: Int)
          let shouldShowProgress = isTTY

          reporterThread <- if shouldShowProgress
            then Just <$> forkIO (verifyProgressLoop counter fileCount)
            else pure Nothing

          result <- finally
            (Verify.verifyLocal cwd (Just counter) concurrency)
            (do
              traverse_ killThread reporterThread
              when shouldShowProgress clearProgress
            )

          printVerifyResult truncateHash " Run 'bit status' for details." result
        else liftIO $ do
          result <- Verify.verifyLocal cwd Nothing concurrency
          printVerifyResult truncateHash " Run 'bit status' for details." result

-- | Verify a filesystem remote by scanning its working directory.
verifyFilesystemRemote :: FilePath -> Concurrency -> BitM ()
verifyFilesystemRemote remotePath concurrency = liftIO $ do
    putStrLn "Verifying remote files..."
    result <- Verify.verifyLocalAt remotePath Nothing concurrency
    printVerifyResult truncateHash "" result

-- | Verify a cloud remote using the fetched bundle.
verifyCloudRemote :: FilePath -> Remote -> Concurrency -> BitM ()
verifyCloudRemote cwd remote concurrency = liftIO $ do
    putStrLn "Fetching remote metadata..."
    putStrLn "Scanning remote files..."

    entries <- Verify.loadMetadataFromBundle fetchedBundle
    let fileCount = length entries

    if fileCount > 5
      then do
        isTTY <- hIsTerminalDevice stderr
        counter <- newIORef (0 :: Int)
        let shouldShowProgress = isTTY

        reporterThread <- if shouldShowProgress
          then Just <$> forkIO (verifyProgressLoop counter fileCount)
          else pure Nothing

        result <- finally
          (Verify.verifyRemote cwd remote (Just counter) concurrency)
          (do
            traverse_ killThread reporterThread
            when shouldShowProgress clearProgress
          )

        printVerifyResult truncateHash "" result
      else do
        result <- Verify.verifyRemote cwd remote Nothing concurrency
        printVerifyResult truncateHash "" result

verifyProgressLoop :: IORef Int -> Int -> IO ()
verifyProgressLoop counter total = go
  where
    go = do
      n <- readIORef counter
      let pct = (n * 100) `div` max 1 total
      reportProgress $ "Checking files: " ++ show n ++ "/" ++ show total ++ " (" ++ show pct ++ "%)"
      threadDelay 100000
      when (n < total) go

-- | Truncate a hash string to 16 characters with ellipsis.
truncateHash :: String -> String
truncateHash s = take 16 s ++ if length s > 16 then "..." else ""

-- | Print VerifyResult: success message or issues plus summary.
-- hashFn: how to display hashes (e.g. truncateHash or id).
-- suffix: appended to the failure line (e.g. " Run 'bit status' for details." or "").
printVerifyResult :: (String -> String) -> String -> Verify.VerifyResult -> IO ()
printVerifyResult hashFn suffix result =
  if null result.vrIssues
    then putStrLn $ "[OK] All " ++ show result.vrCount ++ " files match metadata."
    else do
      mapM_ (printVerifyIssue hashFn) result.vrIssues
      putStrLn $ "Checked " ++ show result.vrCount ++ " files. " ++ show (length result.vrIssues) ++ " issues found." ++ suffix

fsck :: FilePath -> IO ()
fsck = Fsck.doFsck

```

---

## Bit/Device.hs

**Path:** `Bit/Device.hs`

*Source file.*

```haskell
{-# LANGUAGE OverloadedStrings #-}

-- | Device-identity-based remote resolution for filesystem remotes.
-- Cloud remotes (rclone) use URL-based identity; filesystem remotes use
-- UUID + hardware serial (physical) or UUID only (network).
module Bit.Device
  ( -- Types
    StorageType(..)
  , DeviceInfo(..)
  , RemotePathType(..)
  , RemoteTarget(..)
  , ResolveResult(..)
  , RemoteType(..)
    -- Classification
  , classifyRemotePath
  , getRcloneRemotes
    -- Volume ops
  , getVolumeRoot
  , getRelativePath
  , detectStorageType
  , getHardwareSerial
  , getVolumeLabel
  , isFixedDrive
    -- .bit-store
  , readBitStore
  , writeBitStore
    -- Device/remote files
  , readDeviceFile
  , writeDeviceFile
  , readRemoteFile
  , readRemoteType
  , writeRemoteFile
  , listDeviceNames
  , findDeviceByUuid
    -- Resolution
  , resolveRemoteTarget
  , parseRemoteTarget
  , generateStoreUuid
    -- Predicates
  , isFilesystemTarget
  , isFilesystemType
  ) where

import Data.Char (isSpace)
import Data.List (dropWhileEnd, isPrefixOf, intercalate)
import Data.Maybe (fromMaybe, listToMaybe)
import Control.Monad (when, filterM, join)
import Data.Time (getCurrentTime, formatTime, defaultTimeLocale)
import qualified System.Directory as Dir
import System.FilePath ((</>), pathSeparator, takeDrive)
import System.Process (readProcessWithExitCode)
import System.Exit (ExitCode(ExitSuccess))
import qualified System.Info as Info
import Data.UUID (UUID, toString, fromString)
import Data.UUID.V4 (nextRandom)
import Internal.Config (bitDevicesDir, bitRemotesDir)
-- Strict IO imports to avoid Windows file locking issues
import qualified Data.ByteString as BS
import qualified Data.Text as T
import Data.Text.Encoding (decodeUtf8', encodeUtf8)
import Bit.AtomicWrite (atomicWriteFile)

-- ---------------------------------------------------------------------------
-- Types
-- ---------------------------------------------------------------------------

data StorageType = Physical | Network
  deriving (Show, Eq)

data DeviceInfo = DeviceInfo
  { deviceUuid     :: UUID
  , deviceType     :: StorageType
  , hardwareSerial :: Maybe String
  }
  deriving (Show, Eq)

-- | Result of classifying a remote path
data RemotePathType
  = CloudRemote String       -- Pass to rclone as-is (e.g. "gdrive:Projects/foo")
  | FilesystemPath FilePath  -- Enter device flow
  deriving (Show, Eq)

-- | Parsed remote target from .rgit/remotes/<name>
data RemoteTarget
  = TargetCloud String           -- Cloud URL for rclone
  | TargetDevice String FilePath -- device_name : relative_path
  | TargetLocalPath FilePath     -- Legacy: path when .bit-store at volume root cannot be created
  deriving (Show, Eq)

-- | True for targets that resolve to a local filesystem path (device or direct path).
isFilesystemTarget :: RemoteTarget -> Bool
isFilesystemTarget (TargetDevice _ _) = True
isFilesystemTarget (TargetLocalPath _) = True
isFilesystemTarget (TargetCloud _) = False

data ResolveResult
  = Resolved FilePath     -- Runtime path (e.g. E:\Backup)
  | NotConnected String   -- Device not found
  deriving (Show, Eq)

-- | Classification of a remote by transport type.
data RemoteType = RemoteFilesystem | RemoteDevice | RemoteCloud
  deriving (Show, Eq)

-- | True for types that resolve to a local filesystem path.
isFilesystemType :: RemoteType -> Bool
isFilesystemType RemoteCloud = False
isFilesystemType _           = True

-- ---------------------------------------------------------------------------
-- Classification: cloud vs filesystem
-- ---------------------------------------------------------------------------

-- | Check if path is a cloud rclone remote or a filesystem path.
-- Cloud: "remotename:path" where remotename is in rclone listremotes.
classifyRemotePath :: String -> IO RemotePathType
classifyRemotePath path = do
  rcloneRemotes <- getRcloneRemotes
  case break (== ':') path of
    (prefix, _:_rest) | not (null prefix) -> do
      let prefixNorm = dropWhile (== ':') prefix
      if prefixNorm `elem` rcloneRemotes
        then pure (CloudRemote path)
        else pure (FilesystemPath path)
    _ -> pure (FilesystemPath path)

-- | Get list of configured rclone remote names (without trailing colon)
getRcloneRemotes :: IO [String]
getRcloneRemotes = do
  (code, out, _) <- readProcessWithExitCode "rclone" ["listremotes"] ""
  pure $ case code of
    ExitSuccess ->
      [ takeWhile (/= ':') (takeWhile (/= '\n') line)
      | line <- lines out
      , not (null (trimLine line))
      ]
    _ -> []
  where trimLine = dropWhileEnd (== ' ') . dropWhile (== ' ')

-- ---------------------------------------------------------------------------
-- Volume operations (platform-specific)
-- ---------------------------------------------------------------------------

-- | Get the volume root for a path (e.g. D:\Backup -> D:\, \\server\share\foo -> \\server\share\)
getVolumeRoot :: FilePath -> IO FilePath
getVolumeRoot path = do
  absPath <- Dir.makeAbsolute path
  if isWindows then pure (winVolumeRoot absPath)
  else linuxVolumeRootIO absPath

isWindows :: Bool
isWindows = Info.os == "mingw32" || Info.os == "win32"

winVolumeRoot :: FilePath -> FilePath
winVolumeRoot p
  | "\\\\" `isPrefixOf` p || "//" `isPrefixOf` p =
      -- UNC path: \\server\share\path -> \\server\share\
      let sep = if pathSeparator == '\\' then '\\' else '/'
          parts = splitPathOnSep p
      in if length parts >= 3
         then intercalate [sep] (take 3 parts) ++ [sep]
         else p
  | otherwise =
      -- Drive letter: D:\path -> D:\
      let drive = takeDrive p
      in if null drive then p else addTrailingSep drive

-- | Get volume root on Linux using findmnt
linuxVolumeRootIO :: FilePath -> IO FilePath
linuxVolumeRootIO path = do
  (code, out, _) <- readProcessWithExitCode "sh" ["-c", "findmnt -n -o TARGET -T " ++ shellEscape path ++ " 2>/dev/null || echo " ++ shellEscape path] ""
  pure $ case code of
    ExitSuccess | not (null (trim out)) -> trim out
    _ -> path
  where
    shellEscape s = "'" ++ concatMap (\c -> if c == '\'' then "'\\''" else [c]) s ++ "'"

addTrailingSep :: FilePath -> FilePath
addTrailingSep p = case reverse p of
  (c:_) | c == pathSeparator -> p
  _ -> p ++ [pathSeparator]

splitPathOnSep :: FilePath -> [String]
splitPathOnSep = splitOn (== pathSeparator)

splitOn :: (a -> Bool) -> [a] -> [[a]]
splitOn _ [] = []
splitOn p s = case break p s of
  (chunk, [])     -> [chunk]
  (chunk, _:rest) -> chunk : splitOn p rest

-- | Get path relative to volume root
getRelativePath :: FilePath -> FilePath -> FilePath
getRelativePath volumeRoot fullPath = fromMaybe fullPath (stripPrefix' volumeRoot fullPath)

stripPrefix' :: FilePath -> FilePath -> Maybe FilePath
stripPrefix' prefix path =
  let norm = normalisePath
      p = norm prefix
      s = norm path
  in if p `isPrefixOf` s then Just (drop (length p) s) else Nothing

normalisePath :: FilePath -> FilePath
normalisePath = filter (/= '"') . map (\c -> if c == '/' then pathSeparator else c)

-- | Detect physical vs network storage
detectStorageType :: FilePath -> IO StorageType
detectStorageType volumeRoot
  | isWindows = detectStorageTypeWindows volumeRoot
  | otherwise = detectStorageTypeLinux volumeRoot

detectStorageTypeWindows :: FilePath -> IO StorageType
detectStorageTypeWindows volRoot = do
  let drive = take 2 (filter (`elem` ['A'..'Z'] ++ ['a'..'z'] ++ ":") volRoot)
  case drive of
    [] -> pure Physical  -- UNC: treat as network
    (d:_) -> do
      (_code, _out, _) <- readProcessWithExitCode "powershell" ["-NoProfile", "-Command",
        "try { (Get-PSDrive -Name " ++ [d] ++ " -ErrorAction SilentlyContinue).Root } catch { '' }"] ""
      (code2, out2, _) <- readProcessWithExitCode "powershell" ["-NoProfile", "-Command",
        "[int]([System.IO.DriveInfo]::new('" ++ drive ++ "').DriveType)"] ""
      pure $ case code2 of
        ExitSuccess -> case trim out2 of
          "4" -> Network  -- DriveType.Network
          _   -> Physical
        _ -> Physical

detectStorageTypeLinux :: FilePath -> IO StorageType
detectStorageTypeLinux _ = do
  -- Read /proc/mounts and check mount type for the path's mount point
  (code, out, _) <- readProcessWithExitCode "findmnt" ["-n", "-o", "FSTYPE", "-T", "/"] ""
  pure $ case code of
    ExitSuccess ->
      let fstype = trim out
      in if fstype `elem` ["nfs", "nfs4", "cifs", "smb", "smbfs", "sshfs"]
            then Network else Physical
    _ -> Physical

-- | Is the volume a fixed (non-removable, non-network) drive?
-- Fixed drives use RemoteFilesystem; removable/network use RemoteDevice.
isFixedDrive :: FilePath -> IO Bool
isFixedDrive volumeRoot
  | isWindows = isFixedDriveWindows volumeRoot
  | otherwise = (== Physical) <$> detectStorageType volumeRoot

isFixedDriveWindows :: FilePath -> IO Bool
isFixedDriveWindows volRoot = do
  let drive = take 2 (filter (`elem` ['A'..'Z'] ++ ['a'..'z'] ++ ":") volRoot)
  case drive of
    [] -> pure False  -- UNC: treat as network
    _ -> do
      (code, out, _) <- readProcessWithExitCode "powershell" ["-NoProfile", "-Command",
        "[int]([System.IO.DriveInfo]::new('" ++ drive ++ "').DriveType)"] ""
      pure $ case code of
        ExitSuccess -> trim out == "3"  -- DriveType.Fixed == 3
        _ -> True  -- Default to fixed if detection fails

trim :: String -> String
trim = dropWhileEnd isSpace . dropWhile isSpace

-- | Get hardware serial for a physical volume
getHardwareSerial :: FilePath -> IO (Maybe String)
getHardwareSerial volumeRoot
  | isWindows = getHardwareSerialWindows volumeRoot
  | otherwise = getHardwareSerialLinux volumeRoot

getHardwareSerialWindows :: FilePath -> IO (Maybe String)
getHardwareSerialWindows volRoot = do
  let drive = take 1 (filter (`elem` ['A'..'Z'] ++ ['a'..'z']) volRoot)
  if null drive then pure Nothing
  else do
    -- Map partition to physical disk via partition number, then get disk serial
    (code, out, _) <- readProcessWithExitCode "wmic" ["diskdrive", "get", "SerialNumber,Index"] ""
    case code of
      ExitSuccess -> do
        (_code2, _out2, _) <- readProcessWithExitCode "wmic" ["path", "win32_logicaldisk", "where", "DeviceID='" ++ drive ++ ":\\'", "get", "VolumeSerialNumber"] ""
        -- VolumeSerialNumber is the FAT/NTFS serial, not disk serial. Use diskdrive.
        let lines' = filter (not . null . trim) (lines out)
            parseSerial = case lines' of
              _:rest -> listToMaybe [ trim (drop 12 l) | l <- rest, length l > 12 ]
              _      -> Nothing
        pure parseSerial
      _ -> pure Nothing

getHardwareSerialLinux :: FilePath -> IO (Maybe String)
getHardwareSerialLinux _ = do
  (code, out, _) <- readProcessWithExitCode "sh" ["-c", "lsblk -o SERIAL,MOUNTPOINT -n 2>/dev/null | head -20"] ""
  pure $ case code of
    ExitSuccess -> listToMaybe [ trim (takeWhile (/= ' ') l) | l <- lines out, "/" `isPrefixOf` (drop 20 l) ]
    _ -> Nothing

-- | Get volume label for device name suggestion
getVolumeLabel :: FilePath -> IO (Maybe String)
getVolumeLabel volumeRoot
  | isWindows = getVolumeLabelWindows volumeRoot
  | otherwise = getVolumeLabelLinux volumeRoot

getVolumeLabelWindows :: FilePath -> IO (Maybe String)
getVolumeLabelWindows volRoot = do
  let drive = take 2 (filter (`elem` ['A'..'Z'] ++ ['a'..'z'] ++ ":\\") volRoot)
  if length drive < 2 then pure Nothing
  else do
    -- vol is a cmd built-in, not an executable; run via cmd /c
    (code, out, _) <- readProcessWithExitCode "cmd" ["/c", "vol", drive] ""
    pure $ case code of
      ExitSuccess ->
        let lines' = lines out
            volLine = listToMaybe [ l | l <- lines', "Volume" `isPrefixOf` l ]
        in case volLine of
          Just l -> let after = dropWhile (/= ' ') (drop 6 l) in Just (trim after)
          _      -> Nothing
      _ -> Nothing

getVolumeLabelLinux :: FilePath -> IO (Maybe String)
getVolumeLabelLinux _ = do
  (code, out, _) <- readProcessWithExitCode "sh" ["-c", "lsblk -o LABEL,MOUNTPOINT -n 2>/dev/null | head -5"] ""
  pure $ case code of
    ExitSuccess -> listToMaybe [ trim (takeWhile (/= ' ') l) | l <- lines out, not (null (trim l)) ]
    _ -> Nothing

-- ---------------------------------------------------------------------------
-- .bit-store (on device at volume root)
-- ---------------------------------------------------------------------------

bitStoreFileName :: FilePath
bitStoreFileName = ".bit-store"

readBitStore :: FilePath -> IO (Maybe UUID)
readBitStore volumeRoot = do
  let storePath = volumeRoot </> bitStoreFileName
  exists <- Dir.doesFileExist storePath
  if not exists then pure Nothing
  else do
    -- Use strict ByteString reading to avoid Windows file locking issues
    bs <- BS.readFile storePath
    let content = either (const "") T.unpack (decodeUtf8' bs)
    pure (parseBitStoreUuid content)

parseBitStoreUuid :: String -> Maybe UUID
parseBitStoreUuid content =
  join (listToMaybe [ fromString (trim (drop 5 line)) | line <- lines content, "uuid:" `isPrefixOf` line ])

writeBitStore :: FilePath -> UUID -> IO ()
writeBitStore volumeRoot u = do
  now <- getCurrentTime
  let ts = formatTime defaultTimeLocale "%Y-%m-%dT%H:%M:%SZ" now
  let content = unlines
        [ "uuid: " ++ toString u
        , "created: " ++ ts
        ]
  let storePath = volumeRoot </> bitStoreFileName
  -- Use atomic write for crash safety and Windows compatibility
  atomicWriteFile storePath (encodeUtf8 (T.pack content))
  when isWindows $ setHidden storePath

setHidden :: FilePath -> IO ()
setHidden path = do
  (_, _, _) <- readProcessWithExitCode "attrib" ["+H", path] ""
  pure ()

generateStoreUuid :: IO UUID
generateStoreUuid = nextRandom

-- ---------------------------------------------------------------------------
-- Device files (.rgit/devices/<device_name>)
-- ---------------------------------------------------------------------------

parseDeviceFile :: String -> Maybe DeviceInfo
parseDeviceFile content =
  let ls = lines content
      getVal prefix = listToMaybe [ trim (drop (length prefix) l) | l <- ls, prefix `isPrefixOf` l ]
      muuid = getVal "uuid: " >>= fromString
      mtype = getVal "type: "
      mserial = getVal "hardware_serial: "
  in case (muuid, mtype) of
       (Just u, Just "physical") -> Just (DeviceInfo u Physical mserial)
       (Just u, Just "network")  -> Just (DeviceInfo u Network Nothing)
       _                         -> Nothing

readDeviceFile :: FilePath -> String -> IO (Maybe DeviceInfo)
readDeviceFile repoRoot deviceName = do
  let path = repoRoot </> bitDevicesDir </> deviceName
  exists <- Dir.doesFileExist path
  if not exists then pure Nothing
  else do
    -- Use strict ByteString reading to avoid Windows file locking issues
    bs <- BS.readFile path
    let content = either (const "") T.unpack (decodeUtf8' bs)
    pure (parseDeviceFile content)

writeDeviceFile :: FilePath -> String -> DeviceInfo -> IO ()
writeDeviceFile repoRoot deviceName info = do
  Dir.createDirectoryIfMissing True (repoRoot </> bitDevicesDir)
  let path = repoRoot </> bitDevicesDir </> deviceName
  let body = unlines $
        [ "uuid: " ++ toString (deviceUuid info)
        , "type: " ++ (case deviceType info of Physical -> "physical"; Network -> "network")
        ] ++ [ "hardware_serial: " ++ s | Just s <- [hardwareSerial info] ]
  -- Use atomic write for crash safety and Windows compatibility
  atomicWriteFile path (encodeUtf8 (T.pack body))

listDeviceNames :: FilePath -> IO [String]
listDeviceNames repoRoot = do
  let dir = repoRoot </> bitDevicesDir
  exists <- Dir.doesDirectoryExist dir
  if not exists then pure []
  else filter (not . null) <$> Dir.listDirectory dir

findDeviceByUuid :: FilePath -> UUID -> IO (Maybe String)
findDeviceByUuid repoRoot targetUuid = do
  names <- listDeviceNames repoRoot
  foldr go (pure Nothing) names
  where
    go name acc = do
      mInfo <- readDeviceFile repoRoot name
      case mInfo of
        Just info | deviceUuid info == targetUuid -> pure (Just name)
        _ -> acc

-- ---------------------------------------------------------------------------
-- Remote files (.rgit/remotes/<remote_name>)
-- ---------------------------------------------------------------------------

data ParsedTarget = ParsedLocal FilePath | ParsedDevice String FilePath | ParsedCloud String

parseRemoteFile :: String -> Maybe ParsedTarget
parseRemoteFile content =
  let ls = lines content
      getVal prefix = listToMaybe [ trim (drop (length prefix) l) | l <- ls, prefix `isPrefixOf` l ]
  in getVal "target: " >>= parseTarget
  where
    parseTarget s
      | "local:" `isPrefixOf` s = Just (ParsedLocal (trim (drop 6 s)))
      | otherwise = case break (== ':') s of
          (device, ':' : path) | not (null device) -> Just (ParsedDevice device (trim path))
          _ -> Just (ParsedCloud s)

readRemoteFile :: FilePath -> String -> IO (Maybe RemoteTarget)
readRemoteFile repoRoot remoteName = do
  let path = repoRoot </> bitRemotesDir </> remoteName
  exists <- Dir.doesFileExist path
  if not exists then pure Nothing
  else do
    -- Use strict ByteString reading to avoid Windows file locking issues
    bs <- BS.readFile path
    let content = either (const "") T.unpack (decodeUtf8' bs)
    let raw = parseRemoteFile content
    case raw of
      Nothing -> pure Nothing
      Just (ParsedLocal p) -> pure (Just (TargetLocalPath p))
      Just (ParsedCloud url) -> pure (Just (TargetCloud url))
      Just (ParsedDevice device relPath) -> do
        mDev <- readDeviceFile repoRoot device
        pure $ Just $ maybe (TargetCloud (device ++ ":" ++ relPath))
          (const $ TargetDevice device relPath) mDev

-- | Read the remote type from .bit/remotes/<name>.
-- New format: "type: filesystem|device|cloud". Old format: inferred from "target:" line.
readRemoteType :: FilePath -> String -> IO (Maybe RemoteType)
readRemoteType repoRoot name = do
  let path = repoRoot </> bitRemotesDir </> name
  exists <- Dir.doesFileExist path
  if not exists then pure Nothing
  else do
    bs <- BS.readFile path
    let content = either (const "") T.unpack (decodeUtf8' bs)
        ls = lines content
        getVal prefix = listToMaybe [ trim (drop (length prefix) l) | l <- ls, prefix `isPrefixOf` l ]
    case getVal "type: " of
      Just "filesystem" -> pure (Just RemoteFilesystem)
      Just "device"     -> pure (Just RemoteDevice)
      Just "cloud"      -> pure (Just RemoteCloud)
      _ -> -- Old format: infer from target line
        case getVal "target: " of
          Just t | "local:" `isPrefixOf` t -> pure (Just RemoteFilesystem)
          Just t -> case break (== ':') t of
            (dev, ':':_) | not (null dev) -> do
              mDev <- readDeviceFile repoRoot dev
              pure $ Just $ maybe RemoteCloud (const RemoteDevice) mDev
            _ -> pure (Just RemoteCloud)
          Nothing -> pure Nothing

writeRemoteFile :: FilePath -> String -> RemoteType -> Maybe String -> IO ()
writeRemoteFile repoRoot name remoteType mTarget = do
  Dir.createDirectoryIfMissing True (repoRoot </> bitRemotesDir)
  let path = repoRoot </> bitRemotesDir </> name
  let content = case remoteType of
        RemoteFilesystem -> "type: filesystem"
        RemoteCloud      -> "type: cloud\ntarget: " ++ fromMaybe "" mTarget
        RemoteDevice     -> "type: device\ntarget: " ++ fromMaybe "" mTarget
  -- Use atomic write for crash safety and Windows compatibility
  atomicWriteFile path (encodeUtf8 (T.pack content))

-- | Parse a target string (e.g. "black_usb:Backup" or "gdrive:Projects/foo")
parseRemoteTarget :: String -> RemoteTarget
parseRemoteTarget s = case break (== ':') s of
  (prefix, _:rest) | not (null prefix) -> TargetDevice prefix (trim rest)
  _ -> TargetCloud s

-- ---------------------------------------------------------------------------
-- Resolution: device:path -> runtime path
-- ---------------------------------------------------------------------------

resolveRemoteTarget :: FilePath -> RemoteTarget -> IO ResolveResult
resolveRemoteTarget _repoRoot (TargetCloud url) = pure (Resolved url)
resolveRemoteTarget _repoRoot (TargetLocalPath p) = pure (Resolved p)
resolveRemoteTarget repoRoot (TargetDevice deviceName relPath) = do
  mInfo <- readDeviceFile repoRoot deviceName
  maybe (pure (NotConnected ("Device '" ++ deviceName ++ "' not found in .rgit/devices/")))
    (\info -> do
      mMount <- resolveDevice info
      maybe (pure (NotConnected ("Device '" ++ deviceName ++ "' is not connected")))
        (\mountRoot -> pure (Resolved (mountRoot </> relPath)))
        mMount
    ) mInfo

-- | Search for a device and return its volume root if found
resolveDevice :: DeviceInfo -> IO (Maybe FilePath)
resolveDevice info =
  case deviceType info of
    Physical -> resolvePhysicalDevice info
    Network  -> resolveNetworkDevice info

resolvePhysicalDevice :: DeviceInfo -> IO (Maybe FilePath)
resolvePhysicalDevice info = do
  mounts <- getPhysicalMountPoints
  found <- filterM (checkMountForDevice info) mounts
  pure (listToMaybe found)

resolveNetworkDevice :: DeviceInfo -> IO (Maybe FilePath)
resolveNetworkDevice info = do
  mounts <- getNetworkMountPoints
  found <- filterM (checkMountForDevice info) mounts
  pure (listToMaybe found)

checkMountForDevice :: DeviceInfo -> FilePath -> IO Bool
checkMountForDevice info mountRoot = do
  mStoreUuid <- readBitStore mountRoot
  pure $ mStoreUuid == Just (deviceUuid info)

getPhysicalMountPoints :: IO [FilePath]
getPhysicalMountPoints
  | isWindows = getWindowsPhysicalMounts
  | otherwise = getLinuxMountPoints (const False)  -- physical = not network

getNetworkMountPoints :: IO [FilePath]
getNetworkMountPoints
  | isWindows = getWindowsNetworkMounts
  | otherwise = getLinuxMountPoints (const True)  -- network only

getLinuxMountPoints :: (String -> Bool) -> IO [FilePath]
getLinuxMountPoints _typeFilter = do
  -- Parse /proc/mounts for mount points; full impl would filter by fstype
  (code, out, _) <- readProcessWithExitCode "sh" ["-c", "awk '{print $2}' /proc/mounts 2>/dev/null | sort -u"] ""
  pure $ case code of
    ExitSuccess -> filter (not . null) (lines out)
    _ -> ["/"]

getWindowsPhysicalMounts :: IO [FilePath]
getWindowsPhysicalMounts = do
  (code, out, _) <- readProcessWithExitCode "wmic" ["logicaldisk", "where", "DriveType=2 or DriveType=3", "get", "DeviceID"] ""
  pure $ case code of
    ExitSuccess ->
      [ trim l ++ "\\"
      | l <- lines out
      , let t = trim l
      , length t == 2
      , case reverse t of
          (':':_) -> True
          _ -> False
      ]
    _ -> []

getWindowsNetworkMounts :: IO [FilePath]
getWindowsNetworkMounts = do
  (code, out, _) <- readProcessWithExitCode "wmic" ["logicaldisk", "where", "DriveType=4", "get", "DeviceID"] ""
  pure $ case code of
    ExitSuccess ->
      [ trim l ++ "\\"
      | l <- lines out
      , let t = trim l
      , length t == 2
      , case reverse t of
          (':':_) -> True
          _ -> False
      ]
    _ -> []
```

---

## Bit/DevicePrompt.hs

**Path:** `Bit/DevicePrompt.hs`

*Source file.*

```haskell
{-# LANGUAGE LambdaCase #-}
{-# LANGUAGE MultiWayIf #-}

-- | Device name acquisition for filesystem remotes.
-- Supports injectable I/O for testing the interactive path.
module Bit.DevicePrompt
  ( InputSource(..)
  , acquireDeviceName
  , acquireDeviceNameAuto
  , sanitizeDeviceName
  , isValidDeviceName
  ) where

import System.Environment (lookupEnv)
import System.IO (hFlush, stdout, hIsTerminalDevice, stdin, hPutStrLn, stderr)
import Data.Maybe (fromMaybe)
import System.Exit (exitWith, ExitCode(ExitFailure))
import Control.Monad (when, unless)
import Data.Char (isSpace)
import Data.List (dropWhileEnd)

-- | Source of user input for device name.
data InputSource
  = Interactive (IO String)  -- ^ Action that prompts and reads (e.g. getLine)
  | NonInteractive           -- ^ Use default without prompting

-- | Sanitize a label for use as device name (alphanumeric, underscores, hyphens only)
sanitizeDeviceName :: String -> String
sanitizeDeviceName s =
  let replaceSpace c = if c == ' ' then '_' else c
      withUnderscores = map replaceSpace s
      valid c = c `elem` (['a'..'z']++['A'..'Z']++['0'..'9']++"_-")
      cleaned = filter valid withUnderscores
  in if null cleaned then "device" else cleaned

-- | Validate device name: alphanumeric, underscores, hyphens
isValidDeviceName :: String -> Bool
isValidDeviceName s = not (null s) && all (\c -> c `elem` (['a'..'z']++['A'..'Z']++['0'..'9']++"_-")) s

trim :: String -> String
trim = dropWhileEnd isSpace . dropWhile isSpace

-- | Acquire device name from the given input source.
-- Applies sanitization to user input. Empty/whitespace input uses default.
acquireDeviceName
  :: InputSource
  -> Maybe String           -- ^ Volume label (for default suggestion)
  -> (String -> IO Bool)    -- ^ Check if name already exists
  -> IO String
acquireDeviceName inputSource mLabel nameExists = do
  let rawDefault = fromMaybe "device" mLabel
      defaultName = sanitizeDeviceName rawDefault
  finalName <- case inputSource of
    NonInteractive -> pure defaultName
    Interactive ask -> do
      putStrLn "This path is on a storage device."
      putStrLn "bit identifies devices, not drive letters. The remote will stay linked"
      putStrLn "to this device even if the drive letter changes."
      putStrLn ""
      putStr $ "Name this device [" ++ defaultName ++ "]: "
      hFlush stdout
      line <- ask
      let name = trim line
      pure $ if null name then defaultName else sanitizeDeviceName name
  unless (isValidDeviceName finalName) $ do
    hPutStrLn stderr "fatal: Device name must be alphanumeric with underscores/hyphens only."
    exitWith (ExitFailure 1)
  exists <- nameExists finalName
  when exists $ do
    hPutStrLn stderr ("fatal: Device '" ++ finalName ++ "' already exists.")
    exitWith (ExitFailure 1)
  pure finalName

-- | Production entry point: detects TTY or BIT_USE_STDIN for testing.
acquireDeviceNameAuto
  :: Maybe String
  -> (String -> IO Bool)
  -> IO String
acquireDeviceNameAuto mLabel nameExists = do
  isTTY <- hIsTerminalDevice stdin
  useStdin <- (== Just "1") <$> lookupEnv "BIT_USE_STDIN"
  let src = if | isTTY    -> Interactive getLine
               | useStdin -> Interactive getLine  -- For tests: pipe input to stdin
               | otherwise -> NonInteractive
  acquireDeviceName src mLabel nameExists
```

---

## Bit/Diff.hs

**Path:** `Bit/Diff.hs`

*Source file.*

```haskell
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE DuplicateRecordFields #-}
{-# LANGUAGE OverloadedRecordDot #-}

module Bit.Diff
  ( GitDiff(..)
  , LightFileEntry(..)
  , FileIndex(..)
  , buildIndexFromFileEntries
  , computeDiff
  , formatDiff
  ) where

import qualified Data.Map as Map
import qualified Data.Set as Set
import Bit.Types

-- Lightweight identity for planning
data LightFileEntry = LightFileEntry
  { filePath :: Path
  , fileHash :: Hash 'MD5
  } deriving (Eq, Show)

data GitDiff
  = Renamed LightFileEntry LightFileEntry
  | Added LightFileEntry
  | Deleted LightFileEntry
  | Modified LightFileEntry
  deriving (Eq, Show)

type FileMap = Map.Map Path FileEntry

data FileIndex = FileIndex
  { byPath :: FileMap
  , byHash :: Map.Map (Hash 'MD5) (Set.Set Path)
  }

buildIndexFromFileEntries :: [FileEntry] -> FileIndex
buildIndexFromFileEntries files =
    FileIndex
      { byPath = Map.fromList [(entry.path, entry) | entry <- files]
      , byHash = Map.fromListWith Set.union
          [ (h, Set.singleton entry.path)
          | entry <- files
          , h <- case entry.kind of
              File h _ _ -> [h]
              _          -> []
          ]
      }

-- Invariant:
-- FileMap must have paths normalized and hashIndex consistent with files
computeDiff :: FileIndex -> FileIndex -> [GitDiff]
computeDiff local remote =
    modified ++ added ++ deleted ++ renamed
  where
    lFiles :: Map.Map Path (Hash 'MD5)
    lFiles = Map.fromList
      [ (entry.path, h)
      | entry <- Map.elems local.byPath
      , h <- case entry.kind of
          File h _ _ -> [h]
          _          -> []
      ]
    rFiles :: Map.Map Path (Hash 'MD5)
    rFiles = Map.fromList
      [ (entry.path, h)
      | entry <- Map.elems remote.byPath
      , h <- case entry.kind of
          File h _ _ -> [h]
          _          -> []
      ]

    lPaths = Map.keysSet local.byPath
    rPaths = Map.keysSet remote.byPath
    lFilePaths = Map.keysSet lFiles
    rFilePaths = Map.keysSet rFiles

    -- 1. Modified: same path, different hash (only for file paths; directories have no hash)
    modified =
      [ Modified (LightFileEntry p lHash)
      | p <- Set.toList (Set.intersection lFilePaths rFilePaths)
      , Just lHash <- [Map.lookup p lFiles]
      , Just rHash <- [Map.lookup p rFiles]
      , lHash /= rHash
      ]

    -- 2. Added: path exists only locally
    added =
      [ Added (LightFileEntry p hsh)
      | (p, hsh) <- Map.toList lFiles
      , p `Set.notMember` rPaths
      , not (Map.member hsh remote.byHash)
      ]

    -- 3. Deleted: path exists only remotely
    deleted =
      [ Deleted (LightFileEntry p hsh)
      | (p, hsh) <- Map.toList rFiles
      , p `Set.notMember` lPaths
      , not (Map.member hsh local.byHash)
      ]

    -- 4. Renamed: same hash, different path (1:1 only; otherwise we'd emit multiple Move for same source)
    renamed =
      [ Renamed (LightFileEntry oldPath hash) (LightFileEntry newPath hash)
      | (hash, oldPathSet) <- Map.toList remote.byHash
      , oldPath <- Set.toList oldPathSet
      , let localPathsWithHash = Set.filter (/= oldPath) (Map.findWithDefault Set.empty hash local.byHash)
      , Set.size localPathsWithHash == 1
      , newPath <- take 1 (Set.toList localPathsWithHash)
      , Map.member newPath lFiles
      ]

formatDiff :: GitDiff -> String
formatDiff (Added f)      = "[+] Added:    " ++ unPath f.filePath
formatDiff (Deleted f)    = "[-] Deleted:  " ++ unPath f.filePath
formatDiff (Modified f)   = "[*] Modified: " ++ unPath f.filePath
formatDiff (Renamed o n)  = "[M] Moved:    " ++ unPath o.filePath ++ " -> " ++ unPath n.filePath
```

---

## Bit/Effect/Pure.hs

**Path:** `Bit/Effect/Pure.hs`

*Source file.*

```haskell
{-# LANGUAGE GADTs #-}
{-# LANGUAGE FlexibleContexts #-}

module Bit.Effect.Pure
  ( Trace(..)
  , PureEnv(..)
  , runPure
  ) where

import Control.Monad.Free (Free(..))
import Control.Monad.State (State, runState, get, put, modify)
import qualified Data.ByteString as BS
import qualified Data.ByteString.Char8 as BSC
import Data.Foldable (traverse_)
import qualified Data.Map as Map
import Bit.Effect
import System.Exit (ExitCode(..))

-- | Trace of effects for test assertions.
data Trace
  = TGitRaw [String]
  | TGitQuery [String]
  | TRclone String [String]
  | TWrite FilePath String
  | TCopy FilePath FilePath
  | TRemove FilePath
  | TTell String
  | TTellErr String
  | TAskUser String String  -- prompt, simulated response
  | TExit ExitCode
  deriving (Show, Eq)

-- | Simulated environment for pure interpretation.
data PureEnv = PureEnv
  { pureFiles    :: Map.Map FilePath String       -- simulated filesystem
  , pureDirs     :: [FilePath]                    -- simulated directories
  , pureInputs   :: [String]                      -- simulated user inputs (consumed in order)
  , pureTrace    :: [Trace]                       -- accumulated trace (reversed)
  , pureCwd      :: FilePath                      -- simulated working directory
  }

type PureM = State PureEnv

-- | Run a free monad program purely, returning the result and accumulated trace.
runPure :: PureEnv -> Rgit a -> (a, [Trace])
runPure env program =
  let (result, finalEnv) = runState (interpretPure program) env
  in (result, reverse (pureTrace finalEnv))

interpretPure :: Rgit a -> PureM a
interpretPure (Pure a) = pure a
interpretPure (Free f) = case f of
  GitRaw args k -> do
    trace (TGitRaw args)
    interpretPure (k ExitSuccess)
  GitQuery args k -> do
    trace (TGitQuery args)
    interpretPure (k (ExitSuccess, "", ""))
  RcloneExec cmd args k -> do
    trace (TRclone cmd args)
    interpretPure (k ExitSuccess)
  RcloneQuery args k -> do
    trace (TRclone "query" args)
    interpretPure (k (ExitSuccess, "", ""))
  RunProcess _ args _ k -> do
    trace (TGitQuery args)  -- reuse trace type
    interpretPure (k (ExitSuccess, "", ""))
  ReadFileE path k -> do
    env <- get
    interpretPure (k (Map.lookup path (pureFiles env)))
  WriteFileAtomicE path content next -> do
    trace (TWrite path content)
    modify (\e -> e { pureFiles = Map.insert path content (pureFiles e) })
    interpretPure next
  CopyFileE src dest next -> do
    trace (TCopy src dest)
    env <- get
    traverse_ (\content -> modify (\e -> e { pureFiles = Map.insert dest content (pureFiles e) }))
      (Map.lookup src (pureFiles env))
    interpretPure next
  FileExistsE path k -> do
    env <- get
    interpretPure (k (Map.member path (pureFiles env)))
  DirExistsE path k -> do
    env <- get
    interpretPure (k (path `elem` pureDirs env))
  ListDirE _path k -> interpretPure (k [])
  CreateDirE path next -> do
    modify (\e -> e { pureDirs = path : pureDirs e })
    interpretPure next
  RemoveFileE path next -> do
    trace (TRemove path)
    modify (\e -> e { pureFiles = Map.delete path (pureFiles e) })
    interpretPure next
  RemoveDirRecursiveE path next -> do
    trace (TRemove path)
    interpretPure next
  GetFileSizeE path k -> do
    env <- get
    let sz = maybe 0 (fromIntegral . length) (Map.lookup path (pureFiles env))
    interpretPure (k sz)
  ReadFileBytesE path k -> do
    env <- get
    let bs = maybe BS.empty BSC.pack (Map.lookup path (pureFiles env))
    interpretPure (k bs)
  Tell msg next -> do
    trace (TTell msg)
    interpretPure next
  TellErr msg next -> do
    trace (TTellErr msg)
    interpretPure next
  AskUser prompt k -> do
    env <- get
    case pureInputs env of
      (response:rest) -> do
        put env { pureInputs = rest }
        trace (TAskUser prompt response)
        interpretPure (k response)
      [] -> do
        trace (TAskUser prompt "")
        interpretPure (k "")
  GetCurrentDirE k -> do
    env <- get
    interpretPure (k (pureCwd env))
  ExitWithE code next -> do
    trace (TExit code)
    interpretPure next
  LiftIOE _ k -> interpretPure (k (error "liftIOE called in pure interpreter"))

  where
    trace t = modify (\e -> e { pureTrace = t : pureTrace e })
```

---

## Bit/Fsck.hs

**Path:** `Bit/Fsck.hs`

*Source file.*

```haskell
module Bit.Fsck
  ( doFsck
  ) where

import qualified Internal.Git as Git
import System.Exit (ExitCode(..), exitWith)
import System.IO (hPutStr, stderr)
import Control.Monad (unless)

-- | Run git fsck on the internal metadata repository (.bit/index).
-- Passes through git's output and exit code.
doFsck :: FilePath -> IO ()
doFsck _cwd = do
  (code, out, err) <- Git.fsck
  putStr out
  hPutStr stderr err
  unless (code == ExitSuccess) $
    exitWith (ExitFailure 1)
```

---

## Bit/Help.hs

**Path:** `Bit/Help.hs`

*Source file.*

```haskell
module Bit.Help (
    printMainHelp, printTerseHelp, printCommandHelp
) where

import System.Exit (ExitCode(..), exitWith)
import System.IO (hPutStrLn, stderr)
import Control.Monad (unless)

-- | Option or example: (item, description). Record avoids transposition vs (String, String).
data HelpItem = HelpItem { hiItem, hiDescription :: String }
  deriving (Show, Eq)

data CommandHelp = CommandHelp
    { cmdName     :: String
    , cmdSynopsis :: String
    , cmdUsage    :: String
    , cmdDesc     :: [String]
    , cmdOptions  :: [HelpItem]
    , cmdExamples :: [HelpItem]
    }

lookupCommand :: String -> Maybe CommandHelp
lookupCommand name = case filter (\c -> cmdName c == name) commandRegistry of
    (x:_) -> Just x
    []    -> Nothing

-- | Print the main help summary (grouped by category) to stdout.
printMainHelp :: IO ()
printMainHelp = putStr $ unlines
    [ "Usage: bit <command> [options]"
    , ""
    , "Getting started:"
    , "  init                              Initialize a new bit repository"
    , ""
    , "Tracking changes:"
    , "  status                            Show working tree status"
    , "  add <path>                        Add file contents to metadata"
    , "  commit -m <msg>                   Record changes to the repository"
    , "  diff [--staged]                   Show changes"
    , "  log                               Show commit history"
    , ""
    , "File management:"
    , "  rm [options] <path>               Remove files from tracking"
    , "  restore [options] [--] <path>     Restore working tree files"
    , "  checkout [options] -- <path>      Checkout files from index"
    , "  ls-files                          List tracked files"
    , ""
    , "Syncing:"
    , "  push [-u] [<remote>]              Push to remote"
    , "  pull [<remote>] [options]         Pull from remote"
    , "  fetch [<remote>]                  Fetch metadata from remote"
    , ""
    , "Remote management:"
    , "  remote add <name> <url>           Add a remote"
    , "  remote show [<name>]              Show remote information"
    , "  remote repair [<name>]            Verify and repair files against remote"
    , ""
    , "Integrity:"
    , "  verify [--remote]                 Verify files match committed metadata"
    , "  fsck                              Check metadata repository integrity"
    , ""
    , "Merge and branching:"
    , "  merge --continue|--abort          Continue or abort merge"
    , "  branch --unset-upstream           Unset upstream tracking"
    , ""
    , "Remote workspace:"
    , "  --remote <name> <cmd>             Target a remote workspace (portable)"
    , "  @<remote> <cmd>                   Shorthand (needs quoting in PowerShell)"
    , "  Supported: init, add, commit, status, log, ls-files"
    , ""
    , "See 'bit help <command>' for more information on a specific command."
    ]

-- | Print terse help for a command (-h): just the usage line.
printTerseHelp :: String -> IO ()
printTerseHelp name = case lookupCommand name of
    Just cmd -> putStrLn $ "usage: " ++ cmdUsage cmd
    Nothing  -> unknownCommand name

-- | Print detailed help for a command (--help / help <cmd>).
printCommandHelp :: String -> IO ()
printCommandHelp name = case lookupCommand name of
    Just cmd -> do
        putStrLn $ "usage: " ++ cmdUsage cmd
        putStrLn ""
        mapM_ putStrLn (cmdDesc cmd)
        unless (null (cmdOptions cmd)) $ do
            putStrLn ""
            putStrLn "Options:"
            mapM_ printOption (cmdOptions cmd)
        unless (null (cmdExamples cmd)) $ do
            putStrLn ""
            putStrLn "Examples:"
            mapM_ printExample (cmdExamples cmd)
    Nothing -> unknownCommand name

printOption :: HelpItem -> IO ()
printOption item = putStrLn $ "  " ++ padRight 34 (hiItem item) ++ hiDescription item

printExample :: HelpItem -> IO ()
printExample item = putStrLn $ "  " ++ padRight 34 (hiItem item) ++ hiDescription item

padRight :: Int -> String -> String
padRight n s = s ++ replicate (max 1 (n - length s)) ' '

unknownCommand :: String -> IO ()
unknownCommand name = do
    hPutStrLn stderr $ "bit: '" ++ name ++ "' is not a bit command. See 'bit help'."
    exitWith (ExitFailure 1)

-- | Registry of all commands with their help metadata.
commandRegistry :: [CommandHelp]
commandRegistry =
    [ CommandHelp
        { cmdName     = "init"
        , cmdSynopsis = "Initialize a new bit repository"
        , cmdUsage    = "bit init"
        , cmdDesc     = [ "Create an empty bit repository in the current directory."
                        , "This creates a .bit/ directory with an internal Git repository"
                        , "for tracking file metadata." ]
        , cmdOptions  = []
        , cmdExamples = [HelpItem "bit init" "Initialize in current directory"]
        }
    , CommandHelp
        { cmdName     = "status"
        , cmdSynopsis = "Show working tree status"
        , cmdUsage    = "bit status"
        , cmdDesc     = [ "Show the state of the working tree: which files are modified,"
                        , "added, or deleted relative to the last commit." ]
        , cmdOptions  = []
        , cmdExamples = [HelpItem "bit status" "Show status"]
        }
    , CommandHelp
        { cmdName     = "add"
        , cmdSynopsis = "Add file contents to metadata"
        , cmdUsage    = "bit add <path>"
        , cmdDesc     = [ "Compute metadata (hash, size) for the specified files and stage"
                        , "the changes in the internal Git repository."
                        , ""
                        , "Use 'bit add .' to add all modified and new files." ]
        , cmdOptions  = []
        , cmdExamples = [ HelpItem "bit add file.txt" "Add a single file"
                        , HelpItem "bit add ." "Add all files" ]
        }
    , CommandHelp
        { cmdName     = "commit"
        , cmdSynopsis = "Record changes to the repository"
        , cmdUsage    = "bit commit -m <msg>"
        , cmdDesc     = ["Commit staged metadata changes with the given message."]
        , cmdOptions  = [HelpItem "-m <msg>" "Commit message"]
        , cmdExamples = [HelpItem "bit commit -m \"Add new files\"" "Commit with message"]
        }
    , CommandHelp
        { cmdName     = "diff"
        , cmdSynopsis = "Show changes"
        , cmdUsage    = "bit diff [--staged]"
        , cmdDesc     = [ "Show hash/size changes between the working tree and the index."
                        , "With --staged, show changes between the index and HEAD." ]
        , cmdOptions  = [HelpItem "--staged" "Show staged changes"]
        , cmdExamples = [ HelpItem "bit diff" "Show unstaged changes"
                        , HelpItem "bit diff --staged" "Show staged changes" ]
        }
    , CommandHelp
        { cmdName     = "log"
        , cmdSynopsis = "Show commit history"
        , cmdUsage    = "bit log"
        , cmdDesc     = ["Show the commit history of the repository."]
        , cmdOptions  = []
        , cmdExamples = [HelpItem "bit log" "Show full log"]
        }
    , CommandHelp
        { cmdName     = "rm"
        , cmdSynopsis = "Remove files from tracking"
        , cmdUsage    = "bit rm [options] <path>"
        , cmdDesc     = ["Remove files from tracking and optionally from the working tree."]
        , cmdOptions  = []
        , cmdExamples = [HelpItem "bit rm file.txt" "Remove a file"]
        }
    , CommandHelp
        { cmdName     = "restore"
        , cmdSynopsis = "Restore working tree files"
        , cmdUsage    = "bit restore [options] [--] <path>"
        , cmdDesc     = [ "Restore working tree files from the index or a specific commit."
                        , "Supports full git restore syntax: --staged, --worktree, --source=, etc." ]
        , cmdOptions  = [ HelpItem "--staged" "Restore staged changes"
                        , HelpItem "--worktree" "Restore working tree (default)"
                        , HelpItem "--source=<commit>" "Restore from specific commit" ]
        , cmdExamples = [ HelpItem "bit restore -- file.txt" "Restore file from index"
                        , HelpItem "bit restore --staged file.txt" "Unstage a file" ]
        }
    , CommandHelp
        { cmdName     = "checkout"
        , cmdSynopsis = "Checkout files from index"
        , cmdUsage    = "bit checkout [options] -- <path>"
        , cmdDesc     = [ "Restore working tree files from the index (legacy syntax)."
                        , "Prefer 'bit restore' for new usage." ]
        , cmdOptions  = []
        , cmdExamples = [HelpItem "bit checkout -- file.txt" "Restore file from index"]
        }
    , CommandHelp
        { cmdName     = "ls-files"
        , cmdSynopsis = "List tracked files"
        , cmdUsage    = "bit ls-files"
        , cmdDesc     = ["List all files tracked by bit."]
        , cmdOptions  = []
        , cmdExamples = [HelpItem "bit ls-files" "List all tracked files"]
        }
    , CommandHelp
        { cmdName     = "push"
        , cmdSynopsis = "Push to remote"
        , cmdUsage    = "bit push [-u|--set-upstream] [<remote>]"
        , cmdDesc     = [ "Push metadata and files to a remote. Verifies local files match"
                        , "committed metadata before pushing (proof of possession)."
                        , ""
                        , "If no remote is specified, pushes to the upstream remote or falls"
                        , "back to 'origin' if it exists." ]
        , cmdOptions  = [ HelpItem "-u, --set-upstream <remote>" "Push and set upstream tracking"
                        , HelpItem "--force" "Force push (skip ancestry check)"
                        , HelpItem "--force-with-lease" "Force push if remote matches expected state" ]
        , cmdExamples = [ HelpItem "bit push" "Push to default remote"
                        , HelpItem "bit push origin" "Push to named remote"
                        , HelpItem "bit push -u origin" "Push and set upstream tracking" ]
        }
    , CommandHelp
        { cmdName     = "pull"
        , cmdSynopsis = "Pull from remote"
        , cmdUsage    = "bit pull [<remote>] [options]"
        , cmdDesc     = [ "Pull metadata and files from a remote. Verifies remote files"
                        , "match remote metadata before pulling (proof of possession)." ]
        , cmdOptions  = [ HelpItem "--accept-remote" "Accept remote file state as truth"
                        , HelpItem "--manual-merge" "Interactive per-file conflict resolution" ]
        , cmdExamples = [ HelpItem "bit pull" "Pull from default remote"
                        , HelpItem "bit pull origin" "Pull from named remote"
                        , HelpItem "bit pull --accept-remote" "Accept remote state" ]
        }
    , CommandHelp
        { cmdName     = "fetch"
        , cmdSynopsis = "Fetch metadata from remote"
        , cmdUsage    = "bit fetch [<remote>]"
        , cmdDesc     = [ "Fetch metadata from a remote without syncing files."
                        , "Only transfers the metadata bundle, no file content is downloaded." ]
        , cmdOptions  = []
        , cmdExamples = [ HelpItem "bit fetch" "Fetch from default remote"
                        , HelpItem "bit fetch origin" "Fetch from named remote" ]
        }
    , CommandHelp
        { cmdName     = "remote"
        , cmdSynopsis = "Manage remotes"
        , cmdUsage    = "bit remote <subcommand>"
        , cmdDesc     = [ "Manage remote repositories."
                        , ""
                        , "Available subcommands:"
                        , "  add <name> <url>   Add a remote"
                        , "  show [<name>]      Show remote information"
                        , "  repair [<name>]    Verify and repair files against remote" ]
        , cmdOptions  = []
        , cmdExamples = [ HelpItem "bit remote add origin gdrive:Projects/foo" "Add a cloud remote"
                        , HelpItem "bit remote show" "Show all remotes"
                        , HelpItem "bit remote repair origin" "Repair files against remote" ]
        }
    , CommandHelp
        { cmdName     = "remote add"
        , cmdSynopsis = "Add a remote"
        , cmdUsage    = "bit remote add <name> <url>"
        , cmdDesc     = [ "Add a named remote pointing to the given URL."
                        , "Does not set upstream tracking (use 'bit push -u' for that)."
                        , ""
                        , "For network shares under Git Bash / MINGW, use forward slashes:"
                        , "  //server/share/path   (not \\\\server\\share\\path)" ]
        , cmdOptions  = []
        , cmdExamples = [ HelpItem "bit remote add origin gdrive:Projects/foo" "Add a cloud remote"
                        , HelpItem "bit remote add backup /mnt/usb/myproject" "Add a filesystem remote"
                        , HelpItem "bit remote add nas //server/share/project" "Add a network share" ]
        }
    , CommandHelp
        { cmdName     = "remote show"
        , cmdSynopsis = "Show remote information"
        , cmdUsage    = "bit remote show [<name>]"
        , cmdDesc     = [ "Show information about configured remotes."
                        , "With no arguments, lists all remotes."
                        , "With a name, shows detailed information about that remote." ]
        , cmdOptions  = []
        , cmdExamples = [ HelpItem "bit remote show" "List all remotes"
                        , HelpItem "bit remote show origin" "Show details for origin" ]
        }
    , CommandHelp
        { cmdName     = "remote repair"
        , cmdSynopsis = "Verify and repair files against remote"
        , cmdUsage    = "bit remote repair [<name>]"
        , cmdDesc     = [ "Verify both local and remote files against their metadata,"
                        , "then repair broken or missing files by copying from the other side."
                        , "Uses content-addressable lookup (matches by hash, not path)." ]
        , cmdOptions  = [HelpItem "--sequential" "Run verification sequentially (no parallelism)"]
        , cmdExamples = [ HelpItem "bit remote repair" "Repair against default remote"
                        , HelpItem "bit remote repair origin" "Repair against named remote" ]
        }
    , CommandHelp
        { cmdName     = "verify"
        , cmdSynopsis = "Verify files match committed metadata"
        , cmdUsage    = "bit verify [--remote]"
        , cmdDesc     = [ "Verify that files match their committed metadata (hash, size)."
                        , "Without --remote, checks local working tree files."
                        , "With --remote, checks files on the remote." ]
        , cmdOptions  = [ HelpItem "--remote" "Verify remote files instead of local"
                        , HelpItem "--sequential" "Run verification sequentially" ]
        , cmdExamples = [ HelpItem "bit verify" "Verify local files"
                        , HelpItem "bit verify --remote" "Verify remote files" ]
        }
    , CommandHelp
        { cmdName     = "fsck"
        , cmdSynopsis = "Check metadata repository integrity"
        , cmdUsage    = "bit fsck"
        , cmdDesc     = [ "Run 'git fsck' on the internal metadata repository (.bit/index)."
                        , "Checks the integrity of the object store -- that all commits,"
                        , "trees, and blobs are valid and consistent." ]
        , cmdOptions  = []
        , cmdExamples = [HelpItem "bit fsck" "Check integrity"]
        }
    , CommandHelp
        { cmdName     = "merge"
        , cmdSynopsis = "Continue or abort merge"
        , cmdUsage    = "bit merge --continue|--abort"
        , cmdDesc     = [ "Manage an in-progress merge."
                        , ""
                        , "Available subcommands:"
                        , "  --continue   Continue after conflict resolution"
                        , "  --abort      Abort current merge" ]
        , cmdOptions  = []
        , cmdExamples = [ HelpItem "bit merge --continue" "Continue merge after resolving conflicts"
                        , HelpItem "bit merge --abort" "Abort current merge" ]
        }
    , CommandHelp
        { cmdName     = "merge --continue"
        , cmdSynopsis = "Continue after conflict resolution"
        , cmdUsage    = "bit merge --continue"
        , cmdDesc     = ["Continue a merge after manually resolving conflicts."]
        , cmdOptions  = []
        , cmdExamples = [HelpItem "bit merge --continue" "Continue merge"]
        }
    , CommandHelp
        { cmdName     = "merge --abort"
        , cmdSynopsis = "Abort current merge"
        , cmdUsage    = "bit merge --abort"
        , cmdDesc     = ["Abort the current merge operation and restore the pre-merge state."]
        , cmdOptions  = []
        , cmdExamples = [HelpItem "bit merge --abort" "Abort merge"]
        }
    , CommandHelp
        { cmdName     = "branch"
        , cmdSynopsis = "Branch management"
        , cmdUsage    = "bit branch --unset-upstream"
        , cmdDesc     = [ "Branch management commands."
                        , ""
                        , "Available subcommands:"
                        , "  --unset-upstream   Remove upstream tracking configuration" ]
        , cmdOptions  = []
        , cmdExamples = [HelpItem "bit branch --unset-upstream" "Remove upstream tracking"]
        }
    , CommandHelp
        { cmdName     = "branch --unset-upstream"
        , cmdSynopsis = "Unset upstream tracking"
        , cmdUsage    = "bit branch --unset-upstream"
        , cmdDesc     = ["Remove the upstream tracking configuration for the current branch."]
        , cmdOptions  = []
        , cmdExamples = [HelpItem "bit branch --unset-upstream" "Remove upstream tracking"]
        }
    ]
```

---

## Bit/Internal/Metadata.hs

**Path:** `Bit/Internal/Metadata.hs`

*Source file.*

```haskell
{-# LANGUAGE BangPatterns #-}
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE KindSignatures #-}
module Bit.Internal.Metadata
  ( MetaContent(..)
  , serializeMetadata
  , parseMetadata
  , parseMetadataFile
  , readMetadataOrComputeHash
  , displayHash
  , hashFileBytes
  , hashFile
  , hasConflictMarkers
  , validateMetadataDir
  , listAllFiles
  ) where

import Bit.Types (Hash(..), HashAlgo(..), hashToText)
import System.Directory (doesFileExist, doesDirectoryExist, listDirectory)
import System.FilePath ((</>))
import System.IO (withFile, IOMode(ReadMode), hIsEOF)
import Data.List (dropWhileEnd, isPrefixOf, isInfixOf)
import Data.Maybe (listToMaybe)
import Control.Monad (filterM)
import qualified Data.ByteString as BS
import qualified Data.Text as T
import Data.Text.Encoding (decodeUtf8, decodeUtf8')
import qualified Crypto.Hash.MD5 as MD5
import Data.ByteString.Base16 (encode)

-- | The single source of truth for what a metadata file contains.
data MetaContent = MetaContent
  { metaHash :: Hash 'MD5
  , metaSize :: Integer
  } deriving (Show, Eq)

-- | Serialize metadata to canonical format. Total function.
-- Format: "hash: <raw_hash>\nsize: <size>\n"
serializeMetadata :: MetaContent -> String
serializeMetadata mc =
  "hash: " ++ T.unpack (hashToText (metaHash mc)) ++ "\n"
  ++ "size: " ++ show (metaSize mc) ++ "\n"

-- | Parse metadata from string content. Pure function.
-- Accepts both raw hash values and legacy Hash "..." wrappers.
-- Returns Nothing if content doesn't look like metadata (e.g. it's a text file's content).
parseMetadata :: String -> Maybe MetaContent
parseMetadata content = do
  let ls = lines content
  hashLine <- listToMaybe [ drop (length ("hash: " :: String)) l
                          | l <- ls, "hash: " `isPrefixOf` l ]
  sizeLine <- listToMaybe [ drop (length ("size: " :: String)) l
                          | l <- ls, "size: " `isPrefixOf` l ]
  let hashVal = cleanHash (trim hashLine)
  size <- readMaybeInt (trim sizeLine)
  if null hashVal
    then Nothing
    else Just MetaContent
      { metaHash = Hash (T.pack hashVal)
      , metaSize = size
      }
  where
    -- Strip legacy wrappers: Hash "...", bare quotes "...", or raw value
    cleanHash s
      | "Hash \"" `isPrefixOf` s =
          let rest = drop (length ("Hash \"" :: String)) s
          in dropWhileEnd (== '"') rest
      | ('"':rest) <- s = case reverse rest of
          ('"':middle) -> reverse middle
          _ -> s
      | otherwise = s
    trim = dropWhileEnd isSpaceChar . dropWhile isSpaceChar
    isSpaceChar c = c == ' ' || c == '\t'
    readMaybeInt s = case reads s of
      [(n, "")] -> Just n
      [(n, r)] | all isSpaceChar r -> Just n
      _ -> Nothing

-- | Read + parse a metadata file. Returns Nothing if file doesn't exist or is not valid metadata.
parseMetadataFile :: FilePath -> IO (Maybe MetaContent)
parseMetadataFile fp = do
  exists <- doesFileExist fp
  if not exists
    then pure Nothing
    else do
      bs <- BS.readFile fp
      pure $ either (const Nothing) (parseMetadata . T.unpack) (decodeUtf8' bs)

-- | Read a metadata file OR (if it's a text file whose content is stored directly)
-- compute hash/size from the file bytes. This is the replacement for the fallback
-- logic in Rgit.Scan.readMetadataFile and Rgit.Verify.loadBinaryMetadata.
readMetadataOrComputeHash :: FilePath -> IO (Maybe MetaContent)
readMetadataOrComputeHash fp = do
  exists <- doesFileExist fp
  if not exists
    then pure Nothing
    else do
      bs <- BS.readFile fp
      case decodeUtf8' bs of
        Left _ -> do
          -- Binary file — compute hash from bytes
          let h = hashFileBytes bs
              sz = fromIntegral (BS.length bs)
          pure (Just (MetaContent h sz))
        Right txt ->
          case parseMetadata (T.unpack txt) of
            Just mc -> pure (Just mc)
            Nothing -> do
              -- Not a metadata file — treat as text file content, compute hash from bytes
              let h = hashFileBytes bs
                  sz = fromIntegral (BS.length bs)
              pure (Just (MetaContent h sz))

-- | Truncate hash for human-readable display.
-- Shows first 16 chars + "..." if longer.
displayHash :: Hash 'MD5 -> String
displayHash h =
  let s = T.unpack (hashToText h)
  in take 16 s ++ if length s > 16 then "..." else ""

-- | Compute MD5 hash of raw bytes. Single source of truth for hashing.
hashFileBytes :: BS.ByteString -> Hash 'MD5
hashFileBytes bs =
  let md5hex = decodeUtf8 (encode (MD5.hash bs))
  in Hash (T.pack "md5:" <> md5hex)

-- | Compute MD5 hash of a file on disk using streaming (constant memory).
-- Reads file in 64KB chunks to avoid loading entire file into RAM.
hashFile :: FilePath -> IO (Hash 'MD5)
hashFile fp = withFile fp ReadMode $ \handle -> do
  let loop !ctx = do
        eof <- hIsEOF handle
        if eof
          then do
            let md5hex = decodeUtf8 (encode (MD5.finalize ctx))
            pure (Hash (T.pack "md5:" <> md5hex))
          else do
            chunk <- BS.hGet handle 65536  -- 64KB chunks
            loop (MD5.update ctx chunk)
  loop MD5.init

-- Conflict marker utilities (preserved from old Internal.Metadata) --

conflictMarkers :: [String]
conflictMarkers = ["<<<<<<<", "=======", ">>>>>>>"]

hasConflictMarkers :: FilePath -> IO Bool
hasConflictMarkers path = do
  bs <- BS.readFile path
  pure $ either (const False) (\txt -> any (`isInfixOf` T.unpack txt) conflictMarkers) (decodeUtf8' bs)

listAllFiles :: FilePath -> IO [FilePath]
listAllFiles dir = do
  entries <- listDirectory dir
  concat <$> mapM (\name -> do
    let full = dir </> name
    isDir <- doesDirectoryExist full
    if isDir then listAllFiles full else do
      isFile <- doesFileExist full
      pure (if isFile then [full] else [])) entries

validateMetadataDir :: FilePath -> IO [FilePath]
validateMetadataDir dir = do
  files <- listAllFiles dir
  filterM hasConflictMarkers files
```

---

## Bit/Pipeline.hs

**Path:** `Bit/Pipeline.hs`

*Source file.*

```haskell
{-# LANGUAGE DataKinds #-}

module Bit.Pipeline
  ( -- * Pure core (property-testable)
    diffAndPlan
    -- * Composed pipelines
  , pushSyncFiles
  , pullSyncFiles
  ) where

import Bit.Types
import Bit.Diff (buildIndexFromFileEntries, computeDiff)
import Bit.Plan (RcloneAction(..), planAction, resolveSwaps)
import Bit.Utils (filterOutBitPaths)

-- | Pure core: given source-of-truth files and current target files,
-- produce the list of actions to make target match source.
-- This is the entire "diff >>> plan" section — no IO, fully property-testable.
-- For push: local is "source of truth", remote is "target to update".
-- For pull: remote is "source of truth", local is "target to update".
diffAndPlan :: [FileEntry] -> [FileEntry] -> [RcloneAction]
diffAndPlan sourceFiles targetFiles =
  let sourceIndex = buildIndexFromFileEntries sourceFiles
      targetIndex = buildIndexFromFileEntries targetFiles
      diffs = computeDiff sourceIndex targetIndex
  in  resolveSwaps (map planAction diffs)

-- | Push pipeline: compute actions to make remote match local.
-- Takes pre-scanned local and remote file lists, returns planned actions.
pushSyncFiles :: [FileEntry] -> [FileEntry] -> [RcloneAction]
pushSyncFiles localFiles remoteFiles =
  diffAndPlan localFiles (filterOutBitPaths remoteFiles)

-- | Pull pipeline: compute actions to make local match remote.
-- Note the reversed argument order: remote is source of truth, local is target.
pullSyncFiles :: [FileEntry] -> [FileEntry] -> [RcloneAction]
pullSyncFiles localFiles remoteFiles =
  diffAndPlan (filterOutBitPaths remoteFiles) localFiles
```

---

## Bit/Plan.hs

**Path:** `Bit/Plan.hs`

*Source file.*

```haskell
{-# LANGUAGE DuplicateRecordFields #-}
{-# LANGUAGE OverloadedRecordDot #-}

module Bit.Plan
  ( RcloneAction(..)
  , planAction
  , resolveSwaps
  ) where

import Bit.Diff (GitDiff(..), LightFileEntry(..))
import Bit.Types
import Data.List (foldl')
import qualified Data.Map.Strict as Map

-- Specific instructions to be executed via rclone
data RcloneAction
    = Move Path Path
    | Copy Path Path
    | Delete Path
    | Swap Path Path Path  -- Move via a temporary file (TempPath, Source, Dest)
    deriving (Eq, Show)

-- | The Planner: Translates abstract Diffs into concrete Rclone actions
planAction :: GitDiff -> RcloneAction
planAction (Modified f)      = Copy f.filePath f.filePath -- Upload over existing
planAction (Renamed old new) = Move old.filePath new.filePath
planAction (Added f)         = Copy f.filePath f.filePath
planAction (Deleted f)       = Delete f.filePath

-- | Detect mirrored Move pairs (A→B and B→A) and replace each pair with a
-- single Swap action that uses a temporary path to avoid overwriting.
-- Non-paired actions pass through unchanged. Only handles pairwise swaps;
-- longer cycles (A→B→C→A) are left as individual Moves (known limitation).
resolveSwaps :: [RcloneAction] -> [RcloneAction]
resolveSwaps actions =
    let moves = [(src, dest) | Move src dest <- actions]
        moveMap = Map.fromList moves
        -- A swap pair exists when Move A B and Move B A both appear
        swapPairs = [ (a, b)
                    | (a, b) <- moves
                    , Map.lookup b moveMap == Just a
                    , a < b  -- canonical ordering to avoid emitting the pair twice
                    ]
        swappedPaths = foldl' (\acc (a, b) -> Map.insert a () (Map.insert b () acc))
                              Map.empty swapPairs
        isSwapped action = case action of
            Move src dest -> Map.member src swappedPaths && Map.member dest swappedPaths
            _             -> False
        kept = filter (not . isSwapped) actions
        newSwaps = [ Swap (Path (unPath a <> ".bit-swap-tmp")) a b | (a, b) <- swapPairs ]
    in  kept ++ newSwaps
```

---

## Bit/Process.hs

**Path:** `Bit/Process.hs`

*Source file.*

```haskell
{-# LANGUAGE NoImplicitPrelude #-}
{-# LANGUAGE ScopedTypeVariables #-}

-- | Strict process output capture utilities.
--
-- This module provides safe alternatives to using lazy IO with processes.
-- All functions strictly read process output before returning, preventing
-- "delayed read on closed handle" errors.
--
-- This module intentionally does NOT expose 'System.IO.hGetContents'.
-- Use 'readProcessStrict' or 'readProcessStrictWithStderr' instead.
--
-- All operations:
-- * Use strict 'ByteString' to ensure handles are fully read before closing
-- * Read stdout and stderr concurrently to avoid deadlocks
-- * Properly clean up handles and wait for process termination
-- * Work correctly on Windows (no handle leaks or timing issues)
--
-- Import this module for process operations that need output capture:
--
-- @
-- import Bit.Process (readProcessStrict, readProcessStrictWithStderr)
-- @
module Bit.Process
  ( -- * Strict process execution
    readProcessStrict
  , readProcessStrictWithStderr
  
    -- * Re-exports (safe operations only)
  , BS.ByteString
  , ExitCode(..)
  ) where

import Prelude (FilePath, IO, String, Maybe(..), Either(..), ($), pure, error)
import Data.Foldable (traverse_)
import Control.Concurrent.Async (async, wait)
import Control.Exception (bracket, try, SomeException)
import Control.Monad (void)
import System.Exit (ExitCode(..))
import System.IO (hClose)
import System.Process
  ( CreateProcess(..)
  , StdStream(..)
  , proc
  , createProcess
  , waitForProcess
  )
import qualified Data.ByteString as BS

-- ============================================================================
-- Strict process execution
-- ============================================================================

-- | Run a process and strictly capture its stdout and stderr.
-- Returns (exitCode, stdout, stderr).
--
-- Both stdout and stderr are read concurrently to avoid deadlocks when
-- the process fills either buffer. All handles are properly closed before
-- returning, even if an exception occurs.
--
-- This is the preferred way to capture process output. Unlike using
-- 'createProcess' with 'System.IO.hGetContents', this function:
--
-- * Reads all output strictly (no lazy IO)
-- * Reads stdout and stderr concurrently (no deadlock)
-- * Closes all handles before 'waitForProcess'
-- * Works correctly on Windows
--
-- Example:
--
-- @
-- (exitCode, out, err) <- readProcessStrict "git" ["status"]
-- @
readProcessStrict :: FilePath -> [String] -> IO (ExitCode, BS.ByteString, BS.ByteString)
readProcessStrict cmd args = do
  let cp = (proc cmd args)
        { std_out = CreatePipe
        , std_err = CreatePipe
        , std_in  = CreatePipe
        }
  
  bracket (createProcess cp) cleanupProcess $ \(mStdin, mStdout, _mStderr, ph) -> do
    case (mStdin, mStdout, _mStderr) of
      (Just hIn, Just hOut, Just hErr) -> do
        -- Close stdin immediately (we don't send input)
        hClose hIn
        
        -- Read stdout and stderr concurrently to avoid deadlocks
        -- Both reads are strict and will fully drain the handles
        asyncOut <- async (BS.hGetContents hOut)
        asyncErr <- async (BS.hGetContents hErr)
        
        -- Wait for both reads to complete
        -- hGetContents will close the handles when done
        out <- wait asyncOut
        err <- wait asyncErr
        
        -- Now it's safe to wait for the process
        exitCode <- waitForProcess ph
        
        pure (exitCode, out, err)
      
      _ -> error "Bit.Process.readProcessStrict: failed to create pipes"
  where
    cleanupProcess (mStdin, mStdout, _mStderr, ph) = do
      -- Try to close any handles that are still open
      -- (BS.hGetContents closes the handle, but we need cleanup for stdin)
      let tryClose h = void (try (hClose h) :: IO (Either SomeException ()))
      traverse_ tryClose mStdin
      traverse_ tryClose mStdout
      traverse_ tryClose _mStderr
      -- Ensure process is cleaned up
      void (try (waitForProcess ph) :: IO (Either SomeException ExitCode))

-- | Run a process, strictly capture stdout, and inherit stderr to the terminal.
-- Returns (exitCode, stdout).
--
-- This is useful when you want to capture the command's output but still see
-- progress or error messages in real-time on the terminal (e.g., git clone,
-- rclone copy with --progress).
--
-- The stdout is read strictly before waiting for the process to complete,
-- ensuring no "delayed read on closed handle" errors.
--
-- Example:
--
-- @
-- -- Capture git output while showing progress on terminal
-- (exitCode, out) <- readProcessStrictWithStderr "git" ["clone", "--progress", url]
-- @
readProcessStrictWithStderr :: FilePath -> [String] -> IO (ExitCode, BS.ByteString)
readProcessStrictWithStderr cmd args = do
  let cp = (proc cmd args)
        { std_out = CreatePipe
        , std_err = Inherit  -- stderr goes to terminal
        , std_in  = CreatePipe
        }
  
  bracket (createProcess cp) cleanupProcess $ \(mStdin, mStdout, _mStderr, ph) -> do
    case (mStdin, mStdout) of
      (Just hIn, Just hOut) -> do
        -- Close stdin immediately (we don't send input)
        hClose hIn
        
        -- Read stdout strictly
        -- We don't need async here since stderr is inherited (not a pipe)
        out <- BS.hGetContents hOut
        -- hGetContents closes the handle
        
        -- Now it's safe to wait for the process
        exitCode <- waitForProcess ph
        
        pure (exitCode, out)
      
      _ -> error "Bit.Process.readProcessStrictWithStderr: failed to create pipes"
  where
    cleanupProcess (mStdin, mStdout, _mStderr, ph) = do
      -- Try to close any handles that are still open
      let tryClose h = void (try (hClose h) :: IO (Either SomeException ()))
      traverse_ tryClose mStdin
      traverse_ tryClose mStdout
      -- Ensure process is cleaned up
      void (try (waitForProcess ph) :: IO (Either SomeException ExitCode))
```

---

## Bit/Progress.hs

**Path:** `Bit/Progress.hs`

*Source file.*

```haskell
{-# LANGUAGE BangPatterns #-}

-- | Centralized progress reporting for terminal operations.
-- Eliminates duplicated progress loops and ensures consistent output behavior.
module Bit.Progress
  ( reportProgress
  , clearProgress
  , withProgressReporter
  , simpleProgressLoop
  ) where

import System.IO (hPutStr, hFlush, hIsTerminalDevice, stderr)
import Data.IORef (IORef, newIORef, readIORef)
import Control.Concurrent (forkIO, threadDelay, killThread)
import Control.Exception (finally)
import Control.Monad (when)

-- | Write a progress line to stderr. Overwrites the current line using carriage return.
-- This is the single correct way to update a progress indicator.
reportProgress :: String -> IO ()
reportProgress msg = do
  hPutStr stderr ("\r\ESC[K" ++ msg)
  hFlush stderr

-- | Clear the current progress line. Used in cleanup.
clearProgress :: IO ()
clearProgress = do
  hPutStr stderr "\r\ESC[K"
  hFlush stderr

-- | Bracket pattern for progress reporting. Handles TTY detection, thread management, and cleanup.
-- Parameters:
--   threshold: minimum count to show progress (below this, no reporter thread is spawned)
--   total: total item count
--   action: action that receives the counter IORef
-- Returns: the result of the action
withProgressReporter :: Int -> Int -> (IORef Int -> IO a) -> IO a
withProgressReporter threshold total action = do
  isTTY <- hIsTerminalDevice stderr
  counter <- newIORef (0 :: Int)
  
  let shouldShowProgress = isTTY && total > threshold
  
  if shouldShowProgress
    then do
      -- Fork reporter thread, run action with cleanup
      reporterThread <- forkIO (simpleProgressLoop "Processing..." counter total 100000)
      finally
        (action counter)
        (do
          killThread reporterThread
          clearProgress
        )
    else
      -- No progress reporting, just run the action
      action counter

-- | Generic progress loop that formats and reports progress.
-- Parameters:
--   label: prefix for the progress message (e.g., "Scanning...", "Checking files:")
--   counter: IORef tracking current progress
--   total: total item count
--   interval: update interval in microseconds
simpleProgressLoop :: String -> IORef Int -> Int -> Int -> IO ()
simpleProgressLoop label counter total interval = go
  where
    go = do
      n <- readIORef counter
      let pct = (n * 100) `div` max 1 total
      reportProgress $ label ++ " " ++ show n ++ "/" ++ show total ++ " (" ++ show pct ++ "%)"
      threadDelay interval
      when (n < total) go
```

---

## Bit/Remote.hs

**Path:** `Bit/Remote.hs`

*Source file.*

```haskell
module Bit.Remote
  ( Remote          -- export type but NOT constructor/fields
  , mkRemote        -- smart constructor
  , remoteName      -- only the name is public
  , remoteUrl       -- for Transport only (should import from Internal)
  , displayRemote   -- for user-facing messages
  , resolveRemote
  , getDefaultRemote
  , RemoteState(..) -- remote state classification
  , FetchResult(..) -- bundle fetch result
  ) where

import qualified Internal.Git as Git
import qualified Bit.Device as Device
import Data.List (isSuffixOf)

-- | A resolved remote. Bit.hs works with this; only Transport sees the url.
data Remote = Remote
  { _remoteName :: String    -- "origin", "backup", "nas", etc.
  , _remoteUrl  :: String    -- Resolved URL/path for Transport (e.g. "gdrive:Projects/foo", "/mnt/usb/backup")
  } deriving (Show, Eq)

-- | Get the remote name
remoteName :: Remote -> String
remoteName = _remoteName

-- | Get the remote URL (for Transport only)
remoteUrl :: Remote -> String
remoteUrl = _remoteUrl

-- | For user-facing display only. Never use this to construct paths.
displayRemote :: Remote -> String
displayRemote r = _remoteName r ++ " (" ++ _remoteUrl r ++ ")"

-- | Smart constructor
mkRemote :: String -> String -> Remote
mkRemote = Remote

-- | Resolve a remote name to a Remote. Dispatches on RemoteType:
--   Filesystem: reads URL from git config, strips .bit/index suffix
--   Device:     resolves device UUID to mount path
--   Cloud:      reads target from remote file (rclone URL)
--   Nothing:    backward compat fallback
resolveRemote :: FilePath -> String -> IO (Maybe Remote)
resolveRemote cwd name = do
    mType <- Device.readRemoteType cwd name
    case mType of
        Just Device.RemoteFilesystem -> do
            result <- resolveFromGitConfig name
            case result of
                Just _  -> pure result
                Nothing -> resolveOldFormat cwd name  -- fallback for pre-git-remote files
        Just Device.RemoteDevice     -> resolveDeviceRemote cwd name
        Just Device.RemoteCloud      -> resolveCloudRemote cwd name
        Nothing                      -> resolveOldFormat cwd name

-- | Filesystem remote: URL is in git config, strip .bit/index suffix to get base path.
resolveFromGitConfig :: String -> IO (Maybe Remote)
resolveFromGitConfig name = do
    mUrl <- Git.getRemoteUrl name
    case mUrl of
        Just url | not (null url) -> pure (Just (mkRemote name (stripBitIndexSuffix url)))
        _ -> pure Nothing

-- | Device remote: read target from file, resolve device UUID.
resolveDeviceRemote :: FilePath -> String -> IO (Maybe Remote)
resolveDeviceRemote cwd name = do
    mTarget <- Device.readRemoteFile cwd name
    case mTarget of
        Just target -> do
            res <- Device.resolveRemoteTarget cwd target
            case res of
                Device.Resolved url -> pure (Just (mkRemote name url))
                Device.NotConnected _ -> pure Nothing
        Nothing -> pure Nothing

-- | Cloud remote: read target from file (rclone URL).
resolveCloudRemote :: FilePath -> String -> IO (Maybe Remote)
resolveCloudRemote cwd name = do
    mTarget <- Device.readRemoteFile cwd name
    case mTarget of
        Just target -> do
            res <- Device.resolveRemoteTarget cwd target
            case res of
                Device.Resolved url -> pure (Just (mkRemote name url))
                Device.NotConnected _ -> pure Nothing
        Nothing -> pure Nothing

-- | Backward compat: try remote file, then git config.
resolveOldFormat :: FilePath -> String -> IO (Maybe Remote)
resolveOldFormat cwd name = do
    mTarget <- Device.readRemoteFile cwd name
    case mTarget of
        Just target -> do
            res <- Device.resolveRemoteTarget cwd target
            case res of
                Device.Resolved url -> pure (Just (mkRemote name url))
                Device.NotConnected _ -> pure Nothing
        Nothing -> do
            mUrl <- Git.getRemoteUrl name
            case mUrl of
                Just url | not (null url) -> pure (Just (mkRemote name (stripBitIndexSuffix url)))
                _ -> pure Nothing

-- | Strip /.bit/index or \.bit\index suffix from a git remote URL to get the base path.
stripBitIndexSuffix :: String -> String
stripBitIndexSuffix url =
    let normalized = map (\c -> if c == '\\' then '/' else c) url
        suffix = "/.bit/index"
    in if suffix `isSuffixOf` normalized
       then take (length url - length suffix) url
       else url

-- | Get the default remote for push/pull/fetch.
-- Checks branch tracking config, falls back to "origin".
getDefaultRemote :: FilePath -> IO (Maybe Remote)
getDefaultRemote cwd = do
    name <- Git.getTrackedRemoteName  -- defaults to "origin" if not configured
    resolveRemote cwd name

-- | Classification of remote state (used by Bit.hs to determine what action to take)
data RemoteState 
    = StateEmpty                        -- Case A: No files at all
    | StateValidRgit                    -- Case B: .rgit/ exists and looks okay
    | StateNonRgitOccupied [String]     -- Case C: Files exist, but no .rgit/ (stores sample filenames)
    | StateCorruptedRgit String         -- Case D: .rgit/ exists but something is wrong
    | StateNetworkError String          -- Network/Auth failure
    deriving (Show, Eq)

-- | Result of attempting to fetch a bundle from remote
data FetchResult 
    = BundleFound FilePath 
    | RemoteEmpty 
    | NetworkError String
    deriving (Show, Eq)
```

---

## Bit/RemoteWorkspace.hs

**Path:** `Bit/RemoteWorkspace.hs`

*Source file.*

```haskell
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE NamedFieldPuns #-}
{-# LANGUAGE ScopedTypeVariables #-}

module Bit.RemoteWorkspace
  ( initRemote
  , addRemote
  , commitRemote
  , statusRemote
  , logRemote
  , lsFilesRemote
  ) where

import Bit.Types (FileEntry(..), EntryKind(..), ContentType(..), Path(..))
import Bit.Remote (Remote)
import qualified Bit.Remote.Scan as Remote.Scan
import Bit.Scan (hashAndClassifyFile, binaryExtensions)
import qualified Internal.ConfigFile as ConfigFile
import Internal.ConfigFile (TextConfig)
import qualified Internal.Transport as Transport
import Bit.Utils (isBitPath, atomicWriteFileStr, trimGitOutput)
import Bit.Internal.Metadata (MetaContent(..), serializeMetadata)
import System.FilePath ((</>), takeExtension, takeDirectory)
import System.Directory
    ( createDirectoryIfMissing
    , doesDirectoryExist
    , getTemporaryDirectory
    , listDirectory
    , removeDirectoryRecursive
    , removeFile
    )
import System.Exit (ExitCode(..), exitWith)
import System.IO (hPutStrLn, stderr, stdout, hFlush)
import qualified Internal.Git as Git
import Control.Monad (when, unless, forM, forM_, void)
import Data.Char (toLower)
import Data.List (partition)
import Control.Exception (SomeException, bracket, catch)

----------------------------------------------------------------------
-- Temp directory management
----------------------------------------------------------------------

-- | Create a temp directory and run an action in it.
-- Cleans up any leftover from a previous run, then cleans up on exit.
-- Exception-safe via bracket.
withTempDir :: String -> (FilePath -> IO a) -> IO a
withTempDir name action = bracket setup cleanup action
  where
    setup = do
        sysTemp <- getTemporaryDirectory
        let dir = sysTemp </> name
        -- Remove leftover from a previous run (e.g. if cleanup failed)
        removeDirectoryRecursive dir `catch` \(_ :: SomeException) -> pure ()
        createDirectoryIfMissing True dir
        pure dir
    cleanup dir =
        removeDirectoryRecursive dir `catch` \(_ :: SomeException) -> pure ()

----------------------------------------------------------------------
-- Git helpers
----------------------------------------------------------------------

-- | Run a git command at a given path; exit with error on failure.
runOrDie :: FilePath -> [String] -> String -> IO ()
runOrDie dir args desc = do
    (code, _, err) <- Git.runGitAt dir args
    when (code /= ExitSuccess) $ do
        hPutStrLn stderr $ "fatal: failed to " ++ desc ++ "."
        unless (null err) $ hPutStrLn stderr err
        exitWith (ExitFailure 1)

----------------------------------------------------------------------
-- Bundle operations
----------------------------------------------------------------------

-- | Inflate a git bundle into a workspace directory.
-- Uses init + fetch into tracking refs + checkout to avoid:
--   1. "refusing to fetch into checked out branch" (init+fetch into heads)
--   2. "directory already exists" on Windows (git clone)
inflateBundle :: FilePath -> FilePath -> IO ()
inflateBundle bundlePath wsPath = do
    createDirectoryIfMissing True wsPath
    runOrDie wsPath ["init", "--initial-branch=main"] "initialize workspace"
    void $ Git.runGitAt wsPath ["config", "core.quotePath", "false"]
    -- Fetch into remote tracking refs (not refs/heads/*) to avoid conflict
    -- with the checked-out main branch
    runOrDie wsPath ["fetch", bundlePath, "+refs/heads/*:refs/remotes/bundle/*"] "fetch from bundle"
    -- Use reset --hard to update the branch pointer, index, AND working tree.
    -- (checkout -B can skip the working tree update when already on the same branch)
    runOrDie wsPath ["reset", "--hard", "refs/remotes/bundle/main"] "checkout main branch"

-- | Create a bundle from workspace and push to remote.
bundleAndPush :: FilePath -> Remote -> FilePath -> IO ()
bundleAndPush wsPath remote tmpBase = do
    let newBundle = tmpBase </> "new.bundle"
    putStrLn "Creating metadata bundle..."
    (bCode, _, _) <- Git.runGitAt wsPath ["bundle", "create", newBundle, "--all"]
    when (bCode /= ExitSuccess) $ do
        hPutStrLn stderr "fatal: failed to create bundle."
        exitWith (ExitFailure 1)
    putStrLn "Pushing bundle to remote..."
    pCode <- Transport.copyToRemote newBundle remote ".bit/bit.bundle"
    when (pCode /= ExitSuccess) $ do
        hPutStrLn stderr "fatal: failed to push bundle to remote."
        exitWith (ExitFailure 1)
    putStrLn "Changes pushed to remote."

----------------------------------------------------------------------
-- Ephemeral workspace patterns
----------------------------------------------------------------------

-- | Run an action in an ephemeral workspace inflated from the remote's bundle.
-- After the action succeeds and HEAD has changed, re-bundles and pushes.
-- Workspace is cleaned up on exit (exception-safe).
withRemoteWorkspace :: Remote -> (FilePath -> IO ExitCode) -> IO ExitCode
withRemoteWorkspace remote action = withTempDir "bit-remote-ws" $ \tmpBase -> do
    let bundlePath = tmpBase </> "bit.bundle"
        wsPath = tmpBase </> "workspace"
    result <- Transport.copyFromRemoteDetailed remote ".bit/bit.bundle" bundlePath
    case result of
        Transport.CopyNotFound -> do
            hPutStrLn stderr "fatal: no bit repository on remote. Run 'bit @remote init' first."
            pure (ExitFailure 1)
        Transport.CopyNetworkError err -> do
            hPutStrLn stderr $ "fatal: network error: " ++ err
            pure (ExitFailure 1)
        Transport.CopyOtherError err -> do
            hPutStrLn stderr $ "fatal: " ++ err
            pure (ExitFailure 1)
        Transport.CopySuccess -> do
            inflateBundle bundlePath wsPath
            -- Capture HEAD before action
            (_, oldHeadRaw, _) <- Git.runGitAt wsPath ["rev-parse", "HEAD"]
            let oldHead = trimGitOutput oldHeadRaw
            -- Run action
            code <- action wsPath
            case code of
                ExitSuccess -> do
                    (_, newHeadRaw, _) <- Git.runGitAt wsPath ["rev-parse", "HEAD"]
                    let newHead = trimGitOutput newHeadRaw
                    if oldHead == newHead
                        then pure ExitSuccess
                        else do
                            bundleAndPush wsPath remote tmpBase
                            pure ExitSuccess
                _ -> pure code

-- | Read-only: fetches and inflates but never pushes back.
-- Workspace is cleaned up on exit (exception-safe).
withRemoteWorkspaceReadOnly :: Remote -> (FilePath -> IO ExitCode) -> IO ExitCode
withRemoteWorkspaceReadOnly remote action = withTempDir "bit-remote-ro" $ \tmpBase -> do
    let bundlePath = tmpBase </> "bit.bundle"
        wsPath = tmpBase </> "workspace"
    result <- Transport.copyFromRemoteDetailed remote ".bit/bit.bundle" bundlePath
    case result of
        Transport.CopyNotFound -> do
            hPutStrLn stderr "fatal: no bit repository on remote. Run 'bit @remote init' first."
            pure (ExitFailure 1)
        Transport.CopyNetworkError err -> do
            hPutStrLn stderr $ "fatal: network error: " ++ err
            pure (ExitFailure 1)
        Transport.CopyOtherError err -> do
            hPutStrLn stderr $ "fatal: " ++ err
            pure (ExitFailure 1)
        Transport.CopySuccess -> do
            inflateBundle bundlePath wsPath
            action wsPath

----------------------------------------------------------------------
-- File classification (preserved from old implementation)
----------------------------------------------------------------------

-- | Partition remote files into definitely-binary and text-candidates.
-- A file is definitely binary if:
--   1. Size >= textSizeLimit, OR
--   2. Extension is in binaryExtensions
-- Everything else is a text candidate (needs download + classification).
partitionFiles :: TextConfig -> [FileEntry] -> ([FileEntry], [FileEntry])
partitionFiles config = partition isBinary
  where
    isBinary fe = case kind fe of
        File{fSize} ->
            fSize >= ConfigFile.textSizeLimit config
            || map toLower (takeExtension (unPath (path fe))) `elem` binaryExtensions
        _ -> True

-- | Download text candidate files from remote, classify them.
classifyTextCandidates :: Remote -> TextConfig -> [FileEntry] -> IO [FileEntry]
classifyTextCandidates remote config candidates = do
    tempDir <- getTemporaryDirectory >>= \t -> do
        let d = t </> "bit-classify-remote"
        createDirectoryIfMissing True d
        pure d
    classifiedEntries <- forM candidates $ \fe -> do
        let remotePath = path fe
        let localPath = tempDir </> unPath (path fe)
        createDirectoryIfMissing True (takeDirectory localPath)
        code <- Transport.copyFromRemote remote (unPath remotePath) localPath
        case code of
            ExitSuccess ->
                case kind fe of
                    File{fSize} -> do
                        (h, contentType) <- hashAndClassifyFile localPath fSize config
                        pure fe { kind = File { fHash = h, fSize = fSize, fContentType = contentType } }
                    _ -> pure fe
            _ -> do
                hPutStrLn stderr $ "Warning: Could not download " ++ unPath (path fe) ++ " for classification, treating as binary."
                pure fe
    removeDirectoryRecursive tempDir `catch` (\(_ :: SomeException) -> pure ())
    pure classifiedEntries

----------------------------------------------------------------------
-- Workspace operations
----------------------------------------------------------------------

-- | Remove all files from workspace (keeping .git directory).
-- This ensures the workspace reflects the current remote state exactly,
-- so git can detect additions, modifications, AND deletions correctly.
clearWorkspace :: FilePath -> IO ()
clearWorkspace wsPath = do
    contents <- listDirectory wsPath
    forM_ contents $ \item ->
        when (item /= ".git") $ do
            let itemPath = wsPath </> item
            isDir <- doesDirectoryExist itemPath
            if isDir
                then removeDirectoryRecursive itemPath
                else removeFile itemPath

-- | Write metadata files into workspace for all classified remote files.
-- Text files: download actual content from remote.
-- Binary files: write hash+size metadata.
writeFilesToWorkspace :: FilePath -> Remote -> [FileEntry] -> IO ()
writeFilesToWorkspace wsPath remote allFiles = do
    let (textFiles, binaryFiles) = partition isTextFile allFiles
    -- Download text file content from remote
    unless (null textFiles) $ do
        putStrLn $ "Downloading " ++ show (length textFiles) ++ " text files..."
        forM_ textFiles $ \fe -> do
            let localPath = wsPath </> unPath (path fe)
            createDirectoryIfMissing True (takeDirectory localPath)
            code <- Transport.copyFromRemote remote (unPath (path fe)) localPath
            when (code /= ExitSuccess) $
                hPutStrLn stderr $ "Warning: Failed to download text file " ++ unPath (path fe)
    -- Write hash+size metadata for binary files
    forM_ binaryFiles $ \fe -> do
        let metaPath = wsPath </> unPath (path fe)
        createDirectoryIfMissing True (takeDirectory metaPath)
        case kind fe of
            File{fHash, fSize} ->
                atomicWriteFileStr metaPath (serializeMetadata (MetaContent fHash fSize))
            _ -> pure ()
  where
    isTextFile fe = case kind fe of
        File{fContentType = TextContent} -> True
        _ -> False

-- | Scan the remote, classify files, and write metadata into the workspace.
-- Returns True on success, False on failure (errors printed to stderr).
scanAndWriteMetadata :: Remote -> FilePath -> IO Bool
scanAndWriteMetadata remote wsPath = do
    putStrLn "Scanning remote..."
    scanResult <- Remote.Scan.fetchRemoteFiles remote
    case scanResult of
        Left err -> do
            hPutStrLn stderr $ "fatal: error scanning remote: " ++ show err
            pure False
        Right remoteFiles -> do
            let files = filter (not . isBitPath . unPath . path) remoteFiles
            putStrLn $ "Found " ++ show (length files) ++ " files on remote."
            config <- ConfigFile.readTextConfig
            let (binary, textCands) = partitionFiles config files
            putStrLn $ "  " ++ show (length binary) ++ " binary files (by size/extension)"
            putStrLn $ "  " ++ show (length textCands) ++ " small files to classify..."
            classifiedFiles <- if null textCands
                then pure []
                else classifyTextCandidates remote config textCands
            let allFiles = binary ++ classifiedFiles
            clearWorkspace wsPath
            writeFilesToWorkspace wsPath remote allFiles
            pure True

-- | Stage and auto-commit changes in the workspace.
stageAndCommit :: FilePath -> [String] -> IO ExitCode
stageAndCommit wsPath paths = do
    let addPaths = if null paths then ["."] else paths
    (addCode, _, _) <- Git.runGitAt wsPath ("add" : addPaths)
    case addCode of
        ExitSuccess -> do
            (diffCode, _, _) <- Git.runGitAt wsPath ["diff", "--cached", "--quiet"]
            case diffCode of
                ExitFailure 1 -> do
                    (commitCode, _, _) <- Git.runGitAt wsPath ["commit", "-m", "Update remote metadata"]
                    case commitCode of
                        ExitSuccess -> do
                            putStrLn "Remote metadata updated."
                            pure ExitSuccess
                        _ -> do
                            hPutStrLn stderr "error: failed to commit metadata."
                            pure commitCode
                _ -> do
                    putStrLn "Nothing to add — remote metadata is up to date."
                    pure ExitSuccess
        _ -> do
            hPutStrLn stderr "error: failed to stage files."
            pure addCode

----------------------------------------------------------------------
-- Public API: Remote commands
----------------------------------------------------------------------

-- | Initialize a remote repository (create empty bundle and push).
-- Does NOT use withRemoteWorkspace — there's no bundle to fetch yet.
initRemote :: Remote -> String -> IO ()
initRemote remote remoteName = withTempDir "bit-remote-init" $ \tmpBase -> do
    let checkPath = tmpBase </> "check.bundle"
        wsPath = tmpBase </> "workspace"
    -- Check if already initialized
    result <- Transport.copyFromRemoteDetailed remote ".bit/bit.bundle" checkPath
    case result of
        Transport.CopySuccess -> do
            hPutStrLn stderr "fatal: remote already has a bit repository."
            exitWith (ExitFailure 1)
        Transport.CopyNetworkError err -> do
            hPutStrLn stderr $ "fatal: network error: " ++ err
            exitWith (ExitFailure 1)
        Transport.CopyOtherError err -> do
            hPutStrLn stderr $ "fatal: " ++ err
            exitWith (ExitFailure 1)
        Transport.CopyNotFound -> do
            -- Create fresh repo
            createDirectoryIfMissing True wsPath
            runOrDie wsPath ["init", "--initial-branch=main"] "initialize repository"
            void $ Git.runGitAt wsPath ["config", "core.quotePath", "false"]
            runOrDie wsPath ["commit", "--allow-empty", "-m", "Initial remote repository"] "create initial commit"
            -- Bundle and push
            let bundlePath = tmpBase </> "init.bundle"
            (bCode, _, _) <- Git.runGitAt wsPath ["bundle", "create", bundlePath, "--all"]
            when (bCode /= ExitSuccess) $ do
                hPutStrLn stderr "fatal: failed to create bundle."
                exitWith (ExitFailure 1)
            pCode <- Transport.copyToRemote bundlePath remote ".bit/bit.bundle"
            case pCode of
                ExitSuccess ->
                    putStrLn $ "Initialized bit repository on remote '" ++ remoteName ++ "'."
                _ -> do
                    hPutStrLn stderr "fatal: failed to push bundle to remote."
                    exitWith (ExitFailure 1)

-- | Add files in remote workspace (scan + stage + auto-commit).
-- Scans the full remote to reconstruct metadata, writes it into the
-- ephemeral workspace, stages the specified paths, and auto-commits.
addRemote :: Remote -> [String] -> IO ExitCode
addRemote remote paths =
    withRemoteWorkspace remote $ \wsPath -> do
        ok <- scanAndWriteMetadata remote wsPath
        if ok
            then stageAndCommit wsPath paths
            else pure (ExitFailure 1)

-- | Commit in remote workspace (interactive, passes through stdio).
-- Useful for amending the last commit message or creating additional commits.
commitRemote :: Remote -> [String] -> IO ExitCode
commitRemote remote commitArgs =
    withRemoteWorkspace remote $ \wsPath ->
        Git.runGitRawAt wsPath ("commit" : commitArgs)

-- | Show status of remote workspace (read-only).
-- Scans the remote to detect untracked files, then runs git status.
statusRemote :: Remote -> [String] -> IO ExitCode
statusRemote remote rest =
    withRemoteWorkspaceReadOnly remote $ \wsPath -> do
        -- Scan remote and write metadata so git can detect untracked files
        ok <- scanAndWriteMetadata remote wsPath
        hFlush stdout  -- Ensure scan output appears before git status
        if ok
            then do
                -- Update index to match working tree after writing files
                -- This ensures stat info matches and content-identical files show as clean
                void $ Git.runGitAt wsPath ["add", "-u"]
                void $ Git.runGitAt wsPath ["reset", "HEAD"]
                Git.runGitRawAt wsPath ("status" : rest)
            else pure (ExitFailure 1)

-- | Show log of remote workspace (read-only).
logRemote :: Remote -> [String] -> IO ExitCode
logRemote remote rest =
    withRemoteWorkspaceReadOnly remote $ \wsPath ->
        Git.runGitRawAt wsPath ("log" : "--decorate-refs=refs/heads/" : rest)

-- | List tracked files in remote workspace (read-only).
-- Scans the remote to reconstruct metadata, then runs git ls-files.
lsFilesRemote :: Remote -> [String] -> IO ExitCode
lsFilesRemote remote rest =
    withRemoteWorkspaceReadOnly remote $ \wsPath -> do
        ok <- scanAndWriteMetadata remote wsPath
        hFlush stdout
        if ok
            then do
                void $ Git.runGitAt wsPath ["add", "-u"]
                void $ Git.runGitAt wsPath ["reset", "HEAD"]
                Git.runGitRawAt wsPath ("ls-files" : rest)
            else pure (ExitFailure 1)
```

---

## Bit/Remote/Scan.hs

**Path:** `Bit/Remote/Scan.hs`

*Source file.*

```haskell
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE OverloadedRecordDot #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE DeriveGeneric #-}

module Bit.Remote.Scan
  ( fetchRemoteFiles
  , RemoteError(..)
  ) where

import GHC.Generics (Generic)
import qualified Data.Map as Map
import Data.Map (Map)
import Data.Aeson (FromJSON(..), decode, withObject, (.:), (.:?))
import System.Exit (ExitCode(..))
import System.FilePath (normalise)
import Data.Maybe
import qualified Data.Text as T
import Bit.Types
import qualified Internal.Transport as Transport
import Bit.Remote (Remote)

----------------------------------------------------------------------
-- Errors
----------------------------------------------------------------------

data RemoteError
  = RcloneFailed
  | DecodeFailed String
  deriving (Show, Eq)

----------------------------------------------------------------------
-- Public API
----------------------------------------------------------------------

fetchRemoteFiles :: Remote -> IO (Either RemoteError [FileEntry])
fetchRemoteFiles remote = do
    (code, rawBytes, _err) <- Transport.listRemoteJsonWithHash remote
    case code of
        ExitSuccess -> pure $ maybe
            (Left (DecodeFailed "Invalid rclone JSON output"))
            (Right . map rcloneFileToFileEntry . filter (not . rfIsDir))
            (decode rawBytes :: Maybe [RcloneFile])
        _ -> pure (Left RcloneFailed)

----------------------------------------------------------------------
-- Conversion
----------------------------------------------------------------------

rcloneFileToFileEntry :: RcloneFile -> FileEntry
rcloneFileToFileEntry rf =
  FileEntry
    { path = Path (normalise rf.rfPath)
    , kind = File { fHash = Hash (T.pack ("md5:" ++ md5)), fSize = rf.rfSize, fContentType = BinaryContent }
    }
    where
      md5 =
        Map.findWithDefault "" "md5" (fromMaybe Map.empty rf.rfHashes)
----------------------------------------------------------------------
-- rclone JSON model
----------------------------------------------------------------------

data RcloneFile = RcloneFile
  { rfPath   :: FilePath
  , rfSize   :: Integer
  , rfHashes :: Maybe (Map String String)
  , rfIsDir  :: Bool
  } deriving (Show, Generic)

instance FromJSON RcloneFile where
    parseJSON = withObject "RcloneFile" $ \v ->
        RcloneFile
          <$> v .:  "Path"
          <*> v .:  "Size"
          <*> v .:? "Hashes"
          <*> v .:  "IsDir"
```

---

## Bit/Scan.hs

**Path:** `Bit/Scan.hs`

*Source file.*

```haskell
{-# LANGUAGE BangPatterns #-}
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE NamedFieldPuns #-}
{-# LANGUAGE OverloadedStrings #-}

module Bit.Scan
  ( scanWorkingDir
  , writeMetadataFiles
  , readMetadataFile
  , listMetadataPaths
  , getFileHashAndSize
  , hashAndClassifyFile
  , binaryExtensions
  , FileEntry(..)
  , EntryKind(..)
  ) where

import Bit.Types (Hash(..), HashAlgo(..), FileEntry(..), EntryKind(..), ContentType(..), hashToText, Path(..))
import System.FilePath
import System.Directory
    ( doesDirectoryExist,
      doesFileExist,
      listDirectory,
      getFileSize,
      createDirectoryIfMissing,
      copyFileWithMetadata,
      getModificationTime )
import System.IO (withFile, IOMode(ReadMode), hIsEOF, hPutStr, hPutStrLn, hIsTerminalDevice, stderr)
import Data.List (dropWhileEnd, isPrefixOf, isSuffixOf)
import Data.Either (isRight)
import Data.Maybe (listToMaybe)
import qualified Data.ByteString as BS
import Control.Monad (void, when, forM_)
import Data.Foldable (traverse_)
import Data.Text.Encoding (decodeUtf8, decodeUtf8', encodeUtf8)
import Data.Char (toLower)
import qualified Internal.ConfigFile as ConfigFile
import Bit.Utils (atomicWriteFileStr, toPosix)
import Bit.Internal.Metadata (MetaContent(..), readMetadataOrComputeHash, hashFile, serializeMetadata)
import qualified Data.Set as Set
import qualified Crypto.Hash.MD5 as MD5
import Data.ByteString.Base16 (encode)
import qualified Data.Text as T
import Control.Concurrent.Async (mapConcurrently)
import Control.Concurrent (getNumCapabilities, forkIO, threadDelay, killThread)
import Control.Concurrent.QSem (newQSem, waitQSem, signalQSem)
import Control.Exception (bracket_, finally)
import Bit.Progress (reportProgress, clearProgress)
import Data.IORef (IORef, newIORef, readIORef, atomicModifyIORef')
import Data.Time.Clock.POSIX (utcTimeToPOSIXSeconds)

-- Binary file extensions that should never be treated as text (hardcoded, not configurable)
binaryExtensions :: [String]
binaryExtensions = [".mp4", ".zip", ".bin", ".exe", ".dll", ".so", ".dylib", ".jpg", ".jpeg", ".png", ".gif", ".pdf", ".gz", ".bz2", ".xz", ".tar", ".rar", ".7z", ".iso", ".img", ".dmg", ".deb", ".rpm", ".msi"]

-- | Internal: scanned path before hashing. Distinguishes dirs from files without boolean blindness.
data ScannedEntry = ScannedFile FilePath | ScannedDir FilePath
  deriving (Show, Eq)

-- | Single-pass file hash and classification. Returns (hash, contentType).
-- For large files or binary extensions: streams hash only, returns BinaryContent.
-- For others: reads first 8KB for text classification, then streams remaining chunks for hash.
hashAndClassifyFile :: FilePath -> Integer -> ConfigFile.TextConfig -> IO (Hash 'MD5, ContentType)
hashAndClassifyFile filePath size config = do
    let ext = map toLower (takeExtension filePath)
    
    -- Fast path: large files or known binary extensions - just stream hash
    if size >= ConfigFile.textSizeLimit config || ext `elem` binaryExtensions
        then do
            h <- streamHash filePath
            pure (h, BinaryContent)
        else
            -- Single-pass: read first 8KB for classification, continue streaming for hash
            withFile filePath ReadMode $ \handle -> do
                firstChunk <- BS.hGet handle 8192
                let contentType = if not (BS.elem 0 firstChunk) && isRight (decodeUtf8' firstChunk)
                                  then TextContent else BinaryContent
                
                -- Continue streaming hash from where we left off
                let loop !ctx = do
                        eof <- hIsEOF handle
                        if eof
                            then do
                                let md5hex = decodeUtf8 (encode (MD5.finalize ctx))
                                pure (Hash (T.pack "md5:" <> md5hex))
                            else do
                                chunk <- BS.hGet handle 65536
                                loop (MD5.update ctx chunk)
                
                -- Start with first chunk already included
                h <- loop (MD5.update MD5.init firstChunk)
                pure (h, contentType)
  where
    -- Stream hash for files we're not classifying
    streamHash fp = withFile fp ReadMode $ \h -> do
        let loop !ctx = do
                eof <- hIsEOF h
                if eof
                    then do
                        let md5hex = decodeUtf8 (encode (MD5.finalize ctx))
                        pure (Hash (T.pack "md5:" <> md5hex))
                    else do
                        chunk <- BS.hGet h 65536
                        loop (MD5.update ctx chunk)
        loop MD5.init

-- | Cache entry for file metadata to skip re-hashing unchanged files
data CacheEntry = CacheEntry
  { ceMtime :: Integer
  , ceSize :: Integer
  , ceHash :: Hash 'MD5
  , ceContentType :: ContentType
  } deriving (Show, Eq)

-- | Serialize cache entry to string format (isText for backward compatibility)
serializeCacheEntry :: CacheEntry -> String
serializeCacheEntry ce =
  "mtime: " ++ show (ceMtime ce) ++ "\n"
  ++ "size: " ++ show (ceSize ce) ++ "\n"
  ++ "hash: " ++ T.unpack (hashToText (ceHash ce)) ++ "\n"
  ++ "isText: " ++ show (ceContentType ce == TextContent) ++ "\n"

-- | Parse cache entry from string format
parseCacheEntry :: String -> Maybe CacheEntry
parseCacheEntry content = do
  let ls = lines content
  mtimeLine <- listToMaybe [ drop (length ("mtime: " :: String)) l
                           | l <- ls, "mtime: " `isPrefixOf` l ]
  sizeLine <- listToMaybe [ drop (length ("size: " :: String)) l
                          | l <- ls, "size: " `isPrefixOf` l ]
  hashLine <- listToMaybe [ drop (length ("hash: " :: String)) l
                          | l <- ls, "hash: " `isPrefixOf` l ]
  isTextLine <- listToMaybe [ drop (length ("isText: " :: String)) l
                            | l <- ls, "isText: " `isPrefixOf` l ]
  mtime <- readMaybeInt (trim mtimeLine)
  size <- readMaybeInt (trim sizeLine)
  let hashVal = trim hashLine
  isText <- readMaybeBool (trim isTextLine)
  let contentType = if isText then TextContent else BinaryContent
  if null hashVal
    then Nothing
    else Just CacheEntry
      { ceMtime = mtime
      , ceSize = size
      , ceHash = Hash (T.pack hashVal)
      , ceContentType = contentType
      }
  where
    trim = dropWhileEnd isSpaceChar . dropWhile isSpaceChar
    isSpaceChar c = c == ' ' || c == '\t' || c == '\r' || c == '\n'
    readMaybeInt s = case reads s of
      [(n, "")] -> Just n
      [(n, r)] | all isSpaceChar r -> Just n
      _ -> Nothing
    readMaybeBool s = case s of
      "True" -> Just True
      "False" -> Just False
      _ -> Nothing

-- | Load cache entry for a file, returns Nothing if missing or malformed
loadCacheEntry :: FilePath -> FilePath -> IO (Maybe CacheEntry)
loadCacheEntry root relPath = do
  let cachePath = root </> ".bit" </> "cache" </> relPath
  exists <- doesFileExist cachePath
  if not exists
    then pure Nothing
    else do
      -- Use strict bytestring reading to avoid lazy file handle issues on Windows
      bs <- BS.readFile cachePath
      pure $ either (const Nothing) (parseCacheEntry . T.unpack) (decodeUtf8' bs)

-- | Save cache entry for a file (non-atomic write, cache corruption is acceptable)
saveCacheEntry :: FilePath -> FilePath -> CacheEntry -> IO ()
saveCacheEntry root relPath entry = do
  let bitRoot = root </> ".bit"
  bitExists <- doesDirectoryExist bitRoot
  when bitExists $ do
    let cachePath = bitRoot </> "cache" </> relPath
    createDirectoryIfMissing True (takeDirectory cachePath)
    -- Use strict bytestring writing to ensure file handle is closed immediately
    BS.writeFile cachePath (encodeUtf8 (T.pack (serializeCacheEntry entry)))

-- | Normalize a file path for consistent comparison (forward slashes, trimmed)
normalizePath :: FilePath -> FilePath
normalizePath = toPosix . filter (/= '\r')

-- | Check if a filename matches a gitignore-style pattern.
-- Supports: *.ext (extension match), filename (exact match)
matchesPattern :: String -> FilePath -> Bool
matchesPattern pattern path =
    let filename = takeFileName path
        whitespace = ['\r', '\n', ' '] :: [Char]
        normalizedPattern = filter (`notElem` whitespace) pattern
    in if "*." `isPrefixOf` normalizedPattern
       then -- Extension pattern like *.log
            let ext = drop 1 normalizedPattern  -- Remove the *
            in ext `isSuffixOf` filename
       else -- Exact filename match
            normalizedPattern == filename

-- | Check which files should be ignored based on .bitignore patterns.
-- Reads patterns from .bit/index/.gitignore and matches against paths.
checkIgnoredFiles :: FilePath -> [FilePath] -> IO (Set.Set FilePath)
checkIgnoredFiles root paths = do
    let gitignorePath = root </> ".bit" </> "index" </> ".gitignore"
    exists <- doesFileExist gitignorePath
    if not exists
        then pure Set.empty
        else do
            -- Use strict ByteString reading to avoid lazy file handle issues on Windows
            bs <- BS.readFile gitignorePath
            let content = either (const "") T.unpack (decodeUtf8' bs)
            let whitespace = ['\r', '\n', ' '] :: [Char]
            let patterns = filter (not . null) $ 
                           filter (not . ("#" `isPrefixOf`)) $  -- Skip comments
                           map (filter (`notElem` whitespace)) (lines content)
            let isIgnored p = any (`matchesPattern` p) patterns
            pure $ Set.fromList $ filter isIgnored paths

-- | Bounded parallel map: runs up to @bound@ actions concurrently.
mapConcurrentlyBounded :: Int -> (a -> IO b) -> [a] -> IO [b]
mapConcurrentlyBounded bound f xs = do
    sem <- newQSem bound
    mapConcurrently (\x -> bracket_ (waitQSem sem) (signalQSem sem) (f x)) xs

-- | Progress reporter thread that periodically displays scan progress
progressLoop :: IORef Int -> Int -> IO ()
progressLoop counter total = go
  where
    go = do
        n <- readIORef counter
        let pct = (n * 100) `div` max 1 total
        reportProgress $ "Scanning... " ++ show n ++ "/" ++ show total ++ " files (" ++ show pct ++ "%)"
        threadDelay 50000  -- 50ms
        when (n < total) go

-- Main scan function
scanWorkingDir :: FilePath -> IO [FileEntry]
scanWorkingDir root = do
    let bitRoot = root </> ".bit"
    bitExists <- doesDirectoryExist bitRoot
    if not bitExists then pure []
    else do
      -- Read config once for all files
      config <- ConfigFile.readTextConfig
    
      -- First pass: collect all paths (without hashing)
      allPaths <- collectPaths root

      -- Filter through git check-ignore
      let filePaths = [p | ScannedFile p <- allPaths]
      ignoredSet <- checkIgnoredFiles root filePaths

      -- Separate directories from files to hash
      let dirEntries = [FileEntry { path = Path rel, kind = Directory } | ScannedDir rel <- allPaths]
          filesToHash = [(rel, root </> rel) | ScannedFile rel <- allPaths
                                             , not (Set.member (normalizePath rel) ignoredSet)]
    
      -- Setup progress tracking
      let total = length filesToHash
      counter <- newIORef (0 :: Int)
    
      -- Hash/classify files in parallel (bounded by numCapabilities * 4)
      caps <- getNumCapabilities
      let concurrency = max 4 (caps * 4)
    
      let hashWithProgress (rel, fullPath) = do
              size <- getFileSize fullPath
              mtime <- getModificationTime fullPath
              let mtimeInt = floor (utcTimeToPOSIXSeconds mtime) :: Integer
              cached <- loadCacheEntry root rel
              case cached of
                Just ce | ceSize ce == fromIntegral size && ceMtime ce == mtimeInt -> do
                  -- Cache hit: reuse hash and contentType
                  atomicModifyIORef' counter (\n -> (n + 1, ()))
                  pure $ FileEntry
                      { path = Path rel
                      , kind = File { fHash = ceHash ce, fSize = fromIntegral size, fContentType = ceContentType ce }
                      }
                _ -> do
                  -- Cache miss: hash the file, save cache entry
                  (h, contentType) <- hashAndClassifyFile fullPath (fromIntegral size) config
                  saveCacheEntry root rel (CacheEntry mtimeInt (fromIntegral size) h contentType)
                  atomicModifyIORef' counter (\n -> (n + 1, ()))
                  pure $ FileEntry
                      { path = Path rel
                      , kind = File { fHash = h, fSize = fromIntegral size, fContentType = contentType }
                      }
    
      -- Wrap hashing with progress reporter
      let hashingAction = mapConcurrentlyBounded concurrency hashWithProgress filesToHash
    
      fileEntries <- if total > 50
          then do
              -- Show progress for large scans
              reporterThread <- forkIO (progressLoop counter total)
              finally hashingAction $ do
                  killThread reporterThread
                  clearProgress
                  hPutStrLn stderr $ "Scanned " ++ show total ++ " files."
          else
              -- No progress for small scans
              hashingAction
    
      pure $ dirEntries ++ fileEntries
  where
    collectPaths :: FilePath -> IO [ScannedEntry]
    collectPaths path = do
      isDir <- doesDirectoryExist path
      let rel = makeRelative root path

      -- ignore .bit folder, .git, .bitignore, and .gitignore (the latter two are config files)
      if rel == ".bit" || (".bit" `isPrefixOf` rel)
          || rel == ".git" || (".git" `isPrefixOf` rel)
          || rel == ".bitignore"
          || rel == ".gitignore"
        then pure []
        else if isDir
          then do
            names <- listDirectory path
            let children = map (path </>) names
            childPaths <- concat <$> mapM collectPaths children
            pure (ScannedDir rel : childPaths)
        else pure [ScannedFile rel]

writeMetadataFiles :: FilePath -> [FileEntry] -> IO ()
writeMetadataFiles root entries = do
    let bitRoot = root </> ".bit"
    bitExists <- doesDirectoryExist bitRoot
    when bitExists $ do
      let metaRoot = bitRoot </> "index"
      createDirectoryIfMissing True metaRoot

      -- Separate directories from files
      let (dirs, files) = partitionEntries entries
    
      -- First pass: create all directories sequentially (avoid race conditions)
      forM_ dirs $ \dirPath -> do
        let fullPath = metaRoot </> dirPath
        createDirectoryIfMissing True fullPath
    
      -- Second pass: create parent directories for files
      let parentDirs = Set.fromList [takeDirectory (unPath (path e)) | e <- files]
      forM_ parentDirs $ \dirPath -> do
        let fullPath = metaRoot </> dirPath
        createDirectoryIfMissing True fullPath
    
      -- Setup progress tracking
      let total = length files
      isTTY <- hIsTerminalDevice stderr
      counter <- newIORef (0 :: Int)
      skipped <- newIORef (0 :: Int)
    
      -- Start progress reporter thread if we're in a TTY and have enough files
      let shouldShowProgress = isTTY && total > 10
      reporterThread <- if shouldShowProgress
          then Just <$> forkIO (writeProgressLoop counter skipped total)
          else pure Nothing
    
      -- Third pass: write files in parallel (bounded concurrency)
      caps <- getNumCapabilities
      let concurrency = max 4 (caps * 4)
    
      let writeWithProgress entry = do
              let metaPath = metaRoot </> unPath (path entry)
              case kind entry of
                File { fHash, fSize, fContentType } -> do
                  -- Check if file is unchanged before writing
                  needsWrite <- shouldWriteFile root metaPath entry fHash fSize fContentType
                  if needsWrite
                    then do
                      case fContentType of
                        TextContent -> do
                          -- For text files, copy the actual content directly
                          let actualPath = root </> unPath (path entry)
                          copyFileWithMetadata actualPath metaPath
                        BinaryContent -> do
                          -- For binary files, write metadata (hash + size). Spec: raw hash value; atomic write.
                          atomicWriteFileStr metaPath $
                            serializeMetadata (MetaContent fHash fSize)
                      atomicModifyIORef' counter (\n -> (n + 1, ()))
                    else do
                      atomicModifyIORef' skipped (\n -> (n + 1, ()))
                      atomicModifyIORef' counter (\n -> (n + 1, ()))
                Directory -> pure ()  -- Already handled in first pass
                Symlink _ -> pure ()  -- Symlinks handled separately
    
      finally
          (void $ mapConcurrentlyBounded concurrency writeWithProgress files)
          (do
              -- Clean up: kill reporter thread and finalize progress line
              traverse_ killThread reporterThread
              when shouldShowProgress $ do
                  n <- readIORef counter
                  s <- readIORef skipped
                  clearProgress
                  let written = n - s
                  hPutStr stderr $ "Wrote " ++ show written ++ " metadata files"
                  when (s > 0) $ hPutStr stderr $ " (skipped " ++ show s ++ " unchanged)"
                  hPutStrLn stderr "."
          )
  where
    partitionEntries :: [FileEntry] -> ([FilePath], [FileEntry])
    partitionEntries es =
      let dirs = [unPath (path e) | e <- es, case kind e of Directory -> True; _ -> False]
          files = [e | e <- es, case kind e of File{} -> True; _ -> False]
      in (dirs, files)
    
    writeProgressLoop :: IORef Int -> IORef Int -> Int -> IO ()
    writeProgressLoop counter skipped total = go
      where
        go = do
            n <- readIORef counter
            s <- readIORef skipped
            let pct = (n * 100) `div` max 1 total
                written = n - s
                msg = "Writing metadata... " ++ show written ++ "/" ++ show total ++ " files (" ++ show pct ++ "%)"
                      ++ if s > 0 then ", skipped " ++ show s else ""
            reportProgress msg
            threadDelay 50000  -- 50ms
            when (n < total) go

-- | Check if a metadata file needs to be written (returns True if write needed)
shouldWriteFile :: FilePath -> FilePath -> FileEntry -> Hash 'MD5 -> Integer -> ContentType -> IO Bool
shouldWriteFile root metaPath entry fHash fSize fContentType = do
  exists <- doesFileExist metaPath
  if not exists
    then pure True  -- File doesn't exist, must write
    else case fContentType of
      TextContent -> do
        -- For text files: compare mtime and size of source vs destination
        let sourcePath = root </> unPath (path entry)
        sourceMtime <- getModificationTime sourcePath
        sourceSize <- getFileSize sourcePath
        destMtime <- getModificationTime metaPath
        destSize <- getFileSize metaPath
        -- Write if mtime or size differs
        pure (sourceMtime /= destMtime || sourceSize /= destSize)
      BinaryContent -> do
        -- For binary files: read existing metadata and compare hash/size
        existing <- readMetadataOrComputeHash metaPath
        pure $ maybe True  -- Failed to read, must write
          (\(MetaContent existingHash existingSize) ->
            -- Write if hash or size differs
            existingHash /= fHash || existingSize /= fSize) existing

-- | Parse a metadata file (hash/size lines) or read a text file and compute hash/size.
-- Returns Nothing if file is missing or invalid.
-- Text files in .rgit/index/ contain actual content; binary files contain metadata.
readMetadataFile :: FilePath -> IO (Maybe MetaContent)
readMetadataFile = readMetadataOrComputeHash

-- | List all metadata file paths under index dir, relative to index root. Excludes .gitattributes.
listMetadataPaths :: FilePath -> IO [FilePath]
listMetadataPaths indexRoot = go indexRoot ""
  where
    go :: FilePath -> FilePath -> IO [FilePath]
    go full rel = do
      isDir <- doesDirectoryExist full
      if isDir
        then do
          names <- listDirectory full
          let skip name = name == "." || name == ".." || name == ".gitattributes" || name == ".git"
          let children = [ (full </> name, if null rel then name else rel </> name) | name <- names, not (skip name) ]
          concat <$> mapM (\(p, r) -> go p r) children
        else do
          isFile <- doesFileExist full
          pure (if isFile then [rel] else [])

-- | Get hash and size of a file. Returns Nothing if file is missing or not a regular file.
getFileHashAndSize :: FilePath -> FilePath -> IO (Maybe MetaContent)
getFileHashAndSize root relPath = do
  let full = root </> relPath
  exists <- doesFileExist full
  if not exists then pure Nothing
  else do
    h <- hashFile full
    sz <- getFileSize full
    pure (Just (MetaContent h (fromIntegral sz)))
```

---

## Bit/Types.hs

**Path:** `Bit/Types.hs`

*Source file.*

```haskell
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE DeriveGeneric #-}
{-# LANGUAGE DerivingStrategies #-}
{-# LANGUAGE GeneralizedNewtypeDeriving #-}
{-# LANGUAGE KindSignatures #-}

module Bit.Types
  ( Path(..)
  , HashAlgo(..)
  , Hash(..)
  , hashToText
  , ContentType(..)
  , EntryKind(..)
  , FileEntry(..)
  , syncHash
  , ForceMode(..)
  , BitEnv(..)
  , BitM
  , runBitM
  ) where

import Control.Monad.Trans.Reader (ReaderT, runReaderT)
import Data.String (IsString)
import Data.Text (Text)
import GHC.Generics (Generic)
import Bit.Remote (Remote)

newtype Path = Path { unPath :: FilePath }
  deriving stock (Show, Generic)
  deriving newtype (Eq, Ord, IsString)

data HashAlgo = MD5 | SHA256
  deriving (Show, Eq, Generic)

newtype Hash (a :: HashAlgo) = Hash Text
  deriving (Show, Eq, Ord, Generic)

hashToText :: Hash a -> Text
hashToText (Hash t) = t

data ContentType = TextContent | BinaryContent
  deriving (Show, Eq, Generic)

data EntryKind
  = File { fHash :: Hash 'MD5, fSize :: Integer, fContentType :: ContentType }
  | Directory
  | Symlink FilePath
  deriving (Show, Eq, Generic)

-- | Hash to use for sync diff (MD5). File has one hash; Directory/Symlink have none.
syncHash :: EntryKind -> Maybe (Hash 'MD5)
syncHash (File h _ _) = Just h
syncHash _            = Nothing

data FileEntry = FileEntry
  { path :: Path
  , kind :: EntryKind
  } deriving (Show, Eq, Generic)

-- | How to handle force pushing.
data ForceMode
    = NoForce         -- ^ Normal push (check ancestry)
    | Force           -- ^ Overwrite remote unconditionally
    | ForceWithLease  -- ^ Overwrite only if remote matches last fetch
    deriving (Show, Eq)

data BitEnv = BitEnv
    { envCwd            :: FilePath
    , envLocalFiles     :: [FileEntry]
    , envRemote         :: Maybe Remote
    , envForceMode      :: ForceMode
    }

type BitM = ReaderT BitEnv IO

runBitM :: BitEnv -> BitM a -> IO a
runBitM env action = runReaderT action env
```

---

## Bit/Utils.hs

**Path:** `Bit/Utils.hs`

*Source file.*

```haskell
{-# LANGUAGE OverloadedRecordDot #-}
{-# LANGUAGE ScopedTypeVariables #-}

-- | Utility functions for bit.
--
-- This module provides path utilities and re-exports atomic file operations
-- from 'Bit.AtomicWrite'.
module Bit.Utils
  ( -- * Path utilities
    toPosix
  , isBitPath
  , filterOutBitPaths
  
    -- * Formatting utilities
  , formatBytes
  , trimGitOutput
  
    -- * Atomic file writes (re-exported from Bit.AtomicWrite)
  , atomicWriteFile
  , atomicWriteFileStr
  , atomicWriteFileWithLock
  
    -- * Directory locking (re-exported from Bit.AtomicWrite)
  , DirWriteLock
  , newDirWriteLock
  , withDirWriteLock
  
    -- * Lock registry (re-exported from Bit.AtomicWrite)
  , LockRegistry
  , newLockRegistry
  , withLockedDir
  ) where

import Data.List (isPrefixOf, isInfixOf)
import Bit.Types (FileEntry(..), Path(unPath))
import Bit.AtomicWrite
    ( atomicWriteFile
    , atomicWriteFileStr
    , atomicWriteFileWithLock
    , DirWriteLock
    , newDirWriteLock
    , withDirWriteLock
    , LockRegistry
    , newLockRegistry
    , withLockedDir
    )

-- | Convert Windows backslashes to forward slashes (e.g. for rclone paths).
toPosix :: FilePath -> FilePath
toPosix = map (\c -> if c == '\\' then '/' else c)

-- | True if the path is or is under .bit (bit metadata, not user content).
isBitPath :: FilePath -> Bool
isBitPath p = p == ".bit" || ".bit" `isPrefixOf` p || "/.bit" `isInfixOf` p || "\\.bit" `isInfixOf` p

-- | Remove .bit paths from a list of file entries (e.g. remote file list).
filterOutBitPaths :: [FileEntry] -> [FileEntry]
filterOutBitPaths = filter (\e -> not (isBitPath (unPath e.path)))

-- | Format bytes in human-readable form (B, KB, MB, GB, TB).
-- Uses 1 decimal place for KB and above, 1024 base.
formatBytes :: Integer -> String
formatBytes bytes
    | bytes < 1024                  = show bytes ++ " B"
    | bytes < 1024 * 1024           = formatWith (fromIntegral bytes / 1024) ++ " KB"
    | bytes < 1024 * 1024 * 1024    = formatWith (fromIntegral bytes / (1024 * 1024)) ++ " MB"
    | bytes < 1024 * 1024 * 1024 * 1024 = formatWith (fromIntegral bytes / (1024 * 1024 * 1024)) ++ " GB"
    | otherwise                     = formatWith (fromIntegral bytes / (1024 * 1024 * 1024 * 1024)) ++ " TB"
  where
    formatWith :: Double -> String
    formatWith n = showFixed 1 n
    
    -- Show a double with exactly 1 decimal place
    showFixed :: Int -> Double -> String
    showFixed decimals n =
        let multiplier = 10 ^ decimals
            rounded = fromIntegral (round (n * multiplier) :: Integer) / multiplier
        in show rounded

-- | Trim whitespace from git output (newlines, spaces, tabs, carriage returns).
-- Git commands often return commit hashes or refs with trailing newlines.
trimGitOutput :: String -> String
trimGitOutput = filter (\c -> c /= ' ' && c /= '\n' && c /= '\r' && c /= '\t')
```

---

## Bit/Verify.hs

**Path:** `Bit/Verify.hs`

*Source file.*

```haskell
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE OverloadedRecordDot #-}
{-# LANGUAGE LambdaCase #-}

module Bit.Verify
  ( verifyLocal
  , verifyLocalAt
  , verifyRemote
  , VerifyIssue(..)
  , VerifyResult(..)
  , BinaryFileMeta(..)
  , loadBinaryMetadata
  , loadCommittedBinaryMetadata
  , loadMetadataFromBundle
  , MetadataEntry(..)
  , MetadataSource(..)
  , loadMetadata
  , entryPath
  , binaryEntries
  , allEntryPaths
  ) where

import Bit.Types (Hash(..), HashAlgo(..), Path(..), FileEntry(..), EntryKind(..), syncHash, hashToText)
import Bit.Utils (filterOutBitPaths, toPosix)
import Bit.Concurrency (Concurrency(..), runConcurrently, ioConcurrency)
import System.FilePath ((</>), makeRelative, normalise, takeDirectory)
import System.Directory (doesFileExist, listDirectory, doesDirectoryExist, removeFile, createDirectoryIfMissing, removeDirectoryRecursive, getPermissions, setPermissions, setOwnerWritable, setModificationTime)
import Data.Time.Clock.POSIX (posixSecondsToUTCTime)
import Data.List (isPrefixOf)
import Data.Maybe (maybeToList)
import qualified Data.ByteString as BS
import qualified Data.Text as T
import qualified Internal.Git as Git
import Bit.Internal.Metadata (MetaContent(..), parseMetadata, parseMetadataFile, hashFile, serializeMetadata)
import qualified Bit.Remote.Scan as Remote.Scan
import qualified Bit.Remote
import qualified Internal.Transport as Transport
import Internal.Config (fetchedBundle, bitIndexPath, bundleCwdPath, fromCwdPath, BundleName)
import System.Process (readProcessWithExitCode)
import System.Exit (ExitCode(..))
import Data.Char (isSpace)
import System.IO (hPutStrLn, stderr)
import Control.Monad (when, unless, void)
import Data.Foldable (traverse_)
import qualified Data.Map as Map
import qualified Data.Set as Set
import Data.IORef (IORef, atomicModifyIORef')
import qualified Bit.Scan as Scan

-- | Result of comparing one file to metadata.
data VerifyIssue
  = HashMismatch Path String String Integer Integer  -- path, expectedHash, actualHash, expectedSize, actualSize
  | Missing Path                                      -- path (in metadata but no actual file)
  deriving (Show, Eq)

-- | Result of a verification run: count of files checked and list of issues.
-- Replaces bare (Int, [VerifyIssue]) tuple to prevent transposition.
data VerifyResult = VerifyResult
  { vrCount :: Int
  , vrIssues :: [VerifyIssue]
  }
  deriving (Show, Eq)

-- | A metadata entry loaded from any source.
-- Binary files have hash+size metadata that can be verified.
-- Text files are known to exist but their hashes may not be reliably comparable
-- across sources (git normalizes line endings in blobs).
data MetadataEntry
  = BinaryEntry Path (Hash 'MD5) Integer   -- ^ Hash-verifiable: path, hash, size
  | TextEntry Path                          -- ^ Exists but not hash-comparable
  deriving (Show, Eq)

-- | Source for reading metadata entries.
data MetadataSource
  = FromFilesystem FilePath            -- ^ Read .bit/index/ directory on disk
  | FromCommit String                  -- ^ Read from a git commit hash (e.g. refs/remotes/origin/main)
  deriving (Show, Eq)

-- | Extract path from any entry.
entryPath :: MetadataEntry -> Path
entryPath (BinaryEntry p _ _) = p
entryPath (TextEntry p) = p

-- | Binary file metadata: path, hash, size. Replaces bare (Path, Hash, Integer) tuple.
data BinaryFileMeta = BinaryFileMeta
  { bfmPath :: Path
  , bfmHash :: Hash 'MD5
  , bfmSize :: Integer
  }
  deriving (Show, Eq)

-- | Extract verifiable (binary) entries only.
binaryEntries :: [MetadataEntry] -> [BinaryFileMeta]
binaryEntries = concatMap go
  where go (BinaryEntry p h s) = [BinaryFileMeta p h s]
        go (TextEntry _) = []

-- | Extract all known paths (binary + text).
allEntryPaths :: [MetadataEntry] -> Set.Set Path
allEntryPaths = Set.fromList . map entryPath

-- | Filter to user files only (exclude .git internals and .gitignore).
-- Used by BOTH filesystem and commit-tree paths.
isUserFile :: FilePath -> Bool
isUserFile filePath = not (isGitPath filePath) && filePath /= ".gitignore"

-- | Resolve concurrency setting to a concrete bound.
resolveConcurrency :: Concurrency -> IO Int
resolveConcurrency Sequential = pure 1
resolveConcurrency (Parallel 0) = ioConcurrency
resolveConcurrency (Parallel n) = pure n

-- | List all regular files under dir, with paths relative to baseDir.
listFilesRecursive :: FilePath -> FilePath -> IO [FilePath]
listFilesRecursive baseDir dir = do
  entries <- listDirectory dir
  concat <$> mapM (\name -> do
    let full = dir </> name
    isDir <- doesDirectoryExist full
    if isDir
      then listFilesRecursive baseDir full
      else pure [makeRelative baseDir full]
    ) entries

-- | Check if a path is within the .git directory.
isGitPath :: FilePath -> Bool
isGitPath filePath = ".git" `isPrefixOf` normalise filePath || normalise filePath == ".git"

-- | Load metadata entries from any source.
-- Handles file enumeration, parsing, and binary/text classification uniformly.
loadMetadata :: MetadataSource -> Concurrency -> IO [MetadataEntry]
loadMetadata (FromFilesystem indexDir) concurrency = do
  exists <- doesDirectoryExist indexDir
  if not exists
    then pure []
    else do
      relPaths <- listFilesRecursive indexDir indexDir
      let userPaths = filter isUserFile relPaths
      bound <- resolveConcurrency concurrency
      runConcurrently (Parallel bound) (readEntryFromFilesystem indexDir) userPaths

loadMetadata (FromCommit commitHash) _concurrency = do
  -- ls-tree at ROOT level (no prefix!) to enumerate all files
  (code, out, _) <- readProcessWithExitCode "git"
    [ "-C", bitIndexPath, "-c", "core.quotePath=false", "ls-tree", "-r", "--name-only", commitHash ] ""
  case code of
    ExitSuccess -> do
      let paths = filter isUserFile $ filter (not . null) $ lines out
      mapM (readEntryFromCommit commitHash) paths
    _ -> pure []

-- | Read a single metadata entry from a filesystem path.
readEntryFromFilesystem :: FilePath -> FilePath -> IO MetadataEntry
readEntryFromFilesystem indexDir relPath =
  classifyMetadataFile (Path relPath) (indexDir </> relPath)

-- | Read a single metadata entry from a git commit tree.
readEntryFromCommit :: String -> FilePath -> IO MetadataEntry
readEntryFromCommit commitHash relPath = do
  -- NOTE: path is at root level in the commit tree, NOT under index/
  (code, content, _) <- readProcessWithExitCode "git"
    [ "-C", bitIndexPath, "show", commitHash ++ ":" ++ relPath ] ""
  pure $ case code of
    ExitSuccess -> classifyMetadata (Path relPath) content
    _           -> TextEntry (Path relPath)

-- | Classify metadata content string as binary or text entry.
classifyMetadata :: Path -> String -> MetadataEntry
classifyMetadata p content =
  case parseMetadata content of
    Just mc -> BinaryEntry p (metaHash mc) (metaSize mc)
    Nothing -> TextEntry p

-- | Classify a metadata file on disk as binary or text entry.
classifyMetadataFile :: Path -> FilePath -> IO MetadataEntry
classifyMetadataFile p filePath =
  parseMetadataFile filePath >>= \case
    Just mc -> pure (BinaryEntry p (metaHash mc) (metaSize mc))
    Nothing -> pure (TextEntry p)

-- | Load only binary (hash-verifiable) metadata entries from the index.
-- Text files are excluded. If you need all entries, use 'loadMetadata' directly.
loadBinaryMetadata :: FilePath -> Concurrency -> IO [BinaryFileMeta]
loadBinaryMetadata indexDir concurrency =
  binaryEntries <$> loadMetadata (FromFilesystem indexDir) concurrency

-- | Load binary metadata from the committed (HEAD) state of a .bit/index repo.
-- This returns what the metadata *should* be, immune to scan updates.
loadCommittedBinaryMetadata :: FilePath -> IO [BinaryFileMeta]
loadCommittedBinaryMetadata indexDir = do
  (code, out, _) <- Git.runGitAt indexDir ["rev-parse", "HEAD"]
  case code of
    ExitSuccess -> do
      let headHash = filter (not . isSpace) out
      binaryEntries <$> loadMetadata (FromCommit headHash) Sequential
    _ -> pure []

-- | Verify working tree at an arbitrary root path against its committed metadata.
-- Scans the working directory to update .bit/index/ metadata, then uses
-- git diff to find files whose metadata changed from the committed state.
-- Returns (number of files checked, list of issues).
-- If an IORef counter is provided, it will be incremented after each file is checked.
verifyLocalAt :: FilePath -> Maybe (IORef Int) -> Concurrency -> IO VerifyResult
verifyLocalAt root mCounter _concurrency = do
  let indexDir = root </> bitIndexPath

  -- 1. Scan working directory and update .bit/index/ metadata
  entries <- Scan.scanWorkingDir root
  Scan.writeMetadataFiles root entries

  -- 2. Defeat racy git: set mtime to epoch so git always re-reads content.
  --    Without this, git's stat cache can skip content comparison when metadata
  --    files have the same byte length (e.g. two 2-digit file sizes).
  let metaFiles = [indexDir </> unPath (path e) | e <- entries
                  , case kind e of File{} -> True; _ -> False]
  mapM_ (\f -> setModificationTime f (posixSecondsToUTCTime 0)) metaFiles

  -- 3. git diff in the index repo to find files changed from committed state
  (diffCode, diffOut, _) <- Git.runGitAt indexDir ["diff", "--name-only"]
  let changedPaths
        | diffCode == ExitSuccess = filter (not . null) (lines diffOut)
        | otherwise               = []

  -- 4. Also check for missing files: committed paths not in working tree
  (lsCode, lsOut, _) <- Git.runGitAt indexDir ["ls-tree", "-r", "--name-only", "HEAD"]
  let committedPaths
        | lsCode == ExitSuccess = filter isUserFile $ filter (not . null) (lines lsOut)
        | otherwise             = []

  -- 5. Build issues from changed files (hash mismatches)
  mismatchIssues <- concat <$> mapM (checkChanged indexDir) changedPaths

  -- 6. Build issues from missing files (committed but not in working tree)
  missingFiltered <- fmap concat $ mapM (\p -> do
    exists <- doesFileExist (root </> p)
    pure [Missing (Path p) | not exists]
    ) committedPaths

  let allIssues = mismatchIssues ++ missingFiltered
      totalChecked = length committedPaths

  -- Update counter
  traverse_ (\ref -> atomicModifyIORef' ref (\_ -> (totalChecked, ()))) mCounter

  pure (VerifyResult totalChecked allIssues)
  where
    checkChanged indexDir relPath = do
      (showCode, committedContent, _) <- Git.runGitAt indexDir ["show", "HEAD:" ++ relPath]
      let fsPath = indexDir </> relPath
      fsExists <- doesFileExist fsPath
      case (showCode, fsExists) of
        (ExitSuccess, True) -> do
          let committed = classifyMetadata (Path relPath) committedContent
          actual <- classifyMetadataFile (Path relPath) fsPath
          case (committed, actual) of
            (BinaryEntry _ eh es, BinaryEntry _ ah as') ->
              pure [HashMismatch (Path relPath)
                      (T.unpack (hashToText eh))
                      (T.unpack (hashToText ah))
                      es
                      as']
            (BinaryEntry _ eh es, TextEntry _) -> do
              actualHash <- hashFile fsPath
              actualSize <- fromIntegral . BS.length <$> BS.readFile fsPath
              pure [HashMismatch (Path relPath)
                      (T.unpack (hashToText eh))
                      (T.unpack (hashToText actualHash))
                      es
                      actualSize]
            (TextEntry _, TextEntry _) ->
              reportTextMismatch fsPath relPath
            (TextEntry _, BinaryEntry _ _ _) ->
              reportTextMismatch fsPath relPath
        _ -> pure []

    reportTextMismatch fsPath relPath = do
      actualHash <- hashFile fsPath
      actualSize <- fromIntegral . BS.length <$> BS.readFile fsPath
      pure [HashMismatch (Path relPath)
              "(committed)"
              (T.unpack (hashToText actualHash))
              0
              actualSize]

-- | Verify local working tree against committed metadata in .bit/index.
-- Returns (number of files checked, list of issues).
-- If an IORef counter is provided, it will be incremented after each file is checked.
verifyLocal :: FilePath -> Maybe (IORef Int) -> Concurrency -> IO VerifyResult
verifyLocal cwd = verifyLocalAt cwd

-- | Extract metadata from a bundle's HEAD commit.
-- Reads the hash from the bundle, then loads metadata from that commit.
-- The objects must already be in the repo (via git fetch <name> in saveFetchedBundle).
-- Falls back to fetching from the bundle file directly if objects aren't available.
-- Returns all metadata entries (binary + text). Callers extract what they need
-- via 'binaryEntries' or 'allEntryPaths'.
loadMetadataFromBundle :: BundleName -> IO [MetadataEntry]
loadMetadataFromBundle bundleName = do
  -- Get the hash from the bundle
  mHash <- Git.getHashFromBundle bundleName
  case mHash of
    Nothing -> pure []
    Just headHash -> do
      -- Try to load from the commit (objects should already be fetched)
      entries <- loadMetadata (FromCommit headHash) Sequential
      if null entries
        then do
          -- Objects not in repo yet — fetch from bundle file directly
          let bundlePath = fromCwdPath (bundleCwdPath bundleName)
          (fetchCode, _, _) <- readProcessWithExitCode "git"
            ["-C", bitIndexPath, "fetch", bundlePath, "+refs/heads/*:refs/remotes/bundle/*"] ""
          case fetchCode of
            ExitSuccess -> loadMetadata (FromCommit headHash) Sequential
            _ -> pure []
        else pure entries

-- | Verify remote files match remote metadata.
-- Sets up a temporary working tree in .bit/vremotes/<name>/, checks out the
-- expected state from the bundle, overwrites with actual remote data (rclone
-- metadata for binaries, downloaded content for text files), then runs a single
-- git diff to find all mismatches.
-- Returns (number of files checked, list of issues).
-- If an IORef counter is provided, it will be incremented after each file is checked.
verifyRemote :: FilePath -> Bit.Remote.Remote -> Maybe (IORef Int) -> Concurrency -> IO VerifyResult
verifyRemote cwd remote mCounter _concurrency = do
  -- 1. Fetch the remote bundle if needed
  let fetchedPath = fromCwdPath (bundleCwdPath fetchedBundle)
  bundleExists <- doesFileExist fetchedPath
  unless bundleExists $ do
    let localDest = ".bit/temp_remote.bundle"
    fetchResult <- Transport.copyFromRemoteDetailed remote ".bit/bit.bundle" localDest
    case fetchResult of
      Transport.CopySuccess -> do
        BS.readFile localDest >>= BS.writeFile fetchedPath
        when (localDest /= fetchedPath) $ safeRemove localDest
      _ -> do
        hPutStrLn stderr "Error: Could not fetch remote bundle."
        pure ()

  bundleExistsNow <- doesFileExist fetchedPath
  if not bundleExistsNow
    then pure (VerifyResult 0 [])
    else do
      -- 2. Load metadata from bundle (classifies entries; fetches bundle into .bit/index)
      entries <- loadMetadataFromBundle fetchedBundle
      let allKnownPaths = allEntryPaths entries

      -- 3. Fetch remote file list via rclone ls
      Remote.Scan.fetchRemoteFiles remote >>= either
        (const $ hPutStrLn stderr "Error: Could not fetch remote file list." >> pure (VerifyResult 0 []))
        (\remoteFiles -> do
          let filteredRemoteFiles = filterOutBitPaths remoteFiles
              remoteFileMap = Map.fromList
                [ (normalise (unPath e.path), (h, e.kind))
                | e <- filteredRemoteFiles
                , h <- maybeToList (syncHash e.kind)
                ]

          -- 4. Set up verification working tree
          let vremoteDir = cwd </> ".bit" </> "vremotes" </> Bit.Remote.remoteName remote
          forceRemoveDir vremoteDir
          createDirectoryIfMissing True vremoteDir

          -- Checkout expected state from the remote's bundle
          let absBundlePath = cwd </> fetchedPath
          void $ Git.runGitAt vremoteDir ["init", "-q"]
          void $ Git.runGitAt vremoteDir ["fetch", "-q", absBundlePath, "refs/heads/main:refs/heads/main"]
          void $ Git.runGitAt vremoteDir ["checkout", "-q", "main"]

          -- 5. Overwrite working tree with actual remote state:
          --    Binary: construct metadata from rclone ls data
          --    Text: download actual content from remote
          --    Missing: delete the checked-out file
          mapM_ (\entry -> do
            let p = entryPath entry
                destFile = vremoteDir </> unPath p
            case entry of
              BinaryEntry _ _ _ ->
                case Map.lookup (normalise (unPath p)) remoteFileMap of
                  Just (h, File _ sz _) ->
                    writeFile destFile (serializeMetadata (MetaContent h sz))
                  _ -> safeRemove destFile
              TextEntry _ -> do
                code <- Transport.copyFromRemote remote (toPosix (unPath p)) destFile
                when (code /= ExitSuccess) $ safeRemove destFile
            traverse_ (\ref -> atomicModifyIORef' ref (\n -> (n + 1, ()))) mCounter
            ) entries

          -- 6. Single git diff: compares actual working tree against expected (HEAD)
          (_, diffOut, _) <- Git.runGitAt vremoteDir
            ["diff", "--ignore-cr-at-eol", "--name-status", "HEAD"]
          let entryMap = Map.fromList
                [(normalise (unPath (entryPath e)), e) | e <- entries]
              issues = concatMap (parseDiffLine entryMap remoteFileMap)
                (filter (not . null) (lines diffOut))

          -- 7. Extra files on remote not in bundle metadata
          let filePaths = Set.fromList (map Path (Map.keys remoteFileMap))
              extraPaths = filePaths `Set.difference` allKnownPaths
              extraIssues = map (\p -> HashMismatch p "(not in metadata)" "(exists on remote)" 0 0) (Set.toList extraPaths)

          -- 8. Clean up (.git/objects are read-only, need forceRemoveDir)
          forceRemoveDir vremoteDir

          pure (VerifyResult (length entries) (issues ++ extraIssues)))
  where
    parseDiffLine entryMap remoteFileMap line =
      let (status, rest) = break (== '\t') line
          path = drop 1 rest
          npath = normalise path
      in case status of
        "D" -> [Missing (Path path)]
        "M" -> case (Map.lookup npath entryMap, Map.lookup npath remoteFileMap) of
          (Just (BinaryEntry p eh es), Just (ah, File _ as' _)) ->
            [HashMismatch p (T.unpack (hashToText eh)) (T.unpack (hashToText ah)) es as']
          _ -> [HashMismatch (Path path) "(committed)" "(remote differs)" 0 0]
        _ -> []

-- Helper to safely remove a file
safeRemove :: FilePath -> IO ()
safeRemove filePath = do
  exists <- doesFileExist filePath
  when exists $ removeFile filePath

-- | Remove a directory tree, handling read-only files (e.g. .git/objects).
forceRemoveDir :: FilePath -> IO ()
forceRemoveDir dir = do
  exists <- doesDirectoryExist dir
  when exists $ do
    makeWritable dir
    removeDirectoryRecursive dir
  where
    makeWritable d = do
      contents <- listDirectory d
      mapM_ (\name -> do
        let full = d </> name
        isDir <- doesDirectoryExist full
        if isDir
          then makeWritable full
          else do
            perms <- getPermissions full
            setPermissions full (setOwnerWritable True perms)
        ) contents
```

---

## Internal/Config.hs

**Path:** `Internal/Config.hs`

*Source file.*

```haskell
module Internal.Config where

import System.FilePath ((</>))

-- | Logical bundle name (e.g. "fetched_remote", "bit"). NOT a file path.
newtype BundleName = BundleName String deriving (Show, Eq)

-- | Path relative to the git working directory (.bit/index/).
-- Used for git commands that run with -C .bit/index.
newtype GitRelPath = GitRelPath FilePath deriving (Show, Eq)

-- | Path relative to CWD. Used for direct filesystem operations (copyFile, doesFileExist, etc).
newtype CwdPath = CwdPath FilePath deriving (Show, Eq)

bitDir, bitTargetPath, bitGitDir, bitIndexPath, bitDevicesDir, bitRemotesDir :: FilePath
bitDir           = ".bit"
bitTargetPath    = bitDir </> "target"
bitDevicesDir    = bitDir </> "devices"
bitRemotesDir    = bitDir </> "remotes"
bitIndexPath     = bitDir </> "index"
bitGitDir        = bitIndexPath </> ".git"

-- | The logical name of the fetched remote bundle
fetchedBundle :: BundleName
fetchedBundle = BundleName "fetched_remote"

-- | Legacy string name for gradual migration
fetchedBundleName :: FilePath
fetchedBundleName = "fetched_remote"

-- | Legacy CWD path for gradual migration
fetchedBundlePath :: FilePath
fetchedBundlePath = bitIndexPath </> ".git" </> (fetchedBundleName ++ ".bundle")

-- | Convert a bundle name to a path relative to git working directory (.bit/index/)
-- Use this for git commands that run with -C .bit/index
bundleGitRelPath :: BundleName -> GitRelPath
bundleGitRelPath (BundleName n) = GitRelPath (".git" </> (n ++ ".bundle"))

-- | Convert a bundle name to a path relative to CWD
-- Use this for filesystem operations (copyFile, doesFileExist, etc.)
bundleCwdPath :: BundleName -> CwdPath
bundleCwdPath (BundleName n) = CwdPath (bitIndexPath </> ".git" </> (n ++ ".bundle"))

-- | Unwrap CwdPath to FilePath
fromCwdPath :: CwdPath -> FilePath
fromCwdPath (CwdPath p) = p

-- | Unwrap GitRelPath to FilePath
fromGitRelPath :: GitRelPath -> FilePath
fromGitRelPath (GitRelPath p) = p
```

---

## Internal/ConfigFile.hs

**Path:** `Internal/ConfigFile.hs`

*Source file.*

```haskell
{-# LANGUAGE OverloadedStrings #-}

module Internal.ConfigFile
  ( TextConfig(..)
  , defaultTextConfig
  , readConfig
  , readTextConfig
  ) where

import System.FilePath ((</>))
import System.Directory (doesFileExist)
import Data.Char (isSpace)
import Data.List (dropWhileEnd)
import Data.Maybe (fromMaybe, listToMaybe)
import qualified Data.Text as T
import qualified Data.Text.Encoding as T
import qualified Data.ByteString as BS
import Internal.Config (bitDir)

-- | Configuration for text file classification
data TextConfig = TextConfig
  { textSizeLimit :: Integer  -- Files larger than this are always binary
  , textExtensions :: [String]  -- Extensions that are always text (if other checks pass)
  } deriving (Show, Eq)

-- | Default text file configuration
defaultTextConfig :: TextConfig
defaultTextConfig = TextConfig
  { textSizeLimit = 1048576  -- 1MB
  , textExtensions = [".txt", ".md", ".yaml", ".yml", ".json", ".xml", ".html", ".css", ".js", ".py", ".hs", ".rs"]
  }

-- | Path to config file
configPath :: FilePath
configPath = bitDir </> "config"

-- | Read the entire config file and return TextConfig
-- Falls back to defaultTextConfig if file doesn't exist or parsing fails
-- Uses strict ByteString reading to avoid Windows file locking issues
readTextConfig :: IO TextConfig
readTextConfig = do
  exists <- doesFileExist configPath
  if not exists
    then pure defaultTextConfig
    else do
      bs <- BS.readFile configPath
      let content = either (const T.empty) id (T.decodeUtf8' bs)
      pure $ parseConfig content

-- | Read config file (for future expansion)
readConfig :: IO TextConfig
readConfig = readTextConfig

-- | Parse config file content (INI-style format)
parseConfig :: T.Text -> TextConfig
parseConfig content = 
  let linesOfText = T.lines content
      -- Find [text] section
      textSection = extractSection "text" linesOfText
      -- Parse size-limit
      sizeLimit = fromMaybe (textSizeLimit defaultTextConfig) (parseSizeLimit textSection)
      -- Parse extensions
      extensions = fromMaybe (textExtensions defaultTextConfig) (parseExtensions textSection)
  in TextConfig { textSizeLimit = sizeLimit, textExtensions = extensions }

-- | Find index of first element matching predicate
findIndex' :: (a -> Bool) -> [a] -> Maybe Int
findIndex' p xs = listToMaybe [i | (i, x) <- zip [0..] xs, p x]

-- | Extract lines for a given section (between [section] and next [section] or EOF)
extractSection :: String -> [T.Text] -> [T.Text]
extractSection sectionName linesOfText =
  let sectionHeader = "[" ++ sectionName ++ "]"
      -- Find start of section
      startIdx = maybe (length linesOfText) (+ 1) $
        findIndex' (\l -> T.strip l == T.pack sectionHeader) linesOfText
      -- Find end of section (next [section] or EOF)
      endIdx = maybe (length linesOfText) (+ startIdx) $
        findIndex' (\l -> T.stripStart l `T.isPrefixOf` T.pack "[") (drop startIdx linesOfText)
  in map T.strip $ take (endIdx - startIdx) (drop startIdx linesOfText)

-- | Parse size-limit from section lines
parseSizeLimit :: [T.Text] -> Maybe Integer
parseSizeLimit linesOfText =
  let findLine prefix = [T.unpack (T.drop (T.length (T.pack prefix)) (T.strip l)) | l <- linesOfText, T.stripStart l `T.isPrefixOf` T.pack prefix]
      sizeLines = findLine "size-limit"
  in listToMaybe sizeLines >>= \sizeStr ->
      let cleaned = takeWhile (/= '#') sizeStr
          trimmed = dropWhile isSpace $ dropWhileEnd isSpace cleaned
      in case reads trimmed of
        [(n, "")] -> Just n
        _ -> Nothing

-- | Parse extensions from section lines
parseExtensions :: [T.Text] -> Maybe [String]
parseExtensions linesOfText =
  let findLine prefix = [T.unpack (T.drop (T.length (T.pack prefix)) (T.strip l)) | l <- linesOfText, T.stripStart l `T.isPrefixOf` T.pack prefix]
      extLines = findLine "extensions"
  in listToMaybe extLines >>= \extStr ->
      let cleaned = takeWhile (/= '#') extStr
          trimmed = dropWhile isSpace $ dropWhileEnd isSpace cleaned
          exts = map (dropWhile isSpace . dropWhileEnd isSpace) $ splitComma trimmed
      in Just exts
  where
    splitComma s = case break (== ',') s of
      (part, "") -> [part]
      (part, _:rest) -> part : splitComma rest
```

---

## Internal/Git.hs

**Path:** `Internal/Git.hs`

*Source file.*

```haskell
{-# LANGUAGE MultiWayIf #-}

module Internal.Git
    ( add
    , commit
    , commitFile
    , diff
    , init
    , reset
    , rm
    , mv
    , branch
    , merge
    , createBundle
    , config
    , getLocalHead
    , AncestorQuery(..)
    , checkIsAhead
    , getHashFromBundle
    , restore
    , checkout
    , status
    , addRemote
    , getRemoteUrl
    , getTrackedRemoteName
    , updateRemoteTrackingBranch
    , updateRemoteTrackingBranchToHead
    , updateRemoteTrackingBranchToHash
    , setupBranchTracking
    , setupBranchTrackingFor
    , unsetBranchUpstream
    , mergeAbort
    , isMergeInProgress
    , checkoutRemoteAsMain
    , getConflictedFiles
    , getConflictType
    , checkoutOurs
    , checkoutTheirs
    , runGitRaw
    , runGitRawAt
    , runGitWithOutput
    , runGitAt
    , rewriteGitHints
    , DeletedSide(..)
    , ConflictType(..)
    , readFileFromRef
    , listFilesInRef
    , fsck
    , hasStagedChanges
    , getDiffNameStatus
    , getFilesAtCommit
    , remoteTrackingRef
    , NameStatusChange(..)
    , parseNameStatusOutput
    ) where

import Data.Maybe (mapMaybe, listToMaybe)

import System.Process (readProcessWithExitCode)
import System.Exit (ExitCode(..))
import Internal.Config
import Data.Char (isSpace)
import Control.Monad (when, guard)
import Prelude hiding (init)
import Data.List (isPrefixOf)
import System.IO (hPutStr, hPutStrLn, stderr)
import System.Environment (lookupEnv)

baseFlags :: [String]
baseFlags = ["-C", bitIndexPath]

-- | Build the tracking ref for a named remote: @refs/remotes/\<name\>/main@
remoteTrackingRef :: String -> String
remoteTrackingRef name = "refs/remotes/" ++ name ++ "/main"

-- | Query: is aqAncestor an ancestor of aqDescendant? Record avoids transposing the two String hashes.
data AncestorQuery = AncestorQuery { aqAncestor, aqDescendant :: String }
  deriving (Show, Eq)

-- | Represents the subset of Git functionality rgit uses
data GitCommand
    = Init { _separateGitDir :: FilePath }
    | Config { _configName :: String, _configValue :: String }
    | RevParse { _revParseRef :: String }
    | CommitFile { _commitMessage :: String, _commitFile :: FilePath }
    | RevList { _revListLeft :: String, _revListRight :: String }
    | CreateBundle { _createBundlePath :: FilePath }
    | GetBundleHead { _getBundleHeadPath :: FilePath }
    | IsAncestor AncestorQuery
    | GetHead

-- | Run a Git command and return (ExitCode, StdOut, StdErr)
runGit :: GitCommand -> IO (ExitCode, String, String)
runGit cmd = do
    let subArgs = translateCommand cmd
    let fullArgs = baseFlags ++ subArgs
    -- We use readProcessWithExitCode so we can handle errors without crashing
    readProcessWithExitCode "git" fullArgs ""
  where
    translateCommand :: GitCommand -> [String]
    translateCommand c = case c of
        Init _dir ->
            -- dir is the full path to .git directory (e.g., .rgit/index/.git)
            -- We need to change to the parent directory and run git init there
            -- Git will automatically create .git in the current directory
            ["init"]

        Config k v ->
            ["config", k, v]

        RevParse r ->
            ["rev-parse", r]

        CommitFile msg f ->
            -- Using "-- f" at the end tells Git to only look at that path
            ["commit", "-m", msg, "--", f]

        RevList l r ->
            ["rev-list", "--left-right", "--count", l ++ "..." ++ r]

        CreateBundle path ->
            ["bundle", "create", path, "--all"]

        GetBundleHead path ->
            ["bundle", "list-heads", path]

        IsAncestor (AncestorQuery a d) ->
            ["merge-base", "--is-ancestor", a, d]

        GetHead ->
            ["rev-parse", "HEAD"]

getLocalHead :: IO (Maybe String)
getLocalHead = do
    (code, out, _) <- runGit GetHead
    pure (guard (code == ExitSuccess) >> Just (filter (not . isSpace) out))

getHashFromBundle :: BundleName -> IO (Maybe String)
getHashFromBundle bundleName = do
    let (GitRelPath relPath) = bundleGitRelPath bundleName
    (code, out, _) <- runGit (GetBundleHead relPath)
    pure (guard (code == ExitSuccess && not (null out)) >> listToMaybe (words out))

runGitCommand :: GitCommand -> IO ExitCode
runGitCommand cmd = do
    (c, o, e) <- runGit cmd
    -- Don't print error messages for IsAncestor since non-zero exit codes are expected
    -- (they indicate "no, not an ancestor" which is a valid answer, not an error)
    when (c /= ExitSuccess && not (isAncestorCommand cmd)) $ 
        hPutStrLn stderr ("bit: git command failed: " ++ e)
    putStr o
    hPutStr stderr e
    pure c
  where
    isAncestorCommand (IsAncestor _) = True
    isAncestorCommand _ = False

commitFile :: String -> FilePath -> IO ExitCode
commitFile msg filePath = runGitCommand (CommitFile msg filePath)

init :: FilePath -> IO ExitCode
init dir = runGitCommand (Init dir)

createBundle :: BundleName -> IO ExitCode
createBundle bundleName =
    let (GitRelPath relPath) = bundleGitRelPath bundleName
    in runGitCommand (CreateBundle relPath)

config :: String -> String -> IO ExitCode
config configName configValue = runGitCommand (Config configName configValue)

-- | Check if aqDescendant is ahead of aqAncestor (i.e., aqAncestor is an ancestor of aqDescendant).
checkIsAhead :: AncestorQuery -> IO Bool
checkIsAhead q =
    (== ExitSuccess) <$> runGitCommand (IsAncestor q)

replace :: String -> String -> String -> String
replace _ _ [] = []
replace old new str@(c:cs)
    | old `isPrefixOf` str = new ++ replace old new (drop (length old) str)
    | otherwise            = c : replace old new cs

rewriteGitHints :: String -> String
rewriteGitHints =
    replace "(use \"git " "(use \"bit "



runGitRaw :: [String] -> IO ExitCode
runGitRaw args = do
  noColor <- lookupEnv "BIT_NO_COLOR"
  let colorFlag = case noColor of
        Just "1" -> "never"
        Just "true" -> "never"
        _ -> "auto"
  let fullArgs =
        baseFlags
        ++ ["-c", "color.ui=" ++ colorFlag]
        ++ args

  (code, out, err) <- readProcessWithExitCode "git" fullArgs ""

  putStr (rewriteGitHints out)
  hPutStr stderr (rewriteGitHints err)

  case code of
    ExitSuccess   -> pure ()
    ExitFailure n ->
      hPutStrLn stderr ("bit: git exited with code " ++ show n)

  pure code

-- | Like runGitRaw but targets an arbitrary directory instead of .bit/index.
runGitRawAt :: FilePath -> [String] -> IO ExitCode
runGitRawAt dir args = do
  noColor <- lookupEnv "BIT_NO_COLOR"
  let colorFlag = case noColor of
        Just "1" -> "never"
        Just "true" -> "never"
        _ -> "auto"
  let fullArgs =
        ["-C", dir]
        ++ ["-c", "color.ui=" ++ colorFlag]
        ++ args

  (code, out, err) <- readProcessWithExitCode "git" fullArgs ""

  putStr (rewriteGitHints out)
  hPutStr stderr (rewriteGitHints err)

  case code of
    ExitSuccess   -> pure ()
    ExitFailure n ->
      hPutStrLn stderr ("bit: git exited with code " ++ show n)

  pure code

add :: [String] -> IO ExitCode
add     = runGitRaw . ("add" :)
commit :: [String] -> IO ExitCode
commit  = runGitRaw . ("commit" :)
diff :: [String] -> IO ExitCode
diff    = runGitRaw . ("diff" :)
restore :: [String] -> IO ExitCode
restore  = runGitRaw . ("restore" :)
checkout :: [String] -> IO ExitCode
checkout = runGitRaw . ("checkout" :)
status :: [String] -> IO ExitCode
status   = runGitRaw . ("status" :)
reset :: [String] -> IO ExitCode
reset   = runGitRaw . ("reset" :)
rm :: [String] -> IO ExitCode
rm      = runGitRaw . ("rm" :)
mv :: [String] -> IO ExitCode
mv      = runGitRaw . ("mv" :)
branch :: [String] -> IO ExitCode
branch  = runGitRaw . ("branch" :)
merge :: [String] -> IO ExitCode
merge   = runGitRaw . ("merge" :)

-- | Add or update a remote (Git-style: git remote add <name> <url> / set-url if exists)
addRemote :: String -> String -> IO ExitCode
addRemote remoteName url = do
    (code, _, _) <- readProcessWithExitCode "git" (baseFlags ++ ["remote", "get-url", remoteName]) ""
    case code of
        ExitSuccess -> do
            readProcessWithExitCode "git" (baseFlags ++ ["remote", "set-url", remoteName, url]) "" >>= \(c, _, _) -> pure c
        ExitFailure _ -> do
            readProcessWithExitCode "git" (baseFlags ++ ["remote", "add", remoteName, url]) "" >>= \(c, _, _) -> pure c

-- | Get the URL for a remote by name (git remote get-url <name>). Returns Nothing if remote missing.
getRemoteUrl :: String -> IO (Maybe String)
getRemoteUrl remoteName = do
    (code, out, _) <- readProcessWithExitCode "git" (baseFlags ++ ["remote", "get-url", remoteName]) ""
    pure $ case code of
        ExitSuccess -> Just (filter (/= '\n') out)
        _ -> Nothing

-- | Get the remote name that the current branch tracks (branch.main.remote).
-- Falls back to "origin" if not configured — this means commands work with
-- a reasonable default even without explicit -u, but callers should be aware
-- that "origin" fallback doesn't mean tracking IS configured.
getTrackedRemoteName :: IO String
getTrackedRemoteName = do
    (code, out, _) <- readProcessWithExitCode "git" (baseFlags ++ ["config", "--get", "branch.main.remote"]) ""
    pure $ case code of
        ExitSuccess -> filter (/= '\n') out
        _ -> "origin"

-- | Update the remote tracking branch refs/remotes/<name>/main to point to the hash from the bundle.
-- Use when the objects are already in the repo (e.g. after push).
updateRemoteTrackingBranch :: String -> BundleName -> IO ExitCode
updateRemoteTrackingBranch name bundleName =
    getHashFromBundle bundleName >>= maybe (pure (ExitFailure 1)) (updateRemoteTrackingBranchToHash name)

-- | Set refs/remotes/<name>/main to a specific hash. Use after a successful pull so status shows
-- "up to date with '<name>/main'" instead of "ahead by N commits".
updateRemoteTrackingBranchToHash :: String -> String -> IO ExitCode
updateRemoteTrackingBranchToHash name hash =
    readProcessWithExitCode "git" (baseFlags ++ ["update-ref", remoteTrackingRef name, hash]) "" >>= \(c, _, _) -> pure c

-- | Set refs/remotes/<name>/main to current HEAD.
-- WARNING: Only correct after PUSH (where remote now matches local HEAD).
-- After PULL/MERGE, use updateRemoteTrackingBranchToHash with the remote hash instead,
-- because HEAD includes local merge commits the remote doesn't have.
-- See: "Tracking Ref Invariant" in docs/spec.md.
updateRemoteTrackingBranchToHead :: String -> IO ExitCode
updateRemoteTrackingBranchToHead name = do
    (code, out, _) <- readProcessWithExitCode "git" (baseFlags ++ ["rev-parse", "HEAD"]) ""
    case filter (/= '\n') out of
        hash | code == ExitSuccess && not (null hash) ->
            updateRemoteTrackingBranchToHash name hash
        _ -> pure (ExitFailure 1)

-- | Set up the local branch to track a specific remote
-- Configures branch.main.remote and branch.main.merge
setupBranchTrackingFor :: String -> IO ExitCode
setupBranchTrackingFor remoteName = do
    (code1, _, _) <- readProcessWithExitCode "git"
        (baseFlags ++ ["config", "branch.main.remote", remoteName]) ""
    (code2, _, _) <- readProcessWithExitCode "git"
        (baseFlags ++ ["config", "branch.main.merge", "refs/heads/main"]) ""
    case (code1, code2) of
        (ExitSuccess, ExitSuccess) -> pure ExitSuccess
        _ -> pure (ExitFailure 1)

-- | Set up the local branch to track origin/main
-- This configures branch.main.remote and branch.main.merge so git status knows what to compare
setupBranchTracking :: IO ExitCode
setupBranchTracking = setupBranchTrackingFor "origin"

-- | Unset the upstream for the current branch (clears "upstream is gone" when remote refs are missing)
unsetBranchUpstream :: IO ExitCode
unsetBranchUpstream = do
    (code, _, _) <- readProcessWithExitCode "git" (baseFlags ++ ["branch", "--unset-upstream"]) ""
    pure code

-- | Run git with baseFlags; returns (exitCode, stdout, stderr). Does not rewrite hints.
runGitWithOutput :: [String] -> IO (ExitCode, String, String)
runGitWithOutput args = do
  let fullArgs = baseFlags ++ ["-c", "color.ui=never"] ++ args
  readProcessWithExitCode "git" fullArgs ""

-- | Abort an in-progress merge.
mergeAbort :: IO ExitCode
mergeAbort = do
  (code, out, err) <- runGitWithOutput ["merge", "--abort"]
  putStr (rewriteGitHints out)
  hPutStr stderr (rewriteGitHints err)
  pure code

-- | True if a merge is in progress (MERGE_HEAD exists).
isMergeInProgress :: IO Bool
isMergeInProgress = do
  (code, _, _) <- runGitWithOutput ["rev-parse", "--verify", "MERGE_HEAD"]
  pure (code == ExitSuccess)

-- | Checkout refs/remotes/<name>/main as the local main branch.
-- Used on first pull when there are no local commits (unborn branch).
-- This avoids the need for merge and gives us the remote's history directly.
--
-- TRACKING CONTRACT: This function NEVER modifies branch tracking config.
-- Uses --no-track to prevent git from auto-setting branch.main.remote.
-- Callers who need tracking must call setupBranchTrackingFor explicitly.
--
-- Uses -f (force) to overwrite any local files created during init.
checkoutRemoteAsMain :: String -> IO ExitCode
checkoutRemoteAsMain name = do
  -- Use checkout -B to create/reset branch and checkout in one step
  -- Use -f to force overwrite of any local files (like .gitattributes from init)
  -- Use --no-track to prevent auto-setting branch.main.remote (git-standard: require explicit -u)
  (code, _, _) <- runGitWithOutput ["checkout", "-f", "-B", "main", "--no-track", remoteTrackingRef name]
  pure code

-- | Paths relative to work tree (index/...) that are unmerged.
getConflictedFiles :: IO [FilePath]
getConflictedFiles = do
  (code, out, _) <- runGitWithOutput ["diff", "--name-only", "--diff-filter=U"]
  pure $ case code of
    ExitSuccess -> filter (not . null) (lines out)
    _ -> []

-- | Which side deleted the file in a modify/delete conflict.
data DeletedSide
  = DeletedInOurs   -- ^ Deleted in HEAD (ours); modified in theirs
  | DeletedInTheirs -- ^ Deleted in theirs; modified in HEAD (ours)
  deriving (Show, Eq)

-- | Conflict type for Git-like messages. Path is work-tree relative (e.g. index/src/model.bin).
data ConflictType
  = ContentConflict FilePath
  | ModifyDelete FilePath DeletedSide
  | AddAdd FilePath
  deriving (Show, Eq)

-- | Determine conflict type using git ls-files -u. Path is as in index (e.g. index/foo).
-- Format: "mode SP oid SP stage TAB name"
getConflictType :: FilePath -> IO ConflictType
getConflictType path = do
  (_, out, _) <- runGitWithOutput ["ls-files", "-u", "--", path]
  let beforeTab line = takeWhile (/= '\t') line
  let stageNums :: [Int]
      stageNums = mapMaybe stageNum (lines out)
      stageNum line = case reverse (words (beforeTab line)) of
        ("1":_) -> Just (1 :: Int)
        ("2":_) -> Just 2
        ("3":_) -> Just 3
        _ -> Nothing
  let has1 = 1 `elem` stageNums
  let has2 = 2 `elem` stageNums
  let has3 = 3 `elem` stageNums
  pure $ if | has2 && has3 && has1     -> ContentConflict path
            | has2 && has3 && not has1 -> AddAdd path
            | has2 && not has3         -> ModifyDelete path DeletedInTheirs
            | has3 && not has2         -> ModifyDelete path DeletedInOurs
            | otherwise                -> ContentConflict path

-- | Check out our version for path (work-tree path under .rgit/index).
checkoutOurs :: FilePath -> IO ExitCode
checkoutOurs path = do
  (code, out, err) <- runGitWithOutput ["checkout", "--ours", "--", path]
  putStr (rewriteGitHints out)
  hPutStr stderr (rewriteGitHints err)
  pure code

-- | Check out their version for path.
checkoutTheirs :: FilePath -> IO ExitCode
checkoutTheirs path = do
  (code, out, err) <- runGitWithOutput ["checkout", "--theirs", "--", path]
  putStr (rewriteGitHints out)
  hPutStr stderr (rewriteGitHints err)
  pure code

-- | Read file content from a Git ref (e.g., "refs/remotes/origin/main:path/to/file").
-- Returns Nothing if file doesn't exist in that ref.
readFileFromRef :: String -> FilePath -> IO (Maybe String)
readFileFromRef gitRef path = do
  (code, out, _err) <- runGitWithOutput ["show", gitRef ++ ":" ++ path]
  pure $ case code of
    ExitSuccess | not (null out) -> Just out
    _ -> Nothing

-- | List all files in a Git ref's tree (recursive). Returns paths relative to work tree root.
listFilesInRef :: String -> IO [FilePath]
listFilesInRef gitRef = do
  (code, out, _) <- runGitWithOutput ["ls-tree", "-r", "--name-only", gitRef]
  pure $ case code of
    ExitSuccess -> filter (not . null) (lines out)
    _ -> []

-- | Run git fsck to check metadata history integrity.
-- Returns (exitCode, output, errorOutput).
fsck :: IO (ExitCode, String, String)
fsck = runGitWithOutput ["fsck"]

-- | Check if there are staged changes ready to commit.
-- Returns True if there are staged changes, False otherwise.
hasStagedChanges :: IO Bool
hasStagedChanges = do
  (code, _, _) <- runGitWithOutput ["diff", "--cached", "--quiet"]
  pure (code == ExitFailure 1)  -- git diff --cached --quiet exits with 1 if there are changes

-- | Parsed line from `git diff --name-status` output.
-- Makes invalid states unrepresentable (no bare Char + Maybe tuple).
data NameStatusChange
    = Added FilePath
    | Deleted FilePath
    | Modified FilePath
    | Renamed FilePath FilePath  -- ^ old path, new path
    | Copied FilePath FilePath   -- ^ old path, new path
    deriving (Show, Eq)

-- | Parse raw `git diff --name-status` output into structured changes.
parseNameStatusOutput :: String -> [NameStatusChange]
parseNameStatusOutput = mapMaybe parseLine . lines
  where
    parseLine line = case line of
        (fileStatus:rest)
            | fileStatus == 'R' || fileStatus == 'C' ->
                case words (dropWhile (\c -> c /= '\t' && c /= ' ') rest) of
                    (old:new:_) -> Just $ case fileStatus of
                        'R' -> Renamed old new
                        _  -> Copied old new
                    _ -> Nothing
            | fileStatus == 'A' ->
                case words rest of (path:_) -> Just (Added path); _ -> Nothing
            | fileStatus == 'D' ->
                case words rest of (path:_) -> Just (Deleted path); _ -> Nothing
            | fileStatus == 'M' ->
                case words rest of (path:_) -> Just (Modified path); _ -> Nothing
            | otherwise -> Nothing
        _ -> Nothing

-- | Get the list of file changes between two commits.
getDiffNameStatus :: String -> String -> IO [NameStatusChange]
getDiffNameStatus oldHead newHead = do
    (code, out, _) <- runGitWithOutput ["diff", "--name-status", oldHead, newHead]
    pure $ case code of
        ExitSuccess -> parseNameStatusOutput out
        _ -> []

-- | Get all file paths at a given commit. Used when there's no old HEAD to diff against.
getFilesAtCommit :: String -> IO [FilePath]
getFilesAtCommit gitRef = do
    (code, out, _) <- runGitWithOutput ["ls-tree", "-r", "--name-only", gitRef]
    pure $ case code of
        ExitSuccess -> filter (not . null) (lines out)
        _ -> []

-- | Run a git command targeting a specific index path (for filesystem remotes).
-- This is used when operating on a remote filesystem repo directly.
-- The indexPath should be the path to the .bit/index directory (NOT the .git subdirectory).
runGitAt :: FilePath -> [String] -> IO (ExitCode, String, String)
runGitAt indexPath args = readProcessWithExitCode "git" (["-C", indexPath] ++ args) ""
```

---

## Internal/Transport.hs

**Path:** `Internal/Transport.hs`

*Source file.*

```haskell
{-# LANGUAGE DeriveGeneric #-}
{-# LANGUAGE DeriveAnyClass #-}

module Internal.Transport
    ( copyToRemote
    , copyFromRemote
    , copyFromRemoteDetailed
    , CopyResult(..)
    , moveRemote
    , deleteRemote
    , purgeRemote
    , mkdirRemote
    , listRemoteJson
    , listRemoteJsonWithHash
    , listRemoteItems
    , TransportItem(..)
    , checkRemote
    , CheckResult(..)
    ) where

import System.Process (readProcessWithExitCode, CreateProcess(..), StdStream(..), proc, waitForProcess, createProcess)
import System.Exit (ExitCode(..))
import System.IO (hGetLine, hIsEOF, hClose, Handle)
import System.FilePath (normalise)
import Control.Monad (unless, void)
import Data.Foldable (traverse_)
import Control.Concurrent.Async (async, wait)
import Data.List (dropWhileEnd, isInfixOf)
import qualified Data.Aeson as Aeson
import qualified Data.ByteString as BS
import qualified Data.ByteString.Lazy as LBS
import GHC.Generics (Generic)
import Data.String (fromString)
import Bit.Remote (Remote, remoteUrl)
import Data.IORef (IORef, modifyIORef')
import Control.Exception (bracket, try, SomeException)

-- | Run a process and capture stdout as raw bytes (avoiding locale encoding issues).
-- Returns (ExitCode, ByteString, String) where stdout is raw bytes and stderr is String.
-- Uses bracket for exception-safe resource cleanup.
-- Reads stdout and stderr concurrently to avoid pipe deadlocks.
readProcessBytes :: FilePath -> [String] -> IO (ExitCode, LBS.ByteString, String)
readProcessBytes cmd args = do
    let cp = (proc cmd args)
            { std_out = CreatePipe
            , std_err = CreatePipe
            , std_in = Inherit
            }
    bracket (createProcess cp) cleanupProcess $ \(_, mStdout, mStderr, ph) -> do
        case (mStdout, mStderr) of
            (Just hOut, Just hErr) -> do
                -- Read stdout and stderr concurrently to avoid deadlocks
                -- BS.hGetContents is strict and closes handle when done
                asyncOut <- async (BS.hGetContents hOut)
                asyncErr <- async (hGetContents' hErr)
                -- Wait for both reads to complete
                outBytes <- wait asyncOut
                errStr   <- wait asyncErr
                code     <- waitForProcess ph
                pure (code, LBS.fromStrict outBytes, errStr)
            _ -> error "readProcessBytes: failed to create pipes"
  where
    -- Cleanup: close any handles that might still be open and wait for process
    cleanupProcess (mStdin, mStdout, mStderr, ph) = do
        -- Try to close handles (may already be closed by hGetContents)
        traverse_ (\h -> void (try (hClose h) :: IO (Either SomeException ()))) mStdin
        traverse_ (\h -> void (try (hClose h) :: IO (Either SomeException ()))) mStdout
        traverse_ (\h -> void (try (hClose h) :: IO (Either SomeException ()))) mStderr
        -- Ensure process is cleaned up
        void (try (waitForProcess ph) :: IO (Either SomeException ExitCode))
    
    -- Strict reading of handle contents
    hGetContents' :: Handle -> IO String
    hGetContents' h = go []
      where
        go acc = do
            eof <- hIsEOF h
            if eof
                then pure (concat (reverse acc))
                else do
                    line <- hGetLine h
                    go ((line ++ "\n") : acc)

-- | Build a full remote path from Remote + relative path.
-- Handles trailing-slash normalization internally.
remoteFilePath :: Remote -> FilePath -> String
remoteFilePath remote relPath =
    let base = remoteUrl remote
        -- Ensure exactly one separator between base and relative path
        base' = case base of
            [] -> base
            _  -> dropWhileEnd (== '/') base
    in base' ++ "/" ++ relPath

-- Dumb transport-level data types
data TransportItem = TransportItem
    { tiName  :: String
    , tiIsDir :: Bool
    } deriving (Show, Eq)

-- Internal type for JSON parsing
data RcloneItem = RcloneItem 
    { name   :: String
    , isDir  :: Bool 
    } deriving (Show, Generic)

-- We manually map lowercase Haskell fields to Capitalized JSON keys
instance Aeson.FromJSON RcloneItem where
    parseJSON = Aeson.withObject "RcloneItem" $ \v -> RcloneItem
        <$> v Aeson..: fromString "Name"
        <*> v Aeson..: fromString "IsDir"

-- Detailed copy result for domain-level error handling
data CopyResult 
    = CopySuccess 
    | CopyNotFound 
    | CopyNetworkError String 
    | CopyOtherError String
    deriving (Show, Eq)

-- | Result of an rclone check operation (--combined output).
data CheckResult = CheckResult
    { checkMatches     :: [FilePath]  -- '=' lines: identical on both sides
    , checkDiffers     :: [FilePath]  -- '*' lines: present on both but different
    , checkMissingDest :: [FilePath]  -- '+' lines: local only, not on remote
    , checkMissingSrc  :: [FilePath]  -- '-' lines: remote only, not on local
    , checkErrors      :: [FilePath]  -- '!' lines: error reading/hashing
    , checkExitCode    :: ExitCode
    , checkRawOutput   :: String      -- raw --combined output for .rgit/last-check.txt
    , checkStderr      :: String      -- stderr (for network/auth error messages)
    } deriving (Show)

-- Low-level rclone operations

-- | Copy local file to remote at relative path
copyToRemote :: FilePath -> Remote -> FilePath -> IO ExitCode
copyToRemote localPath remote relPath =
    let fullRemote = remoteFilePath remote relPath
    in (\(code, _, _) -> code) <$> readProcessWithExitCode "rclone" ["copyto", localPath, fullRemote] ""

-- | Copy file from remote (relative path) to local
copyFromRemote :: Remote -> FilePath -> FilePath -> IO ExitCode
copyFromRemote remote relPath localPath =
    let fullRemote = remoteFilePath remote relPath
    in (\(code, _, _) -> code) <$> readProcessWithExitCode "rclone" ["copyto", fullRemote, localPath] ""

-- | Copy from remote with detailed error classification
copyFromRemoteDetailed :: Remote -> FilePath -> FilePath -> IO CopyResult
copyFromRemoteDetailed remote relPath localPath = do
    let fullRemote = remoteFilePath remote relPath
    (code, _, err) <- readProcessWithExitCode "rclone" ["copyto", fullRemote, localPath] ""
    case code of
        ExitSuccess -> pure CopySuccess
        ExitFailure _ 
            | "directory not found" `isInfixOf` err || "object not found" `isInfixOf` err || "doesn't exist" `isInfixOf` err -> pure CopyNotFound
            | "no such host" `isInfixOf` err || "dial tcp" `isInfixOf` err -> pure (CopyNetworkError err)
            | otherwise -> pure (CopyOtherError err)

-- | Move a file on remote (both paths relative to remote root)
moveRemote :: Remote -> FilePath -> FilePath -> IO ExitCode
moveRemote remote srcRel destRel =
    let src = remoteFilePath remote srcRel
        dest = remoteFilePath remote destRel
    in (\(code, _, _) -> code) <$> readProcessWithExitCode "rclone" ["moveto", src, dest] ""

-- | Delete a file on remote (relative path)
deleteRemote :: Remote -> FilePath -> IO ExitCode
deleteRemote remote relPath =
    (\(code, _, _) -> code) <$> readProcessWithExitCode "rclone" ["deletefile", remoteFilePath remote relPath] ""

-- | Purge entire remote (no relative path — purges the remote root)
purgeRemote :: Remote -> IO ExitCode
purgeRemote remote =
    (\(code, _, _) -> code) <$> readProcessWithExitCode "rclone" ["purge", remoteUrl remote] ""

-- | Create directory on remote (relative path)
mkdirRemote :: Remote -> FilePath -> IO ExitCode
mkdirRemote remote relPath =
    (\(code, _, _) -> code) <$> readProcessWithExitCode "rclone" ["mkdir", remoteFilePath remote relPath] ""

-- | List remote directory as JSON (at remote root)
-- Returns (ExitCode, ByteString, String) where stdout is raw bytes for proper UTF-8 handling
listRemoteJson :: Remote -> Int -> IO (ExitCode, LBS.ByteString, String)
listRemoteJson remote maxDepth =
    readProcessBytes "rclone" ["lsjson", "--max-depth", show maxDepth, remoteUrl remote]

-- | List remote directory items (at remote root, parsed)
listRemoteItems :: Remote -> Int -> IO (Either String [TransportItem])
listRemoteItems remote maxDepth = do
    (code, outBytes, err) <- listRemoteJson remote maxDepth
    case code of
        ExitFailure _ -> 
            if "directory not found" `isInfixOf` err 
            then pure (Right [])  -- Empty directory
            else pure (Left err)  -- Network or other error
        ExitSuccess -> do
            case Aeson.decode outBytes :: Maybe [RcloneItem] of
                Nothing -> pure (Left "Failed to parse rclone JSON output")
                Just items -> pure (Right [TransportItem (name item) (isDir item) | item <- items])

-- | List remote recursively with hashes
-- Returns (ExitCode, ByteString, String) where stdout is raw bytes for proper UTF-8 handling
listRemoteJsonWithHash :: Remote -> IO (ExitCode, LBS.ByteString, String)
listRemoteJsonWithHash remote =
    readProcessBytes "rclone" ["lsjson", remoteUrl remote, "--hash", "--recursive"]

-- | Check local against remote with optional progress tracking
checkRemote :: FilePath -> Remote -> Maybe (IORef Int) -> IO CheckResult
checkRemote localPath remote mCounter = do
    let args = [ "check"
               , localPath
               , remoteUrl remote
               , "--combined", "-"
               , "--exclude", ".bit/**"
               ]
    case mCounter of
        Nothing -> do
            -- No progress tracking - use simple blocking version
            (code, out, err) <- readProcessWithExitCode "rclone" args ""
            let parsed = parseCombinedOutput out
            pure CheckResult
                { checkMatches     = parsed '='
                , checkDiffers     = parsed '*'
                , checkMissingDest = parsed '+'
                , checkMissingSrc  = parsed '-'
                , checkErrors      = parsed '!'
                , checkExitCode    = code
                , checkRawOutput   = out
                , checkStderr      = err
                }
        Just counter -> do
            -- Stream output and track progress
            let cp = (proc "rclone" args)
                    { std_out = CreatePipe
                    , std_err = CreatePipe
                    , std_in = Inherit
                    }
            bracket (createProcess cp) cleanup $ \(_, mStdout, mStderr, ph) -> do
                case (mStdout, mStderr) of
                    (Just hOut, Just hErr) -> do
                        -- CRITICAL: drain stderr concurrently to avoid pipe deadlock
                        -- When rclone checks large repos (especially cloud remotes like Google Drive),
                        -- it produces stderr output (rate limits, retries, transfer stats).
                        -- If we read stdout first, stderr pipe buffer can fill (~64KB on Windows)
                        -- causing rclone to block on stderr write while we block on stdout read.
                        asyncErr <- async (hGetContents' hErr)
                        -- Read stdout line by line, accumulating and counting
                        outLines <- readLinesWithProgress hOut counter
                        -- Now join stderr read
                        errOutput <- wait asyncErr
                        -- Wait for process to finish
                        code <- waitForProcess ph
                        let out = unlines outLines
                            parsed = parseCombinedOutput out
                        pure CheckResult
                            { checkMatches     = parsed '='
                            , checkDiffers     = parsed '*'
                            , checkMissingDest = parsed '+'
                            , checkMissingSrc  = parsed '-'
                            , checkErrors      = parsed '!'
                            , checkExitCode    = code
                            , checkRawOutput   = out
                            , checkStderr      = errOutput
                            }
                    _ -> error "checkRemote: failed to create pipes"
  where
    cleanup (_, mOut, mErr, ph) = do
        -- Try to close any handles that are still open
        traverse_ (\h -> void (try (hClose h) :: IO (Either SomeException ()))) mOut
        traverse_ (\h -> void (try (hClose h) :: IO (Either SomeException ()))) mErr
        -- Ensure process is cleaned up
        void (try (waitForProcess ph) :: IO (Either SomeException ExitCode))
    
    -- Read lines from handle, incrementing counter for each line
    readLinesWithProgress :: Handle -> IORef Int -> IO [String]
    readLinesWithProgress h counter = go []
      where
        go acc = do
            eof <- hIsEOF h
            if eof
                then pure (reverse acc)
                else do
                    line <- hGetLine h
                    unless (null line) $ modifyIORef' counter (+1)
                    go (line : acc)
    
    -- Strict reading of handle contents to avoid lazy IO issues
    hGetContents' :: Handle -> IO String
    hGetContents' h = go []
      where
        go acc = do
            eof <- hIsEOF h
            if eof
                then pure (concat (reverse acc))
                else do
                    line <- hGetLine h
                    go ((line ++ "\n") : acc)
    
    -- Parse "<symbol> <path>" lines; path is everything after first space.
    parseCombinedOutput :: String -> (Char -> [FilePath])
    parseCombinedOutput raw =
        let lines' = lines raw
            go sym = [ normalise path
                     | line <- lines'
                     , not (null line)
                     , let (symChar, path) = case span (/= ' ') line of
                               (s, ' ' : r) -> (s, r)
                               (s, _)       -> (s, "")
                     , not (null symChar)
                     , case symChar of
                         (c : _) -> c == sym
                         []      -> False
                     ]
        in go


```

---

## bit.cabal

**Path:** `bit.cabal`

*Build configuration — package metadata and dependencies.*

```cabal
cabal-version:      2.4
name:               bit
version:            0.1.0.0

executable bit
    main-is:          Bit.hs
    ghc-options:
        -threaded
        -rtsopts
        -with-rtsopts=-N
        -Wall
    other-modules:    Bit.AtomicWrite,
                      Bit.Commands,
                      Bit.Help,
                      Bit.Concurrency,
                      Bit.Conflict,
                      Bit.ConcurrentIO,
                      Bit.ConcurrentFileIO,
                      Bit.CopyProgress,
                      Bit.Device,
                      Bit.DevicePrompt,
                      Bit.Diff,
                      Bit.Process,
                      Bit.Progress,
                      Bit.Utils,
                      Bit.Plan,
                      Bit.Pipeline,
                      Bit.Remote,
                      Bit.Remote.Scan,
                      Bit.RemoteWorkspace,
                      Bit.Scan,
                      Bit.Types,
                      Bit.Verify,
                      Bit.Fsck,
                      Bit.Core,
                      Bit.Core.Helpers,
                      Bit.Core.Init,
                      Bit.Core.GitPassthrough,
                      Bit.Core.Fetch,
                      Bit.Core.Transport,
                      Bit.Core.Push,
                      Bit.Core.Pull,
                      Bit.Core.RemoteManagement,
                      Bit.Core.Verify,
                      Internal.Git,
                      Internal.Config,
                      Internal.ConfigFile,
                      Bit.Internal.Metadata,
                      Internal.Transport
    build-depends:    base16-bytestring ^>=1.0.2.0,
                      base,
                      async,
                      directory,
                      uuid ^>=1.3.13,
                      filepath,
                      time,
                      yaml,
                      bytestring,
                      cryptohash-sha256,
                      cryptohash-md5,
                      aeson,
                      process,
                      containers,
                      transformers,
                      text,
                      free,
                      mtl
    default-language: Haskell2010

executable generate-literate
    main-is:             GenerateLiterate.hs
    hs-source-dirs:      scripts
    ghc-options:         -Wall
    build-depends:       base, directory, filepath, process, bytestring, text
    default-language:    Haskell2010

test-suite cli
    type:                exitcode-stdio-1.0
    main-is:             RunCliTests.hs
    hs-source-dirs:      test
    ghc-options:         -Wall
    build-depends:       base, directory, filepath, process
    default-language:    Haskell2010

test-suite cli-fast
    type:                exitcode-stdio-1.0
    main-is:             RunCliTestsFast.hs
    hs-source-dirs:      test
    ghc-options:         -Wall
    build-depends:       base, directory, filepath, process
    default-language:    Haskell2010

test-suite device-prompt
    type:                exitcode-stdio-1.0
    main-is:             DevicePromptTests.hs
    hs-source-dirs:      test, .
    other-modules:       Bit.DevicePrompt
    ghc-options:         -Wall
    build-depends:       base,
                        directory,
                        tasty,
                        tasty-hunit
    default-language:    Haskell2010

test-suite pipeline
    type:                exitcode-stdio-1.0
    main-is:             PipelineSpec.hs
    hs-source-dirs:      test, .
    other-modules:       Bit.AtomicWrite,
                         Bit.Types,
                         Bit.Diff,
                         Bit.Plan,
                         Bit.Pipeline,
                         Bit.Utils,
                         Bit.Internal.Metadata,
                         Bit.Device,
                         Bit.Remote,
                         Internal.Git,
                         Internal.Config,
                         Internal.ConfigFile
    ghc-options:         -Wall
    build-depends:       base,
                         containers,
                         text,
                         tasty,
                         tasty-hunit,
                         tasty-quickcheck,
                         QuickCheck,
                         free,
                         transformers,
                         bytestring,
                         base16-bytestring ^>=1.0.2.0,
                         cryptohash-md5,
                         directory,
                         filepath,
                         process,
                         time,
                         uuid ^>=1.3.13
    default-language:    Haskell2010

test-suite generate-literate-docs
    type:                exitcode-stdio-1.0
    main-is:             GenerateLiterate.hs
    hs-source-dirs:      scripts
    ghc-options:         -Wall
    build-depends:       base, process, directory, filepath, bytestring, text
    build-tool-depends:  bit:generate-literate
    default-language:    Haskell2010

test-suite lint-tests
    type:                exitcode-stdio-1.0
    main-is:             LintTestFiles.hs
    hs-source-dirs:      test
    ghc-options:         -Wall
    build-depends:       base,
                         directory,
                         filepath,
                         tasty,
                         tasty-hunit
    default-language:    Haskell2010
```

---

## scripts/GenerateLiterate.hs

**Path:** `scripts/GenerateLiterate.hs`

*Source file.*

```haskell
-- | Generate three literate-programming Markdown files:
--   * bit-source-literate.md — bit.cabal + all .hs source files (excluding test/)
--   * bit-tests-literate.md  — everything under test/
--   * bit-docs-literate.md   — all .md files under docs/
--
-- After generation, stages them in git, commits, and pushes to origin.
--
-- Run: cabal run generate-literate

module Main where

import Control.Exception (SomeException, try)
import Control.Monad (forM_, when)
import Data.Char (toLower)
import Data.List (isPrefixOf, sort)
import System.Directory (doesDirectoryExist, doesFileExist, getCurrentDirectory, listDirectory, createDirectoryIfMissing)
import System.Exit (ExitCode(..))
import System.FilePath (takeDirectory, takeExtension, (</>))
import System.IO (IOMode (WriteMode), hPutStr, hPutStrLn, hSetEncoding, utf8, withFile)
import System.Process (callProcess, readProcessWithExitCode)
import qualified Data.ByteString as BS
import qualified Data.Text as T
import qualified Data.Text.Encoding as T

-- | Directories to skip.
excludedDirs :: [String]
excludedDirs = [".git", ".bit", "dist-newstyle", "dist", ".stack-work", ".vscode", ".history"]

-- | Ascend until we find bit.cabal.
findRepoRoot :: FilePath -> IO FilePath
findRepoRoot dir = do
  exists <- doesFileExist (dir </> "bit.cabal")
  if exists
    then pure dir
    else let parent = takeDirectory dir
         in  if parent == dir
               then fail "Could not find repo root (bit.cabal)"
               else findRepoRoot parent

-- | Subdirectories of test/ to skip (generated at runtime).
excludedTestDirs :: [String]
excludedTestDirs = ["work", "work_a", "work_b"]

-- | Files inside test/ to skip.
excludedTestFiles :: [String]
excludedTestFiles = [".bit-store"]

-- | Recursively collect all .hs files under @root@, relative to @root@.
-- Skips the test/ directory (handled separately by gatherTestFiles).
gatherHsFiles :: FilePath -> IO [FilePath]
gatherHsFiles root = go ""
  where
    go rel = do
      let full = if null rel then root else root </> rel
      entries <- listDirectory full
      fmap concat $ mapM (visit rel) entries

    visit rel name
      | name `elem` excludedDirs = pure []
      | name == "test"           = pure []  -- test/ handled separately
      | isPrefixOf "." name      = pure []
      | otherwise = do
          let relPath  = if null rel then name else rel </> name
              fullPath = root </> relPath
          isDir <- doesDirectoryExist fullPath
          if isDir
            then go relPath
            else pure [ relPath | takeExtension name == ".hs" ]

-- | Collect all files under test/, relative to @root@.
gatherTestFiles :: FilePath -> IO [FilePath]
gatherTestFiles root = go "test"
  where
    go rel = do
      let full = root </> rel
      exists <- doesDirectoryExist full
      if not exists then pure []
      else do
        entries <- listDirectory full
        fmap concat $ mapM (visit rel) entries

    visit rel name
      | name `elem` excludedTestDirs  = pure []
      | name `elem` excludedTestFiles = pure []
      | isPrefixOf "." name           = pure []
      | otherwise = do
          let relPath  = rel </> name
              fullPath = root </> relPath
          isDir <- doesDirectoryExist fullPath
          if isDir
            then go relPath
            else pure [relPath]

-- | Collect all .md files under docs/, relative to @root@.
gatherDocsFiles :: FilePath -> IO [FilePath]
gatherDocsFiles root = do
  let docsDir = root </> "docs"
  exists <- doesDirectoryExist docsDir
  if not exists then pure []
  else do
    entries <- listDirectory docsDir
    let mdFiles = [ "docs" </> name | name <- entries
                  , takeExtension name == ".md"
                  , not (isPrefixOf "." name) ]
    pure (sort mdFiles)

-- | Map file extension to fenced-code-block language tag.
getLang :: FilePath -> String
getLang path = case map toLower (takeExtension path) of
  ".hs"    -> "haskell"
  ".cabal" -> "cabal"
  ".sh"    -> "shell"
  ".md"    -> "markdown"
  ".yaml"  -> "yaml"
  ".yml"   -> "yaml"
  _        -> "text"  -- .test, .txt, etc.

-- | Brief explanation based on path.
getExplanation :: FilePath -> String
getExplanation rel
  | rel == "bit.cabal"                = "*Build configuration — package metadata and dependencies.*"
  | "Internal/" `isPrefixOf` rel       = "*Internal module — implementation details.*"
  | "bit/Internal/" `isPrefixOf` rel  = "*Internal module — implementation details.*"
  | rel == "bit.hs"                   = "*Entry point — main executable.*"
  | "bit/" `isPrefixOf` rel           = "*Core module — application logic.*"
  | "test/cli/" `isPrefixOf` rel       = "*CLI test — shell-based integration test.*"
  | "test/" `isPrefixOf` rel           = "*Test module.*"
  | "scripts/" `isPrefixOf` rel        = "*Build script.*"
  | "docs/" `isPrefixOf` rel           = "*Documentation file.*"
  | otherwise                          = "*Source file.*"

-- | Normalise backslashes to forward slashes.
toPosix :: FilePath -> FilePath
toPosix = map (\c -> if c == '\\' then '/' else c)

-- | Run a git command in the given directory. Returns True on success.
gitIn :: FilePath -> [String] -> IO Bool
gitIn dir args = do
  (code, _out, _err) <- readProcessWithExitCode "git" (["-C", dir] ++ args) ""
  pure (code == ExitSuccess)

main :: IO ()
main = do
  cwd  <- getCurrentDirectory
  root <- findRepoRoot cwd

  hsFiles   <- gatherHsFiles root
  testFiles <- gatherTestFiles root
  docsFiles <- gatherDocsFiles root

  let sourceFiles = sort $ "bit.cabal" : hsFiles
      testSorted  = sort testFiles
      docsSorted  = sort docsFiles

  let litDir = root </> "literate-output"
  createDirectoryIfMissing True litDir

  -- Source files
  let sourcePath = litDir </> "bit-source-literate.md"
  writeDocument sourcePath
    "bit — Literate Programming Document"
    "This document contains all Haskell source files and the cabal\n\
    \file for the bit project, presented in literate-programming style."
    root sourceFiles
  putStrLn $ "Wrote " ++ sourcePath ++ " (" ++ show (length sourceFiles) ++ " files)"

  -- Test files
  let testsPath = litDir </> "bit-tests-literate.md"
  writeDocument testsPath
    "bit — Tests (Literate Programming Document)"
    "This document contains all test files for the bit project:\n\
    \Haskell test modules, shell-based integration tests, and test infrastructure."
    root testSorted
  putStrLn $ "Wrote " ++ testsPath ++ " (" ++ show (length testSorted) ++ " files)"

  -- Docs files
  let docsPath = litDir </> "bit-docs-literate.md"
  writeDocument docsPath
    "bit — Documentation (Literate Programming Document)"
    "This document contains all documentation files for the bit project:\n\
    \specifications, refactoring plans, and design documents."
    root docsSorted
  putStrLn $ "Wrote " ++ docsPath ++ " (" ++ show (length docsSorted) ++ " files)"

  -- Git: stage, commit, push (gracefully handle failures)
  putStrLn "Committing literate output..."
  addResult <- try (callProcess "git" ["-C", litDir, "add",
    "bit-source-literate.md",
    "bit-tests-literate.md",
    "bit-docs-literate.md"]) :: IO (Either SomeException ())
  case addResult of
    Left _ -> putStrLn "Warning: git add failed (continuing anyway)"
    Right () -> do
      -- Only commit if there are staged changes
      -- git diff --cached --quiet returns success (0) when there are NO changes
      noChanges <- gitIn litDir ["diff", "--cached", "--quiet"]
      if noChanges
        then putStrLn "No changes to literate output."
        else do
          commitResult <- try (callProcess "git" ["-C", litDir, "commit", "-m", "Update literate output"]) :: IO (Either SomeException ())
          case commitResult of
            Left _ -> putStrLn "Warning: git commit failed (continuing anyway)"
            Right () -> do
              putStrLn "Pushing to origin..."
              pushResult <- try (callProcess "git" ["-C", litDir, "push", "origin"]) :: IO (Either SomeException ())
              case pushResult of
                Left _ -> putStrLn "Warning: git push failed (continuing anyway)"
                Right () -> putStrLn "Done."

-- | Write a single literate document.
writeDocument :: FilePath -> String -> String -> FilePath -> [FilePath] -> IO ()
writeDocument outputPath title description root files =
  withFile outputPath WriteMode $ \h -> do
    hSetEncoding h utf8
    let write = hPutStrLn h

    write $ "# " ++ title
    write ""
    write description
    write ""
    write "---"
    write ""

    forM_ files $ \rel -> do
      let posix = toPosix rel
          lang  = getLang rel
          expl  = getExplanation rel
          full  = root </> rel

      write $ "## " ++ posix
      write ""
      write $ "**Path:** `" ++ posix ++ "`"
      write ""
      write expl
      write ""
      write $ "```" ++ lang

      -- Use strict ByteString reading to avoid lazy IO on Windows
      result <- try (BS.readFile full) :: IO (Either SomeException BS.ByteString)
      case result of
        Right bs ->
          case T.decodeUtf8' bs of
            Right content -> do
              let contentStr = T.unpack content
              hPutStr h contentStr
              when (not (T.null content) && T.last content /= '\n') $
                hPutStrLn h ""
            Left err ->
              hPutStrLn h $ "-- Error decoding UTF-8: " ++ show err
        Left err ->
          hPutStrLn h $ "-- Error reading file: " ++ show err

      write "```"
      write ""
      write "---"
      write ""
```

---

