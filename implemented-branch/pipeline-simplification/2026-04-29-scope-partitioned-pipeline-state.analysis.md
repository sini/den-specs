# Analysis: Scope-Partitioned Pipeline State

## Verdict
partially-implemented (core shipped, trait system deleted)

The core scope infrastructure — `mkScopeId`, scope-partitioned state fields, push-scope/restore-scope handlers, provide-to deletion, currentCtx removal, runSubPipeline elimination, and forward sub-pipeline replacement — fully shipped on feat/fx-pipeline. The trait system (dynamic registration, `register-trait-schema`, `inheritTraits`, `traitModuleForScope`, `scopedTraits`) was implemented then entirely deleted in commit `5ba68cf3` for clean reimplementation under a different design. The remaining spec gaps (unified policy dispatch, flat state removal, scope-prefixed dedup) were partially closed but differ from spec.

## Delivery Target
feat/fx-pipeline

## Evidence

**mkScopeId — shipped:**
- `pipeline.nix:108-129` — exact implementation matches spec. Injective key=value comma-separated string, sorted, with type fallback `<type:key>`.

**Scope-partitioned state fields — shipped:**
- `pipeline.nix:131-170` — `defaultState` has `scopedClassImports`, `scopedAspectPolicies`, `scopedDeferredIncludes`, `scopedIncludesChain`, `scopedConstraintRegistry`, `scopedConstraintFilters`, `scopedRoutes`, `scopedInstantiates`, `scopedProvides`, `scopedPipeEffects`, `currentScope`, `scopeContexts`, `scopeParent`. Missing from spec: `scopeChildren` never added (not in defaultState). Extra vs spec: `scopedProvides`, `scopedPipeEffects`, `scopedEmittedLocs` — post-spec additions.

**push-scope / restore-scope handlers — shipped:**
- `handlers/push-scope.nix:13-88` — atomically sets `currentScope`, `scopeContexts`, `scopeParent`, inherits `scopedAspectPolicies`.
- `handlers/restore-scope.nix:5-27` — restores `currentScope` to `param.parentScope`.
- Added commit: `4dc0d41b` (2026-05-04).

**Scope-prefixed dedup — shipped (both fields):**
- `handlers/check-dedup.nix:20-21` — `includeSeen` key is `${scope}/${rawDedupKey}`.
- `handlers/ctx.nix:28-32` — `ctxSeen` key is `${scope}/${key}`. Commit `4fe4d782`.

**currentCtx removal — shipped:**
- Commit `2bd63e09` — removed from `defaultState`, all readers use `scopeContexts.${currentScope}`. Remaining uses at `pipeline.nix:88-89` and in `policy/` are local variables, not state fields.

**runSubPipeline elimination — shipped:**
- Commit `08f5291e` — `runSubPipeline` deleted from `pipeline.nix`. `resolveFanOut` inlines the logic.
- `fxFullResolve` retained (`pipeline.nix:211-218`) as thin wrapper over `mkPipeline`, still exported.

**Forward sub-pipeline elimination — shipped (via route model):**
- Commit `637af5e3` — Tier 2 forwards write to `scopedRoutes` with `sourceScopeId`; `applyComplexRoute` in `route/apply.nix:51-70` reads from `wrappedPerScope.${sourceScopeId}` using `getCollectedSource`. No `runSubPipeline` for forwards.
- Tier 1 auto-classification in `handlers/compile-forward.nix:16-44` — `isTier1` check matches spec proposal.

**provide-to deletion — shipped:**
- Commit `849b919e` (2026-04-29) — deleted `handlers/provide-to.nix`, `distribute-cross-entity.nix`, `crossEntityTraits` parameter, `resolveSiblingTransition`.

**class-collector writes to scoped partition — shipped:**
- `handlers/class-collector.nix:28-62` — writes to `scopedClassImports.${scope}.${class}`.

**Flat dual-writes removed — shipped:**
- Commits `892dbf12`, `1ebe9c5b`, `4a35afb7`, `13238921` — progressive removal of flat state, scoped partitions as sole source of truth.

**Dynamic trait schema registration (`register-trait-schema`, `emitTraitSchemas`, `inheritTraits`, `traitModuleForScope`) — deleted:**
- Commits `e61340e1`, `daf283a3`, `5572cc67` — implemented these functions.
- Commit `5ba68cf3` (2026-05-01) — deleted the entire trait pipeline: `handlers/trait.nix` (-255 lines), `handlers/tree.nix` trait parts, `aspect.nix` trait classification, `key-classification.nix` trait branch, all trait test files. Message: "Traits will be reimplemented with fleet + den.exports design."

**Unified policy dispatch — not shipped as specced:**
- `dispatch-policy-includes` handler referenced in spec does not exist in current code. `handlers/dispatch-policies.nix` is an `mkDispatchPoliciesHandler` factory for entity-scope dispatch. No `includeOnly` filter found in codebase. Commit `aa32f77e` collapsed `resolveChildren` from 5 phases to 3.

**scopeChildren, scopeStack, scopeProvenance — not shipped:**
- None of these fields exist in `defaultState` or any handler. `scopeParent` exists but `scopeChildren` was never added. Upward propagation uses `scopeParent` lookups at read time (e.g., `assemble-pipes.nix:132-135`, `resolve.nix:178`).

**DLQ (`deadLetterQueue`, `drain-dead-letters`) — deleted:**
- Commits `c58a7058`, `109c5ebe` — DLQ removed entirely. Unknown keys emit as classes immediately. The `register-trait-schema` → `drain-dead-letters` integration therefore never reached production.

## Current Status

Core scope infrastructure still exists and is active. The spec's scope-partitioned model drives all current pipeline output. Traits are gone — `key-classification.nix:52-82` shows two-branch dispatch (class, pipe) with no trait branch. `den.traits` no longer consulted.

## Supersession

This spec's "What remains" section explicitly names `2026-04-29-policy-route-class-delivery.md` as its continuation. `policy.route` shipped (commits `8ab1009f` through `a1d1806b`). Trait reimplementation is deferred to a not-yet-written spec (referenced in `5ba68cf3` as "fleet + den.exports design").

## Gaps

1. **Trait system** — `register-trait-schema`, `emitTraitSchemas`, `inheritTraits`, `traitModuleForScope`, `traitArgHandler`, `scopedTraits` all deleted. Zero trait infrastructure in current pipeline.
2. **scopeChildren** — spec calls for explicit `scopeChildren.${scopeId} = [...]` tracking. Not present. Child lookups inferred via `scopeParent` traversal at read time instead.
3. **scopeStack** — spec used explicit push/pop stack. Implemented differently: `push-scope` effect sets `currentScope` directly, `restore-scope` receives `parentScope` as param. No stack in state.
4. **scopeProvenance** — never added.
5. **Unified policy dispatch** — spec's unified dispatch (eliminating `dispatch-policy-includes` / `includeOnly` split) not implemented as described. Current model uses `installPolicies` at entity scope entry.
6. **DLQ** — deleted. The `drain-dead-letters` integration with trait registration is moot.
7. **`aspect-schema.nix:collectFromAspects` removal** — spec deferred this; it was removed as part of the broader trait deletion (`5ba68cf3`), but for a different reason (full trait removal, not migration).

## Drift

**Scope switching mechanism differs from spec.** Spec shows inline `fx.bind (state.modify push) → aspectToEffect → state.modify pop` with an explicit `scopeStack`. Implementation uses effect-based `push-scope` / `restore-scope` handlers that receive `parentScope` as a parameter — no stack in state, parent ID passed explicitly by the caller (`resolve-schema-entity.nix`).

**Forward model diverges significantly.** Spec describes `applyForwardSpecs` reading from `scopedClassImports.${scopeId}` in a post-pipeline step. Actual implementation routes forwards through `scopedRoutes` with a `compile-forward` handler that classifies Tier 1/2 at emit time, then `applyRoutes` applies them via `route/apply.nix`. The `buildForwardAspect` wrapping is retained but invoked from `applyComplexRoute`, not a standalone `applyForwardSpecs`. Complex forwards still call `resolveSourceFallback` which runs `fxResolve` as a sub-pipeline for external-source aspects (`route/apply.nix:24-38`) — sub-pipeline elimination is partial, not total.

**Trait design replaced entirely.** Spec assumed traits would be the primary cross-entity data mechanism, with scope inheritance replacing `provide-to`. Implementation replaced `provide-to` with scope inheritance for policy effects, but traits were deleted outright. The cross-entity data story is now `den.exports` / fleet (future design).

**DLQ never reached the dynamic trait integration.** Spec's battery aspect example depended on DLQ reclassifying dead-lettered keys when `register-trait-schema` fired. Both DLQ and trait registration were removed before this integration was exercised in production.
