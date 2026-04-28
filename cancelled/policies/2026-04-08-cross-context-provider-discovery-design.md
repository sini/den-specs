# Cross-Context Provider Discovery

**Date:** 2026-04-08
**Branch:** TBD (on top of sf5/forward-consumption)
**Status:** Design approved, pending implementation

## Summary

Two new adapters (`detectProvider`, `collectProviders`) and an entity-level
`hasProvider` helper that enable cross-context provider discovery. These
compose with existing `den._.forward` to express cross-host forwarding
patterns declaratively.

## Goals

- Detect whether a provider exists in another entity's resolved aspect tree
- Collect all providers matching a name across entities with metadata
- Provide a convenient `host.hasProvider "name"` predicate on entities
- Demonstrate the pattern with practical examples (prometheus, hosts file,
  SSH known_hosts, primary-user homeManager)

## Non-Goals

- No changes to forwards, contexts, or schemas
- No new `into` transitions
- No self-reference handling (cross-context only; self uses normal includes)
- No changes to existing batteries

## Components

### detectProvider adapter

A reusable adapter for `resolve.withAdapter` that detects if a named provider
exists anywhere in an aspect tree.

```nix
# In nix/lib/aspects/adapters.nix
detectProvider = name: { aspect, recurse, ... }: {
  found = (aspect.provides or {}) ? ${name}
    || lib.any (i: (recurse i).found or false) (aspect.includes or []);
};
```

Usage:
```nix
(resolve.withAdapter (adapters.detectProvider "prometheus-exporter") host.class
  (den.ctx.host { host = otherHost; })).found
```

### collectProviders adapter

A reusable adapter that collects all providers matching a name, returning
each with the providing aspect's name, the provider value, and the
aspect's meta (for accessing metadata like ports, keys, etc.).

```nix
# In nix/lib/aspects/adapters.nix
collectProviders = name: { aspect, recurse, ... }:
  let
    self = lib.optional ((aspect.provides or {}) ? ${name}) {
      name = aspect.name or "<unknown>";
      meta = aspect.meta or {};
      provider = aspect.provides.${name};
    };
    children = lib.concatMap (i: (recurse i).collected or []) (aspect.includes or []);
  in
  { collected = self ++ children; };
```

Usage:
```nix
(resolve.withAdapter (adapters.collectProviders "prometheus-exporter") host.class
  (den.ctx.host { host = otherHost; })).collected
# Returns: [{ name = "postgres"; meta = { scrapePort = 9187; }; provider = <aspect>; } ...]
```

**Note on provider visibility:** The root context result from `ctxApply` does
not carry `provides` — it returns `{ name, __provider, includes }`. However,
the individual aspects in the includes list (produced by `buildIncludes` /
parametric wrapping) ARE full aspects with `provides`. The adapters find
providers on these child aspects, not the root.

### hasProvider on entity types

A convenience function on host and home entity types that wraps
`detectProvider`. Defined as a proper module option.

```nix
# On hostType in nix/lib/types.nix
hasProvider = lib.mkOption {
  internal = true;
  visible = false;
  type = lib.types.functionTo lib.types.bool;
  default = name:
    (den.lib.aspects.resolve.withAdapter
      (den.lib.aspects.adapters.detectProvider name)
      config.class
      (den.ctx.host { host = config; })
    ).found or false;
};
```

Usage:
```nix
otherHost.hasProvider "prometheus-exporter"  # => true/false
```

**Circular reference note:** `hasProvider` resolves the target entity's
context. Two types of cycles can occur:

1. **Self-reference:** Host A's aspect calls `hostA.hasProvider` during
   hostA's own resolution. Always cycles. Use normal includes for self.
2. **Mutual reference:** Host A queries host B, and host B queries host A.
   This cycles because evaluating A forces B, which forces A. Avoid mutual
   `hasProvider` calls. Design patterns should have a clear direction
   (e.g., collector queries sources, not vice versa).

## Cross-Context Forward Pattern

The two-forward pattern for cross-context provider collection:

**Self-context forward** (existing pattern):
```nix
den.aspects.monitoring.includes = [ den.aspects.prometheus-collector ];
# Own exporters already in resolution through normal includes
```

**Cross-context forward** (new pattern using these utilities):
```nix
den.aspects.prometheus-collector = { host }:
  let
    otherHosts = lib.filter (h: h != host) (lib.attrValues den.hosts.${host.system});
    hostsWithExporters = lib.filter (h: h.hasProvider "prometheus-exporter") otherHosts;
  in
  den._.forward {
    each = hostsWithExporters;
    fromAspect = srcHost: den.ctx.host { host = srcHost; };
    fromClass = _: "prometheus-exporter";
    intoClass = _: host.class;
    intoPath = _: [ "services" "prometheus" "scrapeConfigs" ];
  };
```

No cycle — each `den.ctx.host { host = srcHost; }` evaluates an independent
host's context.

## Example Patterns

### /etc/hosts from host metadata

```nix
den.aspects.fleet-hosts = { host }:
  let
    otherHosts = lib.filter (h: h != host) (lib.attrValues den.hosts.${host.system});
    hostsWithNet = lib.filter (h: h.hasProvider "network-identity") otherHosts;
    collectNet = srcHost:
      (den.lib.aspects.resolve.withAdapter
        (den.lib.aspects.adapters.collectProviders "network-identity")
        srcHost.class
        (den.ctx.host { host = srcHost; })
      ).collected;
  in {
    nixos.networking.hosts = lib.mkMerge (map (srcHost:
      let nets = collectNet srcHost;
      in lib.listToAttrs (map (n: {
        name = n.meta.ip or "127.0.0.1";
        value = [ srcHost.hostName ];
      }) nets)
    ) hostsWithNet);
  };
```

### SSH known_hosts

```nix
den.aspects.fleet-ssh = { host }:
  let
    otherHosts = lib.filter (h: h != host) (lib.attrValues den.hosts.${host.system});
    hostsWithKeys = lib.filter (h: h.hasProvider "ssh-host-key") otherHosts;
    collectKeys = srcHost:
      (den.lib.aspects.resolve.withAdapter
        (den.lib.aspects.adapters.collectProviders "ssh-host-key")
        srcHost.class
        (den.ctx.host { host = srcHost; })
      ).collected;
  in {
    nixos.services.openssh.knownHosts = lib.listToAttrs (lib.concatMap (srcHost:
      let keys = collectKeys srcHost;
      in lib.optional (keys != []) {
        name = srcHost.hostName;
        value = {
          hostNames = [ srcHost.hostName ];
          publicKey = (lib.head keys).meta.publicKey or "";
        };
      }
    ) hostsWithKeys);
  };
```

### Host homeManager aspects to primary user

```nix
# Aspect that auto-forwards all host-level homeManager content to primary user
den.aspects.auto-hm-to-primary = { host }:
  let
    primaryUser = lib.findFirst (u: u.isPrimary or false) null (lib.attrValues host.users);
  in
  lib.optionalAttrs (primaryUser != null) (
    den._.forward {
      each = lib.singleton true;
      fromAspect = _: den.ctx.host { inherit host; };
      fromClass = _: "homeManager";
      intoClass = _: host.class;
      intoPath = _: [ "home-manager" "users" primaryUser.userName ];
    }
  );
```

## Files Changed

| File | Change |
|------|--------|
| `nix/lib/aspects/adapters.nix` | Add `detectProvider`, `collectProviders` |
| `nix/lib/types.nix` | Add `hasProvider` to hostType, homeType |
| `templates/ci/modules/features/cross-context-providers.nix` | Tests and examples |

## Relationship to Prior Work

This builds on the adapter machinery from the context-aspect-resolution
feature (SF1-SF5). Specifically:

- `resolve.withAdapter` (SF2) enables custom resolution adapters
- `meta` on aspects (existing) carries provider metadata
- `provides` / `_` namespace (existing) defines providers on aspects
- Cross-host context evaluation is safe (independent resolution paths)
