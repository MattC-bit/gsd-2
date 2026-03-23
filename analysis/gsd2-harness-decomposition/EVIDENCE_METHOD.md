# Evidence Method

This document establishes the methodological contract for the GSD-2 Harness Decomposition Pack. All subsequent analysis documents must follow these conventions to maintain evidence integrity and reader trust.

---

## Source Precedence

Claims in this pack follow a strict source precedence hierarchy (per Decision D002):

1. **Repo code** — Primary source of truth. TypeScript source files, configuration, and tests under the GSD-2 repository take precedence over all other sources.

2. **Repo docs** — Secondary source. README files, inline documentation, and official docs are treated as supporting evidence but may lag behind or diverge from actual code behavior.

3. **External context** — Tertiary source. Comparisons with other systems, prior art, and external documentation are informative but never override repo evidence.

4. **Synthesis** — Labeled as inference. Cross-system patterns, architectural abstractions, and extrapolations must be explicitly labeled and never presented as repo-grounded fact.

**Why this matters:** This pack is meant to support real architectural decisions later. Evidence strength must stay explicit so readers can weigh claims appropriately.

---

## Evidence Tiers

Every factual claim in this pack maps to one of four evidence tiers:

| Tier | Source | Label | Weight | Example |
|------|--------|-------|--------|---------|
| **Tier 1** | Repo code | `[repo code]` | Highest | "The `Session` class in `packages/pi-agent-core/session.ts` exports `createSession()`" |
| **Tier 2** | Repo docs | `[repo doc]` | High | "The README states that GSD uses `.gsd/` for workflow state" |
| **Tier 3** | External | `[external]` | Medium | "LangChain implements a similar planner-executor pattern" |
| **Tier 4** | Inference | `[inference]` | Low | "This pattern suggests GSD treats milestones as isolated execution units" |

**Labeling in practice:**

```markdown
The dispatcher selects slices by matching rules against the current context [repo code].
This is consistent with the workflow model described in the architecture docs [repo doc].
Similar to how Maestro handles task routing [external].
The separation appears designed to support parallel execution [inference].
```

**Unresolved claims:** When evidence is insufficient to ground a claim, use `[unresolved]`:

```markdown
The exact trigger conditions for auto-mode phase transitions remain unclear [unresolved].
```

---

## Fact vs Inference Labeling

All documents in this pack must label claims according to their evidence tier:

### Required Labels

| Label | Meaning | When to Use |
|-------|---------|-------------|
| `[repo code]` | Directly observed in repository source | When citing specific files, functions, types, or code paths |
| `[repo doc]` | Found in repository documentation | When citing README, inline docs, or official docs |
| `[external]` | Sourced from outside the repository | When comparing to other systems or citing external references |
| `[inference]` | Synthesized or extrapolated | When drawing architectural conclusions from multiple sources |
| `[unresolved]` | Evidence insufficient | When a claim cannot be grounded and needs further investigation |

### Labeling Examples

**Code reference:**
```markdown
The `AutoMode` class in `src/resources/extensions/gsd/auto-mode.ts` exports `runAutoMode()` 
which orchestrates the dispatch loop [repo code].
```

**Doc reference:**
```markdown
According to `.gsd/PROJECT.md`, this project is produced from inside a live GSD run 
in a fork of GSD-2 [repo doc].
```

**External comparison:**
```markdown
This dispatch pattern resembles the ReAct loop in LangGraph agents [external].
```

**Inference:**
```markdown
The separation between Pi packages and GSD extension layer suggests a deliberate 
plugin architecture where GSD is one possible orchestration implementation [inference].
```

**Unresolved:**
```markdown
The relationship between worktree lifecycle and session cleanup is not explicit 
in the codebase [unresolved].
```

---

## Subject-vs-Runner Guardrail

> **Critical:** This pack analyzes GSD-2 as a *subject system*. It is produced *inside* a live GSD run in a fork of GSD-2. This creates a real conflation risk. The subject-vs-runner separation must be maintained throughout all pack documents.

### The Distinction

| Concept | What It Means | Where to Find It |
|---------|---------------|------------------|
| **Subject** | The GSD-2 system being analyzed | The analyzed `.gsd/` workflow model, GSD source code, GSD architecture |
| **Runner** | The live GSD execution producing this pack | The current reverse-engineering run's `.gsd/` planning artifacts |

### Guardrail Statement

> **[D007]** Throughout this pack, references to `.gsd/`, milestones, slices, and workflow state refer to the **analyzed GSD-2 system** unless explicitly noted otherwise. The `.gsd/` artifacts in this repository that are *producing* this analysis (the runner) are separate from the `.gsd/` model being *analyzed* (the subject).

### How to Avoid Conflation

1. **Use "analyzed GSD-2"** when referring to the subject system's architecture and behavior.
2. **Use "this reverse-engineering run"** when referring to the live execution context.
3. **Explicitly qualify `.gsd/` references** when context is ambiguous:

   ```markdown
   # Good
   The analyzed GSD-2 stores workflow state in `.gsd/milestones/` [repo code].
   This reverse-engineering run's own `.gsd/` artifacts are in the worktree at 
   `.gsd/worktrees/M001/` [repo doc].
   
   # Avoid
   GSD stores state in `.gsd/`. (Ambiguous — subject or runner?)
   ```

4. **Link to this guardrail** when introducing analysis that touches `.gsd/` artifacts.

---

## Cross-Reference Conventions

### Citing Files

Use relative paths from the repository root:

```markdown
The `Session` class is defined in `packages/pi-agent-core/session.ts` [repo code].
```

### Citing Decisions

Reference decisions from `.gsd/DECISIONS.md` by ID:

```markdown
Per Decision D002, repo code takes precedence over repo docs for evidence.
```

### Linking Pack Documents

Use relative links within the pack:

```markdown
See [EVIDENCE_METHOD.md](./EVIDENCE_METHOD.md) for labeling conventions.
The [README.md](./README.md) provides the full document index.
```

### Cross-System Synthesis

When drawing conclusions that span multiple sources:

```markdown
The dispatch loop in `src/resources/extensions/gsd/auto-mode.ts` [repo code] 
matches the workflow model described in the architecture docs [repo doc] 
and resembles the ReAct pattern from LangChain [external]. 
This suggests GSD implements a planner-executor architecture with 
explicit phase transitions [inference].
```

---

## Summary Checklist

Before submitting any analysis document, verify:

- [ ] All claims are labeled with evidence tier
- [ ] Repo code claims cite specific files
- [ ] Inference claims are not presented as fact
- [ ] Unresolved claims are marked `[unresolved]`
- [ ] `.gsd/` references distinguish subject from runner
- [ ] Cross-references use consistent conventions
- [ ] Claims reference relevant decisions (D001-D007)

---

## Decision References

| ID | Scope | Relevance to This Document |
|----|-------|----------------------------|
| D001 | Subject of analysis | Defines GSD-2 as harness/orchestration system |
| D002 | Source precedence | Establishes code > docs > external > inference |
| D003 | Workspace location | Defines `analysis/gsd2-harness-decomposition/` |
| D004 | Terminology strategy | Preserve GSD terms, add Atlas via glossary |
| D005 | Milestone shape | One milestone with seven analytical slices |
| D006 | Comparative boundary | Keep comparison separate from synthesis |
| D007 | Subject-vs-runner | Guardrail against `.gsd/` conflation |
