# Routes and Forwards

Component reference for the routing subsystem on feat/fx-pipeline (753/753 tests).

Routes move module content collected during the aspect pipeline from one class into another. Forwards are the user-facing API for declaring routing intent; the pipeline compiles them into route specs that the post-pipeline assembly applies. Simple forwards become lightweight routes; complex forwards (those needing adapters, dynamic paths, or cross-entity resolution) carry the full forward spec through a separate application path.

---

## 1. policy.route

### Effect shape

```nix
route = spec: {
  __policyEffect = "route";
  value = spec;
};
```

A policy returns `policy.route { fromClass; intoClass; path; guard?; adaptArgs?; ... }`. The pipeline's policy dispatch classifies it into `routeEffects` and emits:

```nix
fx.send "register-route" e.value
```

### register-route handler

Defined in `handlers/route.nix`. On receiving a `"register-route"` effect:

1. Sets `sourceScopeId` to `param.sourceScopeId` or the current scope.
2. Computes a composite dedup key: `"${fromClass}>${intoClass}@${sourceScopeId}/${path}"`.
3. Checks `state.registeredRouteKeys` -- if the key exists, the handler is a no-op.
4. Otherwise appends the route to `state.scopedRoutes` under the current scope and records the key.

```nix
registerRouteHandler = {
  "register-route" = { param, state }:
    let
      route = param // { sourceScopeId = param.sourceScopeId or scope; };
      routeKey = "${route.fromClass or "?"}>${route.intoClass or "?"}@${route.sourceScopeId}/${...}";
    in
    if alreadyRegistered then { resume = null; state = state; }
    else { resume = null; state = scopedAppend state "scopedRoutes" scope route // { ... }; };
};
```

### Scope-aware dedup

`dedupRoutes` in `route/apply.nix` performs two dedup passes before application:

- **Child suppresses root.** `findChildScopeKeys` collects all `adapterKey` values present at non-root scopes. Any root-scope route sharing an `adapterKey` with a child-scope route is dropped (`isRedundantRoot`).
- **Same key at same scope.** Routes are deduped by `"${adapterKey}@${sourceScopeId}"` -- first occurrence wins.

This prevents duplicate module injection when the same forward fires at both root scope (host entity) and child scope (user entity).

---

## 2. Forward Compilation

### compile-forward handler

Defined in `handlers/compile-forward.nix`. The `"compile"` handler in `compile.nix` dispatches to `"compile-forward"` when the aspect payload has `meta.__forward`.

The handler extracts `spec = param.aspect.meta.__forward` and classifies the forward into one of two tiers:

**Tier 1 (simple route):** All three conditions hold:

- `isSimpleSpec`: `canDirectImport && !needsAdapter && !evalConfig`
- `sourceIsLocal`: source aspect has no `__scopeHandlers` (same entity, no cross-scope resolution)
- `sourceAlreadyCollected`: source class already exists in `scopedClassImports` for the current scope

Produces a minimal route shape:

```nix
simpleRoute = {
  inherit (spec) fromClass intoClass;
  path = spec.staticIntoPath;
  guard = null;
  adaptArgs = null;
  sourceScopeId = scope;
};
```

**Tier 2 (complex route):** Any condition fails. The full forward spec is stored with `__complexForward = true`:

```nix
complexRoute = spec // { sourceScopeId = scope; __complexForward = true; };
```

Both tiers are appended to `state.scopedRoutes`. The handler resumes `[]` -- forwards bypass dedup and constraint checking.

---

## 3. Post-Pipeline Route Application

### applyRoutes entry point

Defined in `route/apply.nix`. Called as Phase 3 of the post-pipeline assembly in `resolve.nix`, after provides (Phase 2) and class wrapping (Phase 1).

```
Phase 1: wrapPerScope       -- wrap raw class imports per scope
Phase 2: applyProvides      -- inject policy.provide modules
Phase 3: applyRoutes        -- move content between classes
Phase 4: applyInstantiates  -- evaluate entities into flake output
```

`applyRoutes` deduplicates via `dedupRoutes`, then folds over the route list, branching on `route.__complexForward`:

### applySimpleRoute

For routes without `__complexForward`:

1. Reads source modules from `wrappedPerScope.${sourceScopeId}.${fromClass}`.
2. Appends any `adapterModule` from the route spec.
3. If the route has an `adapterKey`, delegates to `mkAdapterFunctor` (see Section 5).
4. Otherwise, wraps modules via `wrapRouteModules` (see Section 4).
5. Appends the wrapped modules to `classImports.${intoClass}`.

An `ensureEntry` fallback creates a placeholder module when `adaptArgs` is set but no source modules exist, ensuring the target path is created.

### applyComplexRoute

For routes with `__complexForward = true`:

1. Collects source modules via `getCollectedSource`:
   - If a root scope exists and the route's `sourceScopeId` differs from root, takes the child scope's modules plus `@default`-keyed modules from the root scope (via `isDenDefaultModule` filter).
   - Otherwise uses the full flattened `classImports.${fromClass}`.
2. Falls back to `resolveSourceFallback` if no collected modules found -- runs a sub-pipeline `fxResolve` call against the source aspect.
3. Applies `spec.mapModule` to compose a source module.
4. Calls `buildForwardAspect` to produce an aspect with adapter machinery.
5. Extracts class modules via `collectClassMods` and appends to the target class.

The `isDenDefaultModule` filter:

```nix
isDenDefaultModule = mod: lib.hasSuffix "@default" (mod.key or mod._file or "");
```

This ensures shared infrastructure (den.default aspects) propagates into child scopes without duplicating entity-specific content.

---

## 4. Route Wrapping

### wrapRouteModules

Defined in `route/wrap.nix`. Transforms source modules for injection into a target class by applying a three-stage pipeline to each module:

```nix
wrapRouteModules = { modules, path, guard ? null, adaptArgs ? null }:
  map (mod: guardModule guard (nestModule path adaptArgs (adaptModule adaptArgs path mod))) modules;
```

### adaptModule

Handles top-level `adaptArgs` (when `path == []`). If `adaptArgs != null` and the module is a function, wraps it: `args: mod (adaptArgs args)`. For `path != []`, adaptArgs is handled inside `nestModule` instead.

### nestModule

Nests a module at a target path. Three branches:

- **`path == []`**: Passthrough, no nesting.
- **With adaptArgs**: `nestWithAdaptArgs` evaluates source modules in a freeform submodule with `specialArgs = adaptArgs(fullArgs)`, then places `evaluated.config` at the target path via `lib.setAttrByPath`.
- **Without adaptArgs**: `nestPlain` resolves source module function imports with full outer module args (`args // config._module.args`), then places the result at the target path.

### guardModule

Wraps the nested module in a conditional:

```nix
guardModule = guard: mod:
  if guard == null then mod
  else args: { config = lib.mkIf (guard args) (inner.config or inner); };
```

The guard receives the target class's module args.

See [class-module-wrapping.md](class-module-wrapping.md) for the `wrapClassModule` transform applied during Phase 1 wrapping.

---

## 5. Adapter Functor

### den.fwd.${key} submodule pattern

For routes with an `adapterKey` (requiring adapter modules, dynamic paths, or `adaptArgs`), `applySimpleRoute` delegates to `mkAdapterFunctor` instead of `wrapRouteModules`. This produces a functor module:

```nix
{
  __functionArgs = guardArgs // intoPathArgs // adaptArgv;
  __functor = _: args: {
    options.den.fwd.${key} = lib.mkOption {
      default = { };
      type = lib.types.submoduleWith {
        specialArgs = adaptArgsFn args;
        modules = adapterMods ++ [ sourceModule ];
      };
    };
    config = guardFn args (lib.setAttrByPath (intoPathFn args) args.config.den.fwd.${key});
  };
}
```

The functor:

1. Declares a `den.fwd.${key}` option as a `submoduleWith` type, with `specialArgs` from `adaptArgsFn` and the adapter + source modules.
2. Evaluates the submodule via the NixOS module system fixpoint.
3. Applies the guard function to the result.
4. Places the guarded result at the target path (which may be dynamic via `intoPathFn`).

The `adapterKey` is computed as `lib.concatStringsSep "/" ([fromClass intoClass] ++ staticIntoPath)`.

### Complex route adapter (buildForwardAspect)

Defined in `handlers/forward.nix`. For complex routes, `buildForwardAspect` dispatches between three aspect constructors:

- **`mkDirectAspect`**: No adapter needed. Places source module at `staticIntoPath` via `lib.setAttrByPath`. Optionally evaluates config if `evalConfig = true`.
- **`mkAdapterAspect`**: Has adapter key. Creates a nested aspect with `den.fwd.${adapterKey}` submodule and `includes` containing a direct aspect for the source module.
- **`mkTopLevelAdapterAspect`**: Has adapter but `intoPath == []`. Evaluates source through adapter modules with `lib.evalModules`, applying guard to the result.

---

## 6. forwardTo

Static routing declaration on `den.classes`. Defined as an option in `modules/options.nix`:

```nix
options.forwardTo = lib.mkOption {
  description = "Optional forward target for class evaluation.";
  type = lib.types.nullOr lib.types.raw;
  default = null;
};
```

When `forwardItem` resolves its `intoClass` and `intoPath`, it checks `den.classes.${fromClass}.forwardTo`:

```nix
intoClass =
  if fwd ? intoClass then fwd.intoClass item
  else if classForwardTo != null then classForwardTo.class
  else throw "forward: no intoClass for fromClass=${fromClass}";

intoPath =
  if fwd ? intoPath then fwd.intoPath item
  else if classForwardTo != null then classForwardTo.path or []
  else [];
```

This provides a default routing target for custom classes without requiring every forward to specify `intoClass`/`intoPath`. Explicit `intoClass` on the forward overrides the class-level `forwardTo`.

Example declaration:

```nix
den.classes.custom = {
  description = "Custom class with forwardTo";
  forwardTo = { class = "nixos"; path = []; };
};
```

---

## 7. forwardItem API

### forwardItem

Defined in `nix/lib/forward.nix`. User-facing function for declaring a forward:

```nix
forwardItem = {
  item,            # The aspect or entity being forwarded
  guard ? null,    # args -> bool or args -> item -> attrset filter
  adaptArgs ? null, # args -> specialArgs for adapter submodule
  adapterModule ? null, # NixOS module merged into the adapter submodule
  evalConfig ? false, # Whether to evaluate the source module eagerly
  fromClass,       # item -> string: source class name
  intoClass ?,     # item -> string: target class (falls back to forwardTo)
  intoPath ?,      # item -> path or item -> args -> path: target path
  fromAspect ?,    # item -> aspect: override source aspect
  fromCtx ?,       # item -> ctx: override source context
  mapModule ?,     # item -> module -> module: transform source
  ...
}
```

`forwardItem` does not produce modules directly. It returns an aspect with `meta.__forward` containing the compiled forward spec. The pipeline's `"compile"` handler detects this marker and dispatches to `"compile-forward"`.

Key computed fields in `meta.__forward`:

| Field | Computation |
|-------|-------------|
| `adapterKey` | `concatStringsSep "/" ([fromClass intoClass] ++ staticIntoPath)` |
| `needsAdapter` | `guard != null \|\| adaptArgs != null \|\| adapterModule != null \|\| isFunction intoPath` |
| `needsTopLevelAdapter` | `needsAdapter && intoPath == []` |
| `canDirectImport` | `adapterModule == null` |
| `staticIntoPath` | The path if it is a list; `[]` if it is a function |
| `guardFn` | Normalizes guard to `args -> attrset -> attrset` |
| `adaptArgsFn` | Normalizes adaptArgs, supports `args -> item -> attrset` |

### forwardEach

Batch variant that maps `forwardItem` over a list:

```nix
forwardEach = fwd: {
  includes = map (item: forwardItem (fwd // { inherit item; })) fwd.each;
};
```

---

## 8. Invariants

### Provides before routes

The post-pipeline phase ordering (Phase 2 before Phase 3) ensures that `policy.provide`-injected modules are visible to routes. Complex forward-derived routes read from `classImports`/`perScope`, which must include provide-injected content. This preserves the semantic that a provide adding a module to class X can be picked up by a forward routing class X content into class Y.

### Child suppresses root

When both root scope and child scope register a route with the same `adapterKey`, the root-scope route is suppressed. This prevents duplicate module injection in multi-entity pipelines (e.g., a fleet walk where the same forward fires for both the host entity at root scope and the user entity at child scope). The child scope's route handles the forward with proper scope isolation.

### __complexForward classification

The `compile-forward` handler's tier classification determines the application path:

- **No `__complexForward`**: `applySimpleRoute` -- reads from already-wrapped `perScope` modules, applies `wrapRouteModules` or `mkAdapterFunctor`.
- **`__complexForward = true`**: `applyComplexRoute` -- resolves source modules with scope-aware fallback, applies `buildForwardAspect` with full adapter machinery.

A forward is classified as complex when any of: it needs an adapter, the source is cross-entity (has `__scopeHandlers`), or source modules were not yet collected in the current scope.

### Route dedup composite key

`register-route` uses `"${fromClass}>${intoClass}@${sourceScopeId}/${path}"` to prevent duplicate registration from multiple dispatch levels. `applyRoutes` additionally deduplicates by `"${adapterKey}@${sourceScopeId}"`.

See [scope-partitioning.md](scope-partitioning.md) for how routes interact with scope-partitioned state.

---

## 9. Key Files

| File | Role |
|------|------|
| `nix/lib/aspects/fx/route/apply.nix` | `applyRoutes`, `applySimpleRoute`, `applyComplexRoute`, `dedupRoutes`, `mkAdapterFunctor` |
| `nix/lib/aspects/fx/route/wrap.nix` | `wrapRouteModules`, `nestModule`, `guardModule`, `collectClassMods` |
| `nix/lib/aspects/fx/handlers/compile-forward.nix` | `compileForwardHandler` -- tier classification |
| `nix/lib/aspects/fx/handlers/forward.nix` | `buildForwardAspect`, `mkAdapterAspect`, `mkDirectAspect`, `mkTopLevelAdapterAspect` |
| `nix/lib/aspects/fx/handlers/route.nix` | `registerRouteHandler` |
| `nix/lib/forward.nix` | `forwardItem`, `forwardEach` |
| `nix/lib/aspects/fx/resolve.nix` | Phase ordering (provides before routes), `fxResolve` orchestration |
| `modules/options.nix` | `den.classes.*.forwardTo` option definition |
