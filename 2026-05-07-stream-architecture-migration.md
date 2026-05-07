# Stream Architecture Migration

Den's fx-pipeline currently runs all entities through a single shared trampoline, forcing ~2,500 lines of coordination machinery (scope trees, deferred includes, late dispatch, 4-phase post-pipeline assembly). Ned demonstrates that the same semantics — topology fan-out, cross-host data sharing, scoped DI, class forwarding — can be achieved with stream composition and a Cycle.js fixed-point, using ~50x less infrastructure code.

This spec defines the migration from Den's shared-pipeline model to a stream-based architecture (Approach A), preserving all user-facing APIs while eliminating the internal coordination overhead.

## Architecture Overview

### Current Model (Shared Pipeline)

```
aspect tree → single DFS trampoline (35+ handlers, 22 state fields)
  → scopedClassImports, scopedRoutes, scopedProvides (accumulated state)
  → Phase 1: wrapPerScope (DI partial application, cross-scope dedup)
  → Phase 2: applyProvides (provide injection)
  → Phase 3: applyRoutes (forward route application)
  → Phase 4: applyInstantiates (per-host subtree extraction → nixosSystem)
```

### Proposed Model (Stream Composition)

```
topology stream → per-entity ctxD scoping (effect rotation)
  → aspect resolution as stream elements
  → class-typed sink streams (nixos, homeManager, darwin, ...)
  → Cycle fixed-point wires cross-entity data (pipes, forwards)
  → .toList → nixosSystem / darwinSystem
```

The fundamental shift: instead of walking all entities in one pipeline and re-partitioning results afterward, each entity resolves independently in its own scoped stream. Cross-entity concerns (pipe.collect, forwards, policy visibility) are handled by the Cycle fixed-point, which gives all sinks simultaneous access to all sources via Nix laziness.

### Core Types (from Ned)

| Type | Convention | Role in Den |
|------|-----------|-------------|
| `ST a` | `nameS` | Wrapped fx.stream with fluent API. Replaces raw class import lists. |
| `Driver` (`ST → ST`) | `nameD` | Stream transformer. Replaces handlers + post-pipeline phases. |
| `Cycle` (`sources → sinks`) | `nameC` | Fixed-point composition. Replaces scope tree + late dispatch. |
| `Topology` | `nameT` | Fan-out driver. Replaces push-scope + restore-scope. |

### Laziness Contract

The Cycle fixed-point (`ned.run`) ties sources and sinks in mutual recursion. This is safe because:

- Stream construction is lazy (building `More`/`Done`/`bind` trees)
- Stream consumption is strict (`.toList` runs the trampoline)
- Drivers transform stream *descriptions*, never forcing values

**Safe:** Host A references host B's topology data (static). Host A's NixOS config lazily references a merged stream of all hosts' pipe data.

**Breaks:** Calling `.toList` inside a cycle body, `builtins.length` on an unresolved stream, or any strict operation that forces the cycle back on itself.

This is the same laziness contract Den's config thunks already rely on — it becomes explicit rather than special-cased.

## Concern 1: Topology Fan-Out

### Current (Den)

`push-scope` creates scope IDs, records `scopeContexts`/`scopeParent`/`scopeEntityClass`, copies deferred includes, returns scope handlers. `restore-scope` pops the stack. `resolve-schema-entity` orchestrates push→resolve→drain→propagate→restore. Late dispatch re-visits siblings after all are resolved. ~400 lines across 6 handlers + schema.nix.

### Proposed

Adopt Ned's `hostsT`/`usersT` pattern directly:

```nix
hostsT = topoS: compS:
  let
    hosts = buildHostList topoS;  # materialize topology (safe — upstream data)
  in
  builtins.foldl' (accS: host:
    accS (ctxD { inherit host; } compS)
  ) st hosts;

usersT = compS:
  st ({ host }:
    let users = buildUserList host;
    in builtins.foldl' (accS: user:
      accS (ctxD { inherit user; } compS)
    ) st users
  );
```

**What disappears:**
- `push-scope` / `restore-scope` handlers (~110 lines)
- `scopeContexts`, `scopeParent`, `scopeEntityClass` state fields
- `mkScopeId` canonical key computation
- Deferred include propagation from parent to child scope
- `inLateDispatchStack` management

**What stays:**
- `buildHostList` — the topology materialization logic (merging duplicate host keys, inferring class from system)
- `buildUserList` — iterating `host.users`
- The `ctxD` function itself (effect rotation via `fx.rotate`)

### Custom Entity Kinds and the Static/Dynamic Split

Den's topology is policy-driven: `policy.resolve.to "fleet" {...}` creates entity scopes during resolution. This seems incompatible with Ned's static Cycle definition. But in practice, topology structure and content are separable:

**Topology structure is static.** Topology-shaping policies live at the schema level:

```nix
den.schema.flake.includes = [ den.policies.to-fleet ];
den.schema.fleet.includes = [ den.policies.fleet-to-hosts ];
```

These are known at definition time. The schema chain tells you: "flake scope creates fleet scopes, fleet scope creates host scopes." The policies iterate `den.hosts` (also static) to enumerate instances. This deterministic chain becomes the Cycle's driver composition.

**Content within scopes is dynamic.** Policy enrichment (`isNixos`, `isDarwin`), aspect includes, and pipe data are resolved at stream evaluation time within each scope.

| Concern | When resolved | Mechanism |
|---------|--------------|-----------|
| Topology structure (entity kind nesting) | Definition time | Schema policy chain → driver composition |
| Entity instances (which hosts/users/fleets) | Definition time | `den.hosts`, `den.schema` → topology streams |
| Policy enrichment | Stream evaluation time | `dispatchD` + `converge` within each scope |
| Aspect includes | Stream evaluation time | Policy effects → stream concat |
| Class routing | Stream evaluation time | Tagged elements + filter |

**Generic entity driver factory.** All entity kinds share the same driver shape — only the context bindings differ:

```nix
mkEntityDriver = kind: entitiesS: compS:
  entitiesS.flatMap (entity:
    ctxD { ${kind} = entity; } compS
  );

fleetT = mkEntityDriver "fleet";
hostT  = mkEntityDriver "host";
userT  = mkEntityDriver "user";
```

The **nesting order** comes from the schema policy chain and becomes driver composition in the Cycle:

```nix
mainC = sources: {
  perFleet = sources.aspects (fleetT fleetListS);
  perHost  = sources.perFleet (hostT hostListS);
  perUser  = sources.perHost (userT userListS);
  # ...
};
```

**pipe.collect entity kind filtering** falls out of stream topology naturally. User scopes are *downstream* of host scopes (nested inside `userT`), not siblings. Only elements at the same topology level are siblings. The current `findMatchingSiblings` entity kind check becomes unnecessary — the stream structure enforces it.

### Schema-to-Driver Generation

Each `den.schema.<kind>` entry with `isEntity = true` generates a topology driver via `mkEntityDriver`. The schema's `includes` (policies that produce `policy.resolve.to` effects) determine which entity instances the driver fans out over. Generation happens at Cycle definition time by evaluating schema-level policies to extract their resolve targets.

```nix
# Schema declares the chain
den.schema.flake.includes = [ policies.to-fleet ];
den.schema.fleet.includes = [ policies.fleet-to-hosts ];

# At Cycle definition time, extract topology from schema policies:
# 1. Evaluate flake-level policies → discover "fleet" resolve targets
# 2. Evaluate fleet-level policies → discover "host" resolve targets
# 3. Build driver chain: fleetT → hostT → userT
```

This is a one-time evaluation of schema-level policies (not aspect-level), producing the static Cycle structure that all dynamic content flows through.

## Concern 1b: User-Defined Classes

### Class Registration and Discovery

Classes are registered via `den.classes.<name>.description = "..."` (explicit) or implicitly — any aspect key that isn't structural, a pipe key, or a recognized nested key becomes a class key during `classifyKeys`.

### Stream Mapping

Aspect resolution produces **tagged stream elements**. Class routing is a filter on the tag:

```nix
# Aspect resolution emits tagged elements
resolveAspect = aspect:
  map (k: { class = k; module = aspect.${k}; }) (classKeysOf aspect);

# Class routing — dynamic, no upfront class list needed
nixosS     = allModulesS.filter (e: e.class == "nixos").map (e: e.module);
darwinS    = allModulesS.filter (e: e.class == "darwin").map (e: e.module);
customS    = allModulesS.filter (e: e.class == "custom").map (e: e.module);
```

Classes don't need to be known at Cycle definition time — the routing driver splits by tag dynamically. New user-defined classes appear as new tag values and get their own filtered stream.

### Static Forwarding (`forwardTo`)

`den.classes.os.forwardTo = "nixos"` declares that `os` class content automatically flows into `nixos`. In the stream model, this is a concat wired from the class registry:

```nix
nixosS = nixosS.concat (osS);  # os class → nixos via forwardTo
```

### Custom Class Instantiation

| Pattern | Stream terminal |
|---------|----------------|
| Built-in class (nixos, darwin) | `.toList` → `nixosSystem` / `darwinSystem` |
| Custom with `forwardTo` | Content already concat'd into target sink |
| Custom with own `instantiate` | `.toList` → custom instantiate function |
| Raw module list | Sink exposed directly for user consumption |

## Concern 2: Policy Dispatch

### Current (Den)

Policies are registered during tree walk, dispatched at entity boundaries via fixed-point enrichment loop (up to 10 iterations), late-dispatched to siblings after fan-out. ~800+ lines across iterate.nix, dispatch.nix, classify.nix, schema.nix, emit-policy-effects.nix, dispatch-policies.nix, constraint.nix, policy.nix.

### Proposed

Policies become stream elements processed by a dispatch driver.

**Context-dependent dispatch** — the `{ host, ... }:` signature becomes a stream filter:

```nix
dispatchD = policiesS: ctxS:
  let
    satisfiable = p: ctx:
      let fargs = lib.functionArgs p.fn;
          required = filter (k: !fargs.${k}) (attrNames fargs);
      in all (k: ctx ? ${k}) required;
  in
  ctxS.flatMap (ctx:
    let
      active = filter (p: satisfiable p ctx) policiesS.toList;
      # ... converge enrichment, emit effects
    in
    ...
  );
```

**Enrichment convergence** — same fixed-point algorithm, expressed as a recursive function instead of handler-state-mutation:

```nix
converge = policies: ctx:
  let
    fired = filter (satisfiable ctx) policies;
    effects = concatMap (p: p.fn ctx) fired;
    enrichment = extractEnrichment effects;
    newCtx = ctx // enrichment;
  in
  if enrichment == {} then { inherit effects ctx; }
  else converge (removeBy name fired policies) newCtx;
```

~15 lines instead of ~100. Same semantics, no handler dispatch overhead.

**Late dispatch — eliminated.** In the Cycle fixed-point, all policies are visible to all entities simultaneously. Policies are a shared source stream, not sequential registrations. There is no "sibling A registered a policy that B can't see" because the policy stream is available to all sinks at construction time.

**Constraints/exclusions** — stream filter:

```nix
activeS = policiesS.filter (p: !(excluded ? p.name));
```

Scope ancestry for subtree exclusions: the exclusion is part of the scope context, inherited by child scopes via `ctxD`.

### Policy Effects

| Effect | Current | Stream equivalent |
|--------|---------|-------------------|
| `policy.resolve` | `emit-policy-effects` → `processSchemaResolves` → `push-scope` | Topology driver call (`hostsT`, `usersT`, custom) |
| `policy.include` | `emit-include` → tree walk | Concat into aspect stream |
| `policy.exclude` | `register-constraint` → checked at gate | Filter on aspect stream |
| `policy.route` | `register-route` → Phase 3 | Stream concat between class sinks |
| `policy.provide` | `register-provide` → Phase 2 | Direct injection into class sink |
| `policy.instantiate` | `register-instantiate` → Phase 4 | `.toList` → `nixosSystem` at terminal |
| `pipe.*` | `register-pipe-effect` → `assemblePipes` | Stream combinators (below) |

## Concern 3: Forwards / Routes

### Current (Den)

Two-tier system: Tier 1 (simple routes, direct module movement) and Tier 2 (complex routes with guards, adaptArgs, adapter functors). Routes registered during walk, applied in post-pipeline Phase 3. ~580 lines across compile-forward.nix, forward.nix, route/apply.nix, route/wrap.nix, propagate-routes.nix.

### Proposed

A forward is a stream operation: take elements from one class sink, transform, put into another class sink.

**Simple forwards** (class-to-class):

```nix
mainC = sources: {
  nixos = sources.aspects (ST.sel.flat "nixos")
    (sources.aspects (ST.sel.flat "custom"));  # custom content flows into nixos
  # ...
};
```

One line of stream concat replaces the entire Tier 1 route system.

**Guarded forwards** (`lib.mkIf` on target config):

Guards are a NixOS module-level concern, not a pipeline concern. The module itself carries the guard:

```nix
{ config, ... }: lib.mkIf config.programs.vim.enable { ... }
```

This works identically regardless of how the module is delivered. No pipeline-level guard machinery needed.

**Adapter functor pattern** (`den.fwd.${key}`):

The `den.fwd.${key}` submodule pattern (isolated namespace with custom `specialArgs`) is essential NixOS module system scaffolding. It stays (~80 lines). But the delivery mechanism changes: instead of registering a route and applying it post-pipeline, the adapter aspect is a stream element that flows into the target class sink directly.

**`adaptArgs`** (injecting `osConfig` etc.):

Becomes a `ctxD` wrapping the forwarded stream segment:

```nix
forwardedS = sourceS (ctxD { osConfig = config; });
```

**What disappears:**
- Route registration handlers (~50 lines)
- `propagate-routes` handler (~50 lines)
- `applyRoutes` / `applySimpleRoute` / `applyComplexRoute` (~150 lines)
- `dedupRoutes` / `findChildScopeKeys` (~35 lines)
- Route dedup key tracking in state
- Tier 1/2 classification logic (~30 lines)
- Post-pipeline Phase 3 entirely

**What stays:**
- `mkAdapterAspect` / adapter functor construction (~80 lines) — NixOS module system integration
- `forwardItem` / `forwardEach` user API — rewritten to produce stream operations instead of route specs

## Concern 4: Pipes / Quirks (Cross-Entity Data Sharing)

### Current (Den)

`assemblePipes` (632 lines) runs post-pipeline. Extracts raw pipe entries from `scopedClassImports`, applies transform stages, handles `pipe.collect` (sibling harvesting via scope tree walk), `pipe.expose` (bottom-up tree accumulation), `pipe.to` (targeted delivery), provenance tagging, config thunk resolution.

### Proposed

Each pipe operation maps to a stream combinator. The Cycle fixed-point provides cross-entity visibility natively.

**Local aggregation** (same host):

Aspects emit on a named quirk sink. All emissions within a scope are naturally concatenated by the stream. A driver merges them and provides via `ctxD`:

```nix
mainC = sources: {
  firewall = st ({ host }: { ports = [ 80 443 ]; });
  networking = sources.firewall (ctxD { firewall = sources.firewall; });
  # modules in networking can access `firewall` as a function arg via effect handler
};
```

**`pipe.collect`** (sibling → sibling):

`hostsT` already concatenates all hosts into one stream. The merged stream IS the collection:

```nix
mainC = sources: {
  backends = st ({ host }: { addr = host.ip; port = 80; });
  nixos = sources.backends.map (allBackends: ...);
  # allBackends contains every host's contribution via Cycle fixed-point
};
```

Self-exclusion (current host filtered out) is a `.filter`:

```nix
peersS = allBackendsS.filter (b: b.host.name != currentHost.name);
```

**`pipe.expose`** (child → parent):

User-scoped sinks are sub-streams of host-scoped sinks (because `usersT` concatenates results back into the host stream). Data flows upward by construction — no explicit expose needed.

**`pipe.to`** (targeted delivery):

A driver that filters the source stream by aspect identity before injecting:

```nix
targetedD = targetAspects: dataS:
  dataS.filter (d: elem d.source targetAspects);
```

**`pipe.withProvenance`**:

```nix
dataS.map (val: { value = val; source = host; })
```

One line.

**Config thunks** (lazy NixOS config access):

The Cycle fixed-point handles this natively. `sinks.nixos.toList` feeds into `nixosSystem`, and Nix laziness lets you reference `config` from the result. Same forward-reference trick, but it falls out of the architecture.

**Transform stages** (`pipe.filter`, `pipe.transform`, `pipe.fold`, `pipe.append`, `pipe.for`):

Direct stream combinator equivalents:

| Stage | Stream equivalent |
|-------|-------------------|
| `pipe.filter pred` | `dataS.filter pred` |
| `pipe.transform f` | `dataS.map f` |
| `pipe.fold f init` | `st (dataS.toList |> foldl' f init)` |
| `pipe.append val` | `dataS (st val)` |
| `pipe.for f` | `st (f dataS.toList)` |

**What disappears:**
- `assemblePipes` entirely (632 lines)
- `findMatchingSiblings` / `collectFromPeers` (~45 lines)
- `collectAllExposed` (~65 lines)
- `buildTargetedData` (~35 lines)
- Config thunk marking/resolution (~40 lines)
- Provenance tracking infrastructure (~70 lines)
- Scope tree walking utilities

## Concern 5: Class Module DI

### Current (Den)

`wrapClassModule` (254 lines) inspects module function args, partially applies Den-injected values, builds collision validator, resolves 3-level collision policy cascade, strips enrichment-only args. `bind` handler (96 lines) probes arg availability via `hasHandler` effects, defers if unsatisfied. `drain` handler (40 lines) retries deferred includes when context widens.

### Proposed

**Single injection path via `ctxD`.** All context values (host, user, class, enrichment keys, pipe data) flow through effect handlers. Modules receive them via the same mechanism:

```nix
# Module receives host from effect handler, not from partial application
myModule = { host, config, ... }: {
  networking.hostName = host.name;
};

# ctxD installs host as a handler; module resolves it via fx.bind.fn
ctxD { inherit host; } (st myModule)
```

**Collision detection — eliminated.** With one injection path (effect handlers), there is no second source to collide with. The module never advertises `host` in its NixOS `functionArgs` — the effect handler resolves it before NixOS sees the module. The entire `mkCollisionValidator`, `resolveCollisionPolicy`, 3-level policy cascade, `class-wins`/`den-wins` merge ordering — all gone.

**Enrichment args — just scope context.** Values like `isNixos`, `isDarwin` are added to the handler stack via `ctxD`, same as `host` and `user`. No special stripping needed because the module doesn't advertise them to NixOS.

**Bind/defer — replaced by stream topology.** A module needing `{ host, user }` is placed in the stream segment downstream of both `hostsT` and `usersT`. The stream guarantees args are available by construction. No probing, no deferral, no drain cycle.

**Pipe args** — same: pipe data flows through `ctxD` after pipe combinators produce it. Modules downstream of the pipe segment see the data. No special `hasPipeArgs` detection.

**What disappears:**
- `wrapClassModule` complex path (~120 lines) — the wrapper, collision validator, policy resolution
- `mkCollisionValidator` (~30 lines)
- `resolveCollisionPolicy` (~20 lines)
- `mergeEnrichment` / `stripEnrichmentArgs` (~50 lines)
- `wrapDeferredImports` (~35 lines)
- Bind handler probe/defer logic (~90 lines)
- Drain handler (~40 lines)
- `scope-widened` handler (~37 lines)

**What stays:**
- Simple partial application for the "no remaining NixOS args" case (~10 lines)
- Module `key` / `_file` / location tagging for NixOS module system compatibility (~20 lines)

## Concern 5b: Constraint System (Exclude, Substitute, Filter)

### Current (Den)

Three constraint types in `constraint.nix` (153 lines) + `constraints.nix` (51 lines):
- **Exclude** — blocks an aspect from resolving in a subtree
- **Substitute** — replaces one aspect with another at resolution time
- **Filter** — conditional inclusion based on predicate

### Proposed

**Exclude** — stream filter, as described above. Straightforward.

**Substitute** — stream `.map` that replaces matching elements:

```nix
substituteD = from: to: stream:
  stream.map (aspect:
    if identity.key aspect == identity.key from then to
    else aspect
  );
```

This is NOT a simple filter — it's an inline replacement. The stream model handles it naturally because `.map` can transform elements, not just remove them.

**Filter** (conditional) — stream `.filter` with the predicate applied to current context:

```nix
conditionalD = pred: stream:
  stream.filter (aspect: pred (aspectContext aspect));
```

**What stays:** The constraint *semantics* (3 types) and user-facing API (`meta.excludes`, `policy.exclude`). **What changes:** constraint *enforcement* moves from handler-based registry + scope-ancestry checks to stream operators applied at the point of aspect resolution.

## Concern 5c: Config Thunk Laziness Boundaries

### The Problem

Config thunks are pipe/quirk values that are functions of `{ config, ... }` — they read from an instantiated NixOS system's config. Cross-host collection means host A's pipe value might reference host B's `config`:

```nix
den.aspects.iceberg.ssh-keys = { config, ... }: [ "key-${config.networking.hostName}" ];
```

This creates: stream → `.toList` → `nixosSystem` → config → stream. The Cycle fixed-point handles this *only if* the config access is never forced during stream construction.

### Laziness Boundary Rules

**Safe operations** (during stream construction):
- Building stream descriptions (`More`/`Done`/`bind` trees)
- Referencing `config` as a thunk (captured in a closure, not forced)
- Passing config thunks through stream transformations

**Unsafe operations** (would create strict cycles):
- `builtins.length (stream.toList)` inside a stream body
- Pattern-matching on config values to decide stream structure
- Any operation that forces a thunk chain back to the stream being defined

### Proposed Handling

Config thunks flow through the stream as opaque values. Resolution happens at two points:

1. **Local thunks** — resolved inside `evalModules` when NixOS evaluates the module. The thunk captures `config` from the module fixpoint. No stream involvement.

2. **Cross-host thunks** — the Cycle fixed-point provides lazy references to other hosts' configs:

```nix
mainC = sources: {
  nixos = sources.perHost (ST.sel.flat "nixos");
  # hostConfigs is a lazy attrset: { igloo = nixosSystem {...}; iceberg = ...; }
  # Each host's config is a thunk — not forced until a thunk reads from it
};
```

The key invariant: `hostConfigs` is built from `sinks.nixos.toList` (per host), which is a lazy thunk. Cross-host config thunks reference this thunk. As long as no host's config thunk triggers evaluation of its *own* host's config during stream construction, the cycle resolves.

This is the same invariant Den's current `hostConfigs` mechanism relies on — it's not new, just made explicit.

## Concern 5d: Stream Ordering Invariants

Within each scope, stream elements must flow in a defined order to ensure correct resolution:

```
1. Policy dispatch (converge enrichment)
   ↓ enriched context
2. ctxD with enriched context
   ↓ parametric aspects can resolve
3. Aspect resolution (flatMap + distinctByIdentity)
   ↓ class modules
4. Class sink routing (nixos, homeManager, darwin, pipe keys)
   ↓ pipe data available
5. Pipe combinators (collect, expose, transform stages)
   ↓ pipe-dependent modules can resolve
6. Terminal: .toList → nixosSystem
```

This ordering replaces the implicit ordering in Den's handler dispatch chain. It is enforced by stream topology — each stage is downstream of its dependencies. The `ctxD` driver at stage 2 consumes the enriched context from stage 1, so parametric aspects that need enrichment keys (like `isNixos`) are guaranteed to see them.

## Concern 6: Aspect Dedup

### Current (Den)

7 dedup points: include-level (check-dedup), gate composite, ctx-seen, class collector, cross-scope Phase 1, route dedup, policy dispatch dedup. Plus provide dedup, expose effect dedup, subtree module dedup.

### Proposed

**2 essential dedup points survive:**

1. **Include-level dedup** — DAG linearization. When expanding an aspect's children into a stream, skip already-seen identities:

```nix
distinctByIdentity = identityFn: stream:
  # Track seen set, emit only first occurrence
  stream.filter (seen-tracking via stateful stream or fold)
```

This replaces `check-dedup`, `gate`, `ctx-seen`, and `class-collector` dedup. One operator instead of four handlers.

2. **NixOS module `key` field** — native to the module system. Already used as a safety net. No change needed.

**Dedup points that disappear:**

- Cross-scope module dedup (Phase 1) — per-scope streams don't produce cross-scope duplicates
- Subtree extraction dedup — no subtree extraction (no shared pipeline)
- Route dedup — no routes (forwards are stream concat)
- Provide/expose effect dedup — no shared accumulator state

**Policy dispatch dedup** stays as a domain concern: don't fire the same policy twice for the same entity. This is a `distinctBy(policy.name)` on the dispatch stream.

## Concern 7: Parametric Aspects

### Current (Den)

`compile-parametric` handler (72 lines) gates, binds (via bind handler), resolves function args from effect handlers, re-enters the pipeline with the result. Supports depth-bounded recursion (parametric returning parametric, max 10). `__fn`/`__args` wrapper type, `__scopeHandlers` for pre-binding.

### Proposed

A parametric aspect is a stream combinator: `Ctx → Aspect`.

```nix
# A parametric aspect is just a function in the stream
aspectS = st ({ host, ... }: { nixos.networking.hostName = host.name; });

# ctxD resolves the function's args from the handler stack
resolvedS = ctxD { inherit host; } aspectS;
```

The `fx.bind.fn` mechanism (resolve function args from effect handlers) is already what Ned's `ctxD` uses internally. The difference is that Den wraps this in `compile-parametric` → `bind` → `defer` → `drain` machinery. In the stream model, `ctxD` handles it directly.

**Depth-bounded recursion** (parametric returning parametric): becomes bounded stream unfolding. A stream element that produces another function gets re-fed through `ctxD`, with a depth counter on the stream transformation.

**`__scopeHandlers`** (pre-binding): becomes partial `ctxD` application. An aspect pre-bound with `host = "igloo"` is `ctxD { host = "igloo"; } aspectFn`.

**What disappears:**
- `compile-parametric` handler (~72 lines)
- `__fn`/`__args` wrapper type machinery
- `mkParametricBase` / `mkParametricNext` / `tagParametricResult`
- `prepareParametricFn`

**What stays:**
- `lib.functionArgs` introspection (used by `ctxD` to know which args to resolve)
- Depth bound (10 levels) as a recursion guard on stream unfolding

## Summary: What Stays, What Goes

### Eliminated (~2,500 lines)

| Component | Lines | Reason eliminated |
|-----------|-------|-------------------|
| Scope push/pop/widen handlers | ~150 | `ctxD` nesting |
| Deferred includes + drain | ~120 | Stream topology guarantees availability |
| Late dispatch | ~100 | Cycle fixed-point gives simultaneous visibility |
| 4-phase post-pipeline assembly | ~550 | No shared state to re-partition |
| `assemblePipes` | ~630 | Stream combinators + Cycle fixed-point |
| Route registration + application | ~300 | Stream concat between class sinks |
| Collision detection + enrichment stripping | ~100 | Single injection path |
| Bind handler complex path | ~90 | Stream topology |
| compile-parametric + wrappers | ~100 | `ctxD` handles directly |
| Cross-scope / artifact dedup | ~200 | Per-scope streams |
| Scope state fields (16 of 22) | scattered | No shared pipeline |

### Preserved (user-facing API)

- `den.aspects` — aspect declaration syntax unchanged
- `den.policies` — policy declaration syntax unchanged, `{ host, ... }:` dispatch contract unchanged
- `den.quirks` — pipe/quirk declaration unchanged
- `den.provides` — forward/provide API unchanged
- `den.schema` — entity kind definitions unchanged
- All policy effect constructors (`policy.resolve`, `policy.include`, `policy.exclude`, `policy.route`, `policy.provide`, `policy.instantiate`, `pipe.*`)

### Preserved (essential infrastructure)

- Adapter functor pattern (`den.fwd.${key}` submodule) — ~80 lines
- `forwardItem`/`forwardEach` user API — rewritten internals, same interface
- NixOS module `key`/`_file` tagging — ~20 lines
- Enrichment convergence algorithm — ~15 lines (rewritten as recursive function)
- `lib.functionArgs` introspection for dispatch/binding — ~15 lines
- Include-level dedup (`distinctByIdentity`) — ~20 lines
- Topology materialization (host list building, class inference) — ~60 lines

### New Infrastructure (from Ned)

- `ST` type + fluent API (`map`, `flatMap`, `filter`, `concat`, `sel`) — ~90 lines (from Ned, proven)
- `ctxD` / `scopeD` — ~40 lines (from Ned, proven)
- `run` (Cycle fixed-point) — ~5 lines (from Ned, proven)
- `hostsT` / `usersT` — ~60 lines (from Ned, proven)
- `hostUserD` (class routing driver) — ~15 lines (from Ned, proven)
- Class sink routing (splitting aspects into nixos/homeManager/darwin sinks) — ~30 lines (new)
- Policy dispatch driver — ~50 lines (new)
- Pipe combinator drivers — ~40 lines (new)

**Estimated new total: ~700-900 lines** of pipeline infrastructure, down from ~7,300 current pipeline code. The difference from Ned's ~277 lines accounts for Den's additional concerns: policy dispatch with enrichment convergence (~100 lines), pipe combinators with collect predicates and provenance (~120 lines), constraint handling (~50 lines), forward adapter machinery + integration (~110 lines), class sink routing with dedup (~50 lines), custom entity kind driver generation (~40 lines).

## Migration Strategy

This is not a rewrite-from-scratch. The migration path:

1. **Import Ned's proven primitives** (`ST`, `ctxD`, `scopeD`, `run`, `hostsT`, `usersT`) into Den's lib
2. **Build the Cycle structure** — define `mainC` with class sinks, wire topology drivers
3. **Port policy dispatch** — rewrite as dispatch driver using `converge` function
4. **Port pipe combinators** — map each `pipe.*` effect to a stream operation
5. **Port forwards** — rewrite `forwardItem` to produce stream operations
6. **Port aspect resolution** — the tree-walk becomes stream flatMap with `distinctByIdentity`
7. **Remove old infrastructure** — delete handlers, post-pipeline phases, scope machinery
8. **Validate** — all 713 existing tests must pass against the new implementation

Each step can be validated independently. The user-facing API never changes — only the internal resolution machinery.

## Open Questions

1. **Aspect merge semantics.** Den's `types.nix` has complex merge logic for aspects (454 lines). How much of this is essential vs artifact of the attrset representation? Stream elements don't merge — they concatenate. Need to verify all merge scenarios.

2. **Diagnostic/tracing integration.** Den's diag subsystem (~3,000 lines) traces the pipeline. Stream operations need equivalent tracing hooks for sequence diagrams and topology visualizations.

3. **Error messages.** Den's handler-based architecture produces detailed error context (which handler, which scope, which aspect). Stream errors need comparable diagnostics. Consider a `traceD` driver that wraps streams with provenance information.

4. **Performance.** The shared pipeline evaluates each aspect exactly once (dedup). Per-scope streams might re-evaluate shared static aspects in different scopes. Need to verify that Nix's thunk sharing prevents redundant evaluation, or add memoization.

5. **Incremental migration.** The handler-based trampoline and stream composition are mutually exclusive execution models. Incremental coexistence is likely not feasible — this should be planned as a flag-day switch with the 713 existing tests as the safety net.

6. **Schema-to-driver generation.** `den.schema` defines custom entity kinds (fleet, environment, etc.). Each needs a corresponding topology driver. The generation mechanism (who creates the driver, from what definition) needs design. Currently `resolve-schema-entity` handles all kinds uniformly — the stream model needs a driver factory or registration pattern.

7. **Provides subtree targeting.** `den.aspects.vim.provides.lsp = { ... }` creates named sub-aspects with targeting (`to-users`, `to-hosts`, specific name). The `provide.nix` handler (155 lines) manages this. Stream equivalent: provides become injections into specific class sinks, scoped by the target selector. The naming and targeting API needs a stream-native expression.

8. **Test strategy.** Of the 713 tests, how many validate observable behavior (final NixOS module sets) vs internal pipeline behavior (handler state, scope IDs)? Behavioral tests should pass unchanged. Internal tests need rewriting or deletion. Audit needed before migration begins.

9. **`types.nix` input layer.** The aspect type system (454 lines: `providerType`, `aspectContentType`, merge dispatch) is the input normalization layer — how `den.aspects.foo = { ... }` declarations become pipeline-ready structures. This layer is independent of the pipeline execution model and survives the migration unchanged. However, the parametric wrapper coercion (`mergeFunctions`, `__fn`/`__args`) may simplify since `ctxD` handles parametric resolution natively.

10. **`{ imports = [fn]; }` style modules.** When a module is an attrset with `imports` containing functions that take Den context args, the current `wrapDeferredImports` recursion handles this. In the stream model, `ctxD` must process these recursively or the imports must be pre-resolved before reaching `.toList`. Verify this case works.
