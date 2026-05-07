# Policy System

Policies are the central mechanism for topology construction, context enrichment, and content routing in Den. A policy is a plain Nix function that receives entity context and returns a list of typed effects. The function's parameter signature determines when it fires: the pipeline inspects `builtins.functionArgs` and only calls the policy when every required argument is present in the resolve context. This signature-based dispatch, combined with fixed-point enrichment iteration, allows policies to express complex conditional topology without imperative control flow.

## Policy Shape

A policy is a function from context attrset to a list of effects:

```nix
den.policies.<name> = { host, user, ... }: [
  (policy.resolve { ... })
  (policy.include den.aspects.foo)
];
```

The NixOS module system merges policy definitions through `policyRegistryType` (`nix/lib/aspects/policy-type.nix`), a `lazyAttrsOf policyFnType`. The merge coerces each definition into a `{ __isPolicy, name, fn }` record:

- Raw functions become `{ __isPolicy = true; name = "<attr-name>"; fn = <the-function>; }`.
- Attrsets with `__isPolicy = true` are passed through, with `name` overwritten to match the attribute name.
- Anything else throws.

A policy may return a single effect or a list; dispatch normalizes both to a list. Returning `[]` is valid and means the policy produced no effects for the current context.

## Effect Types

All constructors live in `nix/lib/policy-effects.nix` and are exposed as `den.lib.policy.*`. Each returns an attrset with a `__policyEffect` string discriminator. The dispatch validates that every returned effect has a known `__policyEffect` value.

| Constructor | `__policyEffect` | Purpose |
|---|---|---|
| `policy.resolve bindings` | `"resolve"` | Isolated fan-out / enrichment |
| `policy.resolve.shared bindings` | `"resolve"` | Shared fan-out (`__shared = true`) |
| `policy.resolve.to kind bindings` | `"resolve"` | Explicit target entity kind |
| `policy.resolve.shared.to kind bindings` | `"resolve"` | Shared + explicit target |
| `policy.resolve.withIncludes includes bindings` | `"resolve"` | Per-resolve scoped includes |
| `policy.resolve.to.withIncludes kind includes bindings` | `"resolve"` | Target + per-resolve includes |
| `policy.resolve.shared.withIncludes includes bindings` | `"resolve"` | Shared + per-resolve includes |
| `policy.include aspect` | `"include"` | Inject aspect into current scope |
| `policy.exclude aspect` | `"exclude"` | Remove aspect from subtree |
| `policy.route spec` | `"route"` | Route class content to a target class |
| `policy.instantiate spec` | `"instantiate"` | Request post-pipeline instantiation |
| `policy.provide spec` | `"provide"` | Deliver module directly into a class |
| `policy.pipe.from name stages` | `"pipe"` | Attach transform stages to a pipe |

### resolve

Creates a new context scope. Classification against `schemaEntityKinds` (entity kinds where `isEntity = true` in `den.schema`) determines the effect's role:

- **Schema resolve:** All binding keys match schema entity kinds, or `__targetKind` is set. Creates a child entity via `resolve-schema-entity`. See [entity-resolution.md](entity-resolution.md).
- **Enrichment resolve:** No binding keys match schema entity kinds. Merges bindings into the current scope's context. No child entity is created; subsequent policy iterations and class modules see the new keys.
- **Mixed resolve:** Some keys are schema, some are not. The resolve is split: schema keys create a child entity, non-schema keys enrich the current context.

`policy.resolve {}` (empty bindings) is filtered out as a no-op.

`resolve.shared` sets `__shared = true`, causing sibling scopes to share context rather than getting isolated partitions. Used by `host-to-users` so all users share the host scope.

`resolve.to kind` sets `__targetKind`, overriding the default kind derivation from binding keys. Used for non-schema routing (e.g., `resolve.to "flake-system" { inherit system; }`).

`resolve.withIncludes includes bindings` carries an `includes` list on the resolve effect. These aspects are emitted inside the child scope's resolution.

### include

Injects an aspect into the current resolution context:

```nix
policy.include den.aspects.sudo
# => { __policyEffect = "include"; value = <aspect>; }
```

Accepts aspect references and inline attrsets (coerced to anonymous aspects). When the include value has no `name` attribute, it is tagged with `name = "<policy:${policyName}>"` to prevent identity dedup collisions across distinct policy outputs. Multiple includes from the same policy get an `[${idx}]` suffix.

### exclude

Removes an aspect from the current resolution subtree:

```nix
policy.exclude den.aspects.desktop
# => { __policyEffect = "exclude"; value = <aspect>; }
```

Emitted via `register-constraint` with `type = "exclude"`, `scope = "subtree"`. The identity is derived from the aspect's path key. Excludes propagate into child scopes. See [constraint-system.md](constraint-system.md).

### route

Routes class content from one scope partition into a target class:

```nix
policy.route {
  fromClass = "packages";
  intoClass = "flake";
  path = [ "flake" "packages" system ];
  adaptArgs = _: { pkgs = inputs.nixpkgs.legacyPackages.${system}; };
}
```

Emitted via `register-route`. The route spec is processed post-pipeline by `applyForwardSpecs` / `wrapRouteModules`.

### instantiate

Requests post-pipeline instantiation of an entity's class content. The entity carries `instantiate`, `intoAttr`, and `mainModule` metadata:

```nix
policy.instantiate hostEntity
```

Emitted via `register-instantiate`. Used by flake policies to wire OS and HM outputs (`to-os-outputs`, `to-hm-outputs`).

### provide

Delivers a module directly into a target class, bypassing the aspect tree walk:

```nix
policy.provide { class = "nixos"; module = myModule; }
```

Emitted via `register-provide`. Unlike `policy.include` (which walks content through the tree and can create duplicates when content matches routes), `policy.provide` bypasses the tree entirely.

### pipe.from

Attaches transform stages to a named pipe. See [pipes-and-quirks.md](pipes-and-quirks.md) for the pipe system.

```nix
policy.pipe.from "my-pipe" [
  (policy.pipe.filter (item: item.enable or false))
  (policy.pipe.transform (item: item // { processed = true; }))
]
```

Available pipe sub-stage constructors:

| Constructor | `__pipeStage` | Purpose |
|---|---|---|
| `pipe.filter pred` | `"filter"` | Filter items by predicate |
| `pipe.transform fn` | `"transform"` | Transform each item |
| `pipe.fold fn init` | `"fold"` | Fold items into accumulator |
| `pipe.append value` | `"append"` | Append a static value |
| `pipe.for fn` | `"for"` | Map items with a function |
| `pipe.withProvenance` | `"withProvenance"` | Attach provenance metadata |
| `pipe.to aspects` | `"to"` | Route pipe output to aspects |
| `pipe.expose` | `"expose"` | Expose pipe as flake output |
| `pipe.collect pred` | `"collect"` | Collect items matching predicate |

## Dispatch

### Entry Point

`installPolicies` (`nix/lib/aspects/fx/policy/default.nix`) fires when an aspect has `__entityKind` set (i.e., the aspect represents a schema entity boundary).

**Dedup:** Each entity scope dispatches policies at most once. The dispatch is keyed by `"${entityKind}@${currentScope}"` and tracked in `state.dispatchedPolicies`. If already dispatched, returns immediately via `fx.pure []`.

**Context assembly:** The resolve context is built from two sources, merged with later sources taking precedence:

1. `state.scopeContexts.${currentScope}` -- scope context from prior enrichment
2. `ctxFromHandlers(aspect.__scopeHandlers)` -- handler-derived context from the aspect's lexical scope

The merged context is augmented with `__entityKind = entityKind` before dispatch.

### resolveArgsSatisfied

`resolveArgsSatisfied` (`nix/lib/synthesize-policies.nix`) checks whether a policy's required arguments are present in context:

```nix
resolveArgsSatisfied = policy: ctx:
  if !lib.isFunction policy then false
  else
    let
      fargs = lib.functionArgs policy;
      requiredArgs = builtins.filter (k: !fargs.${k}) (builtins.attrNames fargs);
    in
    builtins.all (k: ctx ? ${k}) requiredArgs;
```

- `_:` or `{ ... }:` fires unconditionally (no required args).
- `{ host, ... }:` fires only when `host` is in context.
- `{ host, isNixos ? false, ... }:` fires when `host` is in context; `isNixos` is optional.

### Policy Sources

Policies enter the dispatch system from two sources:

| Source | How activated | Scope |
|---|---|---|
| **Schema/default includes** | Policy referenced in `den.schema.*.includes` or `den.default.includes` | Dispatched at entity boundaries matching the schema kind |
| **Aspect-scoped** | Policy declared in `aspect.policies.*`, registered via `register-aspect-policy` during tree walk | Dispatched at the scope where the owning aspect was included |

`den.policies` is a named registry only — it does NOT auto-dispatch. Policies must be explicitly referenced from an activation site.

The `dispatch-policies` handler (`nix/lib/aspects/fx/handlers/dispatch-policies.nix`) receives both sets, filters out any policies excluded by the constraint registry, then calls `mkDispatch` which runs `dispatchAspect` against the combined set.

For each policy, `dispatchAspect` checks `resolveArgsSatisfied` on the `.fn` field and that the policy name hasn't already been fired. Satisfied policies are called with the resolve context and their effects are validated (must have valid `__policyEffect` tags) and classified.

### Enrichment Convergence

The `iterate` function (`nix/lib/aspects/fx/policy/iterate.nix`) implements enrichment-to-fixpoint:

1. **Dispatch** all policies against current context (via `dispatch-policies` effect).
2. **Collect** enrichment keys (non-schema resolve bindings).
3. If **new enrichment keys** appeared (keys not in the accumulated enrichment):
   - Merge new enrichment into accumulated context.
   - Install enrichment as scoped handlers via `enterScope(constantHandler(combinedEnrichment))`.
   - Send `widen-context` effect to update scope state.
   - **Re-dispatch** (go to step 1 with widened context, increment iteration counter).
4. If **no new enrichment keys** -- enrichment is stable. Record fired policies and emit all accumulated effects via `emit-policy-effects`.

**Termination:** Iteration is capped at `maxPolicyIterations = 10`. Exceeding this throws an error naming the fired policies and accumulated enrichment keys.

**Key-monotonic invariant:** Enrichment convergence checks new keys only. Value changes to existing keys do not trigger re-dispatch. This guarantees termination as long as the total number of enrichment keys is finite.

**Commutativity:** Enrichment flows downward only. A child scope's enrichment does not affect parent or sibling scopes. Multiple enrichment resolves merge additively; later values shadow earlier ones for the same key.

## Effect Classification

`classifyPolicyResult` (`nix/lib/aspects/fx/policy/classify.nix`) sorts each policy's effects into buckets:

| Bucket | Contents |
|---|---|
| `schemaEffects` | Resolve effects with schema keys or explicit `__targetKind` |
| `mergedEnrichment` | All non-schema resolve bindings merged into one attrset |
| `includeEffects` | `__policyEffect = "include"` effects |
| `excludeEffects` | `__policyEffect = "exclude"` effects |
| `routeEffects` | `__policyEffect = "route"` effects |
| `instantiateEffects` | `__policyEffect = "instantiate"` effects |
| `provideEffects` | `__policyEffect = "provide"` effects |
| `pipeEffects` | `__policyEffect = "pipe"` effects |

**Cross-provider tagging:** When a policy produces both schema resolve effects (with `__targetKind`) and include effects, it is tagged `isCrossProvider = true`. Include effects are attached to the schema effects as `__policyIncludes` rather than emitted independently. This enables patterns where a policy routes content to a target entity and includes aspects that should be walked within that target's scope.

**Effect emission order** (after enrichment stabilizes):

1. Excludes via `register-constraint`
2. Routes, instantiates, provides via their respective handlers
3. Pipe effects via `register-pipe-effect`
4. If schema resolves exist: `processSchemaResolves` handles entity creation; cross-provider includes go with schema resolves, independent includes are emitted separately
5. If no schema resolves: all includes emitted via `emit-include`

## Scoping

Policies are registered at three sites with different activation semantics:

### den.policies (Named Registry)

```nix
den.policies.my-policy = { host, ... }: [ ... ];
```

`den.policies` is a typed attrset providing a named namespace for policy functions. Entries are coerced to `{ __isPolicy, name, fn }` records via `policyRegistryType`. **Declaring a policy here does NOT activate it.** Policies only fire when explicitly referenced from an activation site (see below).

### Activation: den.schema.*.includes / den.default.includes

```nix
den.schema.host.includes = [ den.policies.host-to-users ];
den.schema.flake-system.includes = [ den.policies.to-os-outputs ];
den.default.includes = [ den.policies.host-guards ];
```

Schema and default includes are the primary activation paths. Placing a policy reference in `den.schema.*.includes` causes it to be dispatched when entities of that kind are resolved. `den.default.includes` activates a policy for all entity scopes.

### aspect.policies (Aspect-scoped)

```nix
den.aspects.myBattery = {
  policies.deliver-niri = { host, user, ... }: [ ... ];
};
```

Aspects can declare policies in their `.policies` attribute. During the tree walk, `emitAspectPolicies` registers each entry via `register-aspect-policy` with name `"${aspectName}/${policyName}"`, the policy function, and the owning aspect's identity. These are stored in `state.scopedAspectPolicies` keyed by scope and in `state.flatAspectPolicies` for global access.

Aspect policies are scope-local: they fire at the scope where their owning aspect was included. A **late dispatch pass** corrects ordering dependencies when fan-out creates multiple siblings -- aspect policies registered by later siblings are re-dispatched against earlier siblings that may not have seen them.

## Combinators

### policy.for

Wraps a policy (or list of policies) to fire only for a specific entity, using `id_hash` for robust identity matching:

```nix
den.policies.igloo-only = policy.for igloo (
  { host, ... }: [ (policy.include den.aspects.gaming) ]
);
```

The wrapper checks `ctx.${ctx.__entityKind}.id_hash == entity.id_hash`. Returns `[]` on mismatch. Preserves the inner policy's `name` and `__isPolicy` tag. Raw functions are auto-wrapped with `name = "<inline>"`.

### policy.when

Wraps a policy (or list of policies) to fire only when a predicate returns true:

```nix
den.policies.linux-only = policy.when (ctx: ctx.isNixos or false) (
  { host, ... }: [ (policy.include den.aspects.linux-desktop) ]
);
```

Returns `[]` when the predicate returns false. Preserves `name` and `__isPolicy`. Composes with `policy.for`:

```nix
policy.when pred (policy.for entity myPolicy)
```

Both combinators accept either a single policy or a list of policies. When given a list, each element is individually wrapped.

## pipelineOnly

Not an effect -- a value wrapper. Tags a value with `collisionPolicy = "class-wins"` so that when the value reaches a class module that also receives the same argument from the module system (e.g., NixOS provides `lib`), the pipeline value wins silently:

```nix
policy.pipelineOnly value
```

For attrsets: `value // { collisionPolicy = "class-wins"; }`. For functions: `{ __functor = _: value; collisionPolicy = "class-wins"; }`. Asserts the input is a function or attrset.

Used by `den.provides.flake-scope` to inject `lib`, `inputs`, `den` into the pipeline without colliding with NixOS's own `lib`.

## Built-in Policies

### core.nix

**`host-to-users`** -- the core traversal policy. Registered globally and activated via `den.schema.host.includes`:

```nix
den.policies.host-to-users = { host, ... }:
  map (user: resolve.shared { inherit user; }) (lib.attrValues host.users);

den.schema.host.includes = [ den.policies.host-to-users ];
```

When resolving a host entity, fans out to each user with `resolve.shared` so users share the host scope.

### flake.nix

Flake output wiring policies, all activated via schema includes:

| Policy | Activated at | Effect |
|---|---|---|
| `to-systems` | `den.schema.flake.includes` | Fan-out per system via `resolve.to "flake-system"` |
| `to-os-outputs` | `den.schema.flake-system.includes` | `resolve.to "host"` + `policy.instantiate` for each host with `intoAttr` |
| `to-hm-outputs` | `den.schema.flake-system.includes` | `resolve.to "home"` + `policy.instantiate` for each home with `intoAttr` |
| `to-packages` | `den.schema.flake-system.includes` | `policy.route` from packages class into flake at `["flake" "packages" system]` |
| `to-apps` | `den.schema.flake-system.includes` | Same pattern for apps |
| `to-checks` | `den.schema.flake-system.includes` | Same pattern for checks |
| `to-devShells` | `den.schema.flake-system.includes` | Same pattern for devShells |
| `to-legacyPackages` | `den.schema.flake-system.includes` | Same pattern for legacyPackages |

The `to-${output}` policies conditionally fire based on whether the flake output option exists (`has-flake-output`). Each uses `policy.route` with `adaptArgs` to inject `pkgs` from `nixpkgs.legacyPackages.${system}`.

flake.nix also registers system output names (`packages`, `apps`, `checks`, `devShells`, `legacyPackages`) as classes via `den.classes`.

## Invariants

1. **Policies are pure functions.** A policy receives context and returns effects. No side effects, no state mutation, no dynamic registration within a policy body.

2. **Signature-based dispatch.** Firing is determined solely by `builtins.functionArgs` against the resolve context. No explicit enable/disable flags.

3. **Enrichment converges.** The key-monotonic invariant (only new keys trigger re-dispatch) plus the `maxPolicyIterations = 10` cap guarantee termination. Cycles throw with diagnostic information.

4. **No duplicate dispatch.** Each `"${entityKind}@${scope}"` combination dispatches at most once, tracked in `state.dispatchedPolicies`.

5. **Effect validation.** Every effect returned by a policy must have a valid `__policyEffect` tag from the set `{resolve, include, exclude, route, instantiate, provide, pipe}`. Invalid effects throw with the policy name and index.

6. **Fired tracking.** Policies that produce effects are tracked per-scope in `state.firedPolicyNames` and per-iteration in the `firedPolicies` accumulator. A policy fires at most once per dispatch key.

7. **Late dispatch correction.** Fan-out siblings get a late dispatch pass to ensure aspect policies registered by later siblings are visible to earlier ones. Schema effects from late dispatch are intentionally not processed to prevent duplicate entity creation.

## Key Files

| File | Role |
|---|---|
| `nix/lib/policy-effects.nix` | All effect constructors and combinators |
| `nix/lib/synthesize-policies.nix` | `resolveArgsSatisfied` |
| `nix/lib/aspects/policy-type.nix` | `policyRegistryType` (NixOS module type) |
| `nix/lib/aspects/fx/policy/default.nix` | `installPolicies` entry point |
| `nix/lib/aspects/fx/policy/dispatch.nix` | `mkDispatch`, `dispatchAspect` |
| `nix/lib/aspects/fx/policy/classify.nix` | `classifyPolicyResult`, cross-provider tagging |
| `nix/lib/aspects/fx/policy/iterate.nix` | Fixed-point enrichment loop |
| `nix/lib/aspects/fx/policy/schema.nix` | Schema resolve processing, late dispatch pass |
| `nix/lib/aspects/fx/policy/apply.nix` | Effect emission (`policyEmitIncludes`, `policyEmitExcludes`, etc.) |
| `nix/lib/aspects/fx/handlers/dispatch-policies.nix` | Dispatch handler (constraint filtering) |
| `nix/lib/aspects/fx/handlers/emit-policy-effects.nix` | Final emission handler |
| `nix/lib/aspects/fx/handlers/policy.nix` | `register-aspect-policy` handler |
| `modules/policies/core.nix` | `host-to-users` policy |
| `modules/policies/flake.nix` | Flake output policies |
