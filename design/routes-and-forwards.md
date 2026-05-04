# Routes and Forwards: Design Spec (feat/fx-pipeline, 629/629 tests)

Status: Documents the **implemented** route and forward infrastructure as of 2026-05-04.

---

## 1. policy.route: Effect Shape, Registration, Application

### Effect shape

Defined in `nix/lib/policy-effects.nix`:

```nix
route = spec: {
  __policyEffect = "route";
  value = spec;
};
```

Policies return `policy.route { fromClass; intoClass; path; guard?; adaptArgs?; ... }` in their effect list. The pipeline classifies it during dispatch (`policy-dispatch.nix` `classifyPolicyResult`) into `routeEffects`.

### Registration via registerRouteHandler

In `nix/lib/aspects/fx/handlers/tree.nix`, the `registerRouteHandler` handles the `"register-route"` effect:

- Captures `sourceScopeId` (defaults to `state.currentScope`).
- Deduplicates via a composite key: `"${fromClass}>${intoClass}@${sourceScopeId}/${path}"`.
- Appends the route to `state.scopedRoutes` under the current scope.
- Tracked separately in `state.registeredRouteKeys` to prevent re-registration from multiple dispatch levels.

The effect is emitted by `policyEmitEffects` in `policy-dispatch.nix`:
```nix
fx.send "register-route" e.value
```

### Post-pipeline application in applyRoutes

In `nix/lib/aspects/fx/route.nix`, `applyRoutes` runs after pipeline execution:

1. Collects all routes from `scopedRoutes` into a flat list.
2. Deduplicates adapter routes by `adapterKey@scopeId`.
3. Suppresses root-scope specs when child-scope copies exist (scope isolation).
4. Folds over the route list, branching on `route.__complexForward`:
   - **Simple routes** (route-eligible): reads `wrappedPerScope.${sourceScopeId}.${fromClass}`, wraps via `wrapRouteModules`, appends to `classImports.${intoClass}`.
   - **Complex routes** (`__complexForward = true`, adapter-requiring): uses `buildForwardAspect` with full adapter/guard machinery, can resolve external sources via `fxResolve` sub-call.

---

## 2. wrapRouteModules: Path Nesting, Guard Wrapping, adaptArgs

Defined in `nix/lib/aspects/fx/route.nix`. Transforms source modules for injection into a target class.

### Pipeline: `adaptModule -> nestModule -> guardModule`

Each source module passes through three stages (applied right-to-left in the map):

**adaptModule** (only for `path == []`):
- If `adaptArgs != null` and `path == []` and module is a function, wraps it: `args: mod (adaptArgs args)`.
- For `path != []`, adaptArgs is handled inside nestModule instead.

**nestModule** (for `path != []`):
- Two sub-branches depending on whether `adaptArgs` is set:
  - **With adaptArgs**: Evaluates source modules in a freeform submodule with `specialArgs = adaptArgs(fullArgs)`, then places `evaluated.config` at the target path via `lib.setAttrByPath`. Supports both raw module functions and wrapped attrsets.
  - **Without adaptArgs**: Resolves source module's function imports with full outer module args (`args // config._module.args`), then places the result at the target path. Works for submodule targets (e.g., `users.users.<name>`) and plain option namespaces (e.g., `wsl`).
- For `path == []`: passthrough (no nesting).

**guardModule** (optional):
- Wraps the nested module in `lib.mkIf (guard args) inner.config`. The guard receives the target class's module args.

### Adapter route variant

For routes with `adapterKey != null` (adapter-requiring forwards), `applyRoutes` skips `wrapRouteModules` and instead emits a functor module:
```nix
{
  __functionArgs = guardArgs // intoPathArgs // adaptArgv;
  __functor = _: args: {
    options.den.fwd.${key} = mkOption { type = submoduleWith { specialArgs; modules = adapterMods ++ [sourceModule]; }; };
    config = guardFn args (setAttrByPath (intoPathFn args) args.config.den.fwd.${key});
  };
}
```
This evaluates source modules inside a submodule with adapter options, then places the result at the dynamic path.

---

## 3. policy.provide: Direct Content Delivery

### provideHandler

In `nix/lib/aspects/fx/handlers/provide.nix`, the `"register-provide"` handler stores specs in `state.scopedProvides` keyed by scope.

### Application in fxResolve

Provides run **before** routes in fxResolve (so complex forward-derived routes can see provide-injected modules). For each provide spec:

1. Resolves the scope context.
2. Wraps the module with `wrapClassModule` (argument injection, policy guards).
3. Sets a dedup location: `"${targetClass}@<provide>/${path}"`.
4. Appends to both `classImports` and the per-scope tracking.

Deduplication: keyed by `"${policyName}/${class}/${path}"` to prevent identical provides from multi-scope firing.

### When to use route vs provide

| Use case | Mechanism |
|----------|-----------|
| Move already-collected pipeline content between classes | `policy.route` |
| Inject new content that was never walked by the pipeline | `policy.provide` |
| Adapter modules, dynamic paths, list-dedup options | Complex forward (via `den.provides.forward`) |

---

## 4. Complex Forwards: What They Are and Why They Exist

Complex forwards are routing operations that `policy.route` cannot express. They use `den.provides.forward` / `forwardItem` and produce aspects with `meta.__forward` markers that the pipeline emits via `"emit-forward"`.

### Why they still exist

1. **Adapter modules**: Custom option declarations (e.g., `types.submoduleWith`) that merge source content through a typed intermediate. Example: home-manager integration where source modules are evaluated in a submodule with `specialArgs`.

2. **Dynamic paths**: `intoPath` as a function of target config — e.g., `forwardPathFn { host, user }` computing `["home-manager" "users" user.name]`.

3. **Cross-entity source resolution**: Forwards with `fromAspect = den.lib.resolveEntity "user" ctx` — source content comes from a different entity whose pipeline output may not be in the current scope partition.

4. **mapModule transformations**: Pre-processing source modules before injection (used by `home-env.nix`).

### Current complex forward users

- `home-env.nix` (`makeHomeEnv`): home-manager, hjem, maid integration
- User-defined custom forwards with adapter modules

---

## 5. Forward Scope Isolation: Root-to-Child Propagation

### The problem

When a forward fires at both root scope (host entity) and child scope (user entity), naive aggregation produces duplicates — root scope contains ALL modules while child scope contains only that entity's modules.

### The solution in applyRoutes

For complex forward-derived routes:

1. **Child-scope forwards**: Source modules come from `perScope.${childScopeId}.${fromClass}` plus a **filtered root fallback** — only `@default` identity modules (den.default shared content) from root scope are included.

2. **Root-scope forwards**: Use the full flattened `classImports.${fromClass}` (aggregate of all scopes).

3. **Root suppression**: When child-scope copies of the same `adapterKey` exist, root-scope specs are suppressed entirely. The child copies handle the forward with proper isolation.

The `isDenDefaultModule` filter:
```nix
isDenDefaultModule = mod:
  let k = mod.key or mod._file or "";
  in lib.hasSuffix "@default" k;
```

This ensures shared infrastructure (den.default aspect) is available in child scopes without duplicating entity-specific content.

---

## 6. Forward Auto-Detection (Implemented)

The `forwardHandler` in `handlers/forward.nix` classifies forwards into route-eligible (Tier 1) vs complex (Tier 2) at handler time:

```nix
isTier1 = isSimpleSpec && sourceIsLocal && sourceAlreadyCollected;
```

Where:
- `isSimpleSpec`: `canDirectImport && !needsAdapter && !evalConfig`
- `sourceIsLocal`: no `__scopeHandlers` on source aspect (same entity)
- `sourceAlreadyCollected`: source class exists in current scope's `scopedClassImports`

When all three conditions hold, the forward is stored as a simple route shape (no `__complexForward` marker). Otherwise, the full forward spec is stored with `__complexForward = true`. Both tiers go into `scopedRoutes` and are handled uniformly by `applyRoutes` in `route.nix`.

This automatic classification means users writing simple `den.provides.forward` entries get route performance without migration.

**Related completions:**
- **Scope-prefixed dedup keys:** Both `includeSeen` (`check-dedup.nix`) and `ctxSeen` (`ctx.nix`) are scope-prefixed, ensuring correct dedup across scopes.
- **Flat state field removal:** `scopedClassImports` is the sole source of truth during the walk. No flat `classImports` state field exists. Post-pipeline flattening is a local variable in `fxResolve`, not a state field.

---

## 7. Built-in Routes That Replaced Built-in Forwards

On feat/fx-pipeline, the legacy `provides.os-class`, `provides.os-user`, and `provides.wsl` modules from main branch no longer exist as standalone provides. Instead:

| Legacy forward | Replacement mechanism |
|---|---|
| `os-class.nix` (os -> nixos/darwin) | Auto-detected as route-eligible: forward with `fromClass = "os"`, `intoClass = host.class`, `path = []` classifies as simple route |
| `os-user.nix` (user -> OS class) | Part of `makeHomeEnv` complex forward: uses `forwardPathFn`, `resolveEntity`, adapter machinery |
| `wsl.nix` (wsl -> OS) | Forward with guard, path nesting: classifies as complex route with `guardFn` |
| `provides.to-hosts` / `provides.to-users` | Built-in: `emitAspectPolicies` translates to `policy.include` effects during aspect walk |

Cross-entity routing (`provides.to-users`, `provides.to-hosts`, `provides.<name>`) is built into `emitAspectPolicies` in `include-emit.nix`. The old `provides-compat.nix` shim was deleted and its functionality folded in as a first-class pipeline feature. No deprecation — provides is a permanent API.

---

## 8. The Forward Application Sequence in fxResolve

The complete post-pipeline sequence in `nix/lib/aspects/fx/pipeline.nix` (`fxResolve`):

```
1. Pipeline execution (collect)
   -> state.scopedClassImports: { scopeId -> { class -> [raw entries] } }

2. Wrap per-scope (wrapCollectedClasses)
   -> wrappedPerScope: { scopeId -> { class -> [keyed modules] } }
   - Applies wrapClassModule (arg injection, policy guards)
   - Strips enrichment-only args
   - Sets module identity/dedup keys

3. Flatten per-scope -> wrappedClassImports
   -> { class -> [all modules across scopes] }

4. Apply provides (policy.provide)
   -> withProvides / wrappedPerScopeWithProvides
   - Injects new modules directly into target classes
   - Deduped by policyName/class/path

5. Apply routes (applyRoutes)
   -> withRoutes.classImports
   - Simple routes: wrapRouteModules (path nest + guard)
   - Complex routes: buildForwardAspect (adapter submodule eval)
   - Scope isolation: root suppression, @default fallback
   - Adapter dedup: adapterKey@scopeId

6. Apply instantiates (policy.instantiate)
   -> withInstantiates
   - Evaluates entities, places in flake.* output

7. Select target class modules
   -> finalModules = withInstantiates.${class}
   -> { imports = finalModules }
```

The critical ordering invariant: **provides before routes**. Complex forward-derived routes read from `classImports`/`perScope` which must include provide-injected content. This preserves the semantic that provides can feed into forwards (e.g., a provide adding a module to class X, which a forward then routes into class Y).

---

## Key Files

| File | Role |
|------|------|
| `nix/lib/aspects/fx/route.nix` | `wrapRouteModules`, `applyRoutes` |
| `nix/lib/aspects/fx/handlers/forward.nix` | `forwardHandler` (emit-forward), `buildForwardAspect`, route-eligible auto-detect |
| `nix/lib/aspects/fx/handlers/provide.nix` | `provideHandler` (register-provide) |
| `nix/lib/aspects/fx/handlers/tree.nix` | `registerRouteHandler` (register-route) |
| `nix/lib/aspects/fx/pipeline.nix` | `fxResolve` orchestration, application sequence |
| `nix/lib/forward.nix` | `forwardItem`, `forwardEach` (user-facing API) |
| `nix/lib/policy-effects.nix` | `policy.route`, `policy.provide` constructors |
| `nix/lib/aspects/fx/policy-dispatch.nix` | Policy dispatch, effect classification, `policyEmitEffects` |
| `nix/lib/aspects/fx/include-emit.nix` | `emitAspectPolicies` — handles provides cross-entity routing (replaced provides-compat.nix) |
| `nix/lib/home-env.nix` | `makeHomeEnv` (complex forward example) |
| `nix/lib/aspects/fx/wrap-classes.nix` | `wrapCollectedClasses` (post-pipeline module wrapping) |
