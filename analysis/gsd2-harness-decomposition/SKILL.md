---
name: gsd2-harness-pack
description: >
  Parse and query GSD2_HARNESS_PACK.xml — a 323KB XML context file containing
  the complete GSD-2 agent harness decomposition. Use when an agent needs to
  understand GSD-2's architecture (runtime, orchestration, context engineering,
  git isolation, comparative analysis) or apply the rebuild/wrap/delegate
  decisions for Atlas-style harness construction.
---

<objective>
GSD2_HARNESS_PACK.xml is a compiled, single-file context artifact containing 9
source-grounded analysis documents about GSD-2 as an agent harness system. This
skill teaches agents how to load it efficiently, navigate its structure, and
extract targeted answers without processing the full 323KB payload.

Use cases:
- Understand GSD-2's runtime architecture before extending or wrapping it
- Apply the rebuild/wrap/delegate decision table to Atlas harness construction
- Resolve GSD-2 terminology via the normalized glossary (59 entries)
- Understand the disk-state workflow model before building equivalent state machines
- Verify architectural claims against source-grounded evidence tiers
</objective>

<quick_start>
The pack has a predictable structure. Navigate by document id, never by line number.

**Document IDs (in reading order):**
```
evidence_method         — Evidence tiers and subject-vs-runner guardrail (read first)
system_overview         — Pi vs GSD boundary inventory, repo structure
runtime_architecture    — Session lifecycle, JSONL, event flow, retry/compaction
orchestration_layer     — Auto-mode, dispatch rules, verification gates, tool registration
context_engineering     — Prompt assembly, templates, context budgets, disk-state model
git_isolation_model     — Worktrees, branch naming, session locking, parallel orchestrator
comparative_analysis    — GSD-2 vs Claude Code, Codex CLI, ACP
glossary                — 59-entry GSD→Atlas terminology crosswalk
harness_model           — Rebuild/wrap/delegate synthesis per architectural layer
```

**XPath to jump directly to a document:**
```xpath
//document[@id="harness_model"]/content
```

**Python one-liner to extract a document:**
```python
import xml.etree.ElementTree as ET
tree = ET.parse("GSD2_HARNESS_PACK.xml")
doc = tree.find('.//document[@id="harness_model"]')
print(doc.find('content').text)
```
</quick_start>

<file_structure>
```xml
<gsd2_harness_pack version="1.0" generated="...">

  <toc>                          <!-- Reading order index with per-document summaries -->
    <entry id="..." title="..."> <!-- One entry per document -->
      One-sentence description
    </entry>
  </toc>

  <evidence_system>              <!-- Evidence tier legend — read before any claims -->
    <tier label="[repo code]" weight="highest"/>
    <tier label="[repo doc]"  weight="high"/>
    <tier label="[external]"  weight="medium"/>
    <tier label="[inference]" weight="low"/>
    <tier label="[unresolved]" weight="none"/>
    <guardrail name="subject_vs_runner">...</guardrail>
  </evidence_system>

  <document id="evidence_method" source="EVIDENCE_METHOD.md" lines="198" bytes="7795">
    <title>...</title>
    <summary>...</summary>
    <content><![CDATA[
      ... full markdown content ...
    ]]></content>
  </document>

  <!-- 8 more <document> blocks in reading order -->

  <pack_stats documents="9" total_lines="6438" total_bytes="324371"/>

</gsd2_harness_pack>
```

Content is wrapped in `<![CDATA[...]]>` blocks — all markdown, code fences, and
special characters are preserved as-is. No HTML-escaping is applied inside CDATA.
</file_structure>

<parsing_strategy>
**Token-efficient loading — do not load the full pack unless you need all 9 documents.**

Match your load strategy to the task:

| Task | Documents to load |
|------|-------------------|
| What is GSD-2? | `system_overview` |
| How does the agent loop work? | `runtime_architecture` |
| How does auto-mode dispatch work? | `orchestration_layer` |
| How are prompts assembled? | `context_engineering` |
| How does worktree isolation work? | `git_isolation_model` |
| How does GSD-2 compare to Claude Code? | `comparative_analysis` |
| What does term X mean in Atlas terms? | `glossary` |
| What should Atlas rebuild vs delegate? | `harness_model` |
| How should I interpret claim strength? | `evidence_method` |
| Full architecture deep-dive | All 9 (sequential) |

**Extraction pattern (pseudocode):**
```
1. Parse XML → document tree (one-time cost)
2. Find <document id="TARGET"> node
3. Extract CDATA content from <content> child
4. Search within that content for the specific section
```

**Never:** Scan the full file as a string. The XML structure exists precisely to
avoid this. Parse first, then navigate by id.
</parsing_strategy>

<evidence_interpretation>
Every factual claim in the pack carries an evidence label. Interpret them as follows:

- `[repo code]` — Traced to a specific file and line in the GSD-2 repository.
  High confidence. Treat as fact unless you have newer source evidence.

- `[repo doc]` — Found in README or inline documentation. May lag code.
  Use as supporting evidence, not primary.

- `[external]` — Comparison with other systems (Claude Code, Codex, ACP).
  Informative but not GSD-2 source truth.

- `[inference]` — Cross-system synthesis or architectural abstraction.
  Explicitly speculative. Useful for design decisions; verify before implementation.

- `[unresolved]` — Known unknown. The pack explicitly marks gaps.

**The subject-vs-runner guardrail:** When the pack references `.gsd/`, milestones,
slices, and workflow state, it refers to the *analyzed GSD-2 system*, not the
live GSD session that produced this documentation. They are different things.
The pack was produced inside a GSD execution running on a fork of GSD-2 — do not
conflate the runner's artifacts with the subject's architecture.
</evidence_interpretation>

<use_cases>
<use_case name="atlas_build_decisions">
Load `harness_model`. It contains a Master Rebuild/Wrap/Delegate Table with 33
components across 6 layers. Each row has: component, layer, action, complexity,
and Atlas priority. Start here for any Atlas harness construction task.
</use_case>

<use_case name="term_lookup">
Load `glossary`. 59 entries organized by domain (Runtime, Orchestration, Workflow
State, Isolation, Context Engineering, Tool System, Comparative). Each entry has:
GSD-native definition, Atlas equivalent, reuse strategy, key capability, and
partial equivalence notes where concepts don't map cleanly.
</use_case>

<use_case name="dispatch_model">
Load `orchestration_layer`. The Dispatch Table section documents 18 declarative
rules in evaluation order with matching conditions and actions. The Phase→Unit
mapping diagram shows how disk state maps to execution phases.
</use_case>

<use_case name="session_persistence">
Load `runtime_architecture`. The Persistence Model section documents the JSONL
format, entry types, tree structure, blob externalization, and context reconstruction
logic — all `[repo code]` grounded.
</use_case>

<use_case name="worktree_lifecycle">
Load `git_isolation_model`. Covers createAutoWorktree, enterAutoWorktree,
teardownAutoWorktree, mergeMilestoneToMain with full sequence diagrams and
the three isolation modes (worktree/branch/none).
</use_case>

<use_case name="full_architecture">
Load documents in TOC order. Each document cross-references the others — start
with `evidence_method` to understand claim labeling, then `system_overview` for
structural orientation, then whichever layer you need.
</use_case>
</use_cases>

<anti_patterns>
- **Do not** grep the raw XML for keywords — parse the XML first, then search
  within the extracted document content.
- **Do not** load all 9 documents to answer a single-layer question — targeted
  extraction saves ~280KB of unnecessary context.
- **Do not** treat `[inference]` claims as implementation specs — they are
  design reasoning, not code-grounded facts.
- **Do not** conflate subject and runner `.gsd/` references — the guardrail in
  `evidence_method` explains why this matters.
</anti_patterns>

<success_criteria>
A correctly parsed session:
- Extracts the target document by id attribute, not by file position
- Interprets evidence labels before acting on claims
- Respects the subject-vs-runner guardrail when reasoning about `.gsd/` artifacts
- Loads only the documents needed for the task
</success_criteria>
