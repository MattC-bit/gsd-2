---
estimated_steps: 5
estimated_files: 3
skills_used:
  - debug-like-expert
---

# T02: Document verification gates and dispatch guards

**Slice:** S03 — Orchestration Layer and Execution Control
**Milestone:** M001

## Description

Document GSD's safety rails: verification gates that run quality checks after execute-task units, and dispatch guards that prevent out-of-order slice execution. These are the enforcement mechanisms that ensure workflow integrity.

## Steps

1. Read `src/resources/extensions/gsd/verification-gate.ts` completely — trace command discovery order, gate execution, failure context formatting, and security sanitization.

2. Read `src/resources/extensions/gsd/auto-verification.ts` — trace verification integration into auto-mode, auto-fix retry logic, and pause/retry outcomes.

3. Read `src/resources/extensions/gsd/dispatch-guard.ts` — trace out-of-order prevention logic, dependency-aware ordering, and cross-milestone blocking.

4. Add to `GSD2_ORCHESTRATION_LAYER.md`:
   - **Verification Gate** — Command discovery order (preference → task-plan → package.json), execution model, auto-fix retry logic, advisory failure handling
   - **Dispatch Guard** — Slice ordering enforcement, dependency-aware ordering, prior slice completion blocking
   - Use evidence tier labels on all claims

5. Add diagrams for verification flow and guard check flow.

## Must-Haves

- [ ] Verification Gate section with discovery order (D003 reference)
- [ ] Auto-fix retry logic documented with max retries and pause behavior
- [ ] Dispatch Guard section with ordering rules
- [ ] Dependency-aware ordering explanation
- [ ] Evidence tier labels on all claims

## Verification

- `grep -q "Verification Gate" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`
- `grep -q "Dispatch Guard" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`
- `grep -q "preference.*task-plan\|discovery.*order" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`
- `grep -c "\[repo code\]" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md` returns >= 5 (cumulative with T01)

## Inputs

- `src/resources/extensions/gsd/verification-gate.ts` — Gate execution and command discovery
- `src/resources/extensions/gsd/auto-verification.ts` — Verification integration and retry logic
- `src/resources/extensions/gsd/dispatch-guard.ts` — Out-of-order prevention

## Expected Output

- `analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md` — Extended with verification gate and dispatch guard sections
