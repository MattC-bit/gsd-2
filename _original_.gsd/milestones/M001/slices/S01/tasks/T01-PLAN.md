---
estimated_steps: 4
estimated_files: 2
skills_used: []
---

# T01: Create analysis workspace and write EVIDENCE_METHOD.md

**Slice:** S01 — Source Map, Boundary Inventory, and Evidence Method
**Milestone:** M001

## Description

Create the dedicated analysis workspace at `analysis/gsd2-harness-decomposition/` and write the foundational `EVIDENCE_METHOD.md` document. This document establishes the methodological contract for the entire pack: source precedence rules, evidence tier definitions, fact vs inference labeling conventions, and the subject-vs-runner guardrail. All subsequent analysis documents must follow these conventions.

## Steps

1. Create the directory `analysis/gsd2-harness-decomposition/`
2. Write `README.md` as the pack index with:
   - Brief introduction to the pack purpose
   - Document list (will grow as slices complete)
   - Intended audience (Atlas harness builders)
   - How to read the pack (suggested order)
3. Write `EVIDENCE_METHOD.md` with the following sections:
   - **Source Precedence**: repo code first, repo docs second, external comparisons third, synthesis labeled as inference (reference D002)
   - **Evidence Tiers**: define Tier 1 (repo code), Tier 2 (repo docs), Tier 3 (external context), Tier 4 (inference/synthesis) with concrete examples
   - **Fact vs Inference Labeling**: conventions for marking claims (e.g., `[repo code]`, `[repo doc]`, `[inference]`, `[unresolved]`)
   - **Subject-vs-Runner Guardrail**: explicit statement that the analyzed GSD-2 `.gsd/` workflow model is distinct from the live `.gsd/` planning artifacts of this reverse-engineering run (reference D007)
   - **Cross-Reference Conventions**: how to cite files, link to decisions, and reference other pack documents
4. Ensure all conventions reference relevant decisions from `.gsd/DECISIONS.md` (D001-D007)

## Must-Haves

- [ ] `analysis/gsd2-harness-decomposition/` directory exists
- [ ] `README.md` exists with pack intro and document index
- [ ] `EVIDENCE_METHOD.md` exists with sections for source precedence, evidence tiers, labeling, and subject-vs-runner guardrail
- [ ] All labeling conventions have concrete examples
- [ ] Subject-vs-runner guardrail is prominent and unambiguous

## Verification

- `test -d analysis/gsd2-harness-decomposition/` returns 0
- `test -f analysis/gsd2-harness-decomposition/README.md` returns 0
- `test -f analysis/gsd2-harness-decomposition/EVIDENCE_METHOD.md` returns 0
- `grep -c "^## " analysis/gsd2-harness-decomposition/EVIDENCE_METHOD.md` returns >= 4 (4+ sections)
- `grep -q "subject-vs-runner" analysis/gsd2-harness-decomposition/EVIDENCE_METHOD.md` returns 0

## Inputs

- `.gsd/DECISIONS.md` — Contains D001-D007 which define the conventions to document
- `.gsd/PROJECT.md` — Project context for audience and scope
- `.gsd/REQUIREMENTS.md` — R010, R011, R012, R013 requirements this task advances

## Expected Output

- `analysis/gsd2-harness-decomposition/README.md` — Pack index document
- `analysis/gsd2-harness-decomposition/EVIDENCE_METHOD.md` — Evidence method documentation
