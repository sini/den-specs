# Incremental Spec Audit

**Purpose:** Repeatable process for analyzing new/unaudited specs and integrating them into SUMMARY.md.

## When to Run

- After new specs are written and implemented
- After specs move between categories (tbd → implemented-branch)
- When `find ~/Documents/den-specs -name "*.md" ! -name "*.analysis.md" | while read f; do [ ! -f "${f%.md}.analysis.md" ] && echo "$f"; done` returns results

## Phase 1: Generate Analysis Files

For each spec missing an `.analysis.md`, dispatch a sonnet agent with:

**Inputs:**
- The spec file content
- Access to grep/glob the repo for proposed functions, effects, files
- Access to `git log --all --oneline` for implementation commits

**Agent prompt template:**
```
Read the spec at <path>. Analyze its implementation status in the repo at
~/Documents/repos/den. Write a <name>.analysis.md file next to the spec.

Use caveman lite style. Follow this structure exactly:

# Analysis: <Spec Title>

## Verdict
<implemented / partially-implemented / superseded / not-implemented / in-progress>
<2-3 sentence summary of what shipped vs what didn't>

## Delivery Target
<main / feat/fx-pipeline / nix-effects / not delivered>

## Evidence
<Grep/glob findings proving implementation. Include file paths and line numbers.
For each major spec proposal, state whether it exists in code.>

## Current Status
<Still exists / replaced by X / simplified out in Y>

## Supersession
<Which spec replaced this, or which this replaced, or "none">

## Gaps
<Proposed features that didn't ship or shipped differently>

## Drift
<Where implementation diverged from spec intent>

Constraints:
- Read-only against repo (grep, glob, read, git log only)
- Write only the .analysis.md file in ~/Documents/den-specs/
- Be specific: cite file paths, line numbers, commit hashes
```

**Batch size:** Up to 5 parallel sonnet agents. Group by category for coherent supersession analysis.

## Phase 2: SUMMARY.md Integration

After analysis files land, dispatch one sonnet agent that:

1. Reads all NEW `.analysis.md` files from Phase 1
2. Reads current `SUMMARY.md`
3. For specs not yet in the overview table: adds rows (sorted by date)
4. For specs already in the table: updates verdict/status if analysis differs
5. Cross-references supersession chains — adds new chains or extends existing ones
6. Updates gap inventory (§4a, §4b, §4c) if analysis reveals new gaps
7. Updates component summaries (§6) if new specs affect a component
8. Updates verdict distribution count (§2)

**Write changes directly to SUMMARY.md.**

## Phase 3: Human Review

Review the SUMMARY.md diff. Check:
- Are verdicts accurate?
- Are supersession chains correct?
- Any gaps that should be in memory but aren't?

## Execution Log

Track completed runs here:

| Date | Specs analyzed | Batch | Notes |
|------|---------------|-------|-------|
| 2026-04-28 | 44 original specs | Full audit | Initial DESIGN-spec-audit.md run |
| 2026-04-29 | 5 implemented specs | Batch 1 | DLQ, enrichment, flake-scope, forward-elim, compat-shims — all analyzed + SUMMARY.md integrated |
| (pending) | 10 tbd/vision specs | Batch 2 | unified-aspect-key, class-dedup, provides-removal-post, scope-partitioned, onlyIf-adapter, recursive-aspects-vision, 4× diag |
