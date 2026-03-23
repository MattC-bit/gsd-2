# GSD-2 Context Engineering and Disk-State Workflow Model

This document explains how GSD-2 constructs prompts and manages workflow state via disk artifacts. It answers three fundamental questions:

1. **How are prompts assembled?** From Pi-owned core system prompts through GSD-authored unit-specific builders.
2. **How does workflow state work?** Via disk-derived state — `.gsd/` artifacts are the source of truth, not mutable runtime tracking.
3. **Why must analyzed and runner artifacts stay distinct?** The subject-vs-runner guardrail prevents conflation between the system being analyzed and the live execution producing this documentation.

---

## Overview

GSD-2's context engineering and disk-state workflow model defines how the system constructs prompts for each unit type and how workflow progress is tracked through disk artifacts rather than in-memory state. This model is central to understanding GSD-2's architecture because:

- **Prompt assembly determines what the agent sees**: Every unit type (execute-task, plan-slice, etc.) receives a focused prompt constructed from templates, inlined files, and computed budgets.
- **Disk-state determines what happens next**: Phase transitions, unit selection, and progress tracking all derive from reading `.gsd/` files on disk after each agent completion.
- **Separation enables reliability**: The two-layer system (Pi-owned core + GSD-authored builders) and disk-based state model together enable crash recovery, parallel worker isolation, and audit trails.

The document is organized into three main parts:

1. **Context Assembly Pipeline** — How prompts are built from templates, inlined content, and budgets
2. **Disk-State Workflow Model** — How `.gsd/` artifacts function as workflow state
3. **Subject-vs-Runner Guardrail** — Why and how to avoid conflating the analyzed system with the live runner

---

## Context Assembly Pipeline

The context assembly pipeline is a two-layer system: Pi-owned core infrastructure provides the foundational system prompt construction, while GSD-authored builders add unit-specific context assembly. This separation ensures Pi remains generic while GSD adds workflow-specific intelligence.

### Pi-Owned Core: `buildSystemPrompt()`

The `buildSystemPrompt()` function in `packages/pi-coding-agent/src/core/system-prompt.ts` is the foundation of all prompt construction. It builds a structured system prompt from composable parts [repo code].

**Function signature and options**:

```typescript
// packages/pi-coding-agent/src/core/system-prompt.ts:19-29 [repo code]
export interface BuildSystemPromptOptions {
  customPrompt?: string;           // Replaces default prompt entirely
  selectedTools?: string[];        // Tools to include (default: read, bash, edit, write)
  toolSnippets?: Record<string, string>;  // One-line tool descriptions
  promptGuidelines?: string[];     // Additional guideline bullets
  appendSystemPrompt?: string;     // Text appended after main prompt
  cwd?: string;                    // Working directory (default: process.cwd())
  contextFiles?: Array<{ path: string; content: string }>;  // Pre-loaded context
  skills?: Skill[];                // Pre-loaded skills
}
```

**Core responsibilities**:

| Responsibility | Implementation | Lines |
|---------------|----------------|-------|
| Tool descriptions | Maps tool names to one-line descriptions via `toolDescriptions` record | L6-12 |
| Guidelines construction | Builds tool-aware guidelines based on selected tools | L100-138 |
| Project context appending | Appends context files under `# Project Context` heading | L151-156 |
| Skills section formatting | Uses `formatSkillsForPrompt()` when read tool available | L159-161 |
| Date/time injection | Formats locale-aware datetime string | L37-44, L167 |
| Working directory injection | Resolves and includes `cwd` in prompt | L35, L168 |

**Guideline construction is tool-aware**:

The function dynamically constructs guidelines based on which tools are available. For example, if both `read` and `edit` are available, it adds "Use read to examine files before editing" [repo code]. If `lsp` is available, it adds comprehensive LSP usage guidelines including navigation, understanding, refactoring, and verification patterns [repo code L117-128].

**Two prompt paths**:

```typescript
// packages/pi-coding-agent/src/core/system-prompt.ts [repo code]
if (customPrompt) {
  // Custom prompt path: minimal framework, just append context/skills
  let prompt = customPrompt;
  // ... append context files, skills, datetime, guidelines
  return prompt;
}

// Default prompt path: full structured prompt
let prompt = `You are an expert coding assistant operating inside pi...
Available tools:
${toolsList}
Guidelines:
${guidelines}
Pi documentation: ...`;
// ... append context files, skills, datetime
return prompt;
```

The custom prompt path is used by GSD when it provides its own structured prompts; the default path provides a fallback for interactive sessions [repo code].

### GSD-Authored Unit-Specific Builders

GSD's `auto-prompts.ts` contains **27+ prompt builder functions**, one for each unit type in the auto-mode workflow [repo code]. These builders are pure async functions that load templates, inline file content, and construct focused prompts for specific phases.

**Builder function inventory**:

| Function | Unit Type | Purpose |
|----------|-----------|---------|
| `buildDiscussMilestonePrompt` | discuss-milestone | Interactive context gathering with user |
| `buildResearchMilestonePrompt` | research-milestone | Codebase exploration and research |
| `buildPlanMilestonePrompt` | plan-milestone | Roadmap and slice decomposition |
| `buildResearchSlicePrompt` | research-slice | Slice-specific research |
| `buildPlanSlicePrompt` | plan-slice | Task decomposition for a slice |
| `buildExecuteTaskPrompt` | execute-task | Single task execution |
| `buildCompleteSlicePrompt` | complete-slice | Slice summary and UAT generation |
| `buildCompleteMilestonePrompt` | complete-milestone | Milestone summary |
| `buildValidateMilestonePrompt` | validate-milestone | Validation and remediation |
| `buildReplanSlicePrompt` | replan-slice | Replanning after blocker |
| `buildRunUatPrompt` | run-uat | UAT execution |
| `buildReassessRoadmapPrompt` | reassess-roadmap | Post-slice assessment |
| `buildReactiveExecutePrompt` | reactive-execute | Parallel task dispatch |

Each builder follows a consistent pattern [repo code]:

1. **Resolve file paths** using `resolve*()` functions from `paths.ts`
2. **Inline content** using `inlineFile()`, `inlineFileOptional()`, or `inlineFileSmart()`
3. **Load template** using `loadPrompt()` with variable substitution
4. **Return complete prompt** ready for injection

**Inline helpers**:

Three inline helpers provide progressively smarter content inclusion:

```typescript
// src/resources/extensions/gsd/auto-prompts.ts:96-105 [repo code]
export async function inlineFile(
  absPath: string | null, relPath: string, label: string,
): Promise<string> {
  const content = absPath ? await loadFile(absPath) : null;
  if (!content) {
    return `### ${label}\nSource: \`${relPath}\`\n\n_(not found — file does not exist yet)_`;
  }
  return `### ${label}\nSource: \`${relPath}\`\n\n${content.trim()}`;
}
```

`inlineFile()` always returns content (or a "not found" fallback) — use when the file is expected to exist [repo code].

```typescript
// src/resources/extensions/gsd/auto-prompts.ts:110-116 [repo code]
export async function inlineFileOptional(
  absPath: string | null, relPath: string, label: string,
): Promise<string | null> {
  const content = absPath ? await loadFile(absPath) : null;
  if (!content) return null;
  return `### ${label}\nSource: \`${relPath}\`\n\n${content.trim()}`;
}
```

`inlineFileOptional()` returns `null` if the file doesn't exist — use for truly optional content [repo code].

```typescript
// src/resources/extensions/gsd/auto-prompts.ts:124-145 [repo code]
export async function inlineFileSmart(
  absPath: string | null, relPath: string, label: string,
  query?: string, threshold = 3000,
): Promise<string> {
  // ...loads content...
  // For large files, truncate at section boundary
  const truncated = truncateAtSectionBoundary(content, threshold).content;
  return `### ${label}\nSource: \`${relPath}\`\n\n${truncated}`;
}
```

`inlineFileSmart()` uses semantic chunking for large files, truncating at section boundaries rather than mid-content [repo code].

**Dependency summary inlining**:

```typescript
// src/resources/extensions/gsd/auto-prompts.ts:150-178 [repo code]
export async function inlineDependencySummaries(
  mid: string, sid: string, base: string, budgetChars?: number,
): Promise<string> {
  // Load roadmap, find slice dependencies
  const roadmap = parseRoadmap(roadmapContent);
  const sliceEntry = roadmap.slices.find(s => s.id === sid);
  // For each dependency, inline its summary
  for (const dep of sliceEntry.depends) {
    const summaryFile = resolveSliceFile(base, mid, dep, "SUMMARY");
    // ... inline summary ...
  }
  // Apply budget truncation if specified
}
```

This ensures each unit receives context from completed dependencies without requiring explicit file reads [repo code].

**DB-aware inline helpers**:

Two helpers query the GSD database when available, falling back to filesystem:

```typescript
// src/resources/extensions/gsd/auto-prompts.ts:211-231 [repo code]
export async function inlineDecisionsFromDb(
  base: string, milestoneId?: string, scope?: string, level?: InlineLevel,
): Promise<string | null> {
  try {
    const { isDbAvailable } = await import("./gsd-db.js");
    if (isDbAvailable()) {
      const { queryDecisions, formatDecisionsForPrompt } = await import("./context-store.js");
      const decisions = queryDecisions({ milestoneId, scope });
      if (decisions.length > 0) {
        // Use compact format for non-full levels to save ~35% tokens
        const formatted = inlineLevel !== "full"
          ? formatDecisionsCompact(decisions)
          : formatDecisionsForPrompt(decisions);
        return `### Decisions\nSource: \`.gsd/DECISIONS.md\`\n\n${formatted}`;
      }
    }
  } catch {
    // DB not available — fall through to filesystem
  }
  return inlineGsdRootFile(base, "decisions.md", "Decisions");
}
```

`inlineDecisionsFromDb()` and `inlineRequirementsFromDb()` enable scoped queries (milestone, slice) with compact formatting for token efficiency [repo code].

**Skill activation block construction**:

```typescript
// src/resources/extensions/gsd/auto-prompts.ts:339-367 [repo code]
export function buildSkillActivationBlock(params: {
  base: string;
  milestoneId: string;
  milestoneTitle?: string;
  sliceId?: string;
  sliceTitle?: string;
  taskId?: string;
  taskTitle?: string;
  extraContext?: string[];
  taskPlanContent?: string | null;
  preferences?: GSDPreferences;
}): string {
  // Tokenize context from IDs and titles
  const contextTokens = tokenizeSkillContext(...);
  
  // Match skills against context
  for (const skill of visibleSkills) {
    if (skillMatchesContext(skill, contextTokens)) {
      matched.add(normalizeSkillReference(skill.name));
    }
  }
  
  // Format as skill_activation block
  return formatSkillActivationBlock(ordered);
}
```

The skill activation system uses context-aware matching to suggest relevant skills based on milestone, slice, and task titles [repo code].

**Example: `buildExecuteTaskPrompt()`**:

```typescript
// src/resources/extensions/gsd/auto-prompts.ts:567-660 [repo code]
export async function buildExecuteTaskPrompt(
  mid: string, sid: string, sTitle: string,
  tid: string, tTitle: string, base: string,
  level?: InlineLevel | ExecuteTaskPromptOptions,
): Promise<string> {
  // 1. Resolve prior summaries for carry-forward
  const priorSummaries = opts.carryForwardPaths ?? await getPriorTaskSummaryPaths(mid, sid, tid, base);
  
  // 2. Inline task plan
  const taskPlanInline = taskPlanContent
    ? ["## Inlined Task Plan...", taskPlanContent.trim()].join("\n")
    : ["## Inlined Task Plan...", "Task plan not found..."].join("\n");
  
  // 3. Extract slice plan excerpt
  const slicePlanExcerpt = extractSliceExecutionExcerpt(slicePlanContent, ...);
  
  // 4. Build resume section from continue file
  const resumeSection = buildResumeSection(continueContent, ...);
  
  // 5. Build carry-forward from prior summaries
  const carryForwardSection = await buildCarryForwardSection(effectivePriorSummaries, base);
  
  // 6. Compute verification budget
  const budgets = computeBudgets(contextWindow);
  
  // 7. Load template with all variables
  return loadPrompt("execute-task", {
    taskPlanInline,
    slicePlanExcerpt,
    carryForwardSection,
    resumeSection,
    verificationBudget,
    skillActivation: buildSkillActivationBlock({...}),
  });
}
```

This pattern — resolve paths → inline content → compute budgets → load template — is consistent across all builders [repo code].

### Context Budget System

The context budget system in `src/resources/extensions/gsd/context-budget.ts` allocates the executor's context window across content categories using proportional ratios [repo code].

**Budget ratio constants**:

```typescript
// src/resources/extensions/gsd/context-budget.ts:12-23 [repo code]
/** Proportion of context window for dependency/prior-task summaries */
const SUMMARY_RATIO = 0.15;

/** Proportion of context window for inline context (plans, decisions, code) */
const INLINE_CONTEXT_RATIO = 0.40;

/** Proportion of context window for verification sections in prompts */
const VERIFICATION_RATIO = 0.10;

/** Approximate chars-per-token conversion factor */
const CHARS_PER_TOKEN = 4;

/** Default context window when none can be resolved (D002) */
const DEFAULT_CONTEXT_WINDOW = 200_000;
```

**Budget allocation structure**:

```typescript
// src/resources/extensions/gsd/context-budget.ts:49-59 [repo code]
export interface BudgetAllocation {
  /** Character budget for dependency/prior-task summaries */
  summaryBudgetChars: number;
  /** Character budget for inline context (plans, decisions, code snippets) */
  inlineContextBudgetChars: number;
  /** Recommended task count range for the executor at this context window */
  taskCountRange: { min: number; max: number };
  /** Percentage of context consumed before suggesting continue checkpoint */
  continueThresholdPercent: number;
  /** Character budget for verification sections */
  verificationBudgetChars: number;
}
```

**`computeBudgets()` function**:

```typescript
// src/resources/extensions/gsd/context-budget.ts:74-88 [repo code]
export function computeBudgets(contextWindow: number, provider?: TokenProvider): BudgetAllocation {
  const effectiveWindow = contextWindow > 0 ? contextWindow : DEFAULT_CONTEXT_WINDOW;
  const charsPerToken = provider ? getCharsPerToken(provider) : CHARS_PER_TOKEN;
  const totalChars = effectiveWindow * charsPerToken;

  return {
    summaryBudgetChars: Math.floor(totalChars * SUMMARY_RATIO),
    inlineContextBudgetChars: Math.floor(totalChars * INLINE_CONTEXT_RATIO),
    verificationBudgetChars: Math.floor(totalChars * VERIFICATION_RATIO),
    continueThresholdPercent: CONTINUE_THRESHOLD_PERCENT,
    taskCountRange: {
      min: TASK_COUNT_MIN,
      max: resolveTaskCountMax(effectiveWindow),
    },
  };
}
```

The function converts the context window from tokens to chars, then applies the ratio constants. The result guides prompt builders on how much content to inline [repo code].

**Task count scaling**:

```typescript
// src/resources/extensions/gsd/context-budget.ts:33-39 [repo code]
const TASK_COUNT_TIERS: [number, number][] = [
  [500_000, 8],   // 500K+ tokens → up to 8 tasks
  [200_000, 6],   // 200K+ tokens → up to 6 tasks
  [128_000, 5],   // 128K+ tokens → up to 5 tasks
  [0, 3],         // anything smaller → up to 3 tasks
];
```

Larger context windows support more tasks per slice, preventing executor overload on smaller models [repo code].

**Section-boundary truncation**:

```typescript
// src/resources/extensions/gsd/context-budget.ts:104-135 [repo code]
export function truncateAtSectionBoundary(content: string, budgetChars: number): TruncationResult {
  if (!content || content.length <= budgetChars) {
    return { content, droppedSections: 0 };
  }

  // Split on section markers: ### headings or --- dividers
  const sections = splitIntoSections(content);

  // Greedily keep sections that fit
  let usedChars = 0;
  let keptCount = 0;
  for (const section of sections) {
    if (usedChars + sectionLen > budgetChars && keptCount > 0) break;
    usedChars += sectionLen;
    keptCount++;
  }

  const droppedCount = sections.length - keptCount;
  const kept = sections.slice(0, keptCount).join("");
  return {
    content: kept.trimEnd() + `\n\n[...truncated ${droppedCount} sections]`,
    droppedSections: droppedCount,
  };
}
```

Per Decision D003, section-boundary truncation is mandatory — mid-section cuts are unacceptable because they break semantic coherence [repo code].

**Executor context window resolution**:

```typescript
// src/resources/extensions/gsd/context-budget.ts:150-171 [repo code]
export function resolveExecutorContextWindow(
  registry: MinimalModelRegistry | undefined,
  preferences: MinimalPreferences | undefined,
  sessionContextWindow?: number,
): number {
  // Step 1: Try configured executor model
  if (preferences?.models?.execution && registry) {
    const model = findModelById(registry, modelId);
    if (model && model.contextWindow > 0) return model.contextWindow;
  }

  // Step 2: Fall back to session context window
  if (sessionContextWindow && sessionContextWindow > 0) return sessionContextWindow;

  // Step 3: Fall back to default (D002)
  return DEFAULT_CONTEXT_WINDOW;
}
```

The resolution chain ensures budgeting works even when model registry is unavailable [repo code].

### Template Loading System

The template loader in `src/resources/extensions/gsd/prompt-loader.ts` provides prompt templates with variable substitution and eager caching [repo code].

**Eager caching pattern**:

```typescript
// src/resources/extensions/gsd/prompt-loader.ts:11-23 [repo code]
// Cache all templates eagerly at module load — a running session uses the
// template versions that were on disk at startup, immune to later overwrites.
const templateCache = new Map<string, string>();

function warmCache(): void {
  try {
    for (const file of readdirSync(promptsDir)) {
      if (!file.endsWith(".md")) continue;
      const name = file.slice(0, -3);
      templateCache.set(name, readFileSync(join(promptsDir, file), "utf-8"));
    }
  } catch { /* prompts/ may not exist in test environments */ }
}

// Snapshot all templates at module load time
warmCache();
```

This prevents mid-session crashes when another `gsd` launch overwrites `~/.gsd/agent/` with newer templates. The in-memory code (expecting variable set A) stays compatible with cached templates (providing variable set A) [repo code].

**Template loading with validation**:

```typescript
// src/resources/extensions/gsd/prompt-loader.ts:52-85 [repo code]
export function loadPrompt(name: string, vars: Record<string, string> = {}): string {
  let content = templateCache.get(name);
  if (content === undefined) {
    // Lazy load if not in cache (test environments)
    content = readFileSync(join(promptsDir, `${name}.md`), "utf-8");
    templateCache.set(name, content);
  }

  // Check BEFORE substitution: verify all declared {{var}} have values
  const declared = content.match(/\{\{[a-zA-Z][a-zA-Z0-9_]*\}\}/g);
  if (declared) {
    const missing = [...new Set(declared)]
      .map(m => m.slice(2, -2))
      .filter(key => !(key in effectiveVars));
    if (missing.length > 0) {
      throw new GSDError(GSD_PARSE_ERROR,
        `loadPrompt("${name}"): template declares {{${missing.join("}}, {{")}}}} but no value was provided. ` +
        `Restart pi to reload the extension.`
      );
    }
  }

  // Perform substitution
  for (const [key, value] of Object.entries(effectiveVars)) {
    content = content.replaceAll(`{{${key}}}`, value);
  }
  return content.trim();
}
```

Pre-substitution validation catches mismatched template/code versions early with a clear error message [repo code].

**Template inlining helper**:

```typescript
// src/resources/extensions/gsd/prompt-loader.ts:100-104 [repo code]
export function inlineTemplate(name: string, label: string): string {
  const content = loadTemplate(name);
  return `${content}\n\n### Output Template: ${label}\nSource: \`templates/${name}.md\``;
}
```

Templates from `templates/` directory are inlined into prompts with source attribution, giving the LLM output format guidance [repo code].

### Pi-Owned vs GSD-Authored Responsibilities

| Aspect | Pi-Owned | GSD-Authored |
|--------|----------|--------------|
| **Core system prompt** | `buildSystemPrompt()` in `system-prompt.ts` | Unit-specific builders in `auto-prompts.ts` |
| **Tool descriptions** | `toolDescriptions` record | None (uses Pi's) |
| **Guidelines** | Tool-aware guideline construction | Skill activation blocks |
| **Context files** | Appends pre-loaded `contextFiles[]` | Resolves and inlines via `inlineFile*()` |
| **Skills** | Formats via `formatSkillsForPrompt()` | Context-aware skill matching |
| **Templates** | None (no template system) | Full template system in `prompt-loader.ts` |
| **Budgeting** | None | Full budget system in `context-budget.ts` |

The separation is clean: Pi provides a generic prompt framework, GSD provides workflow-specific context assembly on top of it [repo code].

---

## Disk-State Workflow Model

The disk-state workflow model defines `.gsd/` artifacts as the source of truth for workflow state. Phase transitions, active units, and progress are derived by reading files from disk on each dispatch cycle — not by maintaining mutable runtime state. This design ensures crash resilience, enables clean restarts, and supports parallel worker isolation.

### `deriveState()`: The Source of Truth

The `deriveState()` function in `src/resources/extensions/gsd/state.ts` reconstructs complete GSD state from files on disk. It is called at the start of every dispatch cycle to determine the current position in the workflow [repo code].

**Function signature and return type**:

```typescript
// src/resources/extensions/gsd/state.ts [repo code]
export async function deriveState(basePath: string): Promise<GSDState> {
  // Return cached result if within the TTL window for the same basePath
  if (
    _stateCache &&
    _stateCache.basePath === basePath &&
    Date.now() - _stateCache.timestamp < CACHE_TTL_MS
  ) {
    return _stateCache.result;
  }
  // ... derive state from disk ...
}
```

**Memoization cache**:

```typescript
// src/resources/extensions/gsd/state.ts [repo code]
interface StateCache {
  basePath: string;
  result: GSDState;
  timestamp: number;
}

const CACHE_TTL_MS = 100;
let _stateCache: StateCache | null = null;
```

Within a single dispatch cycle (~100ms window), repeated calls return the cached value instead of re-reading the entire `.gsd/` tree. The cache is invalidated by `invalidateStateCache()` whenever planning files change [repo code].

**State derivation algorithm**:

The `_deriveStateImpl()` function performs a single-pass scan through all milestones [repo code]:

1. **Batch-parse file cache**: When the native Rust parser is available, read every `.md` file under `.gsd/` in one call and build an in-memory content map. This eliminates O(N) individual `fs.readFile` calls during traversal [repo code].

2. **Phase 1: Build roadmap cache and completeness set**: Parse each milestone's roadmap once, caching results. Track which milestones are complete [repo code].

3. **Phase 2: Build registry using cached roadmaps**: No re-parsing or re-reading. Determine active milestone by finding the first incomplete milestone with satisfied dependencies [repo code].

4. **Find active slice**: Within the active milestone's roadmap, find the first incomplete slice with all dependencies satisfied [repo code].

5. **Find active task**: Within the active slice's plan, find the first incomplete task [repo code].

**Phase derivation from disk artifacts**:

Phase is determined by examining file presence and content — not by tracking state transitions:

```typescript
// src/resources/extensions/gsd/state.ts [repo code]
// No roadmap — check for CONTEXT-DRAFT.md to distinguish draft from blank
const contextFile = resolveMilestoneFile(basePath, mid, "CONTEXT");
const draftFile = resolveMilestoneFile(basePath, mid, "CONTEXT-DRAFT");
if (!contextFile && draftFile) activeMilestoneHasDraft = true;

// Phase determination
const phase = activeMilestoneHasDraft ? 'needs-discussion' : 'pre-planning';
```

**Phase enum values**:

| Phase | Meaning | Disk Evidence |
|-------|---------|---------------|
| `pre-planning` | No milestones, or milestone has no roadmap | No ROADMAP.md |
| `needs-discussion` | Milestone has CONTEXT-DRAFT.md but no CONTEXT.md | CONTEXT-DRAFT.md exists, CONTEXT.md absent |
| `planning` | Slice exists but has no plan file | No S##-PLAN.md |
| `executing` | Active task found with plan file | T##-PLAN.md exists, not done |
| `summarizing` | All tasks done but slice not marked complete | All tasks `[x]`, no S##-SUMMARY.md |
| `validating-milestone` | All slices done, no terminal validation | VALIDATION.md with `needs-remediation` |
| `completing-milestone` | All slices done, terminal validation, no summary | VALIDATION.md with `pass`/`needs-attention`, no M##-SUMMARY.md |
| `complete` | Summary exists | M##-SUMMARY.md exists |
| `blocked` | Dependencies unmet or blocker discovered | `depends_on` in CONTEXT.md not satisfied |
| `replanning-slice` | Blocker discovered during execution | Task summary with `blocker_discovered: true` |

**Ghost milestone detection**:

A "ghost" milestone directory contains only META.json (and no substantive files). These appear when a milestone is created but never initialised:

```typescript
// src/resources/extensions/gsd/state.ts [repo code]
export function isGhostMilestone(basePath: string, mid: string): boolean {
  const context   = resolveMilestoneFile(basePath, mid, "CONTEXT");
  const draft     = resolveMilestoneFile(basePath, mid, "CONTEXT-DRAFT");
  const roadmap   = resolveMilestoneFile(basePath, mid, "ROADMAP");
  const summary   = resolveMilestoneFile(basePath, mid, "SUMMARY");
  return !context && !draft && !roadmap && !summary;
}
```

Ghost milestones are skipped during state derivation — they don't count as active or complete [repo code].

**Completeness helpers**:

```typescript
// src/resources/extensions/gsd/state.ts [repo code]
export function isSliceComplete(plan: SlicePlan): boolean {
  return plan.tasks.length > 0 && plan.tasks.every(t => t.done);
}

export function isMilestoneComplete(roadmap: Roadmap): boolean {
  return roadmap.slices.length > 0 && roadmap.slices.every(s => s.done);
}
```

**Parallel worker isolation**:

When `GSD_MILESTONE_LOCK` environment variable is set, the process is a parallel worker scoped to a single milestone. State derivation filters the milestone list so this worker only sees its assigned milestone:

```typescript
// src/resources/extensions/gsd/state.ts [repo code]
const milestoneLock = process.env.GSD_MILESTONE_LOCK;
if (milestoneLock && milestoneIds.includes(milestoneLock)) {
  milestoneIds.length = 0;
  milestoneIds.push(milestoneLock);
}
```

This gives each worker complete isolation without modifying any other state derivation logic [repo code].

### Path Resolution Hierarchy

Path resolution in `src/resources/extensions/gsd/paths.ts` follows a strict hierarchy from `.gsd/` root to task files [repo code].

**Resolution flow**:

```
gsdRoot(basePath)
    └── milestonesDir(basePath) → .gsd/milestones/
            └── resolveMilestonePath(basePath, mid) → M001/
                    ├── resolveMilestoneFile(basePath, mid, "ROADMAP") → M001-ROADMAP.md
                    └── resolveSlicePath(basePath, mid, sid) → slices/S01/
                            ├── resolveSliceFile(basePath, mid, sid, "PLAN") → S01-PLAN.md
                            └── resolveTasksDir(basePath, mid, sid) → tasks/
                                    └── resolveTaskFile(basePath, mid, sid, tid, "PLAN") → T01-PLAN.md
```

**GSD root discovery**:

The `gsdRoot()` function resolves the `.gsd` directory with a multi-step probe:

```typescript
// src/resources/extensions/gsd/paths.ts [repo code]
export function gsdRoot(basePath: string): string {
  // 1. Fast path — check the input path directly
  const local = join(rawBasePath, ".gsd");
  if (existsSync(local)) return local;

  // 2. Git root anchor — handles cwd-is-a-subdirectory
  let gitRoot: string | null = null;
  // ... spawn git rev-parse --show-toplevel ...

  if (gitRoot) {
    const candidate = join(gitRoot, ".gsd");
    if (existsSync(candidate)) return candidate;
  }

  // 3. Walk up from basePath to the git root
  // ... walk up directory tree ...

  // 4. Fallback for init/creation
  return local;
}
```

This handles worktrees, subdirectories, and init scenarios [repo code].

**Directory and file naming conventions**:

| Artifact | Convention | Example |
|----------|------------|---------|
| Milestone dir | Bare ID | `M001/` |
| Slice dir | Bare ID | `S01/` |
| Milestone file | ID-SUFFIX.md | `M001-ROADMAP.md` |
| Slice file | ID-SUFFIX.md | `S01-PLAN.md` |
| Task file | T##-SUFFIX.md | `T03-SUMMARY.md` |

**Legacy compatibility**:

Resolvers handle legacy descriptor-suffixed names via prefix matching:

```typescript
// src/resources/extensions/gsd/paths.ts [repo code]
export function resolveDir(parentDir: string, idPrefix: string): string | null {
  // Exact match first (current convention: bare ID)
  const exact = entries.find(e => e.isDirectory() && e.name === idPrefix);
  if (exact) return exact.name;
  // Prefix match for legacy descriptor dirs: M001-SOMETHING
  const prefixed = entries.find(
    e => e.isDirectory() && e.name.startsWith(idPrefix + "-")
  );
  return prefixed ? prefixed.name : null;
}

export function resolveFile(dir: string, idPrefix: string, suffix: string): string | null {
  // Direct match: ID-SUFFIX.md
  const direct = entries.find(e => e.toUpperCase() === target);
  if (direct) return direct;
  // Legacy pattern match: ID-DESCRIPTOR-SUFFIX.md
  const pattern = new RegExp(`^${idPrefix}-.*-${suffix}\\.md$`, "i");
  const match = entries.find(e => pattern.test(e));
  if (match) return match;
  // Legacy fallback: suffix.md
  const legacy = entries.find(e => e.toLowerCase() === `${suffix.toLowerCase()}.md`);
  if (legacy) return legacy;
  return null;
}
```

Existing projects work without migration [repo code].

**Relative path builders**:

For prompt construction, relative path builders return `.gsd/milestones/...` paths:

```typescript
// src/resources/extensions/gsd/paths.ts [repo code]
export function relMilestonePath(basePath: string, milestoneId: string): string {
  const dir = resolveDir(milestonesDir(basePath), milestoneId);
  if (dir) return `.gsd/milestones/${dir}`;
  return `.gsd/milestones/${milestoneId}`;
}

export function relSliceFile(
  basePath: string, milestoneId: string, sliceId: string, suffix: string
): string {
  const sRel = relSlicePath(basePath, milestoneId, sliceId);
  const sDir = resolveSlicePath(basePath, milestoneId, sliceId);
  if (sDir) {
    const file = resolveFile(sDir, sliceId, suffix);
    if (file) return `${sRel}/${file}`;
  }
  return `${sRel}/${buildSliceFileName(sliceId, suffix)}`;
}
```

**Native tree cache**:

When the native module is available, `nativeScanGsdTree()` scans the entire `.gsd/` tree in one call:

```typescript
// src/resources/extensions/gsd/paths.ts [repo code]
let nativeTreeCache: Map<string, GsdTreeEntry[]> | null = null;
let nativeTreeBase: string | null = null;

function getNativeTree(gsdDir: string): Map<string, GsdTreeEntry[]> | null {
  if (nativeTreeCache && nativeTreeBase === gsdDir) return nativeTreeCache;
  const entries = nativeScanGsdTree(gsdDir);
  // Build a map of parent directory -> entries
  const tree = new Map<string, GsdTreeEntry[]>();
  for (const entry of entries) {
    const parentPath = parts.slice(0, -1).join('/');
    if (!tree.has(parentPath)) tree.set(parentPath, []);
    tree.get(parentPath)!.push(entry);
  }
  nativeTreeCache = tree;
  return tree;
}
```

Directory listings are served from memory instead of individual `readdirSync` calls [repo code].

### File Parsing Layer

The file parsing layer in `src/resources/extensions/gsd/files.ts` provides parsers for roadmap, plan, summary, and continue files [repo code].

**Parse cache**:

```typescript
// src/resources/extensions/gsd/files.ts [repo code]
function cacheKey(content: string): string {
  const len = content.length;
  const head = content.slice(0, 100);
  const midStart = Math.max(0, Math.floor(len / 2) - 50);
  const mid = len > 200 ? content.slice(midStart, midStart + 100) : '';
  const tail = len > 100 ? content.slice(-100) : '';
  return `${len}:${head}:${mid}:${tail}`;
}

const _parseCache = new Map<string, unknown>();

function cachedParse<T>(content: string, tag: string, parseFn: (c: string) => T): T {
  const key = tag + '|' + cacheKey(content);
  if (_parseCache.has(key)) return _parseCache.get(key) as T;
  if (_parseCache.size >= CACHE_MAX) _parseCache.clear();
  const result = parseFn(content);
  _parseCache.set(key, result);
  return result;
}
```

The cache key combines length with samples from head, middle, and tail to detect changes anywhere in the file (not just endpoints) [repo code].

**Native parser bridge**:

Each parser tries the native Rust parser first for performance:

```typescript
// src/resources/extensions/gsd/files.ts [repo code]
export function parseRoadmap(content: string): Roadmap {
  return cachedParse(content, 'roadmap', _parseRoadmapImpl);
}

function _parseRoadmapImpl(content: string): Roadmap {
  // Try native parser first for better performance
  const nativeResult = nativeParseRoadmap(content);
  if (nativeResult) {
    return nativeResult;
  }
  // Fall back to JavaScript implementation
  // ...
}
```

**Roadmap parser**:

```typescript
// src/resources/extensions/gsd/files.ts [repo code]
function _parseRoadmapImpl(content: string): Roadmap {
  const lines = content.split('\n');
  const h1 = lines.find(l => l.startsWith('# '));
  const title = h1 ? h1.slice(2).trim() : '';
  const vision = extractBoldField(content, 'Vision') || '';
  const scSection = extractSection(content, 'Success Criteria', 2);
  const successCriteria = scSection ? parseBullets(scSection) : [];
  const slices = parseRoadmapSlices(content);
  // ... boundary map parsing ...
  return { title, vision, successCriteria, slices, boundaryMap };
}
```

**Plan parser**:

```typescript
// src/resources/extensions/gsd/files.ts [repo code]
export function parsePlan(content: string): SlicePlan {
  return cachedParse(content, 'plan', _parsePlanImpl);
}

function _parsePlanImpl(content: string): SlicePlan {
  const [, body] = splitFrontmatter(content);
  const nativeResult = nativeParsePlanFile(body);
  if (nativeResult) return nativeResult; // ...map to SlicePlan...

  // JavaScript fallback: parse tasks from checkbox format
  const cbMatch = line.match(/^-\s+\[([ xX])\]\s+\*\*([\w.]+):\s+(.+?)\*\*\s*(.*)/);
  // or heading format: ### T01 -- Title
  const hdMatch = line.match(/^#{2,4}\s+([\w.]+)\s*(?:--|—|:)\s*(.+)/);
  // ...
}
```

**Summary parser**:

```typescript
// src/resources/extensions/gsd/files.ts [repo code]
export function parseSummary(content: string): Summary {
  return cachedParse(content, 'summary', _parseSummaryImpl);
}

function _parseSummaryImpl(content: string): Summary {
  const nativeResult = nativeParseSummaryFile(content);
  if (nativeResult) return nativeResult; // ...map to Summary...

  // JavaScript fallback
  const [fmLines, body] = splitFrontmatter(content);
  const fm = fmLines ? parseFrontmatterMap(fmLines) : {};
  const frontmatter: SummaryFrontmatter = {
    id: (fm.id as string) || '',
    parent: (fm.parent as string) || '',
    // ... other fields ...
    blocker_discovered: fm.blocker_discovered === 'true',
  };
  // ...
}
```

**Section extraction helpers**:

```typescript
// src/resources/extensions/gsd/files.ts [repo code]
export function extractSection(body: string, heading: string, level: number = 2): string | null {
  const nativeResult = nativeExtractSection(body, heading, level);
  if (nativeResult !== NATIVE_UNAVAILABLE) return nativeResult as string | null;

  const prefix = '#'.repeat(level) + ' ';
  const regex = new RegExp(`^${prefix}${escapeRegex(heading)}\\s*$`, 'm');
  const match = regex.exec(body);
  if (!match) return null;
  // Extract content up to next heading of same or higher level
  // ...
}

export function extractAllSections(body: string, level: number = 2): Map<string, string> {
  const prefix = '#'.repeat(level) + ' ';
  const regex = new RegExp(`^${prefix}(.+)$`, 'gm');
  // Build heading → content map
  // ...
}
```

### `.gsd/` Artifact Structure

The `.gsd/` directory contains the complete workflow state as a structured set of markdown files:

**Root files** (`/ .gsd/`):

| File | Purpose |
|------|---------|
| `PROJECT.md` | Project context, constraints, and conventions |
| `DECISIONS.md` | Architectural, pattern, library, and observability decisions |
| `REQUIREMENTS.md` | Validated requirements with status tracking |
| `KNOWLEDGE.md` | Project-specific rules, patterns, and lessons learned |
| `STATE.md` | Cached output of `deriveState()` (optional) |
| `QUEUE.md` | Queued milestones awaiting activation |
| `OVERRIDES.md` | User-issued overrides that supersede plan content |

**Milestone files** (`.gsd/milestones/M###/`):

| File | Purpose |
|------|---------|
| `M###-CONTEXT.md` | Milestone context, scope, and objectives |
| `M###-CONTEXT-DRAFT.md` | Draft context awaiting discussion |
| `M###-ROADMAP.md` | Slice definitions with checkboxes and dependencies |
| `M###-RESEARCH.md` | Research findings for the milestone |
| `M###-SUMMARY.md` | Terminal artifact — milestone completion summary |
| `M###-VALIDATION.md` | Validation report with verdict |
| `M###-SECRETS.md` | Secrets manifest for env var collection |
| `M###-PARKED.md` | Marker file for parked milestones |
| `META.json` | Metadata (timestamps, status) |

**Slice files** (`.gsd/milestones/M###/slices/S##/`):

| File | Purpose |
|------|---------|
| `S##-PLAN.md` | Slice plan with goal, demo, must-haves, and tasks |
| `S##-RESEARCH.md` | Slice-specific research findings |
| `S##-SUMMARY.md` | Slice completion summary |
| `S##-CONTINUE.md` | Interrupted work state for resume |
| `S##-REPLAN.md` | Replanning results after blocker |
| `S##-REPLAN-TRIGGER.md` | Triage-initiated replan marker |
| `tasks/` | Task plan and summary files |

**Task files** (`.gsd/milestones/M###/slices/S##/tasks/`):

| File | Purpose |
|------|---------|
| `T##-PLAN.md` | Task plan with steps, inputs, outputs |
| `T##-SUMMARY.md` | Task completion summary with frontmatter |

**Frontmatter fields for task summaries**:

```yaml
---
id: T03
parent: S01
milestone: M001
provides:
  - What this task provides for downstream units
key_files:
  - Path to primary file created/modified
key_decisions:
  - Decision made during this task
patterns_established:
  - Pattern discovered or established
observability_surfaces:
  - Status endpoint, log, or diagnostic command
duration: 15m
verification_result: passed
completed_at: 2026-03-22T15:30:00Z
blocker_discovered: false  # Set true only if plan is fundamentally invalid
---
```

**Artifacts as workflow state**:

The key insight is that every phase transition is encoded as file presence/content:

- Slice `done: true` checkbox → ROADMAP.md updated
- Task `done: true` checkbox → S##-PLAN.md updated
- `blocker_discovered: true` in T##-SUMMARY.md → triggers replanning-slice phase
- M###-SUMMARY.md exists → milestone is complete regardless of roadmap checkboxes (#864)
- M###-PARKED.md exists → milestone is not eligible for activation

This disk-based state model means:
1. **Crash resilience**: Any interrupted session can resume by re-reading disk
2. **Clean restarts**: No in-memory state to corrupt
3. **Parallel workers**: Each worker can operate on isolated milestones
4. **Auditability**: Every state transition leaves a file artifact

---

## Subject-vs-Runner Guardrail

> **Critical**: This document analyzes GSD-2 as a *subject system*. It is produced *inside* a live GSD run in a fork of GSD-2. This creates a real conflation risk for `.gsd/` artifacts specifically.

### The Distinction

| Concept | What It Means | Where to Find It |
|---------|---------------|------------------|
| **Subject** | The GSD-2 system being analyzed | The analyzed `.gsd/` workflow model, `deriveState()`, path resolution, file parsing |
| **Runner** | The live GSD execution producing this documentation | The current reverse-engineering run's `.gsd/` planning artifacts |

### Concrete Example: `.gsd/` Artifacts

The analyzed GSD-2 system stores workflow state in `.gsd/milestones/` directories containing ROADMAP.md, CONTEXT.md, SLICE-PLAN.md files [repo code]. Each milestone has its own worktree under `.gsd/worktrees/` for execution isolation [repo code]. The `deriveState()` function reads these artifacts on each dispatch cycle to determine the current workflow position [repo code].

**This reverse-engineering run** (the runner) has its own `.gsd/` artifacts that are producing this documentation:
- The planning artifacts are in `.gsd/worktrees/M001/.gsd/milestones/M001/` [repo doc]
- The ROADMAP.md defines slices S01-S07 for the decomposition pack [repo doc]
- The slice plans define tasks T01, T02, etc. for each analytical step [repo doc]

These runner artifacts are **not the subject of analysis**. When this document describes `deriveState()` reading ROADMAP.md, it refers to the analyzed GSD-2 system's behavior — not to the live M001 roadmap that defines this analysis project.

### Why This Matters for Context Engineering

The disk-state workflow model documented here is particularly susceptible to conflation because:

1. **Same file names, different roles**: Both subject and runner have ROADMAP.md, S##-PLAN.md, T##-SUMMARY.md files. The names are identical but the content serves completely different purposes.

2. **Self-referential complexity**: This document describes how GSD-2 builds prompts that *describe `.gsd/` artifacts*. Those descriptions are themselves `.gsd/` artifacts. The recursion is real.

3. **Evidence from subject, not runner**: When this document shows `deriveState()` parsing ROADMAP.md [repo code], the evidence comes from the GSD-2 source code, not from the runner's current state.

### Guardrail Statement

> **[D007]** Throughout this document, references to `.gsd/`, milestones, slices, tasks, roadmaps, and workflow state refer to the **analyzed GSD-2 system** unless explicitly noted otherwise. The `.gsd/` artifacts producing this analysis (the runner) are separate from the `.gsd/` model being analyzed (the subject).

### Avoiding Conflation in This Document

This document uses these conventions:

- **"analyzed GSD-2"** when referring to the subject system's architecture and behavior
- **"this reverse-engineering run"** when referring to the live execution context
- **Explicit qualification** of `.gsd/` references when context is ambiguous:
  
  ```markdown
  # Good
  The analyzed GSD-2 stores workflow state in `.gsd/milestones/` [repo code].
  This reverse-engineering run's own `.gsd/` artifacts are in `.gsd/worktrees/M001/`.
  
  # Avoid
  GSD stores state in `.gsd/`. (Ambiguous — subject or runner?)
  ```

- **Links to EVIDENCE_METHOD.md** when introducing analysis that touches `.gsd/` artifacts

---

## Cross-References

- **S02 Runtime Architecture**: See [GSD2_RUNTIME_ARCHITECTURE.md](./GSD2_RUNTIME_ARCHITECTURE.md) for session context and lifecycle.
- **S03 Orchestration Layer**: See [GSD2_ORCHESTRATION_LAYER.md](./GSD2_ORCHESTRATION_LAYER.md) for phase transitions and dispatch logic.
- **Evidence Method**: See [EVIDENCE_METHOD.md](./EVIDENCE_METHOD.md) for evidence tier definitions and the subject-vs-runner guardrail.

---

## Summary

The context assembly pipeline is a two-layer system: Pi-owned `buildSystemPrompt()` provides the foundational system prompt with tool descriptions and guidelines, while GSD-authored unit-specific builders add workflow intelligence through template loading, file inlining, and context budgeting. The budget system ensures prompts fit within the executor's context window using proportional allocation and section-boundary truncation. Template caching prevents mid-session crashes when templates are updated on disk.

All references to `.gsd/` in this document denote the **analyzed GSD-2 system**, not the live runner state producing this documentation.
