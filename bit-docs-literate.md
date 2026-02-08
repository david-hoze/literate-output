# bit — Documentation (Literate Programming Document)

This document contains all documentation files for the bit project:
specifications, refactoring plans, and design documents.

---

## docs/spec.md

**Path:** `docs/spec.md`

*Source file.*

```markdown
# bit Implementation Specification (v3)

## Context and Vision

**bit** is a version control system designed for large files that leverages Git as a metadata-tracking engine while storing actual file content separately. The core insight: Git excels at tracking small text files, so we feed it exactly that — tiny metadata files instead of large binaries.

**Mental Model**: bit = Git(metadata) + rclone(sync) + [optional CAS(content) for bit-solid]

### bit-lite vs Git vs bit-solid: Content Authority

The three systems differ in where content lives and what guarantees that provides:

**Git** stores file content directly in its object store (`.git/objects/`). Every blob, tree, and commit is content-addressed by SHA-1. When you push, you're transferring objects that are self-verifying. When you pull, the objects you receive are self-verifying. Metadata (commits, trees) and content (blobs) live in the same store. The object store is the single source of truth, and it can always back up any metadata claim.

**bit-solid** (planned) will add a content-addressed store (CAS) alongside Git's metadata tracking. Like Git, every version of every file will be stored by its hash. The CAS backs up every metadata claim unconditionally — if the metadata says a file existed at commit N with hash X, the CAS has that blob. This enables full binary history, time travel, and sparse checkout.

**bit-lite** (current) has no object store and no CAS. Git tracks only metadata files (2-line hash+size records). The actual binary content lives exclusively in the working tree. There is exactly one copy of each file — the one on disk right now. Old versions are gone the moment you overwrite a file.

This creates a fundamental architectural constraint that Git and bit-solid don't have: **metadata can become hollow.** If a binary file is deleted, corrupted, or modified without running `bit add`, the metadata in `.bit/index/` claims something that is no longer true. In Git, this situation is impossible — the object store is append-only and self-verifying. In bit-lite, it's the normal consequence of working with mutable files on a regular filesystem.

This constraint gives rise to the **proof of possession** rule (see below): bit-lite must verify that content matches metadata before transferring metadata to or from another repo. Without this rule, hollow metadata propagates between repos, and the system's core value proposition — knowing the true state of your files — is undermined.

### Origin

The key architectural idea: instead of a custom manifest, keep small text files in `.bit/index/` mirroring the working tree's directory structure, each containing just hash and size. Define Git's working tree as `.bit/index/`, and you get `add`, `commit`, `diff`, `log`, branching, and the entire Git command set for free. Git becomes the manifest manager, diff engine, and history store — without ever seeing a large file.

### Comparison with Alternatives

bit occupies a different niche than git-lfs and git-annex:

- **git-lfs**: Stores pointer files in Git, actual files on a server. Server-dependent, transparent but limited. Pragmatic hack.
- **git-annex**: Extremely powerful distributed system with pluggable remotes and policy-driven placement. Extremely complex.
- **bit**: Minimal, explicit, correctness-oriented. Git never touches large files. Dumb remotes via rclone. Filesystem-first.

bit's killer feature is **clarity** — users always know what state their files are in, what bit is about to do, and how to recover from errors.

---

## Core Architecture

### Directory Structure

```
project/
├── actual_files/           # User's working directory (large files live here)
│   ├── src/
│   │   └── video.mp4
│   └── data/
│       └── dataset.bin
│
└── .bit/
    ├── index/              # Metadata files live here (Git working tree)
    │   ├── .git/           # Git's internal directory
    │   ├── src/
    │   │   └── video.mp4   # Metadata file (NOT the actual video)
    │   └── data/
    │       └── dataset.bin # Metadata file (NOT the actual data)
    ├── remotes/            # Named remote configs (device-aware)
    │   └── origin          # Remote target (cloud URL or device:path)
    ├── devices/            # Device identity files
    ├── target              # Legacy: single remote URL
    └── ignore              # Gitignore-style rules
```

### The Index Invariant

**Git is the sole authority over `.bit/index/`.** After any git operation that
changes HEAD (merge, checkout, commit), `.bit/index/` is correct by
definition. The only remaining work is mirroring those changes onto the actual
working directory (downloading binaries from remote, copying text files from
the index).

This invariant applies uniformly across all pull/merge paths:

- `git merge` → determines correct metadata → we sync actual files
- `git checkout` (first pull, `--accept-remote`) → determines correct metadata → we sync actual files

No path should write metadata files to `.bit/index/` directly and then commit.
Scanning the remote via rclone and writing the result to the index bypasses git
and **will** produce wrong metadata (rclone cannot distinguish text from binary,
so text files get `hash:/size:` metadata instead of their actual content).

### The Proof of Possession Rule

In Git and in bit-solid (planned), content is always recoverable from an object store. Git stores file content in `.git/objects/`; bit-solid will store it in a content-addressed store (CAS). In both systems, metadata and content are either the same thing or the content is always reachable from the metadata. You can push any metadata you want — the objects back it up unconditionally.

bit-lite is fundamentally different. There is no object store. The working tree is the **only copy** of binary file content. The metadata in `.bit/index/` is just a *claim* — "there exists a file at this path with this hash and this size." If that file is gone, corrupted, or modified since the last commit, the claim is hollow. Pushing hollow claims to a remote means the remote now has metadata that nobody can fulfill. Pulling hollow claims from a remote means your local repo has metadata pointing to content that doesn't exist.

This is the **proof of possession** rule: a repo must not transfer metadata it cannot back up with actual content.

```
             Git / bit-solid                    bit-lite
             ──────────────                     ────────
Content:     Object store (always available)    Working tree (only copy)
Metadata:    Always backed by objects           Only valid if working tree matches
Push safety: Unconditional                      Requires verification first
Pull safety: Unconditional                      Requires remote verification first
```

The rule applies symmetrically:

**Push (sender must prove possession):**
1. Verify local — every binary file's hash must match its metadata
2. If verification fails, refuse to push
3. If verified, push metadata then copy files

**Pull (sender must prove possession):**
1. Verify remote — every binary file on the remote must match the remote's metadata
2. If verification fails, refuse to pull (suggest `--accept-remote`, `--force`, or `--manual-merge`)
3. If verified, pull metadata then copy files

**What happens when verification fails:**

On push failure (local doesn't match metadata):
```
$ bit push
error: Working tree does not match metadata.
  Modified: data/model.bin (expected md5:a1b2..., got md5:f8e9...)
  Missing:  data/weights.bin
hint: Run 'bit verify' to see all mismatches.
hint: Run 'bit add' to update metadata, or 'bit restore' to restore files.
```

On pull failure (remote doesn't match its metadata):
```
$ bit pull
error: Remote files do not match remote metadata.
  Modified: data/model.bin (expected md5:a1b2..., got md5:c3d4...)
hint: Run 'bit verify --remote' to see all mismatches.
hint: Run 'bit pull --accept-remote' to accept the remote's actual file state.
hint: Run 'bit push --force' to overwrite remote with local state.
```

The existing divergence resolution mechanisms (`--accept-remote`, `--force`, `--manual-merge`) serve as the escape hatches when the proof of possession check fails.

**Why this matters:** Without this rule, corruption propagates silently through metadata. Repo A has a corrupted file, pushes metadata claiming the file is fine, Repo B pulls that metadata, and now both repos have a lie in their history. The proof of possession rule stops corruption at the boundary — you cannot export claims you can't substantiate.

**Cost:** Verification requires hashing every binary file, which means reading the entire working tree. For a repo with 100GB of binary files, this takes real time. This is the price of not having an object store — the working tree must be checked because it's the only source of truth. Future optimization: cache verification results keyed on (path, mtime, size) to skip re-hashing unchanged files.

**Remote verification cost by transport type:**

- **Cloud remotes (Google Drive, S3, etc.):** `rclone lsjson --hash` returns MD5 hashes as free metadata — Google Drive stores them natively. Verification is essentially a single API call followed by in-memory comparison. Cheap.
- **Cloud remotes (other backends):** Some backends provide native hashes, some don't. Where hashes aren't free, rclone may need to download file content to hash it. Check per-backend.
- **Filesystem remotes:** Requires reading and hashing every binary file on the remote. Same cost as local verification.

**Implementation status:** The proof of possession rule is fully implemented as of this version:

- `bit push` verifies local working tree before pushing (unless `--force` or `--skip-verify`)
- `bit pull` verifies remote before pulling (unless `--accept-remote`, `--manual-merge`, or `--skip-verify`)
- `bit fetch` does NOT verify (fetch only transfers metadata, no file sync happens)
- `bit fetch` is silent when already up to date (no stdout), similar to `git fetch`
- Cloud remotes: verified via `Verify.verifyRemote` using `rclone lsjson --hash`
- Filesystem remotes: verified via `Verify.verifyLocalAt` which hashes the remote's working tree
- Verification runs in parallel using bounded concurrency (`Parallel 0` = auto-detect based on CPU cores)

### Metadata File Format

Each metadata file under `.bit/index/` mirrors the path of its corresponding real file and contains **ONLY**:

```
hash: md5:a1b2c3d4e5f6...
size: 1048576
```

Two fields only:
- `hash` — MD5 hash of file content (prefixed with `md5:`)
- `size` — File size in bytes

Parsing and serialization are handled by a single canonical module (`bit/Internal/Metadata.hs`) which enforces the round-trip property: `parseMetadata . serializeMetadata ≡ Just`.

**Known deviation from original spec**: The original spec called for SHA256 and `hash=`/`size=` (equals-sign) format. The implementation uses MD5 (matching rclone's native hash) and `hash: `/`size: ` (colon-space) format. This is intentional — MD5 is used everywhere for consistency with rclone comparisons. SHA256 may be added later alongside MD5, not as a replacement.

### Git Configuration

bit runs Git with:
- Git repository initialized in `.bit/index/.git` during `bit init`
- All git operations use the repo at `.bit/index/.git`
- Working tree is `.bit/index` (not the project root)
- `core.excludesFile` points to `.bit/ignore`

### File Handling

- Regular files (binary → metadata, text → content stored directly in index)
- Symlinks — **ignored** (too many edge cases with cloud remotes)
- Device files, sockets, named pipes — **ignored**
- Empty directories — **ignored** (not tracked; many cloud backends don't support them)

---

## Module Architecture

### Layer Contract

The codebase follows strict layer boundaries:

```
bit/Commands.hs → Bit.hs → Internal/Transport.hs → rclone (only here!)
                      ↓
                   Internal/Git.hs → git (only here!)
```

- **Internal/Transport.hs** — Dumb rclone wrapper. Knows how to `copyTo`, `moveTo`, `deleteFile`, `listJson`, `check`. Takes `Remote` + relative paths. Does NOT know about `.bit/`, bundles, `RemoteState`, or `FetchResult`. Captures rclone JSON output as raw UTF-8 bytes to correctly handle non-ASCII filenames (Hebrew, Chinese, emoji, etc.). Uses `bracket` for exception-safe subprocess resource cleanup.
- **Internal/Git.hs** — Dumb git wrapper. Knows how to run git commands. Takes args. Does NOT interpret results in domain terms.
- **Bit.hs** — Smart business logic. All domain knowledge lives here. Knows about remotes, bundles, `.bit/` layout, sync strategy. Calls Transport and Git, never calls `readProcessWithExitCode` directly.
- **bit/Commands.hs** — Entry point. Parses CLI, resolves the remote, builds `BitEnv`, dispatches to `Bit`.

### Key Types

| Type | Module | Purpose |
|------|--------|---------|
| `Hash (a :: HashAlgo)` | `bit.Types` | Phantom-typed hash — compiler distinguishes MD5 vs SHA256 |
| `FileEntry` | `bit.Types` | Path + EntryKind (hash, size, isText) |
| `BitEnv` | `bit.Types` | Reader environment: cwd, local files, remote, force flags |
| `BitM` | `bit.Types` | `ReaderT BitEnv IO` — the application monad |
| `MetaContent` | `bit.Internal.Metadata` | Canonical metadata: hash + size, single parser/serializer |
| `Remote` | `bit.Remote` | Resolved remote: name + URL. Smart constructor, `remoteUrl` for Transport only |
| `RemoteState` | `bit.Remote` | Remote classification: Empty, ValidRgit, NonRgitOccupied, Corrupted, NetworkError |
| `FetchResult` | `bit.Remote` | Bundle fetch result: BundleFound, RemoteEmpty, NetworkError |
| `GitDiff` | `bit.Diff` | Added, Modified, Deleted, Renamed — pure diff result |
| `RcloneAction` | `bit.Plan` | Copy, Move, Delete, Swap — concrete rclone operations |
| `FileIndex` | `bit.Diff` | Dual-indexed file map (byPath + byHash) for efficient diff/rename detection |
| `Resolution` | `bit.Conflict` | KeepLocal or TakeRemote — conflict resolution choice |
| `DeviceInfo` | `bit.Device` | UUID + storage type + optional hardware serial |
| `RemoteTarget` | `bit.Device` | TargetCloud, TargetDevice, TargetLocalPath |

### Sync Pipeline

The sync pipeline is composed as pure function composition with effectful endpoints:

```
scan   :: FilePath → IO [FileEntry]        -- effectful (reads filesystem)
diff   :: FileIndex → FileIndex → [GitDiff]  -- pure!
plan   :: GitDiff → RcloneAction              -- pure!
exec   :: RcloneAction → IO ()               -- effectful (calls rclone)
```

The pure middle (`diff >>> plan`) is factored into `bit.Pipeline` and is fully property-testable:

```haskell
diffAndPlan :: [FileEntry] -> [FileEntry] -> [RcloneAction]  -- pure core
pushSyncFiles :: [FileEntry] -> [FileEntry] -> [RcloneAction]
pullSyncFiles :: [FileEntry] -> [FileEntry] -> [RcloneAction]
```

### Working Tree Sync: The `oldHead` Pattern

After any git operation that changes HEAD (merge, checkout), the working
directory must be updated to reflect what changed in `.bit/index/`.
The mechanism:

```haskell
oldHead <- getLocalHeadE                        -- 1. Capture HEAD *before* the git operation
-- ... git merge / git checkout ...             -- 2. Git changes HEAD + index
applyMergeToWorkingDir transport cwd oldHead    -- 3. Diff old HEAD vs new HEAD, sync files
```

`applyMergeToWorkingDir` uses `git diff --name-status oldHead newHead` to
determine what changed, then:
- **Added/Modified files**: download binary from remote, or copy text from index
- **Deleted files**: remove from working directory
- **Renamed files**: delete old, download/copy new

**CRITICAL**: `applyMergeToWorkingDir` always reads the actual HEAD after merge
from git (via `getLocalHeadE`). It never accepts `newHead` as a parameter. This
prevents the bug where `remoteHash` (the remote's tip) was passed instead of the
actual merged HEAD, causing local-only files to appear deleted during merge.

This is used consistently across:
- Clean merges (fast-forward and three-way)
- Conflict resolution merges
- `--accept-remote` (force-checkout)
- `mergeContinue`

The only exception is **first pull** (`oldHead = Nothing`): there is no
previous HEAD to diff against, so `transportSyncAllFiles` (full sync from
current HEAD after checkout) is used as fallback.

### File Transport Abstraction

To eliminate duplication between cloud and filesystem pull paths, the merge
orchestration logic is now unified with a `FileTransport` abstraction that
captures the differences in how files are copied:

```haskell
data FileTransport = FileTransport
  { transportDownloadFile :: FilePath -> FilePath -> SyncProgress -> IO ()
  , transportSyncAllFiles :: FilePath -> IO ()
  }
```

**Cloud transport** (rclone-based):
- `transportDownloadFile`: Downloads a single file via `rclone copyto`, or copies text from index
- `transportSyncAllFiles`: Scans remote via `rclone lsjson`, diffs against local, syncs files

**Filesystem transport** (direct file copy):
- `transportDownloadFile`: Copies a single file directly, or copies text from index
- `transportSyncAllFiles`: Uses `git ls-tree HEAD` after checkout, copies files from remote working tree

The unified `pullLogic` and `pullAcceptRemoteImpl` now accept a `FileTransport`
parameter. Both cloud and filesystem paths use the same merge orchestration,
`oldHead` capture pattern, conflict resolution (`Conflict.resolveAll`), and
tracking ref updates. The only difference is the transport used to copy files.

### Conflict Resolution

Conflict resolution is structured as a fold over a list of conflicts (`bit.Conflict`). Each conflict is resolved identically via `resolveConflict`, and the traversal guarantees every conflict is visited exactly once with correct progress tracking (1/N, 2/N, ...). The decision logic (KeepLocal vs TakeRemote) is cleanly separated from the git checkout/merge mechanics.

**Critical**: After resolving all conflicts, the merge commit must **always** be
created, regardless of whether the index has staged changes. When the user
chooses "keep local" (`--ours`), `git checkout --ours` + `git add` restores
HEAD's version — the index becomes identical to HEAD. A naïve `hasStagedChanges`
check would skip the commit, leaving `MERGE_HEAD` dangling. Git's `commit`
command always succeeds when `MERGE_HEAD` exists (it knows it's recording a
merge), even if the tree is identical to HEAD's tree. Skipping the commit
breaks the next push (ancestry check fails because HEAD was never advanced
past the merge).

---

## Command Line Interface (Git-Compatible)

**CRITICAL**: The CLI mirrors Git's interface. Users familiar with Git should feel immediately at home.

### Command Mapping

| Command | Git Equivalent | bit Behavior |
|---------|---------------|---------------|
| `bit init` | `git init` | Initialize `.bit/` with internal Git repo |
| `bit add <path>` | `git add` | Compute metadata, write to `.bit/index/`, stage in Git |
| `bit add .` | `git add .` | Add all modified/new files |
| `bit commit -m "msg"` | `git commit` | Commit staged metadata changes |
| `bit status` | `git status` | Show working tree vs metadata vs staged |
| `bit diff` | `git diff` | Show hash/size changes (human-readable) |
| `bit diff --staged` | `git diff --staged` | Show staged metadata changes |
| `bit log` | `git log` | Show commit history |
| `bit restore [options] [--] <path>` | `git restore` | Restore metadata; full git syntax: --staged, --worktree, --source=, etc. |
| `bit checkout [options] -- <path>` | `git checkout --` | Restore working tree from index (legacy syntax) |
| `bit reset` | `git reset` | Reset staging area |
| `bit rm <path>` | `git rm` | Remove file from tracking |
| `bit mv <src> <dst>` | `git mv` | Move/rename tracked file |
| `bit branch` | `git branch` | Branch management |
| `bit merge` | `git merge` | Merge branches |
| `bit remote add <name> <url>` | `git remote add` | Add named remote (does NOT set upstream) |
| `bit remote show [name]` | `git remote show` | Show remote status |
| `bit remote check [name]` | — | Check remote connectivity and state |
| `bit push [<remote>]` | `git push [<remote>]` | Push to specified or default remote |
| `bit push -u <remote>` | `git push -u <remote>` | Push and set upstream tracking |
| `bit push --set-upstream <remote>` | `git push --set-upstream <remote>` | Push and set upstream tracking (alt) |
| `bit pull [<remote>]` | `git pull [<remote>]` | Pull from specified or default remote |
| `bit pull <remote> --accept-remote` | — | Pull from remote, accept remote state |
| `bit pull --accept-remote` | — | Accept remote file state as truth |
| `bit pull --manual-merge` | — | Interactive per-file conflict resolution |
| `bit fetch [<remote>]` | `git fetch [<remote>]` | Fetch metadata from specified or default remote |
| `bit verify` | — | Verify local files match metadata |
| `bit verify --remote` | — | Verify remote files match remote metadata |
| `bit fsck` | `git fsck` | Full integrity check (local + remote + git) |
| `bit merge --continue` | `git merge --continue` | Continue after conflict resolution |
| `bit merge --abort` | `git merge --abort` | Abort current merge |
| `bit branch --unset-upstream` | `git branch --unset-upstream` | Remove tracking config |
| `bit @<remote> init` | — | Scan remote and build metadata workspace |
| `bit @<remote> add <path>` | — | Stage files in remote workspace |
| `bit @<remote> commit -m <msg>` | — | Commit and push metadata bundle to remote |
| `bit @<remote> status` | — | Show remote workspace status |
| `bit @<remote> log` | — | Show remote workspace history |

---

## Upstream Tracking (Git-Standard Behavior)

**IMPORTANT**: bit follows git's upstream tracking conventions exactly:

1. **`bit remote add <name> <url>` does NOT set upstream** — unlike old bit behavior, adding a remote (even "origin") does not auto-configure `branch.main.remote`. This matches git.

2. **`bit push -u <remote>` sets upstream** — the `-u` / `--set-upstream` flag pushes and configures `branch.main.remote = <remote>` in one operation. This is the standard way to establish upstream tracking.

3. **Commands accept explicit remote names**:
   - `bit push <remote>` — push to named remote (no tracking change)
   - `bit pull <remote>` — pull from named remote (no tracking change)
   - `bit fetch <remote>` — fetch from named remote (no tracking change)

4. **Default remote selection**:
   - If `branch.main.remote` is set, it's used as the default
   - If not set and "origin" exists, **`bit push` uses it as fallback** (git-standard behavior)
   - If not set and "origin" exists, **`bit pull` and `bit fetch` require explicit remote** (no fallback)
   - If neither upstream nor "origin", commands fail with error suggesting `bit push <remote>` or `bit push -u <remote>`

5. **First pull does NOT set upstream**: When pulling for the first time (unborn branch), `checkoutRemoteAsMain` uses `git checkout -B main --no-track refs/remotes/origin/main`. This prevents automatic upstream tracking setup. Users must use `bit push -u <remote>` to explicitly configure tracking.

6. **Internal git remote vs upstream tracking**: bit's internal git repo has a remote named "origin" (used for fetching refs from bundles), but this is distinct from upstream tracking config (`branch.main.remote`). The internal remote is set up automatically; upstream tracking is never automatic. This distinction is critical: `Git.setupRemote` configures the internal git remote (required for bundle operations), while `Git.setupBranchTracking` sets `branch.main.remote` (must only be called from `push -u`).

This makes bit's remote behavior predictable for git users: explicit tracking setup via `-u`, explicit remote selection via argument, sensible defaults when configured, and git-standard fallback to "origin" for push operations.

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
2. **Then**: Git operation (merge or checkout) updates `.bit/index/`
3. **Then**: Mirror index changes to working directory (download binaries, copy text from index)

**Rationale**: Push files first so the remote is never in a state where metadata references missing content. Pull metadata first so we know what content to fetch. After the git operation, the index is authoritative — we only need to bring actual files into alignment.

### Remote Types and Device Resolution

bit supports two kinds of remotes:

- **Cloud remotes**: rclone-based (e.g., `gdrive:Projects/foo`). Identified by URL. Uses bundle + rclone sync.
- **Filesystem remotes**: Local/network paths (USB drives, network shares, local directories). Creates a **full bit repository** at the remote location.

The `bit.Device` module handles filesystem remote resolution:
- Physical storage: Identified by UUID + hardware serial (survives drive letter changes)
- Network storage: Identified by UUID only
- Each volume can have a `.bit-store` file at its root containing its UUID
- Remote targets are stored in `.bit/remotes/<name>` as either cloud URLs or `device_name:relative_path`

The `bit.Remote` module provides the `Remote` type which abstracts over both:
```
resolveRemote :: FilePath -> String -> IO (Maybe Remote)
-- Tries .bit/remotes/<name> (device-aware), falls back to git config
```

### Transport Strategies

The transport strategy is determined by `RemoteTarget` classification:

```
Device.classifyRemotePath
  ├── TargetCloud    → Cloud transport (bundle + rclone, existing flow)
  ├── TargetDevice   → Filesystem transport (full repo at remote)
  └── TargetLocalPath → Filesystem transport (full repo at remote)
```

#### Cloud Transport (Bundle + Rclone)

For cloud remotes (Google Drive, S3, etc.), bit uses **dumb storage**:
- Metadata is serialized as a Git bundle and uploaded via rclone
- Files are synced via rclone copy/move/delete operations
- The remote is just a directory of files — no Git repo, no bit commands work there

#### Filesystem Transport (Full Repo)

For filesystem remotes, bit creates a **complete bit repository** at the remote:
- The remote has `.bit/index/.git/` just like a local repo
- Anyone at the remote location can run `bit status`, `bit log`, `bit commit`, etc.
- No bundles needed — Git can talk directly repo-to-repo via `git fetch /path/to/other/.bit/index/.git/`

**Key insight**: Bundles exist to serialize git history over dumb transports that can only copy files. With filesystem access, git speaks its native protocol.

#### Filesystem Push Flow

```
filesystemPush :: FilePath -> Remote -> IO ()
```

1. **First push (no `.bit/` at remote)**: Initialize a bit repo at the remote via `initializeRepoAt`
2. **Fetch local into remote**: `git -C remote/.bit/index fetch local/.bit/index/.git main:refs/remotes/origin/main`
3. **Fast-forward check**: Verify remote HEAD is ancestor of what we're pushing (`git merge-base --is-ancestor`)
4. **Merge at remote**: `git -C remote/.bit/index merge --ff-only refs/remotes/origin/main`
5. **Sync files**: Copy changed files from local working tree to remote working tree
   - Text files: Copy from remote's updated index (git put the content there)
   - Binary files: Copy from local working tree (metadata in index, content in working tree)
6. **Update tracking ref**: Set local `refs/remotes/origin/main` to current HEAD

If the fast-forward check fails, the remote has diverged:
```
error: Remote has local commits that you don't have.
hint: Run 'bit pull' to merge remote changes first, then push again.
```

#### Filesystem Pull Flow

```
filesystemPull :: FilePath -> Remote -> PullOptions -> IO ()
```

1. **Fetch remote into local**: `git -C local/.bit/index fetch remote/.bit/index/.git main:refs/remotes/origin/main`
2. **Proof of possession**: Verify remote working tree matches remote metadata (unless `--accept-remote` or `--skip-verify`)
3. **Build filesystem transport**: Create a `FileTransport` that copies files directly from remote working tree
4. **Merge locally**: Delegate to unified `pullLogic` or `pullAcceptRemoteImpl`
   - Note: Upstream tracking (`branch.main.remote`) is NOT auto-set; user must use `bit push -u <remote>`
5. **Sync files**: The unified logic uses the filesystem transport to copy files
   - Text files: Copy from local index (git merged the content there)
   - Binary files: Copy from remote working tree
6. **Update tracking ref**: Set `refs/remotes/origin/main` to the remote's HEAD hash

**Key change**: Filesystem pull now uses the same merge orchestration as cloud pull,
just with a different `FileTransport`. The merge follows the same patterns:
- First pull (unborn branch): `checkoutRemoteAsMain` (with `--no-track` to prevent auto-setting upstream) then `transportSyncAllFiles`
- Normal: `git merge --no-commit --no-ff` then `applyMergeToWorkingDir transport cwd oldHead`
- Conflicts: Same `Conflict.resolveAll` flow with (l)ocal/(r)emote choices
- `--accept-remote`: Force-checkout (with `--no-track`) then sync files via transport

This eliminates the previous duplication where `filesystemPullNormal` and
`filesystemPullAcceptRemote` reimplemented the merge flow separately.

#### Text vs Binary File Sync

For filesystem remotes, file sync distinguishes text from binary by examining the metadata file content:

```haskell
isTextMetadataFile :: FilePath -> IO Bool
-- Returns True if file exists and does NOT contain "hash: " line
-- Text files: metadata IS the content (stored directly in index)
-- Binary files: metadata contains "hash: " and "size: " (pointer to actual file)
```

- **Text files**: Content lives in `.bit/index/path`. After git merge/checkout, copy from index to working tree.
- **Binary files**: Content lives in working tree. Copy from source working tree to destination working tree.

#### The `git push` Antipattern

Do NOT use `git push` to a non-bare repo. Git refuses to update the checked-out branch:
```
error: refusing to update checked-out branch: refs/heads/main
```

The correct approach: Have the remote **fetch** from local, then **merge --ff-only**. This is what filesystem push does.

---

## Remote-Targeted Commands (`@<remote>` Syntax)

### Problem Statement

When a user has large amounts of data already on a remote (e.g., 10GB of files on Google Drive), the traditional workflow requires:
1. Download everything locally (expensive, time-consuming)
2. Run `bit init`, `bit add`, `bit commit` locally
3. Push everything back to the remote (redundant upload)

This is wasteful when the files are already at the destination. The user wants to **create metadata for files already on the remote** without downloading them.

### Solution: Remote Workspaces

The `@<remote>` syntax allows commands to operate against a remote as if it were a working directory, while only downloading small files (for text classification). Large binary files stay on the remote — bit just reads their hashes from `rclone lsjson --hash`.

### User Workflow

```bash
# Files already exist on gdrive:Projects/footage (10GB of video + some .txt/.md)
bit init                                    # local repo (empty working dir)
bit remote add origin gdrive:Projects/footage

bit @origin init                            # scan remote, build metadata workspace
bit @origin add .                           # stage all files
bit @origin commit -m "Initial commit"      # commit metadata, push bundle to remote

bit pull                                    # pull metadata locally (instant — just the bundle)
# Working dir is still empty, but bit knows about all 847 files
# User can then selectively download, or bit pull will sync everything
```

### Architecture

#### The `@<remote>` Prefix

`@<remote>` is parsed in `Commands.hs` before command dispatch. When present, it switches the execution context from "local working directory" to "remote-targeted workspace."

The remote name after `@` is resolved via the existing `resolveRemote` function, same as named remotes for push/pull.

#### Remote Workspace Structure

A remote workspace is a temporary `.bit/index` built from the remote's actual file state. It lives at `.bit/remote-workspaces/<remote-name>/` inside the local repo:

```
.bit/
├── index/              ← local metadata (normal)
├── remote-workspaces/
│   └── origin/         ← remote-targeted workspace
│       ├── .git/       ← git repo for this workspace
│       └── <metadata>  ← files mirroring remote structure
```

#### Text File Classification Without Full Download

The `rclone lsjson --hash` scan returns hashes for all files, but `fIsText = False` for everything (rclone can't classify text without reading content). bit's text classification needs:
1. File size (available from rclone scan)
2. File extension (available from path)
3. First 8KB of content (for NULL-byte and UTF-8 checks)

Strategy:
- Files above `textSizeLimit` (default 1MB) → binary, no download needed
- Files with `binaryExtensions` (`.mp4`, `.zip`, etc.) → binary, no download needed
- Remaining small files → download to temp dir, classify with existing `hashAndClassifyFile`

For a typical 10GB media repo, this downloads maybe 50KB of text files while skipping the 10GB of video.

### Supported Commands

| Command | Behavior |
|---------|----------|
| `bit @<remote> init` | Scan remote, classify files, build metadata workspace |
| `bit @<remote> add <path>` | Stage files in remote workspace (git add in workspace) |
| `bit @<remote> commit -m <msg>` | Commit workspace metadata, create bundle, push to remote |
| `bit @<remote> status` | Show remote workspace status (git status in workspace) |
| `bit @<remote> log` | Show remote workspace history (git log in workspace) |

All other commands are not supported in remote context (e.g., `bit @origin push` will error).

### Implementation Details

#### 1. `bit @origin init`

Located in `Bit.RemoteWorkspace.initRemoteWorkspace`:

1. Creates workspace directory structure at `.bit/remote-workspaces/<remote>/`
2. Runs `rclone lsjson --hash --recursive` on the remote (via `fetchRemoteFiles`)
3. Partitions files into "definitely binary" (by size/extension) and "maybe text" (candidates)
4. Downloads text candidates to a temp directory via rclone
5. Classifies downloaded files using existing `hashAndClassifyFile`
6. For text files: downloads actual content to workspace
7. For binary files: writes hash+size metadata
8. Initializes a git repo in the workspace (`git init --initial-branch=main`)

#### 2. `bit @origin add/commit/status/log`

These commands operate on the remote workspace git repository:

- `bit @origin add .` → `git -C .bit/remote-workspaces/origin add .`
- `bit @origin commit -m "msg"` → `git -C .bit/remote-workspaces/origin commit -m "msg"` + create bundle + push to remote
- `bit @origin status` → `git -C .bit/remote-workspaces/origin status`
- `bit @origin log` → `git -C .bit/remote-workspaces/origin log`

On commit, the workspace commits its metadata, then:
1. Creates a git bundle at `.bit/remote-workspaces/origin/.git/bit.bundle`
2. Uploads the bundle to the remote at `.bit/bit.bundle` via `rclone copyto`
3. Cleans up the local bundle file

The remote now has a `.bit/bit.bundle` that can be fetched via normal `bit pull`.

### Integration with Normal Pull

After `bit @origin commit`, the remote has a bundle at `.bit/bit.bundle`. A local `bit pull` will:
1. Fetch the bundle (existing `fetchBundle` logic)
2. Import it to local `.bit/index/.git/`
3. Merge the metadata
4. Sync files as needed (download from remote or copy from index)

The existing pull flow handles this transparently.

### Edge Cases

- **Workspace already exists**: Error with instructions (delete workspace or use `status`)
- **Remote is empty**: `fetchRemoteFiles` returns `[]`, workspace created but empty
- **Network failure during classification**: Failed downloads treated as binary, warning printed
- **Filesystem remotes**: `@<remote>` works for filesystem remotes too, though less useful (user could just `cd` there)
- **Multiple `@origin init` calls**: Error on second call, user must delete workspace first

### Limitations

This feature does NOT provide:
- **Selective file download** — that's a separate feature (sparse working tree). After `bit pull`, all files are expected locally.
- **Incremental remote re-scan** — `bit @origin init` always scans from scratch.
- **`bit @origin push`** — pushing *to* a remote workspace doesn't make sense. Push targets the actual remote.
- **Conflict resolution in remote context** — not needed. The workspace is single-writer (the local user).

---

## IO Safety and Concurrency

### Strict IO for Windows Compatibility

**Problem**: Lazy IO on Windows causes "permission denied" and "file is locked" errors due to file handles remaining open until garbage collection. When scanning hundreds of files concurrently, this manifests as random failures.

**Solution**: Eliminate all lazy IO operations and use strict `ByteString` operations exclusively.

### Implementation

#### 1. Strict IO Modules

**`Bit.ConcurrentFileIO`** — Drop-in replacements for `Prelude` file operations:
- `readFileBinaryStrict` — strict `ByteString.readFile` wrapper
- `readFileUtf8Strict` — strict UTF-8 text reading
- `readFileMaybe` / `readFileUtf8Maybe` — safe reading with `Maybe` return
- `writeFileBinaryStrict` / `writeFileUtf8Strict` — strict writing

All operations use `ByteString.readFile` / `ByteString.writeFile` which read/write the entire file and close the handle before returning.

**`Bit.Process`** — Strict process output capture:
- `readProcessStrict` — runs a process, strictly captures stdout and stderr as `ByteString`, returns `(ExitCode, ByteString, ByteString)`
- `readProcessStrictWithStderr` — runs a process with inherited stderr (for live progress), strictly captures stdout

Both functions:
- Use strict `Data.ByteString.hGetContents` (not lazy `System.IO.hGetContents`)
- Read stdout and stderr concurrently using `async` to avoid deadlocks when buffers fill
- Ensure all handles are closed and process is waited on in all code paths (using `bracket`)
- Prevent "delayed read on closed handle" errors that occur when using lazy IO with `createProcess`

**`Bit.ConcurrentIO`** — Type-safe concurrent IO newtype:
- Constructor is **not exported** to prevent `liftIO` smuggling
- Only whitelisted strict operations are exposed
- No `MonadIO` instance (intentional restriction)
- Provides `MonadUnliftIO` for `async` integration
- Includes concurrency primitives: `mapConcurrentlyBoundedC`, `QSem` operations

**`Bit.AtomicWrite`** — Atomic file writes with Windows retry logic:
- `atomicWriteFile` — temp file + rename pattern
- `DirWriteLock` — directory-level locking (MVar-based thread coordination)
- `LockRegistry` — process-wide lock registry for multiple workers
- Retry logic for Windows transient "permission denied" errors (up to 5 retries with exponential backoff)

#### 2. Module Updates

All lazy IO replaced with strict operations:
- `Bit/Scan.hs` — `.gitignore` reading uses strict ByteString
- `Bit/Device.hs` — `.bit-store`, device files, remote files use strict ByteString + atomic writes
- `Bit/Core.hs` — `readFileMaybe`, `writeFileAtomicE` (now truly atomic), metadata reading
- `Bit/Commands.hs` — `.bitignore` reading/writing uses strict ByteString + atomic writes
- `Internal/ConfigFile.hs` — Config reading uses strict ByteString instead of lazy Text IO

#### 3. HLint Enforcement

**`.hlint.yaml`** bans lazy IO functions project-wide:
- Banned: `Prelude.readFile`, `Prelude.writeFile`, `System.IO.hGetContents`, `System.IO.hGetLine`
- Banned: `Data.ByteString.Lazy.readFile`, `Data.Text.IO.readFile`
- Banned: Entire modules `Data.ByteString.Lazy`, `Data.Text.Lazy.IO`
- Suggests: `BS.readFile` over `readFile`, `atomicWriteFile` over `writeFile`
- Suggests: `Bit.Process.readProcessStrict` over `createProcess` for output capture

Process-specific rules:
- `System.IO.hGetContents` banned with message pointing to `Bit.Process.readProcessStrict`
- `System.IO.hGetLine` banned (reading in a loop is error-prone)

HLint errors appear in IDE and CI, preventing lazy IO from being reintroduced.

### Rationale

**Why strict ByteString?**
- Reads entire file into memory and closes handle immediately
- No lazy thunks keeping file handles open
- Predictable memory usage (acceptable for metadata files < 1MB)
- Cross-platform consistent behavior

**Why not lazy ByteString?**
- Lazy ByteString uses chunked IO, keeping handles open
- Garbage collection timing is unpredictable
- On Windows, this causes "file is locked" errors in concurrent scenarios

**Why atomic writes?**
- Crash safety — partial writes leave temp file, not corrupt destination
- Windows retry logic handles transient locking from antivirus/indexing
- Directory-level locking coordinates concurrent writes within process

**Why HLint rules?**
- Enforcement at development time (IDE warnings)
- Prevents lazy IO from being reintroduced during refactoring
- Documents the policy explicitly

### Concurrent File Scanning and Metadata Writing

The file scanner (`Bit/Scan.hs`) uses bounded parallelism for both scanning and writing:

**Scanning**:
- `QSem` limits concurrent file reads (default: `numCapabilities * 4`)
- Each file is fully read, hashed, and closed before moving to next
- Progress reporting uses `IORef` with `atomicModifyIORef'` for thread-safe updates
- Cache entries use strict ByteString read/write

**Metadata Writing** (`writeMetadataFiles`):
- Parallel execution with same bounded concurrency as scanning
- Skip-unchanged optimization: Before writing, checks if metadata already matches
  - Binary files: reads existing metadata, compares hash/size
  - Text files: compares mtime/size of source vs destination
- Three-phase write: (1) create directories sequentially, (2) create file parent directories, (3) write files in parallel
- Progress reporting shows files written, skipped count, and percentage
- Atomic writes for binary metadata using temp-file-rename pattern

### File Copy Progress Reporting

File copy operations during push/pull now have progress reporting (`Bit/CopyProgress.hs`):

**Implementation**:
- Chunked binary copy with byte-level progress (64KB chunks, strict `ByteString`)
- Small files (<1MB) use plain `copyFile` (no overhead), large files use chunked copy with progress
- Progress state (`SyncProgress`) tracks: total files, bytes total/copied, current file name
- Reporter thread updates at 100ms intervals via `IORef` (thread-safe, strict updates)
- TTY detection: shows in-place progress on terminal, silent on non-TTY (for log capture)

**Progress Display**:
- **Aggregate**: `Syncing files: 3/12 files, 5.1 GB / 18.3 GB (28%)`
- **Final summary**: `Synced 12 files (18.3 GB).`
- Human-readable byte formatting: `formatBytes` (B, KB, MB, GB, TB with 1 decimal place)

**Filesystem Remotes** (direct copy):
- `filesystemSyncAllFiles`, `filesystemSyncChangedFiles` (push)
- `filesystemSyncRemoteFilesToLocal`, `filesystemApplyMergeToWorkingDir` (pull)
- Progress: counts binary files only (text files are small and fast via index copy)
- Byte progress: sums file sizes from metadata before starting copy loop

**Cloud Remotes** (rclone):
- `syncRemoteFiles` (push), `syncRemoteFilesToLocal` (pull)
- Progress: file-count only (number of rclone actions completed / total)
- Simpler approach: tracks subprocess completion rather than byte-level progress

**Design Notes**:
- Complies with project's strict IO rules: no lazy `ByteString`, no lazy IO
- Windows compatible: uses `withBinaryFile` for chunked reads/writes
- Pattern matches existing scan progress in `Bit/Scan.hs`: `IORef` + reporter thread + TTY detection + `finally` cleanup

---

## Performance Optimizations

### Scan-on-Demand Architecture

**Problem**: The original design scanned the entire working directory *before* dispatching to a command, then maintained a growing `skipScan` whitelist of commands that don't need it. This was backwards:

1. **Wrong default** — new commands scan by default, silently wasting time if someone forgets to add them to `skipScan`
2. **Duplication** — the command must be listed in *both* `skipScan` and the `case` dispatch
3. **Fragile** — easy to miss commands (e.g., `remote add` was discovered to be missing from `skipScan`, causing 860 files to be scanned for a config-only operation)

**Solution**: Invert the logic — scan on demand, not by default. Commands are now classified into three tiers, with each command explicitly declaring its needs:

**Implementation** (`bit/Commands.hs`):

```haskell
-- Lightweight env (no scan) — for read-only commands
let baseEnv = do
        mRemote <- getDefaultRemote cwd
        return $ BitEnv cwd [] mRemote isForce isForceWithLease

-- Full env (scan + bitignore sync + metadata write) — for write commands
let scannedEnv = do
        syncBitignoreToIndex cwd
        localFiles <- Scan.scanWorkingDir cwd
        Scan.writeMetadataFiles cwd localFiles
        mRemote <- getDefaultRemote cwd
        return $ BitEnv cwd localFiles mRemote isForce isForceWithLease

case cmd of
    -- ── No env needed ────────────────────────────────────
    ["init"]              -> Bit.init
    ["remote", "add", ...] -> Bit.remoteAdd name url
    ["fsck"]              -> Bit.fsck cwd ...
    ["merge", "--abort"]  -> Bit.mergeAbort
    
    -- ── Lightweight env (no scan) ────────────────────────
    ("log":rest)          -> Bit.log rest >>= exitWith
    ("ls-files":rest)     -> Bit.lsFiles rest >>= exitWith
    ["remote", "show"]    -> baseEnv >>= \env -> runBitM env $ Bit.remoteShow Nothing
    ["verify"]            -> baseEnv >>= \env -> runBitM env $ Bit.verify False ...
    
    -- ── Full scanned env (needs working directory state) ─
    ("add":rest)          -> do _ <- scannedEnv; Bit.add rest >>= exitWith
    ("status":rest)       -> scannedEnv >>= \env -> runBitM env (Bit.status rest) >>= exitWith
    ("push":...)          -> scannedEnv >>= \env -> runBitM env Bit.push
    ("pull":...)          -> scannedEnv >>= \env -> runBitM env $ Bit.pull ...
```

**Key Changes**:

1. **No `skipScan` variable** — it no longer exists
2. **Lazy env builders** — `baseEnv` and `scannedEnv` are `IO BitEnv` actions, only executed when called
3. **Explicit tier assignment** — every command branch explicitly picks: no env, `baseEnv`, or `scannedEnv`
4. **Safe by default** — new commands default to not scanning (if you forget to call `scannedEnv`, you just get an empty `localFiles` list, which is harmless)
5. **Bitignore sync colocated with scan** — `syncBitignoreToIndex` only runs when `scannedEnv` is called, keeping related concerns together

**Command Classification**:

1. **No env needed**: Commands that operate on simple config or don't need any environment (`init`, `remote add`, `fsck`, `merge --abort`)

2. **Lightweight env (no scan)**: Read-only commands that read git history, index, or config without needing working directory state
   - `log`, `ls-files` — read git objects only
   - `remote show`, `remote check` — read config/connectivity
   - `verify`, `verify --remote` — compare against existing metadata

3. **Full scanned env**: Commands that need current working directory state
   - `status`, `restore`, `checkout` — need `localFiles` from env
   - `add`, `commit`, `diff` — need scan for side effects (metadata write), even though they don't use `localFiles` directly
   - `push`, `pull`, `fetch` — need working directory state for sync
   - `merge --continue` — needs working directory state to resolve conflicts

**Performance Impact**: In large repositories, read-only commands now have instant response times. The old design would scan 860 files for `bit remote add`, the new design scans zero.

---

## Verification and Consistency

### `bit verify`

Verifies local working tree matches local metadata. For each file in metadata, checks that the hash matches the actual file on disk.

**Progress reporting**: On TTY with >5 files, displays live progress: `Checking files: N/Total (X%)...`

### `bit verify --remote`

Verifies that remote files match what the remote Git bundle says they should be:
1. Fetch remote bundle (metadata)
2. Scan remote files via `rclone lsjson --hash`
3. Compare: Do actual remote files match what remote metadata claims?

**Progress reporting**: On TTY with >5 files, displays live progress during comparison phase.

### `bit fsck`

Full integrity check combining local verify, remote verify, and `git fsck` on the internal repository.

**Progress reporting**: On TTY with >5 files, displays step-by-step progress:
- `[1/2] Checking working tree: N/Total (X%)...`
- `[2/2] Checking metadata history...`

### `bit remote check`

Checks local working tree against remote using `rclone check --combined`. Reports files that match, differ, or are missing from either side. This is a lower-level check than verify — it compares actual file content between local and remote without involving metadata.

**Implementation**: Uses `rclone check --combined -` which outputs one line per file as it processes. For repos with >5 files, streams this output line-by-line to enable progress tracking.

**Progress reporting**: On TTY with >5 files, displays live progress: `Checking files: N/Total (X%)...`
- Counts files using `git ls-files` to get total expected file count upfront
- Streams rclone output using `createProcess` with pipes instead of blocking `readProcessWithExitCode`
- Updates progress counter via `IORef` as each line is received from rclone
- Progress reporter thread runs at 100ms intervals, cleared on completion

**Output**: Full comparison report saved to `.bit/last-check.txt` for detailed analysis.

---

## Handling Remote Divergence

When remote files don't match remote metadata (detected via `bit verify --remote` or during `bit pull`):

### Resolution Option 1: Accept Remote Reality (`--accept-remote`)

Force-checkout the remote branch so git puts the correct metadata in
`.bit/index/`, then mirror the changes to the working directory. This is
architecturally identical to a normal pull — just a force-checkout instead of
a merge. Git manages the index; we only sync actual files.

The flow:
1. Fetch remote bundle (git gets remote history)
2. Record current HEAD (for diff-based sync)
3. `git checkout -f -B main --no-track refs/remotes/origin/main` (force-checkout remote without setting upstream)
4. `applyMergeToWorkingDir` (diff old HEAD vs new HEAD, sync files)
5. Update tracking ref

**Important**: `--accept-remote` must NOT scan remote files via rclone and write
metadata directly. Rclone cannot distinguish text from binary files
(`fIsText = False` for everything), so text files would get `hash:/size:`
metadata instead of their actual content.

### Resolution Option 2: Force Local (`bit push --force`)

Upload all local files, overwriting remote. Push metadata bundle. Requires confirmation.

### Resolution Option 3: Manual Merge (`--manual-merge`)

Interactive per-file conflict resolution:
- For each conflict, displays local hash/size vs remote hash/size
- User chooses (l)ocal or (r)emote for each file
- Resolution is applied via git checkout mechanics
- Supports `bit merge --continue` and `bit merge --abort`

---

## Design Decisions

### What We Chose

1. **Phantom-typed hashes**: `Hash (a :: HashAlgo)` — the compiler distinguishes MD5 from SHA256. Mixing algorithms is a compile error. The `DataKinds` extension is used per-module.

2. **Unified metadata parser and loader**: A single `bit/Internal/Metadata.hs` module handles all parsing and serialization, and `Bit/Verify.hs` provides unified metadata loading via the `MetadataSource` abstraction. This eliminates the class of bugs where multiple parsers or loaders handle edge cases differently. The `MetadataEntry` type forces callers to explicitly handle the binary vs. text file distinction, ensuring text files (whose git blobs may have normalized line endings) are not incorrectly hash-verified across sources.

3. **`ReaderT BitEnv IO` (no free monad)**: The application monad is `ReaderT BitEnv IO`. A free monad effect system was considered (for testability and dry-run mode) but rejected as premature — no pure tests or dry-run usage existed to justify the complexity. Direct IO with `ReaderT` for environment threading is cleaner for now.

4. **Pure sync pipeline**: `diff` and `plan` are pure functions composed in `bit.Pipeline`. The intermediate `[GitDiff]` is preserved (not merged with `plan`) for display and testing. Property tests in `test/PipelineSpec.hs` verify the pipeline.

5. **Structured conflict resolution**: Conflict handling is a fold over a conflict list (`bit.Conflict`), not an imperative block. Decision logic is separated from IO mechanics.

6. **Remote as opaque type**: `Remote` is exported without its constructor. Only `remoteName` is public for display. `remoteUrl` exists for Transport to extract the URL, but business logic in Bit.hs should use `displayRemote` for user-facing messages.

7. **Tracking ref invariant**: `refs/remotes/origin/main` must always reflect
   what the remote actually has — never a local-only commit. After **push**,
   updating to HEAD is correct (the remote now has our history). After
   **pull/merge**, update to the hash from the fetched bundle, because HEAD
   includes merge commits the remote doesn't know about. Violating this
   causes the next `fetchFromBundle` to encounter a non-fast-forward update
   and silently fail to update the tracking ref, making subsequent merges
   operate against stale history.

8. **Git is the sole authority over `.bit/index/`**: No code path should write
   metadata files to the index and commit them directly. The index is always
   populated by git operations (merge, checkout, commit via `bit add`). After
   any git operation that changes HEAD, we only need to mirror those changes
   onto the actual working directory. This invariant applies to all pull paths
   including `--accept-remote`.

9. **Always commit when MERGE_HEAD exists**: After conflict resolution,
   `git commit` must always be called — never guarded by `hasStagedChanges`.
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

11. **Proof of possession on push/pull**: bit-lite must verify that the sender's
    working tree matches its metadata before transferring that metadata. On push,
    the local repo is verified; on pull, the remote is verified. This is necessary
    because bit-lite has no object store — the working tree is the only copy of
    binary content, and metadata without matching content is meaningless. Git and
    bit-solid don't need this rule because their object stores back up every claim
    unconditionally. The existing divergence resolution mechanisms
    (`--accept-remote`, `--force`, `--manual-merge`) serve as escape hatches when
    verification fails.

12. **Transport strategy split**: Push and pull dispatch based on `RemoteTarget`
    classification. Cloud remotes use bundle + rclone (dumb storage). Filesystem
    remotes use direct git fetch/merge (smart storage — full bit repo at remote).
    This split is keyed off `Device.classifyRemotePath` and happens in
    `Bit.Core.push` and `Bit.Core.pull`. **Merge orchestration is unified**: Both
    cloud and filesystem pull paths use the same `pullLogic` and
    `pullAcceptRemoteImpl` functions, parameterized by a `FileTransport` that
    abstracts how files are copied. The transport is the only difference — all
    `oldHead` capture, conflict resolution, tracking ref updates, and MERGE_HEAD
    handling is shared. This eliminates the duplication that previously existed
    where filesystem pull reimplemented the merge flow separately.

13. **Filesystem remotes are full repos**: When pushing to a filesystem path,
    bit creates a complete bit repository at the remote (via `initializeRepoAt`).
    Anyone at that location can run bit commands directly. This is the natural
    model for USB drives, network shares, and local collaboration directories.
    Bundles are skipped entirely — git talks repo-to-repo via filesystem paths.

14. **Upstream tracking requires explicit `-u` flag**: Following git-standard
    behavior, `branch.main.remote` is never set automatically. Users must use
    `bit push -u <remote>` to establish upstream tracking. This includes:
    - `bit remote add` does NOT set upstream (unlike old bit behavior)
    - First pull uses `git checkout -B main --no-track` to prevent auto-tracking
    - `bit push` falls back to "origin" if it exists and no upstream is configured
    - `bit pull` and `bit fetch` require explicit remote if no upstream is set
    This makes bit's remote behavior predictable for git users.

15. **Strict ByteString IO exclusively**: All file operations use strict
    `ByteString` (`Data.ByteString.readFile` / `writeFile`), never lazy IO
    (`Prelude.readFile`, `Data.ByteString.Lazy`, `Data.Text.Lazy.IO`). Lazy IO
    on Windows keeps file handles open until garbage collection, causing
    "permission denied" and "file is locked" errors in concurrent scenarios.
    Strict IO reads/writes the entire file and closes the handle immediately.
    HLint rules enforce this project-wide.

16. **Atomic writes with retry logic**: All important writes use the temp-file +
    rename pattern (`Bit.AtomicWrite.atomicWriteFile`). On Windows, includes
    retry logic (5 attempts with exponential backoff) to handle transient
    "permission denied" errors from antivirus and file indexing. Directory-level
    locking coordinates concurrent writers within the process.

17. **ConcurrentIO newtype without MonadIO**: For modules that need type-level
    IO restrictions, `Bit.ConcurrentIO` provides a newtype wrapper where the
    constructor is not exported. This prevents smuggling arbitrary lazy IO via
    `liftIO`. Only whitelisted strict operations are exposed. Used where the
    type system should enforce IO discipline (currently available but not widely
    adopted; `Bit.ConcurrentFileIO` with plain `MonadIO` is used in most places
    for simplicity).

### What We Deliberately Do NOT Do

- **`RemoteState` does not need a typed state machine.** The pattern match in push logic is clear and total.
- **`FileIndex` does not need a Representable functor.** The dual indices (`byPath`/`byHash`) are an implementation detail.
- **`GitDiff` does not need a Group structure.** bit computes diffs fresh each time; inverse/compose would be dead code.
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

- **Transaction logging**: For resumable push/pull operations.
- **Error messages**: Some need polish to match Git's style and include actionable hints.
- **`isTextFileInIndex` fragility**: The current check (looking for `"hash: "` prefix) works but is indirect. A more robust approach might check whether the file parses as metadata vs. has arbitrary content. Low priority since current approach works correctly.
- **Verification caching**: Cache verification results keyed on (path, mtime, size) to skip re-hashing unchanged files on subsequent push/pull operations. Would significantly speed up verification for large repos where most files haven't changed.

### Future (bit-solid)

- Content-Addressed Storage (CAS)
- Sparse checkout via symlinks to CAS blobs
- `bit materialize` / `bit checkout --sparse`

---

## Current Implementation State

### Implemented and Working

- `bit init` — creates `.bit/`, initializes Git in `.bit/index/.git`
- `bit add` — scans files, computes MD5 hashes, writes metadata, stages in Git
- `bit commit`, `diff`, `status`, `log`, `restore`, `checkout`, `reset`, `rm`, `mv`, `branch`, `merge` — delegate to Git
- `bit remote add/show/check` — named remotes with device-aware resolution
- `bit push` — Cloud: diff-based file sync via rclone, then push metadata bundle. Filesystem: fetch+merge at remote, then sync files
- `bit pull` — Cloud: fetch metadata bundle, then diff-based file sync via `applyMergeToWorkingDir`. Filesystem: fetch from remote, merge locally, sync files
- `bit pull --accept-remote` — force-checkout remote branch, then mirror changes to working directory
- `bit pull --manual-merge` — interactive per-file conflict resolution
- `bit merge --continue / --abort` — merge lifecycle management
- `bit fetch` — fetch metadata bundle only
- `bit verify` — local file verification against metadata
- `bit verify --remote` — remote file verification against remote metadata
- `bit fsck` — full integrity check
- Pipeline: pure diff → plan → action generation with property tests
- Device-identity system for filesystem remotes (UUID + hardware serial)
- Filesystem remote transport (full bit repo at remote, direct git fetch/merge)
- Conflict resolution module with structured fold (always commits when MERGE_HEAD exists)
- Unified metadata parsing/serialization
- `oldHead` pattern for diff-based working-tree sync across all pull/merge paths
- Strict ByteString IO throughout — no lazy IO, eliminates Windows file locking issues
- Atomic file writes with Windows retry logic — crash-safe, handles antivirus/indexing conflicts
- Concurrent file scanning with bounded parallelism and progress reporting
- HLint enforcement of IO safety rules
- Proof of possession verification for push and pull operations
  - `verifyLocalAt` function for verifying arbitrary repo paths (used for filesystem remotes)
  - `--skip-verify` flag to bypass verification when needed
  - Integration with existing escape hatches (`--force`, `--accept-remote`, `--manual-merge`)

### Module Map

| Module | Role |
|--------|------|
| `bit/Commands.hs` | CLI dispatch, env setup |
| `Bit.hs` | All business logic |
| `Internal/Git.hs` | Git command wrapper (`runGitAt` for arbitrary paths) |
| `Internal/Transport.hs` | Rclone command wrapper |
| `Internal/Config.hs` | Path constants |
| `Internal/ConfigFile.hs` | Config file parsing (strict ByteString) |
| `bit/Types.hs` | Core types: Hash, FileEntry, BitEnv, BitM |
| `bit/Internal/Metadata.hs` | Canonical metadata parser/serializer |
| `bit/Scan.hs` | Working directory scanning, hash computation, parallel metadata writing with skip-unchanged optimization (concurrent, strict IO) |
| `bit/Concurrency.hs` | Bounded parallelism helpers: concurrency level calculation, sequential/parallel mode switching |
| `bit/Diff.hs` | Pure diff: FileIndex → FileIndex → [GitDiff] |
| `bit/Plan.hs` | Pure plan: GitDiff → RcloneAction |
| `bit/Pipeline.hs` | Composed pipeline: diffAndPlan, pushSyncFiles, pullSyncFiles |
| `bit/Verify.hs` | Local and remote verification (parallelized); unified metadata loading via `MetadataSource` abstraction (`FromFilesystem`, `FromCommit`); `MetadataEntry` type distinguishes binary (hash-verifiable) from text files (existence-only); `verifyLocalAt` for filesystem remotes |
| `bit/Fsck.hs` | Full integrity check (parallelized) |
| `bit/Remote.hs` | Remote type, resolution, RemoteState, FetchResult |
| `bit/Remote/Scan.hs` | Remote file scanning via rclone |
| `bit/Device.hs` | Device identity, volume detection, .bit-store (strict IO, atomic writes) |
| `bit/DevicePrompt.hs` | Interactive device setup prompts |
| `bit/Conflict.hs` | Conflict resolution: Resolution, resolveAll |
| `bit/Utils.hs` | Path utilities, filtering, atomic write re-exports |
| `bit/AtomicWrite.hs` | Atomic file writes, directory locking, lock registry |
| `bit/ConcurrentIO.hs` | Type-safe concurrent IO newtype (no MonadIO) |
| `bit/ConcurrentFileIO.hs` | Strict ByteString file operations |
| `bit/Process.hs` | Strict process output capture (concurrent stdout/stderr reading) |
| `bit/Progress.hs` | Centralized progress reporting for terminal operations |
| `bit/CopyProgress.hs` | Progress tracking for file copy operations (push/pull sync) |

---

## Test Infrastructure

### Lint Test Suite (Pattern Safety + Format Validation)

**Purpose**: Prevent dangerous Windows environment variable patterns and shelltest format errors in test files.

**Two Categories of Violations**:

1. **Pattern Safety**: Dangerous Windows environment variables that can cause commands to escape test sandboxes
2. **Format Validation**: Shelltest Format 3 syntax violations that cause parse errors and prevent tests from running

**Problem 1 (Pattern Safety)**: Windows expands environment variables like `%CD%` before command chains execute. Example:
```batch
cd test\cli\work & bit remote add origin "%CD%\test\cli\remote_mirror"
```
If the `cd` fails, `%CD%` still expands to the current directory (potentially the main repo), causing `bit remote add` to modify the development repo's remote URL instead of the test repo's.

**Problem 2 (Format Validation)**: Shelltest Format 3 allows only one of each directive (`<<<`, `>>>`, `>>>2`, `>>>=`) per test case. Multiple directives cause parse errors that silently prevent tests from running. Example:
```batch
# WRONG - Two >>>2 directives:
command
>>>2 /error 1/
>>>2 /error 2/
>>>= 1
```
This causes a parse error and the test never executes, giving false confidence that the test is passing.

**Solution**: Automated guards enforcing both relative path usage and shelltest format rules.

#### Lint Test (Primary Guard)

**Location**: `test/LintTestFiles.hs`

**Test Suite**: `cabal test lint-tests`

**What it does**:
- Recursively scans all `.test` files under `test/cli/`
- **Pattern Safety Checks** (case-insensitive):
  - `%CD%` — current directory, expands before command execution
  - `%~dp0` — batch script directory variable
  - `%USERPROFILE%`, `%APPDATA%`, `%HOMEDRIVE%`, `%HOMEPATH%` — user directory variables
- **Format Validation Checks**:
  - Parses test cases (separated by blank lines)
  - Detects duplicate directives within a single test case:
    - Multiple `<<<` (stdin)
    - Multiple `>>>` (stdout)
    - Multiple `>>>2` (stderr)
    - Multiple `>>>=` (exit code)
  - Distinguishes between new directives and multi-line continuation
- Fails with detailed error showing file, line number, violation type, and fix
- Runs as part of `cabal test` and CI

**Example Output on Violation**:
```
DANGEROUS PATTERN in test/cli/remote-check.test:15
  Found: %CD%
  Line:  cd test\cli\work & bit remote add origin "%CD%\test\cli\remote_mirror"

  Why dangerous: Windows expands %CD% before the command chain executes.
  If the preceding `cd` fails, commands run in the main repo directory.
  Fix: Use relative paths (e.g., ..\remote_mirror) instead.
```

#### Pre-commit Hook (Secondary Guard)

**Location**: `scripts/pre-commit`

**Installation**: `scripts\install-hooks.bat` (run once after cloning)

**What it does**:
- Scans only staged `.test` files before commit
- Checks for the same dangerous patterns
- Blocks commit if violations found
- Provides clear error message with fix guidance

**Note**: Optional developer convenience; the lint test is the primary enforcement mechanism.

#### Forbidden Patterns List

The patterns are centrally defined in `test/LintTestFiles.hs`:
- `%CD%` — expands to current directory before command chains execute
- `%~dp0` — batch script directory, same timing issue
- `%USERPROFILE%` — could resolve to real user directories
- `%APPDATA%` — could resolve to real app data directories
- `%HOMEDRIVE%` / `%HOMEPATH%` — could resolve to real user paths

**Correct Pattern**: Use relative paths that resolve at command execution time:
```batch
# WRONG (banned):
cd test\cli\work & bit remote add origin "%CD%\test\cli\remote_mirror"

# CORRECT:
cd test\cli\work & bit remote add origin ..\remote_mirror
```

#### Documentation

See `test/cli/README.md` "Forbidden Patterns" section for detailed explanation and examples.

---

## Guardrails

**DO NOT:**
- Reintroduce a Manifest abstraction (we removed it intentionally)
- Store content in Git (only metadata or text files in the index)
- Use `rclone sync` — use action-based sync with explicit operations
- Add fields to metadata beyond `hash` and `size`
- Track symlinks or empty directories
- Implement CAS yet (that's bit-solid, mark as TODO)
- Add MTL-style type classes or free monad effects (premature)
- Merge `diff` and `plan` into a single function
- Write metadata to `.bit/index/` directly and then commit (bypasses git;
  rclone scans set `fIsText = False` for everything, producing wrong metadata
  for text files)
- Guard merge commits on `hasStagedChanges` when `MERGE_HEAD` exists (the
  commit must always be created to finalize the merge, even when the tree is
  unchanged — e.g., "keep local" resolution)
- Re-scan the working directory after sync to "fix" metadata (the index is
  already correct after the git operation; rescanning is redundant or harmful)
- Use `git push` to a filesystem remote (git refuses to update checked-out
  branches; use fetch+merge at the remote instead)
- Auto-set upstream tracking (`branch.main.remote`) on pull, fetch, or remote
  add operations — this must only be done via explicit `bit push -u <remote>`
- Use lazy IO (`Prelude.readFile`, `writeFile`, `hGetContents`, `Data.ByteString.Lazy`,
  `Data.Text.Lazy.IO`) — causes "file is locked" errors on Windows; use strict
  ByteString operations exclusively
- Use `createProcess` with `System.IO.hGetContents` for capturing process output —
  causes "delayed read on closed handle" errors; use `Bit.Process.readProcessStrict` instead
- Use plain `writeFile` for important files — use `atomicWriteFile` for crash safety
  and Windows compatibility
- Push metadata from an unverified working tree (the metadata may reference
  files that are missing or corrupted — see Proof of Possession rule)
- Pull metadata from an unverified remote (corruption propagates through
  metadata; verify remote first, suggest `--accept-remote` if verification fails)
- Create `.bit/` from non-init code paths — only `init` and `initializeRepoAt`
  may create `.bit/` from scratch. All other functions (`scanWorkingDir`,
  `writeMetadataFiles`, `saveCacheEntry`) must check that `.bit/` already
  exists and silently no-op if it doesn't. This prevents accidental `.bit/`
  directory creation in non-repo directories (e.g., from Windows `&`-chaining
  after a failed `cd`)

**ALWAYS:**
- Prefer `rclone moveto` over delete+upload when hash matches
- Push files before metadata, pull metadata before files (cloud remotes)
- For filesystem remotes, use fetch+merge (not `git push` to non-bare repos)
- Use `Bit.AtomicWrite.atomicWriteFile` for all important file writes (temp file + rename pattern)
- Use strict `ByteString` operations for all file IO — never `Prelude.readFile`, `writeFile`, or lazy ByteString/Text
- Use `Bit.ConcurrentFileIO.readFileBinaryStrict` / `readFileUtf8Strict` for reading
- Match Git's CLI conventions and output format
- Keep Transport dumb — no domain knowledge in Transport
- Keep Git.hs dumb — no domain interpretation
- All business logic in Bit.hs
- Use the unified metadata parser from `bit/Internal/Metadata.hs`
- After pull/merge, set refs/remotes/origin/main to the bundle hash, not HEAD
- Capture `oldHead` before any git operation that changes HEAD, then use
  `applyMergeToWorkingDir` to sync the working directory
- Let git manage `.bit/index/` — all pull paths (normal, `--accept-remote`,
  `--manual-merge`, `mergeContinue`) must update the index via git operations
  (merge, checkout), never by writing files directly
- Always call `git commit` after conflict resolution when `MERGE_HEAD` exists
- Update tracking ref after filesystem pull (same invariant as cloud pull)
- Use `--no-track` flag for any `git checkout` that should not set upstream
  tracking (e.g., `checkoutRemoteAsMain`, `--accept-remote` flows)
- Verify local working tree before push (proof of possession)
- Verify remote before pull (proof of possession); for cloud remotes use
  `rclone lsjson --hash` (free on Google Drive), for filesystem remotes hash
  the files
- When verification fails, refuse the operation and suggest resolution:
  `bit add` / `bit restore` for local issues, `--accept-remote` / `--force` /
  `--manual-merge` for remote issues
```

---

