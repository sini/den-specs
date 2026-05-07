# Scope-Partitioned Pipeline State

**Branch:** feat/fx-pipeline (753/753 tests)

---

## 1. Overview

The fx-pipeline partitions all mutable output state by *scope* — a context-derived identity that isolates emissions from different entity levels (flake, host, user). Each scope has its own class imports, routes, provides, policies, and constraints. Scopes form a tree tracked by flat maps in pipeline state. The pipeline walks entity trees depth-first, entering and exiting scopes via `push-scope`/`restore-scope` effects, while `scope.provide` keeps algebraic effect handlers synchronized with the current scope's context.

---

## 2. Scope Identity

### mkScopeId

`mkScopeId` (defined in `pipeline.nix`) computes a canonical string from a context attribute set. It is the sole source of scope identity throughout the pipeline.

**Algorithm:**

1. Extract attribute names from the context, sort lexicographically.
2. For each key, extract a string value:
   - Attrsets with a `name` field: use `v.name` (entity references like `{ name = "igloo"; ... }`).
   - Strings: use the string directly.
   - Integers and floats: use `toString`.
   - Anything else: encode as `<type:key>` (e.g., `<lambda:builder>`).
3. Join as `key=value` pairs separated by commas.

**Examples:**

```
{}                                                  -> ""
{ host = { name = "igloo"; }; }                     -> "host=igloo"
{ host = { name = "igloo"; }; user = { name = "tux"; }; }
                                                    -> "host=igloo,user=tux"
```

**Injectivity:** The separator characters (`=`, `,`, `<`, `>`) cannot appear in Nix attribute names (alphanumeric + hyphens), so distinct contexts always produce distinct scope IDs. The `<type:key>` fallback for exotic values preserves injectivity by encoding both the type and the key name.

**Edge cases:**

- Empty context `{}` produces `""` — the root scope for flake-level resolution.
- Single-key contexts produce `"host=igloo"` (no commas).

### mkCtxId (vestigial)

The older `mkCtxId` helper (which dropped key names, producing e.g. `igloo,tux`) has been fully removed from the codebase. All scope identity uses `mkScopeId`.

---

## 3. Scope Tree

Scopes form a tree tracked by two flat maps in pipeline state. The tree structure is determined by policy-driven context expansion (e.g., a host-resolve policy creates child scopes for each user), not by scope ID string structure.

### scopeParent

Flat map from child scope ID to parent scope ID. Populated by `push-scope` when a new scope is created. Root scope has no entry.

```nix
scopeParent = {
  "host=igloo" = "";                          # parent is root
  "host=igloo,user=tux" = "host=igloo";       # parent is host scope
  "host=igloo,user=pingu" = "host=igloo";
};
```

When `push-scope` detects that `newScopeId == parentScope` (same scope re-entered), no entry is added — the scope is already known.

### scopeContexts

Flat map from scope ID to the full context attribute set at that scope. Populated alongside `scopeParent`. Used post-pipeline by `fxResolve` to wrap each scope's class modules with the correct entity context.

```nix
scopeContexts = {
  "" = {};
  "host=igloo" = { host = { name = "igloo"; ... }; };
  "host=igloo,user=tux" = { host = ...; user = { name = "tux"; ... }; };
};
```

### Tree shape (typical fleet)

```
""                                              <- root (flake scope)
+-- "host=igloo"                                <- host scope
|   +-- "host=igloo,user=tux"                   <- user scope
|   +-- "host=igloo,user=pingu"                 <- user scope
+-- "host=server"
    +-- "host=server,user=admin"
```

For scope creation mechanics during entity resolution, see [entity-resolution.md](entity-resolution.md).

---

## 4. Scope-Partitioned State

All fields from `defaultState` in `pipeline.nix`, classified by partitioning strategy.

### Scoped fields (keyed by scope ID)

Each stored as `scopedX.${scopeId}` in state. Emission handlers write to `scopedX.${state.currentScope}`. All are thunk-wrapped (`_: value`) to survive `deepSeq` at each trampoline step.

| Field | Init | Purpose |
|-------|------|---------|
| `scopedClassImports` | `{}` per scope | Class modules emitted per scope, keyed by class name. Core pipeline output. |
| `scopedRoutes` | `{}` per scope | `policy.route` specs registered per scope. Consumed by post-pipeline phase 3. |
| `scopedInstantiates` | `{}` per scope | `policy.instantiate` specs per scope. Consumed by post-pipeline phase 4. |
| `scopedProvides` | `{}` per scope | `policy.provide` specs per scope. Consumed by post-pipeline phase 2. |
| `scopedPipeEffects` | `{}` per scope | Pipe effect registrations per scope. Consumed by `assemblePipes`. |
| `scopedAspectPolicies` | `{}` per scope | Policies discovered during walk, keyed by policy name. Fire when args match context. |
| `scopedDeferredIncludes` | `[]` per scope | Includes deferred until context widens. Drained when scope context satisfies args. |
| `scopedIncludesChain` | `[]` per scope | Chain stack tracking include nesting depth per scope. |
| `scopedConstraintRegistry` | `{}` per scope | Constraint predicates registered per scope. |
| `scopedConstraintFilters` | `[]` per scope | Constraint filter functions per scope. |
| `scopedEmittedLocs` | `{}` per scope | Tracks emitted module locations for dedup within a scope. |
| `scopeEntityClass` | `{}` per scope | Entity class per scope (e.g., `"nixos"`). Separate from `scopeContexts` to avoid affecting provides/enrichment. Read by bind's state fallback and subtree extraction. |
| `scopeSourcePolicy` | `{}` per scope | Source policy name per scope. Prevents self-referential policy dispatch cycles. |

### Flat caches (incrementally maintained)

| Field | Init | Purpose |
|-------|------|---------|
| `flatAspectPolicies` | `{}` | Pre-merged view of all aspect policies. Avoids O(S) cross-scope merge per dispatch call. |
| `flatConstraintRegistry` | `{}` | Pre-merged constraint entries. O(1) lookup per constraint check. |
| `flatConstraintFilters` | `[]` | Pre-merged filter list. |

### Global fields (shared pipeline-wide)

| Field | Init | Purpose |
|-------|------|---------|
| `currentScope` | `mkScopeId ctx` (root) | Active scope for emissions. Set by `push-scope`, restored by `restore-scope`. |
| `rootScopeId` | `mkScopeId ctx` | Immutable root scope identity. |
| `scopeParent` | `{}` | Child-to-parent scope map. Grows as scopes are created. |
| `scopeContexts` | `{ ${rootScopeId} = ctx; }` | Scope-to-context map. Grows as scopes are created. |
| `seen` | `{}` | Dedup for context-aware emissions (keys prefixed by scope). |
| `pathSet` | `{}` | Per-scope `hasAspect` tracking (keys prefixed by scope). |
| `includeSeen` | `{}` | Include dedup (keys prefixed by scope). |
| `firedPolicyNames` | `{}` | Policies already fired, keyed by dispatch key. |
| `dispatchedPolicies` | `{}` | Policy dispatch tracking. |
| `registeredRouteKeys` | `{}` | Route dedup keys. |
| `inLateDispatch` | `false` | Guard flag for late dispatch pass. |

---

## 5. Scope Lifecycle

### push-scope

The `push-scope` effect handler (`handlers/push-scope.nix`) atomically creates a new scope. It receives `{ scopedCtx, entityClass, parentScope, sourcePolicyName? }` and performs:

1. **Compute identity:** `newScopeId = mkScopeId scopedCtx`.
2. **Set current scope:** `state.currentScope = newScopeId`.
3. **Record tree:** Add `scopeContexts.${newScopeId} = scopedCtx` and `scopeParent.${newScopeId} = parentScope` (skipped if same-scope re-entry).
4. **Initialize policy partition:** Ensure `scopedAspectPolicies.${newScopeId}` exists (empty if new). No parent policy inheritance — policies fire where registered, cascade is through effects.
5. **Record entity class:** If `entityClass` is non-null, store in `scopeEntityClass.${newScopeId}`.
6. **Record source policy:** If `sourcePolicyName` is non-null, store in `scopeSourcePolicy.${newScopeId}`. This prevents the source policy from re-dispatching at entities it created.
7. **Propagate deferred includes:** Copy parent scope's deferred includes to child scope (appended, not replaced).
8. **Save and reset late dispatch:** Push current `inLateDispatch` onto `inLateDispatchStack`, then set `inLateDispatch = false`. Each scope level gets its own late-dispatch opportunity.

**Return value:** `{ scopeHandlers, scopeId }` — the caller uses `scopeHandlers` with `scope.provide` to install context handlers for the new scope.

### restore-scope

The `restore-scope` effect handler (`handlers/restore-scope.nix`) restores state after exiting a child scope. It receives `{ parentScope }` and performs:

1. **Restore current scope:** `state.currentScope = parentScope`.
2. **Restore late dispatch flag:** Pop `inLateDispatchStack` to restore the parent's `inLateDispatch` value.

The handler does not clean up scope tree entries — `scopeParent` and `scopeContexts` are append-only and persist for post-pipeline consumption.

### Typical call pattern

```
push-scope { scopedCtx; entityClass; parentScope = state.currentScope; }
  -> { scopeHandlers, scopeId }
scope.provide scopeHandlers (
  ... walk entity's aspect tree, emit into scopedX.${scopeId} ...
)
restore-scope { parentScope; }
```

The `scope.provide` call is lexically scoped: when the continuation completes, effect handlers automatically revert to the parent's handlers.

---

## 6. Sync Invariant

The pipeline uses two parallel mechanisms during scope transitions:

1. **`scope.provide`** (nix-effects algebraic effect): Installs scoped effect handlers (entity context like `host`, `user`). Lexically scoped — handlers revert when the continuation completes. Controls what parametric aspects can resolve via `has-handler`.

2. **`state.modify`** (nix-effects state effect): Updates `currentScope` in pipeline state. Controls which partition emission handlers write to.

**These must stay synchronized.** Desynchronization manifests as:

- `scope.provide` active for scope X, but `currentScope` still at parent Y: emissions from aspects resolving in X's context land in Y's partition.
- `currentScope` at X, but `scope.provide` reverted to parent: parametric aspects fail `has-handler` probes for X's context while emissions still target X.

**How sync is maintained:**

1. `push-scope` sets `currentScope = childScopeId` via state update.
2. `scope.provide` installs child context handlers and runs the child computation.
3. When `scope.provide` returns, `restore-scope` runs `currentScope = parentScope` via state update.

`scope.provide`'s lexical scoping is the safety net: even if the child computation throws, the handler frame is restored. The `restore-scope` call is explicit in the continuation, ensuring it runs after `scope.provide` returns.

**Sequential sibling processing.** Siblings (e.g., multiple users of a host) are processed sequentially in the `fx.bind` chain. Nix's strict evaluation prevents interleaving. The scope enter/exit for sibling A completes entirely before sibling B begins.

---

## 7. Late Dispatch

### Problem

When a scope fans out into multiple siblings (e.g., a host-resolve policy creates user scopes for tux, pingu, admin), policies registered by earlier siblings are not visible to later siblings during the main walk. This is because policies accumulate in `scopedAspectPolicies` as siblings are processed sequentially — tux's policies are registered after tux's subtree is walked, but before pingu's walk begins.

### Mechanism

After all siblings in a fan-out complete their main walk, a **late dispatch pass** (`lateDispatchPass` in `policy/schema.nix`) re-visits each sibling scope:

1. **Set guard:** `state.modify: { inLateDispatch = true }`.
2. **For each sibling:** Collect all aspect policies registered so far (`flatAspectPolicies`), subtract policies already fired at that sibling's scope (`firedPolicyNames`), and dispatch the remainder.
3. **Per-sibling scope entry:** Each sibling gets a `push-scope`/`scope.provide`/`restore-scope` cycle, dispatching newly-visible policies within the correct scope context.

### inLateDispatch guard

The `inLateDispatch` flag prevents recursive late dispatch. When `processSchemaResolves` detects a fan-out (multiple schema effects), it checks:

- If `inLateDispatch` is already true: skip the late pass (already inside one).
- If false: run the main fold, then run `lateDispatchPass`, then return results.

Each scope level maintains its own `inLateDispatch` independently. `push-scope` saves the current value on `inLateDispatchStack` and resets to `false`. `restore-scope` pops the saved value. This allows nested fan-outs (e.g., fleet -> hosts -> users) to each have their own late dispatch opportunity without interference.

### One pass per scope level

Late dispatch runs exactly once per fan-out at each scope level. The guard prevents O(N^2) re-dispatch within the same level while allowing each nesting depth its own pass.

---

## 8. Post-Pipeline Scope Usage

After the pipeline completes (all scopes created, all emissions collected), `fxResolve` in `resolve.nix` assembles final output through four phases that consume scope-partitioned data:

### Phase 1: Per-scope wrapping

Each scope's `scopedClassImports` are wrapped with that scope's context from `scopeContexts`. This is the mechanism that makes `{ host, config, ... }: ...` class modules work: den-specific args (`host`, `user`) are pre-applied from the scope's context, NixOS args (`config`, `pkgs`) pass through.

Modules are deduplicated by key across scopes — when a shared aspect is included by both host and user scopes, the first occurrence wins.

### Phase 2: Provide application

`scopedProvides` are collected across all scopes, deduplicated by composite key (`policyName/class/path`), and injected into target classes. Each provide spec carries its `sourceScopeId`, used to look up the correct context for wrapping.

### Phase 3: Route application

`scopedRoutes` are applied per-scope. Routes are processed with scope-aware context from `scopeContexts`. For route mechanics, see [routes-and-forwards.md](routes-and-forwards.md).

### Phase 4: Instantiate application

`scopedInstantiates` trigger entity output evaluation. For fleet walks, the implementation re-runs phases 1-3 per host subtree using `extractSubtreeModules`, which walks `scopeParent` to collect all descendant scopes of a host scope. This produces correct routing while reusing the walk's scope data.

### Pipe assembly (between pipeline and phases)

Before phase 1, `assemblePipes` consumes `scopedPipeEffects` to augment `scopeContexts` with pipe-derived data. This produces `augmentedScopeContexts` used by all subsequent phases. For pipe mechanics, see [pipes-and-quirks.md](pipes-and-quirks.md).

---

## 9. Invariants

1. **Injective scope IDs.** `mkScopeId` is injective over context attribute sets. Distinct contexts always produce distinct scope IDs.

2. **Parent always exists.** Every entry in `scopeParent` references a scope ID that exists in `scopeContexts`. The root scope is initialized in `mkPipeline` before any child scope can be created.

3. **Routing kinds transparent.** Scope partitioning is orthogonal to entity kind (host, user, fleet). The same scope machinery handles all entity types. Kind-specific behavior lives in entity resolution and policy dispatch, not in scope management.

4. **Append-only tree.** `scopeParent` and `scopeContexts` grow monotonically during the pipeline. Entries are never removed or modified. Post-pipeline phases can safely read them without synchronization concerns.

5. **Late dispatch is bounded.** Each scope level gets exactly one late dispatch opportunity. The `inLateDispatch` flag + stack mechanism prevents recursive re-dispatch.

6. **Scope isolation.** Emissions always target `state.currentScope`. There is no mechanism for a handler to write to a scope other than the current one. Cross-scope data flow happens only post-pipeline through explicit phase logic.

---

## 10. Key Files

| File | Purpose |
|------|---------|
| `nix/lib/aspects/fx/pipeline.nix` | `mkScopeId`, `defaultState`, `mkPipeline` — scope identity and initial state |
| `nix/lib/aspects/fx/handlers/push-scope.nix` | `push-scope` effect handler — scope creation |
| `nix/lib/aspects/fx/handlers/restore-scope.nix` | `restore-scope` effect handler — scope exit |
| `nix/lib/aspects/fx/resolve.nix` | `fxResolve` — post-pipeline assembly consuming scope-partitioned state |
| `nix/lib/aspects/fx/policy/schema.nix` | `lateDispatchPass` — late dispatch mechanism |
| `nix/lib/aspects/fx/assemble-pipes.nix` | `assemblePipes` — pipe data injection into scope contexts |
| `nix/lib/aspects/fx/handlers/provide.nix` | `provideHandler` — scope-aware provide registration |
| `nix/lib/aspects/fx/handlers/class-collector.nix` | Scope-aware class import emission |
