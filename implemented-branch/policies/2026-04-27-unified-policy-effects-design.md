# Unified Policy Effects Design

## Status: Implemented

Shipped on `feat/fx-pipeline` across Phases A-E. All policies are plain functions returning typed effects. `policyFns→policies` rename, `resolve.to` API, `__functor` removal all landed.

## Core Insight

Policies are effect-producing functions. Their function signature determines when they activate (context matching), and they return typed policy effects that the pipeline processes. Evaluation is commutative — order between policies doesn't matter; dependencies are implicit in signatures.

Function signatures are the universal API. Aspects declare what context they need (`{ host, user, secrets }:`), the pipeline matches arg names to handlers — whether the value comes from an entity binding, trait collection, or policy injection.

## Policy Shape

Policies are functions from context to a list of typed policy effects:

```nix
den.policies.<name> = { host, user, ... }: [
  (policy.resolve { ... })
  (policy.include den.aspects.foo)
  (policy.exclude den.aspects.bar)
];
```

Effect constructors (`policy.resolve`, `policy.include`, `policy.exclude`) are provided via `den.lib.policy` and return tagged attrsets (e.g., `{ __policyEffect = "resolve"; value = {...}; }`). The pipeline dispatches on the tag.

### Migration from Current Policy Type

The current `policyType` submodule (`from`, `to`, `as`, `resolve`, `aspects`, `handlers`, `isolateFanOut`, `_core`) is replaced:

| Current field | New model |
|---------------|-----------|
| `from` / `to` | Implicit in function signature — `{ host }:` replaces `from = "host"` |
| `as` | The key name in `policy.resolve` — `policy.resolve { user = ...; }` |
| `resolve` | The function body itself — policies ARE resolve functions |
| `aspects` | `policy.include` effects in the return list |
| `handlers` | Aspect-included handlers or `scope.provide` in the pipeline |
| `isolateFanOut` | Default behavior for `policy.resolve` — each resolve creates an isolated sub-pipeline. Opt-out via `policy.resolve.shared` if needed (to be specified during implementation) |
| `_core` | Core policies are registered in `den.lib.corePolicies` and always active. User policies are registered in `den.policies`. Both use the same effect types. |

Entity kind is inferred from the `policy.resolve` binding key: `policy.resolve { user = ...; }` creates a context with key `user`, which the pipeline matches against `den.schema.user` if it exists. Keys not in `den.schema` (like `flake-system`) are valid — they create context scopes without entity schema association.

### Effect Types

Three effect types. No others are needed — substitution is expressed as exclude + include.

**`policy.resolve`** — Create a new context scope (fan-out). Each resolve effect creates a **parallel branch** — a sibling context with the new bindings merged into the parent context. Multiple `policy.resolve` effects from the same or different policies create multiple branches. For colliding keys, the new value shadows the existing value. This matches handler semantics (`scope.provide` overlays onto existing handlers). `policy.resolve {}` (empty bindings) is a no-op.

```nix
# Fan-out: produce user contexts from host — each user is a parallel branch
den.policies.host-users = { host }:
  map (user: policy.resolve { inherit user; })
    (builtins.attrValues host.users);
```

**`policy.include`** — Inject an aspect into the current resolution context. Accepts both aspect references and inline attrsets (coerced to anonymous aspects, same as how `includes = [ { nixos.foo = true; } ]` works today).

```nix
# Conditional injection: admin users get extra aspects
den.policies.admin-users = { host, user }:
  lib.optionals (builtins.elem "wheel" user.groups) [
    (policy.include den.aspects.sudo)
    (policy.include den.aspects.admin-tools)
  ];

# Inline attrset — coerced to anonymous aspect
den.policies.igloo-to-alice = { host, user }:
  lib.optional (user.name == "alice")
    (policy.include { homeManager.programs.vim.enable = true; });
```

**`policy.exclude`** — Remove/gate an aspect from the current resolution tree. Scoping is context-matched: the exclude applies to all contexts matching the policy's function signature. A policy taking `{ host }:` applies per-host — server hosts exclude an aspect, non-server hosts don't. The exclude propagates into sub-contexts (user, home) created from that host. Excludes don't leak into unrelated contexts (other hosts, standalone homes).

```nix
# Constraint: no home-manager on servers
den.policies.no-server-hm = { host }:
  lib.optional (host.hasAspect den.aspects.server)
    (policy.exclude den.aspects.home-manager-base);
```

### Chained Policies

Policies that produce new contexts enable further policies to match. The pipeline iterates until stable:

```nix
# Produces user contexts
den.policies.host-users = { host }:
  map (user: policy.resolve { inherit user; })
    (builtins.attrValues host.users);

# Runs after user contexts exist — produces additional users
den.policies.sini-test = { host, user }:
  lib.optional (user.name == "sini")
    (policy.resolve { user = mkUser "sini-test" {}; });

# Runs for every user context (including sini-test)
den.policies.admin-users = { host, user }:
  lib.optionals (builtins.elem "wheel" user.groups) [
    (policy.include den.aspects.sudo)
  ];
```

## Pipeline Evaluation Model

Policy effects are processed in two phases within each entity resolution:

**Phase A — Include/exclude (tree-walk time):** After an entity's aspects resolve but before transitions fire, the pipeline dispatches matching aspect-included policies and processes `policy.include` / `policy.exclude` effects immediately via `emit-include` / `register-constraint`. This ensures injected aspects participate in the entity's tree-walk — critically, they're visible to class forwarding sub-pipelines (e.g., HM forward) that collect emissions from the entity's resolution.

**Phase B — Resolve (transition time):** `policy.resolve` effects are processed during transitions, creating new parallel context branches. Each branch triggers child entity resolution (same as today's fan-out). The transition handler iterates:

1. Match policies by function signature to available context
2. Collect all returned effects
3. Apply `policy.include` / `policy.exclude` effects immediately (Phase A, during entity tree-walk)
4. Process `policy.resolve` effects — each creates a new parallel context branch (merged into parent context)
5. Re-match policies against new contexts (including newly discovered aspect-included policies)
6. Iterate until no new contexts AND no new policies appear (fixed-point)

**Why two phases:** Class forwarding batteries (home-manager, maid) run sub-pipelines during entity resolution to collect class emissions and place them at the correct NixOS module path (e.g., `home-manager.users.${userName}`). If `policy.include` effects were deferred to transition time, the injected aspects would miss the forward sub-pipeline. Processing includes during tree-walk ensures they're captured.

Evaluation is **commutative** at each context level — the order policies are evaluated does not affect the result. Dependencies between policies are expressed entirely through function signatures (context requirements), not through explicit ordering.

### Fixed-Point Termination

Context dedup determines what counts as "new": two resolve effects producing the same context bindings (by value identity) are deduplicated to one branch. This is the same mechanism as the existing `ctxSeen` handler. Termination is guaranteed because:

- The set of aspects is finite (each can only be included once due to dedup)
- Each aspect can only contribute a finite number of policies
- Each policy produces a finite set of resolve effects per invocation
- Context dedup prevents unbounded growth

### Shadowing Conflicts

If two policies at the same level both resolve with conflicting values for the same key (e.g., policy A resolves `{ secrets = x; }` and policy B resolves `{ secrets = y; }`), each creates a **separate parallel branch** — they don't merge with each other. Shadowing only applies within a single `policy.resolve` effect relative to its parent context. This preserves commutativity.

## Traits as Context

Traits unify with entity context as "named values provided by handlers." The pipeline doesn't distinguish between entity args (`host`, `user`) and trait args (`secrets`, `impermanence`) — both are context values matched by function signature.

### Trait Emission

Aspects emit traits as structural keys:

```nix
# Static trait (Tier 1)
den.aspects.wifi.secrets = [ "wifi-password" ];

# Parametric trait (Tier 2) — scoped to pipeline context
den.aspects.steam.impermanence = { user }: {
  directories = [ (user.home + "/.local/share/Steam") ];
};
```

Trait keys are registered in `den.traits` with collection strategy:

```nix
den.traits.secrets = { collection = "list"; };
den.traits.impermanence = { collection = "list"; };
```

### Trait Consumer Deferral

Trait consumers defer until collection is complete — same mechanism as entity context deferral:

1. Trait arg handlers are **not installed** at pipeline start
2. Consumer aspects with `{ secrets }:` probe via `has-handler` — no handler exists — they defer
3. Peer aspects resolve, emitting traits into `state.traits`
4. Pipeline signals trait collection complete at current level
5. Trait handlers installed, deferred consumers drain with full collection

This mirrors entity context exactly: `{ user }:` defers until user context exists via `policy.resolve`; `{ secrets }:` defers until secrets are collected from peer emissions. Same `drain-deferred` code path.

### Cycle Detection

Circular trait dependencies (A emits X, consumes Y; B emits Y, consumes X) are handled:

- Both A and B defer because their consumed trait has no handler yet
- At drain time, neither has emitted (because neither has resolved)
- Pipeline detects: deferred consumers exist but no new emissions occurred — cycle error
- Existing dedup mechanisms (includeSeen, ctxSeen, pathSet) catch within-phase cycles

## Scoped Trait Filtering via Policy

Policies can shadow trait context for a subtree using `policy.resolve`. Because policies are parametric on context args (including traits), a policy that takes `{ secrets }:` defers until trait collection is complete — then fires and provides a filtered view to its subtree.

```nix
# All secrets collected at host level.
# This policy shadows secrets for the user subtree with a filtered view.
den.policies.user-secrets = { user, secrets }:
  let
    userSecrets = builtins.filter (s: s.owner == user.name) secrets;
  in
  [ (policy.resolve { secrets = userSecrets; }) ];
```

### How timing works

1. Peer aspects resolve, emitting `secrets` trait data
2. Pipeline signals trait collection complete, installs `secrets` handler
3. Policies with `{ secrets }` in signature were deferred — they now drain
4. `user-secrets` fires, returns `policy.resolve { secrets = userSecrets; }`
5. New context branch created with shadowed `secrets` binding (merged into parent)
6. All aspects in the user subtree that request `{ secrets }:` see the filtered value

The policy's own signature (`{ user, secrets }`) guarantees correct ordering: it can't fire until both user context exists AND secrets collection is complete. No explicit phase annotation needed — the deferral mechanism handles it.

### Retaining unfiltered access

If some consumers need the original unfiltered collection alongside the filtered view:

```nix
den.policies.user-secrets = { user, secrets }:
  let
    userSecrets = builtins.filter (s: s.owner == user.name) secrets;
  in
  [ (policy.resolve { secrets = userSecrets; all-secrets = secrets; }) ];
```

Downstream aspects choose which to consume: `{ secrets }:` for filtered, `{ all-secrets }:` for full.

### Enriching and transforming trait data

Policies can do more than filter — they can enrich, normalize, or transform collected trait data before downstream consumers see it. The consumer's function signature stays the same; the policy controls what value it receives.

```nix
# Aspects emit raw impermanence paths
den.aspects.steam.impermanence = { user }: {
  directories = [ (user.home + "/.local/share/Steam") ];
};

den.aspects.firefox.impermanence = { user }: {
  directories = [ (user.home + "/.mozilla") ];
  files = [ (user.home + "/.mozilla/firefox/profiles.ini") ];
};

# Policy enriches: merge lists, add metadata, inject defaults
den.policies.impermanence-enrichment = { user, impermanence }:
  let
    # Merge all directory/file lists from collected trait emissions
    allDirs = lib.concatMap (e: e.directories or []) impermanence;
    allFiles = lib.concatMap (e: e.files or []) impermanence;

    # Enrich: add XDG base dirs that every user gets
    enriched = {
      directories = allDirs ++ [
        (user.home + "/.config")
        (user.home + "/.local/state")
      ];
      files = allFiles;
      user = user.name;
      home = user.home;
    };
  in
  [ (policy.resolve { impermanence = enriched; }) ];

# Consumer sees the enriched, merged result — not raw emissions
den.aspects.impermanence-module = { impermanence, host }: {
  nixos.environment.persistence."/persist".users.${impermanence.user} = {
    inherit (impermanence) directories files home;
  };
};
```

The raw trait emissions are a list of partial attrsets. The policy consumes the list, merges it, adds defaults, and shadows `impermanence` with a single enriched attrset. The consumer doesn't need to know about merging or defaults — it receives a ready-to-use value.

This pattern applies broadly:
- **Normalization**: convert heterogeneous trait emissions into a uniform schema
- **Validation**: reject or warn on trait data that doesn't meet requirements
- **Aggregation**: merge lists, deduplicate, compute summaries
- **Injection**: add default values, environment-specific overrides, or metadata

### Centralized responsibility

This pattern enables security policies to control data flow without consumer cooperation:
- Secrets filtering owned by a single policy, not scattered across modules
- Consumers don't need to know filtering exists — they see `{ secrets }:` and get the appropriate view
- Policy is auditable: visible in traces with provenance

## Cross-Entity Trait Flow (Two-Phase Architecture)

Cross-entity configuration requires two phases to prevent cycles. This architecture is unchanged; the payload becomes trait state instead of freeform labeled data.

### Phase 1: Entity Pipeline Resolution

Each entity's pipeline runs independently:
- Aspects resolve, emitting traits into `state.traits`
- Trait consumers defer, then drain with collected data
- Policy fan-out creates sub-pipelines per context scope (each `policy.resolve` branch runs in an isolated sub-pipeline)
- Sub-pipeline trait state is captured at completion

### Phase 2: Cross-Entity Distribution

After all entity pipelines complete:
- Sub-pipeline trait collections are grouped by target entity
- Merged trait state is injected into target entity's context as handler bindings
- Cross-entity trait data is available only to **Tier 3 consumers** (deferred traits evaluated inside `evalModules` after the pipeline completes). Pipeline-time consumers (Tier 1/2) that drained in phase 1 see only within-entity data. This is consistent with the current model where `provide-to` data arrives at module evaluation time.

### Example: Backup Server

```nix
# Client hosts emit trait — unaware of cross-entity routing
den.aspects.postgres-host.backup-targets = {
  paths = [ "/var/lib/postgresql" ];
};

# Policy creates the cross-entity fan-out
den.policies.backup-clients = { host }:
  map (client: policy.resolve { backup-client = client; })
    (getBackupClients host);

# Consumer receives collected trait as context arg
# Same syntax whether within-entity or cross-entity
den.aspects.backup-server = { backup-targets, host }: {
  nixos.services.restic.backups = mkBackups backup-targets;
};
```

The policy doesn't explicitly route traits — traits collected in the sub-pipeline flow to phase 2 automatically. The policy creates the context scope; trait transport is a pipeline concern.

## Replacing mutual-provider

The current `mutual-provider` battery implements bidirectional host-user configuration via nested `provides` attributes and explicit routing:

```nix
# Today: manual nesting with provides syntax
den.schema.user.includes = [ den.provides.mutual-provider ];

den.aspects.igloo = {
  provides.alice.homeManager.programs.vim.enable = true;
  provides.to-users = { user, ... }: {
    homeManager.programs.helix.enable = user.name == "alice";
  };
};

den.aspects.alice = {
  provides.igloo.nixos.programs.emacs.enable = true;
  provides.to-hosts = { host, ... }: {
    nixos.programs.nh.enable = host.name == "igloo";
  };
};
```

In the unified model, mutual configuration is expressed as **aspect-included policies**. The `provides` structural key is replaced by policies that inject inline aspects into peer contexts:

```nix
# Host provides config to specific user (targeted)
den.aspects.igloo = {
  policies.to-alice = { host, user }:
    lib.optional (user.name == "alice")
      (policy.include { homeManager.programs.vim.enable = true; });

  # Host provides config to all users (broadcast)
  policies.to-users = { host, user }:
    [ (policy.include {
        homeManager.programs.helix.enable = user.name == "alice";
      }) ];
};

# User provides config to specific host (targeted)
den.aspects.alice = {
  policies.to-igloo = { host, user }:
    lib.optional (host.name == "igloo")
      (policy.include { nixos.programs.emacs.enable = true; });

  # User provides config to all hosts (broadcast)
  policies.to-hosts = { host, user }:
    [ (policy.include {
        nixos.programs.nh.enable = host.name == "igloo";
      }) ];
};
```

**Targeted vs broadcast** is expressed through policy guards — `lib.optional (user.name == "alice")` for targeted, unconditional for broadcast. No separate mechanism needed.

**Important distinction: provides vs traits.** `provides` forwards class configuration (inline aspects with `nixos`/`homeManager` keys) — these become `policy.include` effects (inline attrsets are coerced to anonymous aspects). Traits are for aggregated data (`secrets`, `impermanence`) consumed by parametric aspects via function signature. Different concerns, same pipeline.

### What this eliminates

- **`mutual-provider` battery**: replaced by aspect-included policies on each aspect
- **`provides` structural key on aspects**: replaced by `policies` structural key with `policy.include`
- **`find-mutual` / `to-hosts` / `to-users` routing logic**: replaced by policy guards
- **`mutual-user-user` cross-user routing**: policies naturally run for all user-user combinations when context matches
- **`mutual-standalone-home` special case**: handled by home-host policy with same `{ home }:` signature

### Why this is simpler

1. **No special routing battery** — aspects declare their own cross-entity policies inline
2. **Targeted vs broadcast is just a guard** — `lib.optional (name == "alice")` vs unconditional
3. **Same effect types everywhere** — `policy.include` for injection, `policy.resolve` for fan-out
4. **Self-contained** — excluding an aspect removes its policies too; no orphaned routing

## Aspect-Included Policies

Aspects can include policies via the `policies` structural key. The policy is still a first-class policy — visible in traces, typed with standard effect types, evaluated through the same fixed-point iteration. The only difference from top-level `den.policies` is provenance: the policy was installed because the aspect was included.

```nix
# Battery is self-contained: one include brings routing + integration
den.aspects.home-manager-battery = {
  policies.user-to-home = { host, user }:
    map (home: policy.resolve { inherit home; })
      (getHomes user);

  includes = [ den.aspects.hm-base ];
};

# MicroVM battery installs its own routing
den.aspects.microvm-battery = {
  policies.host-to-vm = { host }:
    map (vm: policy.resolve { inherit vm; })
      (getVMs host);

  includes = [ den.aspects.microvm-base ];
};
```

### Semantics

- **Discovery**: policies are discovered as aspects resolve — the pipeline processes them through the same fixed-point iteration as top-level policies. The fixed-point converges on both context stability AND policy set stability.
- **Provenance**: traced as "policy X installed by aspect Y" for auditability
- **Removal**: excluding an aspect (via `policy.exclude`) removes its included policies too. If the excluded aspect's policies have already produced resolve effects in an earlier iteration, those effects are rolled back (same mechanism as the existing `include-unseen` rollback for excluded aspects).
- **No special mechanism**: aspect-included policies use the same `policies` structural key, same effect types, same evaluation model — they're just included rather than declared at the top level

### Policies vs Aspects: Preserved Distinction

Aspects and policies remain distinct concepts:
- **Aspects** are the configuration layer — composable units that emit classes, traits, includes, and optionally include policies
- **Policies** are the routing layer — effect-producing functions that control context scope, fan-out, constraints, and aspect injection

An aspect can include a policy. A policy can include an aspect. But they serve different roles and are evaluated differently (aspects via tree-walk resolution, policies via signature-matched fixed-point iteration).

## Battery Migration

How existing batteries map to the unified model. Batteries fall into categories by complexity.

### Simple Batteries (unchanged shape)

These are plain aspects that take context and return class keys. No migration needed — they already fit the model.

| Battery | Signature | What it does |
|---------|-----------|-------------|
| `hostname` | `{ host }:` | Sets `networking.hostName` from host entity |
| `define-user` | `{ user }:` | Creates OS user account with home/name |
| `primary-user` | `{ user, host }:` | Adds wheel/networkmanager groups |
| `tty-autologin` | `__functor(username)` | Enables getty autologin |
| `user-shell` | `__functor(shell)` | Enables shell at OS + HM levels |

### Predicate Batteries (unchanged, traits optional)

`unfree` and `insecure` each have a factory + predicate-builder pair (4 files total, 2 logical batteries). The factory emits package names, the predicate-builder collects them into `nixpkgs.config`. These could optionally migrate to traits:

```nix
# Current: factory + auto-included predicate builder
den.provides.unfree [ "discord" "steam" ];
# unfree-predicate-builder collects via NixOS option, builds allowUnfreePredicate

# New (optional): trait-based collection
den.aspects.gaming.unfree-packages = [ "discord" "steam" ];
# pipeline collects trait, predicate builder consumes { unfree-packages }:
```

Not urgent — the current pattern works. Trait migration is a simplification, not a requirement.

### Forward Router Batteries (simplified)

`forward.nix` is a higher-order battery that routes one class to another (e.g., `homeManager` to `home-manager.users.<name>`). Used by `os-class`, `os-user`, `wsl`.

These continue to work as-is. The `forward` pattern is orthogonal to the policy/trait model — it's about class module routing, not entity/context routing.

| Battery | Routes | Used by |
|---------|--------|---------|
| `os-class` | `os` to `[nixos, darwin]` | Auto-included |
| `os-user` | `user` to `users.users.<name>` | home-manager battery |
| `wsl` | `wsl` to host class | Guarded by `host.wsl.enable` |

### Home Environment Batteries (major simplification)

`home-manager` and `maid` are the most complex batteries. Today they install policies + aspects + forwarding routers across multiple files. In the unified model, they become self-contained aspect-included policies:

```nix
# Current: spread across multiple modules
# - modules/aspects/provides/home-manager.nix (battery definition)
# - modules/policies/ (host-to-homeManager-users policy)
# - modules/aspects/ (homeManager-host-module, homeManager-user-forward aspects)
# - den.schema.host.imports + den.schema.host.policies (registration)

# New: self-contained battery
den.aspects.home-manager-battery = {
  # Routing policy included with the battery
  policies.host-to-hm-users = { host, user }:
    lib.optional (user.classes ? homeManager)
      (policy.resolve { home-env = makeHomeEnv user; });

  # Forwarding aspects included directly
  includes = [
    den.aspects.hm-host-forward   # homeManager class to home-manager.users.<name>
    den.aspects.hm-user-forward   # user homeManager to OS integration
  ];
};
```

One include brings the entire home-manager integration: routing policy, forwarding aspects, class setup. Removing the battery (`policy.exclude den.aspects.home-manager-battery`) removes everything.

Same pattern for `maid`:

```nix
den.aspects.maid-battery = {
  policies.host-to-maid-users = { host, user }:
    lib.optional (user.classes ? maid)
      (policy.resolve { home-env = makeMaidEnv user; });

  includes = [ den.aspects.maid-user-forward ];
};
```

### Cross-Entity Batteries (replaced by traits + policies)

**`mutual-provider`** is eliminated entirely. See "Replacing mutual-provider" section above. Bidirectional host-user config becomes aspect-included policies on each aspect. No battery needed.

**`host-aspects`** (projects host aspect's user-class config onto users) becomes a policy:

```nix
# Current: manual context injection with __scopeHandlers
den.provides.host-aspects = { host, user, ... }: ...;

# New: policy that includes host-provided user config
den.policies.host-aspects-to-users = { host, user }:
  let hostAspect = den.aspects.${host.aspect}; in
  lib.optional (hostAspect ? user-config)
    (policy.include hostAspect.user-config);
```

### Import Tree Battery (unchanged)

`import-tree` is a factory that recursively loads class-segregated `.nix` files from a directory. It's a pure include mechanism — no policies, no cross-entity routing. The `den.provides` namespace is retained for factory batteries (it is a top-level factory registry, distinct from the `provides` structural key on aspects which is removed):

```nix
den.schema.host.includes = [ (den.provides.import-tree.provides.host ./hosts) ];
den.schema.user.includes = [ (den.provides.import-tree.provides.user ./users) ];
```

### Flake Output Batteries (policy-driven, unchanged shape)

`flakeSystemOutputs`, `hmConfigurations`, `osConfigurations` are internal batteries driven by core policies. They already use the policy pattern. In the unified model, they become aspect-included policies on the flake-level entity. Context keys like `flake-system` are not in `den.schema` — they create context scopes without entity schema association, which is valid:

```nix
den.aspects.flake-outputs = {
  policies.system-to-os = { flake-system }:
    map (host: policy.resolve { flake-os = host; })
      (getHosts flake-system);

  policies.system-to-hm = { flake-system }:
    map (home: policy.resolve { flake-hm = home; })
      (getHomes flake-system);
};
```

### Migration Summary

| Category | Batteries | Migration |
|----------|-----------|-----------|
| Simple (hostname, user-shell, etc.) | 7 | None — already fits |
| Predicate (unfree, insecure) | 2 (4 files) | Optional trait migration |
| Forward router (os-class, wsl) | 3 | None — orthogonal to model |
| Home environment (HM, maid) | 2 | Aspect-included policies (major simplification) |
| Cross-entity (mutual, host-aspects) | 2 | Eliminated / replaced by policies |
| Import tree | 1 | None |
| Flake outputs | 3 | Aspect-included policies (shape change only) |

## Resolved Decisions

- **Targeted vs broadcast provides**: expressed as policy guards — `lib.optional (user.name == "alice")` for targeted, unconditional for broadcast
- **Trait filtering**: `policy.resolve` shadows trait context for a subtree; policy signature guarantees correct timing (defers until collection complete)
- **Provides vs traits**: `provides` (directed class config injection) becomes `policy.include`; traits (collected data) remain separate — different concerns, same pipeline
- **Aspect-included policies**: aspects can include policies via `policies` structural key; discovery during resolution, removal with aspect exclusion
- **`policy.exclude` scoping**: context-matched; the exclude applies to all contexts matching the policy's function signature and propagates into sub-contexts; excludes don't leak into unrelated contexts
- **No `policy.substitute`**: substitution is expressed as `policy.exclude` + `policy.include` — composing existing effects
- **`policy.resolve` creates parallel branches**: each resolve effect creates a sibling context branch, not a mutation of the current context; this preserves commutativity when multiple policies resolve with conflicting keys
- **`policy.resolve` merges into parent**: within a single resolve effect, new bindings merge into the parent context; colliding keys are shadowed by the new value
- **`policy.include` accepts inline attrsets**: coerced to anonymous aspects, same as how `includes = [ { nixos.foo = true; } ]` works today
- **Cross-entity traits are Tier 3 only**: phase 2 data arrives at module evaluation time, consistent with current `provide-to` timing
