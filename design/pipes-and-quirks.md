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
- A config-dependent thunk (`{ config, ... }: { port = config.services.nginx.port; }` — collected as-is, resolved post-pipeline; see Section 5.5)

**List-valued quirks are auto-flattened.** If a quirk value is a list, each element becomes a separate entry in the pipe. A quirk `firewall = [ { ports = [80]; } { ports = [443]; } ]` contributes two entries, not one list-in-a-list.

### Pipe

A declared data route. Establishes that a key name is a pipe, not a class:

```nix
den.pipes.firewall = {
  description = "Firewall port declarations";
};
```

Top-level, declared by consumers. Complete before pipeline starts. The declaration is minimal — just metadata. No schema validation, no collection strategy. Quirks are always collected as a list; consumers transform however they want.

### Policy pipe builder

A pipe is constructed as a sequence of stages with optional routing. The policy's own context (`{ host, user, ... }:`) determines WHERE and WHEN the pipe applies. The stages describe WHAT happens to the data. The optional routing stage controls WHERE it goes.

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
- **Who/where** — the optional routing stage (to, expose, collect)

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
den.policies.dc-backends =
  let inherit (den.lib.policy) pipe; in
  { host, ... }:
  let receiver = host; in [
    (pipe.from den.pipes.http-backends [
      (pipe.collect ({ host, ... }: host.datacenter == receiver.datacenter))
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

### 5.1 Key insight: reuse `scopedClassImports`

Unregistered class keys already collect into `scopedClassImports` and sit inert unless a post-pipeline phase reads them. Routes read `fromClass` and deposit into `intoClass`. Pipes add a new reader.

No separate collection bucket. No separate emission handler. No separate walk-time infrastructure.

### 5.2 Classification

`classifyKeys` gains a third category:

```
key in den.classes  → class key (existing behavior)
key in den.pipes    → pipe key (collected same as unregistered, consumed differently)
neither             → unregistered class key (backwards compat default)
```

Pipe keys and unregistered keys both flow through `emit-class` into `scopedClassImports.${scope}.${key}`. The distinction matters only at consumption time — `assemblePipes` reads pipe keys, the final output reads the target class.

Implementation: `classifyKeys` currently returns `{ classKeys, nestedKeys, unregisteredClassKeys }`. Pipe keys get a new `pipeKeys` return field. This allows `wrapPerScope` to skip class-wrapping on pipe entries (they don't need module identity/location computation). Pipe keys still flow through `emit-class` into `scopedClassImports` — the separate field only affects post-pipeline processing.

### 5.3 Collection

No change. `emit-class` with `class = "firewall"` deposits into `scopedClassImports.${scope}.firewall`. The existing scope-partitioned, thunk-wrapped, dedup-tracked infrastructure handles it.

### 5.4 Post-pipeline assembly

`assemblePipes` runs BEFORE the existing post-pipeline phases — it augments scope contexts with pipe data, which `wrapPerScope` then delivers via `wrapClassModule`. The existing phases are NOT renumbered:

```
NEW:    assemblePipes       (build pipe data, augment scope contexts)
phase1: wrapPerScope        (class wrapping — ctx now includes pipe data)
phase2: applyProvides       (policy.provide injection)
phase3: applyRoutes         (forwards consume from scopedClassImports)
phase4: applyInstantiates   (entity instantiation)
final:  imports = phase4.${class} or []
```

`assemblePipes` takes `scopeContexts` from pipeline state and returns an augmented version with pipe data added as keys. The augmented `scopeContexts` is passed to `wrapPerScope`, which passes it to `wrapClassModule`. Because pipe names appear as keys in the scope context, `wrapClassModule`'s existing `ctx ? ${k}` check detects them and pre-applies the data — no new delivery mechanism needed.

```nix
# In fxResolve (resolve.nix) — assemblePipes inserted before existing phases:
let
  # assemblePipes references instantiatedConfigs (forward reference — Nix recursive let)
  augmentedScopeContexts = assemblePipes {
    inherit scopeContexts scopedClassImports scopedPipeEffects;
    hostConfigs = instantiatedConfigs;  # lazy forward ref to phase4 output
  };
  phase1 = wrapPerScope ctx augmentedScopeContexts scopedClassImports;
  phase2 = applyProvides ... phase1;
  phase3 = applyRoutes ... phase2;
  phase4 = applyInstantiates ... phase3;
  instantiatedConfigs = extractLazyConfigs phase4;  # { scopeId → lazy config }
in ...
```

`assemblePipes` steps:

1. For each declared pipe, read raw entries from `scopedClassImports.${scope}.${pipeName}` per scope
2. Apply `pipe.collect` effects — read quirks from matching peer scopes' raw `scopedClassImports` entries (pre-assembly, not post-expose — collect and expose are independent)
3. Apply policy pipe stages (filter, transform, fold, append, for) from `scopedPipeEffects`
4. Resolve config-dependent thunks (see Section 5.5)
5. Auto-flatten list-valued entries (thunk results that are lists are also flattened; non-list thunk results become single entries)
6. For every declared pipe in every scope, inject assembled data into `scopeContexts` — pipes with no emissions get `[]` (guarantees consumers always receive a list, never missing/null)
7. Process `pipe.expose` effects bottom-up: merge exposed data into parent scope's pool before the parent's own assembly

### 5.5 Config-dependent thunk resolution

A quirk like `http-backends = { config, ... }: [{ addr = config.networking.hostName; }]` is a function requiring NixOS `config`. During the walk, it is collected as-is (a bare function). During `assemblePipes`, it must be resolved against the source scope's instantiated config.

**Mechanism:** `assemblePipes` receives `hostConfigs` — a lazy forward reference to the instantiated configs produced by `applyInstantiates` (phase 4). This works via Nix's recursive `let`: `assemblePipes` stores the reference as a thunk, which is only forced when a consumer accesses the resolved value.

```nix
# For each config-dependent entry in a scope:
resolvedValue = entry.value {
  config = hostConfigs.${sourceScopeId};  # lazy — forces source host's evalModules on access
  inherit lib;
};
```

The resolved value is itself lazy — Nix doesn't force the source config until someone accesses fields on `resolvedValue`. When HOST-A's haproxy accesses `b.addr`, that forces HOST-B's `evalModules` through the lazy reference chain:

```
HOST-A evalModules → haproxy module → http-backends list → resolvedValue.addr
  → forces hostConfigs.${hostBScope} → HOST-B evalModules → config.networking.hostName
```

**Detection:** A quirk entry is config-dependent if it's a function whose args include module-system names (`config`, `lib`, `pkgs`, `options`, `modulesPath`). The same detection logic used by `wrapClassModule` applies.

**Mutual dependencies.** Two hosts can read each other's config-dependent quirks as long as they access different attributes. Nix evaluates attributes lazily and independently — `HOST-A.config.services.X` and `HOST-A.config.networking.Y` are separate thunks. True infinite recursion only occurs when the exact same attribute transitively depends on itself across hosts. This is the same constraint as NixOS module system self-references (`config.services.X` reading `config.services.Y` within the same host). Nix reports `infinite recursion encountered` with a stack trace identifying the cycle.

### 5.6 Policy pipe effect collection

During the walk, policies produce pipe effects alongside existing effects (resolve, include, exclude, etc.). A new effect type `__policyEffect = "pipe"` carries the pipe builder (source pipe ref, ordered stages, optional routing) and is collected into `scopedPipeEffects.${scope}` — same scope-partitioned state pattern as routes and provides.

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
| `pipe.withProvenance` | `→ stage` | Wrap each entry as `{ value, source }` where `source` is the emitting scope's context |

### Routing stages

| Stage | Signature | Purpose |
|-------|-----------|---------|
| `pipe.to` | `[ aspects ] → stage` | Narrow delivery to specific aspects within current scope |
| `pipe.expose` | `→ stage` | Push pipe data up to the parent scope (one level) |
| `pipe.collect` | `(peerCtx → bool) → stage` | Resolve matching peer scopes, harvest their quirks into this scope |

Transform stages apply in declared order. Routing stages are terminal — at most one per `pipe.from`, must be last if present.

**Exception:** `pipe.collect` is NON-terminal. Transform stages after `pipe.collect` operate on the combined pool (local quirks + collected peer quirks). This enables collect-then-filter patterns:

```nix
(pipe.from den.pipes.http-backends [
  (pipe.collect ({ host, ... }: host.datacenter == receiver.datacenter))
  (pipe.filter (b: b.port != 8080))
  (pipe.to [ den.aspects.haproxy ])
])
```

Only `pipe.to` and `pipe.expose` are strictly terminal. `pipe.collect` composes with `pipe.expose` — collect from peers then expose the combined result upward:

```nix
(pipe.from den.pipes.http-backends [
  (pipe.collect ({ host, ... }: true))
  (pipe.expose)
])
```

### `pipe.collect` — peer scope predicate

`pipe.collect` takes a predicate function that receives each peer scope's context. The pipeline iterates all walked scopes, calls the predicate with each scope's context, and harvests quirks from matching scopes. Entity kind filtering is implicit in the destructuring:

```nix
# Collect from host scopes only (destructures { host, ... })
(pipe.collect ({ host, ... }: host.datacenter == receiver.datacenter))

# Collect from all host scopes
(pipe.collect ({ host, ... }: true))

# Collect from user scopes
(pipe.collect ({ user, ... }: true))
```

The predicate follows the same convention as policy signatures — entity kind matching via function args. The current scope is automatically excluded (a host doesn't collect from itself).

`pipe.collect` reads from **already-walked pipeline state** — it does NOT re-walk peer scopes. All host scopes are subgraphs of one pipeline run and have already been walked by the time policies fire. `pipe.collect` reads `scopedClassImports.${peerScope}.${pipeName}` for matching scopes.

### `pipe.withProvenance` — source context access

Wraps each entry as `{ value, source }` where `source` is the scope context of the scope that emitted the quirk. Typically used after `pipe.collect` to annotate entries with their origin:

```nix
(pipe.from den.pipes.backup-targets [
  (pipe.collect ({ host, ... }: true))
  (pipe.withProvenance)
  (pipe.transform (e: e.value // { source-host = e.source.host.name; }))
  (pipe.to [ den.aspects.backup-server ])
])
```

After the transform, the consumer sees flat entries with source metadata baked in. The provenance wrapper is an intermediate form — downstream stages (filter, transform, fold) operate on the wrapped shape until a transform unwraps it.

`pipe.withProvenance` also works on local quirks (not just collected ones) — every entry has a source scope.

### `pipe.for` — aggregate transform

`pipe.for` replaces the list with an arbitrary value. Useful when a consumer expects a single attrset rather than a list:

```nix
# Reduce to a single merged attrset
(pipe.from den.pipes.secrets [
  (pipe.filter (_: false))
  (pipe.append { db-password = "/run/secrets/pg-pass"; })
  (pipe.for lib.mergeAttrsList)
  (pipe.to [ den.aspects.postgres ])
])
```

After `pipe.for`, the consumer receives whatever the function returns — not necessarily a list. At most one `pipe.for` per pipe per scope; multiple is an error with provenance.

**Override pattern:** `pipe.filter (_: false)` + `pipe.append` discards all emitted quirks and injects policy-constructed data. This is the standard way to deliver specific values to specific aspects — the policy replaces the pool entirely rather than filtering it. Used in the secrets examples below.

### `pipe.expose` — push to parent scope

```nix
# User-scope data visible at host scope
den.policies.user-prefs-to-host = { host, user, ... }:
  let inherit (den.lib.policy) pipe; in [
    (pipe.from den.pipes.user-preferences [
      (pipe.expose)
    ])
  ];
```

Pushes the pipe's data (after any preceding transform stages) up one level to the parent scope. The parent scope's consumers see the exposed data alongside their own local quirks.

### No destination (scope-wide delivery)

```nix
# All consumers in this scope see the filtered firewall data
(pipe.from den.pipes.firewall [
  (pipe.filter (e: !(e.internal or false)))
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

    # Secrets: aspect-targeted, reduced to single attrset
    (pipe.from den.pipes.secrets [
      (pipe.filter (_: false))
      (pipe.append { db-password = "/run/secrets/pg-pass"; })
      (pipe.for lib.mergeAttrsList)
      (pipe.to [ den.aspects.postgres ])
    ])

    # Backends: collect from peers, filter, deliver to haproxy
    (pipe.from den.pipes.http-backends [
      (pipe.collect ({ host, ... }: host.datacenter == receiver.datacenter))
      (pipe.to [ den.aspects.haproxy ])
    ])
  ];
```

### Composability across policies

Multiple policies can build pipes for the same pipe ref in the same scope:

Each `pipe.from` is an independent effect that produces its own output by running its stages against the quirk pool. When multiple pipe effects target the same pipe ref in the same scope:

- **Untargeted (no routing stage):** Each pipe effect runs independently against the pool. Results are merged — the consumer sees the union. `pipe.for` is singular per pipe ref per scope; multiple `pipe.for` across policies is an error with provenance.
- **Different targets:** If Policy A targets `[postgres]` and Policy B targets `[nginx]`, each aspect receives only its respective pipe's output.
- **Same target:** If two policies both target `[postgres]` on the same pipe, each pipe effect runs independently and the results are concatenated. Postgres sees the combined output.
- **Duplicate collection:** If two policies both `pipe.collect` the same peers for the same pipe, collected quirks are deduplicated by source scope + pipe name (same quirk from the same scope is not counted twice).

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

- **Discriminators** (in `includes`): pipe-arg discriminators are DEFERRED during `emitIncludes` and drained after all sibling aspects in the current scope have emitted. This ensures they see the complete local quirk pool, not partial walk-order-dependent data. The existing bind/defer/drain lifecycle handles this: `bind` recognizes pipe args and defers them; the drain point after `emitIncludes` re-evaluates with the full scope data. Config-dependent thunks are NOT forced — they appear as opaque values.
- **Class modules** (post-pipeline): see the full tree. All scopes walked. Policy pipe effects applied. Config-dependent thunks resolved via lazy config references (Section 5.5).

### Empty pipes

When no quirks are emitted for a pipe, consumers receive `[]`.

### Resolution via `wrapClassModule`

Pipe args are delivered through the existing `wrapClassModule` partial application mechanism. When a class module's function signature includes a declared pipe name, `wrapClassModule` pre-applies the assembled pipe data — same mechanism as entity context args (`host`, `user`).

For pipeline-time discriminators, pipe data is available via the `bind` handler. The `bind` handler is extended with a `pipeRegistry` check: when resolving function args, if a name is not a scope handler (entity context) but IS a key in `pipeRegistry` (the declared pipes set, available at pipeline construction time), `bind` reads the accumulated entries from `scopedClassImports.${currentScope}.${pipeName}` in pipeline state. However, this data may be incomplete during `emitIncludes` (not all sibling aspects have emitted yet).

To eliminate walk-order sensitivity, pipe-arg discriminators are DEFERRED: when `bind` detects a pipe arg, it defers the discriminator (same mechanism as unsatisfied entity context args). The deferred discriminator is drained after `emitIncludes` completes for the current scope — at that point, all sibling aspects have emitted their quirks. The drain re-evaluates `bind`, which reads the now-complete local pipe data from `scopedClassImports`. This ensures discriminators see the full local quirk pool regardless of include order.

---

## 8. Relationship to Existing Systems

### Pipes vs classes

Both collect into `scopedClassImports`. The difference is consumption: classes feed `evalModules` via `imports`, pipes deliver as function args. A key cannot be both — `den.classes` and `den.pipes` must not overlap.

### Pipes vs forwards/routes

Forwards intercept class content and redirect it to another class at a path. Pipes intercept quirk data and deliver it as function args. Same collection infrastructure (`scopedClassImports`), different post-pipeline readers. Routes read `scopedClassImports.*.${fromClass}`; pipes read `scopedClassImports.*.${pipeName}`.

### Pipes vs provides/\_

Completely orthogonal. The `_`/`provides` namespace is for sub-aspect organization. Pipes are for structured data flow. A sub-aspect under `_` can emit quirks like any other aspect.

---

## 9. Examples

### Firewall aggregation (single host)

All aspects included on the same host — scope-local visibility is sufficient.

```nix
den.pipes.firewall = { description = "Firewall port declarations"; };

den.hosts.x86_64-linux.igloo.users.tux = {};

# Host aspect includes all relevant aspects
den.aspects.igloo = {
  includes = [
    den.aspects.nginx
    den.aspects.postgres
    den.aspects.networking
  ];
};

# Producers (don't know about pipes)
den.aspects.nginx = {
  nixos.services.nginx.enable = true;
  firewall = { ports = [ 80 443 ]; };
};
den.aspects.postgres = {
  nixos.services.postgresql.enable = true;
  firewall = { ports = [ 5432 ]; };
};

# Consumer — same scope, sees all local quirks
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

# Policy constructs and delivers specific secrets to specific aspects
den.policies.app-secrets = { host, ... }:
  let inherit (den.lib.policy) pipe; in [
    (pipe.from den.pipes.secrets [
      (pipe.filter (_: false))
      (pipe.append { db-password = "/run/secrets/pg-pass"; })
      (pipe.for lib.mergeAttrsList)
      (pipe.to [ den.aspects.postgres ])
    ])
    (pipe.from den.pipes.secrets [
      (pipe.filter (_: false))
      (pipe.append { cert-key = "/run/secrets/nginx-key"; })
      (pipe.for lib.mergeAttrsList)
      (pipe.to [ den.aspects.nginx ])
    ])
  ];

# Each aspect receives a single attrset (not a list)
den.aspects.postgres = {
  nixos = { secrets, ... }: {
    services.postgresql.settings.password_file = secrets.db-password;
  };
};
den.aspects.nginx = {
  nixos = { secrets, ... }: {
    security.acme.certs."example.com".credentialFiles.key = secrets.cert-key;
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
den.policies.fleet-backends =
  let inherit (den.lib.policy) pipe; in
  { host, ... }: [
    (pipe.from den.pipes.http-backends [
      (pipe.collect ({ host, ... }: true))
    ])
  ];
```

### Cross-host loadbalancer with datacenter filtering

```nix
# Policy — collect from same-datacenter peers, filter out health-check ports
den.policies.dc-backends =
  let inherit (den.lib.policy) pipe; in
  { host, ... }:
  let receiver = host; in [
    (pipe.from den.pipes.http-backends [
      (pipe.collect ({ host, ... }: host.datacenter == receiver.datacenter))
      (pipe.filter (b: b.port != 8081))
      (pipe.to [ den.aspects.haproxy ])
    ])
  ];
```

### Cross-host eval-time data (SSH host keys)

```nix
den.pipes.ssh-host-keys = { description = "SSH public host keys"; };

# Each host emits its keys (config-dependent thunk — resolved lazily)
den.aspects.ssh-server = {
  ssh-host-keys = { config, ... }:
    map (k: {
      hostNames = [ config.networking.hostName ];
      publicKey = builtins.readFile k.path;
    }) (builtins.filter (k: k.type == "ed25519") config.services.openssh.hostKeys);
};

# Policy collects host keys from all peers
den.policies.fleet-ssh-keys =
  let inherit (den.lib.policy) pipe; in
  { host, ... }: [
    (pipe.from den.pipes.ssh-host-keys [
      (pipe.collect ({ host, ... }: true))
    ])
  ];

# Consumer — each entry is a single key record (list auto-flattened)
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

### Expose: user preferences visible to host scope

```nix
den.pipes.user-preferences = { description = "User preference declarations"; };

# User aspect emits preferences
den.aspects.tux = {
  user-preferences = { editor = "vim"; shell = "zsh"; };
};

# Policy exposes user prefs to parent (host) scope
den.policies.expose-user-prefs = { host, user, ... }:
  let inherit (den.lib.policy) pipe; in [
    (pipe.from den.pipes.user-preferences [
      (pipe.transform (p: p // { userName = user.name; }))
      (pipe.expose)
    ])
  ];

# Host-level consumer sees all users' preferences
den.aspects.user-shell-setup = {
  nixos = { user-preferences, lib, ... }: {
    programs.zsh.enable = lib.any (p: p.shell == "zsh") user-preferences;
    programs.fish.enable = lib.any (p: p.shell == "fish") user-preferences;
  };
};
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
- **Provenance by default** — consumers see flat quirk values by default. `pipe.withProvenance` opts into provenance wrapping when source context is needed (e.g., annotating collected entries with their origin host).
- **Multi-level expose** — `pipe.expose` pushes one level up. This may be revisited if the pattern proves too verbose.

---

## 11. Implementation Phases

Each phase is a shippable increment with its own test suite. Earlier phases are prerequisites for later ones. CI must pass at each phase boundary.

### Phase 1: Registry + Classification (foundation)

**Goal:** Pipe keys are recognized and collected separately from class keys. No consumption yet — quirks sit in `scopedClassImports` as inert data, same as today's unregistered keys.

**Changes:**
- Add `den.pipes` option (top-level, `lazyAttrsOf` metadata submodule)
- Add `pipeRegistry` to `key-classification.nix` (read from `den.pipes`)
- Update `classifyKeys` to return `pipeKeys` as a separate category
- Pipe keys still flow through `emit-class` into `scopedClassImports`
- Add collision assertion: `den.classes` and `den.pipes` must not overlap

**Tests:** Declare `den.pipes.firewall`, put `firewall = { ports = [80]; }` on an aspect, verify it reaches `scopedClassImports` under the pipe key name. Verify class modules are NOT generated for pipe keys. Verify all existing tests pass unchanged (no behavioral change).

**Why first:** Everything else depends on the registry existing and classification working. This is purely additive — zero risk to existing behavior.

### Phase 2: Local scope consumption (the core value)

**Goal:** Aspects on the same host can emit quirks and consume them as function args. This is the firewall aggregation use case — the highest-value scenario with the smallest surface area.

**Changes:**
- Add `assemblePipes` function (phase 0 in post-pipeline assembly)
- `assemblePipes` reads pipe keys from `scopedClassImports`, assembles per-scope lists
- Inject assembled pipe data into `scopeContexts` before `wrapPerScope`
- Extend `wrapClassModule` to recognize pipe names alongside entity context args
- Extend `bind` handler to recognize pipe names (for pipeline-time discriminators)
- Implement pipe-arg deferral in `bind` (defer pipe args during `emitIncludes`, drain after)
- Auto-flatten list-valued quirk entries

**Tests:** Firewall aggregation (multiple aspects emit, one consumes on same host). Pipeline-time discriminator (capabilities gating). Empty pipe = `[]`. Diamond dependency dedup. Discriminator deferral (producer after discriminator in includes — verify correct result regardless of order).

**Why second:** This delivers the core user value. Local-scope pipes with no policies, no cross-scope routing. Users can start using `den.pipes` + quirks immediately.

### Phase 3: Policy pipe builder — transform stages

**Goal:** Policies can filter, transform, fold, append, and aggregate pipe data within a scope.

**Changes:**
- Add `pipe` namespace to `den.lib.policy` (alongside existing `resolve`, `include`, `exclude`)
- Implement `pipe.from` returning a single `__policyEffect = "pipe"` effect
- Implement transform stages: `pipe.filter`, `pipe.transform`, `pipe.fold`, `pipe.append`, `pipe.for`
- Add `scopedPipeEffects` state field (accumulated during walk, same pattern as `scopedRoutes`)
- `assemblePipes` applies pipe effects per scope in declared order
- Enforce `pipe.for` singularity (at most one per pipe per scope, error with provenance)

**Tests:** Filter internal firewall entries. Transform to annotate. Fold to reduce to flat list. Append monitoring ports. `pipe.for` aggregate transform. `pipe.for` conflict error. Combined stage chain. Multiple `pipe.from` in one policy.

**Why third:** Transform stages require the builder API and effect collection infrastructure. They build on phase 2's consumption mechanism. No cross-scope concerns yet — purely local data transformation.

### Phase 4: Aspect targeting — `pipe.to`

**Goal:** Policies can narrow pipe delivery to specific aspects. This is the secrets use case.

**Changes:**
- Implement `pipe.to [ aspects ]` as a terminal routing stage
- `assemblePipes` tracks targeted vs untargeted pipe effects
- Targeted delivery: pipe data injected per-aspect in `wrapClassModule` ctx (aspect identity available from class entry metadata)
- Aspect-targeted takes precedence over scope-wide for the targeted aspect

**Tests:** Secrets delivered to postgres only. Two policies targeting different aspects on same pipe. Two policies targeting same aspect (concatenation). Untargeted + targeted coexistence.

**Why fourth:** Aspect targeting requires knowing which aspect each class module belongs to during `assemblePipes`. This is more complex than scope-wide delivery and should be validated after the basic mechanism is proven.

### Phase 5: `pipe.expose` — upward scope flow

**Goal:** Child scope data can be pushed to parent scope. This is the user-preferences-to-host use case.

**Changes:**
- Implement `pipe.expose` as a terminal routing stage
- `assemblePipes` processes expose effects: after assembling a child scope's pipe data, merge exposed data into the parent scope's pool
- Ordering: child scopes assemble before parents (bottom-up)

**Tests:** User quirks exposed to host scope. Exposed data visible to host-scope consumers. Exposed data NOT visible to sibling scopes (only parent). Chain: user → host → fleet (two expose policies).

**Why fifth:** Expose requires bottom-up assembly ordering in `assemblePipes`. This is a structural change to assembly logic that should come after the basic top-down flow is proven.

### Phase 6: `pipe.collect` — peer scope harvesting

**Goal:** Policies can reach into peer scopes and harvest their quirks. This is the cross-host service discovery use case.

**Changes:**
- Implement `pipe.collect (predicate)` as a non-terminal routing stage
- Predicate receives peer scope context, matches via destructuring (`{ host, ... }:`)
- `assemblePipes` evaluates collect predicates against all walked scopes
- Auto-exclude current scope from collection
- Dedup collected quirks by source scope + pipe name
- Post-collect transform stages operate on combined pool (local + collected)

**Tests:** Fleet-wide collection (all hosts). Filtered collection (same datacenter). Collect + filter composition. Self-exclusion. Dedup across overlapping policies. Collect from scope with no quirks = no contribution.

**Why sixth:** Cross-scope collection is the most complex mechanism. It requires iterating all scopes and evaluating predicates. It should only be built after local-scope, targeting, and expose are all proven correct.

### Phase 7: `pipe.withProvenance` + config-dependent thunks

**Goal:** Full cross-host eval-time data flow. Source context access via provenance wrapping.

**Changes:**
- Implement `pipe.withProvenance` stage (wraps entries as `{ value, source }`)
- Config-dependent thunk detection (function args include `config`, `lib`, etc.)
- Config thunk resolution: lazy forward reference to instantiated configs via Nix recursive `let` in `fxResolve`
- Auto-flatten thunk results that are lists

**Tests:** SSH host keys (config-dependent thunk, list-valued, cross-host). Provenance wrapping + transform to extract source host. Thunk not forced until consumer access (verify laziness). Mutual dependency (two hosts reading each other's non-overlapping config attributes). Config thunk on same host (no cross-host, resolves in own fixpoint).

**Why last:** Config-dependent thunks require the lazy forward reference to instantiated configs — the most subtle mechanism in the entire design. All other features should be stable before introducing this complexity. Most use cases (firewall, capabilities, secrets) work with static or parametric quirks and don't need this phase.

### Phase summary

| Phase | Feature | Key test scenario | Risk |
|-------|---------|-------------------|------|
| 1 | Registry + classification | Pipe key recognized, not emitted as class | Low — additive only |
| 2 | Local consumption | Firewall aggregation on same host | Medium — new post-pipeline phase |
| 3 | Transform stages | Filter + fold + combined chains | Low — builds on proven phase 2 |
| 4 | `pipe.to` | Per-aspect secrets | Medium — aspect identity in assembly |
| 5 | `pipe.expose` | User prefs to host scope | Medium — bottom-up assembly ordering |
| 6 | `pipe.collect` | Cross-host service discovery | High — cross-scope predicate matching |
| 7 | Provenance + thunks | Cross-host eval-time data | High — lazy forward references |

---

## 12. Migration

### From current state (no pipes)

Purely additive. No existing behavior changes. Unregistered keys continue to work as class modules (backwards compat). Declaring a key in `den.pipes` opts it into pipe semantics.

### From deleted trait system

The pipe system replaces traits with:
- No evaluation tiers (the pipeline collects uniformly; Nix laziness handles eval-time)
- No DLQ (unregistered keys are class modules, always)
- No `_den.traits` injection (delivery via `wrapClassModule` and `bind`)
- No provide-to sub-pipelines (`pipe.collect` reads from already-walked state)
- No `partialOk` / tier visibility validation (discriminators see what's available; class modules see everything)

### Spec disposition

| Spec | Action |
|------|--------|
| `design/traits.md` | Superseded by this document |
| `design/fleet-and-exports.md` | Cross-host case handled by `pipe.collect` + config-dependent thunks. `fleet`/`nodes` module arg remains a useful orthogonal convenience but is not required. |
| `tbd/emission-pipeline.md` | Shared `emit-content` concept validated but unnecessary — pipes reuse `emit-class` collection. `wrapClassModule` decomposition deferred (function is cohesive at current size). |
