# S04: Context Engineering and Disk-State Workflow Model

**Goal:** A reader can explain how prompts/context are assembled, how analyzed-system `.gsd/` artifacts function as workflow state, and why that model must not be conflated with the live planning artifacts of this analysis run.

**Demo:** The `analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` document exists with complete coverage of context assembly, disk-state derivation, and subject-vs-runner guardrails.

## Must-Haves

- Document the context assembly pipeline: Pi-owned `buildSystemPrompt()`, GSD-authored unit-specific prompt builders, context budget system, and template loading
- Document the disk-state workflow model: `deriveState()` as source of truth, path resolution, file parsing, and `.gsd/` artifact structure
- Document how phase transitions work via disk artifacts (not mutable state tracking)
- Include subject-vs-runner guardrail specific to `.gsd/` artifacts with concrete examples
- Cross-reference S02 runtime architecture and S03 orchestration layer where relevant
- Use evidence tier labels (`[repo code]`, `[inference]`) consistently

## Proof Level

- This slice proves: contract (document coverage and cross-reference integrity)
- Real runtime required: no
- Human/UAT required: no

## Verification

- `test -f analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` returns 0
- `grep -c "^## " analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` returns >= 6 (6+ major sections)
- `grep -q "Context Assembly" analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` returns 0
- `grep -q "Disk-State Workflow Model" analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` returns 0
- `grep -q "subject-vs-runner" analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` returns 0
- `grep -c "\[repo code\]" analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` returns >= 20 (evidence labels present)
- `grep -q "buildSystemPrompt" analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` returns 0 (diagnostic: key function documented)
- `grep -q "computeBudgets" analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` returns 0 (diagnostic: budget function documented)

## Integration Closure

- Upstream surfaces consumed: S02's `GSD2_RUNTIME_ARCHITECTURE.md` (session context), S03's `GSD2_ORCHESTRATION_LAYER.md` (phase transitions, dispatch)
- New wiring introduced in this slice: none (documentation only)
- What remains before the milestone is truly usable end-to-end: S05 (git/isolation), S06 (comparison), S07 (Atlas synthesis)

## Tasks

- [x] **T01: Document context assembly pipeline** `est:45m`
  - Why: Context assembly is the first half of S04's goal — understanding how prompts are built from templates, inlined content, and budget constraints.
  - Files: `packages/pi-coding-agent/src/core/system-prompt.ts`, `src/resources/extensions/gsd/auto-prompts.ts`, `src/resources/extensions/gsd/context-budget.ts`, `src/resources/extensions/gsd/prompt-loader.ts`, `analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md`
  - Do: Trace the context assembly pipeline from Pi-owned `buildSystemPrompt()` through GSD-authored unit-specific builders (e.g., `buildPlanSlicePrompt`, `buildExecuteTaskPrompt`). Document the inline helpers (`inlineFile`, `inlineDependencySummaries`), context budget system (SUMMARY_RATIO, INLINE_CONTEXT_RATIO, VERIFICATION_RATIO), section-boundary truncation, and template loading with eager caching. Include code evidence with line references.
  - Verify: `grep -q "Context Assembly" analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` returns 0, `grep -c "\[repo code\]" analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` returns >= 10
  - Done when: The document has a complete "Context Assembly Pipeline" section covering Pi-owned core, GSD-authored builders, budget system, and template loading with evidence labels.

- [x] **T02: Document disk-state workflow model and phase derivation** `est:45m`
  - Why: The disk-state model is the second half of S04's goal — understanding how `.gsd/` artifacts function as workflow state.
  - Files: `src/resources/extensions/gsd/state.ts`, `src/resources/extensions/gsd/paths.ts`, `src/resources/extensions/gsd/files.ts`, `analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md`
  - Do: Trace the disk-state workflow model from `deriveState()` as the source of truth. Document how phase is derived from disk artifacts (not tracked), the path resolution hierarchy (gsdRoot → milestone → slice → task), file parsing functions (parseRoadmap, parsePlan, parseSummary), and the native batch parsing optimization. Include code evidence with line references.
  - Verify: `grep -q "Disk-State Workflow Model" analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` returns 0, `grep -c "\[repo code\]" analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` returns >= 20
  - Done when: The document has a complete "Disk-State Workflow Model" section covering deriveState(), path resolution, file parsing, and phase derivation with evidence labels.

- [x] **T03: Add subject-vs-runner guardrail and cross-references** `est:20m`
  - Why: The subject-vs-runner guardrail is critical for preventing conflation between the analyzed GSD-2 `.gsd/` model and the live runner's artifacts.
  - Files: `analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md`, `analysis/gsd2-harness-decomposition/GSD2_SYSTEM_OVERVIEW.md`
  - Do: Add a subject-vs-runner guardrail section specific to `.gsd/` artifacts with concrete examples. Add cross-references to S02's runtime architecture (session context) and S03's orchestration layer (phase transitions, dispatch). Update GSD2_SYSTEM_OVERVIEW.md to reference the new document.
  - Verify: `grep -q "subject-vs-runner" analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` returns 0, `grep -q "GSD2_RUNTIME_ARCHITECTURE" analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` returns 0, `grep -q "GSD2_ORCHESTRATION_LAYER" analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` returns 0
  - Done when: The document has a subject-vs-runner guardrail section and cross-references to S02 and S03 documents.

## Observability / Diagnostics

This is a documentation-only slice with no runtime signals. Inspection surfaces are:

- **Document coverage**: `grep -c "^## " analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` — major section count
- **Evidence quality**: `grep -c "\[repo code\]" analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` — grounded claims
- **Cross-reference integrity**: `grep -q "GSD2_RUNTIME_ARCHITECTURE\|GSD2_ORCHESTRATION_LAYER" analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` — links to related docs
- **Subject-vs-runner guardrail**: `grep -q "subject-vs-runner" analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` — conflation prevention

Failure modes:
- Missing evidence labels → claims not grounded in code
- Missing cross-references → readers can't navigate the pack
- Missing guardrail section → conflation risk between subject and runner artifacts

## Files Likely Touched

- `analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` — New document
- `analysis/gsd2-harness-decomposition/GSD2_SYSTEM_OVERVIEW.md` — Add cross-reference
