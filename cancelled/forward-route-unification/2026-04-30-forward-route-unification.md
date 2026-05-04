# Forward → Route Unification

**Date**: 2026-04-30
**Status**: Partial — Tasks 0/1/6 shipped, compat shim blocked (see Blockers)
**Scope**: Eliminate sub-pipelines by converting all forwards to `policy.route` effects

## Problem

The forward mechanism uses post-pipeline sub-pipelines (`resolveForwardSource` in `pipeline.nix`) to re-resolve source aspects and extract class modules. This is the single most complex piece of the pipeline:

- `resolveForwardSource` runs `fxFullResolve` — a full pipeline re-evaluation
- `applyForwardSpecs` recursively applies forwards within sub-pipelines
- The Tier 1/Tier 2 classification split creates two code paths
- ~250 lines of recursive, hard-to-modify code

Previous attempts to simplify or remove sub-pipelines failed because agents attacked the problem piecemeal — trying to migrate individual sub-pipeline instances or remove transitions — without a unified replacement mechanism.

## Key Insight

At route-application time (post-walk), all scopes are fully populated. The source modules that sub-pipelines re-resolve are already in `scopedClassImports` under the source entity's scope. A route just needs to read from the correct scope ID — no re-resolution needed.

Tier 1 (shipped) already proves this: simple forwards become routes that read from `scopedClassImports`. This design extends that to ALL forwards.

Routing is fundamentally a policy concern — "when entity X is resolved, take its class A modules and place them in class B at path P." `policy.route` already exists as an effect type, is already used in production (flake output routing, OS class routing), and is already collected and dispatched by `transition.nix`. Forwards are just routes that were declared in the wrong place.

## Design

### Routes as Policy Effects (No New API)

`policy.route` already exists in `policy-effects.nix` and is already consumed by the pipeline. The existing spec shape:

```nix
den.lib.policy.route {
  fromClass = "home";
  intoClass = "nixos";
  path = [ "home-manager" "users" "tux" ];
}
```

This design enriches `policy.route` with the capabilities currently exclusive to forwards:

| Field | Purpose | Status |
|---|---|---|
| `fromClass` | Source class bucket | Existing |
| `intoClass` | Target class bucket | Existing |
| `path` | Where to nest in target class | Existing |
| `guard` | Conditional application | **New** |
| `adaptArgs` | Arg injection for submodule contexts | **New** |
| `adapterModule` | Additional module import for submodule nesting | **New** |
| `fromTrait` | Route trait data instead of class modules | Existing |

No new `den.route` API, no route envelopes, no envelope detection in `include.nix`. Forwards become `policy.route` effects emitted from policies — which is where all existing production forward consumers already live.

### Forward Handler and `include.nix` Detection — Deleted

With forwards expressed as `policy.route` effects:

- `forwardHandler` (`emit-forward` effect) is removed from the handler stack
- `include.nix`'s `meta.__forward` detection is removed
- No route envelope detection needed
- The `emit-forward` effect ceases to exist

Routes are registered by `transition.nix` via `register-route` (existing path) during policy dispatch. No new handler needed.

### Route Application Changes

**`mapModule` dropped**: `mapModule` has zero users outside the forward infrastructure (no templates, modules, or production consumers set `fwd.mapModule`). The evaluation-and-placement pattern it served is now handled by `policy.instantiate`. Omitted from the enriched route spec.

**`adapterModule` on routes**: New field. When present, the adapter module is included alongside the source module in the structural nesting (see adapterModule mapping below).

**`guard` and `adaptArgs` on routes**: `applyRoutes` in `route.nix` already handles `guard` and `adaptArgs` for Tier 1 routes. These fields are now available on all `policy.route` declarations.

**`lib.evalModules` elimination**: Current `wrapRouteModules` uses `lib.evalModules` for path nesting and adaptArgs injection. Replaced with structural placement:

```nix
# Current (evaluates eagerly via lib.evalModules):
let
  evaluated = lib.evalModules {
    modules = [ freeformMod sourceModule ];
  };
in
{ config = lib.setAttrByPath path (removeAttrs evaluated.config ["_module"]); }

# New (structural, lazy — consumer's module system handles evaluation):
args: {
  config = lib.setAttrByPath path (_: { imports = [ sourceModule ]; });
}
```

For `adaptArgs` with path nesting, the current sub-pipeline uses `lib.evalModules { specialArgs = adapted; }`. The replacement must use `_module.specialArgs` (not `_module.args`) because adapted args like `pkgs` would conflict with NixOS's own `_module.args.pkgs`:

```nix
args:
let adapted = adaptArgs args;
in {
  config = lib.setAttrByPath path (_: {
    imports = [ sourceModule ];
    _module.specialArgs = adapted;
  });
}
```

`_module.specialArgs` is settable as a module option in modern nixpkgs and bypasses option checks, matching the semantics of the current `lib.evalModules { specialArgs = ... }` call.

This aligns with den's "no lock-in" principle: den places modules structurally and lets the user's chosen module system handle evaluation.

**Missing scope warning**: When `applyRoutes` encounters a `sourceScopeId` not present in `scopedClassImports`, it emits a `builtins.trace` warning and produces no modules. This handles the cross-pipeline case gracefully until fleet-level scope is implemented.

### Compat Shim for `forwardEach`

The existing `den.provides.forward` / `forwardEach` API gets a compatibility shim. Since forwards now must live inside policies, the shim wraps the old forward call as an aspect-included policy that emits `policy.route` effects.

The shim transforms:
```nix
# Old: forward as aspect include
den.provides.forward {
  each = [ user ];
  fromClass = _: "home";
  intoClass = _: host.class;
  intoPath = _: [ "home-manager" "users" user.userName ];
  fromAspect = _: den.lib.resolveEntity "user" { inherit host user; };
}
```

Into an aspect with a policy that emits routes:
```nix
# New: generated by compat shim
{
  policies."fwd-home-to-nixos" = { host, user, ... }:
    map (item:
      den.lib.policy.route {
        fromClass = "home";
        intoClass = host.class;
        path = [ "home-manager" "users" user.userName ];
      }
    ) each;
}
```

Field mapping:

| Old field | New mechanism | Transformation |
|---|---|---|
| `fromClass` | `policy.route { fromClass }` | Direct (call with item) |
| `intoClass` | `policy.route { intoClass }` | Direct |
| `intoPath` (static) | `policy.route { path }` | Direct |
| `intoPath` (function) | `policy.route { path = [] }` + eval-time wrapping | Dynamic path resolved at eval time |
| `fromAspect` | Dropped | Source scope derived from policy dispatch context |
| `fromCtx` | Dropped | Policy dispatch context provides scope identity |
| `guard` | `policy.route { guard }` | Direct |
| `adaptArgs` | `policy.route { adaptArgs }` | Direct |
| `adapterModule` | `policy.route { adaptArgs }` wrapping | See adapterModule mapping below |
| `evalConfig` | Dropped | No longer a concept (structural nesting replaces it) |
| `mapModule` | Dropped | Zero external users; `policy.instantiate` covers this pattern |

#### `fromAspect` Elimination

`fromAspect` is unnecessary because `policy.route` fires during policy dispatch inside `transition.nix`. At dispatch time, the pipeline knows:
- Which entity is being resolved (`sourceEntityKind`)
- The current scope (`state.currentScope`)
- All parent scopes and their contexts (`state.scopeContexts`)

The `sourceScopeId` on the registered route is set to `state.currentScope` by `registerRouteHandler`. For cross-entity routes within the same pipeline (e.g., user modules routed into host), the policy fires during the user entity's transition — the user's scope is already the current scope.

This eliminates the four `fromAspect` patterns:

| Old pattern | New mechanism |
|---|---|
| `lib.head aspect-chain` | Default — policy fires in current entity's scope |
| `host.aspect` (same pipeline) | Policy fires in host's scope (root or transition) |
| `den.lib.resolveEntity "user" ctx` | Policy fires during user transition — scope matches |
| `den.lib.parametric.fixedTo { host } aspect` | Policy fires in appropriate transition scope |

#### Scope Correctness for `home-env.nix`

The `userForward` is included via `policy.include userForward` in the host-to-user policy. The shim converts this to an aspect with a policy requiring `{ host, user, ... }:`. This policy fires during the USER entity's `into-transition` (since both `host` and `user` are required, and `user` only enters context during the user transition). At that point, `state.currentScope` is the user's scope. `registerRouteHandler` stamps `sourceScopeId = state.currentScope` = user scope. The route's `fromClass = "home"` reads `wrappedPerScope.${userScopeId}.home` — which is correct, because the user's home class modules are emitted into the user's scope during the user entity's resolution.

#### `adapterModule` Mapping

The current `adapterModule` field creates `mkAdapterAspect` / `mkTopLevelAdapterAspect` shapes that inject options via `lib.evalModules`. Adapter modules declare OPTIONS (e.g., `options.settings.spawn-at-startup`), not just args — so they cannot be mapped through `adaptArgs`. Instead, the adapter module becomes an additional import in the structural nesting:

```nix
config = lib.setAttrByPath path (_: {
  imports = [ sourceModule adapterModule ];  # adapter declares options, source sets them
  _module.specialArgs = adapted;             # if adaptArgs also present
});
```

The compat shim carries `adapterModule` as a field on the route spec. `wrapRouteModules` includes it as an import alongside the source module when present.

The shim emits `builtins.trace` deprecation warnings for all usage.

#### Policy Name Generation

The compat shim wraps each `forwardItem` as a policy. Multiple forwards within a single `forwardEach` need unique policy names to avoid collision in aspect-policy registration. The shim generates names as `"fwd-compat/${fromClass}-to-${intoClass}/${index}"` where `index` is the position in `fwd.each`.

#### `adaptArgs` Semantics: `_module.args` vs `specialArgs`

The current sub-pipeline injects adapted args via `lib.evalModules { specialArgs = adapted; }`. The structural replacement uses `_module.args`. These have different semantics: `specialArgs` bypass the module system's option check while `_module.args` go through it.

For routes that inject `pkgs` via `adaptArgs` (e.g., flake output routing), `_module.args.pkgs` may conflict with NixOS's own `pkgs` binding. If this is the case, the structural nesting should use `config._module.specialArgs` or a dedicated specialArgs injection mechanism instead of `_module.args`. This must be validated during Phase 1 against existing `adaptArgs` patterns.

### Cross-Host Forwarding (Deferred)

Cross-pipeline forwards (host A pulling from host B's tree) are not supported by this design. They require a fleet-level parent scope where all hosts are children of a shared pipeline. This is deferred because:

- Only exists in test files (`cross-context-forward.nix`), all authored by the maintainer
- Zero production consumers
- The route system is fleet-scope-ready by construction (reads from arbitrary `sourceScopeId`)
- Fleet scope can be added later without changing the route mechanism

## Files Changed

### Deleted

**`nix/lib/aspects/fx/handlers/forward.nix`** — entire file:
- `forwardHandler` (`emit-forward` effect handler)
- `mkDirectAspect`, `mkAdapterAspect`, `mkTopLevelAdapterAspect`
- `buildForwardAspect`, `evalImport`, `guardTree`

### Modified

**`nix/lib/aspects/fx/pipeline.nix`**:
- Delete: `resolveForwardSource`, `applyForwardSpecs`, `collectClassMods`
- Delete: `scopedForwardSpecs` from `defaultState`
- Remove `forwardHandler` from `defaultHandlers` composition
- Simplify `fxResolve`: remove forward post-processing block, `classImports` comes from `withInstantiates` directly

**`nix/lib/aspects/fx/route.nix`**:
- `wrapRouteModules`: replace `lib.evalModules` calls with structural `lib.setAttrByPath` nesting
- `applyRoutes`: add `adapterModule` support, add missing-scope trace warning

**`nix/lib/aspects/fx/handlers/include.nix`**:
- Remove `meta.__forward` detection (the `isForward` branch)

**`nix/lib/forward.nix`**:
- `forwardEach` / `forwardItem`: becomes compat shim producing aspect with policy that emits `policy.route` effects
- Emits deprecation trace for all usage

**`nix/lib/policy-effects.nix`**:
- Enrich `route` constructor to accept `guard`, `adaptArgs`, `adapterModule` fields

### Test Changes

**Delete**:
- Cross-pipeline forward tests in `cross-context-forward.nix` (host A → host B patterns: `test-cross-context-forward-with-ctx`, `test-forward-each-filter-excludes-self`, `test-host-hm-aspects-forward-to-primary-user`, `test-forward-hm-from-other-host-to-local-user`, `test-forward-carries-source-context-data`)

**Preserve** (not forward tests, just entity resolution tests):
- `cross-context-forward.nix`: `test-resolve-other-host-context`, `test-entities-have-resolved`, `test-user-resolved-produces-aspect` — move to a general entity resolution test file

**Rewrite**:
- `debug-fwd.nix`: structural nesting instead of `evalConfig = true`
- `trait-forward-delivery.nix`: evalConfig tests become structural; "evalConfig loses traits" bug disappears
- `flake-scope-pipeline-args.nix`: rewrite evalConfig usage

**Unchanged** (work via compat shim):
- All `fromAspect = _: lib.head aspect-chain` tests
- All guard/adaptArgs/static intoPath tests
- `forward-to.nix`, `forward-from-custom-class.nix`, `forward-alias-class.nix`, `guarded-forward.nix`

### Production Consumers

**Unchanged via compat shim**:
- `nix/lib/home-env.nix` — user→host forward within same pipeline (user resolved by transition)
- `templates/nvf-standalone/modules/nvf-integration.nix` — same-entity forward (`fromAspect = _: lib.head aspect-chain`)

**Require manual migration** (source entity outside current pipeline's scope tree):
- `templates/flake-parts-modules/modules/perSystem-forward.nix` — `fromAspect` resolves `flake-parts` entity independently. Rewrite: make `flake-parts` entity resolution happen within the `flake-parts-system` pipeline's transition chain (add `flake-parts` → `flake-parts-system` transition so parent scope contains source modules), then use `policy.route` to read from parent scope.
- `templates/microvm/modules/microvm-integration.nix` — `fromAspect` resolves guest VM as a host entity in a separate pipeline. Rewrite: resolve the VM as a host within the microvm transition chain (e.g., `resolve.to "host" { host = vm; }` so the VM's host modules land in the current pipeline's scope tree), then use `policy.route` to read from the VM-host scope.

## Migration Phases

### Phase 1 — Enrich `policy.route` (nothing breaks)
1. Add `guard`, `adaptArgs`, `adapterModule` fields to `policy.route` in `policy-effects.nix`
2. Add `adapterModule` support to `route.nix` `applyRoutes`
3. Add missing-scope trace warning to `applyRoutes`
4. Replace `lib.evalModules` in `wrapRouteModules` with structural nesting — validate against existing Tier 1 route tests first, since `evalModules` flattening may have been load-bearing for option merge semantics

### Phase 2 — Forward compat shim
1. `forward.nix` becomes compat shim — `forwardEach` produces aspects with policies emitting `policy.route`
2. `include.nix` drops `meta.__forward` detection

### Phase 3 — Deletion
1. Delete `handlers/forward.nix` entirely
2. Delete `resolveForwardSource`, `applyForwardSpecs`, `collectClassMods` from `pipeline.nix`
3. Delete `scopedForwardSpecs` from `defaultState`
4. Remove `forwardHandler` from `defaultHandlers` in `pipeline.nix`
5. Simplify `fxResolve` post-pipeline block
6. Delete/rewrite affected tests

## Estimated Impact

- **Lines deleted**: ~300 (entire forward handler file, sub-pipeline machinery, evalModules wrappers)
- **Lines added**: ~30 (adapterModule/guard/adaptArgs in route application, compat shim policy wrapper)
- **Net**: ~270 lines removed from the most complex part of the pipeline
- **lib.evalModules calls removed**: 3 (mkDirectAspect, wrapRouteModules x2)
- **Concepts removed**: `emit-forward` effect, forward handler, forward aspect builders, Tier 1/2 classification, sub-pipelines, `mapModule`
- **Production breakage**: None (compat shim covers all consumers)

## Blockers (discovered during implementation 2026-04-30)

The compat shim for `den.provides.forward` cannot be built with the current scope model. Three gaps:

1. **den.default stripping**: Child entities (resolved via transitions) have `den.default` stripped from includes to prevent duplicate NixOS defs. Sub-pipelines re-resolve without stripping. Scope partition reads miss `den.default` content. Scope-tree aggregation (reading ancestors) collects TOO MUCH (parent modules cause duplicates).

2. **Unregistered classes**: Classes not in `den.classes` (e.g., "custom", "src") go to DLQ during tree walk. Sub-pipelines set `class = fromClass` making them recognized as `targetClass`. No equivalent mechanism for scope partition reads.

3. **Guard/adaptArgs semantics**: Forward guards are config transformers (`lib.optionalAttrs`), not boolean gates (`lib.mkIf`). Forward adaptArgs use `evalModules { specialArgs }`, routes use `_module.args`. Direct mapping doesn't work.

**Next step**: Design a scope read model that produces the same module set as a fresh entity resolution, without running a sub-pipeline. The envelope approach (wrapSource closure carrying `buildForwardAspect`) is viable IF source module collection is solved.

## Future Work (Not In Scope)

- **Fleet-level scope**: Parent entity above hosts for cross-host routing. Adds when production need arises.
- **`forwardEach` removal**: Once all consumers migrate to `policy.route`.
