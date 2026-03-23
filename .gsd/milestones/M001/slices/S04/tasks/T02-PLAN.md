---
estimated_steps: 5
estimated_files: 5
skills_used:
  - best-practices
---

# T02: Document Disk-State Workflow Model and Phase Derivation

**Slice:** S04 — Context Engineering and Disk-State Workflow Model
**Milestone:** M001

## Description

Trace and document the disk-state workflow model from `deriveState()` as the source of truth. This covers the second half of S04's goal — understanding how `.gsd/` artifacts function as workflow state and how phase transitions work via disk artifacts.

## Steps

1. Read `src/resources/extensions/gsd/state.ts` and document:
   - `deriveState()` as the source of truth for workflow state
   - Phase derivation from disk artifacts (not mutable tracking)
   - Memoization cache (CACHE_TTL_MS = 100)
   - `getActiveMilestoneId()` for milestone selection
   - `isSliceComplete()` and `isMilestoneComplete()` helpers
   - Ghost milestone detection
   - State cache invalidation

2. Read `src/resources/extensions/gsd/paths.ts` and document:
   - Path resolution hierarchy: `gsdRoot()` → `milestonesDir()` → `resolveMilestonePath()` → `resolveSlicePath()` → `resolveTaskFile()`
   - Directory and file naming conventions (bare IDs: M001/, S01/, T##-SUFFIX.md)
   - Legacy compatibility handling (descriptor-suffixed names)
   - Relative path builders for prompts
   - Native tree cache for batch directory listing

3. Read `src/resources/extensions/gsd/files.ts` and document:
   - File parsing functions: `parseRoadmap()`, `parsePlan()`, `parseSummary()`, `parseContinue()`
   - Section extraction helpers: `extractSection()`, `extractAllSections()`
   - Native parser bridge for performance
   - Parse cache for repeated parsing

4. Document the `.gsd/` artifact structure:
   - Root files: PROJECT.md, DECISIONS.md, REQUIREMENTS.md, KNOWLEDGE.md, STATE.md
   - Milestone files: CONTEXT.md, ROADMAP.md, RESEARCH.md, SUMMARY.md, VALIDATION.md
   - Slice files: PLAN.md, RESEARCH.md, SUMMARY.md, CONTINUE.md, REPLAN.md
   - Task files: T##-PLAN.md, T##-SUMMARY.md

5. Write the "Disk-State Workflow Model" section in `GSD2_CONTEXT_ENGINEERING_MODEL.md` with:
   - Phase derivation flow diagram (text-based)
   - Path resolution hierarchy
   - File parsing layer
   - Code evidence with line references
   - Evidence tier labels (`[repo code]`)

## Must-Haves

- [ ] Document `deriveState()` as source of truth with code evidence
- [ ] Document path resolution hierarchy with code evidence
- [ ] Document file parsing layer with code evidence
- [ ] Document `.gsd/` artifact structure
- [ ] Use `[repo code]` evidence labels consistently

## Verification

- `grep -q "Disk-State Workflow Model" analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` returns 0
- `grep -c "\[repo code\]" analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` returns >= 20

## Inputs

- `src/resources/extensions/gsd/state.ts` — State derivation source of truth
- `src/resources/extensions/gsd/paths.ts` — Path resolution hierarchy
- `src/resources/extensions/gsd/files.ts` — File parsing utilities
- `analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` — Partial document from T01

## Expected Output

- `analysis/gsd2-harness-decomposition/GSD2_CONTEXT_ENGINEERING_MODEL.md` — Updated document with Disk-State Workflow Model section
