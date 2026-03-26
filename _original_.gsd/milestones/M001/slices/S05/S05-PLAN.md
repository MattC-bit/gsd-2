# S05: Git, Worktree, and Isolation Model

**Goal:** Produce `GSD2_GIT_AND_ISOLATION_MODEL.md` explaining how GSD owns git/worktree isolation, what execution boundaries it enforces, and how isolation relates to recovery and milestone control.

**Demo:** A reader can trace the complete isolation flow from git primitives → worktree management → execution isolation modes → recovery mechanisms, and explain why each layer exists.

## Must-Haves

- Document the git operations layer (`git-service.ts`, `native-git-bridge.ts`) including native libgit2 bridge and fallback paths
- Document worktree management (`worktree.ts`, `worktree-manager.ts`, `auto-worktree.ts`) including creation, entry, teardown, and merge
- Document three isolation modes (worktree, branch, none) and when each applies
- Document execution isolation for subagents (`subagent/isolation.ts`) with worktree and FUSE overlay backends
- Document session locking (`session-lock.ts`) and crash recovery mechanisms
- Document WorktreeResolver's role in orchestrating worktree lifecycle
- Document dispatch guard's dependency-aware ordering and disk-state vs git-branch reading
- Document parallel orchestrator's worker isolation
- Include evidence tier labels (`[repo code]`, `[repo doc]`, `[external]`, `[inference]`) on all claims
- Include subject-vs-runner guardrail specific to git/isolation context

## Proof Level

- This slice proves: **contract** — the document accurately describes the git/worktree/isolation architecture with evidence labels
- Real runtime required: **no** — documentation slice
- Human/UAT required: **no** — mechanical verification

## Verification

- `test -f analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md` returns 0
- `grep -c "^## " analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md` returns >= 8 (8+ major sections)
- `grep -q "Git Operations Layer" analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md` returns 0
- `grep -q "Worktree Isolation Model" analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md` returns 0
- `grep -q "Execution Isolation Modes" analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md` returns 0
- `grep -q "Recovery Mechanisms" analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md` returns 0
- `grep -c "\[repo code\]" analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md` returns >= 30 (30+ evidence labels)
- `grep -q "subject-vs-runner" analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md` returns 0

## Tasks

- [x] **T01: Document Git Operations Layer** `est:45m`
  - Why: Establishes the foundational git primitives that worktree isolation builds upon
  - Files: `src/resources/extensions/gsd/git-service.ts`, `src/resources/extensions/gsd/native-git-bridge.ts`, `analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md`
  - Do: Document GitServiceImpl, native git bridge with libgit2, fallback paths, integration branch management, commit message generation, and pre-merge checks. Include evidence labels on all claims.
  - Verify: `grep -q "Git Operations Layer" analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md` && `grep -q "native-git-bridge" analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md`
  - Done when: Git operations section exists with native bridge, fallback, and integration branch subsections, all claims have evidence labels

- [x] **T02: Document Worktree Isolation Model** `est:1h`
  - Why: Documents the core isolation mechanism GSD uses for milestone execution
  - Files: `src/resources/extensions/gsd/worktree.ts`, `src/resources/extensions/gsd/worktree-manager.ts`, `src/resources/extensions/gsd/auto-worktree.ts`, `src/resources/extensions/gsd/worktree-resolver.ts`, `analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md`
  - Do: Document worktree creation/deletion lifecycle, branch naming conventions (`worktree/<name>`, `milestone/<MID>`), auto-worktree for milestones, WorktreeResolver's merge/exit lifecycle, state synchronization between worktree and project root, and plan reconciliation for crash recovery. Include evidence labels.
  - Verify: `grep -q "Worktree Isolation Model" analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md` && `grep -q "WorktreeResolver" analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md`
  - Done when: Worktree section exists with lifecycle, naming, auto-worktree, WorktreeResolver, and sync subsections, all claims have evidence labels

- [x] **T03: Document Execution Isolation, Recovery, and Guards** `est:1h`
  - Why: Documents the higher-level isolation mechanisms and how they integrate with recovery
  - Files: `src/resources/extensions/gsd/session-lock.ts`, `src/resources/extensions/subagent/isolation.ts`, `src/resources/extensions/gsd/dispatch-guard.ts`, `src/resources/extensions/gsd/parallel-orchestrator.ts`, `analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md`
  - Do: Document three isolation modes (worktree, branch, none), subagent isolation backends (worktree, FUSE overlay), session locking with proper-lockfile, dispatch guard's dependency-aware ordering, and parallel orchestrator's worker isolation. Add cross-references to orchestration layer and runtime docs. Include subject-vs-runner guardrail specific to git/isolation. Include evidence labels.
  - Verify: `grep -q "Execution Isolation Modes" analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md` && `grep -q "Recovery Mechanisms" analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md` && `grep -q "subject-vs-runner" analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md`
  - Done when: All three major sections exist (execution isolation, recovery, guards), cross-references to S02/S03 docs present, subject-vs-runner guardrail included, all claims have evidence labels

## Files Likely Touched

- `analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md` — new document
