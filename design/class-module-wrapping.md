# Class Module Wrapping

**Branch:** `feat/fx-pipeline` (753/753 tests passing)
**Status:** Stable reference

## 1. Overview

Class module wrapping bridges den's pipeline context (entity bindings like `host`, `user`, `fleet`) and NixOS/darwin/homeManager module systems. `wrapClassModule` inspects a module's `functionArgs`, identifies which args correspond to den context keys, partially applies them, and hands the module system a function that only expects module-system args (`config`, `pkgs`, etc.). Collision policies govern what happens when a den arg name overlaps with a module-system arg.

## 2. Module Forms

`wrapClassModule` dispatches on the shape of its `module` argument:

| Form | Example | Handling |
|------|---------|----------|
| **Flat function** | `{ host, config, pkgs, ... }: { ... }` | `wrapFunctionModule` -- den args detected and partially applied |
| **Curried / two-layer** | `{ host }: { config, ... }: { ... }` | First call is full application (all args are den args), returns inner function unchanged |
| **Functor passthrough** | Non-function attrset without `imports` | Returned unchanged, `wrapped = false` |
| **Full application** | All args are den args, no module-system args | Called immediately with den args, result is a plain value |
| **Attrset with imports** | `{ imports = [ fn ]; }` | `wrapImportsModule` -- recurses into imports via `wrapDeferredImports` |

Dispatch logic in `wrapClassModule`:
1. If `builtins.isAttrs module && module ? imports` -- attrset-with-imports path
2. If `!builtins.isFunction module` -- non-function passthrough
3. Otherwise -- function wrapping path

## 3. Den Arg Detection

Detection uses `builtins.functionArgs` introspection against the scope context:

```nix
allArgs = builtins.functionArgs module;
argNames = builtins.attrNames allArgs;
denArgNames = builtins.filter (k: ctx ? ${k}) argNames;
```

An arg `k` is a den arg if and only if `ctx ? ${k}`. No hardcoded allowlist -- fully dynamic based on what entities are in scope. The `ctx` attrset is reconstructed at emit time from scope handlers.

**Missing arg detection:** Args not in ctx are checked against `den.lib.schemaUtil.schemaArgKinds` (all schema-defined entity kind names). If a missing arg matches a schema kind AND has no default value (`!(allArgs.${k} or false)`), a `lib.warn` fires. Module-system args like `config` or `pkgs` never match schema kinds, avoiding false warnings.

## 4. Partial Application

Two cases based on whether non-den args remain:

**Full application** (`effectiveRemainingArgs == {}`): Every arg is a den arg. The module is called immediately:

```nix
{ module = warnedModule denArgs; wrapped = true; }
```

**Wrapper case** (`effectiveRemainingArgs != {}`): A wrapper function is created that merges den args with module-system args at call time:

```nix
wrapper = moduleArgs: warnedModule (classWinsDen // moduleArgs // denWinsDen);
```

The merge order implements collision policy:
- `classWinsDen` (den args with `class-wins` policy) goes first, then `moduleArgs` shadows them
- `denWinsDen` (den args with `den-wins` or `error` policy) goes last, shadowing `moduleArgs`

The wrapper is tagged with `lib.setFunctionArgs wrapper advertisedArgs` where `advertisedArgs` includes both remaining module-system args AND den arg names. Den args are advertised so NixOS passes thunks for them via `_module.args`, enabling collision detection without forcing evaluation.

**Config thunk handling:** When pipe args contain `__configThunk` markers, the wrapper resolves them using the `evalModules` fixpoint config. This breaks the circular dependency between `assemblePipes` and the module system. The presence of config thunks forces `config` into `effectiveRemainingArgs`, ensuring the wrapper path is taken even if no other module-system args exist.

## 5. Collision Detection

### Three policies at three levels

`resolveCollisionPolicy { ctx, aspectPolicy, globalPolicy } name` returns the policy for a specific arg name. First match wins:

| Level | Source | Scope |
|-------|--------|-------|
| Aspect | `aspect.meta.collisionPolicy` | All class modules in that aspect (single enum) |
| Entity instance | `ctx.${name}.collisionPolicy` | Per-entity instance |
| Entity schema | `ctx.__collisionPolicies.${name}` | Schema-level, from `den.schema.<kind>.collisionPolicy` |
| Global | `den.config.classModuleCollisionPolicy` | Entire flake, default `"error"` |

The schema-level policy is captured eagerly in `resolveEntity` and stored in `ctx.__collisionPolicies` to avoid circular evaluation through the module system.

### Policy values

- **`"error"`** (default): Throw on collision.
- **`"class-wins"`**: Module-system value wins. Den value dropped. Warning emitted.
- **`"den-wins"`**: Den value wins. Module-system value shadowed. Warning emitted.

### mkCollisionValidator

When partial application occurs, a separate `validator` module is returned alongside the wrapped module:

```nix
mkCollisionValidator = policy: denArgNames: moduleArgs:
```

At eval time, the validator probes `moduleArgs.config._module.args` for each den arg name using `builtins.tryEval` + `builtins.seq` to safely detect presence without forcing thunks. If a collision is found, it applies the resolved policy and returns `{ warnings = [...]; }` as a NixOS module.

The validator is emitted as a separate module so collision detection flows through the module system's standard warning machinery.

## 6. Enrichment & pipelineOnly

### Pipeline-injected args

Batteries like `den.provides.flake-scope` inject `lib`, `inputs`, and `den` into pipeline context. These are useful for parametric aspect resolution but should yield to the module system's own values in class modules.

### pipelineOnly semantics

`pipelineOnly` tags a value with `collisionPolicy = "class-wins"`:

```nix
pipelineOnly = value:
  if builtins.isAttrs value then
    value // { collisionPolicy = "class-wins"; }
  else
    { __functor = _: value; collisionPolicy = "class-wins"; };
```

The `collisionPolicy` attribute travels with the entity value through scope context and is read by `resolveCollisionPolicy` at the entity instance level.

### stripEnrichmentArgs

**File:** `nix/lib/aspects/fx/wrap-classes.nix`

After wrapping, `stripEnrichmentArgs` removes enrichment-only keys from the module's advertised `functionArgs`. Without this, NixOS probes `_module.args.${name}` for every advertised arg and crashes when the key doesn't exist.

Two modes:
- **Wrapped modules:** Strip enrichment-only keys (those injected by den scope context but not in the wrapper's `advertisedArgs`)
- **Unwrapped modules:** Strip args with defaults that aren't in ctx (unknown to both den and NixOS)

Keys listed in the wrapper's `advertisedArgs` are preserved -- stripping them would prevent collision detection.

## 7. Deferred Imports

**Function:** `wrapDeferredImports` in `class-module.nix`

Class modules may arrive as `{ imports = [fn]; }`. `wrapDeferredImports` recursively descends the imports tree:

1. **Function element:** Wrap via `wrapClassModule` (recursive call with same ctx/policies)
2. **Attrset with `imports`:** Recurse into its imports list
3. **Otherwise:** Pass through unchanged

Returns `{ wrapped = bool; imports = [...]; }` where `wrapped` is true if any inner function was wrapped. The parent call uses this to decide whether to attach a collision validator.

## 8. Unsatisfied Guard

If any schema-matching args are missing without defaults, `wrapFunctionModule` returns:

```nix
{ module = warnedModule; wrapped = false; unsatisfied = true; missingArgs = [...]; }
```

The caller (`processEntry` in `wrapCollectedClasses`) checks `result.unsatisfied` and returns an empty list `[]`, silently dropping the module. This prevents class modules from reaching the module system when their required den context was never provided -- typically a misconfiguration where an aspect expects an entity kind that no policy delivers.

## 9. Post-Pipeline Wrapping

**Function:** `wrapCollectedClasses` in `nix/lib/aspects/fx/wrap-classes.nix`

After the pipeline completes, `fxResolve` calls `wrapCollectedClasses` per scope with scope-specific enriched context:

```nix
wrappedPerScope = lib.mapAttrs (scopeId: scopeClasses:
  let scopeCtx = scopeContexts.${scopeId} or ctx;
  in wrapCollectedClasses scopeCtx scopeClasses
) scopedClassImportsRaw;
```

### processEntry pipeline

For each raw entry (`__rawEntry = true`):

1. **Pipe targeting:** `applyPipeTargeting` applies per-aspect pipe overrides from `ctx.__pipeTargeted`
2. **Merge enrichment:** `mergeEnrichment` adds scope-context keys not already in the entry's emit-time ctx, preserving entity bindings from the original scope
3. **Wrap:** Calls `wrapClassModule` with enriched ctx
4. **Strip enrichment args:** Removes enrichment-only keys from advertised `functionArgs`
5. **Compute identity:** Determines anonymous vs named, strips ctxId suffix for non-context-dependent modules
6. **Module system wrapping:** Anonymous modules get `lib.setDefaultModuleLocation`; named modules get `{ key = loc; _file = loc; imports = [module]; }` for NixOS key-based dedup
7. **Validator attachment:** If `wrapClassModule` returned a `validator`, it's wrapped as a separate module at `${class}@${identity}/<collision-validator>`
8. **Unsatisfied check:** If `result.unsatisfied`, returns `[]` (module dropped)

### Cross-scope dedup

Module identity keys follow the pattern `${class}@${identity}`. Named modules use NixOS key-based dedup: if the same key appears in multiple scopes, the module system merges them. Context-dependent modules retain their full identity (with ctxId suffix), so different scope contexts produce distinct keys.

## 10. Invariants

- **Den args stripped from advertised signature after wrapping.** `stripEnrichmentArgs` ensures the final module only advertises args the module system can satisfy.
- **Collision policy cascade order.** Aspect > entity instance > entity schema > global. First match wins, no fallthrough.
- **Unsatisfied = silent drop.** Modules with missing required den args are dropped with a `lib.warn`, never delivered to the module system.
- **Enrichment keys never overwrite entity bindings.** `mergeEnrichment` only adds keys not already present in the entry's emit-time ctx.
- **Config thunks force wrapper path.** Pipe args with `__configThunk` markers add `config` to remaining args, ensuring the module goes through partial application even if all other args are den args.

## 11. Key Files

| File | Contents |
|------|----------|
| `nix/lib/aspects/fx/class-module.nix` | `wrapClassModule`, `wrapFunctionModule`, `wrapImportsModule`, `wrapDeferredImports`, `mkCollisionValidator`, `resolveCollisionPolicy` |
| `nix/lib/aspects/fx/wrap-classes.nix` | `wrapCollectedClasses`, `processEntry`, `stripEnrichmentArgs`, `mergeEnrichment`, `computeModuleIdentity`, `wrapModule`, `buildValidatorModule` |
| `nix/lib/aspects/fx/aspect.nix` | `emitClasses` -- emits raw `emit-class` effects during aspect compilation |
| `nix/lib/aspects/fx/handlers/class-collector.nix` | `emit-class` handler -- scope-partitioned collection with dedup |

**Cross-references:**
- [aspect-compilation.md](aspect-compilation.md) -- when wrapping occurs in the compilation pipeline
- [scope-partitioning.md](scope-partitioning.md) -- per-scope context construction and scope handler propagation
