---
estimated_steps: 6
estimated_files: 4
skills_used:
  - debug-like-expert
---

# T01: Document auto-mode state machine and dispatch table

**Slice:** S03 — Orchestration Layer and Execution Control
**Milestone:** M001

## Description

Document the core of GSD's orchestration: the auto-mode state machine and dispatch table. This is the control flow heart of the system — it reads disk state, determines the next unit of work, and drives the agent through workflow phases.

## Steps

1. Read `src/resources/extensions/gsd/auto-dispatch.ts` completely — trace all dispatch rules, their evaluation order, and what actions they produce.

2. Read `src/resources/extensions/gsd/auto.ts` sections related to the main loop (handleAgentEnd, startAuto, stopAuto) — understand how dispatch actions feed back into the state machine.

3. Read `src/resources/extensions/gsd/auto/session.ts` — trace AutoSession class properties and how they track dispatch state.

4. Read `src/resources/extensions/gsd/state.ts` sections for `deriveState()` — understand how disk state maps to GSDState phases.

5. Write `GSD2_ORCHESTRATION_LAYER.md` with sections:
   - **Auto-Mode State Machine** — Phase state machine diagram, state derivation from disk, session lifecycle
   - **Dispatch Table** — Rule evaluation order, rule-to-phase mapping, action types (dispatch/stop/skip)
   - Use evidence tier labels (`[repo code]`, `[repo doc]`, `[inference]`) on all claims

6. Add dispatch rule table with rule names, matching conditions, and produced actions.

## Must-Haves

- [ ] Auto-Mode State Machine section with phase diagram and transitions
- [ ] Dispatch Table section with rule evaluation explanation
- [ ] Dispatch rule table with at least 10 rules documented
- [ ] Evidence tier labels on all claims
- [ ] Subject-vs-runner guardrail mentioned (will be expanded in T04)

## Verification

- `grep -q "Auto-Mode State Machine" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`
- `grep -q "Dispatch Table" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`
- `grep -c "\[repo code\]" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md` returns >= 3
- `grep -q "needs-discussion\|pre-planning\|planning\|executing" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`

## Inputs

- `src/resources/extensions/gsd/auto-dispatch.ts` — Dispatch rule definitions and resolver
- `src/resources/extensions/gsd/auto.ts` — Main auto-mode loop and session management
- `src/resources/extensions/gsd/auto/session.ts` — AutoSession class with dispatch state
- `src/resources/extensions/gsd/state.ts` — State derivation from disk

## Expected Output

- `analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md` — New document with auto-mode state machine and dispatch table sections
