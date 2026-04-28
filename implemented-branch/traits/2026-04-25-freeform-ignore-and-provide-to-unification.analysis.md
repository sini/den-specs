# Analysis: Freeform-Ignore and Provide-To Unification

## Verdict

FULLY IMPLEMENTED. Both phases shipped on feat/fx-pipeline. No gaps.

## Delivery Target

feat/fx-pipeline branch. Four commits:
- `e9687e9d` feat: freeform-ignore + targetClass recognition in classifyKeys
- `ec3ad18a` refactor: remove provide-to structural key from aspect pipeline
- `57129393` refactor: distribute-cross-entity replaces distribute-provide-to
- `ab126627` feat: fxResolve cross-entity trait injection

## Evidence

### Phase 1: Freeform-Ignore

`classifyKeys` in `nix/lib/aspects/fx/aspect.nix` now takes `targetClass` as first argument:

```nix
classifyKeys = targetClass: aspect:
```

Step 4 recognition: `classRegistry ? ${k} || (targetClass != null && k == targetClass)` — forward-scoped class aliases recognized via targetClass.

`compileStatic` (inside `aspectToEffect`) reads `targetClass` via `fx.effects.hasHandler "class"` + `fx.send "class" null` at pipeline-run time. `allClassKeys = classKeys` — unregistered keys excluded. Trace warning emitted per unregistered key:

```nix
builtins.trace "den: ignoring unregistered key '${k}' in aspect '${rawName}' — register in den.classes or den.traits" null
```

Backward compat fallback preserved: `isEmpty = classRegistry == {} && traitRegistry == {}` branch returns all keys as classKeys with empty unregisteredClassKeys.

`den.classes.funny.description` registered in `templates/ci/provider/modules/den.nix:45`.

`"provide-to"` absent from `structuralKeysSet` (lines 12-33 in aspect.nix). No `provide-to.*` emission in `resolveChildren`.

### Phase 2: Provide-To Unification

`distribute-cross-entity.nix` exists at `nix/lib/aspects/fx/distribute-cross-entity.nix`. Implements `groupByTarget`, `mergeTraits`, `distributeCrossEntityTraits`, `distribute`. Collection-strategy-aware (list/map). Map strategy uses `collectTrait` helper for merge + duplicate detection.

`provideToHandler` in `handlers/provide-to.nix` accumulates `{ targetEntity, traits }` shaped emissions (not labeled `content` data).

`resolveSiblingTransition` in `handlers/transition.nix` runs sub-pipeline via `runSubPipeline` per peer, captures `sub.traits`, emits `fx.send "provide-to" { inherit targetEntity traits; }`.

`fxResolve` in `pipeline.nix` accepts `crossEntityTraits ? {}` parameter. `traitModule` merges cross-entity data: list strategy appends `crossEntity`, map strategy calls `mergeMaps raw (deferredData ++ [crossEntity])`.

`default.nix` exports `resolveWithState = fxResolveTreeFull` for callers that need `state.provideTo`. `fxResolveTree` (used by entity `mainModule` default) still calls `fxResolve` without `crossEntityTraits` — single-entity path unchanged.

## Current Status

All spec-listed files modified:
- `nix/lib/aspects/fx/aspect.nix` — classifyKeys signature, step 4, structuralKeysSet
- `nix/lib/aspects/fx/distribute-cross-entity.nix` — new file (renamed from distribute-provide-to)
- `nix/lib/aspects/fx/handlers/provide-to.nix` — new payload shape
- `nix/lib/aspects/fx/handlers/transition.nix` — sibling sub-pipeline execution
- `nix/lib/aspects/fx/pipeline.nix` — fxResolve crossEntityTraits param
- `templates/ci/provider/modules/den.nix` — funny class registered
- `templates/ci/modules/features/provide-to.nix` — rewritten for trait-based routing

## Supersession

Supersedes `distribute-provide-to.nix` (renamed). The `constantHandler`-based backward-compat wrapper (`distribute`) is retained in `distribute-cross-entity.nix` for parametric aspect binding.

## Gaps

One architectural gap from spec: the two-phase orchestration in `fxResolveTree` / output modules is NOT wired. `default.nix:fxResolveTree` calls `fxResolve` without `crossEntityTraits`. The `osConfigurations.nix` and entity `mainModule` defaults never call `distributeCrossEntityTraits` or pass `crossEntityTraits`. So cross-entity trait injection into end configs is not plumbed — only the component pieces exist. Tests in `provide-to.nix` exercise `fxResolve { crossEntityTraits = ... }` directly, bypassing the orchestration layer. The spec's Phase 2 integration point (`fxResolveTree` two-phase flow) remains unimplemented.

## Drift

Spec listed `nix/lib/aspects/default.nix` for two-phase orchestration (`fxResolveTree`). Actual: `default.nix` only added `resolveWithState = fxResolveTreeFull` as an export hook, leaving the two-phase wiring to callers. This is consistent with the rest of the pipeline simplification direction but is a delivery gap against the spec's Phase 2 orchestration section.

The `distribute` function in `distribute-cross-entity.nix` wraps results in `constantHandler` bindings — this is a backward-compat shim not described in the spec, added to support parametric aspects consuming cross-entity data via `bind.fn`.
