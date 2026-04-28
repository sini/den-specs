# Vision: recursive aspects + structural effects on nix-effects

**Date:** 2026-04-12
**Status:** Vision document, builds on nix-effects-resolution-spec
**Depends on:** nix-effects-resolution-spec, PR #449 (class registration)
**Supersedes:** `2026-04-12-recursive-aspects-vision.md` (used nfx primitives)

## The end state

Users write hierarchical aspects as plain nested attrs. Structural
queries and conditional includes work naturally. The module system
handles declaration. nix-effects handles resolution. The manually-
threaded context-passing machinery that caused every recent regression
is gone.

```nix
# Declare a taxonomy: just nested attrs
den.aspects.dev-tools.cli.bat.nixos = { pkgs, ... }: {
  environment.systemPackages = [ pkgs.bat ];
};
den.aspects.dev-tools.gui.alacritty.homeManager.programs.alacritty.enable = true;

# Conditional includes: hasAspect is a named effect, not cyclic
den.aspects.impermanence = { host, hasAspect, ... }: {
  includes =
    lib.optional (hasAspect <zfs-root>) <zfs-impermanence>
    ++ lib.optional (hasAspect <btrfs-root>) <btrfs-impermanence>;
};

# Structural queries from class-config bodies (still works)
den.aspects.monitoring.nixos = { host, ... }:
  lib.mkIf (host.hasAspect <prometheus>) {
    services.node-exporter.enable = true;
  };

# Providers stay explicit
den.aspects.igloo.provides.to-users = { user, ... }: { ... };

# Include the root: hierarchy auto-flows
den.aspects.igloo.includes = [ den.aspects.dev-tools ];
```

Three features working together:

1. **Natural nesting** via recursive freeform children
2. **Structural effects** via nix-effects named requests and handlers
3. **Clean separation** of children (freeform) and providers (`provides`)

## Feature 1: recursive freeform children

(Unchanged from the previous vision doc. This is a module-system-level
change that's independent of the effects integration.)

PR #449 registers class names as typed options on `den.schema.aspect`.
Once classes are declared options, the freeform can mean "child aspect"
instead of "class config." Opt-in via flakeModule for backward compat.

```nix
# flakeModules/recursiveAspects.nix
{ den, lib, ... }: {
  den.schema.aspect = { config, ... }: {
    _module.freeformType = lib.types.lazyAttrsOf (
      den.lib.aspects.types.aspectType {
        inherit (den.lib.aspects.types) defaultFunctor;
        providerPrefix = (config.meta.provider or []) ++ [ config.name ];
      }
    );
    config.includes = builtins.attrValues (
      lib.filterAttrs (n: _: !(config._module.options ? ${n})) config
    );
  };
}
```

Users opt in with one import. Default behavior unchanged.

## Feature 2: structural effects via nix-effects

### hasAspect as a named effect

With nix-effects' `send`/`handle`/`resume`, `hasAspect` becomes
a request that any computation can make:

```nix
# Inside bind.fn, hasAspect is just another context request
aspect = fx.bind.fn ({ host, hasAspect }:
  fx.pure {
    includes =
      lib.optional (hasAspect <zfs-root>) <zfs-impermanence>;
  }
);
```

The handler at the resolve root:

```nix
hasAspectHandler = structuralSet: {
  has-aspect = { param, state }:
    { resume = structuralSet ? ${pathKey (aspectPath param)};
      inherit state; };
};
```

The computation suspends at `send "has-aspect" ref`. The handler
looks up the answer in the precomputed structural set. Resumes
with a boolean. The computation continues from where it left off.

No phased resolution. No re-evaluation. No fixed-point iteration.
No cycle. The handler has the structural set before any conditional
includes are evaluated, because the structural pass walks the
DECLARED tree (before effects fire) and the effect pass evaluates
AGAINST that.

### The (iii) case is now supported

The case we rejected during hasAspect design (hasAspect inside
`includes`) works because nix-effects' `resume` semantics decouple
the query from the tree construction:

1. **Structural pass.** Walk all aspects, collect declared includes
   (ignoring effect requests). Build the structural presence set.
2. **Effect pass.** `fx.run` evaluates the aspects as effectful
   computations. Each `send "has-aspect"` is answered by the handler
   via `resume`. Conditional includes materialize.
3. **Module pass.** Extract class modules from the resolved tree.
   Build the final NixOS/home-manager module.

The structural pass uses the same `collectPaths` walker we already
shipped. The effect pass is `fx.run` with the handler stack. The
module pass is the existing module adapter.

### onlyIf becomes a one-liner

```nix
# Today: meta.adapter with raw tree walk (~15 lines)
onlyIf = guard: target: { ... meta.adapter = ...; };

# With nix-effects: plain effect composition
onlyIf = guard: target:
  fx.bind (fx.send "has-aspect" guard) (present:
    if present then fx.pure target else fx.pure {}
  );
```

No meta.adapter. No raw tree walk. No bypassing filterIncludes.
The `send` goes to the handler. The handler answers. Done.

### Context decomposition via bind.fn

Vic's key insight from PR #446: "With bind.fn, each of these
function arguments are independent requests to the environment."

```nix
# Today: fixed context shape, canTake discriminates
aspect = { host, user, ... }: { nixos = ...; };
# canTake.atLeast checks { host = false; user = false; }
# Both must be present in ctx or the function doesn't fire

# With bind.fn: each arg is an independent send
aspect = fx.bind.fn ({ host, user }:
  fx.pure { nixos = ...; }
);
# send "host" → handler provides host
# send "user" → handler provides user
# Each satisfied independently. No canTake. No fixed shapes.
```

This eliminates:
- `canTake.atLeast` / `exactly` / `upTo`
- `take.atLeast` / `exactly` / `upTo`
- `isSubmoduleFn` / `isProviderFn` / `isOtherCtxFn`
- `coercedProviderType`
- `carryAttrs` / `carryMeta`
- `applyDeep`
- `parametric.withOwn` (static vs non-static branching)

The entire zoo. ~230 lines of bug-prone code replaced by `bind.fn`
+ scoped handlers.

### Conditions for diagnostics

```nix
# Strict mode: signal condition, handler decides
resolveChild = child:
  if isEmpty child then
    fx.bind
      (fx.send "condition" {
        type = "empty-aspect";
        key = child.name;
        hint = "register as class?";
      })
      (_: fx.pure (tombstone child {}))
  else
    resolveAspect child;

# Strict handler aborts
strictHandlers.condition = { param, state }:
  { abort = throw "STRICT: ${param.hint}"; };

# Lenient handler warns and resumes
lenientHandlers.condition = { param, state }:
  { resume = builtins.trace "warn: ${param.hint}" null;
    inherit state; };
```

## Feature 3: providers stay explicit

`provides` (aliased `_`) remains for cross-entity contributions.
Unchanged from the previous vision doc. Children are freeform.
Providers are named. The two don't conflate.

## End-to-end example

```nix
# modules/aspects/dev-tools.nix
{ den, ... }: {
  den.aspects.dev-tools.cli.bat.nixos = { pkgs, ... }: {
    environment.systemPackages = [ pkgs.bat ];
  };
  den.aspects.dev-tools.cli.ripgrep.nixos = { pkgs, ... }: {
    environment.systemPackages = [ pkgs.ripgrep ];
  };
  den.aspects.dev-tools.gui.alacritty.homeManager.programs.alacritty.enable = true;
}

# modules/aspects/impermanence.nix
{ den, lib, ... }: {
  # hasAspect in includes: works because it's a named effect,
  # not a cyclic read on the tree being constructed
  den.aspects.impermanence = { host, hasAspect, ... }: {
    includes =
      lib.optional (hasAspect <zfs-root>) <zfs-impermanence>
      ++ lib.optional (hasAspect <btrfs-root>) <btrfs-impermanence>;
  };
  den.aspects.zfs-impermanence.nixos = { ... };
  den.aspects.btrfs-impermanence.nixos = { ... };
}

# modules/aspects/monitoring.nix
{ den, lib, ... }: {
  # hasAspect in class body: same as today, still works
  den.aspects.monitoring.nixos = { host, ... }: lib.mkMerge [
    { services.prometheus.enable = true; }
    (lib.mkIf (host.hasAspect <loki>) {
      services.promtail.enable = true;
    })
  ];
}

# modules/aspects/secrets.nix
{ den, ... }: {
  den.aspects.secrets = {
    includes = [ <agenix-rekey> <sops-nix> ];
    meta.adapter = den.lib.aspects.adapters.oneOfAspects [
      <agenix-rekey>
      <sops-nix>
    ];
  };
}

# modules/aspects/igloo.nix
{ den, ... }: {
  den.aspects.igloo = {
    includes = [
      den.aspects.dev-tools
      den.aspects.impermanence
      den.aspects.monitoring
      den.aspects.secrets
    ];
    provides.to-users = { user, ... }: {
      homeManager.programs.git.userName = user.name;
    };
  };
}

# flake.nix
{
  inputs.den.url = "github:vic/den";

  outputs = { den, ... }: den.lib.mkFlake {
    imports = [
      den.flakeModule
      den.flakeModules.recursiveAspects  # opt-in: freeform = children
    ];
    den.hosts.x86_64-linux.igloo.users.tux = {};
  };
}
```

## Implementation phases

### Phase 1: recursive freeform (no nix-effects, lands first)

- Lift class registrations from strict-only to unconditional
- Add `recursiveAspects` flakeModule (freeform override)
- Auto-include freeform children
- Ship `onlyIf` adapter (current meta.adapter approach)
- Tests, template examples, docs

Works on the current architecture. No nix-effects dependency.

### Phase 2: nix-effects replaces context threading (Vic's work)

- Add nix-effects as a den input
- Delete `take.nix`, `can-take.nix`, `statics.nix`
- Rewrite `parametric.nix` to use `bind.fn` + scoped handlers
- Rewrite `ctx-apply.nix` as handler installation
- Rewrite `resolve.nix` as `fx.run` + handler composition
- Rewrite `adapters.nix` as handler combinators (public API stays)
- Rewrite `strict.nix` as condition handlers
- Verify all existing tests pass

### Phase 3: structural effects (builds on Phase 2)

- `hasAspect` becomes `send "has-aspect"` (enables iii)
- `onlyIf` migrates from meta.adapter to effect combinator
- `bind.fn` provides `hasAspect` as a function arg alongside
  `host`, `user`, etc.
- Conditions for structural diagnostics
- The (iii) rejection lifts

### Phase 4: type kernel integration (future)

- Aspect schemas verified by nix-effects' MLTT kernel
- `refined "Port" Int (x: x >= 1 && x <= 65535)` for host options
- Dependent record types for aspect configuration
- Proof terms for universal properties ("every service with a port
  has a matching firewall rule")

## What each phase delivers

| Phase | User gets | Internal win |
|---|---|---|
| 1 | Natural nesting, onlyIf | Module-level change only |
| 2 | Same features, faster eval | ~300 lines of bugs deleted |
| 3 | hasAspect in includes, cleaner onlyIf | Architecture unlocked |
| 4 | Verified aspect schemas | Domain-specific type safety |

Each phase is independently shippable. Phase 1 needs no nix-effects.
Phase 2 is a refactor (no new user features). Phase 3 is the user-
facing payoff. Phase 4 is future exploration.

## Relationship to existing work

Everything we've built evolves in place:

- **hasAspect (shipped):** stays in Phase 1. Becomes a `send` handler
  in Phase 2. Gains includes-level usage in Phase 3.
- **oneOfAspects (shipped):** stays in Phase 1. Becomes a handler
  combinator in Phase 2. Simplifies in Phase 3.
- **onlyIf (next PR):** ships as meta.adapter in Phase 1. Becomes
  an effect combinator in Phase 3.
- **collectPaths (shipped):** stays. Becomes the structural-presence
  handler's implementation in Phase 2.
- **PR #449 strict mode:** prerequisite for Phase 1. Diagnostics
  migrate to conditions in Phase 2.
- **carryMeta fix (shipped today):** the LAST bug fix in the old
  manual-threading stack. Phase 2 makes it unnecessary because
  nix-effects carries identity through the handler chain.

Nothing is wasted. Each piece has a migration path.
