# Aspect Compilation Pipeline

**Branch:** feat/fx-pipeline (753/753 tests)
**Scope:** How aspect attrsets enter the pipeline via `resolve`, get dispatched to type-specific compilers, and produce class module emissions in scope-partitioned state.

---

## 1. Overview

Every aspect enters the pipeline as a `resolve` effect. The resolve handler delegates to `compile`, which inspects the aspect's shape and dispatches to one of four compilers: `compile-static`, `compile-parametric`, `compile-forward`, or `compile-conditional`. Static compilation classifies the aspect's keys, emits class modules, processes nested sub-aspects, and resolves children (policies, includes, entity policies). Parametric compilation probes scope for required arguments, either binding the function and re-resolving the result or deferring the aspect for later drain. Forward compilation extracts route metadata and registers it in scoped state. Conditional compilation evaluates a guard predicate against the resolved path set.

Before any compiler runs, the **gate** composite checks for deduplication (scope-prefixed key in `includeSeen`) and constraint violations (exclude/substitute from the constraint registry). If the gate blocks, the entire subtree is skipped.

---

## 2. Compile Router

**File:** `nix/lib/aspects/fx/handlers/compile.nix`

The `compile` handler receives `{ aspect, identity, ctx }` and inspects the aspect to select a compiler:

| Condition | Effect dispatched |
|---|---|
| `meta.__forward` exists | `compile-forward` |
| `meta.guard` exists | `compile-conditional` |
| `__args` is non-empty | `compile-parametric` |
| Otherwise | `compile-static` |

The router forwards the entire `param` payload unchanged to the selected compiler. The `resolve` handler (`resolve.nix`) is a trivial indirection that sends `compile` with the same payload.

---

## 3. Gate

**Files:** `nix/lib/aspects/fx/handlers/gate.nix`, `nix/lib/aspects/fx/handlers/gate-tag.nix`

The gate is a composite effect used by `compile-static` and `compile-parametric` (but not `compile-forward` or `compile-conditional`). It runs two checks in sequence:

### 3.1 Dedup (check-dedup)

Sends `check-dedup` with the aspect. Returns `{ isDuplicate, dedupKey }`. If `isDuplicate` is true, the gate returns `{ blocked = true; result = []; }` and the compiler produces nothing.

See [check-dedup](#4a-dedup-key-construction) for key construction.

### 3.2 Constraint check (check-constraint)

If dedup passes, sends `check-constraint` with `{ identity, aspect }`. Three outcomes:

- **exclude**: Emits a tombstone via `resolve-complete`, optionally rolls back the dedup registration via `include-unseen` (if `dedupKey` is non-null). Returns `{ blocked = true; result = [tombstone]; }`.
- **substitute**: Emits a tombstone for the original, then sends `resolve` for the replacement aspect. Returns `{ blocked = true; result = [tombstone] ++ replacementResult; }`.
- **keep** (or with `owner`): Returns `{ passed = true; }`, optionally with `owner` for constraint tagging.

### 3.3 Gate-tag wrapper

`gate-tag.nix` provides `gateAndTag`, used by both static and parametric compilers. It skips the gate if `param.gated` is true (parametric re-entry after bind). On pass-through, if the gate returns an `owner`, the aspect's `meta.constraintOwner` is set. The tagged aspect is passed to the compiler's continuation.

See: [constraint-system.md](constraint-system.md)

---

## 4. Static Compilation

**File:** `nix/lib/aspects/fx/handlers/compile-static.nix`

Static compilation processes fully-resolved aspect attrsets (no `__args`, no `__forward`, no `guard`). After gating, it runs:

### Step 1: Target class probe

Checks whether a `"class"` handler is installed via `fx.effects.hasHandler`. If present, sends a `"class"` effect to obtain a `targetClass` name. This is used during policy-driven resolution where the pipeline knows which class an aspect should emit into.

### Step 2: Classify keys

Sends a `"classify"` effect with `{ aspect, targetClass }`. The classify handler calls `classifyKeys` (see [Section 8](#8-key-classification)) and returns `{ classKeys, nestedKeys, pipeKeys }`. The classify handler merges `unregisteredClassKeys` into `classKeys`.

### Step 3: Emit classes + constraints + nested aspects

Runs in parallel via `fx.seq`:

- **emit-classes**: Sends `"emit-classes"` with `{ aspect, classKeys, pipeKeys, identity }`. See [emit-classes](#4b-emit-classes).
- **registerConstraints**: Sends `"register-constraint"` effects for `meta.handleWith` and `excludes` entries.
- **Nested keys**: For each nested key, builds a sub-aspect (inheriting `__scopeHandlers`, `__ctxId`, provider chain) and sends `"resolve"`.

### Step 4: Resolve children

Sends `"resolve-children"` with `{ aspect, isMeaningful, chainIdentity }`. See [Section 10](#10-resolve-children).

Returns `[ resolved ]` (singleton list of the resolved aspect with populated `includes`).

### 4a. Dedup key construction

**File:** `nix/lib/aspects/fx/handlers/check-dedup.nix`

The dedup key is `"${scope}/${identityKey}"` where:
- `scope` is `state.currentScope` (e.g., `"host:myhost"`)
- `identityKey` is `identity.key child` (provider path + name + optional ctxId suffix)

Synthetic names (wrapped in `<>`) and non-meaningful names produce `null` dedup keys and are never deduplicated. On first encounter of a non-null key, the handler eagerly registers it in `state.includeSeen` (a thunked attrset). Rollback is possible via `include-unseen` when a constraint excludes a previously-registered child.

### 4b. Emit-classes

**File:** `nix/lib/aspects/fx/handlers/emit-classes.nix`

For each class key and pipe key, unwraps the aspect content type to a list of modules via `unwrapContentValuesList`, then sends one `"emit-class"` effect per module element. Each emission carries:

- `class`: the class name (e.g., `"nixos"`, `"home"`)
- `identity`: node identity, with `[idx]` suffix for multi-element values
- `module`: the raw NixOS/HM module value
- `ctx`: context reconstructed from `__scopeHandlers` via `ctxFromHandlers`
- `aspectPolicy` / `globalPolicy`: collision policies for `wrapClassModule`
- `isContextDependent`: true when `__parametricResolvedArgs` overlap with context keys or `meta.contextDependent` is set
- `__isPipeEntry`: true for pipe keys (these bypass class module wrapping)

See: [class-module-wrapping.md](class-module-wrapping.md)

---

## 5. Parametric Compilation

**File:** `nix/lib/aspects/fx/handlers/compile-parametric.nix`, `nix/lib/aspects/fx/handlers/bind.nix`

Parametric compilation handles aspects with non-empty `__args` (function arguments awaiting resolution from scope handlers).

### Depth guard

Before any work, checks `__parametricDepth >= maxParametricDepth` (10). If exceeded, throws an error. The depth is incremented on each parametric recursion cycle.

### Gate

Runs `gateAndTag` (same as static). Skipped on parametric re-entry (`param.gated = true`).

### Bind

Sends a `"bind"` effect with `{ aspect, compileFn }`. The bind handler (`bind.nix`) determines argument availability:

1. **Required keys**: Filters `__args` to keys where `!childArgs.${k}` (no default value).
2. **Scope handler check**: Removes keys already present in `__scopeHandlers`.
3. **State fallback**: At non-root scopes, checks `state.scopeContexts` for remaining keys.
4. **Pipeline handler probe**: For any remaining keys, sequentially probes `fx.effects.hasHandler` for each key. If any probe fails, availability is false.
5. **Pipe arg detection**: If any required key matches a name in `den.quirks`, unconditionally defers (pipe data is assembled post-pipeline).

If all required args are available:
- Augments `__scopeHandlers` with state-context values for all requested keys (including optional ones)
- Calls `compileFn` which runs `prepareParametricFn` (the actual `fx.bind.fn` call), builds `mkParametricBase` + `mkParametricNext` + `tagParametricResult`, increments depth
- Returns `{ value = result; }`

If any required args are unavailable:
- Sends `"defer"` with `{ child, requiredKeys, requiredArgs, hasPipeArgs }`
- Returns `{ deferred = true; }`

### Re-resolution

When bind returns `{ value }`, the parametric compiler sends `"resolve"` with the result and `gated = true` (to skip the gate on re-entry). This re-enters the compile router, which will dispatch to `compile-static` if args are exhausted, or `compile-parametric` again if still curried.

### Defer

**File:** `nix/lib/aspects/fx/handlers/defer.nix`

The defer handler emits a stub via `resolve-complete` (recording the attempt in the path set) and queues the child in `state.scopedDeferredIncludes` for the current scope. Deferred includes are drained when entity context widens during policy installation.

---

## 6. Forward Compilation

**File:** `nix/lib/aspects/fx/handlers/compile-forward.nix`

Forward compilation handles aspects where `meta.__forward` is set. It bypasses the gate entirely (no dedup or constraint checks).

### Tier classification

The handler classifies the forward as Tier 1 (simple route) or complex:

**Tier 1** requires all of:
- `spec.canDirectImport && !spec.needsAdapter && !(spec.evalConfig or false)`
- Source aspect has no `__scopeHandlers` (local origin)
- Source class already exists in `state.scopedClassImports` for the current scope

**Tier 1 route shape:**
```
{ fromClass, intoClass, path = spec.staticIntoPath, guard = null, adaptArgs = null, sourceScopeId = scope }
```

**Complex route shape:**
```
spec // { sourceScopeId = scope, __complexForward = true }
```

### Route registration

The route is appended to `state.scopedRoutes` for the current scope via `scopedAppend`. The handler resumes with `[]` (no child results).

See: [routes-and-forwards.md](routes-and-forwards.md)

---

## 7. Conditional Compilation

**File:** `nix/lib/aspects/fx/handlers/compile-conditional.nix`

Conditional compilation handles aspects where `meta.guard` is set. It does not run the gate (conditionals have their own guard mechanism).

### Guard evaluation

1. Sends `"get-path-set"` to retrieve the set of all resolved aspect paths.
2. Builds a `guardCtx` with `hasAspect = ref: pathSet ? ${identity.key ref}`.
3. Calls `condNode.meta.guard guardCtx`.

### Outcomes

- **Guard passes**: Delegates `condNode.meta.aspects` to `emitIncludes` for normal resolution.
- **Guard fails**: Emits tombstones for all aspects in `meta.aspects` via `resolve-complete`.

---

## 8. Key Classification

**File:** `nix/lib/aspects/fx/key-classification.nix`

`classifyKeys` partitions all non-structural aspect keys into four categories:

### Structural keys (always excluded)

`name`, `description`, `meta`, `includes`, `provides`, `policies`, `into`, `classes`, `__fn`, `__args`, `__functor`, `__functionArgs`, `__scopeHandlers`, `__ctxId`, `__entityKind`, `__parametricResolvedArgs`, `_module`, `_`

### Classification logic

When both `den.classes` and `den.quirks` registries are empty (no batteries), all non-structural keys become `classKeys` (backward-compatible fallback).

When registries are populated:

| Category | Test |
|---|---|
| **Pipe keys** | Key exists in `den.quirks` registry |
| **Class keys** | Key exists in `den.classes` registry, or matches `targetClass` from class handler probe |
| **Nested keys** | Remaining keys whose unwrapped value contains recognized sub-keys (class registry members), depth-limited to 3 levels via `hasRecognizedSubKeys` |
| **Unregistered class keys** | Everything else (merged into `classKeys` by the classify handler) |

The `classify` handler (`classify.nix`) consumes this and returns `{ classKeys = classified.classKeys ++ classified.unregisteredClassKeys; nestedKeys; pipeKeys }`.

---

## 9. Identity

**File:** `nix/lib/aspects/fx/identity.nix`

Identity computation provides the path-based naming used for dedup, provenance, and constraint matching.

### aspectPath

```
aspectPath a = (a.meta.provider or []) ++ [a.name or "<anon>"] ++ optional (a ? __ctxId) "{${a.__ctxId}}"
```

Produces a list of path segments. Example: `["batteries" "git"]` or `["batteries" "ssh" "{myhost}"]`.

### pathKey

Joins a path list with `/`: `pathKey ["batteries" "git"] = "batteries/git"`.

### key

Composed: `key a = pathKey (aspectPath a)`. This is the primary identity string used throughout the pipeline.

### Provider chain

Nested aspects inherit their parent's provider chain. When `compile-static` builds a sub-aspect for a nested key `k`, it sets:
```
meta.provider = (aspect.meta.provider or []) ++ [aspect.name or "<anon>"]
```

### ctxId suffix

Parametric aspects resolved in different contexts get a `{ctxId}` suffix in their identity, ensuring the same aspect resolved for different hosts/users gets distinct identity keys.

### Utility functions

- `isAnonIdentity`: True for non-meaningful names, `<root>/` prefixed, or `/<anon>:` containing identities.
- `stripCtxSuffix`: Removes `/{ctxId}` suffix from an identity string.
- `tombstone`: Creates an excluded marker node with `name = "~${originalName}"` and `meta.excluded = true`.
- `collectPathsHandler`: Handles `resolve-complete` by recording the path (and base path without ctxId) in `state.pathSet`.
- `pathSetHandler`: Handles `get-path-set` by returning the current `state.pathSet`.

---

## 10. Resolve-Children

**File:** `nix/lib/aspects/fx/handlers/resolve-children.nix`, `nix/lib/aspects/fx/aspect/children.nix`, `nix/lib/aspects/fx/aspect/provide.nix`

The `resolve-children` handler orchestrates the post-emission child processing sequence.

### Chain wrapping

If the aspect has a meaningful name, the entire child sequence is wrapped with `chain-push` / `chain-pop` effects. These maintain the includes chain for provenance tracking and anonymous child naming.

### Child sequence

The sequence runs three steps in order via `fx.bind`:

1. **emitAspectPolicies** (`provide.nix`): Processes `aspect.provides` and `aspect.policies`.
   - **Self-provide**: If `provides.${aspectName}` exists, emits it as an `"emit-include"` effect (re-enters the resolution pipeline). Handles positional functions, parametric wrappers, and static values.
   - **Cross-provide shims**: For `provides` keys that are not the aspect's own name and not schema entity kinds, emits `"register-aspect-policy"` effects. Handles `to-hosts`, `to-users`, and named-target patterns.
   - **Aspect policies**: Each entry in `aspect.policies` is registered via `"register-aspect-policy"`.

2. **emitIncludes** (`children.nix`): Processes `aspect.includes` list sequentially. For each child:
   - Policy values (`__isPolicy`) are routed to `"register-aspect-policy"` instead of the aspect walk.
   - Lists are flattened, with policies extracted first.
   - Non-policy children are wrapped (`wrapChild`), given anonymous names if needed, then dispatched via `"resolve"`.
   - Parent `__scopeHandlers` and `__ctxId` propagate to children that lack them.

3. **installPolicies** (entity-only): Only runs if `aspect.__entityKind` is set (schema entity root). Dispatches global and aspect-scoped policies against the entity context.

### Completion

After all children resolve, the handler sends `"resolve-complete"` with the aspect (recording it in the path set), then returns the resolved aspect with populated `includes`.

See: [policy-system.md](policy-system.md), [scope-partitioning.md](scope-partitioning.md)

---

## 11. Invariants

- **Dedup is scope-prefixed**: The dedup key is `"${currentScope}/${identityKey}"`. The same aspect resolved in different entity scopes (e.g., `host:alpha` vs `host:beta`) will not collide.
- **Gate blocks entire subtree**: When the gate returns `{ blocked = true; }`, no children, nested aspects, or policies are processed. The compiler returns the gate's result directly.
- **Parametric depth is bounded**: `maxParametricDepth = 10`. Exceeded depth throws, preventing infinite recursion from self-referential parametric aspects.
- **Forward bypasses gate**: Forward compilation does not run dedup or constraint checks. Forwards are structural routing declarations, not aspect content.
- **Conditional bypasses gate**: Conditional compilation uses its own guard mechanism instead of the standard gate.
- **Eager dedup registration**: `check-dedup` registers the key in `includeSeen` on first encounter, before the aspect is fully resolved. This prevents concurrent resolution of the same aspect. Rollback via `include-unseen` handles constraint exclusions.
- **Pipe args force deferral**: If any required argument matches a `den.quirks` pipe name, the bind handler unconditionally defers, because pipe data is assembled post-pipeline.

---

## 12. Key Files

| File | Role |
|---|---|
| `handlers/compile.nix` | Shape router: dispatches to compile-{static,parametric,forward,conditional} |
| `handlers/compile-static.nix` | Static compilation: gate, classify, emit-classes, nested, resolve-children |
| `handlers/compile-parametric.nix` | Parametric compilation: gate, bind, re-resolve or defer |
| `handlers/compile-forward.nix` | Forward compilation: tier classification, route registration |
| `handlers/compile-conditional.nix` | Conditional compilation: guard evaluation against path set |
| `handlers/gate.nix` | Gate composite: check-dedup then check-constraint |
| `handlers/gate-tag.nix` | Shared gate+tag wrapper for static and parametric compilers |
| `handlers/check-dedup.nix` | Dedup via scope-prefixed `includeSeen` state |
| `handlers/constraint.nix` | Constraint registry: register-constraint, check-constraint |
| `handlers/bind.nix` | Parametric bind: probe scope, call compileFn or defer |
| `handlers/defer.nix` | Deferred include: stub + queue in scopedDeferredIncludes |
| `handlers/emit-classes.nix` | Class/pipe emission: unwrap content, send emit-class per module |
| `handlers/classify.nix` | Classify handler: calls classifyKeys, merges unregistered into classKeys |
| `handlers/resolve.nix` | Resolve handler: trivial delegation to compile |
| `handlers/resolve-children.nix` | Child orchestration: policies, includes, entity policies, completion |
| `key-classification.nix` | Key classification: structural/class/pipe/nested/unregistered partitioning |
| `identity.nix` | Identity: aspectPath, pathKey, key, tombstone, path set handlers |
| `aspect/children.nix` | Include walker: wrapChild, anonymous naming, resolve dispatch |
| `aspect/provide.nix` | Policy + provide emission: self-provide, cross-provide shims, aspect policies |

All file paths are relative to `nix/lib/aspects/fx/`.
