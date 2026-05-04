# Class Module Wrapping and Collision Policies

**Branch:** `feat/fx-pipeline` (629/629 tests passing)
**Date:** 2026-05-04
**Status:** Implemented

## 1. The Problem

Den aspects emit class modules (NixOS, darwin, homeManager) that are consumed by their respective module systems. These module systems expect functions of the form `{ config, pkgs, lib, ... }: { ... }` where args come from `_module.args` / `specialArgs`.

Den has its own context args -- `host`, `user`, `fleet`, and any schema-defined entity kinds -- that live in the pipeline's scope context (`__scopeHandlers`). The ergonomic form is flat:

```nix
{ name = "foo"; nixos = { host, config, pkgs, ... }: { networking.hostName = host.name; }; }
```

Without wrapping, the NixOS module system would need to know about `host` via `specialArgs`, coupling den's context model to each class system. `wrapClassModule` bridges this: it detects which function args are den context keys, pre-applies them, and hands the module system a function that only expects module-system args.

## 2. wrapClassModule

**File:** `nix/lib/aspects/fx/class-module.nix`

### Inputs

```nix
wrapClassModule {
  module;        # The raw class module value
  ctx;           # Den context attrset (reconstructed from scope handlers)
  aspectPolicy;  # aspect.meta.collisionPolicy or null
  globalPolicy;  # den.config.classModuleCollisionPolicy or "error"
}
```

The `ctx` is reconstructed at emit time via `ctxFromHandlers`, which invokes each scope handler with dummy args to extract the original value. This works for all aspects in the tree because `__scopeHandlers` propagates to children.

### Three code paths

**Path 1: Attrset with `imports` key.** The module is `{ imports = [...]; ... }`. Delegates to `wrapDeferredImports` (section 4). Returns the attrset with wrapped imports, plus a collision validator if any inner function was wrapped and ctx is non-empty.

**Path 2: Non-function.** Returns the module unchanged with `wrapped = false`. Plain attrsets and other non-callable values pass through.

**Path 3: Function.** The main wrapping logic:

1. `builtins.functionArgs module` extracts the full argument pattern.
2. **Den arg detection:** An arg `k` is a den arg if and only if `ctx ? ${k}`. No hardcoded allowlist -- fully dynamic based on what entities are in scope.
3. **Missing arg detection:** Args not in ctx are checked against `den.lib.schemaUtil.schemaArgKinds` (all schema-defined entity kind names, excluding `conf`, `aspect`, and `_`-prefixed internal keys). If a missing arg matches a schema kind AND has no default value (`!(allArgs.${k} or false)`), a `lib.warn` fires. This avoids false warnings on module-system args like `config` or `pkgs`.

### The unsatisfied guard

If any schema-matching args are missing without defaults, the function returns:

```nix
{ module = warnedModule; wrapped = false; unsatisfied = true; missingArgs = [...]; }
```

The caller (`wrapCollectedClasses`) checks `unsatisfied` and emits a trace message, skipping the module entirely. This prevents class modules from being delivered to the module system when their required den context was never provided -- typically a misconfiguration where an aspect expects an entity that no policy delivers.

### Partial vs full application

When den args are found:

- **Full application** (`remainingArgs == {}`): All args are den args. The module is called immediately: `warnedModule denArgs`. Result is a plain value, `wrapped = true`.
- **Partial application** (`remainingArgs != {}`): Some args are den args, others are module-system args. A wrapper function is created:

```nix
wrapper = moduleArgs: warnedModule (classWinsDen // moduleArgs // denWinsDen);
```

The merge order implements collision policy at call time:
- `classWinsDen` (den args with `class-wins` policy) goes first, then `moduleArgs` shadows them
- `denWinsDen` (den args with `den-wins` or `error` policy) goes last, shadowing `moduleArgs`

The wrapper is tagged with `lib.setFunctionArgs wrapper advertisedArgs` where `advertisedArgs` includes both remaining module-system args AND den arg names. Den args are advertised so NixOS passes thunks for them (which are then lazily shadowed by den values without forcing evaluation). Without advertising, NixOS would not pass them at all, preventing collision detection.

### Collision validator

When partial application occurs, a separate `validator` function is returned alongside the wrapped module. This validator:

- Advertises `remainingArgs // { config = true; }` (module-system args + config)
- At eval time, probes `moduleArgs.config._module.args` for each den arg name
- If a collision is detected, applies the resolved collision policy (error/warn)
- Returns `{ warnings = [...]; }` as a NixOS module

The validator is emitted as a separate module (not merged into the wrapper) so that collision detection happens through the module system's standard warning machinery.

## 3. Collision Policies

### Three-level resolution

`resolveCollisionPolicy { ctx, aspectPolicy, globalPolicy } name` returns the policy for a specific arg name. First match wins:

| Level | Source | Scope |
|-------|--------|-------|
| Aspect | `aspect.meta.collisionPolicy` | All class modules in that aspect (single enum, not per-field) |
| Entity | `ctx.${name}.collisionPolicy` | The specific entity value declares its own policy via schema |
| Global | `den.config.classModuleCollisionPolicy` | Entire flake, default `"error"` |

### Policy values

- **`"error"`** (default): Throw on collision. Forces the user to resolve the ambiguity.
- **`"class-wins"`**: Module-system value wins. Den value is dropped. Warning emitted.
- **`"den-wins"`**: Den value wins. Module-system value is shadowed. Warning emitted.

### pipelineOnly semantics

**File:** `nix/lib/policy-effects.nix`

`pipelineOnly` is a convenience wrapper that tags a value with `collisionPolicy = "class-wins"`:

```nix
pipelineOnly = value:
  if builtins.isAttrs value then
    value // { collisionPolicy = "class-wins"; }
  else
    { __functor = _: value; collisionPolicy = "class-wins"; };
```

This is used by batteries like `den.provides.flake-scope` that inject `lib`, `inputs`, and `den` into the pipeline context. These values are useful for parametric aspect resolution but should silently yield to the module system's own `lib`/`pkgs` when they reach class modules. The `collisionPolicy` attribute travels with the entity value through `__ctx` and is read by `resolveCollisionPolicy` at the entity level.

Non-attrset values are wrapped in a `__functor` attrset to carry the `collisionPolicy` attribute alongside the callable value.

## 4. Deferred-import wrapping

**Function:** `wrapDeferredImports` in `class-module.nix`

Class modules may arrive as `{ imports = [fn]; }` -- an attrset wrapping one or more module functions in an imports list. This pattern appears with deferred imports and `aspectContentType` unwrapping.

`wrapDeferredImports` recursively descends the imports tree:

1. If an element is a **function**: wrap it via `wrapClassModule` (recursive call with same ctx/policies).
2. If an element is an **attrset with `imports`**: recurse into its imports list.
3. Otherwise: pass through unchanged.

Returns `{ wrapped = bool; imports = [...]; }` where `wrapped` is true if any inner function was wrapped. The parent `wrapClassModule` call uses this to decide whether to attach a collision validator.

## 5. Post-pipeline wrapping via wrapCollectedClasses

**File:** `nix/lib/aspects/fx/wrap-classes.nix`

Class module wrapping is a **two-phase** process:

**Phase 1 (emit time):** `emitClasses` in `aspect.nix` sends raw `emit-class` effects with `__rawEntry = true`. The module, ctx, and policy args are captured but wrapping is deferred.

**Phase 2 (post-pipeline):** After the pipeline completes, `fxResolve` in `pipeline.nix` calls `wrapCollectedClasses` per scope:

```nix
wrappedPerScope = lib.mapAttrs (scopeId: scopeClasses:
  let scopeCtx = scopeContexts.${scopeId} or ctx;
  in wrapCollectedClasses scopeCtx scopeClasses
) scopedClassImportsRaw;
```

Each scope gets its own enriched context from `scopeContexts` -- a state field populated by `resolve-schema-entity` and `policy-dispatch` as entities are resolved. This means a class module emitted in a host scope gets host-specific context (with `isNixos`, `isDarwin`, etc.), while the same module in a user scope gets user-specific context.

### wrapCollectedClasses processing per entry

For each entry in the class imports:

1. **Skip non-raw entries:** Legacy or already-wrapped entries pass through unchanged.
2. **Merge enrichment:** `mergeEnrichment` adds scope-context keys that aren't already in the entry's emit-time ctx. This preserves entity bindings from the original scope while adding enrichment args.
3. **Wrap:** Calls `wrapClassModule` with the enriched ctx.
4. **Strip enrichment args:** `stripEnrichmentArgs` removes enrichment-only keys from the module's advertised `functionArgs`. Without this, NixOS would probe `_module.args` for keys that don't exist and crash.
5. **Compute identity:** `computeModuleIdentity` determines whether the module is anonymous and computes a final identity. Context-dependent modules keep the full identity (with ctxId); non-context-dependent modules strip the ctxId suffix.
6. **Wrap for module system:** `wrapModule` applies location and key-based dedup:
   - Anonymous modules: `lib.setDefaultModuleLocation` (location only, no key dedup)
   - Named modules: `{ key = loc; _file = loc; imports = [module]; }` (key enables NixOS dedup)
7. **Attach validator:** If `wrapClassModule` returned a `validator`, `buildValidatorModule` wraps it with `setFunctionArgs` and `setDefaultModuleLocation` at `${class}@${identity}/<collision-validator>`.
8. **Unsatisfied check:** If `result.unsatisfied` is true, the module is skipped with a trace message.

## 6. emit-class handler

**File:** `nix/lib/aspects/fx/handlers/tree.nix`, `classCollectorHandler`

The `emit-class` handler runs during pipeline evaluation (Phase 1) and collects class modules into scope-partitioned state.

### Scope-partitioned collection

State field `scopedClassImports` is an attrset keyed by scope ID, each containing an attrset keyed by class name, each containing a list of module entries:

```
scopedClassImports.${scopeId}.${className} = [ entry1, entry2, ... ]
```

The current scope comes from `state.currentScope`, set by the entity resolver when entering a new scope.

### scopedEmittedLocs dedup

State field `scopedEmittedLocs` tracks which `loc` strings have been emitted per scope. Before adding a module:

```nix
loc = "${param.class}@${baseIdentity}";
alreadyEmitted = scopeLocs ? ${loc};
```

If `alreadyEmitted` is true, the handler returns `state` unchanged -- the duplicate is silently dropped.

### Module identity/location computation

**Raw entries** (`__rawEntry = true`, the current standard path): The handler stores the full `param` attrset plus `__loc` for dedup. Identity computation is deferred to post-pipeline wrapping (`wrapCollectedClasses`).

For raw entries, `baseIdentity` always equals `nodeIdentity` (full identity including ctxId). This is conservative -- post-pipeline wrapping may relax it if the module turns out not to be context-dependent, but the collector cannot know this at emit time.

**Legacy entries** (non-raw, preserved for backward compatibility): The handler constructs the module location wrapper inline:
- Anonymous modules: `lib.setDefaultModuleLocation loc param.module`
- Named modules: `{ key = loc; _file = loc; imports = [param.module]; }`

### Thunk wrapping

All growing state fields (`scopedClassImports`, `scopedEmittedLocs`, etc.) are wrapped as thunks (`_: value`) so the trampoline's `deepSeq` doesn't re-materialize accumulated lists at every step. Access requires `(state.field or (_: default)) null`.

## 7. emitClasses in aspect.nix

**Function:** `emitClasses aspect classKeys nodeIdentity`

Called from `compileStatic` for each aspect node. For every class key:

1. Unwraps `aspectContentType` wrappers via `contentUtil.unwrapContentValuesList` to recover raw module values. Lists are processed per-element; bare values become singletons.
2. For multi-element lists, appends `[${idx}]` to the identity for per-element dedup.
3. Sends `emit-class` effect with:
   - `class`: the class key name (e.g., `"nixos"`, `"homeManager"`)
   - `identity`: the aspect's node identity (path key)
   - `module`: the raw module value
   - `ctx`, `aspectPolicy`, `globalPolicy`: for deferred wrapping
   - `__rawEntry = true`: marks for post-pipeline wrapping
   - `isContextDependent`: computed from `__parametricResolvedArgs` (whether any resolved parametric args match ctx keys) and `meta.contextDependent`

## 8. Future: Trait args

Traits are not yet reimplemented on `feat/fx-pipeline`, but the design accommodates them. When traits land:

- Each scope will define a `_den.traits` option in the module system, populated from `scopedTraits` state
- Trait names will become additional den context args available via `__scopeHandlers`
- `wrapClassModule` will detect trait args via the same `ctx ? ${k}` check -- no special-casing needed
- A class module like `{ host, myTrait, config, ... }: ...` will have both `host` and `myTrait` pre-applied from the scope context
- The `_den.traits` thunk pattern (evaluated lazily within the module system) allows trait values to depend on other module system config without circular evaluation
- Collision policy applies identically: if a trait name collides with a module-system arg, the three-level resolution determines the outcome
