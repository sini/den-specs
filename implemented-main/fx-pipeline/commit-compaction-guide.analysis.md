# Analysis: Commit Compaction Guide — feat/fx-resolution

## Verdict

Spec was **not followed**. The proposed compaction into ~26 clean logical commits was bypassed entirely. PR #462 (`68c26555`) landed as a **single squash commit** on main: `refactor: den-fx (#462)`. All 131 branch commits (plus ~6 code-review chains 16-18 added later) were collapsed into one.

## Delivery Target

PR #462 (`68c26555 refactor: den-fx (#462)`) — merged 2026-04-17 by Jason Bowman. This is the one and only commit that represents all of feat/fx-resolution in main's history.

## Evidence

- `git show 68c26555 --format="%P"` returns a single parent (`927f4d8e`), confirming this is a squash merge, not a regular merge commit (which would have two parents).
- `git log 927f4d8e..68c26555` returns only `68c26555` itself — no branch commits preserved.
- The commit message for `68c26555` is a PR-style summary describing the full fx pipeline architecture (aspectToEffect compiler, constraint system, handler-owned recursion, etc.) — consistent with a GitHub squash-and-merge operation.
- The spec proposed 18 squash chains + ~12 standalone commits = ~26 resulting commits. The actual result: 1 commit.

## Current Status

The fx pipeline code is fully live on main inside the single squash commit `68c26555`. The architectural changes described in the compaction guide — aspectToEffect, constraint registry, chain provenance, handler-owned recursion, module split under `nix/lib/aspects/fx/` — are all present in the tree. The commit history is just non-granular.

Post-merge follow-up commits on main (`#463`, `#464`, `#465`) address immediate regressions (unresolvable includes in applyDeep, mixed function+attrset provides sub-aspects), not items from the compaction plan.

## Supersession

The compaction guide was written as preparation for a rebase/interactive-squash workflow. GitHub's squash-and-merge for PR #462 superseded it entirely. The guide's logical groupings were never realized as individual commits; the PR description for `68c26555` serves as the only structured summary.

feat/fx-pipeline (the current branch, branching from main post-`68c26555`) continues the work with its own commit sequence — 20+ commits covering policy simplification, pipeline phase E, and compat shims — none of which are covered by the compaction guide.

## Gaps

- The compaction guide covers only feat/fx-resolution (131 commits). The subsequent feat/fx-pipeline branch is not addressed.
- Chains 16–18 (code review commits from 2026-04-16, added as a late appendix) were also absorbed into the squash. There is no record of whether those 11 commits existed on the branch at merge time or were pre-squashed before the PR was submitted.
- The "Commits to DROP" list (16 entries) is now moot — all were absorbed or discarded silently by the squash.

## Drift

- **Intended**: ~26 clean commits, reviewable, bisectable, matching logical spec structure.
- **Actual**: 1 commit. Bisectability and granular blame are lost for the entire fx pipeline introduction.
- The spec's proposed commit subjects (e.g., "feat(fx): aspectToEffect compiler", "fix(fx): wrapChild normalization") do not appear in main's git log at all.
- No semantic drift in the *code* — the architecture described in the guide matches what landed. Drift is entirely in the commit history layer.
