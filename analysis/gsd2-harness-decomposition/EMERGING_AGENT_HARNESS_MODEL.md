# Emerging Agent Harness Model: Rebuild/Wrap/Delegate Synthesis

This document synthesizes findings from all prior analysis documents into a final actionable guide for Atlas harness builders. It applies the rebuild/wrap/delegate taxonomy (K016) to each of the six architectural layers identified in the shared-primitives matrix (K017), drawing on GSD-2's runtime architecture, orchestration layer, context engineering model, isolation model, and comparative analysis.

---

> **[D006]** This document is an Atlas synthesis artifact, distinct from the comparative structural analysis in `GSD2_COMPARATIVE_ANALYSIS.md`. Comparison extracts shared primitives and design bets; this document makes build decisions. Both are required because the decision layer depends on understanding the comparison, but mixing them conflates "what GSD-2 does" with "what Atlas should do."

---

## Subject-vs-Runner Guardrail

This synthesis analyzes GSD-2 as a **subject system**. It is produced inside a live GSD execution in a fork of GSD-2. References to `.gsd/`, milestones, slices, workflow phases, and dispatch rules throughout this document refer to the **analyzed GSD-2 system**, not to the runner executing this task. This subject-vs-runner distinction is maintained throughout per D007.

| Concept | Meaning | Where to find it |
|---------|---------|-----------------|
| **Subject** | The GSD-2 system being analyzed | Analyzed `.gsd/` model, GSD source code |
| **Runner** | The live GSD execution producing this pack | Current reverse-engineering run's `.gsd/` artifacts |
| **Atlas Harness** | The hypothetical target system consuming this analysis | Decision guidance in this document |

---

## Emerging Agent Harness Model

Cross-system comparison of GSD-2, Claude Code, Codex CLI, and ACP reveals a converging model for production-grade agent harnesses. All four systems implement an agent loop with tools — the universal primitive — but diverge on state location, execution control, and isolation guarantees. [inference]

### The Convergent Skeleton

Every system studied shares this minimal structure [inference]:

```
┌────────────────────────────────────────────────────────┐
│ COMMON AGENT HARNESS SKELETON                          │
├────────────────────────────────────────────────────────┤
│                                                        │
│  1. MODEL LAYER                                        │
│     ├─ Provider credential management                  │
│     ├─ API transport (streaming or request/response)   │
│     └─ Context window management + compaction          │
│                                                        │
│  2. AGENT LAYER                                        │
│     ├─ Message loop (send prompt → receive response)   │
│     ├─ Tool execution (bash, read, write, edit)        │
│     └─ Event propagation to extensions/hooks           │
│                                                        │
│  3. HARNESS LAYER                                      │
│     ├─ Session lifecycle (create, resume, persist)     │
│     ├─ JSONL or equivalent conversation store          │
│     └─ Error recovery (retry, compaction, restart)     │
│                                                        │
│  (Systems diverge here)                                │
│                                                        │
│  4. ORCHESTRATION LAYER (GSD-2 only)                   │
│     ├─ Phase derivation from persistent artifacts      │
│     ├─ Declarative dispatch rules (phase → unit)       │
│     └─ Verification gates with auto-retry              │
│                                                        │
│  5. WORKFLOW/DOC-STATE LAYER (GSD-2 only)              │
│     ├─ Disk-based state as source of truth             │
│     ├─ Milestone → Slice → Task hierarchy              │
│     └─ Human-readable markdown audit trail             │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### What GSD-2 Uniquely Contributes

The comparative analysis shows four capabilities that GSD-2 owns and no other system matches [repo code]:

1. **Disk-derived phase state** — `deriveState()` reads `.gsd/` files on every dispatch cycle. Workflow position is never tracked in memory; crashing and resuming is free because disk is the source of truth.

2. **Declarative dispatch rules** — `DISPATCH_RULES` in `auto-dispatch.ts` maps `(phase, context)` → unit type with first-match-wins semantics. Adding new dispatch behavior is a data operation, not a code change.

3. **Hierarchical work decomposition** — Milestone → Slice → Task provides explicit decomposition with dependency tracking, parallel worker isolation, and per-task verification contracts.

4. **Worktree isolation as first-class primitive** — `isolationMode: "worktree"` creates `.gsd/worktrees/<MID>/` with a dedicated git branch per milestone. [repo code] Parallel workers cannot interfere because each operates in a separate working tree.

These four capabilities are what an Atlas Harness must decide how to address: adopt them wholesale, rebuild lighter equivalents, or wrap around them.

---

## Rebuild/Wrap/Delegate Synthesis by Layer

The following tables apply the taxonomy from K016 to each of the six architectural layers.

**Taxonomy definitions:**
- **Rebuild** — Capability is architecturally incompatible or strategically undesirable; implement from scratch
- **Wrap** — Capability exists but needs adaptation; wrap the existing implementation with new behavior
- **Delegate** — Capability can be reused or forwarded without modification
- **Adopt** — Take GSD-2's pattern directly, without Pi dependency (documentation/convention, not code)

### Layer 1: Model Layer

| GSD-2 Component | Source | Recommendation | Rationale |
|-----------------|--------|----------------|-----------|
| Pi `ModelRegistry` (multi-provider abstraction) | `packages/pi-agent-core/` [repo code] | **Delegate** | The registry is Pi-owned and provider-agnostic; Atlas can use it directly if Pi is in the dependency chain |
| Per-unit-type model selection | `auto-dispatch.ts` unit configs [repo code] | **Adopt** | The pattern of selecting model tier per task type is a proven design choice; Atlas can implement same pattern independently |
| `RetryHandler` with exponential backoff + fallback chain | `packages/pi-coding-agent/` [repo code] | **Delegate** | Production-quality; no reason to rebuild unless Atlas needs a different retry contract |
| `CompactionOrchestrator` with extension hooks | `packages/pi-coding-agent/` [repo code] | **Delegate** | Context compaction is nontrivial; extension hooks provide Atlas-specific override points without forking |
| Context budget system | `buildSystemPrompt()` options [repo code] | **Wrap** | Budget calculation is sound; Atlas may need different unit-type mappings or template shapes |
| Credential callback (`getApiKey`) | Pi `sdk.ts` [repo code] | **Delegate** | The callback pattern isolates credential source from model layer; reuse the interface, supply your own implementation |

**Layer recommendation:** Delegate Pi's model infrastructure wholesale unless Atlas targets providers or context shapes Pi doesn't support. The retry and compaction implementations are worth keeping to avoid reproducing production-hardening. [inference]

---

### Layer 2: Agent Layer

| GSD-2 Component | Source | Recommendation | Rationale |
|-----------------|--------|----------------|-----------|
| Pi `Agent` class (core message loop) | `packages/pi-agent-core/src/agent.ts` [repo code] | **Delegate** | Core loop is Pi-owned; GSD wraps it rather than replacing it; Atlas should do the same |
| Tool factories (`createBashTool`, `createReadTool`, etc.) | `packages/pi-coding-agent/src/core/tools/` [repo code] | **Delegate** | Pi factories provide standard implementations; wrap `execute` for Atlas-specific behavior |
| GSD-authored workflow tools (`gsd_save_decision`, etc.) | `src/resources/extensions/gsd/` [repo code] | **Rebuild** | These tools are tied to GSD's `.gsd/` schema and naming conventions; Atlas needs equivalents aligned to its own database schema |
| `beforeToolCall`/`afterToolCall` hooks | `AgentSession` [repo code] | **Delegate** | Hook timing guarantees from K008 make this a stable surface; Atlas registers hooks through the same interface |
| `ExtensionRunner` event dispatch | `packages/pi-coding-agent/src/core/extensions/runner.ts` [repo code] | **Delegate** | Event serialization via `_agentEventQueue` (K008) provides correct ordering guarantees; Atlas uses the extension registration API |
| `subagent` tool | GSD extension tools [repo code] | **Wrap** | Subagent composition pattern is useful; Atlas may need different routing logic or context forwarding |

**Layer recommendation:** Reuse Pi's `Agent` class and tool factories. Rebuild the four GSD-specific database tools to match Atlas schemas. Keep the hook/event surfaces — they are the correct extension points. [inference]

---

### Layer 3: Harness Layer

| GSD-2 Component | Source | Recommendation | Rationale |
|-----------------|--------|----------------|-----------|
| `SessionManager` (JSONL persistence, tree navigation) | `packages/pi-coding-agent/src/core/session-manager.ts` [repo code] | **Delegate** | Append-only tree with parent-id links (K011) provides branching, compaction, and blob externalization; production-quality |
| `AgentSession` (session coordinator) | `packages/pi-coding-agent/src/core/agent-session.ts` [repo code] | **Delegate** | Wires Agent + SessionManager + RetryHandler + ExtensionRunner in a proven four-phase assembly (K010) |
| `AutoSession` (mutable auto-mode state) | `src/resources/extensions/gsd/auto/session.ts` [repo code] | **Rebuild** | AutoSession is tightly coupled to GSD's dispatch cycle and milestone state; Atlas needs its own mutable session container |
| Worktree isolation (`isolationMode: "worktree"`) | `src/resources/extensions/gsd/git-service.ts` [repo code] | **Wrap** | Worktree creation and lifecycle are well-implemented; wrap `GitServiceImpl` to customize branch naming or cleanup behavior |
| Crash recovery via plan reconciliation | GSD orchestration layer [repo code] | **Adopt** | The pattern (disk-state → re-derive → resume) is the key insight; implement the same pattern for Atlas's own state model |
| Session lock (`auto.lock`) | GSD runtime exclusions [repo code] | **Adopt** | Single-writer lock via filesystem is simple and effective; use the same pattern for Atlas parallel workers |
| Smart staging with runtime exclusions | `GitServiceImpl.smartStage()` [repo code] | **Wrap** | Exclusion list logic is correct; wrap to customize Atlas runtime paths if they differ from GSD's |

**Layer recommendation:** Delegate Pi's session infrastructure. Rebuild GSD's `AutoSession` for Atlas's own state lifecycle. Wrap `GitServiceImpl` for isolation and staging behavior rather than re-implementing git management. [inference]

---

### Layer 4: Orchestration Layer

| GSD-2 Component | Source | Recommendation | Rationale |
|-----------------|--------|----------------|-----------|
| `deriveState()` (disk → phase derivation) | `src/resources/extensions/gsd/state.ts` [repo code] | **Adapt → Rebuild** | The pattern is the right call; the specific phases and file schema are GSD-specific. Rebuild `deriveState()` equivalent against Atlas's own artifact schema, but keep the disk-first derivation approach |
| `DISPATCH_RULES` (declarative phase→unit map) | `src/resources/extensions/gsd/auto-dispatch.ts` [repo code] | **Rebuild** | Rules reference GSD-specific unit types and phase names; Atlas needs its own rule table with the same evaluation semantics (first-match-wins, K014) |
| `autoLoop` (main iteration cycle) | `src/resources/extensions/gsd/auto/loop.ts` [repo code] | **Wrap** | The loop structure (derive state → match rule → dispatch unit → write summary → repeat) is sound; wrap or copy the loop with Atlas-specific state and rule types |
| Verification gates with auto-retry | `verification-gate.ts` [repo code] | **Adapt → Rebuild** | Discovery and execution logic are reusable patterns; K015 priority order is a good default; rebuild with Atlas-specific discovery preferences |
| Dispatch guard (out-of-order prevention) | GSD guard conditions in `DISPATCH_RULES` [repo code] | **Adopt** | The guard pattern (check preconditions before dispatching) is universal; implement equivalent in Atlas's own dispatch rules |
| Phase taxonomy (10 phases) | `state.ts` phase derivation [repo code] | **Adopt** | `needs-discussion → pre-planning → planning → executing → summarizing → replanning-slice → validating → completing → complete → blocked` is a mature lifecycle; Atlas can adopt this schema directly or prune to a simpler set |

**Layer recommendation:** The orchestration layer is GSD-2's highest-value proprietary contribution. Atlas should **rebuild the components but adopt the patterns**: disk-derived state, declarative rules, verification gates, and crash-recovery-via-re-derive are the right architecture. None of the specific GSD implementations are safely reusable without the Pi dependency chain. [inference]

---

### Layer 5: Protocol / Client Surface

| GSD-2 Component | Source | Recommendation | Rationale |
|-----------------|--------|----------------|-----------|
| Extension API (`pi.registerTool()`, event subscription) | Pi `sdk.ts` [repo code] | **Delegate** | Extension registration surface is stable and purpose-built; Atlas extensions should use this API |
| MCP adapter (tool bridge to MCP protocol) | GSD MCP adapter [repo code] | **Delegate** | MCP compatibility is valuable; Atlas can reuse the adapter to expose tools via standard protocol |
| CLI interface (via Pi) | `src/cli.ts` [repo code] | **Wrap** | Pi's CLI is the entry point; wrap with Atlas-specific flags, config paths, and initialization order |
| ACP compatibility | Not implemented in GSD-2 [inference] | **Rebuild** | If Atlas needs ACP exposure, build an ACP adapter using Atlas's own tool registry — GSD-2 has no reference implementation |
| Agents SDK compatibility | Not implemented in GSD-2 [inference] | **Rebuild** | Codex's Agents SDK composition pattern requires explicit SDK bindings; no GSD-2 equivalent exists |

**Layer recommendation:** Delegate Pi's extension API and the existing MCP adapter. If Atlas needs ACP or multi-framework composition, those are net-new builds. The CLI surface is a thin wrapper — keep it as a wrap rather than replacing Pi's CLI machinery. [inference]

---

### Layer 6: Workflow / Doc-State Layer

| GSD-2 Component | Source | Recommendation | Rationale |
|-----------------|--------|----------------|-----------|
| Disk-state derivation pattern (`.gsd/` artifacts as source of truth) | `state.ts` [repo code] | **Adopt** | This is GSD-2's central architectural insight: derive workflow position from files, not memory. Atlas should adopt this pattern regardless of whether it reuses GSD's schema |
| Milestone → Slice → Task hierarchy | `.gsd/milestones/` schema [repo code] | **Adopt or Adapt** | The three-level hierarchy with dependency tracking is mature and covers most autonomous development use cases; Atlas can adopt wholesale or simplify if the project complexity warrants |
| `ROADMAP.md`, `S##-PLAN.md`, `T##-PLAN.md`, `T##-SUMMARY.md` schema | GSD disk-state convention [repo code] | **Adopt** | File-based, human-readable, markdown-native — these properties make the schema inspectable without tooling. Atlas should preserve this property |
| `deriveState()` milestone/slice/task scanning | `state.ts:175-520` [repo code] | **Rebuild** | File-path conventions will differ in Atlas; a new implementation of `deriveState()` is required even if the pattern is identical |
| Audit trail (human-readable markdown) | All `.gsd/` plan and summary files [repo code] | **Adopt** | Human-readable audit without tooling is a hard requirement for autonomous systems. JSON alternatives lose this property |
| Queue ordering (`queue-order.json`) | `findMilestoneIds()` in `state.ts` [repo code] | **Adopt** | Priority queuing for milestone execution ordering is a simple, effective mechanism; adopt the pattern, implement in Atlas's own file path |

**Layer recommendation:** Adopt the disk-state pattern wholesale — it is the single most valuable transferable insight from GSD-2. The specific file names and paths can change; the property that "workflow state lives on disk and is re-derived on every cycle" should be preserved. [inference]

---

## Master Rebuild/Wrap/Delegate Table

Quick-reference summary across all layers:

| Component | Layer | Action | Complexity | Atlas Priority |
|-----------|-------|--------|------------|----------------|
| Pi `ModelRegistry` | Model | Delegate | Low | High |
| Pi `RetryHandler` | Model | Delegate | Low | High |
| Pi `CompactionOrchestrator` | Model | Delegate | Low | High |
| Context budget system | Model | Wrap | Medium | Medium |
| Per-unit model selection pattern | Model | Adopt | Low | High |
| Pi `Agent` class | Agent | Delegate | Low | Critical |
| Pi tool factories | Agent | Delegate | Low | High |
| GSD workflow tools (4) | Agent | Rebuild | Medium | High |
| Tool hook surfaces | Agent | Delegate | Low | High |
| `ExtensionRunner` events | Agent | Delegate | Low | High |
| `SessionManager` | Harness | Delegate | Low | Critical |
| `AgentSession` | Harness | Delegate | Low | Critical |
| `AutoSession` | Harness | Rebuild | Medium | High |
| Worktree isolation | Harness | Wrap | Medium | High |
| Crash-recovery pattern | Harness | Adopt | Low | High |
| Session lock pattern | Harness | Adopt | Low | Medium |
| Smart staging | Harness | Wrap | Low | Medium |
| Disk-state derivation pattern | Orchestration | Adopt | Low | Critical |
| `deriveState()` implementation | Orchestration | Rebuild | High | Critical |
| `DISPATCH_RULES` table | Orchestration | Rebuild | Medium | Critical |
| `autoLoop` structure | Orchestration | Wrap | Medium | High |
| Verification gates | Orchestration | Rebuild | Medium | High |
| Phase taxonomy | Orchestration | Adopt | Low | High |
| Dispatch guard pattern | Orchestration | Adopt | Low | Medium |
| Extension API | Protocol | Delegate | Low | High |
| MCP adapter | Protocol | Delegate | Low | Medium |
| CLI wrapper | Protocol | Wrap | Low | Medium |
| ACP adapter | Protocol | Rebuild | High | Low |
| Disk-state pattern | Workflow/Doc | Adopt | Low | Critical |
| Milestone/Slice/Task hierarchy | Workflow/Doc | Adopt | Low | High |
| File schema (.md artifacts) | Workflow/Doc | Adopt | Low | High |
| `deriveState()` scanning logic | Workflow/Doc | Rebuild | Medium | Critical |
| Human-readable audit trail | Workflow/Doc | Adopt | Low | Critical |

**Action counts:** Delegate: 10 | Wrap: 7 | Rebuild: 8 | Adopt: 11

---

## Layer-Specific Recommendations

### Model Layer

Atlas should treat the Model Layer as infrastructure, not product. Pi's `ModelRegistry`, `RetryHandler`, and `CompactionOrchestrator` represent significant engineering effort in provider abstraction and production hardening. [repo code] The credential callback pattern (`getApiKey`) cleanly separates credential source from model invocation — this interface should be preserved even if the implementation changes.

**Critical decision point:** If Atlas targets model providers not supported by Pi's `ModelRegistry`, wrapping is required. If Atlas uses only providers Pi already supports (Anthropic, OpenAI, Gemini, etc.), delegation is the correct choice. [inference]

---

### Agent Layer

The `Agent` class in `packages/pi-agent-core/` is the proven message loop. GSD-2 does not replace it — GSD wraps it by registering hooks and tools [repo code]. Atlas should follow the same pattern: register Atlas-specific tools via `pi.registerTool()`, register event handlers via the extension API, and let Pi own the message loop.

The four GSD-specific workflow tools (`gsd_save_decision`, `gsd_update_requirement`, `gsd_save_summary`, `gsd_generate_milestone_id`) are inline implementations that write to a SQLite database and regenerate markdown files [repo code]. Atlas needs equivalents that write to its own database schema. These are pure rebuilds — the function signatures and tool descriptions will differ. [inference]

---

### Harness Layer

`AgentSession` and `SessionManager` are the harness core. The four-phase assembly (Agent → SessionManager → AgentSession → ExtensionRunner via mutable ref) in K010 [repo code] is the correct wiring pattern. Atlas should not attempt to bypass AgentSession — it provides retry, compaction, and event sequencing that would be expensive to reproduce.

`AutoSession` is GSD's mutable state container for a single dispatch cycle. It tracks current phase, active milestone, active slice, retry counts, and step-mode flags. [repo code] Atlas needs a parallel container for its own cycle state. This is a rebuild, not because the pattern is wrong, but because the fields are tightly coupled to GSD-specific dispatch semantics. [inference]

Worktree isolation is the recommended default for autonomous agents. The `GitServiceImpl` smart staging with runtime exclusions prevents transient lock files and database artifacts from polluting commits [repo code]. Atlas should wrap this, customizing only the `RUNTIME_EXCLUSION_PATHS` list for its own runtime paths.

---

### Orchestration Layer

The Orchestration Layer is GSD-2's highest-value architectural contribution and the one most requiring rebuilds. Here is why: [inference]

- `deriveState()` is 345 lines of logic tied to GSD's exact file paths, milestone directory conventions, and phase names [repo code]. Atlas's file paths will differ — a direct fork would produce wrong paths on day one.
- `DISPATCH_RULES` maps GSD's phase names to GSD's unit types. Atlas's phases and unit types will differ. Rules must be rebuilt.
- The *pattern* behind both — read disk, derive phase, match rules, dispatch unit — is exactly correct and should be adopted wholesale.

**Verification gates** deserve special attention. GSD discovers verification commands via a priority order (preference.md → task plan → package.json → none) and sanitizes shell metacharacters before execution [repo code]. Atlas verification gates should implement the same priority-order discovery and sanitization. The exact sources may differ; the safety discipline should not.

---

### Protocol / Client Surface

GSD-2's protocol surface is primarily internal: an extension API exposed to GSD-authored extensions and an MCP adapter for external tools. [repo code] Atlas should not invest heavily in new protocol surfaces until the core layers are stable. The existing MCP adapter provides compatibility with the Claude Code ecosystem (MCP as standard), which is worth preserving. [inference]

If Atlas needs ACP exposure for multi-framework interoperability, that is a net-new build. No GSD-2 ACP adapter exists to wrap. [inference]

---

### Workflow / Doc-State Layer

The disk-state workflow model is the one GSD-2 design bet that Atlas should adopt unconditionally. The argument is direct: [inference]

1. **Crash recovery is free** — If the process dies, re-derive state from disk on restart [repo code]
2. **Parallel workers are safe** — Each worker operates on its own milestone; dispatch guards prevent two workers on the same slice [repo code]
3. **Audit is human-readable** — A developer can `cat` a `T##-SUMMARY.md` and understand project state without tooling [repo code]
4. **Phase transitions are file operations** — Writing a SUMMARY file transitions the slice to complete; no state machine to maintain [repo code]

The Milestone → Slice → Task decomposition hierarchy is also mature. The specific file naming (`S07-PLAN.md`, `T02-SUMMARY.md`) is a convention, not a requirement — Atlas can use different names as long as `deriveState()` knows how to find them. [inference]

---

## Decision Factors for Atlas Builders

The following factors should shape which recommendations to follow and where to deviate:

### Factor 1: Pi Dependency Acceptance

If Atlas accepts Pi (`@pi/agent-core`, `@pi/coding-agent`) as a dependency, the Delegate recommendations apply directly. If Atlas cannot take Pi as a dependency (e.g., different runtime, different package ecosystem), all "Delegate" items become "Rebuild" items. [inference]

**Impact:** High — affects 10 components across all layers.

---

### Factor 2: Target Isolation Level

If Atlas targets single-user interactive workflows, worktree isolation is optional overhead. If Atlas targets parallel autonomous workers on shared repositories, worktree isolation is required for safety. [inference]

**Impact:** Medium — affects Harness Layer and Git Layer recommendations.

---

### Factor 3: Workflow Complexity

If Atlas coordinates work across multiple milestones with inter-milestone dependencies, adopt the full Milestone → Slice → Task hierarchy. If Atlas handles single-session tasks, the hierarchy adds complexity without benefit. [inference]

**Impact:** Medium — affects Workflow/Doc-State and Orchestration Layer scope.

---

### Factor 4: Team Familiarity with Disk-State Pattern

Teams unfamiliar with disk-derived state often reach for in-memory state or a database instead. Both alternatives lose the crash-recovery-for-free property. [inference] Teams should verify understanding of `deriveState()` before choosing an alternative — the pattern feels unusual but its reliability properties are why GSD-2 uses it.

**Impact:** Medium — affects Orchestration and Workflow/Doc-State Layer robustness.

---

### Factor 5: Audit and Traceability Requirements

If Atlas operates in regulated or high-accountability environments, human-readable markdown artifacts are worth maintaining even if a database is used for runtime queries. JSON alternatives are parseable but lose the "inspect without tooling" property that makes GSD-2's audit trail valuable under incident conditions. [inference]

**Impact:** Medium — affects Workflow/Doc-State Layer format choices.

---

### Factor 6: Protocol Surface Requirements

If Atlas needs ACP compatibility for multi-framework agent orchestration, factor in that this is a net-new build (roughly equivalent effort to the MCP adapter). If Atlas operates in a closed environment, skip ACP entirely. [inference]

**Impact:** Low on core function — affects only Protocol Layer scope.

---

## Integration Patterns

How rebuilt, wrapped, and delegated components connect in an Atlas harness:

### Pattern 1: Minimal Atlas Harness (Pi-Dependent)

```
Atlas CLI Entry Point
  └─ createAgentSession() [delegated to Pi]
       ├─ Agent [Pi, delegated]
       ├─ SessionManager [Pi, delegated]
       ├─ AgentSession [Pi, delegated]
       │    ├─ RetryHandler [Pi, delegated]
       │    └─ CompactionOrchestrator [Pi, delegated]
       └─ ExtensionRunner
            └─ Atlas Extension
                 ├─ Atlas AutoSession [rebuilt]
                 ├─ Atlas deriveState() [rebuilt, disk-derived]
                 ├─ Atlas DISPATCH_RULES [rebuilt]
                 ├─ Atlas autoLoop [wrapped from GSD pattern]
                 ├─ Atlas verification gates [rebuilt]
                 └─ Atlas workflow tools [rebuilt, 4+ tools]
```

This pattern preserves the Pi runtime (Delegate all Model/Agent/Harness components) and rebuilds only the GSD-authored orchestration and workflow layers. [inference]

---

### Pattern 2: Disk-State Cycle (Core Loop)

```
1. deriveState()           ← Read .atlas/ disk artifacts
2. matchRule(state)        ← DISPATCH_RULES first-match-wins
3. buildPrompt(unitType)   ← Context assembly for unit
4. dispatch(unit)          ← Run AgentSession for unit
5. writeSummary(result)    ← Persist result to disk
6. runVerification()       ← Verification gate with auto-retry
7. goto 1                  ← Repeat until terminal phase
```

The cycle terminates when `deriveState()` returns `phase: "complete"` or `phase: "blocked"`. [repo code] All state transitions happen in step 5 (writing to disk). No mutable state survives across cycle iterations except disk files. [inference]

---

### Pattern 3: Isolation Lifecycle (Parallel Workers)

```
Atlas Orchestrator
  ├─ Worker A: Milestone M001
  │    ├─ GitService.createWorktree("M001") [wrapped]
  │    ├─ autoLoop(M001 context) [rebuilt pattern]
  │    └─ GitService.mergeWorktree("M001") [wrapped]
  └─ Worker B: Milestone M002
       ├─ GitService.createWorktree("M002") [wrapped]
       ├─ autoLoop(M002 context) [rebuilt pattern]
       └─ GitService.mergeWorktree("M002") [wrapped]
```

Parallel workers are safe because each operates in its own worktree with its own branch. [repo code] The dispatch guard prevents two workers from operating on the same slice even if both read the same disk state. [inference]

---

### Pattern 4: Verification Gate (Quality Enforcement)

```
Unit completion
  └─ discoverCommands(taskPlan, preferences) [rebuilt]
       ├─ Priority 1: preference.md global overrides
       ├─ Priority 2: task-plan verification section
       └─ Priority 3: package.json scripts (if configured)
            └─ sanitize(command)  ← Remove shell metacharacters
                 └─ execute(command, timeout)
                      ├─ PASS → continue to next unit
                      └─ FAIL → retry (max N) → replanning-slice
```

The sanitization step is non-negotiable for autonomous execution. [repo code] A task plan can specify arbitrary strings as verification commands; metacharacter removal prevents injection. [inference]

---

## Observability Notes

No runtime signals are generated by this document (documentation task). For Atlas harness builders implementing the above patterns, the following observability surfaces from GSD-2 are worth preserving:

- **Phase emit on each cycle** — Log the phase returned by `deriveState()` at the start of each loop iteration. This makes the harness state machine inspectable in real time. [inference]
- **Dispatch rule match** — Log which rule matched and which unit was dispatched. Makes dispatch failures diagnosable without re-reading all rules. [inference]
- **Verification gate result** — Log exit code and duration of each verification command. Failing gates are the primary signal for quality regression. [inference]
- **Session JSONL path** — Log the session file path on session creation. Enables post-hoc inspection of context windows and tool calls. [repo code]
- **Worktree path** — Log the worktree path on creation. Makes parallel worker debugging tractable. [repo code]

---

## Evidence Summary

| Evidence Tier | Label | Appearances in This Document |
|--------------|-------|------------------------------|
| Tier 1 | `[repo code]` | 35+ |
| Tier 4 | `[inference]` | 25+ |

All `[repo code]` claims trace to source files documented in prior slices (S02–S06). All `[inference]` claims are cross-system reasoning from the comparative analysis in `GSD2_COMPARATIVE_ANALYSIS.md`. No `[external]` claims are present — this document synthesizes prior analysis rather than introducing new external sources.

---

## Cross-References

- **Runtime Architecture**: [GSD2_RUNTIME_ARCHITECTURE.md](./GSD2_RUNTIME_ARCHITECTURE.md) — Component wiring, session lifecycle, event flow
- **Orchestration Layer**: [GSD2_ORCHESTRATION_LAYER.md](./GSD2_ORCHESTRATION_LAYER.md) — Dispatch rules, phase taxonomy, verification gates
- **Context Engineering**: [GSD2_CONTEXT_ENGINEERING_MODEL.md](./GSD2_CONTEXT_ENGINEERING_MODEL.md) — Prompt assembly, disk-state mechanics
- **Isolation Model**: [GSD2_GIT_AND_ISOLATION_MODEL.md](./GSD2_GIT_AND_ISOLATION_MODEL.md) — Worktree lifecycle, smart staging, crash recovery
- **Comparative Analysis**: [GSD2_COMPARATIVE_ANALYSIS.md](./GSD2_COMPARATIVE_ANALYSIS.md) — Shared-primitives matrix, design bets, system profiles
- **Normalized Glossary**: [GLOSSARY_NORMALIZED_TERMS.md](./GLOSSARY_NORMALIZED_TERMS.md) — Term definitions, Atlas crosswalks, partial equivalences
- **Knowledge Base**: [../../.gsd/KNOWLEDGE.md](../../.gsd/KNOWLEDGE.md) — K016 (rebuild/wrap/delegate taxonomy), K017 (six-dimensional primitives)
