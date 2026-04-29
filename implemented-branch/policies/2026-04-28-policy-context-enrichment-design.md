# Policy Context Enrichment & Post-Pipeline Class Wrapping

**Date:** 2026-04-28
**Branch:** `feat/fx-pipeline`
**Status:** Implemented

Shipped on `feat/fx-pipeline` (commits a91eee15..36341531). Fix 1 (schema/enrichment classification) and Fix 2 (post-pipeline class wrapping pass) both landed. All spec test cases passing.

## Problem

Two related gaps prevent policy-injected context from reaching class modules:

### 1. Non-schema resolve keys create empty child entities

When a policy returns `resolve { isNixos = true; isDarwin = false; }`, the transition handler infers `targetKey` from the resolve binding keys. Since `isNixos`/`isDarwin` don't match any schema kind, `targetKey` falls back to `builtins.head firstResolveKeys` (e.g., `"isNixos"`). This creates a transition to entity kind `"isNixos"` — which doesn't exist in `den.schema`. `resolveEntity "isNixos"` produces an empty entity. The bindings go into an empty child context that resolves nothing.

The unified policy effects spec states: "Keys not in `den.schema` are valid — they create context scopes without entity schema association." The intent is context enrichment of the current entity, not child entity creation.

### 2. Class modules are wrapped too early

`wrapClassModule` runs during `emitClasses` in Phase A (tree-walk). Policy resolve effects fire in Phase B (transitions). A flat-form class module like `{ isNixos, lib, ... }:` is processed before `isNixos` enters the pipeline context. Since `isNixos` is not in `ctx`, it's treated as a module-system arg and advertised to NixOS, which can't provide it — causing infinite recursion.

The parametric wrapper form `{ isNixos }: { nixos = ... }` works because it defers as an include. Flat-form class modules have no equivalent mechanism.

### Key insight: class modules are terminal leaves

Class modules produce NixOS/Darwin/HM configuration. No information flows back into the pipeline. Their wrapping can be deferred to after the pipeline fully resolves. This eliminates the detection problem: by the time we wrap, all context is available and `ctx ? isNixos` is trivially true. No heuristics, no allowlists, no `has-handler` probing.

## Design

Two changes. Fix 1 routes non-schema resolve effects as context enrichment. Fix 2 defers all class module wrapping to post-pipeline. Together they ensure policy-injected context reaches class modules regardless of form.

### Fix 1: Non-schema resolve routing (context enrichment)

**File:** `nix/lib/aspects/fx/handlers/transition.nix`

In `transitionHandler`, when building transitions from policy resolve effects, separate non-schema resolve effects from schema resolve effects. Use the same `schemaKinds` definition already in `transition.nix` (entity-filtered, lines 16-18):

```nix
classifyResolve = e:
  let
    keys = builtins.attrNames e.value;
    schemaKeys = builtins.filter (k: builtins.elem k schemaKinds) keys;
    enrichKeys = builtins.filter (k: !builtins.elem k schemaKinds) keys;
    hasTarget = e.__targetKind or null != null;
  in
  if hasTarget then
    # Explicit target kind — always a schema resolve (full value)
    { schema = e; enrichment = null; }
  else if schemaKeys == [] then
    # Pure enrichment — no schema keys
    { schema = null; enrichment = e.value; }
  else if enrichKeys == [] then
    # Pure schema resolve — all keys match schema kinds
    { schema = e; enrichment = null; }
  else
    # Mixed resolve — split: schema keys create transition, enrichment keys enrich context
    {
      schema = e // { value = lib.filterAttrs (k: _: builtins.elem k schemaKinds) e.value; };
      enrichment = lib.filterAttrs (k: _: !builtins.elem k schemaKinds) e.value;
    };
```

**Mixed resolves** are split: schema keys create child entity transitions, enrichment keys enrich the current context. For example, `resolve { nixos = config; isNixos = true; }` creates a `nixos` transition AND installs `isNixos = true` as enrichment. This avoids silently dropping enrichment keys that happen to coexist with schema keys.

**Schema resolves** (all keys match schema kinds, or `__targetKind` is set) proceed as today — they create child entity transitions.

**Enrichment resolves** (no keys match schema kinds) are processed inline within the transition handler's `resume` computation, after `dispatchPolicies` and `dispatchAspectPolicies` produce their transition lists but before `resolveTransition` processes child entity transitions:

1. Collect all enrichment bindings from all enrichment resolves into a single merged attrset
2. Install new constant handlers via `scope.provide`
3. Update `state.currentCtx` with the enriched bindings
4. Drain deferred includes (`drain-deferred`) — parametric aspects requesting the new keys now resolve

The `__shared` flag on enrichment resolves is irrelevant (no fan-out) and ignored.

**Multiple enrichment resolves** from different policies merge additively. If two policies provide the same key, later value shadows (same semantics as `scope.provide` overlay). Merge order follows policy dispatch order: global policies (in schema-defined order) before aspect policies (in aspect registration order). Within each group, the order is deterministic per-evaluation. This differs from schema resolves where conflicting values create separate parallel branches. The asymmetry is intentional: enrichment adds derived values to the current scope, not parallel entity branches.

**Interaction with fixed-point:** Enrichment bindings become available in `state.currentCtx` before subsequent policy dispatch iterations. A policy that takes `{ isNixos, ... }:` can fire in the next iteration after another policy provides `isNixos` via enrichment.

**Drain-deferred re-entrancy:** `drain-deferred` after enrichment may resolve deferred includes that emit new `emit-class` effects. These class emissions are collected normally by the class collector handler (they carry their ctx snapshot). Deferred includes cannot emit further resolve effects — they produce aspect content, not policy effects — so there is no re-entrant enrichment processing risk.

### Fix 2: Post-pipeline class module wrapping

**File:** `nix/lib/aspects/fx/aspect.nix`, `nix/lib/aspects/fx/pipeline.nix`

All class module wrapping is deferred to a single post-pipeline pass. During tree-walk, `emitClasses` collects raw entries without calling `wrapClassModule`.

#### During tree-walk: collect raw entries

`emitClasses` skips `wrapClassModule` and emits raw class module entries:

```nix
fx.send "emit-class" {
  class = k;
  identity = elemIdentity;
  module = module;                    # raw, unwrapped
  ctx = ctx;                          # pipeline context snapshot at emission time
  aspectPolicy = aspectPolicy;
  globalPolicy = globalPolicy;
  traitNames = traitRegistry;
  __rawEntry = true;                  # explicit marker for post-pipeline detection
  isContextDependent =
    (builtins.isFunction module)
    || (aspect.__parametricResolved or false)
    || (aspect.meta.contextDependent or false);
}
```

The `ctx` snapshot captures the pipeline context at the point of emission. For modules emitted at the entity root level (tree-walk), this is the entity-level ctx (e.g., `{ host = ...; }`). For modules emitted within a transition context (e.g., user fan-out), this is the scoped ctx (e.g., `{ host = ...; user = ...; }`). Enrichment bindings (Fix 1) are in `state.currentCtx` and propagate to scope handlers, so modules emitted after enrichment capture the enriched ctx.

Static attrset class modules (not functions) are also emitted raw. `wrapClassModule` already handles these as pass-through (`!builtins.isFunction module` → `{ inherit module; wrapped = false; }`), so the post-pipeline pass is a no-op for them.

The `classCollectorHandler` in `tree.nix` detects raw entries via `param.__rawEntry or false` and stores the full param with metadata. All raw entries use full identity unconditionally for conservative dedup. Non-raw entries follow the existing identity logic.

#### Post-pipeline: wrap all entries

After `runPipeline` completes and `state.classImports` is finalized, a new pass in `pipeline.nix` processes all collected entries:

```nix
wrapCollectedClasses = classImports:
  lib.mapAttrs (class: entries:
    map (entry:
      if !(entry.__rawEntry or false) then
        # Legacy or already-wrapped entry — pass through
        entry
      else
        let
          result = wrapClassModule {
            inherit (entry) module ctx aspectPolicy globalPolicy traitNames;
          };
        in
        if result.unsatisfied or false then
          # Context never provided — trace and drop entirely
          builtins.trace
            "den: class module ${class}@${entry.identity} skipped — context never provided: ${toString result.missingArgs}"
            null
        else
          classCollectorEntry {
            inherit (entry) identity;
            inherit (result) module;
            isContextDependent = result.wrapped || entry.isContextDependent;
            validator = result.validator or null;
            validatorAdvertisedArgs = result.advertisedArgs or null;
          }
    ) entries
  ) classImports;
```

The `classCollectorEntry` helper constructs the module attrset that `classCollectorHandler` currently produces (with `key`/`_file`/`imports` or `setDefaultModuleLocation`, depending on `isAnon`). This logic is extracted from `classCollectorHandler` into a shared function used by both the collector (for the module structure) and the post-pipeline pass (for the final wrapped module).

**Fan-out correctness:** Each `resolveContextValue` call (one per user/home context) produces its own `emit-class` effects with its own `ctx` snapshot. Post-pipeline wrapping processes each entry with its captured ctx, so `{ user, ... }:` correctly sees a different `user` for each fan-out branch.

**Safety net:** When `wrapClassModule` returns `unsatisfied` at post-pipeline time, the module's required args were never provided by any policy or entity binding. The module is dropped entirely (not emitted) and a trace is emitted. No crash, no infinite recursion.

#### What `emitClassFromDLQ` becomes

The dead letter queue's `emitClassFromDLQ` currently calls `wrapClassModule` inline. With post-pipeline wrapping, DLQ entries should also be emitted raw (with ctx snapshot) and wrapped in the post-pipeline pass. `emitClassFromDLQ` simplifies to emit raw entries with the DLQ's ctx, following the same pattern as `emitClasses`.

## Interaction with existing mechanisms

### Deferred includes (parametric wrappers)

Unchanged. `{ isNixos }: { nixos = ... }` continues to defer via the existing deferred include mechanism. Context enrichment (Fix 1) installs handlers that satisfy deferred includes, which then resolve and emit raw class entries with the enriched ctx.

### Dead letter queue (unregistered keys)

DLQ entries are emitted raw (like all other class entries) and wrapped in the post-pipeline pass. The DLQ continues to handle unregistered keys; the post-pipeline pass handles wrapping. The DLQ fires after all enrichment has been applied (enrichment runs during transition handling, DLQ runs after transitions complete), so the ctx snapshot captured at DLQ emission time includes all accumulated enrichment bindings. This is guaranteed by handler stack ordering: transition handler → enrichment → drain-deferred → DLQ processing.

### Collision validator

Collision validators are emitted during the post-pipeline pass alongside their wrapped modules. The validator uses `validatorAdvertisedArgs` (from the earlier fix). No changes to collision detection logic.

### Include-level dedup

Unchanged. Class module collection is orthogonal to include dedup.

### `isContextDependent` and dedup

Currently, `wrapClassModule` sets `isContextDependent = true` when wrapping occurs, which affects the class module's `baseIdentity` in `classCollectorHandler` (context-dependent modules use full identity including ctxId). With post-pipeline wrapping, the raw entry must conservatively set `isContextDependent` based on what's known at emission time (parametric resolution, aspect meta). The post-pipeline pass updates it if wrapping occurs. This is reflected in the `classCollectorEntry` construction above.

**Dedup safety:** The class collector deduplicates entries by identity during collection (before post-pipeline wrapping). Since `isContextDependent` can change after wrapping, the collector must use the *conservative* identity for dedup — treating function modules as potentially context-dependent (full identity with ctxId). This is safe because:
1. If two entries share the same full identity (including ctxId), they originate from the same aspect in the same context scope — deduping them is correct regardless of whether wrapping later confirms context dependence.
2. If wrapping reveals a module is NOT context-dependent (no ctx args consumed), the post-pipeline pass simply sets `isContextDependent = false` on the entry. The dedup already happened correctly because full-identity dedup is strictly more conservative than base-identity dedup.
3. Static attrset modules are already known non-context-dependent at emission time and use base identity for dedup — no change needed.

In practice: the collector treats all function-typed modules as `isContextDependent = true` for dedup purposes. The post-pipeline pass may relax this flag, but never needs to tighten it.

## Files changed

| File | Change |
|---|---|
| `nix/lib/aspects/fx/handlers/transition.nix` | Fix 1: `classifyResolve` splits mixed/pure enrichment/schema resolves; enrichment processed inline with `scope.provide` + drain |
| `nix/lib/aspects/fx/aspect.nix` | Fix 2: `emitClasses` emits raw entries without calling `wrapClassModule`; simplify `emitClassFromDLQ` |
| `nix/lib/aspects/fx/handlers/tree.nix` | Extract `classCollectorEntry` helper from `classCollectorHandler` |
| `nix/lib/aspects/fx/pipeline.nix` | Fix 2: `wrapCollectedClasses` post-pipeline pass |
| `templates/ci/modules/features/policy-context-enrichment.nix` | Tests (already committed, 1/3 passing) |

## Test cases

### Already committed (policy-context-enrichment.nix)

- **test-parametric-wrapper-defers-for-policy-context** — (passing) Parametric wrapper `{ isNixos }:` defers until policy enriches context
- **test-flat-form-class-defers-for-policy-context** — (failing) Flat-form `{ isNixos, lib, ... }:` should resolve after enrichment
- **test-policy-context-darwin-branch** — (failing) Same pattern on darwin with `{ isDarwin, lib, ... }:`

### To add

- **test-enrichment-fan-out** — Policy resolves `{ user }` for multiple users AND `{ isNixos }` for enrichment. Class module `{ user, isNixos, ... }:` fires once per user with correct values.
- **test-enrichment-never-provided** — Class module requests `{ neverProvided, ... }:`. No policy provides it. Silently dropped with trace, no crash.
- **test-enrichment-multiple-policies** — Two policies each provide different enrichment keys. Class module uses both.
- **test-static-class-module-unchanged** — Static attrset class module unaffected by post-pipeline wrapping.
- **test-enrichment-with-traits** — Trait arg and enrichment arg coexist in class module signature.
- **test-enrichment-chained-policies** — Policy A provides `isNixos = true` via enrichment. Policy B takes `{ isNixos, ... }:` and provides `platform = "linux"` via enrichment. Class module `{ isNixos, platform, ... }:` sees both. Validates fixed-point iteration across enrichment rounds.
- **test-mixed-resolve-split** — Policy returns `resolve { nixos = config; isNixos = true; }`. The `nixos` key creates a child entity transition; `isNixos` enriches current context. Both effects apply correctly.

## Non-goals

- Changing how parametric wrapper deferral works (already correct)
- Modifying the collision validator mechanism (already fixed)
- Supporting enrichment resolve with `resolve.to` (explicit target kind always creates transitions)
- Changing `resolve.shared` semantics for enrichment (irrelevant — no fan-out)
- Static allowlists of module-system args (detection is handled by deferring to post-pipeline where full ctx is available)
