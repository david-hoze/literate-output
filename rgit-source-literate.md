# rgit — Literate Programming Document

This document contains all Haskell source files and the cabal
file for the rgit project, presented in literate-programming style.

---

## Bit.hs

**Path:** `Bit.hs`

*Source file.*

```haskell
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE ScopedTypeVariables #-}
{-# LANGUAGE TypeApplications #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE DuplicateRecordFields #-}
{-# LANGUAGE OverloadedRecordDot #-}
{-# LANGUAGE LambdaCase #-}

module Bit
    ( -- Repo initialization
      init

      -- Git passthrough (these take args from the CLI)
    , add
    , commit
    , diff
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
import Control.Monad (when, unless, void, forM_)
import qualified Data.ByteString.Lazy as LBS
import qualified Data.ByteString.Lazy.Char8 as LBSC
import System.Exit (ExitCode(..), exitWith)
import qualified Data.Map as Map
import qualified Internal.Git as Git
import qualified Internal.Transport as Transport
import Internal.Config (rgitDir, rgitIgnore, rgitGitDir, fetchedBundle, rgitIndexPath, rgitDevicesDir, rgitRemotesDir, bundleCwdPath, bundleGitRelPath, fromCwdPath, fromGitRelPath, BundleName(..))
import qualified Rgit.Internal.Metadata as Metadata
import Rgit.Internal.Metadata (MetaContent(..), parseMetadata, displayHash, serializeMetadata)
import Data.Char (isSpace)
import qualified Rgit.Scan as Scan
import qualified Rgit.Diff as Diff
import qualified Rgit.Plan as Plan
import qualified Rgit.Pipeline as Pipeline
import qualified Rgit.Verify as Verify
import qualified Rgit.Fsck as Fsck
import Rgit.Plan (RcloneAction(..))
import Rgit.Types
import qualified Rgit.Remote.Scan as Remote.Scan
import Control.Monad.Trans.Reader (asks)
import Control.Monad.IO.Class (liftIO)
import Control.Monad.Trans.Class (lift)
import Control.Exception (try, throwIO, SomeException, IOException)
import System.IO (hFlush, stdout, stderr, hPutStrLn, hIsTerminalDevice)
import Data.Maybe (fromMaybe, listToMaybe, maybe, maybeToList)
import Data.Either (either)
import Rgit.Utils (toPosix, filterOutRgitPaths)
import qualified Rgit.Device as Device
import qualified Rgit.DevicePrompt as DevicePrompt
import qualified Rgit.Conflict as Conflict
import Rgit.Remote (Remote, remoteName, displayRemote, resolveRemote, remoteUrl, RemoteState(..), FetchResult(..))
import Prelude hiding (init)
import Control.Exception (bracket)

-- ============================================================================
-- Types
-- ============================================================================

data PullOptions = PullOptions
    { pullAcceptRemote :: Bool
    , pullManualMerge :: Bool
    } deriving (Show)

defaultPullOptions :: PullOptions
defaultPullOptions = PullOptions False False

-- ============================================================================
-- Git passthrough (thin wrappers)
-- ============================================================================

add :: [String] -> IO ExitCode
add args = Git.runGitRaw ("add" : args)

commit :: [String] -> IO ExitCode
commit args = Git.runGitRaw ("commit" : args)

diff :: [String] -> IO ExitCode
diff args = Git.runGitRaw ("diff" : args)

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

fsck :: FilePath -> IO ()
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

    putStrLn $ "Initializing rgit in: " ++ cwd

    -- 1. Create .rgit directory
    Dir.createDirectoryIfMissing True rgitDir

    -- 2. Create .rgit/index directory (needed before git init)
    Dir.createDirectoryIfMissing True (rgitDir </> "index")

    -- 3. Init Git in the index directory
    hasGit <- Dir.doesDirectoryExist rgitGitDir
    if hasGit
        then putStrLn "Git repository already exists in .rgit/index/.git"
        else do
            putStrLn "Running: git init in .rgit/index"
            -- Initialize git in .rgit/index, which will create .rgit/index/.git
            void $ Git.init rgitGitDir

    -- 3a. Create .git/bundles directory for storing bundle files
    Dir.createDirectoryIfMissing True (rgitGitDir </> "bundles")

    -- 4. Configure Git to look for the ignore file inside .rgit
    -- This is the magic step!
    Git.config "core.excludesFile" rgitIgnore
    
    -- 5. Configure default branch name to "main" (for the repo we just created)
    -- This affects future branch operations in this repo
    Git.config "init.defaultBranch" "main"
    
    -- 6. Rename the initial branch to "main" if it's "master"
    -- Git init creates "master" by default, so we rename it
    (code, _, _) <- Git.runGitWithOutput ["branch", "-m", "master", "main"]
    when (code /= ExitSuccess) $
        -- If rename failed (e.g., no commits yet), that's okay - first commit will use "main"
        return ()

    -- 4. Create the ignore file inside .rgit
    -- Now we only need one rule: ignore everything. 
    -- Because the git-dir and ignore file are inside .rgit, 
    -- they are "invisible" to the work-tree anyway.
    LBS.writeFile rgitIgnore (LBSC.pack $ unlines [
        "*",                -- Ignore everything in the root
        "!.rgit/",          -- Allow the .rgit folder
        "!.rgit/index/",
        "!.rgit/ignore",
        "!.rgit/target",
        "!.rgit/devices/",
        "!.rgit/remotes/",
        ".rgit/index/.git/"   -- EXPLICITLY ignore the git metadata folder
        ])

    -- 4. Create other .rgit subdirectories (index already created above)
    Dir.createDirectoryIfMissing True rgitDevicesDir
    Dir.createDirectoryIfMissing True rgitRemotesDir

    -- 5a. Create config file with default values
    let configPath = rgitDir </> "config"
    configExists <- Dir.doesFileExist configPath
    unless configExists $ do
        let defaultConfig = unlines
                [ "[text]"
                , "    size-limit = 1048576  # 1MB, files larger are always binary"
                , "    extensions = .txt,.md,.yaml,.yml,.json,.xml,.html,.css,.js,.py,.hs,.rs"
                ]
        writeFile configPath defaultConfig

    -- 5b. Merge driver: prevent Git from writing conflict markers; rgit resolves whole-file only
    -- Use .rgit/index/.git/info/attributes instead of .gitattributes in the working tree
    -- This way it doesn't conflict with files from the remote on first pull
    -- Also disable text/CRLF conversion (-text) to prevent spurious "modified" status
    void $ Git.config "merge.rgit-metadata.name" "rgit metadata"
    void $ Git.config "merge.rgit-metadata.driver" "false"
    Dir.createDirectoryIfMissing True (rgitGitDir </> "info")
    writeFile (rgitGitDir </> "info" </> "attributes") "* merge=rgit-metadata -text\n"

    -- Note: We do NOT create an initial commit here.
    -- This keeps the repo empty until first real commit or pull.
    -- On first pull, we simply checkout the remote's history (no merge needed).

    putStrLn "rgit initialized successfully!"

addRemote :: String -> String -> IO ()
addRemote name pathOrUrl = do
    cwd <- Dir.getCurrentDirectory
    Dir.createDirectoryIfMissing True rgitDevicesDir
    Dir.createDirectoryIfMissing True rgitRemotesDir
    pathType <- Device.classifyRemotePath pathOrUrl
    case pathType of
        Device.CloudRemote url -> do
            Device.writeRemoteFile cwd name (Device.TargetCloud url)
            void $ Git.addRemote name url
            when (name == "origin") $ void Git.setupBranchTracking
            putStrLn $ "Remote '" ++ name ++ "' added (" ++ url ++ ")."
        Device.FilesystemPath path -> addRemoteFilesystem cwd name path

promptDeviceName :: FilePath -> FilePath -> Maybe String -> IO String
promptDeviceName cwd _volRoot mLabel =
    DevicePrompt.acquireDeviceNameAuto mLabel $ \name -> (name `elem`) <$> Device.listDeviceNames cwd

addRemoteFilesystem :: FilePath -> String -> FilePath -> IO ()
addRemoteFilesystem cwd name path = do
    absPath <- Dir.makeAbsolute path
    exists <- Dir.doesDirectoryExist absPath
    unless exists $ do
        hPutStrLn stderr ("fatal: Path does not exist or is not accessible: " ++ path)
        exitWith (ExitFailure 1)
    volRoot <- Device.getVolumeRoot absPath
    let relPath = Device.getRelativePath volRoot absPath
    mStoreUuid <- Device.readRgitStore volRoot
    mExistingDevice <- case mStoreUuid of
        Just u -> Device.findDeviceByUuid cwd u
        Nothing -> return Nothing
    result <- try @IOException $ case (mStoreUuid, mExistingDevice) of
        (Just _u, Just dev) -> do
            putStrLn $ "Using existing device '" ++ dev ++ "'."
            mInfo <- Device.readDeviceFile cwd dev
            let storeType = maybe Device.Physical Device.deviceType mInfo
            Device.writeRemoteFile cwd name (Device.TargetDevice dev relPath)
            when (name == "origin") $ void Git.setupBranchTracking
            putStrLn $ "Remote '" ++ name ++ "' Γזע " ++ dev ++ ":" ++ relPath
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
            when (name == "origin") $ void Git.setupBranchTracking
            putStrLn $ "Remote '" ++ name ++ "' Γזע " ++ deviceName' ++ ":" ++ relPath
            putStrLn $ "Device '" ++ deviceName' ++ "' registered (" ++ (case storeType' of Device.Physical -> "physical"; Device.Network -> "network") ++ ")."
            return ()
        (Nothing, _) -> do
            mLabel <- Device.getVolumeLabel volRoot
            deviceName' <- promptDeviceName cwd volRoot mLabel
            u <- Device.generateStoreUuid
            Device.writeRgitStore volRoot u
            storeType' <- Device.detectStorageType volRoot
            mSerial <- case storeType' of
                Device.Physical -> Device.getHardwareSerial volRoot
                Device.Network -> return Nothing
            Device.writeDeviceFile cwd deviceName' (Device.DeviceInfo u storeType' mSerial)
            Device.writeRemoteFile cwd name (Device.TargetDevice deviceName' relPath)
            when (name == "origin") $ void Git.setupBranchTracking
            putStrLn $ "Remote '" ++ name ++ "' Γזע " ++ deviceName' ++ ":" ++ relPath
            putStrLn $ "Device '" ++ deviceName' ++ "' registered (" ++ (case storeType' of Device.Physical -> "physical"; Device.Network -> "network") ++ ")."
            return ()
    case result of
        Right () -> return ()
        Left _err -> do
            -- Cannot create .rgit-store at volume root (e.g. permission denied on C:\)
            -- Fall back to path-based storage for local directories
            Device.writeRemoteFile cwd name (Device.TargetLocalPath absPath)
            void $ Git.addRemote name absPath
            when (name == "origin") $ void Git.setupBranchTracking
            putStrLn $ "Remote '" ++ name ++ "' added (" ++ absPath ++ ")."

doMergeAbort :: IO ()
doMergeAbort = do
    cwd <- Dir.getCurrentDirectory
    let conflictsDir = cwd </> ".rgit" </> "conflicts"
    
    -- Abort git merge
    code <- Git.mergeAbort
    if code /= ExitSuccess
        then hPutStrLn stderr "error: no merge in progress."
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
            let path = dir </> item
            isDir <- doesDirectoryExist path
            if isDir
                then removeDirectoryRecursive path
                else removeFile path
        removeDirectory dir

-- ============================================================================
-- Stateful passthrough (needs RgitEnv)
-- ============================================================================

status :: [String] -> RgitM ()
status args = do
    liftIO $ void $ Git.runGitRaw ("status" : args)

restore :: [String] -> RgitM ()
restore = doRestore

checkout :: [String] -> RgitM ()
checkout = doCheckout

-- ============================================================================
-- Core operations (full business logic)
-- ============================================================================

push :: RgitM ()
push = withRemote $ \remote -> do
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
            liftIO $ putStrLn "Remote is an rgit repo. Checking history..."
            fetchResult <- liftIO $ fetchBundle remote
            case fetchResult of
                BundleFound bPath -> do
                    let fetchedPath = fromCwdPath (bundleCwdPath fetchedBundle)
                    liftIO $ copyFile bPath fetchedPath
                    liftIO $ safeRemove bPath
                    processExistingRemote
                _ -> liftIO $ hPutStrLn stderr "Error: Remote .rgit found but metadata is missing."

        StateNonRgitOccupied samples -> do
            if force
                then do
                    liftIO $ hPutStrLn stderr "Warning: --force used. Overwriting non-rgit remote..."
                    syncRemoteFiles
                    liftIO $ pushBundle remote
                    updateLocalBundleAfterPush
                else do
                    liftIO $ hPutStrLn stderr "-------------------------------------------------------"
                    liftIO $ hPutStrLn stderr "[!] STOP: Remote is NOT an rgit repository!"
                    liftIO $ hPutStrLn stderr $ "Found existing files: " ++ List.intercalate ", " samples
                    liftIO $ hPutStrLn stderr "To initialize anyway (destructive): rgit init --force"
                    liftIO $ hPutStrLn stderr "-------------------------------------------------------"

        StateNetworkError err ->
            liftIO $ hPutStrLn stderr $ "Aborting: Network error -> " ++ err

        StateCorruptedRgit msg ->
            liftIO $ hPutStrLn stderr $ "Aborting: [X] Corrupted remote -> " ++ msg

pull :: PullOptions -> RgitM ()
pull opts = withRemote $ \remote ->
    if pullAcceptRemote opts
        then pullAcceptRemoteImpl remote
        else if pullManualMerge opts
            then pullManualMergeImpl remote
            else pullWithCleanup remote

fetch :: RgitM ()
fetch = withRemote $ \remote -> do
    mb <- liftIO $ fetchRemoteBundle remote
    liftIO $ saveFetchedBundle remote mb

verify :: Bool -> RgitM ()
verify isRemote
  | isRemote = withRemote $ \remote -> do
      cwd <- asks envCwd
      liftIO $ putStrLn "Fetching remote metadata... done."
      liftIO $ putStrLn "Scanning remote files... done."
      liftIO $ putStrLn "Comparing..."
      (fileCount, issues) <- liftIO $ Verify.verifyRemote cwd remote
      liftIO $ putStrLn $ "Verifying " ++ show fileCount ++ " files..."
      if null issues
        then liftIO $ putStrLn "[OK] All files match metadata."
        else do
          liftIO $ mapM_ (printVerifyIssue (\s -> take 16 s ++ if length s > 16 then "..." else "")) issues
          liftIO $ putStrLn $ show (length issues) ++ " issues found."
  | otherwise = do
      cwd <- asks envCwd
      (fileCount, issues) <- liftIO $ Verify.verifyLocal cwd
      liftIO $ putStrLn $ "Verifying " ++ show fileCount ++ " files..."
      if null issues
        then liftIO $ putStrLn "[OK] All files match metadata."
        else do
          liftIO $ mapM_ (printVerifyIssue (\s -> take 16 s ++ if length s > 16 then "..." else "")) issues
          liftIO $ putStrLn $ show (length issues) ++ " issues found. Run 'rgit status' for details."

remoteShow :: Maybe String -> RgitM ()
remoteShow mRemoteName = do
    cwd <- asks envCwd
    name <- case mRemoteName of
        Just n -> return n
        Nothing -> liftIO Git.getTrackedRemoteName
    mRemote <- liftIO $ resolveRemote cwd name
    mTarget <- liftIO $ Device.readRemoteFile cwd name
    display <- liftIO $ case mTarget of
        Just _ -> formatRemoteDisplay cwd name mTarget
        Nothing -> return (name ++ " Γזע " ++ maybe "(not configured)" displayRemote mRemote)
    case mRemote of
        Nothing -> do
            liftIO $ putStrLn "No remote configured. (Use 'rgit remote add <name> <url>')"
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
                            putStrLn "  Local branch configured for 'rgit pull':"
                            putStrLn "    main merges with remote (unknown)"
                            putStrLn ""
                            putStrLn "  Local refs configured for 'rgit push':"
                            putStrLn "    main pushes to main (unknown)"

remoteCheck :: Maybe String -> RgitM ()
remoteCheck mName = do
    cwd <- asks envCwd
    (mRemote, name) <- liftIO $ case mName of
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
            liftIO $ hPutStrLn stderr "hint: Set remote with 'rgit remote add <name> <url>'"
            liftIO $ exitWith (ExitFailure 1)
        Just remote -> do
            liftIO $ putStrLn $ "Checking local against remote: " ++ displayRemote remote
            liftIO $ putStrLn ""
            liftIO $ putStrLn "Using: rclone check --combined"
            res <- liftIO $ try @IOException (Transport.checkRemote cwd remote)
            case res of
                Left _ -> do
                    liftIO $ hPutStrLn stderr "fatal: rclone not found. Install rclone: https://rclone.org/install/"
                    liftIO $ exitWith (ExitFailure 1)
                Right cr -> do
                    let reportPath = cwd </> rgitDir </> "last-check.txt"
                    liftIO $ createDirectoryIfMissing True (cwd </> rgitDir)
                    liftIO $ writeFile reportPath (Transport.checkRawOutput cr)
                    let matches = Transport.checkMatches cr
                        differs = Transport.checkDiffers cr
                        missingDest = Transport.checkMissingDest cr
                        missingSrc = Transport.checkMissingSrc cr
                        errs = Transport.checkErrors cr
                        nMatch = length matches
                        hasDiff = not (null differs && null missingDest && null missingSrc && null errs)
                    if not hasDiff && Transport.checkExitCode cr == ExitSuccess
                        then do
                            liftIO $ putStrLn $ show nMatch ++ " files match between local and remote."
                            liftIO $ exitWith ExitSuccess
                        else if Transport.checkExitCode cr /= ExitSuccess && Transport.checkExitCode cr /= ExitFailure 1
                        then do
                            liftIO $ hPutStrLn stderr "fatal: Could not read from remote."
                            liftIO $ hPutStrLn stderr ""
                            liftIO $ hPutStrLn stderr "Please make sure you have the correct access rights"
                            liftIO $ hPutStrLn stderr "and the remote exists."
                            unless (null (Transport.checkStderr cr)) $ liftIO $ hPutStrLn stderr (Transport.checkStderr cr)
                            liftIO $ exitWith (ExitFailure 1)
                        else do
                            liftIO $ putStrLn ""
                            when (not (null differs)) $ forM_ ("  content differs:" : formatPathList differs) $ \s -> liftIO $ putStrLn s
                            when (not (null missingDest)) $ forM_ ("  local only (not on remote):" : formatPathList missingDest) $ \s -> liftIO $ putStrLn s
                            when (not (null missingSrc)) $ forM_ ("  remote only (not in local):" : formatPathList missingSrc) $ \s -> liftIO $ putStrLn s
                            when (not (null errs)) $ forM_ ("  errors:" : formatPathList errs) $ \s -> liftIO $ putStrLn s
                            let errWord = if length errs == 1 then "1 error" else show (length errs) ++ " errors"
                            liftIO $ putStrLn $ show (length differs + length missingDest + length missingSrc) ++ " differences, "
                                ++ errWord ++ ". " ++ show nMatch ++ " files matched."
                            liftIO $ putStrLn ""
                            liftIO $ hPutStrLn stderr "hint: Content differences may indicate an incomplete push or pull."
                            liftIO $ hPutStrLn stderr "hint: Run 'rgit verify' and 'rgit verify --remote' to check metadata consistency."
                            liftIO $ hPutStrLn stderr "hint: Full report saved to .rgit/last-check.txt"
                            liftIO $ exitWith (ExitFailure 1)

mergeContinue :: RgitM ()
mergeContinue = do
    cwd <- asks envCwd
    mRemote <- asks envRemote
    let conflictsDir = cwd </> ".rgit" </> "conflicts"
    conflictsExist <- liftIO $ Dir.doesDirectoryExist conflictsDir

    gitConflicts <- liftIO Conflict.getConflictedFilesE

    if not (null gitConflicts)
        then liftIO $ hPutStrLn stderr "error: you have not resolved your conflicts yet."
        else if not conflictsExist
            then do
                (code, _, _) <- liftIO $ Git.runGitWithOutput ["rev-parse", "--verify", "MERGE_HEAD"]
                if code == ExitSuccess
                    then do
                        liftIO $ void $ Git.runGitRaw ["commit", "-m", "Merge remote"]
                        liftIO $ putStrLn "Merge complete."
                        maybe (return ()) (\_ -> do
                            liftIO $ putStrLn "Syncing binaries... done."
                            syncRemoteFilesToLocal
                            liftIO $ updateMetadataAfterSync cwd
                            liftIO $ void Git.updateRemoteTrackingBranchToHead) mRemote
                    else liftIO $ hPutStrLn stderr "error: no merge in progress."
            else do
                invalid <- liftIO $ Metadata.validateMetadataDir (cwd </> rgitIndexPath)
                unless (null invalid) $ do
                    liftIO $ hPutStrLn stderr "fatal: Metadata files contain conflict markers. Merge aborted."
                    liftIO $ throwIO (userError "Invalid metadata")

                (code, _, _) <- liftIO $ Git.runGitWithOutput ["rev-parse", "--verify", "MERGE_HEAD"]
                when (code /= ExitSuccess) $ do
                    (mergeCode, _, _) <- liftIO $ Git.runGitWithOutput ["merge", "--no-commit", "--no-ff", "refs/remotes/origin/main"]
                    when (mergeCode /= ExitSuccess) $
                        liftIO $ hPutStrLn stderr "warning: Could not start merge. Proceeding anyway."

                liftIO $ void $ Git.runGitRaw ["commit", "-m", "Merge remote (manual merge resolved)"]
                liftIO $ putStrLn "Merge complete."

                liftIO $ removeDirectoryRecursive conflictsDir
                liftIO $ putStrLn "Conflict directories cleaned up."

                maybe (return ()) (\_ -> do
                    liftIO $ putStrLn "Syncing binaries... done."
                    syncRemoteFilesToLocal
                    liftIO $ updateMetadataAfterSync cwd
                    liftIO $ void Git.updateRemoteTrackingBranchToHead) mRemote

-- ============================================================================
-- Internal helpers (not exported, moved from Commands.hs)
-- ============================================================================

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
isTextFileInIndex localRoot path = do
    let metaPath = localRoot </> rgitIndexPath </> path
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
copyFromIndexToWorkTree localRoot path = do
    let metaPath = localRoot </> rgitIndexPath </> path
        workPath = localRoot </> path
    createDirectoryIfMissing True (takeDirectory workPath)
    copyFile metaPath workPath

-- | Helper to read a file safely (returns Nothing on error)
readFileMaybe :: FilePath -> IO (Maybe String)
readFileMaybe path = do
    exists <- Dir.doesFileExist path
    if exists
        then Just <$> readFile path
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

askUser :: String -> IO String
askUser prompt = do
    putStr prompt
    hFlush stdout
    getLine

gitRaw :: [String] -> IO ExitCode
gitRaw = Git.runGitRaw

gitQuery :: [String] -> IO (ExitCode, String, String)
gitQuery = Git.runGitWithOutput


readFileE :: FilePath -> IO (Maybe String)
readFileE = readFileMaybe

writeFileAtomicE :: FilePath -> String -> IO ()
writeFileAtomicE = writeFile

copyFileE :: FilePath -> FilePath -> IO ()
copyFileE = copyFile

fileExistsE :: FilePath -> IO Bool
fileExistsE = Dir.doesFileExist

dirExistsE :: FilePath -> IO Bool
dirExistsE = Dir.doesDirectoryExist

createDirE :: FilePath -> IO ()
createDirE = createDirectoryIfMissing True

removeFileE :: FilePath -> IO ()
removeFileE = Dir.removeFile

removeDirRecursiveE :: FilePath -> IO ()
removeDirRecursiveE = Dir.removeDirectoryRecursive

getCurrentDirE :: IO FilePath
getCurrentDirE = Dir.getCurrentDirectory

exitWithE :: ExitCode -> IO ()
exitWithE = exitWith

safeRemoveE :: FilePath -> IO ()
safeRemoveE = safeRemove

-- | Run an action with the remote, or print error if not configured.
withRemote :: (Remote -> RgitM ()) -> RgitM ()
withRemote action = do
  mRemote <- asks envRemote
  case mRemote of
    Nothing -> liftIO $ hPutStrLn stderr "Error: No remote configured."
    Just remote -> action remote

-- | Executes/Prints the command to be run in the shell (push: local -> remote).
executeCommand :: FilePath -> Remote -> RcloneAction -> IO ()
executeCommand localRoot remote action = case action of
        Copy src dest -> do
            let localPath = toPosix (localRoot </> src)
            void $ Transport.copyToRemote localPath remote (toPosix dest)

        Move src dest ->
            void $ Transport.moveRemote remote (toPosix src) (toPosix dest)

        Delete path ->
            void $ Transport.deleteRemote remote (toPosix path)

        Swap _ _ _ -> return ()  -- not produced by planAction; future-proofing

-- | Execute a single pull action: copy from remote to local or delete local file.
-- Text files are already in the git bundle (index); copy from index to work dir instead of rclone.
executePullCommand :: FilePath -> Remote -> RcloneAction -> IO ()
executePullCommand localRoot remote action = case action of
        Copy src dest -> do
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
        Delete path -> do
            let localPath = localRoot </> path
            exists <- Dir.doesFileExist localPath
            when exists $ Dir.removeFile localPath
        Swap _ _ _ -> return ()

-- Helper to ensure we don't crash if cleanup fails (IO version for use outside RgitM)
safeRemove :: FilePath -> IO ()
safeRemove path = do
    exists <- Dir.doesFileExist path
    when exists (Dir.removeFile path)


-- | Show up to 10 paths; if more than 20, show first 10 then "... and N more".
formatPathList :: [FilePath] -> [String]
formatPathList paths
  | length paths <= 20 = map (\p -> "        " ++ toPosix p) paths
  | otherwise         = map (\p -> "        " ++ toPosix p) (take 10 paths)
                        ++ ["        ... and " ++ show (length paths - 10) ++ " more"]

-- ============================================================================
-- Domain logic functions (moved from Transport in Step 4)
-- ============================================================================

-- | Classify remote state (empty, valid rgit, non-rgit, corrupted, network error)
-- This is domain logic: it knows what .rgit/ means and interprets remote contents
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
    | ".rgit" `elem` map Transport.tiName items = StateValidRgit
    | otherwise = StateNonRgitOccupied (take 3 (map Transport.tiName items))

-- | Download the remote bundle for comparison. Returns temp bundle path or error.
-- This is domain logic: it knows about .rgit/ layout and bundle files
fetchBundle :: Remote -> IO FetchResult
fetchBundle remote = do
    let localDest = ".rgit/temp_remote.bundle"
    
    result <- Transport.copyFromRemoteDetailed remote ".rgit/rgit.bundle" localDest
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
    let tempBundle = BundleName "rgit"
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
    rCode <- Transport.copyToRemote src remote ".rgit/rgit.bundle"
    if rCode == ExitSuccess
        then putStrLn "Metadata push complete."
        else hPutStrLn stderr "Error uploading bundle."

-- Helper for cleanup that doesn't crash if the file was never made
cleanupTemp :: FilePath -> IO ()
cleanupTemp path = do
    exists <- Dir.doesFileExist path
    when exists (Dir.removeFile path)

-- fetchBundle moved to Internal.Transport


-- | Sync files, push bundle, and update local tracking. Used after remote checks pass.
pushToRemote :: Remote -> RgitM ()
pushToRemote remote = do
  syncRemoteFiles
  liftIO $ pushBundle remote
  updateLocalBundleAfterPush

-- | After a successful push, update the local fetched_remote.bundle to current HEAD
-- so rgit status shows up to date instead of "ahead of remote".
updateLocalBundleAfterPush :: RgitM ()
updateLocalBundleAfterPush = do
    code <- liftIO $ Git.createBundle fetchedBundle
    when (code == ExitSuccess) $ do
        void $ liftIO $ Git.updateRemoteTrackingBranch fetchedBundle

syncRemoteFiles :: RgitM ()
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
                else mapM_ (\a -> liftIO $ executeCommand cwd remote a) actions)
        remoteResult


processExistingRemote :: RgitM ()
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
                                Just fHash | rHash == fHash -> do
                                    lift $ tell "Remote check passed (--force-with-lease). Proceeding with push..."
                                    maybe (lift $ tellErr "Error: No remote configured.") pushToRemote mRemote
                                Just _fHash -> lift $ do
                                    tellErr "---------------------------------------------------"
                                    tellErr "ERROR: Remote has changed since last fetch!"
                                    tellErr "Someone else pushed to the remote."
                                    tellErr "Run 'rgit fetch' to update your local view of the remote."
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
                                    tellErr "Please run 'rgit pull' before pushing."
                                    tellErr "---------------------------------------------------"

                        _ -> lift $ tellErr "Error: Could not extract hashes for comparison."

-- | Sync files from remote to local (make local match remote). Used after pull.
syncRemoteFilesToLocal :: RgitM ()
syncRemoteFilesToLocal = withRemote $ \remote -> do
    cwd <- asks envCwd
    localFiles <- asks envLocalFiles
    remoteResult <- lift $ Remote.Scan.fetchRemoteFiles remote
    either
        (\_ -> lift $ tellErr "Error: Failed to fetch remote file list.")
        (\remoteFiles -> do
            let actions = Pipeline.pullSyncFiles localFiles remoteFiles
            lift $ tell "--- Pulling changes from remote ---"
            if null actions
                then lift $ tell "Working tree already up to date with remote."
                else mapM_ (\a -> lift $ executePullCommand cwd remote a) actions)
        remoteResult

-- | Update metadata files after syncing binaries from remote.
-- Rescans working directory and commits the updated metadata.
updateMetadataAfterSync :: FilePath -> IO ()
updateMetadataAfterSync cwd = do
    -- Rescan working directory to get updated file hashes after binary sync
    updatedFiles <- Scan.scanWorkingDir cwd
    -- Write updated metadata files to .rgit/index
    Scan.writeMetadataFiles cwd updatedFiles
    -- Add and commit the updated metadata (if there are changes)
    Git.add ["."]
    -- Only commit if there are staged changes to avoid "nothing to commit" error
    hasChanges <- Git.hasStagedChanges
    when hasChanges $ void $ Git.commit ["-m", "Update metadata after syncing binaries"]

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
saveFetchedBundle remote Nothing = pure ()
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

    _ <- Git.setupRemote (remoteUrl remote)
    _ <- Git.setupBranchTracking

    case maybeNewHash of
        Just _ -> void $ Git.fetchFromBundle fetchedBundle
        Nothing -> return ()

    -- Output fetch results in git format
    case (maybeOldHash, maybeNewHash) of
        (Nothing, Just newHash) -> do
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

-- | Pull with --accept-remote: accept remote file state as truth.
-- Scans actual remote files, updates local metadata to match, syncs files, and commits.
pullAcceptRemoteImpl :: Remote -> RgitM ()
pullAcceptRemoteImpl remote = do
    cwd <- asks envCwd
    lift $ tell "Accepting remote file state as truth..."
    lift $ tell "Scanning remote files..."

    result <- lift $ Remote.Scan.fetchRemoteFiles remote
    case result of
        Left _ -> lift $ tellErr "Error: Failed to fetch remote file list."
        Right remoteFiles -> do
            let filteredRemoteFiles = filterOutRgitPaths remoteFiles
            lift $ tell "Updating local metadata to match remote state..."
            lift $ Scan.writeMetadataFiles cwd filteredRemoteFiles
            lift $ void $ gitRaw ["add", "."]
            hasChanges <- lift hasStagedChangesE
            when hasChanges $ lift $ void $ gitRaw ["commit", "-m", "Accept remote file state as truth"]
            lift $ tell "Syncing files from remote..."
            syncRemoteFilesToLocal
            lift $ updateMetadataAfterSync cwd
            lift $ void $ Git.updateRemoteTrackingBranchToHead
            lift $ tell "Pull with --accept-remote completed."

-- | Pull with --manual-merge: detect remote divergence and create conflict directories.
pullManualMergeImpl :: Remote -> RgitM ()
pullManualMergeImpl remote = do
    cwd <- asks envCwd
    lift $ tell "Fetching remote metadata... done."

    maybeBundlePath <- lift $ fetchRemoteBundle remote
    case maybeBundlePath of
        Nothing -> lift $ tellErr "Error: Could not fetch remote bundle."
        Just bPath -> do
            lift $ saveFetchedBundle remote (Just bPath)

            remoteMeta <- lift $ Verify.loadMetadataFromBundle fetchedBundle
            lift $ tell "Scanning remote files... done."
            result <- lift $ Remote.Scan.fetchRemoteFiles remote
            case result of
                Left _ -> lift $ tellErr "Error: Could not fetch remote file list."
                Right remoteFiles -> do
                    let filteredRemoteFiles = filterOutRgitPaths remoteFiles
                    localMeta <- lift $ Verify.loadMetadataIndex (cwd </> rgitIndexPath)

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
                            pullWithCleanup remote
                        else do
                            oldHash <- lift getLocalHeadE
                            (remoteCode, remoteOut, _) <- lift $ gitQuery ["rev-parse", "refs/remotes/origin/main"]
                            let newHash = takeWhile (/= '\n') remoteOut

                            (mergeCode, mergeOut, mergeErr) <- lift $ gitQuery ["merge", "--no-commit", "--no-ff", "refs/remotes/origin/main"]
                            (_finalMergeCode, _, _) <- lift $ if mergeCode /= ExitSuccess && "refusing to merge unrelated histories" `List.isInfixOf` (mergeOut ++ mergeErr)
                                then do tell "Merging unrelated histories (e.g. first pull)..."; gitQuery ["merge", "--no-commit", "--no-ff", "--allow-unrelated-histories", "refs/remotes/origin/main"]
                                else return (mergeCode, mergeOut, mergeErr)

                            createConflictDirectories remote divergentFiles remoteFileMap remoteMetaMap localMetaMap

                            lift $ printConflictList divergentFiles remoteFileMap remoteMetaMap localMetaMap
                            lift $ do
                                tell ""
                                tell "To resolve:"
                                tell "  1. Examine files in .rgit/conflicts/<path>/"
                                tell "  2. Copy your chosen version to <path>"
                                tell "  3. Run 'rgit add <path>'"
                                tell "  4. Run 'rgit merge --continue'"
                                tell ""
                                tell "Or abort: 'rgit merge --abort'"

-- | Find files where remote actual files don't match remote metadata.
findDivergentFiles :: Map.Map FilePath (Hash 'MD5, EntryKind) -> Map.Map FilePath (Hash 'MD5, Integer) -> Map.Map FilePath (Hash 'MD5, Integer) -> [(FilePath, Hash 'MD5, Hash 'MD5, Integer, Integer)]
findDivergentFiles remoteFileMap remoteMetaMap localMetaMap =
    Map.foldlWithKey (\acc path (metaHash, metaSize) ->
        case Map.lookup path remoteFileMap of
            Nothing -> acc  -- File missing on remote, skip
            Just (actualHash, kind) ->
                case kind of
                    File _ actualSize _ ->
                        if actualHash == metaHash && actualSize == metaSize
                            then acc  -- Matches, no divergence
                            else (path, metaHash, actualHash, metaSize, actualSize) : acc  -- Divergence!
                    _ -> acc
        ) [] remoteMetaMap

-- | Create conflict directories for divergent files.
createConflictDirectories :: Remote -> [(FilePath, Hash 'MD5, Hash 'MD5, Integer, Integer)] -> Map.Map FilePath (Hash 'MD5, EntryKind) -> Map.Map FilePath (Hash 'MD5, Integer) -> Map.Map FilePath (Hash 'MD5, Integer) -> RgitM ()
createConflictDirectories remote divergentFiles remoteFileMap remoteMetaMap localMetaMap = do
    cwd <- asks envCwd
    let conflictsDir = cwd </> ".rgit" </> "conflicts"
    lift $ createDirE conflictsDir

    forM_ divergentFiles $ \(path, metaHash, actualHash, metaSize, actualSize) -> do
        let conflictDir = conflictsDir </> path
        lift $ createDirE (takeDirectory conflictDir)

        let localPath = cwd </> path
        localExists <- lift $ fileExistsE localPath
        when localExists $ lift $ copyFileE localPath (conflictDir </> "LOCAL")

        code <- liftIO $ Transport.copyFromRemote remote (toPosix path) (conflictDir </> "REMOTE")
        when (code /= ExitSuccess) $ lift $ tellErr $ "Warning: Could not download remote file: " ++ path

        lift $ case Map.lookup (normalise path) localMetaMap of
            Just (localHash, localSize) ->
                writeFileAtomicE (conflictDir </> "METADATA_LOCAL") $
                    serializeMetadata (MetaContent localHash localSize)
            Nothing -> writeFileAtomicE (conflictDir </> "METADATA_LOCAL") "hash: (not tracked)\nsize: 0\n"

        lift $ writeFileAtomicE (conflictDir </> "METADATA_REMOTE") $
            serializeMetadata (MetaContent metaHash metaSize)

-- | Print conflict list in spec format.
printConflictList :: [(FilePath, Hash 'MD5, Hash 'MD5, Integer, Integer)] -> Map.Map FilePath (Hash 'MD5, EntryKind) -> Map.Map FilePath (Hash 'MD5, Integer) -> Map.Map FilePath (Hash 'MD5, Integer) -> IO ()
printConflictList divergentFiles remoteFileMap remoteMetaMap localMetaMap = do
    putStrLn ""
    putStrLn "Γ£ק Remote divergence detected:"
    putStrLn ""
    
    forM_ divergentFiles $ \(path, metaHash, actualHash, metaSize, actualSize) -> do
        putStrLn $ "  " ++ toPosix path ++ ":"
        
        -- Get local metadata (use displayHash for Hash 'MD5 values)
        let localInfo = case Map.lookup (normalise path) localMetaMap of
                Just (localHash, localSize) -> (displayHash localHash, show localSize)
                Nothing -> ("(not tracked)", "0")
        
        putStrLn $ "    Local:           " ++ fst localInfo ++ " (" ++ snd localInfo ++ " bytes)"
        putStrLn $ "    Remote actual:   " ++ displayHash actualHash ++ " (" ++ show actualSize ++ " bytes)"
        putStrLn $ "    Remote metadata: " ++ displayHash metaHash ++ " (" ++ show metaSize ++ " bytes)"
        putStrLn $ ""
        putStrLn $ "    Files saved to: .rgit/conflicts/" ++ toPosix path ++ "/"
        putStrLn ""
    
    putStrLn "This can happen when:"
    putStrLn "  - Files were modified directly on the remote (not via rgit)"
    putStrLn "  - A partial push from another client"
    putStrLn "  - Remote storage corruption"

pullWithCleanup :: Remote -> RgitM ()
pullWithCleanup remote = do
    env <- asks id
    result <- liftIO $ try @SomeException (runRgitM env (pullLogic remote))
    case result of
        Left ex -> do
            inProgress <- lift $ Git.isMergeInProgress
            if inProgress
                then do
                    lift $ void $ gitRaw ["merge", "--abort"]
                    lift $ tell "Merge aborted. Your working tree is unchanged."
                else lift $ throwIO ex
        Right _ -> return ()

pullLogic :: Remote -> RgitM ()
pullLogic remote = do
    cwd <- asks envCwd
    maybeBundlePath <- lift $ fetchRemoteBundle remote
    case maybeBundlePath of
        Nothing -> return ()
        Just bPath -> do
            lift $ saveFetchedBundle remote (Just bPath)
            (_, countOut, _) <- lift $ gitQuery ["rev-list", "--count", "refs/remotes/origin/main"]
            let n = takeWhile (`elem` ['0'..'9']) (filter (/= '\n') countOut)
            lift $ tell $ "remote: Counting objects: " ++ (if null n then "0" else n) ++ ", done."

            oldHash <- lift getLocalHeadE
            (remoteCode, remoteOut, _) <- lift $ gitQuery ["rev-parse", "refs/remotes/origin/main"]
            let newHash = takeWhile (/= '\n') remoteOut

            case oldHash of
                Nothing -> do
                    lift $ tell $ "Checking out " ++ take 7 newHash ++ " (first pull)"
                    checkoutCode <- lift $ Git.checkoutRemoteAsMain
                    if checkoutCode == ExitSuccess
                        then do
                            syncRemoteFilesToLocal
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
                        syncRemoteFilesToLocal
                        lift $ tell "Syncing binaries... done."
                        lift $ updateMetadataAfterSync cwd
                        lift $ void $ Git.updateRemoteTrackingBranchToHead
                    else do
                        lift $ tell finalMergeOut
                        lift $ tellErr finalMergeErr
                        lift $ tell "Automatic merge failed."
                        lift $ tell "rgit requires you to pick a version for each conflict."
                        lift $ tell ""
                        lift $ tell "Resolving conflicts..."

                        conflicts <- lift Conflict.getConflictedFilesE
                        resolutions <- lift $ Conflict.resolveAll conflicts
                        let total = length resolutions

                        invalid <- lift $ Metadata.validateMetadataDir (cwd </> rgitIndexPath)
                        unless (null invalid) $ do
                            lift $ void $ gitRaw ["merge", "--abort"]
                            lift $ tellErr "fatal: Metadata files contain conflict markers. Merge aborted."
                            lift $ throwIO (userError "Invalid metadata")

                        conflictsNow <- lift Conflict.getConflictedFilesE
                        when (null conflictsNow) $ do
                            hasChanges <- lift hasStagedChangesE
                            when hasChanges $ lift $ do
                                void $ gitRaw ["commit", "-m", "Merge remote (resolved " ++ show total ++ " conflict(s))"]
                                tell $ "Merge complete. " ++ show total ++ " conflict(s) resolved."
                        syncRemoteFilesToLocal
                        lift $ tell "Syncing binaries... done."
                        lift $ updateMetadataAfterSync cwd
                        when (null conflictsNow) $ lift $ void $ Git.updateRemoteTrackingBranchToHead

printVerifyIssue :: (String -> String) -> Verify.VerifyIssue -> IO ()
printVerifyIssue fmtHash = \case
  Verify.HashMismatch path expectedHash actualHash _expectedSize _actualSize -> do
    hPutStrLn stderr $ "[ERROR] Hash mismatch: " ++ toPosix path
    hPutStrLn stderr $ "  Expected: " ++ fmtHash expectedHash
    hPutStrLn stderr $ "  Actual:   " ++ fmtHash actualHash
  Verify.Missing path ->
    hPutStrLn stderr $ "[ERROR] Missing: " ++ toPosix path

-- | Format remote display line (e.g. "origin Γזע black_usb:Backup (physical, connected at E:\)")
formatRemoteDisplay :: FilePath -> String -> Maybe Device.RemoteTarget -> IO String
formatRemoteDisplay cwd name mTarget = case mTarget of
    Just (Device.TargetLocalPath p) -> return (name ++ " Γזע " ++ p ++ " (local path)")
    Just (Device.TargetDevice dev path) -> do
        res <- Device.resolveRemoteTarget cwd (Device.TargetDevice dev path)
        mInfo <- Device.readDeviceFile cwd dev
        let typ = maybe "unknown" (\i -> case Device.deviceType i of Device.Physical -> "physical"; Device.Network -> "network") mInfo
        case res of
            Device.Resolved mount -> return (name ++ " Γזע " ++ dev ++ ":" ++ path ++ " (" ++ typ ++ ", connected at " ++ mount ++ ")")
            Device.NotConnected _ -> return (name ++ " Γזע " ++ dev ++ ":" ++ path ++ " (" ++ typ ++ ", NOT CONNECTED)")
    Just (Device.TargetCloud u) -> return (name ++ " Γזע " ++ u ++ " (cloud)")
    Nothing -> return (name ++ " Γזע (no target)")

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
            putStrLn "  Local branch configured for 'rgit pull':"
            putStrLn "    main merges with remote (unknown)"
            putStrLn ""
            putStrLn "  Local refs configured for 'rgit push':"
            putStrLn "    main pushes to main (local out of date)"

        (Just lHash, Just rHash) -> do
            putStrLn "  HEAD branch: main"
            putStrLn ""
            if lHash == rHash
                then do
                    putStrLn "  Local branch configured for 'rgit pull':"
                    putStrLn "    main merges with remote main"
                    putStrLn ""
                    putStrLn "  Local refs configured for 'rgit push':"
                    putStrLn "    main pushes to main (up to date)"
                else do
                    localAhead  <- Git.checkIsAhead rHash lHash
                    remoteAhead <- Git.checkIsAhead lHash rHash

                    putStrLn "  Local branch configured for 'rgit pull':"
                    putStrLn "    main merges with remote main"
                    putStrLn ""
                    putStrLn "  Local refs configured for 'rgit push':"
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
    let indexRoot = cwd </> rgitIndexPath
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
doRestore :: [String] -> RgitM ()
doRestore args = do
    cwd <- asks envCwd
    code <- lift $ gitRaw ("restore" : args)
    when (code == ExitSuccess) $ do
        let stagedOnly = ("--staged" `elem` args || "-S" `elem` args) &&
                         not ("--worktree" `elem` args || "-W" `elem` args)
        unless stagedOnly $ do
            let rawPaths = restoreCheckoutPaths args
            paths <- lift $ expandPathsToFiles cwd rawPaths
            forM_ paths $ \path -> do
                let metaPath = cwd </> rgitIndexPath </> path
                let workPath = cwd </> path
                metaExists <- lift $ fileExistsE metaPath
                when metaExists $ do
                    mcontent <- lift $ readFileE metaPath
                    let isBinaryMetadata = maybe True (\content -> any ("hash: " `isPrefixOf`) (lines content)) mcontent
                    unless isBinaryMetadata $ do
                        lift $ createDirE (takeDirectory workPath)
                        lift $ copyFileE metaPath workPath

-- | Checkout paths from index/HEAD (git checkout [options] -- <path>).
-- Same as restore for path form: restores metadata, copies text files to working dir.
doCheckout :: [String] -> RgitM ()
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
        forM_ paths $ \path -> do
            let metaPath = cwd </> rgitIndexPath </> path
            let workPath = cwd </> path
            metaExists <- lift $ fileExistsE metaPath
            when metaExists $ do
                mcontent <- lift $ readFileE metaPath
                let isBinaryMetadata = maybe True (\content -> any ("hash: " `isPrefixOf`) (lines content)) mcontent
                unless isBinaryMetadata $ do
                    lift $ createDirE (takeDirectory workPath)
                    lift $ copyFileE metaPath workPath

```

---

## Internal/Config.hs

**Path:** `Internal/Config.hs`

*Source file.*

```haskell
module Internal.Config where

import System.FilePath ((</>))

-- | Logical bundle name (e.g. "fetched_remote", "rgit"). NOT a file path.
newtype BundleName = BundleName String deriving (Show, Eq)

-- | Path relative to the git working directory (.rgit/index/).
-- Used for git commands that run with -C .rgit/index.
newtype GitRelPath = GitRelPath FilePath deriving (Show, Eq)

-- | Path relative to CWD. Used for direct filesystem operations (copyFile, doesFileExist, etc).
newtype CwdPath = CwdPath FilePath deriving (Show, Eq)

rgitDir, rgitTargetPath, rgitIgnore, rgitGitDir, rgitIndexPath, rgitDevicesDir, rgitRemotesDir :: FilePath
rgitDir           = ".rgit"
rgitTargetPath    = rgitDir </> "target"
rgitIgnore        = rgitDir </> "ignore"
rgitDevicesDir    = rgitDir </> "devices"
rgitRemotesDir    = rgitDir </> "remotes"
rgitIndexPath     = rgitDir </> "index"
rgitGitDir        = rgitIndexPath </> ".git"

-- | The logical name of the fetched remote bundle
fetchedBundle :: BundleName
fetchedBundle = BundleName "fetched_remote"

-- | Legacy string name for gradual migration
fetchedBundleName :: FilePath
fetchedBundleName = "fetched_remote"

-- | Legacy CWD path for gradual migration
fetchedBundlePath :: FilePath
fetchedBundlePath = rgitIndexPath </> ".git" </> (fetchedBundleName ++ ".bundle")

-- | Convert a bundle name to a path relative to git working directory (.rgit/index/)
-- Use this for git commands that run with -C .rgit/index
bundleGitRelPath :: BundleName -> GitRelPath
bundleGitRelPath (BundleName n) = GitRelPath (".git" </> (n ++ ".bundle"))

-- | Convert a bundle name to a path relative to CWD
-- Use this for filesystem operations (copyFile, doesFileExist, etc.)
bundleCwdPath :: BundleName -> CwdPath
bundleCwdPath (BundleName n) = CwdPath (rgitIndexPath </> ".git" </> (n ++ ".bundle"))

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
import Data.Char (isSpace, toLower)
import Data.List (isPrefixOf)
import Data.Maybe (fromMaybe)
import Control.Monad (when)
import qualified Data.Text as T
import qualified Data.Text.IO as TIO
import Internal.Config (rgitDir)

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
configPath = rgitDir </> "config"

-- | Read the entire config file and return TextConfig
-- Falls back to defaultTextConfig if file doesn't exist or parsing fails
readTextConfig :: IO TextConfig
readTextConfig = do
  exists <- doesFileExist configPath
  if not exists
    then return defaultTextConfig
    else do
      content <- TIO.readFile configPath
      return $ parseConfig content

-- | Read config file (for future expansion)
readConfig :: IO TextConfig
readConfig = readTextConfig

-- | Parse config file content (INI-style format)
parseConfig :: T.Text -> TextConfig
parseConfig content = 
  let lines = T.lines content
      -- Find [text] section
      textSection = extractSection "text" lines
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
extractSection sectionName lines =
  let sectionHeader = "[" ++ sectionName ++ "]"
      -- Find start of section
      startIdx = case findIndex (\l -> T.strip l == T.pack sectionHeader) lines of
        Nothing -> length lines  -- Section not found
        Just idx -> idx + 1
      -- Find end of section (next [section] or EOF)
      endIdx = case findIndex (\l -> T.stripStart l `T.isPrefixOf` T.pack "[") (drop startIdx lines) of
        Nothing -> length lines
        Just idx -> startIdx + idx
  in map T.strip $ take (endIdx - startIdx) (drop startIdx lines)

-- | Parse size-limit from section lines
parseSizeLimit :: [T.Text] -> Maybe Integer
parseSizeLimit lines =
  let findLine prefix = [T.unpack (T.drop (T.length (T.pack prefix)) (T.strip l)) | l <- lines, T.stripStart l `T.isPrefixOf` T.pack prefix]
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
parseExtensions lines =
  let findLine prefix = [T.unpack (T.drop (T.length (T.pack prefix)) (T.strip l)) | l <- lines, T.stripStart l `T.isPrefixOf` T.pack prefix]
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
    , ConflictType(..)
    , readFileFromRef
    , listFilesInRef
    , fsck
    , hasStagedChanges
    ) where

import Data.List (lines)
import Data.Maybe (mapMaybe, listToMaybe)

import System.Process (readProcessWithExitCode)
import System.Exit (ExitCode(..))
import System.FilePath (takeDirectory, (</>))
import Internal.Config
import Data.Char (isSpace)
import Control.Monad (when, guard)
import Prelude hiding (init)
import Data.List (isPrefixOf)
import System.IO (hPutStr, hPutStrLn, stderr)
import System.Environment (lookupEnv)

baseFlags = ["-C", rgitIndexPath]
gitDir = ".git"

-- | Represents the subset of Git functionality rgit uses
data GitCommand
    = Init { separateGitDir :: FilePath }
    | Config { name :: String, value :: String }
    | RevParse { ref :: String }
    | CommitFile { message :: String, file :: FilePath }
    | RevList { left :: String, right :: String }
    | CreateBundle { createBundlePath :: FilePath }
    | GetBundleHead { getBundleHeadPath :: FilePath }
    | IsAncestor { ancestor :: String, descendant :: String }
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
        Init dir ->
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
getHashFromBundle name = do
    let (GitRelPath relPath) = bundleGitRelPath name
    (code, out, _) <- runGit (GetBundleHead relPath)
    return $ guard (code == ExitSuccess && not (null out)) >> listToMaybe (words out)

runGitCommand :: GitCommand -> IO ExitCode
runGitCommand cmd = do
    (c, o, e) <- runGit cmd
    -- Don't print error messages for IsAncestor since non-zero exit codes are expected
    -- (they indicate "no, not an ancestor" which is a valid answer, not an error)
    when (c /= ExitSuccess && not (isAncestorCommand cmd)) $ 
        hPutStrLn stderr ("rgit: git command failed: " ++ e)
    putStr o
    hPutStr stderr e
    return c
  where
    isAncestorCommand (IsAncestor _ _) = True
    isAncestorCommand _ = False

commitFile message file = runGitCommand (CommitFile message file)

init dir = runGitCommand (Init dir)

createBundle :: BundleName -> IO ExitCode
createBundle name = do
    let (GitRelPath relPath) = bundleGitRelPath name
    runGitCommand (CreateBundle relPath)

config name value = runGitCommand (Config name value)

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
    replace "(use \"git " "(use \"rgit "


guardedArgs :: [String] -> IO [String]
guardedArgs args =
  if any (\a -> "--git-dir" `isPrefixOf` a || "--work-tree" `isPrefixOf` a || a == "-C") args
     then fail "rgit: overriding git-dir/work-tree/-C is not allowed"
     else pure args

runGitRaw :: [String] -> IO ExitCode
runGitRaw args = do
  noColor <- lookupEnv "RGIT_NO_COLOR"
  let colorFlag = case noColor of
        Just "1" -> "never"
        Just "true" -> "never"
        _ -> "always"
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
      hPutStrLn stderr ("rgit: git exited with code " ++ show n)

  pure code

add     = runGitRaw . ("add" :)
commit  = runGitRaw . ("commit" :)
diff    = runGitRaw . ("diff" :)
restore  = runGitRaw . ("restore" :)
checkout = runGitRaw . ("checkout" :)
status   = runGitRaw . ("status" :)
reset   = runGitRaw . ("reset" :)
rm      = runGitRaw . ("rm" :)
mv      = runGitRaw . ("mv" :)
branch  = runGitRaw . ("branch" :)
merge   = runGitRaw . ("merge" :)

-- | Add or update a remote (Git-style: git remote add <name> <url> / set-url if exists)
addRemote :: String -> String -> IO ExitCode
addRemote name url = do
    (code, _, _) <- readProcessWithExitCode "git" (baseFlags ++ ["remote", "get-url", name]) ""
    case code of
        ExitSuccess -> do
            readProcessWithExitCode "git" (baseFlags ++ ["remote", "set-url", name, url]) "" >>= \(c, _, _) -> return c
        ExitFailure _ -> do
            readProcessWithExitCode "git" (baseFlags ++ ["remote", "add", name, url]) "" >>= \(c, _, _) -> return c

-- | Get the URL for a remote by name (git remote get-url <name>). Returns Nothing if remote missing.
getRemoteUrl :: String -> IO (Maybe String)
getRemoteUrl name = do
    (code, out, _) <- readProcessWithExitCode "git" (baseFlags ++ ["remote", "get-url", name]) ""
    if code /= ExitSuccess then return Nothing
    else return (Just (filter (/= '\n') out))

-- | Get the remote name that the current branch tracks (branch.main.remote). Defaults to "origin".
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
fetchFromBundle name = do
    let (GitRelPath bundle) = bundleGitRelPath name
    (code, out, err) <- readProcessWithExitCode "git"
        (baseFlags ++ ["fetch", bundle, "refs/heads/main:refs/remotes/origin/main"]) ""
    putStr out
    hPutStr stderr err
    return code

-- | Update the remote tracking branch refs/remotes/origin/main to point to the hash from the bundle.
-- Use when the objects are already in the repo (e.g. after push); for fetch/pull use fetchFromBundle.
updateRemoteTrackingBranch :: BundleName -> IO ExitCode
updateRemoteTrackingBranch name = do
    maybeHash <- getHashFromBundle name
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

-- | Set refs/remotes/origin/main to current HEAD. Use after a successful pull so status shows
-- "up to date with 'origin/main'" instead of "ahead by N commits".
updateRemoteTrackingBranchToHead :: IO ExitCode
updateRemoteTrackingBranchToHead = do
    (code, out, _) <- readProcessWithExitCode "git" (baseFlags ++ ["rev-parse", "HEAD"]) ""
    case filter (/= '\n') out of
        hash | code == ExitSuccess && not (null hash) ->
            updateRemoteTrackingBranchToHash hash
        _ -> return (ExitFailure 1)

-- | Set up the local branch to track origin/main
-- This configures branch.main.remote and branch.main.merge so git status knows what to compare
setupBranchTracking :: IO ExitCode
setupBranchTracking = do
    -- Set branch.main.remote = origin and branch.main.merge = refs/heads/main
    (code1, _, _) <- readProcessWithExitCode "git" (baseFlags ++ ["config", "branch.main.remote", "origin"]) ""
    (code2, _, _) <- readProcessWithExitCode "git" (baseFlags ++ ["config", "branch.main.merge", "refs/heads/main"]) ""
    case (code1, code2) of
        (ExitSuccess, ExitSuccess) -> return ExitSuccess
        _ -> return (ExitFailure 1)

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
-- Uses -f (force) to overwrite any local files created during init.
checkoutRemoteAsMain :: IO ExitCode
checkoutRemoteAsMain = do
  -- Use checkout -B to create/reset branch and checkout in one step
  -- Use -f to force overwrite of any local files (like .gitattributes from init)
  (code, _, _) <- runGitWithOutput ["checkout", "-f", "-B", "main", "refs/remotes/origin/main"]
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
readFileFromRef ref path = do
  (code, out, err) <- runGitWithOutput ["show", ref ++ ":" ++ path]
  if code == ExitSuccess && not (null out)
    then return (Just out)
    else return Nothing

-- | List all files in a Git ref's tree (recursive). Returns paths relative to work tree root.
listFilesInRef :: String -> IO [FilePath]
listFilesInRef ref = do
  (code, out, _) <- runGitWithOutput ["ls-tree", "-r", "--name-only", ref]
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

import System.Process (readProcessWithExitCode)
import System.Exit (ExitCode(..))
import System.Directory (removeFile, doesFileExist)
import System.IO (hPutStrLn, stderr)
import System.FilePath (normalise)
import Control.Monad (when)
import Data.List (isInfixOf, isPrefixOf)
import qualified Data.Aeson as Aeson
import qualified Data.ByteString.Lazy.Char8 as LBS
import GHC.Generics (Generic)
import Data.Maybe (fromMaybe)
import Data.String (fromString)
import Rgit.Remote (Remote, remoteUrl)

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

-- | Purge entire remote (no relative path Γאפ purges the remote root)
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
listRemoteJson :: Remote -> Int -> IO (ExitCode, String, String)
listRemoteJson remote maxDepth =
    readProcessWithExitCode "rclone" ["lsjson", "--max-depth", show maxDepth, remoteUrl remote] ""

-- | List remote directory items (at remote root, parsed)
listRemoteItems :: Remote -> Int -> IO (Either String [TransportItem])
listRemoteItems remote maxDepth = do
    (code, out, err) <- listRemoteJson remote maxDepth
    case code of
        ExitFailure _ -> 
            if "directory not found" `isInfixOf` err 
            then return (Right [])  -- Empty directory
            else return (Left err)  -- Network or other error
        ExitSuccess -> do
            case Aeson.decode (LBS.pack out) :: Maybe [RcloneItem] of
                Nothing -> return (Left "Failed to parse rclone JSON output")
                Just items -> return (Right [TransportItem (name item) (isDir item) | item <- items])

-- | List remote recursively with hashes
listRemoteJsonWithHash :: Remote -> IO (ExitCode, String, String)
listRemoteJsonWithHash remote =
    readProcessWithExitCode "rclone" ["lsjson", remoteUrl remote, "--hash", "--recursive"] ""

-- | Check local against remote
checkRemote :: FilePath -> Remote -> IO CheckResult
checkRemote localPath remote = do
    let args = [ "check"
               , localPath
               , remoteUrl remote
               , "--combined", "-"
               , "--exclude", ".rgit/**"
               ]
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
  where
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

## Rgit.hs

**Path:** `Rgit.hs`

*Entry point — main executable.*

```haskell
import Rgit.Commands

main = Rgit.Commands.run
```

---

## Rgit/Commands.hs

**Path:** `Rgit/Commands.hs`

*Source file.*

```haskell
{-# LANGUAGE LambdaCase #-}

module Rgit.Commands (run) where

import qualified Bit
import Rgit.Types (RgitEnv(..), runRgitM)
import qualified Rgit.Scan as Scan  -- Only for the pre-scan in runCommand
import Rgit.Remote (getDefaultRemote, resolveRemote)
import System.Environment (getArgs)
import System.Exit (ExitCode(..), exitWith)
import System.IO (hPutStrLn, stderr)
import Control.Monad (when, unless, void)
import qualified System.Directory as Dir

run :: IO ()
run = do
    args <- getArgs
    case args of
        [] -> hPutStrLn stderr "Usage: rgit [init|status|add|commit|restore|checkout|fetch|pull|push|verify|verify --remote|fsck|branch --unset-upstream|remote add <name> <url>|remote show [<name>]|remote check [<name>]]"
        _  -> runCommand args

runCommand :: [String] -> IO ()
runCommand args = do
    let isForce = "--force" `elem` args || "-f" `elem` args
    let isForceWithLease = "--force-with-lease" `elem` args
    when (isForce && isForceWithLease) $ do
        hPutStrLn stderr "fatal: Cannot use both --force and --force-with-lease"
        exitWith (ExitFailure 1)
    let cmd = filter (`notElem` ["--force", "-f", "--force-with-lease"]) args

    cwd <- Dir.getCurrentDirectory

    -- Pre-scan (skip for commands that don't need it)
    -- TODO: Move this scan into Bit.hs later - it builds the env, so keeping it here for now
    let skipScan = cmd == ["init"]
                || cmd `elem` [["verify"], ["verify", "--remote"], ["fsck"], ["remote", "check"]]
                || (length cmd == 3 && take 2 cmd == ["remote", "check"])
    localFiles <- if skipScan then return [] else Scan.scanWorkingDir cwd
    unless skipScan $ Scan.writeMetadataFiles cwd localFiles
    mRemote <- getDefaultRemote cwd

    let env = RgitEnv cwd localFiles mRemote isForce isForceWithLease

    case cmd of
        ["init"]                        -> Bit.init
        ["remote", "add", name, url]    -> Bit.remoteAdd name url
        ["remote", "show"]              -> runRgitM env $ Bit.remoteShow Nothing
        ["remote", "show", name]        -> do
            mNamedRemote <- resolveRemote cwd name
            let envWithRemote = env { envRemote = mNamedRemote }
            runRgitM envWithRemote $ Bit.remoteShow (Just name)
        ["remote", "check"]             -> runRgitM env $ Bit.remoteCheck Nothing
        ["remote", "check", name]       -> do
            mNamedRemote <- resolveRemote cwd name
            let envWithRemote = env { envRemote = mNamedRemote }
            runRgitM envWithRemote $ Bit.remoteCheck (Just name)
        ["verify"]                      -> runRgitM env $ Bit.verify False
        ["verify", "--remote"]          -> runRgitM env $ Bit.verify True
        ["fsck"]                        -> Bit.fsck cwd
        ("add":rest)                    -> void $ Bit.add rest
        ("commit":rest)                 -> void $ Bit.commit rest
        ("diff":rest)                   -> void $ Bit.diff rest
        ("restore":rest)                -> runRgitM env $ Bit.restore rest
        ("checkout":rest)               -> runRgitM env $ Bit.checkout rest
        ("status":rest)                 -> runRgitM env $ Bit.status rest
        ["fetch"]                       -> runRgitM env Bit.fetch
        ["pull"]                        -> runRgitM env $ Bit.pull Bit.defaultPullOptions
        ["pull", "--accept-remote"]     -> runRgitM env $ Bit.pull Bit.defaultPullOptions { Bit.pullAcceptRemote = True }
        ["pull", "--manual-merge"]      -> runRgitM env $ Bit.pull Bit.defaultPullOptions { Bit.pullManualMerge = True }
        ["push"]                        -> runRgitM env Bit.push
        ["merge", "--continue"]         -> runRgitM env Bit.mergeContinue
        ["merge", "--abort"]            -> Bit.mergeAbort
        ["branch", "--unset-upstream"]  -> Bit.unsetUpstream
        _                               -> hPutStrLn stderr "Unknown command."
```

---

## Rgit/Conflict.hs

**Path:** `Rgit/Conflict.hs`

*Source file.*

```haskell
{-# LANGUAGE DataKinds #-}

module Rgit.Conflict
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
import Rgit.Internal.Metadata (MetaContent(..), parseMetadata, displayHash)

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

## Rgit/Device.hs

**Path:** `Rgit/Device.hs`

*Source file.*

```haskell
{-# LANGUAGE OverloadedStrings #-}

-- | Device-identity-based remote resolution for filesystem remotes.
-- Cloud remotes (rclone) use URL-based identity; filesystem remotes use
-- UUID + hardware serial (physical) or UUID only (network).
module Rgit.Device
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
    -- .rgit-store
  , readRgitStore
  , writeRgitStore
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
import System.FilePath ((</>), pathSeparator, takeDrive, dropDrive)
import System.Process (readProcessWithExitCode)
import System.Exit (ExitCode(ExitSuccess))
import qualified System.Info as Info
import Data.UUID (UUID, toString, fromString)
import Data.UUID.V4 (nextRandom)
import Internal.Config (rgitDevicesDir, rgitRemotesDir)

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
  | TargetLocalPath FilePath     -- Legacy: path when .rgit-store at volume root cannot be created
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
    (prefix, _:rest) | not (null prefix) -> do
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
    , not (null (trim line))
    ]
  where trim = reverse . dropWhile (== ' ') . reverse . dropWhile (== ' ')

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
    (code, out, _) <- readProcessWithExitCode "powershell" ["-NoProfile", "-Command",
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
      (code2, out2, _) <- readProcessWithExitCode "wmic" ["path", "win32_logicaldisk", "where", "DeviceID='" ++ drive ++ ":\\'", "get", "VolumeSerialNumber"] ""
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
-- .rgit-store (on device at volume root)
-- ---------------------------------------------------------------------------

rgitStoreFileName :: FilePath
rgitStoreFileName = ".rgit-store"

readRgitStore :: FilePath -> IO (Maybe UUID)
readRgitStore volumeRoot = do
  let storePath = volumeRoot </> rgitStoreFileName
  exists <- Dir.doesFileExist storePath
  if not exists then return Nothing
  else do
    content <- readFile storePath
    return (parseRgitStoreUuid content)

parseRgitStoreUuid :: String -> Maybe UUID
parseRgitStoreUuid content =
  listToMaybe [ fromString (trim (drop 5 line)) | line <- lines content, "uuid:" `isPrefixOf` line ]
  >>= id  -- join: Maybe (Maybe UUID) -> Maybe UUID

writeRgitStore :: FilePath -> UUID -> IO ()
writeRgitStore volumeRoot u = do
  now <- getCurrentTime
  let ts = formatTime defaultTimeLocale "%Y-%m-%dT%H:%M:%SZ" now
  let content = unlines
        [ "uuid: " ++ toString u
        , "created: " ++ ts
        ]
  let storePath = volumeRoot </> rgitStoreFileName
  writeFile storePath content
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
  let path = repoRoot </> rgitDevicesDir </> deviceName
  exists <- Dir.doesFileExist path
  if not exists then return Nothing
  else parseDeviceFile <$> readFile path

writeDeviceFile :: FilePath -> String -> DeviceInfo -> IO ()
writeDeviceFile repoRoot deviceName info = do
  Dir.createDirectoryIfMissing True (repoRoot </> rgitDevicesDir)
  let path = repoRoot </> rgitDevicesDir </> deviceName
  let body = unlines $
        [ "uuid: " ++ toString (deviceUuid info)
        , "type: " ++ (case deviceType info of Physical -> "physical"; Network -> "network")
        ] ++ [ "hardware_serial: " ++ s | Just s <- [hardwareSerial info] ]
  writeFile path body

listDeviceNames :: FilePath -> IO [String]
listDeviceNames repoRoot = do
  let dir = repoRoot </> rgitDevicesDir
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
  let path = repoRoot </> rgitRemotesDir </> remoteName
  exists <- Dir.doesFileExist path
  if not exists then return Nothing
  else do
    raw <- parseRemoteFile <$> readFile path
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
  Dir.createDirectoryIfMissing True (repoRoot </> rgitRemotesDir)
  let path = repoRoot </> rgitRemotesDir </> remoteName
  let line = case target of
        TargetCloud url -> "target: " ++ url
        TargetDevice dev p -> "target: " ++ dev ++ ":" ++ p
        TargetLocalPath p -> "target: local:" ++ p
  writeFile path line

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
  mStoreUuid <- readRgitStore mountRoot
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

## Rgit/DevicePrompt.hs

**Path:** `Rgit/DevicePrompt.hs`

*Source file.*

```haskell
{-# LANGUAGE LambdaCase #-}

-- | Device name acquisition for filesystem remotes.
-- Supports injectable I/O for testing the interactive path.
module Rgit.DevicePrompt
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
      putStrLn "rgit identifies devices, not drive letters. The remote will stay linked"
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

-- | Production entry point: detects TTY or RGIT_USE_STDIN for testing.
acquireDeviceNameAuto
  :: Maybe String
  -> (String -> IO Bool)
  -> IO String
acquireDeviceNameAuto mLabel nameExists = do
  isTTY <- hIsTerminalDevice stdin
  useStdin <- (== Just "1") <$> lookupEnv "RGIT_USE_STDIN"
  let src = if isTTY
            then Interactive getLine
            else if useStdin
                 then Interactive getLine  -- For tests: pipe input to stdin
                 else NonInteractive
  acquireDeviceName src mLabel nameExists
```

---

## Rgit/Diff.hs

**Path:** `Rgit/Diff.hs`

*Source file.*

```haskell
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE DuplicateRecordFields #-}
{-# LANGUAGE OverloadedRecordDot #-}

module Rgit.Diff
  ( GitDiff(..)
  , LightFileEntry(..)
  , FileIndex(..)
  , buildIndexFromFileEntries
  , computeDiff
  , formatDiff
  ) where

import qualified Data.Map as Map
import qualified Data.Set as Set
import Rgit.Types

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
      [ Modified (LightFileEntry path lHash)
      | path <- Set.toList (Set.intersection lFilePaths rFilePaths)
      , let lHash = lFiles Map.! path
      , let rHash = rFiles Map.! path
      , lHash /= rHash
      ]

    -- 2. Added: path exists only locally
    added =
      [ Added (LightFileEntry path hash)
      | (path, hash) <- Map.toList lFiles
      , path `Set.notMember` rPaths
      , not (Map.member hash remote.byHash)
      ]

    -- 3. Deleted: path exists only remotely
    deleted =
      [ Deleted (LightFileEntry path hash)
      | (path, hash) <- Map.toList rFiles
      , path `Set.notMember` lPaths
      , not (Map.member hash local.byHash)
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

## Rgit/Effect/Pure.hs

**Path:** `Rgit/Effect/Pure.hs`

*Source file.*

```haskell
{-# LANGUAGE GADTs #-}
{-# LANGUAGE FlexibleContexts #-}

module Rgit.Effect.Pure
  ( Trace(..)
  , PureEnv(..)
  , runPure
  ) where

import Control.Monad.Free (Free(..))
import Control.Monad.State (State, runState, get, put, modify)
import qualified Data.ByteString as BS
import qualified Data.ByteString.Char8 as BSC
import qualified Data.Map as Map
import Rgit.Effect
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

## Rgit/Fsck.hs

**Path:** `Rgit/Fsck.hs`

*Source file.*

```haskell
{-# LANGUAGE LambdaCase #-}
{-# LANGUAGE OverloadedStrings #-}

module Rgit.Fsck
  ( doFsck
  ) where

import qualified Rgit.Verify as Verify
import qualified Rgit.Utils as Utils
import qualified Internal.Git as Git
import System.FilePath ((</>))
import System.Exit (ExitCode(..), exitWith)
import Control.Monad (unless, when)
import System.IO (hPutStr, hPutStrLn, hFlush, hSetBuffering, BufferMode(..), stderr)

-- | Run local-only integrity check in the spirit of git fsck: no network.
-- [1/2] Local working tree vs local metadata (rgit equivalent of checking objects).
-- [2/2] git fsck on .rgit/index/.git (metadata history integrity).
-- Prints one line per problem (git-style: "missing <path>", "hash mismatch <path>")
-- and passes through git fsck output. Exits 1 if any check finds issues.
doFsck :: FilePath -> IO ()
doFsck cwd = do
  hSetBuffering stderr NoBuffering
  -- [1/2] Working tree vs local metadata
  (_, localIssues) <- Verify.verifyLocal cwd
  let localOk = null localIssues
  unless localOk $ do
    mapM_ (printIssue Utils.toPosix) localIssues
    hFlush stderr

  -- [2/2] Metadata history (git fsck in .rgit/index/.git)
  (gitCode, gitOut, gitErr) <- Git.fsck
  let gitOk = gitCode == ExitSuccess
  unless gitOk $ do
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
```

---

## Rgit/Internal/Metadata.hs

**Path:** `Rgit/Internal/Metadata.hs`

*Source file.*

```haskell
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE KindSignatures #-}
module Rgit.Internal.Metadata
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

import Rgit.Types (Hash(..), HashAlgo(..), hashToText)
import System.Directory (doesFileExist, doesDirectoryExist, listDirectory)
import System.FilePath ((</>))
import Data.List (isPrefixOf, isInfixOf)
import Data.Maybe (listToMaybe)
import Control.Monad (filterM)
import qualified Data.ByteString as BS
import qualified Data.Text as T
import Data.Text.Encoding (decodeUtf8)
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
    else parseMetadata <$> readFile fp

-- | Read a metadata file OR (if it's a text file whose content is stored directly)
-- compute hash/size from the file bytes. This is the replacement for the fallback
-- logic in Rgit.Scan.readMetadataFile and Rgit.Verify.loadMetadataIndex.
readMetadataOrComputeHash :: FilePath -> IO (Maybe MetaContent)
readMetadataOrComputeHash fp = do
  exists <- doesFileExist fp
  if not exists
    then pure Nothing
    else do
      content <- readFile fp
      case parseMetadata content of
        Just mc -> pure (Just mc)
        Nothing -> do
          -- Not a metadata file Γאפ treat as text file content, compute hash from bytes
          bs <- BS.readFile fp
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

-- | Compute MD5 hash of a file on disk.
hashFile :: FilePath -> IO (Hash 'MD5)
hashFile fp = hashFileBytes <$> BS.readFile fp

-- Conflict marker utilities (preserved from old Internal.Metadata) --

conflictMarkers :: [String]
conflictMarkers = ["<<<<<<<", "=======", ">>>>>>>"]

hasConflictMarkers :: FilePath -> IO Bool
hasConflictMarkers path = do
  content <- readFile path
  return $ any (\m -> m `isInfixOf` content) conflictMarkers

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

## Rgit/Pipeline.hs

**Path:** `Rgit/Pipeline.hs`

*Source file.*

```haskell
{-# LANGUAGE DataKinds #-}

module Rgit.Pipeline
  ( -- * Pure core (property-testable)
    diffAndPlan
    -- * Composed pipelines
  , pushSyncFiles
  , pullSyncFiles
  ) where

import Rgit.Types
import Rgit.Diff (buildIndexFromFileEntries, computeDiff, GitDiff)
import Rgit.Plan (RcloneAction(..), planAction)
import Rgit.Utils (filterOutRgitPaths)

-- | Pure core: given source-of-truth files and current target files,
-- produce the list of actions to make target match source.
-- This is the entire "diff >>> plan" section Γאפ no IO, fully property-testable.
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
  diffAndPlan localFiles (filterOutRgitPaths remoteFiles)

-- | Pull pipeline: compute actions to make local match remote.
-- Note the reversed argument order: remote is source of truth, local is target.
pullSyncFiles :: [FileEntry] -> [FileEntry] -> [RcloneAction]
pullSyncFiles localFiles remoteFiles =
  diffAndPlan (filterOutRgitPaths remoteFiles) localFiles
```

---

## Rgit/Plan.hs

**Path:** `Rgit/Plan.hs`

*Source file.*

```haskell
{-# LANGUAGE DuplicateRecordFields #-}
{-# LANGUAGE OverloadedRecordDot #-}

module Rgit.Plan
  ( RcloneAction(..)
  , planAction
  ) where

import Rgit.Diff (GitDiff(..), LightFileEntry(..))
import Rgit.Types

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

-- | Safety Planner: Logic to handle complex swaps using temporary paths
-- Thanks to ADTs, the system always knows if a temp file is required
makeSwapPlan :: Path -> Path -> [RcloneAction]
makeSwapPlan pathA pathB =
    let tempPath = pathA ++ ".tmp"
    in [ Swap tempPath pathA pathB ]

```

---

## Rgit/Remote.hs

**Path:** `Rgit/Remote.hs`

*Source file.*

```haskell
module Rgit.Remote
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
import qualified Rgit.Device as Device
import Internal.Config (rgitRemotesDir)
import System.FilePath ((</>))
import System.Directory (doesFileExist)
import Data.Maybe (fromMaybe)

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

## Rgit/Remote/Scan.hs

**Path:** `Rgit/Remote/Scan.hs`

*Source file.*

```haskell
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE OverloadedRecordDot #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE DeriveGeneric #-}

module Rgit.Remote.Scan
  ( fetchRemoteFiles
  , RemoteError(..)
  ) where

import GHC.Generics (Generic)
import qualified Data.Map as Map
import Data.Map (Map)
import qualified Data.ByteString.Lazy.Char8 as LBSC
import Data.Aeson (FromJSON(..), decode, withObject, (.:), (.:?))
import System.Exit (ExitCode(..))
import System.FilePath (normalise)
import Data.Maybe
import qualified Data.Text as T
import Data.Time (UTCTime)
import Rgit.Types
import qualified Internal.Transport as Transport
import Rgit.Remote (Remote)

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
    (code, raw, _err) <- Transport.listRemoteJsonWithHash remote
    case code of
        ExitSuccess -> pure $ maybe
            (Left (DecodeFailed "Invalid rclone JSON output"))
            (Right . map rcloneFileToFileEntry . filter (not . rfIsDir))
            (decode (LBSC.pack raw) :: Maybe [RcloneFile])
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

-- TODO: rclone provides ModTime; wire it later
stubTime :: UTCTime
stubTime = read "2026-01-01 00:00:00 UTC"
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

## Rgit/Scan.hs

**Path:** `Rgit/Scan.hs`

*Source file.*

```haskell
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE NamedFieldPuns #-}
{-# LANGUAGE OverloadedStrings #-}

module Rgit.Scan
  ( scanWorkingDir
  , writeMetadataFiles
  , readMetadataFile
  , listMetadataPaths
  , getFileHashAndSize
  , FileEntry(..)
  , EntryKind(..)
  ) where

import Rgit.Types
import System.FilePath
import System.Directory
    ( doesDirectoryExist,
      doesFileExist,
      listDirectory,
      getFileSize,
      createDirectoryIfMissing,
      copyFileWithMetadata )
import Data.List
import qualified Data.ByteString as BS
import Data.Text (unpack)
import Control.Monad
import Data.Text.Encoding (decodeUtf8')
import Data.Char (toLower)
import qualified Internal.ConfigFile as ConfigFile
import Rgit.Utils (atomicWriteFileStr)
import Rgit.Internal.Metadata (MetaContent(..), readMetadataOrComputeHash, hashFile, serializeMetadata)

-- Binary file extensions that should never be treated as text (hardcoded, not configurable)
binaryExtensions :: [String]
binaryExtensions = [".mp4", ".zip", ".bin", ".exe", ".dll", ".so", ".dylib", ".jpg", ".jpeg", ".png", ".gif", ".pdf", ".gz", ".bz2", ".xz", ".tar", ".rar", ".7z", ".iso", ".img", ".dmg", ".deb", ".rpm", ".msi"]

-- | Classify a file as text or binary based on heuristics:
-- 1. Size < configured limit (from .rgit/config)
-- 2. No NULL bytes in first 8KB
-- 3. Valid UTF-8 (or ASCII subset)
-- 4. Not in binary extension list
-- 5. Extension matches configured text extensions (optional hint)
classifyFile :: FilePath -> Integer -> IO Bool
classifyFile filePath size = do
    config <- ConfigFile.readTextConfig
    -- Check size limit first (fast path)
    if size >= ConfigFile.textSizeLimit config
        then return False
        else do
            -- Check extension
            let ext = map toLower (takeExtension filePath)
            -- Binary extensions always win
            if ext `elem` binaryExtensions
                then return False
                else do
                    -- Read first 8KB and check for NULL bytes and UTF-8 validity
                    content <- BS.readFile filePath
                    let sample = BS.take 8192 content
                    -- Check for NULL bytes
                    if BS.elem 0 sample
                        then return False
                        else do
                            -- Check UTF-8 validity
                            case decodeUtf8' sample of
                                Left _ -> return False  -- Invalid UTF-8
                                Right _ -> return True   -- Valid text file

-- Main scan function
scanWorkingDir :: FilePath -> IO [FileEntry]
scanWorkingDir root = go root
  where
    go :: FilePath -> IO [FileEntry]
    go path = do
      isDir <- doesDirectoryExist path
      let rel = makeRelative root path

      -- ignore .rgit folder and .git (git metadata / pointer)
      if rel == ".rgit" || (".rgit" `isPrefixOf` rel)
          || rel == ".git" || (".git" `isPrefixOf` rel)
        then pure []
        else if isDir
          then do
            names <- listDirectory path
            let children = map (path </>) names
            childEntries <- concat <$> mapM go children
            let dirEntry = FileEntry { path = rel, kind = Directory }
            pure (dirEntry : childEntries)
        else do
          h <- hashFile path
          size <- getFileSize path
          isText <- classifyFile path (fromIntegral size)
          let fEntry = FileEntry
                { path = rel
                , kind = File { fHash = h, fSize = fromIntegral size, fIsText = isText }
                }
          pure [fEntry]

writeMetadataFiles :: FilePath -> [FileEntry] -> IO ()
writeMetadataFiles root entries = do
    let metaRoot = root </> ".rgit/index"
    createDirectoryIfMissing True metaRoot

    forM_ entries $ \entry ->
      case kind entry of
        Directory -> do
          let dirPath = metaRoot </> path entry
          createDirectoryIfMissing True dirPath

        File { fHash, fSize, fIsText } -> do
          let metaPath = metaRoot </> path entry
          createDirectoryIfMissing True (takeDirectory metaPath)
          
          if fIsText
            then do
              -- For text files, copy the actual content directly
              let actualPath = root </> path entry
              copyFileWithMetadata actualPath metaPath
            else do
              -- For binary files, write metadata (hash + size). Spec: raw hash value; atomic write.
              atomicWriteFileStr metaPath $
                serializeMetadata (MetaContent fHash fSize)

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

## Rgit/Types.hs

**Path:** `Rgit/Types.hs`

*Source file.*

```haskell
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE DeriveGeneric #-}
{-# LANGUAGE KindSignatures #-}

module Rgit.Types
  ( Path
  , HashAlgo(..)
  , Hash(..)
  , hashToText
  , EntryKind(..)
  , FileEntry(..)
  , syncHash
  , RgitEnv(..)
  , RgitM
  , runRgitM
  ) where

import Control.Monad.Trans.Reader (ReaderT, runReaderT)
import Data.Text (Text, unpack)
import GHC.Generics (Generic)
import Rgit.Remote (Remote)

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

data RgitEnv = RgitEnv
    { envCwd            :: FilePath
    , envLocalFiles     :: [FileEntry]
    , envRemote         :: Maybe Remote
    , envForce          :: Bool
    , envForceWithLease :: Bool
    }

type RgitM = ReaderT RgitEnv IO

runRgitM :: RgitEnv -> RgitM a -> IO a
runRgitM env action = runReaderT action env
```

---

## Rgit/Utils.hs

**Path:** `Rgit/Utils.hs`

*Source file.*

```haskell
{-# LANGUAGE OverloadedRecordDot #-}
{-# LANGUAGE ScopedTypeVariables #-}

module Rgit.Utils
  ( toPosix
  , isRgitPath
  , filterOutRgitPaths
  , atomicWriteFile
  , atomicWriteFileStr
  ) where

import Data.List (isPrefixOf, isInfixOf)
import Rgit.Types (FileEntry(..))
import System.Directory (renameFile, removeFile)
import System.IO (openTempFile, hClose)
import System.FilePath (takeDirectory)
import qualified Data.ByteString as BS
import Data.Text.Encoding (encodeUtf8)
import Data.Text (pack)
import Control.Exception (bracketOnError, catch, IOException, try)
import Control.Monad (void)
import Control.Concurrent (threadDelay)

-- | Convert Windows backslashes to forward slashes (e.g. for rclone paths).
toPosix :: FilePath -> FilePath
toPosix = map (\c -> if c == '\\' then '/' else c)

-- | True if the path is or is under .rgit (rgit metadata, not user content).
isRgitPath :: FilePath -> Bool
isRgitPath p = p == ".rgit" || ".rgit" `isPrefixOf` p || "/.rgit" `isInfixOf` p || "\\.rgit" `isInfixOf` p

-- | Remove .rgit paths from a list of file entries (e.g. remote file list).
filterOutRgitPaths :: [FileEntry] -> [FileEntry]
filterOutRgitPaths = filter (\e -> not (isRgitPath e.path))

-- | Write content to target atomically (temp file + rename). Spec ┬º Atomic Operations.
-- Uses bracketOnError to clean up temp file if an exception occurs.
-- On Windows, retries the rename a few times if the target is locked.
atomicWriteFile :: FilePath -> BS.ByteString -> IO ()
atomicWriteFile target content = do
  let tempDir = takeDirectory target
  bracketOnError
    (openTempFile tempDir ".rgit-tmp")
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
```

---

## Rgit/Verify.hs

**Path:** `Rgit/Verify.hs`

*Source file.*

```haskell
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE OverloadedRecordDot #-}

module Rgit.Verify
  ( verifyLocal
  , verifyRemote
  , VerifyIssue(..)
  , loadMetadataIndex
  , loadMetadataFromBundle
  ) where

import Data.Traversable (traverse)
import Rgit.Types (Hash(..), HashAlgo(..), Path, FileEntry(..), EntryKind(..), syncHash, hashToText)
import Rgit.Utils (filterOutRgitPaths)
import System.FilePath ((</>), makeRelative, normalise)
import System.Directory (doesFileExist, listDirectory, doesDirectoryExist, removeFile)
import Data.List (isPrefixOf)
import Data.Maybe (maybeToList)
import Data.Either (either)
import qualified Data.ByteString as BS
import qualified Data.Text as T
import qualified Internal.Git as Git
import Rgit.Internal.Metadata (MetaContent(..), parseMetadata, readMetadataOrComputeHash, hashFile)
import qualified Rgit.Remote.Scan as Remote.Scan
import qualified Rgit.Remote
import qualified Internal.Transport as Transport
import Internal.Config (fetchedBundle, fetchedBundlePath, rgitIndexPath, bundleCwdPath, fromCwdPath, BundleName)
import System.Process (readProcessWithExitCode)
import System.Exit (ExitCode(..))
import Data.Char (isSpace)
import System.IO (hPutStrLn, stderr)
import Control.Monad (when)
import Control.Monad (when, unless)
import qualified Data.Map as Map
import qualified Data.Set as Set
import System.IO (hPutStrLn, stderr)

-- | Result of comparing one file to metadata.
data VerifyIssue
  = HashMismatch Path String String Integer Integer  -- path, expectedHash, actualHash, expectedSize, actualSize
  | Missing Path                                      -- path (in metadata but no actual file)
  deriving (Show, Eq)

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
isGitPath path = ".git" `isPrefixOf` normalise path || normalise path == ".git"

-- | Load all metadata from .rgit/index: list of (relative path, expected hash, expected size).
-- Handles both text files (content stored directly) and binary files (metadata stored).
loadMetadataIndex :: FilePath -> IO [(Path, Hash 'MD5, Integer)]
loadMetadataIndex indexDir = do
  exists <- doesDirectoryExist indexDir
  if not exists
    then return []
    else do
      relPaths <- listFilesRecursive indexDir indexDir
      pairs <- mapM (\relPath -> do
        let fullPath = indexDir </> relPath
        mc <- readMetadataOrComputeHash fullPath
        return $ case mc of
          Just mc' -> [(relPath, metaHash mc', metaSize mc')]
          Nothing -> []
        ) relPaths
      return (concat pairs)

-- | Verify local working tree against metadata in .rgit/index.
-- Returns (number of files checked, list of issues).
verifyLocal :: FilePath -> IO (Int, [VerifyIssue])
verifyLocal cwd = do
  let indexDir = cwd </> ".rgit/index"
  meta <- loadMetadataIndex indexDir
  -- Filter out .git directory entries
  let filteredMeta = filter (\(relPath, _, _) -> not (isGitPath relPath)) meta
  issues <- mapM (checkOne cwd) filteredMeta
  return (length filteredMeta, concat issues)
  where
    checkOne root (relPath, expectedHash, expectedSize) = do
      let actualPath = root </> relPath
      exists <- doesFileExist actualPath
      if not exists
        then return [Missing relPath]
        else do
          actualHash <- hashFile actualPath
          actualSize <- fromIntegral . BS.length <$> BS.readFile actualPath
          if actualHash == expectedHash && actualSize == expectedSize
            then return []
            else return [HashMismatch relPath (T.unpack (hashToText expectedHash)) (T.unpack (hashToText actualHash)) expectedSize actualSize]

-- | Extract metadata from a bundle's HEAD commit.
-- First fetches the bundle into the repo, then reads metadata from refs/remotes/origin/main.
-- Returns list of (relative path, hash, size) for files under index/.
loadMetadataFromBundle :: BundleName -> IO [(Path, Hash 'MD5, Integer)]
loadMetadataFromBundle bundleName = do
  -- First, fetch the bundle into the repo so we can read from it
  fetchCode <- Git.fetchFromBundle bundleName
  if fetchCode /= ExitSuccess
    then return []
    else do
      -- Get the remote HEAD hash (now available as refs/remotes/origin/main)
      (code, out, _) <- readProcessWithExitCode "git"
        [ "-C", rgitIndexPath
        , "rev-parse"
        , "refs/remotes/origin/main"
        ] ""
      case filter (not . isSpace) out of
        [] -> return []
        headHash -> do
          -- List all files in the commit that are under index/
          (code2, out2, _) <- readProcessWithExitCode "git"
            [ "-C", rgitIndexPath
            , "ls-tree"
            , "-r"
            , "--name-only"
            , headHash
            , "--"
            , "index/"
            ] ""
          if code2 /= ExitSuccess
            then return []
            else do
              -- Filter to get only paths under index/ and remove the "index/" prefix
              let paths = filter (not . null) $ map (drop 6) $ filter ("index/" `isPrefixOf`) $ lines out2
              -- Read each metadata file from the commit
              metaList <- mapM (readMetadataFromCommit headHash) paths
              return $ concat metaList
  where
    -- Read a metadata file from a specific commit
    readMetadataFromCommit :: String -> FilePath -> IO [(Path, Hash 'MD5, Integer)]
    readMetadataFromCommit commitHash relPath = do
      let gitPath = "index/" ++ relPath
      (code, content, _) <- readProcessWithExitCode "git"
        [ "-C", rgitIndexPath
        , "show"
        , commitHash ++ ":" ++ gitPath
        ] ""
      if code /= ExitSuccess
        then return []
        else case parseMetadata content of
          Nothing -> return []
          Just (MetaContent { metaHash = h, metaSize = sz }) -> return [(relPath, h, sz)]

-- | Verify remote files match remote metadata.
-- Returns (number of files checked, list of issues).
verifyRemote :: FilePath -> Rgit.Remote.Remote -> IO (Int, [VerifyIssue])
verifyRemote cwd remote = do
  -- 1. Fetch the remote bundle if needed
  let fetchedPath = fromCwdPath (bundleCwdPath fetchedBundle)
  bundleExists <- doesFileExist fetchedPath
  when (not bundleExists) $ do
    let localDest = ".rgit/temp_remote.bundle"
    fetchResult <- Transport.copyFromRemoteDetailed remote ".rgit/rgit.bundle" localDest
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
      -- 2. Load metadata from the bundle
      remoteMeta <- loadMetadataFromBundle fetchedBundle
      
      -- 3. Fetch actual remote files
      Remote.Scan.fetchRemoteFiles remote >>= either
        (\_ -> hPutStrLn stderr "Error: Could not fetch remote file list." >> return (0, []))
        (\remoteFiles -> do
          let filteredRemoteFiles = filterOutRgitPaths remoteFiles
          
          -- 4. Build maps for comparison (both use MD5 hashes)
          let remoteFileMap = Map.fromList
                [ (normalise e.path, (h, e.kind))
                | e <- filteredRemoteFiles
                , h <- maybeToList (syncHash e.kind)
                ]
              remoteMetaMap = Map.fromList [(normalise p, (h, sz)) | (p, h, sz) <- remoteMeta]
          
          -- 5. Compare metadata with actual files
          issues <- traverse (checkRemoteFile remoteFileMap) remoteMeta
          
          -- 6. Check for files on remote that aren't in metadata
          let metaPaths = Set.fromList (Map.keys remoteMetaMap)
              filePaths = Set.fromList (Map.keys remoteFileMap)
              extraPaths = filePaths `Set.difference` metaPaths
              extraIssues = map (\p -> HashMismatch p "(not in metadata)" "(exists on remote)" 0 0) (Set.toList extraPaths)
          
          return (length remoteMeta, concat issues ++ extraIssues))
  where
    -- Check one file from metadata against remote (both use MD5)
    checkRemoteFile :: Map.Map FilePath (Hash 'MD5, EntryKind) -> (Path, Hash 'MD5, Integer) -> IO [VerifyIssue]
    checkRemoteFile remoteFileMap (relPath, expectedHash, expectedSize) = do
      let normalizedPath = normalise relPath
      case Map.lookup normalizedPath remoteFileMap of
        Nothing -> return [Missing relPath]
        Just (actualHash, File _ actualSize _) ->
          if actualHash == expectedHash && actualSize == expectedSize
            then return []
            else return [HashMismatch relPath (T.unpack (hashToText expectedHash)) (T.unpack (hashToText actualHash)) expectedSize actualSize]
        Just _ -> return []

-- Helper to safely remove a file
safeRemove :: FilePath -> IO ()
safeRemove path = do
  exists <- doesFileExist path
  when exists $ removeFile path
```

---

## rgit.cabal

**Path:** `rgit.cabal`

*Build configuration — package metadata and dependencies.*

```cabal
cabal-version:      2.4
name:               rgit
version:            0.1.0.0

executable rgit
    main-is:          Rgit.hs
    ghc-options:
        -- -Wall
        -- -Wunused-imports
        -- -Widentities
        -- -Wincomplete-patterns
    other-modules:    Rgit.Commands,
                      Rgit.Conflict,
                      Rgit.Device,
                      Rgit.DevicePrompt,
                      Rgit.Diff,
                      Rgit.Utils,
                      Rgit.Plan,
                      Rgit.Pipeline,
                      Rgit.Remote,
                      Rgit.Remote.Scan,
                      Rgit.Scan,
                      Rgit.Types,
                      Rgit.Verify,
                      Rgit.Fsck,
                      Bit,
                      Internal.Git,
                      Internal.Config,
                      Internal.ConfigFile,
                      Rgit.Internal.Metadata,
                      Internal.Transport
    build-depends:    base16-bytestring ^>=1.0.2.0,
                      base,
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
    build-depends:       base, directory, filepath, process
    default-language:    Haskell2010

test-suite cli
    type:                exitcode-stdio-1.0
    main-is:             RunCliTests.hs
    hs-source-dirs:      test
    build-depends:       base, directory, filepath, process
    default-language:    Haskell2010

test-suite device-prompt
    type:                exitcode-stdio-1.0
    main-is:             DevicePromptTests.hs
    hs-source-dirs:      test, .
    other-modules:       Rgit.DevicePrompt
    build-depends:       base,
                        directory,
                        tasty,
                        tasty-hunit
    default-language:    Haskell2010

test-suite pipeline
    type:                exitcode-stdio-1.0
    main-is:             PipelineSpec.hs
    hs-source-dirs:      test, .
    other-modules:       Rgit.Types,
                         Rgit.Diff,
                         Rgit.Plan,
                         Rgit.Pipeline,
                         Rgit.Utils,
                         Rgit.Internal.Metadata,
                         Rgit.Device,
                         Rgit.Remote,
                         Internal.Git,
                         Internal.Config,
                         Internal.ConfigFile
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
    build-depends:       base, process, directory, filepath
    build-tool-depends:  rgit:generate-literate
    default-language:    Haskell2010
```

---

## scripts/GenerateLiterate.hs

**Path:** `scripts/GenerateLiterate.hs`

*Source file.*

```haskell
-- | Generate three literate-programming Markdown files:
--   * rgit-source-literate.md Γאפ rgit.cabal + all .hs source files (excluding test/)
--   * rgit-tests-literate.md  Γאפ everything under test/
--   * rgit-docs-literate.md   Γאפ all .md files under docs/
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

-- | Directories to skip.
excludedDirs :: [String]
excludedDirs = [".git", ".rgit", "dist-newstyle", "dist", ".stack-work", ".vscode", ".history"]

-- | Ascend until we find rgit.cabal.
findRepoRoot :: FilePath -> IO FilePath
findRepoRoot dir = do
  exists <- doesFileExist (dir </> "rgit.cabal")
  if exists
    then return dir
    else let parent = takeDirectory dir
         in  if parent == dir
               then fail "Could not find repo root (rgit.cabal)"
               else findRepoRoot parent

-- | Subdirectories of test/ to skip (generated at runtime).
excludedTestDirs :: [String]
excludedTestDirs = ["work", "work_a", "work_b"]

-- | Files inside test/ to skip.
excludedTestFiles :: [String]
excludedTestFiles = [".rgit-store"]

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
  | rel == "rgit.cabal"                = "*Build configuration Γאפ package metadata and dependencies.*"
  | "Internal/" `isPrefixOf` rel       = "*Internal module Γאפ implementation details.*"
  | "Rgit/Internal/" `isPrefixOf` rel  = "*Internal module Γאפ implementation details.*"
  | rel == "Rgit.hs"                   = "*Entry point Γאפ main executable.*"
  | "Rgit/" `isPrefixOf` rel           = "*Core module Γאפ application logic.*"
  | "test/cli/" `isPrefixOf` rel       = "*CLI test Γאפ shell-based integration test.*"
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

  let sourceFiles = sort $ "rgit.cabal" : hsFiles
      testSorted  = sort testFiles
      docsSorted  = sort docsFiles

  let litDir = root </> "literate-output"
  createDirectoryIfMissing True litDir

  -- Source files
  let sourcePath = litDir </> "rgit-source-literate.md"
  writeDocument sourcePath
    "rgit Γאפ Literate Programming Document"
    "This document contains all Haskell source files and the cabal\n\
    \file for the rgit project, presented in literate-programming style."
    root sourceFiles
  putStrLn $ "Wrote " ++ sourcePath ++ " (" ++ show (length sourceFiles) ++ " files)"

  -- Test files
  let testsPath = litDir </> "rgit-tests-literate.md"
  writeDocument testsPath
    "rgit Γאפ Tests (Literate Programming Document)"
    "This document contains all test files for the rgit project:\n\
    \Haskell test modules, shell-based integration tests, and test infrastructure."
    root testSorted
  putStrLn $ "Wrote " ++ testsPath ++ " (" ++ show (length testSorted) ++ " files)"

  -- Docs files
  let docsPath = litDir </> "rgit-docs-literate.md"
  writeDocument docsPath
    "rgit Γאפ Documentation (Literate Programming Document)"
    "This document contains all documentation files for the rgit project:\n\
    \specifications, refactoring plans, and design documents."
    root docsSorted
  putStrLn $ "Wrote " ++ docsPath ++ " (" ++ show (length docsSorted) ++ " files)"

  -- Git: stage, commit, push (gracefully handle failures)
  putStrLn "Committing literate output..."
  addResult <- try (callProcess "git" ["-C", litDir, "add",
    "rgit-source-literate.md",
    "rgit-tests-literate.md",
    "rgit-docs-literate.md"]) :: IO (Either SomeException ())
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

      result <- try (readFile full) :: IO (Either SomeException String)
      case result of
        Right content -> do
          hPutStr h content
          if not (null content) && last content /= '\n'
            then hPutStrLn h ""
            else return ()
        Left err ->
          hPutStrLn h $ "-- Error reading file: " ++ show err

      write "```"
      write ""
      write "---"
      write ""
```

---

