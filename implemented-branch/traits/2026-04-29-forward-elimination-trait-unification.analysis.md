# Analysis: Forward Sub-Pipeline Elimination and Trait Delivery Unification

## Verdict
partially-implemented
The fxResolve reorder and subTraitModule injection for evalConfig forward trait delivery shipped in commit b44fd284. Same-entity forward sub-pipeline elimination was attempted but reverted due to shared `classImports` bucket contamination across entities; deferred with three documented future approaches.

## Delivery Target
feat/fx-pipeline

## Evidence

### fxResolve reorder (wrap → traitModule → forwards): SHIPPED

`nix/lib/aspects/fx/pipeline.nix:571-581`:
```
# Wrap BEFORE forwards — forward source modules need wrapped class data + traitModule.
wrappedClassImports = wrapCollectedClasses finalCtx rawClassImports;
# Apply forwards AFTER wrapping + traitModule synthesis.
# Forward source modules now include traitModule, fixing Tier 3 trait delivery.
forwarded = applyForwardSpecs {
  inherit forwardSpecs traitModule hasTraitSchemas;
  classImports = wrappedClassImports;
};
```

### applyForwardSpecs attrset signature: SHIPPED

`nix/lib/aspects/fx/pipeline.nix:267-273` — changed from positional args to `{ forwardSpecs, classImports, traitModule, hasTraitSchemas }:`.

### subTraitModule synthesis in applyForwardSpecs: SHIPPED

`nix/lib/aspects/fx/pipeline.nix:287-312` — synthesizes `subTraitModule` from `sub.traits`, includes it in `rawSourceModule.imports` alongside `sub.classImports.${spec.fromClass}`. Comment at line 300 explicitly notes "Tier 1/2 only — no deferred or cross-entity data in sub-pipeline scope."

### runSubPipeline reorder: SHIPPED

`nix/lib/aspects/fx/pipeline.nix:354-359` — `wrapCollectedClasses` before `applyForwardSpecs`. Passes `traitModule = null; hasTraitSchemas = false` because sub-pipeline trait injection is handled inside `applyForwardSpecs` itself.

### New test file: SHIPPED

`templates/ci/modules/features/trait-forward-delivery.nix` — two tests:
- `test-non-evalConfig-trait-collection` (line 13): regression guard, pipeline-level trait state survives fxFullResolve.
- `test-evalConfig-forward-loses-traits` (line 45): evalConfig forward with `config._den.traits.greeting or ["MISSING"]`, expects `"igloo-greeting"`. Was failing before b44fd284.

Commits:
- `6b97c074` — adds the failing test (trait-forward-delivery.nix, 90 lines, test-only)
- `b44fd284` — ships the fix (pipeline.nix +54/-14, trait-forward-delivery.nix -6 lines to fix hostname pattern)

### mkDirectAspect evalConfig path: UNCHANGED (by design)

`nix/lib/aspects/fx/handlers/forward.nix:20-33` — `lib.evalModules { modules = [freeformMod sourceModule]; }` still called without traitModule. Fix is upstream: `sourceModule` now carries `subTraitModule` in its imports before reaching `mkDirectAspect`, so no change needed here.

### Same-entity forward sub-pipeline elimination: NOT SHIPPED

No evidence of per-entity classImports side-table, entity-tagged emit-class, or forward caching in the codebase. Grep for `entityClassImports`, `entity-tagged`, and `ctxId.*classImports` returns nothing.

Root cause confirmed alive: `handlers/transition.nix:280` — `lib.zipAttrsWith (_: builtins.concatLists) [current subClassImports]` merges ALL sub-pipeline `classImports` into shared parent state. A forward spec for one user reads the merged bucket and gets modules from all users.

## Current Status

Shipped portions are stable and integrated. The `fxResolve` pipeline ordering is now `wrap → traitModule → applyForwardSpecs` as spec describes. The evalConfig trait delivery fix is in effect for 659/659 passing tests.

Same-entity forward sub-pipeline elimination: deferred, still uses `runSubPipeline` for all forwards. Three future approaches documented in spec (per-entity side-table, entity-tagged emit-class entries, forward-specific caching) — none started as of HEAD `09c8cbee`.

## Supersession

Supersedes the in-pipeline-forward limitation documented in project memory (forward spec from 1c077a06). Partially superseded by scope-partitioned state work (commits `671ebf0c`–`09c8cbee`) which may eventually enable entity-scoped classImports via `scopeContexts`/`scopeChildren`, though no explicit connection yet.

## Gaps

1. **Same-entity forward sub-pipeline elimination** — The optimization to skip `runSubPipeline` by reading from the shared `classImports` bucket is blocked and not started. Multi-user hosts will continue paying sub-pipeline re-walk cost per forward spec.
2. **Per-entity classImports side-table** — Future approach 1 from spec (`entityClassImports.${ctxId} = sub.classImports` at `resolveFanOut` merge time) not present in `handlers/transition.nix`.
3. **Entity-tagged emit-class** — Future approach 2 not started, no entity identity on emit-class params.
4. **Forward-specific sub-pipeline caching** — Future approach 3 not started, no caching keyed by `(sourceAspect, resolveCtx, fromClass)`.

## Drift

- Spec describes `subTraitModule` as using `sub.traits` directly (Tier 1/2 only). Implementation matches exactly at `pipeline.nix:291-308` — no drift.
- Spec says `runSubPipeline` passes `traitModule = null; hasTraitSchemas = false`. Implementation matches at `pipeline.nix:358-359`.
- Spec says `applyForwardSpecs` previously used positional `forwardSpecs: classImports:` signature. Confirmed replaced with attrset at `pipeline.nix:268-273`.
- Spec references `transition.nix:208` for the `zipAttrsWith concatLists` merge. Actual location is `handlers/transition.nix:280` (file is `handlers/transition.nix`, not `transition.nix` — spec path was wrong but code is identical in structure).
- Spec test count "659/659" — current HEAD `09c8cbee` is several commits past b44fd284 (scope-partitioned state work added), test count not re-verified but no regressions introduced per commit messages.
