# Handler File Cleanup — aspect.nix decomposition + transition.nix dedup

## Status: Draft

## Problem

`aspect.nix` is 1119 lines with 6+ distinct responsibilities: class module wrapping, key classification, DLQ management, include normalization, provides compat, and the core aspect compiler. `transition.nix` is 859 lines with duplicated policy classification logic between global and aspect dispatch.

## Changes

### aspect.nix decomposition (1119 → ~595 lines)

**1. Extract `wrapClassModule` → `nix/lib/aspects/fx/wrap.nix` (~190 lines)**

Pure function, no effect system dependency. Takes `{ module, ctx, aspectPolicy, globalPolicy, traitNames }`, returns `{ module, wrapped, validator?, ... }`. Currently exported from aspect.nix, consumed by pipeline.nix's `wrapCollectedClasses`. Also extract `resolveCollisionPolicy` and `wrapDeferredImports` which are only used by `wrapClassModule`.

**2. Extract DLQ cluster → `nix/lib/aspects/fx/handlers/dead-letter.nix` (~100 lines)**

Move `drainDeadLettersHandler`, `emitClassFromDLQ`, `emitTraitFromDLQ`. Self-contained subsystem: reads `state.deadLetterQueue` + `state.traitSchemas`, reclassifies entries, re-emits. Handler construction needs `classRegistry` (static from `den.classes`).

Update pipeline.nix `defaultHandlers` to import from new file instead of aspect.nix.

**3. Extract include normalization → `nix/lib/aspects/fx/handlers/include.nix` (~65 lines)**

Move `mkPositionalInclude` and `mkNamedInclude` into the existing include handler file. They're pure normalization helpers consumed by the include handler's `emit-include` path. Currently in aspect.nix only because of historical file layout.

**4. Delete `dispatchPolicyIncludes` after unified dispatch (~85 lines)**

Depends on unified policy dispatch (removing `includeOnly` filter). Once all policy effects process uniformly in `into-transition`, `dispatchPolicyIncludes` in aspect.nix and `emitTransitions`'s call to it are deleted. `emitTransitions` simplifies to just sending `into-transition`.

**5. Delete `emitSelfProvide` + `emitCrossProvideShims` after provides removal (~75 lines)**

Depends on provides removal completing. Both are provides-era code. `emitCrossProvideShims` already emits deprecation warnings.

### transition.nix dedup (859 → ~500 lines)

**1. Extract `classifyPolicyEffects` (~40 lines, new)**

The effect classification pattern (filter resolves/includes/excludes/routes, classify resolves via `classifyResolve`, merge enrichment) is duplicated identically between global policy dispatch (lines 538-621) and aspect policy dispatch (lines 625-704). Extract once:

```nix
classifyPolicyEffects = rawEffects: {
  resolves = filter isResolve effects;
  includes = filter isInclude effects;
  excludes = filter isExclude effects;
  routes = filter isRoute effects;
  enrichment = foldl' mergeEnrichment {} (filter isEnrich classified);
  transitions = buildTransitions schemaResolves routing;
};
```

Both global and aspect dispatch call `classifyPolicyEffects (policy resolveCtx)`.

**2. Delete `emitCrossProvider` (~40 lines)**

Dead code from provides era. Called in `resolveTransition` but produces no effects when `sourceProvides.${targetKey}` is null (which it always is after provides removal).

**3. Delete `processIncludeOnly` (~25 lines)**

After unified dispatch, include-only transitions merge into the main `resolveTransition` path. No separate handler needed.

**4. Simplify `resolveFanOut`**

After policy.route ships and sub-pipeline is fully eliminated, `resolveFanOut` is just scope push + `aspectToEffect` + scope pop (~20 lines). Currently ~80 lines with sub-pipeline + merge logic.

## Dependencies

| Change | Depends on |
|--------|-----------|
| wrapClassModule extraction | None — can ship now |
| DLQ extraction | None — can ship now |
| Include normalization move | None — can ship now |
| dispatchPolicyIncludes deletion | Unified policy dispatch |
| emitSelfProvide/CrossProvideShims deletion | Provides removal |
| classifyPolicyEffects dedup | None — can ship now |
| emitCrossProvider deletion | Provides removal |
| processIncludeOnly deletion | Unified policy dispatch |
| resolveFanOut simplification | policy.route spec (full sub-pipeline elimination) |

## Execution order

Phase 1 (no dependencies, ship immediately):
- Extract `wrapClassModule` → wrap.nix
- Extract DLQ → dead-letter.nix
- Move include normalization → include.nix
- Extract `classifyPolicyEffects` in transition.nix

Phase 2 (after policy.route):
- Simplify `resolveFanOut`
- Delete `processIncludeOnly`

Phase 3 (after provides removal):
- Delete `emitSelfProvide`, `emitCrossProvideShims`, `emitCrossProvider`

Phase 4 (after unified dispatch):
- Delete `dispatchPolicyIncludes`
