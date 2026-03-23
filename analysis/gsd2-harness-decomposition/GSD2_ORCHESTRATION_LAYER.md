# GSD-2 Orchestration Layer and Execution Control

This document describes how GSD-2 transforms the Pi runtime substrate into an orchestration system via dispatch, auto mode, verification, guards, prompts, and control surfaces — the control flow heart of the workflow engine.

---

> **[D008]** Throughout this document, references to `.gsd/`, milestones, slices, and workflow state refer to the **analyzed GSD-2 system** unless explicitly noted otherwise. This document is produced *inside* a live GSD run in a fork of GSD-2. The `.gsd/` artifacts producing this analysis (the runner) are separate from the `.gsd/` model being analyzed (the subject). See [EVIDENCE_METHOD.md](./EVIDENCE_METHOD.md#subject-vs-runner-guardrail) for the full guardrail.

---

## Overview

The GSD-2 orchestration layer is responsible for:

1. **State Derivation** — Reading `.gsd/` disk artifacts to determine workflow position
2. **Dispatch Resolution** — Declarative phase→unit mapping via ordered rules
3. **Auto-Mode Loop** — State machine that cycles between units until terminal condition
4. **Verification Gates** — Quality checks that block or retry on failure
5. **Dispatch Guards** — Out-of-order execution prevention
6. **Tool Registration Boundary** — Where Pi-owned tools meet GSD-authored workflow

The orchestration layer is **GSD-authored** but built on **Pi-owned runtime surfaces** (Agent, AgentSession, SessionManager). GSD contributes the workflow semantics; Pi provides the execution substrate.

---

## Auto-Mode State Machine

### Phase Taxonomy

[repo code] The auto-mode state machine is driven by `deriveState()` in `src/resources/extensions/gsd/state.ts`. Phases are derived from disk state, not in-memory transitions:

| Phase | Condition | Next Action |
|-------|-----------|-------------|
| `needs-discussion` | No CONTEXT file, CONTEXT-DRAFT exists | Discuss draft context |
| `pre-planning` | No roadmap, no active slice | Plan milestone |
| `planning` | Active slice, no slice plan or empty tasks | Plan slice |
| `executing` | Active task in active slice | Execute task |
| `summarizing` | All tasks done, slice not complete | Write slice summary |
| `replanning-slice` | Blocker discovered in completed task | Replan slice |
| `validating-milestone` | All slices done, no terminal validation | Run validation |
| `completing-milestone` | Terminal validation, no summary | Write milestone summary |
| `complete` | Summary file exists | Stop (all done) |
| `blocked` | Unmet dependencies or no eligible slice | Stop (needs fix) |

### State Derivation from Disk

[repo code] `deriveState()` in `state.ts:175-520` is the source of truth for workflow position:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ STATE DERIVATION FLOW                                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. Scan .gsd/milestones/ for milestone directories                        │
│     └─ findMilestoneIds() respects queue-order.json                        │
│                                                                             │
│  2. For each milestone:                                                     │
│     ├─ Skip if PARKED file exists                                           │
│     ├─ Skip if SUMMARY file exists (complete)                               │
│     ├─ Parse ROADMAP.md for slice status                                    │
│     └─ Check depends_on in CONTEXT/CONTEXT-DRAFT                            │
│                                                                             │
│  3. Select active milestone:                                                │
│     ├─ First incomplete with satisfied dependencies                         │
│     └─ If all deps unmet → phase: "blocked"                                 │
│                                                                             │
│  4. For active milestone:                                                   │
│     ├─ All slices done + no validation → validating-milestone               │
│     ├─ Terminal validation + no summary → completing-milestone              │
│     └─ Find first incomplete slice with satisfied deps                      │
│                                                                             │
│  5. For active slice:                                                       │
│     ├─ No PLAN file → planning                                              │
│     ├─ Empty tasks array → planning                                         │
│     ├─ All tasks done → summarizing                                         │
│     ├─ blocker_discovered in task summary → replanning-slice                │
│     └─ First incomplete task → executing                                    │
│                                                                             │
│  Output: GSDState { phase, activeMilestone, activeSlice, activeTask, ... }  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Session Lifecycle

[repo code] The `AutoSession` class in `auto/session.ts:43-180` encapsulates all mutable auto-mode state:

| Property | Purpose |
|----------|---------|
| `active` / `paused` | Session running state |
| `stepMode` | Single-step vs continuous mode |
| `currentUnit` | In-flight unit { type, id, startedAt } |
| `completedUnits` | History of finished units |
| `unitDispatchCount` | Per-unit retry counter |
| `currentMilestoneId` | Active milestone tracking |
| `sidecarQueue` | Pending hooks/triage/quick-tasks |
| `pendingVerificationRetry` | Gate failure with retry context |

**Encapsulation invariant:** [repo code] All mutable auto-mode state lives in `AutoSession`. The `auto.ts` module has NO module-level `let` or `var` declarations — enforced by tests in `auto-session-encapsulation.test.ts`.

### Main Loop Structure

[repo code] The auto-mode loop in `auto/loop.ts:30-130` iterates through five phases:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ AUTO-MODE MAIN LOOP (autoLoop)                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  while (s.active) {                                                         │
│                                                                             │
│    ┌──────────────────────────────────────────────────────────────────────┐ │
│    │ Phase 1: PRE-DISPATCH (runPreDispatch)                               │ │
│    │ ├─ Resource version guard (stale worktree escape)                    │ │
│    │ ├─ Pre-dispatch health gate (doctor --fix)                           │ │
│    │ ├─ State derivation (deriveState)                                    │ │
│    │ ├─ Milestone transition handling                                     │ │
│    │ └─ Terminal condition check (complete/blocked/no-milestone)          │ │
│    └──────────────────────────────────────────────────────────────────────┘ │
│                              ↓                                              │
│    ┌──────────────────────────────────────────────────────────────────────┐ │
│    │ Phase 2: GUARDS (runGuards)                                          │ │
│    │ ├─ Budget ceiling check (pause/halt/warn)                            │ │
│    │ ├─ Context window threshold                                          │ │
│    │ └─ Secrets re-collection gate                                        │ │
│    └──────────────────────────────────────────────────────────────────────┘ │
│                              ↓                                              │
│    ┌──────────────────────────────────────────────────────────────────────┐ │
│    │ Phase 3: DISPATCH (runDispatch)                                      │ │
│    │ ├─ resolveDispatch() — rule evaluation                               │ │
│    │ ├─ Stuck detection (sliding window)                                  │ │
│    │ ├─ Pre-dispatch hooks                                                │ │
│    │ └─ Prior slice completion guard                                      │ │
│    └──────────────────────────────────────────────────────────────────────┘ │
│                              ↓                                              │
│    ┌──────────────────────────────────────────────────────────────────────┐ │
│    │ Phase 4: UNIT EXECUTION (runUnitPhase)                               │ │
│    │ ├─ Worktree health check                                             │ │
│    │ ├─ Model selection with tier escalation                              │ │
│    │ ├─ Prompt construction (retry context, observability)                │ │
│    │ ├─ runUnit() → newSession() → sendMessage()                          │ │
│    │ └─ Artifact verification + closeout                                   │ │
│    └──────────────────────────────────────────────────────────────────────┘ │
│                              ↓                                              │
│    ┌──────────────────────────────────────────────────────────────────────┐ │
│    │ Phase 5: FINALIZE (runFinalize)                                      │ │
│    │ ├─ Post-unit pre-verification (commit, doctor, state rebuild)        │ │
│    │ ├─ Verification gate execution                                       │ │
│    │ └─ Post-verification (hooks, triage, quick-tasks)                    │ │
│    └──────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  }  // end while                                                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Dispatch Table

### Rule Evaluation Order

[repo code] The dispatch table in `auto-dispatch.ts:83-430` defines 20 declarative rules, evaluated in order (first match wins):

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ DISPATCH RULE EVALUATION                                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  for (const rule of DISPATCH_RULES) {                                       │
│    const result = await rule.match(context);                                │
│    if (result) return result;  // First match wins                          │
│  }                                                                          │
│  return { action: "stop", reason: "Unhandled phase" };                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Dispatch Action Types

| Action | Effect |
|--------|--------|
| `dispatch` | Send prompt to agent with unitType/unitId |
| `stop` | Exit auto-mode with reason and level |
| `skip` | No action, re-derive state on next iteration |

### Dispatch Rule Table

[repo code] The following rules are defined in `DISPATCH_RULES` array:

| # | Rule Name | Matching Condition | Action |
|---|-----------|-------------------|--------|
| 1 | `rewrite-docs (override gate)` | Pending overrides exist | dispatch `rewrite-docs` |
| 2 | `summarizing → complete-slice` | phase === "summarizing" | dispatch `complete-slice` |
| 3 | `run-uat (post-completion)` | UAT file exists, needs execution | dispatch `run-uat` |
| 4 | `uat-verdict-gate` | Non-PASS UAT verdict found | stop (warning) |
| 5 | `reassess-roadmap (post-completion)` | Slice done, reassess flag set | dispatch `reassess-roadmap` |
| 6 | `needs-discussion → discuss-milestone` | phase === "needs-discussion" | dispatch `discuss-milestone` |
| 7 | `pre-planning (no context) → discuss-milestone` | phase === "pre-planning", no CONTEXT | dispatch `discuss-milestone` |
| 8 | `pre-planning (no research) → research-milestone` | phase === "pre-planning", no RESEARCH | dispatch `research-milestone` |
| 9 | `pre-planning (has research) → plan-milestone` | phase === "pre-planning" | dispatch `plan-milestone` |
| 10 | `planning (no research, not S01) → research-slice` | phase === "planning", no slice RESEARCH | dispatch `research-slice` |
| 11 | `planning → plan-slice` | phase === "planning" | dispatch `plan-slice` |
| 12 | `replanning-slice → replan-slice` | phase === "replanning-slice" | dispatch `replan-slice` |
| 13 | `executing → reactive-execute (parallel dispatch)` | Reactive enabled, >1 ready task | dispatch `reactive-execute` |
| 14 | `executing → execute-task (recover missing task plan)` | Task PLAN file missing | dispatch `plan-slice` |
| 15 | `executing → execute-task` | phase === "executing", active task | dispatch `execute-task` |
| 16 | `validating-milestone → validate-milestone` | phase === "validating-milestone" | dispatch `validate-milestone` |
| 17 | `completing-milestone → complete-milestone` | phase === "completing-milestone" | dispatch `complete-milestone` |
| 18 | `complete → stop` | phase === "complete" | stop (info) |

### Rule-to-Phase Mapping

[repo code] The dispatch table maps workflow phases to unit types:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ PHASE → UNIT TYPE MAPPING                                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  needs-discussion    ────────────────────────────► discuss-milestone        │
│                                                                             │
│  pre-planning        ──┬─ no context ─────────────► discuss-milestone        │
│                      ├─ no research ────────────► research-milestone        │
│                      └─ has research ───────────► plan-milestone            │
│                                                                             │
│  planning            ──┬─ no research (not S01) ─► research-slice           │
│                      └─ default ────────────────► plan-slice                │
│                                                                             │
│  replanning-slice    ────────────────────────────► replan-slice             │
│                                                                             │
│  executing           ──┬─ reactive enabled ─────► reactive-execute          │
│                      └─ sequential ────────────► execute-task               │
│                                                                             │
│  summarizing         ────────────────────────────► complete-slice           │
│                                                                             │
│  validating-milestone ───────────────────────────► validate-milestone       │
│                                                                             │
│  completing-milestone ───────────────────────────► complete-milestone       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Verification Gate

### Overview

[repo code] Verification gates are the quality enforcement layer in GSD's orchestration. They run after `execute-task` units in Phase 5 (finalize), executing commands discovered from multiple sources. The gate can pass (continue), fail with retry (auto-fix), or fail with pause (human intervention needed).

### Command Discovery Order (D003)

[repo code] The discovery order follows **D003** — first non-empty source wins, implemented in `verification-gate.ts:discoverCommands()`:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ COMMAND DISCOVERY ORDER (D003)                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. Preference commands                                                     │
│     └─ preferences.verification_commands array (user-configured)           │
│     └─ Source: "preference"                                                 │
│                                                                             │
│  2. Task plan verify field                                                  │
│     └─ T01-PLAN.md → frontmatter: verify: "npm test && npm run lint"       │
│     └─ Split on "&&", sanitize each command                                 │
│     └─ Source: "task-plan"                                                  │
│                                                                             │
│  3. package.json scripts                                                    │
│     └─ Probe keys in order: typecheck, lint, test                          │
│     └─ Run as: npm run <key>                                               │
│     └─ Source: "package-json"                                               │
│                                                                             │
│  4. No commands found                                                       │
│     └─ Gate passes trivially (no checks configured)                         │
│     └─ Source: "none"                                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Gate Execution Flow

[repo code] The full gate execution is implemented in `verification-gate.ts:runVerificationGate()`:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ VERIFICATION GATE EXECUTION                                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  runVerificationGate({ basePath, unitId, cwd, preferenceCommands, ... })    │
│                                                                             │
│  1. DISCOVER COMMANDS                                                       │
│     └─ discoverCommands() → { commands[], source }                          │
│     └─ Empty commands → return { passed: true, checks: [] }                 │
│                                                                             │
│  2. EXECUTE EACH COMMAND (sequential, all run regardless of pass/fail)      │
│     ├─ spawnSync(shellBin, shellArgs, { cwd, stdio: "pipe", timeout })      │
│     ├─ Truncate stdout/stderr to 10 KB each                                 │
│     ├─ Capture exitCode, durationMs                                         │
│     └─ Build VerificationCheck { command, exitCode, stdout, stderr, ... }   │
│                                                                             │
│  3. CAPTURE RUNTIME ERRORS (async scan)                                     │
│     ├─ bg-shell processes: crashed status, non-zero exit, fatal signals     │
│     ├─ Browser console: unhandled rejections, general errors                │
│     └─ Severity: "crash" (blocking) vs "error"/"warning" (non-blocking)     │
│                                                                             │
│  4. RUN DEPENDENCY AUDIT (if package.json changed)                          │
│     ├─ git diff --name-only HEAD → detect dependency file changes           │
│     ├─ npm audit --audit-level=moderate --json                              │
│     └─ Parse vulnerabilities → AuditWarning[]                               │
│                                                                             │
│  5. RETURN VERIFICATION RESULT                                              │
│     └─ { passed, checks[], discoverySource, runtimeErrors[], auditWarnings }│
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Security Sanitization

[repo code] Task plan verify commands are untrusted input. GSD applies sanitization in `verification-gate.ts:sanitizeCommand()`:

| Check | Behavior |
|-------|----------|
| Shell injection pattern | Reject commands containing `;`, `|`, `` ` ``, `$(` |
| Prose detection | `isLikelyCommand()` rejects English sentences |
| Known command prefix | Allow if first token is in `KNOWN_COMMAND_PREFIXES` set |
| Path-like first token | Allow if starts with `/`, `./`, or `../` |

The `KNOWN_COMMAND_PREFIXES` set includes: npm, npx, yarn, pnpm, bun, deno, node, tsc, sh, bash, make, cargo, go, python, git, eslint, prettier, vitest, jest, pytest, etc.

### Runtime Error Capture

[repo code] `captureRuntimeErrors()` scans two sources for blocking conditions:

**bg-shell processes:**
| Condition | Severity | Blocking |
|-----------|----------|----------|
| `status === "crashed"` | crash | Yes |
| `!alive && exitCode !== 0` | crash | Yes |
| Signal SIGABRT/SIGSEGV/SIGBUS | crash | Yes |
| `alive && recentErrors.length > 0` | error | No |

**Browser console:**
| Condition | Severity | Blocking |
|-----------|----------|----------|
| Error with "unhandled" in text | crash | Yes |
| General `console.error` | error | No |
| Warning with "deprecated" in text | warning | No |

### Auto-Fix Retry Logic

[repo code] The retry mechanism in `auto-verification.ts:runPostUnitVerification()`:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ AUTO-FIX RETRY FLOW                                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Preferences:                                                               │
│    verification_auto_fix: true/false (default: true)                        │
│    verification_max_retries: number (default: 2)                            │
│                                                                             │
│  When gate fails:                                                           │
│                                                                             │
│    1. Check if advisory failure (skip retry)                                │
│       ├─ discoverySource === "package-json" → advisory (no retry)           │
│       └─ isInfraVerificationFailure(stderr) → advisory (no retry)           │
│          └─ ENOENT, ENOTFOUND, ETIMEDOUT, ECONNRESET, etc.                  │
│                                                                             │
│    2. If NOT advisory and attempt + 1 <= maxRetries:                        │
│       ├─ Set s.pendingVerificationRetry = { unitId, failureContext, attempt}│
│       ├─ Increment verificationRetryCount for unit                          │
│       ├─ Notify: "Verification failed — auto-fix attempt N/M"               │
│       └─ Return "retry" → loop re-iterates with retry context in prompt     │
│                                                                             │
│    3. If retries exhausted:                                                 │
│       ├─ Clear pendingVerificationRetry                                     │
│       ├─ Notify: "Verification gate FAILED after N retries — pausing"       │
│       ├─ Call pauseAuto(ctx, pi)                                            │
│       └─ Return "pause" → human intervention required                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Failure Context Formatting

[repo code] Failed checks are formatted into prompt-injectable context via `formatFailureContext()`:

```markdown
## Verification Failures

### ❌ `npm test` (exit code 1)
```stderr
FAIL src/example.test.ts
  ● test suite failed
  ...
```

Each check is truncated to 2,000 chars stderr, total output capped at 10,000 chars.

### Advisory Failure Handling

[repo code] Some failures are treated as advisory rather than blocking:

1. **Auto-discovered package.json checks** — May not be configured correctly for the project
2. **Infrastructure failures** — Missing commands, network issues, environment problems

Advisory failures log a warning and continue without retry, avoiding false-positive blockers in unfamiliar environments.

### Evidence Persistence

[repo code] Verification results are written to JSON for audit:

```
.gsd/milestones/M001/slices/S01/tasks/T01-VERIFICATION.json
```

Contains: checks, discoverySource, runtimeErrors, auditWarnings, attempt number, maxRetries.

---

## Dispatch Guard

### Purpose

[repo code] The dispatch guard prevents out-of-order slice execution, ensuring workflow integrity. It blocks dispatching to a slice if prior slices (or declared dependencies) are incomplete.

### Out-of-Order Prevention Logic

[repo code] Implemented in `dispatch-guard.ts:getPriorSliceCompletionBlocker()`:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ DISPATCH GUARD CHECK                                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  getPriorSliceCompletionBlocker(base, mainBranch, unitType, unitId)         │
│                                                                             │
│  1. FILTER BY UNIT TYPE                                                     │
│     └─ Only applies to: research-slice, plan-slice, replan-slice,           │
│                        execute-task, complete-slice                         │
│     └─ Other unit types return null (clear to dispatch)                     │
│                                                                             │
│  2. PARSE TARGET                                                            │
│     └─ unitId format: "M001/S01/T01" or "M001/S01"                          │
│     └─ Extract targetMid, targetSid                                         │
│                                                                             │
│  3. ITERATE MILESTONES IN QUEUE ORDER                                       │
│     └─ findMilestoneIds(base) respects queue-order.json                     │
│     └─ Only check milestones BEFORE target in queue order                   │
│                                                                             │
│  4. FOR EACH MILESTONE:                                                     │
│     ├─ Skip if PARKED file exists                                           │
│     ├─ Skip if SUMMARY file exists (complete)                               │
│     ├─ Read ROADMAP.md from disk (working tree)                             │
│     └─ Parse slice status via parseRoadmapSlices()                          │
│                                                                             │
│  5. CHECK PRIOR MILESTONES                                                  │
│     └─ If any slice incomplete → return blocker message                     │
│                                                                             │
│  6. CHECK TARGET MILESTONE                                                  │
│     ├─ Find targetSlice by id                                               │
│     ├─ If depends_on declared:                                              │
│     │   └─ Check only declared dependencies (not positional)                │
│     └─ Else:                                                                │
│         └─ Check all positionally earlier slices                            │
│                                                                             │
│  7. RETURN                                                                   │
│     ├─ null → clear to dispatch                                             │
│     └─ blocker message → dispatch blocked                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Working Tree vs Git Branch Reading

[repo code] Critical implementation detail from `dispatch-guard.ts:readRoadmapFromDisk()`:

> Prior implementation used `git show <branch>:<path>` which read committed state on a specific branch. This caused false-positive blockers when work was committed on a milestone/worktree branch but the integration branch (main) hadn't been updated yet — the guard would see prior slices as incomplete on main even though they were done in the working tree (#530).

**Resolution:** Reading from disk always reflects the latest state, regardless of which branch is checked out or whether changes have been committed.

### Dependency-Aware Ordering

[repo code] The guard supports `depends_on` declarations in roadmap slices:

```markdown
- [ ] S05: Integration Tests
  - depends_on: [S06]  # S05 runs after S06 despite positional order
```

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ DEPENDENCY-AWARE ORDERING                                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  When targetSlice.depends.length > 0:                                       │
│    ├─ Build sliceMap = { id → slice }                                       │
│    ├─ For each depId in targetSlice.depends:                                │
│    │   ├─ Find dep slice in sliceMap                                        │
│    │   └─ If dep exists && !dep.done → BLOCK                                │
│    └─ Cross-milestone dependencies ignored (handled elsewhere)              │
│                                                                             │
│  When targetSlice.depends is empty:                                         │
│    └─ Fall back to positional ordering                                      │
│    └─ Check all slices with index < targetIndex                             │
│                                                                             │
│  This prevents deadlocks when:                                              │
│    └─ S05 depends_on S06                                                    │
│    └─ Positional order would require S04 done before S05                    │
│    └─ But S05 only waits for S06, not S04                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Cross-Milestone Blocking

[repo code] The guard enforces milestone ordering via queue order:

1. `findMilestoneIds(base)` returns milestones in queue order (respects `queue-order.json`)
2. For target milestone M002, only M001 is checked for completion
3. If M001 has any incomplete slice → M002 dispatch blocked
4. PARKED and SUMMARY milestones are skipped

### Blocker Message Format

[repo code] When blocking, returns a human-readable message:

```
Cannot dispatch execute-task M001/S03/T02: earlier slice M001/S02 is not complete.
Cannot dispatch execute-task M002/S01/T01: dependency slice M002/S02 is not complete.
```

These messages appear in the dispatch log and inform the user why dispatch was blocked.

---

## Control Surfaces

### Start/Stop/Pause Operations

[repo code] `auto.ts` exports these control functions:

| Function | Effect |
|----------|--------|
| `startAuto(ctx, pi, base, verbose, options)` | Begin auto-mode loop |
| `stopAuto(ctx, pi, reason?)` | Exit auto-mode, clean up all state |
| `pauseAuto(ctx, pi)` | Pause without state destruction |
| `isAutoActive()` | Check if running |
| `isAutoPaused()` | Check if paused |
| `isStepMode()` | Check if single-step mode |

### Crash Recovery Integration

[repo code] Auto-mode integrates with GSD's crash recovery system:

1. **Lock file**: `.gsd/auto.lock` with PID, unitType, unitId, timestamp
2. **Remote stop**: `stopAutoRemote(projectRoot)` sends SIGTERM to running session
3. **Session forensics**: `synthesizeCrashRecovery()` extracts tool calls from paused session
4. **Resume recovery**: Paused sessions can be resumed with context restoration

---

## Tool Registration Boundary

### Pi-Owned Tool Factories

[repo code] The Pi runtime exports tool factories from `packages/pi-coding-agent/src/core/tools/`:

| Factory | File | Tool Name | Purpose |
|---------|------|-----------|---------|
| `createBashTool` | `bash.ts:175-320` | `bash` | Execute shell commands with streaming output |
| `createReadTool` | `read.ts:50-85` | `read` | Read file contents with image detection |
| `createEditTool` | `edit.ts:45-95` | `edit` | Surgical file edits via oldText/newText |
| `createWriteTool` | `write.ts:40-80` | `write` | Create/overwrite files atomically |

[repo code] Each factory returns an `AgentTool` object with:
- `name`: Tool identifier (e.g., `"bash"`)
- `label`: Human-readable label
- `description`: Prompt-visible description
- `parameters`: TypeBox schema for input validation
- `execute`: Async function implementing tool logic

[repo code] Factories are exported via `packages/pi-coding-agent/src/core/tools/index.ts` and re-exported from `packages/pi-coding-agent/src/index.ts` for external consumption.

### GSD Tool Wrapping Pattern

[repo code] GSD wraps Pi-owned factories in `src/resources/extensions/gsd/bootstrap/dynamic-tools.ts:19-68`:

```typescript
// dynamic-tools.ts:4 — Import from Pi SDK
import { createBashTool, createEditTool, createReadTool, createWriteTool } from "@gsd/pi-coding-agent";

// dynamic-tools.ts:20-34 — Wrap bash with default timeout
export function registerDynamicTools(pi: ExtensionAPI): void {
  const baseBash = createBashTool(process.cwd(), {
    spawnHook: (ctx) => ({ ...ctx, cwd: process.cwd() }),
  });
  const dynamicBash = {
    ...baseBash,
    execute: async (toolCallId, params, signal, onUpdate, ctx) => {
      const paramsWithTimeout = {
        ...params,
        timeout: params.timeout ?? DEFAULT_BASH_TIMEOUT_SECS,
      };
      return baseBash.execute(toolCallId, paramsWithTimeout, signal, onUpdate, ctx);
    },
  };
  pi.registerTool(dynamicBash);
```

[repo code] The wrapping pattern:
1. **Create base tool** — Call factory with `process.cwd()` as working directory
2. **Spread base properties** — Copy `name`, `label`, `description`, `parameters` from base
3. **Override execute** — Wrap execute function with GSD-specific behavior
4. **Register via API** — Call `pi.registerTool(wrappedTool)`

[repo code] GSD adds these behaviors via wrapping:

| Tool | GSD Behavior Added | Source |
|------|-------------------|--------|
| `bash` | Default timeout (120s) if not specified | `dynamic-tools.ts:26-29` |
| `read` | Fresh tool per-execute for cwd resolution | `dynamic-tools.ts:48-51` |
| `edit` | Fresh tool per-execute for cwd resolution | `dynamic-tools.ts:58-61` |
| `write` | Fresh tool per-execute for cwd resolution | `dynamic-tools.ts:38-41` |

### GSD-Owned Tools

[repo code] GSD defines entirely GSD-specific tools in `src/resources/extensions/gsd/bootstrap/db-tools.ts`:

| Tool | Purpose | File Location |
|------|---------|---------------|
| `gsd_save_decision` | Record architectural decision to DB, regenerate DECISIONS.md | `db-tools.ts:11-54` |
| `gsd_update_requirement` | Update requirement status/notes in DB, regenerate REQUIREMENTS.md | `db-tools.ts:56-108` |
| `gsd_save_summary` | Save artifact (SUMMARY/RESEARCH/CONTEXT/ASSESSMENT) to disk | `db-tools.ts:110-170` |
| `gsd_generate_milestone_id` | Generate next milestone ID respecting preferences | `db-tools.ts:172-205` |

[repo code] These tools are registered directly without Pi factory involvement:

```typescript
// db-tools.ts:10
export function registerDbTools(pi: ExtensionAPI): void {
  pi.registerTool({
    name: "gsd_save_decision",
    label: "Save Decision",
    description: "Record a project decision to the GSD database...",
    parameters: Type.Object({ ... }),
    async execute(_toolCallId, params, ...) {
      // GSD-specific implementation using gsd-db.js
    },
  });
```

### Registration Pipeline

[repo code] Tool registration occurs during extension initialization in `register-extension.ts:31-35`:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ TOOL REGISTRATION PIPELINE                                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  registerGsdExtension(pi: ExtensionAPI)                                     │
│                                                                             │
│  1. Register GSD commands                                                   │
│     ├─ registerGSDCommand(pi)      — /gsd main command                      │
│     ├─ registerWorktreeCommand(pi) — /worktree                               │
│     └─ registerExitCommand(pi)     — /exit, /kill                           │
│                                                                             │
│  2. Register wrapped Pi tools                                               │
│     └─ registerDynamicTools(pi)    — bash, read, edit, write                │
│         ├─ Import factories from @gsd/pi-coding-agent                       │
│         ├─ Create base tools with process.cwd()                             │
│         ├─ Wrap execute with GSD-specific behaviors                         │
│         └─ pi.registerTool(wrappedTool) for each                            │
│                                                                             │
│  3. Register GSD-owned tools                                                │
│     └─ registerDbTools(pi)         — gsd_save_decision, etc.                │
│         ├─ Define tool inline with TypeBox schema                           │
│         └─ pi.registerTool(gsdTool) for each                                │
│                                                                             │
│  4. Register extension hooks                                                │
│     └─ registerHooks(pi), registerShortcuts(pi)                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Seam #2 Resolution: Tool Definition Boundaries

**Original issue** (from S01): "Pi provides tool definitions in `packages/pi-coding-agent/src/core/tools/`, but GSD registers dynamic tools and database tools via `registerDynamicTools()` and `registerDbTools()`. The relationship between Pi-provided tools and GSD-added tools is implicit in the code rather than explicitly documented."

**Resolution:**

[repo code] **Ownership is now traced:**

| Component | Owner | Evidence |
|-----------|-------|----------|
| Tool factory definitions | Pi | `packages/pi-coding-agent/src/core/tools/{bash,read,edit,write}.ts` |
| Factory exports | Pi | `packages/pi-coding-agent/src/core/tools/index.ts:1-20` |
| Factory re-exports | Pi | `packages/pi-coding-agent/src/index.ts` |
| Tool wrapping logic | GSD | `src/resources/extensions/gsd/bootstrap/dynamic-tools.ts:19-68` |
| GSD-owned tools | GSD | `src/resources/extensions/gsd/bootstrap/db-tools.ts:10-205` |
| Registration API | Pi | `ExtensionAPI.registerTool()` from `@gsd/pi-coding-agent` |
| Registration call sites | GSD | `register-extension.ts:31-35` |

**Ownership claim:**
- **Pi owns**: Tool factories, `AgentTool` interface, `registerTool()` API, execution infrastructure
- **GSD owns**: Wrapping behavior, GSD-specific tool implementations, registration orchestration
- **Boundary**: The `ExtensionAPI.registerTool()` call is the handoff point — GSD passes wrapped or GSD-defined tools to Pi's registration API

**RESOLVED**: Seam #2 (Tool Definition Boundaries) is now traced with `[repo code]` evidence. The relationship is: Pi provides factory functions that return `AgentTool` objects; GSD imports these factories, optionally wraps the `execute` function, and registers the result via Pi's `registerTool()` API. GSD also defines entirely GSD-owned tools using the same `AgentTool` structure and registers them via the same API.

---

## Orchestration Ownership Map

[repo code] The orchestration layer sits atop the Pi runtime substrate. The ownership boundary determines who is responsible for what behavior and where to trace issues.

### Pi-Owned Components (Runtime Substrate)

| Component | Location | Responsibility |
|-----------|----------|----------------|
| Agent | `packages/pi-agent-core/src/agent.ts` | LLM interaction, message loop |
| AgentSession | `packages/pi-coding-agent/src/core/agent-session.ts` | Session coordination, retry, compaction |
| SessionManager | `packages/pi-coding-agent/src/core/session-manager.ts` | Persistence, tree navigation |
| ExtensionRunner | `packages/pi-coding-agent/src/core/extensions/runner.ts` | Event emission, hook dispatch |
| Tool Factories | `packages/pi-agent-core/src/tools/` | Bash, Read, Edit, Write, LSP |

### GSD-Authored Components (Orchestration Layer)

| Component | Location | Responsibility |
|-----------|----------|----------------|
| AutoSession | `src/resources/extensions/gsd/auto/session.ts` | Mutable auto-mode state |
| deriveState | `src/resources/extensions/gsd/state.ts` | Disk → phase derivation |
| DISPATCH_RULES | `src/resources/extensions/gsd/auto-dispatch.ts` | Phase → unit mapping |
| autoLoop | `src/resources/extensions/gsd/auto/loop.ts` | Main iteration cycle |
| Dispatch Guard | `src/resources/extensions/gsd/dispatch-guard.ts` | Out-of-order prevention |
| Verification Gate | `src/resources/extensions/gsd/auto-verification.ts` | Quality checks |

### Cross-References to Runtime Architecture

This document describes the **GSD-authored orchestration layer** that runs on top of the **Pi-owned runtime substrate**. For runtime-level details, see:

- **[GSD2_RUNTIME_ARCHITECTURE.md](./GSD2_RUNTIME_ARCHITECTURE.md)** — Session lifecycle, event flow, persistence model
- **Runtime Assembly** — How Agent, AgentSession, and ExtensionRunner are wired
- **Event Propagation** — Agent → AgentSession → ExtensionRunner chain

---

## Subject-vs-Runner Guardrail for Orchestration

[repo code] A critical distinction when analyzing GSD's orchestration layer: **the orchestration system being analyzed is NOT the same as the orchestration system running this analysis**.

### The Distinction

| Aspect | Subject (Analyzed GSD-2) | Runner (This Analysis) |
|--------|--------------------------|------------------------|
| `.gsd/` directory | The project being documented | The metadata producing this document |
| AutoSession state | Analyzed session's auto-mode tracking | Current session's auto-mode tracking |
| Dispatch decisions | Historical dispatches in subject | Live dispatch producing this content |
| Verification gates | Subject's configured checks | This run's verification flow |

### Specific Guardrails for Orchestration Documentation

[repo code] When reading orchestration code, apply these filters:

1. **AutoSession is the runner's auto-mode tracking**
   - `auto/session.ts` manages the *current* analysis session's auto-mode state
   - Do NOT interpret it as the analyzed system's session tracking
   - Evidence: `[repo code]` labels reference the analyzed repo, not runner state

2. **deriveState() reads runner's `.gsd/`, not subject's**
   - The phase derivation logic is documented, not a specific phase result
   - Phase values in examples are illustrative, not actual state

3. **Dispatch rules are the mechanism, not the dispatch**
   - DISPATCH_RULES defines how dispatch works
   - Any specific rule match in this document is self-referential

4. **Verification gate implementation vs. this verification**
   - The gate's command discovery and execution is the documented mechanism
   - The verification running *now* is the runner verifying this document

### How This Document Maintains the Guardrail

[repo code] All code references use `[repo code]` to indicate traced evidence from the analyzed repository. The document describes:

- **Mechanisms**: How dispatch works (rule evaluation order, action types)
- **Algorithms**: How state derivation works (disk → phase mapping)
- **Patterns**: How verification gates work (discovery, execution, retry)

It does NOT describe:

- **Specific state**: What phase the analyzed system was in
- **Specific dispatches**: Which units ran in the analyzed system
- **Specific verification results**: What checks passed/failed in the analyzed system

This distinction is essential for future readers: the orchestration layer documentation describes the *engine*, not any particular *execution* of that engine.

---

## Appendix: File Reference

| File | Purpose |
|------|---------|
| `src/resources/extensions/gsd/auto-dispatch.ts` | Dispatch rule definitions and resolver |
| `src/resources/extensions/gsd/auto.ts` | Main auto-mode entry point and control |
| `src/resources/extensions/gsd/auto/session.ts` | AutoSession class with mutable state |
| `src/resources/extensions/gsd/auto/loop.ts` | Main iteration loop (5 phases) |
| `src/resources/extensions/gsd/auto/phases.ts` | Phase implementations |
| `src/resources/extensions/gsd/state.ts` | State derivation from disk |
| `src/resources/extensions/gsd/dispatch-guard.ts` | Out-of-order prevention |
| `src/resources/extensions/gsd/auto-verification.ts` | Verification gate execution |
| `src/resources/extensions/gsd/bootstrap/dynamic-tools.ts` | GSD tool registration |
