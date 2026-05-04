# Scope-Partitioned Pipeline State

## Status: Partially shipped (rev 4)

### What shipped (feat/fx-pipeline, 9 commits)

- **Scope infrastructure:** `mkScopeId`, scope-partitioned state fields, root scope init in mkPipeline
- **Dual-write handlers:** All emission handlers (classImports, traits, forwardSpecs) and bookkeeping handlers (constraints, chain, deferred, DLQ, aspectPolicies) write to both flat and scoped partitions
- **Scope push/pop in transitions:** `resolveFanOut` and `resolveContextValue` push/pop scopes, populating scopeParent/scopeChildren/scopeContexts tree
- **Dynamic trait schema registration:** `register-trait-schema` + `get-trait-schemas` effects, `emitTraitSchemas` in resolveChildren, `classifyKeys` reads dynamic registry
- **Trait inheritance:** `inheritTraits` walks scope tree (list/map/single strategies), `traitModuleForScope` synthesizes per-scope trait modules, `traitArgHandler` reads with scope-tree inheritance
- **Provide-to removal:** Handler, distribute-cross-entity.nix, crossEntityTraits parameter, resolveSiblingTransition all deleted. Scope-inheritance tests replace provide-to tests
- **currentCtx removal:** All readers use `scopeContexts.${currentScope}`
- **runSubPipeline elimination:** Function deleted, logic inlined at call sites. fxResolve trait module uses `traitModuleForScope` (scoped reads)

### What remains — see `2026-04-29-policy-route-class-delivery.md`

The remaining gaps are addressed by the **policy.route** spec, which replaces forwards with policy-driven routing over scope partitions:

1. **fxResolve class imports still read flat state.** Forward source content bypasses scoped partitions because `applyForwardSpecs` runs post-pipeline sub-pipelines. **Fix:** `policy.route` (Tier 1, ~90% of forwards) reads from scope partitions directly. Simplified forwards (Tier 2, ~10%) read via `sourceScopeId`. No sub-pipelines.

2. **Scope-prefixed dedup keys.** `ctxSeen` and `includeSeen` need scope-prefixed keys for Tier 2 forward inline-walk. Deferred from Task 2 of this spec.

3. **Flat state field removal.** Blocked on #1. Once fxResolve reads scoped, dual-writes can be removed.

4. **Unified policy dispatch.** Orthogonal. `dispatchPolicyIncludesHandler` still uses `includeOnly` filter. Deferred.

5. **Forward auto-detection.** After policy.route ships, the `emit-forward` handler can auto-classify simple forwards as routes — `!(spec ? adapterModule) && !(spec ? mapModule) && builtins.isList (spec.intoPath or [])`. Zero user migration: `den.provides.forward` silently routes Tier 1 cases through `policy.route` internally.

**Resume point:** Implement `2026-04-29-policy-route-class-delivery.md` migration steps 1-7. Then return here for items 3-5.

## Problem

The fx pipeline uses isolated sub-pipelines (separate `fx.run` executions with fresh state) for entity resolution during transitions. This creates three problems:

1. **Forward sub-pipelines are redundant.** `applyForwardSpecs` re-walks source aspects via `runSubPipeline` to collect class modules that the parent pipeline already collected during transition sub-pipelines. Eliminating this requires entity-scoped class module access, which the flat `classImports` structure doesn't provide.

2. **Cross-entity trait data requires explicit routing.** The `provide-to` mechanism (handler → accumulation → `distribute-cross-entity` → `crossEntityTraits` parameter on `fxResolve`) exists solely because sub-pipelines are isolated — sibling entities can't see each other's trait emissions. This is ~150 lines of routing infrastructure.

3. **`classImports` merging loses entity scoping.** When transition sub-pipelines merge results into the parent via `zipAttrsWith concatLists`, entity identity is lost. `classImports.homeManager` contains modules from ALL users — there is no way to retrieve a specific user's homeManager modules without re-walking.

4. **Trait schema registration is static, unlike policy registration.** `den.traits` is populated at module evaluation time by `aspect-schema.nix:collectFromAspects()`, which only folds from `den.aspects.<name>.traits` and `den.ful.<ns>.<name>.traits`. Anonymous inline aspects (`includes = [{ traits.foo = {...}; foo = [...]; }]`) declare the `traits` structural key but it's never collected into `den.traits`. Meanwhile, policy registration is dynamic — `emitAspectPolicies` discovers `aspect.policies` during tree-walk and registers via `register-aspect-policy` effect. This asymmetry prevents self-contained battery aspects: an aspect like `agenix-rekey` should bring its own trait schema (`traits.secrets = { collection = "list"; }`), emission pattern, AND routing policy — but including it via `includes` doesn't register the trait schema.

## Design

### Core Change

Replace sub-pipeline isolation with scope-partitioned state within a single pipeline execution. Scopes form a tree matching the entity context hierarchy. All pipeline output state is partitioned by scope. Handler bookkeeping state uses scope-prefixed keys. Context handlers are lexically scoped via `scope.provide` inside `aspectToEffect` (existing mechanism — no nix-effects changes). Trait schema registration becomes dynamic (discovered during tree-walk, like policies), enabling self-contained battery aspects.

### Scope Identity

A scope is identified by a deterministic string computed from context key-value pairs. The current `mkCtxId` (transition.nix:55-74) drops key names, which is safe for dedup but not for partitioning. The scope ID must be **injective** — different contexts must produce different IDs.

**New `mkScopeId`** includes key names:

```nix
mkScopeId = ctx:
  lib.concatStringsSep "," (
    lib.sort (a: b: a < b) (
      map (k:
        let v = ctx.${k}; in
        "${k}=${if builtins.isAttrs v && v ? name then v.name
               else if builtins.isString v then v
               else "<${builtins.typeOf v}:${k}>"}"
      ) (builtins.attrNames ctx)
    )
  );
# { host = { name = "igloo"; }; user = { name = "tux"; }; }
# → "host=igloo,user=tux"
```

The fallback encodes type + key name (e.g., `<int:priority>`) to preserve injectivity for non-string, non-attrset context values. Entity names are Nix identifiers (alphanumeric + hyphens) so `=`, `,`, `<`, `>` cannot appear in normal scope IDs.

Scopes form a tree determined by context nesting:

```
"host=igloo"                              ← host scope ({host})
├── "host=igloo,user=tux"                 ← user scope ({host, user})
│   └── "host=igloo,home=home-tux,user=tux"
└── "host=igloo,user=pingu"
    └── "host=igloo,home=home-pingu,user=pingu"
```

The parent-child relationship is tracked explicitly via `state.scopeParent.${childId} = parentId` (set at scope entry time), not inferred from key substrings.

### State Partitioning — Complete Inventory

Every pipeline state field classified as **scoped** (per-scope partition), **prefixed** (scope-prefixed keys in shared map), or **global** (shared across scopes).

| Field | Current Init | Scoping | Rationale |
|-------|-------------|---------|-----------|
| `classImports` | `_: {}` | **Scoped** | Core requirement — entity-scoped class module access |
| `traits` | `_: {}` | **Scoped** | Trait data per entity scope, inherited via scope tree |
| `deferredTraits` | `_: {}` | **Scoped** | Tier 3 trait functions belong to emitting scope |
| `consumedTraits` | `_: {}` | **Scoped** | partialOk validation per scope |
| `forwardSpecs` | `_: []` | **Scoped** | Forward specs processed against emitting scope's classImports |
| `aspectPolicies` | `_: {}` | **Scoped** | Policies discovered in scope fire within that scope's resolution |
| `deferredIncludes` | `_: []` | **Scoped** | Deferred includes drain into the scope that deferred them |
| `deadLetterQueue` | `_: []` | **Scoped** | Unregistered keys belong to emitting scope |
| `provideTo` | `_: []` | **Deleted** | Replaced by scope inheritance |
| `seen` (ctxSeen) | `_: {}` | **Prefixed** | `${scopeId}/${ctxKey}` — same context processable in different scopes |
| `includeSeen` | (dynamic) | **Prefixed** | `${scopeId}/${aspectId}` — same aspect includable in different scopes |
| `pathSet` | `_: {}` | **Prefixed** | `${scopeId}/${aspectPath}` — per-scope hasAspect tracking |
| `constraintRegistry` | `_: {}` | **Scoped** | Constraints registered in child scope don't affect siblings |
| `constraintFilters` | `_: []` | **Scoped** | Predicate filters scoped to registration scope |
| `includesChain` | `_: []` | **Scoped** | Chain stack is per-scope nesting depth |
| `currentCtx` | `_: ctx` | **Global** | Replaced by `scopeContexts.${currentScope}` lookup |
| `class` | `class` | **Global** | Target class is pipeline-wide |
| `transitionDepth` | (dynamic) | **Global** | Recursion guard is pipeline-wide |

**New state fields:**

| Field | Init | Purpose |
|-------|------|---------|
| `currentScope` | root scopeId | Active scope for emissions |
| `scopeStack` | `[]` | Stack of parent scope IDs for pop |
| `scopeContexts.${scopeId}` | `{}` | Saved ctx per scope for post-pipeline wrapping |
| `scopeParent.${scopeId}` | `null` | Parent scope ID for inheritance tree |
| `scopeChildren.${scopeId}` | `[]` | Child scope IDs (for aggregation) |
| `scopeProvenance.${scopeId}` | `null` | Policy name that created this scope (trace metadata, not dispatchable) |
| `traitSchemas` | seeded from `den.traits` | Dynamic trait schema registry, grows during tree-walk |

### Dynamic Trait Schema Registration

Trait schemas and policies should use the same registration pattern: dynamic discovery during tree-walk, not static collection at module evaluation time.

**Current asymmetry:**

| Concern | Registration | Mechanism |
|---------|-------------|-----------|
| Policies | Dynamic (tree-walk) | `emitAspectPolicies` → `register-aspect-policy` effect |
| Trait schemas | Static (module eval) | `aspect-schema.nix:collectFromAspects()` → `den.traits` |

**New unified pattern:** Both use dynamic tree-walk registration.

**New effect: `register-trait-schema`**

When `resolveChildren` processes an aspect's structural keys, a new `emitTraitSchemas` step (analogous to `emitAspectPolicies`) sends a `register-trait-schema` effect for each entry in `aspect.traits`:

```nix
emitTraitSchemas =
  aspect:
  let
    schemas = aspect.traits or {};
    aspectName = aspect.name or "<anon>";
  in
  if schemas == {} then fx.pure null
  else
    fx.seq (
      lib.mapAttrsToList (traitName: schema:
        fx.send "register-trait-schema" {
          name = traitName;
          inherit schema;
          ownerIdentity = identity.pathKey (identity.aspectPath aspect);
        }
      ) schemas
    );
```

**Handler: `register-trait-schema`**

Adds the schema to `state.traitSchemas` (the dynamic registry that replaces the static `den.traits` for `classifyKeys`). Then triggers `drain-dead-letters` to reclassify any keys that were previously unrecognized:

```nix
"register-trait-schema" =
  { param, state }:
  let
    current = state.traitSchemas null;  # always initialized (seeded from den.traits)
  in
  {
    resume = null;
    state = state // {
      traitSchemas = _: current // { ${param.name} = param.schema; };
    };
  };
```

**Integration with `classifyKeys`:** Currently reads `traitRegistry = den.traits or {}` (aspect.nix:299). Changes to read from `state.traitSchemas` (the dynamic registry). Static `den.traits` entries are seeded into `state.traitSchemas` at pipeline initialization, so existing top-level trait declarations continue to work.

**Integration with DLQ:** When `classifyKeys` encounters an unknown key (not in `classRegistry` or `traitSchemas`), it goes to `deadLetterQueue`. When `register-trait-schema` fires later (from an included battery aspect), `drain-dead-letters` reclassifies matching entries as trait emissions. This mechanism already exists for class registration — trait registration uses the same pattern.

**`resolveChildren` order becomes:**

```
emitTraitSchemas → emitAspectPolicies → emitIncludes → emitTransitions
```

Trait schemas register first so that sibling aspects' keys can be classified correctly. Policies register second. This matches the logical dependency: trait schemas define what keys exist, policies define how data routes.

**Late registration via includes:** If an aspect's `includes` brings in another aspect with trait schemas, those schemas register when the included aspect is walked — which is after `emitIncludes` for the parent. This is handled correctly by the DLQ mechanism: keys emitted before the schema was registered are dead-lettered, then `register-trait-schema` triggers `drain-dead-letters` to reclassify them. No ordering issue.

**What this enables — self-contained battery aspects:**

```nix
# agenix-rekey battery: one include brings schema + emission + routing
den.aspects.agenix-rekey = {
  # Trait schema — registered dynamically via register-trait-schema
  traits.secrets = {
    description = "Secrets managed by agenix-rekey";
    collection = "list";
  };

  # Routing policy — registered dynamically via register-aspect-policy
  policies.user-secrets = { user, secrets }:
    let userSecrets = builtins.filter (s: s.owner == user.name) secrets;
    in [ (policy.resolve { secrets = userSecrets; }) ];

  # The battery's own secret emissions
  secrets = [ { name = "wifi-password"; owner = "root"; } ];
};

# Consumer just includes the battery:
den.aspects.igloo.includes = [ den.aspects.agenix-rekey ];
# → trait schema registered, policy registered, secrets emitted — all from one include
```

Anonymous inline aspects also work:

```nix
den.aspects.igloo.includes = [
  {
    traits.greeting = { collection = "list"; };
    greeting = [ "hello" ];
  }
];
# → greeting trait schema registered, greeting data emitted
```

**What `aspect-schema.nix:collectFromAspects` becomes:** Retained for backward compatibility during this spec's implementation — top-level `den.aspects.<name>.traits` declarations continue to work. The authoritative registry is `state.traitSchemas` (dynamic), not `den.traits` (static). During this implementation, `den.traits` seeds `state.traitSchemas` at pipeline init. Removal of `collectFromAspects` for traits is explicitly deferred to a follow-up — it requires auditing all consumers of `den.traits` outside the pipeline (e.g., `traitOptions` in trait.nix module layer).

**New state field:**

| Field | Init | Scoping | Purpose |
|-------|------|---------|---------|
| `traitSchemas` | seeded from `den.traits` | **Global** | Dynamic trait schema registry, grows during tree-walk |

Global scoping because trait schemas define key classification — a trait registered in any scope should be recognizable everywhere (an aspect in scope "igloo,tux" registering `secrets` schema should cause `secrets` keys in scope "igloo" to be classified correctly too). This matches how `den.classes` (the class registry) is global.

**Note on `traitSchemas` (global) vs `aspectPolicies` (scoped) asymmetry:** This is intentional. Trait schemas answer "what kind of key is this?" — a classification question that must be consistent pipeline-wide (otherwise the same key would be classified as trait in one scope and dead-lettered in another). Policies answer "how should data route in this scope?" — a dispatch question that is inherently scoped. A battery aspect included in scope `"host=igloo,user=tux"` registers its trait schema globally (so all scopes can classify the key) but its policy only fires in the user scope (as expected).

### Unified Policy Dispatch

Currently policy dispatch is split across two phases:

1. **Tree-walk phase** (`dispatch-policy-includes` in tree.nix:259-314): Fires for entity roots. **Only processes policies with include/exclude effects.** Policies with `policy.resolve` effects are silently skipped (line 294: `includeOnly = filter (!hasResolve)`).

2. **Transition phase** (`dispatchAspectPolicies` in transition.nix): Fires during `into-transition`. Processes ALL policy effects including resolve. But only fires when `hasPolicies` is true and the transition handler matches.

This split creates a gap: aspect-included policies with `policy.resolve` effects (e.g., enrichment policies producing `isNixos`) don't fire during tree-walk and may not fire during transitions for entity kinds with flat transition topology (notably home entities, which resolve as standalone root entities with no fan-out transitions).

**In the scope-partitioned model:** The split is eliminated. All policy dispatch — include, exclude, AND resolve — happens during scope resolution. Each scope's policies fire as part of that scope's resolution, regardless of entity kind. The enrichment loop runs within each scope. No entity-kind-specific gaps.

**What gets deleted:**
- `dispatch-policy-includes` handler (tree.nix) — replaced by unified scope dispatch
- `dispatchPolicyIncludes` caller in `emitTransitions` (aspect.nix:685) — replaced by scope-level dispatch
- `dispatchAspectPolicies` in transition handler — folded into scope dispatch
- The `includeOnly` filter that skips resolve effects — all effects processed uniformly

### Future: Policy Deactivation

No mechanism currently exists to prevent a specific policy from firing in a scope. `policy.exclude` only targets aspects, not policies. This is a UX gap — a user cannot say "disable `host-to-users` for this entity."

The scope-partitioned model creates the foundation for policy deactivation: the scope-aware policy registry (`state.aspectPolicies` per scope) can support filtering. A `policy.deactivate policyRef` effect could remove entries from the current scope's registry. Design deferred to a follow-up spec — the mechanism is orthogonal to scope partitioning but benefits from it.

### Scope Switching

Transitions become inline scope changes within the same `fx.run`. No new effects needed — `aspectToEffect` already uses `scope.provide` internally when it detects `__scopeHandlers` on the tagged aspect (aspect.nix). Context handler scoping is lexical and automatic.

**Current (sub-pipeline fork):**
```
resolveFanOut → runSubPipeline(class, self, ctx)
  → fxFullResolve → mkPipeline → fx.run (separate execution, fresh state)
  → merge sub.classImports via zipAttrsWith concatLists
```

**New (inline scope change):**
```nix
# In resolveFanOut, replacing the runSubPipeline call:
fx.bind
  (fx.effects.state.modify (st: st // {
    currentScope = newScopeId;
    scopeStack = _: (st.scopeStack null) ++ [ st.currentScope ];
    scopeContexts = _: (st.scopeContexts null) // { ${newScopeId} = scopedCtx; };
    scopeParent = _: (st.scopeParent null) // { ${newScopeId} = st.currentScope; };
    scopeChildren = _: (st.scopeChildren null) // {
      ${st.currentScope} = ((st.scopeChildren null).${st.currentScope} or []) ++ [ newScopeId ];
    };
  }))
  (_:
    fx.bind (aspectToEffect tagged) (childResult:
      # scope.provide inside aspectToEffect restores parent context handlers
      fx.bind (fx.effects.state.modify (st: st // {
        currentScope = lib.last (st.scopeStack null);
        scopeStack = _: lib.init (st.scopeStack null);
      }))
      (_: fx.pure childResult)))
```

**Why this works:**
- `aspectToEffect tagged` detects `tagged.__scopeHandlers` and uses them for context resolution. For **parametric** aspects (functions), `scope.provide(newHandlers, computation)` wraps the resolution — when the computation completes, `scope.provide` automatically restores parent handlers. For **static** aspects (attrsets), `__scopeHandlers` propagate through the attrset and `ctxFromHandlers` extracts context directly (no `scope.provide` needed since static aspects don't send handler effects). Both paths produce correct scoping — the distinction matters for implementors reading `aspect.nix` but not for the scope switching mechanism.
- `state.modify` sets `currentScope` so emission handlers (emit-class, emit-trait) write to the correct scope partition.
- The scope stack tracks nesting for pop. After `aspectToEffect` completes, `state.modify` restores the parent scope.
- No nix-effects changes required. `scope.provide` and `state.modify` are existing primitives.

**Invariant: sibling transitions are sequential.** Within a scope, `resolveFanOut` processes sibling transitions one at a time (push scope A, walk, pop, push scope B, walk, pop). Nix's strict evaluation model enforces this — there is no concurrent or interleaved sibling processing. The linear `scopeStack` is correct because scope push/pop is always balanced within a single `fx.bind` chain.

**Nested transitions (e.g., host → user → home):** Work naturally. Each transition pushes a scope, walks the child aspect (which may itself contain transitions that push deeper scopes), and pops when done. The scope stack grows and shrinks with nesting depth.

### Scope Inheritance

Two directions of data flow replace `provide-to`:

**Upward propagation:** Emissions go into the emitting scope's partition. Parent scopes see children's data by walking `scopeChildren` links. No explicit routing — the tree structure is the routing.

**Downward inheritance:** When a consumer in scope `"host=igloo,user=tux"` needs trait data, it sees:
1. Own scope: `scopedTraits."host=igloo,user=tux".${traitName}`
2. Parent scope: `scopedTraits."host=igloo".${traitName}`
3. Recursively up to root

```nix
inheritTraits = scopeId: traitName: strategy:
  let
    own = scopedTraits.${scopeId}.${traitName} or emptyDefault;
    parentId = scopeParent.${scopeId} or null;
    parentData = if parentId == null then emptyDefault
                 else inheritTraits parentId traitName strategy;
  in
  if strategy == "single" then own  # single-valued: child wins, no inheritance
  else if strategy == "map" then mergeMaps parentData own
  else parentData ++ own;  # "list" (default)
```

`"single"` collection traits (e.g., a hostname) don't inherit — the emitting scope's value is authoritative. `"list"` and `"map"` accumulate up the scope tree.

**Sibling visibility:** Siblings never see each other directly. Cross-scope data flows UP to the parent (aggregation via `scopeChildren` walk), then DOWN via policy (filtering/transformation). This preserves commutativity — sibling evaluation order doesn't affect results.

**Fleet distribution example (sibling routing):**

```
"flake-system=x86_64-linux"
├── "flake-system=x86_64-linux,host=igloo"      ← emits network-hosts trait
└── "flake-system=x86_64-linux,host=server"      ← emits network-hosts trait
```

Both emit `network-hosts` to their own scope. A policy at the flake-system scope aggregates via `scopeChildren` walk and distributes:

```nix
den.policies.fleet-hosts = { flake-system, network-hosts }:
  # network-hosts = aggregated from all child scopes (all hosts)
  map (host: policy.resolve { network-hosts = allHosts; })
    (getHosts flake-system);
```

**Semantic evolution from unified-effects spec:** The unified-effects spec (2026-04-27) describes cross-entity traits as Tier 3 only (available at `evalModules` time via phase 2 distribution). With scope inheritance, cross-entity traits become available at **pipeline time** — a richer semantic. This is an intentional evolution: scope inheritance replaces the two-phase architecture with a single-phase model where the scope tree provides the routing that phase 2 previously handled explicitly. The unified-effects spec's Phase 2 section becomes obsolete.

### `has-handler` Probe Semantics

Parametric aspects use `has-handler` to check if a context value exists (for deferral). Under scope-partitioned state, `has-handler` probes the handler set installed by `scope.provide` for the current scope. This is unchanged from current behavior — `scope.provide` installs handlers lexically, and `has-handler` checks the current handler frame.

For trait consumers deferring on trait args: the `traitArgHandler` is installed per-scope after trait collection completes in that scope. A child scope's trait arg handler should also see parent-scope trait data via inheritance. This means trait arg handlers should read from `inheritTraits` (own + parent chain), not just the current scope's traits.

### Enrichment and Scopes

Enrichment policies (isNixos, isDarwin, etc.) fire within a scope and inject enrichment keys into that scope's context. `scopeContexts.${scopeId}` captures the enriched context at scope completion.

**Enrichment ordering within a scope:** Enrichment policies fire during scope dispatch (unified policy dispatch), which runs before child scope entry. This ensures that when a child scope is created via `push-scope`, the parent context (`parentCtx // newCtx`) already contains enrichment keys. The sequence within a scope is: enter scope → dispatch scope policies (enrichment loop) → walk aspect tree (which may create child scopes) → exit scope.

Child scopes inherit enrichment from parent context via `push-scope` (which installs `parentCtx // newCtx`, where `parentCtx` already has enrichment keys). Enrichment is downward-flowing only — a child scope's enrichment doesn't affect the parent or siblings. This matches current behavior where enrichment policies fire within a sub-pipeline and their effects stay within that sub-pipeline.

### Forward Elimination

With scope-partitioned classImports, forwards become pure lookups:

```nix
applyForwardSpecs = { forwardSpecs, scopeId, ... }:
  fold over forwardSpecs:
    bucket = scopedClassImports.${scopeId}.${spec.fromClass} or [];
    sourceModule = spec.mapModule {
      imports = bucket ++ lib.optional hasTraitSchemas (traitModuleForScope scopeId);
    };
    # buildForwardAspect, collectClassMods — unchanged
```

No `runSubPipeline`. The scope partition has exactly the right entity-scoped data.

**Invariant:** A forward spec emitted in scope S expects its source class content in scope S. This holds by construction: `emit-forward` writes to `scopedForwardSpecs.${state.currentScope}`, and `emit-class` writes to `scopedClassImports.${state.currentScope}`. Both use the same scope.

**Forward spec registration simplification:** `forwardHandler` currently captures `__resolveCtx` (the source entity's context for sub-pipeline scoping) and `__aspectPolicies` (for policy dispatch within the sub-pipeline). These exist solely because the sub-pipeline needs to reconstruct the source entity's resolution environment from scratch. With scope partitioning, the source entity's class modules are already in `scopedClassImports.${scopeId}` — no reconstruction needed. `forwardHandler` becomes a simple spec registration.

**Why this differs from the failed in-pipeline forward attempt:** A previous investigation (pre-scope-partitioning) tried resolving forwards inline by re-walking the source include tree with `scope.provide` + `aspectToEffect`. This caused infinite recursion because the source's include tree recurses into the parent pipeline's aspect graph — there is no dedup boundary. Scope-partitioned forwards avoid this entirely: they don't re-walk anything. `applyForwardSpecs` reads already-collected modules from the scope partition. The include tree was walked once during the transition that created the scope — the partition holds the result.

**`traitModuleForScope`** synthesizes a traitModule from the scope's collected traits plus inherited parent traits:

```nix
traitModuleForScope = scopeId:
  { ... }:
  {
    options._den.traits = lib.mkOption {
      type = lib.types.attrsOf lib.types.anything;
      default = {};
      internal = true;
    };
    config._den.traits = lib.mapAttrs (traitName: schema:
      let
        strategy = schema.collection or "list";
        ownData = scopedTraits.${scopeId}.${traitName} or emptyDefault;
        parentData = inheritTraits scopeId traitName strategy;
      in
      if strategy == "map" then mergeMaps ownData parentData
      else parentData ++ ownData
    ) traitSchemas;
  };
```

### fxResolve Composition

Post-pipeline, `fxResolve` composes per-scope results:

```nix
fxResolve = { class, self, ctx, ... }:
  let
    result = mkPipeline { inherit class; } { inherit self ctx; };
    allScopes = attrNames (result.state.scopedClassImports null);

    # Per-scope: wrap with scope's own ctx
    wrappedPerScope = mapAttrs (scopeId: classImports:
      wrapCollectedClasses (result.state.scopeContexts.${scopeId}) classImports
    ) (result.state.scopedClassImports null);

    # Per-scope: apply that scope's forward specs
    forwardedPerScope = mapAttrs (scopeId: wrappedImports:
      applyForwardSpecs {
        forwardSpecs = (result.state.scopedForwardSpecs null).${scopeId} or [];
        classImports = wrappedImports;
        traitModule = traitModuleForScope scopeId;
        hasTraitSchemas = traitSchemas != {};
      }
    ) wrappedPerScope;

    # Flatten: collect target class from all scopes
    allImports = concatMap (scopeId:
      forwardedPerScope.${scopeId}.classImports.${class} or []
    ) allScopes;

    rootScopeId = mkScopeId ctx;
  in
  {
    imports = allImports ++ lib.optional hasTraitSchemas (traitModuleForScope rootScopeId);
  };
```

**Key properties:**
1. Wrapping is per-scope — each scope's entries wrapped with that scope's ctx. No cross-entity contamination.
2. Forwards are per-scope — each scope's forward specs only see that scope's classImports.
3. Final flatten is safe — after wrapping and forwarding, results are opaque NixOS modules.
4. One traitModule per scope — root scope's traitModule gets the aggregated view with inheritance.

## Changes Per File

### `nix/lib/aspects/fx/pipeline.nix`

- **State initialization:** Replace flat `classImports`, `traits`, `forwardSpecs`, `provideTo` with scoped variants. Add `currentScope`, `scopeStack`, `scopeContexts`, `scopeParent`, `scopeChildren`.
- **`fxResolve`:** Per-scope wrapping, per-scope forward application, flatten. Remove `crossEntityTraits` parameter. Remove `provideTo` handling.
- **`applyForwardSpecs`:** Read from scope partition. Remove `runSubPipeline` call, `subTraitModule` synthesis, `provideTo` accumulation. Remove `__resolveCtx` usage.
- **`runSubPipeline`:** Delete. Retained only if needed for `fxFullResolve` external callers during migration (thin wrapper over single-pipeline execution with scope).
- **`wrapCollectedClasses`:** No changes to function. Called per-scope with scope-specific ctx.
- **New: `traitModuleForScope`:** Synthesizes traitModule from scope's traits + inherited parent traits.
- **New: `inheritTraits`:** Recursive scope-tree walk for trait inheritance.
- **New: `mkScopeId`:** Injective scope identity from context key-value pairs.

### `nix/lib/aspects/fx/handlers/transition.nix`

- **`resolveFanOut`:** Replace `runSubPipeline` call with inline `state.modify` (push scope) → `aspectToEffect` → `state.modify` (pop scope). Remove `mergeImports`.
- **`resolveContextValue`:** Unchanged — still creates scoped context and tags aspect.
- **`resolveSiblingTransition`:** Remove `provide-to` emission. Sibling data flows via scope inheritance.
- **`mkCtxId`:** Retained for `ctx-seen` dedup (backward compatible). New `mkScopeId` used for scope partitioning.

### `nix/lib/aspects/fx/handlers/tree.nix`

- **`classCollectorHandler`:** Write to `scopedClassImports.${state.currentScope}.${class}`.

### `nix/lib/aspects/fx/handlers/trait.nix`

- **`traitCollectorHandler`:** Write to `scopedTraits.${state.currentScope}.${traitName}`.
- **`traitArgHandler`:** Read from `inheritTraits` (own scope + parent chain).

### `nix/lib/aspects/fx/handlers/forward.nix`

- **`forwardHandler`:** Write to `scopedForwardSpecs.${state.currentScope}`. Remove `__resolveCtx` and `__aspectPolicies` capture.

### `nix/lib/aspects/fx/handlers/include.nix`

- **Include dedup:** Use `${state.currentScope}/${aspectId}` as key in `includeSeen`.

### `nix/lib/aspects/fx/handlers/ctx.nix`

- **`ctxSeenHandler`:** Prefix `key` with `${state.currentScope}/`.

### Files Deleted

- **`nix/lib/aspects/fx/handlers/provide-to.nix`** — replaced by scope inheritance.
- **`nix/lib/aspects/fx/distribute-cross-entity.nix`** — replaced by scope inheritance.

### Files Unchanged (specific functions)

- **`forward.nix`** `buildForwardAspect`, `mkDirectAspect`, `mkAdapterAspect` — module wrapping logic unchanged. Operates on source modules from scope partitions instead of sub-pipelines.
- **`nix/lib/forward.nix`** — `den.provides.forward` user-facing API unchanged.
- **`nix/lib/aspects/fx/aspect.nix`** — `wrapClassModule` unchanged. `emitClasses` unchanged. New: `emitTraitSchemas` (analogous to `emitAspectPolicies`), `classifyKeys` reads from `state.traitSchemas` instead of static `den.traits`. `resolveChildren` order: `emitTraitSchemas → emitAspectPolicies → emitIncludes → emitTransitions`.

## Migration

### Callers of `runSubPipeline`

| Caller | File | Migration |
|--------|------|-----------|
| `resolveFanOut` | transition.nix:198 | Replace with inline scope push/pop |
| `applyForwardSpecs` | pipeline.nix:220 | Replace with scope partition read |
| `has-aspect.nix` | aspects/has-aspect.nix:21 | Calls fxFullResolve — migrate to search across all scope partitions: `any (scopeId: (scopedClassImports.${scopeId}.${class} or []) != []) allScopes` |

### Callers of `fxFullResolve`

| Caller | Migration |
|--------|-----------|
| Test: fx-full-pipeline.nix (4 calls) | Read `scopedClassImports` |
| Test: include-dedup.nix (7 calls) | Read `scopedClassImports` |
| Test: provide-to.nix (7 calls) | Replace with scope inheritance tests |
| Test: traits.nix (2 calls) | Read `scopedTraits` |

### Test Evolution

**Tests whose signatures evolve (richer semantics):** Update assertions to match scope-partitioned output. The data is the same or richer — scope identity is preserved where it was previously lost.

**Tests that regress (lose identity → anon):** Stop work immediately. Investigate and restore identity before proceeding. Identity loss indicates a scope isolation bug.

**Provide-to tests (11 tests):** Replace entirely with scope inheritance tests covering the same scenarios: sibling routing, fleet distribution, map/list collection, cross-entity injection.

**New test scenarios needed:**
- Scope isolation: tux's classImports don't contain pingu's modules
- Scope inheritance: child scope sees parent's traits
- Sibling aggregation: parent scope aggregates children's traits via scopeChildren walk
- Forward reads from scope partition (not sub-pipeline)
- Nested scopes (host → user → home) maintain correct partitioning
- `mkScopeId` injectivity: different contexts produce different IDs
- Handler restore: after child scope completes, parent scope's handlers are active

## Guardrails

- **Identity regression = stop.** If any test shows identity loss (named → anonymous, scoped → unscoped), pause implementation and restore before continuing.
- **Semantic evolution = update tests.** If scope partitioning enriches test output (more context, better identity), update test expectations.
- **Multi-user isolation is primary invariant.** `tuxHm ≠ pinguHm` must hold across all 20+ multi-user tests at every commit.

## Risk Assessment

**Scope switching correctness.** `aspectToEffect` uses `scope.provide` for lexical handler scoping. `state.modify` tracks the active scope for emission handlers. These two mechanisms must stay in sync — if `scope.provide` restores parent handlers but `state.modify` hasn't popped the scope yet (or vice versa), emissions would go to the wrong partition. Mitigant: both happen in the same `fx.bind` chain with deterministic ordering.

**Scope-prefixed dedup equivalence.** `includeSeen.${scopeId}/${aspectId}` must be semantically equivalent to fresh `includeSeen` per sub-pipeline. The current sub-pipeline model starts with empty `includeSeen`, so the same aspect CAN be included in different sub-pipelines. Scope-prefixed dedup preserves this — same aspect includable in different scopes. No regression.

**Performance.** One pipeline execution instead of N sub-pipelines. State is larger (scope-partitioned) but no redundant re-walks. Net performance should improve for multi-entity flakes.

**`mkScopeId` injectivity.** Including key names (`host=igloo,user=tux`) prevents collisions that the old `mkCtxId` (`igloo,tux`) would have. Edge case: entity names containing `=` or `,` could collide. Mitigant: entity names are Nix identifiers (alphanumeric + hyphens), which cannot contain `=` or `,`.

**Unified-effects spec alignment.** This spec intentionally replaces the two-phase cross-entity architecture (unified-effects spec lines 279-293) with single-phase scope inheritance. Cross-entity traits become available at pipeline time rather than only at Tier 3 (evalModules time). This is a semantic enrichment. The unified-effects spec's Phase 2 section should be updated to reflect this evolution.
