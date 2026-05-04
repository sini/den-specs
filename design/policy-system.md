# Den Policy System — Design Spec (feat/fx-pipeline, 629/629)

**Date:** 2026-05-04
**Branch:** feat/fx-pipeline
**Source of truth:** Code in nix/lib/policy-effects.nix, nix/lib/aspects/fx/policy-dispatch.nix, nix/lib/aspects/fx/aspect.nix, nix/lib/aspects/fx/include-emit.nix, nix/lib/aspects/fx/handlers/tree.nix, modules/policies/core.nix, modules/policies/flake.nix

---

## 1. Policy Shape

A policy is a plain Nix function from context to a list of typed effects:

```nix
den.policies.<name> = { host, user, ... }: [
  (policy.resolve { ... })
  (policy.include den.aspects.foo)
];
```

Policies carry no metadata — no `from`, `to`, `__functor`, `_core`, or `isolateFanOut` fields. The function's parameter signature determines when it fires: the pipeline matches required parameter names against available context keys. A policy fires when every required argument (non-default parameter) is present in the resolve context. Optional arguments (those with defaults) do not gate firing.

The `resolveArgsSatisfied` function (synthesize-policies.nix) implements this check:

```nix
resolveArgsSatisfied = policy: ctx:
  let
    fargs = lib.functionArgs policy;
    requiredArgs = builtins.filter (k: !fargs.${k}) (builtins.attrNames fargs);
  in
  builtins.all (k: ctx ? ${k}) requiredArgs;
```

A policy taking `_:` or `{ ... }:` fires unconditionally. A policy taking `{ host, ... }:` fires only when `host` is in context.

A policy may return a single effect or a list of effects. The dispatch normalizes both to a list.

---

## 2. Effect Types

All effect constructors live in `nix/lib/policy-effects.nix` and are exposed as `den.lib.policy.*`. Each returns a tagged attrset with a `__policyEffect` string discriminator.

### 2.1 resolve

Creates a new context scope (fan-out). Each resolve creates a parallel sibling branch with new bindings merged into the parent context.

```nix
policy.resolve { user = tux; }
# => { __policyEffect = "resolve"; __shared = false; value = { user = tux; }; includes = []; }
```

**Variants:**

| Constructor | Semantics |
|---|---|
| `policy.resolve bindings` | Isolated fan-out. Child scope gets its own scope partition. |
| `policy.resolve.shared bindings` | Shared fan-out. `__shared = true`. Multiple siblings share context rather than getting isolated partitions. Used by `host-to-users` so users share the host scope. |
| `policy.resolve.to kind bindings` | Explicit target entity kind. `__targetKind = kind` overrides the default derivation from binding keys. Used for non-schema routing (e.g., `resolve.to "flake-system"`). |
| `policy.resolve.shared.to kind bindings` | Shared + explicit target. |
| `policy.resolve.withIncludes includes bindings` | Per-resolve scoped includes. The `includes` list is carried on the resolve effect and emitted inside the child scope's `scope.provide` frame. |
| `policy.resolve.to.withIncludes kind includes bindings` | Explicit target + per-resolve includes. |
| `policy.resolve.shared.withIncludes includes bindings` | Shared + per-resolve includes. |

**Resolve classification:** The pipeline classifies each resolve effect's binding keys against `schemaEntityKinds` (entity kinds from `den.schema` where `isEntity = true`):

- **Schema resolve:** All keys match schema entity kinds, or `__targetKind` is set. Creates a child entity via `resolve-schema-entity`.
- **Enrichment resolve:** No keys match schema entity kinds. Enriches the current entity's context — no child entity created. The bindings become available to subsequent policy dispatch iterations and to class modules via `wrapClassModule`.
- **Mixed resolve:** Some keys match, others don't. Split: schema keys create child entities, non-schema keys enrich current context.

`policy.resolve {}` (empty bindings) is a no-op — filtered out before processing.

### 2.2 include

Injects an aspect into the current resolution context:

```nix
policy.include den.aspects.sudo
# => { __policyEffect = "include"; value = <aspect>; }
```

Accepts aspect references and inline attrsets (coerced to anonymous aspects). Emitted via `emit-include` with source policy name tagging for identity dedup — this prevents `emit-class` LOC dedup from collapsing distinct policy outputs. When the include value has no `name` attribute, it is tagged with `name = "<policy:${policyName}>"`.

### 2.3 exclude

Removes/gates an aspect from the current resolution subtree:

```nix
policy.exclude den.aspects.desktop
# => { __policyEffect = "exclude"; value = <aspect>; }
```

Emitted via `register-constraint` with `type = "exclude"`, `scope = "subtree"`. The identity is derived from the aspect's path. Excludes propagate into child scopes.

### 2.4 route

Routes class content from one scope partition into a target class. Replaces `den.provides.forward` for the common (route-eligible) case:

```nix
policy.route {
  fromClass = "packages";
  intoClass = "flake";
  path = [ "flake" "packages" system ];
  adaptArgs = _: { pkgs = inputs.nixpkgs.legacyPackages.${system}; };
}
# => { __policyEffect = "route"; value = <spec>; }
```

Emitted via `register-route`. The route spec is processed post-pipeline by `applyForwardSpecs` / `wrapRouteModules`.

### 2.5 instantiate

Requests post-pipeline instantiation of an entity's class content. The entity carries `instantiate`, `intoAttr`, and `mainModule` metadata:

```nix
policy.instantiate hostEntity
# => { __policyEffect = "instantiate"; value = <entity-spec>; }
```

Emitted via `register-instantiate`. Used by flake policies to wire OS and HM outputs (`to-os-outputs`, `to-hm-outputs`).

### 2.6 provide

Delivers a new module directly into a target class, bypassing the aspect tree walk:

```nix
policy.provide { class = "nixos"; module = myModule; }
# => { __policyEffect = "provide"; value = { class = "nixos"; module = myModule; }; }
```

Emitted via `register-provide`. Created to fix duplicate emissions from the provides-compat shim — `policy.include` walks content through the tree (creating duplicates when content matches routes), while `policy.provide` bypasses the tree entirely.

### 2.7 pipelineOnly

Not a policy effect — a value wrapper. Tags a value with `collisionPolicy = "class-wins"` so that when it reaches a class module that also receives the same arg from the module system (e.g., NixOS provides `lib`), the pipeline value wins silently:

```nix
policy.pipelineOnly value
# For attrsets: value // { collisionPolicy = "class-wins"; }
# For non-attrsets: { __functor = _: value; collisionPolicy = "class-wins"; }
```

Used by `den.provides.flake-scope` to inject `lib`, `inputs`, `den` into the pipeline without colliding with NixOS's own `lib`.

---

## 3. Dispatch

### 3.1 Entry Point: installPolicies

`installPolicies` (policy-dispatch.nix, invoked from aspect.nix:resolveChildren) is the entry point. It fires when an aspect has `__entityKind` set — i.e., the aspect represents a schema entity boundary.

**Dedup:** Each entity scope dispatches policies at most once. The dispatch is keyed by `"${entityKind}@${currentScope}"` and tracked in `state.dispatchedPolicies`. If already dispatched, returns immediately.

**Context assembly:** The resolve context is built from three sources, merged with later sources taking precedence:
1. `state.scopeContexts.${currentScope}` — scope context from prior enrichment
2. `ctxFromHandlers(aspect.__scopeHandlers)` — handler-derived context from the aspect's lexical scope
3. `{ __entityKind = entityKind }` — entity kind tag injected for policies that discriminate on it

### 3.2 Policy Sources

Three policy registries are merged for dispatch:

| Registry | Source | Scope |
|---|---|---|
| **Global** | `den.policies` | Fires at every entity scope where signature matches |
| **Schema-scoped** | `den.schema.${entityKind}.policies` | Fires only at entities of matching kind |
| **Aspect-included** | `state.scopedAspectPolicies` (accumulated via `register-aspect-policy`) | Fires when the aspect containing the policy has been included in the tree walk |

Global and schema-scoped policies are merged into `allDirectPolicies` (schema policies shadow global policies with the same name). Aspect policies are dispatched separately via `dispatchAspect`.

### 3.3 Dispatch Mechanics

`dispatchDirect` iterates `allDirectPolicies` — for each policy, checks `resolveArgsSatisfied` and that the policy name hasn't already been fired. If satisfied, calls the policy with the resolve context and collects its effects.

`dispatchAspect` does the same for aspect policies, checking `lib.functionArgs` of the `.fn` field rather than the policy itself (aspect policies are stored as `{ fn, ownerIdentity }` records).

Results from both dispatchers are concatenated, then classified by `classifyPolicyResult` into per-effect-type buckets:

- `schemaEffects` — resolve effects with schema keys or explicit target kinds
- `mergedEnrichment` — all enrichment keys merged into one attrset
- `includeEffects`, `excludeEffects`, `routeEffects`, `instantiateEffects`, `provideEffects`

### 3.4 Cross-Provider Tagging

When a policy produces both schema resolve effects (with `__targetKind`) and include effects, it is tagged `isCrossProvider = true`. In this case, the include effects are attached to the schema effects as `__policyIncludes` rather than emitted independently. This allows cross-provider patterns where a policy routes content to a target entity and includes aspects that should be walked within that target's scope.

Non-cross-provider include effects are emitted normally, tagged with `__sourcePolicyName` for identity tracking.

---

## 4. Enrichment: Fixed-Point Iteration

The `iterate` function implements enrichment-to-fixpoint. Starting from the initial resolve context:

1. **Dispatch** all policies against current context
2. **Collect** enrichment keys (non-schema resolve bindings)
3. If **new enrichment keys** appeared (keys not in the accumulated enrichment):
   - Merge new enrichment into accumulated context
   - Install enrichment as scoped handlers via `scope.provide(constantHandler(combinedEnrichment))`
   - Update `state.scopeContexts` for the current scope
   - **Drain deferred includes** — parametric aspects that were deferred because their required args weren't available may now resolve with the new enrichment keys
   - **Re-dispatch** (go to step 1 with widened context)
4. If **no new enrichment keys** — enrichment is stable. Emit all accumulated effects.

**Termination:** Iteration is capped at `maxPolicyIterations = 10`. Exceeding this throws an error indicating a likely enrichment cycle.

**Deferred drain:** After enrichment widens context, `drainEnrichmentDeferred` sends a `drain-deferred` effect. Satisfiable deferred aspects are retrieved, tagged with the enriched context's scope handlers, and resolved via `aspectToEffect`. This allows parametric aspects like `{ isNixos, ... }: { ... }` to resolve once `isNixos` becomes available through enrichment.

**Commutativity:** Enrichment is downward-flowing only. A child scope's enrichment does not affect parent or sibling scopes. Multiple enrichment resolves from different policies merge additively — later values shadow earlier ones for the same key. Merge order follows dispatch order (global before schema before aspect), but the result is deterministic per-evaluation.

---

## 5. Aspect-Included Policies

Aspects can declare policies in their `.policies` attribute:

```nix
den.aspects.myBattery = {
  policies.deliver-niri = { host, user, ... }: [ ... ];
  # ...aspect content...
};
```

### 5.1 Registration

During the tree walk, when `resolveChildren` processes an aspect, it calls `emitAspectPolicies` (include-emit.nix). For each entry in `aspect.policies`, a `register-aspect-policy` effect is sent with:
- `name`: `"${aspectName}/${policyName}"` — scoped by the owning aspect
- `fn`: the policy function
- `ownerIdentity`: the aspect's identity path key

### 5.2 Handler

The `registerAspectPolicyHandler` (tree.nix) stores aspect policies in `state.scopedAspectPolicies`, keyed by scope:

```nix
state.scopedAspectPolicies.${currentScope}.${name} = { fn, ownerIdentity };
```

Policies from all scopes are merged (via `foldl'` over `builtins.attrValues`) when `installPolicies` reads aspect policies. This means an aspect policy registered at any scope is visible to all subsequent entity dispatches.

### 5.3 Provides-Compat Registration

The `provides-compat.nix` handler also uses `register-aspect-policy` to register compatibility policies for `provides.to-hosts`, `provides.to-users`, and named target patterns. These are registered with names like `"${aspectName}/compat:${key}"`.

### 5.4 Late Dispatch Pass

When `installPolicies` processes fan-out (multiple sibling schema resolves), aspect policies registered by later siblings may not have been visible when earlier siblings were dispatched. The `lateDispatchPass` corrects this:

1. After all sibling resolves complete, iterate over sibling metadata
2. For each sibling, find aspect policies not yet fired at that scope
3. Dispatch those late policies against the sibling's context
4. Emit include, route, instantiate, provide, and exclude effects within the sibling's scope (using `scope.provide` with the sibling's context handlers)

Schema effects from late dispatch are intentionally NOT processed — they would cause duplicate module emissions with different identities.

---

## 6. Schema-Scoped Policies

`den.schema.${kind}.policies` provides entity-kind-scoped policy registration. These policies fire only when `installPolicies` runs at an entity of the matching kind.

### 6.1 Built-in Schema Policies

**modules/policies/core.nix:**

```nix
den.schema.host.policies.host-to-users = { host, ... }:
  map (user: resolve.shared { inherit user; }) (lib.attrValues host.users);
```

The core traversal policy. When resolving a host entity, it fans out to each user with `resolve.shared` — users share the host scope rather than getting isolated partitions.

**modules/policies/flake.nix:**

Schema-scoped policies for flake output wiring:

| Policy | Schema | Effect |
|---|---|---|
| `to-systems` | `den.schema.flake` | Fan-out per system via `resolve.to "flake-system"` |
| `to-os-outputs` | `den.schema.flake-system` | `policy.instantiate` for each host with `intoAttr` |
| `to-hm-outputs` | `den.schema.flake-system` | `policy.instantiate` for each home with `intoAttr` |
| `to-${output}` | `den.schema.flake-system` | `policy.route` from output class into flake at `["flake" output system]` |

The `to-${output}` policies are generated for system outputs (packages, apps, checks, devShells, legacyPackages) and conditionally fire based on whether the flake output option exists.

### 6.2 Merge with Global

In `installPolicies`, schema policies are merged after global policies (`globalPolicies // schemaPolicies`). Schema policies shadow global policies with the same name.

---

## 7. Effect Emission

After enrichment stabilizes, `emitFinalEffects` processes accumulated effects in order:

1. **Excludes** — `policyEmitExcludes` sends `register-constraint` for each exclude effect
2. **Routes, instantiates, provides** — `policyEmitEffects` sends `register-route`, `register-instantiate`, `register-provide` for each respective effect
3. **Schema resolves** — if present, `processSchemaResolves` handles entity creation via `resolve-schema-entity`
4. **Includes** (only if no schema resolves) — `policyEmitIncludes` sends `emit-include` for each include effect

When schema resolves exist, include effects are folded into the resolve processing (their aspect values are passed as `includeAspects` to `processSingleResolve`), not emitted independently.

### 7.1 Schema Resolve Processing

`processSingleResolve` handles each schema resolve effect:

1. Determine `targetKind` — from `__targetKind` if set, otherwise from the first schema key in bindings
2. Build `scopedCtx` by merging enriched context with resolve bindings
3. Compute `ctxNames` via `mkScopeId(scopedCtx)` — the scope identifier
4. Compute `ctxKey` — `"${targetKind}/{${ctxNames}}"` for fan-out, or just `targetKind` for single resolve
5. Resolve entity class from bindings if available
6. Send `ctx-seen` effect to check dedup — returns `{ isFirst, newAspectValues }`
7. If first visit: send `resolve-schema-entity` to walk the entity tree within the child scope
8. If revisit with new aspects: emit supplemental includes into the existing scope
9. If revisit with no new aspects: no-op

---

## 8. Fired Policy Tracking

Fired policy names are tracked at two granularities:

1. **Per-iteration within `iterate`:** `firedPolicies` list prevents re-firing a policy within the same enrichment loop. Policies that produce effects are added to this list.

2. **Per-scope in pipeline state:** `state.firedPolicyNames.${dispatchKey}` (where `dispatchKey = "${entityKind}@${scopeId}"`) records all policies fired at each scope. This is read by `lateDispatchPass` to avoid re-firing policies during cross-sibling late dispatch.

---

## 9. Future Direction: Policy Scoping Redesign

(From consolidated spec Section 9 — not yet implemented.)

### 9.1 Problem

Policy scoping is implicit and inconsistent:
- Defining `den.aspects.foo.policies.bar` both registers and activates the policy — there is no way to store a policy without activating it
- Scope is determined by definition site (global, schema, aspect) but this is emergent behavior, not an explicit design
- No fine-grained scoping — "this policy applies only to igloo's subtree" requires guards in the policy body
- `meta.handleWith` and `meta.excludes` are parallel mechanisms that are semantically scoped policies but use separate constraint registry machinery

### 9.2 Proposed Design: Registry vs Activation

`.policies` becomes the **registry** (stores policy functions). `includes`/`excludes` becomes the **activation method** (controls where policies fire).

Policy scoping levels:

| Level | Scope | Activation |
|---|---|---|
| Pipeline | Entire pipeline run | Pipeline configuration |
| Global | All entities, all scopes | `den.policies.*` (preserved) |
| Entity-kind | All entities of a kind | `den.schema.*.policies.*` (preserved) |
| Entity-instance | A specific entity | Include policy on the entity's aspect |
| Aspect subtree | Within an aspect's include tree | Include policy in aspect's includes |

Policies become first-class values placeable in `includes` and `excludes`:

```nix
den.aspects.igloo.includes = [ den.aspects.myBattery.policies.deliver-niri ];
den.aspects.server.excludes = [ den.policies.strict-firewall ];
```

### 9.3 meta.handleWith Unification

`meta.handleWith` (constraint handlers) and `meta.excludes` (sugar for exclude constraints) would be replaced by policy effects in `includes`:

```nix
# Current:
den.aspects.server.meta.excludes = [ den.aspects.desktop ];

# Proposed:
den.aspects.server.includes = [ (policy.exclude den.aspects.desktop) ];
```

A new `policy.filter` effect would replace the filter type in the constraint registry.

### 9.4 Open Questions

1. Policy identity for `excludes` matching — function identity in Nix is referential (same thunk = same policy); whether this suffices or explicit naming is needed
2. Ordering sensitivity between included policies and included aspects
3. Whether entity-instance excludes can override global policies
4. Migration path for `meta.handleWith` users with custom handler types

### 9.5 Dependency

This work depends on provides removal — the constraint registry simplification should happen first to reduce the surface area.

---

## 9. Planned Redesign: Unified Resolve Effects

**Spec:** `design/unified-resolve-effects.md`

### Changes to Policy Dispatch

- **Enrichment drain:** `drainEnrichmentDeferred` (explicit function call inside iterate's scope.provide) is replaced by `enterScope` + automatic `scope-widened` handler. The iterate loop uses `enterScope` instead of raw `scope.provide` for enrichment widen — drain fires automatically at scope entry.

- **installPolicies entry:** Unchanged externally. Internally, it calls the iterate loop which uses `enterScope`.

- **Late dispatch pass:** Unchanged. Uses raw `scope.provide` (no drain needed — late dispatch only emits effects, doesn't create new deferrable aspects).

### File Changes

- `policy/iterate.nix`: `drainEnrichmentDeferred` deleted. `iterate`'s widen branch uses `enterScope` instead of `scope.provide + explicit drain`.
- `policy/default.nix`: `installPolicies` unchanged (entry point semantics preserved).
- `policy/schema.nix`: `lateDispatchPass` unchanged (no drain involved).
