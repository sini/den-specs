# Pipes and Quirks: Unified Data Flow

**Date:** 2026-05-05
**Status:** Design
**Branch:** feat/fx-pipeline (713/713 tests)
**Supersedes:** `design/traits.md`, `design/fleet-and-exports.md`, `tbd/emission-pipeline.md`

---

## 1. Problem

Aspects emit content that falls into three semantic categories:

- **Class modules** (`nixos`, `homeManager`) — functions evaluated by an external module system, becoming part of `config`
- **Structured data** (`firewall`, `http-backends`, `secrets`) — data collected across aspects and delivered to consumers
- **Nested aspects** (`tux`, `node-exporter`) — sub-aspects scoped to a named entity or feature

Today, all non-structural keys emit as class modules. There is no mechanism to collect structured data across aspects and deliver it to consumers. An nginx aspect and a postgres aspect both know what firewall ports they need, but there is no way to aggregate that data and hand it to a firewall-configuration aspect. Each must independently set `networking.firewall.allowedTCPPorts` in a class module, with no coordination.

### What the deleted trait system attempted

The original trait implementation (2463 lines, deleted) solved this with three evaluation tiers, a DLQ, deferred thunk management, provide-to sub-pipelines, and `_den.traits` injection modules. It was architecturally fragile — every pipeline concern interacted with it.

### What changed

The pipeline rewrite introduced:
- Dynamic context expansion (`push-scope`, `widen-context`, `enterScope`)
- `bind` resolving against live state handlers
- Scope-partitioned state with parent inheritance
- All hosts as subgraphs of one pipeline run

Most critically: **unregistered class keys are already collected into `scopedClassImports` and sit inert unless a post-pipeline phase reads them.** This is the same mechanism forwards/routes use. Pipes reuse it.

---

## 2. Concepts

### Quirk

Structured data emitted by an aspect on a named key. The aspect doesn't know or care whether a pipe exists for that key — it just declares its data:

```nix
den.aspects.nginx = {
  nixos.services.nginx.enable = true;       # class module
  firewall = { ports = [ 80 443 ]; };      # quirk
};
```

A quirk can be:
- A plain value (available at pipeline-time)
- Parametric on den context (`{ host, ... }: { ports = [...]; }` — resolved during walk)
- A config-dependent thunk (`{ config, ... }: { port = config.services.nginx.port; }` — collected as-is, resolved lazily at eval time)

### Pipe

A declared data route. Establishes that a key name is a pipe, not a class:

```nix
den.pipes.firewall = {
  description = "Firewall port declarations";
};
```

Top-level, declared by consumers. Complete before pipeline starts. The declaration is minimal — just metadata. No schema validation, no collection strategy. Quirks are always collected as a list; consumers transform however they want.

### Policy pipe builder

A pipe is constructed as a sequence of stages with an optional destination. The policy's own context (`{ host, user, ... }:`) determines WHERE and WHEN the pipe applies. The stages describe WHAT happens to the data. The optional destination narrows WHO receives it.

```nix
den.policies.user-secret-filter = { host, user, ... }:
  let inherit (den.lib.policy) pipe; in [
    (pipe.from den.pipes.secrets [
      (pipe.filter (s: builtins.elem user.name (s.users or [])))
      (pipe.transform (s: removeAttrs s [ "users" ]))
      (pipe.to [ den.aspects.postgres den.aspects.nginx ])
    ])
  ];
```

Three orthogonal concerns:
- **Where/when** — the policy's arg signature (fires at a scope boundary with specific context)
- **What** — the pipe stages (filter, transform, fold, append)
- **Who** — the optional `pipe.to` destination (narrows receiver within the scope)

---

## 3. Three Roles

| Role | Writes | Knows about |
|------|--------|-------------|
| **Producer** (aspect author) | Quirk data on a key | Nothing — just puts data on a key |
| **Architect** (relationship author) | Pipe declaration + policy pipe builder | Which keys are pipes, how data flows between scopes |
| **Consumer** (aspect author) | `{ firewall, ... }:` in signature | Pipe name — receives data as a function arg |

The producer doesn't know about pipes. The consumer doesn't know about routing. The policy describes the relationship.

---

## 4. Scope Model

All hosts are subgraphs of one pipeline run:

```
flake (root scope)
├── host:igloo (scope)
│   ├── user:tux (scope)
│   └── user:pingu (scope)
├── host:iceberg (scope)
│   └── user:alice (scope)
```

### Visibility

Quirks are **scope-local by default**. Data stays in the scope where it was emitted. A host's quirks are visible to consumers within that host's scope, but NOT to other hosts or parent scopes. Cross-scope data flow requires explicit policy.

### Policy-mediated flow

Policies control data flow between scopes:

- **Narrowing** — filter or target within a scope (`pipe.filter`, `pipe.to`)
- **Collecting** — reach into peer scopes and harvest quirks (`pipe.collect`)
- **Exposing** — push data up to parent scope (`pipe.expose`)
- **Transforming** — modify data at scope boundaries (`pipe.transform`, `pipe.fold`)

```nix
# Collect http-backends from same-datacenter peers
den.policies.dc-backends = { host, ... }:
  let
    inherit (den.lib.policy) pipe;
    sameDC = builtins.filter
      (h: h.datacenter == host.datacenter && h.name != host.name)
      (builtins.attrValues den.hosts.x86_64-linux);
  in [
    (pipe.from den.pipes.http-backends [
      (pipe.collect (map (peer: { host = peer; }) sameDC))
    ])
  ];

# Narrow secrets at host→user boundary
den.policies.user-secrets = { host, user, ... }:
  let inherit (den.lib.policy) pipe; in [
    (pipe.from den.pipes.secrets [
      (pipe.filter (s: builtins.elem user.name (s.users or [])))
      (pipe.to [ den.aspects.app ])
    ])
  ];
```

---

## 5. Pipeline Mechanics

### Key insight: reuse `scopedClassImports`

Unregistered class keys already collect into `scopedClassImports` and sit inert unless a post-pipeline phase reads them. Routes read `fromClass` and deposit into `intoClass`. Pipes add a new reader.

No separate collection bucket. No separate emission handler. No separate walk-time infrastructure.

### Classification

`classifyKeys` gains a third category:

```
key in den.classes  → class key (existing behavior)
key in den.pipes    → pipe key (collected same as unregistered, consumed differently)
neither             → unregistered class key (backwards compat default)
```

Pipe keys and unregistered keys both flow through `emit-class` into `scopedClassImports.${scope}.${key}`. The distinction matters only at consumption time — `assemblePipes` reads pipe keys, the final output reads the target class.

Implementation note: `classifyKeys` currently returns `{ classKeys, nestedKeys, unregisteredClassKeys }`. Pipe keys can be routed into `unregisteredClassKeys` (simplest — no structural change) or into a new `pipeKeys` return field (cleaner for the optimization of skipping class-wrapping on pipe entries). Either works for correctness since both paths reach `scopedClassImports`.

Optional optimization: `wrapPerScope` could skip class-wrapping on pipe entries (they don't need module identity/location computation). Not required for correctness.

### Collection

No change. `emit-class` with `class = "firewall"` deposits into `scopedClassImports.${scope}.firewall`. The existing scope-partitioned, thunk-wrapped, dedup-tracked infrastructure handles it.

### Post-pipeline assembly

`assemblePipes` runs BEFORE `wrapPerScope` — it produces pipe data that `wrapPerScope` then delivers via the existing `wrapClassModule` mechanism:

```
phase0: assemblePipes       (build pipe data, inject into scope contexts) ← NEW
phase1: wrapPerScope        (class wrapping — ctx now includes pipe data)
phase2: applyProvides       (policy.provide injection)
phase3: applyRoutes         (forwards consume from scopedClassImports)
phase4: applyInstantiates   (entity instantiation)
final:  imports = phase4.${class} or []
```

`assemblePipes`:

1. For each declared pipe, read raw entries from `classImports.${pipeName}` across scopes
2. Walk scope tree for inheritance (merge parent → child, following `scopeParent`)
3. Apply policy pipe builders from `scopedPipeEffects` (new state field, accumulated during walk)
4. Inject assembled data into `scopeContexts` per scope — pipe data becomes part of the ctx that `wrapPerScope` passes to `wrapClassModule`

Because pipe data is in `scopeContexts` before `wrapPerScope` runs, class modules requesting `{ firewall, ... }:` get pipe args pre-applied through the existing `wrapClassModule` partial application — no new delivery mechanism needed.

### Policy pipe effect collection

During the walk, policies produce pipe effects alongside existing effects (resolve, include, exclude, etc.). A new effect type `__policyEffect = "pipe"` carries the pipe builder (source pipe ref, ordered stages, optional destination) and is collected into `scopedPipeEffects.${scope}` — same scope-partitioned state pattern as routes and provides.

---

## 6. Pipe Builder API

A pipe is constructed with `pipe.from` — a source pipe reference and an ordered list of stages. Each `pipe.from` call returns a single pipe effect. Stage ordering within the effect is guaranteed by the list.

```nix
pipe.from <pipe-ref> [ <stage> ... ]
```

### Transform stages

| Stage | Signature | Purpose |
|-------|-----------|---------|
| `pipe.filter` | `(elem → bool) → stage` | Remove entries that don't match predicate |
| `pipe.transform` | `(elem → elem) → stage` | Transform each entry |
| `pipe.fold` | `(elem → acc → acc) → init → stage` | Reduce the list |
| `pipe.append` | `value → stage` | Add a value to the list |
| `pipe.for` | `(entries → value) → stage` | Final transform on the aggregate (at most one per pipe per scope) |

### Routing stages (terminal)

| Stage | Signature | Purpose |
|-------|-----------|---------|
| `pipe.to` | `[ aspects ] → stage` | Narrow delivery to specific aspects within current scope |
| `pipe.expose` | `→ stage` | Push pipe data up to the parent scope (one level) |
| `pipe.collect` | `[ entity-bindings ] → stage` | Resolve peer entity scopes, harvest their quirks into this scope |

Stages apply in declared order within a single `pipe.from`. Routing stages are terminal — they must be last if present. Only one routing stage per `pipe.from`.

- `pipe.to` narrows within the current scope
- `pipe.expose` widens to the parent scope
- `pipe.collect` reaches into peer scopes and brings their quirks back

`pipe.collect` takes a list of entity bindings (same shape as `resolve.to`). Each binding creates a child scope, walks the entity, and collects quirks on this pipe. The collected values are unwrapped — consumers see flat quirk data, not provenance wrappers.

### No destination (scope-wide delivery)

```nix
# All consumers in this scope see the filtered firewall data
(pipe.from den.pipes.firewall [
  (pipe.filter (e: !(e.internal or false)))
])
```

### Aspect-targeted delivery

```nix
# Only postgres and nginx see the secrets
(pipe.from den.pipes.secrets [
  (pipe.filter (s: builtins.elem user.name (s.users or [])))
  (pipe.to [ den.aspects.postgres den.aspects.nginx ])
])
```

### Multiple pipes in one policy

A policy can build multiple pipes:

```nix
den.policies.app-config = { host, user, ... }:
  let inherit (den.lib.policy) pipe; in [
    # Firewall: scope-wide, add monitoring port
    (pipe.from den.pipes.firewall [
      (pipe.append { ports = [ 9100 ]; source = "monitoring"; })
    ])

    # Secrets: aspect-targeted, filtered per-user
    (pipe.from den.pipes.secrets [
      (pipe.filter (s: builtins.elem user.name (s.users or [])))
      (pipe.to [ den.aspects.postgres ])
    ])

    # Backends: scope-wide, annotated with host info
    (pipe.from den.pipes.http-backends [
      (pipe.transform (b: b // { datacenter = host.datacenter; }))
    ])
  ];
```

### Composability across policies

Multiple policies can build pipes for the same pipe ref in the same scope:

```nix
# Policy A: filter internal entries
(pipe.from den.pipes.firewall [
  (pipe.filter (e: !(e.internal or false)))
])

# Policy B: append monitoring ports
(pipe.from den.pipes.firewall [
  (pipe.append { ports = [ 9100 ]; })
])
```

Each `pipe.from` is an independent effect that produces its own output by running its stages against the quirk pool. When multiple pipe effects target the same pipe ref in the same scope:

- **Untargeted (no `pipe.to`):** Each pipe effect runs independently against the pool. Results are merged — the consumer sees the union. `pipe.for` is singular per pipe ref per scope; multiple `pipe.for` across policies is an error with provenance.
- **Different targets:** If Policy A targets `[postgres]` and Policy B targets `[nginx]`, each aspect receives only its respective pipe's output.
- **Same target:** If two policies both target `[postgres]` on the same pipe, each pipe effect runs independently and the results are concatenated. Postgres sees the combined output.

---

## 7. Consumption

### Syntax

Same everywhere: `{ pipeName, ... }:`

```nix
# Pipeline-time discriminator
den.aspects.secure-server = {
  includes = [
    ({ firewall, ... }:
      let hasHttps = builtins.any (f: builtins.elem 443 (f.ports or [])) firewall;
      in lib.optionalAttrs hasHttps {
        includes = [ den.aspects.tls-hardening ];
      })
  ];
};

# Class module consumer
den.aspects.hardened-server = {
  nixos = { firewall, config, ... }: {
    networking.firewall.allowedTCPPorts =
      lib.concatMap (f: f.ports or []) firewall;
  };
};
```

### Pipeline-time vs eval-time

- **Discriminators** (in `includes`): see concrete + parametric-resolved quirks collected so far in walk order. Config-dependent thunks are NOT forced — they appear as opaque values.
- **Class modules** (post-pipeline): see the full tree. All scopes walked. Policy pipe effects applied. Config-dependent thunks resolve lazily via Nix's `evalModules` fixpoint when accessed.

### Empty pipes

When no quirks are emitted for a pipe, consumers receive `[]`.

### Resolution via `wrapClassModule`

Pipe args are delivered through the existing `wrapClassModule` partial application mechanism. When a class module's function signature includes a declared pipe name, `wrapClassModule` pre-applies the assembled pipe data — same mechanism as entity context args (`host`, `user`).

For pipeline-time discriminators, pipe data is available via the `bind` handler. The `bind` handler is extended to resolve pipe names: when a discriminator requests `{ firewall, ... }:` and `firewall` is not an entity context key but IS a declared pipe, `bind` reads accumulated entries from `scopedClassImports` for the current scope (and parent scopes via inheritance). The discriminator sees whatever has been collected so far in walk order — this is partial data, same limitation as entity context during the walk. Cross-scope visibility follows the same inheritance as entity context: a discriminator in user scope sees host-scope and flake-scope pipe entries.

### Cross-host eval-time data

A config-dependent quirk emitted on host A:

```nix
den.aspects.nginx = {
  http-backends = { config, ... }: [{
    addr = config.networking.hostName;
    port = config.services.nginx.defaultHTTPListenPort;
  }];
};
```

This function is collected as-is during the walk. When host B's class module requests `{ http-backends, ... }:` and accesses an entry, Nix lazily forces host A's `evalModules` fixpoint. Same pipeline, same pipe, same list — Nix evaluation handles the ordering.

---

## 8. Relationship to Existing Systems

### Pipes vs classes

Both collect into `scopedClassImports`. The difference is consumption: classes feed `evalModules` via `imports`, pipes deliver as function args. A key cannot be both — `den.classes` and `den.pipes` must not overlap.

### Pipes vs forwards/routes

Forwards intercept class content and redirect it to another class at a path. Pipes intercept quirk data and deliver it as function args. Same collection infrastructure, different post-pipeline readers. Routes read `classImports.${fromClass}`; pipes read `classImports.${pipeName}`.

### Pipes vs provides/\_

Completely orthogonal. The `_`/`provides` namespace is for sub-aspect organization. Pipes are for structured data flow. A sub-aspect under `_` can emit quirks like any other aspect.

---

## 9. Examples

### Firewall aggregation

```nix
# Pipe declaration (by the networking consumer aspect)
den.pipes.firewall = { description = "Firewall port declarations"; };

# Producers (don't know about pipes)
den.aspects.nginx = {
  nixos.services.nginx.enable = true;
  firewall = { ports = [ 80 443 ]; };
};
den.aspects.postgres = {
  nixos.services.postgresql.enable = true;
  firewall = { ports = [ 5432 ]; };
};

# Consumer
den.aspects.networking = {
  nixos = { firewall, lib, ... }: {
    networking.firewall.allowedTCPPorts =
      lib.concatMap (f: f.ports or []) firewall;
  };
};
```

### Pipeline-time conditional inclusion

```nix
den.pipes.firewall = { description = "Firewall port declarations"; };

den.aspects.secure-server = {
  includes = [
    ({ firewall, ... }:
      let hasHttps = builtins.any (f: builtins.elem 443 (f.ports or [])) firewall;
      in lib.optionalAttrs hasHttps {
        includes = [ den.aspects.tls-hardening den.aspects.acme ];
      })
  ];
};
```

### Aspect-targeted secrets

```nix
den.pipes.secrets = { description = "Per-aspect secret paths"; };

# Policy delivers specific secrets to specific aspects
den.policies.app-secrets = { host, ... }:
  let inherit (den.lib.policy) pipe; in [
    (pipe.from den.pipes.secrets [
      (pipe.to [ den.aspects.postgres ])
    ])
    (pipe.from den.pipes.secrets [
      (pipe.to [ den.aspects.nginx ])
    ])
  ];

# Or: construct the value directly via append + target
den.policies.app-secrets-v2 = { host, ... }:
  let inherit (den.lib.policy) pipe; in [
    (pipe.from den.pipes.secrets [
      (pipe.filter (_: false))  # clear the pool
      (pipe.append { db-password = "/run/secrets/pg-pass"; })
      (pipe.to [ den.aspects.postgres ])
    ])
    (pipe.from den.pipes.secrets [
      (pipe.filter (_: false))
      (pipe.append { cert-key = "/run/secrets/nginx-key"; })
      (pipe.to [ den.aspects.nginx ])
    ])
  ];

# Each aspect sees only its own secrets
den.aspects.postgres = {
  nixos = { secrets, ... }: {
    services.postgresql.settings.password_file = secrets.db-password;
  };
};
```

### Cross-host loadbalancer

```nix
den.pipes.http-backends = { description = "HTTP backend endpoints"; };

# Producers — just data, no routing awareness
den.aspects.nginx-backend = {
  nixos.services.nginx.enable = true;
  http-backends = { config, ... }: [{
    addr = config.networking.hostName;
    port = config.services.nginx.defaultHTTPListenPort;
  }];
};

# Consumer — just consumption
den.aspects.haproxy = {
  nixos = { http-backends, lib, ... }: {
    services.haproxy.enable = true;
    services.haproxy.config = lib.concatMapStringsSep "\n"
      (b: "server ${b.addr}:${toString b.port}") http-backends;
  };
};

# Policy — collect backends from ALL peer hosts
den.policies.fleet-backends = { host, ... }:
  let
    inherit (den.lib.policy) pipe;
    peers = builtins.filter (h: h.name != host.name)
      (builtins.attrValues den.hosts.x86_64-linux);
  in [
    (pipe.from den.pipes.http-backends [
      (pipe.collect (map (peer: { host = peer; }) peers))
    ])
  ];
```

### Cross-host loadbalancer with datacenter filtering

```nix
# Policy — collect backends only from same-datacenter peers
den.policies.dc-backends = { host, ... }:
  let
    inherit (den.lib.policy) pipe;
    sameDC = builtins.filter
      (h: h.datacenter == host.datacenter && h.name != host.name)
      (builtins.attrValues den.hosts.x86_64-linux);
  in [
    (pipe.from den.pipes.http-backends [
      (pipe.collect (map (peer: { host = peer; }) sameDC))
    ])
  ];
```

### Cross-host eval-time data

```nix
den.pipes.ssh-host-keys = { description = "SSH public host keys"; };

# Each host emits its keys (config-dependent thunk)
den.aspects.ssh-server = {
  ssh-host-keys = { config, ... }:
    map (k: {
      hostNames = [ config.networking.hostName ];
      publicKey = builtins.readFile k.path;
    }) (builtins.filter (k: k.type == "ed25519") config.services.openssh.hostKeys);
};

# Policy collects host keys from all peers
den.policies.fleet-ssh-keys = { host, ... }:
  let
    inherit (den.lib.policy) pipe;
    peers = builtins.filter (h: h.name != host.name)
      (builtins.attrValues den.hosts.x86_64-linux);
  in [
    (pipe.from den.pipes.ssh-host-keys [
      (pipe.collect (map (peer: { host = peer; }) peers))
    ])
  ];

# Consumer sees all hosts' keys — config-dependent thunks force lazily
den.aspects.known-hosts = {
  nixos = { ssh-host-keys, lib, ... }: {
    programs.ssh.knownHosts = lib.listToAttrs (map (k:
      lib.nameValuePair (builtins.head k.hostNames) k
    ) ssh-host-keys);
  };
};
```

### User-scoped secret filtering via policy

```nix
den.pipes.secrets = { description = "Per-scope secrets"; };

# Host-level secrets emitted as quirks
den.aspects.secret-store = {
  secrets = [
    { name = "db-pass"; path = "/run/secrets/db"; users = [ "admin" ]; }
    { name = "api-key"; path = "/run/secrets/api"; users = null; }
  ];
};

# Policy fires at host→user boundary, filters secrets for this user
den.policies.user-secret-filter = { host, user, ... }:
  let inherit (den.lib.policy) pipe; in [
    (pipe.from den.pipes.secrets [
      (pipe.filter (s:
        s.users == null || builtins.elem user.name s.users))
    ])
  ];
```

### Combined transformation: annotate + filter + reduce

```nix
den.pipes.firewall = { description = "Firewall port declarations"; };

den.policies.production-firewall = { host, ... }:
  let inherit (den.lib.policy) pipe; in [
    (pipe.from den.pipes.firewall [
      # annotate each entry with source host
      (pipe.transform (e: e // { source-host = host.name; }))
      # drop internal-only entries
      (pipe.filter (e: !(e.internal or false)))
      # add monitoring ports
      (pipe.append { ports = [ 9100 9090 ]; source = "monitoring"; })
      # reduce to a flat port list
      (pipe.fold (elem: acc: acc ++ (elem.ports or [])) [])
    ])
  ];
```

---

## 10. Non-Goals

- **Pipe schema validation** — pipes don't validate quirk shape. Quirks are arbitrary data. Consumers handle the structure.
- **Collection strategies** — no `list`/`map`/`single` on the schema. Always a list. Consumers and policies transform as needed via `fold`/`for`.
- **`_den.traits` injection modules** — no generated NixOS options. Pipe data is delivered via `wrapClassModule` pre-application and `bind` handlers, not through the module system's option machinery.
- **Dynamic pipe registration** — no `register-pipe` effect during the walk. All pipes declared at `den.pipes` before the pipeline starts.
- **Provenance in consumer API** — consumers see flat quirk values, not provenance wrappers. The pipeline tracks provenance internally for error messages. Consumer-facing provenance may be added later if needed.

---

## 11. Migration

### From current state (no pipes)

Purely additive. No existing behavior changes. Unregistered keys continue to work as class modules (backwards compat). Declaring a key in `den.pipes` opts it into pipe semantics.

### From deleted trait system

The pipe system replaces traits with:
- No evaluation tiers (the pipeline collects uniformly; Nix laziness handles eval-time)
- No DLQ (unregistered keys are class modules, always)
- No `_den.traits` injection (delivery via `wrapClassModule` and `bind`)
- No provide-to sub-pipelines (scope tree inheritance + policy pipe effects)
- No `partialOk` / tier visibility validation (discriminators see what's available; class modules see everything)

### Spec disposition

| Spec | Action |
|------|--------|
| `design/traits.md` | Superseded by this document |
| `design/fleet-and-exports.md` | Cross-host case handled by config-dependent thunks + Nix laziness. `fleet`/`nodes` module arg remains a useful orthogonal convenience but is not required for the pipe system. |
| `tbd/emission-pipeline.md` | Shared `emit-content` concept validated but unnecessary — pipes reuse `emit-class` collection. `wrapClassModule` decomposition deferred (function is cohesive at current size). |
