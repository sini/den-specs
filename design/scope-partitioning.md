# Scope-Partitioned Pipeline State

**Date:** 2026-05-04
**Branch:** feat/fx-pipeline (629/629 tests)
**Purpose:** Design spec describing scope partitioning as implemented today

---

## 1. mkScopeId: Injective Scope Identity

`mkScopeId` computes a deterministic string from a context attribute set. The format is comma-separated `key=value` pairs, sorted lexicographically by key name.

**Value extraction rules:**
- Attrsets with a `name` attribute: use `v.name` (entity references like `{ name = "igloo"; users = ...; }`)
- Strings: use the string directly
- Anything else: encode as `<type:key>` (e.g., `<int:priority>`)

**Example:**
```
{ host = { name = "igloo"; }; user = { name = "tux"; }; }
  -> "host=igloo,user=tux"
```

**Injectivity guarantee:** The separator characters (`=`, `,`, `<`, `>`) cannot appear in Nix identifiers (alphanumeric + hyphens), so no two distinct contexts can produce the same scope ID. The type+key fallback for non-string, non-attrset values (`<int:priority>`) preserves injectivity for exotic context values.

**Relationship to mkCtxId:** The older `mkCtxId` drops key names (producing e.g. `igloo,tux`), which is safe for dedup but not for partitioning. `mkScopeId` replaced it for all scope-partitioning uses. `mkCtxId` is retained only in `ctx-seen` dedup logic.

**Edge cases:**
- Empty context `{}` produces `""` (the root scope for flake-level resolution).
- Single-key contexts produce `"host=igloo"` (no commas).
- Integer context values (e.g., from enrichment) use the type fallback.

---

## 2. Scope Tree: scopeParent and scopeContexts

Scopes form a tree determined by policy-driven context expansion. The tree is tracked by two flat maps in pipeline state, not derived from scope ID string structure.

**`scopeParent`** — flat map from child scope ID to parent scope ID:
```
scopeParent = {
  "host=igloo" = "";                          # parent is root
  "host=igloo,user=tux" = "host=igloo";       # parent is host scope
  "host=igloo,user=pingu" = "host=igloo";
};
```

Populated at scope creation time when `resolve-schema-entity` processes a `policy.resolve` effect. The handler records `scopeParent.${childScopeId} = state.currentScope` before entering the child scope.

**`scopeContexts`** — flat map from scope ID to the full context attrset at that scope:
```
scopeContexts = {
  "" = {};
  "host=igloo" = { host = { name = "igloo"; ... }; };
  "host=igloo,user=tux" = { host = { name = "igloo"; ... }; user = { name = "tux"; ... }; };
};
```

Populated alongside `scopeParent`. Used post-pipeline by `fxResolve` to wrap each scope's class modules with the correct entity context (for `wrapClassModule` partial application).

**Tree shape example (typical multi-user host):**
```
""                                              <- root (flake scope)
+-- "host=igloo"                                <- host scope
|   +-- "host=igloo,user=tux"                   <- user scope
|   +-- "host=igloo,user=pingu"                 <- user scope
+-- "host=server"
    +-- "host=server,user=admin"
```

**What was removed:** The original spec included `scopeChildren` (child scope ID list per parent), `scopeStack` (explicit stack for push/pop), and `scopeProvenance` (policy name that created a scope). All three were eliminated during the cleanup arc (Phase 5). `scope.provide`'s lexical scoping makes an explicit stack unnecessary. `scopeChildren` is not needed because post-pipeline operations iterate `scopeParent` keys or `scopedClassImports` keys directly. `scopeProvenance` was trace metadata only.

---

## 3. State Fields: Complete Inventory

All fields from `defaultState` in `pipeline.nix`, classified as **scoped** (per-scope partition keyed by scope ID), **prefixed** (shared map with scope-prefixed keys), or **global** (single value shared across all scopes).

### Scoped fields (per-scope partition)

Each stored as `scopedX.${scopeId}` in state. Emission handlers write to `scopedX.${state.currentScope}`.

| Field | Init | Description |
|-------|------|-------------|
| `scopedClassImports` | `{}` per scope | Class modules emitted per scope, keyed by class name. Core output of the pipeline. |
| `scopedForwardSpecs` | `[]` per scope | Forward specs registered in each scope, processed post-pipeline. |
| `scopedAspectPolicies` | `{}` per scope | Policies discovered during tree walk, keyed by policy name. Fire when their args are satisfied by current context. |
| `scopedDeferredIncludes` | `[]` per scope | Includes deferred until context widens (parametric aspects needing unavailable handlers). Drain into the scope that deferred them. |
| `scopedConstraintRegistry` | `{}` per scope | Constraint predicates registered per scope. |
| `scopedConstraintFilters` | `[]` per scope | Constraint filter functions per scope. |
| `scopedIncludesChain` | `[]` per scope | Chain stack tracking include nesting depth per scope. |
| `scopedRoutes` | `[]` per scope | `policy.route` specs registered per scope. Processed post-pipeline. |
| `scopedInstantiates` | `[]` per scope | `policy.instantiate` specs registered per scope. Processed post-pipeline. |

### Prefixed fields (scope-prefixed keys in a shared map)

| Field | Key format | Description |
|-------|------------|-------------|
| `seen` (ctxSeen) | `${scopeId}/${ctxKey}` | Dedup for context-aware emissions. Same context key processable in different scopes without collision. |
| `includeSeen` | `${scopeId}/${aspectIdentity}` | Include dedup. Same aspect includable in different scopes (preserving the semantic from when scopes were isolated sub-pipelines with fresh state). |
| `pathSet` | `${scopeId}/${aspectPath}` | Per-scope `hasAspect` tracking. |

### Global fields (shared pipeline-wide)

| Field | Init | Description |
|-------|------|-------------|
| `currentScope` | root scope ID (`mkScopeId ctx`) | Active scope for emissions. Set/restored during scope transitions. |
| `class` | target class name | Pipeline-wide target class. Immutable after init. |
| `scopeParent` | `{}` | Flat child-to-parent scope map. Grows as scopes are created. |
| `scopeContexts` | `{ ${rootScopeId} = ctx; }` | Scope-to-context map. Grows as scopes are created. |

### Fields removed from the original spec

| Field | Original classification | Why removed |
|-------|----------------------|-------------|
| `scopeStack` | Global (stack of parent scope IDs) | `scope.provide` lexical scoping eliminates explicit push/pop stack. |
| `scopeChildren` | Scoped (child scope ID list) | Not needed; post-pipeline iterates scope keys directly. |
| `scopeProvenance` | Scoped (policy name metadata) | Trace metadata only; removed for simplicity. |
| `scopedTraits` | Scoped | Trait system deleted for fleet/den.exports reimplementation. |
| `scopedDeferredTraits` | Scoped | Deleted with traits. |
| `scopedConsumedTraits` | Scoped | Deleted with traits. |
| `traitSchemas` | Global | Deleted with traits. Dynamic registration design validated but removed. |
| `scopedDeadLetterQueue` | Scoped | DLQ eliminated; unregistered keys emit as classes directly. |
| `dispatchedPolicies` | Global | Replaced by `installPolicies` per-scope dispatch. |
| `transitionDepth` | Global | Transition boundaries eliminated. |
| `currentCtx` | Global | Replaced by `scopeContexts.${currentScope}` lookup. |
| `provideTo` | Scoped | Cross-entity routing replaced by scope inheritance. |

---

## 4. Scope Transitions via resolve-schema-entity

`resolve-schema-entity` is the reusable handler that performs entity resolution with scope management. It replaces the monolithic `into-transition` handler from `transition.nix` (deleted, -1011 lines).

**Trigger:** A `policy.resolve` effect fires during `installPolicies` dispatch in `aspect.nix`.

**Sequence:**

1. **Enrich context.** Merge the resolve effect's bindings into the current context: `enrichedCtx = currentCtx // eff.bindings`.

2. **Compute child scope.** `childScopeId = mkScopeId enrichedCtx`.

3. **Check dedup.** If `childScopeId` already exists in `scopeContexts`, this scope was already processed (fan-out dedup). Skip.

4. **Record scope tree.**
   ```
   state.modify: {
     scopeParent.${childScopeId} = state.currentScope;
     scopeContexts.${childScopeId} = enrichedCtx;
     currentScope = childScopeId;
   }
   ```

5. **Install context handlers.** `scope.provide` installs entity context handlers (via `constantHandler` closures) for the child scope. This makes `host`, `user`, etc. available to parametric aspects via `has-handler`/`perform` within the scope.

6. **Look up entity.** `resolveEntity` reads the entity config from `den.schema` / `den.hosts` / `den.users` etc.

7. **Walk entity's aspect tree.** `aspectToEffect` processes the entity's includes. All emissions (`emit-class`, `emit-forward`, etc.) write to `scopedX.${childScopeId}` because `state.currentScope` was set in step 4.

8. **Install child-scope policies.** `installPolicies` runs within the child scope, dispatching any policies whose args are now satisfied by `enrichedCtx`. This may trigger recursive scope creation (e.g., host scope creates user scopes).

9. **Restore scope.** When `scope.provide` returns (step 5's lexical scoping), context handlers automatically revert to the parent's handlers. State is restored: `state.modify: { currentScope = parentScopeId; }`.

**Invariant: scope.provide and state.modify must stay in sync.** `scope.provide` controls which context handlers are visible to `has-handler` probes and parametric aspect resolution. `state.currentScope` controls which partition emissions write to. If these diverge, emissions land in the wrong scope or aspects resolve against the wrong context. Both are set in the same `fx.bind` chain with deterministic ordering, and `scope.provide`'s lexical restore guarantees handler cleanup. The `state.modify` restore runs in the continuation after `scope.provide` returns.

**Enrichment iteration.** Within a scope, `installPolicies` may discover that firing one policy widens context enough to satisfy additional policies. This is handled by natural recursion: each `policy.resolve` is processed immediately, widening context for subsequent policies. Fired policies are removed from candidates. A max recursion depth cap (matching the former `maxEnrichmentIterations = 10`) prevents divergence.

---

## 5. Post-Pipeline Composition in fxResolve

After the pipeline executes (walking the entire aspect/entity tree, creating all scopes, collecting all emissions), `fxResolve` composes the final output. The sequence is:

### Step 1: Per-scope wrapping

Each scope's class imports are wrapped with that scope's entity context for `wrapClassModule` partial application:

```
wrappedPerScope = mapAttrs (scopeId: classImports:
  wrapCollectedClasses (scopeContexts.${scopeId}) classImports
) scopedClassImports;
```

This is the mechanism that makes `{ host, config, ... }: ...` class modules work: den-specific args (`host`) are pre-applied from the scope's context, NixOS args (`config`, `pkgs`) pass through.

**Per-scope wrapping prevents cross-entity contamination.** Tux's modules are wrapped with `{ user = tux; }`, pingu's with `{ user = pingu; }`. No mixing.

### Step 2: Route application

`policy.route` specs are applied per-scope. Each route reads source content from `wrappedPerScope.${route.sourceScopeId}.${route.fromClass}` and injects it into the target class via `wrapRouteModules` (path nesting + optional guard + optional adaptArgs).

Routes apply before forwards. This ordering matters because complex forwards may read from post-route data.

### Step 3: Forward application

Remaining `forwardSpecs` (complex forwards that couldn't be expressed as routes) are applied per-scope. Each forward reads from the scope partition's class imports. No sub-pipelines: `buildForwardAspect` reads already-collected modules from `wrappedPerScope.${scopeId}`.

**Forward scope isolation.** Forwards from child scopes need access to root-scope content (e.g., `den.default` modules). The implementation uses filtered root fallback: only `@default` identity modules from the root scope are visible to child-scope forwards. This prevents child scopes from seeing all root content while still providing shared defaults.

### Step 4: Flatten across scopes

All scopes' results are flattened into the final class imports for the target class:

```
allImports = concatMap (scopeId:
  forwardedPerScope.${scopeId}.classImports.${class} or []
) allScopes;
```

After wrapping and forwarding, results are opaque NixOS modules. Flattening is safe because each module carries its own context via partial application.

### Step 5: Instantiate application

`policy.instantiate` specs trigger entity output evaluation. Each instantiate reads the entity's `mainModule` (which uses `fxResolve` lazily) and places the evaluated config at the entity's `intoAttr` in the flake output.

---

## 6. The sync invariant: scope.provide and state.modify

The pipeline uses two parallel mechanisms during scope transitions:

1. **`scope.provide`** (nix-effects primitive): Installs scoped effect handlers (entity context like `host`, `user`) that are lexically scoped. When the continuation completes, handlers automatically revert. Controls what parametric aspects can resolve via `has-handler`.

2. **`state.modify`** (nix-effects primitive): Updates the mutable pipeline state, specifically `currentScope`. Controls which partition emission handlers write to.

These must stay synchronized. Desynchronization manifests as:
- `scope.provide` active for scope X, but `currentScope` still points to parent Y: emissions from aspects resolving in scope X's context land in Y's partition.
- `currentScope` points to X, but `scope.provide` has reverted to parent: parametric aspects fail `has-handler` probes for X's context while emissions still go to X.

**How sync is maintained:** Both operations occur in the same `fx.bind` chain:
1. `state.modify` sets `currentScope = childScopeId`
2. `scope.provide` installs child context handlers and runs the child computation
3. When `scope.provide` returns, the continuation runs `state.modify` to restore `currentScope`

`scope.provide`'s lexical scoping is the safety net: even if the child computation throws or short-circuits, the handler frame is restored. The `state.modify` restore is explicit in the continuation, so it always runs after `scope.provide` returns.

**Sequential sibling processing.** Within a scope, siblings (e.g., multiple users of a host) are processed sequentially in the `fx.bind` chain. Nix's strict evaluation model prevents interleaving. The scope set/restore for sibling A completes entirely before sibling B begins. No concurrent access to `currentScope`.

---

## 7. What Was Removed From the Original Spec

The original scope-partitioned pipeline state spec (2026-04-29) described a larger system. The cleanup arc (Phase 5, May 1-2) simplified it substantially. Here is what was removed and why.

### scopeStack
**Original:** Explicit stack of parent scope IDs, pushed on scope entry, popped on exit. Used to restore `currentScope` after child processing.
**Removed because:** `scope.provide` provides lexical scoping natively. The handler frame restore is automatic. `currentScope` restore is a simple set-to-saved-value in the continuation, not a stack pop. The stack was redundant with the bind chain structure.

### scopeChildren
**Original:** Per-scope list of child scope IDs, for aggregation walks (e.g., trait inheritance up the tree via `scopeChildren`).
**Removed because:** Post-pipeline operations iterate `builtins.attrNames scopedClassImports` or `builtins.attrNames scopeParent` directly. Trait inheritance (the primary consumer) was deleted. If trait reimplementation needs child lists, they can be derived from `scopeParent` by inverting the map.

### scopeProvenance
**Original:** Per-scope record of which policy created the scope. Trace metadata for debugging.
**Removed because:** Simplification. Not used by any pipeline logic. Can be reintroduced as part of structured tracing if needed.

### Trait state fields (scopedTraits, scopedDeferredTraits, scopedConsumedTraits, traitSchemas)
**Original:** Trait system with schema-driven classification, per-scope trait collection, deferred trait evaluation, consumed-trait tracking for `partialOk` validation, and dynamic trait schema registration.
**Removed because:** The trait system (-2463 lines across 23 files) was deleted as pre-work for the transition elimination redesign. Traits interacted with every pipeline concern (DLQ, transitions, dispatch, forwards). Removing them enabled the cleanup arc's simplifications. Reimplementation planned as pipeline-time-only traits (see `design/traits.md`) + fleet/`den.exports` for config-dependent cross-host data (see `design/fleet-and-exports.md`).

### Dead letter queue (scopedDeadLetterQueue, drain-dead-letters)
**Original:** Unregistered keys queued as dead letters, reclassified when schemas registered later.
**Removed because:** With traits deleted, the only unregistered keys are class keys. These emit as classes directly. The DLQ mechanism (-178 lines) was eliminated.

### dispatchedPolicies
**Original:** Global set tracking which policies have fired, for dedup.
**Removed because:** `installPolicies` uses `scope.provide`-based dispatch with natural recursion. Fired policies are removed from the candidate set within each scope's dispatch, not tracked globally.

### Dual-write pattern
**Original:** All emission handlers wrote to both flat state (e.g., `classImports`) and scoped partitions (e.g., `scopedClassImports`). The flat fields were for backward compatibility during migration.
**Removed because:** `fxResolve` reads exclusively from scoped partitions. Flat fields are not needed. Single-write to scoped partitions only.

---

## 8. Planned Redesign: Scope-Widened Effect

**Spec:** `design/unified-resolve-effects.md`

### New State Fields

| Field | Init | Description |
|-------|------|-------------|
| `flatConstraintRegistry` | `{}` | Pre-merged flat view of all constraint entries (O(1) lookup per check) |
| `flatConstraintFilters` | `[]` | Pre-merged flat filter list |
| `flatAspectPolicies` | `{}` | Pre-merged flat aspect policy map |

These flat caches are maintained incrementally by `register-constraint` and `register-aspect-policy` handlers, eliminating O(S) cross-scope merges on every check/dispatch call.

### Drain Mechanism Change

- **Current:** `drain-deferred` is an explicit effect sent at two call sites (resolve-schema-entity, policy/iterate).
- **After redesign:** `drain` replaces `drain-deferred` (same semantics). Additionally, `scope-widened` provides automatic drain for simple scope entries via `enterScope` wrapper.
- **Entity resolution:** Retains explicit `drain` at its specific orchestration point. Does NOT use `enterScope`.

### Sync Invariant (Section 6) — Unchanged

The `scope.provide` / `state.modify` sync invariant is preserved. `enterScope` wraps `scope.provide` and emits `scope-widened` inside the new scope (after handlers are installed). The `scope-widened` handler reads state that was already updated by `pushScope` (which runs before `scope.provide`).
