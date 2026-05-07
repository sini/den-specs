# Entity Resolution

Den's entity system defines the structural units of a configuration: hosts, users, homes, and the flake-level routing scaffolding that connects them. Each entity kind is declared through `den.schema`, a freeform registry whose merge logic separates structural entity kinds (which carry content and participate in the aspect pipeline) from routing kinds (which exist solely for policy-driven traversal). Entity resolution is the process of converting a schema entry and a context into a pipeline-ready aspect with scope handlers, self-provide, and static includes. The topology -- which entities contain which others -- is not declared explicitly but emerges from policy wiring.

## Schema Declaration

`den.schema` is a freeform submodule option with `lazyAttrsOf schemaEntryType`. Each attribute key names an entity kind; the value is a deferred module with extended merge semantics.

### Schema Entry Type

`schemaEntryType` extends `deferredModule` with a custom merge that performs four operations on the collected definitions:

1. **Extract includes.** Definitions that contain a list-valued `includes` field have those lists concatenated into a combined include set. The `includes` field is stripped from each definition before the deferred module merge.

2. **Extract excludes.** Same extraction for list-valued `excludes` fields.

3. **Deferred merge.** The stripped definitions are passed to the base `deferredModule` merge, producing the structural module content.

4. **Entity gating.** Two predicates determine the entry's role:

   - `hasStructuralContent` -- true when any definition contributes module content beyond `includes` and `excludes` (the stripped definition is non-empty or not an attrset).
   - `hasEntityContent` -- true when the kind is not `conf` AND at least one of: includes are non-empty, excludes are non-empty, or there is structural content.

The merged result is a callable attrset with `__functor`, `includes`, `excludes`, and `isEntity`:

- When `hasEntityContent` is true, the functor wraps the merged module and injects three options into the entity's module evaluation: `id_hash`, `resolved`, and `collisionPolicy` (described below). `isEntity` equals `hasStructuralContent`.
- When `hasEntityContent` is false, the functor passes through the bare module. `isEntity` is false and `includes`/`excludes` are empty.

### Injected Options

Entity kinds with content receive three options:

**`id_hash`** (internal, read-only, `str`) -- a SHA-256 hash of `"kind|key1=val1|key2=val2|..."` computed by reflecting on all non-internal, primitive-typed options (`str`, `int`, `bool`) declared on the entity. Provides stable identity comparison: use `a.id_hash != b.id_hash` instead of Nix's `==`, which does deep structural comparison and diverges across module system boundaries.

**`resolved`** (read-only, `raw`) -- the entity's pipeline-ready representation. Built by filtering `_module.args` to known schema kinds, adding `{ ${kind} = config; }`, and passing the result to the entity resolution function. This is the bridge from the NixOS module system into the pipeline.

**`collisionPolicy`** (nullable enum: `"error"`, `"class-wins"`, `"den-wins"`) -- per-entity override for how argument collisions between den context and module-system arguments are handled in flat-form class modules.

### Default Schema Entries

The framework registers these entries by default:

| Kind | Source | Body | Imports |
|---|---|---|---|
| `conf` | `modules/options.nix` | empty | -- |
| `host` | `modules/options.nix` | empty (structural content comes from entity type) | `den.schema.conf` |
| `user` | `modules/options.nix` | empty (structural content comes from entity type) | `den.schema.conf` |
| `home` | `modules/options.nix` | empty (structural content comes from entity type) | `den.schema.conf` |
| `flake` | `modules/context/flake-schema.nix` | empty | -- |
| `flake-system` | `modules/context/flake-schema.nix` | empty | -- |
| `default` | `modules/context/flake-schema.nix` | empty | -- |

`conf` is a shared configuration base module imported by host, user, and home. It is hardcoded as a non-entity (`kind != "conf"` in the gating predicate) and never participates in pipeline resolution.

When `den.default` is defined, `modules/aspects/defaults.nix` injects it as a schema include for host, user, and home, so the default aspect is resolved automatically for all structural entity kinds.

## Entity Kinds

### Structural Entities vs Routing Kinds

`schemaEntityKinds` (computed in `nix/lib/schema-util.nix`) filters the schema to kinds that are true entities:

```
filter each kind where: kind != "conf" AND NOT prefixed with "_" AND isEntity == true
```

A kind qualifies as an entity when its schema entry has structural module content -- actual options, imports, or configuration beyond just `includes`/`excludes`. Structural entities get self-provide (their `.aspect` field is resolved through the pipeline) and appear in derived binding lookups.

Routing kinds (`flake`, `flake-system`, `default`) have empty schema bodies, so `isEntity` is false. They exist for policy dispatch targeting but do not carry aspects or participate in self-provide.

### Built-in Entity Kinds

| Kind | `isEntity` | Self-Provide | Carries | Purpose |
|---|---|---|---|---|
| `host` | true | `host.aspect` | name, hostName, system, class, users, instantiate, intoAttr | OS configuration unit |
| `user` | true | `user.aspect` | name, userName, classes, host (back-ref) | User account within a host |
| `home` | true | `home.aspect` | userName, hostName, system, pkgs, instantiate, intoAttr | Home-manager configuration (standalone or host-bound) |
| `conf` | always false | none | -- | Shared base module imported by host/user/home |
| `flake` | false | none | -- | Root of flake-level pipeline traversal |
| `flake-system` | false | none | system | Per-system scope for flake outputs |
| `default` | false | special: `den.default` | -- | Global default aspect |

### Custom Entity Kinds

Any module can register new entity kinds by setting `den.schema.<name>`. If the entry has structural content, it becomes a full entity kind with `isEntity = true`, self-provide, and pipeline participation. If empty, it is a routing kind for policy dispatch only. The `fleet` kind used in fleet configurations is an example of a user-defined entity kind, not a framework built-in.

## Three-Layer Composition

Entity resolution combines content from three layers, each with different activation semantics.

### Layer 1: Aspects (Registry)

`den.aspects` is the global aspect registry. Each entity's `.aspect` field references a named aspect. During resolution, self-provide injects this aspect as an include, making it the root of the entity's content tree. Aspects may be parametric (taking entity context arguments) or static, and they declare class content, nested aspects, includes, and policies.

### Layer 2: Policies (Inclusion)

Policies are functions from entity context to lists of typed effects. They control topology (creating child entity scopes via resolve effects), content routing (via route and provide effects), and conditional inclusion/exclusion. Policies fire based on argument signature satisfaction -- a policy `{ host, user, ... }:` only fires when both `host` and `user` are present in the resolve context. See [policy-system.md](policy-system.md) for the full policy reference.

Policies are activated by being referenced in `den.schema.*.includes` or `den.default.includes`. The `den.policies` namespace is a named registry only -- it does not auto-dispatch.

### Layer 3: Schema Includes (Static)

`den.schema.<kind>.includes` holds a list of aspects and policies statically included when any entity of that kind is resolved. These are extracted during the schema entry type merge and concatenated after self-provide. Schema includes do not depend on runtime context -- they are identical for every entity of a given kind.

### Composition Order

When an entity is resolved, includes are assembled in this order:

1. **Self-provide** -- the entity's own `.aspect` (for structural entity kinds) or `den.default` (for the default kind)
2. **Schema includes** -- static includes from `den.schema.<kind>.includes`
3. **Policy includes** -- aspects injected by policy `include` effects during dispatch
4. **Resolve includes** -- per-resolve scoped includes carried on resolve effects

## Entity Resolution Lifecycle

The `resolve-schema-entity` effect handler orchestrates a complete entity resolution cycle. Each entity kind that a policy resolves into goes through this sequence:

### 1. Push Scope

A new scope partition is created. The entity's context bindings are installed as scope handlers (constant effect handlers returning context values). The scope is registered in pipeline state with a parent reference and inherits the parent's aspect policies.

### 2. Resolve Entity

The entity resolution function is called with the target kind and context. It produces an aspect-shaped record containing:

- `name` -- the entity kind
- `includes` -- self-provide + schema includes + policy includes + resolve includes
- `__entityKind` -- tags the aspect for policy dispatch gating
- `__scopeHandlers` -- constant handlers wrapping the entity context

Self-provide for structural entity kinds is a parametric wrapper that defers resolution of the entity's `.aspect` field until the scope's handlers are established. This prevents eager evaluation of the aspect before context is available.

### 3. Resolve (Tree Walk)

The merged entity is submitted to the resolve effect, which walks the aspect tree: compiling static content, emitting class modules, resolving child includes, and dispatching policies at entity boundaries.

### 4. Drain

After the entity's tree walk completes, deferred parametric includes that became satisfiable in this scope are collected. Each satisfiable deferred is tagged with the scope's handlers and resolved through the tree walk.

### 5. Propagate Routes

Routes registered at root scope that target this scope's partition are propagated. This ensures class content routed across scope boundaries reaches the correct destination.

### 6. Restore Scope

The scope is popped, returning to the parent scope. Results from this entity's resolution are accumulated with results from sibling entities.

## Topology Chain

Den's entity topology is not declared explicitly. It emerges from policy wiring: each policy that emits a resolve effect creates a parent-child relationship between entity kinds. The built-in policies produce this chain:

```
flake
  └─ flake-system (one per system in den.systems)
       ├─ host (one per host in den.hosts.${system})
       │    └─ user (one per user in host.users, shared scope)
       ├─ home (one per home in den.homes.${system})
       └─ [output routes: packages, apps, checks, devShells, legacyPackages]
```

### How the Chain Forms

1. **flake -> flake-system:** The `to-systems` policy, activated via `den.schema.flake.includes`, maps each system string in `den.systems` to a `resolve.to "flake-system" { inherit system; }` effect.

2. **flake-system -> host:** The `to-os-outputs` policy, activated via `den.schema.flake-system.includes`, iterates `den.hosts.${system}` and emits `resolve.to "host" { inherit host; }` plus an instantiation effect for each host with a non-empty `intoAttr`.

3. **host -> user:** The `host-to-users` policy, activated via `den.schema.host.includes`, maps each user in `host.users` to `resolve.shared { inherit user; }`. The `shared` variant means users share the host scope rather than getting isolated partitions.

4. **flake-system -> home:** The `to-hm-outputs` policy follows the same pattern as `to-os-outputs` for standalone home-manager configurations.

5. **flake-system -> outputs:** Per-output route policies (`to-packages`, `to-apps`, etc.) use `policy.route` to wire class content from per-system scope partitions into the flake output tree.

### Extending the Topology

User configurations extend this chain by declaring new entity kinds and policies. For example, a fleet configuration adds a `fleet` kind between `flake-system` and `host`, creating: `flake -> flake-system -> fleet -> host -> user`.

## Derived Bindings

When an entity is resolved, the entity resolution function checks whether the entity record carries references to other entity kinds. For example, a `home` entity may carry `host` and `user` fields that reference the associated host and user entities.

The derived bindings logic:

1. Reads the entity record from context (e.g., `ctx.home`)
2. Filters its attributes to those matching `schemaEntityKinds` names
3. Excludes the entity's own kind, null values, non-attrset values, and keys already present in context
4. Merges the discovered bindings into the context

This enables downstream policies and class modules to access related entities without explicit wiring. A `home` entity that carries `host` and `user` fields automatically makes those available in the resolve context, so a policy `{ home, host, ... }:` can fire without the host being explicitly passed through a resolve effect.

The augmented context also propagates any `collisionPolicy` set on the schema entry, accumulating it in `__collisionPolicies` keyed by entity kind.

## Entity Types

### Host

Declared in `nix/lib/entities/host.nix` as a nested submodule. Hosts are grouped by system under `den.hosts`:

```nix
den.hosts.x86_64-linux.myhost = { ... };
```

Key fields:

| Field | Type | Default | Description |
|---|---|---|---|
| `name` | `str` | attribute name | Configuration name |
| `hostName` | `str` | `name` | Network hostname |
| `system` | `str` | parent key | Platform system string |
| `class` | `str` | inferred from system | `"nixos"` or `"darwin"` |
| `aspect` | `raw` | looked up from `den.aspects` | Root aspect for this host |
| `users` | `attrsOf userType` | `{}` | User accounts on this host |
| `instantiate` | `raw` | from `class` | System builder function (e.g., `nixpkgs.lib.nixosSystem`) |
| `intoAttr` | `listOf str` | from `class` | Flake output path (e.g., `["nixosConfigurations" name]`) |
| `mainModule` | `deferredModule` | resolved from aspect | Final module passed to instantiation |

The host type imports `den.schema.host`, inheriting any shared `conf` content and entity options.

### User

Declared as a submodule nested within a host's `users` attribute:

```nix
den.hosts.x86_64-linux.myhost.users.tux = { ... };
```

Key fields:

| Field | Type | Default | Description |
|---|---|---|---|
| `name` | `str` | attribute name | Configuration name |
| `userName` | `str` | `name` | Account name |
| `classes` | `listOf str` | `["user"]` | Home management nix classes |
| `aspect` | `raw` | looked up from `den.aspects` | Root aspect for this user |
| `host` | (back-ref) | parent host | Reference to the containing host |

The user type imports `den.schema.user` and receives `host` via `_module.args`.

### Home

Declared in `nix/lib/entities/home.nix`. Homes are grouped by system under `den.homes` and support a `user@host` naming convention for host-bound configurations:

```nix
den.homes.x86_64-linux."tux@myhost" = { ... };
den.homes.x86_64-linux.standalone-user = { ... };
```

When the name contains `@`, the home automatically resolves the host from `den.hosts` and the user from that host's users, making both available as context bindings.

Key fields:

| Field | Type | Default | Description |
|---|---|---|---|
| `name` | `str` | user part of attribute name | Configuration name |
| `userName` | `str` | `name` | Account name |
| `hostName` | `str?` | host part after `@`, or null | Associated host name |
| `system` | `str` | parent key | Platform system string |
| `class` | `str` | `"homeManager"` | Always homeManager |
| `aspect` | `raw` | looked up from `den.aspects` | Root aspect for this home |
| `host` | (back-ref) | resolved from name, or null | Associated host entity |
| `user` | (back-ref) | resolved from name, or null | Associated user entity |
| `pkgs` | `raw` | `nixpkgs.legacyPackages.${system}` | Nixpkgs instance |
| `instantiate` | `raw` | `homeManagerConfiguration` | Builder function |
| `intoAttr` | `listOf str` | `["homeConfigurations" name]` | Flake output path |

For host-bound homes (`user@host`), the instantiation function is wrapped to inject `osConfig` from the host's evaluated configuration via `extraSpecialArgs`.

## Invariants

1. **Entity gating separates entities from routing kinds.** Only kinds with `isEntity = true` (structural content) get self-provide and participate in aspect resolution. Routing kinds exist for policy dispatch targeting only.

2. **Topology is implicit, not declared.** No entity kind knows its position in the hierarchy. Nesting order emerges entirely from policy wiring -- the same kind can appear at different positions in different configurations.

3. **Self-provide for entity kinds.** Each structural entity kind automatically includes a parametric wrapper that resolves its `.aspect` field. This is the bridge from entity declarations (`den.hosts`, `den.homes`) to the aspect registry (`den.aspects`).

4. **conf is never an entity.** The `conf` kind is hardcoded as excluded from entity processing, regardless of content. It serves as a shared base module only.

5. **Derived bindings are additive.** Entity-carried references to other entity kinds are merged into context only when the target kind is not already present. Existing context values are never overwritten.

6. **Identity hashing over structural equality.** Entity comparison uses `id_hash` (SHA-256 of primitive option values) rather than Nix's `==` operator, which diverges on recursive module system thunks.

7. **Schema includes are kind-uniform.** Every entity of a given kind receives the same static includes. Per-entity variation comes from aspects and policies, not from schema includes.

## Key Files

| File | Role |
|---|---|
| `modules/options.nix` | `den.schema` option, `schemaEntryType` merge with entity gating, injected options |
| `nix/lib/resolve-entity.nix` | Entity resolution: self-provide, schema includes, scope handlers, derived bindings |
| `nix/lib/schema-util.nix` | `schemaEntityKinds` computation (isEntity filter) |
| `nix/lib/entities/host.nix` | Host entity type definition |
| `nix/lib/entities/home.nix` | Home entity type definition |
| `nix/lib/aspects/fx/handlers/resolve-schema-entity.nix` | Resolution lifecycle: push-scope, resolve, drain, propagate, restore |
| `modules/policies/core.nix` | `host-to-users` traversal policy |
| `modules/policies/flake.nix` | Flake topology policies and output routing |
| `modules/context/flake-schema.nix` | Flake/flake-system/default routing kind registration |
| `modules/aspects/defaults.nix` | `den.default` schema include injection |

See also: [scope-partitioning.md](scope-partitioning.md) for how entity scopes partition pipeline state, [policy-system.md](policy-system.md) for the full policy dispatch and effect reference.
