# S03: Orchestration Layer and Execution Control

**Goal:** Document how GSD turns the Pi runtime substrate into an orchestration system via dispatch, auto mode, verification, guards, prompts, and control surfaces — producing a complete picture of GSD's workflow engine.

**Demo:** A reader can explain the dispatch table's rule-based phase→unit mapping, describe how auto-mode's state machine cycles between units, identify where verification gates block or retry, understand dispatch guards prevent out-of-order execution, and trace the tool registration boundary between Pi and GSD.

## Must-Haves

- `GSD2_ORCHESTRATION_LAYER.md` with sections covering: auto-mode state machine, dispatch table rules, verification gates, dispatch guards, control surfaces, and orchestration ownership map
- Resolution of seam #2 (tool definition boundaries) with evidence-traced ownership claims
- Subject-vs-runner guardrail applied throughout
- Evidence tier labels (`[repo code]`, `[repo doc]`, `[inference]`, `[unresolved]`) on all claims
- Clear Pi-vs-GSD ownership map for orchestration responsibilities

## Proof Level

- This slice proves: contract (documentation with evidence-grounded claims)
- Real runtime required: no (documentation only)
- Human/UAT required: no (automated verification sufficient)

## Verification

- `test -f analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`
- `grep -c "^## " analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md` returns >= 6 (6+ major sections)
- `grep -q "Auto-Mode State Machine" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`
- `grep -q "Dispatch Table" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`
- `grep -q "Verification Gate" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`
- `grep -q "Dispatch Guard" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`
- `grep -q "\[repo code\]" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`
- `grep -q "subject-vs-runner" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`
- `grep -q "Tool.*boundar\|tool.*registr" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md` (tool boundary documentation)

## Tasks

- [x] **T01: Document auto-mode state machine and dispatch table** `est:1h30m`
  - Why: The dispatch table is the heart of GSD's orchestration — it defines phase→unit mapping declaratively and is the control flow for auto-mode.
  - Files: `src/resources/extensions/gsd/auto.ts`, `src/resources/extensions/gsd/auto-dispatch.ts`, `src/resources/extensions/gsd/auto/session.ts`, `src/resources/extensions/gsd/state.ts`
  - Do: Read `auto-dispatch.ts` to trace dispatch rules, `auto.ts` for the main loop, `auto/session.ts` for session state, and `state.ts` for state derivation. Document the state machine phases, rule evaluation order, and unit dispatch flow. Include the workflow phase state machine diagram and dispatch rule table.
  - Verify: `grep -c "DispatchRule" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md` returns >= 1, `grep -q "needs-discussion\|pre-planning\|planning\|executing" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`
  - Done when: Document has Auto-Mode State Machine section with phase transitions, Dispatch Table section with rule evaluation explanation, and evidence labels on claims.

- [x] **T02: Document verification gates and dispatch guards** `est:1h`
  - Why: Verification gates and dispatch guards are GSD's safety rails — they prevent invalid state transitions and ensure quality gates before progression.
  - Files: `src/resources/extensions/gsd/auto-verification.ts`, `src/resources/extensions/gsd/verification-gate.ts`, `src/resources/extensions/gsd/dispatch-guard.ts`
  - Do: Read verification files to document gate execution, command discovery, failure context formatting, and auto-fix retry logic. Read dispatch-guard.ts to document out-of-order prevention and dependency-aware ordering. Include verification command discovery order.
  - Verify: `grep -q "verification.*gate\|Verification Gate" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`, `grep -q "dispatch.*guard\|Dispatch Guard" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`
  - Done when: Document has Verification Gate section with discovery order and retry logic, Dispatch Guard section with ordering rules, and evidence labels on claims.

- [x] **T03: Document tool boundaries and resolve seam #2** `est:1h`
  - Why: Seam #2 (tool definition boundaries) was left unresolved by S02 — this task traces how tools flow from Pi factories to GSD registration, completing the ownership picture.
  - Files: `src/resources/extensions/gsd/bootstrap/dynamic-tools.ts`, `src/resources/extensions/gsd/bootstrap/db-tools.ts`, `src/cli.ts`, `packages/pi-coding-agent/src/core/sdk.ts`
  - Do: Read dynamic-tools.ts and db-tools.ts to trace tool registration. Search for `createBashTool`, `createEditTool`, etc. in Pi packages to find factory locations. Document the factory→wrap→register pipeline. Resolve seam #2 with explicit ownership claims.
  - Verify: `grep -q "seam #2\|Seam #2\|tool.*boundar" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`, `grep -q "createBashTool\|createEditTool" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`
  - Done when: Document has Tool Registration Boundaries section with factory→wrap→register pipeline, seam #2 resolution with evidence, and ownership map for tool responsibilities.

- [x] **T04: Write orchestration ownership map and cross-references** `est:30m`
  - Why: The orchestration layer is GSD-authored but uses Pi-owned runtime surfaces — this task produces a clear ownership map and wires cross-references to S02's runtime documentation.
  - Files: `analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`, `analysis/gsd2-harness-decomposition/GSD2_RUNTIME_ARCHITECTURE.md`
  - Do: Add Orchestration Ownership Map section to the document with Pi-owned vs GSD-authored responsibilities. Add cross-references to GSD2_RUNTIME_ARCHITECTURE.md where orchestration calls into runtime. Ensure subject-vs-runner guardrail is explicit.
  - Verify: `grep -q "Orchestration.*Ownership\|Ownership Map" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`, `grep -q "GSD2_RUNTIME_ARCHITECTURE" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`
  - Done when: Document has ownership map, cross-references to runtime architecture, subject-vs-runner guardrail section, and all prior sections have evidence labels.

## Files Likely Touched

- `analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`
