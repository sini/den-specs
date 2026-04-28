# Spec Implementation Audit — Design

**Date:** 2026-04-28
**Scope:** All 45 implemented specs (20 main, 25 branch)
**Output:** Per-spec `.analysis.md` files + executive `SUMMARY.md`

## Goal

Evaluate every spec classified as "implemented" (on main or feat/fx-pipeline) for:
1. Implementation accuracy
2. Gaps between spec and implementation
3. Drift from spec intent
4. Supersession chains between specs
5. Current status (still exists, replaced, simplified out)

## Evaluation Checklist (per spec)

| Field | Description |
|-------|-------------|
| **Verdict** | implemented / partially-implemented / superseded / not-implemented |
| **Delivery target** | (A) main or (B) feat/fx-pipeline |
| **Evidence** | Commits, files, functions proving implementation |
| **Current status** | Still exists / replaced by X / simplified out in Y |
| **Supersession** | Which spec replaced this, or which this replaced |
| **Gaps** | Proposed features that didn't ship or shipped differently |
| **Drift** | Where implementation diverged from spec intent |

## Three-Phase Execution

### Phase 1: Per-Spec Analysis (45 sonnet agents, parallel)

Each agent receives:
- The spec file content
- Access to grep/glob the repo for proposed functions, effects, files
- Access to `git log --all --oneline` for implementation commits
- The analysis template

Each agent writes `<spec-name>.analysis.md` next to its spec.

### Phase 2: Batch Review (5 sonnet agents, parallel)

Each agent reads all specs + analyses within its component group, cross-references supersession chains, corrects misattributions, updates analysis files.

| Batch | Components | Specs |
|-------|------------|-------|
| 1 | aspect-rewrites (main) + hasAspect (main) | 5 |
| 2 | fx-pipeline (main) | 14 |
| 3 | legacy-removal (branch) + policies (branch) | 13 |
| 4 | stages (branch) + traits (branch) + class-modules (branch) | 8 |
| 5 | pipeline-simplification (branch) + provides-removal (branch) | 4 |

### Phase 3: Synthesis (1 sonnet agent)

Reads all 45 finalized analyses, produces `~/Documents/den-specs/SUMMARY.md`:
- Overview table (spec, verdict, target, status, one-line)
- Supersession chains
- Gap inventory
- Component-level summaries

## Constraints

- All agents read-only against repo
- Analysis files written to `~/Documents/den-specs/` only
- Sonnet model, caveman lite style for token efficiency
