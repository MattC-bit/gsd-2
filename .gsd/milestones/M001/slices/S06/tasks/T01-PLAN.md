---
estimated_steps: 6
estimated_files: 1
skills_used:
  - review
---

# T01: Research external systems and build primitives matrix

**Slice:** S06 — Comparative Structural Analysis
**Milestone:** M001

## Description

Establish the comparison foundation by documenting each system's primitives from external sources, then synthesize the shared-primitives matrix. This task creates the core structural analysis document with the six-dimensional primitives matrix that enables readers to compare systems at a glance.

## Steps

1. Create the document with overview and subject-vs-runner guardrail
2. Document Claude Code architecture from external sources:
   - Tech stack: TypeScript, React, Ink, Yoga, Bun [external]
   - Agent primitives: agent loop, system prompt, tools, permissions [external]
   - Session management: fresh session per interaction pattern [external]
   - MCP integration for external tools [external]
3. Document Codex CLI architecture from external sources:
   - AGENTS.md and SKILL.md files for workflow capture [external]
   - MCP integration for tools [external]
   - Auto/approval modes for execution [external]
4. Document ACP architecture from protocol specification:
   - RESTful API for agent interoperability [external]
   - Message structure and discovery mechanisms [external]
   - Stateful/stateless operation patterns [external]
5. Build shared-primitives matrix with six dimensions:
   - Model Layer: How systems interact with LLMs
   - Agent Layer: Core agent loop and tooling
   - Harness Layer: Session management, recovery, isolation
   - Orchestration Layer: Workflow control, dispatch, phases
   - Protocol/Client Surface: How capabilities are exposed
   - Workflow/Doc-State Layer: How state is persisted and managed
6. Add GSD-2 column using S02-S05 analysis with `[repo code]` labels

## Must-Haves

- [ ] Document exists with overview section
- [ ] Subject-vs-runner guardrail section present
- [ ] Claude Code documented with `[external]` labels
- [ ] Codex CLI documented with `[external]` labels
- [ ] ACP documented with `[external]` labels
- [ ] Shared-primitives matrix with six dimensions and four system columns
- [ ] GSD-2 column uses `[repo code]` labels from prior analysis

## Verification

- `test -f analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md`
- `grep -q "Shared-Primitives Matrix" analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md`
- `grep -q "Claude Code" analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md`
- `grep -q "Codex CLI" analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md`
- `grep -q "ACP" analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md`
- `grep -q "subject-vs-runner" analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md`
- `grep -c "\[external\]" analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md` returns >= 10

## Inputs

- `analysis/gsd2-harness-decomposition/GSD2_RUNTIME_ARCHITECTURE.md` — GSD-2 runtime layer primitives
- `analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md` — GSD-2 orchestration primitives
- `analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` — GSD-2 context/state primitives
- `analysis/gsd2-harness-decomposition/GSD2_GIT_AND_ISOLATION_MODEL.md` — GSD-2 isolation primitives
- `analysis/gsd2-harness-decomposition/EVIDENCE_METHOD.md` — Evidence tier definitions

## Expected Output

- `analysis/gsd2-harness-decomposition/GSD2_COMPARATIVE_ANALYSIS.md` — Comparative analysis document with shared-primitives matrix

## Observability Impact

- **Signals change:** No runtime signals — this is a documentation generation task.
- **Inspection:** Future agents can verify the document structure via the grep-based verification commands. The evidence-tier labels (`[external]`, `[repo code]`) serve as inline traceability markers.
- **Failure visibility:** Missing sections or insufficient evidence labels surface as grep exit code failures during verification.
