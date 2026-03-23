---
id: S03
milestone: M001
status: ready
---

# S03: Orchestration Layer and Execution Control — Context

<!-- Slice-scoped context. Milestone-only sections (acceptance criteria, completion class,
     milestone sequence) do not belong here — those live in the milestone context. -->

## Goal

Document how GSD transforms the Pi runtime substrate into an orchestration system via dispatch rules, auto-mode state machine, verification gates, guards, prompt builders, and control surfaces — and synthesize a repo-faithful harness model.

## Why this Slice

S02/S02b establish the runtime substrate (session assembly, events, retry, compaction, persistence). S03 explains how GSD orchestrates that substrate into a coherent workflow system. Without this, the boundary between "what Pi provides" and "what GSD orchestrates" remains incomplete.

S04 (context engineering), S05 (git/worktree isolation), and S06 (comparative analysis) all depend on understanding the orchestration layer first.

## Scope

### In Scope

**Core orchestration components:**
- **Dispatch system**: `auto-dispatch.ts` — DispatchRule interface, rule matching logic, rule chaining
- **Auto-mode state machine**: `auto.ts` — main loop with every step enumerated
- **AutoSession**: All ~40 properties enumerated with types and purposes
- **State derivation**: `state.ts` — derivation algorithm with edge cases (ghost milestones, validation verdicts)
- **Prompt builders**: `auto-prompts.ts` — full prompt templates inline for all ~15 builder functions
- **Verification gate**: `auto-verification.ts` — gate design, auto-fix retry logic, evidence JSON
- **Recovery**: All recovery functions from `auto-recovery.ts`, `auto-timeout-recovery.ts`, `auto-stuck-detection.ts`
- **Guards**: Key guards with rationale (dispatch-guard.ts and others)
- **Model routing**: `auto-model-selection.ts`, `model-router.ts` — routing as orchestration responsibility
- **Budget enforcement**: `auto-budget.ts` — alerts, enforcement actions, state machine
- **Control surfaces**: `/gsd` commands (dispatcher structure + categories), dashboard overlay
- **Orchestration ownership map**: Module-level table for orchestration components

**Produces:**
- `GSD2_ORCHESTRATION_LAYER.md` — orchestration documentation with inline prompt templates
- `GSD2_HARNESS_MODEL.md` — first-pass repo-faithful control-model synthesis

### Out of Scope

- **Worktree logic**: `auto-worktree.ts` deferred to S05 (Git, Worktree, and Isolation Model)
- **Git operations**: `git-service.ts` deferred to S05
- **Runtime substrate**: Session assembly, events, retry, compaction, persistence (S02/S02b)
- **Context engineering**: Prompt assembly model, `.gsd/` workflow state (S04)
- **External comparisons**: Comparative analysis (S06)
- **Atlas synthesis**: Glossary, crosswalk, rebuild/wrap/delegate (S07)

## Constraints

- **Exhaustive detail**: All AutoSession properties, all recovery functions, derivation algorithm detail, full prompt templates, every loop step
- **Inline prompts**: Full prompt templates inline in `GSD2_ORCHESTRATION_LAYER.md` (not separate appendix)
- **Separate harness model**: `GSD2_HARNESS_MODEL.md` is a separate document
- **Model routing in scope**: Routing is orchestration responsibility, not runtime substrate
- **Worktree deferred**: Worktree operations are mentioned but detailed in S05

## Integration Points

### Consumes

- `.gsd/milestones/M001/slices/S01/S01-CONTEXT.md` — evidence method, boundary inventory approach
- `.gsd/milestones/M001/slices/S02/S02-CONTEXT.md` — runtime architecture context
- `.gsd/milestones/M001/slices/S02b/S02b-CONTEXT.md` — session internals context (retry/compaction/persistence)
- `analysis/gsd2-harness-decomposition/GSD2_RUNTIME_ARCHITECTURE.md` — runtime architecture (after S02)
- `analysis/gsd2-harness-decomposition/GSD2_SESSION_INTERNALS.md` — session internals (after S02b)

### Produces

- `analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md` — orchestration documentation with inline prompts
- `analysis/gsd2-harness-decomposition/GSD2_HARNESS_MODEL.md` — first-pass harness model synthesis
- Orchestration ownership table — module-level granularity with CODE references

## Open Questions

- **Harness model scope**: Should the harness model focus only on orchestration patterns, or attempt a broader synthesis of runtime + orchestration? Current thinking: ground it in orchestration but reference S02/S02b for runtime context.

- **Prompt template evolution**: If prompts change in future versions, how should the analysis pack be maintained? Current thinking: document what exists now; version history is out of scope.

- **Loop step granularity**: "Every loop step" could mean line-by-line or logical phases. Current thinking: document logical phases with key code references, not every line.
