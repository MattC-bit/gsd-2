# GSD-2 Git and Isolation Model

This document traces the complete isolation flow from git primitives through worktree management to execution isolation modes and recovery mechanisms. It explains why each layer exists and how they compose to provide safe, isolated execution for autonomous coding agents.

> **Subject-vs-Runner Guardrail**: This pack analyzes GSD-2 as a *subject system*. References to `.gsd/`, milestones, slices, and workflow state in this document refer to the **analyzed GSD-2 system**, not the live runner state producing this documentation. See [GSD2_SYSTEM_OVERVIEW.md](./GSD2_SYSTEM_OVERVIEW.md#subject-vs-runner-guardrail) for the canonical definition.

---

## Git Operations Layer

The Git Operations Layer provides the foundational abstraction that worktree isolation and milestone management build upon. It consists of three main components: GitServiceImpl (the high-level API), the native git bridge (libgit2 integration with CLI fallback), and integration branch management.

### GitServiceImpl Overview

`GitServiceImpl` is the primary interface for git operations in GSD [repo code]. It wraps the native git bridge with workflow-aware methods that understand GSD's isolation model and milestone context.

**Location**: `src/resources/extensions/gsd/git-service.ts` [repo code]

**Constructor and State**:
- Takes a `basePath` (repo root) and optional `GitPreferences` [repo code]
- Maintains a private `_milestoneId` for integration branch resolution [repo code]
- Tracks `_runtimeFilesCleanedUp` to avoid repeated index cleanup [repo code]

**Key Methods**:

| Method | Purpose |
|--------|---------|
| `commit(opts)` | Smart staging with runtime exclusion, then commit [repo code] |
| `autoCommit(unitType, unitId, extraExclusions, taskContext)` | Auto-commit dirty working tree with meaningful message [repo code] |
| `getMainBranch()` | Resolve integration branch with preference → milestone → worktree → auto-detect order [repo code] |
| `getCurrentBranch()` | Get current branch via native libgit2 or CLI fallback [repo code] |
| `createSnapshot(label)` | Create recovery ref under `refs/gsd/snapshots/` [repo code] |
| `runPreMergeCheck()` | Execute pre-merge verification (tests, custom command) [repo code] |

**Smart Staging**: The `commit()` and `autoCommit()` methods use `smartStage()` which stages all changes via `git add -A` while excluding GSD runtime paths [repo code]. This prevents transient files (activity logs, lock files, database) from entering version control [repo code].

**Runtime Exclusions**: The `RUNTIME_EXCLUSION_PATHS` constant lists paths that should never be staged [repo code]:
- `.gsd/activity/`, `.gsd/runtime/`, `.gsd/worktrees/` [repo code]
- `.gsd/auto.lock`, `.gsd/metrics.json`, `.gsd/completed-units.json` [repo code]
- `.gsd/STATE.md`, `.gsd/gsd.db`, `.gsd/DISCUSSION-MANIFEST.json` [repo code]

**One-Time Cleanup**: The `smartStage()` method performs a one-time cleanup of runtime files that may have been tracked by older GSD versions [repo code]. This removes them from the index via `nativeRmCached()` and commits the cleanup separately to avoid contaminating work commits [repo code].

---

### Native Git Bridge

The native git bridge provides high-performance git operations backed by libgit2 via the Rust native module (`@gsd/native`), with automatic fallback to CLI commands when the native module is unavailable [repo code].

**Location**: `src/resources/extensions/gsd/native-git-bridge.ts` [repo code]

#### Native Module Loading

The bridge attempts to load `@gsd/native` once per session [repo code]. Loading is gated by the `GSD_ENABLE_NATIVE_GSD_GIT=1` environment variable [repo code]. When disabled or unavailable, all functions fall back to `execFileSync("git", ...)` calls [repo code].

```typescript
// Issue #453: keep auto-mode bookkeeping on the stable git CLI path unless a
// caller explicitly opts into the native helper.
const NATIVE_GSD_GIT_ENABLED = process.env.GSD_ENABLE_NATIVE_GSD_GIT === "1";
```
[repo code]

#### Function Categories

**Read Functions** (query git state without side effects):

| Function | Native | Fallback |
|----------|--------|----------|
| `nativeGetCurrentBranch` | libgit2 HEAD ref | `git branch --show-current` |
| `nativeDetectMainBranch` | libgit2 refs check | `git symbolic-ref` + `git show-ref` |
| `nativeBranchExists` | libgit2 refs/heads check | `git show-ref --verify` |
| `nativeHasChanges` | libgit2 status | `git status --short` (10s cache) |
| `nativeHasStagedChanges` | libgit2 diff | `git diff --cached --stat` |
| `nativeHasMergeConflicts` | libgit2 index check | `git diff --name-only --diff-filter=U` |
| `nativeWorkingTreeStatus` | libgit2 status porcelain | `git status --porcelain` |
| `nativeCommitCountBetween` | libgit2 revwalk | `git rev-list --count` |
| `nativeDiffStat` | libgit2 diff stats | `git diff --stat` |
| `nativeDiffNameStatus` | libgit2 tree diff | `git diff --name-status` |
| `nativeDiffNumstat` | libgit2 line stats | `git diff --numstat` |
| `nativeDiffContent` | libgit2 diff print | `git diff` |
| `nativeLogOneline` | libgit2 revwalk | `git log --oneline` |
| `nativeWorktreeList` | libgit2 worktree API | `git worktree list --porcelain` |
| `nativeBranchList` | libgit2 branch iterator | `git branch --list` |
| `nativeBranchListMerged` | libgit2 merge-base check | `git branch --merged` |
| `nativeLsFiles` | libgit2 index iteration | `git ls-files` |
| `nativeForEachRef` | libgit2 references | `git for-each-ref` |
| `nativeConflictFiles` | libgit2 index conflicts | `git diff --name-only --diff-filter=U` |
| `nativeBatchInfo` | single libgit2 call | multiple git commands |
| `nativeIsRepo` | libgit2 Repository::open | `git rev-parse --git-dir` |
| `nativeIsAncestor` | libgit2 merge-base | `git merge-base --is-ancestor` |
| `nativeLastCommitEpoch` | libgit2 commit time | `git log -1 --format=%ct` |
| `nativeUnpushedCount` | libgit2 revwalk | `git rev-list --count` |

[repo code]

**Write Functions** (mutate repository state):

| Function | Native | Fallback |
|----------|--------|----------|
| `nativeInit` | libgit2 Repository::init | `git init -b` |
| `nativeAddAll` | libgit2 index add_all | `git add -A` |
| `nativeAddAllWithExclusions` | CLI only (pathspec exclusions) | `git add -A -- ':!pattern'` |
| `nativeAddPaths` | libgit2 index add | `git add --` |
| `nativeResetPaths` | libgit2 reset_default | `git reset HEAD --` |
| `nativeCommit` | libgit2 commit create | `git commit --no-verify -F -` |
| `nativeCheckoutBranch` | libgit2 checkout + set_head | `git checkout` |
| `nativeCheckoutTheirs` | libgit2 conflict resolution | `git checkout --theirs --` |
| `nativeMergeSquash` | libgit2 merge squash | `git merge --squash` |
| `nativeMergeAbort` | libgit2 reset + cleanup | `git merge --abort` |
| `nativeRebaseAbort` | libgit2 reset + cleanup | `git rebase --abort` |
| `nativeResetHard` | libgit2 reset(Hard) | `git reset --hard HEAD` |
| `nativeBranchDelete` | libgit2 branch delete | `git branch -D/-d` |
| `nativeBranchForceReset` | libgit2 branch create force | `git branch -f` |
| `nativeRmCached` | libgit2 index remove | `git rm --cached -r` |
| `nativeRmForce` | libgit2 index + fs delete | `git rm --force` |
| `nativeWorktreeAdd` | libgit2 worktree API | `git worktree add` |
| `nativeWorktreeRemove` | libgit2 worktree prune | `git worktree remove` |
| `nativeWorktreePrune` | libgit2 worktree validation | `git worktree prune` |
| `nativeRevertCommit` | libgit2 revert | `git revert --no-commit` |
| `nativeRevertAbort` | libgit2 reset + cleanup | `git revert --abort` |
| `nativeUpdateRef` | libgit2 reference create/delete | `git update-ref` |

[repo code]

#### Pathspec Exclusion Handling

The `nativeAddAllWithExclusions()` function always uses the CLI path because libgit2's `add_all` does not support pathspec exclusion syntax [repo code]. This prevents staging of large untracked artifact trees (observed: 57GB+, 11K+ files) that would hang git [repo code].

**Special Cases**:
- When excluded paths are already covered by `.gitignore`, git may exit with code 1 and an "ignored by .gitignore" warning — this is suppressed as harmless [repo code]
- When `.gsd` is a symlink, git rejects `:!.gsd/...` pathspecs with "beyond a symbolic link" — falls back to plain `git add -A` which respects `.gitignore` [repo code]

#### Caching

The `nativeHasChanges()` fallback implements a 10-second TTL cache per `basePath` [repo code]. This avoids repeated `git status --short` calls during quick polling loops when the native module is unavailable [repo code].

---

### Integration Branch Management

Integration branch management determines the target branch for slice merges — the branch that slice branches are created from and merged back into [repo code]. This is often `main` or `master`, but users may start GSD on a feature branch, making that the integration target instead [repo code].

**Location**: `src/resources/extensions/gsd/git-service.ts` [repo code]

#### Resolution Order

The `getMainBranch()` method resolves the integration branch in this priority order [repo code]:

1. **Explicit `main_branch` preference** (user override, highest priority) [repo code]
2. **Milestone integration branch** from metadata file (recorded at milestone start) [repo code]
3. **Worktree base branch** (`worktree/<name>` if in a worktree) [repo code]
4. **Auto-detected default** (`origin/HEAD` → `main` → `master` → current branch) [repo code]

The method uses `setMilestoneId()` to enable milestone-aware resolution [repo code].

#### Metadata Persistence

Integration branches are persisted in milestone metadata files [repo code]:

```
.gsd/milestones/<MID>/<MID>-META.json
```

The `readIntegrationBranch()` function reads the `integrationBranch` field from this JSON file [repo code]. The `writeIntegrationBranch()` function records the branch when auto-mode starts [repo code].

**Branch Exclusions**:
- Slice branches matching `SLICE_BRANCH_RE` are not recorded [repo code]
- Quick-task branches matching `QUICK_BRANCH_RE` (`gsd/quick/...`) are not recorded [repo code]
- Invalid branch names (failing `VALID_BRANCH_NAME` regex) are rejected [repo code]

**Idempotency**: `writeIntegrationBranch()` is idempotent — if the same branch is already recorded, it returns without I/O [repo code]. If a different branch is recorded (user started from a new branch), it updates the record [repo code].

#### Resolution Status Helper

The `resolveMilestoneIntegrationBranch()` function provides detailed resolution status [repo code]:

| Status | Meaning |
|--------|---------|
| `recorded` | Recorded branch exists and is usable |
| `fallback` | Recorded branch missing, using fallback |
| `missing` | No usable branch found |

This helper is scoped to milestones with existing metadata [repo code]. It attempts fallback in this order when the recorded branch no longer exists [repo code]:
1. Configured `git.main_branch` preference [repo code]
2. Auto-detected main branch via `nativeDetectMainBranch()` [repo code]

---

### Commit Message Generation

GSD generates meaningful conventional commit messages from task execution context rather than using generic messages [repo code].

**Location**: `src/resources/extensions/gsd/git-service.ts` [repo code]

#### TaskCommitContext Interface

```typescript
interface TaskCommitContext {
  taskId: string;        // e.g. "S01/T02"
  taskTitle: string;     // Planned task description
  oneLiner?: string;     // What was actually built (from summary)
  keyFiles?: string[];   // Files modified by this task
  issueNumber?: number;  // GitHub issue for trailer
}
```
[repo code]

#### Message Format

The `buildTaskCommitMessage()` function produces conventional commits [repo code]:

```
{type}({scope}): {description}

- file1.ts
- file2.ts

Resolves #123
```

**Components**:
- **Type**: Inferred from title and one-liner keywords via `inferCommitType()` [repo code]
- **Scope**: The `taskId` (e.g., "S01/T02" or just "T02") [repo code]
- **Description**: The `oneLiner` if available (what was built), otherwise `taskTitle` (what was planned) [repo code]
- **Body**: Key files (capped at 8) and issue trailer if present [repo code]

**Subject Truncation**: Description is truncated to ~72 characters for the subject line [repo code].

#### Commit Type Inference

The `inferCommitType()` function maps keywords to conventional commit types [repo code]:

| Keywords | Type |
|----------|------|
| fix, fixed, fixes, bug, patch, hotfix, repair, correct | `fix` |
| refactor, restructure, reorganize | `refactor` |
| doc, docs, documentation, readme, changelog | `docs` |
| test, tests, testing, spec, coverage | `test` |
| perf, performance, optimize, speed, cache | `perf` |
| chore, cleanup, clean up, dependencies, deps, bump, config, ci, archive, remove, delete | `chore` |

[repo code]

Keywords are matched case-insensitively with word boundaries [repo code]. Multi-word phrases like "clean up" use substring matching [repo code]. The default type is `feat` when no keywords match [repo code].

---

## Worktree Isolation Model

The Worktree Isolation Model provides the core mechanism GSD uses for milestone execution isolation. It builds on the Git Operations Layer to create isolated working directories where autonomous agents can safely make changes without affecting the project root until merge time.

### Worktree Lifecycle Overview

GSD manages git worktrees under `.gsd/worktrees/<name>/` with each worktree getting its own branch for parallel work streams [repo code]. The lifecycle consists of three phases:

1. **Create** — `git worktree add .gsd/worktrees/<name> -b <branch>` creates an isolated working copy [repo code]
2. **Work** — Agent operates in the worktree with full isolation from project root [repo code]
3. **Teardown** — Merge back to integration branch, then remove worktree and branch [repo code]

**Location**: `src/resources/extensions/gsd/worktree-manager.ts` [repo code]

#### Core Operations

| Function | Purpose |
|----------|---------|
| `createWorktree()` | Create new worktree with branch, handle stale directories [repo code] |
| `listWorktrees()` | List all GSD-managed worktrees under `.gsd/worktrees/` [repo code] |
| `removeWorktree()` | Remove worktree and optionally delete its branch [repo code] |
| `worktreePath()` | Resolve path `.gsd/worktrees/<name>/` [repo code] |
| `worktreeBranchName()` | Generate branch name `worktree/<name>` [repo code] |

**Stale Directory Handling**: If a worktree directory exists without a valid `.git` file (file containing `gitdir:` pointer), it's removed as a stale leftover from a prior crash [repo code]. This prevents crashes from blocking future worktree creation.

**Auto-Chdir on Removal**: When removing a worktree, if the current process is inside it, `process.chdir()` moves to the base path first — git cannot remove an in-use directory [repo code].

---

### Branch Naming Conventions

GSD uses distinct branch naming patterns for different isolation contexts.

#### Manual Worktrees

Manual worktrees (created via `/worktree` command) use the `worktree/<name>` pattern [repo code]:

```
worktree/my-feature
worktree/bugfix-123
```

The branch name is derived directly from the worktree name [repo code]:
```typescript
export function worktreeBranchName(name: string): string {
  return `worktree/${name}`;
}
```

#### Auto-Mode Milestones

Auto-mode worktrees use the `milestone/<MID>` pattern to distinguish them from manual worktrees [repo code]:

```
milestone/M001
milestone/M002-r5jzab
```

The `autoWorktreeBranch()` function generates these names [repo code]:
```typescript
export function autoWorktreeBranch(milestoneId: string): string {
  return `milestone/${milestoneId}`;
}
```

#### Slice Branches

Slice branches are namespaced by worktree context to prevent conflicts when multiple worktrees work on the same milestone/slice IDs [repo code]:

| Context | Pattern |
|---------|---------|
| Main tree | `gsd/<milestoneId>/<sliceId>` |
| In worktree | `gsd/<worktreeName>/<milestoneId>/<sliceId>` |

Git doesn't allow a branch to be checked out in more than one worktree simultaneously [repo code]. The worktree-namespaced pattern prevents this conflict.

**Detection**: The `SLICE_BRANCH_RE` regex matches both patterns and extracts components [repo code]:
```typescript
export const SLICE_BRANCH_RE = /^gsd\/(?:([a-zA-Z0-9_-]+)\/)?(M\d+(?:-[a-z0-9]{6})?)\/(S\d+)$/;
```

---

### Auto-Worktree for Milestones

Auto-mode creates worktrees with `milestone/<MID>` branches (distinct from manual `/worktree` which uses `worktree/<name>` branches) [repo code]. This module manages the complete lifecycle for auto-mode isolation.

**Location**: `src/resources/extensions/gsd/auto-worktree.ts` [repo code]

#### createAutoWorktree

Creates a new auto-worktree for a milestone, enters it via `process.chdir()`, and stores the original base path for later teardown [repo code].

**Branch Existence Check**: If the milestone branch already exists, the worktree re-attaches to it WITHOUT resetting — preserving committed work from prior sessions [repo code]. Only creates a fresh branch from the integration branch when no prior work exists.

**Planning Artifact Copy**: Worktrees are fresh git checkouts — untracked files don't carry over. The function copies `.gsd/` planning artifacts from the source repo [repo code]:
- `milestones/` directory (plans, roadmaps, research)
- `DECISIONS.md`, `REQUIREMENTS.md`, `PROJECT.md`, `QUEUE.md`, `STATE.md`, `KNOWLEDGE.md`, `OVERRIDES.md`
- `gsd.db` database (if present)

**Post-Create Hook**: Runs user-configured `git.worktree_post_create` hook script after creation (e.g., to copy `.env` files, symlink assets) [repo code].

#### enterAutoWorktree

Enters an existing auto-worktree for resume scenarios [repo code]. Validates that the path is a real git worktree (has a `.git` file with `gitdir:` pointer) rather than a stray directory [repo code].

#### teardownAutoWorktree

Teardown sequence [repo code]:
1. `process.chdir()` back to original base path
2. Remove worktree via `git worktree remove`
3. Delete milestone branch (unless `preserveBranch: true`)
4. Clear module state

**Cleanup Verification**: Warns if the worktree directory still exists after teardown (observed on Windows with bash-based cleanup) and attempts direct filesystem removal as fallback [repo code].

#### mergeMilestoneToMain

Squash-merges the milestone branch into main with a rich commit message [repo code].

**Merge Sequence** [repo code]:
1. Auto-commit dirty worktree state
2. Reconcile worktree DB into main DB
3. Parse roadmap for slice listing
4. `chdir` to original base path
5. Resolve integration branch (milestone metadata → preference → auto-detect)
6. Clear transient project-root state files
7. Checkout integration branch
8. Build rich commit message with completed slices
9. Fast-forward branch ref if worktree HEAD is ahead (#1846)
10. Squash merge with `.gsd/` conflict auto-resolution
11. Commit (handle "nothing to commit" gracefully)
12. Auto-push if enabled
13. Auto-create PR if enabled
14. Remove worktree directory
15. Delete milestone branch

**Safety Check**: If squash merge produces "nothing to commit", verifies the milestone code is already on the integration branch before allowing teardown [repo code]. Prevents silent data loss when unanchored code changes would be orphaned.

---

### WorktreeResolver

The `WorktreeResolver` class encapsulates worktree path state and merge/exit lifecycle, replacing scattered `s.basePath`/`s.originalBasePath` mutation and duplicated merge-or-teardown blocks [repo code].

**Location**: `src/resources/extensions/gsd/worktree-resolver.ts` [repo code]

**Design**: Mutates `AutoSession` fields directly so existing `s.basePath` reads continue to work everywhere without wiring changes [repo code].

**Key Invariant**: `createAutoWorktree()` and `enterAutoWorktree()` call `process.chdir()` internally — this class MUST NOT double-chdir [repo code].

#### Getters

| Property | Purpose |
|----------|---------|
| `workPath` | Current working path (may be worktree or project root) [repo code] |
| `projectRoot` | Original project root (always non-worktree path) [repo code] |
| `lockPath` | Path for auto.lock file (same as old `lockBase()`) [repo code] |

#### Key Methods

| Method | Purpose |
|--------|---------|
| `enterMilestone()` | Enter or create worktree for milestone [repo code] |
| `exitMilestone()` | Exit worktree: auto-commit, teardown, reset basePath [repo code] |
| `mergeAndExit()` | Merge milestone branch to main and exit [repo code] |
| `mergeAndEnterNext()` | Merge current milestone, then enter next one [repo code] |

#### enterMilestone

Only acts if `shouldUseWorktreeIsolation()` returns true [repo code]. On failure, notifies a warning and does NOT update `s.basePath` — staying in project root as fallback.

#### mergeAndExit with Three Isolation Modes

The method handles all three isolation modes [repo code]:

| Mode | Behavior |
|------|----------|
| `worktree` | Read roadmap, merge, teardown worktree, reset paths [repo code] |
| `branch` | Check if on milestone branch, merge if so (no chdir/teardown) [repo code] |
| `none` | No-op [repo code] |

**Error Recovery**: On merge failure, always restores `s.basePath` to `s.originalBasePath` and `process.chdir(s.originalBasePath)` [repo code]. Cleans up stale merge state (`SQUASH_MSG`, `MERGE_HEAD`, `MERGE_MSG`) [repo code].

---

### State Synchronization

State synchronization ensures planning artifacts and execution state flow correctly between worktree and project root.

**Location**: `src/resources/extensions/gsd/auto-worktree.ts` [repo code]

#### syncGsdStateToWorktree

Syncs `.gsd/` state from the main repo into the worktree [repo code]. When `.gsd/` is a symlink to an external state directory, both the main repo and worktree share the same directory — no sync needed [repo code].

**Forward Sync** (main → worktree) copies [repo code]:
- Root-level files: `DECISIONS.md`, `REQUIREMENTS.md`, `PROJECT.md`, `KNOWLEDGE.md`, `OVERRIDES.md`, `QUEUE.md`, `completed-units.json`
- Missing milestone directories (entire `milestones/<MID>/` copied)
- Missing slice directories within existing milestones

**Authoritative Rule**: Only adds missing content — never overwrites existing files in the worktree (the worktree's execution state is authoritative for in-progress work) [repo code].

#### syncWorktreeStateBack

Syncs milestone artifacts from worktree back to the main external state directory before merge [repo code]. Called before `mergeMilestoneToMain` to ensure completion artifacts survive teardown.

**Reverse Sync** (worktree → main) copies [repo code]:
- Root-level files: `DECISIONS.md`, `REQUIREMENTS.md`, `PROJECT.md`, `KNOWLEDGE.md`, `OVERRIDES.md`, `QUEUE.md`, `completed-units.json`
- ALL milestone directories (not just current milestone — complete-milestone may create next-milestone artifacts) [repo code]

**Overwrite Semantics**: The worktree is authoritative — these files overwrite main's copies [repo code]. Critical because `.gsd/` files are often untracked (gitignored), so squash merge carries nothing.

**Task Summaries**: Recursively syncs task summary files (`T01-SUMMARY.md`, etc.) from `tasks/` subdirectories [repo code].

---

### Plan Reconciliation for Crash Recovery

When auto-mode stops via crash (not graceful stop), the milestone branch HEAD may be behind the filesystem state at the project root [repo code]. This is because `syncStateToProjectRoot()` runs after every task completion but the final git commit may not have happened.

**Problem**: On restart, the worktree is re-attached to the branch HEAD (which has `[ ]` for the crashed task), causing `verifyExpectedArtifact()` to fail and triggering an infinite dispatch/skip loop [repo code].

#### reconcilePlanCheckboxes

Forward-merges plan checkbox state from the project root into a freshly re-attached worktree [repo code].

**Algorithm** [repo code]:
1. Walk all markdown files in the milestone directory at the project root
2. Extract checked task IDs from source: `- [x] **T<id>:` or `- [x] **S<id>:`
3. Forward-apply: replace `[ ]` → `[x]` for checked IDs in destination
4. Never downgrade `[x]` → `[ ]` (forward-only)

**Safety**: This is safe because `syncStateToProjectRoot()` is the authoritative source of post-task state — it writes the same `[x]` the LLM produced, then the auto-commit follows [repo code]. If the commit never happened, the filesystem copy is still valid.

---

### Worktree Detection and Path Resolution

Utility functions for detecting worktree context and resolving paths correctly.

**Location**: `src/resources/extensions/gsd/worktree.ts` [repo code]

#### detectWorktreeName

Detects the active worktree name from the current working directory [repo code]. Returns `null` if not inside a GSD worktree.

**Path Layouts Supported** [repo code]:
- Direct layout: `/.gsd/worktrees/<name>/`
- Symlink-resolved layout: `/.gsd/projects/<hash>/worktrees/<name>/`

When `.gsd` is a symlink to `~/.gsd/projects/<hash>`, resolved paths contain the intermediate `projects/<hash>/` segment [repo code].

#### resolveProjectRoot

Resolves the project root from a path that may be inside a worktree [repo code].

**Resolution Layers** [repo code]:
1. **GSD_PROJECT_ROOT env var**: If the coordinator passed the real project root, use it directly
2. **String-slice heuristic**: Find `/.gsd/` marker and return portion before it
3. **Git file recovery**: Read worktree's `.git` file to find `gitdir:` pointer, then resolve to real project root

**Home Directory Guard**: When the candidate resolves to `~` (because `.gsd` is a symlink into `~/.gsd/projects/<hash>`), falls back to git file recovery to avoid catastrophically wrong resolution [repo code].

---

## Execution Isolation Modes

Execution isolation modes determine how GSD isolates autonomous work from the project root. The `git.isolation` preference controls this behavior with three options: `worktree`, `branch`, and `none` [repo code].

### Mode Definitions

| Mode | Behavior | Use Case |
|------|----------|----------|
| `worktree` | Creates `.gsd/worktrees/<MID>/` with dedicated branch (default) | Standard isolated execution |
| `branch` | Works in project root, switches to milestone branch | Submodule-heavy repos where worktrees are problematic |
| `none` | Commits directly to current branch | No isolation, direct integration |

[repo code]

### Resolution via getIsolationMode

The `getIsolationMode()` function in `preferences.ts` resolves the effective isolation mode [repo code]:

```typescript
export function getIsolationMode(): "none" | "worktree" | "branch" {
  const prefs = loadEffectiveGSDPreferences()?.preferences?.git;
  if (prefs?.isolation === "none") return "none";
  if (prefs?.isolation === "branch") return "branch";
  return "worktree"; // default
}
```

### WorktreeResolver Mode Handling

The `WorktreeResolver.mergeAndExit()` method handles all three modes differently [repo code]:

**Worktree Mode** [repo code]:
1. Read roadmap for slice listing
2. Squash merge milestone branch to integration branch
3. Teardown worktree and delete branch
4. Reset `basePath` to project root

**Branch Mode** [repo code]:
1. Check if currently on milestone branch
2. If so, merge to integration branch (no worktree teardown)
3. No `process.chdir()` required

**None Mode** [repo code]:
- No-op: work was already on current branch

### ShouldUseWorktreeIsolation Helper

The `shouldUseWorktreeIsolation()` function gates worktree creation [repo code]:

```typescript
export function shouldUseWorktreeIsolation(): boolean {
  return getIsolationMode() === "worktree";
}
```

This is called before `WorktreeResolver.enterMilestone()` and `createAutoWorktree()` to skip worktree operations in branch/none modes [repo code].

---

## Subagent Isolation

Subagent isolation provides filesystem isolation for concurrent task execution, preventing subagents from stomping on each other's files. The isolation module provides two backends: git worktree and FUSE overlay [repo code].

**Location**: `src/resources/extensions/subagent/isolation.ts` [repo code]

### Isolation Modes

| Mode | Implementation | Platform | Performance |
|------|----------------|----------|-------------|
| `worktree` | Git worktree with detached HEAD | All | Good (copies git metadata) |
| `fuse-overlay` | FUSE overlay filesystem | Linux only | Excellent (copy-on-write) |
| `none` | No isolation | All | N/A (no protection) |

[repo code]

### IsolationEnvironment Interface

All isolation backends implement a common interface [repo code]:

```typescript
export interface IsolationEnvironment {
  workDir: string;              // Isolated working directory
  cleanup: () => Promise<void>; // Teardown isolation
  captureDelta: () => Promise<DeltaPatch[]>; // Capture changes
}
```

### Worktree Backend: createWorktreeIsolation

Creates a git worktree at `~/.gsd/wt/<encoded-cwd>/<taskId>/` [repo code].

**Creation Sequence** [repo code]:
1. Generate isolation directory path
2. Remove stale worktree if present (crash recovery)
3. `git worktree add --detach <dir> HEAD` — detached HEAD, no branch
4. Capture baseline from parent repo (staged/unstaged diff, untracked files)
5. Apply baseline to worktree
6. Commit baseline state for clean diff base

**Baseline Capture**: The `captureBaseline()` function captures [repo code]:
- Staged diff via `git diff --cached --binary`
- Unstaged diff via `git diff --binary`
- Untracked files via `git ls-files --others --exclude-standard -z` (capped at 10MB each)

**Baseline Application**: The `applyBaseline()` function [repo code]:
1. Apply staged diff (non-fatal if conflicts)
2. Apply unstaged diff on top
3. Copy untracked files to worktree
4. Commit all as "gsd: baseline snapshot"

### FUSE Overlay Backend: createFuseOverlayIsolation

Creates a FUSE overlay filesystem on Linux for zero-copy isolation [repo code].

**Directory Structure** [repo code]:
```
~/.gsd/wt/<encoded-cwd>/<taskId>/
├── upper/   # Subagent writes go here
├── work/    # Overlay work directory
└── merged/  # Merged view (what subagent sees)
```

**Mount Command** [repo code]:
```bash
fuse-overlayfs -o lowerdir=<repo>,upperdir=<upper>,workdir=<work> <merged>
```

**Delta Capture**: Unlike worktree backend, FUSE captures only files in `upper/` directory [repo code]. Parent dirty files are excluded by tracking them at creation time.

**Fallback**: If `fuse-overlayfs` binary is not found, automatically falls back to worktree backend [repo code].

### Delta Capture and Merge

**captureDeltaPatch**: Captures changes made in isolation [repo code]:
1. `git add -A` to stage all changes
2. `git diff --cached --binary HEAD` for full binary-capable diff
3. Returns array of `DeltaPatch` objects

**mergeDeltaPatches**: Applies captured patches back to main repo [repo code]:
1. Combine all patches into single file
2. Dry run with `git apply --check --binary`
3. If dry run passes, apply for real
4. Returns `MergeResult` with applied/failed patch lists

### Exit Handler Registration

Both backends register a process exit handler for cleanup on crash [repo code]:

```typescript
process.on("exit", () => {
  for (const dir of activeIsolations) {
    try {
      execFileSync("git", ["worktree", "remove", "--force", dir]);
    } catch { /* fallback to rm */ }
    fs.rmSync(dir, { recursive: true, force: true });
  }
});
```

---

## Session Locking

Session locking prevents multiple GSD processes from running auto-mode concurrently on the same project. It uses `proper-lockfile` for OS-level file locking (flock/lockfile), eliminating TOCTOU race conditions [repo code].

**Location**: `src/resources/extensions/gsd/session-lock.ts` [repo code]

### Lock File Location

The lock file lives at `.gsd/auto.lock` [repo code]:

```typescript
const LOCK_FILE = "auto.lock";

function lockPath(basePath: string): string {
  return join(gsdRoot(basePath), LOCK_FILE);
}
```

### SessionLockData Interface

The lock file contains JSON metadata for diagnostics [repo code]:

```typescript
export interface SessionLockData {
  pid: number;            // Process ID
  startedAt: string;      // ISO timestamp
  unitType: string;       // Current unit type
  unitId: string;         // Current unit ID
  unitStartedAt: string;  // Unit start timestamp
  completedUnits: number; // Units completed so far
  sessionFile?: string;   // Session file path
}
```

### acquireSessionLock

Attempts to acquire an exclusive session lock [repo code].

**Algorithm** [repo code]:
1. Check for re-entrant acquire on same path (release old lock first)
2. Create `.gsd/` directory if missing
3. Clean up stray lock file variants from cloud sync conflicts (#1315)
4. Call `proper-lockfile.lockSync()` with:
   - `stale: 1_800_000` (30 minutes — safe for laptop sleep)
   - `update: 10_000` (heartbeat every 10s)
   - `onCompromised` handler for mtime drift
5. Write lock metadata via atomic write
6. Register exit handler for cleanup

**Stale Lock Recovery**: If lock file exists but process is dead, cleans up and retries [repo code]:

```typescript
if (!existingData || (existingPid && !isPidAlive(existingPid))) {
  // Clean up stale lock directory and file
  rmSync(lockDir, { recursive: true, force: true });
  unlinkSync(lp);
  // Retry acquisition
}
```

**Fallback Mode**: When `proper-lockfile` is unavailable, uses PID-based liveness checking [repo code]:

```typescript
function acquireFallbackLock(basePath, lp, lockData): SessionLockResult {
  const existing = readExistingLockData(lp);
  if (existing && existing.pid !== process.pid && isPidAlive(existing.pid)) {
    return { acquired: false, reason: "...", existingPid: existing.pid };
  }
  // Stale or no lock — write and succeed
  atomicWriteSync(lp, JSON.stringify(lockData));
  return { acquired: true };
}
```

### onCompromised Handler

The `proper-lockfile` library fires `onCompromised` when it detects mtime drift (system sleep, event loop stall) [repo code].

**False-Positive Suppression** (#1362) [repo code]:
1. If elapsed time < stale window (30 min), suppress as event loop stall
2. Check if lock file still contains our PID — if so, treat as false positive
3. Only set `_lockCompromised = true` for real takeovers

```typescript
onCompromised: () => {
  const elapsed = Date.now() - _lockAcquiredAt;
  if (elapsed < 1_800_000) {
    // Event loop stall within stale window — suppress
    process.stderr.write("[gsd] Lock heartbeat mismatch — event loop stall, continuing.\n");
    return;
  }
  // Check PID ownership before declaring compromise
  const existing = readExistingLockData(lp);
  if (existing && existing.pid === process.pid) {
    // Our PID still owns the lock file — false positive
    return;
  }
  _lockCompromised = true;
}
```

### validateSessionLock and getSessionLockStatus

Called periodically during dispatch to detect takeover [repo code].

**Status Return** [repo code]:
```typescript
export interface SessionLockStatus {
  valid: boolean;
  failureReason?: "compromised" | "missing-metadata" | "pid-mismatch";
  existingPid?: number;
  expectedPid?: number;
  recovered?: boolean;  // True if re-acquired after false positive
}
```

**Recovery Gate** (#1512) [repo code]: When `onCompromised` fired but lock file still contains our PID, attempts re-acquisition instead of giving up:

```typescript
if (_lockCompromised) {
  const existing = readExistingLockData(lp);
  if (existing && existing.pid === process.pid) {
    try {
      const result = acquireSessionLock(basePath);
      if (result.acquired) {
        return { valid: true, recovered: true };
      }
    } catch { /* fall through */ }
  }
  return { valid: false, failureReason: "compromised", ... };
}
```

### releaseSessionLock

Called on clean stop/pause to release the lock [repo code].

**Cleanup Sequence** [repo code]:
1. Call `_releaseFunction()` from proper-lockfile
2. Remove `.gsd/auto.lock` file
3. Remove `.gsd.lock/` directory (proper-lockfile's lock dir)
4. Clean ALL registered lock paths (handles worktree/project root accumulation)
5. Clear `_lockDirRegistry`
6. Clean stray lock file variants

### Stray Lock Cleanup

Cloud sync (iCloud/Dropbox/OneDrive) creates numbered lock file variants via copy-on-conflict [repo code]. The `cleanupStrayLockFiles()` function removes these [repo code]:

- `auto 2.lock`, `auto (2).lock` inside `.gsd/`
- `.gsd 2.lock/`, `.gsd (2).lock/` directories adjacent to `.gsd/`

---

## Recovery Mechanisms

GSD implements multiple recovery mechanisms to handle crashes, lock compromises, and incomplete state transitions.

### Lock Compromise Recovery

When `proper-lockfile` detects mtime drift, GSD attempts recovery before giving up [repo code].

**Detection Signals** [repo code]:
- Lock file mtime older than expected (heartbeat not updating)
- Process was asleep or event loop stalled

**Recovery Steps** [repo code]:
1. Check if lock file PID still matches our PID
2. If match, attempt re-acquisition
3. On success, set `recovered: true` and continue
4. On failure, return `valid: false` to trigger graceful stop

### Plan Reconciliation for Crash Recovery

When auto-mode crashes after completing tasks but before committing, the worktree branch HEAD may be behind filesystem state [repo code].

**Problem**: On restart, the worktree re-attaches to branch HEAD with `[ ]` checkboxes, causing verification failures and infinite dispatch/skip loops [repo code].

**Solution**: `reconcilePlanCheckboxes()` forward-merges plan state from project root [repo code]:

```typescript
export function reconcilePlanCheckboxes(
  basePath: string,
  worktreePath: string,
  milestoneId: string,
): void
```

**Algorithm** [repo code]:
1. Walk all markdown files in milestone directory at project root
2. Extract checked task IDs: `- [x] **T<id>:` or `- [x] **S<id>:`
3. Forward-apply: replace `[ ]` → `[x]` for checked IDs in worktree
4. Never downgrade `[x]` → `[ ]` (forward-only)

**Safety Guarantee**: `syncStateToProjectRoot()` runs after every task completion — the filesystem copy is authoritative even if git commit failed [repo code].

### State Synchronization

State synchronization ensures planning artifacts flow correctly between worktree and project root [repo code].

**Forward Sync** (main → worktree) via `syncGsdStateToWorktree()` [repo code]:
- Root-level files: `DECISIONS.md`, `REQUIREMENTS.md`, etc.
- Missing milestone directories (entire `milestones/<MID>/` copied)
- **Authoritative rule**: Only adds missing content — never overwrites existing worktree files

**Reverse Sync** (worktree → main) via `syncWorktreeStateBack()` [repo code]:
- Called before `mergeMilestoneToMain` to preserve completion artifacts
- **Overwrite semantics**: Worktree is authoritative — these files overwrite main's copies
- Critical because `.gsd/` files are often gitignored, not carried by squash merge

### Crash Recovery on Worktree Creation

Stale worktree directories from prior crashes are handled during creation [repo code]:

```typescript
// In createWorktree()
if (existsSync(worktreePath) && !isGitWorktree(worktreePath)) {
  // Stale directory without valid .git file — crash leftover
  rmSync(worktreePath, { recursive: true, force: true });
}
```

### Worktree Health Check

During auto-mode execution, `runWorktreeHealthCheck()` verifies worktree integrity [repo code]:

1. Verify `.gsd/` directory exists
2. Verify `.git` file contains `gitdir:` pointer
3. If invalid, log warning and continue in project root (fallback)

### Parallel Orchestrator Crash Recovery

The parallel orchestrator persists state to `.gsd/orchestrator.json` for crash recovery [repo code]:

```typescript
export interface PersistedState {
  active: boolean;
  workers: Array<{ milestoneId, pid, worktreePath, ... }>;
  totalCost: number;
  startedAt: number;
  configSnapshot: { max_workers, budget_ceiling };
}
```

**Restore Flow** (`restoreState()`) [repo code]:
1. Read `.gsd/orchestrator.json`
2. Filter to workers with living PIDs
3. If no survivors, clean up and return null
4. Otherwise, rebuild orchestrator state from persisted data

**Fallback**: If orchestrator.json is missing/corrupt, rebuild from session status files under `.gsd/parallel/` [repo code]:

```typescript
// Fallback: rebuild from live session status files
const statuses = readAllSessionStatuses(basePath);
for (const status of statuses) {
  state.workers.set(status.milestoneId, { ...status, process: null });
}
```

---

## Dispatch Guard

The dispatch guard prevents out-of-order slice dispatch, ensuring slices execute in dependency order [repo code].

**Location**: `src/resources/extensions/gsd/dispatch-guard.ts` [repo code]

### getPriorSliceCompletionBlocker

Returns a blocking message if prior slices or dependencies are incomplete [repo code]:

```typescript
export function getPriorSliceCompletionBlocker(
  base: string,
  _mainBranch: string,  // Unused — reads from disk, not git branch
  unitType: string,
  unitId: string,
): string | null
```

### Guarded Dispatch Types

Only these dispatch types are guarded [repo code]:

```typescript
const SLICE_DISPATCH_TYPES = new Set([
  "research-slice",
  "plan-slice",
  "replan-slice",
  "execute-task",
  "complete-slice",
]);
```

### Disk-State Reading

The guard reads roadmap files from disk (working tree), not from git branches [repo code]:

```typescript
function readRoadmapFromDisk(base: string, milestoneId: string): string | null {
  const absPath = resolveMilestoneFile(base, milestoneId, "ROADMAP");
  return readFileSync(absPath, "utf-8").trim();
}
```

**Why disk instead of git**: Prior implementation used `git show <branch>:<path>` which caused false-positive blockers when work was committed on milestone branch but integration branch hadn't been updated (#530) [repo code].

### Dependency-Aware Ordering

Slices can declare `depends_on` in ROADMAP.md for non-sequential dependencies [repo code].

**Logic** [repo code]:
1. Parse slice's `depends` array from roadmap
2. If `depends.length > 0`: only require those specific slices complete
3. If `depends.length === 0`: fall back to positional ordering (backward compatibility)

**Example** (#530) [repo code]:
```
S05 depends_on: [S06]  # S05 depends on later slice
```

Without dependency-aware ordering, positional check would block S05 because S04 isn't done — but S04 might intentionally come after S05.

### Queue Order Respect

Milestone order is determined by `findMilestoneIds()` which respects `queue-order.json` [repo code]:

```typescript
const allIds = findMilestoneIds(base);
const targetIdx = allIds.indexOf(targetMid);
const milestoneIds = allIds.slice(0, targetIdx + 1);
```

Only milestones BEFORE the target in queue order are checked for completion [repo code].

---

## Parallel Orchestrator

The parallel orchestrator manages concurrent milestone execution with worker processes in isolated worktrees [repo code].

**Location**: `src/resources/extensions/gsd/parallel-orchestrator.ts` [repo code]

### Orchestrator State

**In-Memory State** [repo code]:

```typescript
export interface OrchestratorState {
  active: boolean;
  workers: Map<string, WorkerInfo>;
  config: ParallelConfig;
  totalCost: number;
  startedAt: number;
}
```

**WorkerInfo** [repo code]:

```typescript
export interface WorkerInfo {
  milestoneId: string;
  title: string;
  pid: number;
  process: ChildProcess | null;  // null after exit or coordinator restart
  worktreePath: string;
  startedAt: number;
  state: "running" | "paused" | "stopped" | "error";
  completedUnits: number;
  cost: number;
}
```

### Worker Spawning

Workers are separate Node.js processes spawned via `child_process.spawn()` [repo code]:

```typescript
child = spawn(process.execPath, [binPath, "--mode", "json", "--print", "/gsd auto"], {
  cwd: worker.worktreePath,
  env: {
    ...process.env,
    GSD_MILESTONE_LOCK: milestoneId,    // Isolate state derivation
    GSD_PROJECT_ROOT: basePath,          // Real project root
    GSD_PARALLEL_WORKER: "1",            // Prevent nested parallelism
  },
  stdio: ["ignore", "pipe", "pipe"],
  detached: false,
});
```

**Environment Variables** [repo code]:
- `GSD_MILESTONE_LOCK`: Set to milestone ID so worker derives state from that milestone only
- `GSD_PROJECT_ROOT`: Pass real project root so workers don't mis-derive from symlinks
- `GSD_PARALLEL_WORKER`: Prevent workers from spawning their own parallel sessions

### Session Status Files

Workers write heartbeat files under `.gsd/parallel/<milestoneId>.json` [repo code]:

```typescript
writeSessionStatus(basePath, {
  milestoneId,
  pid: worker.pid,
  state: "running",
  completedUnits: 0,
  cost: 0,
  lastHeartbeat: Date.now(),
  startedAt: now,
  worktreePath: wtPath,
});
```

These files enable [repo code]:
- Dashboard refresh without in-memory state
- Crash recovery by scanning surviving workers
- Status polling across coordinator restarts

### NDJSON Monitoring

Workers run with `--mode json` and emit NDJSON events on stdout [repo code]:

```typescript
// Parse message_end events for cost tracking
if (type === "message_end" && event.message) {
  const cost = event.message.usage?.cost?.total;
  if (typeof cost === "number") {
    worker.cost += cost;
    state.totalCost = sum(all worker costs);
  }
}
```

### Budget Tracking

**Aggregate Cost**: Sum of all worker costs tracked in `state.totalCost` [repo code].

**Budget Ceiling Check** [repo code]:

```typescript
export function isBudgetExceeded(): boolean {
  if (!state) return false;
  if (state.config.budget_ceiling == null) return false;
  return state.totalCost >= state.config.budget_ceiling;
}
```

Checked before each worker spawn to prevent over-budget execution [repo code].

### Worker Lifecycle

**Start** (`startParallel()`) [repo code]:
1. Prevent nested parallelism (check `GSD_PARALLEL_WORKER` env)
2. Try restore from previous crash
3. Create worktrees for each milestone
4. Spawn worker processes
5. Write session status files
6. Persist orchestrator state

**Stop** (`stopParallel()`) [repo code]:
1. Send stop signal via file-based IPC
2. Send SIGTERM to worker processes
3. Wait up to 750ms for graceful exit
4. Send SIGKILL if still running
5. Update worker state to "stopped"
6. Remove session status files
7. Remove persisted state file

**Pause/Resume** [repo code]:
- Pause: Send "pause" signal via `sendSignal()`, set state to "paused"
- Resume: Send "resume" signal, set state to "running"

### Crash Recovery

**PID Liveness Check** [repo code]:

```typescript
function isPidAlive(pid: number): boolean {
  try {
    process.kill(pid, 0);
    return true;
  } catch {
    return false;
  }
}
```

**Restore Flow** (`restoreState()`) [repo code]:
1. Read `.gsd/orchestrator.json`
2. Filter workers to only those with living PIDs
3. Remove dead workers from state
4. Rebuild in-memory state from persisted data

**Fallback Recovery** (`restoreRuntimeState()`) [repo code]:
1. If orchestrator.json missing, scan `.gsd/parallel/` for session files
2. Clean up stale sessions (dead PIDs)
3. Rebuild coordinator state from surviving session files

---

## Why Each Layer Exists

The git and isolation architecture provides four fundamental guarantees that justify each layer:

### 1. Transactional Milestone Execution

**Problem**: Autonomous agents make mistakes. A failed milestone shouldn't corrupt the main codebase.

**Solution**: Worktree isolation creates a sandbox. Work happens in `.gsd/worktrees/<MID>/` with a dedicated branch. Only squash-merges bring completed work back to the integration branch [repo code].

**Layers involved**: Git Operations Layer, Worktree Isolation Model

### 2. Concurrent Execution Safety

**Problem**: Multiple agents working on the same files cause conflicts and data loss.

**Solution**: Three isolation mechanisms prevent interference:
- **Worktree isolation**: Each milestone gets its own working directory
- **Session locking**: OS-level locks prevent multiple GSD processes on same project
- **Subagent isolation**: Worktree or FUSE overlay for parallel subagent tasks

**Layers involved**: Execution Isolation Modes, Session Locking, Subagent Isolation

### 3. Crash Recovery Without Data Loss

**Problem**: Agents crash mid-task. How do we resume without losing work or getting stuck?

**Solution**: Multiple recovery mechanisms:
- **Lock compromise recovery**: Re-acquire lock after false positive from sleep/stall
- **Plan reconciliation**: Forward-merge checkbox state from project root to worktree
- **State synchronization**: Copy planning artifacts between worktree and project root
- **Parallel orchestrator recovery**: Restore from persisted state or session files

**Layers involved**: Recovery Mechanisms, Parallel Orchestrator, Worktree Isolation Model

### 4. Dependency-Aware Dispatch

**Problem**: Slices may have non-sequential dependencies. Executing out of order causes failures.

**Solution**: Dispatch guard reads `depends_on` declarations and only blocks on actual dependencies, not positional predecessors [repo code].

**Layers involved**: Dispatch Guard

### Layer Dependencies

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ LAYER DEPENDENCY GRAPH                                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Git Operations Layer (foundation)                                          │
│  └─ Native git bridge, smart staging, integration branch resolution        │
│                                                                             │
│           ↓ builds on                                                       │
│                                                                             │
│  Worktree Isolation Model                                                   │
│  └─ Worktree lifecycle, branch naming, state sync, plan reconciliation     │
│                                                                             │
│           ↓ configures                                                      │
│                                                                             │
│  Execution Isolation Modes                                                  │
│  └─ Worktree/branch/none preference, WorktreeResolver integration          │
│                                                                             │
│           ↓ extends for subagents                                           │
│                                                                             │
│  Subagent Isolation                                                         │
│  └─ Worktree/FUSE overlay backends, delta capture, patch merge             │
│                                                                             │
│  Session Locking (orthogonal)                                               │
│  └─ OS-level exclusive lock, proper-lockfile, crash recovery               │
│                                                                             │
│  Dispatch Guard (orthogonal)                                                │
│  └─ Dependency-aware ordering, disk-state reading, queue respect           │
│                                                                             │
│  Parallel Orchestrator (consumer)                                           │
│  └─ Worker spawning, session status, budget tracking, crash recovery       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Cross-References

### Related Documentation

- **[GSD2_RUNTIME_ARCHITECTURE.md](./GSD2_RUNTIME_ARCHITECTURE.md)** — Session lifecycle, runtime assembly, and event flow
- **[GSD2_ORCHESTRATION_LAYER.md](./GSD2_ORCHESTRATION_LAYER.md)** — Auto-mode state machine, dispatch resolution, verification gates
- **[GSD2_SYSTEM_OVERVIEW.md](./GSD2_SYSTEM_OVERVIEW.md)** — High-level architecture and component relationships
- **[EVIDENCE_METHOD.md](./EVIDENCE_METHOD.md)** — Subject-vs-runner guardrail and evidence labeling conventions

### Key Integration Points

| This Document | Runtime Architecture | Orchestration Layer |
|---------------|---------------------|---------------------|
| Session locking | Session lifecycle | Auto-mode loop guards |
| Worktree isolation | AgentSession cwd management | State derivation |
| Plan reconciliation | Crash recovery | Dispatch guards |
| Parallel orchestrator | NDJSON monitoring | Budget tracking |
| Execution isolation modes | — | Phase transitions |

---

## Summary

The Git Operations Layer provides three foundational capabilities:

1. **GitServiceImpl**: A workflow-aware git abstraction that understands milestone context, performs smart staging with runtime exclusions, and generates meaningful commit messages from task context.

2. **Native Git Bridge**: High-performance git operations via libgit2 with automatic CLI fallback, supporting both read queries and write operations with consistent semantics.

3. **Integration Branch Management**: Resolution and persistence of the target branch for slice merges, handling the common case where users work on feature branches rather than the repo default.

These components form the foundation that worktree isolation and execution modes build upon to provide safe, isolated execution for autonomous coding agents.
