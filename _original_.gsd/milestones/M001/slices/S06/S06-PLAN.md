# S06: Comparative Structural Analysis

**Goal:** A reader can compare GSD-2 structurally with Claude Code, Codex, and ACP-style systems by shared primitives, owned layers, and design bets using clearly labeled evidence tiers.
**Demo:** The `GSD2_COMPARATIVE_ANALYSIS.md` document contains a shared-primitives matrix, structural comparison tables, design-bet synthesis, and evidence-tier labels throughout.

## Must-Haves

- Shared-primitives matrix covering model, agent, harness, orchestration layer, protocol/client surface, and workflow/doc-state layer
- Structural comparison tables for GSD-2 vs Claude Code vs Codex CLI vs ACP-style systems
- Design-bet synthesis identifying architectural choices each system makes
- Evidence-tier labels (`[repo code]`, `[repo doc]`, `[external]`, `[inference]`) applied consistently
- Subject-vs-runner guardrail maintained throughout

## Proof Level

- This slice proves: contract (document structure and content requirements)
- Real runtime required: no
- Human/UAT required: no

## Verification

- `test -f analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md`
- `grep -c "^## " analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md` returns >= 6 (6+ major sections)
- `grep -q "Shared-Primitives Matrix" analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md`
- `grep -q "Claude Code" analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md`
- `grep -q "Codex CLI" analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md`
- `grep -q "ACP" analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md`
- `grep -c "\[repo code\]\|\[repo doc\]\|\[external\]\|\[inference\]" analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md` returns >= 20
- `grep -q "subject-vs-runner" analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md`
- `grep -c "\[external\]" analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md` returns >= 10 (diagnostic: confirms external research populated)

## Tasks

- [x] **T01: Research external systems and build primitives matrix** `est:1h`
  - Why: Establish the comparison foundation by documenting each system's primitives from external sources, then synthesize the shared-primitives matrix.
  - Files: `analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md`
  - Do:
    1. Document Claude Code architecture from external sources (Pragmatic Engineer article, Anthropic docs) with `[external]` labels
    2. Document Codex CLI architecture from external sources (OpenAI docs, blog posts) with `[external]` labels
    3. Document ACP architecture from protocol specification with `[external]` labels
    4. Build shared-primitives matrix with six dimensions: model, agent, harness, orchestration, protocol/client, workflow/doc-state
    5. Include subject-vs-runner guardrail section
  - Verify: `test -f analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md` and grep checks pass
  - Done when: Document exists with shared-primitives matrix and all four systems documented

- [x] **T02: Write comparison tables and design-bet synthesis** `est:1h`
  - Why: Complete the comparative analysis with detailed structural comparison tables and synthesis of architectural choices.
  - Files: `analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md`
  - Do:
    1. Add structural comparison tables for each primitive dimension
    2. Document design bets for each system (what architectural choices they make and why)
    3. Add synthesis section comparing owned layers and responsibility boundaries
    4. Apply evidence-tier labels throughout with `[inference]` for synthesis claims
    5. Add cross-references to other pack documents
  - Verify: `grep -c "^## " analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md` returns >= 6
  - Done when: Document has 6+ major sections with comparison tables, design-bet synthesis, and 20+ evidence labels

## Files Likely Touched

- `analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md`
