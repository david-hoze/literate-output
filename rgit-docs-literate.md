# rgit — Documentation (Literate Programming Document)

This document contains all documentation files for the rgit project:
specifications, refactoring plans, and design documents.

---

## docs/SPEC_CONFORMANCE_AND_REFACTOR.md

**Path:** `docs/SPEC_CONFORMANCE_AND_REFACTOR.md`

*Source file.*

```markdown
# Spec Conformance and Refactoring Report

## Spec Conformance Issues

### 1. **Metadata file format** (spec ┬º Metadata File Format)

- **Spec**: Metadata files contain ONLY:
  ```
  hash: sha256:a1b2c3d4e5f6...
  size: 1048576
  ```
  Two fields; hash is the raw value (no type wrapper).

- **Code**: `Rgit/Scan.hs` writes `"hash: " <> show fHash`, which produces `hash: Hash "md5:hex..."` (Haskell `Show` adds the type name and quotes).

- **Fix**: Write the raw hash value, e.g. `"hash: " <> (T.unpack . hashToText $ fHash)` and `"size: " <> show fSize`. Parsers in `Internal/Metadata.hs` and `Rgit/Scan.hs` must accept unquoted hash values (and remain backward-compatible with quoted if needed).

### 2. **Hash algorithm** (spec ┬º Metadata)

- **Spec**: "hash Γאפ SHA256 of file content".

- **Code**: MD5 is used everywhere (`Rgit/Scan.hs`, `Rgit/Verify.hs`, `Rgit/Remote/Scan.hs`) for both metadata and rclone comparison.

- **Note**: Switching to SHA256 in metadata would require adding SHA256 hashing and possibly keeping MD5 only for rclone sync (rclone uses MD5). Defer or document as intentional deviation.

### 3. **rgit diff output** (spec ┬º Output Format)

- **Spec**: Diff hunks should show:
  ```
  -hash=sha256:abc123...
  -size=1048576
  +hash=sha256:def456...
  +size=1049000
  ```
  (equals signs in the diff body.)

- **Code**: `rgit diff` delegates to `Git.diff args`. The index contains metadata files with `hash:` and `size:` (colons), so git diff shows colons.

- **Fix**: Either implement a custom diff formatter that reads metadata and prints lines with `hash=`/`size=`, or keep delegating and accept the current format.

### 4. **Merge conflict display** (spec ┬º Resolution Option 3)

- **Spec**: For each conflict:
  ```
  Local:           sha256:abc123... (1048576 bytes)
  Remote actual:   sha256:xyz789... (1049000 bytes)
  Remote metadata: sha256:abc123... (1048576 bytes)
  ```

- **Code**: `Rgit/Commands.hs` (merge conflict resolution) uses:
  `"Local:  hash=" ++ hashO ++ " size=" ++ sizeO` (and similar for Remote).

- **Fix**: Use "(N bytes)" format and align wording with spec: "Local:", "Remote actual:", "Remote metadata:" where applicable.

### 5. **Atomic file writes** (spec ┬º Atomic Operations)

- **Spec**: All file write operations must be atomic (write to temp file, then rename).

- **Code**: Plain `writeFile` / `LBS.writeFile` is used in:
  - `Rgit/Commands.hs`: init (ignore, config, .gitattributes), conflict files (METADATA_LOCAL, METADATA_REMOTE, etc.)
  - `Rgit/Scan.hs`: metadata files in `writeMetadataFiles`

- **Fix**: Introduce `atomicWriteFile` (and optionally `atomicDirectoryReplace`) and use for metadata and other critical writes.

### 6. **rclone delete** (spec ┬º Rclone Commands Used)

- **Spec**: "rclone delete remote:path" for delete.

- **Code**: `Rgit/Commands.hs` uses `run "deletefile" [remoteRoot ++ toPosix path]`. rcloneΓאשs `deletefile` removes a single file; `delete` can target multiple. For a single path, `deletefile` is appropriate; confirm naming vs spec if desired.

### 7. **Transaction log** (spec ┬º Transaction Log)

- **Spec**: `.rgit/transaction.log` for resumable push/pull.

- **Code**: Not implemented (spec also lists this under "Not Yet Implemented"). No change for now.

---

## Refactoring Opportunities

### 1. **Atomic writes**

- Add a small module (e.g. `Internal.Atomic` or `Rgit.Utils`) with `atomicWriteFile :: FilePath -> ByteString -> IO ()` (and optionally lazy variant / directory replace).
- Use it in `Rgit/Scan.hs` for metadata and in `Rgit/Commands.hs` for init and conflict files.

### 2. **Duplicate metadata parsing**

- **Internal.Metadata**: `readMetadataFile` returns `(String, Integer)`; `parseHashLine` / `parseSizeLine` expect quoted hash.
- **Rgit.Scan**: `readMetadataFile` returns `(Hash, Integer)` with its own parsing (hash/size lines + `parseHash` / `readInteger`).
- **Rgit.Verify**: `parseMetadataFile` returns `(Hash, Integer)` with local `parseHashLine` / `parseSizeLine`.
- **Rgit.Commands**: `parseMetadataForDisplay` returns `(String, String)` for display.

- **Refactor**: Centralize format in one place (e.g. Internal.Metadata): single parser that supports both quoted and unquoted hash; expose helpers for Hash and for display strings; have Scan/Verify/Commands call into that.

### 3. **Duplicate hash computation**

- **Rgit.Scan**: `hashFile`, `hashFileFromBytes` (MD5).
- **Rgit.Verify**: `fileHash` (MD5).

- **Refactor**: Extract shared `fileHash :: FilePath -> IO Hash` (and optionally `hashBytes :: ByteString -> Hash`) into `Internal.Hash` or `Rgit.Utils` and reuse in Scan and Verify.

### 4. **Commands.hs size and structure**

- **Rgit/Commands.hs** is very long (~1193 lines) and mixes init, add, commit, merge, push, pull, verify, etc.

- **Refactor**: Split by command or domain (e.g. `Rgit.Commands.Init`, `Rgit.Commands.Merge`, `Rgit.Commands.Remote`) and keep `Rgit.Commands` as a thin dispatcher.

### 5. **Verify display and hash format**

- **Rgit.Commands** uses "sha256:" prefix in verify/merge output while hashes are actually MD5. Either:
  - Store and show SHA256 in metadata and verify, or
  - Use "md5:" in display to match reality.

---

## Summary

| Item                         | Severity   | Action                                      |
|------------------------------|------------|---------------------------------------------|
| Metadata write format        | High       | Write raw hash; update parsers              |
| Merge conflict display       | Medium     | Use "(N bytes)" and spec wording            |
| Atomic writes                | High       | Add atomicWriteFile; use for metadata/init  |
| Metadata parser (unquoted)   | Follow-up  | Support unquoted hash in Internal.Metadata  |
| Hash algorithm (SHA256)      | Low/Defer  | Document or add SHA256 later                 |
| rgit diff format             | Medium     | Custom formatter or accept current          |
| Duplicate parsing/hash       | Refactor   | Centralize parsing and hash computation     |
| Commands.hs split            | Refactor   | Split into submodules by command/domain     |
```

---

## docs/refactor-push-remote-into-transport.md

**Path:** `docs/refactor-push-remote-into-transport.md`

*Source file.*

```markdown
# Refactor: Push `Remote` into Transport Γאפ `remoteUrl` never leaves Transport.hs

## Principle

`remoteUrl` should only be accessed inside `Internal/Transport.hs`. Every other module works with the `Remote` type and relative paths. No module outside Transport should ever write `remoteUrl remote`.

The one exception is **display/logging** Γאפ Bit.hs may show the URL to the user in messages like `"Inspecting remote 'origin' (gdrive:Projects/foo)"`. For that, add a `displayRemote` helper to `Rgit/Remote.hs` so Bit.hs still never accesses `remoteUrl` directly.

## Step-by-step changes

### 1. Make `remoteUrl` field internal

In `Rgit/Remote.hs`, stop exporting the `Remote` constructor with its fields directly. Instead export a smart constructor and a display function:

```haskell
module Rgit.Remote
  ( Remote          -- export type but NOT constructor/fields
  , mkRemote        -- smart constructor
  , remoteName      -- only the name is public
  , displayRemote   -- for user-facing messages: "origin (gdrive:Projects/foo)"
  , resolveRemote
  , getDefaultRemote
  ) where

data Remote = Remote
  { _remoteName :: String
  , _remoteUrl  :: String
  } deriving (Show, Eq)

remoteName :: Remote -> String
remoteName = _remoteName

-- | For user-facing display only. Never use this to construct paths.
displayRemote :: Remote -> String
displayRemote r = _remoteName r ++ " (" ++ _remoteUrl r ++ ")"

mkRemote :: String -> String -> Remote
mkRemote = Remote
```

Then add an **internal** export for Transport only. The cleanest way: create a small internal module `Rgit/Remote/Internal.hs` that exports the url accessor, and only Transport imports it:

```haskell
-- Rgit/Remote/Internal.hs
module Rgit.Remote.Internal (remoteUrl) where
import Rgit.Remote (Remote(..))  -- or re-export from where Remote is defined

remoteUrl :: Remote -> String
remoteUrl = _remoteUrl
```

**Alternatively** (simpler, less ceremony): just keep `remoteUrl` exported from `Rgit.Remote` but establish the convention via code review. The prompt below works either way Γאפ the mechanical changes are the same. Pick whichever approach fits your style. If you want the hard enforcement, create `Rgit/Remote/Internal.hs`. If convention is fine, skip it and just remove all `remoteUrl` calls from Bit.hs, Remote.Scan, and Verify.

### 2. Update `Internal/Transport.hs`

Transport imports `Remote` (and `remoteUrl` from the internal module or directly) and provides functions that take `Remote` instead of raw URL strings.

**Add import:**
```haskell
import Rgit.Remote (Remote)
import Rgit.Remote.Internal (remoteUrl)  -- or just import remoteUrl from Rgit.Remote
```

**Add internal path helper (replaces `ensureRemoteRoot` from Utils):**
```haskell
-- | Build a full remote path from Remote + relative path.
-- Handles trailing-slash normalization internally.
remoteFilePath :: Remote -> FilePath -> String
remoteFilePath remote relPath =
    let base = remoteUrl remote
        -- Ensure exactly one separator between base and relative path
        base' = if not (null base) && last base == '/' then init base else base
    in base' ++ "/" ++ relPath
```

**Change every public function signature.** The pattern is: where a function currently takes a `String` that represents a remote URL or remote path, it now takes `Remote` (and optionally a relative path). Here are all the functions:

```haskell
-- BEFORE                                           -- AFTER
copyToRemote :: FilePath -> String -> IO ExitCode    -> copyToRemote :: FilePath -> Remote -> FilePath -> IO ExitCode
copyFromRemote :: String -> FilePath -> IO ExitCode  -> copyFromRemote :: Remote -> FilePath -> FilePath -> IO ExitCode
copyFromRemoteDetailed :: String -> FilePath -> IO CopyResult -> copyFromRemoteDetailed :: Remote -> FilePath -> FilePath -> IO CopyResult
moveRemote :: String -> String -> IO ExitCode        -> moveRemote :: Remote -> FilePath -> FilePath -> IO ExitCode
deleteRemote :: String -> IO ExitCode                -> deleteRemote :: Remote -> FilePath -> IO ExitCode
purgeRemote :: String -> IO ExitCode                 -> purgeRemote :: Remote -> IO ExitCode
mkdirRemote :: String -> IO ExitCode                 -> mkdirRemote :: Remote -> FilePath -> IO ExitCode
listRemoteJson :: String -> Int -> IO (...)           -> listRemoteJson :: Remote -> Int -> IO (...)
listRemoteJsonWithHash :: String -> IO (...)          -> listRemoteJsonWithHash :: Remote -> IO (...)
listRemoteItems :: String -> Int -> IO (...)          -> listRemoteItems :: Remote -> Int -> IO (...)
checkRemote :: FilePath -> String -> IO CheckResult   -> checkRemote :: FilePath -> Remote -> IO CheckResult
```

**Implementation pattern Γאפ each function constructs the full rclone path internally:**

```haskell
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

-- | List remote items (at remote root, parsed)
listRemoteItems :: Remote -> Int -> IO (Either String [TransportItem])
listRemoteItems remote maxDepth = do
    (code, out, err) <- listRemoteJson remote maxDepth
    -- ... same parsing logic, just uses remote instead of string ...

-- | List remote recursively with hashes
listRemoteJsonWithHash :: Remote -> IO (ExitCode, String, String)
listRemoteJsonWithHash remote =
    readProcessWithExitCode "rclone" ["lsjson", remoteUrl remote, "--hash", "--recursive"] ""

-- | Check local against remote
checkRemote :: FilePath -> Remote -> IO CheckResult
checkRemote localPath remote = do
    let args = [ "check", localPath, remoteUrl remote
               , "--combined", "-", "--exclude", ".rgit/**" ]
    (code, out, err) <- readProcessWithExitCode "rclone" args ""
    -- ... same parsing ...
```

### 3. Update `Rgit/Remote/Scan.hs`

**Change `fetchRemoteFiles` to take `Remote`:**

```haskell
import Rgit.Remote (Remote)
-- Do NOT import remoteUrl here Γאפ Remote.Scan calls Transport, not rclone directly

fetchRemoteFiles :: Remote -> IO (Either RemoteError [FileEntry])
fetchRemoteFiles remote = do
    (code, raw, _err) <- Transport.listRemoteJsonWithHash remote
    case code of
        ExitSuccess -> pure $ maybe
            (Left (DecodeFailed "Invalid rclone JSON output"))
            (Right . map rcloneFileToFileEntry . filter (not . rfIsDir))
            (decode (LBSC.pack raw) :: Maybe [RcloneFile])
        _ -> pure (Left RcloneFailed)
```

### 4. Update `Rgit/Verify.hs`

Any function that currently takes a URL string for remote verification should take `Remote` instead. Find `verifyRemote` and change it:

```haskell
-- BEFORE:
verifyRemote :: FilePath -> String -> IO (Int, [VerifyIssue])

-- AFTER:
import Rgit.Remote (Remote)
verifyRemote :: FilePath -> Remote -> IO (Int, [VerifyIssue])
```

Inside `verifyRemote`, where it calls `Transport.checkRemote cwd url`, change to `Transport.checkRemote cwd remote`. If it calls other Transport functions with the URL, change those too.

### 5. Update `Bit.hs` Γאפ the big change

**Remove `import Rgit.Remote (remoteUrl)` (or stop using it).** Bit.hs should only import:
```haskell
import Rgit.Remote (Remote, remoteName, displayRemote)
```

**Update `executeCommand` to take `Remote` instead of `remoteRoot :: String`:**

```haskell
-- BEFORE:
executeCommand :: FilePath -> String -> RcloneAction -> IO ()
executeCommand localRoot remoteRoot action = case action of
    Copy src dest -> do
        let localPath = toPosix (localRoot </> src)
            remotePath = remoteRoot ++ toPosix dest
        void $ Transport.copyToRemote localPath remotePath
    Move src dest ->
        void $ Transport.moveRemote (remoteRoot ++ toPosix src) (remoteRoot ++ toPosix dest)
    Delete path ->
        void $ Transport.deleteRemote (remoteRoot ++ toPosix path)
    Swap _ _ _ -> return ()

-- AFTER:
executeCommand :: FilePath -> Remote -> RcloneAction -> IO ()
executeCommand localRoot remote action = case action of
    Copy src dest -> do
        let localPath = toPosix (localRoot </> src)
        void $ Transport.copyToRemote localPath remote (toPosix dest)
    Move src dest ->
        void $ Transport.moveRemote remote (toPosix src) (toPosix dest)
    Delete path ->
        void $ Transport.deleteRemote remote (toPosix path)
    Swap _ _ _ -> return ()
```

**Update `executePullCommand` similarly:**

```haskell
-- BEFORE:
executePullCommand :: FilePath -> String -> RcloneAction -> IO ()
executePullCommand localRoot remoteRoot action = case action of
    Copy src dest -> do
        fromIndex <- isTextFileInIndex localRoot dest
        if fromIndex
        then copyFromIndexToWorkTree localRoot dest
        else do
            let remotePath = remoteRoot ++ toPosix dest
                localPath = toPosix (localRoot </> dest)
            createDirectoryIfMissing True (takeDirectory (localRoot </> dest))
            void $ Transport.copyFromRemote remotePath localPath
    ...

-- AFTER:
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
```

**Update `syncRemoteFiles`:**

```haskell
-- BEFORE:
syncRemoteFiles :: RgitM ()
syncRemoteFiles = withRemote $ \remote -> do
    cwd <- asks envCwd
    localFiles <- asks envLocalFiles
    remoteResult <- liftIO $ Remote.Scan.fetchRemoteFiles (remoteUrl remote)
    either
        (\_ -> liftIO $ hPutStrLn stderr "Error: Failed to fetch remote file list.")
        (\remoteFiles -> do
            let actions = Pipeline.pushSyncFiles localFiles remoteFiles
                remoteRoot = ensureRemoteRoot (remoteUrl remote)
            liftIO $ putStrLn "--- Pushing Changes to Remote ---"
            if null actions
                then liftIO $ putStrLn "Remote is already up to date."
                else mapM_ (\a -> liftIO $ executeCommand cwd remoteRoot a) actions)
        remoteResult

-- AFTER:
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
```

**Update `syncRemoteFilesToLocal` Γאפ same pattern.**

**Update `classifyRemoteState`:**
```haskell
-- BEFORE:
classifyRemoteState :: Remote -> IO RemoteState
classifyRemoteState remote = do
    result <- Transport.listRemoteItems (remoteUrl remote) 1
    ...

-- AFTER (no change needed if Transport.listRemoteItems already takes Remote):
classifyRemoteState :: Remote -> IO RemoteState
classifyRemoteState remote = do
    result <- Transport.listRemoteItems remote 1
    ...
```

**Update `fetchBundle`:**
```haskell
-- BEFORE:
fetchBundle :: Remote -> IO FetchResult
fetchBundle remote = do
    let remotePath = remoteUrl remote ++ "/.rgit/rgit.bundle"
    let localDest = ".rgit/temp_remote.bundle"
    result <- Transport.copyFromRemoteDetailed remotePath localDest
    ...

-- AFTER:
fetchBundle :: Remote -> IO FetchResult
fetchBundle remote = do
    let localDest = ".rgit/temp_remote.bundle"
    result <- Transport.copyFromRemoteDetailed remote ".rgit/rgit.bundle" localDest
    ...
```

**Update `pushBundle`:**
```haskell
-- BEFORE:
pushBundle :: Remote -> IO ()
pushBundle remote = do
    let tempBundle = "rgit.bundle"
    let remotePath = remoteUrl remote ++ "/.rgit/rgit.bundle"
    bracket
        (Git.createBundle tempBundle)
        (\_ -> cleanupTemp $ Git.bundlePath tempBundle)
        (\gCode -> do
            if gCode /= ExitSuccess
                then hPutStrLn stderr "Error creating bundle"
                else uploadToRemote (Git.bundlePath tempBundle) remotePath
        )

-- AFTER:
pushBundle :: Remote -> IO ()
pushBundle remote = do
    let tempBundle = "rgit.bundle"
    bracket
        (Git.createBundle tempBundle)
        (\_ -> cleanupTemp $ Git.bundlePath tempBundle)
        (\gCode -> do
            if gCode /= ExitSuccess
                then hPutStrLn stderr "Error creating bundle"
                else uploadToRemote (Git.bundlePath tempBundle) remote
        )

-- And uploadToRemote:
-- BEFORE:
uploadToRemote :: FilePath -> String -> IO ()
uploadToRemote src dest = do
    putStrLn "Uploading bundle to remote..."
    rCode <- Transport.copyToRemote src dest
    ...

-- AFTER:
uploadToRemote :: FilePath -> Remote -> IO ()
uploadToRemote src remote = do
    putStrLn "Uploading bundle to remote..."
    rCode <- Transport.copyToRemote src remote ".rgit/rgit.bundle"
    ...
```

**Update `verify` (remote case):**
```haskell
-- BEFORE:
(fileCount, issues) <- liftIO $ Verify.verifyRemote cwd (remoteUrl remote)

-- AFTER:
(fileCount, issues) <- liftIO $ Verify.verifyRemote cwd remote
```

**Update `saveFetchedBundle`:**
```haskell
-- BEFORE (wherever it calls Git.setupRemote with the URL):
_ <- Git.setupRemote (remoteUrl remote)

-- AFTER:
_ <- Git.setupRemote (remoteUrl remote)
-- NOTE: Git.setupRemote needs the actual URL string for git config.
-- This is the ONE place outside Transport where the URL is needed Γאפ because
-- we're configuring git's remote, not calling rclone.
-- Options:
--   (a) Import remoteUrl from Internal module just for this one call
--   (b) Add a helper: Git.setupRemoteFromRemote :: Remote -> IO ExitCode
--   (c) Accept this as a legitimate boundary crossing
-- Recommend (b): add to Git.hs:
--     setupRemoteFromRemote :: Remote -> IO ExitCode
--     setupRemoteFromRemote remote = addRemote "origin" (remoteUrl remote)
-- But Git.hs shouldn't import Remote either...
-- Simplest: keep Git.setupRemote :: String -> IO ExitCode, and in Bit.hs use displayRemote
-- or add a dedicated accessor. Actually, the cleanest solution:
-- In Rgit/Remote.hs, export a function specifically for git config:
--     remoteUrlForGit :: Remote -> String
--     remoteUrlForGit = _remoteUrl
-- This makes the intent explicit. Or just keep remoteUrl exported and
-- accept that Bit.hs uses it in exactly this one Git-config spot.
```

**Update `remoteCheck`:**
```haskell
-- BEFORE:
res <- liftIO $ try @IOException (Transport.checkRemote cwd (ensureRemoteRoot (remoteUrl remote)))

-- AFTER:
res <- liftIO $ try @IOException (Transport.checkRemote cwd remote)
```

**Update display messages to use `displayRemote`:**
```haskell
-- BEFORE:
liftIO $ putStrLn $ "Inspecting remote '" ++ remoteName remote ++ "' (" ++ remoteUrl remote ++ ")"

-- AFTER:
liftIO $ putStrLn $ "Inspecting remote: " ++ displayRemote remote
```

### 6. Remove `ensureRemoteRoot` from `Rgit/Utils.hs`

This function adds a trailing slash to URLs. It's no longer needed Γאפ Transport handles path joining internally via `remoteFilePath`. Delete it from Utils.hs and remove all references.

### 7. Update `rgit.cabal`

If you created `Rgit/Remote/Internal.hs`, add it to `other-modules`.

## Verification

```bash
cabal clean && cabal build rgit
cabal test device-prompt

# remoteUrl should NOT appear in Bit.hs (except possibly one Git.setupRemote call):
grep -n "remoteUrl" Bit.hs
# Should return 0 results (or exactly 1 for the Git config edge case)

# remoteUrl should NOT appear in Remote/Scan.hs:
grep -n "remoteUrl" Rgit/Remote/Scan.hs
# Should return 0 results

# remoteUrl should NOT appear in Verify.hs:
grep -n "remoteUrl" Rgit/Verify.hs
# Should return 0 results

# ensureRemoteRoot should be gone:
grep -rn "ensureRemoteRoot" Bit.hs Rgit/
# Should return 0 results

# Only Transport.hs should reference remoteUrl:
grep -rn "remoteUrl\|_remoteUrl" --include="*.hs" . | grep -v "Internal/Transport.hs" | grep -v "Rgit/Remote"
# Should return 0 results (or only the Git config edge case in Bit.hs)
```

## Edge case: `Git.setupRemote`

`Git.setupRemote` configures git's internal remote URL (`git remote add origin <url>`). This genuinely needs the URL string, and it's not an rclone operation Γאפ it's a git operation. The cleanest solution is to export a `remoteUrlForGit :: Remote -> String` from `Rgit/Remote.hs` with a clear name that signals "this is for git config, not for transport." Or just accept that `Bit.hs` calls `remoteUrl` in exactly this one spot and document why.
```

---

## docs/rgit-refactoring-plan.md

**Path:** `docs/rgit-refactoring-plan.md`

*Source file.*

```markdown
# rgit Path C Refactoring Plan Γאפ Cursor Instructions

## Overview

This is a 7-step refactoring plan. Each step compiles and passes tests independently. Execute them **in order**. Run `cabal clean && cabal build rgit && cabal test device-prompt` after each step.

**Goal:** Clean layer boundaries between Bit.hs (business logic), Transport.hs (dumb rclone wrapper), and Git.hs (dumb git wrapper). Remove the premature free monad effect system. Keep `ReaderT RgitEnv IO` (it's useful). Introduce a `Remote` type so Bit.hs never handles raw URL strings.

**Current module dependency (broken):**
```
Commands.hs Γזע Bit.hs Γזע Transport.hs Γזע rclone
                 Γזף           Γזף
              Git.hs      rclone (direct)  Γזנ WRONG
                 Γזף
              rclone (direct)              Γזנ WRONG
```

**Target module dependency (clean):**
```
Commands.hs Γזע Bit.hs Γזע Transport.hs Γזע rclone (only here!)
                 Γזף
              Git.hs Γזע git (only here!)
```

**Layer contract:**
- **Transport.hs** Γאפ Dumb. Knows how to `copyTo`, `moveTo`, `deleteFile`, `listJson`, `check`. Takes source/dest strings. Does NOT know about `.rgit/`, bundles, `RemoteState`, or `FetchResult`.
- **Git.hs** Γאפ Dumb. Knows how to run git commands. Takes args. Does NOT interpret results in domain terms.
- **Bit.hs** Γאפ Smart. All business logic. Knows about remotes, bundles, `.rgit/` layout, sync strategy. Calls Transport and Git, never calls `readProcessWithExitCode` directly.
- **Commands.hs** Γאפ Entry point. Parses CLI, resolves the remote, builds `RgitEnv`, dispatches to Bit.

---

# Step 1: Introduce the `Remote` type

**Files:** Create `Rgit/Remote.hs`, modify `Rgit/Types.hs`

**What:** Define a proper `Remote` type that replaces bare `String` URLs throughout the codebase. This is a new module with zero breakage Γאפ nothing uses it yet.

## Create `Rgit/Remote.hs`

```haskell
module Rgit.Remote
  ( Remote(..)
  , remoteUrl
  , resolveRemote
  , getDefaultRemote
  ) where

import qualified Internal.Git as Git
import qualified Rgit.Device as Device
import Internal.Config (rgitRemotesDir)
import System.FilePath ((</>))
import System.Directory (doesFileExist)
import Data.Maybe (fromMaybe)

-- | A resolved remote. Bit.hs works with this; only Transport sees the url.
data Remote = Remote
  { remoteName :: String    -- "origin", "backup", "nas", etc.
  , remoteUrl  :: String    -- Resolved URL/path for Transport (e.g. "gdrive:Projects/foo", "/mnt/usb/backup")
  } deriving (Show, Eq)

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
                Device.Resolved url -> return (Just (Remote name url))
                Device.NotConnected _ -> return Nothing
        Nothing -> do
            -- Fall back to git remote URL
            mUrl <- Git.getRemoteUrl name
            case mUrl of
                Just url | not (null url) -> return (Just (Remote name url))
                _ -> return Nothing

-- | Get the default remote for push/pull/fetch.
-- Checks branch tracking config, falls back to "origin".
getDefaultRemote :: FilePath -> IO (Maybe Remote)
getDefaultRemote cwd = do
    name <- Git.getTrackedRemoteName  -- defaults to "origin" if not configured
    resolveRemote cwd name
```

## Update `rgit.cabal`

Add `Rgit.Remote` to `other-modules` (or `exposed-modules`).

## Verification

```bash
cabal clean && cabal build rgit
# Just compiles Γאפ nothing calls it yet
```

## Guardrails
- Do NOT change any existing code in this step. Only add the new module.
- `resolveRemote` consolidates the logic currently scattered across `resolveRemoteByName` in Bit.hs, `getRemoteTarget` in Bit.hs, and raw `Git.getRemoteUrl` calls. But we don't replace those yet.

---

# Step 2: Remove the Free Monad Effect System

**Files:** Delete `Rgit/Effect.hs`, delete `Rgit/Effect/IO.hs`, delete `Rgit/Effect/DryRun.hs`, modify `Rgit/Types.hs`, modify `Bit.hs`, modify `Rgit/Commands.hs`, modify `rgit.cabal`

**What:** The effect system (`Rgit = Free RgitF`) is premature Γאפ zero pure tests exist, zero dry-run usage, and there are ~15 `liftIOE` escape hatches. Remove it. `RgitM` becomes `ReaderT RgitEnv IO`. Functions that were `Rgit a` become `IO a`.

This is the largest single step. Take it carefully.

## 2a. Update `Rgit/Types.hs`

**Change:**
```haskell
-- BEFORE:
import Rgit.Effect (Rgit)
type RgitM = ReaderT RgitEnv Rgit
runRgitM :: RgitEnv -> RgitM a -> Rgit a
runRgitM env action = runReaderT action env

-- AFTER:
type RgitM = ReaderT RgitEnv IO
runRgitM :: RgitEnv -> RgitM a -> IO a
runRgitM env action = runReaderT action env
```

Remove the `import Rgit.Effect (Rgit)` line. Remove `Rgit` from exports.

## 2b. Update `Rgit/Commands.hs`

**Change:**
```haskell
-- BEFORE:
import Rgit.Effect.IO (runIO)
-- ...
["push"] -> runIO $ runRgitM env Bit.push

-- AFTER:
-- Remove import of Rgit.Effect.IO
-- ...
["push"] -> runRgitM env Bit.push
```

Remove **every** `runIO $` wrapper. `runRgitM` now returns `IO a` directly.

For the git passthrough commands that were `Rgit ExitCode` (like `Bit.add`, `Bit.commit`), they become `IO ExitCode` and are called directly:
```haskell
-- BEFORE:
("add":rest) -> void $ runIO $ Bit.add rest

-- AFTER:
("add":rest) -> void $ Bit.add rest
```

## 2c. Convert `Bit.hs` Γאפ the big change

**Conversion rules (apply mechanically throughout the entire file):**

| Before (effect system) | After (plain IO) |
|---|---|
| `someFunc :: Rgit a` | `someFunc :: IO a` |
| `someFunc :: RgitM a` | `someFunc :: RgitM a` (unchanged Γאפ RgitM is now `ReaderT RgitEnv IO`) |
| `lift $ tell msg` | `liftIO $ putStrLn msg` |
| `lift $ tellErr msg` | `liftIO $ hPutStrLn stderr msg` |
| `lift $ liftIOE $ someIOAction` | `liftIO $ someIOAction` |
| `lift $ liftIOE $ Git.someFunc` | `liftIO $ Git.someFunc` |
| `lift $ liftIOE $ Transport.someFunc` | `liftIO $ Transport.someFunc` |
| `lift $ fileExistsE path` | `liftIO $ doesFileExist path` |
| `lift $ readFileE path` | `liftIO $ readFileMaybe path` (write a small helper, or inline) |
| `lift $ writeFileAtomicE path content` | `liftIO $ writeFile path content` |
| `lift $ copyFileE src dest` | `liftIO $ copyFile src dest` |
| `lift $ createDirE path` | `liftIO $ createDirectoryIfMissing True path` |
| `lift $ removeFileE path` | `liftIO $ safeRemove path` |
| `lift $ removeDirRecursiveE path` | `liftIO $ removeDirectoryRecursive path` |
| `lift $ exitWithE code` | `liftIO $ exitWith code` |
| `lift $ askUser prompt` | `liftIO $ putStr prompt >> hFlush stdout >> getLine` |
| `lift $ getCurrentDirE` | `liftIO $ getCurrentDirectory` |
| `lift $ gitRaw args` | `liftIO $ Git.runGitRaw args` (returns ExitCode) |
| `lift $ gitQuery args` | `liftIO $ Git.runGitWithOutput args` (returns (ExitCode, String, String)) |
| `lift $ rcloneExec cmd args` | **Replace with Transport call** (see Step 3) Γאפ for now, `liftIO $ rcloneExecIO cmd args` as temp shim |
| `void $ rcloneExec cmd args` (inside `Rgit` functions) | Same Γאפ temp shim for now |

**Functions that were `Rgit a` and need to become `IO a`:**
- `add`, `commit`, `diff`, `reset`, `rm`, `mv`, `branch`, `merge` Γאפ these become `IO ExitCode`, calling `Git.runGitRaw` directly
- `executeCommand` Γאפ becomes `IO ()` (takes `localRoot`, `remoteRoot`, `RcloneAction`)
- `executePullCommand` Γאפ becomes `IO ()`
- `getLocalHeadE` Γזע `getLocalHead :: IO (Maybe String)`
- `getHashFromBundleE` Γזע `getHashFromBundle :: FilePath -> IO (Maybe String)` (or just use `Git.getHashFromBundle`)
- `checkIsAheadE` Γזע use `Git.checkIsAhead`
- `hasStagedChangesE` Γזע use `Git.hasStagedChanges`
- `safeRemoveE` Γזע `safeRemove` (already exists as IO version)
- `isTextFileInIndex` Γאפ becomes `IO Bool`
- `copyFromIndexToWorkTree` Γאפ becomes `IO ()`

**Functions that stay as `RgitM a`** (they read from `RgitEnv`):
- `push`, `pull`, `fetch`, `verify`, `status`, `restore`, `checkout`
- `remoteShow`, `remoteCheck`
- `pushToRemote`, `syncRemoteFiles`, `syncRemoteFilesToLocal`
- `processExistingRemote`, `mergeContinue`
- `withRemoteUrl`

**For `withRemoteUrl`:**
```haskell
-- BEFORE:
withRemoteUrl :: (String -> RgitM ()) -> RgitM ()
withRemoteUrl action = do
  mUrl <- asks envRemoteUrl
  maybe (lift $ tellErr "Error: No remote URL configured.") action mUrl

-- AFTER:
withRemoteUrl :: (String -> RgitM ()) -> RgitM ()
withRemoteUrl action = do
  mUrl <- asks envRemoteUrl
  maybe (liftIO $ hPutStrLn stderr "Error: No remote URL configured.") action mUrl
```

**Temporary shim for `rcloneExec`** (will be removed in Step 3):
Add this helper at the bottom of Bit.hs:
```haskell
-- TEMPORARY: will be replaced by Transport calls in Step 3
rcloneExecIO :: String -> [String] -> IO ExitCode
rcloneExecIO cmd args = do
    (code, out, err) <- readProcessWithExitCode "rclone" (cmd : args) ""
    putStr out
    hPutStrLn stderr err
    return code
```

## 2d. Update imports in Bit.hs

Remove:
```haskell
import Rgit.Effect (Rgit, tell, tellErr, gitRaw, gitQuery, rcloneExec, readFileE,
    writeFileAtomicE, copyFileE, fileExistsE, dirExistsE, createDirE, removeFileE,
    removeDirRecursiveE, exitWithE, askUser, getCurrentDirE, liftIOE)
import Rgit.Effect.IO (runIO)
```

Add (as needed):
```haskell
import Control.Monad.IO.Class (liftIO)
import Control.Monad.Trans.Reader (asks)
import System.IO (hPutStrLn, hFlush, stdout, stderr)
import System.Exit (ExitCode(..), exitWith)
import System.Directory (doesFileExist, doesDirectoryExist, createDirectoryIfMissing,
    copyFile, removeFile, removeDirectoryRecursive, getCurrentDirectory, getFileSize, listDirectory)
import System.Process (readProcessWithExitCode)
```

## 2e. Delete effect system files

Delete these files entirely:
- `Rgit/Effect.hs`
- `Rgit/Effect/IO.hs`
- `Rgit/Effect/DryRun.hs`

Remove them from `rgit.cabal`. Remove the `free` package dependency from `rgit.cabal` if nothing else uses it.

## Verification

```bash
cabal clean && cabal build rgit
cabal test device-prompt
# No references to old effect system:
grep -rn "Rgit.Effect\|RgitF\|liftIOE\|rcloneExec\b.*Rgit\|Free RgitF" Rgit/ Bit.hs Internal/
# Should return 0 results (except maybe the temporary rcloneExecIO shim)
```

## Guardrails
- Do NOT change any business logic. This is a mechanical effect-removal refactoring.
- Do NOT change function signatures beyond `Rgit a Γזע IO a` and removing `lift $` / `liftIOE $` wrappers.
- The temporary `rcloneExecIO` shim exists only to keep things compiling. It goes away in Step 3.
- `RgitM` functions keep using `asks` and `liftIO` Γאפ that's the correct pattern for `ReaderT RgitEnv IO`.

---

# Step 3: Route All Rclone Through Transport.hs

**Files:** Modify `Bit.hs`, modify `Internal/Transport.hs`

**What:** Every rclone call in Bit.hs goes through Transport.hs. Remove the `rcloneExecIO` temporary shim. Remove the direct `System.Process.readProcessWithExitCode "rclone"` call in `uploadToRemote`.

## 3a. Fix `uploadToRemote` / `pushBundle`

**Current (broken Γאפ calls rclone directly):**
```haskell
uploadToRemote :: FilePath -> String -> IO ()
uploadToRemote src dest = do
    putStrLn "Uploading bundle to remote..."
    (rCode, _, rErr) <- System.Process.readProcessWithExitCode "rclone" ["copyto", src, dest] ""
    if rCode == ExitSuccess
        then putStrLn "Metadata push complete."
        else hPutStrLn stderr $ "Error uploading bundle: " ++ rErr
```

**After (uses Transport):**
```haskell
uploadToRemote :: FilePath -> String -> IO ()
uploadToRemote src dest = do
    putStrLn "Uploading bundle to remote..."
    rCode <- Transport.copyToRemote src dest
    if rCode == ExitSuccess
        then putStrLn "Metadata push complete."
        else hPutStrLn stderr "Error uploading bundle."
```

Note: `Transport.copyToRemote` already exists and does exactly this. We just call it.

## 3b. Replace `rcloneExecIO` in `executeCommand`

**Current:**
```haskell
executeCommand :: FilePath -> String -> RcloneAction -> IO ()
executeCommand localRoot remoteRoot action = case action of
    Copy src dest ->
        let localPath = toPosix (localRoot </> src)
            remotePath = remoteRoot ++ toPosix dest
        in do
            putStrLn $ "[DEBUG] executeCommand Copy: src=" ++ src ++ " dest=" ++ dest
            putStrLn $ "Executing: rclone copyto " ++ localPath ++ " " ++ remotePath
            void $ rcloneExecIO "copyto" [localPath, remotePath]

    Move src dest ->
        void $ rcloneExecIO "moveto" [remoteRoot ++ toPosix src, remoteRoot ++ toPosix dest]

    Delete path ->
        void $ rcloneExecIO "deletefile" [remoteRoot ++ toPosix path]

    Swap _ _ _ -> return ()
```

**After:**
```haskell
executeCommand :: FilePath -> String -> RcloneAction -> IO ()
executeCommand localRoot remoteRoot action = case action of
    Copy src dest -> do
        let localPath = toPosix (localRoot </> src)
            remotePath = remoteRoot ++ toPosix dest
        void $ Transport.copyToRemote localPath remotePath

    Move src dest ->
        void $ Transport.moveRemote (remoteRoot ++ toPosix src) (remoteRoot ++ toPosix dest)

    Delete path ->
        void $ Transport.deleteRemote (remoteRoot ++ toPosix path)

    Swap _ _ _ -> return ()
```

## 3c. Replace `rcloneExecIO` in `executePullCommand`

Apply the same pattern. The pull direction uses `Transport.copyFromRemote`:

```haskell
-- For pull copies (remote Γזע local):
void $ Transport.copyFromRemote remotePath localPath

-- For pull moves: download src, delete local dest if exists
void $ Transport.copyFromRemote remoteSrcPath (toPosix localSrcPath)
```

## 3d. Delete `rcloneExecIO`

Remove the temporary shim entirely from Bit.hs:
```haskell
-- DELETE THIS:
rcloneExecIO :: String -> [String] -> IO ExitCode
rcloneExecIO cmd args = do ...
```

## 3e. Remove direct rclone import from Bit.hs

Remove `import System.Process (readProcessWithExitCode)` from Bit.hs if it's no longer used for anything. (It may still be needed by `Rgit/Remote/Scan.hs` Γאפ check.)

Actually: `Rgit/Remote/Scan.hs` (`fetchRemoteFiles`) also calls `readProcess "rclone"` directly. Change it to use `Transport.listRemoteJson` or a new Transport function. This function:

```haskell
-- Current Remote.Scan:
raw <- readProcess "rclone" ["lsjson", remotePath, "--hash", "--recursive"] ""
```

Should become either:
- A new Transport function `listRemoteJsonRecursiveWithHash :: String -> IO (ExitCode, String, String)`, or
- Simply use the existing `Transport.listRemoteJson` with appropriate args.

Add to Transport.hs:
```haskell
-- | List remote recursively with hashes as JSON
listRemoteJsonWithHash :: String -> IO (ExitCode, String, String)
listRemoteJsonWithHash remotePath =
    readProcessWithExitCode "rclone" ["lsjson", remotePath, "--hash", "--recursive"] ""
```

Update `Remote.Scan.fetchRemoteFiles`:
```haskell
fetchRemoteFiles :: String -> IO (Either RemoteError [FileEntry])
fetchRemoteFiles remotePath = do
    (code, raw, _err) <- Transport.listRemoteJsonWithHash remotePath
    case code of
        ExitSuccess -> pure $ maybe
            (Left (DecodeFailed "Invalid rclone JSON output"))
            (Right . map rcloneFileToFileEntry . filter (not . rfIsDir))
            (decode (LBSC.pack raw) :: Maybe [RcloneFile])
        _ -> pure (Left RcloneFailed)
```

## Verification

```bash
cabal clean && cabal build rgit
cabal test device-prompt
# No direct rclone process calls outside Transport.hs:
grep -rn "readProcessWithExitCode.*rclone\|readProcess.*rclone" Bit.hs Rgit/ --include="*.hs" | grep -v "Internal/Transport.hs"
# Should return 0 results
# Also check for the temporary shim:
grep -rn "rcloneExecIO" Bit.hs Rgit/
# Should return 0 results
```

## Guardrails
- Transport.hs functions stay dumb Γאפ they take paths/URLs, return ExitCodes. No business logic.
- The `-- ESCAPE:` comments referencing rclone should now be removable. Remove them.
- Debug `[DEBUG]` print statements in `executeCommand` / `executePullCommand`: consider removing or guarding them behind a verbose flag. They're noisy. At minimum, remove the ones that print "Executing: rclone ..." since Transport handles that internally.

---

# Step 4: Move `FetchResult`, `classifyRemote`, `fetchBundle` from Transport to Bit

**Files:** Modify `Internal/Transport.hs`, modify `Bit.hs`

**What:** Transport.hs should not know about `.rgit/` layout, bundle files, or remote state classification. These are domain concepts that belong in Bit.hs.

## 4a. Move types out of Transport

Move these **out of** `Internal/Transport.hs`:
```haskell
-- MOVE to Bit.hs (or a new Rgit/RemoteState.hs if you prefer):
data RemoteState 
    = StateEmpty
    | StateValidRgit
    | StateNonRgitOccupied [String]
    | StateCorruptedRgit String
    | StateNetworkError String
    deriving (Show, Eq)

data FetchResult 
    = BundleFound FilePath 
    | RemoteEmpty 
    | NetworkError String
    deriving (Show, Eq)
```

Remove these from `Internal/Transport.hs` exports.

## 4b. Move `classifyRemote` to Bit.hs

**Current (in Transport.hs):** `classifyRemote` calls `listRemoteJson` then interprets the result to decide if the remote has `.rgit/`, is empty, etc.

**After:** Move the function to Bit.hs. It calls `Transport.listRemoteJson` for the raw data, then applies the domain logic:

```haskell
-- In Bit.hs:
classifyRemoteState :: String -> IO RemoteState
classifyRemoteState url = do
    (code, out, err) <- Transport.listRemoteJson url 1
    -- ... same interpretation logic that was in Transport.classifyRemote ...
    -- This is the RIGHT place for it: Bit.hs knows what .rgit/ means
```

Also move `RcloneItem` and its `FromJSON` instance if `classifyRemote` was the only consumer. If `Transport.listRemoteJson` returns raw JSON string, Bit.hs can parse it with its own `RcloneItem` type. Alternatively, Transport can return `[(String, Bool)]` (name, isDir) pairs Γאפ a dumb representation.

**Cleaner approach:** Have Transport return parsed-but-dumb data:

```haskell
-- In Transport.hs:
data TransportItem = TransportItem
    { tiName  :: String
    , tiIsDir :: Bool
    } deriving (Show, Eq)

listRemoteItems :: String -> Int -> IO (Either String [TransportItem])
-- parses JSON, returns items. No domain interpretation.
```

Then in Bit.hs:
```haskell
classifyRemoteState :: String -> IO RemoteState
classifyRemoteState url = do
    result <- Transport.listRemoteItems url 1
    case result of
        Left err -> return (StateNetworkError err)
        Right items -> interpretRemoteItems items

interpretRemoteItems :: [TransportItem] -> RemoteState
-- pure function! Easy to test.
-- Checks for .rgit/ directory, etc.
```

## 4c. Move `fetchBundle` to Bit.hs

**Current (in Transport.hs):**
```haskell
fetchBundle :: String -> IO FetchResult
fetchBundle remoteUrl = do
    let remotePath = remoteUrl ++ "/.rgit/rgit.bundle"  -- Γזנ knows about .rgit/ layout!
    let localDest = ".rgit/temp_remote.bundle"            -- Γזנ knows about local .rgit/ layout!
    (code, _, err) <- readProcessWithExitCode "rclone" ["copyto", remotePath, localDest] ""
    -- ... interprets errors ...
```

**After (in Bit.hs):**
```haskell
fetchBundle :: String -> IO FetchResult
fetchBundle remoteUrl = do
    let remotePath = remoteUrl ++ "/.rgit/rgit.bundle"
    let localDest = ".rgit/temp_remote.bundle"
    code <- Transport.copyFromRemote remotePath localDest
    case code of
        ExitSuccess -> return (BundleFound localDest)
        ExitFailure _ -> return (NetworkError "Failed to fetch bundle from remote")
```

Note: the detailed error classification (checking stderr for "directory not found" vs "no such host") is lost because `Transport.copyFromRemote` only returns `ExitCode`. Two options:

**Option A (simple):** Accept the less detailed error. Most callers only care about success/failure.

**Option B (add Transport helper):** Add a Transport function that returns more detail:
```haskell
-- Transport.hs:
data CopyResult = CopySuccess | CopyNotFound | CopyNetworkError String | CopyOtherError String
    deriving (Show, Eq)

copyFromRemoteDetailed :: String -> FilePath -> IO CopyResult
copyFromRemoteDetailed remotePath localPath = do
    (code, _, err) <- readProcessWithExitCode "rclone" ["copyto", remotePath, localPath] ""
    case code of
        ExitSuccess -> return CopySuccess
        ExitFailure _ 
            | "directory not found" `isInfixOf` err || "object not found" `isInfixOf` err -> return CopyNotFound
            | "no such host" `isInfixOf` err || "dial tcp" `isInfixOf` err -> return (CopyNetworkError err)
            | otherwise -> return (CopyOtherError err)
```

This keeps Transport dumb (it classifies rclone errors, not domain state) while giving Bit.hs the info it needs. **Recommend Option B.**

## 4d. Update Transport.hs exports

After removing `RemoteState`, `FetchResult`, `classifyRemote`, `fetchBundle`:

```haskell
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
```

## 4e. Update all callers

Every place in Bit.hs that says `Transport.classifyRemote` Γזע now calls local `classifyRemoteState`.
Every place that says `Transport.fetchBundle` Γזע now calls local `fetchBundle`.
Every place that says `Transport.StateEmpty` etc Γזע now uses the locally-defined `StateEmpty` etc.
Every place that says `Transport.BundleFound` etc Γזע now uses local `BundleFound` etc.

## Verification

```bash
cabal clean && cabal build rgit
cabal test device-prompt
# Transport.hs should not know about .rgit/:
grep -n "\.rgit\|bundle\|RemoteState\|FetchResult\|classifyRemote\b\|fetchBundle\b" Internal/Transport.hs
# Should return 0 results
```

## Guardrails
- `interpretRemoteItems` (the pure core of `classifyRemote`) should be a pure function for easy testing.
- Transport's `CheckResult` stays in Transport Γאפ it's a parse of rclone's `--combined` output, which is Transport-level knowledge.
- The `RcloneItem` type with its JSON parsing can move entirely to Bit.hs (or a helper) if Transport now returns `TransportItem` instead.

---

# Step 5: Fix `unsetUpstream`

**Files:** Modify `Bit.hs`

**What:** One-line fix. Git already prints its own error message on failure. The redundant message is noise.

**Before:**
```haskell
unsetUpstream :: IO ()
unsetUpstream = do
    code <- Git.unsetBranchUpstream
    when (code /= ExitSuccess) $ hPutStrLn stderr "fatal: Failed to unset upstream."
```

**After:**
```haskell
unsetUpstream :: IO ()
unsetUpstream = void Git.unsetBranchUpstream
```

## Verification
```bash
cabal clean && cabal build rgit
cabal test device-prompt
```

---

# Step 6: Thread `Remote` Through Bit.hs Instead of `String`

**Files:** Modify `Rgit/Types.hs`, modify `Bit.hs`, modify `Rgit/Commands.hs`

**What:** Replace `envRemoteUrl :: Maybe String` with `envRemote :: Maybe Remote`. Replace `withRemoteUrl` with `withRemote`. All Bit.hs functions receive a `Remote` and only extract the URL when calling Transport.

## 6a. Update `RgitEnv`

```haskell
-- Rgit/Types.hs
import Rgit.Remote (Remote)

data RgitEnv = RgitEnv
    { envCwd            :: FilePath
    , envLocalFiles     :: [FileEntry]
    , envRemote         :: Maybe Remote    -- was: envRemoteUrl :: Maybe String
    , envForce          :: Bool
    , envForceWithLease :: Bool
    }
```

## 6b. Update `Commands.hs` env construction

```haskell
-- BEFORE:
mRemote <- Bit.getRemoteTarget
let env = RgitEnv cwd localFiles mRemote isForce isForceWithLease

-- AFTER:
import Rgit.Remote (getDefaultRemote)
mRemote <- getDefaultRemote cwd
let env = RgitEnv cwd localFiles mRemote isForce isForceWithLease
```

The `Bit.getRemoteTarget` function (which currently returns `Maybe String`) is replaced by `Rgit.Remote.getDefaultRemote` (which returns `Maybe Remote`). Remove `getRemoteTarget` from Bit.hs exports.

For `remoteAdd`:
```haskell
-- This stays as-is since it takes explicit name and url from CLI:
["remote", "add", name, url] -> Bit.remoteAdd name url
```

For `remoteShow` / `remoteCheck` with explicit name:
```haskell
-- These may need the explicit name resolved:
["remote", "show", name] -> do
    mRemote <- resolveRemote cwd name
    let env = RgitEnv cwd localFiles mRemote isForce isForceWithLease
    runRgitM env $ Bit.remoteShow (Just name)
```

## 6c. Replace `withRemoteUrl` with `withRemote`

```haskell
-- BEFORE:
withRemoteUrl :: (String -> RgitM ()) -> RgitM ()
withRemoteUrl action = do
  mUrl <- asks envRemoteUrl
  maybe (liftIO $ hPutStrLn stderr "Error: No remote URL configured.") action mUrl

-- AFTER:
import Rgit.Remote (Remote(..), remoteUrl)

withRemote :: (Remote -> RgitM ()) -> RgitM ()
withRemote action = do
  mRemote <- asks envRemote
  case mRemote of
    Nothing -> liftIO $ hPutStrLn stderr "Error: No remote configured."
    Just remote -> action remote
```

## 6d. Update all callers of `withRemoteUrl`

Every function that currently does:
```haskell
withRemoteUrl $ \url -> do
    ...Transport.someFunc url...
    ...pushBundle url...
```

Becomes:
```haskell
withRemote $ \remote -> do
    ...Transport.someFunc (remoteUrl remote)...
    ...pushBundle remote...
```

**Key pattern:** Bit.hs functions receive `Remote`. Only when calling Transport functions do they extract `remoteUrl remote`.

Functions to update (all in Bit.hs):
- `push` Γאפ `withRemoteUrl $ \url ->` Γזע `withRemote $ \remote ->`
- `pull`, `pullWithCleanup`, `pullAcceptRemoteImpl`, `pullManualMergeImpl` Γאפ same pattern
- `fetch` Γאפ same
- `verify` (when `isRemote`) Γאפ same
- `syncRemoteFiles` Γאפ same
- `syncRemoteFilesToLocal` Γאפ same
- `pushToRemote :: String -> RgitM ()` Γזע `pushToRemote :: Remote -> RgitM ()`
- `pushBundle :: String -> IO ()` Γזע `pushBundle :: Remote -> IO ()`
- `fetchBundle :: String -> IO FetchResult` Γזע `fetchBundle :: Remote -> IO FetchResult`
- `classifyRemoteState :: String -> IO RemoteState` Γזע `classifyRemoteState :: Remote -> IO RemoteState`
- `fetchRemoteBundle :: String -> IO (Maybe FilePath)` Γזע `fetchRemoteBundle :: Remote -> IO (Maybe FilePath)`
- `saveFetchedBundle :: String -> Maybe FilePath -> IO ()` Γזע `saveFetchedBundle :: Remote -> Maybe FilePath -> IO ()`

**Inside these functions**, where the URL is needed for Transport:
```haskell
pushBundle :: Remote -> IO ()
pushBundle remote = do
    let url = remoteUrl remote
    let tempBundle = "rgit.bundle"
    let remotePath = url ++ "/.rgit/rgit.bundle"
    ...
```

## 6e. Update `processExistingRemote`

This function currently retrieves `mUrl <- asks envRemoteUrl` directly in several places. Change to:
```haskell
processExistingRemote :: FilePath -> RgitM ()
processExistingRemote bPath = do
    ...
    mRemote <- asks envRemote
    ...
    maybe (liftIO $ hPutStrLn stderr "Error: No remote configured.")
          (\remote -> pushToRemote remote) mRemote
```

## 6f. Update remote display messages

Where Bit.hs prints the URL to the user (e.g. `"Inspecting remote: " ++ url`), use `remoteName` for user-facing messages and `remoteUrl` for Transport calls:

```haskell
-- BEFORE:
putStrLn $ "Inspecting remote: " ++ url

-- AFTER:
let r = remote
liftIO $ putStrLn $ "Inspecting remote '" ++ remoteName r ++ "' (" ++ remoteUrl r ++ ")"
```

## 6g. Remove `resolveRemoteByName` from Bit.hs

This function is replaced by `Rgit.Remote.resolveRemote`. Delete it from Bit.hs. Update `remoteShow` and `remoteCheck` which currently call it.

## 6h. Remove `getRemoteTarget` from Bit.hs

This function is replaced by `Rgit.Remote.getDefaultRemote`. Delete it. Remove from exports.

## Verification

```bash
cabal clean && cabal build rgit
cabal test device-prompt
# No raw URL strings in Bit.hs env access:
grep -n "envRemoteUrl" Bit.hs Rgit/
# Should return 0 results
# withRemoteUrl should not exist:
grep -n "withRemoteUrl" Bit.hs
# Should return 0 results
```

## Guardrails
- `remoteUrl` is ONLY extracted when making a Transport call. Bit.hs logic should pass `Remote` around.
- User-facing messages use `remoteName`. Transport calls use `remoteUrl`.
- `remoteAdd` stays as `String -> String -> IO ()` because it creates a new remote from CLI args Γאפ there's no `Remote` to resolve yet.

---

# Step 7: Cleanup Pass

**Files:** All modified files

**What:** Final cleanup after the structural changes are in place.

## 7a. Remove `-- ESCAPE:` comments

These marked places where the effect system was bypassed. With the effect system gone, they're meaningless. Delete all lines containing `-- ESCAPE:`.

## 7b. Remove `[DEBUG]` print statements

The `executeCommand` and `executePullCommand` functions have debug prints:
```haskell
putStrLn $ "[DEBUG] executeCommand Copy: src=" ++ src ++ " dest=" ++ dest
```
Remove these or guard behind a `--verbose` flag.

## 7c. Remove `ensureRemoteRoot` if possible

This helper adds a trailing slash to URLs. With the `Remote` type, consider normalizing the URL at resolution time (in `Rgit/Remote.hs`) so callers don't need to call `ensureRemoteRoot`:

```haskell
-- In Rgit/Remote.hs resolveRemote:
-- Normalize: ensure URL does NOT end with /
-- Then in Bit.hs, use: remoteUrl remote ++ "/" ++ path
-- This is clearer than ensureRemoteRoot which silently mutates strings
```

## 7d. Audit imports

After all changes, many imports in Bit.hs will be dead. Run:
```bash
cabal build rgit 2>&1 | grep "redundant import\|imported from"
```
Clean up unused imports.

## 7e. Remove `import qualified System.Process` from Bit.hs

After Step 3, Bit.hs should not import `System.Process` at all. If it does, something was missed.

## 7f. Consider moving `RemoteState` and `FetchResult` to `Rgit/Remote.hs`

If you created these types directly in Bit.hs in Step 4, consider whether `Rgit/Remote.hs` is a better home. The `Remote` module already owns the concept of "what a remote is" Γאפ it could also own "what state a remote is in."

## Verification

```bash
cabal clean && cabal build rgit
cabal test device-prompt

# Final audit Γאפ Bit.hs should not:
grep -n "readProcessWithExitCode\|readProcess\b" Bit.hs    # call processes directly
grep -n "rclone" Bit.hs                                     # mention rclone (except in comments)
grep -n "envRemoteUrl" Bit.hs Rgit/                         # use raw URL from env
grep -n "\-\- ESCAPE" Bit.hs                                # have escape markers
grep -n "\[DEBUG\]" Bit.hs                                  # have debug prints

# Transport.hs should not:
grep -n "\.rgit\|bundle\|RemoteState\|FetchResult" Internal/Transport.hs  # know about domain concepts
```

---

# Summary of What Changes Per File

| File | Action |
|------|--------|
| **Create** `Rgit/Remote.hs` | New: `Remote` type, `resolveRemote`, `getDefaultRemote` |
| **Delete** `Rgit/Effect.hs` | Remove free monad functor and smart constructors |
| **Delete** `Rgit/Effect/IO.hs` | Remove IO interpreter |
| **Delete** `Rgit/Effect/DryRun.hs` | Remove dry-run interpreter |
| `Rgit/Types.hs` | `RgitM = ReaderT RgitEnv IO`. `envRemote :: Maybe Remote`. Remove `Rgit` import. |
| `Bit.hs` | **Major.** Remove effect wrappers. All rclone via Transport. `Remote` instead of URL strings. Move `classifyRemote`/`fetchBundle`/`FetchResult`/`RemoteState` here from Transport. Fix `unsetUpstream`. |
| `Internal/Transport.hs` | Remove `FetchResult`, `RemoteState`, `classifyRemote`, `fetchBundle`. Add `CopyResult`, `copyFromRemoteDetailed`, `listRemoteJsonWithHash`, `listRemoteItems`, `TransportItem`. |
| `Rgit/Commands.hs` | Remove `runIO`. Use `runRgitM` directly. Build env with `Maybe Remote` instead of `Maybe String`. |
| `Rgit/Remote/Scan.hs` | Use `Transport.listRemoteJsonWithHash` instead of direct `readProcess "rclone"`. |
| `rgit.cabal` | Add `Rgit.Remote`. Remove `Rgit.Effect`, `Rgit.Effect.IO`, `Rgit.Effect.DryRun`. Possibly remove `free` dependency. |
```

---

## docs/spec.md

**Path:** `docs/spec.md`

*Source file.*

```markdown
# rgit Implementation Specification for Cursor (v2)

## Context and Vision

You are implementing **rgit** Γאפ a version control system designed for large files that leverages Git as a metadata-tracking engine while storing actual file content separately. The core insight is brilliant: Git excels at tracking small text files, so we feed it exactly that Γאפ tiny metadata files instead of large binaries.

**Mental Model**: rgit = Git(metadata) + rclone(sync) + [optional CAS(content) for rgit-solid]

**Current State**: We are implementing **rgit-lite** first. CAS and content-addressed storage come later in **rgit-solid**.

---

## Core Architecture

### Directory Structure

```
project/
Γפ£ΓפאΓפא actual_files/           # User's working directory (large files live here)
Γפג   Γפ£ΓפאΓפא src/
Γפג   Γפג   ΓפפΓפאΓפא video.mp4
Γפג   ΓפפΓפאΓפא data/
Γפג       ΓפפΓפאΓפא dataset.bin
Γפג
ΓפפΓפאΓפא .rgit/
    Γפ£ΓפאΓפא index/              # Metadata files live here
    Γפג   ΓפפΓפאΓפא .git/           # Git's internal directory
    Γפג   Γפ£ΓפאΓפא src/
    Γפג   Γפג   ΓפפΓפאΓפא video.mp4   # Metadata file (NOT the actual video)
    Γפג   ΓפפΓפאΓפא data/
    Γפג       ΓפפΓפאΓפא dataset.bin # Metadata file (NOT the actual data)
    Γפ£ΓפאΓפא target              # Remote URL (rclone remote path)
    ΓפפΓפאΓפא ignore              # Gitignore rules
```

### Metadata File Format

Each metadata file under `.rgit/index/` mirrors the path of its corresponding real file and contains **ONLY**:

```
hash: sha256:a1b2c3d4e5f6...
size: 1048576
```

**That's it. Two fields only.**
- `hash` Γאפ SHA256 of file content
- `size` Γאפ File size in bytes (for quick diff display without computing hash)

### Git Configuration

rgit runs Git with:
- Git repository initialized in `.rgit/index/.git` during init
- All git operations use the repo at `.rgit/index/.git`
- Working tree is `.rgit/index` (not the project root)
- `core.excludesFile` points to `.rgit/ignore`

The `.rgit/ignore` file contains:
```
*
!.rgit/
!.rgit/index/
!.rgit/ignore
!.rgit/target
.rgit/index/.git/
```

---

## Command Line Interface (Git-Compatible)

**CRITICAL**: The CLI must mirror Git's interface as closely as possible. Users familiar with Git should feel immediately at home.

### Command Mapping

| Command | Git Equivalent | rgit Behavior |
|---------|---------------|---------------|
| `rgit init` | `git init` | Initialize `.rgit/` with internal Git repo |
| `rgit add <path>` | `git add` | Compute metadata, write to `.rgit/index/`, stage in Git |
| `rgit add .` | `git add .` | Add all modified/new files |
| `rgit commit -m "msg"` | `git commit` | Commit staged metadata changes |
| `rgit status` | `git status` | Show working tree vs metadata vs staged |
| `rgit diff` | `git diff` | Show hash/size changes (human-readable) |
| `rgit diff --staged` | `git diff --staged` | Show staged metadata changes |
| `rgit log` | `git log` | Show commit history |
| `rgit restore [options] [--] <path>` | `git restore` | Restore metadata; full git syntax: --staged, --worktree, --source=, etc. |
| `rgit checkout [options] -- <path>` | `git checkout --` | Restore working tree from index (legacy syntax, same as restore) |
| `rgit reset` | `git reset` | Reset staging area |
| `rgit rm <path>` | `git rm` | Remove file from tracking |
| `rgit mv <src> <dst>` | `git mv` | Move/rename tracked file |
| `rgit branch` | `git branch` | Branch management |
| `rgit merge` | `git merge` | Merge branches |
| `rgit remote add <name>` | - | Set remote URL (stored in `.rgit/target`) |
| `rgit remote show` | `git remote show` | Show remote status |
| `rgit push` | `git push` | Push metadata bundle + sync files via rclone |
| `rgit pull` | `git pull` | Pull metadata bundle + sync files via rclone |
| `rgit fetch` | `git fetch` | Fetch metadata bundle only |

### Output Format (Match Git Exactly)

```bash
$ rgit status
On branch main
Changes to be committed:
  (use "rgit reset HEAD <file>..." to unstage)
        modified:   src/video.mp4

Changes not staged for commit:
  (use "rgit add <file>..." to update what will be committed)
        modified:   data/dataset.bin

Untracked files:
  (use "rgit add <file>..." to include in what will be committed)
        new_file.mp4
```

```bash
$ rgit diff
diff --git a/src/video.mp4 b/src/video.mp4
--- a/src/video.mp4
+++ b/src/video.mp4
@@ -1,2 +1,2 @@
-hash=sha256:abc123...
-size=1048576
+hash=sha256:def456...
+size=1049000
```

---

## Remote Synchronization (Two-Phase, Action-Based)

### Key Insight: Diff-Based Sync, Not Blind Sync

We do **NOT** use `rclone sync`. Instead:

1. Compute diff between current state and desired state
2. Generate minimal action list (add, delete, move)
3. Execute actions via rclone

This saves bandwidth. For example: renaming a 1GB file becomes `rclone move` instead of delete + upload.

### Sync Order (CRITICAL)

**On Push:**
1. **First**: Sync files via rclone (content must exist before metadata claims it does)
2. **Then**: Push metadata bundle via rclone

**On Pull:**
1. **First**: Fetch metadata bundle via rclone
2. **Then**: Sync files via rclone (need metadata to know what to fetch)

**Rationale**: 
- Push: If metadata arrives first but content fails, remote is in inconsistent state
- Pull: Need metadata to know what content to fetch

### Action Generation

Given:
- `localFiles`: Map of local file paths Γזע (hash, size)
- `remoteFiles`: Map of remote file paths Γזע (hash, size)  
- `localMeta`: Metadata from local Git HEAD
- `remoteMeta`: Metadata from remote bundle

**For Push:**
```
toUpload   = files in localMeta but not in remote (by hash)
toDelete   = files in remote but not in localMeta (by path)
toMove     = files where same hash exists in both but different path
```

**For Pull:**
```
toDownload = files in remoteMeta but not in local (by hash)
toDelete   = files in local but not in remoteMeta (by path)  
toMove     = files where same hash exists in both but different path
```

### Rclone Commands Used

```bash
# Upload new/changed file
rclone copyto local/path remote:path

# Download file
rclone copyto remote:path local/path

# Move/rename (saves re-upload!)
rclone moveto remote:old/path remote:new/path

# Delete
rclone delete remote:path

# List remote for scanning
rclone lsjson remote:path --recursive --hash
```

---

## Verification and Consistency

### `rgit verify`

Verifies local working tree matches local metadata:

```bash
$ rgit verify
Verifying 1,234 files...
Γ£ף All files match metadata.

# Or with errors:
$ rgit verify
Verifying 1,234 files...
Γ£ק Hash mismatch: src/video.mp4
  Expected: sha256:abc123...
  Actual:   sha256:xyz789...
Γ£ק Missing: data/old_file.bin
  
2 issues found. Run 'rgit status' for details.
```

### `rgit verify --remote`

Verifies that remote files match what the remote Git bundle says they should be.

**What this checks:**
1. Fetch remote bundle (metadata)
2. Scan remote files via `rclone lsjson --hash`
3. Compare: Do actual remote files match what remote metadata claims?

```bash
$ rgit verify --remote
Fetching remote metadata... done.
Scanning remote files... done.
Comparing...

Γ£ק Remote divergence detected:
  
  src/video.mp4:
    Metadata says: sha256:abc123... (1048576 bytes)
    Actual remote: sha256:xyz789... (1049000 bytes)
    
  data/extra.bin:
    Not in metadata but exists on remote
    
This can happen when:
  - Files were modified directly on the remote (not via rgit)
  - A partial push from another client
  - Remote storage corruption

Run 'rgit pull --accept-remote' to accept remote reality as truth.
```

### `rgit fsck`

**Local-only** integrity check (like **git fsck**): no network. Two steps, both purely local:

| Step | What it checks |
|------|----------------|
| [1/2] | Local working tree vs local metadata (rgit equivalent of checking objects) |
| [2/2] | **git fsck** on `.rgit/index/.git` (metadata history integrity) |

Terse output, one line per problem, exit 1 if any check fails. When all checks pass, prints nothing.

**Output format (aligned with git fsck):**

- Working tree vs local metadata: `missing <path>` (tracked file absent), `hash mismatch <path>` (content/size differs from metadata). Paths use forward slashes.
- Metadata history: **git fsck** output is printed as-is (e.g. `missing blob <sha>`, `dangling blob <sha>`).

**Example (problems found):**

```bash
$ rgit fsck
hash mismatch path/to/corrupted.bin
missing path/to/deleted.txt
missing blob abc123...
```

**Verification of binary files:** Both **hash** (MD5, same as rclone) and **size** are checked. Text files are verified by content (hash/size derived from stored content).

**Checking against the remote:** Use **`rgit verify --remote`** (see below). `rgit fsck` does *not* touch the network.

| Command | What it checks | Network? |
|---------|----------------|----------|
| `rgit fsck` | local files vs local metadata + git fsck on internal repo | No |
| `rgit verify` | local files vs local metadata (same as fsck step 1, standalone) | No |
| `rgit verify --remote` | remote storage vs remote metadata | Yes |

---

## Handling Remote Divergence

When remote files don't match remote metadata (detected via `rgit verify --remote` or during `rgit pull`):

### Resolution Option 1: Accept Remote Reality (`--accept-remote`)

```bash
$ rgit pull --accept-remote
```

**What happens:**
1. Scan actual remote files (hash + size)
2. Update local metadata to match actual remote state
3. Create new commit: "Accept remote file state as truth"
4. Download any files that differ from local
5. Local now matches actual remote

**Use when:** Someone legitimately changed files on remote outside rgit, and you want to accept those changes.

### Resolution Option 2: Force Local to Remote (`--force-local`)

```bash
$ rgit push --force-local
```

**What happens:**
1. Upload local files to remote, overwriting any divergent files
2. Push metadata bundle
3. Remote now matches local

**Use when:** You know local is correct and remote is wrong (e.g., accidental remote modification).

Γתá∩╕ן **DANGEROUS**: This overwrites remote files. Data loss possible.

### Resolution Option 3: Manual Merge

When neither automatic resolution is appropriate:

```bash
$ rgit pull --manual-merge
```

**Exact behavior:**

1. Fetch remote metadata bundle
2. Scan actual remote files
3. For each divergent file, create a conflict directory:
   ```
   .rgit/conflicts/
   ΓפפΓפאΓפא src/
       ΓפפΓפאΓפא video.mp4/
           Γפ£ΓפאΓפא LOCAL           # symlink or copy of local version
           Γפ£ΓפאΓפא REMOTE          # downloaded from remote
           Γפ£ΓפאΓפא METADATA_LOCAL  # what local metadata says
           ΓפפΓפאΓפא METADATA_REMOTE # what remote metadata says
   ```
4. Print conflict list:
   ```
   CONFLICT: src/video.mp4
     Local:           sha256:abc123... (1048576 bytes)
     Remote actual:   sha256:xyz789... (1049000 bytes)  
     Remote metadata: sha256:abc123... (1048576 bytes)
     
     Files saved to: .rgit/conflicts/src/video.mp4/
     
   To resolve:
     1. Examine files in .rgit/conflicts/src/video.mp4/
     2. Copy your chosen version to src/video.mp4
     3. Run 'rgit add src/video.mp4'
     4. Run 'rgit merge --continue'
   ```
5. User manually resolves by:
   - Examining both versions
   - Copying chosen version to working tree
   - `rgit add <resolved_file>`
   - `rgit merge --continue` (commits resolution and cleans up conflict dir)

Or abort: `rgit merge --abort` (discards conflict state, reverts to pre-pull state)

---

## Atomic Operations

### File Operations

All file write operations must be atomic to prevent corruption:

```haskell
-- Write to temp file, then rename (atomic on POSIX)
atomicWriteFile :: FilePath -> ByteString -> IO ()
atomicWriteFile target content = do
    let tempDir = takeDirectory target
    (tempPath, handle) <- openTempFile tempDir ".rgit-tmp"
    BS.hPut handle content
    hClose handle
    renameFile tempPath target
```

### Directory Operations

For operations affecting multiple files (like checkout):

```haskell
-- 1. Create temp directory with new state
-- 2. Atomic rename to replace target
atomicDirectoryReplace :: FilePath -> IO () -> IO ()
atomicDirectoryReplace targetDir buildAction = do
    let tempDir = targetDir <> ".rgit-tmp"
    createDirectory tempDir
    withCurrentDirectory tempDir buildAction
    renameDirectory tempDir targetDir  -- Atomic on same filesystem
```

### Transaction Log (for multi-step operations)

For push/pull that involve multiple rclone commands:

```
.rgit/transaction.log

OPERATION: push
STARTED: 2024-01-15T10:30:00Z
STEPS:
  [DONE] upload src/video.mp4
  [DONE] upload data/file.bin  
  [PENDING] delete old/removed.mp4
  [PENDING] push metadata bundle
```

If interrupted, `rgit push` can resume from where it left off.

---

## Text Files: Special Handling

### The Idea

Text files (source code, config, markdown, etc.) are small and diff-friendly. For these:

1. Store them **directly** in `.rgit/index/` (actual content, not metadata)
2. Git versions them normally (with full diff capability)
3. Sync them with content intact

This gives you proper Git diffs for text files while still handling large binaries via metadata.

### Detection Heuristics

A file is considered **text** if ALL of:
1. Size < 1MB (configurable via `.rgit/config`)
2. No NULL bytes in first 8KB
3. Valid UTF-8 (or ASCII subset)
4. Not in binary extension list: `.mp4`, `.zip`, `.bin`, `.exe`, `.dll`, `.so`, `.dylib`, `.jpg`, `.png`, `.gif`, `.pdf`, etc.

### Implementation

```haskell
data FileClassification 
    = TextFile      -- Store content directly in .rgit/index/
    | BinaryFile    -- Store metadata only

classifyFile :: FilePath -> IO FileClassification
classifyFile path = do
    size <- getFileSize path
    if size > textFileSizeLimit 
        then return BinaryFile
        else do
            sample <- BS.readFile path  -- Or just first 8KB
            if isValidText sample
                then return TextFile
                else return BinaryFile

-- For text files:
-- .rgit/index/src/config.yaml contains actual YAML content

-- For binary files:
-- .rgit/index/src/video.mp4 contains:
--   hash=sha256:...
--   size=...
```

### Config Option

In `.rgit/config`:
```ini
[text]
    size-limit = 1048576  # 1MB, files larger are always binary
    extensions = .txt,.md,.yaml,.yml,.json,.xml,.html,.css,.js,.py,.hs,.rs
```

---

## Symlinks and Special Files

### What rgit Tracks

rgit tracks **regular files only**. Specifically:

- Γ£ו Regular files (binary Γזע metadata, text Γזע content)
- Γ¥ל Symlinks Γאפ **ignored** (too many edge cases with cloud remotes)
- Γ¥ל Device files Γאפ **ignored**
- Γ¥ל Sockets Γאפ **ignored**
- Γ¥ל Named pipes Γאפ **ignored**

### Empty Directories

**Empty directories are ignored.** They are not tracked.

If a directory becomes empty after `rgit rm`, it is simply not represented in metadata. When someone else pulls, they won't get the empty directory Γאפ that's fine.

Rationale: Empty directories have no content to version. Many cloud backends don't even support them.

---

## Sparse Checkout (rgit-solid only)

**Note:** This feature requires CAS (Content-Addressed Storage), which is part of **rgit-solid**, not rgit-lite.

### The Concept

Instead of materializing all files, create symlinks to CAS blobs:

```
project/
Γפ£ΓפאΓפא src/
Γפג   ΓפפΓפאΓפא video.mp4 Γזע ../.rgit/cas/a1/b2c3d4...  (symlink)
ΓפפΓפאΓפא .rgit/
    ΓפפΓפאΓפא cas/
        ΓפפΓפאΓפא a1/
            ΓפפΓפאΓפא b2c3d4...  (actual content, read-only)
```

### Rules

1. **Only non-executable files** can be symlinked (executables need to be real files)
2. **CAS blobs are read-only** (mode 0444)
3. **Editing requires materialization:**
   ```bash
   $ vim src/video.mp4
   # Attempt to write to read-only file
   # Editor may prompt to force write, or:
   
   $ rgit materialize src/video.mp4
   # Replaces symlink with actual file copy (writable)
   ```

### Commands (rgit-solid)

```bash
$ rgit checkout --sparse main
# Only fetches metadata, creates symlinks to CAS

$ rgit materialize src/video.mp4
# Downloads content if needed, replaces symlink with real file

$ rgit materialize --all
# Materializes everything (like regular checkout)
```

### When to Implement rgit-solid?

We implement rgit-lite first, but **design with CAS in mind**. Specifically:
- Keep hash computation consistent
- Don't couple metadata format to "lite" assumptions

---

## Error Messages (Match Git Style)

```bash
$ rgit add nonexistent.file
error: pathspec 'nonexistent.file' did not match any files

$ rgit commit
On branch main
nothing to commit, working tree clean

$ rgit push
fatal: No remote configured.
hint: Set remote with 'rgit remote add <url>'

$ rgit pull
error: Remote divergence detected.
hint: Remote files do not match remote metadata.
hint: Run 'rgit verify --remote' for details.
hint: Options:
hint:   rgit pull --accept-remote  (accept remote files as truth)
hint:   rgit pull --manual-merge   (resolve conflicts manually)
```

---

## Current Implementation State (from GitHub)

Based on the existing code, here's what exists:

### Already Implemented
- `rgit init` Γאפ creates `.rgit/`, initializes Git with `--separate-git-dir`
- `rgit add` Γאפ delegates to `git add`
- `rgit commit` Γאפ delegates to `git commit`
- `rgit diff` Γאפ delegates to `git diff`
- `rgit status` Γאפ shows Git status
- `rgit restore` Γאפ delegates to `git restore`
- `rgit remote add` Γאפ stores URL in `.rgit/target`
- `rgit remote show` Γאפ shows remote status
- `rgit fetch` Γאפ fetches remote bundle
- `rgit push` Γאפ pushes bundle + syncs files
- `Scan.scanWorkingDir` Γאפ scans files, computes hashes
- `Scan.writeMetadataFiles` Γאפ writes metadata to `.rgit/index/`
- `Rclone.classifyRemote` Γאפ detects remote state (empty, valid, corrupted)
- `Rclone.fetchBundle` / `pushBundle` Γאפ bundle operations
- Bundle-based remote tracking (`fetched_remote.bundle`)

### Needs Refinement
- File sync uses rclone but may not be fully action-based yet
- Divergence detection exists but resolution options incomplete
- Text file handling not yet implemented
- Atomic operations may need hardening

### Not Yet Implemented
- `rgit verify` / `rgit verify --remote`
- `rgit fsck`
- Manual merge conflict resolution
- Sparse checkout (rgit-solid)
- CAS (rgit-solid)
- Transaction logging for resumable operations
```

---

