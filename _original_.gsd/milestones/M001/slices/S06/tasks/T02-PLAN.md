---
estimated_steps: 5
estimated_files: 1
skills_used:
  - review
---

# T02: Write comparison tables and design-bet synthesis

**Slice:** S06 — Comparative Structural Analysis
**Milestone:** M001

## Description

Complete the comparative analysis with detailed structural comparison tables and synthesis of architectural choices. This task adds depth to the primitives matrix by explaining the design bets each system makes and their implications for rebuild/wrap/delegate decisions.

## Steps

1. Add detailed comparison tables for each primitive dimension:
   - Model Layer comparison table (provider abstraction, context management, retry/recovery)
   - Agent Layer comparison table (tool system, event flow, execution model)
   - Harness Layer comparison table (session lifecycle, isolation, recovery)
   - Orchestration Layer comparison table (workflow phases, dispatch, verification)
   - Protocol/Client Surface comparison table (MCP, ACP, custom protocols)
   - Workflow/Doc-State Layer comparison table (persistence format, state derivation)

2. Document design bets for each system:
   - GSD-2: Disk-state as source of truth, worktree isolation, milestone-driven workflow [repo code]
   - Claude Code: Agent-first terminal UX, MCP protocol, subagents for parallelism [external]
   - Codex CLI: AGENTS.md/SKILL.md as project-local workflow capture, approval modes [external]
   - ACP: Protocol-first interoperability, framework-agnostic RESTful API [external]

3. Add synthesis section comparing owned layers:
   - Which responsibilities each system owns vs delegates
   - Where boundaries are explicit vs implicit
   - What each system would need to rebuild/wrap/delegate from GSD-2

4. Apply evidence-tier labels throughout:
   - Use `[repo code]` for GSD-2 claims grounded in S02-S05 analysis
   - Use `[external]` for external system claims
   - Use `[inference]` for synthesis and cross-system conclusions

5. Add cross-references to other pack documents

## Must-Haves

- [ ] Six comparison tables (one per primitive dimension)
- [ ] Design-bet section for each system
- [ ] Owned-layers synthesis section
- [ ] Evidence-tier labels applied throughout (20+ labels total)
- [ ] Cross-references to other pack documents

## Verification

- `grep -c "^## " analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md` returns >= 6
- `grep -q "Design Bets" analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md`
- `grep -q "Owned Layers" analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md`
- `grep -c "\[repo code\]\|\[repo doc\]\|\[external\]\|\[inference\]" analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md` returns >= 20
- `grep -q "GSD2_RUNTIME_ARCHITECTURE" analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md`

## Inputs

- `analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md` — Document from T01 with primitives matrix
- `analysis/gsd2-harness-decomposition/GSD2_RUNTIME_ARCHITECTURE.md` — Runtime details for synthesis
- `analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md` — Orchestration details for synthesis
- `analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` — Context details for synthesis
- `analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md` — Isolation details for synthesis

## Expected Output

- `analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md` — Complete comparative analysis document
