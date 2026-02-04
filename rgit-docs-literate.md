# rgit — Documentation (Literate Programming Document)

This document contains all documentation files for the rgit project:
specifications, refactoring plans, and design documents.

---

## docs/spec.md

**Path:** `docs/spec.md`

*Source file.*

```markdown
# rgit Implementation Specification (v3)

## Context and Vision

**rgit** is a version control system designed for large files that leverages Git as a metadata-tracking engine while storing actual file content separately. The core insight: Git excels at tracking small text files, so we feed it exactly that Γאפ tiny metadata files instead of large binaries.

**Mental Model**: rgit = Git(metadata) + rclone(sync) + [optional CAS(content) for rgit-solid]

**Current State**: We are implementing **rgit-lite**. CAS and content-addressed storage come later in **rgit-solid**.

### Origin

The key architectural idea: instead of a custom manifest, keep small text files in `.rgit/index/` mirroring the working tree's directory structure, each containing just hash and size. Define Git's working tree as `.rgit/index/`, and you get `add`, `commit`, `diff`, `log`, branching, and the entire Git command set for free. Git becomes the manifest manager, diff engine, and history store Γאפ without ever seeing a large file.

### Comparison with Alternatives

rgit occupies a different niche than git-lfs and git-annex:

- **git-lfs**: Stores pointer files in Git, actual files on a server. Server-dependent, transparent but limited. Pragmatic hack.
- **git-annex**: Extremely powerful distributed system with pluggable remotes and policy-driven placement. Extremely complex.
- **rgit**: Minimal, explicit, correctness-oriented. Git never touches large files. Dumb remotes via rclone. Filesystem-first.

rgit's killer feature is **clarity** Γאפ users always know what state their files are in, what rgit is about to do, and how to recover from errors.

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
    Γפ£ΓפאΓפא index/              # Metadata files live here (Git working tree)
    Γפג   Γפ£ΓפאΓפא .git/           # Git's internal directory
    Γפג   Γפ£ΓפאΓפא src/
    Γפג   Γפג   ΓפפΓפאΓפא video.mp4   # Metadata file (NOT the actual video)
    Γפג   ΓפפΓפאΓפא data/
    Γפג       ΓפפΓפאΓפא dataset.bin # Metadata file (NOT the actual data)
    Γפ£ΓפאΓפא remotes/            # Named remote configs (device-aware)
    Γפג   ΓפפΓפאΓפא origin          # Remote target (cloud URL or device:path)
    Γפ£ΓפאΓפא devices/            # Device identity files
    Γפ£ΓפאΓפא target              # Legacy: single remote URL
    ΓפפΓפאΓפא ignore              # Gitignore-style rules
```

### The Index Invariant

**Git is the sole authority over `.rgit/index/`.** After any git operation that
changes HEAD (merge, checkout, commit), `.rgit/index/` is correct by
definition. The only remaining work is mirroring those changes onto the actual
working directory (downloading binaries from remote, copying text files from
the index).

This invariant applies uniformly across all pull/merge paths:

- `git merge` Γזע determines correct metadata Γזע we sync actual files
- `git checkout` (first pull, `--accept-remote`) Γזע determines correct metadata Γזע we sync actual files

No path should write metadata files to `.rgit/index/` directly and then commit.
Scanning the remote via rclone and writing the result to the index bypasses git
and **will** produce wrong metadata (rclone cannot distinguish text from binary,
so text files get `hash:/size:` metadata instead of their actual content).

### Metadata File Format

Each metadata file under `.rgit/index/` mirrors the path of its corresponding real file and contains **ONLY**:

```
hash: md5:a1b2c3d4e5f6...
size: 1048576
```

Two fields only:
- `hash` Γאפ MD5 hash of file content (prefixed with `md5:`)
- `size` Γאפ File size in bytes

Parsing and serialization are handled by a single canonical module (`Rgit/Internal/Metadata.hs`) which enforces the round-trip property: `parseMetadata . serializeMetadata Γיí Just`.

**Known deviation from original spec**: The original spec called for SHA256 and `hash=`/`size=` (equals-sign) format. The implementation uses MD5 (matching rclone's native hash) and `hash: `/`size: ` (colon-space) format. This is intentional Γאפ MD5 is used everywhere for consistency with rclone comparisons. SHA256 may be added later alongside MD5, not as a replacement.

### Git Configuration

rgit runs Git with:
- Git repository initialized in `.rgit/index/.git` during `rgit init`
- All git operations use the repo at `.rgit/index/.git`
- Working tree is `.rgit/index` (not the project root)
- `core.excludesFile` points to `.rgit/ignore`

### File Handling

- Regular files (binary Γזע metadata, text Γזע content stored directly in index)
- Symlinks Γאפ **ignored** (too many edge cases with cloud remotes)
- Device files, sockets, named pipes Γאפ **ignored**
- Empty directories Γאפ **ignored** (not tracked; many cloud backends don't support them)

---

## Module Architecture

### Layer Contract

The codebase follows strict layer boundaries:

```
Rgit/Commands.hs Γזע Bit.hs Γזע Internal/Transport.hs Γזע rclone (only here!)
                      Γזף
                   Internal/Git.hs Γזע git (only here!)
```

- **Internal/Transport.hs** Γאפ Dumb rclone wrapper. Knows how to `copyTo`, `moveTo`, `deleteFile`, `listJson`, `check`. Takes `Remote` + relative paths. Does NOT know about `.rgit/`, bundles, `RemoteState`, or `FetchResult`.
- **Internal/Git.hs** Γאפ Dumb git wrapper. Knows how to run git commands. Takes args. Does NOT interpret results in domain terms.
- **Bit.hs** Γאפ Smart business logic. All domain knowledge lives here. Knows about remotes, bundles, `.rgit/` layout, sync strategy. Calls Transport and Git, never calls `readProcessWithExitCode` directly.
- **Rgit/Commands.hs** Γאפ Entry point. Parses CLI, resolves the remote, builds `RgitEnv`, dispatches to `Bit`.

### Key Types

| Type | Module | Purpose |
|------|--------|---------|
| `Hash (a :: HashAlgo)` | `Rgit.Types` | Phantom-typed hash Γאפ compiler distinguishes MD5 vs SHA256 |
| `FileEntry` | `Rgit.Types` | Path + EntryKind (hash, size, isText) |
| `RgitEnv` | `Rgit.Types` | Reader environment: cwd, local files, remote, force flags |
| `RgitM` | `Rgit.Types` | `ReaderT RgitEnv IO` Γאפ the application monad |
| `MetaContent` | `Rgit.Internal.Metadata` | Canonical metadata: hash + size, single parser/serializer |
| `Remote` | `Rgit.Remote` | Resolved remote: name + URL. Smart constructor, `remoteUrl` for Transport only |
| `RemoteState` | `Rgit.Remote` | Remote classification: Empty, ValidRgit, NonRgitOccupied, Corrupted, NetworkError |
| `FetchResult` | `Rgit.Remote` | Bundle fetch result: BundleFound, RemoteEmpty, NetworkError |
| `GitDiff` | `Rgit.Diff` | Added, Modified, Deleted, Renamed Γאפ pure diff result |
| `RcloneAction` | `Rgit.Plan` | Copy, Move, Delete, Swap Γאפ concrete rclone operations |
| `FileIndex` | `Rgit.Diff` | Dual-indexed file map (byPath + byHash) for efficient diff/rename detection |
| `Resolution` | `Rgit.Conflict` | KeepLocal or TakeRemote Γאפ conflict resolution choice |
| `DeviceInfo` | `Rgit.Device` | UUID + storage type + optional hardware serial |
| `RemoteTarget` | `Rgit.Device` | TargetCloud, TargetDevice, TargetLocalPath |

### Sync Pipeline

The sync pipeline is composed as pure function composition with effectful endpoints:

```
scan   :: FilePath Γזע IO [FileEntry]        -- effectful (reads filesystem)
diff   :: FileIndex Γזע FileIndex Γזע [GitDiff]  -- pure!
plan   :: GitDiff Γזע RcloneAction              -- pure!
exec   :: RcloneAction Γזע IO ()               -- effectful (calls rclone)
```

The pure middle (`diff >>> plan`) is factored into `Rgit.Pipeline` and is fully property-testable:

```haskell
diffAndPlan :: [FileEntry] -> [FileEntry] -> [RcloneAction]  -- pure core
pushSyncFiles :: [FileEntry] -> [FileEntry] -> [RcloneAction]
pullSyncFiles :: [FileEntry] -> [FileEntry] -> [RcloneAction]
```

### Working Tree Sync: The `oldHead` Pattern

After any git operation that changes HEAD (merge, checkout), the working
directory must be updated to reflect what changed in `.rgit/index/`.
The mechanism:

```haskell
oldHead <- getLocalHeadE              -- 1. Capture HEAD *before* the git operation
-- ... git merge / git checkout ...   -- 2. Git changes HEAD + index
applyMergeToWorkingDir remote oldHead -- 3. Diff old HEAD vs new HEAD, sync files
```

`applyMergeToWorkingDir` uses `git diff --name-status oldHead newHead` to
determine what changed, then:
- **Added/Modified files**: download binary from remote, or copy text from index
- **Deleted files**: remove from working directory
- **Renamed files**: delete old, download/copy new

This is used consistently across:
- Clean merges (fast-forward and three-way)
- Conflict resolution merges
- `--accept-remote` (force-checkout)
- `mergeContinue`

The only exception is **first pull** (`oldHead = Nothing`): there is no
previous HEAD to diff against, so `syncRemoteFilesToLocal` (full rclone-based
diff) is used as fallback.

### Conflict Resolution

Conflict resolution is structured as a fold over a list of conflicts (`Rgit.Conflict`). Each conflict is resolved identically via `resolveConflict`, and the traversal guarantees every conflict is visited exactly once with correct progress tracking (1/N, 2/N, ...). The decision logic (KeepLocal vs TakeRemote) is cleanly separated from the git checkout/merge mechanics.

**Critical**: After resolving all conflicts, the merge commit must **always** be
created, regardless of whether the index has staged changes. When the user
chooses "keep local" (`--ours`), `git checkout --ours` + `git add` restores
HEAD's version Γאפ the index becomes identical to HEAD. A na├»ve `hasStagedChanges`
check would skip the commit, leaving `MERGE_HEAD` dangling. Git's `commit`
command always succeeds when `MERGE_HEAD` exists (it knows it's recording a
merge), even if the tree is identical to HEAD's tree. Skipping the commit
breaks the next push (ancestry check fails because HEAD was never advanced
past the merge).

---

## Command Line Interface (Git-Compatible)

**CRITICAL**: The CLI mirrors Git's interface. Users familiar with Git should feel immediately at home.

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
| `rgit checkout [options] -- <path>` | `git checkout --` | Restore working tree from index (legacy syntax) |
| `rgit reset` | `git reset` | Reset staging area |
| `rgit rm <path>` | `git rm` | Remove file from tracking |
| `rgit mv <src> <dst>` | `git mv` | Move/rename tracked file |
| `rgit branch` | `git branch` | Branch management |
| `rgit merge` | `git merge` | Merge branches |
| `rgit remote add <name> <url>` | `git remote add` | Set named remote URL |
| `rgit remote show [name]` | `git remote show` | Show remote status |
| `rgit remote check [name]` | Γאפ | Check remote connectivity and state |
| `rgit push` | `git push` | Push metadata bundle + sync files via rclone |
| `rgit pull` | `git pull` | Pull metadata bundle + sync files via rclone |
| `rgit pull --accept-remote` | Γאפ | Accept remote file state as truth |
| `rgit pull --manual-merge` | Γאפ | Interactive per-file conflict resolution |
| `rgit fetch` | `git fetch` | Fetch metadata bundle only |
| `rgit verify` | Γאפ | Verify local files match metadata |
| `rgit verify --remote` | Γאפ | Verify remote files match remote metadata |
| `rgit fsck` | `git fsck` | Full integrity check (local + remote + git) |
| `rgit merge --continue` | `git merge --continue` | Continue after conflict resolution |
| `rgit merge --abort` | `git merge --abort` | Abort current merge |
| `rgit branch --unset-upstream` | `git branch --unset-upstream` | Remove tracking config |

---

## Remote Synchronization (Two-Phase, Action-Based)

### Key Insight: Diff-Based Sync, Not Blind Sync

We do **NOT** use `rclone sync`. Instead:

1. Compute diff between current state and desired state
2. Generate minimal action list (Copy, Move, Delete)
3. Execute actions via rclone

This saves bandwidth. For example: renaming a 1GB file becomes `rclone moveto` instead of delete + upload.

### Sync Order (CRITICAL)

**On Push:**
1. **First**: Sync files via rclone (content must exist before metadata claims it does)
2. **Then**: Push metadata bundle via rclone

**On Pull:**
1. **First**: Fetch metadata bundle via rclone
2. **Then**: Git operation (merge or checkout) updates `.rgit/index/`
3. **Then**: Mirror index changes to working directory (download binaries, copy text from index)

**Rationale**: Push files first so the remote is never in a state where metadata references missing content. Pull metadata first so we know what content to fetch. After the git operation, the index is authoritative Γאפ we only need to bring actual files into alignment.

### Remote Types and Device Resolution

rgit supports two kinds of remotes:

- **Cloud remotes**: rclone-based (e.g., `gdrive:Projects/foo`). Identified by URL.
- **Filesystem remotes**: Local/network paths. Use device-identity resolution via UUID + hardware serial.

The `Rgit.Device` module handles filesystem remote resolution:
- Physical storage: Identified by UUID + hardware serial (survives drive letter changes)
- Network storage: Identified by UUID only
- Each volume can have a `.rgit-store` file at its root containing its UUID
- Remote targets are stored in `.rgit/remotes/<name>` as either cloud URLs or `device_name:relative_path`

The `Rgit.Remote` module provides the `Remote` type which abstracts over both:
```
resolveRemote :: FilePath -> String -> IO (Maybe Remote)
-- Tries .rgit/remotes/<name> (device-aware), falls back to git config
```

---

## Verification and Consistency

### `rgit verify`

Verifies local working tree matches local metadata. For each file in metadata, checks that the hash matches the actual file on disk.

### `rgit verify --remote`

Verifies that remote files match what the remote Git bundle says they should be:
1. Fetch remote bundle (metadata)
2. Scan remote files via `rclone lsjson --hash`
3. Compare: Do actual remote files match what remote metadata claims?

### `rgit fsck`

Full integrity check combining local verify, remote verify, and `git fsck` on the internal repository.

---

## Handling Remote Divergence

When remote files don't match remote metadata (detected via `rgit verify --remote` or during `rgit pull`):

### Resolution Option 1: Accept Remote Reality (`--accept-remote`)

Force-checkout the remote branch so git puts the correct metadata in
`.rgit/index/`, then mirror the changes to the working directory. This is
architecturally identical to a normal pull Γאפ just a force-checkout instead of
a merge. Git manages the index; we only sync actual files.

The flow:
1. Fetch remote bundle (git gets remote history)
2. Record current HEAD (for diff-based sync)
3. `git checkout -f -B main refs/remotes/origin/main` (force-checkout remote)
4. `applyMergeToWorkingDir` (diff old HEAD vs new HEAD, sync files)
5. Update tracking ref

**Important**: `--accept-remote` must NOT scan remote files via rclone and write
metadata directly. Rclone cannot distinguish text from binary files
(`fIsText = False` for everything), so text files would get `hash:/size:`
metadata instead of their actual content.

### Resolution Option 2: Force Local (`rgit push --force`)

Upload all local files, overwriting remote. Push metadata bundle. Requires confirmation.

### Resolution Option 3: Manual Merge (`--manual-merge`)

Interactive per-file conflict resolution:
- For each conflict, displays local hash/size vs remote hash/size
- User chooses (l)ocal or (r)emote for each file
- Resolution is applied via git checkout mechanics
- Supports `rgit merge --continue` and `rgit merge --abort`

---

## Design Decisions

### What We Chose

1. **Phantom-typed hashes**: `Hash (a :: HashAlgo)` Γאפ the compiler distinguishes MD5 from SHA256. Mixing algorithms is a compile error. The `DataKinds` extension is used per-module.

2. **Unified metadata parser**: A single `Rgit/Internal/Metadata.hs` module handles all parsing and serialization, eliminating the class of bugs where multiple parsers handle edge cases differently.

3. **`ReaderT RgitEnv IO` (no free monad)**: The application monad is `ReaderT RgitEnv IO`. A free monad effect system was considered (for testability and dry-run mode) but rejected as premature Γאפ no pure tests or dry-run usage existed to justify the complexity. Direct IO with `ReaderT` for environment threading is cleaner for now.

4. **Pure sync pipeline**: `diff` and `plan` are pure functions composed in `Rgit.Pipeline`. The intermediate `[GitDiff]` is preserved (not merged with `plan`) for display and testing. Property tests in `test/PipelineSpec.hs` verify the pipeline.

5. **Structured conflict resolution**: Conflict handling is a fold over a conflict list (`Rgit.Conflict`), not an imperative block. Decision logic is separated from IO mechanics.

6. **Remote as opaque type**: `Remote` is exported without its constructor. Only `remoteName` is public for display. `remoteUrl` exists for Transport to extract the URL, but business logic in Bit.hs should use `displayRemote` for user-facing messages.

7. **Tracking ref invariant**: `refs/remotes/origin/main` must always reflect
   what the remote actually has Γאפ never a local-only commit. After **push**,
   updating to HEAD is correct (the remote now has our history). After
   **pull/merge**, update to the hash from the fetched bundle, because HEAD
   includes merge commits the remote doesn't know about. Violating this
   causes the next `fetchFromBundle` to encounter a non-fast-forward update
   and silently fail to update the tracking ref, making subsequent merges
   operate against stale history.

8. **Git is the sole authority over `.rgit/index/`**: No code path should write
   metadata files to the index and commit them directly. The index is always
   populated by git operations (merge, checkout, commit via `rgit add`). After
   any git operation that changes HEAD, we only need to mirror those changes
   onto the actual working directory. This invariant applies to all pull paths
   including `--accept-remote`.

9. **Always commit when MERGE_HEAD exists**: After conflict resolution,
   `git commit` must always be called Γאפ never guarded by `hasStagedChanges`.
   When the user chooses "keep local" (`--ours`), the index becomes identical
   to HEAD. A `hasStagedChanges` check would skip the commit, leaving
   `MERGE_HEAD` dangling. Git's `commit` succeeds when `MERGE_HEAD` exists
   regardless of index state. Skipping the commit breaks the next push
   (ancestry check fails because HEAD was never advanced).

10. **The `oldHead` capture pattern**: Before any git operation that changes
    HEAD (merge, checkout), capture HEAD so `applyMergeToWorkingDir` can diff
    old vs new and sync only what changed. This pattern appears in `pullLogic`,
    `mergeContinue`, and `pullAcceptRemoteImpl`. The only exception is first
    pull (`oldHead = Nothing`), which falls back to `syncRemoteFilesToLocal`.

### What We Deliberately Do NOT Do

- **`RemoteState` does not need a typed state machine.** The pattern match in push logic is clear and total.
- **`FileIndex` does not need a Representable functor.** The dual indices (`byPath`/`byHash`) are an implementation detail.
- **`GitDiff` does not need a Group structure.** rgit computes diffs fresh each time; inverse/compose would be dead code.
- **No Arrow syntax.** Plain `>>=` and function composition are clearer.
- **No MTL-style type classes** (`MonadGit`, `MonadRclone`). Everything is concrete.
- **No post-sync metadata rescanning.** After syncing files to the working
  directory, we do NOT re-scan and rewrite metadata. Git already put the
  correct metadata in the index. Rescanning would be redundant at best,
  wrong at worst (e.g., overwriting text file content with hash/size if the
  scan's text/binary classification differs from git's).

---

## Known Deviations and TODOs

### Remaining Work

- **Atomic file writes**: Plain `writeFile` is used in several places. Should use temp file + rename for metadata and init files.
- **Text file handling**: Classification (`isText`) exists in the type system but text file sync (copy content to index, sync back on pull) needs completion.
- **Transaction logging**: For resumable push/pull operations.
- **Progress reporting**: For long operations (scanning, uploading).
- **Error messages**: Some need polish to match Git's style and include actionable hints.
- **`isTextFileInIndex` fragility**: The current check (looking for `"hash: "` prefix) works but is indirect. A more robust approach might check whether the file parses as metadata vs. has arbitrary content. Low priority since current approach works correctly.

### Future (rgit-solid)

- Content-Addressed Storage (CAS)
- Sparse checkout via symlinks to CAS blobs
- `rgit materialize` / `rgit checkout --sparse`

---

## Current Implementation State

### Implemented and Working

- `rgit init` Γאפ creates `.rgit/`, initializes Git in `.rgit/index/.git`
- `rgit add` Γאפ scans files, computes MD5 hashes, writes metadata, stages in Git
- `rgit commit`, `diff`, `status`, `log`, `restore`, `checkout`, `reset`, `rm`, `mv`, `branch`, `merge` Γאפ delegate to Git
- `rgit remote add/show/check` Γאפ named remotes with device-aware resolution
- `rgit push` Γאפ diff-based file sync via rclone, then push metadata bundle
- `rgit pull` Γאפ fetch metadata bundle, then diff-based file sync via `applyMergeToWorkingDir`
- `rgit pull --accept-remote` Γאפ force-checkout remote branch, then mirror changes to working directory
- `rgit pull --manual-merge` Γאפ interactive per-file conflict resolution
- `rgit merge --continue / --abort` Γאפ merge lifecycle management
- `rgit fetch` Γאפ fetch metadata bundle only
- `rgit verify` Γאפ local file verification against metadata
- `rgit verify --remote` Γאפ remote file verification against remote metadata
- `rgit fsck` Γאפ full integrity check
- Pipeline: pure diff Γזע plan Γזע action generation with property tests
- Device-identity system for filesystem remotes (UUID + hardware serial)
- Conflict resolution module with structured fold (always commits when MERGE_HEAD exists)
- Unified metadata parsing/serialization
- `oldHead` pattern for diff-based working-tree sync across all pull/merge paths

### Module Map

| Module | Role |
|--------|------|
| `Rgit/Commands.hs` | CLI dispatch, env setup |
| `Bit.hs` | All business logic |
| `Internal/Git.hs` | Git command wrapper |
| `Internal/Transport.hs` | Rclone command wrapper |
| `Internal/Config.hs` | Path constants |
| `Rgit/Types.hs` | Core types: Hash, FileEntry, RgitEnv, RgitM |
| `Rgit/Internal/Metadata.hs` | Canonical metadata parser/serializer |
| `Rgit/Scan.hs` | Working directory scanning, hash computation |
| `Rgit/Diff.hs` | Pure diff: FileIndex Γזע FileIndex Γזע [GitDiff] |
| `Rgit/Plan.hs` | Pure plan: GitDiff Γזע RcloneAction |
| `Rgit/Pipeline.hs` | Composed pipeline: diffAndPlan, pushSyncFiles, pullSyncFiles |
| `Rgit/Verify.hs` | Local and remote verification |
| `Rgit/Fsck.hs` | Full integrity check |
| `Rgit/Remote.hs` | Remote type, resolution, RemoteState, FetchResult |
| `Rgit/Remote/Scan.hs` | Remote file scanning via rclone |
| `Rgit/Device.hs` | Device identity, volume detection, .rgit-store |
| `Rgit/DevicePrompt.hs` | Interactive device setup prompts |
| `Rgit/Conflict.hs` | Conflict resolution: Resolution, resolveAll |
| `Rgit/Utils.hs` | Path utilities, filtering |

---

## Guardrails

**DO NOT:**
- Reintroduce a Manifest abstraction (we removed it intentionally)
- Store content in Git (only metadata or text files in the index)
- Use `rclone sync` Γאפ use action-based sync with explicit operations
- Add fields to metadata beyond `hash` and `size`
- Track symlinks or empty directories
- Implement CAS yet (that's rgit-solid, mark as TODO)
- Add MTL-style type classes or free monad effects (premature)
- Merge `diff` and `plan` into a single function
- Write metadata to `.rgit/index/` directly and then commit (bypasses git;
  rclone scans set `fIsText = False` for everything, producing wrong metadata
  for text files)
- Guard merge commits on `hasStagedChanges` when `MERGE_HEAD` exists (the
  commit must always be created to finalize the merge, even when the tree is
  unchanged Γאפ e.g., "keep local" resolution)
- Re-scan the working directory after sync to "fix" metadata (the index is
  already correct after the git operation; rescanning is redundant or harmful)

**ALWAYS:**
- Prefer `rclone moveto` over delete+upload when hash matches
- Push files before metadata, pull metadata before files
- Use temp file + rename for atomic writes (aspiration Γאפ not yet everywhere)
- Match Git's CLI conventions and output format
- Keep Transport dumb Γאפ no domain knowledge in Transport
- Keep Git.hs dumb Γאפ no domain interpretation
- All business logic in Bit.hs
- Use the unified metadata parser from `Rgit/Internal/Metadata.hs`
- After pull/merge, set refs/remotes/origin/main to the bundle hash, not HEAD
- Capture `oldHead` before any git operation that changes HEAD, then use
  `applyMergeToWorkingDir` to sync the working directory
- Let git manage `.rgit/index/` Γאפ all pull paths (normal, `--accept-remote`,
  `--manual-merge`, `mergeContinue`) must update the index via git operations
  (merge, checkout), never by writing files directly
- Always call `git commit` after conflict resolution when `MERGE_HEAD` exists
```

---

