# Pipeline Simplification Targets

**Date:** 2026-04-26
**Branch:** feat/traits
**Status:** Draft — follow-up work after direct-ref-aspects completion
**Prerequisites:** Direct-ref policy aspects spec (in progress), stages elimination (complete)

## Context

A comprehensive review of the fx pipeline identified simplification targets. Most complexity exists in impedance-matching layers between user-facing API and pipeline internals — normalizing diverse input shapes, compensating for shared namespaces, and varied sub-pipeline patterns. The effectful pipeline core is clean.

## Target 1: Delete the `provides` API

**Impact:** ~155 lines, ~13 branches removed
**Files:** `aspect.nix` (emitSelfProvide, mkPositionalInclude, mkNamedInclude), `transition.nix` (emitCrossProvider, crossProvider in resolveTransition)
**Depends on:** Direct-ref-aspects Task 4 (entityProvides/entityIncludes deletion) must complete first — `emitCrossProvider` is live until `entityProvides` is removed.

### What it does now

`provides.${name}` on an aspect is the "self-provide" — the root aspect for an entity. `provides.${otherName}` is a "cross-provide" — aspect data injected into a different entity during transitions.

### Why it can go

- **Self-provide** is replaced by `resolveEntity` deriving root from `ctx.${kind}.aspect` (stages elimination, further simplified in direct-ref-aspects)
- **Cross-provides** are deprecated — trace warning already exists. The intended replacement is direct nesting (sub-aspects as nested keys) or `policy.aspects`
- `emitSelfProvide` has 4 provider value shapes × 2 calling conventions (positional vs named) = ~8 branches. All go away
- `emitCrossProvider` remains live until `entityProvides` is fully removed. After that, `crossProvider` is always null for entity-resolved targets. It remains live for user-defined `provides.otherThing` on aspects until the provides migration is complete

### Migration path

1. **Prerequisite:** direct-ref-aspects entityProvides removal must be complete
2. Audit all `provides.X` usage in den and downstream templates
3. Convert `provides.X = fn` to direct nesting: `X = fn` (sub-aspect) or `policy.aspects = [fn]`
4. Remove `emitSelfProvide` from `resolveChildren` — ordering becomes `emitTransitions → emitIncludes`
5. Remove `emitCrossProvider` and the `crossProvider` fallback in `resolveTransition`
6. Remove `mkPositionalInclude`, `mkNamedInclude`
7. Remove `provides` from `structuralKeysSet`
8. Also co-remove `rootIncludes` — with self-provides gone and entityIncludes gone, `rootIncludes` has no producer. Remove the structural key and the phase from `resolveChildren`
9. Update `den-brackets.nix` — the `provides.` naming convention is encoded as a string transformation there

### Risks

- User aspects with `provides.to-users` or `provides.to-hosts` (mutual-provider pattern) need a migration path. These are currently cross-provides resolved during transitions. Direct nesting or policy.aspects replaces them.
- The `provides` deprecation trace has been live — audit for remaining usage beyond den's own modules.

---

## Target 2: Remove `wrapClassModule` collision detection

**Impact:** ~100-110 lines, 10+ branches removed
**Files:** `aspect.nix` (wrapClassModule, wrapDeferredImports, resolveCollisionPolicy, mkCollisionDetector)

### What it does now

When a den context arg name (e.g., `host`) collides with a NixOS `_module.args` name, the collision detection system:
1. Detects overlapping arg names between den context and module system
2. Emits a validator module that checks collisions at eval time
3. Supports three policies: `error` (default), `class-wins`, `den-wins`
4. The `collisionPolicy` option exists on every schema entity

### Why it can go

The collision exists because NixOS modules receive args via `_module.args` and den injects context via the same function-arg mechanism. If den args were namespaced (e.g., passed as `_den = { host, user, ... }` or using a different injection mechanism), collisions are structurally impossible.

### Scope note

Removing collision detection also removes the `advertisedArgs` / `lib.setFunctionArgs` mechanism that makes den context args optional in NixOS module resolution. Without this, NixOS would throw on unknown args when evaluating class modules that request den context. The migration must replace this mechanism — either by namespacing (so den args aren't in the module function signature at all) or by a wrapper that replicates the optional-arg behavior.

### Migration path

1. Change den context injection to pass a single `_den` arg (or use `_module.args._den`)
2. All class modules that destructure `{ host, user, ... }:` change to `{ _den, ... }: let inherit (_den) host user; in ...`
3. Or: provide a module wrapper that does the unwrapping AND replicates the `advertisedArgs` optional-arg mechanism, preserving backward compat
4. Remove `resolveCollisionPolicy`, `mkCollisionDetector`, `advertisedArgs`, the validator module emission, and the `collisionPolicy` entity option

### Risks

- Breaking change for all class module signatures
- Would need a compat layer or bulk migration
- The `config` arg collision (NixOS `config` vs den `config`) is the most common case — needs special handling
- The `advertisedArgs` removal is a hidden scope increase — it's not just collision detection but also the mechanism that makes den args work in NixOS modules at all

---

## Target 3: Extract sub-pipeline patterns

**Impact:** Clarity improvement, ~30 lines deduped across shared portions
**Risk:** Medium (not Low — the three call sites differ structurally)
**Files:** `transition.nix` (resolveFanOut, resolveSiblingTransition), `forward.nix` (forwardHandler)

### What it does now

Three places call `fxFullResolve` to run a sub-pipeline, but they interact with parent state differently:

1. **`resolveFanOut`** (transition.nix) — splices `imports` into parent via `fx.effects.state.modify`
2. **`resolveSiblingTransition`** (transition.nix) — does NOT splice state. Reads `state.traits` from sub-pipeline result, sends `provide-to` effect. No `state.modify` at all.
3. **`forwardHandler`** (forward.nix) — does NOT splice `imports` into parent via `state.modify`. Instead wraps sub-pipeline imports in an adapter aspect and re-emits as an include. Splices `provideTo` via direct state return (`state = state // { provideTo = ... }`), not via `state.modify`.

### What can be shared

The common pattern is: `fxFullResolve spec → extract state fields → do something with them`. The "do something" differs fundamentally:
- `resolveFanOut`: merge into parent state
- `resolveSiblingTransition`: send as effect payload
- `forwardHandler`: wrap in adapter aspect

A combinator can share the `fxFullResolve` call and state extraction, but the consumption step must remain per-site. A `runSubPipeline :: spec -> { class, self, ctx } -> { value, state }` that returns the materialized result would let each call site extract what it needs without re-implementing the setup.

### Migration path

1. Extract `runSubPipeline` that calls `fxFullResolve` and materializes state thunks
2. Each call site uses the returned `{ value, state }` and performs its own post-processing
3. Do NOT try to unify the post-processing — the three sites do genuinely different things

### Risks

- Medium. Getting the state materialization wrong would silently drop imports or provide-to entries. Each site currently handles its own materialization correctly; extracting it requires understanding all three patterns.

---

## Target 4: Replace `classifyKeys` with declared schemas

**Impact:** ~67 lines of heuristic classification removed
**Files:** `aspect.nix` (classifyKeys, freeformIgnoreSet, depth-limited sub-key scan)

### What it does now

`classifyKeys` takes a resolved aspect's non-structural keys and classifies each as:
- **Class key** — matches `den.classes` registry OR `targetClass` (runtime-resolved via `class` effect)
- **Trait key** — matches `den.traits` registry
- **Nested key** — an attrset or function that looks like a sub-aspect (via depth-limited 4-level probe)
- **Unregistered** — none of the above, warned and dropped

The depth probe (`isNestedAspect`) scans sub-keys to distinguish `{ nixos = ...; homeManager = ...; }` (nested aspect with class keys) from `{ enable = true; }` (option attrset, not nested). This heuristic is the most fragile part.

### Why it can go

If aspects declared their key types at construction time — e.g., via a `__keyTypes` annotation — the runtime classification heuristic is replaced by a trivial lookup.

### `targetClass` problem

The `k == targetClass` branch in `classifyKeys` depends on the runtime `targetClass` (resolved via the `class` effect in the pipeline). A static `__keyTypes` annotation computed at definition time would not have access to `targetClass`. Options:
1. Leave a residual runtime check only for the `targetClass == key` case
2. Require that class module authors annotate `targetClass` explicitly (unlikely to be practical)
3. Accept that the `targetClass` branch is a special case that stays dynamic

Option 1 is the pragmatic answer — most of `classifyKeys` becomes a lookup, with one residual `k == targetClass` dynamic check.

### Migration path

1. Add `__keyTypes` to the aspect type merge (aspectContentType)
2. At merge time, classify keys against the registries and store the result
3. `compileStatic` reads `__keyTypes` for registered keys, keeps dynamic `targetClass` check
4. Remove `classifyKeys` bulk classification, `freeformIgnoreSet`, depth probe

### Risks

- Medium. The classification affects every aspect resolution. Regression testing must be thorough.
- User aspects with dynamic keys (computed at eval time) may resist static classification.
- Should follow Target 1 (provides removal) — once `provides` is gone from structural keys, `classifyKeys`' input surface shrinks and `freeformIgnoreSet` can be designed cleanly.

---

## Target 5: Generalize flake fan-out

**Impact:** ~27 lines + hardcoded special case removed
**Files:** `transition.nix` (resolveFanOut, the `targetClass == "flake"` check in resolveTransition)

### What it does now

`resolveTransition` checks `isFanOut && targetClass == "flake"` and routes to `resolveFanOut` which runs a sub-pipeline instead of using `resolveContextValue`. Non-flake fan-out uses `resolveContextValue` with `foldl'` over indexed contexts.

### Why the special case exists

Flake-level fan-out (system × output) produces many independent contexts that share no pipeline state. Running each in a sub-pipeline prevents state pollution between contexts. Non-flake fan-out (e.g., host → users) shares state (deferred includes, ctx-seen dedup) because the contexts are related.

### Migration path

If Target 3 (sub-pipeline patterns) is done first, generalize the fan-out decision: any fan-out with independent contexts (no shared deferred state) uses the sub-pipeline path. The `targetClass == "flake"` check becomes a property on the entity kind or policy (e.g., `policy.isolateFanOut = true`).

### Risks

- Low if sub-pipeline extraction exists. The hard part is deciding which fan-outs are independent — currently hardcoded to flake.

---

## Target 6: Normalize child shapes at source

**Impact:** ~57 lines in `wrapChild` + `wrapFunctorChild` + `wrapBareFn` + `normalizeModuleFn`
**Files:** `include.nix` (wrapChild, wrapFunctorChild, wrapBareFn, normalizeModuleFn)

### What it does now

The include handler accepts ~7 distinct child shapes and normalizes them into a canonical form before processing. The shapes span: already-wrapped aspects, functor attrsets, submodule functions, bare parametric functions, plain attrsets, and raw values.

### Why it can go (partially)

Some shapes exist because the aspect type system coerces values at different points. If coercion happened once at the `den.aspects` boundary (before entering the pipeline), the include handler could assume canonical shape and skip normalization.

### Scope note

Internal factories like `den.provides.forward` produce functor shapes that go through `wrapFunctorChild`. These are not user-facing API in the ordinary sense but are call sites that would need to produce canonical form before entering the pipeline. A "keep one fallback for raw values" approach would not cover these internal factories.

### Migration path

1. Ensure all aspect type merges produce canonical form (`{ name, meta, includes, __fn?, __args? }`)
2. Ensure internal factories (`den.provides.forward`, `den.provides.mutual-provider`) produce canonical form
3. Remove `wrapChild` normalization — trust that inputs are already canonical
4. Keep one fallback path for genuinely raw user values (defensive)

### Risks

- Medium-high. The shapes are user-facing — users pass bare functions, attrsets, functors, etc. The normalization boundary needs to catch everything before the pipeline, not after.
- Internal factories are a hidden scope increase.

---

## Recommended Order

```
Direct-ref-aspects completion (prerequisite — removes entityIncludes/entityProvides)
  ↓
Target 1 (provides removal)     ← highest leverage, already deprecated, co-removes rootIncludes
  ↓
Target 3 (sub-pipeline patterns)  ← enables Target 5
  ↓
Target 5 (generalize flake fan-out)
  ↓
Target 4 (declared key schemas)  ← should follow Target 1 (provides removal shrinks input surface)
  ↓
Target 2 (collision detection)   ← breaking change, do last
  ↓
Target 6 (child shape normalization) ← dependent on type system changes + Target 2
```

**Dependencies:**
- Target 1 requires direct-ref-aspects Task 4 (entityProvides removal) as prerequisite
- Target 4 should follow Target 1 — `provides` removal shrinks `classifyKeys` input
- Target 5 depends on Target 3
- Target 6 depends on Target 2 (namespace change affects how modules enter the pipeline)
- Targets 1 and 3 are the immediate wins after direct-ref-aspects completes
