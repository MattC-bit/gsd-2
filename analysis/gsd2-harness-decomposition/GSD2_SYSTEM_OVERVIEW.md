# GSD-2 System Overview

This document provides the primary entry point for understanding GSD-2 as a harness/orchestration system. It establishes the initial Pi-vs-GSD boundary inventory with specific repo file references and sets expectations for the rest of the decomposition pack.

---

## What Is GSD-2

GSD-2 is a **harness and orchestration system** for coding agents, not a product feature roadmap [repo doc]. It is built as a standalone CLI on the Pi SDK, which provides direct TypeScript access to the agent runtime layer. This architecture allows GSD to control context windows, session lifecycle, git branches, cost tracking, crash recovery, and autonomous milestone progression — capabilities that a pure prompt framework cannot achieve [repo code].

The system's core value proposition is captured in the README: "One command. Walk away. Come back to a built project with clean git history" [repo doc]. This reflects the orchestration-focused design: GSD manages the entire execution lifecycle from planning through verification, persisting state on disk to enable crash recovery and multi-terminal coordination.

**Per Decision D001**, this pack treats GSD-2 as a harness/orchestration system. The goal is reverse-engineering the control architecture and workflow model, not planning new product features.

---

## Repo Structure Overview

The GSD-2 repository is a TypeScript workspace organized into distinct ownership layers:

### `packages/` — Pi-Owned Runtime Packages

Five packages provide reusable runtime capabilities that are independent of GSD-specific orchestration:

| Package | Purpose |
|---------|---------|
| `pi-agent-core` | Core agent loop, message types, tool interfaces |
| `pi-ai` | LLM provider integrations, streaming, model registry |
| `pi-coding-agent` | CLI framework, session management, extension system |
| `pi-tui` | Terminal UI components, keybindings, rendering |
| `native` | Rust N-API addon for performance-critical operations |

These packages form the Pi SDK, which could theoretically power other agent systems beyond GSD [inference].

### `src/` — GSD-Specific Entry Points and Resource Loading

The `src/` directory contains GSD-specific code that wires the Pi packages into the GSD CLI:

| File/Directory | Responsibility |
|----------------|----------------|
| `cli.ts` | Main entry point, argument parsing, SDK wiring |
| `loader.ts` | Environment setup, dynamic imports, version handling |
| `headless.ts` | Headless/RPC mode for CI orchestration |
| `web-mode.ts` | Browser-based web interface |
| `worktree-cli.ts` | Worktree lifecycle management from CLI |
| `resource-loader.ts` | Syncs bundled resources to `~/.gsd/agent/` |
| `extension-registry.ts` | Extension discovery and loading |

The **two-file loader pattern** is notable: `loader.ts` sets all environment variables with zero SDK imports, then dynamically imports `cli.ts` which does static SDK imports. This ensures `PI_PACKAGE_DIR` is set before any SDK code evaluates [repo code].

### `src/resources/extensions/gsd/` — GSD-Authored Orchestration Layer

The core GSD extension contains **154 TypeScript files** implementing the workflow engine, auto-mode state machine, and project management primitives [repo code]. Key modules include:

- **Auto-mode dispatch**: `auto.ts`, `auto-dispatch.ts`, `auto-start.ts`, `auto-recovery.ts`
- **Worktree/git management**: `auto-worktree.ts`, `git-service.ts`, `worktree-manager.ts`
- **Prompt assembly**: `auto-prompts.ts`, `prompt-loader.ts`
- **Verification**: `auto-verification.ts`
- **Commands**: `commands-*.ts` files for GSD slash commands

This directory represents the primary focus of the decomposition pack — the orchestration logic that transforms Pi's runtime capabilities into a workflow-driven coding agent.

### `docs/` — Secondary Explanatory Material

The `docs/` directory contains architecture overviews, command references, and guides. These documents are secondary sources that may lag behind or diverge from actual code behavior [repo doc]. Per Decision D002, they are treated as supporting evidence but never override repo code.

---

## Pi-vs-GSD Boundary Inventory

This section documents the ownership boundary between Pi-owned runtime modules and GSD-authored orchestration modules. Each entry includes specific file references for verification.

### Pi-Owned Runtime Modules

These modules provide generic agent capabilities that could serve any agent system:

#### `packages/pi-agent-core` — Agent Loop and Message Types

**Responsibilities**: Core agent loop implementation, message type definitions, tool interfaces, context transformation hooks.

**Key files**:
- `packages/pi-agent-core/src/agent.ts` — Agent class using agent-loop directly, no transport abstraction [repo code]
- `packages/pi-agent-core/src/agent-loop.ts` — Core loop logic for streaming LLM interactions [repo code]
- `packages/pi-agent-core/src/types.ts` — Type definitions for AgentContext, AgentState, AgentTool [repo code]

The Agent class provides hooks for `transformContext`, `convertToLlm`, and `beforeToolCall`/`afterToolCall` callbacks that orchestration layers can use to inject behavior [repo code].

#### `packages/pi-ai` — LLM Provider Integrations

**Responsibilities**: Model registry, provider implementations (Anthropic, OpenAI, Google, etc.), streaming abstractions, authentication.

**Key files**:
- `packages/pi-ai/src/models.ts` — Model definitions and capabilities [repo code]
- `packages/pi-ai/src/stream.ts` — Streaming abstraction for LLM responses [repo code]
- `packages/pi-ai/src/providers/anthropic.ts` — Anthropic Claude provider [repo code]
- `packages/pi-ai/src/providers/openai-responses.ts` — OpenAI provider [repo code]

Providers are lazy-loaded on first use to reduce cold-start time — only the connected provider's SDK is loaded [repo code].

#### `packages/pi-coding-agent` — CLI Framework and Extension System

**Responsibilities**: Session management, extension loading, tool definitions, settings storage, interactive mode.

**Key files**:
- `packages/pi-coding-agent/src/core/sdk.ts` — `createAgentSession()` factory, session options [repo code]
- `packages/pi-coding-agent/src/core/extensions/runner.ts` — ExtensionRunner lifecycle management [repo code]
- `packages/pi-coding-agent/src/core/session-manager.ts` — Session persistence and resumption [repo code]
- `packages/pi-coding-agent/src/core/tools/index.ts` — Tool definitions (bash, read, write, edit, etc.) [repo code]

The extension system provides hooks for `agentStart`, `agentEnd`, `beforeToolCall`, `afterToolCall`, `newSession`, and custom slash commands [repo code].

#### `packages/pi-tui` — Terminal UI Components

**Responsibilities**: Differential rendering, component framework, keybindings, input handling.

**Key files**:
- `packages/pi-tui/src/tui.ts` — Minimal TUI implementation with differential rendering [repo code]
- `packages/pi-tui/src/terminal.ts` — Terminal abstraction and capabilities detection [repo code]
- `packages/pi-tui/src/keybindings.ts` — Keybinding management [repo code]

Components implement a `render(width: number): string[]` interface and optionally `handleInput(data: string)` [repo code].

#### `packages/native` — Rust N-API Performance Addon

**Responsibilities**: High-performance operations — grep, glob, syntax highlighting, AST operations, clipboard, TTSR pattern matching.

**Key files**:
- `packages/native/src/native.ts` — Native addon loader with platform-specific resolution [repo code]
- `packages/native/src/grep/` — Native grep implementation [repo code]
- `packages/native/src/ast/` — AST-based code search and editing [repo code]
- `packages/native/src/gsd-parser/` — GSD file parsing (roadmaps, plans, etc.) [repo code]

The native addon gracefully falls back to JS implementations on unsupported platforms [repo code].

### GSD-Authored Orchestration Modules

These modules implement GSD-specific workflow logic using Pi's extension hooks:

#### Auto-Mode State Machine

**Responsibilities**: Dispatch loop, unit selection, phase transitions, recovery, verification.

**Key files**:
- `src/resources/extensions/gsd/auto.ts` — Main auto-mode state machine, fresh session per unit pattern [repo code]
- `src/resources/extensions/gsd/auto-dispatch.ts` — Unit selection logic based on `.gsd/` state [repo code]
- `src/resources/extensions/gsd/auto-start.ts` — Auto-mode initialization and startup sequence [repo code]
- `src/resources/extensions/gsd/auto-recovery.ts` — Crash recovery and session forensics [repo code]

The state machine reads `.gsd/` files after each `agent_end` event, determines the next unit type, creates a fresh session, and injects a focused prompt [repo code].

#### Worktree and Git Management

**Responsibilities**: Worktree lifecycle, branch management, merge operations, isolation enforcement.

**Key files**:
- `src/resources/extensions/gsd/auto-worktree.ts` — Worktree creation, lifecycle, and isolation logic [repo code]
- `src/resources/extensions/gsd/worktree-manager.ts` — Worktree CRUD operations [repo code]
- `src/resources/extensions/gsd/git-service.ts` — Git operations abstraction [repo code]
- `src/resources/extensions/gsd/native-git-bridge.ts` — Native git operations via Rust addon [repo code]

Worktrees provide execution isolation per milestone; the `-w` flag creates auto-named worktrees for ad-hoc sessions [repo code].

#### Prompt Assembly and Context Engineering

**Responsibilities**: Template loading, file inlining, context budgeting, prompt construction for each unit type.

**Key files**:
- `src/resources/extensions/gsd/auto-prompts.ts` — Prompt builders for plan, execute, verify units [repo code]
- `src/resources/extensions/gsd/context-budget.ts` — Context window budgeting for inline content [repo code]
- `src/resources/extensions/gsd/prompt-loader.ts` — Template loading and inlining [repo code]

Prompt builders are pure async functions that load templates and inline file content with no module-level state [repo code].

#### Verification and Guards

**Responsibilities**: Task verification, write gates, tool-call loop detection, hallucination guards.

**Key files**:
- `src/resources/extensions/gsd/auto-verification.ts` — Verification logic for completed units [repo code]
- `src/resources/extensions/gsd/bootstrap/write-gate.ts` — Write gate for protecting concurrent modifications [repo code]
- `src/resources/extensions/gsd/bootstrap/tool-call-loop-guard.ts` — Infinite loop detection [repo code]

The hallucination guard rejects execute-task completions with zero tool calls as fabricated [repo code].

#### Extension Registration

**Responsibilities**: Wiring GSD commands, hooks, and shortcuts into Pi's extension API.

**Key files**:
- `src/resources/extensions/gsd/bootstrap/register-extension.ts` — Main extension registration [repo code]
- `src/resources/extensions/gsd/bootstrap/register-hooks.ts` — Lifecycle event hooks [repo code]
- `src/resources/extensions/gsd/bootstrap/register-shortcuts.ts` — Keyboard shortcuts [repo code]

Registration happens once at startup via `registerGsdExtension(pi)` [repo code].

### Unresolved Seams

The following ownership boundaries are unclear or shared between Pi and GSD:

1. **Session lifecycle ownership** `[unresolved]`: The `pi-coding-agent` package provides `createAgentSession()` and `SessionManager`, but GSD's auto-mode manages fresh session creation via `ctx.newSession()`. The exact division of responsibility for session persistence, recovery, and cleanup is not clearly documented. Does GSD wrap Pi's session management or replace it?

2. **Tool definition boundaries** `[RESOLVED — S03/T03]`: ~~Pi provides tool definitions in `packages/pi-coding-agent/src/core/tools/`, but GSD registers dynamic tools and database tools via `registerDynamicTools()` and `registerDbTools()`. The relationship between Pi-provided tools and GSD-added tools is implicit in the code rather than explicitly documented.~~ **Resolution**: See [GSD2_ORCHESTRATION_LAYER.md#seam-2-resolution-tool-definition-boundaries](./GSD2_ORCHESTRATION_LAYER.md#seam-2-resolution-tool-definition-boundaries) — Pi owns tool factories (`createBashTool`, `createReadTool`, etc.), GSD wraps them with additional behavior and defines entirely GSD-owned tools (`gsd_save_decision`, etc.), and both are registered via Pi's `registerTool()` API.

3. **State persistence model** `[unresolved]`: Pi provides `SettingsManager` and `AuthStorage` for user preferences, but GSD's entire workflow state lives in `.gsd/` on disk. Whether Pi expects extensions to manage their own state or should use Pi's storage abstractions is unclear.

4. **Extension event semantics** `[unresolved]`: Pi's extension system defines events like `agentStart`, `agentEnd`, and `beforeToolCall`, but GSD's use of these events for auto-mode dispatch relies on undocumented timing assumptions. Whether Pi guarantees event ordering and whether extensions can safely mutate state during callbacks needs verification.

---

## Subject-vs-Runner Guardrail

> **Critical**: This pack analyzes GSD-2 as a *subject system*. It is produced *inside* a live GSD run in a fork of GSD-2. This creates a real conflation risk. See the [subject-vs-runner guardrail](./EVIDENCE_METHOD.md#subject-vs-runner-guardrail) in EVIDENCE_METHOD.md for the canonical definition.

### The Distinction

| Concept | What It Means | Where to Find It |
|---------|---------------|------------------|
| **Subject** | The GSD-2 system being analyzed | The analyzed `.gsd/` workflow model, GSD source code, GSD architecture |
| **Runner** | The live GSD execution producing this pack | The current reverse-engineering run's `.gsd/` planning artifacts |

### Concrete Example

The analyzed GSD-2 stores workflow state in `.gsd/milestones/` directories containing ROADMAP.md, CONTEXT.md, SLICE-PLAN.md files [repo code]. Each milestone has its own worktree under `.gsd/worktrees/` for execution isolation [repo code].

**This reverse-engineering run** (the runner) has its own `.gsd/` artifacts that are producing this documentation:
- The planning artifacts are in `.gsd/worktrees/M001/.gsd/milestones/M001/` [repo doc]
- The ROADMAP.md defines slices S01-S07 for the decomposition pack [repo doc]
- The slice plans define tasks T01, T02, etc. for each analytical step [repo doc]

These runner artifacts are **not the subject of analysis**. When this pack discusses "the `.gsd/` model" or "milestone lifecycle," it refers to the analyzed GSD-2's architecture — not the live planning state that produced this document.

**Per Decision D007**, references to `.gsd/`, milestones, slices, and workflow state in this pack refer to the **analyzed GSD-2 system** unless explicitly noted otherwise.

---

## Pack Document Map

This document is the entry point for the GSD-2 Harness Decomposition Pack. Subsequent documents provide deeper analysis:

| Document | Slice | Purpose |
|----------|-------|---------|
| [EVIDENCE_METHOD.md](./EVIDENCE_METHOD.md) | S01 | Methodological contract for evidence labeling |
| [GSD2_RUNTIME_ARCHITECTURE.md](./GSD2_RUNTIME_ARCHITECTURE.md) | S02 | Session/runtime assembly, lifecycle, event flow, persistence |
| [GSD2_ORCHESTRATION_LAYER.md](./GSD2_ORCHESTRATION_LAYER.md) | S03 | Dispatch, auto-mode control, phases, verification, guards |
| [GSD2_CONTEXT_ENGINEERING_MODEL.md](./GSD2_CONTEXT_ENGINEERING_MODEL.md) | S04 | Prompt assembly, `.gsd/` workflow-state model |
| [GSD2_GIT_AND_ISOLATION_MODEL.md](./GSD2_GIT_AND_ISOLATION_MODEL.md) | S05 | Worktree isolation, recovery, execution boundaries |
| [GSD2_COMPARATIVE_ANALYSIS.md](./GSD2_COMPARATIVE_ANALYSIS.md) | S06 | Structural comparison with Claude Code, Codex, ACP |
| [GLOSSARY_NORMALIZED_TERMS.md](./GLOSSARY_NORMALIZED_TERMS.md) | S07 | GSD-native terms with Atlas crosswalks |
| [EMERGING_AGENT_HARNESS_MODEL.md](./EMERGING_AGENT_HARNESS_MODEL.md) | S07 | Rebuild/wrap/delegate synthesis for Atlas harness |

The reading order for maximum comprehension:
1. This document ([GSD2_SYSTEM_OVERVIEW.md](./GSD2_SYSTEM_OVERVIEW.md))
2. [EVIDENCE_METHOD.md](./EVIDENCE_METHOD.md)
3. [GSD2_RUNTIME_ARCHITECTURE.md](./GSD2_RUNTIME_ARCHITECTURE.md)
4. [GSD2_ORCHESTRATION_LAYER.md](./GSD2_ORCHESTRATION_LAYER.md)
5. [GSD2_CONTEXT_ENGINEERING_MODEL.md](./GSD2_CONTEXT_ENGINEERING_MODEL.md)
6. [GSD2_GIT_AND_ISOLATION_MODEL.md](./GSD2_GIT_AND_ISOLATION_MODEL.md)
7. [GSD2_COMPARATIVE_ANALYSIS.md](./GSD2_COMPARATIVE_ANALYSIS.md)
8. [GLOSSARY_NORMALIZED_TERMS.md](./GLOSSARY_NORMALIZED_TERMS.md)
9. [EMERGING_AGENT_HARNESS_MODEL.md](./EMERGING_AGENT_HARNESS_MODEL.md)

---

## Summary

GSD-2 is an orchestration layer built on Pi SDK runtime capabilities. The Pi packages (`pi-agent-core`, `pi-ai`, `pi-coding-agent`, `pi-tui`, `native`) provide generic agent infrastructure: agent loop, LLM providers, CLI framework, TUI components, and native performance primitives. The GSD extension layer (`src/resources/extensions/gsd/`) adds workflow-specific orchestration: auto-mode state machine, worktree isolation, prompt assembly, and verification guards.

The boundary between Pi and GSD is generally clear at the package level but has unresolved seams in session lifecycle, tool ownership, state persistence, and extension event semantics. Subsequent documents in this pack will trace these seams in detail.

All references to `.gsd/` in this pack denote the **analyzed GSD-2 system**, not the live runner state producing this documentation.
