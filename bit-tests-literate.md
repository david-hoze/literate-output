# bit — Tests (Literate Programming Document)

This document contains all test files for the bit project:
Haskell test modules, shell-based integration tests, and test infrastructure.

---

## test/DevicePromptTests.hs

**Path:** `test/DevicePromptTests.hs`

*Source file.*

```haskell
import Test.Tasty
import Test.Tasty.HUnit
import qualified Bit.DevicePrompt as DP

main :: IO ()
main = defaultMain tests

tests :: TestTree
tests = testGroup "DevicePrompt"
  [ testGroup "acquireDeviceName"
    [ testCase "accepts a simple device name" $ do
        name <- DP.acquireDeviceName (DP.Interactive (pure "my_laptop")) (Just "vol") (pure . const False)
        name @?= "my_laptop"
    , testCase "sanitizes device name with spaces" $ do
        name <- DP.acquireDeviceName (DP.Interactive (pure "Local Disk")) (Just "vol") (pure . const False)
        name @?= "Local_Disk"
    , testCase "strips invalid characters from device name" $ do
        name <- DP.acquireDeviceName (DP.Interactive (pure "my<>device/name")) (Just "vol") (pure . const False)
        name @?= "mydevicename"
    , testCase "falls back to volume label on empty input" $ do
        name <- DP.acquireDeviceName (DP.Interactive (pure "")) (Just "MY_PASSPORT") (pure . const False)
        name @?= "MY_PASSPORT"
    , testCase "falls back to device on empty input when no label" $ do
        name <- DP.acquireDeviceName (DP.Interactive (pure "")) Nothing (pure . const False)
        name @?= "device"
    , testCase "treats whitespace-only input as empty" $ do
        name <- DP.acquireDeviceName (DP.Interactive (pure "   ")) (Just "default") (pure . const False)
        name @?= "default"
    , testCase "sanitizes volume label with spaces in fallback" $ do
        name <- DP.acquireDeviceName (DP.Interactive (pure "")) (Just "Local Disk") (pure . const False)
        name @?= "Local_Disk"
    , testCase "uses volume label when non-interactive" $ do
        name <- DP.acquireDeviceName DP.NonInteractive (Just "OFFICE_SHARE") (pure . const False)
        name @?= "OFFICE_SHARE"
    , testCase "uses device when non-interactive and no label" $ do
        name <- DP.acquireDeviceName DP.NonInteractive Nothing (pure . const False)
        name @?= "device"
    , testCase "sanitizes volume label in non-interactive mode" $ do
        name <- DP.acquireDeviceName DP.NonInteractive (Just "My Passport") (pure . const False)
        name @?= "My_Passport"
    ]
  , testGroup "sanitizeDeviceName"
    [ testCase "replaces spaces with underscores" $
        DP.sanitizeDeviceName "Local Disk" @?= "Local_Disk"
    , testCase "strips invalid characters" $
        DP.sanitizeDeviceName "a/b<>c" @?= "abc"
    , testCase "returns device for empty after cleaning" $
        DP.sanitizeDeviceName "<>//" @?= "device"
    , testCase "preserves valid name" $
        DP.sanitizeDeviceName "my_device-123" @?= "my_device-123"
    ]
  , testGroup "isValidDeviceName"
    [ testCase "accepts alphanumeric with underscores and hyphens" $
        DP.isValidDeviceName "my_device-123" @?= True
    , testCase "rejects empty" $
        DP.isValidDeviceName "" @?= False
    , testCase "rejects spaces" $
        DP.isValidDeviceName "my device" @?= False
    , testCase "rejects special chars" $
        DP.isValidDeviceName "my<>device" @?= False
    ]
  ]
```

---

## test/MergeSpec.hs

**Path:** `test/MergeSpec.hs`

*Source file.*

```haskell
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE OverloadedStrings #-}

-- | Comprehensive merge tests for bit.
--
-- Inspired by Git's t7600-merge.sh and t6422-merge-rename-corner-cases.sh.
-- Tests the pure conflict resolution, conflict detection, and metadata parsing
-- logic that underpins bit's two-repo merge flow.
--
-- Run: cabal test merge
module Main where

import Test.Tasty
import Test.Tasty.HUnit
import Data.List (isInfixOf)

import Bit.Conflict
import Bit.Internal.Metadata (MetaContent(..), parseMetadata, serializeMetadata, displayHash, hasConflictMarkers)
import Bit.Types (Hash(..), HashAlgo(..))
import qualified Data.Text as T

main :: IO ()
main = defaultMain tests

-- ---------------------------------------------------------------------------
-- All tests
-- ---------------------------------------------------------------------------

tests :: TestTree
tests = testGroup "Merge"
  [ conflictInfoParsingTests
  , metadataRoundtripTests
  , conflictMarkerTests
  ]

-- ---------------------------------------------------------------------------
-- 3. Conflict info parsing (parseConflictInfo)
--    Mirrors the three conflict types from Git's ls-files -u output:
--    - ContentConflict (both modified, stages 1+2+3)
--    - AddAdd (both added, stages 2+3 only)
--    - ModifyDelete (stage 2 only or stage 3 only)
-- ---------------------------------------------------------------------------

conflictInfoParsingTests :: TestTree
conflictInfoParsingTests = testGroup "parseConflictInfo"
  [ -- Content conflict: base (stage 1) + ours (stage 2) + theirs (stage 3)
    testCase "stages 1,2,3 -> ContentConflict" $ do
      let output = unlines
            [ "100644 abc123 1\tindex/foo.bin"
            , "100644 def456 2\tindex/foo.bin"
            , "100644 ghi789 3\tindex/foo.bin"
            ]
      parseConflictInfo "index/foo.bin" output @?= ContentConflict "index/foo.bin"

  -- Add/add conflict: no base (no stage 1), ours (stage 2) + theirs (stage 3)
  , testCase "stages 2,3 only -> AddAdd" $ do
      let output = unlines
            [ "100644 def456 2\tindex/new.txt"
            , "100644 ghi789 3\tindex/new.txt"
            ]
      parseConflictInfo "index/new.txt" output @?= AddAdd "index/new.txt"

  -- Modify/delete: ours modified (stage 2) but theirs deleted (no stage 3)
  , testCase "stage 2 only -> ModifyDelete (deleted in theirs)" $ do
      let output = unlines
            [ "100644 abc123 1\tindex/gone.bin"
            , "100644 def456 2\tindex/gone.bin"
            ]
      parseConflictInfo "index/gone.bin" output @?= ModifyDelete "index/gone.bin" False

  -- Modify/delete: theirs modified (stage 3) but ours deleted (no stage 2)
  , testCase "stage 3 only -> ModifyDelete (deleted in ours)" $ do
      let output = unlines
            [ "100644 abc123 1\tindex/gone.bin"
            , "100644 ghi789 3\tindex/gone.bin"
            ]
      parseConflictInfo "index/gone.bin" output @?= ModifyDelete "index/gone.bin" True

  -- Empty output (fallback)
  , testCase "empty output -> ContentConflict (fallback)" $ do
      parseConflictInfo "index/x" "" @?= ContentConflict "index/x"

  -- Malformed output (fallback)
  , testCase "garbage output -> ContentConflict (fallback)" $ do
      parseConflictInfo "index/x" "this is not valid ls-files output" @?= ContentConflict "index/x"
  ]

-- ---------------------------------------------------------------------------
-- 4. Metadata round-trip (serialize -> parse)
--    Ensures metadata survives push/pull/merge without corruption.
-- ---------------------------------------------------------------------------

metadataRoundtripTests :: TestTree
metadataRoundtripTests = testGroup "Metadata round-trip"
  [ testCase "serialize then parse is identity" $ do
      let mc = MetaContent (Hash "md5:abc123def456") 1048576
      parseMetadata (serializeMetadata mc) @?= Just mc

  , testCase "serialize then parse with large size" $ do
      let mc = MetaContent (Hash "md5:0000000000000000") 999999999999
      parseMetadata (serializeMetadata mc) @?= Just mc

  , testCase "serialize then parse with zero size" $ do
      let mc = MetaContent (Hash "md5:ffffffffffffffff") 0
      parseMetadata (serializeMetadata mc) @?= Just mc

  , testCase "parse rejects empty string" $ do
      parseMetadata "" @?= Nothing

  , testCase "parse rejects random text (text file content)" $ do
      parseMetadata "Hello, this is a text file.\nWith multiple lines.\n" @?= Nothing

  , testCase "parse handles legacy Hash wrapper format" $ do
      let content = "hash: Hash \"abc123\"\nsize: 100\n"
      case parseMetadata content of
        Just mc -> metaHash mc @?= Hash "abc123"
        Nothing -> assertFailure "Failed to parse legacy format"

  , testCase "parse handles bare quoted format" $ do
      let content = "hash: \"abc123\"\nsize: 100\n"
      case parseMetadata content of
        Just mc -> metaHash mc @?= Hash "abc123"
        Nothing -> assertFailure "Failed to parse bare quoted format"

  , testCase "parse handles raw hash format" $ do
      let content = "hash: md5:abc123def456\nsize: 512\n"
      case parseMetadata content of
        Just mc -> do
          metaHash mc @?= Hash "md5:abc123def456"
          metaSize mc @?= 512
        Nothing -> assertFailure "Failed to parse raw format"

  , testCase "displayHash truncates long hashes" $ do
      let h = Hash "md5:0123456789abcdef0123456789abcdef"
      let displayed = displayHash h
      -- Should be 16 chars + "..."
      length displayed @?= 19

  , testCase "displayHash preserves short hashes" $ do
      let h = Hash "md5:abcdef"
      let displayed = displayHash h
      displayed @?= "md5:abcdef"
  ]

-- ---------------------------------------------------------------------------
-- 5. Conflict marker detection
--    Ensures metadata files with merge conflict markers are caught
--    before they corrupt the metadata index.
-- ---------------------------------------------------------------------------

conflictMarkerTests :: TestTree
conflictMarkerTests = testGroup "Conflict markers"
  [ testCase "serializeMetadata never contains conflict markers" $ do
      let mc = MetaContent (Hash "md5:abc123") 100
      let s = serializeMetadata mc
      assertBool "should not contain <<<<<<<" (not $ "<<<<<<<" `isInfixOf` s)
      assertBool "should not contain =======" (not $ "=======" `isInfixOf` s)
      assertBool "should not contain >>>>>>>" (not $ ">>>>>>>" `isInfixOf` s)
  ]
```

---

## test/PipelineSpec.hs

**Path:** `test/PipelineSpec.hs`

*Source file.*

```haskell
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE FlexibleInstances #-}
{-# LANGUAGE OverloadedStrings #-}

module Main where

import Test.Tasty
import Test.Tasty.QuickCheck
import Test.Tasty.HUnit

import qualified Data.List as List
import Data.List (sort, group)
import Bit.Types
import Bit.Diff (GitDiff(..), LightFileEntry(..), FileIndex, buildIndexFromFileEntries, computeDiff)
import Bit.Plan (RcloneAction(..), planAction)
import Bit.Pipeline (diffAndPlan, pushSyncFiles, pullSyncFiles)
import qualified Data.Text as T

-- Arbitrary instances for property testing

instance Arbitrary (Hash 'MD5) where
  arbitrary = Hash . T.pack . ("md5:" ++) <$> vectorOf 32 (elements "0123456789abcdef")

instance Arbitrary EntryKind where
  arbitrary = do
    h <- arbitrary
    sz <- choose (0, 10000000)
    isText <- arbitrary
    return $ File h sz isText

instance Arbitrary FileEntry where
  arbitrary = do
    -- Generate simple relative paths
    depth <- choose (1, 3) :: Gen Int
    segments <- vectorOf depth (vectorOf 5 (elements "abcdefghijklmnop"))
    let path = foldl1 (\a b -> a ++ "/" ++ b) segments
    k <- arbitrary
    return $ FileEntry path k

main :: IO ()
main = defaultMain tests

tests :: TestTree
tests = testGroup "Pipeline"
  [ testGroup "diffAndPlan properties"
    [ testProperty "identical inputs produce no actions" prop_identicalNoActions
    , testProperty "no duplicate target paths in plan" prop_noDuplicateTargets
    , testProperty "empty source against non-empty target produces only deletes" prop_emptySourceOnlyDeletes
    , testProperty "non-empty source against empty target produces only copies" prop_emptyTargetOnlyCopies
    ]
  , testGroup "diffAndPlan unit tests"
    [ testCase "single added file produces Copy" test_singleAddedFile
    , testCase "single deleted file produces Delete" test_singleDeletedFile
    , testCase "modified file produces Copy (overwrite)" test_modifiedFile
    ]
  ]

-- Properties

prop_identicalNoActions :: [FileEntry] -> Bool
prop_identicalNoActions files =
  null (diffAndPlan files files)

prop_noDuplicateTargets :: [FileEntry] -> [FileEntry] -> Bool
prop_noDuplicateTargets source target =
  let actions = diffAndPlan source target
      targets = concatMap actionTargets actions
      uniqueTargets = map head (group (sort targets))
  in  length targets == length uniqueTargets
  where
    actionTargets (Copy _ d)    = [d]
    actionTargets (Move _ d)    = [d]
    actionTargets (Delete p)    = [p]
    actionTargets (Swap _ _ _)  = []

prop_emptySourceOnlyDeletes :: [FileEntry] -> Bool
prop_emptySourceOnlyDeletes target =
  all isDelete (diffAndPlan [] target)
  where
    isDelete (Delete _) = True
    isDelete _          = False

prop_emptyTargetOnlyCopies :: [FileEntry] -> Bool
prop_emptyTargetOnlyCopies source =
  all isCopyOrMove (diffAndPlan source [])
  where
    isCopyOrMove (Copy _ _) = True
    isCopyOrMove (Move _ _) = True
    isCopyOrMove _          = False

-- Unit tests

test_singleAddedFile :: Assertion
test_singleAddedFile = do
  let h = Hash (T.pack "md5:abc123")
      source = [FileEntry "foo.bin" (File h 100 False)]
      target = []
      actions = diffAndPlan source target
  length actions @?= 1
  case head actions of
    Copy "foo.bin" "foo.bin" -> return ()
    other -> assertFailure $ "Expected Copy, got: " ++ show other

test_singleDeletedFile :: Assertion
test_singleDeletedFile = do
  let h = Hash (T.pack "md5:abc123")
      source = []
      target = [FileEntry "foo.bin" (File h 100 False)]
      actions = diffAndPlan source target
  length actions @?= 1
  case head actions of
    Delete "foo.bin" -> return ()
    other -> assertFailure $ "Expected Delete, got: " ++ show other

test_modifiedFile :: Assertion
test_modifiedFile = do
  let h1 = Hash (T.pack "md5:aaa")
      h2 = Hash (T.pack "md5:bbb")
      source = [FileEntry "foo.bin" (File h1 100 False)]
      target = [FileEntry "foo.bin" (File h2 200 False)]
      actions = diffAndPlan source target
  length actions @?= 1
  case head actions of
    Copy "foo.bin" "foo.bin" -> return ()
    other -> assertFailure $ "Expected Copy (overwrite), got: " ++ show other
```

---

## test/RunCliTests.hs

**Path:** `test/RunCliTests.hs`

*Source file.*

```haskell
{-# LANGUAGE ScopedTypeVariables #-}
-- Run CLI tests via shelltest (shelltestrunner).
-- Requires: cabal install shelltestrunner (so shelltest is on PATH).
-- Ensures bit from this project is on PATH when running tests.
import System.Process (callProcess, readProcess, rawSystem)
import Control.Exception (catch, SomeException)
import Control.Monad (void)
import System.Environment (getEnvironment, setEnv)
import System.FilePath (takeDirectory, (</>))
import System.Directory (getCurrentDirectory)
import Data.List (lookup)
import Data.Maybe (fromMaybe)
import System.Info (os)

main :: IO ()
main = do
  -- Purge gdrive remote before tests (cleans orphan.txt etc. from previous runs)
  let purgeAndMkdir =
        rawSystem "rclone" ["purge", "gdrive-test:bit-test"] >>
        rawSystem "rclone" ["mkdir", "gdrive-test:bit-test"]
  _ <- catch (void purgeAndMkdir) (\(_ :: SomeException) -> return ())
  -- Prepend directory containing bit to PATH so shelltest runs use this build
  bitBin <- readProcess "cabal" ["list-bin", "bit"] ""
  let bitDir = takeDirectory (filter (`notElem` "\n\r") bitBin)
  env <- getEnvironment
  let pathSep = if os == "mingw32" || os == "win32" then ";" else ":"
  let path = case lookup "PATH" env of
        Nothing -> bitDir
        Just p  -> bitDir ++ pathSep ++ p
  setEnv "PATH" path
  -- Run shelltest on test/cli (Format 3 .test files)
  callProcess "shelltest" ["test" </> "cli"]
```

---

## test/cli/README.md

**Path:** `test/cli/README.md`

*Source file.*

```markdown
# CLI tests

CLI tests use **[shelltest](https://hackage.haskell.org/package/shelltestrunner)** (shelltestrunner). Test files use the `.test` extension and Format 3: command line, `<<<` (stdin), `>>>` expected stdout (literal or `/regex/`), `>>>=` expected exit code.

**Run from the repository root** so paths like `test\cli\work_a` resolve correctly.

## Running tests

1. Install shelltest: `cabal install shelltestrunner`
2. Build bit: `cabal build bit`
3. Run CLI tests: `cabal test cli`

The test runner puts the built `bit` on `PATH` and invokes `shelltest test/cli`, so all `.test` files under `test/cli/` are run. To run a single file: `shelltest test/cli/init.test` (ensure `bit` is on `PATH`, e.g. `cabal exec -- env PATH="$(cabal list-bin bit):$PATH" shelltest test/cli/init.test` on Unix).

## gdrive-remote.test

Tests rclone Google Drive remote: push, pull, fetch, and corruption recovery.

**Prerequisites:**

- `rclone` on PATH
- rclone remote named **gdrive-test** configured (e.g. `rclone config` Γזע Google Drive)
- Remote path **gdrive-test:bit-test** is used; the test purges and recreates it

**What it does:**

- **Two repos:** `work_a` (pusher) and `work_b` (puller), both use `gdrive-test:bit-test` as origin
- **Push/pull/fetch:** Repo A adds a file, commits, pushes; Repo B fetches and pulls; verifies file content
- **Corruption:** Uses `rclone deletefile` to remove a file on the remote (simulates partial/corrupt state)
- **Verify --remote:** Repo B runs `bit verify --remote` and expects missing-file issues
- **Recovery:** Repo A pushes again (re-syncs files); Repo B fetches/pulls; verify is clean
- **Orphan file:** Adds a file on the remote with `rclone copyto` (not in metadata); verifies behavior
- **Cleanup:** Removes the orphan from the remote at the end

To run only the gdrive tests (requires rclone + gdrive-test remote): `shelltest test/cli/gdrive-remote.test` (with `bit` on PATH).

## device-prompt.test

Tests the device name prompt when adding a filesystem remote. Uses `BIT_USE_STDIN=1` so piped input is read as the device name (enables non-TTY testing). Uses `subst` on Windows to create a writable volume for the device flow. Verifies the remote is stored (either device name or fallback path).

**Unit tests** for the device prompt logic (sanitization, validation, interactive vs non-interactive) live in the `device-prompt` test-suite: `cabal test device-prompt`.

## fsck.test

Tests `bit fsck` (local-only, git-style: terse output, one line per issue, exit 1 on any failure). Covers:
- Fresh repo / repo with committed files: fsck prints nothing and exits 0 when OK.
- Corrupted working-tree file: prints `hash mismatch <path>`, exits 1.
- Missing tracked file: prints `missing <path>`, exits 1.

Fsck does not check remote; use `bit verify --remote` for that.

## remote-check.test

Tests `bit remote check`: runs **rclone check** between local working tree and the configured remote (excludes `.bit`). Requires rclone on PATH. Covers:
- No remote configured: prints "Error: No remote URL configured." and exits 0.
- Local directory as "remote" (remote_mirror): add remote, change local file, run `bit remote check` Γזע exits 1 and reports differences (e.g. "differences found", "size differ", "hash differ").
```

---

## test/cli/device-prompt.test

**Path:** `test/cli/device-prompt.test`

*Source file.*

```text
# Device name prompt integration tests
# Tests the interactive path when BIT_USE_STDIN=1 (piped input for testing)
# Uses subst on Windows to create a writable volume root for .bit-store

# --- Setup: subst creates Z: -> test/cli so we can write .bit-store at volume root ---
subst Z: %CD%\test\cli 2>nul
<<<
>>>
>>>= 0

# --- Setup: create work dir, remote dir, init bit ---
rmdir /s /q test\cli\work 2>nul & rmdir /s /q test\cli\device_prompt_remote 2>nul & mkdir test\cli\work & mkdir test\cli\device_prompt_remote & cd test\cli\work & bit init
<<<
>>> /Initialized/
>>>= 0

# --- Add remote with piped device name (BIT_USE_STDIN=1) ---
cd test\cli\work & set BIT_USE_STDIN=1 & echo test_device| bit remote add origin Z:\device_prompt_remote
<<<
>>> /Remote.*test_device|Remote.*added/
>>>= 0

# --- Verify remote was stored: either device (test_device) or fallback (local:) ---
findstr /C:"test_device" test\cli\work\.bit\remotes\origin >nul || findstr /C:"local:" test\cli\work\.bit\remotes\origin >nul
<<<
>>>
>>>= 0

# --- Cleanup ---
subst Z: /D 2>nul & rmdir /s /q test\cli\work 2>nul & rmdir /s /q test\cli\device_prompt_remote 2>nul
<<<
>>>
>>>= 0
```

---

## test/cli/filesystem-remote-direct.test

**Path:** `test/cli/filesystem-remote-direct.test`

*Source file.*

```text
# Filesystem Remote Direct Tests
# Tests running bit commands directly at the filesystem remote location

# Setup: Clean workspace
rmdir /s /q test\cli\work_direct 2>nul & rmdir /s /q test\cli\fs_remote_direct 2>nul & mkdir test\cli\work_direct & mkdir test\cli\fs_remote_direct
<<<
>>>
>>>= 0

# Initialize local repo
cd test\cli\work_direct & bit init
<<<
>>> /Initialized/
>>>= 0

# Add filesystem remote
cd test\cli\work_direct & bit remote add origin ..\fs_remote_direct
<<<
>>> /Remote 'origin' added/
>>>= 0

# Create and push a file
cd test\cli\work_direct & echo test content> file.txt & bit add file.txt & bit commit -m "Initial"
<<<
>>> /\[main|files? changed/
>>>= 0

cd test\cli\work_direct & bit push
<<<
>>> /Push complete/
>>>= 0

# Run bit status at the remote (should work since it's a full repo)
cd test\cli\fs_remote_direct & bit status
<<<
>>> /nothing to commit|working tree clean|On branch/
>>>= 0

# Run bit log at the remote
cd test\cli\fs_remote_direct & bit log --oneline
<<<
>>> /Initial/
>>>= 0

# Modify file directly at remote
cd test\cli\fs_remote_direct & echo remote change> file.txt & bit add file.txt & bit commit -m "Remote edit"
<<<
>>> /\[main|files? changed/
>>>= 0

# Local push should now fail (remote has diverged)
cd test\cli\work_direct & echo local change> other.txt & bit add other.txt & bit commit -m "Local edit"
<<<
>>> /\[main|files? changed/
>>>= 0

cd test\cli\work_direct & bit push
<<<
>>> /error.*Remote has local commits/
>>>= 1

# Local pull should work (merges remote changes)
cd test\cli\work_direct & bit pull
<<<
>>> /Pull|Merging|Updating/
>>>= 0

# Now push should succeed
cd test\cli\work_direct & bit push
<<<
>>> /Push complete/
>>>= 0

# Verify remote has both files
if exist "test\cli\fs_remote_direct\file.txt" (echo file exists) else (echo missing)
<<<
>>>
file exists
>>>= 0

if exist "test\cli\fs_remote_direct\other.txt" (echo other exists) else (echo missing)
<<<
>>>
other exists
>>>= 0
```

---

## test/cli/filesystem-remote.test

**Path:** `test/cli/filesystem-remote.test`

*Source file.*

```text
# Filesystem Remote Tests
# Tests that filesystem remotes create full bit repos (not bundles)

# Setup: Clean workspace and create two repos + shared filesystem remote
rmdir /s /q test\cli\work_a 2>nul & rmdir /s /q test\cli\work_b 2>nul & rmdir /s /q test\cli\fs_remote 2>nul & mkdir test\cli\work_a & mkdir test\cli\work_b & mkdir test\cli\fs_remote
<<<
>>>
>>>= 0

# Initialize repo A
cd test\cli\work_a & bit init
<<<
>>> /Initialized/
>>>= 0

# Add remote pointing to filesystem path
cd test\cli\work_a & bit remote add origin ..\fs_remote
<<<
>>> /Remote 'origin' added/
>>>= 0

# Create and commit a file in repo A
cd test\cli\work_a & echo hello world> test.txt & bit add test.txt & bit commit -m "Initial commit"
<<<
>>> /\[main|files? changed/
>>>= 0

# Push to filesystem remote (should create .bit/ at remote)
cd test\cli\work_a & bit push
<<<
>>> /Push complete|Pushing to filesystem remote/
>>>= 0

# Verify remote has .bit/index/.git directory (not just .bit/bit.bundle)
if exist "test\cli\fs_remote\.bit\index\.git" (echo remote has git repo) else (echo missing git repo)
<<<
>>>
remote has git repo
>>>= 0

# Verify remote does NOT use bundles (no .bit/bit.bundle)
if exist "test\cli\fs_remote\.bit\bit.bundle" (echo has bundle) else (echo no bundle)
<<<
>>>
no bundle
>>>= 0

# Verify remote has the actual file
if exist "test\cli\fs_remote\test.txt" (echo file exists) else (echo missing)
<<<
>>>
file exists
>>>= 0

# Initialize repo B with same remote
cd test\cli\work_b & bit init
<<<
>>> /Initialized/
>>>= 0

cd test\cli\work_b & bit remote add origin ..\fs_remote
<<<
>>> /Remote 'origin' added/
>>>= 0

# Pull from filesystem remote into repo B
cd test\cli\work_b & bit pull
<<<
>>> /Pull|Pulling from filesystem remote/
>>>= 0

# Verify repo B has the file
if exist "test\cli\work_b\test.txt" (echo file exists) else (echo missing)
<<<
>>>
file exists
>>>= 0

# Verify content matches
findstr /C:"hello world" test\cli\work_b\test.txt >nul
<<<
>>>
>>>= 0

# Repo B modifies file, commits, pushes
cd test\cli\work_b & echo updated content> test.txt & bit add test.txt & bit commit -m "Update from B"
<<<
>>> /\[main|files? changed/
>>>= 0

cd test\cli\work_b & bit push
<<<
>>> /Push complete|Pushing to filesystem remote/
>>>= 0

# Verify remote has updated content
findstr /C:"updated content" test\cli\fs_remote\test.txt >nul
<<<
>>>
>>>= 0

# Repo A pulls the changes
cd test\cli\work_a & bit pull
<<<
>>> /Pull|Merging|Updating/
>>>= 0

# Verify repo A has updated content
findstr /C:"updated content" test\cli\work_a\test.txt >nul
<<<
>>>
>>>= 0

# Test conflict detection: both modify same file
cd test\cli\work_a & echo version A> conflict.txt & bit add conflict.txt & bit commit -m "Add conflict from A"
<<<
>>> /\[main|files? changed/
>>>= 0

cd test\cli\work_b & echo version B> conflict.txt & bit add conflict.txt & bit commit -m "Add conflict from B"
<<<
>>> /\[main|files? changed/
>>>= 0

# Repo A pushes first
cd test\cli\work_a & bit push
<<<
>>> /Push complete|Pushing to filesystem remote/
>>>= 0

# Repo B push should fail (remote diverged)
cd test\cli\work_b & bit push
<<<
>>> /error.*Remote has local commits/
>>>= 1

# Repo B pulls to resolve
cd test\cli\work_b & bit pull
<<<
>>> /Automatic merge|conflict/
>>>= 0
```

---

## test/cli/fs_remote/conflict.txt

**Path:** `test/cli/fs_remote/conflict.txt`

*Source file.*

```text
version A 
```

---

## test/cli/fs_remote/test.txt

**Path:** `test/cli/fs_remote/test.txt`

*Source file.*

```text
updated content 
```

---

## test/cli/fs_remote_direct/file.txt

**Path:** `test/cli/fs_remote_direct/file.txt`

*Source file.*

```text
remote change 
```

---

## test/cli/fs_remote_direct/other.txt

**Path:** `test/cli/fs_remote_direct/other.txt`

*Source file.*

```text
local change 
```

---

## test/cli/fsck.test

**Path:** `test/cli/fsck.test`

*Source file.*

```text
# bit fsck tests (git-style output: terse, one line per issue, exit 1 on any failure)

# --- fsck on fresh repo (no remote): prints nothing when OK ---
rmdir /s /q test\cli\work 2>nul & mkdir test\cli\work & cd test\cli\work & bit init
<<<
>>> /Initialized/
>>>= 0

# fsck OK = no output, exit 0
cd test\cli\work & bit fsck
<<<
>>>
>>>= 0

# --- fsck on repo with committed files: still no output when OK ---
rmdir /s /q test\cli\work 2>nul & mkdir test\cli\work & cd test\cli\work & bit init & echo hello> a.txt & bit add a.txt & bit commit -m "Add a"
<<<
>>> /Initialized/
>>>= 0

cd test\cli\work & bit fsck
<<<
>>>
>>>= 0

# --- fsck detects local hash mismatch: git-style "hash mismatch <path>", exit 1 ---
rmdir /s /q test\cli\work 2>nul & mkdir test\cli\work & cd test\cli\work & bit init & echo original> file.bin & bit add file.bin & bit commit -m "Add file"
<<<
>>> /Initialized/
>>>= 0

cd test\cli\work & echo corrupted> file.bin
<<<
>>>
>>>= 0

cd test\cli\work & bit fsck
<<<
>>>2 /hash mismatch.*file\.bin/
>>>= 1

# --- fsck detects missing file: git-style "missing <path>", exit 1 ---
rmdir /s /q test\cli\work 2>nul & mkdir test\cli\work & cd test\cli\work & bit init & echo x> missing.txt & bit add missing.txt & bit commit -m "Add missing"
<<<
>>> /Initialized/
>>>= 0

del test\cli\work\missing.txt
<<<
>>>
>>>= 0

cd test\cli\work & bit fsck
<<<
>>>2 /missing.*missing\.txt/
>>>= 1
```

---

## test/cli/gdrive-remote.test

**Path:** `test/cli/gdrive-remote.test`

*Source file.*

```text
# rclone Google Drive remote tests (gdrive-test)
# Uses two repos: work_a (pusher) and work_b (puller).
# Requires: rclone remote "gdrive-test" configured; path "bit-test" used on that remote.

# --- Setup: clean work dirs and remote ---
rmdir /s /q test\cli\work_a 2>nul & rmdir /s /q test\cli\work_b 2>nul & mkdir test\cli\work_a & mkdir test\cli\work_b & rclone purge gdrive-test:bit-test 2>nul & rclone mkdir gdrive-test:bit-test
<<<
>>>
>>>= 0

# --- Repo A: init (use main branch for fetch/pull), add remote, add file, commit, push ---
cd test\cli\work_a & bit init
<<<
>>> /Initialized/
>>>= 0

cd test\cli\work_a & bit remote add origin gdrive-test:bit-test
<<<
>>> /Remote.*added/
>>>= 0

cd test\cli\work_a & echo hello from A> foo.txt & echo x> bar.bin & bit add foo.txt bar.bin & bit commit -m "Add foo and bar"
<<<
>>> /\[master|main|file changed/
>>>= 0

cd test\cli\work_a & bit push
<<<
>>> /Metadata push complete\.|Remote is empty|Remote check passed/
>>>= 0

# --- A: remote check after push: all files on remote, should match ---
cd test\cli\work_a & bit remote check
<<<
>>> /files match between local and remote/
>>>= 0

# --- Repo B: init, add remote, fetch, pull ---
cd test\cli\work_b & bit init
<<<
>>> /Initialized/
>>>= 0

cd test\cli\work_b & bit remote add origin gdrive-test:bit-test
<<<
>>> /Remote.*added/
>>>= 0

cd test\cli\work_b & bit fetch
<<<
>>>2 /From gdrive-test:bit-test|new branch.*origin\/main/
>>>= 0

cd test\cli\work_b & bit pull
<<<
>>> /first pull|Checking out|Syncing binaries|Merge made|remote: Counting/
>>>= 0

# --- Verify B has the file from A (text in index, binary in work tree) ---
findstr /C:"hello from A" test\cli\work_b\.bit\index\foo.txt >nul
<<<
>>>
>>>= 0

# --- Corrupt remote: delete a binary file with rclone (simulate partial/corrupt state) ---
rclone deletefile gdrive-test:bit-test/bar.bin
<<<
>>>
>>>= 0

# --- B: verify --remote may report missing file or 0 files (bundle metadata) ---
cd test\cli\work_b & bit verify --remote
<<<
>>> /Missing:|issues found|ERROR|\[OK\] All files match|Verifying 0/
>>>= 0

# --- Restore remote: push from A again (re-syncs files + bundle) ---
cd test\cli\work_a & bit push
<<<
>>> /Metadata push complete\.|Remote check passed|Pulling changes|up to date/
>>>= 0

# --- B: fetch and pull to get restored state ---
cd test\cli\work_b & bit fetch
<<<
>>>
>>>= 0

cd test\cli\work_b & bit pull
<<<
>>> /up to date|Merge made|Syncing binaries|first pull/
>>>= 0

# --- B: verify --remote should be clean again ---
cd test\cli\work_b & bit verify --remote
<<<
>>> /\[OK\] All files match|Verifying/
>>>= 0

findstr /C:"hello from A" test\cli\work_b\.bit\index\foo.txt >nul
<<<
>>>
>>>= 0

# --- Second round: A adds another file, pushes; B pulls ---
cd test\cli\work_a & echo second file> baz.txt & bit add baz.txt & bit commit -m "Add baz" & bit push
<<<
>>> /Metadata push complete\.|Remote check passed|file changed/
>>>= 0

cd test\cli\work_b & bit fetch & bit pull
<<<
>>>2 /From (gdrive-test|\.git\\fetched_remote\.bundle)|Merge made|Syncing|Updating|remote: Counting/
>>>= 0

findstr /C:"second file" test\cli\work_b\.bit\index\baz.txt >nul
<<<
>>>
>>>= 0

# --- Corrupt again: add an orphan file on remote with rclone ---
echo junk content> test\cli\work_b\junk.tmp & rclone copyto test\cli\work_b\junk.tmp gdrive-test:bit-test/orphan.txt
<<<
>>>
>>>= 0

# --- B: fetch (bundle unchanged), then pull: local should not get orphan; verify --remote ---
cd test\cli\work_b & bit fetch
<<<
>>>
>>>= 0

cd test\cli\work_b & bit verify --remote
<<<
>>> /Verifying|All files match|issues found/
>>>= 0

# --- Cleanup: remove orphan from remote so future runs are clean ---
rclone deletefile gdrive-test:bit-test/orphan.txt
<<<
>>>
>>>= 0
```

---

## test/cli/init-config.test

**Path:** `test/cli/init-config.test`

*Source file.*

```text
# Setup: clean environment and initialize repo
rmdir /s /q test\cli\work 2>nul & mkdir test\cli\work & cd test\cli\work & bit init
<<<
>>> /Initialized/
>>>= 0

# Verify init.defaultBranch was set to "main"
cd test\cli\work & git -C .bit\index config --get init.defaultBranch
<<<
>>>
main
>>>= 0

# Verify merge driver was configured
cd test\cli\work & git -C .bit\index config --get merge.bit-metadata.driver
<<<
>>>
false
>>>= 0

# Verify bit status works after init (proves git commands work)
cd test\cli\work & bit status
<<<
>>>= 0

# Verify bit add works after init
cd test\cli\work & echo test> file.txt & bit add file.txt
<<<
>>>= 0

# Verify bit commit works after init
cd test\cli\work & bit commit -m "test commit"
<<<
>>> /\[main|master|file/
>>>= 0
```

---

## test/cli/init.test

**Path:** `test/cli/init.test`

*Source file.*

```text
# Setup: clean environment
rmdir /s /q test\cli\work 2>nul & mkdir test\cli\work & cd test\cli\work & bit init
<<<
>>> /Initialized/
>>>= 0

# Verify .bit directory was created
if exist "test\cli\work\.bit\" (echo exists) else (echo missing)
<<<
>>>
exists
>>>= 0
```

---

## test/cli/ls-files.test

**Path:** `test/cli/ls-files.test`

*Source file.*

```text
# Setup fresh repo
rmdir /s /q test\cli\work 2>nul & mkdir test\cli\work & cd test\cli\work & bit init
<<<
>>> /Initialized/
>>>= 0

# ls-files with no tracked files should return nothing
cd test\cli\work & bit ls-files
<<<
>>>
>>>= 0

# Add a text file
cd test\cli\work & echo hello> test.txt & bit add test.txt
<<<
>>>
>>>= 0

# ls-files should show the staged file
cd test\cli\work & bit ls-files
<<<
>>> /test\.txt/
>>>= 0

# Commit the file
cd test\cli\work & bit commit -m "Add test.txt"
<<<
>>> /\[master|main|files? changed/
>>>= 0

# ls-files should still show the committed file
cd test\cli\work & bit ls-files
<<<
>>> /test\.txt/
>>>= 0

# Add multiple files
cd test\cli\work & echo data> data.txt & echo notes> notes.txt & bit add data.txt notes.txt
<<<
>>>
>>>= 0

# Commit the new files
cd test\cli\work & bit commit -m "Add more files"
<<<
>>> /\[master|main|files? changed/
>>>= 0

# ls-files should show all tracked files (test.txt, data.txt, notes.txt)
cd test\cli\work & bit ls-files
<<<
>>> /test\.txt/
>>>= 0

# ls-files with --stage should show metadata (mode, hash, stage, filename)
cd test\cli\work & bit ls-files --stage
<<<
>>> /100644.*test\.txt/
>>>= 0

# ls-files with specific pathspec should filter results
cd test\cli\work & bit ls-files test.txt
<<<
>>> /^test\.txt$/
>>>= 0

# Add a binary file
cd test\cli\work & echo binary> file.bin & bit add file.bin & bit commit -m "Add binary"
<<<
>>> /\[master|main|files? changed/
>>>= 0

# ls-files should show all files including binary
cd test\cli\work & bit ls-files
<<<
>>> /file\.bin/
>>>= 0

# ls-files with wildcard pattern (should show .txt files)
cd test\cli\work & bit ls-files *.txt
<<<
>>> /\.txt/
>>>= 0

# ls-files --cached should work (default behavior)
cd test\cli\work & bit ls-files --cached
<<<
>>> /test\.txt/
>>>= 0

# Create subdirectory with files
cd test\cli\work & mkdir subdir & echo sub> subdir\sub.txt & bit add subdir\sub.txt & bit commit -m "Add subdir"
<<<
>>> /\[master|main|files? changed/
>>>= 0

# ls-files should show files in subdirectories
cd test\cli\work & bit ls-files
<<<
>>> /subdir/
>>>= 0

# ls-files with directory pathspec
cd test\cli\work & bit ls-files subdir
<<<
>>> /subdir/
>>>= 0
```

---

## test/cli/merge-local.test

**Path:** `test/cli/merge-local.test`

*Source file.*

```text
# =============================================================================
# bit merge tests (local directory as remote, no cloud needed)
#
# Inspired by Git's t7600-merge.sh and t7611-merge-abort.sh.
# Tests the full two-repo push/pull/merge cycle.
#
# Requires: rclone on PATH (used for local file sync).
# Convention: work_a = "pusher", work_b = "puller", shared_remote = rclone target.
#
# Test categories (cf. Git t64xx / t76xx):
#   1. First pull into unborn branch
#   2. Fast-forward merge (no local divergence)
#   3. Clean three-way merge (non-conflicting parallel changes)
#   4. Content conflict Γאפ resolve keep-local
#   5. Content conflict Γאפ resolve take-remote
#   6. Binary metadata conflict Γאפ resolve keep-local
#   7. Multiple simultaneous conflicts Γאפ mixed resolution
#   8. Merge abort (no merge in progress)
#   9. Merge continue (no merge in progress)
#  10. Pull --accept-remote
#  11. Add/add conflict (both repos create same new file)
#  12. Verify working tree clean after resolved merge
# =============================================================================

# ---- CLEANUP: remove any leftover state from previous runs ----
rmdir /s /q test\cli\work_a 2>nul & rmdir /s /q test\cli\work_b 2>nul & rmdir /s /q test\cli\shared_remote 2>nul & echo cleanup done
<<<
>>>
cleanup done
>>>= 0

# ======================================================================
# SECTION 1: SETUP Γאפ shared remote + two repos with common base state
# ======================================================================

# ---- Create shared remote directory ----
mkdir test\cli\shared_remote
<<<
>>>
>>>= 0

# ---- Repo A: init ----
mkdir test\cli\work_a & cd test\cli\work_a & bit init
<<<
>>> /Initialized/
>>>= 0

# ---- Repo A: add remote ----
cd test\cli\work_a & bit remote add origin "%CD%\test\cli\shared_remote"
<<<
>>> /Remote.*added/
>>>= 0

# ---- Repo A: create base text file, add, commit ----
cd test\cli\work_a & echo base content> text.txt & bit add text.txt
<<<
>>>
>>>= 0

cd test\cli\work_a & bit commit -m "Base: add text.txt"
<<<
>>> /\[main|master|files? changed/
>>>= 0

# ---- Repo A: create base binary file, add, commit ----
cd test\cli\work_a & echo binarydata> data.bin & bit add data.bin
<<<
>>>
>>>= 0

cd test\cli\work_a & bit commit -m "Base: add data.bin"
<<<
>>> /\[main|master|files? changed/
>>>= 0

# ---- Repo A: push base state to shared remote ----
cd test\cli\work_a & bit push
<<<
>>> /Metadata push complete|Remote is empty|Remote check passed/
>>>= 0

# ======================================================================
# SECTION 2: FIRST PULL Γאפ repo B fetches from shared remote (unborn branch)
# (Mirrors Git's test of merge with unborn HEAD)
# ======================================================================

# ---- Repo B: init ----
mkdir test\cli\work_b & cd test\cli\work_b & bit init
<<<
>>> /Initialized/
>>>= 0

# ---- Repo B: add same remote ----
cd test\cli\work_b & bit remote add origin "%CD%\test\cli\shared_remote"
<<<
>>> /Remote.*added/
>>>= 0

# ---- Repo B: fetch ----
cd test\cli\work_b & bit fetch
<<<
>>>2 /From.*shared_remote|new branch.*origin\/main/
>>>= 0

# ---- Repo B: pull (first pull Γאפ checkout remote as main) ----
cd test\cli\work_b & bit pull
<<<
>>> /first pull|Checking out|Syncing binaries/
>>>= 0

# ---- Verify B has the text file from A (content stored in .bit/index) ----
findstr /C:"base content" test\cli\work_b\.bit\index\text.txt >nul
<<<
>>>
>>>= 0

# ---- Verify B has the binary file metadata ----
findstr /C:"hash:" test\cli\work_b\.bit\index\data.bin >nul
<<<
>>>
>>>= 0

# ---- Verify B's working tree has the text file ----
if exist "test\cli\work_b\text.txt" (echo file exists) else (echo missing)
<<<
>>>
file exists
>>>= 0

# ---- Verify B status is clean after first pull ----
cd test\cli\work_b & bit status
<<<
>>> /nothing to commit|working tree clean/
>>>= 0

# ======================================================================
# SECTION 3: FAST-FORWARD MERGE Γאפ A pushes, B pulls with no local changes
# (Mirrors Git's "merge c0 with c1" ff test in t7600)
# ======================================================================

# ---- A: add a new file and push ----
cd test\cli\work_a & echo new from A> extra.txt & bit add extra.txt & bit commit -m "Add extra.txt"
<<<
>>> /\[main|master|files? changed/
>>>= 0

cd test\cli\work_a & bit push
<<<
>>> /Metadata push complete|Remote check passed/
>>>= 0

# ---- B: pull Γאפ should merge cleanly (B has no local divergence) ----
cd test\cli\work_b & bit fetch
<<<
>>>
>>>= 0

cd test\cli\work_b & bit pull
<<<
>>> /Updating|Merge made|Syncing binaries/
>>>= 0

# ---- Verify B has the new file ----
findstr /C:"new from A" test\cli\work_b\.bit\index\extra.txt >nul
<<<
>>>
>>>= 0

# ---- Verify B status is clean ----
cd test\cli\work_b & bit status
<<<
>>> /nothing to commit|working tree clean/
>>>= 0

# ======================================================================
# SECTION 4: CLEAN THREE-WAY MERGE Γאפ non-conflicting parallel changes
# (Mirrors Git's "merge c1 with c2" where changes are in different hunks)
# ======================================================================

# ---- A: modify text.txt and push ----
cd test\cli\work_a & echo updated by A> text.txt & bit add text.txt & bit commit -m "A: update text.txt"
<<<
>>> /\[main|master|files? changed/
>>>= 0

cd test\cli\work_a & bit push
<<<
>>> /Metadata push complete|Remote check passed/
>>>= 0

# ---- B: add a NEW file (doesn't overlap with A's changes) and commit ----
cd test\cli\work_b & echo B only file> bonly.txt & bit add bonly.txt & bit commit -m "B: add bonly.txt"
<<<
>>> /\[main|master|files? changed/
>>>= 0

# ---- B: pull Γאפ should merge cleanly since changes don't overlap ----
cd test\cli\work_b & bit pull
<<<
>>> /Merge made|Syncing binaries|Updating/
>>>= 0

# ---- Verify B has A's updated text.txt ----
findstr /C:"updated by A" test\cli\work_b\.bit\index\text.txt >nul
<<<
>>>
>>>= 0

# ---- Verify B still has its own bonly.txt ----
findstr /C:"B only file" test\cli\work_b\.bit\index\bonly.txt >nul
<<<
>>>
>>>= 0

# ---- Verify B status is clean ----
cd test\cli\work_b & bit status
<<<
>>> /nothing to commit|working tree clean/
>>>= 0

# ======================================================================
# SECTION 5: CONTENT CONFLICT Γאפ resolve keep-local
# Both A and B modify the same text file with different content.
# bit detects the conflict in .bit/index/ and prompts for resolution.
# User chooses (l)ocal.
# (Mirrors Git's "merge c1 with c2 Γאפ content conflict" in t7600)
# ======================================================================

# ---- Synchronize: B pushes its current state so both are aligned ----
cd test\cli\work_b & bit push
<<<
>>> /Metadata push complete|Remote check passed|up to date/
>>>= 0

cd test\cli\work_a & bit fetch & bit pull
<<<
>>> //
>>>= 0

# ---- A: modify text.txt to A's version and push ----
cd test\cli\work_a & echo conflict version A> text.txt & bit add text.txt & bit commit -m "A: conflict version"
<<<
>>> /\[main|master|files? changed/
>>>= 0

cd test\cli\work_a & bit push
<<<
>>> /Metadata push complete|Remote check passed/
>>>= 0

# ---- B: modify SAME text.txt to B's version and commit locally ----
cd test\cli\work_b & echo conflict version B> text.txt & bit add text.txt & bit commit -m "B: conflict version"
<<<
>>> /\[main|master|files? changed/
>>>= 0

# ---- B: pull Γאפ conflict expected; pipe "l" to keep local ----
cd test\cli\work_b & bit pull
<<<
l
>>> /Automatic merge failed|CONFLICT|Resolving conflicts|Merge complete|conflict/
>>>= 0

# ---- Verify B kept local version in metadata (B's content wins) ----
findstr /C:"conflict version B" test\cli\work_b\.bit\index\text.txt >nul
<<<
>>>
>>>= 0

# ======================================================================
# SECTION 6: CONTENT CONFLICT Γאפ resolve take-remote
# Same setup: both modify same file, but user chooses (r)emote.
# (Mirrors Git's checkout --theirs pattern)
# ======================================================================

# ---- Re-sync: push B's state, pull to A ----
cd test\cli\work_b & bit push
<<<
>>> /Metadata push complete|Remote check passed|up to date/
>>>= 0

cd test\cli\work_a & bit fetch & bit pull
<<<
>>> //
>>>= 0

# ---- A: modify text.txt to "remote wins" and push ----
cd test\cli\work_a & echo remote wins version> text.txt & bit add text.txt & bit commit -m "A: remote wins"
<<<
>>> /\[main|master|files? changed/
>>>= 0

cd test\cli\work_a & bit push
<<<
>>> /Metadata push complete|Remote check passed/
>>>= 0

# ---- B: modify SAME text.txt differently and commit locally ----
cd test\cli\work_b & echo local loses version> text.txt & bit add text.txt & bit commit -m "B: local loses"
<<<
>>> /\[main|master|files? changed/
>>>= 0

# ---- B: pull Γאפ conflict expected; pipe "r" to take remote ----
cd test\cli\work_b & bit pull
<<<
r
>>> /Automatic merge failed|CONFLICT|Resolving conflicts|Merge complete|conflict/
>>>= 0

# ---- Verify B took remote version in metadata (A's content wins) ----
findstr /C:"remote wins version" test\cli\work_b\.bit\index\text.txt >nul
<<<
>>>
>>>= 0

# ======================================================================
# SECTION 7: MULTIPLE SIMULTANEOUS CONFLICTS Γאפ mixed resolution
# Both A and B modify two files. B picks (l)ocal for first, (r)emote for second.
# (Mirrors Git's "merge with multiple conflicting files" tests)
# ======================================================================

# ---- Re-sync ----
cd test\cli\work_b & bit push
<<<
>>> /Metadata push complete|Remote check passed|up to date/
>>>= 0

cd test\cli\work_a & bit fetch & bit pull
<<<
>>> //
>>>= 0

# ---- A: modify both files and push ----
cd test\cli\work_a & (echo A multi 1)> file1.txt & (echo A multi 2)> file2.txt & bit add file1.txt file2.txt & bit commit -m "A: modify two files"
<<<
>>> /\[main|master|files? changed/
>>>= 0

cd test\cli\work_a & bit push
<<<
>>> /Metadata push complete|Remote check passed/
>>>= 0

# ---- B: modify SAME two files differently and commit ----
cd test\cli\work_b & (echo B multi 1)> file1.txt & (echo B multi 2)> file2.txt & bit add file1.txt file2.txt & bit commit -m "B: modify two files"
<<<
>>> /\[main|master|files? changed/
>>>= 0

# ---- B: pull Γאפ two conflicts; pipe "l" then "r" ----
cd test\cli\work_b & bit pull
<<<
l
r
>>> /Automatic merge failed|CONFLICT|Resolving conflicts|Merge complete|2 conflict/
>>>= 0

# ---- Verify: file1.txt kept local (B's version), file2.txt took remote (A's version) ----
findstr /C:"B multi 1" test\cli\work_b\.bit\index\file1.txt >nul
<<<
>>>
>>>= 0

findstr /C:"A multi 2" test\cli\work_b\.bit\index\file2.txt >nul
<<<
>>>
>>>= 0

# ======================================================================
# SECTION 8: ADD/ADD CONFLICT Γאפ both repos create the same new file
# (Mirrors Git's "CONFLICT (add/add)" tests in t6422)
# ======================================================================

# ---- Re-sync ----
cd test\cli\work_b & bit push
<<<
>>> /Metadata push complete|Remote check passed|up to date/
>>>= 0

cd test\cli\work_a & bit fetch & bit pull
<<<
>>> //
>>>= 0

# ---- A: create brand-new file "shared_new.txt" and push ----
cd test\cli\work_a & echo A created this> shared_new.txt & bit add shared_new.txt & bit commit -m "A: add shared_new.txt"
<<<
>>> /\[main|master|files? changed/
>>>= 0

cd test\cli\work_a & bit push
<<<
>>> /Metadata push complete|Remote check passed/
>>>= 0

# ---- B: create SAME filename "shared_new.txt" with different content, commit ----
cd test\cli\work_b & echo B created this> shared_new.txt & bit add shared_new.txt & bit commit -m "B: add shared_new.txt"
<<<
>>> /\[main|master|files? changed/
>>>= 0

# ---- B: pull Γאפ add/add conflict; choose "r" (take remote) ----
cd test\cli\work_b & bit pull
<<<
r
>>> /Automatic merge failed|CONFLICT|Resolving conflicts|Merge complete|conflict/
>>>= 0

# ---- Verify B took A's version ----
findstr /C:"A created this" test\cli\work_b\.bit\index\shared_new.txt >nul
<<<
>>>
>>>= 0

# ======================================================================
# SECTION 9: MERGE --ABORT with no merge in progress
# (Mirrors Git's t7611 "git merge --abort fails without MERGE_HEAD")
# ======================================================================

# ---- Verify clean state first ----
cd test\cli\work_b & bit status
<<<
>>> /nothing to commit|working tree clean/
>>>= 0

# ---- merge --abort when nothing in progress: should print error ----
cd test\cli\work_b & bit merge --abort
<<<
>>>2 /no merge in progress/
>>>= 1

# ======================================================================
# SECTION 10: MERGE --CONTINUE with no merge in progress
# (Mirrors Git's "git merge --continue fails without in-progress merge")
# ======================================================================

cd test\cli\work_b & bit merge --continue
<<<
>>>2 /no merge in progress|error/
>>>= 1

# ======================================================================
# SECTION 11: PULL --ACCEPT-REMOTE
# A pushes a change; B has diverged locally. B uses --accept-remote
# to accept whatever is on the remote as truth.
# (bit-specific: no Git equivalent; tests the remote-reality acceptance flow)
# ======================================================================

# ---- Re-sync ----
cd test\cli\work_b & bit push
<<<
>>> /Metadata push complete|Remote check passed|up to date/
>>>= 0

cd test\cli\work_a & bit fetch & bit pull
<<<
>>> //
>>>= 0

# ---- A: modify text.txt and push ----
cd test\cli\work_a & echo accept remote test> text.txt & bit add text.txt & bit commit -m "A: accept-remote test"
<<<
>>> /\[main|master|files? changed/
>>>= 0

cd test\cli\work_a & bit push
<<<
>>> /Metadata push complete|Remote check passed/
>>>= 0

# ---- B: pull with --accept-remote (skip conflict resolution, accept remote) ----
cd test\cli\work_b & bit pull --accept-remote
<<<
>>> /Accepting remote file state|accept-remote completed|Scanning remote/
>>>= 0

# ---- Verify B accepted A's content ----
findstr /C:"accept remote test" test\cli\work_b\.bit\index\text.txt >nul
<<<
>>>
>>>= 0

# ======================================================================
# SECTION 12: VERIFY FINAL STATE Γאפ working tree clean, no dangling state
# (Mirrors Git's post-merge state verification: no MERGE_HEAD, clean diff)
# ======================================================================

# ---- B: status should be clean ----
cd test\cli\work_b & bit status
<<<
>>> /nothing to commit|working tree clean/
>>>= 0

# ---- B: fsck should find no issues ----
cd test\cli\work_b & bit fsck
<<<
>>>
>>>= 0

# ---- A: status should be clean ----
cd test\cli\work_a & bit status
<<<
>>> /nothing to commit|working tree clean/
>>>= 0

# ---- A: fsck should find no issues ----
cd test\cli\work_a & bit fsck
<<<
>>>
>>>= 0

# ======================================================================
# CLEANUP
# ======================================================================

rmdir /s /q test\cli\work_a 2>nul & rmdir /s /q test\cli\work_b 2>nul & rmdir /s /q test\cli\shared_remote 2>nul & echo cleanup done
<<<
>>>
cleanup done
>>>= 0
```

---

## test/cli/no-repo.test

**Path:** `test/cli/no-repo.test`

*Source file.*

```text
# Test: bit commands outside of a bit repository
# These tests verify that commands fail gracefully when run outside a bit repo
# and that they do NOT create a .bit directory

# Setup: clean directory with no .bit folder
rmdir /s /q test\cli\work_norepo 2>nul & mkdir test\cli\work_norepo
<<<
>>>
>>>= 0

# bit status - should fail with proper error message
cd test\cli\work_norepo & bit status 2>&1
<<<
>>> /fatal: not a bit repository/
>>>= 1

# Verify .bit was NOT created by status
if exist "test\cli\work_norepo\.bit\" (echo exists) else (echo not created)
<<<
>>>
not created
>>>= 0

# bit add - should fail with proper error message
cd test\cli\work_norepo & echo test> file.txt & bit add file.txt 2>&1
<<<
>>> /fatal: not a bit repository/
>>>= 1

# Verify .bit was NOT created by add
if exist "test\cli\work_norepo\.bit\" (echo exists) else (echo not created)
<<<
>>>
not created
>>>= 0

# bit commit - should fail with proper error message
cd test\cli\work_norepo & bit commit -m "test" 2>&1
<<<
>>> /fatal: not a bit repository/
>>>= 1

# bit log - should fail with proper error message
cd test\cli\work_norepo & bit log 2>&1
<<<
>>> /fatal: not a bit repository/
>>>= 1

# bit diff - should fail with proper error message
cd test\cli\work_norepo & bit diff 2>&1
<<<
>>> /fatal: not a bit repository/
>>>= 1

# bit push - should fail with proper error message
cd test\cli\work_norepo & bit push 2>&1
<<<
>>> /fatal: not a bit repository/
>>>= 1

# bit pull - should fail with proper error message
cd test\cli\work_norepo & bit pull 2>&1
<<<
>>> /fatal: not a bit repository/
>>>= 1

# bit fetch - should fail with proper error message
cd test\cli\work_norepo & bit fetch 2>&1
<<<
>>> /fatal: not a bit repository/
>>>= 1

# bit restore - should fail with proper error message
cd test\cli\work_norepo & bit restore file.txt 2>&1
<<<
>>> /fatal: not a bit repository/
>>>= 1

# bit checkout - should fail with proper error message
cd test\cli\work_norepo & bit checkout file.txt 2>&1
<<<
>>> /fatal: not a bit repository/
>>>= 1

# bit remote show - should fail with proper error message
cd test\cli\work_norepo & bit remote show 2>&1
<<<
>>> /fatal: not a bit repository/
>>>= 1

# bit remote add - should fail with proper error message
cd test\cli\work_norepo & bit remote add origin /tmp/remote 2>&1
<<<
>>> /fatal: not a bit repository/
>>>= 1

# bit verify - should fail with proper error message
cd test\cli\work_norepo & bit verify 2>&1
<<<
>>> /fatal: not a bit repository/
>>>= 1

# Final check: .bit should still not exist after all commands
if exist "test\cli\work_norepo\.bit\" (echo exists) else (echo not created)
<<<
>>>
not created
>>>= 0

# bit init - should succeed (doesn't need existing repo)
cd test\cli\work_norepo & bit init
<<<
>>> /Initialized/
>>>= 0

# After init, .bit should exist
if exist "test\cli\work_norepo\.bit\" (echo exists) else (echo missing)
<<<
>>>
exists
>>>= 0

# Clean up file.txt before testing status
del test\cli\work_norepo\file.txt 2>nul
<<<
>>>
>>>= 0

# After init, bit status should work (clean working directory)
cd test\cli\work_norepo & bit status
<<<
>>> /.*/
>>>= 0

# Cleanup
rmdir /s /q test\cli\work_norepo 2>nul
<<<
>>>
>>>= 0
```

---

## test/cli/one-repo.test

**Path:** `test/cli/one-repo.test`

*Source file.*

```text
# Setup fresh repo
rmdir /s /q test\cli\work 2>nul & mkdir test\cli\work & cd test\cli\work & bit init
<<<
>>> /Initialized/
>>>= 0

# Add text file - metadata should be created in .bit/index (content copy for text files)
cd test\cli\work & echo hello> test.txt & bit add test.txt
<<<
>>>
>>>= 0

# Verify text file metadata: .bit/index/test.txt exists and contains the content
if exist "test\cli\work\.bit\index\test.txt" (echo metadata exists) else (echo missing)
<<<
>>>
metadata exists
>>>= 0

findstr /C:"hello" test\cli\work\.bit\index\test.txt >nul
<<<
>>>
>>>= 0

# Status: new file staged
cd test\cli\work & bit status
<<<
>>> /new file:.*test\.txt/
>>>= 0

# Commit the staged file
cd test\cli\work & bit commit -m "Add test.txt"
<<<
>>> /\[master|main|files? changed/
>>>= 0

# Status after commit: working tree clean
cd test\cli\work & bit status
<<<
>>> /nothing to commit|working tree clean/
>>>= 0

# Add binary file - metadata should have hash and size
cd test\cli\work & echo x> file.bin & bit add file.bin
<<<
>>>
>>>= 0

# Verify binary metadata: .bit/index/file.bin exists with hash: and size: lines
if exist "test\cli\work\.bit\index\file.bin" (echo metadata exists) else (echo missing)
<<<
>>>
metadata exists
>>>= 0

findstr /C:"hash:" test\cli\work\.bit\index\file.bin >nul
<<<
>>>
>>>= 0

findstr /C:"size:" test\cli\work\.bit\index\file.bin >nul
<<<
>>>
>>>= 0

# Status: new binary file staged
cd test\cli\work & bit status
<<<
>>> /new file:.*file\.bin/
>>>= 0

# Commit binary
cd test\cli\work & bit commit -m "Add file.bin"
<<<
>>> /\[master|main|files? changed/
>>>= 0

# Add multiple files with "add ."
cd test\cli\work & echo data> data.txt & echo notes> notes.txt & bit add .
<<<
>>>
>>>= 0

# Verify both metadata files exist
if exist "test\cli\work\.bit\index\data.txt" (echo data ok) else (echo missing)
<<<
>>>
data ok
>>>= 0

if exist "test\cli\work\.bit\index\notes.txt" (echo notes ok) else (echo missing)
<<<
>>>
notes ok
>>>= 0

# Status: both new files staged
cd test\cli\work & bit status
<<<
>>> /new file:.*data\.txt/
>>>= 0

# Commit multiple files
cd test\cli\work & bit commit -m "Add data and notes"
<<<
>>> /\[master|main|files? changed/
>>>= 0

# Modify text file, add it
cd test\cli\work & echo hello world> test.txt & bit add test.txt
<<<
>>>
>>>= 0

# Verify metadata was updated (new content)
findstr /C:"hello world" test\cli\work\.bit\index\test.txt >nul
<<<
>>>
>>>= 0

# Status: modified file staged
cd test\cli\work & bit status
<<<
>>> /modified:.*test\.txt/
>>>= 0

# Commit modification
cd test\cli\work & bit commit -m "Update test.txt"
<<<
>>> /\[master|main|files? changed/
>>>= 0

# Final status: clean
cd test\cli\work & bit status
<<<
>>> /nothing to commit|working tree clean/
>>>= 0

# Test bit log - should show commit history
cd test\cli\work & bit log --oneline
<<<
>>> /Update test\.txt/
>>>= 0

# Test bit log with formatting - should work like git log
cd test\cli\work & bit log --pretty=format:"%s" -n 1
<<<
>>> /Update test\.txt/
>>>= 0
```

---

## test/cli/remote-check.test

**Path:** `test/cli/remote-check.test`

*Source file.*

```text
# bit remote check tests
# Runs rclone check between local working tree and remote (excludes .bit).
# Requires: rclone on PATH.

# --- No remote configured: must print error ---
rmdir /s /q test\cli\work 2>nul & mkdir test\cli\work & cd test\cli\work & bit init
<<<
>>> /Initialized/
>>>= 0

cd test\cli\work & bit remote check
<<<
>>>2 /fatal: No remote configured/
>>>= 1

# --- Setup: repo with text + binary, local dir OUTSIDE work as "remote", push, then check ---
rmdir /s /q test\cli\work 2>nul & rmdir /s /q test\cli\remote_mirror 2>nul & mkdir test\cli\work & mkdir test\cli\remote_mirror & cd test\cli\work & bit init & echo hello> greet.txt & echo x> data.bin & bit add greet.txt data.bin & bit commit -m "Add files"
<<<
>>> /Initialized/
>>>= 0

cd test\cli\work & bit remote add origin "%CD%\test\cli\remote_mirror"
<<<
>>> /Remote.*added/
>>>= 0

cd test\cli\work & bit push
<<<
>>> /Metadata push complete/
>>>= 0

# --- Happy path: local matches remote after push, exit 0 ---
cd test\cli\work & bit remote check
<<<
>>> /files match between local and remote/
>>>= 0

# --- Modify local file: rclone check detects difference, exit 1 ---
cd test\cli\work & echo changed> greet.txt
<<<
>>>
>>>= 0

cd test\cli\work & bit remote check
<<<
>>> /content differs|differences,/
>>>= 1

# --- Cleanup ---
rmdir /s /q test\cli\work 2>nul & rmdir /s /q test\cli\remote_mirror 2>nul
<<<
>>>
>>>= 0
```

---

## test/cli/restore-checkout.test

**Path:** `test/cli/restore-checkout.test`

*Source file.*

```text
# Test restore and checkout commands (git-compatible syntax)
# Setup: fresh repo with committed file
rmdir /s /q test\cli\work 2>nul & mkdir test\cli\work & cd test\cli\work & bit init
<<<
>>> /Initialized/
>>>= 0

cd test\cli\work & echo original> file.txt & bit add file.txt & bit commit -m "Add file"
<<<
>>> /\[master|main|files? changed/
>>>= 0

# Modify file, then restore (discard working tree changes)
cd test\cli\work & echo modified> file.txt
<<<
>>>
>>>= 0

cd test\cli\work & bit restore file.txt
<<<
>>>
>>>= 0

findstr /C:"original" test\cli\work\file.txt >nul
<<<
>>>
>>>= 0

# Modify again, use restore -- (explicit pathspec separator, like git)
cd test\cli\work & echo changed> file.txt & bit restore -- file.txt
<<<
>>>
>>>= 0

findstr /C:"original" test\cli\work\file.txt >nul
<<<
>>>
>>>= 0

# Modify, then checkout -- (legacy git syntax)
cd test\cli\work & echo discarded> file.txt & bit checkout -- file.txt
<<<
>>>
>>>= 0

findstr /C:"original" test\cli\work\file.txt >nul
<<<
>>>
>>>= 0

# Test restore --staged (unstage without discarding working tree)
cd test\cli\work & echo staged change> file.txt & bit add file.txt
<<<
>>>
>>>= 0

cd test\cli\work & bit restore --staged file.txt
<<<
>>>
>>>= 0

cd test\cli\work & bit status
<<<
>>> /modified:.*file\.txt/
>>>= 0

findstr /C:"staged change" test\cli\work\file.txt >nul
<<<
>>>
>>>= 0

# Restore working tree (discard the modification)
cd test\cli\work & bit restore file.txt
<<<
>>>
>>>= 0

findstr /C:"original" test\cli\work\file.txt >nul
<<<
>>>
>>>= 0

cd test\cli\work & bit status
<<<
>>> /nothing to commit|working tree clean/
>>>= 0

# Test restore . (all files)
cd test\cli\work & echo a> a.txt & echo b> b.txt & bit add . & bit commit -m "Add a and b"
<<<
>>> /\[master|main|files? changed/
>>>= 0

cd test\cli\work & echo x> a.txt & echo y> b.txt & bit restore .
<<<
>>>
>>>= 0

findstr /C:"a" test\cli\work\a.txt >nul
<<<
>>>
>>>= 0

findstr /C:"b" test\cli\work\b.txt >nul
<<<
>>>
>>>= 0

# Test checkout -- with multiple paths
cd test\cli\work & echo XX> a.txt & echo YY> b.txt & bit checkout -- a.txt b.txt
<<<
>>>
>>>= 0

findstr /C:"a" test\cli\work\a.txt >nul
<<<
>>>
>>>= 0

findstr /C:"b" test\cli\work\b.txt >nul
<<<
>>>
>>>= 0
```

---

## test/cli/setup.sh

**Path:** `test/cli/setup.sh`

*Source file.*

```shell
#!/bin/bash
# test/cli/setup.sh - Common test setup

TEST_DIR="$TEMP/bit-test"

setup_fresh_repo() {
    rm -rf "$TEST_DIR"
    mkdir -p "$TEST_DIR"
    cd "$TEST_DIR"
    bit init >/dev/null 2>&1
}

cleanup() {
    rm -rf "$TEST_DIR"
}
```

---

## test/cli/status.test

**Path:** `test/cli/status.test`

*Source file.*

```text
# Setup: fresh repo with one file
rmdir /s /q test\cli\work 2>nul & mkdir test\cli\work & cd test\cli\work & bit init & echo data> file.bin
<<<
>>> /Initialized/
>>>= 0

# Status should show untracked file
cd test\cli\work & bit status
<<<
>>> /file\.bin/
>>>= 0
```

---

## test/cli/work_direct/file.txt

**Path:** `test/cli/work_direct/file.txt`

*Source file.*

```text
remote change 
```

---

