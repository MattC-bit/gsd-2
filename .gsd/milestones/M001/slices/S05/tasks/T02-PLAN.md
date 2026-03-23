---
estimated_steps: 10
estimated_files: 5
skills_used:
  - accessibility
  - best-practices
  - review
---

# T02: Document Worktree Isolation Model

**Slice:** S05 — Git, Worktree, and Isolation Model
**Milestone:** M001

## Description

Document the core worktree isolation mechanism GSD uses for milestone execution. This includes worktree creation/deletion lifecycle, branch naming conventions (`worktree/<name>`, `milestone/<MID>`), auto-worktree for milestones, WorktreeResolver's merge/exit lifecycle, state synchronization between worktree and project root, and plan reconciliation for crash recovery.

## Steps

1. Read `worktree.ts` to understand pure utility functions for worktree name detection, branch naming, and project root resolution
2. Read `worktree-manager.ts` to understand worktree CRUD operations, diff utilities, and merge functions
3. Read `auto-worktree.ts` to understand milestone-specific worktree lifecycle (create, enter, teardown, merge to main)
4. Read `worktree-resolver.ts` to understand the WorktreeResolver class that encapsulates worktree path state
5. Document worktree lifecycle: `createWorktree`, `listWorktrees`, `removeWorktree`
6. Document branch naming: `worktree/<name>` for manual, `milestone/<MID>` for auto-mode
7. Document auto-worktree: `createAutoWorktree`, `enterAutoWorktree`, `teardownAutoWorktree`, `mergeMilestoneToMain`
8. Document WorktreeResolver: `enterMilestone`, `exitMilestone`, `mergeAndExit` with three isolation modes
9. Document state sync: `syncGsdStateToWorktree`, `syncWorktreeStateBack`, `reconcilePlanCheckboxes`
10. Write section with evidence labels on all factual claims

## Must-Haves

- [ ] Worktree Isolation Model section with lifecycle overview
- [ ] Branch Naming Conventions subsection
- [ ] Auto-Worktree for Milestones subsection with create/enter/teardown/merge
- [ ] WorktreeResolver subsection with three isolation modes
- [ ] State Synchronization subsection with forward and backward sync
- [ ] Plan Reconciliation for crash recovery subsection
- [ ] All claims have evidence labels

## Verification

- `grep -q "Worktree Isolation Model" analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md`
- `grep -q "WorktreeResolver" analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md`
- `grep -q "milestone/<MID>" analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md`
- `grep -q "syncWorktreeStateBack" analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md`
- `grep -c "\[repo code\]" analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md` returns >= 30 (cumulative with T01)

## Inputs

- `src/resources/extensions/gsd/worktree.ts` — pure utility functions
- `src/resources/extensions/gsd/worktree-manager.ts` — CRUD operations
- `src/resources/extensions/gsd/auto-worktree.ts` — milestone worktree lifecycle
- `src/resources/extensions/gsd/worktree-resolver.ts` — path state management
- `analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md` — T01 output to append to

## Expected Output

- `analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md` — second major section (Worktree Isolation Model)
