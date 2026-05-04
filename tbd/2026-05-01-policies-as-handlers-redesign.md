# Policies as Handlers — Pipeline Redesign Spike

**Date**: 2026-05-01
**Status**: Brainstorm
**Scope**: Decompose monolithic dispatch-policies into composable effect handlers

## The Problem We Hit

The transition elimination effort (this session) deleted transition.nix (-1011 lines), the DLQ, and forward sub-pipelines. But the replacement — a 524-line `dispatch-policies` handler — recreated the same complexity in a new file. Fixing 20 remaining test failures required 5 dedup layers, parent-scope chain walks, forward output accumulators, and scope-parent self-reference guards. Each fix revealed new implicit contracts.

The root cause: **we translated the old architecture into effect syntax instead of designing for effects.** The monolithic handler batch-dispatches policies, mutates scope state, manages dedup sets, and iterates to fixpoint — exactly what `into-transition` did, just with `fx.bind` instead of explicit state threading.

## The Insight

The effect system (nix-effects) already provides the primitives that the monolithic handler reimplements badly:

| What dispatch-policies does manually | What the effect system provides |
|---|---|
| Scope push/pop via state mutation | `scope.provide` — lexical scoping, automatic restore |
| Include dedup via `includeSeen` set | Handler shadowing — same handler name = override |
| Enrichment iteration loop | Recursive handler composition via `bind` |
| Policy arg matching | Handler installation — only install when args satisfied |
| Forward source cross-scope lookup | Handler reads from own scope — no cross-scope needed |
| 5 dedup mechanisms | Scope-natural isolation — duplicates can't occur |

The algebraic effects ethos: **small handlers that do one thing, composed via `scope.provide` and `bind`**. Not a monolithic handler that reimplements scope management.

## Design Principles

1. **Each policy is a handler, not a classified effect.** When a policy's args are satisfied, it's installed via `scope.provide`. Its effects fire naturally during the tree walk.

2. **Context expansion is `scope.provide`.** `policy.resolve { user = tux; }` installs `user = tux` as a handler. Child tree walks see it. Parent walks don't. No `currentScope` state field, no set/restore, no `scopeParent` tracking.

3. **Include resolution is handler composition.** When an aspect is included, its handler is composed into the current scope. `scope.provide` ensures it's visible to the current entity's tree walk and its descendants, but not to siblings. No `includeSeen` needed — the handler chain IS the dedup.

4. **den.default is just a handler in the chain.** No stripping from child entities. No special treatment. It's provided at the root scope and visible everywhere through normal scope inheritance. Its parametric children resolve when context handlers are available — eagerly, not deferred.

5. **Forward source reads from handler scope.** The source modules are in the current handler chain's class emissions. No `wrappedPerScope` cross-scope lookup. No parent chain walk. The handler chain IS the scope.

6. **No dedup layers.** Scope isolation is structural, not tracked. If handler A is provided at scope X, it's visible at scope X and descendants. It can't leak to siblings. The 5 dedup mechanisms (ctx-seen, dispatchedPolicies, scopedEmittedLocs, adapterKey dedup, routeKey dedup) become unnecessary.

## Concrete Architecture

### Policy Installation

Currently:
```nix
# Monolithic: dispatch-policies reads all policies, matches args, classifies effects
dispatchPoliciesHandler = {
  "dispatch-policies" = { param, state }:
    let
      globalPolicies = den.policies or {};
      schemaPolicies = (den.schema.${entityKind} or {}).policies or {};
      aspectPolicies = flatten (state.scopedAspectPolicies null);
      resolveCtx = traits // currentCtx // { __entityKind = entityKind; };
      dispatched = mkDispatch firedPolicies resolveCtx;  # ~200 lines
      ...iterate enrichment...  # ~100 lines
      ...processSchemaResolves...  # ~100 lines
    in ...;
};
```

Proposed:
```nix
# Each policy installed as a handler when its args are satisfied
installPolicies = policies: ctx:
  let
    matching = filter (p: argsSatisfied p ctx) (attrValues policies);
  in
  foldl' (comp: policy:
    fx.bind comp (_:
      let effects = policy ctx;
      in processEffects effects
    )
  ) (fx.pure null) matching;

# processEffects handles each effect type via existing handlers
processEffects = effects:
  fx.seq (map (eff:
    if eff.__policyEffect == "resolve" then
      expandContext eff.value  # scope.provide + walk entity
    else if eff.__policyEffect == "include" then
      fx.send "emit-include" { child = eff.value; idx = null; }
    else if eff.__policyEffect == "route" then
      fx.send "register-route" eff.value
    else if eff.__policyEffect == "instantiate" then
      fx.send "register-instantiate" eff.value
    else if eff.__policyEffect == "exclude" then
      fx.send "register-constraint" { type = "exclude"; ... }
    else
      fx.pure null
  ) effects);

# Context expansion via scope.provide — the key primitive
expandContext = bindings:
  let
    schemaKeys = filter (k: den.schema ? ${k}) (attrNames bindings);
    enrichKeys = filter (k: !(den.schema ? ${k})) (attrNames bindings);
    contextHandlers = constantHandler bindings;
  in
  fx.effects.scope.provide contextHandlers (
    # Enrichment: just provide the handlers, they're visible to the tree walk
    if schemaKeys == [] then
      fx.pure null  # pure enrichment, no entity to walk
    else
      # Schema resolve: look up entity, walk its tree in this scope
      let kind = head schemaKeys;
      in fx.bind (fx.send "resolve-entity" { inherit kind; }) (entity:
        aspectToEffect entity
      )
  );
```

### What Changes in resolveChildren

Currently (aspect.nix):
```nix
resolveChildren = aspect:
  ...
  fx.bind (emitIncludes ...) (includeResults:
    fx.bind (dispatchPoliciesCall aspect) (  # monolithic dispatch
      policyResults: fx.pure (... ++ policyResults)
    )
  );
```

Proposed:
```nix
resolveChildren = aspect:
  ...
  fx.bind (emitIncludes ...) (includeResults:
    if !(aspect ? __entityKind) then
      fx.pure includeResults
    else
      # Install matching policies as scoped handlers
      fx.bind (installPolicies allPolicies currentCtx) (policyResults:
        fx.pure (includeResults ++ policyResults)
      )
  );
```

The `installPolicies` function is ~50 lines, not 524. It doesn't manage state, doesn't iterate, doesn't dedup. It just installs matching policies and processes their effects. `scope.provide` handles scoping. `bind` handles sequencing. The effect system does the rest.

### What Gets Deleted

- `dispatch-policies.nix` (524 lines) — replaced by ~50-line `installPolicies`
- `dispatchedPolicies` state field — handler shadowing prevents double-fire
- `scopedEmittedLocs` state field — scope isolation prevents duplicate emission
- `registeredRouteKeys` state field — route dedup unnecessary with scope isolation
- `scopeParent` tracking — `scope.provide` handles parent-child relationships
- `scopeContexts` state field — context is in the handler chain
- Forward source parent-chain walk — source is in handler scope
- Deferred include machinery — parametric includes resolve eagerly
- `currentScope` set/restore — `scope.provide` restores automatically
- Forward adapterKey dedup in pipeline.nix — each forward fires once per scope

### What Stays

- `emit-class` handler — collects class modules (simplified: no scopedEmittedLocs)
- `emit-trait` handler — collects trait data
- `emit-include` → include handler → `aspectToEffect` — core include resolution
- `register-route` / `register-instantiate` — route and instantiate registration
- `register-constraint` / `check-constraint` — exclusion constraints
- `register-aspect-policy` — aspect policy registration during tree walk
- `buildForwardAspect` + `applyForwardSpecs` — forward wrapping (post-pipeline)
- `wrapCollectedClasses` — class module wrapping (post-pipeline)
- `applyRoutes` — route application (post-pipeline)

## Traits Simplification (Parallel Effort)

The pipeline redesign enables a parallel simplification of the trait system.

### What traits actually are

Traits are **capability declarations and cross-entity data aggregation** — not just passive data. Real use cases:

| Trait | Purpose | Pipeline-time? | Needs NixOS config? |
|---|---|---|---|
| impermanence | "does this host have impermanence?" | Yes (gates routing) | Yes (persistence rules need `config.services.*.user`) |
| secrets | "agenix or sops-nix?" | Yes (selects backend) | No |
| firewall | "does this host have firewall enabled?" | Yes (gates inclusion) | Yes (rules need `config.services.*.port`) |
| xdg-mime | "does the user have a desktop?" | Yes (gates DE aspects) | No |
| service-discovery | export data for loadbalancers, backup targets, hosts file | No | Yes |

The current three-tier model handles all of these but at significant complexity cost. The key insight: traits span a spectrum from pure pipeline-time flags to NixOS-config-dependent data. **These are not two separate concerns — they're the same data channel at different evaluation stages.**

### The `fleet` pattern

Nix's laziness enables a powerful pattern: expose all evaluated NixOS configs as a self-referential lazy attrset:

```nix
# Built into den's flake policy, one line
_module.args.fleet = lib.mapAttrs (_: sys: sys.config) config.flake.nixosConfigurations;
```

Any host can now read any other host's evaluated config:

```nix
# Firewall: collect ports from all fleet members
{ fleet, host, ... }: {
  networking.firewall.allowedTCPPorts =
    lib.concatMap (peer: peer.den.exports.firewall or [])
      (lib.attrValues fleet);
}

# Hosts file
{ fleet, ... }: {
  networking.hosts = lib.mkMerge (lib.mapAttrsToList
    (name: cfg: cfg.den.exports.hostsEntries or {}) fleet);
}

# Backup targets
{ fleet, ... }: {
  services.borgbackup.repos = lib.concatMap
    (cfg: cfg.den.exports.backupTargets or [])
    (lib.attrValues fleet);
}
```

This works because `fleet.webserver` is a thunk — only evaluated when accessed. No cycles as long as access paths don't loop (host A reads host B's port, host B's port comes from its own service definition).

### Cycle avoidance

The `fleet` pattern has one structural constraint: **no circular evaluation paths.** Host A can read host B's config, but host B's value must not depend on host A's read. In practice this is rarely an issue — service ports, persistence paths, and backup targets are all derived from the host's own service definitions, not from other hosts reading them.

Where it matters: a load balancer reading `fleet.*.den.exports.backendPorts` is fine (backends declare their ports independently). A mutual dependency where host A's firewall depends on host B's firewall would infinite-recurse. This is the same constraint as any lazy self-referential structure in Nix — the `fleet` pattern doesn't create new risks, it just makes cross-host reads explicit where they were previously hidden behind trait inheritance.

### Uniform export interface

For the "option doesn't exist if module isn't imported" problem, add a single freeform option to every host:

```nix
# Injected into every host's mainModule
options.den.exports = lib.mkOption {
  type = lib.types.lazyAttrsOf lib.types.anything;
  default = {};
};
```

Any module can write `den.exports.firewall = [...]`. Any reader gets `{}` if nothing writes to it. No import dependency. No trait schema registration needed.

### Proposed simplification

**Keep pipeline-time traits** as lightweight capability flags (booleans, enums, small lists). These gate policy routing decisions during the pipeline walk — the one thing that genuinely can't wait for NixOS eval. The `classifyKeys` function becomes: `if traitSchema ? k then trait else class`. No DLQ, no structural sniffing, no 4-step classification.

**Replace Tier 3 traits** with `fleet` + `den.exports`. Cross-host service discovery becomes plain NixOS module code. The impermanence case works as: `impermanence = true` is a pipeline-time flag (gates whether impermanence aspects are included), the actual persistence *rules* are `den.exports.impermanence.paths = [...]` read by other modules via `fleet`.

**Delete**: `traitModuleForScope`, `scopedDeferredTraits`, deferred trait evaluation, `inheritTraits` scope walking, three-tier classification in `classifyKeys`

## Migration Strategy

### Phase 1: Policies as handlers (core redesign)
1. Implement `installPolicies` / `expandContext` using `scope.provide`
2. Replace `dispatchPoliciesCall` in aspect.nix
3. Delete `dispatch-policies.nix`
4. Remove den.default stripping (unnecessary with scope.provide)
5. Remove deferred include machinery (unnecessary with eager resolution)
6. Remove dedup state fields
7. Target: all 667 tests pass, net -400 lines

### Phase 2: Forward simplification
1. Forward source reads from handler scope (no cross-scope lookup)
2. `applyForwardSpecs` simplified (no parent chain, no accumulator separation)
3. Evaluate: can Tier 2 forwards become policy.route effects entirely?
4. Target: delete buildForwardAspect if possible

### Phase 3: Traits simplification
1. Add `fleet` lazy attrset to flake module args (one line in flake policy)
2. Add `den.exports` freeform option to every host's mainModule
3. Simplify `classifyKeys`: `if traitSchema ? k then trait else class` — no DLQ, no structural sniffing
4. Migrate Tier 3 trait consumers (impermanence rules, firewall ports, service discovery) to `fleet` + `den.exports`
5. Delete Tier 3 machinery: `scopedDeferredTraits`, `traitModuleForScope`, `inheritTraits`, deferred trait evaluation
6. Keep Tier 1-2 traits as pipeline-time capability flags (booleans/enums for routing)
7. Target: -200 lines, `classifyKeys` becomes trivial

## Open Questions

1. **Handler installation timing.** Policies are discovered during tree walk (aspect policies registered via `register-aspect-policy`). Global and schema policies are known upfront. How do dynamically-registered aspect policies get installed as handlers? Do they need a "policy-registered" effect that triggers installation?

2. **Fan-out isolation.** When a policy resolves multiple users (`map (user: policy.resolve { inherit user; }) users`), each `expandContext` call uses `scope.provide` which creates a nested scope. Do sibling user scopes correctly isolate from each other? `scope.provide` is lexical — sibling provides don't see each other. This should work but needs verification.

3. **Enrichment + schema in same policy.** A policy that produces both `resolve { user = tux; isNixos = true; }` — mixed schema and enrichment in one effect. `expandContext` needs to install enrichment handlers AND walk the entity. The `scope.provide` for enrichment must wrap the entity walk so the entity's tree sees the enrichment.

4. **Post-pipeline reads.** `applyRoutes`, `applyForwardSpecs`, and `wrapCollectedClasses` run after the tree walk. They read from `scopedClassImports` which is scope-partitioned state. With `scope.provide`, is `scopedClassImports` still the right abstraction, or does the handler chain replace it?

5. **`scope.provide` performance.** Each `scope.provide` creates a new handler frame. With many policies and nested entities, the handler stack could get deep. Is this a performance concern in practice? The nix-effects implementation uses shallow handler lookup so depth shouldn't matter, but verify.

## What This Achieves

The pipeline becomes what the effect system was designed for: **small, composable handlers** that each do one thing. Policy dispatch is handler installation. Context expansion is `scope.provide`. Include resolution is handler composition. Dedup is structural scope isolation. No monolithic orchestrator needed.

The remaining pipeline code would be:
- `aspect.nix`: classifyKeys, emitClasses, emitTraits, resolveChildren (~600 lines)
- `pipeline.nix`: mkPipeline, fxResolve, applyRoutes, applyForwardSpecs, wrapCollectedClasses (~400 lines)
- `tree.nix`: emit-class, emit-trait, register-*, chain-*, constraint handlers (~350 lines)
- `include.nix`: include handler (~200 lines)
- `route.nix`: applyRoutes, wrapRouteModules (~150 lines)
- `forward.nix`: forwardHandler, buildForwardAspect (~200 lines)

Total: ~1900 lines, down from ~2800 current. Every component single-purpose, composable, testable in isolation.
