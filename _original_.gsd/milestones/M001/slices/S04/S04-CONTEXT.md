---
id: S04
milestone: M001
status: ready
---

# S04: Context Engineering and Disk-State Workflow Model — Context

<!-- Slice-scoped context. Milestone-only sections (acceptance criteria, completion class,
     milestone sequence) do not belong here — those live in the milestone context. -->

## Goal

Document how GSD assembles context from `.gsd/` workflow-state artifacts, explain the context budget system with exact constants and algorithms, specify all `.gsd/` file formats, and explicitly distinguish the analyzed-system workflow model from the live runner state of this reverse-engineering effort.

## Why this Slice

S03 explains orchestration control, but doesn't explain how context flows into prompts. S04 fills this gap by documenting:

1. **Context assembly model** — how `.gsd/` artifacts become LLM context
2. **Budget system** — exact ratios, thresholds, and truncation algorithms
3. **File formats** — frontmatter schemas, section structures, and relationships
4. **Subject-vs-runner guardrail** — concrete examples for workflow state

S05 (git/worktree isolation) and S06 (comparative analysis) both depend on understanding how `.gsd/` functions as workflow state. Without this, comparisons would conflate the analyzed model with the live artifacts.

## Scope

### In Scope

**Context assembly model (implementation-level detail):**
- How prompts are assembled from `.gsd/` artifacts
- Inline helper functions: `inlineDecisionsFromDb`, `inlineRequirementsFromDb`, `inlineProjectFromDb`, `inlinePriorMilestoneSummary`, `buildCarryForwardSection`
- Carry-forward pattern from prior task summaries
- Section extraction (`extractSection`, `extractMarkdownSection`)
- Context injection from prior step artifacts (`context-injector.ts`)

**Context budget system (all constants and algorithms):**
- Ratio constants: `SUMMARY_RATIO`, `INLINE_CONTEXT_RATIO`, `VERIFICATION_RATIO`
- Truncation algorithm: `truncateAtSectionBoundary`
- Context window resolution: model registry lookup, `DEFAULT_CONTEXT_WINDOW`
- Task count tiers: `TASK_COUNT_TIERS`
- Continue threshold: `CONTINUE_THRESHOLD_PERCENT`

**`.gsd/` file formats (full specifications):**
- Root files: `STATE.md`, `DECISIONS.md`, `REQUIREMENTS.md`, `PROJECT.md`, `QUEUE.md`, `KNOWLEDGE.md`
- Milestone files: `M###-ROADMAP.md`, `M###-CONTEXT.md`, `M###-SUMMARY.md`, `M###-META.json`
- Slice files: `S##-PLAN.md`, `S##-SUMMARY.md`, `S##-UAT.md`, `S##-CONTEXT.md`
- Task files: `T##-PLAN.md`, `T##-SUMMARY.md`, `continue.md`
- Runtime files: `runtime/`, `auto.lock`, `worktrees/`
- Frontmatter schemas for each file type
- Section structure and required/optional fields

**DB integration (all queries and caching):**
- `gsd-db.ts` — DB availability check, connection management
- `context-store.ts` — `queryDecisions`, `queryRequirements`, `queryProject`
- Fallback pattern: DB → filesystem
- Cache behavior: when cached, when invalidated

**Subject-vs-runner guardrail extension:**
- Table or list showing which `.gsd/` files are subject vs runner
- Concrete examples with file paths
- Explicit distinction between analyzed workflow model and live artifacts

**Produces:**
- `GSD2_CONTEXT_ENGINEERING_MODEL.md` — context assembly, budget system, file formats, DB integration, subject-vs-runner extension

### Out of Scope

- **Prompt templates**: `buildDiscussMilestonePrompt`, `buildExecuteTaskPrompt`, etc. are S03 scope
- **Dispatch logic**: How units are selected for dispatch (S03)
- **Git/worktree isolation**: Worktree lifecycle, sync behavior (S05)
- **External comparisons**: Comparative analysis (S06)

## Constraints

- **Exhaustive detail**: All budget constants, all file formats with frontmatter schemas, all inline helpers with logic
- **Subject-vs-runner table**: Concrete examples showing which `.gsd/` paths are subject vs runner
- **Self-contained**: File format specs must be readable without the actual `.gsd/` files present
- **Structural verification**: Machine-checkable artifact existence and section presence

## Integration Points

### Consumes

- `.gsd/milestones/M001/slices/S01/S01-CONTEXT.md` — evidence method, boundary inventory approach
- `.gsd/milestones/M001/slices/S03/S03-CONTEXT.md` — orchestration context (for prompt template scope)
- `analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md` — orchestration layer (after S03, for context)
- `analysis/gsd2-harness-decomposition/GSD2_HARNESS_MODEL.md` — harness model (after S03, for context flow)
- `analysis/gsd2-harness-decomposition/README.md` — existing guardrail definition

### Produces

- `analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` — context assembly, budget system, file formats, DB integration, subject-vs-runner extension
- Normalized context/state concepts usable in S06 comparison

## Open Questions

- **File format evolution**: If `.gsd/` formats change in future versions, how should the analysis pack be maintained? Current thinking: document what exists now; version history is out of scope.

- **DB cache invalidation**: The exact cache invalidation triggers are scattered. Should S04 trace all invalidation paths? Current thinking: document the primary invalidation points (`invalidateAllCaches`, file change detection); exhaustive tracing may be out of scope.

- **Continue.md format**: The continue file has a specific frontmatter schema. Should S04 include the full continue file spec? Current thinking: yes, since it's part of the workflow-state model.
