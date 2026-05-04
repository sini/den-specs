# Transition Elimination — Policies as Context Expansion

**Date**: 2026-05-01
**Status**: Substantially shipped — 612/616 tests pass (99.4%), 4 edge cases remain
**Scope**: Replace transition boundaries with policy-driven context expansion

### Implementation Status (2026-05-02)

**Phase 1: Core transition elimination (shipped)**
- transition.nix deleted (-1011 lines)
- DLQ eliminated — unregistered keys emit as classes immediately (-178 lines)
- Trait system deleted for fleet/den.exports reimplementation (-2463 lines across 23 files)
- `dispatch-policies.nix` deleted, replaced by inline `installPolicies` in aspect.nix using `scope.provide`
- Forward sub-pipelines replaced with scope-lookup (buildForwardAspect reads wrappedPerScope)
- State fields removed: `scopeStack`, `scopeChildren`, `scopeProvenance`, `scopedDeadLetterQueue`, `scopedDeferredTraits`, `dispatchedPolicies`
- Net: **-3024 lines**

**Phase 1b: Forward scope isolation (shipped)**
- mkScopeId integer handling fix
- Per-policy include pairing for `resolve.to` (cross-provider includes scoped to their policy)
- `policy.resolve __includes` API (per-resolve include scoping)
- Filtered forward root fallback (only `@default` identity modules from root scope)
- Post-pipeline forward source resolution (mini-pipeline for synthetic fromAspect)

**4 remaining failures:**
1. `den-as-lib.test-module-can-resolve-custom-domain` — resolveEntity entities don't install `__scopeHandlers` for non-parametric aspects
2. `deadbugs-cybolic-routes.test-has-no-dups` — duplicate module from provides-compat shim + routes interaction
3. `user-host-mutual-config.test-host-parametric-unidirectional` — mutual-provider compat: parametric provides pattern
4. `user-host-mutual-config.test-user-provides-to-all-users` — aspect policies from one user invisible to siblings (alphabetical resolution order)

**Architectural evolution:** Initial monolithic `dispatch-policies` handler (524 lines) recreated `into-transition`'s complexity. Redesigned as `installPolicies` using `scope.provide` for context expansion — composable handlers, structural scope isolation. See `2026-05-01-policies-as-handlers-redesign.md`.

**Related specs:**
- `2026-05-01-policies-as-handlers-redesign.md` — design for the installPolicies approach (implemented)
- Forward scope isolation spec — written during Phase 1b investigation
- Traits reimplementation via fleet/den.exports — future work

## Problem

The `into-transition` handler in `transition.nix` (~400 lines) is the most complex piece of the pipeline. It manages:
- Scope push/pop with explicit stack tracking
- Entity resolution via sub-pipelines (forward source resolution)
- Fan-out deduplication
- Enrichment iteration loops
- Cross-provider emission
- Aspect policy inheritance to child scopes

This complexity exists because transitions are treated as hard boundaries — each child entity gets its own scope with independent state. Forward sub-pipelines exist specifically to re-resolve entities across these boundaries.

## Key Insight

Transitions are just context expansion. `policy.resolve { user = tux; }` adds `user` to context. The rest — scope creation, entity lookup, aspect walking — is mechanical consequence. If we make context expansion the primitive (not transitions), the mechanical parts simplify or disappear.

## Design

### Context Expansion Replaces Transitions

Instead of:
```
into-transition fires → mkDispatch → resolveAllTransitions →
  resolveContextValue (pushScope, walk child entity tree, popScope)
```

The pipeline does:
```
dispatchPolicies → policy.resolve { user = tux } →
  enrich context with { user = tux } →
  re-resolve parametric aspects that need { user } →
  modules emitted tagged with { host=igloo, user=tux } context identity
```

No scope push/pop. No transition boundary. The context identity (`mkScopeId { host=igloo, user=tux }`) naturally partitions state — `emit-class` stores modules under the current context identity. This is the same partitioning scopes provided, without the explicit stack machinery.

### Policies Are the Only Mechanism

All context expansion happens through policies:
- `policy.resolve { user = tux; }` — adds user to context
- `policy.include someAspect` — injects an aspect into current resolution
- `policy.route { fromClass; intoClass; path; }` — routes class modules
- `policy.instantiate spec` — requests post-pipeline entity evaluation
- `policy.exclude aspectRef` — removes an aspect from resolution

The `into` function on aspects (`meta.into`, `aspect.into`) is removed. Manual transitions defined on entity kinds are expressed as schema policies instead.

### Entity Resolution Becomes a Lookup

Currently `resolve-entity` is called during transitions to look up entity configs. In the new model, `resolve-entity` is still used but it's a pure lookup — it returns the entity config (aspect, classes, includes) without creating a scope boundary.

When `policy.resolve { user = tux; }` fires:
1. Look up user entity: `den.lib.resolveEntity "user" { host, user }`
2. Enrich context: `currentCtx // { user = tux }`
3. Compute context identity: `mkScopeId enrichedCtx`
4. Set `currentScope = contextIdentity`
5. Walk the user entity's aspect tree (via `aspectToEffect`)
6. Modules emitted to `scopedClassImports.${contextIdentity}`
7. Restore `currentScope` to previous value

Steps 4-7 look like scope push/pop — but there's no stack. The `currentScope` is just set to the context identity and restored afterward. No `scopeStack`, no `scopeParent` tracking needed for scope management. If trait inheritance needs parent identity, it's derivable: parent of `{ host=igloo, user=tux }` is `{ host=igloo }`.

### Fan-Out

When a policy resolves multiple users:
```nix
den.policies.host-to-users = { host, ... }:
  map (user: policy.resolve { inherit user; }) (attrValues host.users);
```

Each `policy.resolve` enriches context independently. Context identity `mkScopeId { host=igloo, user=tux }` vs `mkScopeId { host=igloo, user=pingu }` — different partitions, same mechanism.

Fan-out dedup: if the same context identity was already processed, skip (same as current `ctx-seen` handler).

### Aspect Policies

Currently, aspect policies (`den.aspects.igloo.policies.to-users`) are registered during tree walk and inherited by sub-pipelines via `__aspectPolicies`. In the new model:

Aspect policies registered during tree walk fire when their args are satisfied by the current context. When `policy.resolve { user = tux }` enriches context, previously-unsatisfied aspect policies (needing `{ host, user }`) now match and fire. No inheritance needed — they're in the same pipeline, same state.

This solves the forward sub-pipeline problem: aspect policies don't need to be inherited across pipeline boundaries because there are no boundaries.

### Forward Elimination

With no transition boundaries:
- Forward's `fromAspect` is unnecessary — source modules are in the same pipeline's `scopedClassImports`
- Forward's sub-pipeline is unnecessary — no separate resolution needed
- `policy.route` reads from context-partitioned class imports directly
- Guard/adaptArgs/adapter wrapping moves to route application (already done in Tasks 0/1)

`den.provides.forward` becomes a thin wrapper around `policy.route` — or is deprecated entirely in favor of direct `policy.route` declarations.

### State Fields

**Removed:**
- `scopeStack` — no push/pop
- `scopeChildren` — no tree tracking
- `scopeProvenance` — no provenance tracking
- `scopedForwardSpecs` — no forwards
- `transitionDepth` — no transition depth

**Simplified:**
- `currentScope` — computed from current context identity, set/restored during policy dispatch (no stack)
- `scopeParent` — derivable from context (drop last-added binding) but may keep for performance

**Unchanged:**
- `scopedClassImports` — partitioned by context identity (same as today)
- `scopedTraits`, `scopedDeferredTraits`, `scopedConsumedTraits` — same
- `scopedAspectPolicies` — still accumulated during tree walk
- `scopedRoutes`, `scopedInstantiates` — same
- `scopedDeadLetterQueue`, `scopedDeferredIncludes` — same
- `seen`, `pathSet`, `includeSeen` — same

### Pipeline Flow

```
mkPipeline { class, ctx }:
  1. Set rootScope = mkScopeId ctx
  2. Walk root aspect tree via aspectToEffect:
     a. compileStatic: classify keys, emit classes/traits, register constraints
     b. resolveChildren:
        - emitSelfProvide
        - emitTraitSchemas
        - emitAspectPolicies  (registers aspect policies in state)
        - emitIncludes        (walks child aspects)
        - dispatchPolicies    (replaces emitTransitions):
            For each policy matching current context:
              resolve → enrich context, set currentScope, walk entity tree, restore
              include → emit-include
              route → register-route
              instantiate → register-instantiate
              exclude → register-constraint
  3. Post-walk:
     - wrapCollectedClasses (per context identity)
     - applyRoutes (reads from context-partitioned wrappedPerScope)
     - applyInstantiates
     - Dead letter warnings
  4. Return { imports = classImports.${class} ++ traitModule }
```

### What Gets Deleted

- `into-transition` handler (~400 lines in transition.nix)
- `resolveContextValue` (scope push/pop + entity walking)
- `resolveFanOut` (sub-pipeline fan-out)
- `resolveTransition` (transition resolution loop)
- `processIncludeOnly` (include-only transitions)
- `emitCrossProvider` (cross-provider emission)
- `iterateEnrichment` (enrichment iteration loop)
- `mkDispatch` (policy classification/dispatch — replaced by simpler dispatch)
- `classifyResolve` (resolve effect classification)
- `flattenInto` (into function flattening)
- `emitTransitions` in aspect.nix
- Forward handler + sub-pipeline machinery in pipeline.nix
- Forward detection in include.nix

Estimated: ~600 lines deleted, ~100 lines added for the simplified policy dispatch.

### What Gets Added

- `dispatchPolicies` function (~80 lines) — replaces `emitTransitions`:
  - Iterates registered policies matching current context
  - Processes resolve effects as context expansion + entity walk
  - Processes include/exclude/route/instantiate effects directly
  - Re-dispatches when context widens (for policies needing newly-available args)

### Migration Path

1. **Phase 1**: Add `dispatchPolicies` alongside `emitTransitions` (feature flag)
2. **Phase 2**: Migrate all `into` functions to schema/entity policies
3. **Phase 3**: Remove `emitTransitions`, `into-transition` handler, forward machinery
4. **Phase 4**: Simplify state fields (remove scope stack, tree tracking)

## Relationship to Forward Elimination

This spec supersedes the forward-route-unification spec. The forward elimination is a natural consequence of removing transition boundaries — with no boundaries, there's nothing to re-resolve across.

Tasks 0, 1, 6 from the forward-route-unification plan remain valid:
- Task 0: `policy.route` enrichment (shipped)
- Task 1: `lib.evalModules` removal from route.nix (shipped)
- Task 6: perSystem/microvm migration to `policy.route` (shipped)

Tasks 2-5 are superseded by this spec — the compat shim, forward handler deletion, and test rewrites happen as part of transition elimination instead.

## Research Findings (2026-05-01)

### `dispatchPolicyIncludesHandler` is the template
`tree.nix:314-373` already dispatches aspect policies, matches by args, calls them, processes include/exclude effects. It filters OUT resolve effects (those go to `into-transition`). The new `dispatchPolicies` removes that filter and handles resolve effects as context expansion. ~90% of the dispatch logic already exists.

### `into-transition` callers
Only `aspect.nix:698` calls `fx.send "into-transition"`. Only `pipeline.nix:78` references `transitionHandler`. Clean removal path.

### `den.default` stripping is load-bearing
Tested removing `den.default` stripping from `resolveEntityHandler`. 8 test failures. `includeSeen` dedup doesn't catch it because child entity resolution tags `den.default` with different `__scopeHandlers`, giving it a different `childIdentity` in the include handler.

**Implication for transition elimination:** `den.default` stripping must be preserved in `resolveEntityHandler` even without scope boundaries. The mechanism stays: `resolve-entity` returns the entity with `den.default` filtered from includes.

### Scope set/restore replaces push/pop
`resolveContextValue` (transition.nix:100-181) does: pushScope (80 lines of state mutation) → walk entity → popScope. Can be simplified to: save `currentScope` → set `currentScope = mkScopeId enrichedCtx` → update `scopeContexts` → walk entity → restore `currentScope`. ~60 lines → ~15 lines.

### Surface area
- transition.nix: 900 lines. ~500 lines eliminated, ~100 preserved/refactored.
- aspect.nix: `emitTransitions` (27 lines) + `dispatchPolicyIncludes` (55 lines) → single `dispatchPolicies` (~80 lines). Net: -2 lines in aspect.nix.
- pipeline.nix: remove `transitionHandler` from `defaultHandlers`, remove forward post-processing (~120 lines).
- forward.nix: rewrite to eager inline resolution (main branch style).

Estimated net: ~600 lines deleted, ~150 added. Pipeline conceptually simpler.

## Resolved Questions

### 1. Enrichment Iteration → Natural Recursion

**Resolution**: No explicit fixed-point loop needed. `dispatchPolicies` handles it via natural recursion with fired-policy removal.

When a resolve effect fires, it widens context immediately and re-checks remaining policies against the wider context. Termination is guaranteed: the registered policy set is finite and each fired policy is removed from candidates (same dedup as current `firedPolicies`). Keep max-recursion-depth as a safety cap (matching current `maxEnrichmentIterations = 10`).

Key difference from current model: current batches enrichment per-iteration then re-dispatches everything. New model processes each resolve effect immediately, widening context for subsequent policies in the same pass. Order-independent (context grows monotonically, same final state regardless of dispatch order).

```nix
dispatchPolicies = remaining: currentCtx:
  let
    matching = filter (p: argsSatisfied p currentCtx) remaining;
    rest = removeFrom remaining matching;
    fired = map callPolicy matching;
    resolveEffects = filter isResolve fired;
    processResolve = eff:
      let enrichedCtx = currentCtx // eff.value;
      in walkEntity eff.entity enrichedCtx
         ++ dispatchPolicies rest enrichedCtx;
  in ...;
```

### 2. Cross-Provider Emission → `policy.route` with Synthetic Source Class

**Resolution**: Cross-provider is semantically "source aspect produces modules that land in a child entity's class" — exactly what `policy.route` does.

Mapping:
```nix
# Old: den.aspects.igloo.provides.user = { host, user }: { ... }
# New: schema policy routes source modules to target context
den.schema.host.policies.cross-to-user = { host, ... }:
  policy.route {
    fromClass = "den-user-cross-${host.name}";  # synthetic source class
    intoClass = "den-user";
    path = ["users"];
  };
```

Source aspect emits into a synthetic class (tagged with source context), `policy.route` moves it to target class under target context identity. Eliminates `emitCrossProvider` entirely.

For remaining raw-function cross-providers needing the intersection-args calling convention: convert to schema policies during Phase 2 migration. The provides-compat shim already handles most.

### 3. Per-Entity Policies → Deferred

**Resolution**: Defer. Schema policies with name guards cover all practical cases:

```nix
# Equivalent to hypothetical per-entity policy:
den.schema.host.policies.igloo-to-users = { host, ... }:
  lib.optionals (host.name == "igloo") [ ... ];
```

Dispatch timing concern goes away with transition elimination (policies fire when args are satisfied, period). But per-entity policies add schema complexity for marginal ergonomic gain. If user demand materializes, add later as sugar that auto-generates schema policies with a name guard.

### 4. Trait Inheritance → Explicit `scopeParent` Map

**Resolution**: Keep `scopeParent` as a flat `childScope → parentScope` map, populated trivially during context expansion. No derivation needed.

```nix
# During policy.resolve processing (inside dispatchPolicies):
processResolve = eff:
  let
    enrichedCtx = currentCtx // eff.bindings;
    childScope = mkScopeId enrichedCtx;
    parentScope = state.currentScope;
  in
    # 2 lines: record parent, then walk
    updateState (s: s // { scopeParent = s.scopeParent // { ${childScope} = parentScope; }; })
    >> walkEntity eff.entity enrichedCtx;
```

No `scopeStack`, no `scopeChildren` — just a flat map. Trait walker (`traitArgHandler`) uses it unchanged.

## Additional Notes

### `iterateEnrichment` internals (transition.nix:751-835)
The current loop: calls `mkDispatch` → filters transitions from already-fired policies → collects new enrichment keys → if new: installs into context, drains deferred includes, recurses → if stable: returns accumulated transitions + enrichment. Caps at 10 iterations. The recursive `dispatchPolicies` subsumes this entirely.

### `emitCrossProvider` internals (transition.nix:183-225)
Normalizes cross-provider into parametric wrapper form, tags with scopeHandlers + ctxId, then calls `aspectToEffect`. The synthetic-class + `policy.route` pattern replaces this without special-case handler code.

### `traitArgHandler` scope walking (trait.nix:181-217)
Walks `scopeParent.${currentScope}` recursively. `map` strategy merges from all ancestors down; `list` strategy concatenates. This code stays unchanged — only the population mechanism simplifies.
