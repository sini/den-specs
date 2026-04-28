# Vision: recursive aspects + structural effects

**Date:** 2026-04-12
**Status:** Vision document, builds on nfx-resolution-spec
**Depends on:** nfx-resolution-spec, PR #449 (class registration)

## The end state

Users write hierarchical aspects as plain nested attrs. Structural
queries and conditional includes work naturally. The module system
handles declaration; nfx handles resolution.

```nix
# Declare a taxonomy — just nested attrs, auto-wired
den.aspects.dev-tools.cli.bat.nixos = { pkgs, ... }: {
  environment.systemPackages = [ pkgs.bat ];
};
den.aspects.dev-tools.cli.ripgrep.nixos = { pkgs, ... }: {
  environment.systemPackages = [ pkgs.ripgrep ];
};
den.aspects.dev-tools.gui.alacritty.homeManager.programs.alacritty.enable = true;

# Providers stay explicit and separate
den.aspects.igloo.provides.to-users = { user, ... }: {
  homeManager.programs.helix.enable = user.name == "alice";
};

# Conditional includes — the (iii) case that used to cycle
den.aspects.impermanence = { host, ... }: {
  includes = [
    (den.lib.onlyIf <zfs-root> <zfs-impermanence>)
    (den.lib.onlyIf <btrfs-root> <btrfs-impermanence>)
  ];
};

# Structural queries from class-config bodies
den.aspects.monitoring.nixos = { host, ... }:
  lib.mkIf (host.hasAspect <prometheus>) {
    services.node-exporter.enable = true;
  };

# Include the root — hierarchy auto-flows
den.aspects.igloo.includes = [ den.aspects.dev-tools ];
```

Three features working together:

1. **Natural nesting** via recursive freeform (`dev-tools.cli.bat`)
2. **Structural queries** via nfx request handlers (`hasAspect`, `onlyIf`)
3. **Clean separation** of children (freeform) and providers (`provides`)

## Feature 1: recursive freeform children

### What changes

The aspect submodule's freeform switches from `lazyAttrsOf deferredModule`
to `lazyAttrsOf (recursive aspectType)`. Class names are declared options
(via PR #449's `den.schema.aspect`), so they're captured before reaching
the freeform. Unknown keys become child aspects.

### Opt-in via flakeModule (non-breaking)

```nix
# flakeModules/recursiveAspects.nix
{ den, lib, ... }: {
  den.schema.aspect = { config, ... }: {
    # Override freeform: unknown keys become recursive child aspects
    _module.freeformType = lib.types.lazyAttrsOf (
      den.lib.aspects.types.aspectType {
        inherit (den.lib.aspects.types) defaultFunctor;
        providerPrefix = (config.meta.provider or []) ++ [ config.name ];
      }
    );

    # Auto-include children into the parent
    config.includes = builtins.attrValues (
      lib.filterAttrs (n: _: !(config._module.options ? ${n})) config
    );
  };
}
```

Users opt in:
```nix
imports = [
  inputs.den.flakeModule
  inputs.den.flakeModules.recursiveAspects
];
```

Default behavior unchanged. Existing configs work without modification.

### Identity plumbing

Children get `providerPrefix` from their parent, same mechanism
`provides` already uses. `dev-tools.cli.bat` has:

- `meta.provider = ["dev-tools" "cli"]`
- `name = "bat"`
- `aspectPath = ["dev-tools" "cli" "bat"]`

Same path whether declared as `dev-tools.cli.bat` (freeform child)
or `dev-tools._.cli._.bat` (provides child). Migration is identity-
preserving.

### Split files + parametric parents

Both work. The module system merges attribute paths across files.
A parametric function on the parent gets coerced to `{ includes = [fn] }`
and merges alongside the static freeform children.

```nix
# File A: parametric parent
den.aspects.dev-tools = { host, ... }: {
  nixos = lib.mkIf (host.name == "workstation") { ... };
};

# File B: static child
den.aspects.dev-tools.cli.bat.nixos = { ... };

# Both merge into one aspect. Function goes in includes,
# child goes in freeform.
```

Children inside parametric function bodies don't get the magic
(the function returns a raw attrset, not a submodule). That's fine:
static hierarchy + parametric selection is the right separation.

## Feature 2: structural effects via nfx

### hasAspect as a request

With nfx powering resolution (see nfx-resolution-spec), `hasAspect`
becomes a request ability instead of a post-hoc query:

```nix
# The request (used inside an aspect or class-config body)
hasAspect = ref: nfx.request "has-aspect" ref;

# The handler (established at the resolve root)
hasAspectHandler = pathSet:
  nfx.handle "has-aspect" (ref:
    nfx.pure (pathSet ? ${pathKey (aspectPath ref)})
  );
```

For class-config bodies (`nixos = { host, ... }: ...`), this is the
same as the current shipped `hasAspect` — a post-resolution query.
The nfx handler just answers from the precomputed structural set.

For includes-level decisions (the iii case), the handler can answer
during resolution because it controls evaluation order.

### onlyIf as an effect combinator

```nix
# Today: meta.adapter with raw tree walk
onlyIf = guard: target: {
  name = "onlyIf-${target.name or "<anon>"}";
  includes = [ target ];
  meta.adapter = inherited: args: ...raw collectPathsInner walk...;
};

# With nfx: effect combinator using the request system
onlyIf = guard: target:
  nfx.do [
    (_: nfx.request "has-aspect" guard)
    (present:
      if present
      then nfx.pure target
      else nfx.pure (tombstone target {})
    )
  ];
```

No raw tree walk. No bypassing filterIncludes. The `request` goes
to the structural-presence handler which was precomputed. The handler
answers. The combinator decides.

This is composable:

```nix
# Nested: include C only if both A and B are present
onlyIf <A> (onlyIf <B> <C>)

# oneOfAspects as a composed onlyIf
oneOfAspects = candidates:
  nfx.do [
    (_: nfx.request "structural-presence" {})
    (presence:
      let
        present = filter (c: presence ? ${pathKey (aspectPath c)}) candidates;
        winner = if present == [] then null else head present;
      in
      nfx.pure (map (c:
        if c == winner then c else tombstone c {}
      ) candidates)
    )
  ];
```

### The (iii) case: hasAspect inside includes

With nfx, this becomes possible via phased resolution:

```nix
den.aspects.impermanence = { host, ... }: {
  includes = [
    (onlyIf <zfs-root> <zfs-impermanence>)
    (onlyIf <btrfs-root> <btrfs-impermanence>)
  ];
};
```

Resolution phases:

1. **Structural pass.** Walk all aspects, collect declared includes
   (ignoring `onlyIf` guards). Build the structural presence set.
   This is the "what's declared" view.

2. **Effect pass.** Evaluate `onlyIf` effects against the structural
   set. Each `request "has-aspect"` gets answered by the handler.
   Conditional includes resolve to either the target or a tombstone.

3. **Module pass.** Walk the resolved tree, extract class modules,
   build the final NixOS/home-manager module.

The cycle breaks because the structural pass doesn't evaluate
conditional includes — it just notes their existence. The effect pass
uses the structural set (which is already complete) to answer queries.
No re-entry.

The handler implements this:

```nix
resolveWithEffects = class: aspect:
  let
    # Phase 1: structural presence (ignoring guards)
    structuralSet = collectStructuralPaths aspect;

    # Phase 2 + 3: evaluate effects and extract modules
    resolved = nfx.runFx (
      nfx.provide { inherit class structuralSet; }
        (nfx.via (hasAspectHandler structuralSet)
          (nfx.via (filterIncludesHandler moduleHandler)
            (resolveEffect class aspect)))
    );
  in
  resolved;
```

### Conditions for structural warnings

nfx's condition system (Common Lisp style resumable errors) maps
naturally to den's diagnostic needs:

```nix
# When a freeform child has no class config and no children
# (likely a typo), signal a condition
resolveChild = child:
  if isEmpty child
  then nfx.do [
    (_: nfx.signal {
      type = "empty-aspect";
      aspect = child;
      hint = "Did you mean to register a class? den.schema.aspect.options.${child.name} = ...";
    })
    (_: nfx.pure (tombstone child {}))
  ]
  else resolveEffect class child;

# In strict mode, the condition handler makes it an error:
strictHandler = nfx.handle "empty-aspect" (cond:
  nfx.throw "STRICT MODE: ${cond.hint}"
);

# In non-strict mode, it's a warning via trace:
lenientHandler = nfx.handle "empty-aspect" (cond:
  builtins.trace "warning: ${cond.hint}" (nfx.pure {})
);
```

This replaces den's current ad-hoc strict-mode error generation in
`nix/lib/strict.nix` with a composable condition system. Different
handlers produce different behaviors (error, warn, ignore) without
changing the resolution code.

## Feature 3: providers stay explicit

`provides` (aliased `_`) remains the home for cross-entity
contributions. It is NOT affected by the recursive freeform change.
Providers are consumed by:

- `mutual-provider.nix` via `from.aspect.provides.to-users`
- Forward rules via `fromAspect`
- Direct reference via `den.aspects.igloo.provides.alice`

The separation is:

- **Freeform children** = "part of me" (taxonomy, auto-included)
- **provides** = "I contribute to others" (cross-entity, consumed by name)

Users never need to think about which to use for taxonomy. Nested
attrs are children. Named contributions to other entities go in
`provides`.

## End-to-end example

A complete config showing all three features:

```nix
# modules/aspects/dev-tools.nix
{ den, ... }: {
  # Taxonomy — nested attrs, auto-included
  den.aspects.dev-tools.cli.bat.nixos = { pkgs, ... }: {
    environment.systemPackages = [ pkgs.bat ];
  };
  den.aspects.dev-tools.cli.ripgrep.nixos = { pkgs, ... }: {
    environment.systemPackages = [ pkgs.ripgrep ];
  };
  den.aspects.dev-tools.cli.fd.nixos = { pkgs, ... }: {
    environment.systemPackages = [ pkgs.fd ];
  };
  den.aspects.dev-tools.gui.alacritty.homeManager.programs.alacritty.enable = true;
}

# modules/aspects/impermanence.nix
{ den, lib, ... }: {
  # Conditional includes — onlyIf is an effect, not cyclic
  den.aspects.impermanence.includes = [
    (den.lib.onlyIf <zfs-root> <zfs-impermanence>)
    (den.lib.onlyIf <btrfs-root> <btrfs-impermanence>)
  ];

  den.aspects.zfs-impermanence.nixos = { ... };
  den.aspects.btrfs-impermanence.nixos = { ... };
}

# modules/aspects/monitoring.nix
{ den, lib, ... }: {
  # Structural query from class-config body
  den.aspects.monitoring.nixos = { host, ... }: lib.mkMerge [
    { services.prometheus.enable = true; }
    (lib.mkIf (host.hasAspect <loki>) {
      services.promtail.enable = true;
    })
  ];
}

# modules/aspects/secrets.nix
{ den, ... }: {
  # Provider-style structural decision
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
      den.aspects.dev-tools      # whole taxonomy auto-flows
      den.aspects.impermanence   # conditional on zfs/btrfs
      den.aspects.monitoring
      den.aspects.secrets
    ];

    # Provider: contributes to users
    provides.to-users = { user, ... }: {
      homeManager.programs.git.userName = user.name;
    };
  };
}

# flake.nix
{
  inputs.den.url = "github:vic/den";
  inputs.nfx.url = "github:vic/nfx";

  outputs = { den, ... }: den.lib.mkFlake {
    imports = [
      den.flakeModule
      den.flakeModules.recursiveAspects  # opt-in: freeform = children
    ];

    den.hosts.x86_64-linux.igloo.users.tux = {};
  };
}
```

## Implementation order

### Phase 1: recursive freeform (no nfx, lands first)

- Lift class registrations from strict-only to unconditional (PR #449 adjustment)
- Add `recursiveAspects` flakeModule that overrides the aspect freeform
- Add auto-include of freeform children
- Ship `onlyIf` adapter (current meta.adapter approach, no nfx)
- Tests, template examples, docs

This is achievable with the current architecture. No nfx dependency.
Users get natural nesting and onlyIf today.

### Phase 2: nfx-powered resolution (Vic's exploration)

- Add nfx as a den input
- Rewrite resolve.nix, adapters.nix, ctx-apply.nix using nfx
- Convert adapters to nfx handlers
- Convert structural queries (hasAspect, collectPaths) to request handlers
- Verify all existing tests pass (API unchanged, internals replaced)

This is the resolution rewrite. User-facing API stays. Internal
machinery gets smaller and more composable.

### Phase 3: structural effects (builds on Phase 2)

- Implement phased resolution (structural pass + effect pass)
- hasAspect becomes a request during resolution (enables iii)
- onlyIf migrates from meta.adapter to effect combinator
- Condition system for structural diagnostics (strict mode)
- The (iii) rejection lifts: hasAspect in includes works

This is where the nfx architecture pays off. The features that were
architecturally impossible become natural.

## What each phase delivers to users

| Phase | User gets |
|---|---|
| 1 | `dev-tools.cli.bat`, auto-include, onlyIf (adapter-based) |
| 2 | Same features, cleaner internals, foundation for Phase 3 |
| 3 | hasAspect in includes, onlyIf as pure combinator, structural conditions |

Each phase is independently shippable and valuable. Phase 1 needs no
nfx. Phase 2 is a refactor (no new features). Phase 3 is the payoff
that justifies Phase 2.

## Relationship to existing work

- **hasAspect (shipped):** stays as-is in Phase 1. Becomes a request
  handler in Phase 2. Gains includes-level usage in Phase 3.

- **oneOfAspects (shipped):** stays as-is in Phase 1. Becomes a
  handler in Phase 2. Composition simplifies in Phase 3.

- **onlyIf (next PR):** ships as a meta.adapter in Phase 1. Migrates
  to an effect combinator in Phase 3. API stays the same either way.

- **collectPaths (shipped):** stays as-is in Phase 1. Becomes the
  implementation of the structural-presence handler in Phase 2.

- **PR #449 strict mode:** prerequisite for all phases. Class
  registration enables the freeform change. Strict diagnostics
  migrate to nfx conditions in Phase 3.

Nothing we've built is wasted. Each piece evolves in place.
