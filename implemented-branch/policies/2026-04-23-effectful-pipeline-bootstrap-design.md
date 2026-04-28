# Effectful Pipeline Bootstrap

**Goal:** Move pipeline setup (policy resolution, target assembly) inside `fx.handle` so the entire pipeline lifecycle is interceptable via effects. Enable policies to install named effect handlers scoped to their transition context.

**Motivation:** Code quality (eliminate inline setup logic) + Layer 2 groundwork (per-policy named handlers, interceptable routing).

## New Effects

| Effect | Sent by | Default handler | Override use case |
|--------|---------|----------------|------------------|
| `resolve-policy` | mkPipeline bootstrap | Calls `mergePolicyInto` | Custom routing, conditional policies, priority ordering |
| `resolve-target` | Transition handler | Looks up `den.stages`, assembles aspect, merges policies | Virtual stages, custom target assembly |

Policy `handlers` are installed via the existing `scope.provide` mechanism — not a new effect, just a new use of existing infrastructure.

## Component Changes

### 1. Policy type extension (`policy-types.nix`)

Add optional `handlers` field:

```nix
policyType = lib.types.submodule {
  options = {
    from = lib.mkOption { type = lib.types.str; };
    to = lib.mkOption { type = lib.types.str; };
    resolve = lib.mkOption { type = lib.types.raw; };
    handlers = lib.mkOption {
      type = lib.types.lazyAttrsOf lib.types.raw;
      default = { };
      description = "Named effect handlers installed when this policy fires.";
    };
  };
};
```

Backward compatible — existing policies don't set `handlers`, default is `{}`.

### 2. mkPipeline restructure (`pipeline.nix`)

Move policy merge from pre-computation setup into the computation itself:

```nix
# Before: setup outside fx.handle
mergedInto = mergePolicyInto selfName existingInto;
effectiveSelf =
  if mergedInto != null && mergedInto != existingInto then
    self // { meta = (self.meta or {}) // { into = mergedInto; }; }
  else self;
rootEffect = aspectToEffect effectiveSelf;
fx.handle { handlers = composeHandlers rootHandlers extraHandlers; state; } rootEffect;

# After: setup inside fx.handle via effect
bootstrapAndResolve =
  fx.bind (fx.send "resolve-policy" {
    stageName = self.name or "";
    existingInto = self.meta.into or self.into or null;
  }) (mergedInto:
    let
      # Preserve equality guard: only override into when policies added new transitions
      effectiveSelf =
        if mergedInto != null && mergedInto != existingInto then
          self // { meta = (self.meta or {}) // { into = mergedInto; }; }
        else self;
    in
    aspectToEffect effectiveSelf
  );
fx.handle { handlers = composeHandlers rootHandlers extraHandlers; state; } bootstrapAndResolve;
```

New `resolvePolicyHandler` added to `defaultHandlers`:

```nix
resolvePolicyHandler = {
  "resolve-policy" = { param, state }: {
    resume = den.lib.synthesizePolicies.mergePolicyInto param.stageName param.existingInto;
    inherit state;
  };
};
```

API shape unchanged — `fxFullResolve` and `fxResolve` still take `{ class, self, ctx }`.

### 3. Transition handler (`transition.nix`)

#### 3a: `buildTarget` → `resolve-target` effect

Replace inline `buildTarget` function with an effect:

```nix
# Before: direct function call
effectiveTarget = buildTarget transition;

# After: effect
fx.bind (fx.send "resolve-target" {
  path = transition.path;
  targetClass = state.class or "nixos";
}) (effectiveTarget: ...)
```

Default `resolveTargetHandler`. Contract: MUST resume with either a valid stage aspect (with `.name`, `.meta`) or `null`:

```nix
resolveTargetHandler = {
  "resolve-target" = { param, state }: {
    resume =
      let
        stageAspect = lib.attrByPath param.path null (den.stages or {});
        targetName = if stageAspect != null then stageAspect.name or "" else "";
        existingInto = if stageAspect != null then stageAspect.meta.into or null else null;
        mergedInto = den.lib.synthesizePolicies.mergePolicyInto targetName existingInto;
      in
      if stageAspect != null && mergedInto != null then
        stageAspect // { meta = (stageAspect.meta or {}) // { into = mergedInto; }; }
      else if stageAspect != null then
        stageAspect
      else
        null;
    inherit state;
  };
};
```

#### 3b: Policy handler installation

When a transition fires and matching policies have `handlers`, install them via `scope.provide`. The `targetKey` is the stage name derived from the transition path (e.g., `"user"` from path `["user"]`). Matching uses the policy's `from` field against the root pipeline's stage name (available as `state.class` or the source aspect's stage identity):

```nix
# targetKey from transition path
targetKey = lib.concatStringsSep "." transition.path;

# Collect handlers from policies whose from/to match this transition
matchingPolicies = lib.filter
  (p: p.to == targetKey)
  (builtins.attrValues (den.policies or {}));
policyHandlers = builtins.foldl' (acc: p: acc // (p.handlers or {})) {} matchingPolicies;

# Core pipeline effects that policy handlers MUST NOT shadow.
# Shadowing these would break the pipeline (e.g., infinite loops).
coreEffects = [
  "into-transition" "ctx-seen" "resolve-complete" "emit-class"
  "emit-include" "chain-push" "chain-pop" "check-constraint"
  "register-constraint" "defer-include" "drain-deferred"
  "get-path-set" "has-handler" "resolve-policy" "resolve-target"
];
safePolicyHandlers = builtins.removeAttrs policyHandlers coreEffects;
```

**Scope layering:** Two nested `scope.provide` layers exist in the transition resolution:
1. **Policy scope** (new) — wraps the entire transition sub-computation. Installed here by the transition handler. Contains policy-declared handlers (e.g., `"peer"`).
2. **Context scope** (existing) — wraps individual `aspectToEffect` calls via `__scopeHandlers`. Contains context values (e.g., `host`, `user`) from `constantHandler`.

These nest correctly: context scope is inner (per-aspect), policy scope is outer (per-transition). An aspect resolved under a transition can access both policy handlers and context values.

```nix
# scope.provide installs safe policy handlers for the sub-computation
resolveInScope =
  if safePolicyHandlers != {} then
    fx.effects.scope.provide safePolicyHandlers computation
  else
    computation;
```

Handlers are scoped — active only within that transition's resolution, not globally. Fan-out sub-pipelines (`fxFullResolve`) create fresh handler scopes, so policy handlers don't propagate into them.

#### 3c: `buildTarget` deleted

Its logic moves entirely into the `resolveTargetHandler`. The transition handler becomes pure orchestration — sends effects, delegates decisions.

### 4. Handler registration in defaultHandlers

Both new handlers added to `defaultHandlers` in `pipeline.nix`:

```nix
defaultHandlers = { class, ctx }:
  handlers.constantHandler (...)
  // handlers.classCollectorHandler { ... }
  // handlers.constraintRegistryHandler
  // handlers.chainHandler
  // handlers.includeHandler
  // handlers.transitionHandler
  // handlers.ctxSeenHandler
  // identity.pathSetHandler
  // identity.collectPathsHandler
  // handlers.deferredIncludeHandler
  // handlers.drainDeferredHandler
  // handlers.resolvePolicyHandler      # NEW
  // handlers.resolveTargetHandler      # NEW
  // fx.effects.state.handler;
```

## What This Enables (Layer 2)

```nix
# Policy with named handlers — aspects under this transition
# can query "peer" via effects
den.policies.host-to-user = {
  from = "host";
  to = "user";
  resolve = { host }: map (u: { inherit host; user = u; }) (attrValues host.users);
  handlers.peer = { param, state }: {
    resume = /* peer entity lookup */;
    inherit state;
  };
};

# Aspect using the policy's handler
den.aspects.my-app = { host, user, ... }: {
  nixos = { config, ... }: {
    # This fx.send "peer" works because host-to-user installed the handler
    networking.firewall.allowedTCPPorts = [ 8080 ];
  };
};
```

## Backward Compatibility

- All existing policy declarations work unchanged (`handlers` defaults to `{}`)
- `fxFullResolve` / `fxResolve` API shapes unchanged
- Default handlers preserve exact current behavior
- `extraHandlers` in `mkPipeline` still works for composition
- Existing tests pass without modification

## Files Changed

| File | Change |
|------|--------|
| `nix/lib/policy-types.nix` | Add `handlers` option |
| `nix/lib/aspects/fx/pipeline.nix` | Lift setup into computation; add `resolvePolicyHandler` |
| `nix/lib/aspects/fx/handlers/transition.nix` | Replace `buildTarget` with `resolve-target` effect; install policy handlers via `scope.provide` |
| `nix/lib/aspects/fx/handlers/tree.nix` or new file | `resolveTargetHandler` (could live in transition.nix or separate) |
| New test file | Test effect override, policy handler scoping |

## Open Questions

1. **Where does `resolveTargetHandler` live?** Could be in `transition.nix` (co-located with the sender) or a new `target.nix` handler file (separation of concerns). Recommendation: keep in `transition.nix` since it's tightly coupled to the transition protocol.

2. **Policy handler collection timing:** Should matching policies be looked up per-transition (current design) or cached at pipeline init? Per-transition is simpler and correct; caching is an optimization if needed later.
