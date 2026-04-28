# FX Pipeline: Feature Complete + Integration Flag

**Date:** 2026-04-13
**Status:** Approved design
**Branch:** feat/fx-resolution
**Location:** `nix/lib/aspects/fx/`, `flakeModules/fxPipeline.nix`

## Goal

Make the fx pipeline feature-complete for diagram fork compatibility and provide opt-in activation via `den.schema.conf.fxPipeline` option and `flakeModules/fxPipeline.nix`.

## 1. Pipeline Flag

### Option

```nix
den.schema.conf.fxPipeline = lib.mkOption {
  type = lib.types.bool;
  default = false;
  description = "Use effects-based resolution pipeline (experimental)";
};
```

### FlakeModule

```nix
# flakeModules/fxPipeline.nix
{ lib, inputs, ... }: {
  den.schema.conf.fxPipeline = true;
}
```

Users opt in either way:
```nix
# Via flakeModule import
imports = [ den.flakeModules.fxPipeline ];

# Or directly
den.schema.conf.fxPipeline = true;
```

### Input validation

When `fxPipeline` is enabled, the fx module needs nix-effects. Since den receives `inputs` from the consuming flake, the fx init checks `inputs.nix-effects` and throws a clear error if missing:

```nix
assert inputs ? nix-effects || throw
  "den.schema.conf.fxPipeline requires nix-effects as a flake input";
```

### Wiring

The swap happens at `ctx-types.nix:48`. The context type's `__functor` currently calls `ctxApply`. When fx is enabled, it calls a wrapper that runs `fxResolve` (which chains ctxApply + resolution + module collection internally).

In `types.nix`, `den.lib.aspects.resolve` is still called on `config.resolved`. When fx is enabled, `config.resolved` is already fully resolved by the fx `__functor`, so `resolve` becomes a passthrough (the aspect is already a module set). Alternatively, `types.nix` checks the flag and calls `fxResolve` directly.

The cleaner approach: when fx is enabled, override `den.lib.aspects.resolve` to be the fx resolver. This swaps both call sites (types.nix and forward.nix) with one change.

## 2. structuredTraceHandler

Handler on `resolve-complete` producing flat entries for the diagram pipeline. Tracks parent path in handler state.

```nix
structuredTraceHandler = class: {
  "resolve-complete" = { param, state }:
    let
      entry = {
        name = param.name or "<anon>";
        inherit class;
        parent = state.currentParent or null;
        provider = param.meta.provider or [];
        excluded = param.meta.excluded or false;
        excludedFrom = param.meta.excludedFrom or null;
        replacedBy = param.meta.replacedBy or null;
        isProvider = (param.meta.provider or []) != [];
        hasAdapter = (param.meta.adapter or null) != null;
        hasClass = param ? ${class};
        isParametric = param.meta.isParametric or false;
        fnArgNames = param.meta.fnArgNames or [];
        ctxStage = state.currentStage or null;
        ctxKind = state.currentKind or null;
      };
    in {
      resume = param;
      state = state // {
        entries = (state.entries or []) ++ [ entry ];
      };
    };
};
```

### Parametric detection in wrapIdentity

To populate `meta.isParametric` and `meta.fnArgNames`, `wrapIdentity` in resolve.nix checks whether the aspect was a function:

```nix
wrapIdentity = { aspectVal, resolved }:
  let
    fn = if aspectVal ? __functor then aspectVal.__functor aspectVal else null;
    isParametric = fn != null && lib.isFunction fn && lib.functionArgs fn != {};
    fnArgNames = if isParametric then builtins.attrNames (lib.functionArgs fn) else [];
    # ... existing owned/meta logic ...
  in {
    name = ...;
    meta = {
      adapter = ...;
      provider = ...;
    } // lib.optionalAttrs isParametric { inherit isParametric fnArgNames; };
    # ...
  };
```

### Parent path tracking

The `resolve-complete` handler can track parent paths by using the aspect-chain or by having `resolveDeepEffectful` emit a parent context. The simplest approach: the handler accumulates a path stack. When a non-tombstone child resolves, push its name; when recursion returns, pop. Since effects are sequential within `foldl'`, the state naturally tracks depth.

Actually, the handler doesn't control recursion — it only sees `resolve-complete` events in flat order. For parent tracking, we need the resolver to emit the parent identity alongside the child. Two options:

**(A)** Add parent info to the `resolve-complete` param: `fx.send "resolve-complete" (child // { __parent = parentName; })`. The handler reads `__parent`.

**(B)** Emit a `resolve-enter` / `resolve-exit` effect pair bracketing each subtree. The handler maintains a stack.

Recommendation: **(A)** — simpler, no new effects. `__parent` is stripped by the handler before passing through.

## 3. ctxTraceHandler

Handler on `ctx-traverse` producing `__ctxTrace` items and tracking current stage in state:

```nix
ctxTraceHandler = {
  "ctx-traverse" = { param, state }:
    let
      item = {
        key = param.key;
        selfName = param.self.name or "<anon>";
        prevName = if param.prev != null then param.prev.name or "<anon>" else null;
        hasSelfProvider = (param.self.provides or {}) ? ${param.self.name or ""};
        hasCrossProvider = param.prev != null
          && (param.prev.provides or {}) ? ${lib.head (lib.splitString "." param.key)};
      };
    in {
      resume = null;
      state = state // {
        ctxTrace = (state.ctxTrace or []) ++ [ item ];
        currentStage = param.key;
        currentKind = "aspect";
      };
    };
};
```

The `currentStage` and `currentKind` in state are read by `structuredTraceHandler` when building entries.

## 4. Context stage tagging on providers

When `ctx-provider` resolves a provider, the result should carry stage metadata. Update `buildStageIncludes` in ctx-apply.nix to tag provider results:

```nix
selfProvResult = if selfProv != null then
  let r = selfProv ctx; in
  if builtins.isAttrs r then r // { __ctxStage = key; __ctxKind = "self-provide"; }
  else r
else null;
```

The `structuredTraceHandler` reads `__ctxStage`/`__ctxKind` from the param when present, overriding the state-based stage.

## 5. Test improvements

- Replace `>= 1` / `>= 2` with exact expected counts in fx-full-pipeline.nix and fx-e2e.nix
- Add `mkPipeline` test with `extraHandlers` (structuredTraceHandler)
- Add nested adapter override test
- Export `mkPipeline`, `defaultHandlers`, `defaultState` at init top level

## 6. Trace pipeline integration

Users get tracing by using `mkPipeline` with trace handlers:

```nix
tracingPipeline = fxLib.resolve.mkPipeline {
  class = "nixos";
  extraHandlers = fxLib.adapters.structuredTraceHandler "nixos"
    // fxLib.handlers.ctxTraceHandler;
  extraState = { entries = []; ctxTrace = []; };
};

result = tracingPipeline { ctxNs; self; ctx; };
# result.state.entries — flat trace entries for graph IR
# result.state.ctxTrace — context stage transitions
# result.state.imports — collected modules
```

This replaces the existing `resolve.withAdapter adapters.structuredTrace class aspect` call in the diagram pipeline.

## File plan

```
nix/lib/aspects/fx/
  adapters.nix     — add structuredTraceHandler
  resolve.nix      — add __parent to resolve-complete, parametric detection in wrapIdentity
  handlers.nix     — update ctxTraverseHandler to be the tracing version (or add separate)
  ctx-apply.nix    — add __ctxStage/__ctxKind tagging on providers
  default.nix      — promote mkPipeline/defaultHandlers to top-level exports

flakeModules/fxPipeline.nix  — opt-in flakeModule
modules/schema/fxPipeline.nix — option definition + wiring

templates/ci/modules/features/
  fx-trace.nix             — structuredTraceHandler + ctxTraceHandler tests
  fx-flag.nix              — pipeline flag activation tests
  (existing files)         — tighten assertions, add mkPipeline/nested adapter tests
```

## Test plan

1. **structuredTraceHandler** — produces entries with correct fields for static, parametric, excluded aspects
2. **ctxTraceHandler** — produces ctxTrace items with key, selfName, prevName, provider flags
3. **Parent tracking** — entries have correct parent paths
4. **Context stage tagging** — provider entries carry __ctxStage/__ctxKind
5. **mkPipeline with trace handlers** — extraHandlers compose with defaults, state merges
6. **Nested adapter override** — inner meta.adapter overrides outer
7. **Pipeline flag** — `den.schema.conf.fxPipeline = true` activates fx resolution
8. **Pipeline flag without nix-effects** — throws clear error
9. **Exact assertions** — tightened counts in existing test files
