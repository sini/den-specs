# Forward as Handler

**Goal:** Replace the forward mechanism's fresh pipeline re-resolve with an effect handler that resolves within the current pipeline's handler scope.

**Problem:** `den.provides.forward` calls `den.lib.aspects.resolve fromClass sourceAspect` which creates a brand new `fxResolve` pipeline. This fresh pipeline lacks the current pipeline's context (pkgs, config, host, user, etc.) because it runs with empty handlers. Function-valued class keys like `{ pkgs, ... }: { inherit (pkgs) htop; }` get deferred and lost because `pkgs` has no handler.

On `origin/main` this was masked because `aspect-chain` carried a unified ctx tree where everything was pre-merged. With stages separated, the fresh pipeline gets a stage that has includes but no context to resolve parametric content.

**Root cause confirmed by evidence:**
- CI unit tests prove the pipeline correctly collects function class keys when run directly
- Tracing confirms the transition fires and `perSystemFwd` is called
- Tracing confirms `aspect-chain` carries the correct source stage
- But `packages.x86_64-linux` returns `["hola"]` — den-forwarded packages are missing
- The fresh pipeline in `forwardItem` resolves the source but collected modules don't reach the target eval context correctly

## Design: `emit-forward` effect

Replace the forward's separate pipeline call with an effect that the current pipeline's handler infrastructure processes.

### New effect

```
fx.send "emit-forward" {
  fromClass;    # "packages" — class to extract from source
  intoClass;    # "flake-parts" — class to inject into
  intoPath;     # ["packages"] — path within intoClass module
  source;       # the source aspect (from fromAspect)
  adaptArgs;    # optional: specialArgs for the target module
  guard;        # optional: conditional
  evalConfig;   # optional: evaluate source in freeform submodule
}
```

### Handler: `forwardHandler`

```nix
forwardHandler = {
  "emit-forward" = { param, state }:
    let
      # Resolve source in a sub-pipeline for fromClass.
      # Uses fxFullResolve but inherits the current pipeline's context
      # by passing the current state's context values.
      sourceResult = den.lib.aspects.fx.pipeline.fxFullResolve {
        class = param.fromClass;
        self = param.source;
        ctx = (state.currentCtx or (_: {})) null;
      };
      sourceImports = sourceResult.state.imports null;

      # Wrap as intoClass module
      module = if param.evalConfig or false then
        # Evaluate and set at path
        let
          evaluated = lib.evalModules {
            modules = [ freeformMod ] ++ sourceImports;
            specialArgs = param.adaptArgs or {};
          };
        in {
          ${param.intoClass} = lib.setAttrByPath param.intoPath
            (builtins.removeAttrs evaluated.config ["_module"]);
        }
      else if param.adaptArgs or null != null then
        # Adapter: submodule with specialArgs
        let
          adapterKey = "${param.fromClass}/${param.intoClass}/${lib.concatStringsSep "/" param.intoPath}";
        in {
          ${param.intoClass} = {
            __functionArgs = if param.guard != null then lib.functionArgs param.guard else {};
            __functor = _: args: {
              options.den.fwd.${adapterKey} = lib.mkOption {
                default = {};
                type = lib.types.submoduleWith {
                  specialArgs = param.adaptArgs args;
                  modules = sourceImports;
                };
              };
              config = lib.setAttrByPath param.intoPath
                args.config.den.fwd.${adapterKey};
            };
          };
        }
      else
        # Direct: set imports at path
        {
          ${param.intoClass} = lib.setAttrByPath param.intoPath (_: {
            imports = sourceImports;
          });
          meta.contextDependent = true;
        };
    in {
      resume = null;
      state = state // {
        imports = x: (state.imports x) ++ [ module ];
      };
    };
};
```

### Key difference from current design

**Current:** `forwardItem` calls `den.lib.aspects.resolve` synchronously, producing a module. The module is returned as the forward's aspect value. The current pipeline emits it as a class key.

**New:** `forwardItem` sends `emit-forward` effect. The handler resolves the source using `fxFullResolve` with the current pipeline's context (from `state.currentCtx`). The result is appended directly to `state.imports`. No class key emission needed — the handler manages state directly.

### What changes

| File | Change |
|------|--------|
| `nix/lib/forward.nix` | `forwardItem` sends `emit-forward` effect instead of calling `aspects.resolve`. Returns `fx.pure null` or the effect computation. |
| `nix/lib/aspects/fx/pipeline.nix` | Add `forwardHandler` to `defaultHandlers` |
| `nix/lib/aspects/fx/handlers/forward.nix` | New file: `forwardHandler` implementation |

### Context propagation

The handler uses `state.currentCtx` to get the accumulated pipeline context. This context includes `host`, `user`, `system`, etc. from transitions. The sub-pipeline (`fxFullResolve`) runs with this context, so parametric class keys like `{ pkgs, ... }` can resolve if `pkgs` is in context.

For flake-parts, `pkgs` comes from `adaptArgs = { config, ... }: config.allModuleArgs`. The adapter pattern provides `pkgs` as a specialArg on the target module, not as a pipeline context value. This still works because:
1. The sub-pipeline collects class modules as deferred modules (functions stored as-is)
2. The adapter wraps them in a submodule with `specialArgs = adaptArgs args`
3. The deferred module functions are evaluated in the adapter's submodule where `pkgs` is available

### Backward compatibility

- `den.provides.forward` API unchanged — it still takes the same spec attrset
- `forwardEach` unchanged
- Templates that use `fromAspect`/`aspect-chain` still work (the handler resolves from the source)
- The handler is registered in `defaultHandlers` so it's available in all pipelines

### What this fixes

1. **flake-parts-modules template:** `perSystemFwd` forward now resolves within the current pipeline's context. The `packages` class from `den.aspects.foo` is correctly collected and forwarded.
2. **nvf-standalone template:** The `flakeSystemOutputs.nix` stage fallback becomes unnecessary — the handler uses `state.currentCtx` which has the right context.
3. **aspect-chain dependency:** Forwards no longer need `aspect-chain` at all. The handler accesses the source directly. The `fromAspect = _: lib.head aspect-chain` pattern becomes `fromAspect = _: source` where `source` is the stage/aspect being forwarded.

### Open questions

1. **Should the sub-pipeline share state with the parent?** Currently the handler runs `fxFullResolve` which creates independent state. State sharing would let the sub-pipeline's deferred includes be drained by the parent, but also risks state key conflicts.

2. **Should `emit-forward` be interceptable?** If registered as a named effect handler, users could override forward resolution. This aligns with the effectful bootstrap design (resolve-policy, resolve-target are already interceptable).

3. **Can `forwardItem` become purely effectful?** Currently `forwardItem` builds complex attrsets (adapter, topLevelAdapter). If `emit-forward` handles all this, `forwardItem` simplifies to just sending the effect with the right params.
