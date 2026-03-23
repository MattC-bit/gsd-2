---
estimated_steps: 12
estimated_files: 6
skills_used:
  - accessibility
  - best-practices
  - review
---

# T03: Document Execution Isolation, Recovery, and Guards

**Slice:** S05 — Git, Worktree, and Isolation Model
**Milestone:** M001

## Description

Document the higher-level isolation mechanisms and how they integrate with recovery. This includes three isolation modes (worktree, branch, none), subagent isolation backends (worktree, FUSE overlay), session locking with proper-lockfile, dispatch guard's dependency-aware ordering, and parallel orchestrator's worker isolation. Add cross-references to orchestration layer and runtime docs. Include subject-vs-runner guardrail specific to git/isolation context.

## Steps

1. Read `preferences.ts` and `git-service.ts` to understand `GitPreferences.isolation` mode (worktree, branch, none)
2. Read `subagent/isolation.ts` to understand worktree and FUSE overlay backends for task isolation
3. Read `session-lock.ts` to understand proper-lockfile integration, OS-level locking, and crash recovery
4. Read `dispatch-guard.ts` to understand dependency-aware ordering and disk-state vs git-branch reading
5. Read `parallel-orchestrator.ts` to understand worker isolation in parallel execution
6. Document Execution Isolation Modes: worktree, branch, none with when each applies
7. Document Subagent Isolation: `createWorktreeIsolation`, `createFuseOverlayIsolation`, delta patch capture and merge
8. Document Session Locking: `acquireSessionLock`, `validateSessionLock`, `releaseSessionLock` with proper-lockfile
9. Document Dispatch Guard: `getPriorSliceCompletionBlocker`, dependency-aware ordering, disk-state reading
10. Document Parallel Orchestrator: worker spawning, session status files, crash recovery
11. Add cross-references to `GSD2_RUNTIME_ARCHITECTURE.md` (session lifecycle) and `GSD2_ORCHESTRATION_LAYER.md` (dispatch, guards)
12. Add subject-vs-runner guardrail specific to git/isolation context (analyzed `.gsd/` worktree model vs runner's live worktrees)

## Must-Haves

- [ ] Execution Isolation Modes section with three modes
- [ ] Subagent Isolation section with worktree and FUSE backends
- [ ] Session Locking section with proper-lockfile integration
- [ ] Recovery Mechanisms section with lock compromise, plan reconciliation, crash recovery
- [ ] Dispatch Guard section with dependency-aware ordering
- [ ] Parallel Orchestrator section with worker isolation
- [ ] Cross-references to S02 (runtime) and S03 (orchestration) docs
- [ ] Subject-vs-runner guardrail for git/isolation context
- [ ] All claims have evidence labels

## Verification

- `grep -q "Execution Isolation Modes" analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md`
- `grep -q "Subagent Isolation" analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md`
- `grep -q "Session Locking" analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md`
- `grep -q "Recovery Mechanisms" analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md`
- `grep -q "Dispatch Guard" analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md`
- `grep -q "Parallel Orchestrator" analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md`
- `grep -q "subject-vs-runner" analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md`
- `grep -q "GSD2_RUNTIME_ARCHITECTURE" analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md`
- `grep -q "GSD2_ORCHESTRATION_LAYER" analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md`
- `grep -c "\[repo code\]" analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md` returns >= 45 (cumulative with T01/T02)
- `grep -c "^## " analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md` returns >= 8

## Inputs

- `src/resources/extensions/gsd/git-service.ts` — GitPreferences.isolation mode
- `src/resources/extensions/subagent/isolation.ts` — subagent isolation backends
- `src/resources/extensions/gsd/session-lock.ts` — session locking
- `src/resources/extensions/gsd/dispatch-guard.ts` — dispatch guard
- `src/resources/extensions/gsd/parallel-orchestrator.ts` — parallel execution
- `analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md` — T01/T02 output to append to
- `analysis/gsd2-harness-decomposition/GSD2_RUNTIME_ARCHITECTURE.md` — cross-reference target
- `analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md` — cross-reference target
- `analysis/gsd2-harness-decomposition/EVIDENCE_METHOD.md` — subject-vs-runner guardrail reference

## Expected Output

- `analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md` — complete document with all sections
