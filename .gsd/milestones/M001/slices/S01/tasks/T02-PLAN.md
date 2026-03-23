---
estimated_steps: 5
estimated_files: 1
skills_used: []
---

# T02: Write GSD2_SYSTEM_OVERVIEW.md with boundary inventory

**Slice:** S01 — Source Map, Boundary Inventory, and Evidence Method
**Milestone:** M001

## Description

Write the primary system overview document for the pack. This document serves as the entry point for readers and provides the initial Pi-vs-GSD boundary inventory with specific repo file references. It explains what GSD-2 is at the system level, documents the repo structure, and sets expectations for the rest of the pack.

## Steps

1. Write **What Is GSD-2** section explaining GSD-2 as a harness/orchestration system (not a product feature roadmap) — reference D001
2. Write **Repo Structure Overview** section documenting:
   - `packages/` directory: the Pi-owned runtime packages
   - `src/` directory: GSD-specific CLI, headless mode, and resource loading
   - `src/resources/extensions/gsd/` directory: the GSD-authored orchestration layer
   - `docs/` directory: secondary explanatory material
3. Write **Pi-vs-GSD Boundary Inventory** section with three subsections:
   - **Pi-Owned Runtime Modules**: For each package (`pi-agent-core`, `pi-ai`, `pi-coding-agent`, `pi-tui`, `native`), list key responsibilities and reference 2-3 specific files
   - **GSD-Authored Orchestration Modules**: List key GSD extension modules with responsibilities and file references (e.g., `auto*.ts` files, `commands-*.ts`, git/worktree logic)
   - **Unresolved Seams**: Explicitly note ownership boundaries that are unclear or shared, labeled as `[unresolved]`
4. Write **Subject-vs-Runner Guardrail** section with a concrete example showing the difference between the analyzed `.gsd/` model and the live planning artifacts
5. Write **Pack Document Map** section listing what subsequent documents will cover (forward references to S02-S07 outputs: runtime architecture, orchestration layer, context model, git/isolation, comparison, Atlas synthesis)

## Must-Haves

- [ ] Document explains GSD-2 as a harness/orchestration system
- [ ] Repo structure overview covers packages/, src/, docs/
- [ ] Pi-vs-GSD boundary inventory lists all Pi packages and key GSD modules with file references
- [ ] At least 2 unresolved seams are documented with explicit labels
- [ ] Subject-vs-runner guardrail is clear and includes concrete example
- [ ] Forward references to subsequent pack documents are present

## Verification

- `test -f analysis/gsd2-harness-decomposition/GSD2_SYSTEM_OVERVIEW.md` returns 0
- `grep -c "^## " analysis/gsd2-harness-decomposition/GSD2_SYSTEM_OVERVIEW.md` returns >= 5 (5+ sections)
- `grep -q "Pi-vs-GSD" analysis/gsd2-harness-decomposition/GSD2_SYSTEM_OVERVIEW.md` returns 0
- `grep -q "subject-vs-runner" analysis/gsd2-harness-decomposition/GSD2_SYSTEM_OVERVIEW.md` returns 0
- `grep -q "\[unresolved\]" analysis/gsd2-harness-decomposition/GSD2_SYSTEM_OVERVIEW.md` returns 0
- `grep -c "packages/pi-" analysis/gsd2-harness-decomposition/GSD2_SYSTEM_OVERVIEW.md` returns >= 3 (references Pi packages)
- `grep -c "src/resources/extensions/gsd" analysis/gsd2-harness-decomposition/GSD2_SYSTEM_OVERVIEW.md` returns >= 1 (references GSD extension)

## Inputs

- `analysis/gsd2-harness-decomposition/EVIDENCE_METHOD.md` — Evidence conventions to follow (from T01)
- `.gsd/DECISIONS.md` — D001-D007 for conventions and scope
- `.gsd/milestones/M001/M001-ROADMAP.md` — Boundary map showing what this document must produce for downstream slices

## Expected Output

- `analysis/gsd2-harness-decomposition/GSD2_SYSTEM_OVERVIEW.md` — System overview with boundary inventory
