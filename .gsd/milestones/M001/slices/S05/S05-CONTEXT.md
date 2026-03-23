---
id: S05
milestone: M001
status: ready
---

# S05: Git, Worktree, and Isolation Model — Context

<!-- Slice-scoped context. Milestone-only sections (acceptance criteria, completion class,
     milestone sequence) do not belong here — those live in the milestone context. -->

## Goal

Document how GSD owns git/worktree isolation, the three isolation modes, all git operations via native bridge, the complete worktree lifecycle, merge flow with conflict handling, execution boundaries at milestone/slice/task levels, and how isolation relates to recovery and parallel orchestration.

## Why this Slice

S03 explains orchestration control and mentions worktrees, but doesn't explain the isolation model itself. S05 fills this gap by documenting:

1. **Isolation modes** — worktree, branch, none — with all flags and path resolution
2. **Git service** — all operations via native bridge with operation signatures
3. **Worktree lifecycle** — create → work → merge → teardown with all operations
4. **Merge flow** — squash vs merge strategy, conflict handling, branch cleanup
5. **Execution boundaries** — all enforcement at milestone, slice, and task levels
6. **Recovery relationship** — how isolation affects crash recovery and self-healing
7. **Parallel orchestration** — how multiple worktrees enable parallel execution

S06 (comparative analysis) depends on understanding isolation as a key harness capability. Without this, comparisons would miss how GSD's isolation model differs from other systems.

**Ownership note:** All git/worktree code is GSD-authored. Pi provides no git capabilities.

## Scope

### In Scope

**Isolation model (all flags and path resolution):**
- Three isolation modes: `worktree`, `branch`, `none`
- All `GitPreferences` flags with purposes
- Worktree path resolution: `gsd/worktrees/<milestone>/`
- Branch naming patterns: `milestone/<MID>` vs `worktree/<name>`
- Integration branch detection and tracking
- Project root vs worktree root state management

**Git service (all native bridge operations with signatures):**
- Native bridge pattern: Rust N-API via libgit2 with git CLI fallback
- All native operation signatures:
  - `nativeGetCurrentBranch`, `nativeDetectMainBranch`
  - `nativeBranchExists`, `nativeBranchDelete`, `nativeBranchForceReset`
  - `nativeHasChanges`, `nativeHasStagedChanges`
  - `nativeAddAllWithExclusions`, `nativeAddPaths`, `nativeRmCached`
  - `nativeCommit`, `nativeMergeSquash`
  - `nativeConflictFiles`, `nativeCheckoutTheirs`
  - `nativeWorktreeAdd`, `nativeWorktreeList`, `nativeWorktreePrune`, `nativeWorktreeRemove`
  - `nativeDiffContent`, `nativeDiffNameStatus`, `nativeDiffNumstat`
  - `nativeLogOneline`, `nativeUpdateRef`, `nativeIsAncestor`
- Commit type inference rules
- Commit message building with task context

**Worktree lifecycle (all operations):**
- `createWorktree` — branch creation, path setup, state sync
- `removeWorktree` — worktree teardown, branch deletion, cleanup
- `enterWorktree` / `exitWorktree` — process chdir, state reconciliation
- `isInAutoWorktree` — detection and path resolution
- `getAutoWorktreePath` — worktree directory location
- `clearProjectRootStateFiles` — transient file cleanup
- `captureIntegrationBranch` — branch state preservation
- DB synchronization: `copyWorktreeDb`, `reconcileWorktreeDb`
- Crash recovery: `recoverWorktreeCrash`

**Merge flow (with conflict handling):**
- Squash merge strategy (default) vs merge strategy
- `MergeConflictError` handling with `conflictedFiles`
- Auto-resolution for `.gsd/` conflicts (`nativeCheckoutTheirs`)
- Rollback behavior on merge failure
- Branch cleanup after successful merge
- Milestone branch naming and lifecycle

**Execution boundaries (all enforcement):**
- Milestone isolation: one worktree per milestone
- Slice isolation: no cross-slice state mutation during execution
- Task isolation: fresh session per task, no shared mutable state
- Boundary violation detection and recovery
- Doctor checks for stale/orphaned worktrees

**Recovery relationship:**
- Crash detection: interrupted worktree operations
- Stale worktree identification
- Self-healing: state reconciliation after crash
- `.gsd/` conflict auto-resolution during merge recovery

**Parallel orchestration (full detail):**
- `parallel-orchestrator.ts` — how multiple milestones run concurrently
- Multiple worktrees for parallel execution
- State isolation between parallel milestones
- DB coordination across worktrees
- Parallel completion and merge sequencing

**Produces:**
- `GSD2_GIT_AND_ISOLATION_MODEL.md` — isolation modes, git operations, worktree lifecycle, merge flow, boundaries, recovery, parallel orchestration
- Normalized isolation concepts for S06 comparison

### Out of Scope

- **Runtime recovery internals**: Retry/compaction details (S02)
- **Orchestration dispatch**: How units are selected (S03)
- **Context engineering**: Prompt assembly (S04)
- **External comparisons**: Comparative analysis (S06)

## Constraints

- **Exhaustive detail**: All isolation flags, all native operation signatures, all worktree operations, all boundary enforcement, full parallel orchestration
- **GSD ownership**: Document that all git/worktree code is GSD-authored, Pi provides no git capabilities
- **Structural verification**: Machine-checkable artifact existence and section presence
- **Native bridge signatures**: Document each operation's signature, not the Rust implementation

## Integration Points

### Consumes

- `.gsd/milestones/M001/slices/S01/S01-CONTEXT.md` — evidence method, boundary inventory
- `.gsd/milestones/M001/slices/S02/S02-CONTEXT.md` — runtime recovery context
- `.gsd/milestones/M001/slices/S03/S03-CONTEXT.md` — orchestration context, dispatch flow
- `analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md` — orchestration layer (after S03)
- `analysis/gsd2-harness-decomposition/GSD2_HARNESS_MODEL.md` — harness model (after S03)

### Produces

- `analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md` — git, worktree, isolation, boundaries, recovery, parallel orchestration
- Normalized isolation concepts usable in S06 comparison

## Open Questions

- **Native bridge stability**: Should S05 document which operations are most commonly used vs edge cases? Current thinking: enumerate all operations but note which are primary vs secondary.

- **Parallel orchestration scope**: Should parallel orchestration be split into a separate slice given its complexity? Current thinking: keep in S05 since it directly uses worktrees; S05 is already large.

- **Merge conflict examples**: Should S05 include example conflict scenarios and their resolution? Current thinking: yes, since conflict handling is a key isolation feature.
