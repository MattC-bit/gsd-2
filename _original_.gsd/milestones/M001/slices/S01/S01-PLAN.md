# S01: Source Map, Boundary Inventory, and Evidence Method

**Goal:** Establish the analysis workspace, document the evidence method with explicit source precedence, set the subject-vs-runner guardrail, and produce an initial Pi-vs-GSD boundary inventory grounded in repo references.

**Demo:** A reader can open `analysis/gsd2-harness-decomposition/`, read `EVIDENCE_METHOD.md` to understand how claims will be supported, read `GSD2_SYSTEM_OVERVIEW.md` to understand GSD-2 at a high level with explicit Pi-vs-GSD boundaries, and see the subject-vs-runner separation clearly stated.

## Must-Haves

- Analysis workspace exists at `analysis/gsd2-harness-decomposition/`
- `EVIDENCE_METHOD.md` documents source precedence, evidence tiers, and fact/inference labeling conventions
- `GSD2_SYSTEM_OVERVIEW.md` contains system overview and initial Pi-vs-GSD boundary inventory with repo file references
- Subject-vs-runner guardrail is explicit and prominent in both documents
- All boundary claims cite specific repo files or are labeled as unresolved

## Verification

- `test -d analysis/gsd2-harness-decomposition/` returns 0 (workspace exists)
- `test -f analysis/gsd2-harness-decomposition/EVIDENCE_METHOD.md` returns 0 (evidence method doc exists)
- `test -f analysis/gsd2-harness-decomposition/GSD2_SYSTEM_OVERVIEW.md` returns 0 (system overview exists)
- `grep -c "^## " analysis/gsd2-harness-decomposition/EVIDENCE_METHOD.md` returns >= 4 (4+ sections)
- `grep -c "^## " analysis/gsd2-harness-decomposition/GSD2_SYSTEM_OVERVIEW.md` returns >= 5 (5+ sections)
- `grep -q "Pi-vs-GSD" analysis/gsd2-harness-decomposition/GSD2_SYSTEM_OVERVIEW.md` returns 0 (boundary section exists)
- `grep -q "subject-vs-runner" analysis/gsd2-harness-decomposition/GSD2_SYSTEM_OVERVIEW.md` returns 0 (guardrail exists)

## Tasks

- [x] **T01: Create analysis workspace and write EVIDENCE_METHOD.md** `est:45m`
  - Why: The evidence method is foundational — all subsequent analysis docs must follow these conventions for source precedence and claim labeling. This is the methodological contract for the entire pack.
  - Files: `analysis/gsd2-harness-decomposition/EVIDENCE_METHOD.md`, `analysis/gsd2-harness-decomposition/README.md`
  - Do:
    1. Create `analysis/gsd2-harness-decomposition/` directory
    2. Write `README.md` as the pack index (brief intro, document list, audience, how to read)
    3. Write `EVIDENCE_METHOD.md` with:
       - Source precedence rules (repo code > repo docs > external > inference)
       - Evidence tier definitions with examples
       - Fact vs inference labeling conventions
       - Subject-vs-runner guardrail — explicit statement that analyzed `.gsd/` ≠ live planning artifacts
       - Cross-referencing conventions for other pack documents
    4. Ensure all conventions reference the decisions in `.gsd/DECISIONS.md` (D001-D007)
  - Verify: `test -f analysis/gsd2-harness-decomposition/EVIDENCE_METHOD.md && grep -c "^## " analysis/gsd2-harness-decomposition/EVIDENCE_METHOD.md`
  - Done when: Both files exist, EVIDENCE_METHOD.md has 4+ sections covering source precedence, evidence tiers, labeling, and subject-vs-runner guardrail

- [x] **T02: Write GSD2_SYSTEM_OVERVIEW.md with boundary inventory** `est:1h`
  - Why: This is the primary deliverable of S01 — the entry point document that explains what GSD-2 is, where Pi code ends and GSD code begins, and what the reader should expect from the rest of the pack.
  - Files: `analysis/gsd2-harness-decomposition/GSD2_SYSTEM_OVERVIEW.md`
  - Do:
    1. Write system overview section explaining GSD-2 as a harness/orchestration system
    2. Document repo structure at a high level (packages/, src/, docs/)
    3. Create Pi-vs-GSD Boundary Inventory section with:
       - Pi-owned modules: `packages/pi-agent-core/`, `packages/pi-ai/`, `packages/pi-coding-agent/`, `packages/pi-tui/`, `packages/native/` — with brief responsibility descriptions and key file references
       - GSD-authored modules: `src/resources/extensions/gsd/`, `src/cli.ts`, `src/headless*.ts` — with brief responsibility descriptions and key file references
       - Unresolved seams: places where ownership is unclear or shared (with explicit "unresolved" label)
    4. Document subject-vs-runner guardrail with concrete example
    5. List what subsequent pack documents will cover (forward references to S02-S07 outputs)
  - Verify: `test -f analysis/gsd2-harness-decomposition/GSD2_SYSTEM_OVERVIEW.md && grep -q "Pi-vs-GSD" analysis/gsd2-harness-decomposition/GSD2_SYSTEM_OVERVIEW.md`
  - Done when: Document exists, has 5+ sections, boundary inventory lists Pi modules and GSD modules with file references, subject-vs-runner guardrail is explicit

## Files Likely Touched

- `analysis/gsd2-harness-decomposition/README.md`
- `analysis/gsd2-harness-decomposition/EVIDENCE_METHOD.md`
- `analysis/gsd2-harness-decomposition/GSD2_SYSTEM_OVERVIEW.md`
