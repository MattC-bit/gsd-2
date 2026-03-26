---
estimated_steps: 4
estimated_files: 2
skills_used: []
---

# T04: Write orchestration ownership map and cross-references

**Slice:** S03 — Orchestration Layer and Execution Control
**Milestone:** M001

## Description

Synthesize the orchestration layer documentation with an ownership map that clearly separates Pi-owned runtime responsibilities from GSD-authored orchestration responsibilities. Add cross-references to S02's runtime documentation and ensure the subject-vs-runner guardrail is explicit.

## Steps

1. Review `analysis/gsd2-harness-decomposition/GSD2_RUNTIME_ARCHITECTURE.md` to understand what runtime surfaces orchestration calls into.

2. Add to `GSD2_ORCHESTRATION_LAYER.md`:
   - **Orchestration Ownership Map** — Table of responsibilities with Pi-owned vs GSD-authored classification
   - **Cross-References** — Links to runtime architecture sections for session lifecycle, event flow, persistence

3. Add **Subject-vs-Runner Guardrail** section specific to orchestration — explain that `auto/session.ts` is NOT the analyzed session state, it's the runner's auto-mode tracking.

4. Verify all sections have evidence tier labels. Add missing labels where needed.

## Must-Haves

- [ ] Orchestration Ownership Map section with Pi/GSD classification
- [ ] Cross-references to GSD2_RUNTIME_ARCHITECTURE.md
- [ ] Subject-vs-runner guardrail section for orchestration
- [ ] All prior sections have evidence tier labels
- [ ] Document has 6+ major sections

## Verification

- `grep -q "Orchestration.*Ownership\|Ownership Map" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`
- `grep -q "GSD2_RUNTIME_ARCHITECTURE" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`
- `grep -q "subject-vs-runner" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`
- `grep -c "^## " analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md` returns >= 6

## Inputs

- `analysis/gsd2-harness-decomposition/GSD2_RUNTIME_ARCHITECTURE.md` — Runtime architecture for cross-references
- `analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md` — Current document to extend

## Expected Output

- `analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md` — Complete document with ownership map, cross-references, and guardrail
