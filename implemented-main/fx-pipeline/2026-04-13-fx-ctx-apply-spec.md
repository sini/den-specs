# Effects-Based Context Application

**Date:** 2026-04-13
**Status:** Approved design
**Branch:** feat/fx-resolution
**Location:** `nix/lib/aspects/fx/`

## Goal

Replace `ctx-apply.nix` with an effects-based context traversal that emits named effects for each traversal stage, provider resolution, and deduplication check. This completes the fx pipeline — from context application through aspect resolution to module collection — as a single unified computation.

## Model

Three named effects for context traversal:

- **`ctx-traverse`** — emitted when entering a new context stage. `param = { prev, prevCtx, self, ctx, key }`. Handler can resume (proceed) or abort (skip stage). Tracing hooks attach here.

- **`ctx-provider`** — emitted to request provider includes. `param = { kind = "self"|"cross"; self; ctx; key; prev; prevCtx }`. Handler resumes with the provider function (or null). Makes provider resolution visible and interceptable.

- **`ctx-seen`** — emitted to check/register deduplication. `param = key` (string). Handler resumes with `{ isFirst = bool; }` and tracks seen keys in state. Determines `fixedTo` vs `atLeast` application.

### Why effects instead of plain recursion

The existing `ctx-apply.nix` builds a flat tuple list via recursive `traverse`, then folds with `assembleIncludes` threading a `seen` accumulator. This works but:

1. Tracing requires bolting on `tagStage`/`traceItem` (as `feat/new-trace-demo` does)
2. Provider resolution is inline — failures are silent `noop`
3. Deduplication is manual accumulator threading
4. No hook points for adapters or diagnostics

With effects, each concern is a handler. The traversal emits what it needs, handlers decide what happens. Tracing, dedup, and provider resolution are orthogonal handlers on the same computation.

## Implementation

### ctxApplyEffectful

Returns `Computation [aspect]` — a list of includes for the root aspect, ready to feed into `resolveDeepEffectful`.

```nix
ctxApplyEffectful = ctxNs: self: ctx:
  let
    noop = _: {};
    flattenInto = <reused from ctx-apply.nix>;
    resolveAspect = path: lib.attrByPath path null ctxNs;

    # Recursive traversal. Emits ctx-traverse at each stage,
    # then descends into `into` transitions.
    traverse = { prev, prevCtx, self, ctx, key }:
      fx.bind (fx.send "ctx-traverse" { inherit prev prevCtx self ctx key; }) (_:
        fx.bind (buildStageIncludes { inherit prev prevCtx self ctx key; }) (stageIncludes:
          let
            intoList = flattenInto ((self.into or noop) ctx) [];
          in
          fx.bind (foldInto intoList { inherit self ctx; }) (childIncludes:
            fx.pure (stageIncludes ++ childIncludes)
          )
        )
      );

    foldInto = intoList: { self, ctx }:
      builtins.foldl'
        (acc: { path, into }:
          fx.bind acc (results:
            let
              aspect = resolveAspect path;
              aspectKey = lib.concatStringsSep "." path;
              pathHead = lib.head path;
              hasProvider = self.provides ? ${pathHead};
            in
            if aspect != null then
              foldContexts into results {
                prev = self; prevCtx = ctx;
                self = aspect; key = aspectKey;
              }
            else if builtins.length path == 1 && hasProvider then
              foldContexts into results {
                prev = self; prevCtx = ctx;
                self = { name = pathHead; into = noop; };
                key = pathHead;
              }
            else
              fx.pure results
          )
        )
        (fx.pure [])
        intoList;

    foldContexts = contexts: results: { prev, prevCtx, self, key }:
      builtins.foldl'
        (acc: c:
          fx.bind acc (innerResults:
            fx.bind (traverse { inherit prev prevCtx self key; ctx = c; }) (stageResults:
              fx.pure (innerResults ++ stageResults)
            )
          )
        )
        (fx.pure results)
        contexts;

    # Build includes for a single stage: main aspect + providers.
    buildStageIncludes = { prev, prevCtx, self, ctx, key }:
      fx.bind (fx.send "ctx-seen" key) ({ isFirst }:
        fx.bind (fx.send "ctx-provider" { kind = "self"; inherit self ctx key prev prevCtx; }) (selfProv:
          fx.bind (fx.send "ctx-provider" { kind = "cross"; inherit self ctx key prev prevCtx; }) (crossProv:
            let
              mainAspect =
                if isFirst
                then parametric.fixedTo ctx self
                else parametric.atLeast self ctx;
              selfProvResult = if selfProv != null then selfProv ctx else null;
              crossProvResult = if crossProv != null then crossProv ctx else null;
            in
            fx.pure (
              [ mainAspect ]
              ++ lib.optional (selfProvResult != null) selfProvResult
              ++ lib.optional (crossProvResult != null) crossProvResult
            )
          )
        )
      );
  in
  traverse {
    prev = null;
    prevCtx = null;
    key = self.name;
    inherit self ctx;
  };
```

### Default handlers

```nix
# Deduplication handler. Tracks seen keys in state.
ctxSeenHandler = {
  "ctx-seen" = { param, state }:
    let
      key = param;
      isFirst = !(state.seen ? ${key});
    in {
      resume = { inherit isFirst; };
      state = state // { seen = (state.seen or {}) // { ${key} = true; }; };
    };
};

# Provider resolution handler. Looks up provides chains.
ctxProviderHandler = {
  "ctx-provider" = { param, state }:
    let
      inherit (param) kind self ctx key prev prevCtx;
      noop = _: {};
    in
    if kind == "self" then {
      resume = self.provides.${self.name} or null;
      inherit state;
    }
    else if kind == "cross" && prev != null then {
      resume =
        let
          pathHead = lib.head (lib.splitString "." key);
          provFn = prev.provides.${pathHead} or null;
        in
        if provFn != null then provFn prevCtx else null;
      inherit state;
    }
    else {
      resume = null;
      inherit state;
    };
};

# Traverse handler. Default: proceed (resume with null, meaning "continue").
# Trace handlers can wrap this to record stage entries.
ctxTraverseHandler = {
  "ctx-traverse" = { param, state }: {
    resume = null;
    inherit state;
  };
};
```

### Full pipeline

The complete resolution from context to modules:

```nix
fxFullResolve = { ctxNs, class, self, ctx }:
  let
    comp = fx.bind (ctxApplyEffectful ctxNs self ctx) (includes:
      resolveDeepEffectful {
        ctx = {};
        inherit class;
        aspect-chain = [];
      } {
        name = self.name or "<anon>";
        meta = self.meta or {};
        inherit includes;
      }
    );
  in
  fx.handle {
    handlers =
      ctxTraverseHandler
      // ctxSeenHandler
      // ctxProviderHandler
      // { "resolve-include" = { param, state }: { resume = [ param ]; inherit state; }; }
      // (moduleHandler class);
    state = { seen = {}; imports = []; };
  } comp;
  # Returns { value = resolvedTree; state = { seen = {...}; imports = [...]; }; }
```

The entry point for real den integration:

```nix
# Drop-in replacement for resolve.withAdapter adapters.default
fxResolve = ctxNs: class: self: ctx:
  let result = fxFullResolve { inherit ctxNs class self ctx; }; in
  { imports = result.state.imports; };
```

### Root module collection fix

`resolveDeepEffectful` doesn't emit `resolve-complete` for the root aspect. The root's own class module must be collected separately in `fxFullResolve`:

```nix
# After resolveDeepEffectful returns, extract root's module
comp = fx.bind (ctxApplyEffectful ctxNs self ctx) (includes:
  fx.bind (resolveDeepEffectful { ... } { ... includes ... }) (resolved:
    # Emit resolve-complete for the root itself
    fx.bind (fx.send "resolve-complete" resolved) (_:
      fx.pure resolved
    )
  )
);
```

## File plan

```
nix/lib/aspects/fx/
  ctx-apply.nix  — ctxApplyEffectful, flattenInto, resolveAspect, buildStageIncludes
  handlers.nix   — add ctxSeenHandler, ctxProviderHandler, ctxTraverseHandler
  resolve.nix    — add fxFullResolve, fxResolve entry point
  default.nix    — updated exports
```

## Test plan

Tests in `templates/ci/modules/features/fx-ctx-apply.nix`:

1. **Single context stage** — host with one aspect, traversal produces includes
2. **Into transition** — host → user fan-out, both stages traversed
3. **Self-provider** — `provides.host` resolved via ctx-provider effect
4. **Cross-provider** — previous context's provider resolved
5. **Deduplication** — same key visited twice, second gets `atLeast` not `fixedTo`
6. **Provider not found** — missing provider returns null, no crash
7. **Full pipeline** — ctxApply → resolveDeepEffectful → moduleHandler produces `{ imports }`
8. **With adapter** — root meta.adapter excludes aspect through full pipeline
9. **With includeIf** — conditional include through full pipeline

## What this spec skips

- `feat/new-trace-demo` trace handler integration — the `ctx-traverse` effect provides the hook point, but the actual structuredTrace handler is separate work
- `__ctxStage`/`__ctxKind` tagging — can be added as a ctx-traverse handler later
- Performance optimization
- Wiring into den's module system (`ctx-types.nix` `__functor` replacement) — separate integration step

## Open questions

1. **ctxNs parameter threading.** `ctxApplyEffectful` needs the context namespace (`ctxNs`) to resolve aspect paths. Currently passed as a closure. Could alternatively be an effect (`fx.send "resolve-aspect" path`) so the namespace is handler-provided. Keeping it as a closure for now — simpler.
2. **Provider handler complexity.** The cross-provider logic (`prev.provides.${pathHead} prevCtx`) is dense. If it causes issues during implementation, split into separate `self-provider` and `cross-provider` effects.
3. **Root aspect handling.** The root aspect (context entity itself) doesn't go through `ctx-traverse` — it's the starting point. Its own class module needs explicit collection. The `resolve-complete` emission for root handles this.
