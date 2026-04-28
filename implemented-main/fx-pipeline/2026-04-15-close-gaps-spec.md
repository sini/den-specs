# Close Gaps: Unified Pipeline Completion

**Date:** 2026-04-15
**Branch:** `feat/fx-resolution` (485/485 green)
**Prereq:** Unified pipeline implemented (aspectToEffect, emit-include handler, constantHandler, etc.)
**Status:** Spec (updated with debugging findings)

## Current state

The unified pipeline works end-to-end (485/485 green) with two legacy compat shims:
- `"aspect-chain" = []` in `constantHandler` — handles the effect sent by `{ class, aspect-chain }` provider functions from the type system
- `"aspect-chain" = [self]` override at root — `forward.nix` reads `lib.head aspect-chain`

## Findings from debugging

### Why `defaultFunctor` can't be gated on `den.fxPipeline`

`defaultFunctor` feeds into `typesConf` → `types` → aspect option declarations. Accessing `den.fxPipeline` (a config value) during type declaration creates circular evaluation: `config.den` → aspects → types → `defaultFunctor` → `config.den`. This was verified — it produces `attribute 'den' missing` errors.

**Implication:** The type system will always bake `defaultFunctor` (parametric.withOwn) into every aspect. Provider functions from `providerFnType.merge` (types.nix:37) will always create `{ class, aspect-chain }` functors. The fx pipeline must handle this.

### Why `options.nix` can't gate on `den.fxPipeline`

Same circular evaluation. `config.resolved` in `options.nix` is an option default — it can't access `config.den` without circularity.

**Implication:** `config.resolved` always goes through legacy ctxApply. The fx pipeline receives pre-wrapped (parametric) aspects.

### Why `aspect-chain` must be provided

Provider functions from the type system (`providerFnType.merge`) create `__functor = _: eth.merge loc defs` where the merged function has `functionArgs = { class, aspect-chain }`. When `aspectToEffect` encounters these (via `bind.fn`), it sends `aspect-chain` as an effect. `constantHandler` must handle it.

`forward.nix:27` reads `lib.head aspect-chain` — needs `[self]` at root, not `[]`.

### Option E (aspects never functors) doesn't work as designed

The type system's `__functor` option default and `providerFnType.merge` create `{ class, aspect-chain }` functions throughout the tree. These can't be stripped without breaking the evaluation chain because:
1. `defaultFunctor` feeds into type declarations (can't gate)
2. `providerFnType.merge` wraps user functions at merge time (can't intercept)
3. Children in `includes` may be provider functions with `{ class, aspect-chain }` args

The `hasLegacyArgs` filter in `aspectToEffect` would prevent calling these functors, but then the functor result (which contains the actual class configs) is never produced. The parametric path is load-bearing — it's how the type system materializes aspect data.

## Revised gap analysis

Given these constraints, here are the actual remaining gaps:

### Gap 1: `emitTransitions` and `emitSelfProvide` not wired

Still valid. `compileStatic` doesn't call them. The into-transition and self-provide paths are defined but disconnected.

### Gap 2: Dead ctx handlers

Partially addressed — the agent's commit `6c68cb52` removed `ctxProviderHandler`, `ctxTraverseHandler`, `ctxTraceHandler`, `ctxEmitHandler`. But that commit was reverted as part of `e71917f1` revert. Need to re-apply.

### Gap 3: `ctx-seen` not wired

Still valid. The transition handler should send `ctx-seen` for dedup.

### Gap 4: `emit-include` param shape

Not a gap — chain provides provenance. No change needed.

### Gap 5: `aspect-chain` in constantHandler

**Now understood and resolved.** The `"aspect-chain" = []` handler and `[self]` root override are necessary compat shims for the type system's provider functions. They stay until the type system is reworked. Documented with TODO.

### Gap 6: `resolve-complete` in computation vs handler

Still a design deviation but works correctly. Lower priority — can address when the type system is reworked.

### Gap 7: `wrapAspect` legacy

Still exported. Should be removed or deprecated.

### Gap 8: `emitIncludes`/`foldIncludes` duplication

Still duplicated. Should share one definition.

## Fixes (revised ordering)

### Fix 1: Re-apply dead ctx handler removal

The agent's commit `6c68cb52` correctly removed dead handlers. It was reverted by `e71917f1`. Re-apply the removal:

Remove from `handlers/ctx.nix`:
- `ctxProviderHandler`
- `ctxTraverseHandler`
- `ctxTraceHandler`
- `ctxEmitHandler`

Keep:
- `constantHandler`
- `ctxSeenHandler`

### Fix 2: Wire `emitTransitions` and `emitSelfProvide` into `compileStatic`

Add to the `compileStatic` chain after `registerConstraints`, before `emitIncludes`:

```nix
nxFx.bind (emitClasses ...) (_:
  nxFx.bind (registerConstraints aspect) (_:
    nxFx.bind (emitSelfProvide aspect) (selfProvResults:
      nxFx.bind (emitTransitions aspect) (transitionResults:
        nxFx.bind (chain-push ...) (_:
          nxFx.bind (emitIncludes ...) (children:
            nxFx.bind (chain-pop ...) (_:
              let resolved = aspect // { includes = selfProvResults ++ transitionResults ++ children; }; in
              nxFx.bind (resolve-complete resolved) (_: nxFx.pure resolved))))))))
```

### Fix 3: Wire `ctx-seen` in transitionHandler

Before each scoped run, send `ctx-seen key` for dedup:

```nix
nxFx.bind (nxFx.send "ctx-seen" key) ({ isFirst }:
  nxFx.bind (nxFx.effects.scope.run { ... } (aspectToEffect targetAspect)) ...)
```

### Fix 4: Remove `wrapAspect` legacy

Remove from `aspect.nix`. Update any tests that reference it.

### Fix 5: Deduplicate `emitIncludes`/`foldIncludes`

Keep `emitIncludes` in `aspect.nix`, export it. Have `include.nix:resolveConditional` use `den.lib.aspects.fx.aspect.emitIncludes` instead of local `foldIncludes`.

### Fix 6: Clean up stale tasks

The task list has stale tasks from the unified pipeline plan. Clean them up.

## Files affected

| File | Changes |
|---|---|
| `nix/lib/aspects/fx/aspect.nix` | Wire emitTransitions + emitSelfProvide into compileStatic. Remove wrapAspect. Export emitIncludes. |
| `nix/lib/aspects/fx/handlers/ctx.nix` | Remove dead handlers (ctxProvider, ctxTraverse, ctxTrace, ctxEmit). |
| `nix/lib/aspects/fx/handlers/transition.nix` | Add ctx-seen dedup. |
| `nix/lib/aspects/fx/handlers/include.nix` | Remove local foldIncludes, use shared. |
| `templates/ci/modules/features/fx-*.nix` | Update tests referencing wrapAspect. |

## What stays as-is (now understood)

| Item | Reason |
|---|---|
| `"aspect-chain" = []` in constantHandler | Type system provider functions need it. Can't gate defaultFunctor. |
| `"aspect-chain" = [self]` root override | forward.nix reads lib.head aspect-chain. |
| `defaultFunctor = parametric.withOwn` | Type system circular evaluation prevents gating. |
| `options.nix` uses legacy ctxApply | Circular evaluation prevents gating config.resolved. |
| `resolve-complete` in compileStatic | Works correctly; lower priority design deviation. |

## Future work (type system rework)

When the legacy pipeline is removed:
1. Remove `defaultFunctor` / `parametric.withOwn` from type system
2. Remove `providerFnType.merge`'s `__functor` wrapping
3. Aspects become plain attrsets — no functor needed
4. Remove `"aspect-chain"` handler from pipeline
5. Remove `forward.nix` dependency on `aspect-chain`
6. `options.nix` can use `aspectToEffect` directly (no ctxApply)

## Verification

```bash
nix develop -c just ci ""
```

All 485+ tests must pass.
