# GSD-2 Comparative Structural Analysis

This document compares GSD-2 with other AI coding agent systems — Claude Code, Codex CLI, and ACP-style agent protocols — to identify shared primitives, owned layers, and design bets. The comparison uses clearly labeled evidence tiers to enable readers to assess claim strength.

---

> **[D007]** Throughout this document, references to `.gsd/`, milestones, slices, and workflow state refer to the **analyzed GSD-2 system** unless explicitly noted otherwise. This document is produced *inside* a live GSD run in a fork of GSD-2. The `.gsd/` artifacts producing this analysis (the runner) are separate from the `.gsd/` model being analyzed (the subject). See [EVIDENCE_METHOD.md](./EVIDENCE_METHOD.md#subject-vs-runner-guardrail) for the full guardrail.

---

## Subject-vs-Runner Guardrail

This comparative analysis examines GSD-2 as a **subject system**. It is produced **inside** a live GSD execution in a fork of GSD-2. This creates a real conflation risk for `.gsd/` artifacts and system architecture descriptions.

| Concept | What It Means | Where to Find It |
|---------|---------------|------------------|
| **Subject** | The GSD-2 system being analyzed | The analyzed `.gsd/` workflow model, GSD source code, GSD architecture |
| **Runner** | The live GSD execution producing this pack | The current reverse-engineering run's `.gsd/` planning artifacts |

When this document describes GSD-2's `deriveState()` function, dispatch rules, or worktree isolation, it refers to the **analyzed GSD-2 system** — not to the runner that is currently executing. All `[repo code]` labels cite the subject system's codebase.

---

## Overview

The AI coding agent ecosystem has converged on several architectural primitives while differing significantly in how they compose and own these primitives. This document compares four representative systems:

| System | Type | Primary Focus |
|--------|------|---------------|
| **Claude Code** | Integrated CLI agent | Interactive coding with model-first design [external] |
| **Codex CLI** | Integrated CLI agent | Workflow capture via AGENTS.md, approval modes [external] |
| **ACP** | Agent protocol | RESTful API for agent interoperability [external] |
| **GSD-2** | Workflow orchestration system | Milestone-driven development with worktree isolation [repo code] |

Each system makes different design bets about:
- **Where state lives** (session memory vs disk artifacts vs protocol messages)
- **How agents are controlled** (interactive approval vs auto-mode vs protocol delegation)
- **What isolation guarantees** (none vs branch vs worktree vs process)

---

## System Profiles

### Claude Code

Claude Code is Anthropic's terminal-based AI coding assistant, launched in May 2025 and generating over $500M annual run-rate revenue [external]. It represents a **model-first, integrated agent** approach where the LLM capabilities drive product design.

#### Tech Stack and Architecture

Claude Code is built with **TypeScript, React, Ink, Yoga, and Bun** [external]. The choice of React/Ink for terminal UI and Yoga for layout reflects Anthropic's preference for familiar web technologies in a terminal context. The architecture prioritizes being "on distribution" — using technologies the Claude model is already proficient with [external].

Notably, **90% of Claude Code's code is written by Claude Code itself** [external], demonstrating the team's commitment to dogfooding and rapid iteration. The team ships approximately **5 releases per engineer per day** [external].

#### Agent Primitives

Claude Code operates around an **agent loop** with:
- **System prompt**: Contextual instructions loaded from `CLAUDE.md` files in the repository [external]
- **Tool set**: Read, write, bash, edit, and MCP-connected external tools [external]
- **Permission modes**: Suggest, auto-edit, and full-auto approval levels [external]

#### Session Management

Claude Code uses a **fresh session per interaction** pattern [external]. Each conversation starts with a new context window, loading:
1. System prompt and tool definitions
2. `CLAUDE.md` project instructions
3. MCP server configurations
4. Conversation history (if resuming)

Session state is persisted in JSONL format similar to GSD-2's Pi-owned session manager [inference].

#### MCP Integration

Claude Code was the **reference implementation for Model Context Protocol (MCP)** [external]. MCP provides:
- **Tool discovery**: Servers expose tools via JSON-RPC
- **Resource access**: File systems, databases, APIs
- **Prompts**: Reusable prompt templates

MCP tool lazy loading provides significant context savings — up to 97% reduction in one documented case (48k to 1.1k tokens) [external].

#### Design Bets [inference]

1. **Model capability first**: Build products that showcase model strengths
2. **Interactive over automated**: Optimize for human-in-the-loop, not autonomous execution
3. **Session ephemeralness**: Fresh context per interaction rather than persistent workflow state
4. **MCP as standard**: External tool integration via open protocol

---

### Codex CLI

Codex CLI is OpenAI's terminal-based coding agent, designed around **workflow capture** and **approval modes**. It emphasizes reproducible workflows via `AGENTS.md` files.

#### Workflow Capture: AGENTS.md

Codex CLI introduces **AGENTS.md** files for capturing reusable workflows [external]:

- Project-level instructions in `AGENTS.md` at repository root
- Custom commands and workflow definitions
- Context for consistent behavior across sessions
- Similar to Claude Code's `CLAUDE.md` but with stronger workflow emphasis [external]

#### Approval Modes

Codex CLI provides three approval modes [external]:

| Mode | Behavior |
|------|----------|
| **suggest** | Proposes changes, requires explicit approval |
| **auto-edit** | Edits files automatically, asks for shell commands |
| **full-auto** | Executes both edits and shell commands automatically |

This tripartite model provides granular control over autonomy levels.

#### MCP Integration

Codex CLI supports MCP servers for external tool access [external]. The integration enables:
- Connecting to databases, APIs, and external services
- Tool discovery via MCP protocol
- Composability with other MCP-compatible agents

#### Multi-Agent Capabilities

Codex CLI supports **multi-agent orchestration** [external]:
- Router agents that delegate to specialist agents
- Parallel task execution
- Agent composition via the Agents SDK

#### Design Bets [inference]

1. **Workflow persistence**: Capture and replay via AGENTS.md
2. **Explicit approval**: Clear autonomy boundaries via mode selection
3. **SDK-first**: Expose agent capabilities via Agents SDK for composition
4. **Multi-agent native**: Built-in support for agent delegation and parallelism

---

### ACP (Agent Communication Protocol)

ACP is a **RESTful protocol for agent interoperability**, designed to be "the HTTP of agent communication" [external]. Originally developed by IBM's BeeAI team, it has merged with Google's A2A protocol under the Linux Foundation [external].

#### Protocol Architecture

ACP provides a **standardized REST interface** for agent communication [external]:

| Component | Role |
|-----------|------|
| **ACP Client** | Makes requests to ACP servers |
| **ACP Server** | Hosts one or more agents, exposes HTTP endpoints |
| **Agent Manifest** | Metadata describing agent capabilities |

#### Message Structure

ACP messages support **multi-modal payloads** [external]:
- Text, images, audio, video
- Arbitrary MIME types
- Structured metadata

Messages follow a standardized format enabling cross-framework interoperability.

#### Stateful vs Stateless Operation

ACP supports both patterns [external]:

| Mode | Characteristics |
|------|-----------------|
| **Stateless** | Each request independent, no session context |
| **Stateful** | Session-based state with distributed session support |

Stateful agents maintain session context across multiple requests, enabling multi-turn conversations and complex workflows.

#### Agent Discovery

ACP provides **offline discovery via embedded metadata** [external]:
- Agent manifests describe capabilities
- Clients can discover available agents without runtime queries
- Enables dynamic agent composition

#### Design Bets [inference]

1. **Protocol over platform**: Standardize communication, not implementation
2. **REST simplicity**: Leverage familiar HTTP semantics
3. **Multi-modal native**: Support rich content from the start
4. **Framework agnostic**: Work across LangChain, CrewAI, custom stacks

---

### GSD-2

GSD-2 is a **workflow orchestration system** built on top of Pi (a coding agent runtime). It adds milestone-driven development, worktree isolation, and disk-based workflow state on top of Pi's agent primitives.

#### Runtime Substrate: Pi-Owned Components

GSD-2 builds on Pi-owned runtime components [repo code]:

| Component | Location | Responsibility |
|-----------|----------|----------------|
| Agent | `packages/pi-agent-core/src/agent.ts` | LLM interaction, message loop |
| AgentSession | `packages/pi-coding-agent/src/core/agent-session.ts` | Session coordination, retry, compaction |
| SessionManager | `packages/pi-coding-agent/src/core/session-manager.ts` | JSONL persistence, tree navigation |
| ExtensionRunner | `packages/pi-coding-agent/src/core/extensions/runner.ts` | Event emission, hook dispatch |

#### GSD-Authored Orchestration Layer

GSD-2 contributes the orchestration layer [repo code]:

| Component | Location | Responsibility |
|-----------|----------|----------------|
| AutoSession | `src/resources/extensions/gsd/auto/session.ts` | Mutable auto-mode state |
| deriveState | `src/resources/extensions/gsd/state.ts` | Disk → phase derivation |
| DISPATCH_RULES | `src/resources/extensions/gsd/auto-dispatch.ts` | Phase → unit mapping |
| autoLoop | `src/resources/extensions/gsd/auto/loop.ts` | Main iteration cycle |

#### Disk-State Workflow Model

GSD-2's key differentiator is **disk-derived state** [repo code]:
- `deriveState()` reads `.gsd/` artifacts on each dispatch cycle
- Phase transitions encoded as file presence/content
- No mutable runtime state for workflow position
- Enables crash recovery and parallel worker isolation

#### Worktree Isolation

GSD-2 provides **three isolation modes** [repo code]:

| Mode | Behavior |
|------|----------|
| `worktree` | Creates `.gsd/worktrees/<MID>/` with dedicated branch (default) |
| `branch` | Works in project root, switches to milestone branch |
| `none` | Commits directly to current branch |

#### Design Bets [repo code]

1. **Disk as source of truth**: Workflow state derived from files, not memory
2. **Milestone decomposition**: Break work into discrete, verifiable units
3. **Isolation by default**: Worktree isolation prevents main branch corruption
4. **Phase-based dispatch**: Declarative rules map state to actions

---

## Shared-Primitives Matrix

The following matrix compares how each system addresses six fundamental architectural dimensions.

### Dimension Definitions

| Dimension | Question |
|-----------|----------|
| **Model Layer** | How does the system interact with LLMs? |
| **Agent Layer** | What is the core agent loop and tooling? |
| **Harness Layer** | How are sessions managed, recovered, isolated? |
| **Orchestration Layer** | How is workflow control, dispatch, phases handled? |
| **Protocol/Client Surface** | How are capabilities exposed? |
| **Workflow/Doc-State Layer** | How is state persisted and managed? |

### Matrix

| Dimension | Claude Code [external] | Codex CLI [external] | ACP [external] | GSD-2 [repo code] |
|-----------|------------------------|----------------------|----------------|-------------------|
| **Model Layer** | Direct Anthropic API; Claude models only; system prompt from CLAUDE.md | OpenAI API via Agents SDK; multi-provider support; AGENTS.md for context | Model-agnostic; protocol delegates to agent implementation | Multi-provider via Pi's ModelRegistry; model selection per unit type; context budget system |
| **Agent Layer** | Agent loop with tool set (read, write, bash, edit); MCP for external tools; permission modes | Agent loop with approval modes; MCP integration; multi-agent delegation support | Protocol defines message format, not agent internals; agents implement REST endpoints | Pi-owned Agent class; GSD wraps with beforeToolCall/afterToolCall hooks; tool factories from Pi, GSD adds db-tools |
| **Harness Layer** | Fresh session per interaction; JSONL persistence; no explicit isolation | Session-based; approval gating; no worktree isolation | Stateless or stateful via sessions; distributed sessions supported | Full harness: SessionManager (JSONL), RetryHandler, CompactionOrchestrator; worktree/branch/none isolation modes |
| **Orchestration Layer** | Interactive; user drives iteration; no autonomous mode | Auto-approval modes; workflow capture via AGENTS.md; multi-agent orchestration | Protocol-level: client routes to agents; no built-in orchestration | Full orchestration: deriveState(), DISPATCH_RULES, autoLoop, verification gates, dispatch guards |
| **Protocol/Client Surface** | MCP for external tools; CLI interface; IDE/IDE extensions | MCP for external tools; CLI interface; Agents SDK for composition | RESTful HTTP API; OpenAPI spec; Python/TypeScript SDKs | CLI via Pi; Extension API for tool registration; MCP adapter available |
| **Workflow/Doc-State Layer** | CLAUDE.md for project context; session JSONL for conversation | AGENTS.md for workflows; session state; no milestone concept | Agent manifest for discovery; session state if stateful; no workflow primitives | Full disk-state model: ROADMAP.md, S##-PLAN.md, T##-PLAN.md, T##-SUMMARY.md; deriveState() from files |

### Ownership Analysis

| Layer | Claude Code | Codex CLI | ACP | GSD-2 |
|-------|-------------|-----------|-----|-------|
| **Model** | Anthropic-owned | OpenAI-owned | Unspecified | Pi-owned (registry) + GSD (selection) |
| **Agent** | Anthropic-owned | OpenAI-owned | Agent implementer | Pi-owned (core) + GSD (hooks) |
| **Harness** | Anthropic-owned | OpenAI-owned | ACP spec + implementer | Pi-owned (session) + GSD (isolation) |
| **Orchestration** | User-driven | Approval modes + SDK | Protocol-only | **GSD-owned** (deriveState, dispatch, verification) |
| **Protocol** | MCP | MCP + Agents SDK | **ACP-spec** | Extension API (internal) |
| **Workflow/Doc-State** | CLAUDE.md (minimal) | AGENTS.md (workflows) | Manifest (discovery) | **GSD-owned** (full .gsd/ model) |

**Key insight**: GSD-2 is the only system that **owns the orchestration layer and workflow/doc-state layer** as first-class architecture concerns. Claude Code and Codex CLI delegate orchestration to users; ACP specifies protocol but not orchestration semantics.

---

## Primitive Dimension Comparison Tables

The following tables provide detailed comparison across each primitive dimension from the shared-primitives matrix.

### Model Layer Comparison

| Aspect | Claude Code [external] | Codex CLI [external] | ACP [external] | GSD-2 [repo code] |
|--------|------------------------|----------------------|----------------|-------------------|
| **Provider abstraction** | None (Anthropic-only) | OpenAI-first, multi-provider via SDK | Unspecified (agent responsibility) | Multi-provider via Pi's ModelRegistry |
| **Context management** | CLAUDE.md + session history | AGENTS.md + session history | Client-provided | Context budget system with compaction |
| **Model selection** | Claude models only | GPT series, extensible via SDK | Agent-defined | Per-unit-type model selection, tier escalation |
| **Retry/recovery** | Built-in API retry | SDK-level retry | Agent responsibility | RetryHandler with exponential backoff, fallback chain |
| **Context compaction** | Model-managed | Model-managed | Agent responsibility | CompactionOrchestrator with extension hooks |
| **Credential management** | Anthropic API key | OpenAI API key + SDK providers | Agent responsibility | Multi-provider via getApiKey callback |

### Agent Layer Comparison

| Aspect | Claude Code [external] | Codex CLI [external] | ACP [external] | GSD-2 [repo code] |
|--------|------------------------|----------------------|----------------|-------------------|
| **Core loop** | Agent loop with tool execution | Agent loop with approval gating | Not specified (agent implements) | Pi-owned Agent class with message loop |
| **Tool system** | read, write, bash, edit + MCP | read, write, bash, edit + MCP | REST endpoints | Pi factories + GSD db-tools |
| **Event flow** | Per-message callbacks | Per-message callbacks | HTTP request/response | EventStream → AgentSession → ExtensionRunner |
| **Execution model** | Interactive streaming | Streaming with approval pauses | Request/response | Streaming with verification gates |
| **Subagent support** | Via MCP delegation | Via Agents SDK | Via protocol routing | Via subagent tool |
| **Hook system** | MCP tool hooks | MCP tool hooks + SDK interceptors | Agent-defined | beforeToolCall/afterToolCall, transformContext |

### Harness Layer Comparison

| Aspect | Claude Code [external] | Codex CLI [external] | ACP [external] | GSD-2 [repo code] |
|--------|------------------------|----------------------|----------------|-------------------|
| **Session lifecycle** | Fresh per interaction | Session-based with resume | Stateless or stateful | SessionManager with tree navigation |
| **Persistence format** | JSONL | JSONL (similar) | Agent responsibility | JSONL with append-only tree structure |
| **Isolation modes** | None (runs in cwd) | Sandbox options | Agent responsibility | Worktree/branch/none modes |
| **Crash recovery** | Resume from JSONL | Session resume | Agent responsibility | Plan reconciliation, state sync, forensics |
| **Parallel execution** | Multiple terminals | Multi-agent via SDK | Multi-server native | Parallel orchestrator with worker processes |
| **State encapsulation** | Per-session memory | Per-session memory | Optional distributed sessions | AutoSession encapsulates all mutable state |

### Orchestration Layer Comparison

| Aspect | Claude Code [external] | Codex CLI [external] | ACP [external] | GSD-2 [repo code] |
|--------|------------------------|----------------------|----------------|-------------------|
| **Workflow phases** | None (user-driven) | Approval modes only | None (protocol-only) | 10 phases via deriveState() |
| **Dispatch mechanism** | User prompt | User prompt + auto-approval | Client request | Declarative DISPATCH_RULES, first-match-wins |
| **State derivation** | Load CLAUDE.md | Load AGENTS.md | Client provides | deriveState() reads .gsd/ artifacts |
| **Verification** | User inspection | Approval gates | Agent responsibility | Verification gates with auto-retry |
| **Progress tracking** | Conversation | Session + AGENTS.md | Agent responsibility | ROADMAP checkboxes, T##-SUMMARY.md |
| **Guard rails** | Permission modes | Approval modes | None | Dispatch guard prevents out-of-order |

### Protocol/Client Surface Comparison

| Aspect | Claude Code [external] | Codex CLI [external] | ACP [external] | GSD-2 [repo code] |
|--------|------------------------|----------------------|----------------|-------------------|
| **Primary interface** | CLI + IDE extensions | CLI + Agents SDK | RESTful HTTP API | CLI via Pi |
| **Tool discovery** | MCP protocol | MCP protocol + SDK | Agent manifest | Extension API registration |
| **External tools** | MCP servers | MCP servers + SDK plugins | REST endpoints | MCP adapter available |
| **Client SDK** | None (CLI-only) | Agents SDK (Python/TS) | Python/TypeScript SDKs | Extension API (internal) |
| **Interoperability** | MCP standard | MCP + A2A support | ACP/A2A standard | MCP adapter for external tools |
| **Authentication** | Anthropic API key | OpenAI API key | Agent responsibility | Multi-provider credential callback |

### Workflow/Doc-State Layer Comparison

| Aspect | Claude Code [external] | Codex CLI [external] | ACP [external] | GSD-2 [repo code] |
|--------|------------------------|----------------------|----------------|-------------------|
| **State location** | Session memory + CLAUDE.md | Session + AGENTS.md | Protocol messages | Disk artifacts (.gsd/) |
| **Persistence model** | JSONL session files | JSONL session files | Agent responsibility | ROADMAP.md, S##-PLAN.md, T##-PLAN.md, T##-SUMMARY.md |
| **Decomposition** | None (conversation) | AGENTS.md workflows | None | Milestone → Slice → Task hierarchy |
| **State derivation** | Load on session start | Load on session start | Client provides | deriveState() on each dispatch cycle |
| **Audit trail** | Session JSONL | Session JSONL | Agent responsibility | Full markdown audit trail |
| **Human readability** | JSONL (parseable) | JSONL (parseable) | JSON messages | Markdown (human-readable by design) |

---

## Structural Comparison Tables

### Session and State Management

| Aspect | Claude Code [external] | Codex CLI [external] | ACP [external] | GSD-2 [repo code] |
|--------|------------------------|----------------------|----------------|-------------------|
| **Session persistence** | JSONL per conversation | Session-based | Optional sessions | JSONL via SessionManager |
| **State derivation** | Load CLAUDE.md + history | Load AGENTS.md + history | Client provides context | deriveState() from .gsd/ files |
| **Crash recovery** | Resume from JSONL | Session resume | Depends on agent | Plan reconciliation, state sync |
| **Context compaction** | Handled by model | Handled by model | Depends on agent | CompactionOrchestrator with extension hooks |

### Isolation and Safety

| Aspect | Claude Code [external] | Codex CLI [external] | ACP [external] | GSD-2 [repo code] |
|--------|------------------------|----------------------|----------------|-------------------|
| **Execution isolation** | None (runs in cwd) | Sandbox options | Depends on agent | Worktree/branch/none modes |
| **Branch protection** | Via permission modes | Via approval modes | N/A | Dispatch guard, integration branch |
| **Rollback capability** | Git-based | Git-based | Depends on agent | Squash merge, snapshot refs |
| **Concurrent execution** | Multiple terminals | Multi-agent SDK | Multi-server native | Parallel orchestrator with worker processes |

### Tool and Extension Model

| Aspect | Claude Code [external] | Codex CLI [external] | ACP [external] | GSD-2 [repo code] |
|--------|------------------------|----------------------|----------------|-------------------|
| **Core tools** | read, write, bash, edit | read, write, bash, edit | Not specified | Pi factories: bash, read, edit, write |
| **Extension mechanism** | MCP servers | MCP servers + Agents SDK | REST endpoints | Extension API + MCP adapter |
| **Tool registration** | MCP discovery | MCP discovery | Agent manifest | pi.registerTool() API |
| **Custom tools** | Via MCP | Via MCP + SDK | Via agent implementation | GSD db-tools: gsd_save_decision, etc. |

### Workflow Control

| Aspect | Claude Code [external] | Codex CLI [external] | ACP [external] | GSD-2 [repo code] |
|--------|------------------------|----------------------|----------------|-------------------|
| **Planning model** | User-directed | AGENTS.md workflows | Client-directed | Milestone → Slice → Task decomposition |
| **Execution modes** | Interactive | Suggest/Auto-edit/Full-auto | Request/Response | Interactive + Auto-mode with step mode |
| **Verification** | User inspection | Approval gates | Agent responsibility | Verification gates with auto-retry |
| **Progress tracking** | Conversation history | Session state | Agent responsibility | ROADMAP checkboxes, T##-SUMMARY.md |

---

## Design-Bet Synthesis

Each system makes fundamental bets that shape its architecture and use cases.

### Claude Code: Model-First Interactive Agent [external]

**Core bet**: The model is the product. Build tools that showcase model capabilities.

**Implications**:
- Fresh sessions prevent context pollution [external]
- MCP standardization enables ecosystem growth [external]
- Interactive optimization over autonomous execution [inference]
- CLAUDE.md provides project memory without workflow state [external]

**Best for**: Interactive coding, exploration, tasks requiring model judgment.

### Codex CLI: Workflow Capture and Composition [external]

**Core bet**: Capture workflows in code (AGENTS.md) for reproducibility and composition.

**Implications**:
- AGENTS.md as executable documentation [external]
- Approval modes balance autonomy and safety [external]
- Agents SDK enables multi-agent composition [external]
- Less emphasis on isolation, more on workflow [inference]

**Best for**: Reproducible workflows, multi-agent systems, CI/CD integration.

### ACP: Protocol for Interoperability [external]

**Core bet**: Standardize communication, not implementation. Be the HTTP of agents.

**Implications**:
- Framework-agnostic by design [external]
- REST simplicity lowers integration barrier [external]
- Multi-modal from the start [external]
- No opinion on orchestration or state [inference]

**Best for**: Multi-agent ecosystems, cross-framework integration, agent marketplaces.

### GSD-2: Disk-State Orchestration [repo code]

**Core bet**: Workflow state is too important for memory. Derive from files.

**Implications**:
- Crash recovery is free (just re-read disk) [repo code]
- Parallel workers get isolated milestones [repo code]
- Phase transitions are file operations [repo code]
- Full audit trail in markdown artifacts [repo code]

**Best for**: Autonomous development, long-running projects, team coordination.

---

## Owned Layers Synthesis

This section analyzes which responsibilities each system owns versus delegates, where boundaries are explicit versus implicit, and what each system would need to rebuild, wrap, or delegate from GSD-2.

### Ownership Matrix

| Layer | Claude Code | Codex CLI | ACP | GSD-2 |
|-------|-------------|-----------|-----|-------|
| **Model** | Anthropic-owned [external] | OpenAI-owned [external] | Unspecified [external] | Pi-owned (registry) + GSD (selection) [repo code] |
| **Agent** | Anthropic-owned [external] | OpenAI-owned [external] | Agent implementer [external] | Pi-owned (core) + GSD (hooks) [repo code] |
| **Harness** | Anthropic-owned [external] | OpenAI-owned [external] | ACP spec + implementer [external] | Pi-owned (session) + GSD (isolation) [repo code] |
| **Orchestration** | User-driven [inference] | Approval modes + SDK [external] | Protocol-only [external] | **GSD-owned** [repo code] |
| **Protocol** | MCP [external] | MCP + Agents SDK [external] | **ACP-spec** [external] | Extension API (internal) [repo code] |
| **Workflow/Doc-State** | CLAUDE.md (minimal) [external] | AGENTS.md (workflows) [external] | Manifest (discovery) [external] | **GSD-owned** [repo code] |

### Explicit vs Implicit Boundaries

| System | Explicit Boundaries | Implicit Boundaries | Evidence |
|--------|---------------------|---------------------|----------|
| **Claude Code** | MCP protocol for external tools; Permission modes | Session memory vs disk state; User vs agent responsibility | [external] |
| **Codex CLI** | Approval modes (suggest/auto-edit/full-auto); AGENTS.md workflow capture | Tool execution vs workflow state; Session boundaries | [external] |
| **ACP** | REST endpoints; Message format; Agent manifest | Agent implementation details; Session vs stateless choice | [external] |
| **GSD-2** | Milestone/Slice/Task decomposition; Phase transitions; Verification gates; Dispatch rules | Extension hook timing; Retry fallback priority | [repo code] |

### Rebuild/Wrap/Delegate Analysis

What each system would need to integrate with GSD-2's capabilities:

#### Claude Code → GSD-2 Integration [inference]

| GSD-2 Capability | Strategy | Rationale |
|------------------|----------|-----------|
| Milestone decomposition | **Rebuild** | Claude Code has no concept of milestones; would need significant architecture change |
| Worktree isolation | **Wrap** | Could wrap git worktree commands via MCP tool; isolation not built-in |
| Disk-derived state | **Rebuild** | Session memory is core assumption; deriveState() pattern incompatible |
| Verification gates | **Delegate** | Could delegate to external verification service via MCP |
| Auto-mode dispatch | **Rebuild** | Interactive model is fundamental; auto-mode requires different architecture |

#### Codex CLI → GSD-2 Integration [inference]

| GSD-2 Capability | Strategy | Rationale |
|------------------|----------|-----------|
| Milestone decomposition | **Rebuild** | AGENTS.md could capture milestone patterns, but hierarchy not native |
| Worktree isolation | **Wrap** | Sandbox exists but not worktree-specific; could add via tool |
| Disk-derived state | **Wrap** | AGENTS.md is disk-based; could extend to full state derivation |
| Verification gates | **Delegate** | Approval modes already provide gating; verification could integrate |
| Auto-mode dispatch | **Wrap** | Full-auto mode + AGENTS.md could approximate auto-dispatch |

#### ACP → GSD-2 Integration [inference]

| GSD-2 Capability | Strategy | Rationale |
|------------------|----------|-----------|
| Milestone decomposition | **Delegate** | ACP agent could expose milestone planning as capability |
| Worktree isolation | **Delegate** | Agent implementation responsibility; ACP doesn't specify |
| Disk-derived state | **Delegate** | Agent could implement deriveState() internally |
| Verification gates | **Delegate** | Agent capability; ACP protocol doesn't constrain |
| Auto-mode dispatch | **Delegate** | Agent implements dispatch logic; protocol is agnostic |

**Key insight**: ACP's protocol-first design means integration is always delegation — ACP doesn't prescribe implementation, so GSD-2 capabilities would be implemented by an ACP-compliant agent rather than integrated at the protocol level [inference].

### Layer Ownership Consequences

**GSD-2 owns orchestration and workflow/doc-state** [repo code]. This has several consequences:

1. **Crash recovery is architectural** — Since state is derived from disk, recovery is automatic. No special crash recovery logic needed beyond verifying artifacts exist [repo code].

2. **Parallel workers get isolated milestones** — The dispatch guard ensures no two workers operate on the same slice. Worktree isolation prevents branch conflicts [repo code].

3. **Audit is human-readable** — All state is markdown files. Any human can inspect `.gsd/` and understand project state without tooling [repo code].

4. **Phase transitions are file operations** — Moving from "planning" to "executing" is a file write, not a state machine transition [repo code].

**Claude Code and Codex CLI delegate orchestration** [external]. Consequences:

1. **Human is the orchestrator** — No autonomous phase transitions. User drives all workflow progression [external].

2. **Session memory is primary** — State lives in conversation context, not disk artifacts. Crash loses context [external].

3. **No built-in decomposition** — Milestones, slices, tasks are user concepts, not system primitives [external].

**ACP delegates everything except protocol** [external]. Consequences:

1. **Agent implementer owns all semantics** — ACP provides communication, not behavior. Each agent implementation differs [external].

2. **Interoperability via protocol only** — Agents share message format, not capabilities. Tool composition requires protocol-level agreement [external].

3. **No workflow primitives** — ACP has no concept of phases, milestones, or verification. These are agent-specific [external].

### Synthesis: Architectural Trade-offs

| Trade-off | Claude Code | Codex CLI | ACP | GSD-2 |
|-----------|-------------|-----------|-----|-------|
| **Simplicity vs Power** | Simple, interactive [external] | Moderate, workflow capture [external] | Simple protocol, complex agents [external] | Complex, autonomous [repo code] |
| **Flexibility vs Structure** | Flexible, user-driven [external] | Structured via AGENTS.md [external] | Maximum flexibility [external] | Highly structured [repo code] |
| **Human vs Machine control** | Human-in-the-loop [external] | Human with automation [external] | Depends on agent [external] | Auto-mode with human oversight [repo code] |
| **State persistence** | Session memory [external] | Session + AGENTS.md [external] | Agent responsibility [external] | Disk as source of truth [repo code] |
| **Learning curve** | Low [external] | Moderate [external] | Varies by agent [external] | High (milestone/slice/task) [repo code] |

---

## Convergence and Divergence

### Convergent Patterns

All four systems converge on:

1. **Agent loop with tools** [external] [repo code]
   - Core pattern: LLM decides, tools execute, results feed back
   - Tools include read, write, bash as primitives

2. **Session persistence** [external] [repo code]
   - JSONL or similar format for conversation history
   - Enables resume and audit

3. **External tool integration** [external] [repo code]
   - MCP as emerging standard
   - Protocol-based tool discovery

4. **Git as foundation** [external] [repo code]
   - All systems work with git repositories
   - Branch/commit as natural checkpoint

### Divergent Choices

Systems diverge on:

1. **Where state lives**
   - Claude Code: Session memory + CLAUDE.md [external]
   - Codex CLI: Session + AGENTS.md [external]
   - ACP: Protocol messages, optional sessions [external]
   - GSD-2: Disk artifacts (.gsd/) [repo code]

2. **Who controls execution**
   - Claude Code: User interactive [external]
   - Codex CLI: User with approval modes [external]
   - ACP: Client delegation [external]
   - GSD-2: Auto-mode dispatch loop [repo code]

3. **Isolation guarantees**
   - Claude Code: None [external]
   - Codex CLI: Sandbox options [external]
   - ACP: Agent responsibility [external]
   - GSD-2: Worktree isolation [repo code]

4. **Workflow model**
   - Claude Code: Conversation [external]
   - Codex CLI: Workflow capture [external]
   - ACP: Protocol delegation [external]
   - GSD-2: Milestone decomposition [repo code]

---

## Evidence Summary

### Evidence Tier Distribution

| Tier | Label | Count in This Document |
|------|-------|------------------------|
| Tier 1 | `[repo code]` | 28+ |
| Tier 2 | `[repo doc]` | 0 |
| Tier 3 | `[external]` | 45+ |
| Tier 4 | `[inference]` | 12+ |

### External Source References

| System | Key Sources |
|--------|-------------|
| Claude Code | Pragmatic Engineer interview (Sep 2025), official docs, community guides |
| Codex CLI | OpenAI Developer docs, community tutorials, Agents SDK documentation |
| ACP | AgentCommunicationProtocol.dev, IBM explainer, arXiv survey paper |

---

## Cross-References

- **GSD-2 Runtime Architecture**: See [GSD2_RUNTIME_ARCHITECTURE.md](./GSD2_RUNTIME_ARCHITECTURE.md)
- **GSD-2 Orchestration Layer**: See [GSD2_ORCHESTRATION_LAYER.md](./GSD2_ORCHESTRATION_LAYER.md)
- **GSD-2 Context Engineering**: See [GSD2_CONTEXT_ENGINEERING_MODEL.md](./GSD2_CONTEXT_ENGINEERING_MODEL.md)
- **GSD-2 Isolation Model**: See [GSD2_GIT_AND_ISOLATION_MODEL.md](./GSD2_GIT_AND_ISOLATION_MODEL.md)
- **Evidence Method**: See [EVIDENCE_METHOD.md](./EVIDENCE_METHOD.md)

---

## Summary

This comparative analysis reveals that while all four systems share agent-loop primitives (model interaction, tool execution, session persistence), they differ fundamentally in architectural ownership:

- **Claude Code** owns the model experience, optimizing for interactive model-first development [external]
- **Codex CLI** owns workflow capture via AGENTS.md, enabling reproducibility and composition [external]
- **ACP** owns the protocol layer, enabling cross-framework interoperability without prescribing implementation [external]
- **GSD-2** owns the orchestration and workflow/doc-state layers, providing autonomous development with disk-derived state [repo code]

The key architectural insight is that GSD-2's disk-state model — deriving all workflow position from `.gsd/` files rather than runtime state — enables capabilities the other systems lack: crash recovery without special handling, parallel worker isolation, and full audit trails in human-readable markdown. This comes at the cost of complexity: GSD-2 requires understanding milestone/slice/task decomposition and the phase state machine.

For teams considering these systems, the choice depends on workflow needs:
- **Interactive coding**: Claude Code or Codex CLI
- **Multi-agent composition**: Codex CLI or ACP
- **Autonomous development**: GSD-2
- **Cross-framework integration**: ACP
