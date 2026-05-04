# Forward Sub-Pipeline Elimination and Trait Delivery Unification

## Status: Shipped

**Shipped (commits b44fd284 through 5572cc67 on `feat/fx-pipeline`):**
- fxResolve reorder: wrap → traitModule → applyForwardSpecs
- `applyForwardSpecs` accepts `{ forwardSpecs, classImports, traitModule, hasTraitSchemas }`
- Sub-pipeline path synthesizes `subTraitModule` from `sub.traits` (Tier 1/2 only)
- `runSubPipeline` reordered to match (wrap before forwards)
- evalConfig forwards now receive trait data via subTraitModule

**Previously deferred item — resolved by scope partitioning:**
Same-entity forward sub-pipeline elimination was blocked by shared `classImports` bucket contamination. Resolved by `2026-04-29-scope-partitioned-pipeline-state.md`: scoped state merge across sub-pipeline boundaries + `fxResolve` reads from `scopedClassImports` per scope. Tier 1 forwards auto-detect as routes (read scope partitions). Tier 2 forwards retain `resolveForwardSource` for adapter/cross-entity cases. Flake output forwards eliminated by `policy.instantiate`. 673/673 tests passing.

## Problem

Trait data emitted via `den.traits` never reaches class modules through evalConfig forward paths. A class module like `{ config, ... }: { x = config._den.traits.greeting or ["MISSING"]; }` receives the fallback value because `mkDirectAspect` (forward.nix:20-27) calls `lib.evalModules { modules = [freeformMod sourceModule]; }` — a separate evaluation without `traitModule`.

**Root cause:** `applyForwardSpecs` ran sub-pipelines that produced class modules but no traitModule. The traitModule was only synthesized in `fxResolve` after forward processing completed.

**Trait tier context (per unified-effects spec):** This fixes **Tier 3** trait delivery — trait data consumed inside `evalModules` via NixOS module function args. Tier 1/2 resolve during tree-walk before forwarding occurs. The `traitModule` is the Tier 3 delivery mechanism: a NixOS module that exposes collected trait data as `config._den.traits` at evaluation time. Class modules access traits via lazy thunks in `wrapClassModule` (aspect.nix:222): `moduleArgs.config._den.traits.${name} or []`.

**Non-evalConfig forwards were unaffected:** For non-evalConfig forwards, all modules share the same final `evalModules` with `traitModule`, so `config._den.traits` was always accessible. The bug only manifested with `evalConfig = true`.

## What Was Delivered

### fxResolve Reorder

Post-pipeline order changed from `forwards → wrap → traitModule` to `wrap → traitModule → forwards`:

```
Post-pipeline (fxResolve):
  1. wrapCollectedClasses on rawClassImports
  2. traitModule synthesized (same as before, now earlier)
  3. applyForwardSpecs with wrapped classImports + traitModule
     └─ each forward: runSubPipeline → synthesize subTraitModule from sub.traits
     └─ rawSourceModule = { imports = sub.classImports.${fromClass} ++ [subTraitModule] }
     └─ mapModule → buildForwardAspect → merge into intoClass
  4. Return { imports = forwarded.classImports.${class} ++ [traitModule] }
```

### subTraitModule Injection

Each forward sub-pipeline's `sub.traits` (Tier 1/2 data collected during the sub-pipeline's tree-walk) is used to synthesize a `subTraitModule` included in the forward's source module imports. This gives evalConfig forwards access to `config._den.traits`.

```nix
subTraitModule =
  { ... }:
  {
    options._den.traits = lib.mkOption { type = lib.types.attrsOf lib.types.anything; default = {}; internal = true; };
    # Tier 1/2 only — no deferred or cross-entity data in sub-pipeline scope.
    config._den.traits = lib.mapAttrs (traitName: schema: ...) subTraitSchemas;
  };
```

### runSubPipeline Reorder

`runSubPipeline` also had the same wrap-after-forward ordering. Reordered to match: wrap → applyForwardSpecs. Passes `traitModule = null; hasTraitSchemas = false` since sub-pipeline trait injection happens inside `applyForwardSpecs` itself.

## What Was NOT Delivered (and Why)

### Same-Entity Forward Sub-Pipeline Elimination

**Original proposal:** Skip `runSubPipeline` when `classImports.${spec.fromClass}` has content, reading directly from the bucket instead.

**Why it failed:** `classImports` buckets are shared across ALL entities in the pipeline. Transition sub-pipelines (user resolution) merge their `classImports` back into the parent state (`transition.nix:208`, `lib.zipAttrsWith concatLists`). After the pipeline completes:

- `classImports.homeManager` = tux's HM modules ++ pingu's HM modules ++ ...
- `classImports.os` = host's os modules (possibly + user os modules)

A forward spec for user-tux's HM content reads the shared bucket and gets ALL users' modules — causing duplicate option declarations and conflicting values (13 test failures).

**Sub-pipeline isolation is structurally necessary:** Each forward spec carries `sourceAspect` + `__resolveCtx` that define a specific entity scope. The sub-pipeline re-walks that specific source aspect in that specific context, producing class content scoped to exactly one entity. The shared bucket has no such scoping.

**Future approaches:**
1. **Per-entity classImports side-table** — At transition merge time in `resolveFanOut`, store `entityClassImports.${ctxId} = sub.classImports` before merging. Forward reads from this keyed table.
2. **Entity-tagged emit-class entries** — Add entity identity to each `emit-class` param. `classCollectorHandler` partitions by entity. More invasive but enables other optimizations.
3. **Forward-specific sub-pipeline caching** — Cache `runSubPipeline` results keyed by `(sourceAspect, resolveCtx, fromClass)` to avoid redundant re-walks when multiple forwards share the same source.

## Files Changed

### `nix/lib/aspects/fx/pipeline.nix`

- **`applyForwardSpecs`** — Changed from positional `forwardSpecs: classImports:` to attrset `{ forwardSpecs, classImports, traitModule, hasTraitSchemas }:`. Synthesizes `subTraitModule` from `sub.traits` and includes it in `rawSourceModule.imports`.
- **`fxResolve`** — Reordered: `wrapCollectedClasses` before `applyForwardSpecs`. Passes `traitModule` + `hasTraitSchemas` to `applyForwardSpecs`. Final imports read from `forwarded.classImports`.
- **`runSubPipeline`** — Reordered to match. Passes `traitModule = null; hasTraitSchemas = false`.

### `templates/ci/modules/features/trait-forward-delivery.nix` (new)

- `test-evalConfig-forward-loses-traits` — evalConfig forward with `config._den.traits` access. Was failing before fix, passes after.
- `test-non-evalConfig-trait-collection` — Pipeline-level verification that trait data survives through `fxFullResolve`. Regression guard.

## Test Coverage

**659/659 passing.** All existing forward tests unaffected:
- `forward.nix`, `forward-to.nix`, `forward-flake-level.nix`, `cross-context-forward.nix`
- `guarded-forward.nix`, `forward-from-custom-class.nix`, `forward-alias-class.nix`
- `os-class-host.nix`, `traits.nix`, `provide-to.nix`

New tests verify the specific evalConfig failure path and trait collection regression.
