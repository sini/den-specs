# Aspect Compilation and Include Resolution

**Date:** 2026-05-04
**Branch:** feat/fx-pipeline (629/629 tests)
**Scope:** How aspect attrsets become effect computations, how includes are resolved, and the narrow effect vocabulary that drives the pipeline.

---

## 1. aspectToEffect: Aspect Attrset to Effect Computation

**File:** `nix/lib/aspects/fx/aspect.nix`

`aspectToEffect` is the entry point. It takes an aspect attrset and returns an `fx` computation (a free monad value). The function dispatches on one question: does the aspect have named arguments (`__args != {}`)?

### Parametric path

If `__args` is non-empty, the aspect is parametric -- it is a function waiting for arguments from scoped handlers.

1. Guard against infinite recursion: `__parametricDepth` is incremented on each recursion and capped at 10.
2. If the aspect has `__scopeHandlers`, wrap the resolution in `fx.effects.scope.provide` so those handlers are available to the inner computation.
3. Resolve the function via `fx.bind.fn userArgs fn`. This is the nix-effects mechanism that performs the actual function application: it sends effects for each named argument, and the installed scoped handlers respond with values. The result is the function's return value.
4. Build `next` from the result via `mkParametricNext`:
   - If the result is still a function (curried), wrap it as a new parametric aspect with `__fn`/`__args`.
   - If the result is a submodule function, merge it through the aspect type system.
   - If the result is an attrset, merge it into the base.
5. Tag the result via `tagParametricResult`: merge parent and resolved `__scopeHandlers`, propagate `__ctxId`, accumulate `__parametricResolvedArgs`.
6. Recurse: call `aspectToEffect` on the tagged result. This loop continues until the aspect bottoms out as a static attrset.

### Static path

If `__args` is empty (or exhausted by parametric resolution), strip internal fields (`__fn`, `__args`, `__parametricDepth`, `__parametricResolvedArgs`) and call `compileStatic`.

---

## 2. compileStatic: Key Classification and Emission

**File:** `nix/lib/aspects/fx/aspect.nix`, `nix/lib/aspects/fx/key-classification.nix`

`compileStatic` processes a fully-resolved aspect attrset. It runs in three stages:

### 2.1 Target class query

Before classifying keys, `compileStatic` checks whether a `"class"` handler is installed (via `fx.effects.hasHandler`). If present, it sends a `"class"` effect to obtain the target class name. This is used during policy-driven resolution where the pipeline knows which class an aspect should emit into.

### 2.2 Key classification (`classifyKeys`)

**File:** `nix/lib/aspects/fx/key-classification.nix`

All aspect keys are partitioned into four categories:

- **Structural keys** (`structuralKeysSet`): `name`, `description`, `meta`, `includes`, `provides`, `policies`, `into`, `classes`, `__fn`, `__args`, `__functor`, `__functionArgs`, `__scopeHandlers`, `__ctxId`, `__entityKind`, `__parametricResolved`, `_module`, `_`. These are never emitted as classes or nested aspects -- they are consumed by the pipeline itself.

- **Class keys**: Keys that appear in `den.classes` (the class registry) or match the `targetClass` from step 2.1.

- **Nested keys**: Keys whose unwrapped value contains recognized sub-keys (registered classes). Detection is depth-limited to 3 levels. These become recursive `aspectToEffect` calls via `emitNestedAspect`.

- **Unregistered class keys**: Everything else. Treated as classes for backward compatibility. When the class registry is empty (no batteries loaded), all non-structural keys fall back to class keys.

### 2.3 Emission

`compileStatic` emits three things in parallel via `fx.seq`:

1. **Class emissions** (`emitClasses`): For each class key (registered + unregistered), unwrap the `aspectContentType` wrapper to a list of modules via `content-util.nix`, then send one `"emit-class"` effect per module element. Each emission carries:
   - `class`: the class name (e.g., `"nixos"`, `"home"`)
   - `identity`: the node identity (provider path + aspect name + optional index for multi-element values)
   - `module`: the raw NixOS/HM module value
   - `ctx`: context reconstructed from `__scopeHandlers` via `ctxFromHandlers`
   - `aspectPolicy` / `globalPolicy`: collision policies for `wrapClassModule`
   - `isContextDependent`: whether the aspect used context args during parametric resolution

2. **Constraint registration** (`registerConstraints`): Sends `"register-constraint"` effects for `meta.handleWith` and `meta.excludes` entries. These feed the constraint system that can exclude or substitute includes.

3. **Nested aspect recursion**: For each nested key, `emitNestedAspect` builds a sub-aspect (inheriting `__scopeHandlers`, `__ctxId`, and provider chain) and calls `aspectToEffect` on it.

After emission, control flows to `resolveChildren`.

---

## 3. resolveChildren: Processing Order

**File:** `nix/lib/aspects/fx/aspect.nix`

`resolveChildren` processes the aspect's children in a fixed five-step sequence. Each step is chained via `fx.bind` (monadic sequencing), so they run in order:

### Step 1: emitSelfProvide (vestigial)

**File:** `nix/lib/aspects/fx/include-emit.nix`

If `aspect.provides.${aspect.name}` exists, emit it as an include. This is the legacy `provides` self-reference pattern: an aspect named `"foo"` with `provides.foo = ...` emits the self-provide as a child include to be resolved.

The self-provide value can be a positional function (`fn ctx -> result`), a named-arg function (becomes parametric), or a static value. The result is sent as an `"emit-include"` effect which re-enters the classification pipeline.

**Vestigial:** This mechanism exists solely for backward compatibility with the `provides` API. Pending provides removal (consolidated spec Section 5), aspects will use `policies` or direct `includes` instead.

### Step 2: emitCrossProvideShims (vestigial)

**File:** `nix/lib/aspects/fx/handlers/provides-compat.nix`

For each key in `aspect.provides` that is NOT the aspect's own name and NOT a schema entity kind, emit a `"register-aspect-policy"` effect. Three variants:

- `to-hosts`: Fires for every host/user pair, includes result at current scope.
- `to-users`: Fires for every host/user pair, includes result at user scope.
- Named target (e.g., `provides.myhost`): Fires only when `host.name` or `user.name` matches the key.

Each emits a deprecation warning directing users to migrate to `policies`.

**Vestigial:** Same as emitSelfProvide -- exists only for backward compatibility.

### Step 3: emitAspectPolicies

**File:** `nix/lib/aspects/fx/include-emit.nix`

For each entry in `aspect.policies`, emit a `"register-aspect-policy"` effect. These are aspect-scoped policies (as opposed to global `den.policies`). The policy function and owner identity are registered for later dispatch by `installPolicies`.

### Step 4: emitIncludes

**File:** `nix/lib/aspects/fx/include-emit.nix`

The core include resolution loop. Processes `aspect.includes` (a list) sequentially, classifying each child and dispatching it through the effect vocabulary. See Section 4 for the classification logic and Section 5 for the effect handlers.

### Step 5: installPolicies (entity-only)

**File:** `nix/lib/aspects/fx/policy-dispatch.nix`

Only runs if the aspect has `__entityKind` (i.e., it is a schema entity root like a host or user). Dispatches global and aspect-scoped policies against the entity's context, creating new scopes via `scope.provide` for each entity resolution. This is where `policy.resolve`, `policy.include`, `policy.exclude`, and `policy.route` effects get materialized.

### Chain wrapping

If the aspect has a meaningful name (not `<anon>`, not synthetic), the entire `childResolution` computation is wrapped with `chain-push`/`chain-pop` effects. These maintain the includes chain for provenance tracking and anonymous child naming.

### Completion

After all children resolve, the aspect is marked complete via a `"resolve-complete"` effect (which updates the `pathSet` for conditional guard evaluation).

---

## 4. emitIncludes: Include Classification

**File:** `nix/lib/aspects/fx/include-emit.nix`

`emitIncludes` iterates over an includes list, processing each child through a classification pipeline. For each child:

### 4.1 Wrapping (`wrapChild`)

Raw children are normalized into aspect attrsets:
- **Attrsets with name + includes list**: passed through (already an aspect shape).
- **Functors** (`__functor`): extract the inner function, detect submodule functions, set `__fn`/`__args`.
- **Bare functions**: if submodule function, merge through `aspectType`; otherwise wrap as `{ name, meta, __fn, __args }`.
- **Non-functions**: passed through as-is.

Parent `__scopeHandlers` and `__ctxId` are propagated to children that don't already have them.

### 4.2 Anonymous naming

If the child has no meaningful name and `__skipNameAnon` is false, it gets a synthetic name based on the current chain position: `"<parent>/<anon>:<index>[/<ctxId>]"`.

### 4.3 Classification and dispatch

Each wrapped child goes through this decision tree:

1. **Dedup check** (`"check-dedup"` effect): If the child has already been seen in this scope, skip it entirely (return `[]`).

2. **Forward**: If `child.meta.__forward` exists, send `"emit-forward"` and return `[]`. Forwards are processed post-pipeline.

3. **Conditional**: If `child.meta.guard` exists, send `"resolve-conditional"`. The handler evaluates the guard against the current path set.

4. **Constraint check** (`"check-constraint"` effect): Tests against registered constraints (from `meta.handleWith`/`meta.excludes`). Three possible outcomes:
   - `exclude`: Emit a tombstone, optionally unregister from `includeSeen`.
   - `substitute`: Emit a tombstone for the original, resolve the replacement.
   - `allow`: Continue to step 5.

5. **Parametric vs static**: If `__args != {}`, send `"resolve-parametric"`; otherwise send `"resolve-aspect"`.

---

## 5. The Narrow Effect Vocabulary

The pipeline uses four resolution effects (plus `check-dedup`). Each is handled by a dedicated handler module.

### 5.1 resolve-aspect

**File:** `nix/lib/aspects/fx/handlers/resolve-aspect.nix`

Static resolution. Calls `aspectToEffect` on the child and wraps the result in a singleton list. This is the simple case: the aspect is fully resolved and ready for compilation.

### 5.2 resolve-parametric

**File:** `nix/lib/aspects/fx/handlers/resolve-parametric.nix`

Parametric resolution with deferral. The handler:

1. Determines which required arguments (non-default `__args` keys) lack scoped handlers -- both from `__scopeHandlers` on the child and from the ambient handler environment (probed via `fx.effects.hasHandler`).

2. If all required args are available: calls `aspectToEffect` (which will resolve the function via `fx.bind.fn`), returns the result as a singleton list.

3. If any required args are unavailable: emits a stub tombstone via `"resolve-complete"`, then sends `"defer-include"` carrying the child and the list of missing `requiredArgs`. The child will be retried later when context widens (during entity resolution in `installPolicies`).

### 5.3 resolve-conditional

**File:** `nix/lib/aspects/fx/handlers/resolve-conditional.nix`

Guard evaluation. The handler:

1. Sends `"get-path-set"` to retrieve the set of all resolved aspect paths so far.
2. Builds a `guardCtx` with `hasAspect` (checks path set membership).
3. Calls `condNode.meta.guard guardCtx`. If it returns true, delegates the conditional's `meta.aspects` list to `emitIncludes`. If false, tombstones all aspects in the list.

### 5.4 check-dedup

**File:** `nix/lib/aspects/fx/handlers/check-dedup.nix`

Include dedup via `includeSeen` state. Returns `{ isDuplicate, dedupKey }`:

1. Computes a dedup key from the child's identity path, scoped by `state.currentScope`. Synthetic names (wrapped in `<>`) and anonymous children get `null` dedup keys (never deduplicated).

2. Checks `state.includeSeen` for the scoped key.

3. If not a duplicate and key is non-null, eagerly registers the key in `includeSeen`. This prevents the same aspect from being resolved twice in the same scope.

The eager registration is paired with `"include-unseen"` (in the include handler), which can roll back a registration when a constraint excludes a child after it was already registered.

---

## 6. Include Dedup via includeSeen

**File:** `nix/lib/aspects/fx/handlers/check-dedup.nix`, `nix/lib/aspects/fx/handlers/include.nix`

The `includeSeen` state field is a thunked attrset (`_: { key = true; ... }`) tracking which aspects have been resolved in each scope.

### Dedup key construction

`dedupKey = "${scope}/${identity}"` where:
- `scope` is `state.currentScope` (the entity scope ID, e.g., `"host:myhost"`)
- `identity` is the aspect's path key (`provider/name` plus optional `{ctxId}` suffix)

### Scope isolation

The scope prefix ensures the same aspect can be resolved independently in different entity scopes. For example, `host:alpha/batteries/git` and `host:beta/batteries/git` are distinct dedup keys.

### Rollback

When `excludeChild` fires (from constraint checking), it sends `"include-unseen"` to remove the dedup key. This prevents a constraint-excluded aspect from blocking later legitimate inclusion of the same aspect.

---

## 7. Deferred Includes: Parametric Deferral and Drain

When `resolve-parametric` encounters a child whose required arguments lack handlers, it defers the child:

1. Emits a stub via `"resolve-complete"` (so the path set records the attempt).
2. Sends `"defer-include"` with the child and required args.

### Drain mechanism

Deferred includes are drained when entity context widens during `installPolicies`:

- `installPolicies` creates new scopes via `scope.provide` for each entity, installing `constantHandler`s for entity context keys (`host`, `user`, etc.).
- After scope installation, deferred includes whose required args are now satisfied by the new handlers are retried.
- Each retry re-enters the normal `aspectToEffect` path, where `fx.bind.fn` can now resolve the previously-missing arguments.

This is the mechanism that allows parametric aspects like `{ host, ... }: { nixos = ...; }` to be included at the top level (where `host` is unavailable) and resolved later when a host entity scope provides the `host` handler.

---

## 8. content-util.nix: Unwrapping aspectContentType

**File:** `nix/lib/aspects/fx/content-util.nix`

The aspect type system wraps values in `__contentValues` (a list of `{ value }` records from multi-site module definitions). `content-util.nix` provides four unwrap functions:

- `unwrapContentValues`: Filters empty attrsets, merges remainder. Single value returned directly; multiple values wrapped as `{ imports = vals; }`.
- `unwrapContentValuesRaw`: No empty filtering. Used by `emitNestedAspect`.
- `unwrapContentValuesList`: Returns a list of individual module values for per-element class emission. Lists pass through; `__contentValues` unwrap to filtered singleton or merged imports.
- `unwrapContentValuesForClassification`: Merges all attrset values for sub-key detection in `classifyKeys`. Non-attrsets return `null`.

---

## 9. Vestigial Components

Two steps in `resolveChildren` exist solely for backward compatibility with the `provides` API:

### emitSelfProvide

Translates `provides.${name}` into an include. When an aspect named `"foo"` has `provides.foo = { host, ... }: { nixos = ...; }`, the self-provide is emitted as a child include that re-enters the classification pipeline. The target state is for aspects to use `includes` directly or `policies` for cross-entity delivery.

### emitCrossProvideShims

Translates `provides.to-users`, `provides.to-hosts`, and named provides targets (`provides.myhost`) into aspect-scoped policies via `"register-aspect-policy"`. Each emits a deprecation warning. The target state is for aspects to use `aspect.policies` with explicit `policy.include` effects.

Both are slated for removal once the 114 template `provides` occurrences are migrated (see provides removal plan).
