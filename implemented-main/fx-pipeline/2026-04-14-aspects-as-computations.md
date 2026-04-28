# Aspects as Computations — Design Spec

**Date:** 2026-04-14
**Branch:** `feat/fx-resolution`
**Depends on:** `sini/nix-effects#feat/effectful-handlers`
**Source:** Vic's architectural review on sini/den#2 + docs/design/2026-04-14-fx-v2-architectural-direction.md items 1, 2, 6, 7
**Status:** Draft

## Problem

The current fx pipeline uses effects as a layer ON TOP of explicit tree walking. `resolveDeepEffectful` manually recurses via `go → resolveChild → resolveChildren → go`, with constraint checking and tracing interleaved in the recursion. The tree structure is walked, not composed.

Vic's v2 vision: aspects ARE effectful computations. An aspect's classes are `provide-class` effects. Its includes are `resolve-include` effects. The tree structure emerges from effect composition. Resolution strategy (constraints, recursion, tracing) lives entirely in handlers.

With effectful handlers now available in nix-effects (handlers can return computations as resume values), the handler can own the recursion — receiving a `resolve-include` effect, checking constraints, and recursively resolving the child via effectful resume.

## Design

### Core model

An aspect is a mini-DSL that compiles to three effect types:

| Aspect declaration | Effect sent | Handler responsibility |
|---|---|---|
| `nixos = { ... }` | `provide-class { class; module; identity; }` (one per resolved node) | Accumulate modules, dedup by identity |
| `meta.handleWith = exclude foo` | `register-constraint { type; scope; ... }` | Store in constraint registry |
| `includes = [ child1 child2 ]` | `resolve-include child` (per child) | Check constraints, recurse, emit resolve-complete |

The aspect computation doesn't know about constraints, tracing, or chain tracking. It declares what it provides and what it includes. Handlers decide what happens.

### `resolveAspect` — replaces `go`, `resolveChild`, `resolveChildren`, `resolveConditional`

```nix
resolveAspect = aspectVal:
  fx.bind (resolveOne aspectVal) (resolved:
    let
      nodeIdentity = identity.pathKey (identity.aspectPath resolved);
      rawName = resolved.name or "<anon>";
      isMeaningful = rawName != "<anon>" && rawName != "<function body>"
        && !(lib.hasPrefix "[definition " rawName);
      classes = classesOf resolved;
      handlers = resolved.meta.handleWith or null;
      excludes = resolved.meta.excludes or [];
      # Apply ctx-stage tag propagation before sending includes.
      # tagChild inherits __ctxStage/__ctxKind/__ctxAspect from parent
      # to children that don't have their own. Same as current go.
      includes = map tagChild (resolved.includes or []);
    in
    fx.bind (emitClasses classes nodeIdentity) (_:
      fx.bind (emitConstraints handlers excludes) (_:
        if isMeaningful then
          fx.bind (fx.send "chain-push" { identity = nodeIdentity; }) (_:
            fx.bind (foldIncludes includes) (children:
              fx.bind (fx.send "chain-pop" null) (_:
                fx.pure (resolved // { includes = children; })
              )))
        else
          fx.bind (foldIncludes includes) (children:
            fx.pure (resolved // { includes = children; })
          )
      )
    )
  );
```

Where `emitClasses` sends `provide-class` for each class key, and `emitConstraints` sends `register-constraint` for each handler/exclude entry.

### `foldIncludes` — sends `resolve-include` per child

```nix
foldIncludes = includes:
  builtins.foldl' (acc: child:
    fx.bind acc (results:
      fx.bind (fx.send "resolve-include" child) (childResults:
        fx.pure (results ++ childResults)
      )
    )
  ) (fx.pure []) includes;
```

The computation sends `resolve-include` for each child and collects the results. `childResults` is a list — the handler returns `[resolved]` for kept children, `[tombstone]` for excluded, `[tombstone, replacement]` for substituted. `foldIncludes` flattens, matching current `resolveChildren` behavior.

### `resolveIncludeHandler` — effectful handler owning recursion

This handler intercepts `resolve-include` effects and returns an effectful resume:

```nix
resolveIncludeHandler = resolveAspect: {
  "resolve-include" = { param, state }:
    let
      child = wrapChild param;
      childIdentity = identity.pathKey (identity.aspectPath child);
      isConditional = builtins.isAttrs child && (child.meta.conditional or false);
    in {
      # Handler returns a list: [tombstone] | [tombstone, replacement] | [resolved]
      # This matches current resolveChild behavior where substitutes produce two entries.
      resume =
        if isConditional then
          resolveConditionalComp child
        else
          fx.bind (fx.send "check-constraint" { identity = childIdentity; aspect = child; }) (
            decision:
            if decision.action == "exclude" then
              let ts = identity.tombstone child { excludedFrom = decision.owner; }; in
              fx.bind (fx.send "resolve-complete" ts) (_: fx.pure [ ts ])
            else if decision.action == "substitute" then
              let
                ts = identity.tombstone child {
                  excludedFrom = decision.owner;
                  replacedBy = decision.replacement.name or "<anon>";
                };
              in
              fx.bind (fx.send "resolve-complete" ts) (_:
                fx.bind (resolveAspect decision.replacement) (resolved:
                  fx.bind (fx.send "resolve-complete" resolved) (_:
                    fx.pure [ ts resolved ]
                  )
                )
              )
            else
              fx.bind (resolveAspect child) (resolved:
                fx.bind (fx.send "resolve-complete" resolved) (_:
                  fx.pure [ resolved ]
                )
              )
          );
      inherit state;
    };
};
```

Key points:
- The handler receives the raw child and wraps it (`wrapChild`)
- It checks constraints via `check-constraint` effect (handled by `constraintRegistryHandler`)
- For kept children, it recursively calls `resolveAspect` — this is the effectful resume
- `resolve-complete` is emitted by the handler, not the computation — it's part of resolution strategy
- Conditional includes (`includeIf`) are handled inline by `resolveConditionalComp`

### `resolveConditionalComp` — replaces `resolveConditional`

```nix
resolveConditionalComp = condNode:
  fx.bind (fx.send "get-path-set" null) (pathSet:
    let
      guardCtx = { hasAspect = ref: pathSet ? ${identity.pathKey (identity.aspectPath ref)}; };
      pass = condNode.meta.guard guardCtx;
    in
    if pass then
      foldIncludes condNode.meta.aspects
    else
      builtins.foldl' (acc: a:
        fx.bind acc (results:
          let ts = identity.tombstone a { guardFailed = true; }; in
          fx.bind (fx.send "resolve-complete" ts) (_:
            fx.pure (results ++ [ ts ])
          )
        )
      ) (fx.pure []) condNode.meta.aspects
  );
```

When the guard passes, it calls `foldIncludes` — which sends `resolve-include` for each guarded aspect, hitting the handler for constraint checking and recursion.

### `resolveDeepEffectful` simplification

Currently `resolveDeepEffectful` contains `go`, `resolveChild`, `resolveChildren`, `resolveConditional`, `wrapChild`, `tagChild`. It becomes:

```nix
resolveDeepEffectful = { ctx, class, aspect-chain ? [] }:
  let
    wrapChild = child: ...;  # existing bare-function wrapper
    tagChild = ...;  # existing ctx-stage tag propagation

    resolveOne' = resolveOne { inherit ctx class; aspect-chain = []; };

    resolveAspect = aspectVal:
      fx.bind (resolveOne' aspectVal) (resolved:
        ...  # as above, with tagChild applied to includes
      );

    foldIncludes = ...;  # as above

    resolveConditionalComp = ...;  # as above

    # Effectful handler — closes over resolveAspect for recursion
    resolveIncludeHandler' = resolveIncludeHandler resolveAspect;
  in
  {
    inherit resolveAspect resolveConditionalComp foldIncludes;
    resolveIncludeHandler = resolveIncludeHandler';
  };
```

Returns an attrset with `resolveAspect` (the computation constructor) and `resolveIncludeHandler` (the effectful handler that recurses via `resolveAspect`). Consumers compose the handler into their handler set. No `go`, no `resolveChild`, no `resolveChildren`.

### `defaultHandlers` update

`defaultHandlers` stays pure — no recursion reference. The `resolve-include` passthrough is removed:

```nix
defaultHandlers = { class, ctx }:
  handlers.parametricHandler ctx
  // handlers.staticHandler { inherit class; aspect-chain = []; }
  // handlers.ctxTraverseHandler
  // handlers.ctxSeenHandler
  // handlers.ctxProviderHandler
  // handlers.ctxEmitHandler
  // handlers.provideClassHandler
  // handlers.constraintRegistryHandler
  // handlers.chainHandler
  // identity.pathSetHandler
  // {
    "resolve-complete" = { param, state }: {
      resume = param;
      state = state // {
        paths = (state.paths or []) ++ (lib.optional (!(param.meta.excluded or false)) (identity.aspectPath param));
      };
    };
  };
```

### `mkPipeline` update — composes the recursive handler at the edge

`mkPipeline` is where `resolveIncludeHandler` is wired in. This is the only place that connects the handler to `resolveAspect`, keeping the dependency explicit:

```nix
mkPipeline = { extraHandlers ? {}, extraState ? {}, class }: { ctxNs, self, ctx }:
  let
    resolve = resolveDeepEffectful { inherit ctx class; };
    # resolve is { resolveAspect, resolveIncludeHandler, ... }
    comp = fx.bind (ctxApply.ctxApplyEffectful ctxNs self ctx) (includes:
      fx.bind (resolve.resolveAspect {
        name = self.name or "<anon>";
        meta = self.meta or {};
        inherit includes;
      }) (resolved:
        fx.bind (fx.send "resolve-complete" resolved) (_: fx.pure resolved)
      )
    );
  in
  fx.handle {
    handlers = composeHandlers
      (defaultHandlers { inherit class ctx; }
       // resolve.resolveIncludeHandler)  # Effectful handler wired at the edge
      extraHandlers;
    state = defaultState // extraState;
  } comp;
```

`resolveDeepEffectful` returns both `resolveAspect` and `resolveIncludeHandler` (which closes over `resolveAspect`). `mkPipeline` composes the handler into the handler set at the pipeline edge. Tests that call `resolveDeepEffectful` directly can access the handler the same way.

### Test strategy for resolve-include handler

`resolveDeepEffectful` exports `resolveIncludeHandler`. Tests that provide custom handler sets compose it:

```nix
resolve = fxLib.resolve.resolveDeepEffectful { ctx = {}; class = "nixos"; };
result = fx.handle {
  handlers = {
    "resolve-complete" = ...;
    "provide-class" = ...;
  }
  // fxLib.handlers.constraintRegistryHandler
  // fxLib.handlers.chainHandler
  // resolve.resolveIncludeHandler;  # Recursive handler from resolve
  state = { ... };
} (resolve.resolveAspect parent);
```

### What about the root's `resolve-complete`?

Currently `mkPipeline` emits `resolve-complete` for the root after `resolveDeepEffectful` returns. In the new model, the root aspect goes through `resolveAspect`, but `resolveAspect` doesn't emit `resolve-complete` for itself — the handler does that for children. The root has no parent handler.

Options:
- A) Emit `resolve-complete` for root at the end of `mkPipeline` (like now)
- B) Have `resolveAspect` always emit `resolve-complete` for itself at the end
- C) The root is included by a synthetic parent, so the handler covers it

A is simplest and matches current behavior. The root is special — it has no parent sending `resolve-include`.

### nix-effects dependency update

Update `templates/ci/flake.nix` (and `templates/diag-fx-demo/flake.nix`) to point to the fork with effectful handlers:

```nix
inputs.nix-effects.url = "github:sini/nix-effects/feat/effectful-handlers";
```

### What changes

| Current | New |
|---|---|
| `go` function in `resolveDeepEffectful` | `resolveAspect` (top-level, not nested) |
| `resolveChild` (constraint check + recurse) | `resolveIncludeHandler` (effectful handler) |
| `resolveChildren` (foldl over children) | `foldIncludes` (sends `resolve-include` per child) |
| `resolveConditional` (guard + tombstone) | `resolveConditionalComp` (called by handler) |
| `resolve-include` effect = passthrough | `resolve-include` effect = effectful recursive handler |
| `resolve-complete` emitted in `resolveChild` | `resolve-complete` emitted in `resolveIncludeHandler` |
| Tree walked by explicit recursion | Tree composed via effect composition + effectful resume |

### What stays unchanged

- `resolveOne` / `resolveOneStrict` — still translate functor to resolved attrset
- `wrapIdentity` / `withIdentity` — still wrap identity envelope
- `wrapChild` — still wraps bare functions in envelope
- `tagChild` — still propagates ctx stage tags
- `chainHandler` — still manages includesChain state
- `constraintRegistryHandler` — still manages constraint registry
- `provideClassHandler` — still accumulates modules
- All trace handlers — still passive observers on `resolve-complete`
- `ctxApplyEffectful` — still produces the root includes list
- `fxResolve` / `fxFullResolve` — API unchanged, implementation delegates to `mkPipeline`

### ctx-apply computation chains (item 7)

`buildStageIncludes` currently returns `fx.pure ([main] ++ selfProv ++ crossProv)` — a computation that resolves to a list. In the new model, this naturally becomes part of the aspect computation: each provider result goes through `resolveAspect`, which sends `resolve-include` for its children.

No explicit change needed — the provider results become includes of the root aspect, and `foldIncludes` handles them via `resolve-include` effects. The "computation chain" Vic described emerges from the effect composition.

### Testing strategy

1. All existing tests must pass — the behavior is identical, only the internal structure changes
2. New test: verify effectful resume works end-to-end (handler recursively resolves children)
3. New test: verify constraint checking happens in the handler (not the computation)
4. Parent parity test via `fx-debug.nix` must still show perfect match with legacy

### Migration path

1. Update nix-effects input to `sini/nix-effects#feat/effectful-handlers`
2. Implement `resolveAspect`, `foldIncludes`, `resolveConditionalComp`
3. Implement `resolveIncludeHandler` as effectful handler
4. Update `defaultHandlers` to include `resolveIncludeHandler`
5. Simplify `resolveDeepEffectful` to return `resolveAspect`
6. Remove `go`, `resolveChild`, `resolveChildren`, `resolveConditional`
7. Update all tests that provide custom handlers with a passthrough `resolve-include`
8. Verify 478+ tests pass

### Verification

```
nix develop -c just ci ""
nix eval --json --override-input den path:. path:./templates/diag-fx-demo#debug
```
