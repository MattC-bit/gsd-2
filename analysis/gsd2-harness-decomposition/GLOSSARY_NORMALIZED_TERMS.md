# GSD-2 Normalized Glossary with Atlas Crosswalks

This document provides a normalized glossary of GSD-2 terminology with explicit Atlas-oriented crosswalks. It enables Atlas harness builders to understand GSD-2 terms without forcing one-to-one mappings where concepts don't cleanly align.

---

> **[D007]** Throughout this document, references to `.gsd/`, milestones, slices, and workflow state refer to the **analyzed GSD-2 system** unless explicitly noted otherwise. This document is produced *inside* a live GSD run in a fork of GSD-2. The `.gsd/` artifacts producing this analysis (the runner) are separate from the `.gsd/` model being analyzed (the subject). See [EVIDENCE_METHOD.md](./EVIDENCE_METHOD.md#subject-vs-runner-guardrail) for the full guardrail.

---

## Subject-vs-Runner Guardrail

| Concept | What It Means | Where to Find It |
|---------|---------------|------------------|
| **Subject** | The GSD-2 system being analyzed | The analyzed `.gsd/` workflow model, GSD source code, GSD architecture |
| **Runner** | The live GSD execution producing this pack | The current reverse-engineering run's `.gsd/` planning artifacts |

**Key distinction**: When this glossary defines "milestone" or "slice," it describes the analyzed GSD-2 system's concepts — not the runner's own milestone/slice structure that produced this document [repo code].

---

## How to Use This Glossary

Each term entry includes:

1. **GSD-Native Definition**: The meaning in GSD-2 context with evidence label ([repo code], [repo doc], [inference])
2. **Atlas Crosswalk**: How an Atlas harness would interpret or adapt this concept
3. **Partial Equivalence Note**: Where GSD and Atlas concepts don't map cleanly

**Evidence tiers**:
- `[repo code]` — Directly observed in repository source
- `[repo doc]` — Found in repository documentation
- `[external]` — Sourced from outside the repository
- `[inference]` — Synthesized or extrapolated from code patterns

---

## Runtime Domain

### Agent

**GSD-Native Definition**: The core agent class from `pi-agent-core` that manages LLM interactions, message loops, and tool execution hooks. Provides `transformContext`, `convertToLlm`, `beforeToolCall`, `afterToolCall`, and `onPayload` callbacks for orchestration layers [repo code].

**Atlas Crosswalk**: 
- **Atlas equivalent**: Agent is a standard primitive in most harnesses
- **Reuse strategy**: **Wrap** — Atlas can wrap Pi's Agent class directly or implement compatible interface
- **Key capability**: Hook system for injecting behavior into LLM calls

**Partial Equivalence**: GSD's Agent is Pi-owned, not GSD-authored. Atlas may want its own Agent implementation rather than depending on Pi's.

---

### AgentSession

**GSD-Native Definition**: Pi-owned session coordination class that wires Agent, SessionManager, RetryHandler, and CompactionOrchestrator. Manages session lifecycle (newSession, fork, switchSession, navigateTree), event propagation to ExtensionRunner, and tool hook installation [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Session manager in most harnesses
- **Reuse strategy**: **Wrap** — Atlas can wrap AgentSession or implement similar interface
- **Key capability**: Event queue serialization guarantees extension timing

**Partial Equivalence**: GSD's auto-mode manages fresh session creation per unit via `ctx.newSession()`, but delegates to Pi's AgentSession — there's no GSD-authored session management layer [repo code].

---

### SessionManager

**GSD-Native Definition**: Pi-owned persistence layer that manages JSONL session files with append-only tree structure. Provides `newSession()`, `appendMessage()`, `branch()`, `getLeafId()`, and `buildSessionContext()` for context reconstruction [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Session persistence is a standard primitive
- **Reuse strategy**: **Reuse** — JSONL format is well-designed; Atlas could adopt directly
- **Key capability**: Tree structure for session branching and navigation

**Partial Equivalence**: None — SessionManager is a clean primitive that could be extracted and reused.

---

### RetryHandler

**GSD-Native Definition**: Pi-owned retry mechanism with error classification (retryable vs non-retryable), exponential backoff, and fallback chain across providers and credentials. Distinguishes context overflow (goes to compaction) from server errors (retry with backoff) [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Retry logic is common but varies by implementation
- **Reuse strategy**: **Wrap** — Pattern is sound; Atlas may want provider-specific adaptations
- **Key capability**: Credential fallback and cross-provider fallback chain

**Partial Equivalence**: GSD's RetryHandler is tied to Pi's model registry. Atlas would need to wire to its own provider system.

---

### CompactionOrchestrator

**GSD-Native Definition**: Pi-owned context compaction manager with triggers (overflow vs threshold), extension integration (cancel, provide custom compaction), and LLM-generated summaries. Creates `CompactionEntry` in session file with `summary` and `firstKeptEntryId` [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Context window management is universal
- **Reuse strategy**: **Delegate** — Atlas could delegate to LLM-native compaction if available
- **Key capability**: Extension hooks for custom compaction strategies

**Partial Equivalence**: GSD's compaction creates disk artifacts (CompactionEntry). Atlas may prefer in-memory or different persistence.

---

### EventStream

**GSD-Native Definition**: Pi-owned event emission mechanism in `agent-loop.ts`. Emits events: `agent_start`, `turn_start`, `message_start`, `message_update`, `message_end`, `turn_end`, `agent_end`, `tool_execution_*` [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Event system is standard
- **Reuse strategy**: **Reuse** — Event types are well-designed
- **Key capability**: Streaming events for real-time UI updates

**Partial Equivalence**: None — Event types are domain-appropriate and reusable.

---

### ExtensionRunner

**GSD-Native Definition**: Pi-owned extension lifecycle manager that emits events to registered handlers, manages hook execution order, and processes cancellation results. Provides `emit()` for broadcasting to extension handlers [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Plugin/extension system
- **Reuse strategy**: **Reuse** — Clean extension API design
- **Key capability**: Cancellable events with `result.cancel` pattern

**Partial Equivalence**: GSD's extension registration (`registerGsdExtension`) is GSD-authored but uses Pi's ExtensionRunner.

---

### Mutable Ref Pattern

**GSD-Native Definition**: Pattern for resolving circular Agent↔ExtensionRunner dependency. Agent is created with `transformContext` that delegates to `extensionRunnerRef.current`. ExtensionRunner is created later and stored in the ref [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Dependency injection pattern
- **Reuse strategy**: **Adopt** — Useful for similar initialization ordering issues
- **Key capability**: Safe callback before dependency exists

**Partial Equivalence**: This is an implementation pattern, not a conceptual primitive.

---

### Context Window

**GSD-Native Definition**: Token limit for LLM context, resolved via model registry or session settings. GSD's budget system uses this to allocate character budgets for inline content (summary, verification, etc.) [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Standard LLM constraint
- **Reuse strategy**: **N/A** — Determined by LLM provider
- **Key capability**: Budget ratios for content categories

**Partial Equivalence**: GSD's budget ratios (15% summary, 40% inline, 10% verification) are domain-specific; Atlas may choose different allocations.

---

## Orchestration Domain

### AutoSession

**GSD-Native Definition**: GSD-authored encapsulation of all mutable auto-mode state: `active`, `paused`, `stepMode`, `currentUnit`, `completedUnits`, `unitDispatchCount`, `currentMilestoneId`, `sidecarQueue`, `pendingVerificationRetry` [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Workflow state machine
- **Reuse strategy**: **Rebuild** — Atlas may want different state structure
- **Key capability**: Clean encapsulation invariant (no module-level `let`/`var`)

**Partial Equivalence**: GSD's AutoSession is specific to GSD's dispatch model. Atlas with different workflow structure would need different state.

---

### deriveState()

**GSD-Native Definition**: GSD-authored function that reads `.gsd/` disk artifacts to determine workflow phase, active milestone, active slice, active task, and eligibility. Returns `GSDState` object derived entirely from files, not mutable runtime state [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: State derivation is architectural choice
- **Reuse strategy**: **Rebuild** — Pattern is powerful but GSD-specific
- **Key capability**: Crash recovery without special handling

**Partial Equivalence**: GSD's `deriveState()` is tied to GSD's `.gsd/` artifact structure. Atlas with different workflow model would need different derivation logic.

---

### Phase

**GSD-Native Definition**: Workflow position derived from disk state: `needs-discussion`, `pre-planning`, `planning`, `executing`, `summarizing`, `replanning-slice`, `validating-milestone`, `completing-milestone`, `complete`, `blocked` [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Workflow stage/state
- **Reuse strategy**: **Adapt** — Phases are domain-appropriate
- **Key capability**: Phase transitions encoded as file operations

**Partial Equivalence**: GSD's phases are specific to GSD's workflow model. Different workflow structures would have different phases.

---

### DISPATCH_RULES

**GSD-Native Definition**: GSD-authored declarative dispatch table with 20+ rules mapping phase + context → unit type. Evaluated first-match-wins in array order [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Routing/dispatch table
- **Reuse strategy**: **Adapt** — Pattern is reusable; rules are GSD-specific
- **Key capability**: Declarative dispatch without imperative state machine

**Partial Equivalence**: GSD's rules are specific to GSD's phases and unit types. Atlas would define its own rules.

---

### Unit Type

**GSD-Native Definition**: Dispatch target representing a discrete work operation: `discuss-milestone`, `research-milestone`, `plan-milestone`, `research-slice`, `plan-slice`, `execute-task`, `complete-slice`, `validate-milestone`, `complete-milestone`, `replan-slice`, `run-uat`, `reassess-roadmap`, `reactive-execute` [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Task type / operation type
- **Reuse strategy**: **Adapt** — Pattern is useful; types are GSD-specific
- **Key capability**: Unit-specific prompt builders

**Partial Equivalence**: GSD's unit types are specific to GSD's workflow model. Different workflows would have different unit types.

---

### Dispatch Rule

**GSD-Native Definition**: Declarative mapping from condition to action. Each rule has `match(context) → result` where result contains `action: "dispatch" | "stop" | "skip"` and optional `unitType`, `unitId` [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Rule in routing table
- **Reuse strategy**: **Reuse** — Pattern is clean and extensible
- **Key capability**: First-match-wins evaluation order

**Partial Equivalence**: Rule conditions are GSD-specific. Pattern is reusable.

---

### Verification Gate

**GSD-Native Definition**: GSD-authored quality enforcement layer that runs commands after `execute-task` units. Discovers commands from preference, task plan, or package.json. Supports auto-retry with failure context injection [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: CI/quality gate
- **Reuse strategy**: **Adapt** — Pattern is useful; discovery order is GSD-specific
- **Key capability**: Auto-fix retry with context injection

**Partial Equivalence**: GSD's verification discovery order (D003) is GSD-specific. The pattern is reusable.

---

### Dispatch Guard

**GSD-Native Definition**: GSD-authored out-of-order execution prevention. Blocks dispatching to a slice if prior slices or declared dependencies are incomplete. Reads roadmap from disk (working tree), not git branches [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Dependency enforcement
- **Reuse strategy**: **Adapt** — Pattern is useful for any ordered workflow
- **Key capability**: Dependency-aware ordering beyond positional

**Partial Equivalence**: GSD's guard is specific to GSD's milestone/slice structure. The pattern is reusable.

---

### autoLoop

**GSD-Native Definition**: Main iteration cycle in `auto/loop.ts` with five phases: PRE-DISPATCH, GUARDS, DISPATCH, UNIT EXECUTION, FINALIZE. Loops while `s.active` [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Main loop / control loop
- **Reuse strategy**: **Adapt** — Phase structure is useful; phases are GSD-specific
- **Key capability**: Phase separation with clear responsibilities

**Partial Equivalence**: GSD's loop phases are specific to GSD's model. The pattern is reusable.

---

### Step Mode

**GSD-Native Definition**: Single-step auto-mode that pauses after each unit for human review. Controlled via `s.stepMode` flag in AutoSession [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Single-step execution mode
- **Reuse strategy**: **Reuse** — Standard pattern for human-in-the-loop
- **Key capability**: Pause after each unit for inspection

**Partial Equivalence**: None — step mode is a standard pattern.

---

## Workflow State Domain

### Milestone

**GSD-Native Definition**: Top-level work container with ROADMAP.md (slice definitions), CONTEXT.md (scope), RESEARCH.md (findings), SUMMARY.md (completion). Stored in `.gsd/milestones/M###/` directory [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Project / Epic / Feature
- **Reuse strategy**: **Adapt** — Concept is common; structure is GSD-specific
- **Key capability**: Milestone-level isolation via worktree

**Partial Equivalence**: GSD's milestones are strongly structured with required artifacts. Other systems may use less formal containers.

---

### Slice

**GSD-Native Definition**: Mid-level work container within milestone with PLAN.md (goal, demo, must-haves, tasks), RESEARCH.md (findings), SUMMARY.md (completion). Stored in `.gsd/milestones/M###/slices/S##/` [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Sprint / Phase / Stage
- **Reuse strategy**: **Adapt** — Concept is useful; structure is GSD-specific
- **Key capability**: Slice-level dependency declarations

**Partial Equivalence**: GSD's slices have explicit dependency declarations. Other systems may use different dependency models.

---

### Task

**GSD-Native Definition**: Atomic work unit with PLAN.md (steps, inputs, outputs) and SUMMARY.md (completion with frontmatter). Stored in `.gsd/milestones/M###/slices/S##/tasks/` [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Task / Ticket / Issue
- **Reuse strategy**: **Reuse** — Atomic task concept is universal
- **Key capability**: Task-level verification commands

**Partial Equivalence**: GSD's tasks have structured plans with steps. Other systems may use less formal task descriptions.

---

### ROADMAP.md

**GSD-Native Definition**: Milestone-level artifact defining slices with checkboxes, dependencies, and boundaries. Parsed by `parseRoadmap()` to determine slice status and dependencies [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Project plan / Roadmap
- **Reuse strategy**: **Adapt** — Format is GSD-specific; concept is universal
- **Key capability**: Dependency declarations via `depends_on`

**Partial Equivalence**: GSD's ROADMAP format is specific to GSD's parsing. Other systems use different planning formats.

---

### PLAN.md

**GSD-Native Definition**: Slice or task-level artifact with goal, demo, must-haves, verification, and task list (for slices) or steps, inputs, outputs (for tasks). Parsed by `parsePlan()` [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Work plan / Specification
- **Reuse strategy**: **Adapt** — Format is GSD-specific; concept is universal
- **Key capability**: Structured sections for LLM consumption

**Partial Equivalence**: GSD's PLAN format is optimized for LLM prompt injection. Other formats may serve different purposes.

---

### SUMMARY.md

**GSD-Native Definition**: Completion artifact with frontmatter (id, parent, milestone, duration, verification_result, blocker_discovered) and narrative summary. Presence indicates completion [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Completion record / Report
- **Reuse strategy**: **Adapt** — Format is GSD-specific; concept is universal
- **Key capability**: Frontmatter for structured data extraction

**Partial Equivalence**: GSD's SUMMARY format is specific to GSD's workflow. Other systems may use different completion records.

---

### CONTEXT.md

**GSD-Native Definition**: Milestone-level scope definition with background, objectives, constraints, dependencies. Loaded during planning phases for context [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Project context / PRD section
- **Reuse strategy**: **Adapt** — Concept is common; structure is GSD-specific
- **Key capability**: Dependency declarations (`depends_on`)

**Partial Equivalence**: GSD's CONTEXT includes GSD-specific dependency structure. Other systems use different context formats.

---

### RESEARCH.md

**GSD-Native Definition**: Milestone or slice-level research findings. Created by `research-milestone` and `research-slice` units [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Research notes / Discovery document
- **Reuse strategy**: **Adapt** — Concept is common; structure is GSD-specific
- **Key capability**: LLM-generated research with evidence trails

**Partial Equivalence**: GSD's RESEARCH is tied to GSD's unit types. Other systems generate research differently.

---

### DECISIONS.md

**GSD-Native Definition**: Project-level architectural decisions with auto-generated IDs (D001, D002, ...). Updated by `gsd_save_decision` tool. Serves as institutional memory for design choices [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Architecture Decision Records (ADRs)
- **Reuse strategy**: **Reuse** — ADR pattern is standard
- **Key capability**: Auto-generated IDs with scope tracking

**Partial Equivalence**: GSD's decisions are database-backed with DECISIONS.md as the rendered view. Pure ADRs are typically file-only.

---

### REQUIREMENTS.md

**GSD-Native Definition**: Project-level requirements with status tracking (active, validated, deferred). Updated by `gsd_update_requirement` tool. Links requirements to owning slices [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Requirements document / Spec
- **Reuse strategy**: **Adapt** — Concept is universal; structure is GSD-specific
- **Key capability**: Status tracking with slice ownership

**Partial Equivalence**: GSD's requirements are database-backed. Other systems may use issue trackers.

---

### KNOWLEDGE.md

**GSD-Native Definition**: Project-specific patterns, gotchas, and lessons learned. Entries capture non-obvious rules discovered during execution [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Project wiki / Knowledge base
- **Reuse strategy**: **Adapt** — Concept is common; structure is GSD-specific
- **Key capability**: Agent-readable format for context injection

**Partial Equivalence**: GSD's KNOWLEDGE is optimized for LLM consumption. Other knowledge bases may have different formats.

---

### PROJECT.md

**GSD-Native Definition**: Project context with constraints, conventions, and technology stack. Loaded as root-level context for all units [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: CLAUDE.md / AGENTS.md / Project README
- **Reuse strategy**: **Adapt** — Concept is universal
- **Key capability**: Project-specific context injection

**Partial Equivalence**: GSD's PROJECT.md is specific to GSD's context loading. Other systems use CLAUDE.md, AGENTS.md, or similar.

---

## Isolation Domain

### Worktree

**GSD-Native Definition**: Git worktree at `.gsd/worktrees/<name>/` providing isolated working directory for execution. Each milestone gets its own worktree with dedicated branch [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Sandbox / Isolated environment
- **Reuse strategy**: **Reuse** — Git worktree is a standard isolation mechanism
- **Key capability**: Filesystem isolation without repository duplication

**Partial Equivalence**: GSD's worktree isolation is git-specific. Other systems may use containers or virtual environments.

---

### Isolation Mode

**GSD-Native Definition**: Three execution isolation modes: `worktree` (isolated directory + branch), `branch` (project root + milestone branch), `none` (current branch) [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Execution mode / Safety level
- **Reuse strategy**: **Adapt** — Concept is useful; modes are GSD-specific
- **Key capability**: Configurable isolation level

**Partial Equivalence**: GSD's isolation modes are git-centric. Other systems may have different isolation mechanisms.

---

### Integration Branch

**GSD-Native Definition**: Target branch for milestone merges — typically `main` or `master`. Determined via preference → milestone metadata → worktree base → auto-detect resolution order [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Main branch / Target branch
- **Reuse strategy**: **Reuse** — Standard git concept
- **Key capability**: Persistent recording per milestone

**Partial Equivalence**: GSD's integration branch is recorded per milestone. Other systems may use different tracking.

---

### Session Lock

**GSD-Native Definition**: OS-level file lock at `.gsd/auto.lock` preventing concurrent auto-mode sessions. Uses `proper-lockfile` with heartbeat and stale detection [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Process lock / Mutex
- **Reuse strategy**: **Reuse** — Standard pattern
- **Key capability**: Crash recovery via stale detection

**Partial Equivalence**: GSD's lock includes metadata (PID, unitType, unitId). Other locks may be simpler.

---

### Dispatch Guard (Isolation Context)

**GSD-Native Definition**: Prevention of out-of-order slice execution. Ensures slices execute in dependency order, respecting `depends_on` declarations [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Dependency enforcement / Ordering constraint
- **Reuse strategy**: **Adapt** — Pattern is useful for ordered workflows
- **Key capability**: Dependency-aware beyond positional

**Partial Equivalence**: GSD's guard is specific to milestone/slice structure. Pattern is reusable.

---

## Context Engineering Domain

### Prompt Builder

**GSD-Native Definition**: GSD-authored async function that constructs prompts for a specific unit type. Examples: `buildExecuteTaskPrompt()`, `buildPlanSlicePrompt()` [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Prompt template / Prompt constructor
- **Reuse strategy**: **Adapt** — Pattern is useful; builders are GSD-specific
- **Key capability**: Unit-specific context assembly

**Partial Equivalence**: GSD's prompt builders are tied to GSD's unit types. Pattern is reusable.

---

### buildSystemPrompt()

**GSD-Native Definition**: Pi-owned core system prompt construction with tool descriptions, guidelines, and context files. Foundation for all GSD prompts [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: System prompt constructor
- **Reuse strategy**: **Wrap** — Core prompt is reusable; extend with domain-specific context
- **Key capability**: Tool-aware guideline construction

**Partial Equivalence**: GSD's system prompt is built on Pi's foundation. Other systems have different prompt structures.

---

### Context Budget

**GSD-Native Definition**: Character allocation system using proportional ratios: 15% summary, 40% inline context, 10% verification. Enforces section-boundary truncation [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Token budgeting / Context window management
- **Reuse strategy**: **Adapt** — Pattern is useful; ratios may vary
- **Key capability**: Section-boundary truncation (not mid-content)

**Partial Equivalence**: GSD's ratios are domain-specific. Other systems may allocate differently.

---

### Template Loader

**GSD-Native Definition**: System for loading prompt templates with variable substitution and eager caching. Templates loaded at startup, immune to mid-session changes [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Template system
- **Reuse strategy**: **Reuse** — Pattern is clean
- **Key capability**: Pre-substitution validation catches errors early

**Partial Equivalence**: None — template pattern is standard.

---

### Inline Helper

**GSD-Native Definition**: Functions for including file content in prompts: `inlineFile()` (with fallback), `inlineFileOptional()` (returns null if missing), `inlineFileSmart()` (truncates at section boundaries) [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Content inclusion helper
- **Reuse strategy**: **Reuse** — Pattern is useful
- **Key capability**: Graceful handling of missing files

**Partial Equivalence**: None — pattern is universally applicable.

---

### Skill Activation

**GSD-Native Definition**: Context-aware skill matching based on milestone/slice/task titles. Suggests relevant skills to the LLM via skill_activation block [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Skill suggestion / Tool recommendation
- **Reuse strategy**: **Adapt** — Pattern is useful; matching is GSD-specific
- **Key capability**: Context-aware relevance

**Partial Equivalence**: GSD's skill system is GSD-specific. Pattern is reusable.

---

## Tool System Domain

### Tool Factory

**GSD-Native Definition**: Pi-owned function that creates an `AgentTool` object with name, label, description, parameters, and execute. Examples: `createBashTool()`, `createReadTool()` [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Tool constructor
- **Reuse strategy**: **Reuse** — Pattern is clean
- **Key capability**: Standardized tool structure

**Partial Equivalence**: None — factory pattern is standard.

---

### Tool Wrapping

**GSD-Native Definition**: GSD pattern of importing Pi factories, spreading properties, and overriding execute function with GSD-specific behavior (default timeouts, cwd resolution) [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Tool middleware / Decorator
- **Reuse strategy**: **Reuse** — Pattern is useful
- **Key capability**: Add behavior without modifying factory

**Partial Equivalence**: None — wrapping pattern is standard.

---

### GSD-Owned Tools

**GSD-Native Definition**: Tools defined entirely in GSD without Pi factories: `gsd_save_decision`, `gsd_update_requirement`, `gsd_save_summary`, `gsd_generate_milestone_id` [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Domain-specific tools
- **Reuse strategy**: **Rebuild** — Tools are GSD-specific
- **Key capability**: Database integration

**Partial Equivalence**: GSD's tools are tied to GSD's database and workflow model. Pattern is reusable; implementations are not.

---

### Extension API

**GSD-Native Definition**: Pi's API for registering tools, commands, hooks, and shortcuts. Provided via `pi` parameter in extension registration [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Plugin API
- **Reuse strategy**: **Wrap** — API is clean; could adapt for different extension systems
- **Key capability**: Unified registration interface

**Partial Equivalence**: GSD's extension API is Pi-owned. Atlas may want its own extension system.

---

## Comparative Domain

### Rebuild/Wrap/Delegate

**GSD-Native Definition**: Taxonomy for cross-system integration: **Rebuild** (architecturally incompatible), **Wrap** (exists but needs adaptation), **Delegate** (can delegate without modification) [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Integration strategy
- **Reuse strategy**: **Adopt** — Useful framework for decisions
- **Key capability**: Structured integration analysis

**Partial Equivalence**: None — this is an analytical framework, not a system feature.

---

### Owned Layers

**GSD-Native Definition**: Architectural ownership across six dimensions: Model, Agent, Harness, Orchestration, Protocol, Workflow/Doc-State. GSD-2 uniquely owns Orchestration and Workflow/Doc-State [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Architecture ownership model
- **Reuse strategy**: **Adopt** — Useful for system comparison
- **Key capability**: Clear responsibility boundaries

**Partial Equivalence**: None — this is an analytical framework.

---

### Six Primitives

**GSD-Native Definition**: Fundamental architectural dimensions for agent systems: Model Layer, Agent Layer, Harness Layer, Orchestration Layer, Protocol/Client Surface, Workflow/Doc-State Layer [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Architecture dimensions
- **Reuse strategy**: **Adopt** — Useful for system design
- **Key capability**: Comprehensive stack coverage

**Partial Equivalence**: None — this is an analytical framework.

---

## Evidence Domain

### Evidence Tiers

**GSD-Native Definition**: Five-tier evidence labeling system: `[repo code]` (directly observed), `[repo doc]` (repo documentation), `[external]` (outside source), `[inference]` (synthesized), `[unresolved]` (insufficient evidence) [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Evidence grading / Claim strength
- **Reuse strategy**: **Adopt** — Useful for documentation rigor
- **Key capability**: Explicit claim strength

**Partial Equivalence**: None — this is an analytical framework.

---

### Subject-vs-Runner

**GSD-Native Definition**: Distinction between the system being analyzed (subject) and the live execution producing analysis (runner). Prevents conflation of `.gsd/` artifacts [repo code].

**Atlas Crosswalk**:
- **Atlas equivalent**: Meta-analysis guardrail
- **Reuse strategy**: **Adopt** — Critical for self-referential analysis
- **Key capability**: Prevents category error

**Partial Equivalence**: None — this is specific to analyzing GSD from within GSD.

---

## Atlas Crosswalk Summary

The following table summarizes Atlas crosswalk strategies across all GSD-2 concepts:

| Strategy | Count | Meaning |
|----------|-------|---------|
| **Reuse** | 12 | Concept/primitive is directly reusable with minimal adaptation |
| **Adapt** | 14 | Concept is useful but requires adaptation to Atlas context |
| **Wrap** | 8 | Exists but needs wrapping layer for Atlas use |
| **Rebuild** | 5 | Architecturally incompatible; needs new implementation |
| **Delegate** | 1 | Can delegate to external component |
| **Adopt** | 4 | Useful analytical framework to adopt |
| **N/A** | 1 | Determined externally (LLM provider) |

### Reuse Candidates (Atlas can adopt directly)

| Primitive | Domain | Key Capability |
|-----------|--------|----------------|
| SessionManager | Runtime | JSONL persistence with tree structure |
| EventStream | Runtime | Streaming events for UI |
| ExtensionRunner | Runtime | Cancellable events pattern |
| Step Mode | Orchestration | Human-in-the-loop pause |
| Dispatch Rule | Orchestration | First-match-wins evaluation |
| Task | Workflow | Atomic task concept |
| Worktree | Isolation | Git worktree isolation |
| Session Lock | Isolation | Process lock with stale detection |
| Template Loader | Context | Pre-substitution validation |
| Inline Helper | Context | Graceful file inclusion |
| Tool Factory | Tools | Standardized tool structure |
| Tool Wrapping | Tools | Add behavior without modification |

### Rebuild Candidates (Atlas needs new implementation)

| Primitive | Domain | Why Rebuild |
|-----------|--------|-------------|
| AutoSession | Orchestration | Specific to GSD's dispatch model |
| deriveState() | Orchestration | Tied to GSD's artifact structure |
| GSD-Owned Tools | Tools | Tied to GSD's database/workflow |

### Adapt Candidates (Atlas needs adaptation)

| Primitive | Domain | Adaptation Needed |
|-----------|--------|-------------------|
| Phase | Orchestration | Different workflow phases |
| Unit Type | Orchestration | Different unit types |
| DISPATCH_RULES | Orchestration | Different rules |
| Milestone | Workflow | Less formal structure |
| Slice | Workflow | Different dependency model |
| ROADMAP.md | Workflow | Different format |
| PLAN.md | Workflow | Different format |
| Context Budget | Context | Different ratio allocations |
| Isolation Mode | Isolation | Different isolation mechanisms |

---

## Partial Equivalence Patterns

The following patterns emerge where GSD-2 and Atlas concepts don't map cleanly:

### 1. Orchestration Layer Ownership

**GSD-2** owns orchestration as a distinct layer. **Most systems** delegate orchestration to users or don't have it as a first-class concept. Atlas must decide whether to own orchestration or delegate.

### 2. Disk-State Model

**GSD-2** derives all state from disk artifacts. **Most systems** use session memory. This is a fundamental architectural choice with cascading implications for crash recovery, parallel execution, and audit.

### 3. Milestone/Slice/Task Decomposition

**GSD-2** enforces a three-level decomposition. **Other systems** use different structures (epics/stories, projects/phases, etc.). Atlas must decide on decomposition structure.

### 4. Git-Centric Isolation

**GSD-2** uses git worktrees for isolation. **Other systems** may use containers, VMs, or no isolation. Atlas isolation strategy affects worktree/primitive applicability.

### 5. Evidence-Based Documentation

**GSD-2's decomposition pack** uses explicit evidence tiers. **Most systems** don't enforce this discipline. Atlas should adopt for documentation rigor.

---

## Cross-References

- **System Overview**: [GSD2_SYSTEM_OVERVIEW.md](./GSD2_SYSTEM_OVERVIEW.md)
- **Runtime Architecture**: [GSD2_RUNTIME_ARCHITECTURE.md](./GSD2_RUNTIME_ARCHITECTURE.md)
- **Orchestration Layer**: [GSD2_ORCHESTRATION_LAYER.md](./GSD2_ORCHESTRATION_LAYER.md)
- **Context Engineering**: [GSD2_CONTEXT_ENGINEERING_MODEL.md](./GSD2_CONTEXT_ENGINEERING_MODEL.md)
- **Isolation Model**: [GSD2_GIT_AND_ISOLATION_MODEL.md](./GSD2_GIT_AND_ISOLATION_MODEL.md)
- **Comparative Analysis**: [GSD2_COMPARATIVE_ANALYSIS.md](./GSD2_COMPARATIVE_ANALYSIS.md)
- **Evidence Method**: [EVIDENCE_METHOD.md](./EVIDENCE_METHOD.md)

---

## Summary

This glossary documents 45+ GSD-2 concepts with Atlas crosswalks, organized across seven architectural domains: Runtime, Orchestration, Workflow State, Isolation, Context Engineering, Tool System, and Comparative.

**Key findings for Atlas builders**:

1. **Reuse SessionManager, EventStream, ExtensionRunner** — These Pi-owned primitives are clean and reusable [repo code]

2. **Adapt the dispatch pattern, not the rules** — The declarative dispatch table pattern is reusable, but rules are GSD-specific [repo code]

3. **Rebuild deriveState() for Atlas workflow** — The disk-derivation pattern is powerful but tied to GSD's artifact structure [repo code]

4. **Adopt evidence tiers and subject-vs-runner guardrail** — These analytical frameworks improve documentation rigor [repo code]

5. **Decide on orchestration ownership** — GSD-2 uniquely owns orchestration as a first-class layer; Atlas must make an explicit choice [repo code]

All references to `.gsd/`, milestones, slices, and workflow state in this glossary denote the **analyzed GSD-2 system**, not the live runner state producing this documentation.