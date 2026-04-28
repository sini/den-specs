# Policies and Cross-Entity Resolution

## Problem

Three coupled problems in den's context transition system:

1. **Hardcoded transition topology**: `into` functions on ctx nodes (`den.ctx.host.into.user`) couple topology to entity type definitions. Adding new entity types (kubernetes clusters, environments) requires new ctx nodes with custom wiring. (Note: `den.ctx` is being fully removed — see `2026-04-21-ctx-as-classes-design.md`. Current `into` transitions move to `den.policies`; scoped behavior on ctx nodes moves to `den.stages`.)

2. **`__functor` on submodule-evaluated attrsets**: ctx nodes need to be callable for `den.ctx.host { host = config; }`. The `__functor` causes `lib.isFunction` probing issues during NixOS module loading — forcing evaluation chains that reach `config.den` before it's available. (Resolved by ctx removal — no more callable ctx nodes.)

3. **No cross-entity module routing**: the pipeline resolves aspects in isolated scopes. Context flows down (parent to child) but not across entity boundaries. Host A can't contribute modules to host B's config (fleet `/etc/hosts`, haproxy backends). The forward mechanism starts fresh pipeline runs that lose parent context.

## Design

### Three layers

**Layer 1 — Policies** (module evaluation time): declare what relates to what as first-class data, separate from entity definitions.

**Layer 2 — Effect-based materialization** (pipeline execution time): policies compile into per-policy named effect handlers. The transition handler sends effects and handlers respond with fan-out targets.

**Layer 3 — Cross-entity routing** (orchestration time): when a policy targets a sibling entity (same type as emitter), resolved modules are collected via `provide-to` effects and distributed to target configs in a second phase.

### Entity types

Unchanged. Typed records with schema: `den.hosts`, `den.users`, `den.homes`, etc. Entity types define *structure* (fields, options). Policies define *topology* — separate from structure.

---

## Layer 1: Policies

### Policy declaration

A policy declares a named policy between entity kinds:

```nix
den.policies.host-to-users = {
  from = "host";
  to = "user";
  resolve = { host }: map (user: { inherit host user; }) (lib.attrValues host.users);
};

den.policies.host-wheel-users = {
  from = "host";
  to = "user";
  resolve = { host }:
    builtins.filter
      (user: builtins.elem "wheel" (user.groups or []))
      (lib.attrValues host.users);
};
```

### Resolve function input contract

The `resolve` function receives the **accumulated pipeline context** — the same attrset that parametric aspects receive via `bind.fn`. At host level: `{ host }`. At user level: `{ host, user }`. Each policy that fans out adds its target entity to the context for downstream policies.

Additional entities not in the pipeline context (e.g., `environment` in the ACL case) are accessed via module config at registry time — the resolve function is defined in a NixOS module where `config.environments` etc. are available as closures.

The function returns a **list of context attrsets**, each representing one target entity's context:

```nix
resolve = { host }: map (user: { inherit host user; }) (lib.attrValues host.users);
```

Multiple policies can share the same `from`/`to` pair.

### Execution semantics

**Depth-based ordering**: policies run at the depth determined by their `from` type in the entity graph. `environment-to-hosts` runs before `host-to-users` because the environment→host fan-out creates the host-level context that host-level policies consume. This ordering is implicit from the policy graph, not declared.

**Same-depth independence**: policies sharing the same `from` type at the same pipeline depth execute independently. Each sees the same input context. No sequential dependencies between them. Their results are unioned — all targets from all same-depth policies are resolved.

**Dedup** operates at two layers:
- **Transition-level**: `ctx-seen` deduplicates by transition path key (same as today). Prevents re-entering the same context node via the same path. Two different policies targeting the same entity via different paths are NOT deduped at this layer — both resolve independently.
- **Module-level**: the class collector deduplicates emitted modules by `loc` key (`class@aspectIdentity`). Static aspects reaching the same entity via different paths produce identical `loc` keys → kept once. Parametric aspects with different `__ctxId` are kept separately.
- **Provide-to**: phase 2 module injection tags each module with the *source entity identity* as part of the NixOS module key: `class@aspectIdentity/sourceEntity→targetEntity`. This ensures multiple sources providing the same capability to the same target don't conflict — `nixos@webserver/{web1}→{lb}` and `nixos@webserver/{web2}→{lb}` are distinct keys, both kept. Same source + same aspect + same target deduplicates.

**Diamond resolution**: when the same entity is reachable via multiple paths (A→B→X and A→C→X), each path produces an independent resolution run. The class collector deduplicates by aspect identity + class key — identical aspects reaching the same entity via different paths produce the same dedup key and are kept once. Different aspects or parametric aspects with different contexts (`__ctxId`) are kept separately and NixOS module merge resolves any conflicts.

**Conflict resolution**: policies are pure queries — they produce targets, not modules. Conflicts can only arise in the *modules* emitted for those targets. Since modules are injected into NixOS/homeManager evaluation, the standard NixOS module priority system handles conflicts. No custom conflict resolution needed at the policy level.

**Ordering**: policies at the same depth execute in declaration order for determinism, but correctness must not depend on order.

**Context key collisions**: the `as` field determines the context key name for the target entity. Two policies with the same `as` value targeting the same pipeline depth would collide — the last-evaluated overwrites. Policies with unique `as` values (or distinct `to` types) avoid this. Collisions should emit a trace warning.

**Entity identity**: dedup and tracing require a consistent identity for each entity. Built-in types use `entity.name`. Custom entity types should provide an identity derivation (e.g., via a `name` field or a policy-level `identity` function). Entities without a `name` field use a hash of the context attrset as fallback.

**Error handling**: if `resolve` returns a non-list, emit a trace warning and produce no fan-out. Empty list produces no fan-out silently. Missing entity types (typo in `from`/`to`) produce a warning at pipeline entry.

### ~~Inline sugar on ctx nodes~~ (Removed)

> **Update (2026-04-21):** `den.ctx` is being fully removed per the Data/Policies/Behavior separation spec. There is no inline sugar on ctx nodes. All policies are declared via `den.policies`. The examples below show the old syntax for historical context only.
>
> Previously, ctx nodes could declare policies inline via `into`:
> ```nix
> # OLD — no longer supported after ctx removal:
> den.ctx.host.into = {
>   user = { host }: map (user: { inherit host user; }) (lib.attrValues host.users);
>   default = lib.singleton;
> };
> ```
>
> These are now declared as:
> ```nix
> den.policies.host-to-users = {
>   from = "host"; to = "user";
>   resolve = { host }: map (user: { inherit host user; }) (lib.attrValues host.users);
> };
> den.policies.host-to-default = {
>   from = "host"; to = "default";
>   resolve = lib.singleton;
> };
> ```

### Activation model

Policies are *available* when their module is imported (from den batteries or external flakes) but *active* only when enabled. Same pattern as den batteries (`den.provides.mutual-provider`).

**Four activation levels:**

```nix
# 1. Den core — fundamental policies, always active.
#    Defined in modules/policies/host.nix etc., users don't touch these.
#    (host→user, host→default, user→default, user→home chain)

# 2. den.default.policies — cross-cutting, user opt-in.
#    Applies to all contexts.
den.default.policies = [
  den.policies.host-to-users
  den.policies.mutual-provider
];

# 3. den.schema.<kind>.policies — scoped to entity kind, user opt-in.
#    Applies when resolving any entity of that kind.
den.schema.host.policies = [
  den.policies.host-to-peers
];

# 4. Entity instance — scoped to a specific entity.
#    Applies only when resolving this particular host/user/etc.
den.hosts.x86_64-linux.igloo.policies = [
  den.policies.host-to-peers
];
```

> **Update (2026-04-21):** Level 3 changed from `den.ctx.<kind>.policies` to `den.schema.<kind>.policies` — `den.ctx` is being fully removed. Entity-scoped policy activation lives on the entity schema (the type declaration), consistent with Haskell's typeclass model where capabilities are declared on the data type.

**Battery pattern:** a battery module defines the policy in `den.policies.*` (available). The user enables it via `den.default.policies` or `den.schema.<kind>.policies` (active). Same as `den.stages.user.includes = [ den.provides.mutual-provider ]` today.

**External flakes:** an external flake can provide policies in its flake module. The user imports the flake and enables its policies — no implicit activation from imports.

### Formal declarations only

> **Update (2026-04-21):** With `den.ctx` fully removed, all policies are declared formally via `den.policies.*`. There is no inline sugar. This is simpler and avoids re-introducing the conflation of policies and behavior that `den.ctx` had.

**Formal** (`den.policies.*`) for: all policy declarations — reusable patterns, bidirectional policies, cross-cutting concerns (ACL), custom entity types, and simple transitions alike.

---

## Layer 2: Effect-Based Materialization

### Handler installation

At pipeline entry, all policies are compiled into per-policy effect handlers and installed via `scope.provide`:

```nix
handlers."host-to-users" = { param, state }:
  let targets = policy.resolve param.context;
  in { resume = targets; inherit state; };
```

### Transition dispatch

The transition handler sends per-policy named effects instead of reading `into`:

```nix
# Before (reads into data):
intoResult = aspect.into currentCtx;
transitions = flattenInto intoResult [];

# After (sends per-policy effects):
fx.send "host-to-users" { host = currentHost; }
# handler responds with [{ host, user = tux }, { host, user = alice }, ...]
```

### Routing decision

The transition handler determines routing based on the policy's target entity type relative to the current scope:

| Target type vs current scope | Routing | Mechanism |
|---|---|---|
| **Child** (different type, resolves within current pipeline) | Resolve locally | Tag with `__scopeHandlers`, call `aspectToEffect` |
| **Sibling** (same type as emitter, needs separate pipeline) | Collect for phase 2 | Emit `provide-to` effect (Layer 3) |

One handler per policy — traces show exactly which fired.

### ctxApply dissolves

> **Update (2026-04-21):** With `den.ctx` fully removed, there is no `den.ctx.flake` to reference. Pipeline entry receives the entity's aspect and context directly.

Pipeline entry becomes explicit — no callable ctx nodes:

```nix
# Before:
den.lib.aspects.resolve "flake" (den.ctx.flake { })

# After:
den.lib.aspects.resolve "flake" {
  aspect = den.aspects.flake;
  context = { };
  policies = applicablePolicies;
}
```

No `__functor` on any submodule-evaluated attrset. Scope binding (stamping `__scopeHandlers` from context) moves into the policy handler.

---

## Layer 3: Cross-Entity Routing (provide-to)

### The problem

When a policy targets a sibling entity (host→peer, where peer is another host), the resolved modules must land in the *peer's* config, not the current host's. The current pipeline's `emit-class` always goes to the root classCollector for the *current* pipeline run.

### Two-phase resolution

**Phase 1 — Normal resolution with cross-entity collection**: the pipeline runs normally. When the transition handler detects a sibling target (same entity type as emitter), it collects resolved modules in `state.provideTo` instead of emitting to the current classCollector:

```nix
provideToHandler = {
  "provide-to" = { param, state }: {
    resume = null;
    state = state // {
      provideTo = _: ((state.provideTo or (_: [])) null) ++ [param];
    };
  };
};
```

**Phase 2 — Distribution at orchestration level**: after phase 1 completes, the orchestration layer reads `state.provideTo null`, groups emissions by target entity, and injects them as additional modules when building target configs:

```
flake
  → flake-system (per system)
    → flake-os (per host)         # Phase 1: resolve each host, collect provide-to
    → distribute provide-to       # Phase 2: group by target, inject into targets
    → build target configs        # Targets receive injected modules
```

### Cycle prevention

- Phase 1 completes fully before phase 2 begins
- Phase 2 pipeline runs do NOT install `provideToHandler`
- Injected modules cannot emit further `provide-to` — prevents cycles structurally

### Phase 2 semantics

Phase 2 does NOT re-run the aspect pipeline. It injects `provide-to` modules directly into the target's module list — alongside modules from phase 1. Injected modules are dendritic (have class keys). Zero pipeline runs in phase 2 — only module list concatenation.

For configs without cross-entity contributions, phase 2 is a no-op.

### Effect shape

```nix
{
  target = "<entity-kind>" | ["<kind>" ...];  # single or multi-hop
  content = <aspect attrset with dendritic class keys>;
  emitterCtx = "<entity-kind>";  # set by handler, not user
}
```

Multi-hop targets walk the policy graph, fanning out at each step.

### Forward mechanism migration

`forward.nix` currently starts fresh `den.lib.aspects.resolve` calls, losing parent context. With provide-to:
- Source class resolution stays in phase 1
- mapModule/guards/adapters stay in phase 1
- The final result emits `provide-to` instead of being returned directly
- The orchestration layer injects it into the target

The adapter/guard/mapModule machinery stays intact. Only routing changes.

---

## Examples

### Kubernetes cluster

```nix
den.policies.cluster-to-nodes = {
  from = "cluster";
  to = "node";
  resolve = { cluster }: cluster.nodes;
};

den.policies.cluster-to-services = {
  from = "cluster";
  to = "service";
  resolve = { cluster }: lib.attrValues cluster.services;
};

den.policies.service-to-storage = {
  from = "service";
  to = "node";
  resolve = { service, cluster }:
    builtins.filter
      (node: builtins.any (t: t == service.storageClass) (node.taints or []))
      cluster.nodes;
};
```

### ACL resolution (end-to-end)

This example shows the power of resolve functions querying external registries — not just the source entity's own fields. The environment owns user access bindings. The host doesn't declare its users; the policy computes them from the environment's ACL.

**Entity declarations:**

```nix
# Users are a global registry — not owned by any host
den.users.sini = { identity.displayName = "Jason Bowman"; };
den.users.json = { identity.displayName = "Jason"; };

den.groups = {
  system-access      = { scope = "system"; };
  workstation-access  = { scope = "system"; members = [ "system-access" ]; };
  admins = { scope = "kanidm"; };
  wheel  = { scope = "unix"; };
};

# Environment binds users to groups — the access policy
den.environments.prod = {
  system-access-groups = [ "system-access" ];
  access.sini = [ "admins" "wheel" "system-access" ];
  access.json = [ "admins" ];  # admin, no login — no system-scoped group
};

# Host declares its environment and additional gates. No users field.
den.hosts.x86_64-linux.cortex = {
  environment = "prod";
  system-access-groups = [ "workstation-access" ];
};
```

**Policies:**

```nix
# Step 1: environment fans out to its hosts
den.policies.environment-to-hosts = {
  from = "environment";
  to = "host";
  resolve = { environment }:
    let allHosts = lib.concatMap lib.attrValues (lib.attrValues den.hosts);
    in map (host: { inherit environment host; })
      (builtins.filter (h: h.environment == environment.name) allHosts);
};

# Step 2: host resolves users from the ENVIRONMENT's access registry.
# The resolve function queries den.users (global registry) filtered by
# the environment's ACL — not host.users.
den.policies.host-acl-users = {
  from = "host";
  to = "user";
  resolve = { host, environment }:
    let
      gates = lib.unique (
        (environment.system-access-groups or [])
        ++ (host.system-access-groups or [])
      );
      qualifies = username:
        let
          direct = environment.access.${username} or [];
          resolved = transitiveMembers den.groups direct;
          systemScoped = builtins.filter
            (g: (den.groups.${g}.scope or "") == "system") resolved;
        in builtins.any (g: builtins.elem g gates) systemScoped;
    in
    lib.concatMap (username:
      let
        direct = environment.access.${username} or [];
        resolved = transitiveMembers den.groups direct;
        systemScoped = builtins.filter
          (g: (den.groups.${g}.scope or "") == "system") resolved;
        unixGroups = builtins.filter
          (g: (den.groups.${g}.scope or "") == "unix") resolved;
        hasLogin = builtins.any (g: builtins.elem g gates) systemScoped;
      in
      lib.optional hasLogin {
        inherit environment host;
        user = den.users.${username};
        # Enriched context — derived ACL data rides alongside the entity
        user-acl = {
          enable = true;
          systemGroups = map (g: g) unixGroups;
          loginGroups = systemScoped;
        };
      }
    ) (builtins.attrNames environment.access);
};
```

**Consuming the enriched context** — parametric aspects receive `user-acl` alongside `user`. The policy can also inject aspects into the resolution by including them in the context:

```nix
# Base user account setup — consumes ACL data
den.aspects.user-accounts = { host, user, user-acl, ... }: {
  nixos.users.users.${user.name} = {
    isNormalUser = true;
    extraGroups = user-acl.systemGroups;
  };
};

# Admin role aspect — sudo, ssh, monitoring tools
den.aspects.admin-role = { host, user, ... }: {
  nixos.security.sudo.extraRules = [{
    users = [ user.name ];
    commands = [{ command = "ALL"; options = [ "NOPASSWD" ]; }];
  }];
  homeManager.programs.ssh.enable = true;
};
```

The policy attaches role aspects via `includes` in the enriched context:

```nix
# In the resolve function, include role-based aspects
lib.optional hasLogin {
  inherit environment host;
  user = den.users.${username};
  user-acl = { ... };
  # Role aspects injected by the policy based on group membership
  includes =
    lib.optional (builtins.elem "admins" direct) den.aspects.admin-role
    ++ lib.optional (builtins.elem "wheel" unixGroups) den.aspects.sudo-config;
};
```

The pipeline processes these `includes` alongside the user's own aspect includes — the policy controls not just *who* participates but *what roles they get*.

**How enriched context becomes parametric args:** the transition handler passes the entire context attrset from `resolve` to `constantHandler`, which creates a named effect handler for every key. `scope.provide` installs them. When `bind.fn` resolves `{ user, user-acl, ... }:`, it sends `"user-acl"` as an effect — the handler resumes with the value. Any key in the resolve output is automatically available as a parametric arg to downstream aspects.

**Pipeline context accumulation:**

```
entry: { environment = prod }
  ↓ environment-to-hosts
  { environment = prod, host = cortex }
    ↓ host-acl-users (queries den.users via environment.access)
    { environment = prod, host = cortex, user = sini,
      user-acl = { enable = true; systemGroups = ["wheel" "podman" ...]; } }
    # json does NOT appear — no system-scoped group intersects gates
```

The host never declared `users.sini`. The policy computed both the user set AND per-user derived data (groups, enable) from the environment's ACL and the global user registry. Downstream aspects consume `user-acl` as a parametric arg — the ACL resolution is decoupled from the NixOS config it produces.

### Cross-host fleet `/etc/hosts`

```nix
# Peer policy: each host fans out to sibling hosts.
# to = "host" (same type as emitter) triggers provide-to routing.
# The context key is "peer" — an alias for the target entity in this
# policy. The routing decision uses the `to` declaration ("host"),
# not the context key name.
den.policies.host-to-peers = {
  from = "host";
  to = "host";
  as = "peer";  # context key name (default: same as `to`)
  resolve = { host }:
    map (peer: { inherit peer; })
      (filter (h: h.name != host.name) (attrValues den.hosts.${host.system}));
};

# Aspect: each host publishes its IP to peers via parametric include.
# The { peer } arg is resolved via the host-to-peers policy.
# Because peer is a sibling host, the transition handler routes this
# through provide-to — the resolved nixos module lands in the peer's config.
den.aspects.fleet-hosts = { host, ... }: {
  includes = [
    ({ peer, ... }: {
      nixos.networking.extraHosts = "${host.meta.ip} ${host.name}";
    })
  ];
};

# Result: igloo gets "10.0.0.2 iceberg", iceberg gets "10.0.0.1 igloo"
```

Note: both examples use entity metadata (`host.meta.ip`), not resolved NixOS config. No fixpoint risk — provide-to closures capture source entity data.

### Capability-labeled provides (haproxy backends)

Provide-to emissions don't have to be NixOS modules. They can be **pure data labeled with a capability name**. The target aspect consumes the data and decides how to turn it into config.

Two capability types at the same level:
- **ClassModule** (`nixos`, `homeManager`, custom classes): module-shaped, emitted to class collector, target processes via NixOS module system.
- **SemanticData** (`http-backends`, `dns-records`, custom labels): pure data, accumulated across sources, target aspect consumes via parametric args.

Both flow through provide-to. Structural detection distinguishes them: module-shaped values → class emission. Data values → named capability accumulation.

**Policy wiring:**

```nix
# The host-to-peers policy resolves which hosts are peers.
# This was defined in the fleet example above.
# web1 and web2 both see lb as a peer (and vice versa).
den.policies.host-to-peers = {
  from = "host";
  to = "host";
  as = "peer";
  resolve = { host }:
    map (peer: { inherit peer; })
      (filter (h: h.name != host.name) (attrValues den.hosts.${host.system}));
};
```

**Source aspects** — webservers declare they're http backends (pure data, not NixOS config). Each site on the same host is a separate aspect. They don't know haproxy config structure:

```nix
den.aspects.example-site = { host, ... }: {
  nixos.services.nginx.virtualHosts."example.com".locations."/".proxyPass =
    "http://localhost:8080";

  # Labeled provide: "I am an http-backend" (SemanticData capability)
  provide-to.http-backends = [
    { address = host.meta.ip; port = 8080; vhost = "example.com"; }
  ];
};

den.aspects.foobar-site = { host, ... }: {
  nixos.services.nginx.virtualHosts."foobar.com".locations."/".proxyPass =
    "http://localhost:8081";

  provide-to.http-backends = [
    { address = host.meta.ip; port = 8081; vhost = "foobar.com"; }
  ];
};
```

**Target aspect** — loadbalancer is parametric on `http-backends`. `bind.fn` resolves the capability from accumulated provide-to data across all source peers:

```nix
den.aspects.loadbalancer = { http-backends, ... }: {
  nixos.services.haproxy.enable = true;
  nixos.services.haproxy.frontends = lib.listToAttrs (map (b: {
    name = b.vhost;
    value.backends = [{ inherit (b) address port; }];
  }) http-backends);
};
```

**Host wiring:**

```nix
den.hosts.x86_64-linux.web1 = {};
den.hosts.x86_64-linux.web2 = {};
den.hosts.x86_64-linux.lb = {};

den.aspects.web1.includes = [ den.aspects.example-site den.aspects.foobar-site ];
den.aspects.web2.includes = [ den.aspects.example-site ];  # only example.com
den.aspects.lb.includes = [ den.aspects.loadbalancer ];
```

**Data flow:** the `host-to-peers` policy fans out from each host to all siblings. When `web1` resolves, its `example-site` and `foobar-site` aspects emit `provide-to.http-backends` data. The transition handler detects sibling routing (`to = "host"` = same type) and collects the data in `state.provideTo`. In phase 2, the accumulated `http-backends` data from all sources is merged and installed as a handler on the target (`lb`). When `lb`'s `loadbalancer` aspect resolves, `bind.fn` resolves `{ http-backends }` from the installed handler.

**SemanticData accumulation mechanism:** between phase 1 and phase 2, the orchestration layer groups `provide-to` emissions by target entity and label. For each target, accumulated data for each label is installed as a `constantHandler` binding — so `http-backends` becomes a named effect that resumes with the merged list. This is the same mechanism used for context args (`host`, `user`) but with data instead of entities.

The label (`http-backends`) IS the capability name. Sources append to it, the target requests it by name as a function arg.

This separates concerns cleanly:
- **Source** declares what it IS (an http backend) with structured data
- **Target** declares what it NEEDS (http-backends) and produces config from it
- Neither knows the other's internal configuration structure
- The policy determines which sources are visible to which targets

---

## Backwards compatibility

> **Update (2026-04-21):** `den.ctx` is being fully removed (see `2026-04-21-ctx-as-classes-design.md`). Deprecation shims are provided during migration phases but will be removed.

**`den.ctx.host.into.user = fn`** → Phase 2 deprecation shim forwards to `den.policies`. Phase 3 removes the shim entirely.

**`den.ctx.*.nixos/includes`** (scoped behavior on ctx nodes) → Phase 2 deprecation shim forwards to `den.stages.*`. Phase 3 removes.

**`den.hosts.x86_64-linux.igloo.users.tux = {}`** → unchanged: `users` stays as structure. The `host-to-users` policy reads it.

**`den.ctx.host { host = config; }`** → removed. Pipeline entry is explicit (no callable ctx nodes).

**`provides.to-users` / `provides.to-hosts`** (mutual-provider.nix) → left as-is initially. Operates within host-user mutual pipeline. A future migration could unify with provide-to.

---

## What stays

- Entity types and schema (`den.hosts`, `den.users`, `den.homes`)
- `den.aspects` for aspect definitions (unchanged — reusable behavior stays here)
- `den.schema.*` for validation
- The fx pipeline and effects system
- `__scopeHandlers` for context propagation
- `__fn`/`__args` parametric type for pipeline wrappers

## What changes

- `den.ctx` → fully removed (transitions → `den.policies`, scoped behavior → `den.stages`)
- `ctxApply` → dissolved; scope binding moves into policy handler
- `__functor` on ctxSubmodule → removed (no more callable ctx nodes)
- Transition handler → sends per-policy named effects
- Transition handler → routing decision (child vs sibling)
- Pipeline entry → explicit policy handler installation
- `forward.nix` → emits `provide-to` instead of fresh pipeline runs

## What's new

- `den.policies` option for formal policy declarations
- `den.stages` for scoped behavior bindings (replaces `den.ctx.*.nixos/includes`)
- `den.default.policies` for cross-cutting policy activation
- `den.schema.<kind>.policies` for entity-kind-scoped policy activation
- Per-policy effect handlers
- `provideToHandler` for cross-entity collection
- `distributeProvideTo` orchestration function

## Constraints

**No config references in provide-to closures.** Source entities must capture entity metadata (e.g., `host.meta.ip`), not evaluated NixOS config (e.g., `config.networking.hostName`). This prevents fixpoint evaluation between source and target configs. Violation of this constraint causes infinite recursion. This is intentional — `provide-to` is a one-way push mechanism, not a bidirectional fixed-point like NixOps' `nodes.*` references.

**Phase 2 module injection ordering.** Modules injected via provide-to are appended to the target's module list after locally-resolved modules. Ordering between provide-to sources follows the source entity's pipeline evaluation order (deterministic but arbitrary). Module correctness must not depend on injection order — use NixOS priorities (`mkDefault`, `mkForce`) if ordering matters.

## Observability

**Trace output**: one trace per policy handler fired, showing policy name and target count. Existing pipeline traces (compileStatic, classCollector, includeHandler) are unaffected.

**Inspection utility**: `den.lib.policies.inspect` — given an entity kind and context, returns all applicable policies and their resolved targets without running the full pipeline. Cheap (just calls `resolve` functions). Essential for debugging "why did host X get this module?"

```nix
# Debug: what policies fire for igloo?
den.lib.policies.inspect {
  kind = "host";
  context = { host = den.hosts.x86_64-linux.igloo; };
}
# → { host-to-users = [{ host, user = tux }, ...]; host-to-peers = [...]; }
```

## Implementation notes

**~~Ctx nodes need a `kind` identifier.~~** (Obsolete — `den.ctx` is being removed.) Policies declare `from`/`to` as strings that name entity kinds. The pipeline matches these against entity schema registrations. No ctx node labeling needed.

**`provide-to` must be in `structuralKeys`.** If an aspect declares `provide-to.http-backends`, the pipeline must NOT treat `provide-to` as a class key. Add `"provide-to"` to `structuralKeys` in `aspect.nix`. `compileStatic` extracts `provide-to` keys and emits them as `"provide-to"` effects.

**SemanticData accumulates between phases.** After phase 1, the orchestration layer groups `provide-to` emissions by target + label. For each target, labeled data is installed as `constantHandler` bindings on the target's phase 2 resolution. `bind.fn` resolves capability args (`http-backends`) the same way it resolves context args (`host`, `user`).

**forward.nix creates nested pipeline runs.** Currently `forwardItem` calls `den.lib.aspects.resolve` directly (fresh pipeline). Converting to `provide-to` requires forward to emit effects within the existing pipeline instead of spawning nested runs. This is feasible because forward IS called inside the pipeline (via `emitCrossProvider` in `transition.nix`), but the `aspects.resolve` call must become a `provide-to` emission.

## Future work

- **Policy composition operators** (union, intersection, difference) — not needed now but possible extension
- **Target-side opt-out** — entity declares `meta.policyExclusions` to refuse targeting by specific policies
- **Memoization guidance** for expensive `resolve` functions (compute mapping once at module eval time)
- **Bidirectional config access** — `provide-to` is one-way push; pulling target config requires fixed-point evaluation (NixOps model) which is explicitly out of scope

## What does NOT change

- The nix-effects trampoline — no changes needed
- Core effect handlers (constantHandler, classCollector, chainHandler, constraintRegistry)
- `aspectToEffect` / `compileStatic` compilation
- Single-entity resolution (only multi-entity orchestration changes)
- The `__ctxId` / `__parametricResolved` identity/dedup mechanism

## Rejected alternatives

### Inherited context chain

Thread parent context into child pipeline runs via an `inheritCtx` parameter. Simpler, but cycles are prevented by API discipline rather than by construction. Doesn't generalize to sibling-to-sibling forwarding.

### Simpler ctx threading

Fix `__parentCtx` propagation without new effects. Doesn't address cross-entity use case. The scope.provide migration subsumes this for parent-to-child propagation.

## Pipeline state additions

```nix
defaultState = {
  # ... existing (imports, pathSet, constraintRegistry, etc.) ...
  provideTo = _: [];  # Thunk chain — same pattern as imports
};
```

The thunk wrapping (`_: [...] ++ [param]`) prevents `deepSeq` from forcing aspect content that may reference lazy option defaults. `state.provideTo null` unwraps the chain.

## Implementation components

| Component | Location | Purpose |
|---|---|---|
| Policy option type | `nix/lib/policy-types.nix` (new) | Policy schema, registration |
| Policy module | `nix/nixModule/policies.nix` (new) | `den.policies` option |
| Per-policy handlers | `nix/lib/aspects/fx/handlers/policy.nix` (new) | Compile policies → handlers |
| `provideToHandler` | `nix/lib/aspects/fx/handlers/provide-to.nix` (new) | Cross-entity state collection |
| `transitionHandler` changes | `nix/lib/aspects/fx/handlers/transition.nix` | Routing decision (child vs sibling) |
| `distributeProvideTo` | `nix/lib/aspects/fx/pipeline.nix` or new | Phase 2 orchestration |
| Orchestration changes | `modules/outputs/osConfigurations.nix` | Phase 2 distribution after host resolution |
| Forward migration | `nix/lib/forward.nix` | Emit provide-to instead of fresh pipeline |

## Migration path

> **Update (2026-04-21):** This migration path is coordinated with the Data/Policies/Behavior separation spec (`2026-04-21-ctx-as-classes-design.md`), which defines three phases: Phase 1 (entity file reorganization), Phase 2 (introduce policies, deprecation shims for ctx), Phase 3 (full ctx removal).

1. Add `den.policies` option type and module
2. Implement per-policy effect handlers
3. Move `den.ctx.*.into` declarations to `den.policies` (with deprecation shims on `den.ctx`)
4. Move `den.ctx.*` scoped behavior (`.nixos`, `.includes`) to `den.stages.*` (with deprecation shims)
5. Update transition handler for effect-based dispatch + routing decision
6. Add `provideToHandler` to `defaultHandlers`
7. Add `distributeProvideTo` orchestration function
8. Refactor `osConfigurations.nix` for phase 2 distribution
9. Remove `den.ctx` entirely (Phase 3 — remove shims, delete ctx-types.nix, ctx-apply.nix, nixModule/ctx.nix)
10. Migrate `forward.nix` to emit provide-to
11. Add cross-entity tests (fleet, haproxy)

## Success criteria

- All existing tests pass (zero regressions)
- Policies express current host→user→home transitions
- Cross-entity forwarding works (fleet `/etc/hosts`, haproxy backend tests)
- `forward.nix` migration: results distributed via provide-to
- No `__functor` on any submodule-evaluated attrset
- No performance regression for configs without cross-entity contributions

## scope.provide vs scope.stateful

The pipeline uses `scope.provide` (not `scope.stateful`) for installing context handlers. The distinction matters:

- `scope.stateful` does state fork-join via `state.update`. Inner state overwrites outer state — class collector mutations are lost (79% event drop observed in testing).
- `scope.provide` installs handlers via `rotate` with state-discarding return. Rotated effects modify outer state directly. No state overwrite.
- `scope.run` discards inner state entirely. Equivalent to `scope.provide` mechanically but signals "isolated execution" not "augment handlers."

The pipeline needs `scope.provide` because constantHandler bindings are stateless (resume with constant, pass state through) while emit-class/emit-include effects that rotate outward must modify the shared root state.

## Pre-work completed

- `scope.provide` migration: all `scope.stateful` calls in aspect.nix, transition.nix, ctx-apply.nix replaced with `scope.provide` (commits `2b3ac91a`, `c9b428a4`)
- Parametric type separation: `__fn`/`__args` replace `__functor`/`__functionArgs` on pipeline wrappers
- `__scope`/`__parentScope` removal: derive from `__scopeHandlers` at point of use
- `resolvedCtx` removal: parametric shims stamp `__scopeHandlers` directly via `constantHandler`

---

## TL;DR

Four clean separations: **Data** (entity schemas — `den.schema.*`), **Policies** (topology — `den.policies`), **Stages** (scoped behavior bindings — `den.stages`), **Behavior** (Nix config classes — `den.aspects`). `den.ctx` is fully removed — it conflated policies, stages, and behavior.

**Policies** define topology — how entities connect. Policies are first-class data in `den.policies`, activated via `den.default.policies` or scoped to an entity kind/instance.

At pipeline time, policies compile into **per-policy named effect handlers**. The transition handler sends `"host-to-users"` instead of reading `into` data. One handler per policy = clear traces.

When a policy targets a **sibling entity** (same type as emitter, e.g., host→peer-host), resolved modules route through a **two-phase provide-to mechanism**: phase 1 collects, phase 2 distributes to target configs. No fixed-point — source captures entity metadata, not evaluated config.

`__functor` is removed from all submodule-evaluated attrsets. `ctxApply` is dissolved — scope binding moves into the policy handler. See `2026-04-21-ctx-as-classes-design.md` for the full ctx removal plan and user migration guide.
