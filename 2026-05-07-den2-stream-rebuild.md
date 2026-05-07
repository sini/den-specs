# Den 2: Ground-Up Rebuild on Stream Primitives

Den's fx-pipeline has grown to ~7,300 lines of pipeline infrastructure to coordinate a shared trampoline walk. Ned demonstrates that the same semantics can be expressed in ~277 lines using stream composition and a Cycle.js fixed-point. Rather than migrating Den's internals (carrying architectural debt), this spec proposes rebuilding the resolution engine from scratch on Ned's proven foundation, targeting ~1,000 lines of core implementation with full feature parity.

The user-facing API is an opportunity, not a constraint. Since the fx-pipeline branch hasn't shipped, we can fix known ergonomic issues rather than preserve them.

## Goals

1. **Feature parity** with Den's fx-pipeline branch — all 8 test suites must be expressible
2. **~1,000 lines** of resolution engine (vs ~7,300 today)
3. **Fix API paper cuts** — eliminate ceremony that users shouldn't need
4. **Ned's primitives as foundation** — ST, ctxD, scopeD, run, topology drivers are proven code
5. **Same nix-effects kernel** — fx.rotate, fx.stream, fx.bind.fn stay as the effect substrate

## Non-Goals

- Rewriting the diag subsystem (orthogonal, can adapt later)
- Changing the NixOS module system integration (nixosSystem/darwinSystem stay)
- Reimplementing nix-effects

## Architecture

### Execution Model

```
den.aspects / den.policies / den.hosts (user declarations)
  ↓ NixOS module system evaluation
topology stream (static: which entities exist, how they nest)
  ↓ schema-driven driver composition
per-entity scoped streams (ctxD provides host, user, enrichment)
  ↓ policy dispatch + enrichment convergence
aspect resolution (flatMap + distinctByIdentity)
  ↓ tagged class emission
class sink streams (nixos, darwin, homeManager, custom, pipes)
  ↓ pipe combinators + forward concat
Cycle fixed-point (cross-entity visibility via Nix laziness)
  ↓ .toList at terminals
nixosSystem / darwinSystem / custom instantiate
```

### Core Primitives (from Ned, ~210 lines)

Imported directly from Ned's `ned.nix`:

- **`ST a`** — wrapped fx.stream with fluent API (map, flatMap, filter, concat, sel)
- **`st`** — constructor: `(st val1 val2 (s: s.map f))` chains values and combinators
- **`ST`** — curried combinator namespace for point-free composition
- **`ctxD`** — scoped DI via effect rotation: `ctxD { host = h; } compS`
- **`scopeD`** — raw handler-based scoping (ctxD is sugar over this)
- **`run`** — Cycle.js fixed-point: sources = drivers applied to sinks, sinks = cycle(sources)

### Resolution Engine Layers

Built on top of Ned's core:

```
Layer 1: Entity system        (~70 lines)  — mkEntityDriver, schema-to-driver, topology materialization
Layer 2: Policy dispatch      (~130 lines) — converge, dispatchD, effect classification, constraints
Layer 3: Aspect resolution    (~150 lines) — normalization, classifyKeys, identity, dedup, tree expansion
Layer 4: Pipes                (~70 lines)  — transform stages, collect, expose, targeted delivery
Layer 5: Forwards             (~100 lines) — adapter functor, forwardItem, forwardTo wiring
Layer 6: Class routing        (~30 lines)  — tagged emission, per-class filtering
Layer 7: Module integration   (~30 lines)  — key/file tagging, partial application
Layer 8: Options layer        (~200 lines) — den.aspects, den.policies, den.hosts, etc.
                              ─────────
                              ~990 lines
```

## User-Facing API Changes

### Aspects — unchanged core, eliminated ceremony

The core pattern stays:

```nix
den.aspects.igloo = {
  nixos.networking.hostName = "igloo";
  includes = [ den.aspects.shared-tools ];
};
```

**Change 1: `den.lib.parametric` eliminated.** Context propagation through includes is automatic. When an aspect has includes that need context (e.g., `{ host, ... }: ...`), the stream topology ensures context flows through without manual wrapping.

Before:
```nix
# Den 1: manual wrapping required
den.aspects.foo = den.lib.parametric {
  includes = [ den.provides.hostname ];
};
```

After:
```nix
# Den 2: just works
den.aspects.foo = {
  includes = [ den.provides.hostname ];
};
```

The `ctxD` scoping applies to the entire aspect resolution stream. Any function in the stream that needs `{ host, ... }` gets it from the effect handler stack — no explicit wrapping needed.

**Change 2: `excludes` as top-level key.** Instead of buried `meta.handleWith`:

Before:
```nix
den.aspects.igloo.meta.handleWith = den.lib.aspects.fx.constraints.exclude den.aspects.baz;
```

After:
```nix
den.aspects.igloo.excludes = [ den.aspects.baz ];
den.aspects.igloo.substitutes = [{ from = den.aspects.baz; to = den.aspects.qux; }];
```

**Change 3: Aspect-scoped policies auto-activate.** Policies declared under `den.aspects.X.policies.Y` are automatically included when the aspect is included — no manual `includes = [ den.aspects.X.policies.Y ]` needed. Dispatch gating still applies: a policy `{ host, user, ... }:` only fires when both `host` and `user` are in context. To opt out, set `autoActivate = false` on the policy.

Before:
```nix
den.aspects.alice.policies.to-igloo = { host, ... }: ...;
den.aspects.alice.includes = [ den.aspects.alice.policies.to-igloo ];  # manual activation
```

After:
```nix
den.aspects.alice.policies.to-igloo = { host, ... }: ...;
# auto-activated when den.aspects.alice is included
# dispatch-gated: only fires when host is in context

# Opt out:
den.aspects.alice.policies.library-only = { autoActivate = false; fn = { host, ... }: ...; };
```

### Policies — same contract, cleaner effects

The `{ host, ... }:` dispatch contract is unchanged — it's Den's best API decision:

```nix
den.policies.my-policy = { host, ... }: [ ... ];
```

**Change 4: Shorter effect constructors.** The `den.lib.policy` prefix is verbose. Since policies already return effect lists, the constructors can be imported more naturally:

Before:
```nix
den.policies.foo = { host, ... }:
  let inherit (den.lib.policy) pipe; in [
    (den.lib.policy.include someAspect)
    (den.lib.policy.provide { class = host.class; module = { ... }; })
    (pipe.from "firewall" [ (pipe.filter pred) ])
  ];
```

After:
```nix
den.policies.foo = { host, include, provide, pipe, ... }: [
  (include someAspect)
  (provide { class = host.class; module = { ... }; })
  (pipe.from "firewall" [ (pipe.filter pred) ])
];
```

Policy effect constructors (`include`, `provide`, `route`, `resolve`, `exclude`, `pipe`) are injected as context args alongside entity bindings. They're available from the same `{ host, ... }:` destructuring. This is natural — policies already receive their context this way.

Implementation: `ctxD` provides them as effect handlers alongside `host`, `user`, etc.

**Namespace reservation:** The names `include`, `provide`, `route`, `resolve`, `exclude`, `pipe` are reserved in policy context — users cannot have enrichment keys with these names. This is documented and enforced by the dispatch driver.

### Pipes — rename to channels, same semantics

**Change 5: `den.quirks` → `den.pipes`.** "Quirks" doesn't communicate the concept. "Pipes" is what they are — named data channels with transform stages. (This rename was already in progress per the `aa6677e7` commit.)

```nix
den.pipes.firewall = { description = "Firewall port declarations"; };
```

Production and consumption unchanged:
```nix
# Produce
den.aspects.nginx.firewall = { ports = [ 80 443 ]; };

# Consume
den.aspects.networking.nixos = { firewall, ... }: {
  networking.firewall.allowedTCPPorts = concatMap (f: f.ports or []) firewall;
};
```

### Forwards — `forwardTo` as primary API

**Change 6: `den.provides.forward` retired.** The low-level forward API (`den.provides.forward { each = ...; fromClass = ...; }`) is replaced by:

1. **`den.classes.X.forwardTo`** for static class routing (already exists):
```nix
den.classes.os.forwardTo = { class = "nixos"; };
den.classes.git.forwardTo = { class = "homeManager"; path = [ "programs" "git" ]; };
```

2. **`policy.route`** for dynamic routing (already exists):
```nix
den.policies.os-to-host = { host, route, ... }:
  [ (route { fromClass = "os"; intoClass = host.class; }) ];
```

The adapter functor pattern (`den.fwd.${key}` submodule) stays as internal machinery — users don't interact with it.

### Provides batteries — split the namespace

The `den.provides` namespace is currently overloaded: it contains both forward declarations (`den.provides.forward`) and aspect batteries (`den.provides.hostname`, `den.provides.to-hosts`, `den.provides.to-users`). With `den.provides.forward` retired, the remaining batteries are just aspects with special topology targeting. They stay as `den.provides.*` — their implementation changes from custom handler logic to standard aspect + policy patterns, but the user-facing API is preserved:

```nix
# These continue to work
den.default.includes = [ den.provides.hostname ];
den.aspects.shared.provides.to-hosts.nixos = { ... };
den.aspects.shared.provides.to-users = { user, ... }: { ... };
```

Internally, `provides.to-hosts` desugars to an aspect with a `{ host, ... }:` policy that routes content to the host class. `provides.to-users` desugars similarly for user scope. The `den._.*` internal batteries are unchanged — they produce aspects that feed into the pipeline like any other include.

### Hosts — no changes

**~~Change 7: Dropped.~~** The `users.tux = {}` pattern is 7 extra characters and avoids union type merge ambiguity in the NixOS module system. Not worth the complexity.

### Schema — split concerns

**Change 8: Separate entity options from entity includes.**

Before (overloaded):
```nix
den.schema.host = { host, ... }: {
  options.vpn-alias = lib.mkOption { default = host.name; };  # entity options
};
den.schema.host.includes = [ myPolicy ];  # entity includes — different concern, same key
```

After:
```nix
den.schema.host.options = { host, ... }: {
  options.vpn-alias = lib.mkOption { default = host.name; };
};
den.schema.host.includes = [ myPolicy ];
```

`den.schema.X.options` is a deferred module that extends the entity type. `den.schema.X.includes` is a list of aspects/policies included when the entity resolves. Clear separation.

## Resolution Engine Design

### Layer 1: Entity System (~70 lines)

```nix
# Generic entity driver factory
mkEntityDriver = kind: entitiesS: compS:
  entitiesS.flatMap (entity:
    ctxD { ${kind} = entity; } compS
  );

# Schema-driven driver chain construction
# Evaluates schema-level policies to discover nesting order:
#   flake → fleet → host → user
# Each level becomes a driver in the Cycle composition
buildDriverChain = schema: topology:
  let
    levels = discoverNestingOrder schema;  # from schema includes → resolve.to effects
    drivers = map (level:
      mkEntityDriver level.kind (entitiesFor level topology)
    ) levels;
  in
  composeDrivers drivers;  # nest: outerD (innerD compS)
```

Topology materialization (building host/user/fleet lists from declarations) reuses Ned's pattern — `builtins.foldl'` over the topology attrset, merging duplicates, inferring class from system.

### Layer 2: Policy Dispatch (~130 lines)

```nix
# Enrichment convergence — same algorithm as Den, ~15 lines
converge = policies: ctx: depth:
  let
    satisfiable = p: all (k: ctx ? ${k}) (requiredArgs p.fn);
    fired = filter satisfiable policies;
    effects = concatMap (p: p.fn ctx) fired;
    enrichment = extractEnrichment effects;
    newCtx = ctx // enrichment;
  in
  if enrichment == {} || depth >= 10 then { inherit effects ctx; }
  else converge (removeBy name fired policies) newCtx (depth + 1);

# Dispatch driver — runs convergence within each scope
dispatchD = policiesS: compS:
  compS.flatMap (ctx:
    let result = converge (policiesS.toList) ctx 0;
    in processEffects result.effects result.ctx
  );
```

**Late dispatch is absent by design.** The Cycle fixed-point makes all policies visible to all entities simultaneously. Policies are a shared source stream, not sequential registrations.

**Constraint operators:**

```nix
excludeD  = excluded: stream: stream.filter (a: !(excluded ? (identity a)));
substituteD = from: to: stream: stream.map (a: if identity a == identity from then to else a);
```

### Layer 3: Aspect Resolution (~150 lines)

The aspect tree walk becomes a stream `flatMap` with dedup:

```nix
# Expand an aspect's include tree into a flat stream
expandAspect = seen: aspect:
  let key = identity aspect;
  in if seen ? ${key} then st  # dedup: already expanded
  else
    st aspect  # emit self
    (expandChildren (seen // { ${key} = true; }) aspect);  # then children

expandChildren = seen: aspect:
  let children = aspect.includes or [];
  in builtins.foldl' (accS: child:
    accS (expandAspect seen (normalizeChild child))
  ) st children;
```

`normalizeChild` handles the heterogeneous includes list: named aspects, inline attrsets, functions, policies. Each becomes a normalized aspect record.

`classifyKeys` splits an aspect's top-level keys into class keys, pipe keys, and nested keys — same logic as today but without the handler dispatch (~35 lines).

### Layer 4: Pipes (~70 lines)

Each `pipe.*` operation maps directly to a stream combinator:

```nix
pipeOps = {
  filter = pred: dataS: dataS.filter pred;
  transform = f: dataS: dataS.map f;
  fold = f: init: dataS: st (builtins.foldl' f init dataS.toList);
  append = val: dataS: dataS (st val);
  for = f: dataS: st (f dataS.toList);

  collect = pred: dataS: ctx:
    # Sibling data comes from the Cycle source — all entities' pipe emissions
    # Filter by predicate, exclude self
    allPipeDataS.filter (e: pred e.ctx && e.ctx != ctx);

  withProvenance = dataS: ctx:
    dataS.map (v: { value = v; source = ctx; });

  to = targets: dataS:
    dataS.filter (e: elem e.aspectId targets);
};
```

`pipe.collect` leverages the Cycle fixed-point: all entities' pipe emissions are available as a shared source stream. The predicate filters by context shape, and self-exclusion is a simple identity check. No scope tree walk needed.

`pipe.expose` (child → parent) is implicit: user-scoped emissions are sub-streams within host-scoped streams. When the host stream materializes, user data is already there.

### Layer 5: Forwards (~100 lines)

**Simple forwards:** stream concat between class sinks. Zero machinery.

```nix
# In the Cycle body:
nixosS = nixosS.concat (osS);  # os class → nixos via forwardTo
```

**Adapter functor** (~60 lines): The `den.fwd.${key}` submodule pattern for guards, adaptArgs, and dynamic intoPath. This is NixOS module system scaffolding — irreducible:

```nix
mkAdapterFunctor = { fromClass, intoClass, path, guard ? null, adaptArgs ? null }: {
  ${intoClass} = args: {
    options.den.fwd.${adapterKey} = lib.mkOption {
      type = lib.types.submoduleWith {
        specialArgs = if adaptArgs != null then adaptArgs args else {};
        modules = sourceModules;
      };
    };
    config = let
      content = lib.setAttrByPath path args.config.den.fwd.${adapterKey};
    in if guard != null then guard args content else content;
  };
};
```

**`forwardTo` wiring** (~15 lines): At Cycle definition time, read `den.classes.*.forwardTo` and generate concat operations.

**`policy.route`** (~25 lines): Dynamic routing effects from policies produce the same concat operations, applied during effect processing.

### Layer 6: Class Routing (~30 lines)

Aspect resolution produces tagged stream elements. Routing is filtering:

```nix
# Emit tagged elements from resolved aspects
emitClasses = aspect:
  let keys = classifyKeys aspect;
  in st (map (k: { class = k; module = aspect.${k}; identity = identity aspect; }) keys.classKeys);

# Per-class sink in the Cycle
nixosS = allModulesS.filter (e: e.class == "nixos").map (e: e.module);
darwinS = allModulesS.filter (e: e.class == "darwin").map (e: e.module);
```

Dynamic — no upfront class registry needed for routing. New user-defined classes appear as new tag values.

### Layer 7: Module Integration (~30 lines)

**Single injection path via ctxD.** No collision detection, no enrichment stripping, no dual injection.

Modules that take Den context args (`{ host, user, ... }`) have those args resolved by the effect handler stack before reaching NixOS. The module's NixOS-facing signature only contains NixOS args (`{ config, pkgs, lib, ... }`).

```nix
# Simple case: partial application
wrapModule = ctx: module:
  if !isFunction module then module
  else
    let denArgs = intersectAttrs (functionArgs module) ctx;
        remaining = removeAttrs (functionArgs module) (attrNames denArgs);
    in if remaining == {} then module denArgs  # fully satisfied
    else lib.setFunctionArgs (args: module (denArgs // args)) remaining;
```

Module `key` and `_file` tagging for NixOS module system dedup (~10 lines). This is the only NixOS compatibility layer needed.

### Layer 8: Options Layer (~200 lines)

NixOS module options for user-facing API. Largely preserved from Den with the changes noted above:

- `den.aspects` — `lazyAttrsOf aspectType` (unchanged core, type simplifies)
- `den.policies` — `lazyAttrsOf policyType` (unchanged)
- `den.hosts` — `attrsOf (attrsOf hostType)` (unchanged)
- `den.pipes` — `lazyAttrsOf pipeType` (renamed from quirks)
- `den.classes` — `lazyAttrsOf classType` (unchanged)
- `den.schema` — split into `schema.X.options` and `schema.X.includes`
- `den.default` — `aspectType` (unchanged)

### The Cycle — Putting It All Together

#### What actually creates the cycle?

Not everything needs to be in the Cycle fixed-point. Most resolution is a straight pipeline (chain, not cycle). The genuine circular dependencies are:

1. **pipe.collect** — host A's pipe data includes host B's emissions, and vice versa
2. **Config thunks** — a pipe value references `config` from an instantiated system, which is built from the pipe-enriched module list

Everything else (aspect resolution, policy dispatch, class routing) is a linear chain that happens to fan out over topology.

#### Structure: chain + targeted fixed-point

```nix
resolve = { aspects, policies, hosts, pipes, classes, schema, default }:
  let
    topoS = st (buildTopology hosts);

    # --- Static policy set ---
    # Policies are collected from:
    #   1. den.policies (top-level)
    #   2. den.aspects.*.policies.* (aspect-scoped, auto-activated)
    #   3. den.schema.*.includes (schema-level)
    # All are known at definition time. No dynamic policy registration.
    allPolicies = collectAllPolicies { inherit policies aspects schema; };

    # --- Linear chain: declarations → resolved modules ---
    # This is NOT a cycle — it's a straight pipeline with topology fan-out

    # 1. Expand aspect include trees
    aspectsS = expandAspects (attrValues aspects) default;

    # 2. Schema-driven topology fan-out (flake → fleet → host → user)
    #    Each level applies ctxD with entity bindings + policy dispatch
    perEntityS = applyTopology schema topoS allPolicies aspectsS;

    # 3. Classify and emit tagged class/pipe elements
    taggedS = perEntityS (emitAllClasses pipes);

    # 4. Route class sinks + apply forwardTo
    classSinks = routeClasses classes taggedS;

    # --- Cycle: pipe data cross-references ---
    # This is where the fixed-point lives. Pipe data from all entities
    # must be visible to all entities for pipe.collect.
    pipeDataCycle = sources: {
      pipeData = sources.enriched (collectPipeEntries pipes);
      enriched = classSinks.perEntity (ctxD { pipeData = sources.pipeData; });
    };

    enrichedSinks = run {
      pipeData = identity;  # pipe data source = pipe data sink (self-referential)
      enriched = identity;
    } pipeDataCycle;

    # 5. Terminal: .toList → nixosSystem / darwinSystem
    finalSinks = applyForwards classes enrichedSinks;

  in finalSinks;
```

The Cycle is scoped to exactly the circular part: pipe data cross-referencing. Everything else is a linear transformation chain, which is clearer and avoids accidental strict cycles.

#### Policy bootstrapping invariant

**Policies are never dynamically registered.** The policy set is fully determined at definition time from three static sources:

1. `den.policies.*` — top-level policy declarations
2. `den.aspects.*.policies.*` — aspect-scoped policies (auto-activated, collected during aspect normalization)
3. `den.schema.*.includes` — schema-level policies

`policy.include` can include aspects that contain `policies.*` entries, but those policies are already in `allPolicies` because they were collected from the aspect registry at definition time — before resolution begins. This is different from Den's current model where policies are registered during the tree walk.

If a `policy.include` brings in an *anonymous* aspect with inline policies not in the aspect registry, those policies are NOT auto-activated. They must be explicitly included via the aspect's `includes` list. This is the trade-off for eliminating late dispatch.

#### Config thunk laziness boundary

Config thunks (pipe values that reference `{ config, ... }`) work through the same Cycle mechanism. The `config` comes from `nixosSystem { modules = finalSinks.nixos.toList; }`, which is lazy. The thunk captures a reference to `config` without forcing it during stream construction. It is only forced when the NixOS module system evaluates the final config.

**Invariant:** No stream construction code may call `.toList` or force any value that transitively depends on `config`. This is enforced by convention (same as current Den) and can be validated by a `thunkGuardD` driver that wraps config-dependent values with `builtins.tryEval` to detect strict cycles early.

## What Dies

| Component | Current lines | Why |
|-----------|--------------|-----|
| Pipeline trampoline + 35 handlers | ~2,500 | Stream composition |
| 22 state fields | scattered | No shared mutable state |
| 4-phase post-pipeline assembly | ~550 | No shared results to re-partition |
| assemblePipes | ~630 | Stream combinators + Cycle fixed-point |
| Scope push/pop/widen/defer/drain | ~400 | ctxD nesting |
| Late dispatch | ~100 | Cycle fixed-point visibility |
| wrapClassModule collision system | ~250 | Single injection path |
| Route registration + application | ~300 | Stream concat |
| compile-parametric + bind machinery | ~200 | ctxD handles directly |
| den.lib.parametric wrappers | ~100 | Automatic context propagation |
| den.provides.forward API | ~140 | forwardTo + policy.route |
| types.nix complex merge | ~300 | Stream elements concatenate, not merge |
| **Total eliminated** | **~5,470** | |

## What Stays (Rewritten)

| Component | Lines | Notes |
|-----------|-------|-------|
| Ned core (ST, ctxD, run, etc.) | ~277 | Imported from Ned as-is |
| Entity system | ~80 | mkEntityDriver, schema-to-driver, topology materialization |
| Policy dispatch | ~160 | converge, dispatchD, effect classification, constraints |
| Aspect resolution | ~200 | normalization, classifyKeys, identity, dedup, tree expansion |
| Pipes | ~130 | transform stages, collect with predicates, provenance, config thunks |
| Forwards | ~110 | adapter functor, forwardItem, forwardTo wiring |
| Class routing | ~30 | tagged emission, per-class filtering |
| Module integration | ~70 | wrapModule, key/file tagging, attrset-with-imports handling |
| Options layer | ~280 | types, option declarations, aspect type normalization |
| Identity + key classification | ~55 | preserved from Den |
| **Total** | **~1,392** | ~5x reduction from ~7,300 current pipeline |

Target ~1,400 lines. This is realistic, not aspirational — it accounts for edge cases in module integration, pipe collect predicates, and type normalization that the ~1,000 estimate glossed over.

## Test Strategy

Den has 713 tests across 8 suites. These test **observable behavior** — given declarations X, the resulting NixOS config should contain Y.

### Porting approach

1. **Behavioral tests port directly.** Tests that check `cfg.config.users.users.tux.description == "igloo/tux"` don't care about internals. The declarations go in, the config comes out. These should pass against the new engine without modification.

2. **API-changed tests need updating.** Tests using `den.lib.parametric`, `den.provides.forward`, or `meta.handleWith` need rewriting to use the new API. The assertions stay the same — only the declarations change.

3. **Internal tests get deleted.** Any test that checks handler state, scope IDs, or pipeline internals has no equivalent. These are few — most tests are behavioral.

### Porting sequence

```
Phase 1: Port the simplest suite first (static aspects, basic topology)
         Validates: ST, ctxD, hostsT, usersT, class routing, module wrapping
Phase 2: Port policy suite
         Validates: dispatchD, converge, enrichment, constraints
Phase 3: Port pipe/quirk suite
         Validates: pipe combinators, collect, expose, targeted delivery
Phase 4: Port forward suite
         Validates: adapter functor, forwardTo, policy.route
Phase 5: Port system integration suite
         Validates: full Cycle, nixosSystem/darwinSystem, cross-host data
Phase 6: Port remaining suites (parametric, custom entities, edge cases)
Phase 7: Port diag integration (deferred — separate effort)
```

Each phase validates a layer of the engine. Failures in phase N indicate bugs in layer N.

## Open Questions

1. **Ned integration.** Does Den 2 import Ned as an input, or vendor the ~210 lines? Importing creates a dependency; vendoring means maintaining a copy. Recommendation: import as input, since both repos share the nix-effects kernel.

2. **Namespace/battery system.** Den has namespaces (`inputs.den.namespace "ns" sources`), batteries (`den.provides.hostname`, `den._.*`), and angle-bracket syntax. These are orthogonal to the resolution engine — they produce aspects that feed into the pipeline. They should port with minimal changes, but the `den.provides` namespace is overloaded (batteries vs forward declarations). Splitting these would reduce confusion.

3. **Aspect type simplification.** Den's `types.nix` (454 lines) has complex merge logic because aspects are NixOS module system types that need to merge across multiple declaration sites. With streams, aspect content concatenates rather than merges. How much of the merge logic can be dropped vs is required by the module system's `mkMerge`/`mkOverride` semantics?

4. **Error diagnostics.** Den's handler chain produces rich error context. Stream errors need comparable diagnostics. A `traceD` driver that annotates stream elements with provenance (aspect name, scope context, file location) would support error messages like "while resolving aspect 'igloo' at host scope 'x86_64-linux/igloo'".

5. **Performance baseline.** Need to benchmark Den's current evaluation time on a representative config (e.g., 5 hosts, 20 aspects, 10 policies) before rebuilding, so we can compare. Nix thunk sharing should prevent redundant evaluation in per-scope streams, but this needs verification.

6. **Migration path for existing Den users.** If anyone is using the fx-pipeline branch, they need a migration guide. The API changes (parametric elimination, excludes syntax, policy effect injection) are the breaking surface. A compat shim module could bridge the gap during transition.

7. **`{ imports = [fn]; }` modules.** When a NixOS module is an attrset with `imports` containing functions that take Den context args, `ctxD` must resolve these recursively. Ned's `ctxD` already handles functions in streams via `fx.bind.fn` — verify this covers the nested imports case.

8. **Strict mode.** `den.lib.strict` prevents undeclared keys on entities. This is a validation concern, not a resolution concern. It should work with a stream model (validate the aspect attrset before emitting into the stream), but needs explicit design.
