# Unified Effects Pipeline — Design Spec

**Date:** 2026-04-14
**Branch:** `feat/fx-resolution`
**Source:** Vic's full PR review on sini/den#2 (58 comments)
**Depends on:** nix-effects effectful handlers (merged to vic branch)
**Status:** Draft

## Problem

The fx pipeline has three architectural issues identified in Vic's review:

1. **Module wiring is overcomplicated.** Modules pass explicit args (`identity`, `handlers`, `wrap`, `trace`, etc.) through import chains. An `init` function takes `fx` (nix-effects lib) and threads it everywhere. Long re-export lists duplicate the entire API surface. Every other den module uses `den.lib.*` for cross-module access — the fx modules are the exception.

2. **`resolveAspect` and `resolveOne` are still pipeline-aware.** They know about class config, constraint registration, chain tracking, and include folding. Vic's vision: aspects are a DSL that compiles to effects. An aspect `{ nixos = ...; includes = [...] }` should compile to a computation that emits `emit-class` per class and `emit-include` per child — nothing more. The compilation is `aspectToEffect`. All resolution strategy lives in handlers.

3. **Contexts are treated as a separate system.** `ctx-apply.nix` and `ctx-stage.nix` have specialized traversal machinery (`traverse`/`foldInto`/`foldContexts`/`buildStageIncludes`). But contexts are just aspects whose functor walks `into` transitions. They should compile through the same `aspectToEffect` pipeline, with `into` transitions handled by scoped sub-computations.

## Design

### 1. Module wiring: `{ lib, den }` only

Every fx module takes `{ lib, den, ... }` as its function args. No other parameters. Modules access dependencies via fully qualified paths:

```nix
# Inside any fx module:
inherit (den.lib.fx.identity) pathKey aspectPath;
inherit (den.lib.fx.constraints) exclude;
nxFx = den.lib.nxFx;  # nix-effects lib, set once
```

**nix-effects access:** `den.lib.nxFx` is set once at the top level (in `nix/lib/default.nix` or `nix/lib/aspects/default.nix`) from `inputs.nix-effects.lib`. Modules reference it lazily via `den.lib.nxFx`. Pre-merge fallback:

```nix
nxFx = inputs.nix-effects.lib or (throw "nix-effects required");
```

**`default.nix` becomes a simple barrel:**

```nix
{ lib, den, ... }: {
  identity = import ./identity.nix { inherit lib den; };
  constraints = import ./constraints.nix { inherit lib den; };
  includes = import ./includes.nix { inherit lib den; };
  trace = import ./trace.nix { inherit lib den; };
  handlers = import ./handlers { inherit lib den; };
  aspect = import ./aspect.nix { inherit lib den; };
  pipeline = import ./pipeline.nix { inherit lib den; };
  # No init function. No re-exports. Modules access den.lib.fx.* directly.
}
```

**What gets removed:**
- `init` function
- All explicit arg passing between modules (`identity`, `handlers`, `wrap`, `trace`, `resolveOne`, etc.)
- All flat re-export lists (`inherit (resolve) resolveOne resolveOneStrict ...`)
- `pureIdentity`, `pureConstraints`, `pureIncludes` — the "pure" variants. Since modules use `den.lib.fx.*` lazily, there's no need for fx-free copies. The constructors (`exclude`, `substitute`, etc.) don't use nix-effects at all — they produce plain attrsets.

### 2. `aspectToEffect` — the aspect compiler

Replaces `resolveAspect`, `resolveOne`, `wrapAspect`, `wrapIdentity`, `emitClassConfig`, `registerHandlers`. One function that compiles any aspect into an effectful computation.

**Input:** An aspect attrset — `{ name, meta, nixos, homeManager, includes, __functor, ... }`

**Output:** A `Computation` that, when handled, emits effects for everything the aspect declares.

```nix
aspectToEffect = aspect:
  let
    identity = den.lib.fx.identity;
    nxFx = den.lib.nxFx;
    nodeIdentity = identity.pathKey (identity.aspectPath aspect);

    # Structural keys that are NOT class configs
    structuralKeys = [ "name" "meta" "includes" "provides" "into"
                       "__functor" "__functionArgs" ];

    # Everything else is a class config
    classKeys = builtins.filter
      (k: !(builtins.elem k structuralKeys))
      (builtins.attrNames aspect);

    # Emit one effect per class config
    emitClasses = builtins.foldl' (acc: className:
      nxFx.bind acc (_:
        nxFx.send "emit-class" {
          class = className;
          module = aspect.${className};
          identity = nodeIdentity;
        })
    ) (nxFx.pure null) classKeys;

    # Emit constraints from meta.handleWith
    emitConstraints = ...;  # same pattern, send register-constraint per entry

    # Emit one effect per include
    emitIncludes = builtins.foldl' (acc: child:
      nxFx.bind acc (results:
        nxFx.bind (nxFx.send "emit-include" {
          from = nodeIdentity;
          include = child;
        }) (childResults:
          nxFx.pure (results ++ childResults)))
    ) (nxFx.pure []) (aspect.includes or []);
  in
  nxFx.bind emitClasses (_:
    nxFx.bind emitConstraints (_:
      emitIncludes));
```

**Functor handling:** When an aspect has `__functor`, it's a parametric aspect. The functor args become effect requests:

```nix
# For { host, ... }: { nixos = ...; }
nxFx.bind.fn {
  __functionArgs = lib.functionArgs aspect;
  __functor = _: args: aspectToEffect (aspect args);
}
```

`bind.fn` sends one effect per declared arg (`host`, `user`, etc.). Handlers provide the values. The result is fed to the functor, which produces a new aspect attrset, which is recursively compiled via `aspectToEffect`.

**Identity preservation:** `aspectToEffect` preserves `name` and `meta` from the input aspect. The computation carries the aspect's identity throughout — handlers can inspect it for constraint checking, tracing, etc.

### 3. `emit-include` handler — owns recursion

The handler for `emit-include` replaces `resolveIncludeHandler`:

```nix
"emit-include" = { param, state }:
  let
    child = param.include;
    childIdentity = identity.pathKey (identity.aspectPath child);
    isConditional = child.meta.conditional or false;
  in {
    resume =
      if isConditional then
        resolveConditionalComp child
      else
        nxFx.bind (nxFx.send "check-constraint" {
          identity = childIdentity;
          aspect = child;
        }) (decision:
          if decision.action == "exclude" then
            emitTombstone child decision
          else if decision.action == "substitute" then
            emitSubstitution child decision
          else
            nxFx.bind (aspectToEffect child) (result:
              nxFx.bind (nxFx.send "resolve-complete" result) (_:
                nxFx.pure [ result ])));
    inherit state;
  };
```

Recursion happens via `aspectToEffect child` in the effectful resume. The handler owns constraint checking, tombstoning, and the `resolve-complete` emission.

### 4. Contexts as aspects

A context IS an aspect. It has:
- A name, meta, includes, provides — like any aspect
- An `into` attrset — defines context transitions
- A functor that IS `ctxApply` — applying the context means walking transitions

`aspectToEffect` handles contexts the same way, with two additions:
- `.into` keys are excluded from class emission (they're transitions, not configs)
- An `into-transition` handler processes transitions into scoped sub-computations

**How `into` transitions work:**

When `aspectToEffect` encounters `aspect.into`, it emits an `into-transition` effect per transition key:

```nix
emitTransitions = builtins.foldl' (acc: key:
  nxFx.bind acc (_:
    nxFx.send "into-transition" {
      inherit key;
      transitionFn = aspect.into.${key};
      ctx = currentCtx;
      self = aspect;
    })
) (nxFx.pure null) (builtins.attrNames (aspect.into or {}));
```

The `into-transition` handler:

```nix
"into-transition" = { param, state }:
  let
    # Apply transition: { host } → [{ host, user: alice }, { host, user: bob }]
    newContexts = param.transitionFn param.ctx;
    targetCtx = den.ctx.${param.key};  # e.g., den.ctx.user
  in {
    resume = builtins.foldl' (acc: newCtx:
      nxFx.bind acc (results:
        # Install scoped handlers for the new context values
        nxFx.bind (nxFx.scope.run {
          handlers = constantHandler newCtx;
        } (aspectToEffect targetCtx)) (childResults:
          nxFx.pure (results ++ childResults)))
    ) (nxFx.pure []) newContexts;
    inherit state;
  };
```

Each new context value becomes a scoped handler — `user = alice` is local to that sub-computation. `den.ctx.user` runs inside the scope and sees its local `user` via the `constantHandler`.

**Self-provide auto-include:** If `aspect.provides.${aspect.name}` exists, it's automatically included — the aspect provides its own sub-aspects to itself. This replaces the explicit `selfProv` logic in `buildStageIncludes`.

### 5. Handler simplification

| Current | New | Reason |
|---|---|---|
| `parametricHandler` | `constantHandler` | Name reflects what it does: for each attr, resume with the value |
| `staticHandler` | removed | Same as `constantHandler { class; aspect-chain; }` |
| `contextHandlers` | removed | Same as `constantHandler (ctx // { class; aspect-chain; })` |
| `missingArgError` | inlined | Single call site in `resolveOneStrict` (which itself may be removed) |
| `provide-class` effect | `emit-class` effect | Aspect emits, handler collects |
| `resolve-include` effect | `emit-include` effect | Aspect emits, handler recurses |
| `resolve-complete` effect | `resolve-complete` (kept) | Handler emits after child resolution |

### 6. What gets removed

| File/function | Reason |
|---|---|
| `ctx-apply.nix` | Contexts go through `aspectToEffect` |
| `ctx-stage.nix` | `buildStageIncludes`/`emitProviders` replaced by `aspectToEffect` + handlers |
| `resolve-deep.nix` | `resolveAspect`/`foldIncludes` replaced by `aspectToEffect` |
| `resolve-handler.nix` | `resolveIncludeHandler` replaced by `emit-include` handler |
| `resolve-one.nix` | `resolveOne`/`resolveOneStrict` replaced by `aspectToEffect` |
| `wrap.nix` | `wrapIdentity` replaced by `aspectToEffect` identity preservation |
| `resolve-legacy.nix` | `resolveDeep` — deprecated, remove or keep for test compat |
| `handlers/ctx.nix` | Most handlers removed; `constantHandler` + `ctxSeenHandler` stay |
| `init` function | Modules use `den.lib.fx.*` |

### 7. What stays

| File/function | Reason |
|---|---|
| `identity.nix` | Pure path/identity utilities — unchanged |
| `constraints.nix` | Constraint constructors — unchanged |
| `includes.nix` | `includeIf` — unchanged |
| `trace.nix` | Trace handlers — consume `resolve-complete`, unchanged |
| `handlers/tree.nix` | `constraintRegistryHandler`, `chainHandler`, `provideClassHandler` — renamed `emit-class` handler |
| `pipeline.nix` | `mkPipeline`, `composeHandlers`, `defaultHandlers` — simplified |

### 8. Effect protocol (final)

```
Context:     into-transition          — handler walks transitions with scoped handlers
             ctx-seen                 — dedup handler for context stages

Aspect:      emit-class { class, module, identity }  — handler accumulates modules
             emit-include { from, include }          — handler checks constraints + recurses
             register-constraint { type, scope, ... } — handler stores in registry
             check-constraint { identity, aspect }    — handler checks registry

Tree:        chain-push { identity }  — handler tracks includes chain
             chain-pop                — handler pops includes chain
             resolve-complete         — handler emits trace entries
             get-path-set            — handler returns accumulated paths

Parametric:  <arg-name>              — constantHandler resumes with value
```

### 9. Pipeline flow

```
aspectToEffect(rootAspect)
  → emit-class for each class key
  → register-constraint for meta.handleWith
  → emit-include for each child
    → handler checks constraint
    → handler recurses: aspectToEffect(child)
      → (child may have into transitions)
      → into-transition handler installs scoped handlers
      → aspectToEffect(targetCtx) runs in scope
  → resolve-complete per child (from handler)

fx.handle { handlers = defaultHandlers; } computation
  → { imports = [...]; state = { entries, paths, ... }; }
```

### 10. Migration path

1. Simplify module wiring — `{ lib, den }` only, remove init/re-exports
2. Implement `aspectToEffect` alongside existing `resolveAspect`
3. Implement `emit-include` handler alongside existing `resolveIncludeHandler`
4. Implement `into-transition` handler
5. Switch `mkPipeline` to use `aspectToEffect`
6. Remove old code (`ctx-apply`, `ctx-stage`, `resolve-deep`, `resolve-handler`, `resolve-one`, `wrap`)
7. Update all tests
8. Update diag library callers

### 11. Verification

```
nix develop -c just ci ""
nix eval --json --override-input den path:. path:./templates/diag-fx-demo#debug
```

All tests must pass. Parent parity with legacy trace must hold.
