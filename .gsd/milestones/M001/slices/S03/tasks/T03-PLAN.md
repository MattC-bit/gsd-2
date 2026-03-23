---
estimated_steps: 6
estimated_files: 4
skills_used:
  - debug-like-expert
---

# T03: Document tool boundaries and resolve seam #2

**Slice:** S03 — Orchestration Layer and Execution Control
**Milestone:** M001

## Description

Resolve S01's unresolved seam #2 (tool definition boundaries) by tracing how tools flow from Pi-owned factories to GSD's wrapped registrations. This completes the ownership picture for the orchestration layer.

## Steps

1. Read `src/resources/extensions/gsd/bootstrap/dynamic-tools.ts` completely — trace how GSD wraps Pi's `createBashTool`, `createEditTool`, `createReadTool`, `createWriteTool`.

2. Read `src/resources/extensions/gsd/bootstrap/db-tools.ts` — trace GSD-specific tools registered via `pi.registerTool()`.

3. Search Pi packages for tool factory exports:
   - `rg "createBashTool|createEditTool|createReadTool|createWriteTool" packages/`
   - Trace where these factories are defined and what they return.

4. Read `src/cli.ts` sections around tool registration — understand when/where tools are registered during startup.

5. Add to `GSD2_ORCHESTRATION_LAYER.md`:
   - **Tool Registration Boundaries** — Factory→wrap→register pipeline, Pi-owned factories vs GSD-owned wrappers
   - **Seam #2 Resolution** — Explicit resolution with evidence, ownership claim
   - Use evidence tier labels (`[repo code]` for traced code paths)

6. Update `GSD2_SYSTEM_OVERVIEW.md` boundary inventory to reflect resolved seam.

## Must-Haves

- [ ] Tool Registration Boundaries section with factory→wrap→register pipeline
- [ ] Seam #2 resolution with explicit ownership claim (Pi-owned factories, GSD-owned wrappers)
- [ ] Evidence from code traces (file paths, function names)
- [ ] Cross-reference to S01's unresolved seam documentation

## Verification

- `grep -q "Tool.*Registration\|Tool.*Boundar" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`
- `grep -q "seam #2\|Seam #2" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`
- `grep -q "createBashTool\|createEditTool" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`
- `grep -q "RESOLVED\|resolved" analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md`

## Inputs

- `src/resources/extensions/gsd/bootstrap/dynamic-tools.ts` — GSD tool wrapping and registration
- `src/resources/extensions/gsd/bootstrap/db-tools.ts` — GSD-specific tools
- `src/cli.ts` — CLI startup and tool registration timing
- `packages/pi-coding-agent/src/core/sdk.ts` — Pi tool factory exports

## Expected Output

- `analysis/gsd2-harness-decomposition/GSD2_ORCHESTRATION_LAYER.md` — Extended with tool boundaries section and seam #2 resolution
