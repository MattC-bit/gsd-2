---
estimated_steps: 3
estimated_files: 2
skills_used:
  - best-practices
---

# T03: Add Subject-vs-Runner Guardrail and Cross-References

**Slice:** S04 — Context Engineering and Disk-State Workflow Model
**Milestone:** M001

## Description

Add the critical subject-vs-runner guardrail specific to `.gsd/` artifacts and create cross-references to S02 and S03 documents. This completes S04 by ensuring readers understand the distinction between the analyzed GSD-2 system's `.gsd/` model and the live runner's artifacts.

## Steps

1. Add subject-vs-runner guardrail section to `GSD2_CONTEXT_ENGINEERING_MODEL.md`:
   - Explain the distinction: analyzed GSD-2's `.gsd/` workflow-state model vs live runner's `.gsd/` planning artifacts
   - Provide concrete example: "When this document describes `deriveState()` reading ROADMAP.md, it refers to the analyzed GSD-2 system's behavior, not to the live M001 roadmap that defines this analysis project."
   - Reference D007 (subject-vs-runner separation decision)

2. Add cross-references to related documents:
   - S02's `GSD2_RUNTIME_ARCHITECTURE.md` — for session context and runtime behavior
   - S03's `GSD2_ORCHESTRATION_LAYER.md` — for phase transitions and dispatch behavior
   - Include brief notes on what each cross-reference provides

3. Update `GSD2_SYSTEM_OVERVIEW.md` to reference the new document:
   - Add entry to the document index
   - Note that context engineering and disk-state model are now documented

## Must-Haves

- [ ] Subject-vs-runner guardrail section with concrete example
- [ ] Cross-references to S02 and S03 documents
- [ ] Update to GSD2_SYSTEM_OVERVIEW.md document index

## Verification

- `grep -q "subject-vs-runner" analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` returns 0
- `grep -q "GSD2_RUNTIME_ARCHITECTURE" analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` returns 0
- `grep -q "GSD2_ORCHESTRATION_LAYER" analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` returns 0
- `grep -q "GSD2_CONTEXT_ENGINEERING_MODEL" analysis/gsd2-harness-decomposition/GSD2_SYSTEM_OVERVIEW.md` returns 0

## Inputs

- `analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` — Partial document from T01/T02
- `analysis/gsd2-harness-decomposition/GSD2_SYSTEM_OVERVIEW.md` — Existing system overview for cross-reference update
- `analysis/gsd2-harness-decomposition/GSD2_RUNTIME_ARCHITECTURE.md` — S02 output for reference
- `analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md` — S03 output for reference

## Expected Output

- `analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` — Complete document with guardrail and cross-references
- `analysis/gsd2-harness-decomposition/GSD2_SYSTEM_OVERVIEW.md` — Updated with reference to new document
