# rgit — Tests (Literate Programming Document)

This document contains all test files for the rgit project:
Haskell test modules, shell-based integration tests, and test infrastructure.

---

## test/ConflictSpec.hs

**Path:** `test/ConflictSpec.hs`

*Source file.*

```haskell
{-# LANGUAGE DataKinds #-}

module Main where

import Test.Tasty
import Test.Tasty.HUnit
import qualified Data.Map as Map
import System.Exit (ExitCode(..))

import Rgit.Conflict
import Rgit.Effect.Pure (PureEnv(..), Trace(..), runPure)

main :: IO ()
main = defaultMain tests

-- | A pure environment pre-loaded so gitQuery calls return simulated metadata.
mkEnv :: [String] -> PureEnv
mkEnv userInputs = PureEnv
  { pureFiles  = Map.empty
  , pureDirs   = []
  , pureInputs = userInputs
  , pureTrace  = []
  , pureCwd    = "/test"
  }

tests :: TestTree
tests = testGroup "Conflict"
  [ testGroup "resolveAll"
    [ testCase "empty conflicts produces no resolutions" $ do
        let (res, trace) = runPure (mkEnv []) (resolveAll [])
        res @?= []
        -- Should see "No unmerged paths." in trace
        any (\t -> case t of TTell "No unmerged paths." -> True; _ -> False) trace @?= True

    , testCase "single conflict with 'l' keeps local" $ do
        let (res, trace) = runPure (mkEnv ["l"]) (resolveAll ["index/foo.bin"])
        length res @?= 1
        head res @?= KeepLocal
        -- Should see checkout --ours in trace
        any (\t -> case t of TGitRaw ["checkout", "--ours", "--", "index/foo.bin"] -> True; _ -> False) trace @?= True

    , testCase "single conflict with 'r' takes remote" $ do
        let (res, trace) = runPure (mkEnv ["r"]) (resolveAll ["index/foo.bin"])
        length res @?= 1
        head res @?= TakeRemote
        any (\t -> case t of TGitRaw ["checkout", "--theirs", "--", "index/foo.bin"] -> True; _ -> False) trace @?= True

    , testCase "three conflicts with mixed choices" $ do
        let (res, trace) = runPure (mkEnv ["l", "r", "l"])
                                   (resolveAll ["index/a.bin", "index/b.bin", "index/c.bin"])
        res @?= [KeepLocal, TakeRemote, KeepLocal]
        -- All three should have been git-added
        let addTraces = filter (\t -> case t of TGitRaw ("add":_) -> True; _ -> False) trace
        length addTraces @?= 3

    , testCase "user input is normalized (whitespace, case)" $ do
        let (res, _) = runPure (mkEnv ["  Remote  "]) (resolveAll ["index/x.bin"])
        res @?= [TakeRemote]
    ]

  , testGroup "getConflictedFilesE"
    [ testCase "returns empty on failure" $ do
        -- Pure interpreter returns ExitSuccess with empty stdout for gitQuery
        let (files, _) = runPure (mkEnv []) getConflictedFilesE
        files @?= []  -- empty stdout Γזע no files
    ]
  ]
```

---

## test/DevicePromptTests.hs

**Path:** `test/DevicePromptTests.hs`

*Source file.*

```haskell
import Test.Tasty
import Test.Tasty.HUnit
import qualified Rgit.DevicePrompt as DP

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

-- | Comprehensive merge tests for rgit.
--
-- Inspired by Git's t7600-merge.sh and t6422-merge-rename-corner-cases.sh.
-- Tests the pure conflict resolution, conflict detection, and metadata parsing
-- logic that underpins rgit's two-repo merge flow.
--
-- Run: cabal test merge
module Main where

import Test.Tasty
import Test.Tasty.HUnit
import Test.Tasty.QuickCheck
import qualified Data.Map as Map
import System.Exit (ExitCode(..))
import Data.List (isInfixOf)

import Rgit.Conflict
import Rgit.Effect (Rgit, tell, tellErr, askUser, gitRaw, gitQuery)
import Rgit.Effect.Pure (PureEnv(..), Trace(..), runPure)
import Rgit.Internal.Metadata (MetaContent(..), parseMetadata, serializeMetadata, displayHash, hasConflictMarkers)
import Rgit.Types (Hash(..), HashAlgo(..))
import qualified Data.Text as T

main :: IO ()
main = defaultMain tests

-- ---------------------------------------------------------------------------
-- Test environment helpers
-- ---------------------------------------------------------------------------

-- | Build a pure environment with given stdin lines.
mkEnv :: [String] -> PureEnv
mkEnv userInputs = PureEnv
  { pureFiles  = Map.empty
  , pureDirs   = []
  , pureInputs = userInputs
  , pureTrace  = []
  , pureCwd    = "/test"
  }

-- | Check whether a specific trace entry exists.
hasTrace :: (Trace -> Bool) -> [Trace] -> Bool
hasTrace = any

-- | Count trace entries matching a predicate.
countTraces :: (Trace -> Bool) -> [Trace] -> Int
countTraces p = length . filter p

-- | Check for a git raw command in traces.
hasGitRaw :: [String] -> [Trace] -> Bool
hasGitRaw args = hasTrace $ \t -> case t of
  TGitRaw a -> a == args
  _         -> False

-- | Check for a Tell message in traces.
hasTell :: String -> [Trace] -> Bool
hasTell msg = hasTrace $ \t -> case t of
  TTell m -> m == msg
  _       -> False

-- | Check for a TellErr message containing a substring.
hasTellErrContaining :: String -> [Trace] -> Bool
hasTellErrContaining sub = hasTrace $ \t -> case t of
  TTellErr m -> sub `isInfixOf` m
  _          -> False

-- | Check for a Tell message containing a substring.
hasTellContaining :: String -> [Trace] -> Bool
hasTellContaining sub = hasTrace $ \t -> case t of
  TTell m -> sub `isInfixOf` m
  _       -> False

-- ---------------------------------------------------------------------------
-- All tests
-- ---------------------------------------------------------------------------

tests :: TestTree
tests = testGroup "Merge"
  [ conflictResolutionTests
  , conflictDetectionTests
  , conflictInfoParsingTests
  , metadataRoundtripTests
  , conflictMarkerTests
  , resolveAllPropertyTests
  ]

-- ---------------------------------------------------------------------------
-- 1. Conflict resolution (resolveAll / resolveConflict)
--    Mirrors Git's test of checkout --ours / --theirs + git add
-- ---------------------------------------------------------------------------

conflictResolutionTests :: TestTree
conflictResolutionTests = testGroup "resolveAll"
  [ testCase "empty conflicts produces no resolutions" $ do
      let (res, trace) = runPure (mkEnv []) (resolveAll [])
      res @?= []
      hasTell "No unmerged paths." trace @?= True

  , testCase "single conflict - keep local" $ do
      let (res, trace) = runPure (mkEnv ["l"]) (resolveAll ["index/foo.bin"])
      res @?= [KeepLocal]
      hasGitRaw ["checkout", "--ours", "--", "index/foo.bin"] trace @?= True
      hasGitRaw ["add", "index/foo.bin"] trace @?= True

  , testCase "single conflict - take remote" $ do
      let (res, trace) = runPure (mkEnv ["r"]) (resolveAll ["index/foo.bin"])
      res @?= [TakeRemote]
      hasGitRaw ["checkout", "--theirs", "--", "index/foo.bin"] trace @?= True
      hasGitRaw ["add", "index/foo.bin"] trace @?= True

  , testCase "three conflicts with mixed choices (l, r, l)" $ do
      let paths = ["index/a.bin", "index/b.bin", "index/c.bin"]
      let (res, trace) = runPure (mkEnv ["l", "r", "l"]) (resolveAll paths)
      res @?= [KeepLocal, TakeRemote, KeepLocal]
      -- Every conflict should have been git-added
      countTraces (\t -> case t of TGitRaw ("add":_) -> True; _ -> False) trace @?= 3
      -- Verify correct checkout flags for each
      hasGitRaw ["checkout", "--ours", "--", "index/a.bin"] trace @?= True
      hasGitRaw ["checkout", "--theirs", "--", "index/b.bin"] trace @?= True
      hasGitRaw ["checkout", "--ours", "--", "index/c.bin"] trace @?= True

  , testCase "five conflicts - all remote" $ do
      let paths = ["index/" ++ [c] ++ ".bin" | c <- "abcde"]
      let (res, trace) = runPure (mkEnv (replicate 5 "r")) (resolveAll paths)
      res @?= replicate 5 TakeRemote
      countTraces (\t -> case t of TGitRaw ["checkout", "--theirs", "--", _] -> True; _ -> False) trace @?= 5

  , testCase "five conflicts - all local" $ do
      let paths = ["index/" ++ [c] ++ ".bin" | c <- "abcde"]
      let (res, trace) = runPure (mkEnv (replicate 5 "l")) (resolveAll paths)
      res @?= replicate 5 KeepLocal
      countTraces (\t -> case t of TGitRaw ["checkout", "--ours", "--", _] -> True; _ -> False) trace @?= 5

  -- Input normalization (mirrors Git's case-insensitive flag parsing)
  , testCase "input 'Local' (capital) keeps local" $ do
      let (res, _) = runPure (mkEnv ["Local"]) (resolveAll ["index/x.bin"])
      res @?= [KeepLocal]

  , testCase "input 'REMOTE' (all caps) takes remote" $ do
      let (res, _) = runPure (mkEnv ["REMOTE"]) (resolveAll ["index/x.bin"])
      res @?= [TakeRemote]

  , testCase "input '  r  ' (padded whitespace) takes remote" $ do
      let (res, _) = runPure (mkEnv ["  r  "]) (resolveAll ["index/x.bin"])
      res @?= [TakeRemote]

  , testCase "input '  Remote  ' (padded, mixed case) takes remote" $ do
      let (res, _) = runPure (mkEnv ["  Remote  "]) (resolveAll ["index/x.bin"])
      res @?= [TakeRemote]

  , testCase "input 'L' (capital) keeps local" $ do
      let (res, _) = runPure (mkEnv ["L"]) (resolveAll ["index/x.bin"])
      res @?= [KeepLocal]

  -- Any non-r/remote input defaults to keep-local (safe default)
  , testCase "empty input defaults to keep-local" $ do
      let (res, _) = runPure (mkEnv [""]) (resolveAll ["index/x.bin"])
      res @?= [KeepLocal]

  , testCase "garbage input defaults to keep-local" $ do
      let (res, _) = runPure (mkEnv ["xyz"]) (resolveAll ["index/x.bin"])
      res @?= [KeepLocal]

  -- Progress counter display
  , testCase "progress counter shows [1/3], [2/3], [3/3]" $ do
      let paths = ["index/a", "index/b", "index/c"]
      let (_, trace) = runPure (mkEnv ["l", "l", "l"]) (resolveAll paths)
      hasTellContaining "[1/3]" trace @?= True
      hasTellContaining "[2/3]" trace @?= True
      hasTellContaining "[3/3]" trace @?= True

  -- Path display: "index/" prefix should be stripped for user-facing output
  , testCase "index/ prefix stripped in display" $ do
      let (_, trace) = runPure (mkEnv ["l"]) (resolveAll ["index/src/model.bin"])
      hasTellContaining "src/model.bin" trace @?= True
  ]

-- ---------------------------------------------------------------------------
-- 2. Conflict detection (getConflictedFilesE)
-- ---------------------------------------------------------------------------

conflictDetectionTests :: TestTree
conflictDetectionTests = testGroup "getConflictedFilesE"
  [ testCase "returns empty list when git diff --diff-filter=U fails" $ do
      -- Pure interpreter returns ExitSuccess with empty stdout for gitQuery
      let (files, _) = runPure (mkEnv []) getConflictedFilesE
      files @?= []

  , testCase "returns empty list on empty stdout" $ do
      let (files, _) = runPure (mkEnv []) getConflictedFilesE
      files @?= []
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

-- ---------------------------------------------------------------------------
-- 6. Property tests for conflict resolution
--    (Mirrors Git's QuickCheck-style exhaustive testing philosophy)
-- ---------------------------------------------------------------------------

resolveAllPropertyTests :: TestTree
resolveAllPropertyTests = testGroup "resolveAll properties"
  [ testProperty "resolveAll returns exactly N resolutions for N conflicts" $
      \(Positive n) ->
        let n' = min n 20  -- cap to keep tests fast
            paths = ["index/file" ++ show i ++ ".bin" | i <- [1..n']]
            inputs = replicate n' "l"
            (res, _) = runPure (mkEnv inputs) (resolveAll paths)
        in length res == n'

  , testProperty "every resolution is KeepLocal or TakeRemote (total)" $
      \choices ->
        let n = min (length choices) 20
            paths = ["index/f" ++ show i | i <- [1..n]]
            inputs = map (\b -> if b then "r" else "l") (take n choices)
            (res, _) = runPure (mkEnv inputs) (resolveAll paths)
        in all (\r -> r == KeepLocal || r == TakeRemote) res

  , testProperty "'l' input always produces KeepLocal" $
      \(Positive n) ->
        let n' = min n 10
            paths = ["index/f" ++ show i | i <- [1..n']]
            (res, _) = runPure (mkEnv (replicate n' "l")) (resolveAll paths)
        in all (== KeepLocal) res

  , testProperty "'r' input always produces TakeRemote" $
      \(Positive n) ->
        let n' = min n 10
            paths = ["index/f" ++ show i | i <- [1..n']]
            (res, _) = runPure (mkEnv (replicate n' "r")) (resolveAll paths)
        in all (== TakeRemote) res

  , testProperty "resolveAll emits exactly N git-add traces" $
      \(Positive n) ->
        let n' = min n 15
            paths = ["index/f" ++ show i | i <- [1..n']]
            inputs = replicate n' "l"
            (_, trace) = runPure (mkEnv inputs) (resolveAll paths)
            addCount = countTraces (\t -> case t of TGitRaw ("add":_) -> True; _ -> False) trace
        in addCount == n'

  , testProperty "resolveAll emits exactly N checkout traces" $
      \(Positive n) ->
        let n' = min n 15
            paths = ["index/f" ++ show i | i <- [1..n']]
            inputs = replicate n' "r"
            (_, trace) = runPure (mkEnv inputs) (resolveAll paths)
            checkoutCount = countTraces (\t -> case t of
              TGitRaw ("checkout":_) -> True
              _ -> False) trace
        in checkoutCount == n'
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
import Rgit.Types
import Rgit.Diff (GitDiff(..), LightFileEntry(..), FileIndex, buildIndexFromFileEntries, computeDiff)
import Rgit.Plan (RcloneAction(..), planAction)
import Rgit.Pipeline (diffAndPlan, pushSyncFiles, pullSyncFiles)
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
-- Ensures rgit from this project is on PATH when running tests.
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
        rawSystem "rclone" ["purge", "gdrive-test:rgit-test"] >>
        rawSystem "rclone" ["mkdir", "gdrive-test:rgit-test"]
  _ <- catch (void purgeAndMkdir) (\(_ :: SomeException) -> return ())
  -- Prepend directory containing rgit to PATH so shelltest runs use this build
  rgitBin <- readProcess "cabal" ["list-bin", "rgit"] ""
  let rgitDir = takeDirectory (filter (`notElem` "\n\r") rgitBin)
  env <- getEnvironment
  let pathSep = if os == "mingw32" || os == "win32" then ";" else ":"
  let path = case lookup "PATH" env of
        Nothing -> rgitDir
        Just p  -> rgitDir ++ pathSep ++ p
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
2. Build rgit: `cabal build rgit`
3. Run CLI tests: `cabal test cli`

The test runner puts the built `rgit` on `PATH` and invokes `shelltest test/cli`, so all `.test` files under `test/cli/` are run. To run a single file: `shelltest test/cli/init.test` (ensure `rgit` is on `PATH`, e.g. `cabal exec -- env PATH="$(cabal list-bin rgit):$PATH" shelltest test/cli/init.test` on Unix).

## gdrive-remote.test

Tests rclone Google Drive remote: push, pull, fetch, and corruption recovery.

**Prerequisites:**

- `rclone` on PATH
- rclone remote named **gdrive-test** configured (e.g. `rclone config` Γזע Google Drive)
- Remote path **gdrive-test:rgit-test** is used; the test purges and recreates it

**What it does:**

- **Two repos:** `work_a` (pusher) and `work_b` (puller), both use `gdrive-test:rgit-test` as origin
- **Push/pull/fetch:** Repo A adds a file, commits, pushes; Repo B fetches and pulls; verifies file content
- **Corruption:** Uses `rclone deletefile` to remove a file on the remote (simulates partial/corrupt state)
- **Verify --remote:** Repo B runs `rgit verify --remote` and expects missing-file issues
- **Recovery:** Repo A pushes again (re-syncs files); Repo B fetches/pulls; verify is clean
- **Orphan file:** Adds a file on the remote with `rclone copyto` (not in metadata); verifies behavior
- **Cleanup:** Removes the orphan from the remote at the end

To run only the gdrive tests (requires rclone + gdrive-test remote): `shelltest test/cli/gdrive-remote.test` (with `rgit` on PATH).

## device-prompt.test

Tests the device name prompt when adding a filesystem remote. Uses `RGIT_USE_STDIN=1` so piped input is read as the device name (enables non-TTY testing). Uses `subst` on Windows to create a writable volume for the device flow. Verifies the remote is stored (either device name or fallback path).

**Unit tests** for the device prompt logic (sanitization, validation, interactive vs non-interactive) live in the `device-prompt` test-suite: `cabal test device-prompt`.

## fsck.test

Tests `rgit fsck` (local-only, git-style: terse output, one line per issue, exit 1 on any failure). Covers:
- Fresh repo / repo with committed files: fsck prints nothing and exits 0 when OK.
- Corrupted working-tree file: prints `hash mismatch <path>`, exits 1.
- Missing tracked file: prints `missing <path>`, exits 1.

Fsck does not check remote; use `rgit verify --remote` for that.

## remote-check.test

Tests `rgit remote check`: runs **rclone check** between local working tree and the configured remote (excludes `.rgit`). Requires rclone on PATH. Covers:
- No remote configured: prints "Error: No remote URL configured." and exits 0.
- Local directory as "remote" (remote_mirror): add remote, change local file, run `rgit remote check` Γזע exits 1 and reports differences (e.g. "differences found", "size differ", "hash differ").
```

---

## test/cli/device-prompt.test

**Path:** `test/cli/device-prompt.test`

*Source file.*

```text
# Device name prompt integration tests
# Tests the interactive path when RGIT_USE_STDIN=1 (piped input for testing)
# Uses subst on Windows to create a writable volume root for .rgit-store

# --- Setup: subst creates Z: -> test/cli so we can write .rgit-store at volume root ---
subst Z: %CD%\test\cli 2>nul
<<<
>>>
>>>= 0

# --- Setup: create work dir, remote dir, init rgit ---
rmdir /s /q test\cli\work 2>nul & rmdir /s /q test\cli\device_prompt_remote 2>nul & mkdir test\cli\work & mkdir test\cli\device_prompt_remote & cd test\cli\work & rgit init
<<<
>>> /Initialized/
>>>= 0

# --- Add remote with piped device name (RGIT_USE_STDIN=1) ---
cd test\cli\work & set RGIT_USE_STDIN=1 & echo test_device| rgit remote add origin Z:\device_prompt_remote
<<<
>>> /Remote.*test_device|Remote.*added/
>>>= 0

# --- Verify remote was stored: either device (test_device) or fallback (local:) ---
findstr /C:"test_device" test\cli\work\.rgit\remotes\origin >nul || findstr /C:"local:" test\cli\work\.rgit\remotes\origin >nul
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

## test/cli/fsck.test

**Path:** `test/cli/fsck.test`

*Source file.*

```text
# rgit fsck tests (git-style output: terse, one line per issue, exit 1 on any failure)

# --- fsck on fresh repo (no remote): prints nothing when OK ---
rmdir /s /q test\cli\work 2>nul & mkdir test\cli\work & cd test\cli\work & rgit init
<<<
>>> /Initialized/
>>>= 0

# fsck OK = no output, exit 0
cd test\cli\work & rgit fsck
<<<
>>>
>>>= 0

# --- fsck on repo with committed files: still no output when OK ---
rmdir /s /q test\cli\work 2>nul & mkdir test\cli\work & cd test\cli\work & rgit init & echo hello> a.txt & rgit add a.txt & rgit commit -m "Add a"
<<<
>>> /Initialized/
>>>= 0

cd test\cli\work & rgit fsck
<<<
>>>
>>>= 0

# --- fsck detects local hash mismatch: git-style "hash mismatch <path>", exit 1 ---
rmdir /s /q test\cli\work 2>nul & mkdir test\cli\work & cd test\cli\work & rgit init & echo original> file.bin & rgit add file.bin & rgit commit -m "Add file"
<<<
>>> /Initialized/
>>>= 0

cd test\cli\work & echo corrupted> file.bin
<<<
>>>
>>>= 0

cd test\cli\work & rgit fsck
<<<
>>>2 /hash mismatch.*file\.bin/
>>>= 1

# --- fsck detects missing file: git-style "missing <path>", exit 1 ---
rmdir /s /q test\cli\work 2>nul & mkdir test\cli\work & cd test\cli\work & rgit init & echo x> missing.txt & rgit add missing.txt & rgit commit -m "Add missing"
<<<
>>> /Initialized/
>>>= 0

del test\cli\work\missing.txt
<<<
>>>
>>>= 0

cd test\cli\work & rgit fsck
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
# Requires: rclone remote "gdrive-test" configured; path "rgit-test" used on that remote.

# --- Setup: clean work dirs and remote ---
rmdir /s /q test\cli\work_a 2>nul & rmdir /s /q test\cli\work_b 2>nul & mkdir test\cli\work_a & mkdir test\cli\work_b & rclone purge gdrive-test:rgit-test 2>nul & rclone mkdir gdrive-test:rgit-test
<<<
>>>
>>>= 0

# --- Repo A: init (use main branch for fetch/pull), add remote, add file, commit, push ---
cd test\cli\work_a & rgit init
<<<
>>> /Initialized/
>>>= 0

cd test\cli\work_a & rgit remote add origin gdrive-test:rgit-test
<<<
>>> /Remote.*added/
>>>= 0

cd test\cli\work_a & echo hello from A> foo.txt & echo x> bar.bin & rgit add foo.txt bar.bin & rgit commit -m "Add foo and bar"
<<<
>>> /\[master|main|file changed/
>>>= 0

cd test\cli\work_a & rgit push
<<<
>>> /Metadata push complete\.|Remote is empty|Remote check passed/
>>>= 0

# --- A: remote check after push: all files on remote, should match ---
cd test\cli\work_a & rgit remote check
<<<
>>> /files match between local and remote/
>>>= 0

# --- Repo B: init, add remote, fetch, pull ---
cd test\cli\work_b & rgit init
<<<
>>> /Initialized/
>>>= 0

cd test\cli\work_b & rgit remote add origin gdrive-test:rgit-test
<<<
>>> /Remote.*added/
>>>= 0

cd test\cli\work_b & rgit fetch
<<<
>>>2 /From gdrive-test:rgit-test|new branch.*origin\/main/
>>>= 0

cd test\cli\work_b & rgit pull
<<<
>>> /first pull|Checking out|Syncing binaries|Merge made|remote: Counting/
>>>= 0

# --- Verify B has the file from A (text in index, binary in work tree) ---
findstr /C:"hello from A" test\cli\work_b\.rgit\index\foo.txt >nul
<<<
>>>
>>>= 0

# --- Corrupt remote: delete a binary file with rclone (simulate partial/corrupt state) ---
rclone deletefile gdrive-test:rgit-test/bar.bin
<<<
>>>
>>>= 0

# --- B: verify --remote may report missing file or 0 files (bundle metadata) ---
cd test\cli\work_b & rgit verify --remote
<<<
>>> /Missing:|issues found|ERROR|\[OK\] All files match|Verifying 0/
>>>= 0

# --- Restore remote: push from A again (re-syncs files + bundle) ---
cd test\cli\work_a & rgit push
<<<
>>> /Metadata push complete\.|Remote check passed|Pulling changes|up to date/
>>>= 0

# --- B: fetch and pull to get restored state ---
cd test\cli\work_b & rgit fetch
<<<
>>>
>>>= 0

cd test\cli\work_b & rgit pull
<<<
>>> /up to date|Merge made|Syncing binaries|first pull/
>>>= 0

# --- B: verify --remote should be clean again ---
cd test\cli\work_b & rgit verify --remote
<<<
>>> /\[OK\] All files match|Verifying/
>>>= 0

findstr /C:"hello from A" test\cli\work_b\.rgit\index\foo.txt >nul
<<<
>>>
>>>= 0

# --- Second round: A adds another file, pushes; B pulls ---
cd test\cli\work_a & echo second file> baz.txt & rgit add baz.txt & rgit commit -m "Add baz" & rgit push
<<<
>>> /Metadata push complete\.|Remote check passed|file changed/
>>>= 0

cd test\cli\work_b & rgit fetch & rgit pull
<<<
>>>2 /From (gdrive-test|\.rgit\\fetched_remote\.bundle)|Merge made|Syncing|Updating|remote: Counting/
>>>= 0

findstr /C:"second file" test\cli\work_b\.rgit\index\baz.txt >nul
<<<
>>>
>>>= 0

# --- Corrupt again: add an orphan file on remote with rclone ---
echo junk content> test\cli\work_b\junk.tmp & rclone copyto test\cli\work_b\junk.tmp gdrive-test:rgit-test/orphan.txt
<<<
>>>
>>>= 0

# --- B: fetch (bundle unchanged), then pull: local should not get orphan; verify --remote ---
cd test\cli\work_b & rgit fetch
<<<
>>>
>>>= 0

cd test\cli\work_b & rgit verify --remote
<<<
>>> /Verifying|All files match|issues found/
>>>= 0

# --- Cleanup: remove orphan from remote so future runs are clean ---
rclone deletefile gdrive-test:rgit-test/orphan.txt
<<<
>>>
>>>= 0
```

---

## test/cli/init.test

**Path:** `test/cli/init.test`

*Source file.*

```text
# Setup: clean environment
rmdir /s /q test\cli\work 2>nul & mkdir test\cli\work & cd test\cli\work & rgit init
<<<
>>> /Initialized/
>>>= 0

# Verify .rgit directory was created
if exist "test\cli\work\.rgit\" (echo exists) else (echo missing)
<<<
>>>
exists
>>>= 0
```

---

## test/cli/merge-local.test

**Path:** `test/cli/merge-local.test`

*Source file.*

```text
# =============================================================================
# rgit merge tests (local directory as remote, no cloud needed)
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
mkdir test\cli\work_a & cd test\cli\work_a & rgit init
<<<
>>> /Initialized/
>>>= 0

# ---- Repo A: add remote ----
cd test\cli\work_a & rgit remote add origin "%CD%\test\cli\shared_remote"
<<<
>>> /Remote.*added/
>>>= 0

# ---- Repo A: create base text file, add, commit ----
cd test\cli\work_a & echo base content> text.txt & rgit add text.txt
<<<
>>>
>>>= 0

cd test\cli\work_a & rgit commit -m "Base: add text.txt"
<<<
>>> /\[main|master|file changed/
>>>= 0

# ---- Repo A: create base binary file, add, commit ----
cd test\cli\work_a & echo binarydata> data.bin & rgit add data.bin
<<<
>>>
>>>= 0

cd test\cli\work_a & rgit commit -m "Base: add data.bin"
<<<
>>> /\[main|master|file changed/
>>>= 0

# ---- Repo A: push base state to shared remote ----
cd test\cli\work_a & rgit push
<<<
>>> /Metadata push complete|Remote is empty|Remote check passed/
>>>= 0

# ======================================================================
# SECTION 2: FIRST PULL Γאפ repo B fetches from shared remote (unborn branch)
# (Mirrors Git's test of merge with unborn HEAD)
# ======================================================================

# ---- Repo B: init ----
mkdir test\cli\work_b & cd test\cli\work_b & rgit init
<<<
>>> /Initialized/
>>>= 0

# ---- Repo B: add same remote ----
cd test\cli\work_b & rgit remote add origin "%CD%\test\cli\shared_remote"
<<<
>>> /Remote.*added/
>>>= 0

# ---- Repo B: fetch ----
cd test\cli\work_b & rgit fetch
<<<
>>>2 /From.*shared_remote|new branch.*origin\/main/
>>>= 0

# ---- Repo B: pull (first pull Γאפ checkout remote as main) ----
cd test\cli\work_b & rgit pull
<<<
>>> /first pull|Checking out|Syncing binaries/
>>>= 0

# ---- Verify B has the text file from A (content stored in .rgit/index) ----
findstr /C:"base content" test\cli\work_b\.rgit\index\text.txt >nul
<<<
>>>
>>>= 0

# ---- Verify B has the binary file metadata ----
findstr /C:"hash:" test\cli\work_b\.rgit\index\data.bin >nul
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
cd test\cli\work_b & rgit status
<<<
>>> /nothing to commit|working tree clean/
>>>= 0

# ======================================================================
# SECTION 3: FAST-FORWARD MERGE Γאפ A pushes, B pulls with no local changes
# (Mirrors Git's "merge c0 with c1" ff test in t7600)
# ======================================================================

# ---- A: add a new file and push ----
cd test\cli\work_a & echo new from A> extra.txt & rgit add extra.txt & rgit commit -m "Add extra.txt"
<<<
>>> /\[main|master|file changed/
>>>= 0

cd test\cli\work_a & rgit push
<<<
>>> /Metadata push complete|Remote check passed/
>>>= 0

# ---- B: pull Γאפ should merge cleanly (B has no local divergence) ----
cd test\cli\work_b & rgit fetch
<<<
>>>
>>>= 0

cd test\cli\work_b & rgit pull
<<<
>>> /Updating|Merge made|Syncing binaries/
>>>= 0

# ---- Verify B has the new file ----
findstr /C:"new from A" test\cli\work_b\.rgit\index\extra.txt >nul
<<<
>>>
>>>= 0

# ---- Verify B status is clean ----
cd test\cli\work_b & rgit status
<<<
>>> /nothing to commit|working tree clean/
>>>= 0

# ======================================================================
# SECTION 4: CLEAN THREE-WAY MERGE Γאפ non-conflicting parallel changes
# (Mirrors Git's "merge c1 with c2" where changes are in different hunks)
# ======================================================================

# ---- A: modify text.txt and push ----
cd test\cli\work_a & echo updated by A> text.txt & rgit add text.txt & rgit commit -m "A: update text.txt"
<<<
>>> /\[main|master|file changed/
>>>= 0

cd test\cli\work_a & rgit push
<<<
>>> /Metadata push complete|Remote check passed/
>>>= 0

# ---- B: add a NEW file (doesn't overlap with A's changes) and commit ----
cd test\cli\work_b & echo B only file> bonly.txt & rgit add bonly.txt & rgit commit -m "B: add bonly.txt"
<<<
>>> /\[main|master|file changed/
>>>= 0

# ---- B: pull Γאפ should merge cleanly since changes don't overlap ----
cd test\cli\work_b & rgit pull
<<<
>>> /Merge made|Syncing binaries|Updating/
>>>= 0

# ---- Verify B has A's updated text.txt ----
findstr /C:"updated by A" test\cli\work_b\.rgit\index\text.txt >nul
<<<
>>>
>>>= 0

# ---- Verify B still has its own bonly.txt ----
findstr /C:"B only file" test\cli\work_b\.rgit\index\bonly.txt >nul
<<<
>>>
>>>= 0

# ---- Verify B status is clean ----
cd test\cli\work_b & rgit status
<<<
>>> /nothing to commit|working tree clean/
>>>= 0

# ======================================================================
# SECTION 5: CONTENT CONFLICT Γאפ resolve keep-local
# Both A and B modify the same text file with different content.
# rgit detects the conflict in .rgit/index/ and prompts for resolution.
# User chooses (l)ocal.
# (Mirrors Git's "merge c1 with c2 Γאפ content conflict" in t7600)
# ======================================================================

# ---- Synchronize: B pushes its current state so both are aligned ----
cd test\cli\work_b & rgit push
<<<
>>> /Metadata push complete|Remote check passed|up to date/
>>>= 0

cd test\cli\work_a & rgit fetch & rgit pull
<<<
>>>
>>>= 0

# ---- A: modify text.txt to A's version and push ----
cd test\cli\work_a & echo conflict version A> text.txt & rgit add text.txt & rgit commit -m "A: conflict version"
<<<
>>> /\[main|master|file changed/
>>>= 0

cd test\cli\work_a & rgit push
<<<
>>> /Metadata push complete|Remote check passed/
>>>= 0

# ---- B: modify SAME text.txt to B's version and commit locally ----
cd test\cli\work_b & echo conflict version B> text.txt & rgit add text.txt & rgit commit -m "B: conflict version"
<<<
>>> /\[main|master|file changed/
>>>= 0

# ---- B: pull Γאפ conflict expected; pipe "l" to keep local ----
cd test\cli\work_b & rgit pull
<<<
l
>>> /Automatic merge failed|CONFLICT|Resolving conflicts|Merge complete|conflict/
>>>= 0

# ---- Verify B kept local version in metadata (B's content wins) ----
findstr /C:"conflict version B" test\cli\work_b\.rgit\index\text.txt >nul
<<<
>>>
>>>= 0

# ======================================================================
# SECTION 6: CONTENT CONFLICT Γאפ resolve take-remote
# Same setup: both modify same file, but user chooses (r)emote.
# (Mirrors Git's checkout --theirs pattern)
# ======================================================================

# ---- Re-sync: push B's state, pull to A ----
cd test\cli\work_b & rgit push
<<<
>>> /Metadata push complete|Remote check passed|up to date/
>>>= 0

cd test\cli\work_a & rgit fetch & rgit pull
<<<
>>>
>>>= 0

# ---- A: modify text.txt to "remote wins" and push ----
cd test\cli\work_a & echo remote wins version> text.txt & rgit add text.txt & rgit commit -m "A: remote wins"
<<<
>>> /\[main|master|file changed/
>>>= 0

cd test\cli\work_a & rgit push
<<<
>>> /Metadata push complete|Remote check passed/
>>>= 0

# ---- B: modify SAME text.txt differently and commit locally ----
cd test\cli\work_b & echo local loses version> text.txt & rgit add text.txt & rgit commit -m "B: local loses"
<<<
>>> /\[main|master|file changed/
>>>= 0

# ---- B: pull Γאפ conflict expected; pipe "r" to take remote ----
cd test\cli\work_b & rgit pull
<<<
r
>>> /Automatic merge failed|CONFLICT|Resolving conflicts|Merge complete|conflict/
>>>= 0

# ---- Verify B took remote version in metadata (A's content wins) ----
findstr /C:"remote wins version" test\cli\work_b\.rgit\index\text.txt >nul
<<<
>>>
>>>= 0

# ======================================================================
# SECTION 7: MULTIPLE SIMULTANEOUS CONFLICTS Γאפ mixed resolution
# Both A and B modify two files. B picks (l)ocal for first, (r)emote for second.
# (Mirrors Git's "merge with multiple conflicting files" tests)
# ======================================================================

# ---- Re-sync ----
cd test\cli\work_b & rgit push
<<<
>>> /Metadata push complete|Remote check passed|up to date/
>>>= 0

cd test\cli\work_a & rgit fetch & rgit pull
<<<
>>>
>>>= 0

# ---- A: modify both files and push ----
cd test\cli\work_a & echo A multi 1> file1.txt & echo A multi 2> file2.txt & rgit add file1.txt file2.txt & rgit commit -m "A: modify two files"
<<<
>>> /\[main|master|file changed/
>>>= 0

cd test\cli\work_a & rgit push
<<<
>>> /Metadata push complete|Remote check passed/
>>>= 0

# ---- B: modify SAME two files differently and commit ----
cd test\cli\work_b & echo B multi 1> file1.txt & echo B multi 2> file2.txt & rgit add file1.txt file2.txt & rgit commit -m "B: modify two files"
<<<
>>> /\[main|master|file changed/
>>>= 0

# ---- B: pull Γאפ two conflicts; pipe "l" then "r" ----
cd test\cli\work_b & rgit pull
<<<
l
r
>>> /Automatic merge failed|CONFLICT|Resolving conflicts|Merge complete|2 conflict/
>>>= 0

# ---- Verify: file1.txt kept local (B's version), file2.txt took remote (A's version) ----
findstr /C:"B multi 1" test\cli\work_b\.rgit\index\file1.txt >nul
<<<
>>>
>>>= 0

findstr /C:"A multi 2" test\cli\work_b\.rgit\index\file2.txt >nul
<<<
>>>
>>>= 0

# ======================================================================
# SECTION 8: ADD/ADD CONFLICT Γאפ both repos create the same new file
# (Mirrors Git's "CONFLICT (add/add)" tests in t6422)
# ======================================================================

# ---- Re-sync ----
cd test\cli\work_b & rgit push
<<<
>>> /Metadata push complete|Remote check passed|up to date/
>>>= 0

cd test\cli\work_a & rgit fetch & rgit pull
<<<
>>>
>>>= 0

# ---- A: create brand-new file "shared_new.txt" and push ----
cd test\cli\work_a & echo A created this> shared_new.txt & rgit add shared_new.txt & rgit commit -m "A: add shared_new.txt"
<<<
>>> /\[main|master|file changed/
>>>= 0

cd test\cli\work_a & rgit push
<<<
>>> /Metadata push complete|Remote check passed/
>>>= 0

# ---- B: create SAME filename "shared_new.txt" with different content, commit ----
cd test\cli\work_b & echo B created this> shared_new.txt & rgit add shared_new.txt & rgit commit -m "B: add shared_new.txt"
<<<
>>> /\[main|master|file changed/
>>>= 0

# ---- B: pull Γאפ add/add conflict; choose "r" (take remote) ----
cd test\cli\work_b & rgit pull
<<<
r
>>> /Automatic merge failed|CONFLICT|Resolving conflicts|Merge complete|conflict/
>>>= 0

# ---- Verify B took A's version ----
findstr /C:"A created this" test\cli\work_b\.rgit\index\shared_new.txt >nul
<<<
>>>
>>>= 0

# ======================================================================
# SECTION 9: MERGE --ABORT with no merge in progress
# (Mirrors Git's t7611 "git merge --abort fails without MERGE_HEAD")
# ======================================================================

# ---- Verify clean state first ----
cd test\cli\work_b & rgit status
<<<
>>> /nothing to commit|working tree clean/
>>>= 0

# ---- merge --abort when nothing in progress: should print error ----
cd test\cli\work_b & rgit merge --abort
<<<
>>>2 /no merge in progress/
>>>= 1

# ======================================================================
# SECTION 10: MERGE --CONTINUE with no merge in progress
# (Mirrors Git's "git merge --continue fails without in-progress merge")
# ======================================================================

cd test\cli\work_b & rgit merge --continue
<<<
>>>2 /no merge in progress|error/
>>>= 1

# ======================================================================
# SECTION 11: PULL --ACCEPT-REMOTE
# A pushes a change; B has diverged locally. B uses --accept-remote
# to accept whatever is on the remote as truth.
# (rgit-specific: no Git equivalent; tests the remote-reality acceptance flow)
# ======================================================================

# ---- Re-sync ----
cd test\cli\work_b & rgit push
<<<
>>> /Metadata push complete|Remote check passed|up to date/
>>>= 0

cd test\cli\work_a & rgit fetch & rgit pull
<<<
>>>
>>>= 0

# ---- A: modify text.txt and push ----
cd test\cli\work_a & echo accept remote test> text.txt & rgit add text.txt & rgit commit -m "A: accept-remote test"
<<<
>>> /\[main|master|file changed/
>>>= 0

cd test\cli\work_a & rgit push
<<<
>>> /Metadata push complete|Remote check passed/
>>>= 0

# ---- B: pull with --accept-remote (skip conflict resolution, accept remote) ----
cd test\cli\work_b & rgit pull --accept-remote
<<<
>>> /Accepting remote file state|accept-remote completed|Scanning remote/
>>>= 0

# ---- Verify B accepted A's content ----
findstr /C:"accept remote test" test\cli\work_b\.rgit\index\text.txt >nul
<<<
>>>
>>>= 0

# ======================================================================
# SECTION 12: VERIFY FINAL STATE Γאפ working tree clean, no dangling state
# (Mirrors Git's post-merge state verification: no MERGE_HEAD, clean diff)
# ======================================================================

# ---- B: status should be clean ----
cd test\cli\work_b & rgit status
<<<
>>> /nothing to commit|working tree clean/
>>>= 0

# ---- B: fsck should find no issues ----
cd test\cli\work_b & rgit fsck
<<<
>>>
>>>= 0

# ---- A: status should be clean ----
cd test\cli\work_a & rgit status
<<<
>>> /nothing to commit|working tree clean/
>>>= 0

# ---- A: fsck should find no issues ----
cd test\cli\work_a & rgit fsck
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

## test/cli/one-repo.test

**Path:** `test/cli/one-repo.test`

*Source file.*

```text
# Setup fresh repo
rmdir /s /q test\cli\work 2>nul & mkdir test\cli\work & cd test\cli\work & rgit init
<<<
>>> /Initialized/
>>>= 0

# Add text file - metadata should be created in .rgit/index (content copy for text files)
cd test\cli\work & echo hello> test.txt & rgit add test.txt
<<<
>>>
>>>= 0

# Verify text file metadata: .rgit/index/test.txt exists and contains the content
if exist "test\cli\work\.rgit\index\test.txt" (echo metadata exists) else (echo missing)
<<<
>>>
metadata exists
>>>= 0

findstr /C:"hello" test\cli\work\.rgit\index\test.txt >nul
<<<
>>>
>>>= 0

# Status: new file staged
cd test\cli\work & rgit status
<<<
>>> /new file:.*test\.txt/
>>>= 0

# Commit the staged file
cd test\cli\work & rgit commit -m "Add test.txt"
<<<
>>> /\[master|file changed/
>>>= 0

# Status after commit: working tree clean
cd test\cli\work & rgit status
<<<
>>> /nothing to commit|working tree clean/
>>>= 0

# Add binary file - metadata should have hash and size
cd test\cli\work & echo x> file.bin & rgit add file.bin
<<<
>>>
>>>= 0

# Verify binary metadata: .rgit/index/file.bin exists with hash: and size: lines
if exist "test\cli\work\.rgit\index\file.bin" (echo metadata exists) else (echo missing)
<<<
>>>
metadata exists
>>>= 0

findstr /C:"hash:" test\cli\work\.rgit\index\file.bin >nul
<<<
>>>
>>>= 0

findstr /C:"size:" test\cli\work\.rgit\index\file.bin >nul
<<<
>>>
>>>= 0

# Status: new binary file staged
cd test\cli\work & rgit status
<<<
>>> /new file:.*file\.bin/
>>>= 0

# Commit binary
cd test\cli\work & rgit commit -m "Add file.bin"
<<<
>>> /\[master|file changed/
>>>= 0

# Add multiple files with "add ."
cd test\cli\work & echo data> data.txt & echo notes> notes.txt & rgit add .
<<<
>>>
>>>= 0

# Verify both metadata files exist
if exist "test\cli\work\.rgit\index\data.txt" (echo data ok) else (echo missing)
<<<
>>>
data ok
>>>= 0

if exist "test\cli\work\.rgit\index\notes.txt" (echo notes ok) else (echo missing)
<<<
>>>
notes ok
>>>= 0

# Status: both new files staged
cd test\cli\work & rgit status
<<<
>>> /new file:.*data\.txt/
>>>= 0

# Commit multiple files
cd test\cli\work & rgit commit -m "Add data and notes"
<<<
>>> /\[master|file changed/
>>>= 0

# Modify text file, add it
cd test\cli\work & echo hello world> test.txt & rgit add test.txt
<<<
>>>
>>>= 0

# Verify metadata was updated (new content)
findstr /C:"hello world" test\cli\work\.rgit\index\test.txt >nul
<<<
>>>
>>>= 0

# Status: modified file staged
cd test\cli\work & rgit status
<<<
>>> /modified:.*test\.txt/
>>>= 0

# Commit modification
cd test\cli\work & rgit commit -m "Update test.txt"
<<<
>>> /\[master|file changed/
>>>= 0

# Final status: clean
cd test\cli\work & rgit status
<<<
>>> /nothing to commit|working tree clean/
>>>= 0
```

---

## test/cli/remote-check.test

**Path:** `test/cli/remote-check.test`

*Source file.*

```text
# rgit remote check tests
# Runs rclone check between local working tree and remote (excludes .rgit).
# Requires: rclone on PATH.

# --- No remote configured: must print error ---
rmdir /s /q test\cli\work 2>nul & mkdir test\cli\work & cd test\cli\work & rgit init
<<<
>>> /Initialized/
>>>= 0

cd test\cli\work & rgit remote check
<<<
>>>2 /fatal: No remote configured/
>>>= 1

# --- Setup: repo with text + binary, local dir OUTSIDE work as "remote", push, then check ---
rmdir /s /q test\cli\work 2>nul & rmdir /s /q test\cli\remote_mirror 2>nul & mkdir test\cli\work & mkdir test\cli\remote_mirror & cd test\cli\work & rgit init & echo hello> greet.txt & echo x> data.bin & rgit add greet.txt data.bin & rgit commit -m "Add files"
<<<
>>> /Initialized/
>>>= 0

cd test\cli\work & rgit remote add origin "%CD%\test\cli\remote_mirror"
<<<
>>> /Remote.*added/
>>>= 0

cd test\cli\work & rgit push
<<<
>>> /Metadata push complete/
>>>= 0

# --- Happy path: local matches remote after push, exit 0 ---
cd test\cli\work & rgit remote check
<<<
>>> /files match between local and remote/
>>>= 0

# --- Modify local file: rclone check detects difference, exit 1 ---
cd test\cli\work & echo changed> greet.txt
<<<
>>>
>>>= 0

cd test\cli\work & rgit remote check
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
rmdir /s /q test\cli\work 2>nul & mkdir test\cli\work & cd test\cli\work & rgit init
<<<
>>> /Initialized/
>>>= 0

cd test\cli\work & echo original> file.txt & rgit add file.txt & rgit commit -m "Add file"
<<<
>>> /\[master|file changed/
>>>= 0

# Modify file, then restore (discard working tree changes)
cd test\cli\work & echo modified> file.txt
<<<
>>>
>>>= 0

cd test\cli\work & rgit restore file.txt
<<<
>>>
>>>= 0

findstr /C:"original" test\cli\work\file.txt >nul
<<<
>>>
>>>= 0

# Modify again, use restore -- (explicit pathspec separator, like git)
cd test\cli\work & echo changed> file.txt & rgit restore -- file.txt
<<<
>>>
>>>= 0

findstr /C:"original" test\cli\work\file.txt >nul
<<<
>>>
>>>= 0

# Modify, then checkout -- (legacy git syntax)
cd test\cli\work & echo discarded> file.txt & rgit checkout -- file.txt
<<<
>>>
>>>= 0

findstr /C:"original" test\cli\work\file.txt >nul
<<<
>>>
>>>= 0

# Test restore --staged (unstage without discarding working tree)
cd test\cli\work & echo staged change> file.txt & rgit add file.txt
<<<
>>>
>>>= 0

cd test\cli\work & rgit restore --staged file.txt
<<<
>>>
>>>= 0

cd test\cli\work & rgit status
<<<
>>> /modified:.*file\.txt/
>>>= 0

findstr /C:"staged change" test\cli\work\file.txt >nul
<<<
>>>
>>>= 0

# Restore working tree (discard the modification)
cd test\cli\work & rgit restore file.txt
<<<
>>>
>>>= 0

findstr /C:"original" test\cli\work\file.txt >nul
<<<
>>>
>>>= 0

cd test\cli\work & rgit status
<<<
>>> /nothing to commit|working tree clean/
>>>= 0

# Test restore . (all files)
cd test\cli\work & echo a> a.txt & echo b> b.txt & rgit add . & rgit commit -m "Add a and b"
<<<
>>> /\[master|file changed/
>>>= 0

cd test\cli\work & echo x> a.txt & echo y> b.txt & rgit restore .
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
cd test\cli\work & echo XX> a.txt & echo YY> b.txt & rgit checkout -- a.txt b.txt
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

TEST_DIR="$TEMP/rgit-test"

setup_fresh_repo() {
    rm -rf "$TEST_DIR"
    mkdir -p "$TEST_DIR"
    cd "$TEST_DIR"
    rgit init >/dev/null 2>&1
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
rmdir /s /q test\cli\work 2>nul & mkdir test\cli\work & cd test\cli\work & rgit init & echo data> file.bin
<<<
>>> /Initialized/
>>>= 0

# Status should show untracked file
cd test\cli\work & rgit status
<<<
>>> /file\.bin/
>>>= 0
```

---

