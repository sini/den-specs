# provide-to: Cross-boundary context forwarding via effects

## Problem

The fx pipeline resolves aspects in isolated scopes. Context flows down (parent to child via `__ctx`) but not across resolution boundaries. This blocks cross-host forwarding and is likely related to several existing test failures.

Three boundary types are affected:

1. **Fresh pipeline runs** (forward mechanism): `forward.nix` calls `den.lib.aspects.resolve` starting a new pipeline that loses parent context.
2. **Transition propagation**: Parametric includes in user aspects can't see `host` from the transition that resolved their parent.
3. **Cross-entity** (new, untested): Host A can't contribute modules to host B's config (e.g., `/etc/hosts` with fleet IPs, SSH `known_hosts`, haproxy backends).

## Pre-work: Restore scope.stateful for transitions

Problem 2 (transition propagation) is fixable without provide-to. The ctx-as-data approach (commit `98f2f78`) removed `scope.run`/`scope.stateful` from transitions as a workaround for broken effect rotation. Now that nix-effects has deep handler semantics (`23965f1`), `scope.stateful` should work correctly — handlers propagate through the full subtree including nested includes.

**Action:** Replace ctx-as-data `__ctx` tagging with `scope.stateful { constantHandler { host, user } }` in the transition handler. This makes host/user available as handled effects for the ENTIRE subtree — no `__parentCtx` plumbing needed.

**Verification:** If `scope.stateful` still fails to propagate context, file a nix-effects issue with a reproducing test. The handler rotation semantics SHOULD preserve parent handlers — handlers are part of rotation, not state.

**Note:** Use `scope.stateful` (preserves parent state) not `scope.run` (replaces state).

**What goes away:**
- `__ctx` — context values tagged on aspect attrsets. Replaced by scoped handlers that provide context to the entire subtree naturally.
- `__parentCtx` / `__parentCtxId` — manual plumbing to propagate context from parent to children's includes. Scoped handlers propagate automatically — children inherit parent handlers.

**What stays (identity, not context):**
- `__ctxId` — distinguishes fan-out instances (`tux/{igloo}` vs `tux/{iceberg}`) for module dedup. Still needed, but computed by the transition handler from the fan-out values, no longer derived from `__ctx`.
- `__parametricResolved` — tells `classCollectorHandler` whether to preserve `__ctxId` in module keys. Parametric resolutions produce different content per context and must not dedup.

**Deferred includes** (`defer-include`/`drain-deferred`): with scope.stateful, the common case (`{ host }` in user scope) resolves immediately because `host` is in the handler scope. Multi-hop cases (include needs an arg from a deeper transition) still need deferral. Fewer triggers, but the mechanism stays.

This pre-work fixes problem 2 but NOT problems 1 (forward) or 3 (cross-entity). Those require provide-to.

## Rejected alternatives

### A. Inherited context chain

Thread parent context into child pipeline runs via an `inheritCtx` parameter. Simpler, but cycles are prevented by API discipline (only pass entity metadata, not resolution results) rather than by construction. Also doesn't generalize to sibling-to-sibling (cross-host) forwarding without additional mechanisms.

### B. Simpler ctx threading for existing failures

Some existing failures might be fixable by threading `currentCtx` through `extraState` or fixing `__parentCtx` propagation without new effects. This was considered but doesn't address the cross-entity use case. The pre-work (scope.stateful) subsumes this for problem 2. Problems 1 and 3 need provide-to.

## Solution: Two-phase resolution with `provide-to` effects

### Design principle

Cross-entity contributions look like regular parametric aspects with dendritic class keys. No special `guard`, `module`, or `target` wrappers. The parametric arg name identifies the target context type. The pipeline routes cross-entity emissions internally.

### How it works: ctx transitions for cross-entity

Cross-entity forwarding uses the same mechanism as existing ctx transitions — `into` and `provides`. A new context type (e.g., `peer`) defines the policy:

```nix
# Define the peer policy: each host fans out to sibling hosts
den.ctx.host.into.peer = { host }:
  map (h: { peer = h; })
    (filter (h: h.name != host.name) (attrValues den.hosts.${host.system}));
```

Aspects then use `{ peer }` as a regular parametric arg. The pipeline resolves it via the transition, and the provide-to mechanism routes the result to the peer's config:

```nix
# Cross-provider: host contributes to each peer's config
den.ctx.host.provides.peer = { host }: { peer }:
  lib.optionalAttrs (peer.hasAspect den.aspects.loadbalancer) {
    nixos.services.haproxy.frontends.app.backends = [
      { address = host.meta.ip; port = 8080; }
    ];
  };
```

Or equivalently, as an include on the aspect:

```nix
den.aspects.webserver = { host, ... }: {
  nixos.services.nginx = {
    enable = true;
    virtualHosts."app".locations."/".proxyPass = "http://localhost:8080";
  };

  # Regular parametric include — { peer } resolved via host.into.peer
  includes = [
    ({ peer, ... }:
      lib.optionalAttrs (peer.hasAspect den.aspects.loadbalancer) {
        nixos.services.haproxy.frontends.app.backends = [
          { address = host.meta.ip; port = 8080; }
        ];
      }
    )
  ];
};
```

### The routing problem

The examples above look like regular parametric aspects. The pipeline resolves `{ peer }` via `host.into.peer` fan-out. But the resolved `nixos` modules must land in the **peer's** NixOS config, not the current host's.

In the current pipeline, `emit-class "nixos"` always goes to the root classCollector, which collects for the current pipeline's target class. For sub-entity transitions (host→user), this works because the user's modules are segregated by the user-specific class handling. For sibling transitions (host→peer), the peer IS another host — its modules need to go to a different pipeline run.

This is where the two-phase mechanism comes in. It's an **internal pipeline concern**, not a user-facing API. The user writes regular parametric aspects. The pipeline detects cross-entity transitions and routes accordingly.

### Phase 1: Normal resolution with cross-entity collection

The pipeline runs normally. When a transition targets a sibling entity (same ctx type as emitter — e.g., host→peer where peer is also a host), the transition handler collects the resolved modules in `state.provideTo` instead of emitting them to the current classCollector.

The handler determines routing by comparing the transition's target entity type against the current resolution scope. If the target IS a sibling (would need its own pipeline run), emissions go to `provideTo`. If the target is a child (resolved within the current pipeline), emissions go to the normal classCollector.

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

Note on thunk chain: `param` here is an aspect/attrset, not a NixOS module. The thunk prevents `deepSeq` from forcing aspect content that may reference lazy option defaults. If aspect values are known to be deepSeq-safe, the thunk can be removed.

### Phase 2: Distribution at orchestration level

After phase 1 completes, the orchestration layer (e.g., `osConfigurations.nix`) reads `state.provideTo null`, groups emissions by target entity, and injects them as additional modules when building target configs.

```
flake
  → flake-system (per system)
    → flake-os (per host)         # Phase 1: resolve each host, collect provide-to
    → distribute provide-to       # Phase 2: group by target, inject into targets
    → build target configs        # Targets receive injected modules
```

Phase 2 is one-directional: the `provide-to` handler is NOT installed in phase 2 pipeline runs. Injected modules cannot emit further `provide-to` effects. This prevents cycles structurally.

### Phase 2 semantics: module injection, not re-resolution

Phase 2 does NOT re-run the aspect pipeline. It injects `provide-to` modules directly into the target's NixOS/homeManager module list — alongside the modules already collected in phase 1. This means:

- Injected modules are dendritic (have class keys like `nixos`, `homeManager`). The target's pipeline processes class keys normally via `compileStatic` → `emit-class`.
- Phase 1 pipeline state (constraints, pathSet, chain) is final. Phase 2 doesn't touch it.
- Cost: zero pipeline runs in phase 2. Only module list concatenation.

For the common case (no cross-entity contributions), phase 2 is a no-op. The orchestration layer checks `state.provideTo null == []` and skips distribution entirely.

**Limitation:** Modules injected in phase 2 cannot themselves use `provide-to`. If a cross-host module also needs cross-class forwarding (e.g., fleet SSH config that also needs home-manager integration), that must be expressed as class keys on the injected module directly. This is acceptable — cross-class forwarding within a single entity uses class keys, which already work.

### Forward mechanism migration

`forward.nix` currently starts a fresh `den.lib.aspects.resolve` call, losing all parent context. The migration:

- Step 1 (resolve source class): stays in phase 1 — the source aspect's pipeline still runs normally.
- Steps 2-4 (mapModule, guards, adapters): stay in phase 1 — these transform the resolved module.
- The **result** of steps 1-4 is emitted as a `provide-to` instead of being returned directly as a class key. The orchestration layer injects it into the target.

The adapter/guard/mapModule machinery in `forward.nix` stays intact. Only the routing of the final result changes — from a direct class key return to a `provide-to` emission.

### Target routing: graph-based, context-type agnostic

The `provide-to` routing follows the `den.ctx` transition graph. The emitter's position in the graph determines where emissions land:

| Emitter scope | Target ctx type | Routing | Result |
|---|---|---|---|
| host aspect | `peer` (sibling host) | follows parent's `into.peer` | sibling hosts |
| host aspect | `user` (child) | follows `host.into.user` | my host's users |
| environment aspect | `host` (child) | follows `environment.into.host` | environment's hosts |
| host aspect | `["peer", "user"]` | multi-hop: peers, then their users | users on other hosts |

Single-hop targets (string): one step through the transition graph.
Multi-hop targets (list): walk the graph, fanning out at each step.

This is context-type agnostic — works for any ctx node including user-defined ones (environments, clusters, regions, fleets).

### Effect shape

```nix
{
  target = "<ctx-name>" | ["<ctx-name>" ...];
  content = <aspect attrset with dendritic class keys>;
  emitterCtx = "<ctx-name>";  # set by handler, not user
}
```

The `emitterCtx` tells the orchestration layer where in the transition graph the emission originated. Set automatically by the handler from the pipeline's current ctx state.

Cross-entity contributions are dendritic — they have class keys (`nixos`, `homeManager`, custom classes) just like any aspect. The target's pipeline processes them normally. No `module.nixos` wrapper needed.

### Cycle prevention

- Phase 1 completes fully before phase 2 begins
- Phase 2 pipeline runs do NOT install `provideToHandler`
- No resolution result from phase 1 is required to START phase 2 — only the collected `provideTo` state
- Guard/filter logic uses `lib.optionalAttrs` in regular Nix (closures over phase 1 data), not a special mechanism

## Architecture

### New components

| Component | Location | Purpose |
|---|---|---|
| `provideToHandler` | `handlers/provide-to.nix` | Collects cross-entity emissions in state |
| `transitionHandler` changes | `handlers/transition.nix` | Routing decision: child transitions emit locally (existing), sibling transitions emit `provide-to` (new). The transition handler decides based on whether the target ctx type matches the emitter's — same type = sibling, different type = child. |
| `distributeProvideTo` | `pipeline.nix` or new `orchestrate.nix` | Walks transition graph, groups emissions by target entity, injects into target configs |
| Orchestration changes | `osConfigurations.nix` | Phase 2 distribution after host resolution |

### Pipeline state additions

```nix
defaultState = {
  # ... existing ...
  provideTo = _: [];  # Thunk chain (same pattern as imports/deferredIncludes)
};
```

### Pipeline API

`fxFullResolve` already returns `{ value, state }`. The `state.provideTo` field is new but follows existing patterns. Callers that only use `fxResolve` (extracts `state.imports null`) are unaffected.

### Migration path

1. **Pre-work:** Restore `scope.stateful` for transitions (fixes problem 2)
2. Add `provideToHandler` to `defaultHandlers` in `pipeline.nix`
3. Add `distributeProvideTo` orchestration function
4. Refactor `osConfigurations.nix` to call `distributeProvideTo` after host resolution
5. Migrate `forward.nix` to emit `provide-to` for the transformed module
6. Add cross-entity test (fleet `/etc/hosts`, haproxy backend registration)

Note: the existing `provides.to-users` / `provides.to-hosts` pattern (`mutual-provider.nix`) is left as-is. It operates within the host-user mutual pipeline and has 30+ usages. `provide-to` is for cross-entity forwarding that mutual-provider cannot handle (sibling hosts, cross-subtree targeting). A future migration could unify them, but it's not in scope here.

### What this does NOT change

- The trampoline (`nix-effects`) — no changes needed
- The core effect handlers (constantHandler, classCollector, chainHandler, etc.)
- The `aspectToEffect` / `compileStatic` compilation
- Single-entity resolution (only multi-entity orchestration changes)
- The `intoCtxType` merge or transition handler internals (beyond scope.stateful pre-work)

## Test: Cross-host `/etc/hosts` generation

```nix
test-fleet-etc-hosts = denTest ({ den, lib, igloo, iceberg, ... }: {
  den.hosts.x86_64-linux.igloo.users.tux = {};
  den.hosts.x86_64-linux.iceberg.users.tux = {};

  den.hosts.x86_64-linux.igloo.meta.ip = "10.0.0.1";
  den.hosts.x86_64-linux.iceberg.meta.ip = "10.0.0.2";

  # Peer transition: each host fans out to sibling hosts
  den.ctx.host.into.peer = { host }:
    map (h: { peer = h; })
      (filter (h: h.name != host.name) (attrValues den.hosts.${host.system}));

  # Cross-provider: each host publishes its IP to peers
  den.ctx.host.provides.peer = { host }: { peer }: {
    nixos.networking.extraHosts = "${host.meta.ip} ${host.name}";
  };

  den.aspects.igloo = {};
  den.aspects.iceberg = {};

  expr = {
    igloo-hosts = igloo.networking.extraHosts;
    iceberg-hosts = iceberg.networking.extraHosts;
  };
  expected = {
    igloo-hosts = "10.0.0.2 iceberg";
    iceberg-hosts = "10.0.0.1 igloo";
  };
});
```

## Test: Cross-host haproxy backend registration

```nix
test-haproxy-backend-registration = denTest ({ den, lib, ... }: {
  den.hosts.x86_64-linux.lb = {};
  den.hosts.x86_64-linux.web1 = {};
  den.hosts.x86_64-linux.web2 = {};

  den.hosts.x86_64-linux.lb.meta.ip = "10.0.0.1";
  den.hosts.x86_64-linux.web1.meta.ip = "10.0.0.2";
  den.hosts.x86_64-linux.web2.meta.ip = "10.0.0.3";

  den.aspects.lb.includes = [ den.aspects.loadbalancer ];
  den.aspects.loadbalancer.nixos.services.haproxy.enable = true;

  den.aspects.web1.includes = [ den.aspects.webserver ];
  den.aspects.web2.includes = [ den.aspects.webserver ];

  # Peer transition
  den.ctx.host.into.peer = { host }:
    map (h: { peer = h; })
      (filter (h: h.name != host.name) (attrValues den.hosts.${host.system}));

  # Webserver aspect: serves nginx locally, registers backend with LB peers
  den.aspects.webserver = { host, ... }: {
    nixos.services.nginx = {
      enable = true;
      virtualHosts."app".locations."/".proxyPass = "http://localhost:8080";
    };

    # Regular parametric include — { peer } resolved via host.into.peer
    includes = [
      ({ peer, ... }:
        lib.optionalAttrs (peer.hasAspect den.aspects.loadbalancer) {
          nixos.services.haproxy.frontends.app.backends = [
            { address = host.meta.ip; port = 8080; }
          ];
        }
      )
    ];
  };

  expr = {
    lb-has-haproxy = lb.services.haproxy.enable;
    lb-backends = lb.services.haproxy.frontends.app.backends;
    web1-has-nginx = web1.services.nginx.enable;
  };
  expected = {
    lb-has-haproxy = true;
    lb-backends = [
      { address = "10.0.0.2"; port = 8080; }
      { address = "10.0.0.3"; port = 8080; }
    ];
    web1-has-nginx = true;
  };
});
```

Both tests use den-level entity metadata (`host.meta.ip`), not resolved NixOS config. No fixpoint risk — `provide-to` closures capture source entity data; the target's pipeline doesn't depend on the source's resolution.

## Success criteria

- Pre-work: `scope.stateful` restores transition context propagation
- Cross-entity forwarding works (fleet and haproxy tests pass)
- `forward.nix` migration: forward results distributed via `provide-to` instead of fresh pipeline runs
- No performance regression for configs without cross-entity contributions
