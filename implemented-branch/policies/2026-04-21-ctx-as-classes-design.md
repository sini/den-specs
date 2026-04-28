# Separating Data, Policies, and Behavior

**Date:** 2026-04-21 (revised with Vic's feedback)
**Branch:** feat/rm-legacy
**Status:** Draft — pending peer review

## Terminology

Before anything else, align on terms:

| Term | Den concept | Examples |
|------|-------------|----------|
| **Entity** | A tangible thing in the user's infrastructure | A host, a user, a home |
| **Schema** | The data structure defining what an entity IS | `den.schema.host` + `den.schema.base` + hostSubmodule from types.nix |
| **Class** (Nix config class) | A category of behavior/configuration | `nixos`, `darwin`, `homeManager`, custom classes |
| **`den.ctx`** | Deprecated — conflated policies and behavior. Being fully removed by this spec | `den.ctx.host`, `den.ctx.hm-host`, `den.ctx.default` |
| **Aspect** | A container of behavior (Nix config classes) | `den.aspects.gaming` — defines nixos/darwin/homeManager configs |
| **Policy** | How entities transition into or relate to each other | host fans out into users, host transitions into hm-host |

**"Class" is reserved for Nix configuration classes (behavior).** Not entity types, not data definitions.

## Problem

`den.ctx` conflates three concerns that should be separate:

1. **Policies** — `into.*` transitions define how to move between context stages (`{host}` → `{host, user}`)
2. **Behavior** — each ctx node IS an aspect, so `den.ctx.hm-host.nixos.foo = "bar"` and `den.ctx.default.includes = [z]` work

This conflation exists because the original design needed context-passing between aspect-providers (functions producing collections of Nix classes). `den.ctx` was the mechanism: an aspect augmented with `into.*` transitions. The name "ctx" itself leaks the implementation detail of context-passing from the legacy core.

**What `den.ctx` is NOT:** It is not about entity data types. Entity schemas are defined by schema mixins:
- Host = `den.schema.host` + `den.schema.base` + hostSubmodule (types.nix)
- User = `den.schema.user` + `den.schema.base` + userType (types.nix)
- Home = `den.schema.home` + `den.schema.base` + homeType (types.nix)

The ~24 ctx nodes are not entity type definitions — they are pipeline stages with behavior attached. This is why `hm-host`, `flake-system`, `default` etc. exist as ctx nodes despite not being real-world entities.

### Why this is a problem

Because `den.ctx` is both policies AND behavior:
- Adding a transition requires creating a full aspect node (even if you only need the `into`)
- Behavior scoped to a policy stage (e.g., "only apply this nixos config when home-manager is enabled") requires an intermediate ctx node
- Users struggle to understand what `den.ctx` is for — the name describes an implementation detail, not a declarative concept
- The 3-node `makeHomeEnv` chains (`host.into.X-host` → `X-host.into.X-user` → `X-user.provides`) exist solely to thread context through intermediate aspect nodes

## Design: Four Clean Separations

### The Four Concerns

```
Data (Schema)        — what entities ARE       — den.schema.* + entity types
Policies        — how entities RELATE     — den.policies (new)
Stages               — where behavior BINDS    — den.stages (new, replaces den.ctx scoped behavior)
Behavior             — how entities RESOLVE    — den.aspects.* + fx pipeline
```

`den.ctx` is fully removed. Its two roles split into three:
- `into.*` transitions → `den.policies`
- Scoped behavior (`.nixos`, `.includes` on ctx nodes) → `den.stages`
- Reusable behavior definitions → `den.aspects` (unchanged)

### Data: Entity Schemas

Entity schemas define what entities ARE. Currently scattered across `nix/lib/types.nix` (entity submodule types) and `modules/options.nix` (schema entry wiring).

Each entity kind is defined by schema mixins:
```nix
# A host entity is shaped by these three "mixins":
den.schema.host    # user-extensible schema (deferred module)
den.schema.base    # shared across all entity kinds (hasAspect, etc.)
hostSubmodule      # structural options: name, system, class, aspect, users, instantiate, intoAttr
```

Entities are local to a flake — which hosts, users, and homes exist is defined by the consumer. Schemas can be shared via namespaces, but entity instances are always local.

**No change to the schema system itself.** The reorganization consolidates scattered type definitions into entity-per-file structure (see File Reorganization below).

### Policies: Entity Transitions

Policies declare how entities transition into each other. This is what `into.*` currently does, extracted into its own system.

**Before (policy buried in ctx aspect):**
```nix
den.ctx.host.into.user = { host }:
  map (user: { inherit host user; }) (lib.attrValues host.users);
```

**After (policy as first-class declaration):**
```nix
den.policies.host-to-users = {
  from = "host";
  to = "user";
  resolve = { host }:
    map (user: { inherit host user; }) (lib.attrValues host.users);
};
```

Policies must support nesting for namespace reuse. Den is not only about configuring local infrastructure — the denful project will share reusable policies + behavior via namespaces, organized as a nested tree (like nixpkgs has nested package-sets). `den.policies` should support the same nested structure that aspects do.

```nix
# Namespace-provided policies (denful example)
den.ns.desktop.policies.host-to-display-server = { ... };
den.ns.kubernetes.policies.cluster-to-nodes = { ... };
```

### Behavior: Aspects and Scoped Activation

Behavior is defined as aspects — containers of Nix configuration classes. This is already clean in Den. `den.aspects.gaming` defines the behavior of the gaming feature.

The key question is: **what happens to behavior that was scoped to policy stages?**

Currently, `den.ctx.hm-host` is an aspect node that guarantees home-manager is enabled on the host. Users write:
```nix
den.ctx.hm-host.nixos.foo = "bar";        # only applies where HM is enabled
den.ctx.hm-host.includes = [ someAspect ]; # includes scoped to HM hosts
den.ctx.default.nixos.x = "y";            # applies to all entities
den.ctx.default.includes = [ z ];          # includes for everything
```

After removing `den.ctx`, this scoped behavior needs a new home. Three options were considered:

**Option A: Behavior on policy declarations** (`policy.aspect = ...`)
Rejected — couples topology to behavior. The policy fixes which behavior runs when it fires, re-creating the conflation problem that `den.ctx` had.

**Option B: Behavior declares its own scope** (`aspect.meta.stage = "hm-host"`)
Rejected — couples behavior to topology in the other direction. The aspect knows which stage it belongs to, which is not its concern.

**Option C (rejected form): Pipeline-matched stage names**
Both A and B move the coupling rather than eliminating it. Neither the policy nor the aspect should know about the other.

### Accepted: `den.stages` — Named Scopes

**`den.stages`** is a new namespace: named scopes where behavior can be attached, independent of both the policies that create them and the aspects that run in them. This is the cleanest migration path from `den.ctx` — it's literally what `den.ctx.hm-host.includes = [...]` was doing, minus the transition topology.

```nix
# Policy — pure topology (no behavior reference)
den.policies.host-to-hm-host = {
  from = "host";
  to = "hm-host";
  resolve = detectHost { ... };
};

# Stage — named scope for attaching behavior
den.stages.hm-host.nixos.foo = "bar";
den.stages.hm-host.includes = [ someAspect ];

# Aspect — pure behavior (no stage awareness)
den.aspects.my-hm-config = { host, ... }: {
  nixos.services.something.enable = true;
};
```

**Separation preserved:**
- **Policy** declares "host transitions to hm-host" — pure topology
- **Stage** declares "when at hm-host, include this behavior" — the binding point
- **Aspect** declares "here is configuration" — pure behavior

The stage is the **binding point** that connects topology and behavior without either knowing about the other. Like Kubernetes labels + selectors: neither the pod nor the service owns the binding.

**Migration from `den.ctx`:**
```nix
# Before:
den.ctx.hm-host.nixos.foo = "bar";
den.ctx.hm-host.includes = [ someAspect ];

# After:
den.stages.hm-host.nixos.foo = "bar";
den.stages.hm-host.includes = [ someAspect ];
```

Mechanical rename: `den.ctx.X` → `den.stages.X` for scoped behavior. The `into.*` part goes to `den.policies` separately.

**Pipeline activation:** When a policy resolves and the pipeline enters stage `hm-host`, the pipeline looks up `den.stages.hm-host` and includes its behavior in the resolution scope for that transition. The stage is an aspect-shaped attrset (same structure as `den.aspects.*`) — it has class keys, `includes`, etc. — but it lives in a separate namespace to make its role clear: stages are scoped behavior bindings, aspects are reusable behavior definitions.

**Stages support nesting** for namespace reuse (denful batteries):
```nix
den.ns.desktop.stages.hm-host.includes = [ ... ];
```

### The `default` Stage

`default` is a ground context stage. Currently every entity kind transitions into it (`host.into.default`, `user.into.default`, `home.into.default`).

**No automatic transitions to default.** (Vic's feedback) The legacy automatic `*-to-default` transitions caused duplicate aspect resolution and re-firing — the exact problems that led to documenting "attach to narrower stages like `hm-host` instead of `default`" as best practice. Under the new model, transitions to `default` are opt-in: users who want the pattern declare an explicit policy.

```nix
# Opt-in, not automatic:
den.policies.host-to-default = {
  from = "host";
  to = "default";
  resolve = lib.singleton;
};
```

`default` behavior lives in `den.stages.default` (migrated from `den.ctx.default` / `den.default`). Den core may still ship default-to-* policies as batteries, but they are not structural/automatic ��� users enable them like any other policy.

### Flake Output Stages

`flake`, `flake-system`, `flake-os`, `flake-packages` etc. are not entities in the real-world sense. They are context shapes — attrsets representing arguments (`{system}`, `{host}`) that drive output generation.

Under the new model, these become policies in the output pipeline:
```nix
den.policies.flake-to-systems = {
  from = "flake";
  to = "flake-system";
  resolve = _: map (system: { inherit system; }) den.systems;
};
den.policies.flake-system-to-os = {
  from = "flake-system";
  to = "flake-os";
  resolve = { system }: map (host: { inherit host; }) (builtins.attrValues den.hosts.${system});
};
```

Their behavior (the output adapters that produce `nixosConfigurations`, `homeConfigurations`, etc.) moves to `den.aspects` with policy-scoped activation.

### The makeHomeEnv Chain (host → hm-host → hm-user)

The `makeHomeEnv` factory currently generates a 3-node ctx chain for each home environment (home-manager, hjem, maid):

```
host.into.hm-host → hm-host.into.hm-user → hm-user.provides.hm-user
```

Under the new model, this becomes policy declarations + aspects:

```nix
# Policies — pure topology (what makeHomeEnv generates):
den.policies.host-to-hm-host = {
  from = "host";
  to = "hm-host";
  resolve = detectHost { className = "homeManager"; ... };
};
den.policies.hm-host-to-hm-user = {
  from = "hm-host";
  to = "hm-user";
  resolve = intoClassUsers "homeManager";
};

# Stages — behavior scoped to policy stages:
den.stages.hm-host.includes = [
  ({ host }: { ${host.class}.imports = [ host.homeManager.module ]; })
];
den.stages.hm-user.includes = [ (forwardToHost { ... }) ];
```

The `makeHomeEnv` factory continues to exist but generates policy + aspect pairs instead of ctx nodes. Its interface stays the same — callers pass `{ className, optionPath, getModule, forwardPathFn }` and get declarations back.

### Scope Binding (What ctxApply Becomes)

`ctxApply` currently stamps `__ctx` and `__scopeHandlers` on an aspect so parametric functions receive entity values as args. This mechanism survives but is no longer tied to `den.ctx`.

When the policy pipeline transitions from one stage to another, it creates scope handlers from the context dict (`{host}`, `{host, user}`, etc.) and stamps them on the target aspect. This is a pipeline concern, not a class/entity concern. The function moves from `ctx-apply.nix` into the policy handler.

## File Reorganization

### Current Layout (scattered)

```
nix/lib/types.nix                        # hostType + userType + homeType (290 lines, all mixed)
nix/lib/ctx-types.nix                    # ctxTreeType, ctxSubmodule, intoCtxType
nix/lib/ctx-apply.nix                    # ctxApply functor
nix/nixModule/ctx.nix                    # den.ctx option declaration
modules/context/host.nix                 # den.ctx.host (into, provides)
modules/context/user.nix                 # den.ctx.user (into, provides)
modules/context/has-aspect.nix           # hasAspect query API
modules/context/perHost-perUser.nix      # deprecated guards
modules/options.nix                      # schemaEntryType, den.hosts/homes/schema
modules/aspects/provides/home-manager.nix # den.ctx.home buried here
```

### Proposed Layout

**Entity schemas (entity-per-file):**
```
nix/lib/entities/
  _types.nix          # Shared: strOpt, systemType, homeSystemType, schemaEntryType
  _has-aspect.nix     # hasAspect entity query module (from modules/context/has-aspect.nix)
  host.nix            # hostSubmodule (entity shape options)
  user.nix            # userSubmodule (entity shape options)
  home.nix            # homeSubmodule (entity shape options, extracted from types.nix)
  home-env.nix        # makeHomeEnv factory
```

**Policies (new):**
```
nix/lib/policies/
  types.nix           # policyType (nested-capable for namespace reuse)
  handler.nix         # policy handler for fx pipeline (replaces transitionHandler)

modules/policies/
  host.nix            # host-to-users, host-to-hm-host, etc.
  user.nix            # user-to-* policies
  home.nix            # home-to-* policies
  flake.nix           # flake output pipeline policies
```

**Stages (new, replaces den.ctx scoped behavior):**
```
nix/lib/stages/
  types.nix           # stageType (aspect-shaped, nested-capable for namespace reuse)

modules/stages/
  default.nix         # den.stages.default (from den.ctx.default)
  hm.nix              # den.stages.hm-host, den.stages.hm-user (from makeHomeEnv)
  flake.nix           # den.stages.flake-*, if needed for output behavior
```

**Deletions:**
- `den.ctx` option — fully removed
- `nix/lib/ctx-types.nix` — `ctxSubmodule` and `intoCtxType` no longer needed
- `nix/lib/ctx-apply.nix` — scope binding moves into policy handler
- `nix/nixModule/ctx.nix` — replaced by `den.policies` option
- `modules/context/host.nix` — `into`/`provides` move to `modules/policies/host.nix`
- `modules/context/user.nix` — same
- `modules/context/perHost-perUser.nix` — deprecated, removed
- `nix/lib/types.nix` — split into per-entity files

### Migration Path

**Phase 1 (this branch):** Consolidate entity types into `nix/lib/entities/` (rename + split, no behavioral changes). Keep `den.ctx` working as-is during this phase. The `nix/lib/types.nix` split must provide re-exports from the original path so existing imports (`modules/options.nix`, `modules/aspects/definition.nix`, and any flake-level consumers) continue to work. Verify by grepping for all `types.nix` imports before splitting.

**Phase 2 (policies + stages):** Introduce `den.policies` and `den.stages`. Extract `into.*` from ctx nodes into policy declarations. Scoped behavior on ctx nodes (`.nixos`, `.includes`) moves to `den.stages`.

**Phase 3 (ctx removal):** Remove `den.ctx` entirely. Update all consumers (templates, batteries, tests).

**Downstream migration surface:** Templates and batteries referencing intermediate ctx nodes (`den.ctx.hm-host`, `den.ctx.flake-packages`, etc.) need updating in Phase 3. This includes `templates/ci/modules/features/` test fixtures and `templates/flake-parts-modules/` which builds custom `into` chains. Battery modules contributing `includes` to ctx nodes (`den.ctx.user.includes`, `den.ctx.default.includes`) move to contributing to the equivalent `den.stages` entries.

## Relation to Other Specs

- **Policies** (`2026-04-20-policy-design.md`): Defines the `den.policies` system in detail. This spec provides the motivation and the migration path for getting there. Updated to align with ctx removal — inline sugar removed, activation model level 3 moved from `den.ctx.<kind>` to `den.schema.<kind>` (entity type declares its policy participation, consistent with Haskell typeclass model).
- **Capabilities Design**: Aspects contain capability keys (Nix config classes). Structural detection determines which classes an aspect emits. Orthogonal to this spec — capabilities are about behavior, not data or policies.
- **provide-to Effects**: Cross-entity routing currently in `provides` moves to policies, which emit `provide-to` effects.

## Prior Art and Design Rationale

The separation of entity structure, policies, and resolution behavior is a universal pattern in mature systems. Den adds a fourth concern (stages as binding points) to avoid coupling topology to behavior. See companion document `2026-04-21-ctx-as-classes-prior-art.md` for detailed analysis.

### Key insight

Den separates four concerns:
1. **What an entity IS** (structure/schema) — `den.schema.*` + entity type definitions
2. **How entities RELATE** (transitions) — `den.policies`
3. **Where behavior BINDS** (scoped activation) — `den.stages`
4. **How entities RESOLVE** (behavior) — `den.aspects` + fx pipeline handlers

`den.ctx` conflated concerns 2, 3, and 4 into a single construct. This design separates them. Stages are the key insight: they are the binding point between topology and behavior that neither policies nor aspects should own.

## Resolved Questions (from Vic's feedback)

1. **`default` is not an entity type.** It is a ground stage. Transitions to `default` are **not automatic** — the legacy auto-transitions caused duplicate resolution and re-firing. Users opt in via explicit policies. Behavior lives in `den.stages.default`.

2. **Flake output stages are not entities.** They are context shapes (argument attrsets) for output generation. They become policies in the output pipeline.

3. **Namespaces share policies.** Just as aspects (behavior) are shared via namespaces, policies should be shareable too. `den.policies` must support nesting for the denful project's reusable batteries.

4. **`ctxTreeType` nesting is needed** — for organizational purposes in large namespace trees (denful). Policies inherit this nested structure.

5. **Battery `includes` on intermediate nodes** (e.g., `den.ctx.hm-host.includes`): These are behavior scoped to a policy stage. After migration, the behavior moves to `den.stages.hm-host` — a named scope where behavior is attached independently of both the policy that creates the stage and the aspects that run in it.

6. **`den.ctx` is fully removed.** It is not renamed, not kept for backwards compat. The `into` part becomes `den.policies`. The scoped behavior part becomes `den.stages`. Reusable behavior stays in `den.aspects`. The name "ctx" leaks implementation details of the old context-passing mechanism.

## Open Questions

1. **Scoped behavior activation mechanism**: Resolved — `den.stages` (see Behavior section). Stages are named scopes where behavior is attached, independent of both policies and aspects. Neither policy nor aspect carries a reference to the other. The pipeline looks up `den.stages.${target}` when entering a stage. Needs prototyping to confirm: eager lookup table at pipeline entry vs lazy per-transition query, and whether stages should support the full aspect submodule type or a subset.

2. **Schema entry auto-resolution**: Currently `options.nix` checks `den.ctx ? ${kind}` to gate entity participation. With `den.ctx` removed, what gates this? Likely answer: check whether any policy has `from = kind` or `to = kind` — if an entity kind participates in any policy, it participates in resolution. Alternatively, `den.schema ? ${kind}` may suffice since schema existence already implies the entity kind is registered. Needs confirmation during Phase 2.

3. **Provides self-identity**: Currently `den.ctx.host.provides.host = {host}: host.aspect`. This is universal — every entity's aspect IS its identity. With ctx removed, does the pipeline infer this automatically, or does each policy declare it?

## Expected User Impact

### What breaks

Any flake using `den.ctx` directly will need migration. The main patterns that break:

**`den.ctx.*.nixos/darwin/homeManager` (behavior on ctx nodes):**
```nix
# Before:
den.ctx.hm-host.nixos.foo = "bar";
den.ctx.default.includes = [ myAspect ];

# After:
den.stages.hm-host.nixos.foo = "bar";
den.stages.default.includes = [ myAspect ];
```
Mechanical rename — `den.ctx.X` becomes `den.stages.X` for scoped behavior.

**`den.ctx.*.into` (policy declarations):**
```nix
# Before:
den.ctx.host.into.my-stage = { host }: [ ... ];

# After:
den.policies.host-to-my-stage = {
  from = "host";
  to = "my-stage";
  resolve = { host }: [ ... ];
};
```
Shape change — `into` functions become policy declarations with `from`/`to`/`resolve`.

**`den.ctx.*.provides` (cross-entity forwarding):**
```nix
# Before:
den.ctx.my-stage.provides.my-stage = { host, user }: ...;

# After:
# Self-identity is implicit. Cross-entity provides move to policies.
```

### What doesn't break

- **`den.schema.*`** — unchanged. Schema definitions, entity options, and the mixin system are not affected.
- **`den.hosts` / `den.homes`** — unchanged. Entity declarations stay the same.
- **`den.aspects.*`** — unchanged. Aspect definitions (behavior) work exactly as before.
- **Parametric functions** (`{ host, user }: { nixos = ...; }`) — unchanged. Handler-based resolution continues to work; scope binding just moves from `ctxApply` into the policy handler.
- **`hasAspect`** — unchanged. Entity query API is orthogonal to ctx removal.

### What gets simpler

- **No more `den.ctx` to explain.** Users define entities (data), aspects (behavior), policies (transitions), and stages (scoped behavior bindings) — four concepts with clear purposes instead of one overloaded construct.
- **No intermediate node proliferation.** Users don't need to create `den.ctx.my-custom-stage` nodes with `into` + aspect behavior just to scope configuration to a transition. Policies and stages are declared separately.
- **Namespace batteries become clearer.** Shared batteries (denful) export `policies` + `stages` + `aspects` as separate concerns. Consumers only need to provide their own entities (data). The division of what's reusable vs local is explicit.

### Migration effort by user type

| User type | Impact | Effort |
|-----------|--------|--------|
| **Basic** (only `den.hosts`/`den.homes` + aspects) | None — `den.ctx` was never touched directly | Zero |
| **Intermediate** (uses `den.ctx.default.includes` or `den.ctx.hm-host.nixos`) | Mechanical renames to `den.stages.*` | Low — find/replace |
| **Advanced** (custom `den.ctx` nodes with `into`/`provides`) | Split to `den.policies` + `den.stages` | Medium — requires understanding the new model |
| **Battery authors** (writing reusable Den modules) | Must export policies + stages + aspects separately | Medium — conceptual shift but cleaner result |

### Deprecation timeline

Phase 1 and 2 can provide compatibility shims: `den.ctx.X.nixos` can emit a deprecation warning and forward to `den.stages.X.nixos`. This allows a grace period where existing configs continue to work while users migrate. Phase 3 removes the shims.

## Appendix: Haskell Typeclass Parallel

Den's four-concern architecture maps closely to Haskell's type system. This appendix makes the parallel explicit as a reference for reasoning about where things belong.

### Vocabulary Mapping

| Haskell | Den | Purpose |
|---------|-----|---------|
| `data Host = Host { hostname :: Text, system :: Text, ... }` | `den.schema.host` + hostSubmodule | **Data declaration** — defines the structure of an entity |
| `class Resolvable a where resolve :: a -> Config` | Nix config class (`nixos`, `darwin`, `homeManager`) | **Typeclass** — a behavioral capability interface |
| `instance Resolvable Host where resolve = ...` | `den.aspects.nginx = { nixos.services.nginx.enable = true; }` | **Typeclass instance** — behavior implementation for a capability |
| `class (Resolvable a) => Deployable a` | Policy depth ordering (environment → host → user) | **Typeclass constraint** — one capability requires another |
| `deriving (Show, Eq)` | `den.schema.host.policies = [ ... ]` | **Derived instances** — the type declares which capabilities it participates in |
| Pattern match on constructor | `den.policies.host-to-users.resolve` | **Function dispatch** — deconstruct a value to produce related values |
| Newtype wrapper | `den.stages.hm-host` | **Newtype with instances** — a named scope that carries its own behavior bindings without modifying the underlying type or the behavior definitions |

### Where Things Belong (by Analogy)

**"What type is this?"** → `den.schema.*`
Like `data Host = ...`, the schema defines the structure. In Haskell, you declare derived instances on the type (`deriving Show`). In Den, you declare policy participation on the schema (`den.schema.host.policies`).

**"How do values of this type relate?"** → `den.policies`
Like pattern-matching a `Host` to extract its `[User]` list, a policy's `resolve` function takes an entity value and produces related values. Pure function, no behavior — just structure traversal.

**"What behavior does this scope get?"** → `den.stages`
Like a Haskell newtype wrapper that exists to carry typeclass instances different from the underlying type. `newtype HmHost = HmHost Host` lets you write `instance Configurable HmHost where ...` without modifying `Host` or the `Configurable` class. Similarly, `den.stages.hm-host` carries scoped behavior without modifying the policy or the aspect.

**"What does this behavior do?"** → `den.aspects`
Like a typeclass instance body (`instance Resolvable Host where resolve host = ...`), an aspect defines concrete behavior. It doesn't know which type it's attached to or which scope activated it — it just receives args and produces configuration.

### The Four Activation Levels as Instance Resolution

Haskell resolves typeclass instances by specificity. Den's four activation levels follow the same pattern:

```
1. Den core (always active)              ≈  Prelude instances (always in scope)
2. den.default.policies             ≈  Global orphan instances (apply everywhere)
3. den.schema.<kind>.policies       ≈  deriving clause (type declares participation)
4. den.hosts.*.policies             ≈  Local instance (value-level override)
```

More specific wins. An entity instance's policies override its schema's defaults, just as a local instance overrides a derived one.

### Why `den.ctx` Was the Wrong Abstraction

In Haskell terms, `den.ctx` was like defining a type that is simultaneously:
- A `data` declaration (it had structural keys)
- A typeclass instance (it had `.nixos`, `.includes` — behavior)
- A pattern match function (it had `.into` — policies)
- A newtype wrapper (it scoped behavior to a pipeline stage)

No Haskell programmer would put all four in one construct. The four-concern separation restores the natural boundaries:
- `data` → `den.schema`
- Typeclass instance → `den.aspects`
- Pattern match → `den.policies`
- Newtype scope → `den.stages`
