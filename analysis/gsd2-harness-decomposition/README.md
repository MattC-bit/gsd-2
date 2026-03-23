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

3. Remaining documents (to be produced by subsequent slices) cover runtime lifecycle, orchestration layer, context engineering, git isolation, comparative analysis, and Atlas synthesis.

## Document Index

| Document | Status | Description |
|----------|--------|-------------|
| [EVIDENCE_METHOD.md](./EVIDENCE_METHOD.md) | Complete | Evidence method and source precedence rules |
| [GSD2_SYSTEM_OVERVIEW.md](./GSD2_SYSTEM_OVERVIEW.md) | Complete | High-level system overview with Pi-vs-GSD boundaries |
| GSD2_RUNTIME_ARCHITECTURE.md | Planned | Session creation, configuration, persistence, recovery |
| GSD2_ORCHESTRATION_LAYER.md | Planned | Dispatch, auto-mode, verification, guards, execution flow |
| GSD2_CONTEXT_ENGINEERING_MODEL.md | Planned | Prompt assembly and `.gsd/` disk-state model |
| GSD2_GIT_AND_ISOLATION_MODEL.md | Planned | Branch/worktree isolation and milestone execution |
| GSD2_COMPARATIVE_ANALYSIS.md | Planned | Comparison with adjacent systems for shared primitives |
| GLOSSARY_NORMALIZED_TERMS.md | Planned | GSD-to-Atlas terminology mapping |
| EMERGING_AGENT_HARNESS_MODEL.md | Planned | Rebuild/wrap/delegate decisions for Atlas harness |

## Conventions

All documents in this pack follow the conventions established in `EVIDENCE_METHOD.md`:
- Source precedence: repo code > repo docs > external > inference
- Evidence tiers with explicit labeling
- Subject-vs-runner guardrail (analyzed GSD-2 vs live reverse-engineering run)

## Relationship to Upstream Docs

This pack is **separate from** upstream product documentation. It is a reverse-engineering artifact produced inside a fork of GSD-2, intended for Atlas harness builders, not for GSD users or contributors.
