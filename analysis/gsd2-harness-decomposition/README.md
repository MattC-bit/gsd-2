# GSD-2 Harness Decomposition Pack

## Purpose

This pack is a source-grounded analysis of GSD-2 as an agent harness/orchestration system. It explains how GSD-2 actually works — with explicit boundaries between Pi-owned runtime capabilities and GSD-authored orchestration logic — to support later Atlas-oriented rebuild/wrap/delegate decisions.

The goal is **reverse-engineering the control architecture and workflow model**, not planning new product features.

## Intended Audience

Atlas harness builders who need to understand:
- What GSD-2 provides as a runtime/orchestration layer
- Which capabilities belong to Pi (underlying runtime) vs GSD (orchestration logic)
- What an Atlas-specific harness should rebuild, wrap, or delegate

## How to Read This Pack

**Suggested order:**

1. **[EVIDENCE_METHOD.md](./EVIDENCE_METHOD.md)** — Read this first. It establishes the source precedence rules, evidence tiers, labeling conventions, and the subject-vs-runner guardrail that govern all claims in this pack.

2. **[GSD2_SYSTEM_OVERVIEW.md](./GSD2_SYSTEM_OVERVIEW.md)** — High-level architecture with explicit Pi-vs-GSD boundary inventory and pack document map.

3. **[GSD2_RUNTIME_ARCHITECTURE.md](./GSD2_RUNTIME_ARCHITECTURE.md)** — Session lifecycle, JSONL conversation store, config resolution, crash recovery, compaction.

4. **[GSD2_ORCHESTRATION_LAYER.md](./GSD2_ORCHESTRATION_LAYER.md)** — Auto-mode state machine, dispatch rules, phase derivation, verification gates, retry logic.

5. **[GSD2_CONTEXT_ENGINEERING_MODEL.md](./GSD2_CONTEXT_ENGINEERING_MODEL.md)** — Prompt assembly, disk-state as persistent context, KNOWLEDGE.md/STATE.md roles.

6. **[GSD2_GIT_AND_ISOLATION_MODEL.md](./GSD2_GIT_AND_ISOLATION_MODEL.md)** — Worktree-based isolation, branch-per-milestone, squash merge, worktree `.gsd/` separation.

7. **[GSD2_COMPARATIVE_ANALYSIS.md](./GSD2_COMPARATIVE_ANALYSIS.md)** — GSD-2 vs Claude Code, Codex CLI, and ACP: shared primitives and divergence points.

8. **[GLOSSARY_NORMALIZED_TERMS.md](./GLOSSARY_NORMALIZED_TERMS.md)** — 59-entry GSD→Atlas terminology mapping.

9. **[EMERGING_AGENT_HARNESS_MODEL.md](./EMERGING_AGENT_HARNESS_MODEL.md)** — Rebuild/wrap/delegate decisions per architectural layer for Atlas harness builders.

## Document Index

| Document | Status | Lines | Description |
|----------|--------|-------|-------------|
| [EVIDENCE_METHOD.md](./EVIDENCE_METHOD.md) | Complete | 198 | Evidence method and source precedence rules |
| [GSD2_SYSTEM_OVERVIEW.md](./GSD2_SYSTEM_OVERVIEW.md) | Complete | 270 | High-level system overview with Pi-vs-GSD boundaries |
| [GSD2_RUNTIME_ARCHITECTURE.md](./GSD2_RUNTIME_ARCHITECTURE.md) | Complete | 851 | Session creation, configuration, persistence, recovery |
| [GSD2_ORCHESTRATION_LAYER.md](./GSD2_ORCHESTRATION_LAYER.md) | Complete | 815 | Dispatch, auto-mode, verification, guards, execution flow |
| [GSD2_CONTEXT_ENGINEERING_MODEL.md](./GSD2_CONTEXT_ENGINEERING_MODEL.md) | Complete | 1094 | Prompt assembly and `.gsd/` disk-state model |
| [GSD2_GIT_AND_ISOLATION_MODEL.md](./GSD2_GIT_AND_ISOLATION_MODEL.md) | Complete | 1290 | Branch/worktree isolation and milestone execution |
| [GSD2_COMPARATIVE_ANALYSIS.md](./GSD2_COMPARATIVE_ANALYSIS.md) | Complete | 645 | Comparison with adjacent systems for shared primitives |
| [GLOSSARY_NORMALIZED_TERMS.md](./GLOSSARY_NORMALIZED_TERMS.md) | Complete | 825 | GSD-to-Atlas terminology mapping (59 entries) |
| [EMERGING_AGENT_HARNESS_MODEL.md](./EMERGING_AGENT_HARNESS_MODEL.md) | Complete | 450 | Rebuild/wrap/delegate decisions for Atlas harness |

## Conventions

All documents in this pack follow the conventions established in `EVIDENCE_METHOD.md`:
- Source precedence: repo code > repo docs > external > inference
- Evidence tiers with explicit labeling
- Subject-vs-runner guardrail (analyzed GSD-2 vs live reverse-engineering run)

## Relationship to Upstream Docs

This pack is **separate from** upstream product documentation. It is a reverse-engineering artifact produced inside a fork of GSD-2, intended for Atlas harness builders, not for GSD users or contributors.
