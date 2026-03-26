---
estimated_steps: 5
estimated_files: 6
skills_used:
  - best-practices
---

# T01: Document Context Assembly Pipeline

**Slice:** S04 — Context Engineering and Disk-State Workflow Model
**Milestone:** M001

## Description

Trace and document the context assembly pipeline from Pi-owned `buildSystemPrompt()` through GSD-authored unit-specific prompt builders. This covers the first half of S04's goal — understanding how prompts are constructed from templates, inlined content, and budget constraints.

## Steps

1. Read `packages/pi-coding-agent/src/core/system-prompt.ts` and document:
   - `buildSystemPrompt()` function signature and options
   - Tool descriptions and guideline construction
   - Project context file appending
   - Skills section formatting
   - Date/time and working directory injection

2. Read `src/resources/extensions/gsd/auto-prompts.ts` and document:
   - Unit-specific prompt builders (`buildPlanSlicePrompt`, `buildExecuteTaskPrompt`, etc.)
   - Inline helpers (`inlineFile`, `inlineFileOptional`, `inlineFileSmart`)
   - Dependency summary inlining (`inlineDependencySummaries`)
   - Skill activation block construction
   - DB-aware inline helpers (`inlineDecisionsFromDb`, `inlineRequirementsFromDb`)

3. Read `src/resources/extensions/gsd/context-budget.ts` and document:
   - Budget ratio constants (SUMMARY_RATIO, INLINE_CONTEXT_RATIO, VERIFICATION_RATIO)
   - `computeBudgets()` function and return structure
   - `truncateAtSectionBoundary()` for section-aware truncation
   - `resolveExecutorContextWindow()` for context window resolution

4. Read `src/resources/extensions/gsd/prompt-loader.ts` and document:
   - Template caching via `warmCache()`
   - `loadPrompt()` with variable substitution
   - `loadTemplate()` and `inlineTemplate()` helpers

5. Write the "Context Assembly Pipeline" section in `GSD2_CONTEXT_ENGINEERING_MODEL.md` with:
   - Pi-owned vs GSD-authored responsibilities table
   - Code evidence with line references
   - Evidence tier labels (`[repo code]`)

## Must-Haves

- [ ] Document Pi-owned `buildSystemPrompt()` with code evidence
- [ ] Document GSD-authored unit-specific builders with code evidence
- [ ] Document context budget system with constants and functions
- [ ] Document template loading with caching behavior
- [ ] Use `[repo code]` evidence labels consistently

## Verification

- `grep -q "Context Assembly" analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` returns 0
- `grep -c "\[repo code\]" analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` returns >= 10

## Inputs

- `packages/pi-coding-agent/src/core/system-prompt.ts` — Pi-owned system prompt builder
- `src/resources/extensions/gsd/auto-prompts.ts` — GSD-authored unit-specific prompt builders
- `src/resources/extensions/gsd/context-budget.ts` — Context budget engine
- `src/resources/extensions/gsd/prompt-loader.ts` — Template loading utilities
- `analysis/gsd2-harness-decomposition/GSD2_SYSTEM_OVERVIEW.md` — Existing boundary inventory for context

## Observability Impact

This task produces documentation only. No runtime signals change. The observability surfaces are:

- **Document creation**: `test -f analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` confirms task completion
- **Evidence quality**: `grep -c "\[repo code\]" ... >= 10` ensures sufficient code references
- **Section presence**: `grep -q "Context Assembly"` confirms the required section exists

Future agents can inspect the document to understand context assembly without reading source files.

## Expected Output

- `analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` — New document with Context Assembly Pipeline section
