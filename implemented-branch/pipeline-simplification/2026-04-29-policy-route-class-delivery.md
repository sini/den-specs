# Policy Route — Unified Class and Trait Delivery

## Status: Draft (rev 3 — reviewed, scope expanded)

## Problem

The forward mechanism (`den.provides.forward`, `emit-forward`, `forwardHandler`, `applyForwardSpecs`, `resolveForwardSource`) is legacy infrastructure that violates den's four-concern separation. It embeds routing decisions (intoClass, intoPath, guards, adaptArgs) into the collection layer, runs post-pipeline sub-pipelines to re-resolve source content, and bypasses scope-partitioned state entirely.

With scope partitioning shipped, all class and trait content exists in scope partitions after pipeline execution. Forward source content is already collected — the sub-pipeline redundantly re-resolves it. The routing and transformation logic belongs in policies, not in the collection layer.

## Design

### Two-tier delivery model

**Tier 1 (common case, ~90% of usage):** `policy.route` — a new policy effect that routes class or trait content from scope partitions into a target class at a specified path. No `den.provides.forward` needed. Traits are first-class route sources alongside classes.

**Tier 2 (advanced case, ~10%):** Simplified forward handler that reads from scope partitions via `sourceScopeId`. For cases requiring adapter modules or dynamic `intoPath` from target config.

### Prerequisite: scope-prefixed dedup keys

Tier 2 forwards with cross-entity sources use inline-walk at `emit-forward` time (push scope, `aspectToEffect`, pop scope). This requires scope-prefixed dedup keys so the inline walk has its own dedup namespace. Deferred from scope-partitioning Task 2:

- `ctxSeenHandler` (`state.seen`): key format `"${state.currentScope}/${key}"` — currently keys are `"${targetClass}/${path}"` (single-context) or `"${key}/{ctxNames}"` (fan-out)
- `includeSeen` in include handler (`state.includeSeen`): dedup key `"${state.currentScope}/${childIdentity}"` — currently `childIdentity = identity.pathKey (identity.aspectPath child)`

Without this, an aspect included in both the parent pipeline and the forward source would be incorrectly dedup'd — the second occurrence skipped.

### `policy.route` effect

New policy effect alongside `resolve`, `include`, `exclude`. Effect constructor in `nix/lib/policy-effects.nix`:

```nix
route = spec: {
  __policyEffect = "route";
  value = spec;
};
```

Usage:

```nix
policy.route {
  fromClass = "niri";       # source class bucket (mutually exclusive with fromTrait)
  # fromTrait = "persist";  # OR source trait name
  intoClass = "homeManager";
  path = [ "programs" "niri" ];  # module path nesting in target class
  guard = { options, ... }: options.programs ? niri;  # optional
  adaptArgs = lib.id;  # optional
}
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `fromClass` | string | one of fromClass/fromTrait | Source class bucket to read from scope partition |
| `fromTrait` | string | one of fromClass/fromTrait | Source trait name to read from scope partition |
| `intoClass` | string | yes | Target class to deliver into |
| `path` | `[string]` | yes | Module path nesting in target. `[]` = inject at root (no nesting). |
| `guard` | function | no | `{ options, ... }: bool` — target-config guard, applied as `lib.mkIf` wrapping |
| `adaptArgs` | function | no | `args: args'` — argument adaptation for source modules (class routes only) |

### Route semantics

When a policy emits `policy.route`, the effect is classified during policy dispatch (alongside resolve/include/exclude) and registered as a route in pipeline state.

**`sourceScopeId` capture:** The scope in which the policy fired. This is the scope whose partition contains the source content.

**Post-pipeline application in `fxResolve`:**

1. Read source content:
   - **Class routes:** `wrappedPerScope.${route.sourceScopeId}.${route.fromClass} or []`
   - **Trait routes:** `inheritTraits { ... } route.sourceScopeId route.fromTrait strategy` → synthesize module via `lib.setAttrByPath`
2. Wrap at path via `wrapRouteModules`
3. Inject into `route.intoClass` bucket

**Application ordering (critical):** Routes apply BEFORE Tier 2 forwards. Tier 2 forwards read from post-route data. The sequence in `fxResolve` is:

```
1. Collect → scopedClassImports (pipeline execution)
2. Wrap per-scope → wrappedPerScope
3. Apply Tier 1 routes → withRoutes (reads wrappedPerScope, produces new entries)
4. Apply Tier 2 forwards → forwardedPerScope (reads withRoutes)
5. Flatten across scopes → allImports
```

### Trait-to-module transformation (`fromTrait`)

Trait data is semantic values (lists, maps, scalars). To inject trait data into a class at a module path, synthesize a module:

```nix
# For fromTrait routes, synthesize a NixOS module from trait data:
traitRouteModule = route: traitData:
  { lib, ... }: {
    config = lib.setAttrByPath route.path traitData;
  };
```

The trait data IS the config value. `lib.setAttrByPath` nests it at the target path. The module system handles merging with other definitions at that path based on the option type.

**Example — impermanence persist trait:**

```nix
# Trait schema:
den.traits.persist = { collection = "list"; };

# Aspect emits trait data:
den.aspects.my-app.persist = [ "/var/lib/my-app" "/etc/my-app" ];

# Policy routes trait into nixos:
den.policies.persist-delivery = { host, ... }:
  [ (policy.route {
      fromTrait = "persist";
      intoClass = "nixos";
      path = [ "preservation" "preserveAt" "/persist" ];
    })
  ];

# Synthesized module:
{ lib, ... }: {
  config.preservation.preserveAt."/persist" = [ "/var/lib/my-app" "/etc/my-app" ];
}
```

Trait collection strategy (list/map/single) determines how data aggregates across aspects. The aggregated result becomes the config value. `adaptArgs` is not applicable to trait routes (traits are data, not modules).

**Aggregation scope:** Trait routes use `inheritTraits` with `sourceScopeId` — the same scope-tree-walking aggregation used by `traitModuleForScope`. For `list` traits, values accumulate up the scope tree (child appends to parent). For `map` traits, child overrides parent. For `single`, only the direct scope's value is used. This means a trait route captures the full inherited trait value for the source scope, not just the aspects emitted directly in that scope.

### `wrapRouteModules` function

Wraps source modules (class routes) or synthesized modules (trait routes) with path nesting and optional guard. Follows the same nesting pattern as `buildForwardAspect`'s `mkDirectAspect`:

```nix
wrapRouteModules = { modules, path, guard ? null, adaptArgs ? null }:
  let
    adaptModule = mod:
      if adaptArgs == null then mod
      else if builtins.isFunction mod then
        args: mod (adaptArgs args)
      else mod;

    nestModule = mod:
      if path == [] then mod
      else
        # Produce a top-level NixOS module for the target class.
        # lib.setAttrByPath nests inside config; the inner value is
        # a submodule definition that imports the source module.
        # This requires the option at `path` to be a submodule type.
        { config = lib.setAttrByPath path { imports = [ mod ]; }; };

    guardModule = mod:
      if guard == null then mod
      else
        # Guard wraps the ENTIRE module in mkIf at the top level.
        # Must be a function module to access args for the guard predicate.
        args:
        let inner = if builtins.isFunction mod then mod args else mod;
        in { config = lib.mkIf (guard args) (inner.config or inner); };
  in
  map (mod: guardModule (nestModule (adaptModule mod))) modules;
```

**Constraint:** When `path != []`, the option at that path in the target class must be a submodule type (e.g., `types.submodule`, `types.submoduleWith`). The NixOS module system processes submodule values through `evalModules`, which handles the nested `imports`. For path `[]` (top-level injection), no type constraint applies — source modules are directly imported.

The actual implementation should reuse `buildForwardAspect`'s nesting patterns for consistency and use `lib.setDefaultModuleLocation` for dedup keys.

### Pipeline state changes

**New state field:**

```nix
scopedRoutes = _: { };  # { scopeId = [ routeSpec ]; }
```

**New handler: `registerRouteHandler`**

Handles `"register-route"` effect:

```nix
registerRouteHandler = {
  "register-route" =
    { param, state }:
    let
      scope = state.currentScope;
      route = param // { sourceScopeId = scope; };
    in {
      resume = null;
      state = state // {
        scopedRoutes = _:
          let all = state.scopedRoutes null; in
          all // { ${scope} = (all.${scope} or []) ++ [ route ]; };
      };
    };
};
```

### Edge cases

**Empty source:** If `fromClass` or `fromTrait` doesn't exist in the source scope partition, the route produces an empty module list. Silent no-op — no error, no dead letter. The policy author is responsible for matching routes to existing content.

**Circular routes (`fromClass == intoClass`):** Valid. Useful for path-nesting within the same class (e.g., routing `nixos` content into `nixos` at a subpath). The route reads from the pre-route data and writes to the post-route result — no infinite loop since routes apply in one pass.

**Fan-out (multiple routes for same `fromClass`):** Valid and intentional. The same source content can route to multiple target classes or multiple paths. Each route produces its own wrapped modules independently.

**Route ordering within a scope:** Routes from the same scope are applied in registration order (list append). The resulting module list order in `intoClass` follows this order. Routes do NOT compose — a route's output is never the input of another route. All routes read from `wrappedPerScope` (pre-route data).

### Simplified forward handler (Tier 2)

For advanced cases that `policy.route` doesn't cover:

- **Adapter modules** (`adapterModule`): Custom module wrapping with list-dedup or other NixOS module-system adaptations. These need `buildForwardAspect`'s adapter machinery.
- **Dynamic `intoPath` from target config**: `intoPath = { config, ... }: [ "my-box" config.my-slot ]` — path depends on the target class's evaluated config.

#### `sourceScopeId` computation

The forward handler must distinguish same-entity vs cross-entity sources:

```nix
sourceScopeId =
  if hasOwnContext then
    # Cross-entity: source aspect has __scopeHandlers from a different entity.
    # Compute scope ID from the source entity's context.
    mkScopeId sourceCtx
  else
    # Same-entity: source content is in the declaring scope.
    state.currentScope;
```

Where `hasOwnContext = spec.sourceAspect.__scopeHandlers or {} != {}` and `sourceCtx = ctxFromHandlers spec.sourceAspect.__scopeHandlers`.

For same-entity forwards (`sourceAspect = lib.head aspect-chain`), `sourceScopeId = state.currentScope` — the source content was emitted in the same scope.

For cross-entity forwards (`sourceAspect = den.lib.resolveEntity "user" ctx`), `sourceScopeId = mkScopeId sourceCtx` — the source entity was walked in a child scope during transitions.

#### Inline-walk for unreachable source entities

If a forward references a `resolveEntity` source that was never walked as a transition (e.g., `forward-flake-level.nix` pattern), the source content doesn't exist in any scope partition.

The forward handler detects this case: `sourceScopeId` computed from `sourceCtx` doesn't exist in `state.scopeContexts`. When this happens, the handler inline-walks the source entity:

```nix
# In emit-forward handler, after computing sourceScopeId:
if hasOwnContext && !(state.scopeContexts null) ? ${sourceScopeId} then
  # Source entity not in transition tree — inline-walk it.
  # Push scope, resolve source via aspectToEffect, pop scope.
  # Requires scope-prefixed dedup keys (ctxSeen, includeSeen).
  fx.bind pushSourceScope (_:
    fx.bind (aspectToEffect normalizedSource) (_:
      fx.bind popSourceScope (_:
        # Now sourceScopeId partition is populated
        registerForwardSpec)))
else
  # Source entity already walked — just register the spec
  registerForwardSpec
```

This requires the prerequisite scope-prefixed dedup keys (ctxSeen, includeSeen) so the inline walk doesn't interfere with the parent pipeline's dedup state.

#### What changes vs current

- Delete `resolveForwardSource` (sub-pipeline)
- Delete `__resolveCtx` and `__aspectPolicies` from forward specs
- Forward handler computes `sourceScopeId` (same-entity vs cross-entity)
- Forward handler inline-walks unreachable source entities
- `applyForwardSpecs` reads `scopedClassImports.${spec.sourceScopeId}.${spec.fromClass}`

#### What stays

- `den.provides.forward` user API (for Tier 2 cases only)
- `forwardItem` / `forwardEach` in forward.nix (simplified)
- `buildForwardAspect` (adapter/guard wrapping)
- `emit-forward` effect (with simplified handler)

### Built-in policy routes replacing built-in forwards

| Current forward | Replacement |
|----------------|-------------|
| `os-class.nix` (os → nixos/darwin) | Built-in policy: `policy.route { fromClass = "os"; intoClass = host.class; path = []; }` |
| `os-user.nix` (user → OS) | Built-in policy: `policy.route { fromClass = "user"; intoClass = host.class; path = ["users" "users" user.userName]; adaptArgs = args: args // { osConfig = args.config; }; }` |
| `wsl.nix` (wsl → OS) | Built-in policy: `policy.route { fromClass = "wsl"; intoClass = host.class; path = ["wsl"]; guard = { options, ... }: options ? wsl; }` |
| `disko.nix` user configs | User policy: `policy.route { fromClass = "disko"; intoClass = "nixos"; path = []; }` |
| `niri.nix` user configs | User policy: `policy.route { fromClass = "niri"; intoClass = "homeManager"; path = ["programs" "niri"]; guard = { options, ... }: options.programs ? niri; }` |

The `osConfigurations.nix`, `hmConfigurations.nix`, and `flakeSystemOutputs.nix` forwards use `mapModule` and complex wrapping — these stay as Tier 2 forwards.

### User experience

**Tier 1 — common case (policy.route):**
```nix
den.classes.niri = { description = "Niri"; };

den.policies.niri-delivery = { user, ... }:
  [ (policy.route {
      fromClass = "niri";
      intoClass = "homeManager";
      path = [ "programs" "niri" ];
      guard = { options, ... }: options.programs ? niri;
    })
  ];
```

**Tier 1 — trait route:**
```nix
den.traits.persist = { collection = "list"; };

den.policies.persist-delivery = { host, ... }:
  [ (policy.route {
      fromTrait = "persist";
      intoClass = "nixos";
      path = [ "preservation" "preserveAt" "/persist" ];
    })
  ];
```

**Tier 2 — adapter module (den.provides.forward):**
```nix
den.provides.forward {
  fromClass = "niri";
  intoClass = "homeManager";
  intoPath = [ "programs" "niri" ];
  guard = { options, ... }: options.programs ? niri;
  adapterModule = { ... }: {
    options.settings.spawn-at-startup = lib.mkOption { type = ...; apply = lib.unique; };
  };
}
```

### What this enables for scoped fxResolve

With `policy.route` and simplified forwards both reading from scope partitions:

1. `fxResolve` switches ALL reads to scoped partitions
2. No post-pipeline sub-pipelines
3. Per-scope wrapping with scope-specific ctx
4. Flat state fields can be removed (dual-writes eliminated)

### Changes per file

| File | Change |
|------|--------|
| `nix/lib/aspects/fx/pipeline.nix` | Add `scopedRoutes` to state. Route application in `fxResolve` (after wrapping, before forwards). `applyForwardSpecs` reads scope partitions. Delete `resolveForwardSource`. |
| `nix/lib/aspects/fx/handlers/tree.nix` | Add `registerRouteHandler`. |
| `nix/lib/aspects/fx/handlers/forward.nix` | Compute `sourceScopeId` (same-entity vs cross-entity). Inline-walk for unreachable sources. Remove `__resolveCtx`/`__aspectPolicies`. |
| `nix/lib/aspects/fx/handlers/transition.nix` | Classify `policy.route` effects during policy dispatch. Emit `register-route` for route effects. |
| `nix/lib/aspects/fx/handlers/ctx.nix` | Scope-prefix `ctxSeenHandler` keys: `"${state.currentScope}/${key}"` |
| `nix/lib/aspects/fx/handlers/include.nix` | Scope-prefix `includeSeen` dedup key: `"${state.currentScope}/${childIdentity}"` |
| `nix/lib/policy-effects.nix` | Add `route` effect constructor. |
| `modules/aspects/provides/forward.nix` | Simplify for Tier 2. |
| `modules/aspects/provides/os-class.nix` | Replace with built-in policy. |
| `modules/aspects/provides/os-user.nix` | Replace with built-in policy. |
| `modules/aspects/provides/wsl.nix` | Replace with built-in policy. |
| `nix/lib/forward.nix` | Simplify `forwardItem` for Tier 2. Remove `sourceAspect`/`each`/`fromCtx`. |
| `nix/lib/home-env.nix` | No change — `makeHomeEnv` forwards stay as Tier 2 (uses `mapModule`, adapter wrapping). |
| `templates/ci/modules/features/route.nix` | New test module: class routes, trait routes, guard routes, fan-out, circular, empty-source no-op. |
| `templates/ci/modules/features/route-builtin.nix` | Test: built-in policy routes replace os-class, os-user, wsl forwards (regression coverage). |

### Migration strategy

1. Scope-prefix dedup keys (`state.seen` in ctxSeenHandler, `state.includeSeen` in include handler) — prerequisite
2. Ship `policy.route` effect + `registerRouteHandler` + post-pipeline route application + `wrapRouteModules`
3. Add route test coverage (`route.nix`) — class routes, trait routes, guards, edge cases
4. Convert built-in forwards to built-in policy routes (os-class, os-user, wsl) + regression tests (`route-builtin.nix`)
5. Simplify `emit-forward` handler (sourceScopeId, inline-walk for unreachable sources)
6. Simplify `applyForwardSpecs` (scope partition reads, delete `resolveForwardSource`)
7. Switch `fxResolve` to scoped reads for ALL state (class imports, traits, DLQ)
8. Remove flat state fields and dual-writes
9. Deprecate `den.provides.forward` for Tier 1 cases (keep for Tier 2)
