# GSD-2 Runtime Architecture

This document synthesizes findings from the runtime investigation to provide an authoritative reference for GSD-2's runtime layer: how components are wired, how sessions flow, how events propagate, and how state persists.

---

> **[D007]** Throughout this document, references to `.gsd/`, milestones, slices, and workflow state refer to the **analyzed GSD-2 system** unless explicitly noted otherwise. This document is produced *inside* a live GSD run in a fork of GSD-2. The `.gsd/` artifacts producing this analysis (the runner) are separate from the `.gsd/` model being analyzed (the subject). See [EVIDENCE_METHOD.md](./EVIDENCE_METHOD.md#subject-vs-runner-guardrail) for the full guardrail.

---

## Overview

The GSD-2 runtime layer is responsible for:

1. **Runtime Assembly** — Wiring together Agent, AgentSession, SessionManager, and extension subsystems
2. **Session Lifecycle** — Creating, switching, forking, and navigating session trees
3. **Event Flow** — Propagating events from Agent through AgentSession to ExtensionRunner
4. **State Persistence** — Storing conversation history in JSONL format with append-only tree structure
5. **Error Recovery** — Automatic retry with backoff, context compaction, and crash recovery

The runtime is **Pi-owned throughout** — GSD-2's extension layer uses Pi's session and agent abstractions directly rather than wrapping them. GSD contributes:

- **CLI entry point** (`src/cli.ts`) — Orchestrates `createAgentSession()` with GSD-specific tools and extensions
- **GSD extension** (`src/resources/extensions/gsd/`) — Workflow orchestration, auto-mode, and crash recovery

---

## Runtime Assembly

### Component Hierarchy

[repo code] The runtime is assembled in four phases:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ RUNTIME ASSEMBLY PHASES                                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Phase 1: Agent Instance Creation                                          │
│  ─────────────────────────────────────                                     │
│  sdk.ts:185-223                                                            │
│  ├─ Create Agent with initialState, hooks, transport settings              │
│  ├─ Create mutable extensionRunnerRef: { current?: ExtensionRunner }       │
│  └─ Agent's transformContext/onPayload delegate to ref.current             │
│                                                                             │
│  Phase 2: SessionManager Initialization                                    │
│  ─────────────────────────────────────                                     │
│  sdk.ts:68                                                                 │
│  ├─ SessionManager.create(cwd)      — New persistent session               │
│  ├─ SessionManager.open(path)       — Open existing session file           │
│  ├─ SessionManager.continueRecent() — Most recent or new                   │
│  └─ SessionManager.inMemory(cwd)    — No file persistence                  │
│                                                                             │
│  Phase 3: AgentSession Creation and Wiring                                 │
│  ─────────────────────────────────────                                     │
│  agent-session.ts:168-219                                                  │
│  ├─ Store references to Agent, SessionManager, SettingsManager             │
│  ├─ Create delegated subsystems: RetryHandler, CompactionOrchestrator      │
│  ├─ Subscribe to Agent events: agent.subscribe(_handleAgentEvent)          │
│  ├─ Install tool hooks: beforeToolCall/afterToolCall                       │
│  └─ Call _buildRuntime() to create ExtensionRunner                         │
│                                                                             │
│  Phase 4: ExtensionRunner Attachment                                       │
│  ─────────────────────────────────────                                     │
│  agent-session.ts:2265-2280                                                │
│  ├─ Create ExtensionRunner if extensions or custom tools present           │
│  └─ Store reference in extensionRunnerRef.current → completes circle       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### createAgentSession() Flow

[repo code] The primary factory function is defined in `packages/pi-coding-agent/src/core/sdk.ts`:

```typescript
export async function createAgentSession(
  options: CreateAgentSessionOptions,
): Promise<AgentSession> {
  // Phase 1: Create Agent
  const agent = new Agent({
    initialState: { systemPrompt: "", model, thinkingLevel, tools: [] },
    convertToLlm: convertToLlmWithBlockImages,
    onPayload: async (payload, currentModel) => { ... },
    sessionId: sessionManager.getSessionId(),
    transformContext: async (messages) => {
      const runner = extensionRunnerRef.current;
      if (!runner) return messages;
      return runner.emitContext(messages);
    },
    // ... more hooks
  });

  // Phase 3: Create AgentSession
  const session = new AgentSession({
    agent,
    sessionManager,
    settingsManager,
    cwd,
    // ... more options
  });

  // Phase 4: ExtensionRunner attachment happens inside AgentSession constructor

  return session;
}
```

### Agent Hook System

[repo code] The Agent class (`packages/pi-agent-core/src/agent.ts:51-93`) supports these hooks:

| Hook | Signature | Purpose |
|------|-----------|---------|
| `convertToLlm` | `(messages: AgentMessage[]) => Message[]` | Transform messages before LLM call |
| `transformContext` | `(messages: AgentMessage[], signal?) => Promise<AgentMessage[]>` | Context pruning, external context injection |
| `beforeToolCall` | `(ctx) => BeforeToolCallResult?` | Block or modify tool execution |
| `afterToolCall` | `(ctx) => AfterToolCallResult?` | Modify tool results |
| `getApiKey` | `(provider: string) => Promise<string?>` | Dynamic API key resolution |
| `onPayload` | `(payload, model) => payload` | Inspect/replace provider payloads |

### Mutable Ref Pattern

[repo code] A critical pattern enables Agent to call ExtensionRunner before ExtensionRunner exists:

```typescript
// In sdk.ts
const extensionRunnerRef: { current?: ExtensionRunner } = {};

agent = new Agent({
  transformContext: async (messages) => {
    const runner = extensionRunnerRef.current;
    if (!runner) return messages;  // Safe: returns early if not set
    return runner.emitContext(messages);
  },
  // ...
});

// Later, in AgentSession constructor
this._extensionRunner = new ExtensionRunner(...);
if (this._extensionRunnerRef) {
  this._extensionRunnerRef.current = this._extensionRunner;
}
```

**Safety guarantee:** The first LLM call happens after `createAgentSession()` returns, at which point `extensionRunnerRef.current` is set.

### Dependency Injection Pattern

[repo code] RetryHandler and CompactionOrchestrator receive callbacks rather than holding direct references:

```typescript
// agent-session.ts:182-204
this._retryHandler = new RetryHandler({
  agent: this.agent,
  settingsManager: this.settingsManager,
  modelRegistry: this._modelRegistry,
  getModel: () => this.model,           // Callback
  getSessionId: () => this.sessionId,   // Callback
  emit: (event) => this._emit(event),   // Callback
  // ...
});
```

This enables testing and decouples subsystems from AgentSession internals.

---

## Session Lifecycle

### State Machine

| State | Description | Trigger |
|-------|-------------|---------|
| **empty** | No session loaded | Initial state, after `newSession()` |
| **active** | Session loaded with messages | After `prompt()`, `fork()`, `switchSession()` |
| **streaming** | Agent is processing | During `agent.prompt()` call |
| **compacting** | Compaction in progress | During `compact()` operation |
| **branching** | Session fork in progress | During `fork()` or `navigateTree()` |

### Lifecycle Operations

#### newSession(options?)

[repo code] `agent-session.ts:1505-1581`

```
┌─────────────────────────────────────────────────────────────────┐
│ newSession()                                                    │
├─────────────────────────────────────────────────────────────────┤
│ 1. Emit session_before_switch(reason: "new")  ← can cancel     │
│ 2. _disconnectFromAgent()                                       │
│ 3. abort() + agent.reset()                                      │
│ 4. sessionManager.newSession()                                  │
│ 5. agent.sessionId = newSessionId                               │
│ 6. Clear steering/followUp queues                               │
│ 7. _reconnectToAgent()                                          │
│ 8. Emit session_switch(reason: "new")                           │
│ 9. Emit session_state_changed(reason: "new_session")            │
└─────────────────────────────────────────────────────────────────┘
```

**Extension hooks:** `session_before_switch` (cancellable), `session_switch`

#### switchSession(sessionPath)

[repo code] `agent-session.ts:1703-1776`

```
┌─────────────────────────────────────────────────────────────────┐
│ switchSession(sessionPath)                                      │
├─────────────────────────────────────────────────────────────────┤
│ 1. Emit session_before_switch(reason: "resume")  ← can cancel  │
│ 2. _disconnectFromAgent()                                       │
│ 3. abort()                                                      │
│ 4. sessionManager.setSessionFile(sessionPath)                   │
│ 5. agent.sessionId = loadedSessionId                            │
│ 6. Emit session_switch(reason: "resume")                        │
│ 7. agent.replaceMessages(loadedMessages)                        │
│ 8. Restore model/thinking level from session                    │
│ 9. _reconnectToAgent()                                          │
│ 10. Emit session_state_changed(reason: "switch_session")        │
└─────────────────────────────────────────────────────────────────┘
```

**Extension hooks:** `session_before_switch` (cancellable), `session_switch`

#### fork(entryId)

[repo code] `agent-session.ts:1778-1829`

```
┌─────────────────────────────────────────────────────────────────┐
│ fork(entryId)                                                   │
├─────────────────────────────────────────────────────────────────┤
│ 1. Validate entry is a user message                             │
│ 2. Emit session_before_fork(entryId)  ← can cancel             │
│ 3. If entry.parentId is null:                                   │
│      sessionManager.newSession({ parentSession })               │
│    Else:                                                        │
│      sessionManager.createBranchedSession(parentId)             │
│ 4. agent.sessionId = newSessionId                               │
│ 5. Emit session_fork(previousSessionFile)                       │
│ 6. agent.replaceMessages(forkedMessages)                        │
│ 7. Emit session_state_changed(reason: "fork")                   │
└─────────────────────────────────────────────────────────────────┘
```

**Extension hooks:** `session_before_fork` (cancellable), `session_fork`

#### navigateTree(targetId, options)

[repo code] `agent-session.ts:1835-1957`

```
┌─────────────────────────────────────────────────────────────────┐
│ navigateTree(targetId, { summarize, customInstructions, ... })  │
├─────────────────────────────────────────────────────────────────┤
│ 1. Collect entries to summarize (old leaf → target)             │
│ 2. Emit session_before_tree(preparation)  ← can cancel         │
│    - Extensions can provide summary via result.summary          │
│ 3. If summarize && no extension summary:                        │
│      generateBranchSummary(entries)                             │
│ 4. If summary:                                                  │
│      sessionManager.branchWithSummary(newLeafId, summary)       │
│    Else:                                                        │
│      sessionManager.branch(newLeafId) or resetLeaf()            │
│ 5. agent.replaceMessages(navigatedMessages)                     │
│ 6. Emit session_tree(newLeafId, oldLeafId, summaryEntry)        │
└─────────────────────────────────────────────────────────────────┘
```

**Extension hooks:** `session_before_tree` (cancellable, can provide summary), `session_tree`

---

## Event Flow

### Propagation Chain

[repo code] Events flow through a three-layer architecture:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           EVENT PROPAGATION CHAIN                            │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Agent (pi-agent-core)                                                      │
│   ├─ agent-loop.ts: Emits AgentEvent via EventStream                        │
│   │  Events: agent_start, turn_start, message_start, message_update,        │
│   │          message_end, turn_end, agent_end, tool_execution_*             │
│   │                                                                          │
│   ↓ Agent.subscribe(handler)                                                │
│                                                                              │
│   AgentSession (pi-coding-agent)                                            │
│   ├─ _handleAgentEvent: Synchronous handler                                 │
│   │  ├─ Creates retry promise for agent_end                                 │
│   │  └─ Queues async processing: _agentEventQueue.then(_processAgentEvent) │
│   │                                                                          │
│   ├─ _processAgentEvent:                                                    │
│   │  1. await _emitExtensionEvent(event)  ← Extensions first               │
│   │  2. this._emit(event)                   ← Session listeners            │
│   │  3. Handle persistence (appendMessage)                                   │
│   │  4. Check auto-retry/auto-compaction                                    │
│   │                                                                          │
│   ↓ _emitExtensionEvent(event)                                              │
│                                                                              │
│   ExtensionRunner (pi-coding-agent)                                         │
│   ├─ emit(event): Broadcast to extension handlers                           │
│   │  ├─ For each extension with handlers:                                   │
│   │  │   await handler(event, ctx)                                          │
│   │  │   Process result (cancel, transform, etc.)                           │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Event Queue Serialization

[repo code] `agent-session.ts:262-282`

```typescript
private _agentEventQueue: Promise<void> = Promise.resolve();

private _handleAgentEvent = (event: AgentEvent): void => {
  // Synchronous: create retry promise before queueing
  this._createRetryPromiseForAgentEnd(event);
  
  // Queue async processing
  this._agentEventQueue = this._agentEventQueue.then(
    () => this._processAgentEvent(event),
    () => this._processAgentEvent(event),  // Continue on prior error
  );
  
  // Keep queue alive if handler fails
  this._agentEventQueue.catch(() => {});
};
```

**Guarantee:** Events are processed serially, in order. Extensions always see settled state.

### Timing Guarantees

#### Guarantee 1: Event Ordering

[repo code] Events are queued in a promise chain, ensuring serial processing:
- `message_end` is always emitted before `tool_call` for the same turn
- `turn_end` is always emitted after all tool results settle
- `agent_end` is always emitted after all turns complete

#### Guarantee 2: Extension Settled State

[repo code] `agent-session.ts:426-466`

Tool hooks await `_agentEventQueue` before emitting extension events:

```typescript
this.agent.setBeforeToolCall(async ({ toolCall, args }) => {
  // Wait for all queued agent events to settle
  await this._agentEventQueue;
  
  if (!this._extensionRunner?.hasHandlers("tool_call")) return undefined;
  // ... emit tool_call event
});
```

This prevents races in parallel tool execution where extension handlers could see stale agent state.

#### Guarantee 3: Extension Before Session Listeners

[repo code] `agent-session.ts:474-487`

In `_processAgentEvent`:

```typescript
async _processAgentEvent(event: AgentEvent): Promise<void> {
  // Emit to extensions FIRST
  await this._emitExtensionEvent(event);
  
  // Then notify session listeners
  this._emit(event);
  
  // Then handle persistence
  if (event.type === "message_end") {
    this.sessionManager.appendMessage(event.message);
  }
}
```

#### Non-Guarantee: Handler Execution Order

[repo code] `runner.ts:361-402`

Extension handlers execute in **registration order** across extensions. If multiple extensions register for the same event, there's no guarantee which order they execute **unless** a handler returns `{ done: true }` (cancellation short-circuits).

### Agent Event Types

[repo code] `agent-loop.ts`

| Event | When Emitted | Context |
|-------|--------------|---------|
| `agent_start` | At start of `agentLoop()` | No messages yet |
| `turn_start` | At start of each turn | `turnIndex`, `timestamp` |
| `message_start` | When message begins | Full `message` object |
| `message_update` | During streaming | `assistantMessageEvent` with deltas |
| `message_end` | When message completes | Full `message` object |
| `turn_end` | After all tool results | `message`, `toolResults[]` |
| `agent_end` | Agent loop exits | All `messages[]` from this run |
| `tool_execution_start` | Before tool execute | `toolCallId`, `toolName`, `args` |
| `tool_execution_update` | During tool execution | `partialResult` |
| `tool_execution_end` | After tool completes | `result`, `isError` |

### Extension Event Types

[repo code] `types.ts:168-244`

| Event | Purpose | Cancellable |
|-------|---------|-------------|
| `session_start` | After session loaded | No |
| `session_before_switch` | Before new/resume session | Yes |
| `session_switch` | After session switched | No |
| `session_before_fork` | Before forking | Yes |
| `session_fork` | After fork created | No |
| `session_before_compact` | Before compaction | Yes |
| `session_compact` | After compaction | No |
| `session_before_tree` | Before tree navigation | Yes |
| `session_tree` | After tree navigation | No |
| `session_shutdown` | Process exit | No |

### Cancellation Pattern

[repo code] Session lifecycle operations follow a consistent pattern:

```typescript
// 1. Check if extensions have handlers
if (this._extensionRunner?.hasHandlers("session_before_switch")) {
  // 2. Emit before event
  const result = await this._extensionRunner.emit({
    type: "session_before_switch",
    reason: "new",
  });
  
  // 3. Check for cancellation
  if (result?.cancel) {
    return false;  // Operation cancelled
  }
}

// 4. Perform operation
// ...

// 5. Emit after event
await this._extensionRunner.emit({
  type: "session_switch",
  reason: "new",
  previousSessionFile,
});
```

---

## Persistence Model

### File Format

[repo code] Sessions are stored as **newline-delimited JSON (JSONL)** files with `.jsonl` extension:

```
{"type":"session","version":3,"id":"uuid","timestamp":"2024-...","cwd":"/path","parentSession":"/path/to/parent.jsonl"}
{"type":"message","id":"a1b2c3d4","parentId":null,"timestamp":"2024-...","message":{...}}
{"type":"message","id":"e5f6g7h8","parentId":"a1b2c3d4","timestamp":"2024-...","message":{...}}
{"type":"compaction","id":"i9j0k1l2","parentId":"e5f6g7h8","timestamp":"2024-...","summary":"...","firstKeptEntryId":"c3d4e5f6",...}
```

**Key properties:**
- First line is always `SessionHeader` with `type: "session"`
- Each subsequent line is a `SessionEntry` with `id` and `parentId`
- Append-only: new entries are appended, never modified or deleted
- Tree structure: `parentId` forms parent-child relationships

### Entry Types

[repo code] `session-manager.ts:21-88`

| Entry Type | Purpose | LLM Context |
|------------|---------|-------------|
| `SessionHeader` | Session metadata (id, cwd, timestamp, parent) | No |
| `SessionMessageEntry` | User/assistant/tool message | Yes |
| `ThinkingLevelChangeEntry` | Thinking level change | No |
| `ModelChangeEntry` | Model/provider switch | No |
| `CompactionEntry` | Context compaction summary | Yes (summary only) |
| `BranchSummaryEntry` | Branch summary for navigation | Yes |
| `CustomEntry` | Extension-specific data (persisted but not in context) | No |
| `CustomMessageEntry` | Extension-injected messages | Yes |
| `LabelEntry` | User-defined bookmarks | No |
| `SessionInfoEntry` | Session display name | No |

### Tree Structure

[repo code] `session-manager.ts:550-600`

```
Session Root (parentId: null)
├── Entry A (parentId: null)
│   ├── Entry B (parentId: A)
│   │   └── Entry C (parentId: B)  ← leaf pointer here
│   └── Entry D (parentId: A)      ← branch point
│       └── Entry E (parentId: D)  ← another branch
└── Entry F (parentId: null)       ← orphan (orphan handling)
```

**Navigation:**
- `getLeafId()` returns current leaf position
- `branch(entryId)` moves leaf to entryId, next append creates new branch
- `getBranch()` walks from leaf to root, returning path
- `buildSessionContext()` reconstructs messages for LLM from path

### Append-Only Persistence

[repo code] `session-manager.ts:408-435`

```typescript
_persist(entry: SessionEntry): void {
  if (!this.persist || !this.sessionFile) return;

  const hasAssistant = this.fileEntries.some(
    (e) => e.type === "message" && e.message.role === "assistant"
  );
  if (!hasAssistant) {
    // Defer persistence until first assistant message
    this.flushed = false;
    return;
  }

  let release: (() => void) | undefined;
  try {
    release = tryAcquireLockSync(this.sessionFile);
    if (!this.flushed) {
      // First persist: write all entries
      for (const e of this.fileEntries) {
        const prepared = prepareForPersistence(e, this.blobStore) as FileEntry;
        appendFileSync(this.sessionFile, `${JSON.stringify(prepared)}\n`);
      }
      this.flushed = true;
    } else {
      // Subsequent: append single entry
      const prepared = prepareForPersistence(entry, this.blobStore) as FileEntry;
      appendFileSync(this.sessionFile, `${JSON.stringify(prepared)}\n`);
    }
  } finally {
    release?.();
  }
}
```

**Deferred persistence:** File is not created until first assistant message arrives. This prevents empty/corrupt session files from crashes during initial setup.

### Blob Storage

[repo code] `session-manager.ts:145-175`

Large images (>1KB) are externalized to blob storage:

```typescript
const BLOB_EXTERNALIZE_THRESHOLD = 1024; // 1KB

// In prepareForPersistence():
if (key === "content" && isImageBlock(item)) {
  if (!isBlobRef(item.data) && item.data.length >= BLOB_EXTERNALIZE_THRESHOLD) {
    const blobRef = externalizeImageData(blobStore, item.data);
    return { ...item, data: blobRef };  // "blob:sha256:<hash>"
  }
}
```

Blobs are stored in `~/.pi/agent/blobs/<sha256hash>` and resolved on load via `resolveBlobRefsInEntries()`.

### Context Reconstruction

[repo code] `session-manager.ts:187-264`

```typescript
export function buildSessionContext(
  entries: SessionEntry[],
  leafId?: string | null,
  byId?: Map<string, SessionEntry>,
): SessionContext {
  // Walk from leaf to root
  const path: SessionEntry[] = [];
  let current = leaf ? byId.get(leafId) : entries[entries.length - 1];
  while (current) {
    path.unshift(current);
    current = current.parentId ? byId.get(current.parentId) : undefined;
  }

  // Find compaction entry in path
  let compaction: CompactionEntry | null = null;
  for (const entry of path) {
    if (entry.type === "compaction") compaction = entry;
  }

  // Build messages with compaction handling
  if (compaction) {
    // 1. Emit compaction summary as system message
    // 2. Emit kept messages (from firstKeptEntryId to compaction)
    // 3. Emit messages after compaction
  } else {
    // Emit all messages from path
  }

  return { messages, thinkingLevel, model };
}
```

---

## Retry and Compaction

### Retry Mechanism

#### Error Classification

[repo code] `retry-handler.ts:66-82`

```typescript
isRetryableError(message: AssistantMessage): boolean {
  if (message.stopReason !== "error" || !message.errorMessage) return false;

  // Context overflow is NOT retryable (handled by compaction)
  if (isContextOverflow(message, contextWindow)) return false;

  const err = message.errorMessage;
  return /overloaded|rate.?limit|too many requests|429|500|502|503|504|
         service.?unavailable|server.?error|internal.?error|connection.?error|
         connection.?refused|other side closed|fetch failed|upstream.?connect|
         reset before headers|terminated|retry delay|network.?(?:is\s+)?unavailable|
         credentials.*expired|temporarily backed off/i.test(err);
}
```

**Non-retryable:** Context overflow (goes to compaction), authentication failures

#### Exponential Backoff

[repo code] `retry-handler.ts:139-167`

```typescript
const exponentialDelayMs = settings.baseDelayMs * 2 ** (this._retryAttempt - 1);

// Server-requested delay takes precedence
if (message.retryAfterMs !== undefined) {
  const cap = settings.maxDelayMs > 0 ? settings.maxDelayMs : Infinity;
  if (message.retryAfterMs > cap) {
    // Give up if server delay exceeds max
    return false;
  }
  delayMs = message.retryAfterMs;
} else {
  delayMs = exponentialDelayMs;
}
```

#### Fallback Chain

[repo code] `retry-handler.ts:84-137`

```
┌─────────────────────────────────────────────────────────────────┐
│ RETRY FALLBACK CHAIN                                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Error Detected                                                 │
│       ↓                                                         │
│  1. Classify error type                                         │
│     - quota_exhausted, rate_limit, server_error, unknown        │
│       ↓                                                         │
│  2. Credential Fallback (same provider)                         │
│     - If alternate credential: RETRY IMMEDIATELY                │
│       ↓                                                         │
│  3. Cross-Provider Fallback                                     │
│     - If found: switch model, RETRY IMMEDIATELY                 │
│       ↓                                                         │
│  4. Exponential Backoff Retry                                   │
│     - Await sleep(delay) then agent.continue()                  │
│       ↓                                                         │
│  5. Max Retries Exceeded → GIVE UP                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Compaction Mechanism

#### Triggers

[repo code] `compaction-orchestrator.ts:103-159`

**Two trigger types:**
1. **Overflow:** LLM returned context overflow error → compact → auto-retry
2. **Threshold:** Token count over % → compact → no retry (normal operation continues)

#### Extension Integration

[repo code] `compaction-orchestrator.ts:45-99`

**Extension can:**
1. Cancel compaction (`result.cancel = true`)
2. Provide custom compaction (`result.compaction = {...}`)
3. Receive notification after (`session_compact` event)

#### Compaction Entry Structure

[repo code] `session-manager.ts:51-62`

```typescript
interface CompactionEntry<T = unknown> extends SessionEntryBase {
  type: "compaction";
  summary: string;              // LLM-generated summary of removed context
  firstKeptEntryId: string;     // First entry ID after the truncation point
  tokensBefore: number;         // Token count before compaction
  details?: T;                  // Extension-specific data
  fromHook?: boolean;           // True if extension-provided
}
```

---

## Recovery Mechanisms

### Interrupted Session Detection

[repo code] `session-manager.ts:260-288`

```typescript
wasInterrupted(): boolean {
  // Walk backwards to find last message
  for (let i = this.fileEntries.length - 1; i >= 0; i--) {
    const entry = this.fileEntries[i];
    if (entry.type !== "message") continue;

    const msg = entry.message;
    if (msg.role === "user") return false;  // Clean boundary
    if (msg.role === "assistant") {
      const content = Array.isArray(msg.content) ? msg.content : [];
      const hasToolUse = content.some((block) => block.type === "toolCall");
      if (hasToolUse) {
        // Tool use without subsequent tool_result = interrupted
        return true;
      }
      return false;  // Completed text response
    }
  }
  return false;
}
```

**Interruption heuristic:** Last message is assistant with `tool_use` blocks but no following `tool_result` message.

### GSD Crash Recovery (GSD-authored)

[repo code] `src/resources/extensions/gsd/auto-recovery.ts`

GSD adds crash recovery on top of Pi's persistence:

1. **Artifact Verification** — After auto-mode unit completes, verify expected artifact exists
2. **Self-Healing Runtime Records** — Clear stale dispatched records, auto-fix complete-slice state
3. **Merge State Reconciliation** — Auto-resolve `.gsd/` conflicts, abort and reset for code conflicts
4. **Blocker Placeholders** — Write placeholder files when recovery fails, allowing pipeline to advance

### Forensics and Diagnostics

**Runtime signals:**
- Session creation emits `session:create` event
- Lifecycle transitions emit `session:switch`, `session:fork`, `session:compact`
- RetryHandler logs retry attempts with backoff timing
- CompactionOrchestrator logs compaction triggers
- GSD crash recovery writes forensics to `.gsd/session-forensics/`

**Inspection surfaces:**
- `SessionManager.listSessions()` exposes session tree
- `AgentSession.getSessionInfo()` returns current session metadata
- `sessionManager.wasInterrupted()` detects incomplete tool_use turns
- `retryHandler.retryAttempt` for current attempt (0 if not retrying)
- `compactionOrchestrator.isCompacting` for in-progress compaction
- `listUnitRuntimeRecords(base)` for dispatched/completed records

**Redaction constraints:**
- API keys accessed via `getApiKey` callback are never logged
- Session content redaction handled by `redactForLogging` utility

---

## Runtime Ownership Map

### Pi-Owned Components

[repo code] All core runtime components are Pi-owned:

| Component | Package | Location |
|-----------|---------|----------|
| Agent | `pi-agent-core` | `packages/pi-agent-core/src/agent.ts` |
| AgentSession | `pi-coding-agent` | `packages/pi-coding-agent/src/core/agent-session.ts` |
| SessionManager | `pi-coding-agent` | `packages/pi-coding-agent/src/core/session-manager.ts` |
| ExtensionRunner | `pi-coding-agent` | `packages/pi-coding-agent/src/core/extensions/runner.ts` |
| RetryHandler | `pi-coding-agent` | `packages/pi-coding-agent/src/core/retry-handler.ts` |
| CompactionOrchestrator | `pi-coding-agent` | `packages/pi-coding-agent/src/core/compaction-orchestrator.ts` |
| ModelRegistry | `pi-coding-agent` | `packages/pi-coding-agent/src/core/model-registry.ts` |
| SettingsManager | `pi-coding-agent` | `packages/pi-coding-agent/src/core/settings-manager.ts` |
| ResourceLoader | `pi-coding-agent` | `packages/pi-coding-agent/src/core/resource-loader.ts` |

### GSD-Authored Components

| Component | Location | Purpose |
|-----------|----------|---------|
| CLI Entry | `src/cli.ts` | Orchestrates `createAgentSession()` with GSD tools |
| GSD Extension | `src/resources/extensions/gsd/` | Workflow orchestration, auto-mode, crash recovery |
| Auto-recovery | `src/resources/extensions/gsd/auto-recovery.ts` | Crash detection and recovery |
| Auto-mode | `src/resources/extensions/gsd/auto-mode.ts` | Dispatch loop orchestration |

### Resolved Seams from S01

| Seam | Resolution |
|------|------------|
| **#1: Session lifecycle ownership** | [repo code] Session lifecycle is **Pi-owned throughout**. GSD's `ctx.newSession()` delegates directly to `AgentSession.newSession()` → `SessionManager.newSession()`. No GSD-authored session management layer. |
| **#3: State persistence model** | [repo code] Persistence is **Pi-owned**. JSONL format with append-only tree, documented entry types, blob storage for large images, version migrations applied in-place. |
| **#4: Extension event semantics** | [repo code] Event timing has clear guarantees: serialization via promise chain, extensions receive events before session listeners, tool hooks await event queue, `session_before_*` events are cancellable. |

### Remaining Questions

No unresolved questions remain for the runtime layer. The S01 seams have been resolved through code tracing:

- Session lifecycle: Traced `createAgentSession()` through all phases
- Event timing: Documented event queue serialization and timing guarantees
- Persistence: Documented JSONL format and reconstruction logic
- Retry/compaction: Documented triggers, fallback chains, and extension integration

---

## Appendix: File Reference

| File | Purpose |
|------|---------|
| `packages/pi-coding-agent/src/core/sdk.ts` | Factory function `createAgentSession()` |
| `packages/pi-agent-core/src/agent.ts` | Core Agent class with hooks |
| `packages/pi-coding-agent/src/core/agent-session.ts` | Session coordination and lifecycle |
| `packages/pi-coding-agent/src/core/session-manager.ts` | JSONL persistence and tree structure |
| `packages/pi-coding-agent/src/core/retry-handler.ts` | Retry classification and backoff |
| `packages/pi-coding-agent/src/core/compaction-orchestrator.ts` | Auto-compaction triggers |
| `packages/pi-coding-agent/src/core/extensions/runner.ts` | Extension event emission |
| `packages/pi-agent-core/src/agent-loop.ts` | Agent loop event source |
| `packages/pi-coding-agent/src/core/extensions/types.ts` | Extension event type taxonomy |
| `src/resources/extensions/gsd/auto-recovery.ts` | GSD crash recovery |
| `src/cli.ts` | GSD CLI entry point |
