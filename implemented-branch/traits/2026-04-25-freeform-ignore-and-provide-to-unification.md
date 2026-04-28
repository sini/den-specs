# Freeform-Ignore and Provide-To Unification

**Date:** 2026-04-25
**Status:** Draft
**Authors:** Sini (with agent assist)
**Depends on:** Traits and Structural Nesting (2026-04-25)

## Problem

Two issues remain from the traits implementation:

1. **Unregistered keys default to class emission.** `compileStatic`'s step 4 treats unregistered keys as classes for backward compatibility. The spec says they should be trace-warned and ignored. Forward-scoped class aliases (ad-hoc names like `"src"` used in forward battery sub-pipelines) aren't in `den.classes` but are valid — they need a separate recognition path.

2. **Provide-to and traits are parallel systems.** The `provide-to` structural key on aspects and the `distribute-provide-to.nix` distribution path collect and route labeled data cross-entity using `constantHandler` bindings. Traits do the same thing within a single entity. Two collection/distribution systems for the same pattern is unnecessary tech debt.

## Phase 1: Freeform-Ignore

### What changes

`classifyKeys` in `compileStatic` (aspect.nix) already separates keys into `classKeys` and `unregisteredClassKeys` buckets. Currently `compileStatic` recombines them: `allClassKeys = classKeys ++ unregisteredClassKeys`. The change: stop concatenating `unregisteredClassKeys` into `allClassKeys`. Instead, emit a `builtins.trace` warning for each unregistered key and ignore it.

The existing 4-step classification stays. Only step 4's handling changes:

```
4. k ∉ either registry, no recognized sub-keys → builtins.trace warning, ignored
```

### Forward-scoped class recognition

Forward battery sub-pipelines use ad-hoc class names (`"src"`, `"funny"`, etc.) that aren't in `den.classes`. These are valid — they're the pipeline's `targetClass`, available as `state.class` (already set in `mkPipeline`).

`classifyKeys` currently takes `aspect:` as its only argument and reads `classRegistry`/`traitRegistry` from module-level closures. This changes to accept `targetClass`:

```nix
classifyKeys = aspect: targetClass:
  # ... existing closure reads classRegistry, traitRegistry ...
  classKeys = builtins.filter (k:
    classRegistry ? ${k} || k == targetClass
  ) allKeys;
```

`compileStatic` threads `state.class` to `classifyKeys`. This requires `compileStatic` to read `state.class` — since `compileStatic` runs inside `fx.handle`, the value is available via an effect or passed as a parameter from `aspectToEffect`. The simplest path: `compileStatic` accepts `targetClass` as a parameter, and `aspectToEffect` reads it from the pipeline state at construction time (it's a plain field, not thunk-wrapped).

### Backward compatibility fallback

When both `den.classes` and `den.traits` are empty (no batteries imported, no schema declared), all non-structural keys are still treated as classes. This protects configs that don't use the schema system. This fallback already exists in the current code.

### Test class registration

Test classes used in CI need explicit registration:

- `templates/ci/provider/modules/den.nix`: add `den.classes.funny.description = "Test class for pipeline verification";`
- Any other ad-hoc test classes found by grep

Tests that use `funnyNames` with custom class keys on aspects (like `funny.names = [...]`) work because `funnyNames` calls `resolve "funny" aspect` — the pipeline's `targetClass` is `"funny"`, so `classifyKeys` recognizes it.

### Phase 1 deliverables

After this phase:
- Unregistered keys are ignored with trace warning (not emitted as classes)
- Forward-scoped class names recognized via `targetClass`
- Test classes registered in `den.classes`
- Backward compat fallback for empty registries preserved
- All existing tests passing

### Sunset plan for backward compat fallback

The `isEmpty` fallback can be removed once all templates import at least one battery. Currently all templates except `noflake` import batteries that register classes.

### Files

- Modify: `nix/lib/aspects/fx/aspect.nix` — `classifyKeys` signature change (add `targetClass`), step 4 changed to ignore
- Modify: `nix/lib/aspects/fx/pipeline.nix` — thread `state.class` to `compileStatic`
- Modify: `templates/ci/provider/modules/den.nix` — register `funny` class
- Update: test files as needed

## Phase 2: Provide-To Unification

### Overview

Replace the `provide-to` labeled data system with trait-based cross-entity routing. Aspects emit traits (no cross-entity awareness). Policies control where trait data flows. Targets consume trait data uniformly.

### What's removed

| Component | Location | Reason |
|-----------|----------|--------|
| `"provide-to"` in `structuralKeysSet` | `aspect.nix` | Aspects no longer declare `provide-to.*` keys |
| `provide-to.*` emission in `resolveChildren` | `aspect.nix` | Replaced by trait emissions |
| `distribute-provide-to.nix` | `fx/` | Replaced by `distribute-cross-entity.nix` |
| `provideToHandler` current payload | `handlers/provide-to.nix` | Payload changes to carry trait state |

### What's added/changed

#### Aspect-side: traits replace provide-to keys

```nix
# Before (provide-to labeled data):
den.aspects.example-site = { host, ... }: {
  provide-to.http-backends = [
    { address = host.meta.ip; port = 8080; vhost = "example.com"; }
  ];
};

# After (trait emission):
den.aspects.example-site = { host, ... }: {
  http-backends = [
    { address = host.meta.ip; port = 8080; vhost = "example.com"; }
  ];
};
# With: den.traits.http-backends = { description = "HTTP backends"; collection = "list"; };
```

The aspect declares what it IS (an http backend) without any awareness of cross-entity routing. The policy controls where the data flows.

#### Transport: provide-to effect payload changes

The `"provide-to"` effect name stays as the cross-entity transport mechanism. The payload changes:

```nix
# Before:
fx.send "provide-to" {
  label = "http-backends";
  content = [ { address = ...; } ];
  emitterCtx = ...;
  aspectName = ...;
  targetEntity = null;
}

# After:
fx.send "provide-to" {
  targetEntity = peer;
  traits = subPipelineState.traits;  # captured from sub-pipeline
}
```

#### `provideToHandler` — unchanged accumulation, new payload shape

The handler still appends to `state.provideTo` thunk. The payload structure changes but the accumulation pattern is identical. Forward handler's provideTo splicing (`forward.nix` lines 240-253) is shape-agnostic — it concatenates thunk lists. Correctness depends on all `"provide-to"` emission sites using the new payload.

#### Sibling routing: new sub-pipeline execution

**This is the key architectural change.** `resolveSiblingTransition` in `transition.nix` currently emits a bare `"provide-to"` marker with `content = null` — it does NOT run sub-pipelines. For trait-based routing, sibling routing must resolve the source entity's aspect tree in the peer's context to collect `state.traits`.

The new `resolveSiblingTransition` follows the pattern established by `resolveFanOut` (used in non-sibling transitions):

```nix
resolveSiblingTransition =
  sourceAspect: currentCtx: results: transition:
  builtins.foldl' (acc: indexed:
    fx.bind acc (innerResults:
      let
        newCtx = indexed.ctx;
        scopedCtx = currentCtx // newCtx;
        rawTarget = newCtx.${transition.routing.targetKey} or newCtx;
        targetEntity = rawTarget;

        # Run sub-pipeline for this peer to collect traits.
        # The source entity's stage aspect is resolved with the peer's
        # context merged in, producing trait emissions scoped to that peer.
        subResult = den.lib.aspects.fx.pipeline.fxFullResolve {
          class = transition.routing.targetClass or "nixos";
          self = den.lib.resolveStage transition.routing.from scopedCtx;
          ctx = scopedCtx;
        };
      in
      fx.send "provide-to" {
        inherit targetEntity;
        traits = subResult.state.traits null;
      }
    )
  ) (fx.pure results) (lib.imap0 (i: ctx: { inherit i ctx; }) transition.contexts);
```

**Key decisions:**
- **What aspect tree is the root?** The source entity's stage aspect (`resolveStage from scopedCtx`), re-resolved with the peer's context. This is the same root the non-sibling `resolveTransition` uses via `resolve-target`.
- **What handlers?** `fxFullResolve` uses `defaultHandlers` which includes `traitCollectorHandler`. The sub-pipeline runs with the peer's merged context, so Tier 2 traits resolve with the peer's entity args.
- **Thunk unwrapping:** `subResult.state.traits null` unwraps the thunk before sending. The `"provide-to"` payload carries plain data, not thunks.
- **Class module routing unchanged:** The non-sibling path's `state.imports` capture for class module injection stays as-is. Sibling routing doesn't need `state.imports` — class modules route through the non-sibling path.

#### Distribution: `distribute-cross-entity.nix`

Replaces `distribute-provide-to.nix`. Groups emissions by target entity, merges trait data per-target:

```nix
# Input: list of { targetEntity, traits }
# Output: { "<target-name>" = { "<traitName>" = [ values... ] | { merged-attrset }; }; }

distributeCrossEntityTraits = { traitSchemas }: emissions:
  let
    grouped = groupByTarget emissions;
  in
  lib.mapAttrs (_targetId: traitSets:
    let
      allTraitNames = lib.unique (lib.concatMap builtins.attrNames traitSets);
    in
    lib.genAttrs allTraitNames (name:
      let
        strategy = (traitSchemas.${name} or {}).collection or "list";
        values = lib.concatMap (ts: ts.${name} or []) traitSets;
      in
      if strategy == "map" then
        builtins.foldl' (acc: v:
          let dupes = builtins.filter (k: acc ? ${k}) (builtins.attrNames v);
          in if dupes != [] then
            throw "den: cross-entity trait '${name}' map collection: duplicate key '${builtins.head dupes}'"
          else acc // v
        ) {} values
      else
        values
    )
  ) grouped;
```

The function is collection-strategy-aware: "list" traits concatenate, "map" traits merge attrsets with duplicate detection. `traitSchemas` is passed to the function so it can look up the strategy.

#### Injection: eval-time, not pipeline-time

Cross-entity trait data is injected into the target's `_den.traits` module at eval time — NOT at pipeline time. The target's pipeline already ran in phase 1. Phase 2 does NOT re-run it.

This means cross-entity traits are available to class module consumers (`{ http-backends, config, ... }:`) but NOT to pipeline-time discriminators. This is correct: cross-entity data can't drive pipeline-time decisions because the source's pipeline may not have run yet when the target's discriminators fire.

#### Integration point: fxResolve

`fxResolve` in `pipeline.nix` currently generates a `traitModule` with within-entity data. It needs an additional parameter for cross-entity trait data:

```nix
fxResolve = { class, self, ctx, crossEntityTraits ? {} }:
  # ...
  traitModule = { config, lib, pkgs, options, modulesPath, ... }@moduleArgs: {
    config._den.traits = lib.mapAttrs (traitName: schema:
      let
        strategy = schema.collection or "list";
        pipelineData = ...;
        deferredData = ...;
        crossEntity = crossEntityTraits.${traitName} or (if strategy == "map" then {} else []);
      in
      if strategy == "map" then
        # Merge all sources, duplicate-check at each boundary
        builtins.foldl' (acc: d: ...) (pipelineData // crossEntity) deferredData
      else
        pipelineData ++ deferredData ++ crossEntity
    ) traitSchemas;
  };
```

#### Output-module orchestration

The caller of `fxResolve` must orchestrate the two-phase flow. Currently entity resolution (`nix/lib/aspects/default.nix` `fxResolveTree`) calls `fxResolve` directly. The new flow:

1. **Phase 1:** Run `fxFullResolve` for each entity (existing behavior). Each entity's pipeline collects within-entity traits in `state.traits` and cross-entity emissions in `state.provideTo`.
2. **Collect:** Unwrap all entities' `state.provideTo` thunks.
3. **Distribute:** Call `distributeCrossEntityTraits { inherit traitSchemas; }` on the collected emissions. Produces `{ "<target-name>" = { "<traitName>" = data; }; }`.
4. **Phase 2:** For each entity, call `fxResolve` with `crossEntityTraits = distributedData.${entityName} or {}`. The `traitModule` merges cross-entity data alongside within-entity data.

This orchestration lives in `fxResolveTree` (or a new function wrapping it). The output modules (`osConfigurations.nix`, `hmConfigurations.nix`) consume the result as before — the two-phase flow is internal to the resolution layer.

#### Non-sibling transition path

The non-sibling path in `resolveTransition` (child-entity routing via `resolveFanOut`) is unaffected. It captures `state.imports` for class module injection but does not capture `state.traits`. Cross-entity trait routing is only for sibling policies (`from == to`). If child-entity trait routing is needed in the future, it would be a separate design decision.

### What stays

- **`provideToHandler`** — transport mechanism, accumulates in `state.provideTo`
- **Phase 1/Phase 2 architecture** — two-phase resolution is fundamental
- **Cycle prevention** — phase 2 does not re-run pipelines
- **Class module cross-entity routing** — `state.imports` capture in non-sibling transitions unchanged
- **Forward sub-pipeline provideTo splicing** — shape-agnostic thunk concatenation

### Test coverage

New tests must cover:
1. Trait emission in source pipeline (existing within-entity tests cover this)
2. Sibling routing sub-pipeline capturing `state.traits`
3. `distributeCrossEntityTraits` grouping and merging (list and map collection)
4. Injection into target's `_den.traits` via `crossEntityTraits` parameter
5. End-to-end: source emits trait → policy routes to peer → peer's class module consumes

### Files

- Modify: `nix/lib/aspects/fx/aspect.nix` — remove `"provide-to"` from `structuralKeysSet`, remove `provide-to.*` emission from `resolveChildren`
- Rename: `nix/lib/aspects/fx/distribute-provide-to.nix` → `nix/lib/aspects/fx/distribute-cross-entity.nix`
- Modify: `nix/lib/aspects/fx/handlers/provide-to.nix` — payload shape change
- Modify: `nix/lib/aspects/fx/handlers/transition.nix` — sibling routing sub-pipeline execution
- Modify: `nix/lib/aspects/fx/pipeline.nix` — `fxResolve` accepts `crossEntityTraits`
- Modify: `nix/lib/aspects/default.nix` — two-phase orchestration in `fxResolveTree`
- Update: `templates/ci/modules/features/provide-to.nix` — rewrite tests for trait-based routing

### Migration

Since the policies spec is not yet merged, `provide-to` as a structural key on aspects has no external consumers. This is a clean replacement — no deprecation path needed. The `"provide-to"` effect stays as internal transport.

## End-to-End Example (Phase 2)

```nix
# Trait declaration
den.traits.http-backends = {
  description = "HTTP backend endpoints for load balancing";
  collection = "list";
};

# Source aspects emit trait data — no cross-entity awareness
den.aspects.example-site = { host, ... }: {
  nixos.services.nginx.virtualHosts."example.com".locations."/".proxyPass =
    "http://localhost:8080";
  http-backends = [
    { address = host.meta.ip; port = 8080; vhost = "example.com"; }
  ];
};

# Policy routes trait data to peers
den.policies.host-to-peers = {
  from = "host"; to = "host"; as = "peer";
  resolve = { host, ... }:
    map (peer: { inherit peer; })
      (filter (h: h.name != host.name) (attrValues den.hosts.${host.system}));
};

# Target aspect consumes — same syntax as within-entity
den.aspects.loadbalancer = {
  nixos = { http-backends, config, ... }: {
    services.haproxy.enable = true;
    services.haproxy.frontends = lib.listToAttrs (map (b: {
      name = b.vhost;
      value.backends = [{ inherit (b) address port; }];
    }) http-backends);
  };
};
```

Pipeline flow:
1. Phase 1: each host's pipeline runs, collecting within-entity traits in `state.traits`
2. Transition handler: sibling routing runs sub-pipeline per peer, captures `state.traits`, emits `"provide-to"` with trait data
3. Phase 2: `distributeCrossEntityTraits` groups by target, produces per-target trait maps
4. Target's `traitModule` merges cross-entity data into `_den.traits` at eval time
5. Consumer forces `http-backends` → gets unified list from all peers
