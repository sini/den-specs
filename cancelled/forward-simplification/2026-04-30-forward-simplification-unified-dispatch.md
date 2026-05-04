# Forward Simplification and Unified Policy Dispatch

## Status: Draft

## Problem

Three pieces of the scope-partitioned pipeline and policy-route specs remain unshipped:

1. **Forward sub-pipelines are still running.** `resolveForwardSource` (pipeline.nix:325-414) runs a full `fxFullResolve` per Tier 2 forward spec, redundantly re-resolving source content that already exists in scope partitions. The forward handler captures `__resolveCtx` and `__aspectPolicies` to reconstruct the source entity's environment from scratch.

2. **Policy dispatch is split across two phases.** `dispatch-policy-includes` (tree.nix:314-374) fires during tree-walk for include-only policies. `into-transition` (transition.nix) fires during transitions for all effects. Include-only policies with resolve effects that produce empty values are silently skipped. Policies emitting both include AND resolve effects have their include effects deferred until transition dispatch, which might not fire for flat entity kinds.

3. **Forward infrastructure needs triage.** 17 files reference `emit-forward` or `den.provides.forward`. Most are test templates validating Tier 2 behavior. The auto-detect mechanism (commit `6f1f1656`) already routes Tier 1 forwards through `policy.route` transparently.

## Design

### Section 1: Forward Handler Simplification

The forward handler computes `sourceScopeId` at registration time instead of capturing full context and policies:

- **Same-entity forwards** (no `__scopeHandlers` on source): `sourceScopeId = state.currentScope` — source content was emitted in the declaring scope's partition.
- **Cross-entity forwards** (source has `__scopeHandlers`): `sourceScopeId = mkScopeId (ctxFromHandlers source.__scopeHandlers)` — source entity was walked during a transition and its content is in that scope's partition.
- **Unreachable source** (computed `sourceScopeId` not in `state.scopeContexts`): Source references a `resolveEntity` never walked as a transition. The handler inline-walks it: push scope, `aspectToEffect`, pop scope. Scope-prefixed dedup keys (shipped in `4fe4d782`) ensure the inline walk has its own dedup namespace.

The forward spec stores `sourceScopeId` instead of `__resolveCtx`/`__aspectPolicies`. Post-pipeline, `applyForwardSpecs` reads `scopedClassImports.${spec.sourceScopeId}.${spec.fromClass}` — a direct lookup, no sub-pipeline.

**Deletions:**

| Component | Location | Lines |
|-----------|----------|-------|
| `resolveForwardSource` | pipeline.nix:325-414 | ~90 |
| `__resolveCtx` field on forward specs | forward.nix:260 | — |
| `__aspectPolicies` field on forward specs | forward.nix:261 | — |
| Sub-pipeline `fxFullResolve` call in `applyForwardSpecs` | pipeline.nix:428 | — |

**What stays:**

- `buildForwardAspect`, `mkDirectAspect`, `mkAdapterAspect` — adapter wrapping genuinely needed for Tier 2
- `collectClassMods` — producing target class entries
- `den.provides.forward` user API — for ~10% Tier 2 cases (adapter modules, dynamic intoPath)

### Section 2: `applyForwardSpecs` Rewrite

The function signature changes to accept scope-partitioned data instead of being self-contained:

```nix
applyForwardSpecs = { forwardSpecs, wrappedPerScope, scopedTraits, traitSchemas, scopeParent }:
  builtins.foldl' (acc: spec:
    let
      # Read source content from scope partition — no sub-pipeline
      sourceClassImports = wrappedPerScope.${spec.sourceScopeId}.${spec.fromClass} or [];

      # Synthesize trait module for source scope
      subHasTraitSchemas = traitSchemas != {};
      subTraitModule = if subHasTraitSchemas
        then traitModuleForScope spec.sourceScopeId scopedTraits scopeParent traitSchemas
        else null;

      rawSourceModule = {
        imports = sourceClassImports
          ++ lib.optional subHasTraitSchemas subTraitModule;
      };
      sourceModule = spec.mapModule rawSourceModule;
      forwardAspect = handlers.buildForwardAspect spec sourceModule;
      newMods = collectClassMods spec.intoClass forwardAspect;
    in
    { classImports = acc.classImports // { /* newMods merged */ }; }
  ) { classImports = {}; } forwardSpecs;
```

The caller in `fxResolve` already has `wrappedPerScope`, `scopedTraits`, `traitSchemas`, `scopeParent`.

**Ordering in `fxResolve` unchanged:**

1. Pipeline execution -> scoped state
2. Wrap per-scope -> `wrappedPerScope`
3. Apply Tier 1 routes -> `withRoutes`
4. Apply Tier 2 forwards -> reads from `withRoutes` (post-route data)
5. Flatten across scopes

**Unreachable source handling:** For the inline-walk case, the forward handler populates the scope partition during pipeline execution. By the time `applyForwardSpecs` runs post-pipeline, all source content is in scope partitions regardless of origin.

### Section 3: Unified Policy Dispatch

Remove the split between tree-walk dispatch (`dispatch-policy-includes`) and transition dispatch (`into-transition`). All policy effects — include, exclude, resolve, route, instantiate — are handled uniformly during transition dispatch.

**Current split:**

| Phase | Handler | Effects processed | Filter |
|-------|---------|-------------------|--------|
| Tree-walk | `dispatch-policy-includes` | include, exclude only | Skips policies with ANY resolve effect |
| Transition | `into-transition` | all | None |

**Problem with the split:** Policies emitting both include AND resolve effects have their include effects deferred until transition dispatch — the `hasResolve` filter (tree.nix:344-353) excludes any policy with non-empty resolve effects from `dispatch-policy-includes`, so those policies' include effects only fire during `into-transition`. This might not fire for flat entity kinds. Additionally, a policy that conditionally produces empty vs non-empty resolve values behaves inconsistently — sometimes processed as include-only during tree-walk, sometimes deferred to transitions.

**Fix — merge into transition handler:**

1. In `mkDispatch` (transition.nix), remove the filter at line 739 that drops include-only policy results. All policies produce results uniformly.
2. Add `includeEffects` and `excludeEffects` to the dispatch result (alongside transitions, enrichment, routes, instantiates).
3. The transition handler processes include/exclude effects via `processIncludeOnly`-style logic — emit includes, register exclude constraints.
4. Delete `dispatchPolicyIncludesHandler` from tree.nix (lines 314-374).
5. Delete `dispatchPolicyIncludes` from aspect.nix (lines 622-676).
6. Remove `fx.send "dispatch-policy-includes"` from `emitTransitions` — include/exclude effects handled by transition dispatch.

**Changes per file:**

| File | Change |
|------|--------|
| `transition.nix:mkDispatch` | Remove include-only filter (line 739). Add `includeEffects`/`excludeEffects` to result. |
| `transition.nix:into-transition` | After resolving transitions, process accumulated include/exclude effects via emit-include + register-constraint. |
| `tree.nix` | Delete `dispatchPolicyIncludesHandler` (lines 314-374). |
| `aspect.nix` | Delete `dispatchPolicyIncludes` (lines 622-676). Remove `fx.bind (dispatchPolicyIncludes aspect)` wrapper in `emitTransitions` (line 692). |
| `pipeline.nix` | Remove `handlers.dispatchPolicyIncludesHandler` from handler set (line 85). |

**Ordering justification:** Currently `dispatchPolicyIncludes` fires BEFORE `emitTransitions` sends `into-transition`. With unified dispatch, include/exclude effects fire DURING `into-transition` processing. This is safe because:

- `processIncludeOnly` already handles include/exclude effects inside the transition handler (for policies that also have resolve effects)
- Include aspects are emitted via `emit-include` which goes through normal tree-walk
- The enrichment loop iterates until stable, capturing include effects from enrichment-dependent policies

**Global vs aspect policy asymmetry:** Global policies (transition.nix:624-644) already handle include-only cases correctly via the `hasRouting` check — they create a transition with empty contexts when include/exclude effects are present but no resolve effects. No behavioral change needed for global policies. Only aspect policies (transition.nix:739) have the filter that drops include-only results — this is where the fix applies.

**Invariant:** The `into-transition` handler only fires for entity roots (`isEntityRoot` check at aspect.nix:686). Non-entity aspects don't dispatch policies — unchanged.

### Section 4: Forward Infrastructure Cleanup

After sections 1-3 ship, verify remaining forward callers:

**Keep (Tier 2):**

| File | Reason |
|------|--------|
| `nix/lib/home-env.nix` | Uses `mapModule` for HM wrapping |
| `nix/lib/aspects/fx/handlers/forward.nix` | Simplified handler |
| `modules/aspects/provides/forward.nix` | User API for Tier 2 |
| `nix/lib/aspects/types.nix` | Type definitions |
| `templates/flake-parts-modules/modules/perSystem-forward.nix` | Uses `mapModule` |

**Template CI tests — all keep:**

All forward test modules (`forward.nix`, `forward-from-custom-class.nix`, `forward-to.nix`, `cross-context-forward.nix`, `guarded-forward.nix`, `forward-flake-level.nix`, `dynamic-intopath.nix`, `forward-alias-class.nix`) validate Tier 2 behavior which remains.

**External templates — verify auto-detect:**

- `templates/microvm/modules/microvm-integration.nix` — check if Tier 1 (auto-detected) or Tier 2
- `templates/nvf-standalone/modules/nvf-integration.nix` — check if Tier 1 or Tier 2

**Cleanup work:**

1. Verify auto-detect correctly routes Tier 1 forwards through `policy.route` at runtime
2. Update forward test modules to validate simplified path (scope partition reads)
3. No test deletions — all validate Tier 2 behavior

## Migration Strategy

Bottom-up approach (Approach A), each step independently testable:

1. **Forward handler:** Compute `sourceScopeId`, remove `__resolveCtx`/`__aspectPolicies` capture
2. **`applyForwardSpecs`:** Rewrite to scope partition reads, delete `resolveForwardSource`
3. **Unified dispatch:** Merge `dispatch-policy-includes` into transition handler
4. **Cleanup:** Verify auto-detect, update tests

## Risk Assessment

**Forward simplification correctness.** The unreachable source case (inline-walk) is the riskiest path. The `forward-flake-level.nix` test covers this case. If the inline-walk populates the scope partition correctly, `applyForwardSpecs` reads it identically to any other source.

**Unified dispatch ordering.** Include effects move from tree-walk phase to transition phase. If any policy depends on include effects being visible BEFORE `into-transition` fires, this would regress. Mitigant: `processIncludeOnly` already handles this inside the transition handler for policies with resolve effects, and no policy currently depends on this ordering distinction.

**Trait module synthesis.** `traitModuleForScope` replaces `resolveForwardSource`'s inline `subTraitModule` synthesis. The function already exists and is tested via scope-partitioned trait delivery. No new mechanism needed.
