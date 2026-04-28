# Policy Pipeline Simplification

## Status: Draft

## Prerequisites

- Unified policy effects (2026-04-27) implemented
- Mutual-provider replaced with policyFns
- All test fixture policies converted to `__functor` (in progress)

## Core Insight

With all policies using typed effects, the remaining transition mechanisms are vestigial. The `__functor` + metadata pattern (`from`, `to`, `_core`, `isolateFanOut`) was a migration bridge. The target state is plain functions — no metadata, no activation model, no dual dispatch.

## What Gets Removed

### Metadata fields on policies (~all `from`/`to`/`_core`/`isolateFanOut`)

**`from`** — replaced by function signature matching. A policy taking `{ host, ... }:` fires when `host` is in context. No explicit scoping needed.

**`to`** — replaced by binding key derivation. `policy.resolve { user = ... }` targets entity kind `user`. Non-schema targets (`flake-os`, `flake-hm`, `flake-system`, `default`) become schema kinds so their binding keys derive correctly.

**`_core`** — replaced by unconditional dispatch. All `den.policies` entries fire when signature matches. No activation gate. Core policies are just policies.

**`isolateFanOut`** — becomes a property of `policy.resolve` itself, not the policy. Default `true` for all resolves. Opt-out via `policy.resolve.shared` or similar if needed (only `host-to-users` currently uses shared fan-out).

### Dispatch infrastructure (~200 lines)

Dead after steps 1-6 complete (metadata removal + dispatch collapse):

| Component | File | Dead after step |
|-----------|------|-----------------|
| `oldStyleHandler` | policy-dispatch.nix | Step 3 (all policies plain functions) |
| `compilePolicyHandlers` | policy-dispatch.nix | Step 5 (dispatch collapse) |
| `policyEffectNamesFor` | policy-dispatch.nix | Step 5 (no `from` filtering) |
| `activePoliciesFor` | synthesize-policies.nix | Step 4 (activation removed) |
| `ctxSatisfies` | synthesize-policies.nix | Step 5 (arg matching handles scoping) |
| `dualPolicyType` | policy-types.nix | Step 6 (`types.raw` only) |
| `policyType` submodule | policy-types.nix | Step 6 (no old-style policies) |
| `isOldStylePolicy` | policy-types.nix | Step 6 (no old-style policies) |
| `collectPolicyHandlers` | transition.nix | Step 5 (no `policy.handlers` field) |

### Activation mechanism (~50 lines across files)

| Component | Why dead |
|-----------|----------|
| `den.default.policies` (global activation list) | No activation model |
| `den.schema.host.policies` (schema-kind activation) | Same |
| `entity.policies` option on host/user/home | Same |
| `den.schema.host.policies = [...]` writes in batteries | Same |

### Other dead mechanisms

| Component | File | Why dead |
|-----------|------|----------|
| `emitCrossProvider` | transition.nix | `provides` cross-routing eliminated |
| `collectPolicyHandlers` (installs `policy.handlers`) | transition.nix | No policy declares handlers |
| Manual `into`/`intoFn` | aspect.nix, transition.nix | Verify if still used; if not, remove |
| `emitSelfProvide` | aspect.nix | Verify if `provides.${self.name}` still used anywhere |

## Schema Registration for Flake Entity Kinds

Currently, `flake-os`, `flake-hm`, `flake-system`, `default`, and per-output kinds (`flake-packages`, `flake-apps`, etc.) are handled by `resolve-entity` but are NOT in `den.schema`. They need to be registered so binding key derivation works:

```nix
# These become schema kinds (isEntity = true or a lighter registration):
den.schema.flake-system = { ... };
den.schema.flake-os = { ... };
den.schema.flake-hm = { ... };
den.schema.flake-packages = { ... };
den.schema.flake-apps = { ... };
den.schema.flake-checks = { ... };
den.schema.flake-devShells = { ... };
den.schema.flake-legacyPackages = { ... };
den.schema.default = { ... };
```

With these registered, binding keys derive correctly:

```nix
# Before (with explicit to):
flake-system-to-flake-os = {
  to = "flake-os";
  __functor = _: { system, ... }: [ (policy.resolve { host = ...; }) ];
};

# After (plain function, binding key IS the target):
flake-system-to-flake-os = { system, ... }:
  map (host: policy.resolve { flake-os = host; })
    (builtins.attrValues (den.hosts.${system} or {}));
```

And `*-to-default`:

```nix
# Before:
host-to-default = { to = "default"; __functor = _: ctx: [ (policy.resolve ctx) ]; };

# After:
host-to-default = { host, ... }: [ (policy.resolve { default = host; }) ];
```

**Decision: `*-to-default` deferred to Phase E (pipeline architecture).** Investigation found three fundamental blockers:

1. **Sentinel approach (`resolve { default = true; }`)** fails: the sentinel binding only exists in the child scope, so the parent entity re-dispatches `*-to-default` on every transition iteration, causing infinite recursion. The `ctx-seen` dedup doesn't catch it because each iteration starts fresh from the parent scope.

2. **Implicit default as include** (adding `den.default` to `resolveEntity` includes) fails: `den.default` contains parametric aspects (`{ host, ... }:`) that need proper scope isolation. As an include, they resolve in the same pipeline as the entity and see raw context values (e.g., test fixtures pass `host = "h"` as a string), breaking `den.default`'s parametric aspects. The old transition-based routing provided scope isolation via sub-pipeline.

3. **Schema registration of `default`** is safe (`flake-schema.nix` with empty `{}` takes the passthrough branch in `schemaEntryType`, never runs `resolvedCtx`), but doesn't solve the routing problem — `targetKey` derivation from resolve bindings finds the source entity kind (`host`), not `default`, because `*-to-default` is a context-passthrough policy.

**Conclusion:** `*-to-default` requires transition-based routing with scope isolation — exactly what `to = "default"` provides. Converting it to a plain function requires a pipeline-level change (e.g., built-in default transition in the transition handler, not a policy). This is Phase E work.

**For Phase B:** `*-to-default` policies retain their current `__functor` + `to = "default"` shape. Only `host-to-users` and flake policies convert to plain functions. The `from` and `_core` fields can still be removed from `*-to-default` since `newStyleHandler` handles `__functor` attrsets without them.

## Policy Shape After Simplification

All policies become plain functions. No metadata wrapping.

```nix
# den.policies — top-level registry, all entries are plain functions
den.policies = {
  host-to-users = { host, ... }:
    map (user: policy.resolve { inherit user; })
      (lib.attrValues host.users);

  flake-to-flake-system = _:
    map (system: policy.resolve { flake-system = system; })
      den.systems;

  flake-system-to-flake-os = { flake-system, ... }:
    lib.concatMap (host: [
      (policy.resolve { flake-os = host; })
      (policy.include den.aspects."flake-os")
    ]) (builtins.attrValues (den.hosts.${flake-system} or {}));
};

# Aspect-included policies — same shape, on policyFns key
den.aspects.igloo = {
  policyFns.to-alice = { host, user, ... }:
    lib.optional (user.name == "alice")
      (policy.include { homeManager.programs.vim.enable = true; });
};
```

No `from`, no `to`, no `_core`, no `__functor`. Just functions.

### Wildcard policy scoping

Policies with wildcard args (`_:`) match every context since `resolveArgsSatisfied` with no required args always returns true. Currently `ctxSatisfies` (removed in step 6) prevents flake-scope policies from firing at entity levels — it checks that no entity attrset values exist in context.

After `ctxSatisfies` removal, `flake-to-flake-system = _: ...` would fire at host/user levels too. Options:

1. **Require at least one named arg**: ban wildcard-only policies. `flake-to-flake-system` becomes `{ ... }: ...` with no required args but the policy registers its scope via a lightweight annotation (contradicts "no metadata" goal).
2. **Body guard**: `flake-to-flake-system = ctx: lib.optional (!(ctx ? host)) ...`. Explicit, no metadata, but error-prone — forgetting the guard silently breaks things.
3. **Retain scope check for unscoped policies**: keep a lightweight version of `ctxSatisfies` that only applies to policies with no required args. Policies with named args self-scope via signature; wildcard policies need the guard.
4. **Schema-based scope derivation**: infer scope from `policy.resolve` binding keys. `flake-to-flake-system` resolves `{ system }` — the pipeline can infer this only fires at flake level since `system` is a flake-scope binding.

**Decision: Option 4 — schema-based scope derivation.** The pipeline already knows which binding keys each `policy.resolve` targets. Scope is inferred from those keys: a policy that only resolves `{ system }` (a flake-scope binding) cannot fire when entity attrset values are in context. This keeps "no metadata" clean — the policy's resolve effects ARE the scope declaration. Implementation: at dispatch time, if a policy has no required args, check that its resolve binding keys are compatible with the current context level. Requires the dispatch loop to have visibility into the policy's effect structure (either via lazy pre-analysis or by running the policy speculatively and discarding effects that target incompatible scopes).

### `isolateFanOut` handling

Currently defaults to `true` for new-style, `false` for old-style. After simplification:

- Default: `true` (each resolve creates isolated sub-pipeline)
- `host-to-users` needs `false` (shared state fan-out)
- Option A: `policy.resolve.shared { user = ... }` — variant constructor
- Option B: Metadata on the policy attrset (but we're removing metadata...)
- Option C: Make shared fan-out the default (it's what most entity transitions need)
- Option D: Keep `isolateFanOut` as the one metadata field, via `__functor` only for policies that need it

**Recommended: Option A** — `policy.resolve.shared` variant. Clean, explicit, no metadata:

```nix
host-to-users = { host, ... }:
  map (user: policy.resolve.shared { inherit user; })
    (lib.attrValues host.users);
```

## Pipeline Dispatch After Simplification

### Current flow (complex)

```
compilePolicyHandlers → { "policy:name" = handler } for each den.policies entry
policyEffectNamesFor entityKind → filter by from
transitionHandler → fx.send each "policy:name" effect
each handler: check activation → check ctxSatisfies → check resolveArgsSatisfied → call
dispatchAspectPolicies → direct call for policyFns in state
```

### Target flow (simple)

```
transitionHandler →
  for each policy in den.policies + state.aspectPolicies:
    check resolveArgsSatisfied (function signature matching)
    call policy → partition effects → build transitions
```

No named effects per policy. No activation. No scope guards. Just match-and-call.

This means `compilePolicyHandlers` and the entire `"policy:name"` effect mechanism are eliminated. The transition handler dispatches directly.

### Impact on tracing

Policy trace entries currently use `"policy:name"` effect names for provenance. After simplification, the transition handler calls policies directly and must construct trace entries itself. This is a minor change to `trace.nix`.

## Activation removal and `policyFns` → `policies` rename

Two separate `policies` options exist that collide with the rename:

1. **Aspect submodule** (types.nix:388): `policies = listOf str` — activation list on every aspect (including `den.default`). Coexists with `policyFns = lazyAttrsOf raw` (already renamed in Task 4).
2. **Entity-policies.nix**: `policies = listOf str` — per-entity-kind and per-entity-instance activation, injected via `den.schema.conf.imports`.

Both are part of the activation model removed in step 5. After removal, the `policyFns` → `policies` rename on the aspect submodule (step 10) proceeds without collision.

## Dependency Order

### Phase A: Prerequisites — SHIPPED

```
Step 0: ✅ policy.resolve.shared variant (commit 8b999f35)
Step 1: *-to-default question resolved: include approach viable but blocked
        on Phase E forward elimination (see project_default_routing.md)
Step 2: ✅ Flake + default schema registration (commit 399a3ddd)
```

### Phase B: Metadata removal — SHIPPED (partial)

```
Step 3:  ✅ _core + isolateFanOut removed from core policies (commit 73ac2f50)
         DEFERRED: from/to/__functor retained — needed by policyEffectNamesFor
         until dispatch collapse. from still needed post-collapse (hasPolicies).
Step 4:  ✅ _core removed from flake policies (commit 82864411)
         DEFERRED: from/to/__functor retained (same reason as step 3)
Step 5:  ✅ _core removed from all test fixtures (commit 424888b7)
Step 6:  ✅ Activation mechanism removed (commit 29156fe0)
Step 7:  ✅ Dispatch collapsed to direct iteration (commit a2a7c933)
         Key finding: hasPolicies must check from == entityKind per-policy
         (not just den.policies != {}), otherwise default-level dispatch
         causes user→default→user infinite recursion.
Step 8:  ✅ Dead types removed, policy-dispatch.nix deleted (commit 30ba636e)
```

### Phase C: Dead code removal — SHIPPED (partial)

```
Step 9:  ✅ collectPolicyHandlers + policyTraceHandlers removed (commit 72b34735)
         DEFERRED: emitCrossProvider — part of provides removal (separate plan)
Step 10: DEFERRED: provides infrastructure — separate plan
Step 11: into/intoFn kept — used by ctx-shim compat + tests
Step 12: DEFERRED: host-aspects.nix — part of provides removal (heavily tested)
```

### Phase D: Rename + type cleanup — SHIPPED

```
Step 13: ✅ policyFns → policies rename (commit 23ab3541)
Step 14: ✅ den.policies option → lazyAttrsOf raw (commit 30ba636e)
Step 15: ✅ policy-types.nix → isFunctorAttrset + isNewStylePolicy + policyFnArgs
         ✅ trace.nix policyTraceHandlers removed (commit 72b34735)
         policy-inspect.nix: still uses ctxSatisfies for scope filtering (kept)
```

### Phase E: Pipeline architecture + deferred policy conversion

```
Step 16: Multi-class collection (change classCollectorHandler + fxResolve return shape)
Step 17: Eliminate forward sub-pipeline (depends on step 16)
Step 18: Convert *-to-default to den.default include injection (depends on step 17 —
         4 forward-duplication failures resolve once forward sub-pipelines gone).
         See project_default_routing.md: 3 injection points in options.nix,
         pipeline.nix resolveEntityHandler, transition.nix resolveContextValue.
Step 19: Remove `from` from all policies. Two sites to update:
         - aspect.nix hasPolicies: currently checks from == aspect.name per-policy.
           Replace with precomputed set of entity kinds that have matching policies,
           or derive from schema-based scope (spec option 4).
         - transition.nix scopeOk: fromKind == sourceEntityKind. Replace with
           body guards (like flake-to-flake-system ctx guard) or scope derivation
           from resolve binding keys.
Step 20: Remove `to` from flake policies. Requires targetKey derivation to handle
         the mismatch: resolve keys (host, home, system) ≠ schema kinds
         (flake-os, flake-hm, flake-system). Options: rename resolve keys to
         match schema kinds, or add a binding-key-to-schema-kind mapping.
Step 21: Remove `__functor` wrappers — convert all policies to plain functions.
         Depends on steps 19-20 (from/to no longer needed as attrset metadata).
Step 22: Class keys as lists (depends on step 16 for multi-scope motivation)
```

## Impact Summary

### Phases A-D (shipped)

| Metric | Value |
|--------|-------|
| Lines removed (actual) | ~900 |
| Files with major changes | 6 (policy-dispatch, transition, synthesize-policies, policy-types, aspect, pipeline) |
| Files deleted | 3 (policy-dispatch.nix, entity-policies.nix, policy-activation.nix test suite) |
| Concepts eliminated | 6 (_core, activation lists, named policy effects, policyType/dualPolicyType, collectPolicyHandlers, policyTraceHandlers) |
| Concepts NOT yet eliminated | 3 (from, to, __functor — deferred to Phase E steps 19-21) |

### Phase E (remaining)

| Metric | Value |
|--------|-------|
| Steps remaining | 7 (steps 16-22) |
| Key blockers | Multi-class collection (step 16) unblocks forward elimination (17) which unblocks *-to-default conversion (18) |
| `from` removal (step 19) | Blocked on `hasPolicies` rework + body guards/scope derivation |
| `to` removal (step 20) | Blocked on resolve-key-to-schema-kind mapping for flake policies |

## Eliminate Sub-Pipeline Patterns

### host-aspects battery → delete

host-aspects re-resolves `host.aspect` with user context in a separate `den.lib.aspects.resolve` call per user class. This was needed because the host pipeline ran without user context, so parametric host aspects like `{ user }: { homeManager... }` couldn't resolve.

With policyFns, host aspects declare their own cross-entity routing: `policyFns.to-users = { host, user }: [ (policy.include { homeManager... }) ]`. The include fires during user entity tree-walk with both host and user in context. No re-resolution needed. host-aspects battery is redundant.

**Action:** Delete `modules/aspects/provides/host-aspects.nix`. Convert test fixtures that use `den._.host-aspects` to use policyFns on the host aspect directly.

### forward sub-pipeline → multi-class collection

The forward mechanism runs a sub-pipeline with a different `class` parameter to collect emissions for a non-target class (e.g., `homeManager` while the host pipeline targets `nixos`), then wraps them at a module path (e.g., `home-manager.users.tux`).

**Current architecture:**
```
host pipeline (class=nixos) → emit-class filters to nixos only
forward sub-pipeline (class=homeManager) → collects homeManager → wraps → injects into nixos
```

**Target architecture:**
```
host pipeline → emit-class collects ALL classes into per-class buckets
forward = post-processing: take homeManager bucket, wrap at target path, merge into nixos imports
```

**Change:** `classCollectorHandler` stops filtering by `param.class == targetClass`. Instead, it collects into per-class buckets in state:

```nix
# Current state:
imports = _: [ ];  # flat list, target class only

# After:
classImports = _: { };  # { nixos = [...]; homeManager = [...]; darwin = [...]; }
```

`emitClasses` emits to the appropriate bucket based on `param.class`. `fxResolve` returns per-class imports. The caller (entity resolution, forward) routes each class to its destination.

Forward becomes a post-processing step: extract non-target-class buckets from the entity's resolution result, wrap each at the forward path, and merge into the target class's imports. No sub-pipeline.

**Impact:** Eliminates the most expensive sub-pipeline pattern. Forward currently runs a full pipeline per user per host. With multi-class collection, user entity resolution collects all classes in one pass.

**`fxResolve` caller inventory** (must update for `{ classImports }` return shape):

| Caller | File | Uses |
|--------|------|------|
| `fxResolveTree` (→ `resolve`) | aspects/default.nix:60 | `fxResolve` |
| `fxResolveTreeFull` (→ `resolveWithState`) | aspects/default.nix:72 | `fxFullResolve` |
| `has-aspect.nix` | aspects/has-aspect.nix:21 | `fxFullResolve` |
| `resolveFanOut` (sub-pipeline) | aspects/fx/handlers/transition.nix:209 | `fxFullResolve` |
| Forward sub-pipeline | aspects/fx/handlers/forward.nix | `fxFullResolve` |
| Test: fx-full-pipeline.nix | templates/ci/ (4 calls) | both |
| Test: include-dedup.nix | templates/ci/ (7 calls) | `fxFullResolve` |
| Test: provide-to.nix | templates/ci/ (7 calls) | both |
| Test: traits.nix | templates/ci/ (2 calls) | `fxResolve` |
| Test: fx-e2e.nix | templates/ci/ (3 calls) | `fxResolve` |

Production callers: 4 (default.nix resolve/resolveWithState, transition.nix fan-out, forward.nix sub-pipeline). Test callers: ~23. The forward.nix caller is eliminated by this change; the others need migration.

## Class Keys as Lists

Currently, an aspect's class keys are single attrsets keyed by class name:

```nix
den.aspects.igloo = {
  nixos.programs.git.enable = true;
  homeManager.programs.vim.enable = true;
};
```

This works for static config, but parametric aspects can only have ONE function signature per aspect:

```nix
# This works — one scope:
den.aspects.igloo = { host, ... }: {
  nixos.programs.git.enable = true;
};

# This DOESN'T work — can't have different scopes for different classes:
den.aspects.igloo = {
  nixos = { host, ... }: { programs.git.enable = true; };           # host-level
  homeManager = { host, user, ... }: { programs.vim.enable = true; }; # user-level
};
# ^ homeManager key is a function, not a class module
```

The problem: class keys are detected by `classifyKeys` which checks if the value is a registered class name. A function value for a class key is ambiguous — is it a parametric aspect or a class module that takes NixOS args?

**Solution: allow class keys to accept lists:**

```nix
den.aspects.igloo = {
  # Single module (existing):
  nixos.programs.git.enable = true;

  # List of modules (new):
  homeManager = [
    { programs.vim.enable = true; }                              # static
    ({ host, user, ... }: { programs.direnv.enable = true; })    # parametric (user-scoped)
  ];
};
```

Lists are unambiguous — they can't be confused with class modules or parametric aspects. Each list element is a class module (attrset or NixOS module function). The pipeline processes each element independently, with `wrapClassModule` handling context injection per element.

This also enables mixing scopes in a single aspect file:

```nix
den.aspects.igloo = {
  nixos = [
    { programs.git.enable = true; }                    # static, host-level
    ({ host, ... }: { networking.hostName = host.name; })  # parametric, host-level
  ];
  homeManager = [
    ({ user, ... }: { programs.vim.enable = true; })   # parametric, user-level
  ];
};
```

**Implementation:** In `emitClasses`, coerce all class key values to lists internally:

```nix
# In emitClasses, before processing:
modules =
  if builtins.isList rawValue then rawValue
  else [ rawValue ];
# Then emit each element as a separate emit-class effect.
```

Bare values (the existing API) are coerced to singleton lists. Users never need to wrap single modules in `[ ]`. `wrapClassModule` already handles both attrset and function module shapes — it processes each list element independently.

## From Unified Effects Spec (verify/complete)

### policy.exclude rollback

The unified effects spec requires: "excluding an aspect removes its included policies too. If the excluded aspect's policies have already produced resolve effects in an earlier iteration, those effects are rolled back." Task 4 implemented `ownerIdentity` tracking on aspect policies. The actual rollback (removing imports produced by the excluded aspect's policies) is unverified. Verify the existing `include-unseen` rollback mechanism covers this case, or implement explicit rollback.

### provides structural key removal

Covered in detail by `2026-04-27-provides-removal-post-unified-effects.md`. Key items for this spec:
- Delete `emitSelfProvide`, `mkPositionalInclude`, `mkNamedInclude` from `aspect.nix`
- Delete `emitCrossProvider` from `transition.nix`
- Remove `provides` option from `aspectSubmodule` in `types.nix`
- Remove `"provides"` from `structuralKeysSet` in `aspect.nix`
- Remove `substituteChild` from `include.nix` (substitute = exclude + include)
- Remove `_` alias (`mkAliasOptionModule ["_"] ["provides"]`) — BUT `den._` is `den.provides` factory namespace, not the aspect key. Verify which alias this refers to.
- Clean up `den-brackets.nix` provides fallback

### Scoped trait filtering

The spec describes `policy.resolve` shadowing trait context for subtrees (e.g., filtering secrets per-user). This works in theory (resolve creates context branches, traits are context values matched by signature) but is untested. Add test coverage, not implementation work.

## Risks

### Phase A-B (metadata removal — low risk, incremental)

- **Test fixture volume**: ~17 files with old-style policies being converted. Must complete before steps 3-7.
- **`*-to-default` elimination**: Open decision — see above. Blocks step 1 and step 4 (core policy conversion).
- **`policy.resolve.shared`**: New effect constructor variant, prerequisite for step 4. Needs implementation in `den.lib.policy` and dispatch handling in transition.nix.
- **Flake schema registration**: May trigger evaluation cycles if `den.schema` is referenced during flake-level policy evaluation. `schemaKinds` is already computed at pipeline startup so likely safe, but needs verification in step 2.

### Phase E (pipeline architecture — higher risk)

- **Multi-class collection**: Changes `fxResolve` return shape from `{ imports }` to `{ classImports }`. 4 production callers + ~23 test callers must update (see inventory above).
- **Class key lists**: `classifyKeys` must recognize list values as class content (not structural). `aspectContentType` needs to accept lists for class keys. `wrapClassModule` must handle per-element wrapping. Interaction with outer parametric aspect functions (`{ host }: { nixos = [...] }`) needs design — does the outer function apply once, with list elements processed individually?
