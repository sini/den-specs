# Eliminate `den.stages` — Three-Layer Aspect Model

**Date:** 2026-04-25
**Branch:** feat/traits
**Status:** Draft

## Problem

`den.stages` is a redundant abstraction. Stages are structurally identical to aspects (`stageSubmodule` imports `aspectType.getSubModules`) with two fields removed (`into`, `__functor`). The parametric context gating system already controls when aspects evaluate — an aspect taking `{ host, user }:` cannot resolve until both values are provided by handlers. Policies already provide that context. Stages are a middleman that manually groups what the parametric system knows automatically.

The 13 current stages contain almost no behavior — they are forwarders (`{ host }: host.aspect`, `SystemOutputFwd` wrappers) and include lists. The stage namespace encourages misuse (e.g., `den.stages.host.networking.firewall` for what should be an aspect) and adds conceptual weight without architectural value.

## Design

### Three-Layer Model

Replace stages with three cooperating mechanisms:

| Layer | Purpose | Example |
|-------|---------|---------|
| **`den.aspects`** (registry) | Named aspect definitions exist here. Parametric signatures declare context requirements. No dispatch semantics. | `den.aspects.hm-host-module = { host }: { ... };` |
| **Policy `aspects`** (inclusion) | Policies gain an `aspects` field listing registered aspect names to include when the policy fires. | `policies.host-to-hm-users.aspects = [ "hm-host-module" ];` |
| **Entity `includes`** (static) | `den.schema.<kind>` retains `includes` for unconditional aspect inclusion during entity resolution. | `den.schema.host.includes = [ ... ];` |

### Deletions

- `nix/nixModule/stages.nix` — the `den.stages` option
- `nix/lib/stage-types.nix` — `stageTreeType`, `stageSubmodule`
- `nix/lib/resolve-stage.nix` — `resolveStage`
- `resolveTargetHandler` in `nix/lib/aspects/fx/pipeline.nix` (lines 91–103)

### New: `resolveEntity`

Replaces `resolveStage`. Thinner — no stage namespace lookup, no structural key stripping. Must produce the full aspect shape the pipeline expects (name, meta, provides, includes, scope handlers):

```nix
resolveEntity = kind: ctx:
  let
    scopeHandlers = constantHandler ctx;
  in
  {
    name = kind;
    meta = {
      handleWith = null;
      excludes = [];
      provider = [];
      into = null;
    };
    provides = {};
    includes = entityIncludesFor kind;  # from den.schema.<kind>.includes
    __scopeHandlers = scopeHandlers;
    __ctxStage = kind;  # retained for diagnostics/tracing
  };
```

The entity's own aspect (e.g., `host.aspect`) is no longer inlined here — it arrives via the entity's `provides.host = { host }: host.aspect` pattern, which is now a registered aspect or entity include. `resolveEntity` produces the *root aspect* for a kind: the structural frame that the pipeline recurses into. Actual behavior comes from includes and policy-attached aspects.

Called from `modules/options.nix` (line 105) and `nix/lib/home-env.nix`.

### Policy Dispatch Changes

`from`/`to` reference entity schema kinds directly, not stage names.

**Current:**
```nix
# policy-types.nix
from = "host";         # stage name
to = "hm-host";        # stage name

# synthesize-policies.nix infers entity kind via suffix matching
entityKeyFor = stage: /* exact match, then hasSuffix "-${k}" */;
```

**New:**
```nix
# policy-types.nix — new field
aspects = lib.mkOption {
  type = lib.types.listOf lib.types.str;
  default = [];
  description = "Registered aspect names to include for entities resolved by this policy.";
};

# from/to are entity kinds
from = "host";         # den.schema.host
to = "user";           # den.schema.user

# synthesize-policies.nix simplifies
# entityKeyFor removed — from/to ARE entity kinds
# ctxSatisfies = kind: ctx: ctx ? ${kind} && builtins.isAttrs ctx.${kind};
```

### Migration: Current Stage Content

| Current stage | New home |
|---|---|
| `host` provides (`{ host }: host.aspect`) | Deleted — entity *is* the aspect, `resolveEntity` handles directly |
| `host` includes (from `os-class.nix`, `os-user.nix`) | `den.schema.host.includes` or registered aspects via policy |
| `user` provides (`{ host, user }: user.aspect`) | Deleted — same as host |
| `user` includes | `den.schema.user.includes` or registered aspects via policy |
| `default` | `den.default` already exists; `*-to-default` policies resolve it directly |
| `home` | Entity aspect on `den.schema.home` |
| `hm-host`, `hm-user` | Registered aspects + single policy (see makeHomeEnv below) |
| `hjem-host`, `hjem-user` | Same pattern as hm |
| `maid-host`, `maid-user` | Same pattern as hm |
| `flake-system` (7 provides) | Registered aspects on new `flake` entity (see below) |
| `wsl-host` | Registered aspect with `{ host }:` signature + guard |
| `microvm-host` | Registered aspect with `{ host }:` signature + guard |

### Migration: `makeHomeEnv`

**Current:** generates 2 stages + 2 policies per battery.

```nix
makeHomeEnv = { className, ctxName, ... }: {
  stages = {
    "${ctxName}-host".provides."${ctxName}-host" = { host }: { ... };
    "${ctxName}-user".provides."${ctxName}-user" = forwardToHost { ... };
  };
  policies = {
    "host-to-${ctxName}-host" = { from = "host"; to = "${ctxName}-host"; ... };
    "${ctxName}-host-to-${ctxName}-user" = { from = "${ctxName}-host"; to = "${ctxName}-user"; ... };
  };
};
```

**New:** generates 2 registered aspects + 1 policy.

```nix
makeHomeEnv = { className, ctxName, ... }: {
  aspects = {
    "${ctxName}-host-module" = { host }: {
      ${host.class}.imports = [ host.${optionPath}.module ];
    };
    "${ctxName}-user-forward" = { host, user }:
      forwardToHost { inherit className ctxName forwardPathFn; } { inherit host user; };
  };
  policies = {
    "host-to-${ctxName}-users" = {
      from = "host";
      to = "user";
      aspects = [ "${ctxName}-host-module" "${ctxName}-user-forward" ];
      resolve = { host, ... }:
        let enabled = /* mkDetectHost logic */;
        in lib.optionals enabled (mkIntoClassUsers className { inherit host; });
    };
  };
};
```

Two-hop chain (`host → hm-host → hm-user`) collapses to one policy (`host → user` with class filtering). The intermediate stage is gone.

**`userEnvAspect` migration:**

`home-env.nix` lines 59–67 currently define `userEnvAspect` which calls `resolveStage` twice:

```nix
# Current
userEnvAspect = ctxName: { host, user, ... }: {
  includes = [
    (den.lib.resolveStage "${ctxName}-user" { inherit host user; })
    (den.lib.resolveStage "user" { inherit host user; })
  ];
};
```

This is eliminated. The `${ctxName}-user` stage content is now the `${ctxName}-user-forward` registered aspect. The `user` stage content is the entity resolution itself. Both are included via the policy's `aspects` field — the policy `host-to-${ctxName}-users` attaches `[ "${ctxName}-host-module" "${ctxName}-user-forward" ]`, and the core `host-to-users` policy handles the base user entity resolution. `userEnvAspect` as a function is deleted; its role is subsumed by policy aspect inclusion.

### New: `flake` Entity

Currently there are two scoping levels: `flake` (no entity context) and `flake-system` (has `{ system }` context). The policy chain is `flake → flake-system → flake-{packages,os,hm,...}` (`modules/policies/flake.nix`). The `flake → flake-system` hop fans out by `den.systems`, providing `{ system }` to each.

New model uses two entity kinds:

```nix
den.schema.flake = {
  # Entity representing the flake as a whole — no entity context
};
den.schema.flake-system = {
  # Per-system flake scope — carries { system }
};
```

The `flake → flake-system` fan-out becomes a policy between these two entity kinds:

```nix
den.policies.flake-to-flake-system = {
  _core = true;
  from = "flake";
  to = "flake-system";
  resolve = _: map (system: { inherit system; }) den.systems;
};
```

The 7 `SystemOutputFwd` forwarders become registered aspects taking `{ system }:` (not `{ flake }:` — they need the system context from the fan-out):

```nix
den.aspects = {
  flake-packages = { system }: /* SystemOutputFwd for packages */;
  flake-apps = { system }: /* SystemOutputFwd for apps */;
  flake-checks = { system }: /* SystemOutputFwd for checks */;
  flake-devShells = { system }: /* SystemOutputFwd for devShells */;
  flake-legacyPackages = { system }: /* SystemOutputFwd for legacyPackages */;
  flake-os = { system }: /* forwards host configurations */;
  flake-hm = { system }: /* forwards home configurations */;
};
```

Per-output policies attach the relevant aspect:

```nix
den.policies.flake-system-to-packages = {
  _core = true;
  from = "flake-system";
  to = "flake-system";  # sibling route — same scope
  aspects = [ "flake-packages" ];
  resolve = { system, ... }: lib.singleton { inherit system; output = "packages"; };
};
# ... etc for each output type
```

The flake entity can carry export metadata for multi-flake and template use cases.

### Compat Shim Update

`modules/compat/ctx-shim.nix` currently forwards `den.ctx.*` → `den.stages.*`. Updated to forward to the new target:

- `den.ctx.*.provides.*` → `den.aspects.*` with deprecation warning
- `den.ctx.*.includes` → entity schema includes with deprecation warning
- `den.ctx.*.into` → `den.policies` with deprecation warning (already present)

### Pipeline Changes

**`resolve-target` effect replacement:**

`transition.nix` line 242 emits `resolve-target` to resolve child transitions during cross-context fan-out. This effect currently calls `resolveStage` to look up a stage by path. With stages gone, `resolve-target` is replaced by `resolve-entity`:

- The handler looks up the entity kind from the transition path
- Calls `den.lib.resolveEntity kind ctx` instead of `den.lib.resolveStage`

The effect name changes from `resolve-target` to `resolve-entity` for clarity. `transition.nix` line 242 updates accordingly.

**Policy aspect scoping — per-context, not per-kind:**

Policy aspects apply **only to the specific entities that `resolve` returns**, not to all entities of the target kind. This is critical: when `host-to-users` (no aspects) and `host-to-hm-users` (aspects: `["hm-host-module", "hm-user-forward"]`) both target `to = "user"`, only the users resolved by `host-to-hm-users` receive those aspects.

This requires changes to how the transition handler merges and resolves policy results:

**1. Policy dispatch carries aspects in routing metadata (~3 lines in `policy-dispatch.nix`):**

`compilePolicyHandlers` already returns `{ targets, routing }` per policy. The routing metadata gains an `aspects` field:

```nix
routing = {
  inherit (policy) from to;
  inherit targetKey;
  policyName = name;
  aspects = policy.aspects or [];  # NEW
};
```

**2. Transition merging respects aspect sets (~10 lines in `transition.nix`):**

Currently `mergeByPath` (line 390) merges all transitions targeting the same path by concatenating contexts. With policy aspects, transitions with **different aspect sets must not merge** — otherwise contexts from `host-to-users` (no aspects) and `host-to-hm-users` (has aspects) would be combined, losing the per-policy scoping.

The merge key changes from `pathKey` to `pathKey + sorted aspects hash`:

```nix
mergeByPath = builtins.foldl' (acc: t:
  let
    # Sort aspect names for canonical merge key — list order in policy
    # definitions must not affect merge identity.
    sortedAspects = lib.sort (a: b: a < b) (t.routing.aspects or []);
    aspectsKey = builtins.concatStringsSep "," sortedAspects;
    mergeKey = "${lib.concatStringsSep "." t.path}|${aspectsKey}";
  in
  acc // {
    ${mergeKey} =
      if acc ? ${mergeKey} then
        acc.${mergeKey} // { contexts = acc.${mergeKey}.contexts ++ t.contexts; }
      else t;
  }
) {} rawTransitions;
```

Transitions from policies with identical aspect sets still merge (preserving fan-out dedup). Transitions with different aspect sets stay separate and resolve independently.

**3. Aspect injection during target resolution (~15-20 lines in `transition.nix`):**

`resolveTransition` (line 228) receives the transition which now carries `routing.aspects`. Before calling `resolveContextValue` (line 296), the policy aspects are looked up from `den.aspects` by name and merged into the target's `includes`:

```nix
# In resolveTransition, after resolve-entity returns effectiveTarget:
policyAspectNames = transition.routing.aspects or [];
policyAspects = map (name: den.aspects.${name}) policyAspectNames;
targetWithPolicyAspects = effectiveTarget // {
  includes = (effectiveTarget.includes or []) ++ policyAspects;
};
# Pass targetWithPolicyAspects to resolveContextValue instead of effectiveTarget
```

The injected values are **raw parametric wrappers** (e.g., `{ __fn, __args, name, ... }`) — not pre-resolved values. They carry signatures like `{ host, user }:` and resolve via `bind.fn` + `constantHandler` when `aspectToEffect` processes the target's `includes` during child resolution. This is the same path all parametric includes already take — no special handling needed.

**4. `resolveEntityHandler` needs `den.aspects` in scope (~3 lines in `pipeline.nix`):**

The handler that replaces `resolveTargetHandler` needs access to the aspect registry for the injection step. This is passed as a closure variable, same as `den.stages` was before.

**Summary of per-context scoping impact:** ~55-65 lines of pipeline changes, concentrated in `transition.nix` (merge keys, ctxKey fix, aspect injection, ctx-seen accumulation) and `policy-dispatch.nix` (routing metadata). The rest of the spec (deletions, migrations, `makeHomeEnv`, flake entity) is unaffected.

**Cross-transition `ctx-seen` dedup:**

Currently `ctxKey` (line 280) includes `#${toString indexed.i}` — an index within the transition's context list. This was designed for disambiguating structurally identical contexts *within* a single merged transition. With split transitions, the same entity (alice) may appear at different indices in different transitions, producing non-colliding keys and resolving twice.

Fix: the dedup key must be index-independent for cross-transition dedup. Change `ctxKey` to use only the context identity, not the per-transition index:

```nix
ctxKey = if isFanOut then "${key}/{${ctxNames}}" else key;
```

The original `#${toString indexed.i}` suffix was a workaround for contexts with identical attr names but different values (e.g., `{fromClass=_:"packages"}` vs `{fromClass=_:"files"}`). These are structurally different and produce different `ctxNames` via `mkCtxId`. The only case where `ctxNames` collide for genuinely different contexts is when all context attrs hash identically — this is the same entity and *should* dedup.

**Aspect accumulation, not racing:**

When the same entity appears in multiple transitions with different aspect sets, aspects must **accumulate** rather than race. The `ctx-seen` handler changes from a boolean gate to an aspect collector:

```nix
# ctx-seen handler returns { isFirst, previousAspects }
# First visit: resolve with this transition's aspects, record them
# Subsequent visit: skip full resolution, but emit additional aspects
#   as supplemental includes into the already-resolved target
```

Implementation: `ctx-seen` state tracks `{ resolved = true; aspects = [...]; }` per key. On first visit, the entity resolves with its transition's aspects. On subsequent visits from transitions with different aspects, only the *new* aspects (not already in the accumulated set) are emitted as additional includes into the existing resolution. This avoids full re-resolution while ensuring all policy-scoped aspects reach their target entities.

This handles the multi-policy case cleanly:
- Policy A: `aspects = ["foo"]`, resolves `[alice]`
- Policy B: `aspects = ["bar"]`, resolves `[alice]`
- Policy C: `aspects = []`, resolves `[alice]`

Alice resolves once (first visit), then gets `["foo"]` and `["bar"]` accumulated. Order doesn't matter — all aspects arrive. ~15 lines in the `ctx-seen` handler.

**Concrete example — home-manager battery:**

```
host-to-users:     aspects = []
                   resolve → [alice, bob, carol]

host-to-hm-users:  aspects = ["hm-host-module", "hm-user-forward"]
                   resolve → [alice, bob]  (only HM class users)

Pipeline sees two separate transitions (different aspect sets, won't merge):
  Transition 1: path=user, aspects=[], contexts=[alice, bob, carol]
  Transition 2: path=user, aspects=["hm-host-module","hm-user-forward"], contexts=[alice, bob]

Resolution (order-independent via accumulation):
  alice:  entity resolves on first visit; hm aspects accumulated from Transition 2
  bob:    same as alice
  carol:  entity resolves; no additional aspects (only appears in Transition 1)
```

**Other pipeline changes:**

- `pipeline.nix`: remove `resolveTargetHandler`, add `resolveEntityHandler` (same location, calls `resolveEntity`)
- `options.nix`: replace `den.lib.resolveStage kind ctx` with `den.lib.resolveEntity kind ctx`
- `options.nix`: remove `knownKinds` derived from `den.stages` — derive from `den.schema` only
- `home-env.nix`: replace `resolveStage` calls with direct aspect references (see `userEnvAspect` above)
- `synthesize-policies.nix`: remove `entityKeyFor` suffix matching, simplify `ctxSatisfies`
- Diagnostics (`diag/context.nix`, `diag/fleet.nix`): update `resolveStage` calls

### Files Affected

**Delete:**
- `nix/nixModule/stages.nix`
- `nix/lib/stage-types.nix`
- `nix/lib/resolve-stage.nix`

**New:**
- `nix/lib/resolve-entity.nix`

**Modify:**
- `modules/options.nix` — `resolveEntity`, remove stage-derived `knownKinds`
- `nix/lib/aspects/fx/pipeline.nix` — remove `resolveTargetHandler`, add `resolveEntityHandler`
- `nix/lib/aspects/fx/handlers/policy-dispatch.nix` — add `aspects` to routing metadata
- `nix/lib/aspects/fx/handlers/transition.nix` — per-aspect-set merge keys, aspect injection in `resolveTransition`, aspect-bearing transition ordering
- `nix/lib/policy-types.nix` — add `aspects` field
- `nix/lib/synthesize-policies.nix` — simplify, remove `entityKeyFor`
- `nix/lib/home-env.nix` — rewrite `makeHomeEnv` (aspects + single policy)
- `modules/compat/ctx-shim.nix` — update forward targets
- `modules/context/host.nix` — remove stage provides
- `modules/context/user.nix` — remove stage provides
- `modules/aspects/defaults.nix` — remove stage assignment
- `modules/outputs/flakeSystemOutputs.nix` — register aspects, add flake entity
- `modules/outputs/osConfigurations.nix` — register aspect
- `modules/outputs/hmConfigurations.nix` — register aspect
- `modules/aspects/provides/home-manager.nix` — use new makeHomeEnv shape
- `modules/aspects/provides/hjem.nix` — same
- `modules/aspects/provides/maid.nix` — same
- `modules/aspects/provides/wsl.nix` — register aspect
- `modules/aspects/provides/os-class.nix` — move includes to entity schema
- `modules/aspects/provides/os-user.nix` — move includes to entity schema
- `nix/lib/default.nix` — export `resolveEntity`, remove `resolveStage`
- `nix/lib/diag/context.nix` — update resolution calls
- `nix/lib/diag/fleet.nix` — update resolution calls
- `nix/nixModule/policies.nix` — updated type
- `modules/policies/flake.nix` — rewrite: `from`/`to` reference entity kinds, add `aspects` fields
- `templates/microvm/modules/microvm-integration.nix` — register aspect

**Test impact:** 90+ template test files reference stages — all need updating.

## Prerequisites

This spec executes **after** the traits and structural nesting spec (`2026-04-25-traits-and-structural-nesting-design.md`) is complete. The traits spec lands:

- `den.classes` / `den.traits` top-level registries
- `aspectContentType` replacing `deferredModule` as the aspect freeform type
- `compileStatic` schema-aware key classification
- `traitArgHandler` / `traitCollectorHandler` in the pipeline
- `provides` deprecation via direct nesting
- `provide-to` unification with trait-based cross-entity routing

This spec builds on that foundation. Key implications:

- **`resolveEntity`** produces a root aspect whose `includes` contain raw aspect definitions (possibly parametric wrappers with `__fn`/`__args`). These go through the traits-aware `compileStatic` + `aspectToEffect` pipeline — no special handling needed.
- **Policy aspect injection** emits raw aspect wrappers into `includes`. In the post-traits pipeline, these are classified by `compileStatic` against `den.classes`/`den.traits` registries. Compatible — the injected aspects are full definitions, same as any other include.
- **`stageSubmodule` deletion** is safe because `aspectType` has already evolved to use `aspectContentType`. No stage type depends on the old `aspectType.getSubModules` shape.
- **`ctx-seen` accumulation** emits supplemental aspects that flow through the traits-aware pipeline. The accumulation emits raw wrappers, not pre-classified content, so schema-aware classification applies normally.
- **Cross-entity trait routing** (provide-to unification from traits spec) uses policy sibling routes. The flake per-output sibling route pattern in this spec must be verified against the updated provide-to mechanism.

### Trait System Extension: First-Class Policy Dispatch

This spec extends the traits system in one targeted way: **accumulated Tier 1/2 trait data becomes available to policy `resolve` functions**, making traits first-class dispatch targets alongside entity context.

Currently `policy.resolve` receives only entity context (`{ host, ... }:`). With traits in the pipeline, the policy dispatch handler (`policy-dispatch.nix`) merges accumulated `state.traits` into the context passed to `resolve`:

```nix
# In compilePolicyHandlers, before calling policy.resolve:
traits = ((state.traits or (_: {})) null);
# traits // ctx — entity context wins on name collision, matching
# the pipeline's constantHandler > traitArgHandler priority.
rawResult = if argsOk then policy.resolve (traits // ctx) else [];
```

The unwrapped `state.traits` contains the collected data directly — for `"list"` traits it's already a list of values, for `"map"` traits it's a merged attrset. No additional unwrapping needed.

This enables policies that condition on trait data:

```nix
den.policies.host-to-tls = {
  from = "host";
  to = "host";
  aspects = [ "tls-hardening" ];
  resolve = { host, firewall ? [], ... }:
    let hasHttps = builtins.any (f: builtins.elem 443 (f.ports or [])) firewall;
    in lib.optional hasHttps { inherit host; };
};
```

**Trait arg satisfaction in pre-filtering.** `resolveArgsSatisfied` runs in two places: `compilePolicyHandlers` (pipeline time, has trait state) and `activePoliciesFor` (compile time, entity context only). A policy whose `resolve` requires a trait arg (`{ host, firewall, ... }:`) would normally fail the `activePoliciesFor` pre-filter — it checks `ctx ? ${k}` before trait state exists.

Fix: `resolveArgsSatisfied` checks `den.traits` for any arg not found in `ctx`. A registered trait name is always considered satisfiable — the schema guarantees it will be available at dispatch time:

```nix
# In resolveArgsSatisfied:
requiredArgSatisfied = k: ctx ? ${k} || den.traits ? ${k};
```

This means policy `resolve` functions use bare trait args naturally — `{ host, firewall, ... }:` just works, no defaults required. At compile-time pre-filtering, the trait arg passes because the schema declares it. At pipeline-time dispatch, the actual accumulated data is provided.

**Impact:** ~10 lines across `policy-dispatch.nix` and `transition.nix` (passing the merged context). No changes to `synthesize-policies.nix` logic.

**Files affected (in addition to those listed below):**
- `nix/lib/aspects/fx/handlers/policy-dispatch.nix` — merge traits into policy resolve context
- `nix/lib/aspects/fx/handlers/transition.nix` — pass trait state when sending policy effects

## Non-Goals

- Refactoring the fx pipeline itself — it works, aspects are its native unit
- Removing `den.default` — it serves a different purpose (global defaults)

## Appendix: Reactive Policy Dispatch via Traits

With Tier 1/2 trait data available to policy `resolve` functions, policies shift from **static graph edges** (wired by entity identity) to **reactive graph edges** (wired by semantic content). The entity graph reshapes itself based on what aspects emit.

### Conditional service wiring

Policies fire only when specific capabilities are present — no manual flags needed:

```nix
# Only set up monitoring exporters for hosts that actually expose metrics
den.policies.host-to-monitoring = {
  from = "host";
  to = "host";
  aspects = [ "prometheus-exporters" ];
  resolve = { host, service-endpoints ? {}, ... }:
    let hasMetrics = builtins.any (e: e ? metricsPort) (builtins.attrValues service-endpoints);
    in lib.optional hasMetrics { inherit host; };
};
```

The presence of metrics-bearing service endpoints *is* the signal. The policy reads semantic data, not configuration flags.

### Topology-aware fan-out

Policies shape the entity graph based on what aspects declare:

```nix
# Route database replication only between hosts that declare postgres traits
den.policies.postgres-replication = {
  from = "host";
  to = "host";
  as = "replica";
  aspects = [ "postgres-replica-config" ];
  resolve = { host, service-endpoints ? {}, ... }:
    let
      isPrimary = service-endpoints ? postgres
        && service-endpoints.postgres.role == "primary";
      allHosts = builtins.attrValues den.hosts.${host.system};
      replicas = builtins.filter (h:
        h.name != host.name
        && (h.resolved.traits.service-endpoints.postgres.role or "") == "replica"
      ) allHosts;
    in lib.optionals isPrimary
      (map (r: { inherit host; replica = r; }) replicas);
};
```

Replication topology emerges from trait data — no hardcoded host lists, no manual `replicaOf` options.

### Security policy gating

Policies enforce security constraints based on what services are exposed:

```nix
# Any host exposing low ports gets hardening aspects
den.policies.internet-facing-hardening = {
  from = "host";
  to = "host";
  aspects = [ "fail2ban" "ssh-hardening" "audit-logging" ];
  resolve = { host, firewall ? [], ... }:
    let
      allPorts = lib.concatMap (f: f.ports or []) firewall;
      isInternetFacing = builtins.any (p: p < 1024) allPorts;
    in lib.optional isInternetFacing { inherit host; };
};
```

Add nginx → firewall trait emits port 80 → policy triggers hardening. Remove nginx → hardening goes away. Security follows from declarations, not manual assignment.

### Cross-entity trait-filtered aggregation

Policies select which entities' traits flow where based on trait content:

```nix
# Load balancer only receives backend endpoints for its designated vhosts
den.policies.lb-to-backends = {
  from = "host";
  to = "host";
  as = "backend";
  resolve = { host, http-backends ? [], ... }:
    let
      isLB = host.role == "loadbalancer";
      lbVhosts = host.loadbalancer.vhosts or [];
      backends = builtins.filter (h:
        builtins.any (b: builtins.elem b.vhost lbVhosts)
          (h.resolved.traits.http-backends or [])
      ) (builtins.attrValues den.hosts.${host.system});
    in lib.optionals isLB
      (map (b: { inherit host; backend = b; }) backends);
};
```

The graph edge is data-driven — only hosts whose `http-backends` trait matches the load balancer's vhost list get connected.

### What reactive dispatch enables

| Without trait dispatch | With trait dispatch |
|---|---|
| Policies wire entities by identity | Policies wire entities by capability |
| Graph topology is manually specified | Graph topology emerges from declarations |
| Adding a service requires updating policies | Adding a service's aspect is sufficient |
| Security/monitoring is opt-in per host | Security/monitoring follows from exposure |

### Constraints

- **Tier 1/2 only.** Policy dispatch is pipeline-time — only static and pipeline-parametric trait data is available. Tier 3 (module-evaluated) traits depend on `config` which doesn't exist until `evalModules` runs.
- **No circular trait dependencies.** Traits accumulate during the depth-first walk; policy dispatch reads accumulated state at transition time. A policy cannot cause traits to be emitted that would change its own dispatch decision — the walk is past that point.
- **Trait args are schema-validated.** `resolveArgsSatisfied` recognizes registered trait names via `den.traits` and considers them satisfiable even when trait state hasn't accumulated yet. Policy `resolve` functions use bare trait args naturally: `{ host, firewall, ... }:`.
