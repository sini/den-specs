# Analysis: Provides Removal — Complete the Pipeline Simplification

## Verdict

Partial implementation. The machinery side has been restructured with a compat shim layer, but the spec's primary deliverable — deleting `emitSelfProvide`, `emitCrossProvider`, `mkPositionalInclude`, `mkNamedInclude`, and the `provides` option — has NOT landed. The compat shim approach diverges from the spec's direct-deletion plan.

## Delivery Target

feat/fx-pipeline branch. Spec was authored against feat/traits; current branch is the successor.

## Evidence

**Git log relevant commits** (newest first):
- `2e40b024` feat: add mutual-provider compat shim
- `c52a3779` feat: wire emitCrossProvideShims into resolveChildren bind chain
- `19738452` feat: add provides-compat handler for main-era cross-provide shims
- `12fed6aa` refactor: remove dead synthesize/mergePolicyInto and stale mutual-provider references
- `11625e47` feat: replace mutual-provider with aspect-included policyFns
- `20acb884` fix: __functor conflict in providerType.merge, add provides-path e2e test
- `a01bc53a` feat: provides deprecation shim, rewrite to direct nesting
- `f50fd576` feat: direct nesting for aspects, deprecation trace on provides

**Machinery still present** (`nix/lib/aspects/fx/aspect.nix`):
- `emitSelfProvide` — exists, called at line 760
- `mkPositionalInclude` — exists at line 603
- `mkNamedInclude` — exists at line 634
- `structuralKeysSet` still includes `"provides"` at line 17

**`emitCrossProvider` still present** (`nix/lib/aspects/fx/handlers/transition.nix`):
- `emitCrossProvider` function at line 101
- Called at line 288 inside `resolveTransition`

**`provides` option still in `aspectSubmodule`** (`nix/lib/aspects/types.nix` lines 401-415):
- Full submodule option with `freeformType = lazyAttrsOf providerType`
- `_` alias still present at line 324: `(lib.mkAliasOptionModule [ "_" ] [ "provides" ])`

**Compat shim wired** (`nix/lib/aspects/fx/handlers/provides-compat.nix`):
- `emitCrossProvideShims` added and wired into `resolveChildren` at aspect.nix line 762
- Issues deprecation `lib.warn` for `provides.${key}` usage

**`den-brackets.nix` provides fallback still present** (`nix/lib/den-brackets.nix`):
- `resolveWithProvidesFallback` logic at lines 8-25
- Falls back to `(aspect.provides or {}).${head}` with deprecation warn
- Spec's clean-up (Step 7) not done

**Template migration incomplete** — 114 remaining `provides.X` occurrences across 30 files:
- `templates/ci/modules/features/aspect-path.nix` — multiple `provides.node-exporter`, `provides.alerting`
- `templates/ci/modules/features/has-aspect.nix` — multiple `provides.sub`, `provides.foo`
- `templates/ci/modules/features/fx-e2e.nix`, `fx-coverage.nix`, etc.
- `templates/default/modules/igloo.nix`, `tux.nix`
- `templates/diagram-demo/`, `templates/example/`
- `templates/ci/provider/modules/den.nix`
- `templates/ci/modules/features/provides-parametric.nix` (expected — test file for the compat shim)

**`mutual-provider.nix` test** (`templates/ci/modules/features/mutual-provider.nix`):
- No longer uses `provides.X` write patterns — migrated to `policies.to-tux` pattern
- The framework module `modules/aspects/provides/mutual-provider.nix` appears only in non-main worktrees (`.worktrees/docs-fixes`, `.worktrees/stash-cleanup`) — may already be deleted from main branch

## Current Status

The branch is in a **transitional compat-shim state** not described in the spec:

1. `emitSelfProvide` still runs but its provides check may be short-circuiting for migrated aspects
2. `emitCrossProvideShims` added as a NEW function (not in spec) — wraps old `provides.X` cross-provides in deprecation warnings and policy shims
3. The spec's Step 3 (delete `emitSelfProvide`) and Step 4 (delete `emitCrossProvider`) have not landed
4. The spec's Step 5 (remove `provides` option + `_` alias) has not landed
5. The spec's Step 6 (remove `provides` from `structuralKeysSet`) has not landed
6. The spec's Step 7 (clean up `den-brackets.nix`) has not landed
7. Step 1 (migrate templates) is approximately 0% complete — 114 occurrences remain

## Supersession

The compat shim approach (`emitCrossProvideShims`, `provides-compat.nix`) represents a design divergence: instead of migrating templates first then deleting machinery, the branch inserted a shim layer that auto-translates old `provides.X` patterns at runtime with deprecation warnings. This trades the spec's "migrate then delete" sequence for a "shim then migrate then delete" sequence. The shim is explicitly tagged for removal in comments ("Remove after migration period").

A separate spec file `docs/superpowers/specs/2026-04-27-provides-removal-post-unified-effects.md` exists (untracked) — this likely supersedes the original spec's approach or extends it post-unified-effects landing.

## Gaps

| Spec Step | Status |
|-----------|--------|
| Step 1: Migrate provides.X in templates (~111 occurrences) | NOT DONE — 114 remain |
| Step 2: Migrate mutual-provider reads | DONE — test file uses policies pattern |
| Step 3: Delete emitSelfProvide + mkPositionalInclude + mkNamedInclude | NOT DONE |
| Step 4: Delete emitCrossProvider + crossProvider | NOT DONE |
| Step 5: Remove provides option + _ alias from aspectSubmodule | NOT DONE |
| Step 6: Remove provides from structuralKeysSet | NOT DONE |
| Step 7: Clean up den-brackets.nix provides fallback | NOT DONE |

Added beyond spec (compat shim approach):
- `provides-compat.nix` handler + `emitCrossProvideShims` wired into resolveChildren
- Deprecation warns in `den-brackets.nix` fallback

## Drift

- Spec assumes "migrate first, then delete" — branch chose "shim first, warn, then migrate, then delete"
- Spec's `find-mutual` guard for name collision (Step 2 risk mitigation) not implemented — not needed since mutual-provider was replaced with policies
- `resolveChildren` bind chain has `emitSelfProvide` + `emitCrossProvideShims` both running — spec envisioned removing `emitSelfProvide` entirely; the shim adds a parallel path instead
- `_` alias removal blocked on `provides` option removal (Step 5), which is blocked on template migration completing

## Cross-Reference Notes

Cross-referenced 2026-04-28 against `2026-04-26-pipeline-simplification-targets.analysis.md` and `2026-04-26-pipeline-simplification-design.analysis.md`.

- **Prerequisite status**: The pipeline-simplification targets analysis previously listed "entityProvides removal (direct-ref-aspects Task 4)" as an incomplete gating prerequisite for Target 1. This was incorrect. The design analysis confirms entityIncludes/entityProvides were already deleted on this branch (commits `9a338b50`, `973ac30b`). The gating blocker for provides deletion is template migration (~114 occurrences), consistent with what this analysis documents.
- **Consistent with design analysis**: This file's status (all deletion steps pending, compat shim added, 114 template occurrences remaining) matches the pipeline-simplification design analysis Target 1 section exactly.
- **Host-aspects battery**: Not relevant to this spec. The host-aspects battery (main + feat/fx-pipeline) is a separate self-contained feature with no dependency on provides removal progress.
