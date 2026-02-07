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
import System.IO (hSetEncoding, stdout, stderr, utf8, hIsTerminalDevice)
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
        callCommand "chcp 65001 > nul 2>&1" `catch` \(_ :: SomeException) -> return ()
    
    -- Set UTF-8 for stdout/stderr to properly display Unicode characters
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
        removeFile tempPath `catch` \(_ :: IOException) -> return ())
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
  removeFile dest `catch` \(_ :: IOException) -> return ()
  renameFile src dest
renameWithRetry src dest n = do
  result <- try (renameFile src dest)
  case result of
    Right () -> return ()
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
import Bit.Types (BitEnv(..), runBitM)
import qualified Bit.Scan as Scan  -- Only for the pre-scan in runCommand
import Bit.Remote (getDefaultRemote, resolveRemote)
import Bit.Utils (atomicWriteFileStr)
import Bit.Concurrency (Concurrency(..))
import qualified Bit.RemoteWorkspace as RemoteWorkspace
import System.Environment (getArgs)
import System.Exit (ExitCode(..), exitWith)
import System.FilePath ((</>))
import System.IO (hPutStrLn, stderr)
import System.Process (rawSystem)
import Control.Monad (when, unless, void)
import qualified System.Directory as Dir
import qualified Internal.Git as Git
import qualified Internal.Transport as Transport
import Data.List (isPrefixOf)
import Control.Exception (catch, SomeException)
-- Strict IO imports to avoid Windows file locking issues
import qualified Data.ByteString as BS
import qualified Data.Text as T
import Data.Text.Encoding (decodeUtf8')

run :: IO ()
run = do
    args <- getArgs
    case args of
        [] -> hPutStrLn stderr $ unlines
            [ "Usage: bit <command> [options]"
            , ""
            , "Commands:"
            , "  init                           Initialize a new bit repository"
            , "  status                         Show working tree status"
            , "  add <path>                     Add file contents to metadata"
            , "  commit -m <msg>                Record changes to the repository"
            , "  log                            Show commit history"
            , "  diff                           Show changes"
            , "  restore [options] [--] <path>  Restore working tree files"
            , "  checkout [options] -- <path>   Checkout files from index"
            , ""
            , "  push [-u|--set-upstream] [<remote>] [--skip-verify]"
            , "                                 Push to remote"
            , "  pull [<remote>] [options]      Pull from remote"
            , "      --accept-remote            Accept remote state as truth"
            , "      --manual-merge             Manual conflict resolution"
            , "      --skip-verify              Skip proof of possession check"
            , "  fetch [<remote>]               Fetch metadata from remote"
            , ""
            , "  remote add <name> <url>        Add a remote"
            , "  remote show [<name>]           Show remote information"
            , "  remote check [<name>]          Check remote connectivity"
            , ""
            , "  verify [--remote]              Verify file integrity"
            , "  fsck                           Full integrity check"
            , "  merge --continue|--abort       Continue or abort merge"
            , "  branch --unset-upstream        Unset upstream tracking"
            , ""
            , "Remote-targeted commands:"
            , "  @<remote> init                 Scan remote and build metadata workspace"
            , "  @<remote> add <path>           Stage files in remote workspace"
            , "  @<remote> commit -m <msg>      Commit and push metadata bundle to remote"
            , "  @<remote> status               Show remote workspace status"
            , "  @<remote> log                  Show remote workspace history"
            ]
        _  -> do
            -- Check for @<remote> prefix
            let (mRemoteTarget, remainingArgs) = extractRemoteTarget args
            case mRemoteTarget of
                Nothing -> runCommand remainingArgs       -- normal local execution
                Just remoteName -> runRemoteCommand remoteName remainingArgs

-- | Extract @<remote> from args. Returns (Just remoteName, rest) or (Nothing, args).
extractRemoteTarget :: [String] -> (Maybe String, [String])
extractRemoteTarget [] = (Nothing, [])
extractRemoteTarget (arg:rest)
    | "@" `isPrefixOf` arg && length arg > 1 = (Just (drop 1 arg), rest)
    | otherwise = (Nothing, arg:rest)

-- | Execute a command in the context of a remote workspace
runRemoteCommand :: String -> [String] -> IO ()
runRemoteCommand remoteName args = do
    cwd <- Dir.getCurrentDirectory
    bitExists <- Dir.doesDirectoryExist (cwd </> ".bit")
    unless bitExists $ do
        hPutStrLn stderr "fatal: not a bit repository (or any of the parent directories): .bit"
        exitWith (ExitFailure 1)

    -- Resolve the remote
    mRemote <- resolveRemote cwd remoteName
    case mRemote of
        Nothing -> do
            hPutStrLn stderr $ "fatal: remote '" ++ remoteName ++ "' not found."
            exitWith (ExitFailure 1)
        Just remote -> do
            let wsPath = RemoteWorkspace.remoteWorkspacePath cwd remoteName

            case args of
                ["init"] ->
                    RemoteWorkspace.initRemoteWorkspace cwd remote remoteName

                ("add":paths) -> do
                    -- Stage files in the remote workspace git repo
                    wsExists <- Dir.doesDirectoryExist (wsPath </> ".git")
                    unless wsExists $ do
                        hPutStrLn stderr $ "fatal: remote workspace not initialized. Run 'bit @" ++ remoteName ++ " init' first."
                        exitWith (ExitFailure 1)
                    -- git add in the workspace
                    code <- rawSystem "git" (["-C", wsPath, "add"] ++ paths)
                    exitWith code

                ("commit":commitArgs) -> do
                    wsExists <- Dir.doesDirectoryExist (wsPath </> ".git")
                    unless wsExists $ do
                        hPutStrLn stderr $ "fatal: remote workspace not initialized."
                        exitWith (ExitFailure 1)
                    -- git commit in the workspace
                    code <- rawSystem "git" (["-C", wsPath, "commit"] ++ commitArgs)
                    when (code == ExitSuccess) $ do
                        -- Create bundle and push to remote
                        putStrLn "Pushing metadata bundle to remote..."
                        let bundlePath = wsPath </> ".git" </> "bit.bundle"
                        bCode <- rawSystem "git" ["-C", wsPath, "bundle", "create", bundlePath, "main"]
                        when (bCode == ExitSuccess) $ do
                            rCode <- Transport.copyToRemote bundlePath remote ".bit/bit.bundle"
                            if rCode == ExitSuccess
                                then putStrLn "Remote is now a bit repository."
                                else hPutStrLn stderr "Error uploading bundle to remote."
                            -- Cleanup bundle
                            Dir.removeFile bundlePath `catch` (\(_ :: SomeException) -> return ())
                    exitWith code

                ("status":rest) -> do
                    wsExists <- Dir.doesDirectoryExist (wsPath </> ".git")
                    unless wsExists $ do
                        hPutStrLn stderr $ "fatal: remote workspace not initialized."
                        exitWith (ExitFailure 1)
                    void $ rawSystem "git" (["-C", wsPath, "status"] ++ rest)

                ("log":rest) -> do
                    wsExists <- Dir.doesDirectoryExist (wsPath </> ".git")
                    unless wsExists $ do
                        hPutStrLn stderr $ "fatal: remote workspace not initialized."
                        exitWith (ExitFailure 1)
                    void $ rawSystem "git" (["-C", wsPath, "log"] ++ rest)

                _ -> do
                    hPutStrLn stderr $ "error: command not supported in remote context: " ++ unwords args
                    hPutStrLn stderr "Supported: init, add, commit, status, log"
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
        let content = case decodeUtf8' bs of
              Left _ -> ""
              Right txt -> T.unpack txt
            normalizedLines = filter (not . null) $
              map (trim . filter (/= '\r')) (lines content)
        atomicWriteFileStr dest (unlines normalizedLines)
    
    removeStaleGitignore :: FilePath -> IO ()
    removeStaleGitignore dest = do
        destExists <- Dir.doesFileExist dest
        when destExists $ Dir.removeFile dest
    
    trim :: String -> String
    trim = dropWhile (== ' ') . reverse . dropWhile (== ' ') . reverse

runCommand :: [String] -> IO ()
runCommand args = do
    let isForce = "--force" `elem` args || "-f" `elem` args
    let isForceWithLease = "--force-with-lease" `elem` args
    let isSequential = "--sequential" `elem` args
    let isSkipVerify = "--skip-verify" `elem` args
    when (isForce && isForceWithLease) $ do
        hPutStrLn stderr "fatal: Cannot use both --force and --force-with-lease"
        exitWith (ExitFailure 1)
    let cmd = filter (`notElem` ["--force", "-f", "--force-with-lease", "--sequential", "--skip-verify"]) args

    cwd <- Dir.getCurrentDirectory
    bitExists <- Dir.doesDirectoryExist (cwd </> ".bit")

    -- Lightweight env (no scan) — for read-only commands
    let baseEnv = do
            mRemote <- getDefaultRemote cwd
            return $ BitEnv cwd [] mRemote isForce isForceWithLease isSkipVerify

    -- Full env (scan + bitignore sync + metadata write) — for write commands
    let scannedEnv = do
            syncBitignoreToIndex cwd
            localFiles <- Scan.scanWorkingDir cwd
            Scan.writeMetadataFiles cwd localFiles
            mRemote <- getDefaultRemote cwd
            return $ BitEnv cwd localFiles mRemote isForce isForceWithLease isSkipVerify

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
        ["fsck"]                        -> Bit.fsck cwd (if isSequential then Sequential else Parallel 0)
        ["merge", "--abort"]            -> Bit.mergeAbort
        ["branch", "--unset-upstream"]  -> Bit.unsetUpstream

        -- ── Lightweight env (no scan) ────────────────────────
        ("log":rest)                    -> Bit.log rest >>= exitWith
        ("ls-files":rest)               -> Bit.lsFiles rest >>= exitWith
        ["remote", "show"]              -> runBase $ Bit.remoteShow Nothing
        ["remote", "show", name]        -> runBaseWithRemote name $ Bit.remoteShow (Just name)
        ["remote", "check"]             -> runBase $ Bit.remoteCheck Nothing
        ["remote", "check", name]       -> runBaseWithRemote name $ Bit.remoteCheck (Just name)
        ["verify"]                      -> runBase $ Bit.verify False (if isSequential then Sequential else Parallel 0)
        ["verify", "--remote"]          -> runBase $ Bit.verify True (if isSequential then Sequential else Parallel 0)

        -- ── Full scanned env (needs working directory state) ─
        ("add":rest)                    -> do
            _ <- scannedEnv
            Bit.add rest >>= exitWith
        ("commit":rest)                 -> do
            _ <- scannedEnv
            Bit.commit rest >>= exitWith
        ("diff":rest)                   -> do
            _ <- scannedEnv
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
        ["pull"]                        -> runScanned $ Bit.pull Bit.defaultPullOptions { Bit.pullSkipVerify = isSkipVerify }
        ["pull", name]                  -> runScannedWithRemote name $ Bit.pull Bit.defaultPullOptions { Bit.pullSkipVerify = isSkipVerify }
        ["pull", "--accept-remote"]     -> runScanned $ Bit.pull Bit.defaultPullOptions { Bit.pullAcceptRemote = True, Bit.pullSkipVerify = isSkipVerify }
        ["pull", "--manual-merge"]      -> runScanned $ Bit.pull Bit.defaultPullOptions { Bit.pullManualMerge = True, Bit.pullSkipVerify = isSkipVerify }
        ["pull", name, "--accept-remote"] -> runScannedWithRemote name $ Bit.pull Bit.defaultPullOptions { Bit.pullAcceptRemote = True, Bit.pullSkipVerify = isSkipVerify }
        ["pull", "--accept-remote", name] -> runScannedWithRemote name $ Bit.pull Bit.defaultPullOptions { Bit.pullAcceptRemote = True, Bit.pullSkipVerify = isSkipVerify }
        ["pull", name, "--manual-merge"] -> runScannedWithRemote name $ Bit.pull Bit.defaultPullOptions { Bit.pullManualMerge = True, Bit.pullSkipVerify = isSkipVerify }
        ["pull", "--manual-merge", name] -> runScannedWithRemote name $ Bit.pull Bit.defaultPullOptions { Bit.pullManualMerge = True, Bit.pullSkipVerify = isSkipVerify }
        
        -- fetch
        ["fetch"]                       -> runScanned Bit.fetch
        ["fetch", name]                 -> runScannedWithRemote name Bit.fetch
        
        ["merge", "--continue"]         -> runScanned Bit.mergeContinue
        _                               -> hPutStrLn stderr "Unknown command."
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
ioConcurrency = do
    caps <- getNumCapabilities
    return (max 4 (caps * 4))

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
networkConcurrency = do
    caps <- getNumCapabilities
    return (min 8 (max 2 (caps * 2)))

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

import Prelude (FilePath, Maybe(..), Either(..), pure, ($), (.))
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
  result <- try (BS.readFile path)
  pure $ case result of
    Left (_ :: SomeException) -> Nothing
    Right bs -> Just bs

-- | Read file as UTF-8, returning 'Nothing' on any error (including invalid UTF-8).
readFileUtf8Maybe :: MonadIO m => FilePath -> m (Maybe T.Text)
readFileUtf8Maybe path = liftIO $ do
  result <- try (BS.readFile path)
  pure $ case result of
    Left (_ :: SomeException) -> Nothing
    Right bs -> case T.decodeUtf8' bs of
      Left _ -> Nothing
      Right t -> Just t

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
readFileMaybeC path = UnsafeConcurrentIO $ do
  result <- try (BS.readFile path)
  pure $ case result of
    Left (_ :: SomeException) -> Nothing
    Right bs -> Just bs

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

module Bit.Conflict
  ( Resolution(..)
  , ConflictInfo(..)
  , resolveConflict
  , resolveAll
  , getConflictedFilesE
  , parseConflictInfo
  ) where

import Data.Char (toLower, isSpace)
import Data.List (isPrefixOf)
import Data.Maybe (mapMaybe)
import Control.Monad (void, when)
import System.Exit (ExitCode(..))
import qualified Internal.Git as Git
import System.IO (hPutStrLn, stderr, hFlush, stdout)
import Bit.Internal.Metadata (MetaContent(..), parseMetadata, displayHash)

-- | A conflict resolution choice: keep local or take remote.
data Resolution = KeepLocal | TakeRemote
  deriving (Show, Eq)

-- | Conflict type (mirrors Internal.Git.ConflictType but doesn't depend on IO module).
data ConflictInfo
  = ContentConflict FilePath   -- both modified
  | ModifyDelete FilePath Bool -- True = deleted in HEAD (ours)
  | AddAdd FilePath            -- both added different
  deriving (Show, Eq)

-- | Get list of conflicted files
getConflictedFilesE :: IO [FilePath]
getConflictedFilesE = do
  (code, out, _) <- Git.runGitWithOutput ["diff", "--name-only", "--diff-filter=U"]
  return $ if code /= ExitSuccess then [] else filter (not . null) (lines out)

-- | Detect conflict type
getConflictInfoE :: FilePath -> IO ConflictInfo
getConflictInfoE path = do
  (_, out, _) <- Git.runGitWithOutput ["ls-files", "-u", "--", path]
  return (parseConflictInfo path out)

-- | Pure parsing of `git ls-files -u` output into ConflictInfo.
parseConflictInfo :: FilePath -> String -> ConflictInfo
parseConflictInfo path out =
  let stageNum line = case reverse (words (takeWhile (/= '\t') line)) of
        (s:_) | s `elem` ["1","2","3"] -> Just (read s :: Int)
        _ -> Nothing
      stageNums = mapMaybe stageNum (lines out)
      has1 = 1 `elem` stageNums
      has2 = 2 `elem` stageNums
      has3 = 3 `elem` stageNums
  in if has2 && has3 && has1 then ContentConflict path
     else if has2 && has3 && not has1 then AddAdd path
     else if has2 && not has3 then ModifyDelete path False
     else if has3 && not has2 then ModifyDelete path True
     else ContentConflict path

-- | Print a conflict type announcement (git-style message).
announceConflict :: ConflictInfo -> IO ()
announceConflict (ContentConflict path) =
  putStrLn $ "CONFLICT (content): Merge conflict in " ++ path
announceConflict (ModifyDelete path True) =
  putStrLn $ "CONFLICT (modify/delete): " ++ path ++ " deleted in HEAD and modified in origin/main"
announceConflict (ModifyDelete path False) =
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
normalize = map toLower . trim
  where trim = f . f
        f = reverse . dropWhile isSpace

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
    return SyncProgress
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
                            then return ()  -- EOF
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
    let shouldShowProgress = isTTY && spFilesTotal progress > 5
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
        
        let filesPct = if spFilesTotal progress > 0
                       then (filesCompleted * 100) `div` spFilesTotal progress
                       else 0
        
        -- Show aggregate progress
        let progressLine = "Syncing files: " ++ show filesCompleted ++ "/" ++ show (spFilesTotal progress) 
                         ++ " files, " ++ formatBytes bytesCopied
                         ++ if totalBytes > 0
                            then " / " ++ formatBytes totalBytes ++ " (" ++ show filesPct ++ "%)"
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
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE ScopedTypeVariables #-}
{-# LANGUAGE TypeApplications #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE DuplicateRecordFields #-}
{-# LANGUAGE OverloadedRecordDot #-}
{-# LANGUAGE LambdaCase #-}

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
    , verify
    , fsck

      -- Remote management
    , remoteAdd
    , remoteShow
    , remoteCheck

      -- Merge management
    , mergeContinue
    , mergeAbort

      -- Branch management
    , unsetUpstream

      -- Types re-exported for Commands.hs
    , PullOptions(..)
    , defaultPullOptions
    ) where

import qualified Data.List as List
import Data.List (isPrefixOf)
import qualified System.Directory as Dir
import System.Directory (copyFile, removeFile, createDirectoryIfMissing, removeDirectory, listDirectory, doesDirectoryExist)
import System.FilePath ((</>), normalise, takeDirectory)
import Control.Monad (when, unless, void, forM, forM_)
import System.Exit (ExitCode(..), exitWith)
import qualified Data.Map as Map
import qualified Internal.Git as Git
import qualified Internal.Transport as Transport
import Internal.Config (bitDir, fetchedBundle, bitIndexPath, bitDevicesDir, bitRemotesDir, bundleCwdPath, fromCwdPath, BundleName(..))
import qualified Bit.Internal.Metadata as Metadata
import Bit.Internal.Metadata (MetaContent(..), displayHash, serializeMetadata)
import Data.Char (isSpace)
import qualified Bit.Scan as Scan
import qualified Bit.Pipeline as Pipeline
import qualified Bit.Verify as Verify
import qualified Bit.Fsck as Fsck
import Bit.Plan (RcloneAction(..))
import Bit.Concurrency (Concurrency(..), runConcurrentlyBounded)
import Control.Concurrent (getNumCapabilities)
import Bit.Types
import qualified Bit.Remote.Scan as Remote.Scan
import Control.Monad.Trans.Reader (asks)
import Control.Monad.IO.Class (liftIO)
import Control.Monad.Trans.Class (lift)
import Control.Exception (try, throwIO, SomeException, IOException)
import System.IO (stderr, hPutStrLn, hIsTerminalDevice)
import System.Process (readProcessWithExitCode)
import Data.Maybe (fromMaybe, maybeToList, mapMaybe)
import Bit.Utils (toPosix, filterOutBitPaths, atomicWriteFileStr)
-- Strict IO imports to avoid Windows file locking issues
import qualified Data.ByteString as BS
import qualified Data.Text as T
import Data.Text.Encoding (decodeUtf8')
import qualified Bit.Device as Device
import qualified Bit.DevicePrompt as DevicePrompt
import qualified Bit.Conflict as Conflict
import Bit.Remote (Remote, remoteName, displayRemote, resolveRemote, remoteUrl, RemoteState(..), FetchResult(..))
import Prelude hiding (init, log)
import Control.Exception (bracket)
import qualified Bit.CopyProgress as CopyProgress
import Bit.CopyProgress (SyncProgress)
import Data.IORef (IORef, newIORef, readIORef, writeIORef)
import Control.Concurrent (forkIO, threadDelay, killThread)
import Control.Exception (finally)
import Bit.Progress (reportProgress, clearProgress)

-- ============================================================================
-- Types
-- ============================================================================

data PullOptions = PullOptions
    { pullAcceptRemote :: Bool
    , pullManualMerge :: Bool
    , pullSkipVerify :: Bool
    } deriving (Show)

defaultPullOptions :: PullOptions
defaultPullOptions = PullOptions False False False

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

rm :: [String] -> IO ExitCode
rm args = Git.runGitRaw ("rm" : args)

mv :: [String] -> IO ExitCode
mv args = Git.runGitRaw ("mv" : args)

branch :: [String] -> IO ExitCode
branch args = Git.runGitRaw ("branch" : args)

merge :: [String] -> IO ExitCode
merge args = Git.runGitRaw ("merge" : args)

-- ============================================================================
-- Plain IO functions (no env needed)
-- ============================================================================

init :: IO ()
init = initializeRepo

fsck :: FilePath -> Concurrency -> IO ()
fsck = Fsck.doFsck

mergeAbort :: IO ()
mergeAbort = doMergeAbort

unsetUpstream :: IO ()
unsetUpstream = void Git.unsetBranchUpstream

remoteAdd :: String -> String -> IO ()
remoteAdd = addRemote

-- ============================================================================
-- Plain IO functions (no env needed)
-- ============================================================================

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
        let safePath = map (\c -> if c == '\\' then '/' else c) absIndex
        void $ readProcessWithExitCode "git" ["config", "--global", "--add", "safe.directory", safePath] ""

    -- 3a. Create .git/bundles directory for storing bundle files
    Dir.createDirectoryIfMissing True (targetBitGitDir </> "bundles")

    -- 4. Configure default branch name to "main" (for the repo we just created)
    -- This affects future branch operations in this repo
    void $ Git.runGitAt targetBitIndexPath ["config", "init.defaultBranch", "main"]
    
    -- 4a. Configure core.quotePath to false (display Unicode filenames properly)
    void $ Git.runGitAt targetBitIndexPath ["config", "core.quotePath", "false"]
    
    -- 5. Rename the initial branch to "main" if it's "master"
    -- Git init creates "master" by default, so we rename it
    (code, _, _) <- Git.runGitAt targetBitIndexPath ["branch", "-m", "master", "main"]
    when (code /= ExitSuccess) $
        -- If rename failed (e.g., no commits yet), that's okay - first commit will use "main"
        return ()

    -- 6. Create other .bit subdirectories (index already created above)
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

    -- 5b. Merge driver: prevent Git from writing conflict markers; bit resolves whole-file only
    -- Use .bit/index/.git/info/attributes instead of .gitattributes in the working tree
    -- This way it doesn't conflict with files from the remote on first pull
    -- Also disable text/CRLF conversion (-text) to prevent spurious "modified" status
    void $ Git.runGitAt targetBitIndexPath ["config", "merge.bit-metadata.name", "bit metadata"]
    void $ Git.runGitAt targetBitIndexPath ["config", "merge.bit-metadata.driver", "false"]
    Dir.createDirectoryIfMissing True (targetBitGitDir </> "info")
    atomicWriteFileStr (targetBitGitDir </> "info" </> "attributes") "* merge=bit-metadata -text\n"

    -- 5c. The .git/info/exclude file is created in Commands.hs from .bitignore
    -- (this happens before each scan, not during init)

    -- Note: We do NOT create an initial commit here.
    -- This keeps the repo empty until first real commit or pull.
    -- On first pull, we simply checkout the remote's history (no merge needed).

addRemote :: String -> String -> IO ()
addRemote name pathOrUrl = do
    cwd <- Dir.getCurrentDirectory
    Dir.createDirectoryIfMissing True bitDevicesDir
    Dir.createDirectoryIfMissing True bitRemotesDir
    pathType <- Device.classifyRemotePath pathOrUrl
    case pathType of
        Device.CloudRemote url -> do
            Device.writeRemoteFile cwd name (Device.TargetCloud url)
            void $ Git.addRemote name url
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
        exitWith (ExitFailure 1)
    volRoot <- Device.getVolumeRoot absPath
    let relPath = Device.getRelativePath volRoot absPath
    mStoreUuid <- Device.readBitStore volRoot
    mExistingDevice <- case mStoreUuid of
        Just u -> Device.findDeviceByUuid cwd u
        Nothing -> return Nothing
    result <- try @IOException $ case (mStoreUuid, mExistingDevice) of
        (Just _u, Just dev) -> do
            putStrLn $ "Using existing device '" ++ dev ++ "'."
            _mInfo <- Device.readDeviceFile cwd dev
            Device.writeRemoteFile cwd name (Device.TargetDevice dev relPath)
            putStrLn $ "Remote '" ++ name ++ "' → " ++ dev ++ ":" ++ relPath
            putStrLn $ "(using existing device '" ++ dev ++ "')"
            return ()
        (Just u, Nothing) -> do
            mLabel <- Device.getVolumeLabel volRoot
            deviceName' <- promptDeviceName cwd volRoot mLabel
            storeType' <- Device.detectStorageType volRoot
            mSerial <- case storeType' of
                Device.Physical -> Device.getHardwareSerial volRoot
                Device.Network -> return Nothing
            Device.writeDeviceFile cwd deviceName' (Device.DeviceInfo u storeType' mSerial)
            Device.writeRemoteFile cwd name (Device.TargetDevice deviceName' relPath)
            putStrLn $ "Remote '" ++ name ++ "' → " ++ deviceName' ++ ":" ++ relPath
            putStrLn $ "Device '" ++ deviceName' ++ "' registered (" ++ (case storeType' of Device.Physical -> "physical"; Device.Network -> "network") ++ ")."
            return ()
        (Nothing, _) -> do
            mLabel <- Device.getVolumeLabel volRoot
            deviceName' <- promptDeviceName cwd volRoot mLabel
            u <- Device.generateStoreUuid
            Device.writeBitStore volRoot u
            storeType' <- Device.detectStorageType volRoot
            mSerial <- case storeType' of
                Device.Physical -> Device.getHardwareSerial volRoot
                Device.Network -> return Nothing
            Device.writeDeviceFile cwd deviceName' (Device.DeviceInfo u storeType' mSerial)
            Device.writeRemoteFile cwd name (Device.TargetDevice deviceName' relPath)
            putStrLn $ "Remote '" ++ name ++ "' → " ++ deviceName' ++ ":" ++ relPath
            putStrLn $ "Device '" ++ deviceName' ++ "' registered (" ++ (case storeType' of Device.Physical -> "physical"; Device.Network -> "network") ++ ")."
            return ()
    case result of
        Right () -> return ()
        Left _err -> do
            -- Cannot create .bit-store at volume root (e.g. permission denied on C:\)
            -- Fall back to path-based storage for local directories
            Device.writeRemoteFile cwd name (Device.TargetLocalPath absPath)
            void $ Git.addRemote name absPath
            putStrLn $ "Remote '" ++ name ++ "' added (" ++ absPath ++ ")."

doMergeAbort :: IO ()
doMergeAbort = do
    cwd <- Dir.getCurrentDirectory
    let conflictsDir = cwd </> ".bit" </> "conflicts"
    
    -- Abort git merge
    code <- Git.mergeAbort
    if code /= ExitSuccess
        then do
            hPutStrLn stderr "error: no merge in progress."
            exitWith (ExitFailure 1)
        else do
            putStrLn "Merge aborted. Your working tree is unchanged."
            
            -- Clean up conflict directories
            conflictsExist <- Dir.doesDirectoryExist conflictsDir
            when conflictsExist $ do
                removeDirectoryRecursive conflictsDir
                putStrLn "Conflict directories cleaned up."

-- | Remove directory recursively (helper function).
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
-- Core operations (full business logic)
-- ============================================================================

push :: BitM ()
push = withRemote $ \remote -> do
    cwd <- asks envCwd
    force <- asks envForce
    skipVerify <- asks envSkipVerify
    
    -- NEW: Proof of possession — verify local before pushing
    unless (force || skipVerify) $ do
        liftIO $ putStrLn "Verifying local files..."
        (fileCount, issues) <- liftIO $ Verify.verifyLocal cwd Nothing (Parallel 0)
        if null issues
            then liftIO $ putStrLn $ "Verified " ++ show fileCount ++ " files. All match metadata."
            else do
                liftIO $ hPutStrLn stderr $ "error: Working tree does not match metadata (" ++ show (length issues) ++ " issues)."
                liftIO $ mapM_ (printVerifyIssue (\s -> s)) issues  -- full hash, no truncation
                liftIO $ hPutStrLn stderr "hint: Run 'bit verify' to see all mismatches."
                liftIO $ hPutStrLn stderr "hint: Run 'bit add' to update metadata, or 'bit restore' to restore files."
                liftIO $ hPutStrLn stderr "hint: Run 'bit push --force' to push anyway (unsafe)."
                liftIO $ exitWith (ExitFailure 1)
    
    -- Determine if this is a filesystem or cloud remote
    mTarget <- liftIO $ getRemoteTargetType cwd (remoteName remote)
    case mTarget of
        Just (Device.TargetDevice _ _) -> liftIO $ filesystemPush cwd remote
        Just (Device.TargetLocalPath _) -> liftIO $ filesystemPush cwd remote
        _ -> cloudPush remote  -- Cloud remote or no target info (use cloud flow)

-- | Push to a cloud remote (original flow, unchanged).
cloudPush :: Remote -> BitM ()
cloudPush remote = do
    force <- asks envForce
    liftIO $ putStrLn $ "Inspecting remote: " ++ displayRemote remote
    state <- liftIO $ classifyRemoteState remote

    case state of
        StateEmpty -> do
            liftIO $ putStrLn "Remote is empty. Initializing..."
            syncRemoteFiles
            liftIO $ pushBundle remote
            updateLocalBundleAfterPush

        StateValidRgit -> do
            liftIO $ putStrLn "Remote is a bit repo. Checking history..."
            fetchResult <- liftIO $ fetchBundle remote
            case fetchResult of
                BundleFound bPath -> do
                    let fetchedPath = fromCwdPath (bundleCwdPath fetchedBundle)
                    liftIO $ copyFile bPath fetchedPath
                    liftIO $ safeRemove bPath
                    processExistingRemote
                _ -> liftIO $ hPutStrLn stderr "Error: Remote .bit found but metadata is missing."

        StateNonRgitOccupied samples -> do
            if force
                then do
                    liftIO $ hPutStrLn stderr "Warning: --force used. Overwriting non-bit remote..."
                    syncRemoteFiles
                    liftIO $ pushBundle remote
                    updateLocalBundleAfterPush
                else do
                    liftIO $ hPutStrLn stderr "-------------------------------------------------------"
                    liftIO $ hPutStrLn stderr "[!] STOP: Remote is NOT a bit repository!"
                    liftIO $ hPutStrLn stderr $ "Found existing files: " ++ List.intercalate ", " samples
                    liftIO $ hPutStrLn stderr "To initialize anyway (destructive): bit init --force"
                    liftIO $ hPutStrLn stderr "-------------------------------------------------------"

        StateNetworkError err ->
            liftIO $ hPutStrLn stderr $ "Aborting: Network error -> " ++ err

        StateCorruptedRgit msg ->
            liftIO $ hPutStrLn stderr $ "Aborting: [X] Corrupted remote -> " ++ msg

pull :: PullOptions -> BitM ()
pull opts = withRemote $ \remote -> do
    cwd <- asks envCwd
    
    -- Determine if this is a filesystem or cloud remote
    mTarget <- liftIO $ getRemoteTargetType cwd (remoteName remote)
    case mTarget of
        Just (Device.TargetDevice _ _) -> liftIO $ filesystemPull cwd remote opts
        Just (Device.TargetLocalPath _) -> liftIO $ filesystemPull cwd remote opts
        _ -> cloudPull remote opts  -- Cloud remote or no target info (use cloud flow)

-- | Pull from a cloud remote (uses unified transport abstraction).
cloudPull :: Remote -> PullOptions -> BitM ()
cloudPull remote opts =
    let transport = mkCloudTransport remote
    in if pullAcceptRemote opts
        then pullAcceptRemoteImpl transport remote
        else if pullManualMerge opts
            then pullManualMergeImpl remote
            else pullWithCleanup transport remote opts

fetch :: BitM ()
fetch = withRemote $ \remote -> do
    cwd <- asks envCwd
    
    -- Determine if this is a filesystem or cloud remote
    mTarget <- liftIO $ getRemoteTargetType cwd (remoteName remote)
    case mTarget of
        Just (Device.TargetDevice _ _) -> liftIO $ filesystemFetch cwd remote
        Just (Device.TargetLocalPath _) -> liftIO $ filesystemFetch cwd remote
        _ -> cloudFetch remote  -- Cloud remote or no target info (use cloud flow)

-- | Fetch from a cloud remote (original flow, unchanged).
cloudFetch :: Remote -> BitM ()
cloudFetch remote = do
    mb <- liftIO $ fetchRemoteBundle remote
    liftIO $ saveFetchedBundle remote mb

verify :: Bool -> Concurrency -> BitM ()
verify isRemote concurrency
  | isRemote = withRemote $ \remote -> do
      cwd <- asks envCwd
      liftIO $ putStrLn "Fetching remote metadata..."
      liftIO $ putStrLn "Scanning remote files..."
      
      -- Get file count first by doing a quick metadata load
      remoteMeta <- liftIO $ Verify.loadMetadataFromBundle fetchedBundle
      let fileCount = length remoteMeta
      
      -- Setup progress tracking if we have enough files
      if fileCount > 5
        then liftIO $ do
          isTTY <- hIsTerminalDevice stderr
          counter <- newIORef (0 :: Int)
          let shouldShowProgress = isTTY
          
          -- Start progress reporter thread if in TTY
          reporterThread <- if shouldShowProgress
            then Just <$> forkIO (verifyProgressLoop counter fileCount)
            else return Nothing
          
          -- Run verification with progress
          (actualCount, issues) <- finally
            (Verify.verifyRemote cwd remote (Just counter) concurrency)
            (do
              -- Clean up: kill reporter thread and clear line
              maybe (return ()) killThread reporterThread
              when shouldShowProgress clearProgress
            )
          
          -- Print final result
          if null issues
            then putStrLn $ "[OK] All " ++ show actualCount ++ " files match metadata."
            else do
              mapM_ (printVerifyIssue (\s -> take 16 s ++ if length s > 16 then "..." else "")) issues
              putStrLn $ "Checked " ++ show actualCount ++ " files. " ++ show (length issues) ++ " issues found."
        else liftIO $ do
          -- Few files, no progress needed
          (actualCount, issues) <- Verify.verifyRemote cwd remote Nothing concurrency
          if null issues
            then putStrLn $ "[OK] All " ++ show actualCount ++ " files match metadata."
            else do
              mapM_ (printVerifyIssue (\s -> take 16 s ++ if length s > 16 then "..." else "")) issues
              putStrLn $ "Checked " ++ show actualCount ++ " files. " ++ show (length issues) ++ " issues found."
  
  | otherwise = do
      cwd <- asks envCwd
      -- Get file count first by loading metadata
      let indexDir = cwd </> ".bit/index"
      meta <- liftIO $ Verify.loadBinaryMetadata indexDir concurrency
      let fileCount = length meta
      
      -- Setup progress tracking if we have enough files
      if fileCount > 5
        then liftIO $ do
          isTTY <- hIsTerminalDevice stderr
          counter <- newIORef (0 :: Int)
          let shouldShowProgress = isTTY
          
          -- Start progress reporter thread if in TTY
          reporterThread <- if shouldShowProgress
            then Just <$> forkIO (verifyProgressLoop counter fileCount)
            else return Nothing
          
          -- Run verification with progress
          (actualCount, issues) <- finally
            (Verify.verifyLocal cwd (Just counter) concurrency)
            (do
              -- Clean up: kill reporter thread and clear line
              maybe (return ()) killThread reporterThread
              when shouldShowProgress clearProgress
            )
          
          -- Print final result
          if null issues
            then putStrLn $ "[OK] All " ++ show actualCount ++ " files match metadata."
            else do
              mapM_ (printVerifyIssue (\s -> take 16 s ++ if length s > 16 then "..." else "")) issues
              putStrLn $ "Checked " ++ show actualCount ++ " files. " ++ show (length issues) ++ " issues found. Run 'bit status' for details."
        else liftIO $ do
          -- Few files, no progress needed
          (actualCount, issues) <- Verify.verifyLocal cwd Nothing concurrency
          if null issues
            then putStrLn $ "[OK] All " ++ show actualCount ++ " files match metadata."
            else do
              mapM_ (printVerifyIssue (\s -> take 16 s ++ if length s > 16 then "..." else "")) issues
              putStrLn $ "Checked " ++ show actualCount ++ " files. " ++ show (length issues) ++ " issues found. Run 'bit status' for details."

-- | Progress reporter loop for verify operations
verifyProgressLoop :: IORef Int -> Int -> IO ()
verifyProgressLoop counter total = go
  where
    go = do
      n <- readIORef counter
      let pct = (n * 100) `div` max 1 total
      reportProgress $ "Checking files: " ++ show n ++ "/" ++ show total ++ " (" ++ show pct ++ "%)"
      threadDelay 100000  -- 100ms
      when (n < total) go

-- | Progress reporter loop for remote check operations
checkProgressLoop :: IORef Int -> Int -> IO ()
checkProgressLoop counter total = go
  where
    go = do
      n <- readIORef counter
      let pct = (n * 100) `div` max 1 total
      reportProgress $ "Checking files: " ++ show n ++ "/" ++ show total ++ " (" ++ show pct ++ "%)"
      threadDelay 100000  -- 100ms
      when (n < total) go

remoteShow :: Maybe String -> BitM ()
remoteShow mRemoteName = do
    cwd <- asks envCwd
    case mRemoteName of
        Nothing -> do
            -- List all configured remotes
            let remotesDir = cwd </> bitRemotesDir
            dirExists <- liftIO $ Dir.doesDirectoryExist remotesDir
            if not dirExists
                then liftIO $ putStrLn "No remotes configured. Use 'bit remote add <name> <url>' to add one."
                else do
                    remoteNames <- liftIO $ Dir.listDirectory remotesDir
                    if null remoteNames
                        then liftIO $ putStrLn "No remotes configured. Use 'bit remote add <name> <url>' to add one."
                        else liftIO $ forM_ remoteNames $ \name -> do
                            mTarget <- Device.readRemoteFile cwd name
                            display <- formatRemoteDisplay cwd name mTarget
                            putStrLn display
        Just name -> do
            -- Show detailed info for a specific remote
            mRemote <- liftIO $ resolveRemote cwd name
            mTarget <- liftIO $ Device.readRemoteFile cwd name
            display <- liftIO $ case mTarget of
                Just _ -> formatRemoteDisplay cwd name mTarget
                Nothing -> return (name ++ " → " ++ maybe "(not configured)" displayRemote mRemote)
            case mRemote of
                Nothing -> do
                    liftIO $ putStrLn "No remotes configured. Use 'bit remote add <name> <url>' to add one."
                Just remote -> do
                    liftIO $ putStrLn display
                    liftIO $ putStrLn ""
                    let fetchedPath = fromCwdPath (bundleCwdPath fetchedBundle)
                    hasBundle <- liftIO $ Dir.doesFileExist fetchedPath
                    if hasBundle
                        then liftIO $ showRemoteStatusFromBundle name (Just (remoteUrl remote))
                        else do
                            maybeBundlePath <- liftIO $ fetchRemoteBundle remote
                            case maybeBundlePath of
                                Just bPath -> do
                                    liftIO $ saveFetchedBundle remote (Just bPath)
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

remoteCheck :: Maybe String -> BitM ()
remoteCheck mName = do
    cwd <- asks envCwd
    (mRemote, _name) <- liftIO $ case mName of
        Nothing -> do
            name <- Git.getTrackedRemoteName
            mRemote <- resolveRemote cwd name
            return (mRemote, name)
        Just name -> do
            mRemote <- resolveRemote cwd name
            return (mRemote, name)
    case mRemote of
        Nothing -> do
            liftIO $ if maybe True (const False) mName
                then hPutStrLn stderr "fatal: No remote configured."
                else hPutStrLn stderr $ "fatal: '" ++ fromMaybe "" mName ++ "' does not appear to be a git remote."
            liftIO $ hPutStrLn stderr "hint: Set remote with 'bit remote add <name> <url>'"
            liftIO $ exitWith (ExitFailure 1)
        Just remote -> do
            liftIO $ putStrLn $ "Checking local against remote: " ++ displayRemote remote
            liftIO $ putStrLn ""
            
            -- Get file count for progress tracking
            (_, filesOutput, _) <- liftIO $ Git.runGitWithOutput ["ls-files"]
            let fileCount = length . filter (not . null) . lines $ filesOutput
            
            -- Setup progress tracking if we have enough files and a TTY
            if fileCount > 5
                then liftIO $ do
                    isTTY <- hIsTerminalDevice stderr
                    counter <- newIORef (0 :: Int)
                    let shouldShowProgress = isTTY
                    
                    -- Start progress reporter thread if in TTY
                    reporterThread <- if shouldShowProgress
                        then do
                            putStrLn "Running remote check..."
                            Just <$> forkIO (checkProgressLoop counter fileCount)
                        else do
                            putStrLn "Running remote check..."
                            return Nothing
                    
                    -- Run check with progress
                    res <- try @IOException (Transport.checkRemote cwd remote (Just counter))
                    
                    -- Clean up: kill reporter thread and clear line
                    maybe (return ()) killThread reporterThread
                    when shouldShowProgress clearProgress
                    
                    case res of
                        Left _ -> do
                            hPutStrLn stderr "fatal: rclone not found. Install rclone: https://rclone.org/install/"
                            exitWith (ExitFailure 1)
                        Right cr -> processCheckResult cwd cr
                else liftIO $ do
                    putStrLn "Running remote check..."
                    res <- try @IOException (Transport.checkRemote cwd remote Nothing)
                    case res of
                        Left _ -> do
                            hPutStrLn stderr "fatal: rclone not found. Install rclone: https://rclone.org/install/"
                            exitWith (ExitFailure 1)
                        Right cr -> processCheckResult cwd cr
  where
    processCheckResult cwd cr = do
        let reportPath = cwd </> bitDir </> "last-check.txt"
        createDirectoryIfMissing True (cwd </> bitDir)
        atomicWriteFileStr reportPath (Transport.checkRawOutput cr)
        let matches = Transport.checkMatches cr
            differs = Transport.checkDiffers cr
            missingDest = Transport.checkMissingDest cr
            missingSrc = Transport.checkMissingSrc cr
            errs = Transport.checkErrors cr
            nMatch = length matches
            hasDiff = not (null differs && null missingDest && null missingSrc && null errs)
        if not hasDiff && Transport.checkExitCode cr == ExitSuccess
            then do
                putStrLn $ show nMatch ++ " files match between local and remote."
                exitWith ExitSuccess
            else if Transport.checkExitCode cr /= ExitSuccess && Transport.checkExitCode cr /= ExitFailure 1
            then do
                hPutStrLn stderr "fatal: Could not read from remote."
                hPutStrLn stderr ""
                hPutStrLn stderr "Please make sure you have the correct access rights"
                hPutStrLn stderr "and the remote exists."
                unless (null (Transport.checkStderr cr)) $ hPutStrLn stderr (Transport.checkStderr cr)
                exitWith (ExitFailure 1)
            else do
                putStrLn ""
                when (not (null differs)) $ mapM_ putStrLn ("  content differs:" : formatPathList differs)
                when (not (null missingDest)) $ mapM_ putStrLn ("  local only (not on remote):" : formatPathList missingDest)
                when (not (null missingSrc)) $ mapM_ putStrLn ("  remote only (not in local):" : formatPathList missingSrc)
                when (not (null errs)) $ mapM_ putStrLn ("  errors:" : formatPathList errs)
                let errWord = if length errs == 1 then "1 error" else show (length errs) ++ " errors"
                putStrLn $ show (length differs + length missingDest + length missingSrc) ++ " differences, "
                    ++ errWord ++ ". " ++ show nMatch ++ " files matched."
                putStrLn ""
                hPutStrLn stderr "hint: Content differences may indicate an incomplete push or pull."
                hPutStrLn stderr "hint: Run 'bit verify' and 'bit verify --remote' to check metadata consistency."
                hPutStrLn stderr "hint: Full report saved to .bit/last-check.txt"
                exitWith (ExitFailure 1)

mergeContinue :: BitM ()
mergeContinue = do
    cwd <- asks envCwd
    mRemote <- asks envRemote
    let conflictsDir = cwd </> ".bit" </> "conflicts"
    conflictsExist <- liftIO $ Dir.doesDirectoryExist conflictsDir

    gitConflicts <- liftIO Conflict.getConflictedFilesE

    if not (null gitConflicts)
        then liftIO $ hPutStrLn stderr "error: you have not resolved your conflicts yet."
        else if not conflictsExist
            then do
                (code, _, _) <- liftIO $ Git.runGitWithOutput ["rev-parse", "--verify", "MERGE_HEAD"]
                if code == ExitSuccess
                    then do
                        oldHead <- liftIO getLocalHeadE
                        liftIO $ void $ Git.runGitRaw ["commit", "-m", "Merge remote"]
                        liftIO $ putStrLn "Merge complete."
                        case mRemote of
                            Nothing -> return ()
                            Just remote -> do
                                -- Determine transport type based on remote
                                mTarget <- liftIO $ getRemoteTargetType cwd (remoteName remote)
                                let transport = case mTarget of
                                      Just (Device.TargetDevice _ _) -> mkFilesystemTransport (remoteUrl remote)
                                      Just (Device.TargetLocalPath _) -> mkFilesystemTransport (remoteUrl remote)
                                      _ -> mkCloudTransport remote
                                syncBinariesAfterMerge transport remote oldHead
                    else do
                        liftIO $ hPutStrLn stderr "error: no merge in progress."
                        liftIO $ exitWith (ExitFailure 1)
            else do
                invalid <- liftIO $ Metadata.validateMetadataDir (cwd </> bitIndexPath)
                unless (null invalid) $ do
                    liftIO $ hPutStrLn stderr "fatal: Metadata files contain conflict markers. Merge aborted."
                    liftIO $ throwIO (userError "Invalid metadata")

                oldHead <- liftIO getLocalHeadE
                (code, _, _) <- liftIO $ Git.runGitWithOutput ["rev-parse", "--verify", "MERGE_HEAD"]
                when (code /= ExitSuccess) $ do
                    (mergeCode, _, _) <- liftIO $ Git.runGitWithOutput ["merge", "--no-commit", "--no-ff", "refs/remotes/origin/main"]
                    when (mergeCode /= ExitSuccess) $
                        liftIO $ hPutStrLn stderr "warning: Could not start merge. Proceeding anyway."

                liftIO $ void $ Git.runGitRaw ["commit", "-m", "Merge remote (manual merge resolved)"]
                liftIO $ putStrLn "Merge complete."

                liftIO $ removeDirectoryRecursive conflictsDir
                liftIO $ putStrLn "Conflict directories cleaned up."

                case mRemote of
                    Nothing -> return ()
                    Just remote -> do
                        -- Determine transport type based on remote
                        mTarget <- liftIO $ getRemoteTargetType cwd (remoteName remote)
                        let transport = case mTarget of
                              Just (Device.TargetDevice _ _) -> mkFilesystemTransport (remoteUrl remote)
                              Just (Device.TargetLocalPath _) -> mkFilesystemTransport (remoteUrl remote)
                              _ -> mkCloudTransport remote
                        syncBinariesAfterMerge transport remote oldHead

-- ============================================================================
-- Internal helpers (not exported, moved from Commands.hs)
-- ============================================================================

-- | Determine the remote target type from a remote name.
-- Returns the RemoteTarget if the remote is configured, Nothing otherwise.
getRemoteTargetType :: FilePath -> String -> IO (Maybe Device.RemoteTarget)
getRemoteTargetType cwd remName = Device.readRemoteFile cwd remName

-- Git helpers via effect layer
getLocalHeadE :: IO (Maybe String)
getLocalHeadE = do
    (code, out, _) <- Git.runGitWithOutput ["rev-parse", "HEAD"]
    return $ if code == ExitSuccess then Just (filter (not . isSpace) out) else Nothing

checkIsAheadE :: String -> String -> IO Bool
checkIsAheadE rHash lHash = do
    (code, _, _) <- Git.runGitWithOutput ["merge-base", "--is-ancestor", rHash, lHash]
    return (code == ExitSuccess)

hasStagedChangesE :: IO Bool
hasStagedChangesE = do
    (code, _, _) <- Git.runGitWithOutput ["diff", "--cached", "--quiet"]
    return (code == ExitFailure 1)

-- | True if the path is a text file in the index (content stored in metadata, not hash/size).
-- Used during pull to avoid re-downloading from rclone when content is already in the bundle.
isTextFileInIndex :: FilePath -> FilePath -> IO Bool
isTextFileInIndex localRoot filePath = do
    let metaPath = localRoot </> bitIndexPath </> filePath
    exists <- Dir.doesFileExist metaPath
    if not exists then return False
    else do
        mcontent <- readFileMaybe metaPath
        return $ case mcontent of
            Nothing -> False
            Just content -> not (any ("hash: " `isPrefixOf`) (lines content))

-- | Copy a file from the index to the working tree. Call only when the path
-- is a text file (content in index). Creates parent dirs as needed.
copyFromIndexToWorkTree :: FilePath -> FilePath -> IO ()
copyFromIndexToWorkTree localRoot filePath = do
    let metaPath = localRoot </> bitIndexPath </> filePath
        workPath = localRoot </> filePath
    createDirectoryIfMissing True (takeDirectory workPath)
    copyFile metaPath workPath

-- | Helper to read a file safely (returns Nothing on error).
-- Uses strict ByteString reading to avoid Windows file locking issues.
readFileMaybe :: FilePath -> IO (Maybe String)
readFileMaybe filePath = do
    exists <- Dir.doesFileExist filePath
    if exists
        then do
            bs <- BS.readFile filePath
            return $ case decodeUtf8' bs of
                Left _ -> Nothing  -- Invalid UTF-8
                Right txt -> Just (T.unpack txt)
        else return Nothing

-- ============================================================================
-- Compatibility helpers (effect system removal, Step 2)
-- ============================================================================

-- These are dumb IO wrappers to keep the refactor mechanical. They replace the
-- former Free-monad effect constructors.

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

-- | Atomic write of a String (UTF-8). Uses temp file + rename pattern.
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
-- Filesystem remote operations (new for filesystem remote feature)
-- ============================================================================

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
    
    -- 2. Fetch local into remote
    let localIndexGit = cwd </> ".bit" </> "index" </> ".git"
    let remoteIndex = remotePath </> ".bit" </> "index"
    
    putStrLn "Fetching local commits into remote..."
    (fetchCode, _fetchOut, fetchErr) <- Git.runGitAt remoteIndex 
        ["fetch", localIndexGit, "main:refs/remotes/origin/main"]
    
    when (fetchCode /= ExitSuccess) $ do
        hPutStrLn stderr $ "Error fetching into remote: " ++ fetchErr
        exitWith fetchCode
    
    -- 3. Capture remote HEAD before merge
    (oldHeadCode, oldHeadOut, _) <- Git.runGitAt remoteIndex ["rev-parse", "HEAD"]
    let mOldHead = if oldHeadCode == ExitSuccess 
                   then Just (filter (not . isSpace) oldHeadOut)
                   else Nothing
    
    -- 4. Check if remote HEAD is ancestor of what we're pushing (fast-forward check)
    case mOldHead of
        Just _oldHead -> do
            (checkCode, _, _) <- Git.runGitAt remoteIndex 
                ["merge-base", "--is-ancestor", "HEAD", "refs/remotes/origin/main"]
            when (checkCode /= ExitSuccess) $ do
                hPutStrLn stderr "error: Remote has local commits that you don't have."
                hPutStrLn stderr "hint: Run 'bit pull' to merge remote changes first, then push again."
                exitWith (ExitFailure 1)
        Nothing -> return ()  -- First push, no check needed
    
    -- 5. Merge at remote (ff-only)
    putStrLn "Merging at remote (fast-forward only)..."
    (mergeCode, _mergeOut, mergeErr) <- Git.runGitAt remoteIndex 
        ["merge", "--ff-only", "refs/remotes/origin/main"]
    
    if mergeCode /= ExitSuccess
        then do
            hPutStrLn stderr $ "error: Failed to merge at remote: " ++ mergeErr
            exitWith (ExitFailure 1)
        else do
            -- 6. Get new HEAD at remote
            (newHeadCode, newHeadOut, _) <- Git.runGitAt remoteIndex ["rev-parse", "HEAD"]
            when (newHeadCode /= ExitSuccess) $ do
                hPutStrLn stderr "Error: Could not get remote HEAD after merge"
                exitWith (ExitFailure 1)
            
            let newHead = filter (not . isSpace) newHeadOut
            
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
            void $ Git.updateRemoteTrackingBranchToHead
            
            putStrLn "Push complete."

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
                    then return (filePath, True, 0)
                    else do
                        -- Binary file: get size from local file
                        let srcPath = localRoot </> filePath
                        srcExists <- Dir.doesFileExist srcPath
                        if srcExists
                            then do
                                size <- Dir.getFileSize srcPath
                                return (filePath, False, fromIntegral size)
                            else return (filePath, False, 0)
            
            let binaryFiles = [(p, s) | (p, False, s) <- fileInfo, s > 0]
                totalBytes = sum [s | (_, s) <- binaryFiles]
            
            -- Create progress tracker
            progress <- CopyProgress.newSyncProgress (length binaryFiles)
            writeIORef (CopyProgress.spBytesTotal progress) totalBytes
            
            -- Second pass: copy files with progress (parallelized)
            CopyProgress.withSyncProgressReporter progress $ do
                -- Use lower concurrency for file copies to avoid disk thrashing
                caps <- getNumCapabilities
                let concurrency = max 2 (caps * 2)
                void $ runConcurrentlyBounded concurrency (\(filePath, isText, size) -> do
                    let metaPath = remoteIndex </> filePath
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
                                writeIORef (CopyProgress.spCurrentFile progress) filePath
                                CopyProgress.copyFileWithProgress srcPath destPath size progress
                                CopyProgress.incrementFilesComplete progress
                    ) fileInfo
        _ -> return ()

-- | Check if a metadata file is a text file (content stored directly) or binary (hash/size stored).
-- Text files don't have "hash:" lines, binary files do.
isTextMetadataFile :: FilePath -> IO Bool
isTextMetadataFile metaPath = do
    exists <- Dir.doesFileExist metaPath
    if not exists then return False
    else do
        -- Use strict ByteString reading to avoid Windows file locking issues
        bs <- BS.readFile metaPath
        let content = case decodeUtf8' bs of
              Left _ -> ""
              Right txt -> T.unpack txt
        return $ not (any ("hash: " `isPrefixOf`) (lines content))

-- | Sync only changed files between two commits.
filesystemSyncChangedFiles :: FilePath -> FilePath -> String -> String -> IO ()
filesystemSyncChangedFiles localRoot remotePath oldHead newHead = do
    let remoteIndex = remotePath </> ".bit" </> "index"
    changes <- Git.runGitAt remoteIndex ["diff", "--name-status", oldHead, newHead]
    case changes of
        (ExitSuccess, out, _) -> do
            let parsedChanges = parseFilesystemDiffOutput out
            
            -- First pass: collect paths that will be copied and their sizes
            filesToCopy <- fmap concat $ forM parsedChanges $ \(fileStatus, filePath, mNewPath) -> case fileStatus of
                'A' -> return [filePath]
                'M' -> return [filePath]
                'R' -> return (maybe [] (\p -> [p]) mNewPath)
                _ -> return []
            
            -- Gather file sizes for binary files
            fileInfo <- forM filesToCopy $ \p -> do
                let metaPath = remoteIndex </> p
                isText <- isTextMetadataFile metaPath
                if isText
                    then return (p, True, 0)
                    else do
                        let srcPath = localRoot </> p
                        srcExists <- Dir.doesFileExist srcPath
                        if srcExists
                            then do
                                size <- Dir.getFileSize srcPath
                                return (p, False, fromIntegral size)
                            else return (p, False, 0)
            
            let binaryFiles = [(p, s) | (p, False, s) <- fileInfo, s > 0]
                totalBytes = sum [s | (_, s) <- binaryFiles]
            
            -- Create progress tracker
            progress <- CopyProgress.newSyncProgress (length binaryFiles)
            writeIORef (CopyProgress.spBytesTotal progress) totalBytes
            
            -- Second pass: apply changes with progress (parallelized)
            CopyProgress.withSyncProgressReporter progress $ do
                -- Use lower concurrency for file copies to avoid disk thrashing
                caps <- getNumCapabilities
                let concurrency = max 2 (caps * 2)
                void $ runConcurrentlyBounded concurrency (\(st, p, mNewPath) -> case st of
                    'A' -> filesystemCopyFileToRemote localRoot remotePath remoteIndex p progress
                    'M' -> filesystemCopyFileToRemote localRoot remotePath remoteIndex p progress
                    'D' -> filesystemDeleteFileAtRemote remotePath p
                    'R' -> case mNewPath of
                        Just newPath -> do
                            filesystemDeleteFileAtRemote remotePath p
                            filesystemCopyFileToRemote localRoot remotePath remoteIndex newPath progress
                        Nothing -> return ()
                    _ -> return ()
                    ) parsedChanges
        _ -> return ()

-- | Copy a file from local to remote (handles both text and binary).
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

-- | Parse git diff --name-status output for filesystem operations.
parseFilesystemDiffOutput :: String -> [(Char, FilePath, Maybe FilePath)]
parseFilesystemDiffOutput = mapMaybe parseLine . lines
  where
    parseLine line = case line of
        (fileStatus:rest)
            | fileStatus == 'R' || fileStatus == 'C' ->
                case words (dropWhile (\c -> c /= '\t' && c /= ' ') rest) of
                    (old:new:_) -> Just (fileStatus, old, Just new)
                    _ -> Nothing
            | fileStatus `elem` ("ADM" :: String) ->
                case words rest of
                    (filePath:_) -> Just (fileStatus, filePath, Nothing)
                    _ -> Nothing
            | otherwise -> Nothing
        _ -> Nothing

-- =============================================================================
-- FILE TRANSPORT ABSTRACTION
-- =============================================================================

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
  { transportDownloadFile = \cwd filePath _progress -> downloadOrCopyFromIndex cwd remote filePath
  , transportSyncAllFiles = \cwd -> do
      -- Cloud path uses the existing syncRemoteFilesToLocal logic
      -- which scans remote via rclone and syncs to local.
      localFiles <- Scan.scanWorkingDir cwd
      remoteResult <- Remote.Scan.fetchRemoteFiles remote
      case remoteResult of
        Left _ -> hPutStrLn stderr "Error: Failed to fetch remote file list."
        Right remoteFiles -> do
          let actions = Pipeline.pullSyncFiles localFiles remoteFiles
          putStrLn "--- Pulling changes from remote ---"
          if null actions
            then putStrLn "Working tree already up to date with remote."
            else do
              -- Create progress tracker for cloud operations (file-count only)
              progress <- CopyProgress.newSyncProgress (length actions)
              CopyProgress.withSyncProgressReporter progress $ do
                -- Use lower concurrency for network/subprocess operations
                caps <- getNumCapabilities
                let concurrency = min 8 (max 2 (caps * 2))
                void $ runConcurrentlyBounded concurrency (\a -> do
                  executePullCommand cwd remote a
                  CopyProgress.incrementFilesComplete progress
                  ) actions
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

-- | Fetch from a filesystem remote. Fetches commits without merging or syncing files.
filesystemFetch :: FilePath -> Remote -> IO ()
filesystemFetch _cwd remote = do
    let remotePath = remoteUrl remote
    putStrLn $ "Fetching from filesystem remote: " ++ remotePath
    
    -- Check if remote has .bit/ directory
    let remoteBitDir = remotePath </> ".bit"
    remoteHasBit <- Dir.doesDirectoryExist remoteBitDir
    unless remoteHasBit $ do
        hPutStrLn stderr "error: Remote is not a bit repository."
        exitWith (ExitFailure 1)
    
    -- Fetch remote into local
    let remoteIndexGit = remotePath </> ".bit" </> "index" </> ".git"
    
    putStrLn "Fetching remote commits..."
    (fetchCode, _fetchOut, fetchErr) <- Git.runGitWithOutput 
        ["fetch", remoteIndexGit, "main:refs/remotes/origin/main"]
    
    when (fetchCode /= ExitSuccess) $ do
        hPutStrLn stderr $ "Error fetching from remote: " ++ fetchErr
        exitWith fetchCode
    
    -- Output fetch results similar to cloud fetch
    hPutStrLn stderr $ "From " ++ remoteName remote
    hPutStrLn stderr $ " * [new branch]      main       -> origin/main"
    
    putStrLn "Fetch complete."

-- | Pull from a filesystem remote. Fetches directly from the remote's .bit/index/.git.
filesystemPull :: FilePath -> Remote -> PullOptions -> IO ()
filesystemPull cwd remote opts = do
    let remotePath = remoteUrl remote
    putStrLn $ "Pulling from filesystem remote: " ++ remotePath
    
    -- Check if remote has .bit/ directory
    let remoteBitDir = remotePath </> ".bit"
    remoteHasBit <- Dir.doesDirectoryExist remoteBitDir
    unless remoteHasBit $ do
        hPutStrLn stderr "error: Remote is not a bit repository."
        exitWith (ExitFailure 1)
    
    -- 1. Fetch remote into local
    let remoteIndexGit = remotePath </> ".bit" </> "index" </> ".git"
    
    putStrLn "Fetching remote commits..."
    (fetchCode, _fetchOut, fetchErr) <- Git.runGitWithOutput 
        ["fetch", remoteIndexGit, "main:refs/remotes/origin/main"]
    
    when (fetchCode /= ExitSuccess) $ do
        hPutStrLn stderr $ "Error fetching from remote: " ++ fetchErr
        exitWith fetchCode
    
    -- Output fetch results similar to cloud pull
    hPutStrLn stderr $ "From " ++ remoteName remote
    hPutStrLn stderr $ " * [new branch]      main       -> origin/main"
    
    -- 3. Get remote HEAD hash
    (remoteHeadCode, remoteHeadOut, _) <- Git.runGitWithOutput ["rev-parse", "refs/remotes/origin/main"]
    when (remoteHeadCode /= ExitSuccess) $ do
        hPutStrLn stderr "Error: Could not get remote HEAD"
        exitWith (ExitFailure 1)
    
    let remoteHash = filter (not . isSpace) remoteHeadOut
    
    -- NEW: Proof of possession — verify filesystem remote before pulling
    unless (pullAcceptRemote opts || pullSkipVerify opts) $ do
        putStrLn "Verifying remote repository..."
        (remoteCount, remoteIssues) <- Verify.verifyLocalAt remotePath Nothing (Parallel 0)
        if null remoteIssues
            then putStrLn $ "Verified " ++ show remoteCount ++ " remote files."
            else do
                hPutStrLn stderr $ "error: Remote working tree does not match remote metadata (" ++ show (length remoteIssues) ++ " issues)."
                mapM_ (printVerifyIssue (\s -> s)) remoteIssues
                hPutStrLn stderr "hint: Run 'bit verify' in the remote repo to see all mismatches."
                hPutStrLn stderr "hint: Run 'bit pull --accept-remote' to accept the remote's actual state."
                hPutStrLn stderr "hint: Run 'bit push --force' to overwrite remote with local state."
                exitWith (ExitFailure 1)
    
    -- 4. Build transport and delegate to unified pull logic
    let transport = mkFilesystemTransport remotePath
    
    -- Create a minimal BitEnv to call the shared logic
    localFiles <- Scan.scanWorkingDir cwd
    let env = BitEnv cwd localFiles (Just remote) False False (pullSkipVerify opts)
    
    -- Delegate to the unified path
    if pullAcceptRemote opts
        then runBitM env (filesystemPullAcceptRemoteImpl transport remoteHash)
        else runBitM env (filesystemPullLogicImpl transport remote remoteHash)

-- | Filesystem pull logic (simplified - no bundle fetching, just merge + sync)
filesystemPullLogicImpl :: FileTransport -> Remote -> String -> BitM ()
filesystemPullLogicImpl transport _remote remoteHash = do
    cwd <- asks envCwd
    oldHash <- lift getLocalHeadE
    
    case oldHash of
        Nothing -> do
            lift $ putStrLn $ "Checking out " ++ take 7 remoteHash ++ " (first pull)"
            checkoutCode <- lift $ Git.checkoutRemoteAsMain
            if checkoutCode == ExitSuccess
                then do
                    lift $ transportSyncAllFiles transport cwd
                    lift $ putStrLn "Syncing binaries... done."
                    lift $ void $ Git.updateRemoteTrackingBranchToHash remoteHash
                else lift $ hPutStrLn stderr "Error: Failed to checkout remote branch."
        
        Just localHash -> do
            (mergeCode, mergeOut, mergeErr) <- lift $ Git.runGitWithOutput 
                ["merge", "--no-commit", "--no-ff", "refs/remotes/origin/main"]
            
            (finalMergeCode, finalMergeOut, finalMergeErr) <-
                lift $ if mergeCode /= ExitSuccess && "refusing to merge unrelated histories" `List.isInfixOf` (mergeOut ++ mergeErr)
                then do
                    putStrLn "Merging unrelated histories..."
                    Git.runGitWithOutput ["merge", "--no-commit", "--no-ff", "--allow-unrelated-histories", "refs/remotes/origin/main"]
                else return (mergeCode, mergeOut, mergeErr)
            
            if finalMergeCode == ExitSuccess
                then do
                    lift $ putStrLn $ "Updating " ++ take 7 localHash ++ ".." ++ take 7 remoteHash
                    lift $ putStrLn "Merge made by the 'recursive' strategy."
                    hasChanges <- lift hasStagedChangesE
                    when hasChanges $ lift $ void $ Git.runGitRaw ["commit", "-m", "Merge remote"]
                    -- CRITICAL: Always read actual HEAD after merge, never use remoteHash
                    lift $ applyMergeToWorkingDir transport cwd localHash
                    lift $ putStrLn "Syncing binaries... done."
                    lift $ void $ Git.updateRemoteTrackingBranchToHash remoteHash
                else do
                    lift $ putStrLn finalMergeOut
                    lift $ hPutStrLn stderr finalMergeErr
                    lift $ putStrLn "Automatic merge failed."
                    lift $ putStrLn "bit requires you to pick a version for each conflict."
                    lift $ putStrLn ""
                    lift $ putStrLn "Resolving conflicts..."
                    
                    conflicts <- lift Conflict.getConflictedFilesE
                    resolutions <- lift $ Conflict.resolveAll conflicts
                    let total = length resolutions
                    
                    invalid <- lift $ Metadata.validateMetadataDir (cwd </> bitIndexPath)
                    unless (null invalid) $ do
                        lift $ void $ Git.runGitRaw ["merge", "--abort"]
                        lift $ hPutStrLn stderr "fatal: Metadata files contain conflict markers. Merge aborted."
                        lift $ throwIO (userError "Invalid metadata")
                    
                    conflictsNow <- lift Conflict.getConflictedFilesE
                    if null conflictsNow
                        then do
                            lift $ void $ Git.runGitRaw ["commit", "-m", "Merge remote (resolved " ++ show total ++ " conflict(s))"]
                            lift $ putStrLn $ "Merge complete. " ++ show total ++ " conflict(s) resolved."
                            -- CRITICAL: Always read actual HEAD after merge, never use remoteHash
                            lift $ applyMergeToWorkingDir transport cwd localHash
                            lift $ putStrLn "Syncing binaries... done."
                            lift $ void $ Git.updateRemoteTrackingBranchToHash remoteHash
                        else return ()

-- | Filesystem pull --accept-remote implementation
filesystemPullAcceptRemoteImpl :: FileTransport -> String -> BitM ()
filesystemPullAcceptRemoteImpl transport remoteHash = do
    cwd <- asks envCwd
    lift $ putStrLn "Accepting remote file state as truth..."
    
    -- Record current HEAD before checkout
    oldHead <- lift getLocalHeadE
    
    -- Force-checkout the remote branch
    checkoutCode <- lift Git.checkoutRemoteAsMain
    if checkoutCode /= ExitSuccess
        then lift $ hPutStrLn stderr "Error: Failed to checkout remote state."
        else do
            -- Sync actual files based on what changed
            case oldHead of
                Just oh -> lift $ applyMergeToWorkingDir transport cwd oh
                Nothing -> lift $ transportSyncAllFiles transport cwd
            
            -- Update tracking ref
            lift $ void $ Git.updateRemoteTrackingBranchToHash remoteHash
            lift $ putStrLn "Pull with --accept-remote completed."

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
                then return (filePath, True, 0)
                else do
                    -- Binary file: get size from remote file
                    let srcPath = remotePath </> filePath
                    srcExists <- Dir.doesFileExist srcPath
                    if srcExists
                        then do
                            size <- Dir.getFileSize srcPath
                            return (filePath, False, fromIntegral size)
                        else return (filePath, False, 0)
        
        let binaryFiles = [(p, s) | (p, False, s) <- fileInfo, s > 0]
            totalBytes = sum [s | (_, s) <- binaryFiles]
        
        -- Create progress tracker
        progress <- CopyProgress.newSyncProgress (length binaryFiles)
        writeIORef (CopyProgress.spBytesTotal progress) totalBytes
        
        -- Second pass: copy files with progress (parallelized)
        CopyProgress.withSyncProgressReporter progress $ do
            -- Use lower concurrency for file copies to avoid disk thrashing
            caps <- getNumCapabilities
            let concurrency = max 2 (caps * 2)
            void $ runConcurrentlyBounded concurrency (\(filePath, isText, size) -> do
                let metaPath = localIndex </> filePath
                if isText
                    then do
                        -- Text file: metadata IS the content, copy from local index to working tree
                        let workPath = localRoot </> filePath
                        createDirectoryIfMissing True (takeDirectory workPath)
                        copyFile metaPath workPath
                    else do
                        -- Binary file: metadata is hash/size, copy actual file from remote working tree
                        let srcPath = remotePath </> filePath
                        let destPath = localRoot </> filePath
                        srcExists <- Dir.doesFileExist srcPath
                        when srcExists $ do
                            writeIORef (CopyProgress.spCurrentFile progress) filePath
                            CopyProgress.copyFileWithProgress srcPath destPath size progress
                            CopyProgress.incrementFilesComplete progress
                ) fileInfo

-- filesystemApplyMergeToWorkingDir was removed - now using unified applyMergeToWorkingDir with FileTransport

-- | Download a file from remote or copy from index for filesystem pull.
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

-- | Executes/Prints the command to be run in the shell (push: local -> remote).
executeCommand :: FilePath -> Remote -> RcloneAction -> IO ()
executeCommand localRoot remote action = case action of
        Copy src dest -> do
            let localPath = toPosix (localRoot </> src)
            void $ Transport.copyToRemote localPath remote (toPosix dest)

        Move src dest ->
            void $ Transport.moveRemote remote (toPosix src) (toPosix dest)

        Delete p ->
            void $ Transport.deleteRemote remote (toPosix p)

        Swap _ _ _ -> return ()  -- not produced by planAction; future-proofing

-- | Execute a single pull action: copy from remote to local or delete local file.
-- Text files are already in the git bundle (index); copy from index to work dir instead of rclone.
executePullCommand :: FilePath -> Remote -> RcloneAction -> IO ()
executePullCommand localRoot remote action = case action of
        Copy _src dest -> do
            fromIndex <- isTextFileInIndex localRoot dest
            if fromIndex
            then copyFromIndexToWorkTree localRoot dest
            else do
                let localPath = toPosix (localRoot </> dest)
                createDirectoryIfMissing True (takeDirectory (localRoot </> dest))
                void $ Transport.copyFromRemote remote (toPosix dest) localPath
        Move src dest -> do
            fromIndex <- isTextFileInIndex localRoot src
            if fromIndex
            then copyFromIndexToWorkTree localRoot src
            else do
                let localSrcPath = localRoot </> src
                createDirectoryIfMissing True (takeDirectory localSrcPath)
                void $ Transport.copyFromRemote remote (toPosix src) (toPosix localSrcPath)
            let localDestPath = localRoot </> dest
            exists <- Dir.doesFileExist localDestPath
            when exists $ Dir.removeFile localDestPath
        Delete filePath -> do
            let localPath = localRoot </> filePath
            exists <- Dir.doesFileExist localPath
            when exists $ Dir.removeFile localPath
        Swap _ _ _ -> return ()

-- Helper to ensure we don't crash if cleanup fails (IO version for use outside BitM)
safeRemove :: FilePath -> IO ()
safeRemove filePath = do
    exists <- Dir.doesFileExist filePath
    when exists (Dir.removeFile filePath)


-- | Show up to 10 paths; if more than 20, show first 10 then "... and N more".
formatPathList :: [FilePath] -> [String]
formatPathList paths
  | length paths <= 20 = map (\p -> "        " ++ toPosix p) paths
  | otherwise         = map (\p -> "        " ++ toPosix p) (take 10 paths)
                        ++ ["        ... and " ++ show (length paths - 10) ++ " more"]

-- ============================================================================
-- Domain logic functions (moved from Transport in Step 4)
-- ============================================================================

-- | Classify remote state (empty, valid bit, non-bit, corrupted, network error)
-- This is domain logic: it knows what .bit/ means and interprets remote contents
classifyRemoteState :: Remote -> IO RemoteState
classifyRemoteState remote = do
    result <- Transport.listRemoteItems remote 1
    case result of
        Left err -> return (StateNetworkError err)
        Right items -> return (interpretRemoteItems items)

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
        Transport.CopySuccess -> return (BundleFound localDest)
        Transport.CopyNotFound -> return RemoteEmpty
        Transport.CopyNetworkError _ -> 
            return (NetworkError "Network unreachable: Check your internet connection or remote name.")
        Transport.CopyOtherError err -> return (NetworkError err)

-- ============================================================================
-- Core operations implementations
-- ============================================================================

-- | Push the git bundle to remote. Uses bracket to ensure temp bundle cleanup.
pushBundle :: Remote -> IO ()
pushBundle remote = do
    let tempBundle = BundleName "bit"
        tempBundleCwdPath = fromCwdPath (bundleCwdPath tempBundle)

    -- bracket <setup> <cleanup> <action>
    bracket
        (Git.createBundle tempBundle) -- 1. Acquire
        (\_ -> cleanupTemp tempBundleCwdPath) -- 2. Release (Always runs)
        (\gCode -> do                 -- 3. Work
            if gCode /= ExitSuccess
                then hPutStrLn stderr "Error creating bundle"
                else uploadToRemote tempBundleCwdPath remote
        )

-- Helper for the upload logic to keep the bracket clean
uploadToRemote :: FilePath -> Remote -> IO ()
uploadToRemote src remote = do
    putStrLn "Uploading bundle to remote..."
    rCode <- Transport.copyToRemote src remote ".bit/bit.bundle"
    if rCode == ExitSuccess
        then putStrLn "Metadata push complete."
        else hPutStrLn stderr "Error uploading bundle."

-- Helper for cleanup that doesn't crash if the file was never made
cleanupTemp :: FilePath -> IO ()
cleanupTemp filePath = do
    exists <- Dir.doesFileExist filePath
    when exists (Dir.removeFile filePath)

-- fetchBundle moved to Internal.Transport


-- | Sync files, push bundle, and update local tracking. Used after remote checks pass.
pushToRemote :: Remote -> BitM ()
pushToRemote remote = do
  syncRemoteFiles
  liftIO $ pushBundle remote
  updateLocalBundleAfterPush

-- | After a successful push, update the local fetched_remote.bundle to current HEAD
-- so rgit status shows up to date instead of "ahead of remote".
updateLocalBundleAfterPush :: BitM ()
updateLocalBundleAfterPush = do
    code <- liftIO $ Git.createBundle fetchedBundle
    when (code == ExitSuccess) $ do
        void $ liftIO $ Git.updateRemoteTrackingBranch fetchedBundle

syncRemoteFiles :: BitM ()
syncRemoteFiles = withRemote $ \remote -> do
    cwd <- asks envCwd
    localFiles <- asks envLocalFiles
    remoteResult <- liftIO $ Remote.Scan.fetchRemoteFiles remote
    either
        (\_ -> liftIO $ hPutStrLn stderr "Error: Failed to fetch remote file list.")
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
    force <- asks envForce
    forceWithLease <- asks envForceWithLease
    mRemote <- asks envRemote
    -- Handle --force: skip all checks and push anyway
    if force
        then do
            lift $ tellErr "Warning: --force used. Overwriting remote history..."
            maybe (lift $ tellErr "Error: No remote configured.") pushToRemote mRemote
        else do
            -- Handle --force-with-lease: compare remote bundle hash against fetched_remote.bundle
            if forceWithLease
                then do
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
                else do
                    maybeRemoteHash <- liftIO $ Git.getHashFromBundle fetchedBundle
                    maybeLocalHash <- lift getLocalHeadE

                    case (maybeLocalHash, maybeRemoteHash) of
                        (Just lHash, Just rHash) -> do
                            isAhead <- lift $ checkIsAheadE rHash lHash

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

-- | Sync binaries after a successful merge commit
syncBinariesAfterMerge :: FileTransport -> Remote -> Maybe String -> BitM ()
syncBinariesAfterMerge transport _remote oldHead = do
    cwd <- asks envCwd
    liftIO $ putStrLn "Syncing binaries... done."
    case oldHead of
        Just oh -> liftIO $ applyMergeToWorkingDir transport cwd oh
        Nothing -> do
            -- Fallback: use transportSyncAllFiles (syncs from current HEAD)
            liftIO $ transportSyncAllFiles transport cwd
    maybeRemoteHash <- liftIO $ Git.getHashFromBundle fetchedBundle
    case maybeRemoteHash of
        Just rHash -> liftIO $ void $ Git.updateRemoteTrackingBranchToHash rHash
        Nothing    -> return ()

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
    case newHead of
        Nothing -> return ()  -- shouldn't happen after merge commit
        Just newH -> do
            changes <- Git.getDiffNameStatus oldHead newH
            putStrLn "--- Pulling changes from remote ---"
            if null changes
                then putStrLn "Working tree already up to date with remote."
                else do
                    -- First pass: collect paths that will be copied and their sizes
                    filesToCopy <- fmap concat $ forM changes $ \(fileStatus, filePath, mNewPath) -> case fileStatus of
                        'A' -> return [filePath]
                        'M' -> return [filePath]
                        'R' -> return (maybe [] (\p -> [p]) mNewPath)
                        _ -> return []
                    
                    -- Gather file sizes for binary files (for progress tracking)
                    fileInfo <- forM filesToCopy $ \filePath -> do
                        fromIndex <- isTextFileInIndex cwd filePath
                        if fromIndex
                            then return (filePath, True, (0 :: Integer))
                            else do
                                -- Binary file: try to get size for progress
                                -- (size might not be available yet, that's ok)
                                let _destPath = cwd </> filePath
                                return (filePath, False, 0)  -- Size will be tracked during copy
                    
                    let binaryFiles = [(p, s) | (p, False, s) <- fileInfo]
                        totalFiles = length binaryFiles
                    
                    -- Create progress tracker
                    progress <- CopyProgress.newSyncProgress totalFiles
                    
                    -- Second pass: apply changes with progress (parallelized)
                    CopyProgress.withSyncProgressReporter progress $ do
                        -- Use lower concurrency for file operations to avoid thrashing
                        caps <- getNumCapabilities
                        let concurrency = max 2 (caps * 2)
                        void $ runConcurrentlyBounded concurrency (\(fileStatus, filePath, mNewPath) -> case fileStatus of
                            'A' -> (transportDownloadFile transport) cwd filePath progress
                            'M' -> (transportDownloadFile transport) cwd filePath progress
                            'D' -> safeDeleteWorkFile cwd filePath
                            'R' -> case mNewPath of
                                Just newPath -> do
                                    safeDeleteWorkFile cwd filePath
                                    (transportDownloadFile transport) cwd newPath progress
                                Nothing -> return ()
                            _ -> return ()
                            ) changes

-- | Download a file from remote, or copy from index if it's a text file.
-- Used by cloud transport.
downloadOrCopyFromIndex :: FilePath -> Remote -> FilePath -> IO ()
downloadOrCopyFromIndex cwd remote filePath = do
    fromIndex <- isTextFileInIndex cwd filePath
    if fromIndex
        then copyFromIndexToWorkTree cwd filePath
        else do
            let localPath = cwd </> filePath
            createDirectoryIfMissing True (takeDirectory localPath)
            void $ Transport.copyFromRemote remote (toPosix filePath) (toPosix localPath)

-- | Safely delete a file from the working directory.
safeDeleteWorkFile :: FilePath -> FilePath -> IO ()
safeDeleteWorkFile cwd filePath = do
    let fullPath = cwd </> filePath
    exists <- Dir.doesFileExist fullPath
    when exists $ Dir.removeFile fullPath

-- | Old showRemoteStatus logic: classify remote, fetch bundle if valid rgit.
-- Returns the temp bundle path on success, Nothing otherwise.
fetchRemoteBundle :: Remote -> IO (Maybe FilePath)
fetchRemoteBundle remote = do
    remoteState <- classifyRemoteState remote

    case remoteState of
        StateNetworkError _err -> do
            hPutStrLn stderr $ unlines
                [ _err
                , "fatal: Could not read from remote repository."
                , ""
                , "Please make sure you have the correct access rights"
                , "and the repository exists."
                ]
            return Nothing

        StateEmpty -> do
            -- Git fetch silently succeeds when remote is empty
            return Nothing

        StateNonRgitOccupied samples -> do
            hPutStrLn stderr $ "StateNonRgitOccupied: " ++ show samples
            hPutStrLn stderr $ unlines
                [ "fatal: Could not read from remote repository."
                , ""
                , "Please make sure you have the correct access rights"
                , "and the repository exists."
                ]
            return Nothing

        StateCorruptedRgit _msg -> do
            hPutStrLn stderr $ "StateCorruptedRgit: " ++ _msg
            hPutStrLn stderr $ unlines
                [ "fatal: Could not read from remote repository."
                , ""
                , "Please make sure you have the correct access rights"
                , "and the repository exists."
                ]
            return Nothing

        StateValidRgit -> do
            fetchResult <- fetchBundle remote
            case fetchResult of
                BundleFound bPath -> return $ Just bPath
                _ -> do
                    hPutStrLn stderr $ unlines
                        [ "fatal: Could not read from remote repository."
                        , ""
                        , "Please make sure you have the correct access rights"
                        , "and the repository exists."
                        ]
                    return Nothing

saveFetchedBundle :: Remote -> Maybe FilePath -> IO ()
saveFetchedBundle _remote Nothing = pure ()
saveFetchedBundle remote (Just bPath) = do
    let fetchedPath = fromCwdPath (bundleCwdPath fetchedBundle)
    hadPrevious <- Dir.doesFileExist fetchedPath
    maybeOldHash <- if hadPrevious
        then Git.getHashFromBundle fetchedBundle
        else return Nothing

    -- Copy FIRST, then read hash from the correct location
    copyFile bPath fetchedPath
    safeRemove bPath
    maybeNewHash <- Git.getHashFromBundle fetchedBundle

    -- Ensure the internal git remote "origin" points to the right URL.
    -- This is the git-internal remote used for fetching refs from bundles,
    -- NOT upstream tracking config (branch.main.remote).
    -- 
    -- REMOVED: Git.setupBranchTracking
    -- That line was silently configuring upstream tracking (branch.main.remote=origin)
    -- as a side effect of every fetch/pull. Per bit's spec, upstream tracking should
    -- only be set via explicit `bit push -u` (git-standard behavior).
    _ <- Git.setupRemote (remoteUrl remote)

    case maybeNewHash of
        Just _ -> void $ Git.fetchFromBundle fetchedBundle
        Nothing -> return ()

    -- Output fetch results in git format
    case (maybeOldHash, maybeNewHash) of
        (Nothing, Just _newHash) -> do
            -- First time fetching - show as new branch
            hPutStrLn stderr $ "From " ++ remoteName remote
            hPutStrLn stderr $ " * [new branch]      main       -> origin/main"
        (Just oldHash, Just newHash) ->
            if oldHash == newHash
                then return () -- Up to date, no output
                else do
                    -- Check if this is a normal update (old hash is ancestor of new hash)
                    isNormal <- Git.checkIsAhead oldHash newHash
                    hPutStrLn stderr $ "From " ++ remoteName remote
                    if isNormal
                        then hPutStrLn stderr $ "   " ++ take 7 oldHash ++ ".." ++ take 7 newHash ++ "  main       -> origin/main"
                        else hPutStrLn stderr $ " + " ++ take 7 oldHash ++ "..." ++ take 7 newHash ++ " main       -> origin/main  (forced update)"
        _ -> return () -- Error case, already handled

-- | Full pull with conflict resolution. Uses bracket_ to abort merge on exception (e.g. Ctrl+C).

-- | Pull with --accept-remote: force-checkout the remote branch, then sync files.
-- Git manages .bit/index/ (the metadata); we only sync actual files to the working tree.
pullAcceptRemoteImpl :: FileTransport -> Remote -> BitM ()
pullAcceptRemoteImpl transport remote = do
    cwd <- asks envCwd
    lift $ tell "Accepting remote file state as truth..."

    -- 1. Fetch the remote bundle so git has the remote's history
    maybeBundlePath <- lift $ fetchRemoteBundle remote
    case maybeBundlePath of
        Nothing -> lift $ tellErr "Error: Could not fetch remote bundle."
        Just bPath -> do
            lift $ saveFetchedBundle remote (Just bPath)

            -- 2. Record current HEAD before checkout (for diff-based sync)
            oldHead <- lift getLocalHeadE

            -- 3. Force-checkout the remote branch.
            --    This makes .bit/index/ match the remote's metadata exactly:
            --    text files get actual content, binary files get hash/size.
            lift $ tell "Scanning remote files..."
            checkoutCode <- lift Git.checkoutRemoteAsMain
            if checkoutCode /= ExitSuccess
                then lift $ tellErr "Error: Failed to checkout remote state."
                else do
                    -- 4. Sync actual files to working tree based on what changed in git
                    (_remoteCode, remoteOut, _) <- lift $ gitQuery ["rev-parse", "refs/remotes/origin/main"]
                    let _newHash = takeWhile (/= '\n') remoteOut
                    case oldHead of
                        Just oh -> lift $ applyMergeToWorkingDir transport cwd oh
                        Nothing -> lift $ transportSyncAllFiles transport cwd  -- First time, no diff available

                    -- 5. Update tracking ref
                    maybeRemoteHash <- lift $ Git.getHashFromBundle fetchedBundle
                    case maybeRemoteHash of
                        Just rHash -> lift $ void $ Git.updateRemoteTrackingBranchToHash rHash
                        Nothing    -> return ()

                    lift $ tell "Pull with --accept-remote completed."

-- | Pull with --manual-merge: detect remote divergence and create conflict directories.
pullManualMergeImpl :: Remote -> BitM ()
pullManualMergeImpl remote = do
    cwd <- asks envCwd
    lift $ tell "Fetching remote metadata... done."

    maybeBundlePath <- lift $ fetchRemoteBundle remote
    case maybeBundlePath of
        Nothing -> lift $ tellErr "Error: Could not fetch remote bundle."
        Just bPath -> do
            lift $ saveFetchedBundle remote (Just bPath)

            (remoteMeta, _allBundlePaths) <- lift $ Verify.loadMetadataFromBundle fetchedBundle
            lift $ tell "Scanning remote files... done."
            result <- lift $ Remote.Scan.fetchRemoteFiles remote
            case result of
                Left _ -> lift $ tellErr "Error: Could not fetch remote file list."
                Right remoteFiles -> do
                    let filteredRemoteFiles = filterOutBitPaths remoteFiles
                    localMeta <- lift $ Verify.loadBinaryMetadata (cwd </> bitIndexPath) (Parallel 0)

                    let remoteFileMap = Map.fromList
                          [ (normalise e.path, (h, e.kind))
                          | e <- filteredRemoteFiles
                          , h <- maybeToList (syncHash e.kind)
                          ]
                        remoteMetaMap = Map.fromList [(normalise p, (h, sz)) | (p, h, sz) <- remoteMeta]
                        localMetaMap = Map.fromList [(normalise p, (h, sz)) | (p, h, sz) <- localMeta]

                    lift $ tell "Comparing..."
                    let divergentFiles = findDivergentFiles remoteFileMap remoteMetaMap localMetaMap

                    if null divergentFiles
                        then do
                            lift $ tell "No remote divergence detected. Proceeding with normal pull..."
                            let transport = mkCloudTransport remote
                            pullWithCleanup transport remote defaultPullOptions
                        else do
                            _oldHash <- lift getLocalHeadE
                            (_remoteCode, remoteOut, _) <- lift $ gitQuery ["rev-parse", "refs/remotes/origin/main"]
                            let _newHash = takeWhile (/= '\n') remoteOut

                            (mergeCode, mergeOut, mergeErr) <- lift $ gitQuery ["merge", "--no-commit", "--no-ff", "refs/remotes/origin/main"]
                            (_finalMergeCode, _, _) <- lift $ if mergeCode /= ExitSuccess && "refusing to merge unrelated histories" `List.isInfixOf` (mergeOut ++ mergeErr)
                                then do tell "Merging unrelated histories (e.g. first pull)..."; gitQuery ["merge", "--no-commit", "--no-ff", "--allow-unrelated-histories", "refs/remotes/origin/main"]
                                else return (mergeCode, mergeOut, mergeErr)

                            createConflictDirectories remote divergentFiles remoteFileMap remoteMetaMap localMetaMap

                            lift $ printConflictList divergentFiles remoteFileMap remoteMetaMap localMetaMap
                            lift $ do
                                tell ""
                                tell "To resolve:"
                                tell "  1. Examine files in .bit/conflicts/<path>/"
                                tell "  2. Copy your chosen version to <path>"
                                tell "  3. Run 'bit add <path>'"
                                tell "  4. Run 'bit merge --continue'"
                                tell ""
                                tell "Or abort: 'bit merge --abort'"

-- | Find files where remote actual files don't match remote metadata.
findDivergentFiles :: Map.Map FilePath (Hash 'MD5, EntryKind) -> Map.Map FilePath (Hash 'MD5, Integer) -> Map.Map FilePath (Hash 'MD5, Integer) -> [(FilePath, Hash 'MD5, Hash 'MD5, Integer, Integer)]
findDivergentFiles remoteFileMap remoteMetaMap _localMetaMap =
    Map.foldlWithKey (\acc filePath (expectedHash, expectedSize) ->
        let normalizedPath = normalise filePath
        in case Map.lookup normalizedPath remoteFileMap of
            Nothing -> acc  -- File missing on remote, skip
            Just (actualHash, entryKind) ->
                case entryKind of
                    File _ actualSize _ ->
                        if actualHash == expectedHash && actualSize == expectedSize
                            then acc  -- Matches, no divergence
                            else (filePath, expectedHash, actualHash, expectedSize, actualSize) : acc  -- Divergence!
                    _ -> acc
        ) [] remoteMetaMap

-- | Create conflict directories for divergent files.
createConflictDirectories :: Remote -> [(FilePath, Hash 'MD5, Hash 'MD5, Integer, Integer)] -> Map.Map FilePath (Hash 'MD5, EntryKind) -> Map.Map FilePath (Hash 'MD5, Integer) -> Map.Map FilePath (Hash 'MD5, Integer) -> BitM ()
createConflictDirectories remote divergentFiles _remoteFileMap _remoteMetaMap localMetaMap = do
    cwd <- asks envCwd
    let conflictsDir = cwd </> ".bit" </> "conflicts"
    lift $ createDirE conflictsDir

    forM_ divergentFiles $ \(filePath, _expectedHash, actualHash, _expectedSize, actualSize) -> do
        let conflictDir = conflictsDir </> filePath
        lift $ createDirE (takeDirectory conflictDir)

        let localPath = cwd </> filePath
        localExists <- lift $ fileExistsE localPath
        when localExists $ lift $ copyFileE localPath (conflictDir </> "LOCAL")

        code <- liftIO $ Transport.copyFromRemote remote (toPosix filePath) (conflictDir </> "REMOTE")
        when (code /= ExitSuccess) $ lift $ tellErr $ "Warning: Could not download remote file: " ++ filePath

        lift $ case Map.lookup (normalise filePath) localMetaMap of
            Just (localHash, localSize) ->
                writeFileAtomicE (conflictDir </> "METADATA_LOCAL") $
                    serializeMetadata (MetaContent localHash localSize)
            Nothing -> writeFileAtomicE (conflictDir </> "METADATA_LOCAL") "hash: (not tracked)\nsize: 0\n"

        lift $ writeFileAtomicE (conflictDir </> "METADATA_REMOTE") $
            serializeMetadata (MetaContent actualHash actualSize)

-- | Print conflict list in spec format.
printConflictList :: [(FilePath, Hash 'MD5, Hash 'MD5, Integer, Integer)] -> Map.Map FilePath (Hash 'MD5, EntryKind) -> Map.Map FilePath (Hash 'MD5, Integer) -> Map.Map FilePath (Hash 'MD5, Integer) -> IO ()
printConflictList divergentFiles _remoteFileMap _remoteMetaMap localMetaMap = do
    putStrLn ""
    putStrLn "✗ Remote divergence detected:"
    putStrLn ""
    
    forM_ divergentFiles $ \(filePath, expectedHash, remoteHash, expectedSize, remoteSize) -> do
        putStrLn $ "  " ++ toPosix filePath ++ ":"
        
        -- Get local metadata (use displayHash for Hash 'MD5 values)
        let localInfo = case Map.lookup (normalise filePath) localMetaMap of
                Just (localHash, localSize) -> (displayHash localHash, show localSize)
                Nothing -> ("(not tracked)", "0")
        
        putStrLn $ "    Local:           " ++ fst localInfo ++ " (" ++ snd localInfo ++ " bytes)"
        putStrLn $ "    Remote actual:   " ++ displayHash remoteHash ++ " (" ++ show remoteSize ++ " bytes)"
        putStrLn $ "    Remote metadata: " ++ displayHash expectedHash ++ " (" ++ show expectedSize ++ " bytes)"
        putStrLn $ ""
        putStrLn $ "    Files saved to: .bit/conflicts/" ++ toPosix filePath ++ "/"
        putStrLn ""
    
    putStrLn "This can happen when:"
    putStrLn "  - Files were modified directly on the remote (not via bit)"
    putStrLn "  - A partial push from another client"
    putStrLn "  - Remote storage corruption"

pullWithCleanup :: FileTransport -> Remote -> PullOptions -> BitM ()
pullWithCleanup transport remote opts = do
    env <- asks id
    result <- liftIO $ try @SomeException (runBitM env (pullLogic transport remote opts))
    case result of
        Left ex -> do
            inProgress <- lift $ Git.isMergeInProgress
            if inProgress
                then do
                    lift $ void $ gitRaw ["merge", "--abort"]
                    lift $ tell "Merge aborted. Your working tree is unchanged."
                else lift $ throwIO ex
        Right _ -> return ()

pullLogic :: FileTransport -> Remote -> PullOptions -> BitM ()
pullLogic transport remote opts = do
    cwd <- asks envCwd
    maybeBundlePath <- lift $ fetchRemoteBundle remote
    case maybeBundlePath of
        Nothing -> return ()
        Just bPath -> do
            lift $ saveFetchedBundle remote (Just bPath)
            (_, countOut, _) <- lift $ gitQuery ["rev-list", "--count", "refs/remotes/origin/main"]
            let n = takeWhile (`elem` ['0'..'9']) (filter (/= '\n') countOut)
            lift $ tell $ "remote: Counting objects: " ++ (if null n then "0" else n) ++ ", done."

            -- NEW: Proof of possession — verify remote before pulling
            unless (pullSkipVerify opts) $ do
                lift $ putStrLn "Verifying remote files..."
                (remoteFileCount, remoteIssues) <- lift $ Verify.verifyRemote cwd remote Nothing (Parallel 0)
                if null remoteIssues
                    then lift $ putStrLn $ "Verified " ++ show remoteFileCount ++ " remote files."
                    else do
                        lift $ hPutStrLn stderr $ "error: Remote files do not match remote metadata (" ++ show (length remoteIssues) ++ " issues)."
                        lift $ mapM_ (printVerifyIssue (\s -> s)) remoteIssues
                        lift $ hPutStrLn stderr "hint: Run 'bit verify --remote' to see all mismatches."
                        lift $ hPutStrLn stderr "hint: Run 'bit pull --accept-remote' to accept the remote's actual state."
                        lift $ hPutStrLn stderr "hint: Run 'bit push --force' to overwrite remote with local state."
                        lift $ exitWith (ExitFailure 1)

            oldHash <- lift getLocalHeadE
            (_remoteCode, remoteOut, _) <- lift $ gitQuery ["rev-parse", "refs/remotes/origin/main"]
            let newHash = takeWhile (/= '\n') remoteOut

            case oldHash of
                Nothing -> do
                    lift $ tell $ "Checking out " ++ take 7 newHash ++ " (first pull)"
                    checkoutCode <- lift $ Git.checkoutRemoteAsMain
                    if checkoutCode == ExitSuccess
                        then do
                            lift $ transportSyncAllFiles transport cwd
                            lift $ tell "Syncing binaries... done."
                        else lift $ tellErr "Error: Failed to checkout remote branch."

                Just localHead -> do
                    (mergeCode, mergeOut, mergeErr) <- lift $ gitQuery ["merge", "--no-commit", "--no-ff", "refs/remotes/origin/main"]

                    (finalMergeCode, finalMergeOut, finalMergeErr) <-
                        lift $ if mergeCode /= ExitSuccess && "refusing to merge unrelated histories" `List.isInfixOf` (mergeOut ++ mergeErr)
                        then do tell "Merging unrelated histories..."; gitQuery ["merge", "--no-commit", "--no-ff", "--allow-unrelated-histories", "refs/remotes/origin/main"]
                        else return (mergeCode, mergeOut, mergeErr)

                    if finalMergeCode == ExitSuccess
                    then do
                        lift $ tell $ "Updating " ++ take 7 localHead ++ ".." ++ take 7 newHash
                        lift $ tell "Merge made by the 'recursive' strategy."
                        hasChanges <- lift hasStagedChangesE
                        when hasChanges $ lift $ void $ gitRaw ["commit", "-m", "Merge remote"]
                        lift $ applyMergeToWorkingDir transport cwd localHead
                        lift $ tell "Syncing binaries... done."
                        maybeRemoteHash <- lift $ Git.getHashFromBundle fetchedBundle
                        case maybeRemoteHash of
                            Just rHash -> lift $ void $ Git.updateRemoteTrackingBranchToHash rHash
                            Nothing    -> return ()
                    else do
                        lift $ tell finalMergeOut
                        lift $ tellErr finalMergeErr
                        lift $ tell "Automatic merge failed."
                        lift $ tell "bit requires you to pick a version for each conflict."
                        lift $ tell ""
                        lift $ tell "Resolving conflicts..."

                        conflicts <- lift Conflict.getConflictedFilesE
                        resolutions <- lift $ Conflict.resolveAll conflicts
                        let total = length resolutions

                        invalid <- lift $ Metadata.validateMetadataDir (cwd </> bitIndexPath)
                        unless (null invalid) $ do
                            lift $ void $ gitRaw ["merge", "--abort"]
                            lift $ tellErr "fatal: Metadata files contain conflict markers. Merge aborted."
                            lift $ throwIO (userError "Invalid metadata")

                        conflictsNow <- lift Conflict.getConflictedFilesE
                        if null conflictsNow
                        then do
                            -- Always commit: MERGE_HEAD exists so git commit succeeds
                            -- even when the index matches HEAD (e.g. user chose "keep local"
                            -- and the resolved version is identical to HEAD's tree).
                            -- Skipping this leaves MERGE_HEAD dangling and makes the next
                            -- push fail its ancestry check.
                            lift $ do
                                void $ gitRaw ["commit", "-m", "Merge remote (resolved " ++ show total ++ " conflict(s))"]
                                tell $ "Merge complete. " ++ show total ++ " conflict(s) resolved."
                            -- After auto-resolution, files are already in correct state (we chose local/remote versions)
                            -- Still need to sync any files that changed on remote but weren't in conflict
                            lift $ applyMergeToWorkingDir transport cwd localHead
                            lift $ tell "Syncing binaries... done."
                            maybeRemoteHash <- lift $ Git.getHashFromBundle fetchedBundle
                            case maybeRemoteHash of
                                Just rHash -> lift $ void $ Git.updateRemoteTrackingBranchToHash rHash
                                Nothing    -> return ()
                        else return ()  -- Still have unresolved conflicts, don't sync files yet

printVerifyIssue :: (String -> String) -> Verify.VerifyIssue -> IO ()
printVerifyIssue fmtHash = \case
  Verify.HashMismatch filePath expectedHash actualHash _expectedSize _actualSize -> do
    hPutStrLn stderr $ "[ERROR] Hash mismatch: " ++ toPosix filePath
    hPutStrLn stderr $ "  Expected: " ++ fmtHash expectedHash
    hPutStrLn stderr $ "  Actual:   " ++ fmtHash actualHash
  Verify.Missing filePath ->
    hPutStrLn stderr $ "[ERROR] Missing: " ++ toPosix filePath

-- | Format remote display line (e.g. "origin → black_usb:Backup (physical, connected at E:\)")
formatRemoteDisplay :: FilePath -> String -> Maybe Device.RemoteTarget -> IO String
formatRemoteDisplay cwd name mTarget = case mTarget of
    Just (Device.TargetLocalPath p) -> return (name ++ " → " ++ p ++ " (local path)")
    Just (Device.TargetDevice dev devPath) -> do
        res <- Device.resolveRemoteTarget cwd (Device.TargetDevice dev devPath)
        mInfo <- Device.readDeviceFile cwd dev
        let typ = maybe "unknown" (\i -> case Device.deviceType i of Device.Physical -> "physical"; Device.Network -> "network") mInfo
        case res of
            Device.Resolved mount -> return (name ++ " → " ++ dev ++ ":" ++ devPath ++ " (" ++ typ ++ ", connected at " ++ mount ++ ")")
            Device.NotConnected _ -> return (name ++ " → " ++ dev ++ ":" ++ devPath ++ " (" ++ typ ++ ", NOT CONNECTED)")
    Just (Device.TargetCloud u) -> return (name ++ " → " ++ u ++ " (cloud)")
    Nothing -> return (name ++ " → (no target)")

showRemoteStatusFromBundle :: String -> Maybe String -> IO ()
showRemoteStatusFromBundle name mUrl = do
    maybeLocal <- Git.getLocalHead
    let url = fromMaybe "?" mUrl
    putStrLn $ "* remote " ++ name
    putStrLn $ "  Fetch URL: " ++ url
    putStrLn $ "  Push  URL: " ++ url
    putStrLn ""
    compareHistory maybeLocal fetchedBundle

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
                    localAhead  <- Git.checkIsAhead rHash lHash
                    remoteAhead <- Git.checkIsAhead lHash rHash

                    putStrLn "  Local branch configured for 'bit pull':"
                    putStrLn "    main merges with remote main"
                    putStrLn ""
                    putStrLn "  Local refs configured for 'bit push':"
                    case (localAhead, remoteAhead) of
                        (True, False) -> putStrLn "    main pushes to main (fast-forwardable)"
                        (False, True) -> putStrLn "    main pushes to main (local out of date)"
                        (False, False) -> putStrLn "    main pushes to main (local out of date)"
                        (True, True)   -> putStrLn "    main pushes to main (up to date)"
        _ -> return ()

-- | Extract paths from restore/checkout args, skipping flags and options.
-- Git restore: [options] [--] <pathspec>...
-- Git checkout: [options] [--] <pathspec>...
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
        -- Collect paths (after -- if present, or non-flag args)
        (_, paths) = foldl (\(afterDash, acc) arg ->
            if arg == "--" then (True, acc)
            else if afterDash then (True, arg:acc)
            else if isFlag arg then (False, acc)
            else (False, arg:acc)
            ) (False, []) args
    in reverse paths

-- | Expand pathspecs (e.g. ".", "dir/") to concrete file paths in the index.
expandPathsToFiles :: FilePath -> [String] -> IO [FilePath]
expandPathsToFiles cwd paths = do
    let indexRoot = cwd </> bitIndexPath
    allFiles <- Scan.listMetadataPaths indexRoot
    return $ concatMap (\p ->
        if p == "." || p == "./"
        then allFiles
        else let p' = normalise p
                 pPrefix = p' ++ "/"
                 matches = filter (\f -> let f' = normalise f
                                         in f' == p' || pPrefix `isPrefixOf` (f' ++ "/")) allFiles
             in if null matches then [p] else matches
        ) paths

-- | Restore files from git. For text files, also copies the restored metadata
-- file (which contains the actual content) back to the working directory.
-- Supports full git restore syntax: restore [options] [--] <pathspec>...
doRestore :: [String] -> BitM ExitCode
doRestore args = do
    cwd <- asks envCwd
    code <- lift $ gitRaw ("restore" : args)
    when (code == ExitSuccess) $ do
        let stagedOnly = ("--staged" `elem` args || "-S" `elem` args) &&
                         not ("--worktree" `elem` args || "-W" `elem` args)
        unless stagedOnly $ do
            let rawPaths = restoreCheckoutPaths args
            paths <- lift $ expandPathsToFiles cwd rawPaths
            forM_ paths $ \filePath -> do
                let metaPath = cwd </> bitIndexPath </> filePath
                let workPath = cwd </> filePath
                metaExists <- lift $ fileExistsE metaPath
                when metaExists $ do
                    mcontent <- lift $ readFileE metaPath
                    let isBinaryMetadata = maybe True (\content -> any ("hash: " `isPrefixOf`) (lines content)) mcontent
                    unless isBinaryMetadata $ do
                        lift $ createDirE (takeDirectory workPath)
                        lift $ copyFileE metaPath workPath
    return code

-- | Checkout paths from index/HEAD (git checkout [options] -- <path>).
-- Same as restore for path form: restores metadata, copies text files to working dir.
doCheckout :: [String] -> BitM ExitCode
doCheckout args = do
    let args' = case List.elemIndex "--" args of
          Just _ -> args
          Nothing -> let (opts, paths) = span (\a -> a == "--" || "-" `isPrefixOf` a) args
                     in opts ++ ["--"] ++ paths
    code <- lift $ gitRaw ("checkout" : args')
    when (code == ExitSuccess) $ do
        cwd <- asks envCwd
        let rawPaths = restoreCheckoutPaths args'
        paths <- lift $ expandPathsToFiles cwd rawPaths
        forM_ paths $ \filePath -> do
            let metaPath = cwd </> bitIndexPath </> filePath
            let workPath = cwd </> filePath
            metaExists <- lift $ fileExistsE metaPath
            when metaExists $ do
                mcontent <- lift $ readFileE metaPath
                let isBinaryMetadata = maybe True (\content -> any ("hash: " `isPrefixOf`) (lines content)) mcontent
                unless isBinaryMetadata $ do
                    lift $ createDirE (takeDirectory workPath)
                    lift $ copyFileE metaPath workPath
    return code

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
    -- Classification
  , classifyRemotePath
  , getRcloneRemotes
    -- Volume ops
  , getVolumeRoot
  , getRelativePath
  , detectStorageType
  , getHardwareSerial
  , getVolumeLabel
    -- .bit-store
  , readBitStore
  , writeBitStore
    -- Device/remote files
  , readDeviceFile
  , writeDeviceFile
  , readRemoteFile
  , writeRemoteFile
  , listDeviceNames
  , findDeviceByUuid
    -- Resolution
  , resolveRemoteTarget
  , parseRemoteTarget
  , generateStoreUuid
  ) where

import Data.List (isPrefixOf, intercalate)
import Data.Maybe (fromMaybe, listToMaybe)
import Control.Monad (when, filterM)
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

data ResolveResult
  = Resolved FilePath     -- Runtime path (e.g. E:\Backup)
  | NotConnected String   -- Device not found
  deriving (Show, Eq)

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
        then return (CloudRemote path)
        else return (FilesystemPath path)
    _ -> return (FilesystemPath path)

-- | Get list of configured rclone remote names (without trailing colon)
getRcloneRemotes :: IO [String]
getRcloneRemotes = do
  (code, out, _) <- readProcessWithExitCode "rclone" ["listremotes"] ""
  if code /= ExitSuccess then return []
  else return
    [ takeWhile (/= ':') (takeWhile (/= '\n') line)
    | line <- lines out
    , not (null (trimLine line))
    ]
  where trimLine = reverse . dropWhile (== ' ') . reverse . dropWhile (== ' ')

-- ---------------------------------------------------------------------------
-- Volume operations (platform-specific)
-- ---------------------------------------------------------------------------

-- | Get the volume root for a path (e.g. D:\Backup -> D:\, \\server\share\foo -> \\server\share\)
getVolumeRoot :: FilePath -> IO FilePath
getVolumeRoot path = do
  absPath <- Dir.makeAbsolute path
  if isWindows then return (winVolumeRoot absPath)
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
  if code == ExitSuccess && not (null (trim out))
    then return (trim out)
    else return path
  where
    shellEscape s = "'" ++ concatMap (\c -> if c == '\'' then "'\\''" else [c]) s ++ "'"

addTrailingSep :: FilePath -> FilePath
addTrailingSep p = if not (null p) && last p == pathSeparator then p else p ++ [pathSeparator]

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
  if null drive then return Physical  -- UNC: treat as network
  else do
    (_code, _out, _) <- readProcessWithExitCode "powershell" ["-NoProfile", "-Command",
      "try { (Get-PSDrive -Name " ++ [head drive] ++ " -ErrorAction SilentlyContinue).Root } catch { '' }"] ""
    (code2, out2, _) <- readProcessWithExitCode "powershell" ["-NoProfile", "-Command",
      "[int]([System.IO.DriveInfo]::new('" ++ drive ++ "').DriveType)"] ""
    if code2 == ExitSuccess
      then case trim out2 of
        "4" -> return Network  -- DriveType.Network
        _   -> return Physical
      else return Physical

detectStorageTypeLinux :: FilePath -> IO StorageType
detectStorageTypeLinux _ = do
  -- Read /proc/mounts and check mount type for the path's mount point
  (code, out, _) <- readProcessWithExitCode "findmnt" ["-n", "-o", "FSTYPE", "-T", "/"] ""
  if code == ExitSuccess
    then let fstype = trim out
         in return $ if fstype `elem` ["nfs", "nfs4", "cifs", "smb", "smbfs", "sshfs"]
                    then Network else Physical
    else return Physical

trim :: String -> String
trim = reverse . dropWhile (== ' ') . reverse . dropWhile (== ' ')

-- | Get hardware serial for a physical volume
getHardwareSerial :: FilePath -> IO (Maybe String)
getHardwareSerial volumeRoot
  | isWindows = getHardwareSerialWindows volumeRoot
  | otherwise = getHardwareSerialLinux volumeRoot

getHardwareSerialWindows :: FilePath -> IO (Maybe String)
getHardwareSerialWindows volRoot = do
  let drive = take 1 (filter (`elem` ['A'..'Z'] ++ ['a'..'z']) volRoot)
  if null drive then return Nothing
  else do
    -- Map partition to physical disk via partition number, then get disk serial
    (code, out, _) <- readProcessWithExitCode "wmic" ["diskdrive", "get", "SerialNumber,Index"] ""
    if code /= ExitSuccess then return Nothing
    else do
      (_code2, _out2, _) <- readProcessWithExitCode "wmic" ["path", "win32_logicaldisk", "where", "DeviceID='" ++ drive ++ ":\\'", "get", "VolumeSerialNumber"] ""
      -- VolumeSerialNumber is the FAT/NTFS serial, not disk serial. Use diskdrive.
      let lines' = filter (not . null . trim) (lines out)
          parseSerial = case lines' of
            _:rest -> listToMaybe [ trim (drop 12 l) | l <- rest, length l > 12 ]
            _      -> Nothing
      return (parseSerial)

getHardwareSerialLinux :: FilePath -> IO (Maybe String)
getHardwareSerialLinux _ = do
  (code, out, _) <- readProcessWithExitCode "sh" ["-c", "lsblk -o SERIAL,MOUNTPOINT -n 2>/dev/null | head -20"] ""
  if code == ExitSuccess
    then return (listToMaybe [ trim (takeWhile (/= ' ') l) | l <- lines out, "/" `isPrefixOf` (drop 20 l) ])
    else return Nothing

-- | Get volume label for device name suggestion
getVolumeLabel :: FilePath -> IO (Maybe String)
getVolumeLabel volumeRoot
  | isWindows = getVolumeLabelWindows volumeRoot
  | otherwise = getVolumeLabelLinux volumeRoot

getVolumeLabelWindows :: FilePath -> IO (Maybe String)
getVolumeLabelWindows volRoot = do
  let drive = take 2 (filter (`elem` ['A'..'Z'] ++ ['a'..'z'] ++ ":\\") volRoot)
  if length drive < 2 then return Nothing
  else do
    -- vol is a cmd built-in, not an executable; run via cmd /c
    (code, out, _) <- readProcessWithExitCode "cmd" ["/c", "vol", drive] ""
    if code /= ExitSuccess then return Nothing
    else
      let lines' = lines out
          volLine = listToMaybe [ l | l <- lines', "Volume" `isPrefixOf` l ]
      in return $ case volLine of
        Just l -> let after = dropWhile (/= ' ') (drop 6 l) in Just (trim after)
        _      -> Nothing

getVolumeLabelLinux :: FilePath -> IO (Maybe String)
getVolumeLabelLinux _ = do
  (code, out, _) <- readProcessWithExitCode "sh" ["-c", "lsblk -o LABEL,MOUNTPOINT -n 2>/dev/null | head -5"] ""
  if code == ExitSuccess
    then return (listToMaybe [ trim (takeWhile (/= ' ') l) | l <- lines out, not (null (trim l)) ])
    else return Nothing

-- ---------------------------------------------------------------------------
-- .bit-store (on device at volume root)
-- ---------------------------------------------------------------------------

bitStoreFileName :: FilePath
bitStoreFileName = ".bit-store"

readBitStore :: FilePath -> IO (Maybe UUID)
readBitStore volumeRoot = do
  let storePath = volumeRoot </> bitStoreFileName
  exists <- Dir.doesFileExist storePath
  if not exists then return Nothing
  else do
    -- Use strict ByteString reading to avoid Windows file locking issues
    bs <- BS.readFile storePath
    let content = case decodeUtf8' bs of
          Left _ -> ""
          Right txt -> T.unpack txt
    return (parseBitStoreUuid content)

parseBitStoreUuid :: String -> Maybe UUID
parseBitStoreUuid content =
  listToMaybe [ fromString (trim (drop 5 line)) | line <- lines content, "uuid:" `isPrefixOf` line ]
  >>= id  -- join: Maybe (Maybe UUID) -> Maybe UUID

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
  return ()

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
  if not exists then return Nothing
  else do
    -- Use strict ByteString reading to avoid Windows file locking issues
    bs <- BS.readFile path
    let content = case decodeUtf8' bs of
          Left _ -> ""
          Right txt -> T.unpack txt
    return (parseDeviceFile content)

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
  if not exists then return []
  else filter (not . null) <$> Dir.listDirectory dir

findDeviceByUuid :: FilePath -> UUID -> IO (Maybe String)
findDeviceByUuid repoRoot targetUuid = do
  names <- listDeviceNames repoRoot
  foldr go (return Nothing) names
  where
    go name acc = do
      mInfo <- readDeviceFile repoRoot name
      case mInfo of
        Just info | deviceUuid info == targetUuid -> return (Just name)
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
  if not exists then return Nothing
  else do
    -- Use strict ByteString reading to avoid Windows file locking issues
    bs <- BS.readFile path
    let content = case decodeUtf8' bs of
          Left _ -> ""
          Right txt -> T.unpack txt
    let raw = parseRemoteFile content
    case raw of
      Nothing -> return Nothing
      Just (ParsedLocal p) -> return (Just (TargetLocalPath p))
      Just (ParsedCloud url) -> return (Just (TargetCloud url))
      Just (ParsedDevice device relPath) -> do
        mDev <- readDeviceFile repoRoot device
        return $ Just $ case mDev of
          Just _ -> TargetDevice device relPath
          Nothing -> TargetCloud (device ++ ":" ++ relPath)

writeRemoteFile :: FilePath -> String -> RemoteTarget -> IO ()
writeRemoteFile repoRoot remoteName target = do
  Dir.createDirectoryIfMissing True (repoRoot </> bitRemotesDir)
  let path = repoRoot </> bitRemotesDir </> remoteName
  let line = case target of
        TargetCloud url -> "target: " ++ url
        TargetDevice dev p -> "target: " ++ dev ++ ":" ++ p
        TargetLocalPath p -> "target: local:" ++ p
  -- Use atomic write for crash safety and Windows compatibility
  atomicWriteFile path (encodeUtf8 (T.pack line))

-- | Parse a target string (e.g. "black_usb:Backup" or "gdrive:Projects/foo")
parseRemoteTarget :: String -> RemoteTarget
parseRemoteTarget s = case break (== ':') s of
  (prefix, _:rest) | not (null prefix) -> TargetDevice prefix (trim rest)
  _ -> TargetCloud s

-- ---------------------------------------------------------------------------
-- Resolution: device:path -> runtime path
-- ---------------------------------------------------------------------------

resolveRemoteTarget :: FilePath -> RemoteTarget -> IO ResolveResult
resolveRemoteTarget _repoRoot (TargetCloud url) = return (Resolved url)
resolveRemoteTarget _repoRoot (TargetLocalPath p) = return (Resolved p)
resolveRemoteTarget repoRoot (TargetDevice deviceName relPath) = do
  mInfo <- readDeviceFile repoRoot deviceName
  case mInfo of
    Nothing -> return (NotConnected ("Device '" ++ deviceName ++ "' not found in .rgit/devices/"))
    Just info -> do
      mMount <- resolveDevice info
      case mMount of
        Nothing -> return (NotConnected ("Device '" ++ deviceName ++ "' is not connected"))
        Just mountRoot -> do
          let fullPath = mountRoot </> relPath
          return (Resolved fullPath)

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
  return (listToMaybe found)

resolveNetworkDevice :: DeviceInfo -> IO (Maybe FilePath)
resolveNetworkDevice info = do
  mounts <- getNetworkMountPoints
  found <- filterM (checkMountForDevice info) mounts
  return (listToMaybe found)

checkMountForDevice :: DeviceInfo -> FilePath -> IO Bool
checkMountForDevice info mountRoot = do
  mStoreUuid <- readBitStore mountRoot
  return $ mStoreUuid == Just (deviceUuid info)

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
  if code /= ExitSuccess then return ["/"]
  else return (filter (not . null) (lines out))

getWindowsPhysicalMounts :: IO [FilePath]
getWindowsPhysicalMounts = do
  (code, out, _) <- readProcessWithExitCode "wmic" ["logicaldisk", "where", "DriveType=2 or DriveType=3", "get", "DeviceID"] ""
  if code /= ExitSuccess then return []
  else return
    [ trim l ++ "\\"
    | l <- lines out
    , let t = trim l
    , length t == 2
    , last t == ':'
    ]

getWindowsNetworkMounts :: IO [FilePath]
getWindowsNetworkMounts = do
  (code, out, _) <- readProcessWithExitCode "wmic" ["logicaldisk", "where", "DriveType=4", "get", "DeviceID"] ""
  if code /= ExitSuccess then return []
  else return
    [ trim l ++ "\\"
    | l <- lines out
    , let t = trim l
    , length t == 2
    , last t == ':'
    ]
```

---

## Bit/DevicePrompt.hs

**Path:** `Bit/DevicePrompt.hs`

*Source file.*

```haskell
{-# LANGUAGE LambdaCase #-}

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
trim = reverse . dropWhile isSpace . reverse . dropWhile isSpace

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
    NonInteractive -> return defaultName
    Interactive ask -> do
      putStrLn "This path is on a storage device."
      putStrLn "bit identifies devices, not drive letters. The remote will stay linked"
      putStrLn "to this device even if the drive letter changes."
      putStrLn ""
      putStr $ "Name this device [" ++ defaultName ++ "]: "
      hFlush stdout
      line <- ask
      let name = trim line
      return $ if null name then defaultName else sanitizeDeviceName name
  unless (isValidDeviceName finalName) $ do
    hPutStrLn stderr "fatal: Device name must be alphanumeric with underscores/hyphens only."
    exitWith (ExitFailure 1)
  exists <- nameExists finalName
  when exists $ do
    hPutStrLn stderr ("fatal: Device '" ++ finalName ++ "' already exists.")
    exitWith (ExitFailure 1)
  return finalName

-- | Production entry point: detects TTY or BIT_USE_STDIN for testing.
acquireDeviceNameAuto
  :: Maybe String
  -> (String -> IO Bool)
  -> IO String
acquireDeviceNameAuto mLabel nameExists = do
  isTTY <- hIsTerminalDevice stdin
  useStdin <- (== Just "1") <$> lookupEnv "BIT_USE_STDIN"
  let src = if isTTY
            then Interactive getLine
            else if useStdin
                 then Interactive getLine  -- For tests: pipe input to stdin
                 else NonInteractive
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
      , let lHash = lFiles Map.! p
      , let rHash = rFiles Map.! p
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
      , let newPath = head (Set.toList localPathsWithHash)
      , Map.member newPath lFiles
      ]

formatDiff :: GitDiff -> String
formatDiff (Added f)      = "[+] Added:    " ++ f.filePath
formatDiff (Deleted f)    = "[-] Deleted:  " ++ f.filePath
formatDiff (Modified f)   = "[*] Modified: " ++ f.filePath
formatDiff (Renamed o n)  = "[M] Moved:    " ++ o.filePath ++ " -> " ++ n.filePath
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
interpretPure (Pure a) = return a
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
    case Map.lookup src (pureFiles env) of
      Just content -> modify (\e -> e { pureFiles = Map.insert dest content (pureFiles e) })
      Nothing -> return ()
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
{-# LANGUAGE LambdaCase #-}
{-# LANGUAGE OverloadedStrings #-}

module Bit.Fsck
  ( doFsck
  ) where

import qualified Bit.Verify as Verify
import qualified Bit.Utils as Utils
import qualified Internal.Git as Git
import Bit.Concurrency (Concurrency(..))
import System.FilePath ((</>))
import System.Exit (ExitCode(..), exitWith)
import Control.Monad (when)
import System.IO (hPutStr, hPutStrLn, hFlush, hSetBuffering, BufferMode(..), stderr, hIsTerminalDevice)
import Data.IORef (newIORef, readIORef, IORef)
import Control.Concurrent (forkIO, threadDelay, killThread)
import Control.Exception (finally)
import Bit.Progress (reportProgress, clearProgress)

-- | Run local-only integrity check in the spirit of git fsck: no network.
-- [1/2] Local working tree vs local metadata (rgit equivalent of checking objects).
-- [2/2] git fsck on .rgit/index/.git (metadata history integrity).
-- Prints one line per problem (git-style: "missing <path>", "hash mismatch <path>")
-- and passes through git fsck output. Exits 1 if any check finds issues.
doFsck :: FilePath -> Concurrency -> IO ()
doFsck cwd concurrency = do
  hSetBuffering stderr NoBuffering
  
  -- [1/2] Working tree vs local metadata
  -- Get file count first
  let indexDir = cwd </> ".bit/index"
  meta <- Verify.loadBinaryMetadata indexDir concurrency
  let fileCount = length meta
  
  -- Run verification with progress if enough files
  (actualCount, localIssues) <- if fileCount > 5
    then do
      isTTY <- hIsTerminalDevice stderr
      counter <- newIORef (0 :: Int)
      let shouldShowProgress = isTTY
      
      -- Start progress reporter thread if in TTY
      reporterThread <- if shouldShowProgress
        then Just <$> forkIO (fsckProgressLoop counter fileCount)
        else return Nothing
      
      -- Run verification with progress
      result <- finally
        (Verify.verifyLocal cwd (Just counter) concurrency)
        (do
          -- Clean up: kill reporter thread and clear line
          maybe (return ()) killThread reporterThread
          when shouldShowProgress clearProgress
        )
      return result
    else
      -- Few files, no progress needed
      Verify.verifyLocal cwd Nothing concurrency
  
  let localOk = null localIssues
  if localOk
    then hPutStrLn stderr $ "[1/2] Checked " ++ show actualCount ++ " files. OK."
    else do
      hPutStrLn stderr $ "[1/2] Checked " ++ show actualCount ++ " files. Issues found:"
      mapM_ (printIssue Utils.toPosix) localIssues
      hFlush stderr

  -- [2/2] Metadata history (git fsck in .rgit/index/.git)
  hPutStrLn stderr "[2/2] Checking metadata history..."
  (gitCode, gitOut, gitErr) <- Git.fsck
  let gitOk = gitCode == ExitSuccess
  if gitOk
    then hPutStrLn stderr "[2/2] Metadata history OK."
    else do
      putStr gitOut
      hPutStr stderr gitErr

  when (not localOk || not gitOk) $
    exitWith (ExitFailure 1)

  where
    printIssue :: (FilePath -> FilePath) -> Verify.VerifyIssue -> IO ()
    printIssue toP = \case
      Verify.HashMismatch path _ _ _ _ ->
        hPutStrLn stderr $ "hash mismatch " ++ toP path
      Verify.Missing path ->
        hPutStrLn stderr $ "missing " ++ toP path
    
    -- Progress reporter loop for fsck operation
    fsckProgressLoop :: IORef Int -> Int -> IO ()
    fsckProgressLoop counter total = go
      where
        go = do
          n <- readIORef counter
          let pct = (n * 100) `div` max 1 total
          reportProgress $ "[1/2] Checking working tree: " ++ show n ++ "/" ++ show total ++ " (" ++ show pct ++ "%)"
          threadDelay 100000  -- 100ms
          when (n < total) go
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
import Data.List (isPrefixOf, isInfixOf)
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
          in if not (null rest) && last rest == '"' then init rest else rest
      | length s >= 2 && head s == '"' && last s == '"' = init (tail s)
      | otherwise = s
    trim = dropWhile isSpaceChar . reverse . dropWhile isSpaceChar . reverse
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
      case decodeUtf8' bs of
        Left _ -> pure Nothing  -- Binary file, not valid metadata
        Right txt -> pure (parseMetadata (T.unpack txt))

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
            return (Hash (T.pack "md5:" <> md5hex))
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
  case decodeUtf8' bs of
    Left _ -> return False  -- Binary file, no conflict markers possible
    Right txt -> return $ any (\m -> m `isInfixOf` T.unpack txt) conflictMarkers

listAllFiles :: FilePath -> IO [FilePath]
listAllFiles dir = do
  entries <- listDirectory dir
  concat <$> mapM (\name -> do
    let full = dir </> name
    isDir <- doesDirectoryExist full
    if isDir then listAllFiles full else do
      isFile <- doesFileExist full
      return (if isFile then [full] else [])) entries

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
import Bit.Plan (RcloneAction(..), planAction)
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
  in  map planAction diffs

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
  ) where

import Bit.Diff (GitDiff(..), LightFileEntry(..))
import Bit.Types

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

import Prelude (FilePath, IO, String, Maybe(..), Either(..), ($), pure, return, error)
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
        
        return (exitCode, out, err)
      
      _ -> error "Bit.Process.readProcessStrict: failed to create pipes"
  where
    cleanupProcess (mStdin, mStdout, _mStderr, ph) = do
      -- Try to close any handles that are still open
      -- (BS.hGetContents closes the handle, but we need cleanup for stdin)
      case mStdin of
        Just h -> void (try (hClose h) :: IO (Either SomeException ()))
        Nothing -> pure ()
      
      case mStdout of
        Just h -> void (try (hClose h) :: IO (Either SomeException ()))
        Nothing -> pure ()
      
      case _mStderr of
        Just h -> void (try (hClose h) :: IO (Either SomeException ()))
        Nothing -> pure ()
      
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
        
        return (exitCode, out)
      
      _ -> error "Bit.Process.readProcessStrictWithStderr: failed to create pipes"
  where
    cleanupProcess (mStdin, mStdout, _mStderr, ph) = do
      -- Try to close any handles that are still open
      case mStdin of
        Just h -> void (try (hClose h) :: IO (Either SomeException ()))
        Nothing -> pure ()
      
      case mStdout of
        Just h -> void (try (hClose h) :: IO (Either SomeException ()))
        Nothing -> pure ()
      
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

-- | Resolve a remote name to a Remote. Checks:
--   1. .rgit/remotes/<name> (device resolution via Device.hs)
--   2. Git config (git remote get-url <name>)
-- Returns Nothing if remote doesn't exist or device is not connected.
resolveRemote :: FilePath -> String -> IO (Maybe Remote)
resolveRemote cwd name = do
    -- Try .rgit/remotes/<name> first (device-aware resolution)
    mTarget <- Device.readRemoteFile cwd name
    case mTarget of
        Just target -> do
            res <- Device.resolveRemoteTarget cwd target
            case res of
                Device.Resolved url -> return (Just (mkRemote name url))
                Device.NotConnected _ -> return Nothing
        Nothing -> do
            -- Fall back to git remote URL
            mUrl <- Git.getRemoteUrl name
            case mUrl of
                Just url | not (null url) -> return (Just (mkRemote name url))
                _ -> return Nothing

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
  ( RemoteWorkspace(..)
  , initRemoteWorkspace
  , remoteWorkspacePath
  ) where

import Bit.Types (FileEntry(..), EntryKind(..))
import Bit.Remote (Remote, remoteUrl)
import qualified Bit.Remote.Scan as Remote.Scan
import Bit.Scan (hashAndClassifyFile, binaryExtensions)
import qualified Internal.ConfigFile as ConfigFile
import Internal.ConfigFile (TextConfig)
import qualified Internal.Transport as Transport
import Bit.Utils (isBitPath, atomicWriteFileStr)
import Bit.Internal.Metadata (MetaContent(..), serializeMetadata)
import System.FilePath ((</>), takeExtension, takeDirectory)
import System.Directory 
    ( doesDirectoryExist
    , createDirectoryIfMissing
    , getTemporaryDirectory
    , removeDirectoryRecursive
    )
import System.Exit (ExitCode(..), exitWith)
import System.IO (hPutStrLn, stderr)
import System.Process (callProcess)
import Control.Monad (when, forM, forM_)
import Data.Char (toLower)
import Data.List (partition)
import Control.Exception (catch, SomeException)

-- | A remote workspace contains metadata and git repo for a remote
data RemoteWorkspace = RemoteWorkspace
  { wsPath :: FilePath
  , wsRemote :: Remote
  } deriving (Show)

-- | Path to the remote workspace for a given remote name
remoteWorkspacePath :: FilePath -> String -> FilePath
remoteWorkspacePath cwd remName = cwd </> ".bit" </> "remote-workspaces" </> remName

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
            || map toLower (takeExtension (path fe)) `elem` binaryExtensions
        _ -> True  -- directories are not text candidates

-- | Download text candidate files from remote, classify them, return updated FileEntries.
classifyRemoteTextCandidates :: Remote -> TextConfig -> [FileEntry] -> IO [FileEntry]
classifyRemoteTextCandidates remote config candidates = do
    -- Create temp directory for downloads
    tempDir <- getTemporaryDirectory >>= \t -> do
        let d = t </> "bit-classify-remote"
        createDirectoryIfMissing True d
        return d

    -- Download and classify each candidate file
    classifiedEntries <- forM candidates $ \fe -> do
        let remotePath = path fe  -- Already normalized by rclone
        let localPath = tempDir </> path fe
        createDirectoryIfMissing True (takeDirectory localPath)

        -- Download via rclone
        code <- Transport.copyFromRemote remote remotePath localPath
        case code of
            ExitSuccess -> do
                -- Classify using existing function
                case kind fe of
                    File{fSize} -> do
                        (h, isText) <- hashAndClassifyFile localPath fSize config
                        return fe { kind = File { fHash = h, fSize = fSize, fIsText = isText } }
                    _ -> return fe
            _ -> do
                -- Download failed — treat as binary, keep rclone hash
                hPutStrLn stderr $ "Warning: Could not download " ++ path fe ++ " for classification, treating as binary."
                return fe

    -- Cleanup temp dir
    removeDirectoryRecursive tempDir `catch` (\(_ :: SomeException) -> return ())

    return classifiedEntries

-- | Initialize a remote workspace by scanning the remote and building metadata
initRemoteWorkspace :: FilePath -> Remote -> String -> IO ()
initRemoteWorkspace cwd remote remName = do
    let wsPath = remoteWorkspacePath cwd remName
    let wsGit = wsPath </> ".git"

    -- Check if workspace already exists
    exists <- doesDirectoryExist wsGit
    when exists $ do
        hPutStrLn stderr $ "Remote workspace '" ++ remName ++ "' already exists."
        hPutStrLn stderr $ "Use 'bit @" ++ remName ++ " status' to see its state, or delete .bit/remote-workspaces/" ++ remName ++ "/ to start over."
        exitWith (ExitFailure 1)

    putStrLn $ "Scanning remote '" ++ remName ++ "' (" ++ remoteUrl remote ++ ")..."

    -- Step 1: Fetch file list from remote
    result <- Remote.Scan.fetchRemoteFiles remote
    case result of
        Left err -> do
            hPutStrLn stderr $ "Error scanning remote: " ++ show err
            exitWith (ExitFailure 1)
        Right remoteFiles -> do
            let files = filter (not . isBitPath . path) remoteFiles
            putStrLn $ "Found " ++ show (length files) ++ " files on remote."

            -- Step 2: Partition into binary (certain) and text candidates
            config <- ConfigFile.readTextConfig
            let (definitelyBinary, textCandidates) = partitionFiles config files

            putStrLn $ "  " ++ show (length definitelyBinary) ++ " binary files (by size/extension)"
            putStrLn $ "  " ++ show (length textCandidates) ++ " small files to classify..."

            -- Step 3: Download text candidates to temp dir and classify
            classifiedFiles <- if null textCandidates
                then return []
                else classifyRemoteTextCandidates remote config textCandidates

            -- Step 4: Merge results
            let allFiles = definitelyBinary ++ classifiedFiles

            -- Step 5: Create workspace and write metadata
            putStrLn "Building metadata workspace..."
            createDirectoryIfMissing True wsPath
            
            -- For text files: we need the actual content, but we just classified from remote
            -- For binary files: we just need metadata (hash+size)
            -- Split the files accordingly
            let (textFiles, binaryFiles) = partition isTextFile allFiles
                  where
                    isTextFile fe = case kind fe of
                        File{fIsText} -> fIsText
                        _ -> False
            
            -- For text files from the classified set: download them again to the workspace
            -- (we could optimize by keeping the temp files, but this is simpler and more robust)
            putStrLn $ "Downloading " ++ show (length textFiles) ++ " text files..."
            forM_ textFiles $ \fe -> do
                let localPath = wsPath </> path fe
                createDirectoryIfMissing True (takeDirectory localPath)
                code <- Transport.copyFromRemote remote (path fe) localPath
                when (code /= ExitSuccess) $
                    hPutStrLn stderr $ "Warning: Failed to download text file " ++ path fe

            -- For binary files: write metadata
            forM_ binaryFiles $ \fe -> do
                let metaPath = wsPath </> path fe
                createDirectoryIfMissing True (takeDirectory metaPath)
                case kind fe of
                    File{fHash, fSize} ->
                        atomicWriteFileStr metaPath (serializeMetadata (MetaContent fHash fSize))
                    _ -> return ()

            -- Step 6: Initialize git repo in workspace
            putStrLn "Initializing git repository..."
            callProcess "git" ["-C", wsPath, "init", "--initial-branch=main"]
            callProcess "git" ["-C", wsPath, "config", "core.quotePath", "false"]

            let textCount = length textFiles
            let binCount = length binaryFiles
            putStrLn $ "Remote workspace initialized: " ++ show textCount ++ " text, " ++ show binCount ++ " binary files."
            putStrLn $ "Next steps:"
            putStrLn $ "  bit @" ++ remName ++ " add ."
            putStrLn $ "  bit @" ++ remName ++ " commit -m \"Initial commit\""
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
    { path = normalise rf.rfPath
    , kind = File { fHash = Hash (T.pack ("md5:" ++ md5)), fSize = rf.rfSize, fIsText = False }
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

import Bit.Types (Hash(..), HashAlgo(..), FileEntry(..), EntryKind(..), hashToText)
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
import Data.List
import Data.Maybe (listToMaybe)
import qualified Data.ByteString as BS
import Control.Monad (void, when, forM_)
import Data.Text.Encoding (decodeUtf8, decodeUtf8', encodeUtf8)
import Data.Char (toLower)
import qualified Internal.ConfigFile as ConfigFile
import Bit.Utils (atomicWriteFileStr)
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

-- | Single-pass file hash and classification. Returns (hash, isText).
-- For large files or binary extensions: streams hash only, returns isText=False.
-- For others: reads first 8KB for text classification, then streams remaining chunks for hash.
hashAndClassifyFile :: FilePath -> Integer -> ConfigFile.TextConfig -> IO (Hash 'MD5, Bool)
hashAndClassifyFile filePath size config = do
    let ext = map toLower (takeExtension filePath)
    
    -- Fast path: large files or known binary extensions - just stream hash
    if size >= ConfigFile.textSizeLimit config || ext `elem` binaryExtensions
        then do
            h <- streamHash filePath
            return (h, False)
        else
            -- Single-pass: read first 8KB for classification, continue streaming for hash
            withFile filePath ReadMode $ \handle -> do
                firstChunk <- BS.hGet handle 8192
                let isText = not (BS.elem 0 firstChunk) &&
                             case decodeUtf8' firstChunk of
                                 Left _ -> False
                                 Right _ -> True
                
                -- Continue streaming hash from where we left off
                let loop !ctx = do
                        eof <- hIsEOF handle
                        if eof
                            then do
                                let md5hex = decodeUtf8 (encode (MD5.finalize ctx))
                                return (Hash (T.pack "md5:" <> md5hex))
                            else do
                                chunk <- BS.hGet handle 65536
                                loop (MD5.update ctx chunk)
                
                -- Start with first chunk already included
                h <- loop (MD5.update MD5.init firstChunk)
                return (h, isText)
  where
    -- Stream hash for files we're not classifying
    streamHash fp = withFile fp ReadMode $ \h -> do
        let loop !ctx = do
                eof <- hIsEOF h
                if eof
                    then do
                        let md5hex = decodeUtf8 (encode (MD5.finalize ctx))
                        return (Hash (T.pack "md5:" <> md5hex))
                    else do
                        chunk <- BS.hGet h 65536
                        loop (MD5.update ctx chunk)
        loop MD5.init

-- | Cache entry for file metadata to skip re-hashing unchanged files
data CacheEntry = CacheEntry
  { ceMtime :: Integer
  , ceSize :: Integer
  , ceHash :: Hash 'MD5
  , ceIsText :: Bool
  } deriving (Show, Eq)

-- | Serialize cache entry to string format
serializeCacheEntry :: CacheEntry -> String
serializeCacheEntry ce =
  "mtime: " ++ show (ceMtime ce) ++ "\n"
  ++ "size: " ++ show (ceSize ce) ++ "\n"
  ++ "hash: " ++ T.unpack (hashToText (ceHash ce)) ++ "\n"
  ++ "isText: " ++ show (ceIsText ce) ++ "\n"

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
  if null hashVal
    then Nothing
    else Just CacheEntry
      { ceMtime = mtime
      , ceSize = size
      , ceHash = Hash (T.pack hashVal)
      , ceIsText = isText
      }
  where
    trim = dropWhile isSpaceChar . reverse . dropWhile isSpaceChar . reverse
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
    then return Nothing
    else do
      -- Use strict bytestring reading to avoid lazy file handle issues on Windows
      bs <- BS.readFile cachePath
      case decodeUtf8' bs of
        Left _ -> return Nothing
        Right txt -> return (parseCacheEntry (T.unpack txt))

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
normalizePath = map (\c -> if c == '\\' then '/' else c) . filter (/= '\r')

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
        then return Set.empty
        else do
            -- Use strict ByteString reading to avoid lazy file handle issues on Windows
            bs <- BS.readFile gitignorePath
            let content = case decodeUtf8' bs of
                    Left _ -> ""  -- Invalid UTF-8, treat as empty
                    Right txt -> T.unpack txt
            let whitespace = ['\r', '\n', ' '] :: [Char]
            let patterns = filter (not . null) $ 
                           filter (not . ("#" `isPrefixOf`)) $  -- Skip comments
                           map (filter (`notElem` whitespace)) (lines content)
            let isIgnored p = any (`matchesPattern` p) patterns
            return $ Set.fromList $ filter isIgnored paths

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
    if not bitExists then return []
    else do
      -- Read config once for all files
      config <- ConfigFile.readTextConfig
    
      -- First pass: collect all paths (without hashing)
      allPaths <- collectPaths root
    
      -- Filter through git check-ignore
      let filePaths = [p | (p, False) <- allPaths]  -- Only check files, not directories
      ignoredSet <- checkIgnoredFiles root filePaths
    
      -- Separate directories from files to hash
      let (dirs, _files) = partition snd allPaths
          dirEntries = [FileEntry { path = rel, kind = Directory } | (rel, _) <- dirs]
          filesToHash = [(rel, root </> rel) | (rel, False) <- allPaths
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
                  -- Cache hit: reuse hash and isText
                  atomicModifyIORef' counter (\n -> (n + 1, ()))
                  return $ FileEntry
                      { path = rel
                      , kind = File { fHash = ceHash ce, fSize = fromIntegral size, fIsText = ceIsText ce }
                      }
                _ -> do
                  -- Cache miss: hash the file, save cache entry
                  (h, isText) <- hashAndClassifyFile fullPath (fromIntegral size) config
                  saveCacheEntry root rel (CacheEntry mtimeInt (fromIntegral size) h isText)
                  atomicModifyIORef' counter (\n -> (n + 1, ()))
                  return $ FileEntry
                      { path = rel
                      , kind = File { fHash = h, fSize = fromIntegral size, fIsText = isText }
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
    
      return $ dirEntries ++ fileEntries
  where
    collectPaths :: FilePath -> IO [(FilePath, Bool)]
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
            pure ((rel, True) : childPaths)
        else pure [(rel, False)]

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
      let parentDirs = Set.fromList [takeDirectory (path e) | e <- files]
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
          else return Nothing
    
      -- Third pass: write files in parallel (bounded concurrency)
      caps <- getNumCapabilities
      let concurrency = max 4 (caps * 4)
    
      let writeWithProgress entry = do
              let metaPath = metaRoot </> path entry
              case kind entry of
                File { fHash, fSize, fIsText } -> do
                  -- Check if file is unchanged before writing
                  needsWrite <- shouldWriteFile root metaPath entry fHash fSize fIsText
                  if needsWrite
                    then do
                      if fIsText
                        then do
                          -- For text files, copy the actual content directly
                          let actualPath = root </> path entry
                          copyFileWithMetadata actualPath metaPath
                        else do
                          -- For binary files, write metadata (hash + size). Spec: raw hash value; atomic write.
                          atomicWriteFileStr metaPath $
                            serializeMetadata (MetaContent fHash fSize)
                      atomicModifyIORef' counter (\n -> (n + 1, ()))
                    else do
                      atomicModifyIORef' skipped (\n -> (n + 1, ()))
                      atomicModifyIORef' counter (\n -> (n + 1, ()))
                Directory -> return ()  -- Already handled in first pass
                Symlink _ -> return ()  -- Symlinks handled separately
    
      finally
          (void $ mapConcurrentlyBounded concurrency writeWithProgress files)
          (do
              -- Clean up: kill reporter thread and finalize progress line
              maybe (return ()) killThread reporterThread
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
      let dirs = [path e | e <- es, case kind e of Directory -> True; _ -> False]
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
shouldWriteFile :: FilePath -> FilePath -> FileEntry -> Hash 'MD5 -> Integer -> Bool -> IO Bool
shouldWriteFile root metaPath entry fHash fSize fIsText = do
  exists <- doesFileExist metaPath
  if not exists
    then return True  -- File doesn't exist, must write
    else if fIsText
      then do
        -- For text files: compare mtime and size of source vs destination
        let sourcePath = root </> path entry
        sourceMtime <- getModificationTime sourcePath
        sourceSize <- getFileSize sourcePath
        destMtime <- getModificationTime metaPath
        destSize <- getFileSize metaPath
        -- Write if mtime or size differs
        return (sourceMtime /= destMtime || sourceSize /= destSize)
      else do
        -- For binary files: read existing metadata and compare hash/size
        existing <- readMetadataOrComputeHash metaPath
        case existing of
          Nothing -> return True  -- Failed to read, must write
          Just (MetaContent existingHash existingSize) ->
            -- Write if hash or size differs
            return (existingHash /= fHash || existingSize /= fSize)

-- | Parse a metadata file (hash/size lines) or read a text file and compute hash/size.
-- Returns Nothing if file is missing or invalid.
-- Text files in .rgit/index/ contain actual content; binary files contain metadata.
readMetadataFile :: FilePath -> IO (Maybe (Hash 'MD5, Integer))
readMetadataFile fp = fmap (\mc -> (metaHash mc, metaSize mc)) <$> readMetadataOrComputeHash fp

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
          return (if isFile then [rel] else [])

-- | Get hash and size of a file. Returns Nothing if file is missing or not a regular file.
getFileHashAndSize :: FilePath -> FilePath -> IO (Maybe (Hash 'MD5, Integer))
getFileHashAndSize root relPath = do
  let full = root </> relPath
  exists <- doesFileExist full
  if not exists then return Nothing
  else do
    h <- hashFile full
    sz <- getFileSize full
    return (Just (h, fromIntegral sz))
```

---

## Bit/Types.hs

**Path:** `Bit/Types.hs`

*Source file.*

```haskell
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE DeriveGeneric #-}
{-# LANGUAGE KindSignatures #-}

module Bit.Types
  ( Path
  , HashAlgo(..)
  , Hash(..)
  , hashToText
  , EntryKind(..)
  , FileEntry(..)
  , syncHash
  , BitEnv(..)
  , BitM
  , runBitM
  ) where

import Control.Monad.Trans.Reader (ReaderT, runReaderT)
import Data.Text (Text)
import GHC.Generics (Generic)
import Bit.Remote (Remote)

type Path = String

data HashAlgo = MD5 | SHA256
  deriving (Show, Eq, Generic)

newtype Hash (a :: HashAlgo) = Hash Text
  deriving (Show, Eq, Ord, Generic)

hashToText :: Hash a -> Text
hashToText (Hash t) = t

data EntryKind
  = File { fHash :: Hash 'MD5, fSize :: Integer, fIsText :: Bool }
  | Directory
  | Symlink FilePath
  deriving (Show, Eq, Generic)

-- | Hash to use for sync diff (MD5). File has one hash; Directory/Symlink have none.
syncHash :: EntryKind -> Maybe (Hash 'MD5)
syncHash (File h _ _) = Just h
syncHash _           = Nothing

data FileEntry = FileEntry
  { path :: FilePath
  , kind :: EntryKind
  } deriving (Show, Eq, Generic)

data BitEnv = BitEnv
    { envCwd            :: FilePath
    , envLocalFiles     :: [FileEntry]
    , envRemote         :: Maybe Remote
    , envForce          :: Bool
    , envForceWithLease :: Bool
    , envSkipVerify     :: Bool
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
import Bit.Types (FileEntry(..))
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
filterOutBitPaths = filter (\e -> not (isBitPath e.path))

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
  , loadBinaryMetadata
  , loadMetadataFromBundle
  , MetadataEntry(..)
  , MetadataSource(..)
  , loadMetadata
  , entryPath
  , binaryEntries
  , allEntryPaths
  ) where

import Bit.Types (Hash(..), HashAlgo(..), Path, FileEntry(..), EntryKind(..), syncHash, hashToText)
import Bit.Utils (filterOutBitPaths)
import Bit.Concurrency (Concurrency(..), runConcurrently, ioConcurrency)
import System.FilePath ((</>), makeRelative, normalise)
import System.Directory (doesFileExist, listDirectory, doesDirectoryExist, removeFile)
import Data.List (isPrefixOf)
import Data.Maybe (maybeToList)
import qualified Data.ByteString as BS
import qualified Data.Text as T
import qualified Internal.Git as Git
import Bit.Internal.Metadata (MetaContent(..), parseMetadata, readMetadataOrComputeHash, hashFile, parseMetadataFile)
import qualified Bit.Remote.Scan as Remote.Scan
import qualified Bit.Remote
import qualified Internal.Transport as Transport
import Internal.Config (fetchedBundle, bitIndexPath, bundleCwdPath, fromCwdPath, BundleName)
import System.Process (readProcessWithExitCode)
import System.Exit (ExitCode(..))
import Data.Char (isSpace)
import System.IO (hPutStrLn, stderr)
import Control.Monad (when)
import qualified Data.Map as Map
import qualified Data.Set as Set
import Data.IORef (IORef, atomicModifyIORef')

-- | Result of comparing one file to metadata.
data VerifyIssue
  = HashMismatch Path String String Integer Integer  -- path, expectedHash, actualHash, expectedSize, actualSize
  | Missing Path                                      -- path (in metadata but no actual file)
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

-- | Extract verifiable (binary) entries only.
binaryEntries :: [MetadataEntry] -> [(Path, Hash 'MD5, Integer)]
binaryEntries = concatMap go
  where go (BinaryEntry p h s) = [(p, h, s)]
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
resolveConcurrency Sequential = return 1
resolveConcurrency (Parallel 0) = ioConcurrency
resolveConcurrency (Parallel n) = return n

-- | List all regular files under dir, with paths relative to baseDir.
listFilesRecursive :: FilePath -> FilePath -> IO [FilePath]
listFilesRecursive baseDir dir = do
  entries <- listDirectory dir
  concat <$> mapM (\name -> do
    let full = dir </> name
    isDir <- doesDirectoryExist full
    if isDir
      then listFilesRecursive baseDir full
      else return [makeRelative baseDir full]
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
    then return []
    else do
      relPaths <- listFilesRecursive indexDir indexDir
      let userPaths = filter isUserFile relPaths
      bound <- resolveConcurrency concurrency
      runConcurrently (Parallel bound) (readEntryFromFilesystem indexDir) userPaths

loadMetadata (FromCommit commitHash) _concurrency = do
  -- ls-tree at ROOT level (no prefix!) to enumerate all files
  (code, out, _) <- readProcessWithExitCode "git"
    [ "-C", bitIndexPath, "ls-tree", "-r", "--name-only", commitHash ] ""
  if code /= ExitSuccess
    then return []
    else do
      let paths = filter isUserFile $ filter (not . null) $ lines out
      mapM (readEntryFromCommit commitHash) paths

-- | Read a single metadata entry from a filesystem path.
readEntryFromFilesystem :: FilePath -> FilePath -> IO MetadataEntry
readEntryFromFilesystem indexDir relPath = do
  let fullPath = indexDir </> relPath
  mc <- readMetadataOrComputeHash fullPath
  case mc of
    Just (MetaContent { metaHash = h, metaSize = sz }) ->
      -- Check if this was parsed as metadata (binary) or computed from content (text)
      -- by trying parseMetadata on the raw content
      parseMetadataFile fullPath >>= \case
        Just _ -> return (BinaryEntry relPath h sz)   -- has hash:/size: format → binary
        Nothing -> return (TextEntry relPath)          -- content IS the file → text
    Nothing -> return (TextEntry relPath)  -- shouldn't happen, but safe fallback

-- | Read a single metadata entry from a git commit tree.
readEntryFromCommit :: String -> FilePath -> IO MetadataEntry
readEntryFromCommit commitHash relPath = do
  -- NOTE: path is at root level in the commit tree, NOT under index/
  (code, content, _) <- readProcessWithExitCode "git"
    [ "-C", bitIndexPath, "show", commitHash ++ ":" ++ relPath ] ""
  if code /= ExitSuccess
    then return (TextEntry relPath)
    else case parseMetadata content of
      Just (MetaContent { metaHash = h, metaSize = sz }) ->
        return (BinaryEntry relPath h sz)
      Nothing ->
        return (TextEntry relPath)  -- text file: content IS the data, skip hash verify

-- | Load only binary (hash-verifiable) metadata entries from the index.
-- Text files are excluded. If you need all entries, use 'loadMetadata' directly.
loadBinaryMetadata :: FilePath -> Concurrency -> IO [(Path, Hash 'MD5, Integer)]
loadBinaryMetadata indexDir concurrency =
  binaryEntries <$> loadMetadata (FromFilesystem indexDir) concurrency

-- | Verify working tree at an arbitrary root path against metadata at that path.
-- This is the core verification function used for both local and filesystem remote verification.
-- Returns (number of files checked, list of issues).
-- If an IORef counter is provided, it will be incremented after each file is checked.
verifyLocalAt :: FilePath -> Maybe (IORef Int) -> Concurrency -> IO (Int, [VerifyIssue])
verifyLocalAt root mCounter concurrency = do
  let indexDir = root </> ".bit/index"
  entries <- loadMetadata (FromFilesystem indexDir) concurrency
  -- Filter out .git directory entries
  let filteredEntries = filter (\entry -> not (isGitPath (entryPath entry))) entries
  
  -- Determine concurrency level
  bound <- resolveConcurrency concurrency
  
  -- Parallelize verification (IO-bound: file reads and hashing)
  issues <- runConcurrently (Parallel bound) (checkOne root indexDir) filteredEntries
  return (length filteredEntries, concat issues)
  where
    checkOne rootPath indexPath entry = do
      let relPath = entryPath entry
          actualPath = rootPath </> relPath
      exists <- doesFileExist actualPath
      result <- if not exists
        then return [Missing relPath]
        else case entry of
          BinaryEntry _ expectedHash expectedSize -> do
            actualHash <- hashFile actualPath
            actualSize <- fromIntegral . BS.length <$> BS.readFile actualPath
            if actualHash == expectedHash && actualSize == expectedSize
              then return []
              else return [HashMismatch relPath (T.unpack (hashToText expectedHash)) (T.unpack (hashToText actualHash)) expectedSize actualSize]
          TextEntry _ -> do
            -- Text files: just verify they exist (already checked above)
            -- Could optionally verify content matches index, but that's redundant
            -- since the index content IS the file content for text files
            actualHash <- hashFile actualPath
            actualSize <- fromIntegral . BS.length <$> BS.readFile actualPath
            -- For text files, we need to check against what's in the index
            let indexFilePath = indexPath </> relPath
            indexHash <- hashFile indexFilePath
            indexSize <- fromIntegral . BS.length <$> BS.readFile indexFilePath
            if actualHash == indexHash && actualSize == indexSize
              then return []
              else return [HashMismatch relPath (T.unpack (hashToText indexHash)) (T.unpack (hashToText actualHash)) indexSize actualSize]
      -- Increment counter after checking file (atomicModifyIORef' is thread-safe)
      maybe (return ()) (\ref -> atomicModifyIORef' ref (\n -> (n + 1, ()))) mCounter
      return result

-- | Verify local working tree against metadata in .rgit/index.
-- Returns (number of files checked, list of issues).
-- If an IORef counter is provided, it will be incremented after each file is checked.
verifyLocal :: FilePath -> Maybe (IORef Int) -> Concurrency -> IO (Int, [VerifyIssue])
verifyLocal cwd = verifyLocalAt cwd

-- | Extract metadata from a bundle's HEAD commit.
-- First fetches the bundle into the repo, then reads metadata from refs/remotes/origin/main.
-- Returns (binaryMetadata, allKnownPaths):
--   binaryMetadata: list of (path, hash, size) for binary files (verifiable)
--   allKnownPaths: set of all file paths in the bundle (binary + text, for "extra files" check)
-- Text files are NOT hash-verified because git may normalize line endings (CRLF→LF),
-- causing hash mismatches with the actual files on the remote.
loadMetadataFromBundle :: BundleName -> IO ([(Path, Hash 'MD5, Integer)], Set.Set Path)
loadMetadataFromBundle bundleName = do
  -- First, fetch the bundle into the repo so we can read from it
  fetchCode <- Git.fetchFromBundle bundleName
  if fetchCode /= ExitSuccess
    then return ([], Set.empty)
    else do
      -- Get the remote HEAD hash (now available as refs/remotes/origin/main)
      (_code, out, _) <- readProcessWithExitCode "git"
        [ "-C", bitIndexPath
        , "rev-parse"
        , "refs/remotes/origin/main"
        ] ""
      case filter (not . isSpace) out of
        [] -> return ([], Set.empty)
        headHash -> do
          entries <- loadMetadata (FromCommit headHash) Sequential
          return (binaryEntries entries, allEntryPaths entries)

-- | Verify remote files match remote metadata.
-- Returns (number of files checked, list of issues).
-- If an IORef counter is provided, it will be incremented after each file is checked.
verifyRemote :: FilePath -> Bit.Remote.Remote -> Maybe (IORef Int) -> Concurrency -> IO (Int, [VerifyIssue])
verifyRemote _cwd remote mCounter _concurrency = do
  -- 1. Fetch the remote bundle if needed
  let fetchedPath = fromCwdPath (bundleCwdPath fetchedBundle)
  bundleExists <- doesFileExist fetchedPath
  when (not bundleExists) $ do
    let localDest = ".bit/temp_remote.bundle"
    fetchResult <- Transport.copyFromRemoteDetailed remote ".bit/bit.bundle" localDest
    case fetchResult of
      Transport.CopySuccess -> do
        -- Copy to fetchedPath for consistency
        BS.readFile localDest >>= BS.writeFile fetchedPath
        when (localDest /= fetchedPath) $ safeRemove localDest
      _ -> do
        hPutStrLn stderr "Error: Could not fetch remote bundle."
        return ()
  
  -- Check if bundle exists now (if fetch failed, we can't continue)
  bundleExistsNow <- doesFileExist fetchedPath
  if not bundleExistsNow
    then return (0, [])
    else do
      -- 2. Load metadata from the bundle (binary metadata + all known paths)
      (remoteMeta, allKnownPaths) <- loadMetadataFromBundle fetchedBundle
      
      -- 3. Fetch actual remote files
      Remote.Scan.fetchRemoteFiles remote >>= either
        (\_ -> hPutStrLn stderr "Error: Could not fetch remote file list." >> return (0, []))
        (\remoteFiles -> do
          let filteredRemoteFiles = filterOutBitPaths remoteFiles
          
          -- 4. Build maps for comparison (both use MD5 hashes)
          let remoteFileMap = Map.fromList
                [ (normalise e.path, (h, e.kind))
                | e <- filteredRemoteFiles
                , h <- maybeToList (syncHash e.kind)
                ]
          
          -- 5. Compare binary file metadata with actual files on remote
          issues <- traverse (checkRemoteFile remoteFileMap) remoteMeta
          
          -- 6. Check for files on remote that aren't known to the bundle
          -- Use allKnownPaths (binary + text) to avoid false positives for text files
          let filePaths = Set.fromList (Map.keys remoteFileMap)
              extraPaths = filePaths `Set.difference` allKnownPaths
              extraIssues = map (\p -> HashMismatch p "(not in metadata)" "(exists on remote)" 0 0) (Set.toList extraPaths)
          
          return (length remoteMeta, concat issues ++ extraIssues))
  where
    -- Check one file from metadata against remote (both use MD5)
    checkRemoteFile :: Map.Map FilePath (Hash 'MD5, EntryKind) -> (Path, Hash 'MD5, Integer) -> IO [VerifyIssue]
    checkRemoteFile remoteFileMap (relPath, expectedHash, expectedSize) = do
      let normalizedPath = normalise relPath
      result <- case Map.lookup normalizedPath remoteFileMap of
        Nothing -> return [Missing relPath]
        Just (actualHash, File _ actualSize _) ->
          if actualHash == expectedHash && actualSize == expectedSize
            then return []
            else return [HashMismatch relPath (T.unpack (hashToText expectedHash)) (T.unpack (hashToText actualHash)) expectedSize actualSize]
        Just _ -> return []
      -- Increment counter after checking file
      maybe (return ()) (\ref -> atomicModifyIORef' ref (\n -> (n + 1, ()))) mCounter
      return result

-- Helper to safely remove a file
safeRemove :: FilePath -> IO ()
safeRemove filePath = do
  exists <- doesFileExist filePath
  when exists $ removeFile filePath
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
import Data.Maybe (fromMaybe)
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
    then return defaultTextConfig
    else do
      bs <- BS.readFile configPath
      let content = case T.decodeUtf8' bs of
            Left _ -> T.empty  -- Invalid UTF-8, use defaults
            Right txt -> txt
      return $ parseConfig content

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
findIndex :: (a -> Bool) -> [a] -> Maybe Int
findIndex p xs = case [i | (i, x) <- zip [0..] xs, p x] of
  [] -> Nothing
  (i:_) -> Just i

-- | Extract lines for a given section (between [section] and next [section] or EOF)
extractSection :: String -> [T.Text] -> [T.Text]
extractSection sectionName linesOfText =
  let sectionHeader = "[" ++ sectionName ++ "]"
      -- Find start of section
      startIdx = case findIndex (\l -> T.strip l == T.pack sectionHeader) linesOfText of
        Nothing -> length linesOfText  -- Section not found
        Just idx -> idx + 1
      -- Find end of section (next [section] or EOF)
      endIdx = case findIndex (\l -> T.stripStart l `T.isPrefixOf` T.pack "[") (drop startIdx linesOfText) of
        Nothing -> length linesOfText
        Just idx -> startIdx + idx
  in map T.strip $ take (endIdx - startIdx) (drop startIdx linesOfText)

-- | Parse size-limit from section lines
parseSizeLimit :: [T.Text] -> Maybe Integer
parseSizeLimit linesOfText =
  let findLine prefix = [T.unpack (T.drop (T.length (T.pack prefix)) (T.strip l)) | l <- linesOfText, T.stripStart l `T.isPrefixOf` T.pack prefix]
      sizeLines = findLine "size-limit"
  in case sizeLines of
    [] -> Nothing
    (sizeStr:_) -> 
      -- Remove comments and parse
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
  in case extLines of
    [] -> Nothing
    (extStr:_) ->
      -- Remove comments and parse comma-separated list
      let cleaned = takeWhile (/= '#') extStr
          trimmed = dropWhile isSpace $ dropWhileEnd isSpace cleaned
          -- Split by comma and clean each extension
          exts = map (dropWhile isSpace . dropWhileEnd isSpace) $ splitComma trimmed
      in Just exts
  where
    splitComma s = case break (== ',') s of
      (part, "") -> [part]
      (part, _:rest) -> part : splitComma rest

dropWhileEnd :: (a -> Bool) -> [a] -> [a]
dropWhileEnd p = reverse . dropWhile p . reverse
```

---

## Internal/Git.hs

**Path:** `Internal/Git.hs`

*Source file.*

```haskell
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
    , checkIsAhead
    , getHashFromBundle
    , restore
    , checkout
    , status
    , setupRemote
    , addRemote
    , getRemoteUrl
    , getTrackedRemoteName
    , fetchFromBundle
    , updateRemoteTrackingBranch
    , updateRemoteTrackingBranchToHead
    , updateRemoteTrackingBranchToHash
    , setupBranchTracking
    , setupBranchTrackingFor
    , unsetBranchUpstream
    , mergeOriginMain
    , mergeNoCommit
    , mergeNoCommitAllowUnrelated
    , mergeAbort
    , isMergeInProgress
    , checkoutRemoteAsMain
    , getConflictedFiles
    , getConflictType
    , checkoutOurs
    , checkoutTheirs
    , runGitRaw
    , runGitWithOutput
    , runGitAt
    , ConflictType(..)
    , readFileFromRef
    , listFilesInRef
    , fsck
    , hasStagedChanges
    , getDiffNameStatus
    , getFilesAtCommit
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

-- | Represents the subset of Git functionality rgit uses
data GitCommand
    = Init { _separateGitDir :: FilePath }
    | Config { _configName :: String, _configValue :: String }
    | RevParse { _revParseRef :: String }
    | CommitFile { _commitMessage :: String, _commitFile :: FilePath }
    | RevList { _revListLeft :: String, _revListRight :: String }
    | CreateBundle { _createBundlePath :: FilePath }
    | GetBundleHead { _getBundleHeadPath :: FilePath }
    | IsAncestor { _ancestorHash :: String, _descendantHash :: String }
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

        IsAncestor a d ->
            ["merge-base", "--is-ancestor", a, d]

        GetHead ->
            ["rev-parse", "HEAD"]

getLocalHead :: IO (Maybe String)
getLocalHead = do
    (code, out, _) <- runGit GetHead
    return $ (guard (code == ExitSuccess) >> Just (filter (not . isSpace) out))

getHashFromBundle :: BundleName -> IO (Maybe String)
getHashFromBundle bundleName = do
    let (GitRelPath relPath) = bundleGitRelPath bundleName
    (code, out, _) <- runGit (GetBundleHead relPath)
    return $ guard (code == ExitSuccess && not (null out)) >> listToMaybe (words out)

runGitCommand :: GitCommand -> IO ExitCode
runGitCommand cmd = do
    (c, o, e) <- runGit cmd
    -- Don't print error messages for IsAncestor since non-zero exit codes are expected
    -- (they indicate "no, not an ancestor" which is a valid answer, not an error)
    when (c /= ExitSuccess && not (isAncestorCommand cmd)) $ 
        hPutStrLn stderr ("bit: git command failed: " ++ e)
    putStr o
    hPutStr stderr e
    return c
  where
    isAncestorCommand (IsAncestor _ _) = True
    isAncestorCommand _ = False

commitFile :: String -> FilePath -> IO ExitCode
commitFile msg filePath = runGitCommand (CommitFile msg filePath)

init :: FilePath -> IO ExitCode
init dir = runGitCommand (Init dir)

createBundle :: BundleName -> IO ExitCode
createBundle bundleName = do
    let (GitRelPath relPath) = bundleGitRelPath bundleName
    runGitCommand (CreateBundle relPath)

config :: String -> String -> IO ExitCode
config configName configValue = runGitCommand (Config configName configValue)

checkIsAhead :: String -> String -> IO Bool
checkIsAhead rHash lHash = do
    code <- runGitCommand (IsAncestor rHash lHash)
    return (code == ExitSuccess)

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
            readProcessWithExitCode "git" (baseFlags ++ ["remote", "set-url", remoteName, url]) "" >>= \(c, _, _) -> return c
        ExitFailure _ -> do
            readProcessWithExitCode "git" (baseFlags ++ ["remote", "add", remoteName, url]) "" >>= \(c, _, _) -> return c

-- | Get the URL for a remote by name (git remote get-url <name>). Returns Nothing if remote missing.
getRemoteUrl :: String -> IO (Maybe String)
getRemoteUrl remoteName = do
    (code, out, _) <- readProcessWithExitCode "git" (baseFlags ++ ["remote", "get-url", remoteName]) ""
    if code /= ExitSuccess then return Nothing
    else return (Just (filter (/= '\n') out))

-- | Get the remote name that the current branch tracks (branch.main.remote).
-- Falls back to "origin" if not configured — this means commands work with
-- a reasonable default even without explicit -u, but callers should be aware
-- that "origin" fallback doesn't mean tracking IS configured.
getTrackedRemoteName :: IO String
getTrackedRemoteName = do
    (code, out, _) <- readProcessWithExitCode "git" (baseFlags ++ ["config", "--get", "branch.main.remote"]) ""
    if code /= ExitSuccess then return "origin"
    else return (filter (/= '\n') out)

-- | Set up a git remote named "origin" pointing to the given URL (legacy / internal use)
setupRemote :: String -> IO ExitCode
setupRemote url = addRemote "origin" url

-- | Pull from a bundle file into the local repo: fetch the bundle's refs so all
-- objects and refs/remotes/origin/main exist in .rgit/index/.git. This is the "real"
-- pull from the fetched bundle; without it, the ref would point to a hash not in the repo.
fetchFromBundle :: BundleName -> IO ExitCode
fetchFromBundle bundleName = do
    let (GitRelPath bundle) = bundleGitRelPath bundleName
    (code, out, err) <- readProcessWithExitCode "git"
        (baseFlags ++ ["fetch", bundle, "+refs/heads/main:refs/remotes/origin/main"]) ""
    putStr out
    hPutStr stderr err
    return code

-- | Update the remote tracking branch refs/remotes/origin/main to point to the hash from the bundle.
-- Use when the objects are already in the repo (e.g. after push); for fetch/pull use fetchFromBundle.
updateRemoteTrackingBranch :: BundleName -> IO ExitCode
updateRemoteTrackingBranch bundleName = do
    maybeHash <- getHashFromBundle bundleName
    case maybeHash of
        Just hash -> do
            -- Update the remote tracking branch ref
            -- Use update-ref to create or update refs/remotes/origin/main
            updateRemoteTrackingBranchToHash hash
        Nothing -> return (ExitFailure 1)

-- | Set refs/remotes/origin/main to a specific hash. Use after a successful pull so status shows
-- "up to date with 'origin/main'" instead of "ahead by N commits".
updateRemoteTrackingBranchToHash :: String -> IO ExitCode
updateRemoteTrackingBranchToHash hash =
    readProcessWithExitCode "git" (baseFlags ++ ["update-ref", "refs/remotes/origin/main", hash]) "" >>= \(c, _, _) -> return c

-- | Set refs/remotes/origin/main to current HEAD.
-- WARNING: Only correct after PUSH (where remote now matches local HEAD).
-- After PULL/MERGE, use updateRemoteTrackingBranchToHash with the bundle hash instead,
-- because HEAD includes local merge commits the remote doesn't have.
-- See: "Tracking Ref Invariant" in docs/spec.md.
updateRemoteTrackingBranchToHead :: IO ExitCode
updateRemoteTrackingBranchToHead = do
    (code, out, _) <- readProcessWithExitCode "git" (baseFlags ++ ["rev-parse", "HEAD"]) ""
    case filter (/= '\n') out of
        hash | code == ExitSuccess && not (null hash) ->
            updateRemoteTrackingBranchToHash hash
        _ -> return (ExitFailure 1)

-- | Set up the local branch to track a specific remote
-- Configures branch.main.remote and branch.main.merge
setupBranchTrackingFor :: String -> IO ExitCode
setupBranchTrackingFor remoteName = do
    (code1, _, _) <- readProcessWithExitCode "git"
        (baseFlags ++ ["config", "branch.main.remote", remoteName]) ""
    (code2, _, _) <- readProcessWithExitCode "git"
        (baseFlags ++ ["config", "branch.main.merge", "refs/heads/main"]) ""
    case (code1, code2) of
        (ExitSuccess, ExitSuccess) -> return ExitSuccess
        _ -> return (ExitFailure 1)

-- | Set up the local branch to track origin/main
-- This configures branch.main.remote and branch.main.merge so git status knows what to compare
setupBranchTracking :: IO ExitCode
setupBranchTracking = setupBranchTrackingFor "origin"

-- | Unset the upstream for the current branch (clears "upstream is gone" when remote refs are missing)
unsetBranchUpstream :: IO ExitCode
unsetBranchUpstream = do
    (code, _, _) <- readProcessWithExitCode "git" (baseFlags ++ ["branch", "--unset-upstream"]) ""
    return code

-- | Merge refs/remotes/origin/main into the current branch (HEAD).
-- Used by rgit pull after fetching the remote bundle.
mergeOriginMain :: IO ExitCode
mergeOriginMain = runGitRaw ["merge", "refs/remotes/origin/main", "--no-edit"]

-- | Run git with baseFlags; returns (exitCode, stdout, stderr). Does not rewrite hints.
runGitWithOutput :: [String] -> IO (ExitCode, String, String)
runGitWithOutput args = do
  let fullArgs = baseFlags ++ ["-c", "color.ui=never"] ++ args
  readProcessWithExitCode "git" fullArgs ""

-- | Merge without committing (for pull flow). Returns (exitCode, stdout, stderr).
mergeNoCommit :: IO (ExitCode, String, String)
mergeNoCommit = runGitWithOutput ["merge", "--no-commit", "--no-ff", "refs/remotes/origin/main"]

-- | Like mergeNoCommit but allows merging unrelated histories (e.g. first pull into a fresh init).
mergeNoCommitAllowUnrelated :: IO (ExitCode, String, String)
mergeNoCommitAllowUnrelated = runGitWithOutput ["merge", "--no-commit", "--no-ff", "--allow-unrelated-histories", "refs/remotes/origin/main"]

-- | Abort an in-progress merge.
mergeAbort :: IO ExitCode
mergeAbort = do
  (code, out, err) <- runGitWithOutput ["merge", "--abort"]
  putStr (rewriteGitHints out)
  hPutStr stderr (rewriteGitHints err)
  return code

-- | True if a merge is in progress (MERGE_HEAD exists).
isMergeInProgress :: IO Bool
isMergeInProgress = do
  (code, _, _) <- runGitWithOutput ["rev-parse", "--verify", "MERGE_HEAD"]
  return (code == ExitSuccess)

-- | Checkout refs/remotes/origin/main as the local main branch.
-- Used on first pull when there are no local commits (unborn branch).
-- This avoids the need for merge and gives us the remote's history directly.
--
-- TRACKING CONTRACT: This function NEVER modifies branch tracking config.
-- Uses --no-track to prevent git from auto-setting branch.main.remote.
-- Callers who need tracking must call setupBranchTrackingFor explicitly.
--
-- Uses -f (force) to overwrite any local files created during init.
checkoutRemoteAsMain :: IO ExitCode
checkoutRemoteAsMain = do
  -- Use checkout -B to create/reset branch and checkout in one step
  -- Use -f to force overwrite of any local files (like .gitattributes from init)
  -- Use --no-track to prevent auto-setting branch.main.remote (git-standard: require explicit -u)
  (code, _, _) <- runGitWithOutput ["checkout", "-f", "-B", "main", "--no-track", "refs/remotes/origin/main"]
  return code

-- | Paths relative to work tree (index/...) that are unmerged.
getConflictedFiles :: IO [FilePath]
getConflictedFiles = do
  (code, out, _) <- runGitWithOutput ["diff", "--name-only", "--diff-filter=U"]
  if code /= ExitSuccess then return [] else return (filter (not . null) (lines out))

-- | Conflict type for Git-like messages. Path is work-tree relative (e.g. index/src/model.bin).
data ConflictType
  = ContentConflict FilePath   -- both modified
  | ModifyDelete FilePath Bool -- True = deleted in HEAD (ours)
  | AddAdd FilePath            -- both added different
  deriving (Show, Eq)

-- | Determine conflict type using git ls-files -u. Path is as in index (e.g. index/foo).
-- Format: "mode SP oid SP stage TAB name"
getConflictType :: FilePath -> IO ConflictType
getConflictType path = do
  (_, out, _) <- runGitWithOutput ["ls-files", "-u", "--", path]
  let beforeTab line = takeWhile (/= '\t') line
  let stageNum line = case reverse (words (beforeTab line)) of
        (s:_) | s `elem` ["1","2","3"] -> Just (read s :: Int)
        _ -> Nothing
  let stageNums = mapMaybe stageNum (lines out)
  let has1 = 1 `elem` stageNums
  let has2 = 2 `elem` stageNums
  let has3 = 3 `elem` stageNums
  if has2 && has3 && has1 then return (ContentConflict path)
  else if has2 && has3 && not has1 then return (AddAdd path)
  else if has2 && not has3 then return (ModifyDelete path False)  -- deleted in theirs
  else if has3 && not has2 then return (ModifyDelete path True)   -- deleted in ours (HEAD)
  else return (ContentConflict path)

-- | Check out our version for path (work-tree path under .rgit/index).
checkoutOurs :: FilePath -> IO ExitCode
checkoutOurs path = do
  (code, out, err) <- runGitWithOutput ["checkout", "--ours", "--", path]
  putStr (rewriteGitHints out)
  hPutStr stderr (rewriteGitHints err)
  return code

-- | Check out their version for path.
checkoutTheirs :: FilePath -> IO ExitCode
checkoutTheirs path = do
  (code, out, err) <- runGitWithOutput ["checkout", "--theirs", "--", path]
  putStr (rewriteGitHints out)
  hPutStr stderr (rewriteGitHints err)
  return code

-- | Read file content from a Git ref (e.g., "refs/remotes/origin/main:path/to/file").
-- Returns Nothing if file doesn't exist in that ref.
readFileFromRef :: String -> FilePath -> IO (Maybe String)
readFileFromRef gitRef path = do
  (code, out, _err) <- runGitWithOutput ["show", gitRef ++ ":" ++ path]
  if code == ExitSuccess && not (null out)
    then return (Just out)
    else return Nothing

-- | List all files in a Git ref's tree (recursive). Returns paths relative to work tree root.
listFilesInRef :: String -> IO [FilePath]
listFilesInRef gitRef = do
  (code, out, _) <- runGitWithOutput ["ls-tree", "-r", "--name-only", gitRef]
  if code == ExitSuccess
    then return (filter (not . null) (lines out))
    else return []

-- | Run git fsck to check metadata history integrity.
-- Returns (exitCode, output, errorOutput).
fsck :: IO (ExitCode, String, String)
fsck = runGitWithOutput ["fsck"]

-- | Check if there are staged changes ready to commit.
-- Returns True if there are staged changes, False otherwise.
hasStagedChanges :: IO Bool
hasStagedChanges = do
  (code, _, _) <- runGitWithOutput ["diff", "--cached", "--quiet"]
  return (code == ExitFailure 1)  -- git diff --cached --quiet exits with 1 if there are changes

-- | Get the list of file changes between two commits.
-- Returns list of (status, path, maybe-new-path-for-renames).
-- Status: 'A' = added, 'D' = deleted, 'M' = modified, 'R' = renamed.
getDiffNameStatus :: String -> String -> IO [(Char, FilePath, Maybe FilePath)]
getDiffNameStatus oldHead newHead = do
    (code, out, _) <- runGitWithOutput ["diff", "--name-status", oldHead, newHead]
    if code /= ExitSuccess then return []
    else return (parseNameStatus out)

parseNameStatus :: String -> [(Char, FilePath, Maybe FilePath)]
parseNameStatus = mapMaybe parseLine . lines
  where
    parseLine line = case line of
        (fileStatus:rest)
            | fileStatus == 'R' || fileStatus == 'C' ->
                -- R100\told\tnew or Rnnn old new (tab-separated)
                case words (dropWhile (\c -> c /= '\t' && c /= ' ') rest) of
                    (old:new:_) -> Just (fileStatus, old, Just new)
                    _ -> Nothing
            | fileStatus `elem` "ADM" ->
                case words rest of
                    (path:_) -> Just (fileStatus, path, Nothing)
                    _ -> Nothing
            | otherwise -> Nothing
        _ -> Nothing

-- | Get all file paths at a given commit. Used when there's no old HEAD to diff against.
getFilesAtCommit :: String -> IO [FilePath]
getFilesAtCommit gitRef = do
    (code, out, _) <- runGitWithOutput ["ls-tree", "-r", "--name-only", gitRef]
    if code /= ExitSuccess then return []
    else return (filter (not . null) (lines out))

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
import Control.Concurrent.Async (async, wait)
import Data.List (isInfixOf)
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
                return (code, LBS.fromStrict outBytes, errStr)
            _ -> error "readProcessBytes: failed to create pipes"
  where
    -- Cleanup: close any handles that might still be open and wait for process
    cleanupProcess (mStdin, mStdout, mStderr, ph) = do
        -- Try to close handles (may already be closed by hGetContents)
        maybe (return ()) (const $ return ()) mStdin
        maybe (return ()) (\h -> void (try (hClose h) :: IO (Either SomeException ()))) mStdout
        maybe (return ()) (\h -> void (try (hClose h) :: IO (Either SomeException ()))) mStderr
        -- Ensure process is cleaned up
        void (try (waitForProcess ph) :: IO (Either SomeException ExitCode))
    
    -- Strict reading of handle contents
    hGetContents' :: Handle -> IO String
    hGetContents' h = go []
      where
        go acc = do
            eof <- hIsEOF h
            if eof
                then return (concat (reverse acc))
                else do
                    line <- hGetLine h
                    go ((line ++ "\n") : acc)

-- | Build a full remote path from Remote + relative path.
-- Handles trailing-slash normalization internally.
remoteFilePath :: Remote -> FilePath -> String
remoteFilePath remote relPath =
    let base = remoteUrl remote
        -- Ensure exactly one separator between base and relative path
        base' = if not (null base) && last base == '/' then init base else base
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
copyToRemote localPath remote relPath = do
    let fullRemote = remoteFilePath remote relPath
    (code, _, _) <- readProcessWithExitCode "rclone" ["copyto", localPath, fullRemote] ""
    return code

-- | Copy file from remote (relative path) to local
copyFromRemote :: Remote -> FilePath -> FilePath -> IO ExitCode
copyFromRemote remote relPath localPath = do
    let fullRemote = remoteFilePath remote relPath
    (code, _, _) <- readProcessWithExitCode "rclone" ["copyto", fullRemote, localPath] ""
    return code

-- | Copy from remote with detailed error classification
copyFromRemoteDetailed :: Remote -> FilePath -> FilePath -> IO CopyResult
copyFromRemoteDetailed remote relPath localPath = do
    let fullRemote = remoteFilePath remote relPath
    (code, _, err) <- readProcessWithExitCode "rclone" ["copyto", fullRemote, localPath] ""
    case code of
        ExitSuccess -> return CopySuccess
        ExitFailure _ 
            | "directory not found" `isInfixOf` err || "object not found" `isInfixOf` err -> return CopyNotFound
            | "no such host" `isInfixOf` err || "dial tcp" `isInfixOf` err -> return (CopyNetworkError err)
            | otherwise -> return (CopyOtherError err)

-- | Move a file on remote (both paths relative to remote root)
moveRemote :: Remote -> FilePath -> FilePath -> IO ExitCode
moveRemote remote srcRel destRel = do
    let src = remoteFilePath remote srcRel
        dest = remoteFilePath remote destRel
    (code, _, _) <- readProcessWithExitCode "rclone" ["moveto", src, dest] ""
    return code

-- | Delete a file on remote (relative path)
deleteRemote :: Remote -> FilePath -> IO ExitCode
deleteRemote remote relPath = do
    let fullRemote = remoteFilePath remote relPath
    (code, _, _) <- readProcessWithExitCode "rclone" ["deletefile", fullRemote] ""
    return code

-- | Purge entire remote (no relative path — purges the remote root)
purgeRemote :: Remote -> IO ExitCode
purgeRemote remote = do
    let fullRemote = remoteUrl remote
    (code, _, _) <- readProcessWithExitCode "rclone" ["purge", fullRemote] ""
    return code

-- | Create directory on remote (relative path)
mkdirRemote :: Remote -> FilePath -> IO ExitCode
mkdirRemote remote relPath = do
    let fullRemote = remoteFilePath remote relPath
    (code, _, _) <- readProcessWithExitCode "rclone" ["mkdir", fullRemote] ""
    return code

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
            then return (Right [])  -- Empty directory
            else return (Left err)  -- Network or other error
        ExitSuccess -> do
            case Aeson.decode outBytes :: Maybe [RcloneItem] of
                Nothing -> return (Left "Failed to parse rclone JSON output")
                Just items -> return (Right [TransportItem (name item) (isDir item) | item <- items])

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
            return CheckResult
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
                        return CheckResult
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
        maybe (return ()) (\h -> void (try (hClose h) :: IO (Either SomeException ()))) mOut
        maybe (return ()) (\h -> void (try (hClose h) :: IO (Either SomeException ()))) mErr
        -- Ensure process is cleaned up
        void (try (waitForProcess ph) :: IO (Either SomeException ExitCode))
    
    -- Read lines from handle, incrementing counter for each line
    readLinesWithProgress :: Handle -> IORef Int -> IO [String]
    readLinesWithProgress h counter = go []
      where
        go acc = do
            eof <- hIsEOF h
            if eof
                then return (reverse acc)
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
                then return (concat (reverse acc))
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
                     , head symChar == sym
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
    other-modules:       Bit.Types,
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
import Control.Monad (forM_)
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
    then return dir
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
      | name `elem` excludedDirs = return []
      | name == "test"           = return []  -- test/ handled separately
      | head name == '.'         = return []
      | otherwise = do
          let relPath  = if null rel then name else rel </> name
              fullPath = root </> relPath
          isDir <- doesDirectoryExist fullPath
          if isDir
            then go relPath
            else return [ relPath | takeExtension name == ".hs" ]

-- | Collect all files under test/, relative to @root@.
gatherTestFiles :: FilePath -> IO [FilePath]
gatherTestFiles root = go "test"
  where
    go rel = do
      let full = root </> rel
      exists <- doesDirectoryExist full
      if not exists then return []
      else do
        entries <- listDirectory full
        fmap concat $ mapM (visit rel) entries

    visit rel name
      | name `elem` excludedTestDirs  = return []
      | name `elem` excludedTestFiles = return []
      | head name == '.'              = return []
      | otherwise = do
          let relPath  = rel </> name
              fullPath = root </> relPath
          isDir <- doesDirectoryExist fullPath
          if isDir
            then go relPath
            else return [relPath]

-- | Collect all .md files under docs/, relative to @root@.
gatherDocsFiles :: FilePath -> IO [FilePath]
gatherDocsFiles root = do
  let docsDir = root </> "docs"
  exists <- doesDirectoryExist docsDir
  if not exists then return []
  else do
    entries <- listDirectory docsDir
    let mdFiles = [ "docs" </> name | name <- entries
                  , takeExtension name == ".md"
                  , head name /= '.' ]
    return (sort mdFiles)

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
  return (code == ExitSuccess)

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
              if not (T.null content) && T.last content /= '\n'
                then hPutStrLn h ""
                else return ()
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

