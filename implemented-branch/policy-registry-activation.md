# Policy Registry/Activation Split — Design Spec

**Date:** 2026-05-05
**Branch:** feat/fx-pipeline
**Status:** Implemented (713/713 tests)
**Date completed:** 2026-05-05

---

## 1. Problem

Policy declaration and activation are coupled. Defining a policy at any registration site immediately activates it:

| Site | Behavior |
|------|----------|
| `den.policies.foo = fn` | Active globally, fires at every matching entity |
| `den.schema.host.policies.foo = fn` | Active at all host entities |
| `aspect.policies.foo = fn` | Active wherever the aspect is included |

There is no way to:
- Define a policy without activating it
- Activate a policy selectively (only in a specific subtree)
- Deactivate a policy for a subtree (exclude it)
- Reference a policy by identity for composition or override

Policies lack the lifecycle that aspects have: **define → reference → include → activate with context.**

## 2. Design Principles

1. **Policies are first-class values** — addressable, with stable identity, includable and excludable just like aspects.
2. **`den.policies.*` is a registry** — defining a policy does not activate it.
3. **Activation is via `includes`** — the same mechanism used for aspects.
4. **Excludes are authoritative** — a parent scope's exclude cannot be overridden by child scopes.
5. **Downward cascade** — policies cascade from their inclusion point to all child entities and scopes.
6. **Upward flow only through explicit exports** — no implicit upward propagation. Cross-entity upward data flows through the future traits/exports system.
7. **Consistent API across all sites** — no special cases for global, schema, or aspect policies.
8. **Hard break, not migration** — the branch hasn't shipped. Existing tests are translated to the new API. No compatibility shims.

## 3. Policy Values

### 3.1 Registry Definition

`den.policies.*` is a registry of named policy values. The module system wraps each entry with identity metadata:

```nix
# What the user writes:
den.policies.strict-firewall = { host, ... }: [
  (policy.provide { class = host.class; module.networking.firewall.enable = true; })
  (policy.exclude den.aspects.desktop)
];

# What the module system produces (after policyType processing):
den.policies.strict-firewall = {
  __isPolicy = true;
  name = "strict-firewall";
  fn = { host, ... }: [ ... ];
};
```

The `__isPolicy` tag distinguishes policy values from parametric aspects in `includes` lists. The `name` field provides stable identity for matching in `excludes`.

### 3.2 Aspect-Owned Policies

Aspects can define policies in their `policies.*` namespace. These are logically owned by the aspect but not automatically activated:

```nix
den.aspects.igloo = {
  nixos.services.openssh.enable = true;

  # Registry — defines policies scoped to this aspect
  policies.to-alice = { host, user, ... }:
    lib.optional (user.name == "alice") (
      policy.include { homeManager.programs.tmux.enable = true; }
    );

  # Activation — explicit opt-in
  includes = [
    den.aspects.igloo.policies.to-alice
  ];
};
```

Aspect policies are addressable as `den.aspects.<aspect>.policies.<name>` and carry identity `"<aspect>/<name>"`.

### 3.3 Discriminator

The include handler distinguishes policies from aspects via the `__isPolicy` tag:

- `__isPolicy = true` → route to `register-aspect-policy` (register for dispatch at entity boundaries)
- No `__isPolicy` → treat as aspect, walk for class content via `emit-include`

This avoids collision with parametric aspects (also functions) in the same `includes` list.

## 4. Activation

### 4.1 Via `includes`

Policies are activated by placing them in `includes` at any level:

```nix
# Global — active for all entities (via den.default)
den.default.includes = [ den.policies.strict-firewall ];

# Per entity kind — active for all hosts
den.schema.host.includes = [ den.policies.host-to-users ];

# Per aspect — active in this aspect's subtree
den.aspects.igloo.includes = [ den.policies.strict-firewall ];

# Per aspect, parametric — active when context satisfies
den.aspects.monitoring = { hasPostgres, ... }: {
  includes = [ den.policies.postgres-metrics ];
};
```

### 4.2 Cascade Semantics

A policy included at scope S is active at S and all descendant scopes (child entities, child aspects). This mirrors how aspect includes cascade.

```
den.schema.host.includes = [ P ]
  → P fires at every host entity
  → P cascades to user entities within each host
  → P cascades to home entities within each user

den.aspects.igloo.includes = [ P ]
  → P fires within igloo's resolution subtree
  → P cascades to igloo's users (tux, alice, etc.)
```

### 4.3 Schema-Level Activation

`den.schema.*.policies` is removed. Schema-level policies move to `den.schema.*.includes`:

```nix
# Before (auto-activated):
den.schema.host.policies.host-to-users = { host, ... }:
  map (user: resolve.shared { inherit user; }) (lib.attrValues host.users);

# After (explicit activation):
den.schema.host.includes = [ den.policies.host-to-users ];
```

The policy definition stays in `den.policies.*` (or in a module that sets `den.policies.*`). The schema entry only activates it.

Note: `den.schema.*.includes` is currently extracted by convention (attribute presence check in the `schemaEntryType` merge in `modules/options.nix`), not as a declared typed option. This is sufficient for the activation pattern but a proper `includes` option on the schema entry type could be added for type safety.

## 5. Deactivation

### 5.1 Via `excludes`

Policies can be excluded from a subtree:

```nix
den.aspects.igloo = {
  includes = [ den.policies.strict-firewall ];
  excludes = [ den.policies.desktop-polish ];
};
```

### 5.2 Authoritative Excludes

Excludes are authoritative: a parent scope's exclude **cannot** be overridden by a child scope's include.

```nix
den.aspects.igloo.excludes = [ den.policies.desktop-polish ];

# alice is a user within igloo
den.aspects.alice.includes = [ den.policies.desktop-polish ];
# ↑ NO EFFECT — igloo's exclude is authoritative
```

This provides predictable security guarantees: if a host excludes a policy, no user within that host can re-enable it.

### 5.3 Implementation

Policy excludes use the existing constraint registry (`register-constraint`) with the policy's identity as the constraint key. Before dispatching a policy, the dispatch handler checks `check-constraint` against the policy's identity. If excluded, the policy is skipped.

This is the same mechanism used for aspect excludes (`policy.exclude den.aspects.desktop`), extended to policy values.

## 6. Entity Filtering

Activation via `includes` controls **which subtree** a policy fires in. Entity filtering controls **which entity instances** within that subtree the policy applies to. Together they replace the manual `lib.optional (user.name == "alice")` guard pattern.

### 6.1 `policy.for` — Entity Instance Targeting

Targets a specific entity by identity (matched via `hash_id`). Accepts a single policy or a list of policies:

```nix
# Single policy:
den.aspects.igloo.includes = [
  (policy.for alice den.policies.tmux-config)
];

# List of policies — all scoped to the same entity:
den.aspects.igloo.includes = [
  (policy.for alice [
    den.policies.tmux-config
    den.policies.desktop-polish
    den.policies.gaming-mode
  ])
];
```

`policy.for entity policies` wraps one or more policies to only fire when the current context contains a matching entity (by `hash_id`). The pipeline can skip dispatch entirely rather than calling the policy and getting back `[]`.

Multiple entity targets compose naturally:

```nix
den.aspects.igloo.includes = [
  (policy.for alice den.policies.tmux-config)
  (policy.for bob den.policies.tmux-config)
];
```

### 6.2 `policy.when` — Predicate-Based Filtering

Filters by entity properties at the activation site. Accepts a single policy or a list of policies:

```nix
# Sudo config for users in wheel group
den.policies.sudo-config = { user, ... }: [
  (policy.provide {
    class = "nixos";
    module.security.sudo.extraRules = [{
      users = [ user.userName ];
      commands = [{ command = "ALL"; options = [ "NOPASSWD" ]; }];
    }];
  })
];

# Single policy, filtered:
den.schema.user.includes = [
  (policy.when ({ user, ... }: builtins.elem "wheel" (user.groups or []))
    den.policies.sudo-config)
];

# List of policies, same predicate:
den.schema.user.includes = [
  (policy.when ({ user, ... }: builtins.elem "wheel" (user.groups or [])) [
    den.policies.sudo-config
    den.policies.admin-shell
  ])
];
```

`policy.when predicate policies` wraps one or more policies to only fire when the predicate returns `true` against the current resolve context. The predicate has the same signature as a policy function — it receives the entity context.

### 6.3 Composition

`policy.for` and `policy.when` are composable — they wrap policy values (single or list), producing new policy values:

```nix
# Only alice, and only when on a NixOS host
den.aspects.igloo.includes = [
  (policy.when ({ host, ... }: host.class == "nixos")
    (policy.for alice [
      den.policies.tmux-config
      den.policies.desktop-polish
    ]))
];
```

They also compose with parametric activation (the includes list itself being parametric):

```nix
den.aspects.server = { host, ... }: {
  includes = [
    # All wheel users on this host get sudo + admin shell
    (policy.when ({ user, ... }: builtins.elem "wheel" (user.groups or [])) [
      den.policies.sudo-config
      den.policies.admin-shell
    ])
    # All users get monitoring shell aliases
    den.policies.monitoring-aliases
  ];
};
```

### 6.4 Wrapper Identity Semantics

`policy.for` and `policy.when` produce **per-inner-policy wrappers**, not a single aggregate. When given a list, each inner policy gets its own wrapper preserving its identity:

```nix
policy.when pred [P1, P2]
# produces:
# [ { __isPolicy = true; name = P1.name; fn = <pred-guarded P1.fn>; }
#   { __isPolicy = true; name = P2.name; fn = <pred-guarded P2.fn>; } ]
```

This means:
- **Identity is preserved** — each wrapper carries the inner policy's `name`. `excludes = [ P1 ]` matches `P1` whether it appears bare or inside a `policy.when` wrapper.
- **Excludes work through wrappers** — the constraint system matches on string identity (`name`), not value equality. `register-constraint` receives the policy's `name` string; `check-constraint` in dispatch compares against it.
- **Single policy input is normalized** — `policy.when pred P` is equivalent to `policy.when pred [P]`, producing a single wrapped value (not a list).
- **The wrapper composes the guard into `fn`** — `policy.for entity P` produces `{ __isPolicy = true; name = P.name; fn = ctx: if entity.id_hash == (ctx.${entityKind}.id_hash or null) then P.fn ctx else []; }`.

### 6.5 Design Notes

- **Guards belong at activation, not in policy bodies.** The policy function stays generic and reusable. The activation site decides who it applies to. This inverts the current pattern where every targeted policy carries its own filter logic.
- **`policy.for` uses `id_hash` equality** (the field on entity records, `modules/options.nix` lines 87-117), not name comparison. This is robust against entity name collisions across different entity kinds.
- **`policy.when` predicates receive the full resolve context** — the same attrset passed to policy functions. This means predicates can filter on any context key, including enrichment values from prior iterations.
- **The pipeline can optimize**: `policy.for` and `policy.when` wrappers can be checked before calling the policy function, avoiding unnecessary evaluation of the policy body when the filter doesn't match.

## 7. Identity

### 7.1 Identity Sources

| Policy origin | Identity |
|---------------|----------|
| `den.policies.strict-firewall` | `"strict-firewall"` |
| `den.aspects.igloo.policies.to-alice` | `"igloo/to-alice"` |

Identity is derived from the registry path, same as aspect identity is derived from `den.aspects.*` paths.

### 7.2 Identity Uses

- **Dedup**: a policy with the same identity registered at the same scope is dispatched once
- **Excludes matching**: `excludes = [ den.policies.P ]` matches on P's identity
- **Tracing**: policy identity appears in trace output for debugging
- **Override**: a policy with the same identity at a deeper scope shadows the parent (future consideration)

## 8. Invariant: Policies Don't Apply to Their Own Outputs

When policy P produces a resolve effect that creates child entity E, **P is excluded from dispatch at E's scope.** A policy's output is not its input.

This prevents self-referential cycles: a policy that transforms its context bindings (`v = "${v}!"`) would otherwise fire at its own child entity with the mutated bindings, creating unbounded recursion with unique contexts that `ctx-seen` cannot deduplicate.

### 8.1 Implementation

The source policy name is threaded through the resolve chain:

1. `classifyPolicyResult` preserves `policyName` on classified results
2. `collectSchemaEffects` tags each schema effect with `__sourcePolicyName = policyName`
3. `processSingleResolve` passes `sourcePolicyName` to `resolve-schema-entity`
4. `resolve-schema-entity` passes it to `push-scope`
5. `push-scope` stores `state.scopeSourcePolicy.${newScopeId} = sourcePolicyName`
6. `installPolicies` reads `scopeSourcePolicy` for the current scope and seeds `firedPolicies` with it, so `dispatchAspect` skips the policy

### 8.2 Scope

This invariant applies per-resolve-effect. If policy P produces multiple resolve effects (e.g., `host-to-users` creates multiple users), P is excluded from each child scope. Other policies are unaffected — they fire normally at child entities.

Policies that DON'T produce resolve effects (only include, route, provide, exclude, or enrichment) are unaffected by this mechanism.

---

## 9. Policy Include Dedup Across Scopes

Policies fire commutatively — wherever their args are satisfied. When a policy fires at multiple entity scopes in a cascade, its include effects must not produce duplicate class emissions.

### 9.1 Identity Tagging

Policy-emitted includes carry stable identity tags derived from the source policy name + positional index:

- `<policy:host-to-users:0>` — first include from `host-to-users`
- `<policy:host-to-users:1>` — second include from `host-to-users`

This applies to both paths:
- `__policyIncludes` (cross-provider includes attached to schema effects) — tagged in `classify.nix`
- `policyEmitIncludes` (independent includes emitted directly) — tagged in `apply.nix`

Named aspects in policy includes (e.g., `policy.include den.aspects.foo`) keep their own identity and dedup through the normal aspect path.

### 9.2 Global Dedup

`check-dedup` recognizes `<policy:*>` names and uses them as global dedup keys (no scope prefix). This means the same policy firing at greet-scope and at yell-scope produces its anonymous include only once — the second attempt is caught as a duplicate regardless of scope.

### 9.3 Independent vs Cross-Provider Includes

When multiple policies fire at the same entity scope, `emit-policy-effects` separates their include effects:

- **Cross-provider includes** — from policies that ALSO produced schema resolve effects. These are attached to the schema entities via `__policyIncludes` and resolved within the child entity's tree walk.
- **Independent includes** — from policies that ONLY produced include/route/provide effects. These are emitted directly via `policyEmitIncludes` at the current scope.

The separation uses `__sourcePolicyName` matching: an include effect is cross-provider only if a schema effect from the SAME policy exists in the dispatch results. Without this, unrelated policies' includes would be incorrectly bundled into unrelated schema entity resolution.

---

## 10. Interaction with Enrichment Loop

### 9.1 No New Machinery Needed

The existing iterate loop handles policies-as-includable-values without structural changes. The phase sequence per aspect in `resolve-children.nix` is:

1. `emitAspectPolicies` — registers `aspect.policies.*` entries (now a no-op unless provides-derived; see Section 10.3)
2. `emitIncludes` — walks `aspect.includes`, encounters `__isPolicy` values → `register-aspect-policy`
3. `installPolicies` at entity boundary → dispatches all registered policies
4. Policy emits enrichment → widen context → `enterScope` → `scope-widened` → drain
5. Drain resolves deferred aspects → those aspects may contain policy includes → register new policies
6. Next iteration dispatches newly registered policies
7. Repeat until stable (bounded by `maxPolicyIterations = 10`)

### 9.2 Cascading Enrichment

Chained enrichment across drain boundaries works naturally:

```nix
den.policies.detect-features = { host, ... }: [
  (policy.resolve { hasPostgres = host.services ? postgres; })
];

den.aspects.monitoring = { hasPostgres, ... }: {
  includes = [ den.policies.postgres-metrics ];
};

den.policies.postgres-metrics = { host, ... }: [
  (policy.resolve { metricsPort = 9187; })
];

den.aspects.firewall-rules = { metricsPort, ... }: {
  includes = [ den.policies.open-port ];
};
```

Round 1: `detect-features` fires → enrichment `{ hasPostgres }`
Round 2: drain → `monitoring` resolves → `postgres-metrics` registers → fires → enrichment `{ metricsPort }`
Round 3: drain → `firewall-rules` resolves → `open-port` registers → fires → stable

Each `go` iteration in `iterate.nix` does: dispatch → widen → enterScope (drain → resolve → register) → next iteration catches new policies.

## 11. Concrete API Translation

### 11.1 Core Traversal Policy

```nix
# Before:
den.schema.host.policies.host-to-users = { host, ... }:
  map (user: resolve.shared { inherit user; }) (lib.attrValues host.users);

# After:
den.policies.host-to-users = { host, ... }:
  map (user: resolve.shared { inherit user; }) (lib.attrValues host.users);

den.schema.host.includes = [ den.policies.host-to-users ];
```

### 11.2 Flake Output Policies

```nix
# Before:
den.schema.flake.policies.to-systems = _:
  map (system: resolve.to "flake-system" { inherit system; }) den.systems;

# After:
den.policies.to-systems = _:
  map (system: resolve.to "flake-system" { inherit system; }) den.systems;

den.schema.flake.includes = [ den.policies.to-systems ];
```

Same pattern for `to-os-outputs`, `to-hm-outputs`, `to-${output}`.

### 11.3 Aspect-Included Policies

```nix
# Before (auto-activated, manual guard):
den.aspects.igloo.policies.to-alice = { host, user, ... }:
  lib.optional (user.name == "alice") (
    policy.include { homeManager.programs.tmux.enable = true; }
  );

# After (explicit activation, entity filtering):
den.policies.tmux-config = { user, ... }: [
  (policy.include { homeManager.programs.tmux.enable = true; })
];

den.aspects.igloo.includes = [
  (policy.for alice den.policies.tmux-config)
];
```

The policy body no longer carries the guard — `policy.for` handles entity targeting at the activation site. The policy itself is generic and reusable.

### 11.4 Predicate-Guarded Policies

```nix
# Before (manual guard in policy body):
den.policies.sudo-config = { user, ... }:
  lib.optional (builtins.elem "wheel" (user.groups or [])) (
    policy.provide { class = "nixos"; module.security.sudo.extraRules = [...]; }
  );

# After (predicate at activation site):
den.policies.sudo-config = { user, ... }: [
  (policy.provide {
    class = "nixos";
    module.security.sudo.extraRules = [{
      users = [ user.userName ];
      commands = [{ command = "ALL"; options = [ "NOPASSWD" ]; }];
    }];
  })
];

den.schema.user.includes = [
  (policy.when ({ user, ... }: builtins.elem "wheel" (user.groups or []))
    den.policies.sudo-config)
];
```

### 11.5 Global Policies

```nix
# Before (auto-activated):
den.policies.os-to-host = { host, ... }:
  [ (policy.route { fromClass = "os"; intoClass = host.class; path = []; }) ];

# After (define + activate via den.default):
den.policies.os-to-host = { host, ... }:
  [ (policy.route { fromClass = "os"; intoClass = host.class; path = []; }) ];

den.default.includes = [ den.policies.os-to-host ];
```

## 12. Changes to Module System

### 12.1 New `policyType`

A new option type for `den.policies.*` that wraps function values with identity metadata:

```nix
policyType = lib.types.lazyAttrsOf (lib.types.mkOptionType {
  name = "policyFunction";
  check = lib.isFunction;
  merge = ...;  # produces { __isPolicy = true; name; fn; }
});
```

The type introspects `lib.functionArgs` from the wrapped function to support dispatch matching.

### 12.2 Remove `den.schema.*.policies`

The `policies` field is removed from the schema merge logic in `modules/options.nix`. Schema entries retain `includes` (which now carries both aspects and policies).

### 12.3 Aspect `policies.*` Becomes Registry-Only

The unconditional `register-aspect-policy` loop in `emitAspectPolicies` (`aspect/provide.nix` lines 136-144) is **deleted**. This loop currently auto-registers all `aspect.policies.*` entries — removing it is the core behavioral change that decouples declaration from activation.

After deletion, `emitAspectPolicies` retains only:
- Provides-derived policy registration (cross-entity `provides.to-hosts`/`provides.to-users` keys → `register-aspect-policy`). This is unaffected — provides-derived policies are discovered during the walk, not user-declared `policies.*` entries. The provides-cleanup spec modifies this path independently.
- Self-provide emission (`provides.${aspectName}` → `emit-include`).

The `aspect.policies.*` namespace remains as a sub-registry — a place to define policies owned by the aspect. Activation requires explicit inclusion via `aspect.includes`.

## 13. Pipeline Changes

### 13.1 Include Handler

The `__isPolicy` check is added to `processInclude` in `aspect/children.nix` (the function that processes individual include entries before dispatching them). This is where parametric aspects are already detected and deferred — the policy check fits naturally:

```
processInclude = child:
  if child.__isPolicy or false then
    fx.send "register-aspect-policy" {
      name = child.name;
      fn = child.fn;
      ownerIdentity = child.name;
    }
  else
    # existing aspect walk (dedupAndDispatch → emit-include)
    ...
```

Note: `resolve-children.nix` orchestrates the phase sequence (`emitAspectPolicies` → `emitIncludes` → `installPolicies`) but does not inspect individual include entries. The `__isPolicy` check lives in the include processing path, not the orchestrator.

### 13.2 Constraint Checking for Policies

Before dispatch, `dispatchDirect` and `dispatchAspect` check constraints:

```
for each registered policy:
  if check-constraint policy.name then skip (excluded)
  else if resolveArgsSatisfied policy.fn ctx then fire
```

### 13.3 State Changes

- Remove `den.schema.*.policies` from pipeline state construction
- `state.scopedAspectPolicies` unchanged — still keyed by scope, populated by `register-aspect-policy`
- `state.dispatchedPolicies` unchanged
- `state.firedPolicyNames` unchanged

### 13.4 `installPolicies` Source Change

Currently reads from three sources (`policy/default.nix` lines 95-97):

```nix
globalPolicies = den.policies or { };
schemaPolicies = (den.schema.${entityKind} or { }).policies or { };
allDirectPolicies = globalPolicies // schemaPolicies;
```

After: **delete all three lines.** All policy dispatch goes through `state.flatAspectPolicies` (populated by `register-aspect-policy` during tree walk). No direct reads from `den.policies` or `den.schema.*.policies`.

- **Global**: `den.policies` is a registry only. Policies from it are activated via `includes` (e.g., `den.default.includes`), which routes them through `processInclude` → `register-aspect-policy` during the tree walk. They appear in `state.flatAspectPolicies` by dispatch time.
- **Schema**: `den.schema.*.policies` is removed entirely. Schema-level activation uses `den.schema.*.includes`.
- **Aspect**: unchanged (registered via `register-aspect-policy` during tree walk).

`dispatchDirect` is also updated: since all policies are now `{ __isPolicy; name; fn; }` records (not raw functions), it calls `policy.fn resolveCtx` and checks `resolveArgsSatisfied policy.fn resolveCtx` rather than calling the policy directly.

## 14. `policy.resolve` is One Operation

`policy.resolve` adds bindings to context. Whether the pipeline treats those bindings as entity creation (scope partition) or context enrichment (widening) is determined by schema classification — not by the user. This is the correct abstraction:

- The user's intent is always "these bindings should be available downstream"
- The entity/enrichment boundary is fluid — today `isNixos` is enrichment, but in a traits-first world it could be a registered kind
- Splitting into `resolve` vs `enrich` would leak schema internals into the user API
- The policy grammar naturally extends to enriching, mutating, filtering, and shadowing context — all via `resolve`

No change needed to `policy.resolve`. The variant explosion (7 call signatures from `shared × targetKind × withIncludes`) is a separate usability concern addressed below.

### 14.1 Resolve Variant Cleanup

The 7 variants (`resolve`, `resolve.shared`, `resolve.to`, `resolve.shared.to`, `resolve.withIncludes`, `resolve.to.withIncludes`, `resolve.shared.withIncludes`) are a combinatorial product. Consider collapsing to fewer entry points — e.g., a single `resolve.with { shared = true; targetKind = "foo"; includes = [...]; } bindings` alongside the simple `resolve bindings` form. This is independent of the registry/activation split and can be addressed as a follow-up API cleanup.

## 15. Error Diagnostics

Fold these improvements into the implementation:

### 15.1 Effect Validation

Validate policy return values at the call site. Currently, malformed effects (non-attrsets, missing `__policyEffect`, wrong shape) are silently dropped in `classify.nix`. After: validate immediately after calling `policy.fn ctx` and throw with the policy name:

```
"den: policy '${name}' returned invalid effect at index ${i}: expected attrset with __policyEffect, got ${typeOf eff}"
```

### 15.2 Cycle Diagnostics

The enrichment cycle error (`iterate.nix` line 71) names only the entity kind. Include the fired policy names and current enrichment keys:

```
"den: enrichment cycle at ${entityKind} — fired: ${concatStringsSep ", " (attrNames firedPolicies)}, enrichment keys: ${concatStringsSep ", " (attrNames accEnrichment)}"
```

## 16. `pipelineOnly` Placement

`policy.pipelineOnly` is an internal collision-policy mechanism used only by `den.provides.flake-scope`. Move it out of the public `den.lib.policy.*` namespace into an internal namespace (e.g., `den.lib._internal.pipelineOnly` or inline it into `flake-scope.nix`).

## 17. Provides-Cleanup Interaction

Both this spec and `design/provides-cleanup.md` modify `emitAspectPolicies` in `aspect/provide.nix`:
- **This spec** removes the `policies.*` auto-registration loop (lines 136-144)
- **Provides-cleanup** folds self-provide and cross-entity registration into `emitAspectPolicies`

These are **not conflicting**: provides-derived policies (from `provides.to-hosts`/`provides.to-users`) are discovered during the walk and registered via `register-aspect-policy` — this is walk-time discovery, not user-declared `policies.*` auto-activation. The provides-cleanup registration path is unaffected by removing the `policies.*` auto-registration.

Implementers should be aware: "stop auto-registering `policies.*`" means deleting the loop over `builtins.attrNames (aspect.policies or {})`, NOT the provides-derived registration that produces `register-aspect-policy` for cross-entity keys.

## 18. Traits/Exports Integration

The future traits system and `den.exports` pattern (fleet) provide explicit upward data flow. Policies that need upward resolution (e.g., "export this host's firewall ports for the load balancer") would use `policy.export` or similar — not policy includes. This is out of scope for this spec but the scoping model is designed to accommodate it: downward cascade for policies, explicit exports for upward flow.

## 19. Files to Change

### Delete
- `den.schema.*.policies` merge logic from `modules/options.nix`

### Modify
- `modules/options.nix` — remove `policies` extraction from schema type merge
- `modules/policies/core.nix` — move `host-to-users` to `den.policies` + `den.schema.host.includes`
- `modules/policies/flake.nix` — move all schema policies to `den.policies` + `den.schema.*.includes`
- `modules/aspects/provides/os-class.nix` — `den.policies.os-to-host` already global; add activation via `den.default.includes`
- `modules/aspects/provides/os-user.nix` — `den.policies.user-to-host` already global; add activation via `den.default.includes`
- `modules/aspects/provides/wsl.nix` — `den.policies.wsl-to-host` already global (add activation); move `den.schema.host.policies.host-to-wsl-host` to `den.policies` + `den.schema.host.includes`
- `nix/nixModule/policies.nix` — update `den.policies` option type to `policyType`
- `nix/lib/aspects/fx/aspect/provide.nix` — `emitAspectPolicies` stops auto-registering `policies.*`
- `nix/lib/aspects/fx/aspect/children.nix` — `processInclude` checks `__isPolicy`, routes to `register-aspect-policy`
- `nix/lib/aspects/fx/policy/default.nix` — `installPolicies` reads only from aspect policies
- `nix/lib/aspects/fx/policy/dispatch.nix` — unwrap `{ fn, ... }` records instead of raw functions; add constraint check before dispatch
- `nix/lib/aspects/fx/policy/schema.nix` — verify compatibility with unified policy source (reads `state.flatAspectPolicies`)
- `templates/example/modules/aspects/igloo.nix` — translate to new API
- `templates/example/modules/aspects/alice.nix` — translate to new API
- All `templates/ci/modules/features/policy-*.nix` — translate to new API
- CI files using `den.schema.*.policies` (no longer valid): `class-module-partial-apply.nix`, `cross-provider.nix`, `ctx-chain.nix`, `ctx-compat.nix`, `ctx-pipeline.nix`, `custom-ctx.nix`, `doc-examples.nix`, `fx-coverage.nix`, `named-provider.nix`, `nested-ctx.nix`, `parametric.nix`, `policies.nix`
- CI files using `aspect.policies.*` (auto-activation removed): `host-propagation.nix` and others referencing `den.aspects.*.policies.*`
- All other `templates/ci/modules/features/` files that use `den.policies.*` — add activation

### New
- `nix/lib/aspects/types.nix` or new file — `policyType` definition
- `nix/lib/policy-effects.nix` — add `policy.for` and `policy.when` combinators
- `nix/lib/aspects/fx/policy/dispatch.nix` — evaluate `for`/`when` wrappers before dispatch
