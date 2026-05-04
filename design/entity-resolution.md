# Entity Resolution and Schema System

**Date:** 2026-05-04
**Branch:** feat/fx-pipeline (629/629 tests)
**Scope:** How den defines entity kinds, registers classes, resolves entities through the pipeline, and structures the three-layer model.

This document describes the current implementation, not aspirational design.

---

## 1. den.schema: Entity Kind Registry

### 1.1 Option Definition

`den.schema` is declared in `modules/options.nix` as a freeform submodule with `lazyAttrsOf schemaEntryType`:

```nix
options.den.schema = lib.mkOption {
  type = lib.types.submodule {
    freeformType = lib.types.lazyAttrsOf schemaEntryType;
  };
  default = {};
};
```

Each key under `den.schema` names an **entity kind**. The value is a `schemaEntryType` -- a custom type based on `deferredModule` with an extended merge function that extracts `includes` and `policies` from definitions before deferring the rest.

### 1.2 schemaEntryType Merge

The merge function (`options.nix` lines 33-186) does several things:

1. **Extracts `includes`** from all definitions that provide a list-valued `includes` field. These are concatenated into `allIncludes`.

2. **Extracts `policies`** from all definitions that provide an attrset-valued `policies` field. These are merged with `//` (last-wins).

3. **Strips `includes` and `policies`** from definitions before passing them to the base `deferredModule` merge.

4. **Gates entity behavior** via two predicates:
   - `hasStructuralContent`: true if any definition has module content beyond just `includes`/`policies` (i.e., the stripped definition is non-empty or non-attrset).
   - `hasEntityContent`: true if `kind != "conf"` AND (`allIncludes != []` OR `hasStructuralContent`).

5. **Returns a callable attrset** with:
   - `__functor`: wraps the merged deferred module. If `hasEntityContent`, injects `resolvedCtx` (identity hash, resolved entity, collision policy options). Otherwise, passes through the bare module.
   - `includes`: the extracted include list.
   - `policies`: the extracted policies attrset.
   - `isEntity`: equals `hasStructuralContent`. This controls whether the kind participates in pipeline resolution with self-provide.

### 1.3 Injected Options (resolvedCtx)

When a schema entry has entity content, `resolvedCtx` injects three options into the entity's module evaluation:

- **`id_hash`** (internal, read-only): SHA-256 of `"kind|key1=val1|key2=val2|..."` covering all non-internal primitive-typed options (str, int, bool). Used for entity identity comparison instead of Nix's fragile `==`.

- **`resolved`** (read-only, raw): Calls `den.lib.resolveEntity kind ctx` where `ctx` is built from `_module.args` filtered to known schema kinds, plus `{ ${kind} = config; }`.

- **`collisionPolicy`** (nullable enum): Per-entity override for class module collision handling (`"error"`, `"class-wins"`, `"den-wins"`).

### 1.4 Default Schema Entries

**`modules/options.nix` config section:**
```nix
config.den.schema = {
  conf = {};                        # shared base, not an entity
  host.imports = [ den.schema.conf ];
  user.imports = [ den.schema.conf ];
  home.imports = [ den.schema.conf ];
};
```

`conf` is a shared configuration base module. It is excluded from entity processing (`hasEntityContent` returns false for `kind == "conf"`). `host`, `user`, and `home` all import `conf`, inheriting any shared schema content.

**`modules/context/flake-schema.nix`:**
```nix
den.schema = lib.genAttrs [ "flake" "flake-system" "default" ] (_: {});
```

Registers three routing kinds with empty bodies. Empty body means `isEntity = false` -- these kinds exist for policy routing but don't carry structural content or self-provide aspects.

**`modules/aspects/defaults.nix`:**
```nix
config.den.schema = lib.mkIf (den ? default) (
  lib.genAttrs [ "host" "user" "home" ] (_: {
    includes = [ den.default ];
  })
);
```

Injects `den.default` as a schema include for host, user, and home entities.

### 1.5 Schema Policies

Policies can be declared directly on schema entries:

```nix
den.schema.host.policies.host-to-users = { host, ... }:
  map (user: resolve.shared { inherit user; }) (lib.attrValues host.users);
```

The merge function collects all `policies` attrs from definitions and returns them on the merged result. `installPolicies` in the pipeline reads `(den.schema.${kind} or {}).policies or {}` to discover entity-scoped policies.

---

## 2. den.classes Registry

### 2.1 Option Definition

```nix
options.den.classes = lib.mkOption {
  type = lib.types.lazyAttrsOf classSchemaType;
  default = {};
};
```

`classSchemaType` is a submodule with:
- `description` (str): human-readable description.
- `forwardTo` (nullable raw): optional forward target for class evaluation.

### 2.2 Built-in Classes

`modules/options.nix` registers two:
```nix
config.den.classes = {
  nixos.description = "NixOS system configuration";
  darwin.description = "nix-darwin system configuration";
};
```

`modules/policies/flake.nix` registers flake output classes:
```nix
den.classes = lib.listToAttrs (
  map (output: {
    name = output;
    value.description = "Flake ${output} output class";
  }) [ "packages" "apps" "checks" "devShells" "legacyPackages" ]
);
```

### 2.3 Dynamic Collection (aspect-schema.nix)

`modules/aspect-schema.nix` collects `classes` declarations from aspects and namespaces:

1. Iterates `config.den.aspects`, reading `a.classes or {}` from each.
2. Iterates `config.den.ful.*` namespaces, collecting classes from namespace-level aspects and from `den.ful.<ns>.classes`.
3. Merges everything with `//` into `config.den.classes`.

This enables aspects to declare new class domains:
```nix
den.aspects.foo.classes.hjem = { description = "Hjem home configuration"; };
```

### 2.4 Role in Key Classification

`key-classification.nix` uses `den.classes` as the class registry. When classifying non-structural keys on an aspect:

1. If the key matches a registered class name (or the current target class), it is a **class key** -- content is emitted via `emit-class`.
2. If the key contains sub-keys that match registered classes (depth-limited to 3), it is a **nested aspect key** -- recursed via `emitNestedAspect`.
3. Otherwise, it is an **unregistered class key** -- treated as a class for backward compatibility, with trace warning.

When the registry is empty (no batteries loaded), all non-structural keys are treated as classes.

### 2.5 Current State: No Traits

The `den.traits` option was removed during the cleanup arc. Only `den.classes` exists. Key classification is two-branch (class vs nested aspect), not three-branch. Trait reimplementation is planned via fleet + `den.exports`.

---

## 3. Entity Resolution: resolveEntity

### 3.1 Implementation

`nix/lib/resolve-entity.nix` exports a single function:

```nix
resolveEntity = name: ctx: { ... }
```

**Parameters:**
- `name`: entity kind string (e.g., `"host"`, `"user"`, `"flake-system"`).
- `ctx`: attrset of context bindings (e.g., `{ host = <hostConfig>; user = <userConfig>; }`).

**Returns** an aspect-shaped attrset:
```nix
{
  name = <kind>;
  meta = { handleWith = null; excludes = []; provider = []; };
  includes = selfProvide ++ schemaIncludes;
  __entityKind = <kind>;
  __scopeHandlers = constantHandler ctx;
}
```

### 3.2 Self-Provide

The self-provide mechanism injects the entity's own aspect as an include, resolved lazily via parametric resolution:

- **For `"default"`**: includes `[ den.default ]` (if `den ? default`).
- **For aspect-bearing entity kinds** (host, user, home -- determined by `schemaEntityKinds` from `schema-util.nix`): includes a parametric wrapper:
  ```nix
  {
    __fn = c: c.${name}.aspect;
    __args = { ${name} = false; };
    name = "<self:${name}>";
    meta = {};
    includes = [];
  }
  ```
  This wrapper resolves during parametric handling: `bind.fn` probes the scope for a handler named by the entity kind (e.g., `"host"`), retrieves the entity config, and calls `c.host.aspect` to get the aspect content.
- **For non-aspect kinds** (flake, flake-system, conf): no self-provide. Returns empty `includes` (before schema includes are appended).

### 3.3 schemaEntityKinds

`nix/lib/schema-util.nix` computes entity kinds from the schema:

```nix
schemaEntityKinds = builtins.filter (
  k: k != "conf" && !(lib.hasPrefix "_" k) && (den.schema.${k}.isEntity or false)
) (builtins.attrNames (den.schema or {}));
```

A kind qualifies if: (a) not `"conf"`, (b) not private (no `_` prefix), and (c) `isEntity == true` on its schema entry. `isEntity` is set during `schemaEntryType` merge based on whether the entry has structural content.

### 3.4 Schema Includes

```nix
schemaIncludes = ((den.schema or {}).${name} or {}).includes or [];
```

These are the static includes extracted by `schemaEntryType` merge -- aspects and modules declared directly on `den.schema.<kind>.includes`.

### 3.5 __scopeHandlers

`resolveEntity` wraps the context as `constantHandler ctx`. Each context key becomes a named effect handler that returns its value when probed. This is how context propagates through the pipeline -- `bind.fn` probes for handlers matching function argument names.

### 3.6 __entityKind

Tags the resolved entity with its kind. The pipeline uses this to gate policy dispatch: `installPolicies` only runs for aspects with `__entityKind` set (see `resolveChildren` in `aspect.nix`).

---

## 4. resolve-schema-entity Effect

### 4.1 Purpose

`handlers/resolve-schema-entity.nix` provides the `resolve-schema-entity` effect handler. This is the reusable entity resolution primitive: scope push, entity walk, scope pop. Policy dispatch uses it to resolve target entities.

### 4.2 Parameters

The effect receives:
```nix
{
  targetKind;      # entity kind to resolve (e.g., "user")
  scopedCtx;       # full context for the new scope
  entityClass;     # optional class override
  includeAspects;  # aspects from policy includes
  policyIncludes;  # includes from policy.resolve.withIncludes
  resolveIncludes; # includes from the resolve effect itself
  ctxNames;        # context key names for tracing
  prevResults;     # accumulated results from prior sibling resolves
}
```

### 4.3 Execution Sequence

1. **Compute scope ID**: `mkScopeId scopedCtx` produces a canonical `"key=value,..."` string.

2. **Build scope transition**: `mkScopeTransition` creates three operations:
   - `scopeHandlersForCtx`: `constantHandler` wrapping the scoped context (plus optional class override).
   - `setScope`: `state.modify` that sets `currentScope`, registers the scope in `scopeContexts`, sets `scopeParent`, and inherits `scopedAspectPolicies` from parent.
   - `restoreScope`: `state.modify` that resets `currentScope` to parent.

3. **Push scope**: Execute `setScope`.

4. **Copy parent deferred items**: Parent-scope deferred includes are copied into the new scope. This enables fan-out: a `{ host, user }` deferred at host scope gets a fresh copy in each user scope.

5. **Install scope handlers**: `scope.provide scopeHandlersForCtx` makes entity context available to the subtree.

6. **Resolve entity**: Send `resolve-entity` effect with `kind = targetKind`. The `resolveEntityHandler` in `pipeline.nix` handles this by calling `den.lib.resolveEntity`.

7. **Merge includes**: The raw entity's includes are concatenated with `includeAspects`, `policyIncludes`, and `resolveIncludes` (all stripped of stale `__ctxId` to prevent identity mismatches).

8. **Walk tree**: `aspectToEffect entity` processes the merged entity through the pipeline -- compileStatic, emitClasses, resolveChildren, installPolicies.

9. **Drain deferred**: Send `drain-deferred` to resolve any parametric includes that became satisfiable in this scope.

10. **Walk deferred results**: Each satisfiable deferred include is tagged with the scope's handlers and processed via `aspectToEffect`.

11. **Propagate root routes**: Complex routes registered at root scope are copied to the child scope with `sourceScopeId` pointing at the child's partition.

12. **Restore scope**: Execute `restoreScope` to pop back to parent.

### 4.4 den.default Stripping

The `resolveEntityHandler` in `pipeline.nix` (not `resolve-schema-entity.nix`) strips `den.default` from child entity includes before returning. This prevents duplicate module definitions: `den.default` is walked once at root scope, and child entities get its shared modules via filtered root fallback in `applyForwardSpecs`.

---

## 5. The Three-Layer Model

### 5.1 Layer 1: Aspects (Registry)

`den.aspects` is the global aspect registry. Aspects are named definitions that may be parametric (taking entity context arguments) or static. They declare class content, nested aspects, includes, policies, and constraints.

Aspects are referenced directly by value (not by string name) in policy effects and schema includes. Example:

```nix
den.aspects.my-firewall = { host, ... }: {
  nixos.networking.firewall.enable = true;
};
```

### 5.2 Layer 2: Policies (Inclusion and Routing)

Policies are plain functions returning lists of typed effects. They are declared on schema entries (`den.schema.<kind>.policies.<name>`) or via the provides-compat shim.

Policy effects:
- `policy.resolve { user = tux; }` -- create a new entity scope (fan-out).
- `policy.resolve.shared { user = tux; }` -- shared (non-isolated) fan-out.
- `policy.resolve.to "kind" { bindings }` -- explicit target kind.
- `policy.resolve.withIncludes [aspects] { bindings }` -- resolve with scoped includes.
- `policy.include aspect` -- inject an aspect into current resolution context.
- `policy.exclude aspect` -- gate an aspect from the tree.
- `policy.route { fromClass, intoClass, path, ... }` -- route class content between scope partitions.
- `policy.instantiate entity` -- request post-pipeline instantiation.
- `policy.provide { class, module, ... }` -- deliver content directly to a class.
- `policy.pipelineOnly value` -- tag with `collisionPolicy = "class-wins"`.

Policies fire based on argument satisfaction: a policy `{ host, user, ... }:` fires only when both `host` and `user` are in scope. `installPolicies` dispatches policies via context enrichment iteration.

### 5.3 Layer 3: Schema Includes (Static)

`den.schema.<kind>.includes` holds a list of aspects/modules statically included when an entity of that kind is resolved. These are picked up by `resolveEntity` and concatenated after the self-provide.

Examples:
- `den.schema.host.includes = [ den.schema.conf ];` (base conf module)
- `den.schema.host.includes = [ den.default ];` (from defaults.nix, conditional)

Schema includes do not depend on runtime context -- they are the same for every entity of that kind.

### 5.4 How the Layers Compose

When an entity is resolved:

1. `resolveEntity` produces the root aspect with:
   - Self-provide (the entity's own `.aspect` via parametric wrapper)
   - Schema includes (static, from `den.schema.<kind>.includes`)

2. The pipeline walks this root aspect via `aspectToEffect` -> `compileStatic` -> `resolveChildren`.

3. `resolveChildren` calls `installPolicies` (only for aspects tagged with `__entityKind`). `installPolicies`:
   - Reads `den.schema.${kind}.policies` for entity-scoped policies.
   - Reads accumulated `scopedAspectPolicies` from parent scopes.
   - Dispatches satisfied policies, which emit effects (resolve, include, route, etc.).
   - For `policy.resolve`, sends `resolve-schema-entity` to recursively resolve child entities.

4. Child entities repeat from step 1, inside a `scope.provide` frame that provides the expanded context.

---

## 6. Entity Kinds in Current Use

### 6.1 host

- **Schema**: imports `den.schema.conf`. Structural content from entity type declarations.
- **isEntity**: true (has structural content from host type options).
- **Self-provide**: `{ host }: host.aspect` -- resolves the host's named aspect from the registry.
- **Entity type** (`nix/lib/entities/host.nix`): submodule with `name`, `hostName`, `system`, `class` (nixos/darwin), `aspect`, `users`, `instantiate`, `intoAttr`, `mainModule`. Imports `den.schema.host`.
- **Policies**: `host-to-users` fans out to each user via `resolve.shared`.

### 6.2 user

- **Schema**: imports `den.schema.conf`. Structural content from entity type declarations.
- **isEntity**: true.
- **Self-provide**: `{ user }: user.aspect` -- resolves the user's named aspect.
- **Entity type**: submodule with `name`, `userName`, `classes`, `aspect`, `host` (parent ref). Imports `den.schema.user`. Parent host is injected via `_module.args.host`.

### 6.3 home

- **Schema**: imports `den.schema.conf`. Structural content from entity type declarations.
- **isEntity**: true.
- **Self-provide**: `{ home }: home.aspect`.
- **Entity type** (`nix/lib/entities/home.nix`): submodule with `name`, `userName`, `hostName`, `system`, `class` (homeManager), `aspect`, `pkgs`, `instantiate`, `intoAttr`, `mainModule`. Supports `user@host` naming convention for host-bound homes. Imports `den.schema.home`.

### 6.4 flake

- **Schema**: empty body, registered by `flake-schema.nix`.
- **isEntity**: false (no structural content).
- **Self-provide**: none (not in `schemaEntityKinds`).
- **Purpose**: root entity kind for the flake-level pipeline. Policies on `den.schema.flake` fan out to `flake-system`.

### 6.5 flake-system

- **Schema**: empty body, registered by `flake-schema.nix`.
- **isEntity**: false.
- **Self-provide**: none.
- **Purpose**: per-system scope. Carries `{ system }` context. Policies fan out to OS outputs (`policy.instantiate`), HM outputs, and per-output routes (`policy.route`).

### 6.6 conf

- **Schema**: empty body. Declared in `options.nix`.
- **isEntity**: always false (hardcoded exclusion: `kind != "conf"`).
- **Purpose**: shared base module imported by host, user, and home schema entries. Not an entity kind -- a configuration mixin.

### 6.7 default

- **Schema**: empty body, registered by `flake-schema.nix`.
- **isEntity**: false.
- **Self-provide**: special-cased in `resolveEntity` -- includes `[ den.default ]` if defined.
- **Purpose**: global default aspect applied to all entity kinds via schema includes. The `resolveEntityHandler` strips `den.default` from child entities to prevent duplicate resolution.

### 6.8 Custom Entity Kinds

Any module can register new entity kinds by setting `den.schema.<name>`. If the entry has structural content (non-empty module body beyond includes/policies), it becomes a full entity kind with `isEntity = true`, self-provide, and pipeline participation. If the entry is empty, it is a routing kind that exists only for policy dispatch targeting.

---

## 7. Key Code Locations

| File | Role |
|------|------|
| `modules/options.nix` | `den.schema`, `den.classes`, `den.hosts`, `den.homes` option declarations; `schemaEntryType` with entity gating and `resolvedCtx` injection |
| `nix/lib/resolve-entity.nix` | `resolveEntity` function: self-provide, schema includes, scope handlers |
| `nix/lib/schema-util.nix` | `schemaEntityKinds` (isEntity filter), `schemaArgKinds` (for class-module warnings) |
| `nix/lib/entities/host.nix` | Host entity type with users submodule |
| `nix/lib/entities/home.nix` | Home entity type with host/user resolution |
| `nix/lib/aspects/fx/handlers/resolve-schema-entity.nix` | `resolve-schema-entity` effect: scope push/walk/pop |
| `nix/lib/aspects/fx/pipeline.nix` | `resolveEntityHandler` (strips den.default), `mkScopeId`, `defaultHandlers`, `defaultState` |
| `nix/lib/aspects/fx/aspect.nix` | `resolveChildren`, `installPolicies`, `compileStatic`, `aspectToEffect` |
| `nix/lib/aspects/fx/key-classification.nix` | `classifyKeys`, `structuralKeysSet` |
| `nix/lib/aspects/fx/include-emit.nix` | `emitSelfProvide`, `emitIncludes`, `emitAspectPolicies` |
| `modules/aspect-schema.nix` | Dynamic class collection from aspects |
| `modules/context/flake-schema.nix` | Flake/flake-system/default schema registration |
| `modules/aspects/defaults.nix` | `den.default` option and schema include injection |
| `modules/policies/core.nix` | `host-to-users` policy |
| `modules/policies/flake.nix` | Flake output policies and flake output class registration |
| `nix/lib/policy-effects.nix` | Typed policy effect constructors |

---

## 8. Planned Redesign: Unified Resolve Effects

**Spec:** `design/unified-resolve-effects.md`

### Changes to Entity Resolution

- **Section 4.3** (execution sequence): Steps 8-10 (walk tree, drain deferred, walk deferred results) change:
  - `aspectToEffect entity` → `fx.send "resolve" { aspect = entity, identity, ctx }`
  - `drain-deferred` → `fx.send "drain" { ctx = scopedCtx }` (same semantics, renamed)
  - Deferred walk unchanged (re-resolves satisfiable items via `resolve` effect)

- **Automatic drain:** For simple scope entries (enrichment widen in policy iteration), drain is automatic via `scope-widened` handler + `enterScope` wrapper. Entity resolution retains explicit drain at the precise point in its orchestration (after entity walk, before route propagation).

- **Fan-out copy (copyDeferredToScope):** Unchanged. This is a state setup step (duplicating parent deferred into child scope), not a drain. Remains as direct state mutation in `resolve-schema-entity`.

- **resolve-entity handler:** Unchanged. Still called by `resolve-schema-entity` to look up entity config.

### New Primitive

```nix
enterScope = handlers: computation:
  fx.effects.scope.provide handlers (
    fx.bind (fx.send "scope-widened" { ctx = ctxFromHandlers handlers; }) (
      _: computation
    )
  );
```

Used by `policy/iterate` for enrichment widen. NOT used by `resolve-schema-entity` (which needs precise drain timing).
