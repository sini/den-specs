# Cross-Context Forwards via Service Discovery

**Date:** 2026-04-08
**Branch:** TBD (on top of sf5/forward-consumption)
**Status:** Design approved, pending implementation

## Summary

Three small extensions to Den's forward primitive that enable cross-context
service discovery: entity `.ctx` for automatic context resolution, implicit
`fromAspect` via `.ctx`, empty-resolution skipping, and `sourceGuard` for
entity filtering. Combined, these let forwards declaratively collect class
content from any set of entities and inject it into the current context.

## Changes

### 1. `entity.ctx` on entity types

Each entity type gains a `.ctx` attribute that resolves its context.

In `nix/lib/types.nix`, on hostType:
```nix
config.ctx = den.ctx.host { host = config; };
```

On homeType:
```nix
config.ctx = den.ctx.home { home = config; };
```

~1 line per entity type.

### 2. Default `fromAspect` to `item.ctx`

In `nix/lib/forward.nix`, when `fromAspect` is not provided and the item
has `.ctx`, default to using it.

```nix
asp = if fwd ? fromAspect then fwd.fromAspect item else item.ctx or item;
```

~1 line change in forwardItem.

### 3. Skip empty resolutions

In `nix/lib/forward.nix`, when `resolve fromClass asp` produces a module
with no imports, the forward item produces nothing. This is implicit source
discovery — entities without the requested class are silently skipped.

In `forwardEach`, filter out items where resolve produces empty imports:
```nix
filteredEach = if fwd ? sourceGuard
  then lib.filter fwd.sourceGuard fwd.each
  else fwd.each;
```

And in `forwardItem`, the existing forward/topLevelAdapter/adapter paths
already produce attrsets — empty imports result in empty modules that the
NixOS module system handles gracefully. No explicit skip needed if the
module system tolerates empty imports.

### 4. `sourceGuard` on forward

In `nix/lib/forward.nix`, `forwardEach` filters `each` through
`sourceGuard` before iteration.

Current:
```nix
forwardEach = fwd: {
  includes = map (item: forwardItem (fwd // { inherit item; })) fwd.each;
};
```

Change to:
```nix
forwardEach = fwd:
  let
    items = if fwd ? sourceGuard
      then lib.filter fwd.sourceGuard fwd.each
      else fwd.each;
  in {
    includes = map (item: forwardItem (fwd // { inherit item; })) items;
  };
```

~3 lines changed.

### 5. `aspect.name or "<anon>"` in resolve.nix

Already fixed in SF5. Prevents crash when resolve encounters aspects
without `name` (e.g., context-applied results from cross-context forwards).

## Usage Patterns

### SSH known_hosts (fleet-wide, in den.default)

Provider aspect on each host:
```nix
den.aspects.igloo._.ssh-host-key = {
  ssh-host-key.publicKey = "ssh-ed25519 AAAA igloo";
  ssh-host-key.hostNames = [ "igloo" "igloo.local" ];
};
```

Fleet-wide forward:
```nix
den.default.includes = [
  (den.lib.perHost ({ host }:
    den._.forward {
      each = lib.attrValues den.hosts.${host.system};
      fromClass = _: "ssh-host-key";
      intoClass = _: host.class;
      sourceGuard = src: src != host;
    }
  ))
];
```

### Prometheus exporters (consumer-specific)

Provider:
```nix
den.aspects.postgres._.prometheus-exporter = {
  prometheus-exporter.scrapeConfig = {
    job_name = "postgres";
    static_configs = [{ targets = [ "localhost:9187" ]; }];
  };
};
```

Consumer (only hosts with this aspect get the forward):
```nix
den.aspects.prometheus-collector = { host }:
  den._.forward {
    each = lib.attrValues den.hosts.${host.system};
    fromClass = _: "prometheus-exporter";
    intoClass = _: host.class;
    intoPath = _: [ "services" "prometheus" ];
    sourceGuard = src: src != host;
  };
```

### Kubernetes cluster service

```nix
den.aspects.rook-ceph = { cluster }:
  den._.forward {
    each = cluster.members;
    fromClass = _: "rook-disk-config";
    intoClass = _: "kubernetes";
    intoPath = _: [ "services" "rook-ceph" "disks" ];
  };
```

### Primary user homeManager forwarding

```nix
den.aspects.auto-hm-primary = { host }:
  let primary = lib.findFirst (u: u.isPrimary or false) null
    (lib.attrValues host.users);
  in lib.optionalAttrs (primary != null) (
    den._.forward {
      each = lib.singleton host;
      fromClass = _: "homeManager";
      intoClass = _: host.class;
      intoPath = _: [ "home-manager" "users" primary.userName ];
    }
  );
```

## Files Changed

| File | Change | Lines |
|------|--------|-------|
| `nix/lib/types.nix` | Add `.ctx` to hostType and homeType | ~2 |
| `nix/lib/forward.nix` | Default fromAspect, sourceGuard in forwardEach | ~8 |
| `nix/lib/aspects/resolve.nix` | `aspect.name or "<anon>"` (done in SF5) | ~1 |
| `templates/ci/modules/features/cross-context-forward.nix` | Tests | ~80 |

## Circular Reference Considerations

`entity.ctx` resolves through the context pipeline. Accessing `host.ctx`
from within that host's own resolution would cycle. This is safe because:

- `sourceGuard = src: src != host` excludes self from cross-context forwards
- Self-context forwarding (like primary-user HM) uses `each = singleton host`
  with `fromAspect` implicit via `.ctx` — but this host IS the current host,
  so it would cycle. For self-context, use the existing `fromAspect` pattern
  that resolves through `parametric.fixedTo` instead of `.ctx`.

Self-context forwards should use explicit `fromAspect`:
```nix
fromAspect = _: den.lib.parametric.fixedTo { inherit host; } den.aspects.${host.aspect};
```

Cross-context forwards use implicit `.ctx`:
```nix
each = otherHosts;  # .ctx used automatically
```
