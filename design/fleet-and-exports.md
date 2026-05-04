# Fleet + den.exports: Cross-Host Data Pattern

**Date:** 2026-05-04
**Status:** Planned (not yet implemented)
**Dependencies:** None — can land independently of traits reimplementation, provides removal, or any other pending work

---

## 1. The Problem

NixOS configurations frequently need data from other hosts in the fleet:

- `/etc/hosts` needs IP-to-hostname mappings from all machines
- HAProxy backends need `{ addr, port }` from each upstream service
- Backup orchestration needs persistence paths and retention policies from targets
- SSH `known_hosts` needs public host keys from the fleet
- Firewall rules need ports exposed by peer services

This data is **NixOS-config-dependent** — it only exists after module evaluation. A service's port comes from `config.services.foo.port`, persistence paths come from `config.den.exports.impermanence.paths`, etc.

The den pipeline operates *before* NixOS evaluation. It routes aspects into class modules and resolves policy topology, but it never has access to evaluated `config` values. The pipeline can gate *whether* an aspect is included (via boolean trait flags), but it cannot aggregate *what* a service exports (ports, paths, keys).

The deleted three-tier trait system attempted to solve this by deferring Tier 3 traits to module evaluation time, injecting them via `_den.traits` modules, and distributing cross-entity data through provide-to sub-pipelines. This worked but required ~2463 lines across 23 files, interacted with every pipeline concern, and was architecturally fragile.

The insight: Nix already solves lazy cross-reference evaluation. `nixosConfigurations` is a lazy attrset — accessing one host's config does not force evaluation of another unless there is a true data dependency. We can expose this directly.

---

## 2. fleet: Lazy Cross-Host Access

### Definition

`fleet` is a lazy attrset mapping host names to their fully evaluated NixOS configs. It is injected as `_module.args.fleet` in the flake-level module evaluation, making it available to any NixOS module via standard function arguments.

### Where it lives

```nix
# modules/policies/flake.nix — add to flake-system policy or as a standalone flake module
_module.args.fleet = lib.mapAttrs (_: sys: sys.config) config.flake.nixosConfigurations;
```

This single line makes every host's evaluated config available to every other host. Because `lib.mapAttrs` produces a lazy attrset and NixOS configurations are themselves lazy, no host is evaluated until explicitly accessed.

### Usage in class modules

```nix
# Any den aspect or class module (flat-form):
{ fleet, host, lib, ... }: {
  networking.hosts = lib.mkMerge (lib.mapAttrsToList
    (name: cfg: cfg.den.exports.hostsEntries or {})
    (lib.filterAttrs (n: _: n != host.name) fleet));
}
```

The `fleet` argument is a plain NixOS module argument — no den pipeline involvement, no special effect, no trait machinery. It is available everywhere that `config` is available.

---

## 3. den.exports: Uniform Export Interface

### The problem it solves

Without a uniform export interface, reading cross-host data requires knowing the exact option path on the remote host (`fleet.webserver.services.nginx.virtualHosts`). This is brittle — option paths change between services, and some data (like "all ports this host exposes to the network") has no single canonical location.

### Definition

`den.exports` is a freeform option on every host, providing a flat namespace for declaring exportable data:

```nix
# Injected into every host's mainModule (nix/lib/aspects/fx/pipeline.nix, in instantiateArgs)
options.den.exports = lib.mkOption {
  type = lib.types.lazyAttrsOf lib.types.anything;
  default = {};
  description = "Data exported to other fleet members via the fleet attrset.";
};
```

### Usage — declaring exports

Any module on any host can write to `den.exports`:

```nix
# aspects/services/nginx.nix (class module)
{ config, ... }: {
  den.exports.hostsEntries = {
    ${config.networking.hostName} = config.networking.defaultGateway.address or "127.0.0.1";
  };
  den.exports.httpBackends = [{
    addr = config.networking.hostName;
    port = config.services.nginx.defaultHTTPListenPort;
  }];
}
```

### Usage — reading exports

Any module on any host reads via `fleet`:

```nix
{ fleet, lib, ... }: {
  services.haproxy.backends = lib.concatMap
    (cfg: cfg.den.exports.httpBackends or [])
    (lib.attrValues fleet);
}
```

### Why freeform

- No schema registration required — any module can write to any key
- `default = {}` means unset keys return `{}` (or use `or []` / `or {}` at read sites)
- No import dependency — a host without nginx simply has no `den.exports.httpBackends`
- No pipeline involvement — purely module-evaluation-time

---

## 4. Cycle Avoidance

### The structural constraint

`fleet` introduces a potential for infinite recursion: if host A's config depends on host B's config, and host B's config depends on host A's config, Nix will loop.

Formally: the evaluation graph of `fleet` references must be a DAG. No circular evaluation paths.

### Why it is safe in practice

The data exported via `den.exports` is overwhelmingly **self-derived**:

| Export | Source | Cross-host? |
|--------|--------|-------------|
| Service ports | `config.services.foo.port` | No — host's own config |
| Persistence paths | `config.environment.persistence.*.directories` | No — host's own config |
| Host keys | `config.services.openssh.hostKeys` | No — host's own config |
| IP addresses | `config.networking.interfaces.*.ipv4` | No — host's own config |
| Backup retention | `config.services.borgbackup.repos.*.keep` | No — host's own config |

A host's exports depend only on its own service definitions. Reading another host's exports does not cause that host to read back.

### What would break

```nix
# HOST A
{ fleet, ... }: {
  den.exports.myPort = fleet.hostB.den.exports.theirPort + 1;  # CYCLE if B does the same
}

# HOST B
{ fleet, ... }: {
  den.exports.theirPort = fleet.hostA.den.exports.myPort + 1;  # infinite recursion
}
```

Mutual dependencies where both hosts read each other's exports to compute their own exports will infinite-recurse. This is the same constraint as any `let rec` or fixpoint in Nix — not a new risk introduced by `fleet`, just made explicit.

### Diagnosis

When a cycle occurs, Nix reports `infinite recursion encountered` with a stack trace showing the evaluation path. The fix is always: ensure the export value is derived from the host's own config, not from reading another host's export of the same key.

---

## 5. Relationship to Pipeline-Time Traits

Fleet and traits are complementary systems operating at different evaluation stages:

| Concern | Mechanism | Evaluation time | Example |
|---------|-----------|----------------|---------|
| Routing decisions | Pipeline-time trait flags | Before NixOS eval | `impermanence = true` gates inclusion of persistence aspects |
| Cross-host data | `fleet` + `den.exports` | During NixOS eval | Collecting backup targets from all fleet members |

### Pipeline-time traits (future reimplementation)

Simple boolean/enum flags registered in `den.traits`. The pipeline's `classifyKeys` dispatches them to a trait branch. They gate policy routing:

```nix
# Policy fires only for hosts with impermanence trait
{ host, impermanence, ... }:
  lib.optional impermanence (policy.include den.aspects.persist);
```

These are lightweight — no collection, no cross-host aggregation, no NixOS config access. They answer "does this entity have capability X?" for routing purposes.

### Fleet + den.exports (this design)

Handles the data aggregation that the old Tier 3 traits attempted. Instead of a custom distribution phase in the pipeline, it uses Nix's lazy evaluation directly. The pipeline never sees this data — it flows between evaluated NixOS configs at module time.

### When to use which

- "Should this aspect be included?" — pipeline-time trait flag
- "What ports does this host expose?" — `den.exports` + `fleet`
- "Does this host have a desktop environment?" — pipeline-time trait flag
- "What are the SSH host keys of all machines?" — `den.exports` + `fleet`

---

## 6. Concrete Examples

### /etc/hosts generation

```nix
# aspects/networking/hosts-file.nix — class module
{ config, ... }: {
  # Export: each host declares its own entries
  den.exports.hostsEntries = {
    ${config.networking.hostName} = [
      config.deployment.targetHost or "127.0.0.1"
    ];
  };
}
```

```nix
# aspects/networking/fleet-hosts.nix — class module
{ fleet, host, lib, config, ... }: {
  networking.hosts = lib.mkMerge (lib.mapAttrsToList (name: cfg:
    let entries = cfg.den.exports.hostsEntries or {};
    in lib.mapAttrs (_: addrs: addrs) entries
  ) (lib.filterAttrs (n: _: n != host.name) fleet));
}
```

### HAProxy backend registration

```nix
# aspects/services/web-backend.nix — class module on backend hosts
{ config, ... }: {
  den.exports.httpBackends = [{
    name = config.networking.hostName;
    addr = config.networking.hostName;
    port = config.services.nginx.defaultHTTPListenPort;
    healthCheck = "/health";
  }];
}
```

```nix
# aspects/services/haproxy.nix — class module on the load balancer
{ fleet, lib, ... }: {
  services.haproxy.config = let
    backends = lib.concatMap (cfg: cfg.den.exports.httpBackends or [])
      (lib.attrValues fleet);
  in ''
    backend web
      balance roundrobin
      ${lib.concatMapStringsSep "\n    " (b:
        "server ${b.name} ${b.addr}:${toString b.port} check"
      ) backends}
  '';
}
```

### Backup target collection

```nix
# aspects/services/borgbackup-target.nix — class module on hosts with backups
{ config, ... }: {
  den.exports.backupTargets = [{
    host = config.networking.hostName;
    paths = config.den.exports.impermanence.paths or [ "/persist" ];
    user = "borg";
    repo = "/var/lib/borg/${config.networking.hostName}";
  }];
}
```

```nix
# aspects/services/borgbackup-server.nix — class module on the backup server
{ fleet, lib, config, ... }: {
  services.borgbackup.repos = lib.listToAttrs (lib.concatMap (cfg:
    map (target: lib.nameValuePair target.host {
      path = target.repo;
      authorizedKeys = [ (cfg.den.exports.borgPublicKey or "") ];
    }) (cfg.den.exports.backupTargets or [])
  ) (lib.attrValues fleet));
}
```

### SSH known_hosts

```nix
# aspects/security/ssh-host-keys.nix — class module on all hosts
{ config, ... }: {
  den.exports.sshHostKeys = map (key: {
    hostNames = [ config.networking.hostName "${config.networking.hostName}.local" ];
    publicKey = builtins.readFile key.path;
  }) (builtins.filter (k: k.type == "ed25519") config.services.openssh.hostKeys);
}
```

```nix
# aspects/security/known-hosts.nix — class module on all hosts
{ fleet, lib, ... }: {
  programs.ssh.knownHosts = lib.listToAttrs (lib.concatMap (cfg:
    map (key: lib.nameValuePair (builtins.head key.hostNames) {
      inherit (key) hostNames publicKey;
    }) (cfg.den.exports.sshHostKeys or [])
  ) (lib.attrValues fleet));
}
```

---

## 7. Why This Replaces provide-to and Tier 3 Traits

The original cross-host data flow in den used:

1. **provide-to effects** — emit data tagged with a target entity during pipeline walk
2. **distribute-cross-entity** — post-pipeline phase collecting cross-entity emissions and injecting them into target configs
3. **Tier 3 traits** — trait values that depend on NixOS config, deferred to module evaluation, distributed via `_den.traits` injection modules

All three mechanisms existed to solve one problem: getting data from one host's evaluated config into another host's NixOS modules.

`fleet` + `den.exports` replaces all three with a single, simpler pattern:

| Old mechanism | Lines of code | Replacement |
|---------------|---------------|-------------|
| provide-to handler + distribute phase | ~400 | `fleet` (one line) |
| Tier 3 trait evaluation + deferred injection | ~800 | `den.exports` (one option) |
| Three-tier classification in classifyKeys | ~200 | Eliminated — no tier distinction needed |
| Trait inheritance via scope walking | ~300 | Direct `fleet` reads |
| `traitModuleForScope` injection | ~150 | Standard NixOS module arguments |

**Why it is simpler:**

- No custom distribution phase — Nix laziness handles it
- No pipeline involvement — the pipeline routes aspects, `fleet` handles data
- No schema registration — freeform option, write anything
- No deferred evaluation — values are just NixOS config values, evaluated when accessed
- No trait inheritance semantics — each host reads what it needs directly
- Debugging is standard Nix debugging — `nix eval`, `nix repl`, standard option traces

---

## 8. Implementation

### Required changes

**One line in flake policy** (`modules/policies/flake.nix`):

```nix
# Add to an existing policy module, or as a new top-level flake module:
_module.args.fleet = lib.mapAttrs (_: sys: sys.config) config.flake.nixosConfigurations;
```

This injects `fleet` as a module argument available to all NixOS modules evaluated through den.

**One option in mainModule** (injected via `nix/lib/aspects/fx/pipeline.nix` in the instantiate path, or via a dedicated module in the entity's `mainModule`):

```nix
# Added to the module list passed to nixos-lib.evalModules / home-manager:
{ lib, ... }: {
  options.den.exports = lib.mkOption {
    type = lib.types.lazyAttrsOf lib.types.anything;
    default = {};
    description = "Data exported to other fleet members via the fleet attrset.";
  };
}
```

The injection point is `pipeline.nix` line ~466 where `entity.mainModule` is passed to `entity.instantiate`. The exports option module would be prepended to the modules list:

```nix
modules = [ exportsModule entity.mainModule ];
```

Alternatively, it can live in `den.schema.host` as a schema-level include, ensuring all hosts get it regardless of instantiation path.

### No migration needed

This is purely additive. No existing module or aspect changes behavior. Hosts that don't write to `den.exports` simply export `{}`. Hosts that don't read `fleet` are unaffected.

---

## 9. Home Manager Fleet Equivalent

### The pattern

Home Manager users within a single host share evaluated config via the host's module system. Cross-user data is less common than cross-host data, but the same pattern applies:

```nix
# Injected into home-manager module evaluation:
_module.args.users = lib.mapAttrs (_: hm: hm.config) config.home-manager.users;
```

This gives each user's Home Manager config access to other users' configs on the same host.

### den.exports for Home Manager

```nix
# In the home-manager mainModule:
{ lib, ... }: {
  options.den.exports = lib.mkOption {
    type = lib.types.lazyAttrsOf lib.types.anything;
    default = {};
    description = "Data exported to other users or host-level modules.";
  };
}
```

### Cross-boundary access (user reads host fleet)

When `useGlobalPkgs` is active and Home Manager is evaluated inside the host's module system, `fleet` is already available as a module argument. Users can read cross-host data directly:

```nix
# Home Manager module that reads fleet data:
{ fleet, lib, ... }: {
  programs.ssh.matchBlocks = lib.listToAttrs (lib.mapAttrsToList (name: cfg:
    lib.nameValuePair name { hostname = builtins.head (cfg.den.exports.sshAddresses or [ name ]); }
  ) fleet);
}
```

### When useGlobalPkgs is not active

If Home Manager evaluates independently (standalone mode), `fleet` must be threaded through as a `specialArgs` or `extraSpecialArgs` value during home-manager instantiation. The `policy.instantiate` path in den already controls instantiation arguments — adding `fleet` there is straightforward.

---

## Summary

| Component | What | Where | Lines changed |
|-----------|------|-------|---------------|
| `fleet` | Lazy attrset of evaluated host configs | `modules/policies/flake.nix` | 1 |
| `den.exports` | Freeform option for exportable data | Entity mainModule injection | ~6 |
| Total | Cross-host data without pipeline involvement | | ~7 lines of framework code |

The pattern leverages Nix's existing laziness guarantees rather than building custom distribution machinery. It is the natural replacement for the deleted Tier 3 trait system and provide-to mechanism, trading ~2800 lines of pipeline-integrated code for ~7 lines of module-system integration.
