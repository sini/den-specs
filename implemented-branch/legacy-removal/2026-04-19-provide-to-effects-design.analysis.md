# Analysis: provide-to: Cross-boundary context forwarding via effects

## Verdict

Partially implemented with significant scope evolution. The pre-work (scope.stateful transition propagation) was NOT done as specified — the implementation chose a different mechanism (__scopeHandlers / constantHandler as data on aspects). The provide-to phase 1/2 pipeline IS implemented but diverged from the spec's design: the emission shape changed from NixOS module routing to trait-keyed data distribution, and phase 2 is performed via crossEntityTraits injection into fxResolve rather than re-running the pipeline against osConfigurations. forward.nix was NOT migrated to emit provide-to. The spec's fleet /etc/hosts and haproxy tests exist but in simulated form (not end-to-end pipeline tests). Cross-entity distribution is functional at the library level with strong unit test coverage.

## Delivery Target

feat/fx-pipeline branch. Commits landed: `ed1301a4` (ctx-as-data, deferred includes, fan-out identity), `ec3ad18a` (remove provide-to structural key), `57129393` (distribute-cross-entity replaces distribute-provide-to), `ab126627` (fxResolve cross-entity trait injection).

## Evidence

**Pre-work: scope.stateful NOT restored**
- `grep scope.stateful` returns zero hits in `nix/lib/` (only a stash-cleanup worktree hit).
- The spec's action was to replace ctx-as-data `__ctx` tagging with `scope.stateful { constantHandler { host, user } }`.
- Instead the implementation uses `__scopeHandlers` as an attrset field carried on aspects and propagated through the tree. `constantHandler` is used as a pure Nix lookup (handler-as-data), never installed as an effect scope handler.
- `scope.provide` IS used in `aspect.nix:915` for parametric context: `scopeFn = if scopeHandlers != null then fx.effects.scope.provide scopeHandlers else null;` — this wraps `fx.bind.fn userArgs fn` for any aspect carrying `__scopeHandlers`. It is NOT in transition.nix for policy handler injection (no such usage exists).
- `__ctx` is gone from the pipeline (only appears in comments and a class-module-partial-apply test fixture describing legacy behavior).

**__parentCtx / __parentCtxId: NOT removed**
- `__parentCtxId` is actively used in `nix/lib/aspects/fx/handlers/include.nix:107`, `transition.nix:269`, `transition.nix:369`, and `aspect.nix:489-756`.
- The spec said `__parentCtx` / `__parentCtxId` would be removed once scoped handlers propagate automatically. That removal has not occurred — __parentCtxId is still the mechanism for propagating fan-out identity to deferred includes.
- `__parentCtx` (bare, no Id suffix) does not appear in the codebase — it was never introduced.

**__ctxId: Still present for fan-out dedup**
- Confirmed present in `nix/lib/aspects/fx/identity.nix:7,12`, `nix/lib/aspects/fx/handlers/transition.nix`, and `include.nix`.
- `__parametricResolved` also still present in `aspect.nix:30,442,900`.
- Both retained as spec predicted.

**provideToHandler: Implemented**
- `nix/lib/aspects/fx/handlers/provide-to.nix` exists with the exact thunk-accumulation shape from the spec.
- Wired into `defaultHandlers` in `pipeline.nix:87`.
- `defaultState.provideTo = _: []` in `pipeline.nix:137`.

**Sibling routing via provide-to: Implemented**
- `transition.nix:193-227` implements `isSiblingRoute` (routing.from == routing.to) and `resolveSiblingTransition` which emits `fx.send "provide-to" { targetEntity, traits }`.
- Emission shape diverged from spec: spec described `{ target, content, emitterCtx }` with dendritic class-key content; implementation emits `{ targetEntity, traits }` where traits is a trait-keyed attrset.

**Phase 2 distribution: Implemented but different mechanism**
- `nix/lib/aspects/fx/distribute-cross-entity.nix` implements `groupByTarget`, `mergeTraits`, `distributeCrossEntityTraits`, `distribute` — trait-aware grouping and merging.
- `fxResolve` accepts a `crossEntityTraits` parameter and injects cross-entity trait data directly into the traitModule inside the NixOS evalModules fixpoint.
- This is NOT the spec's phase 2 (re-running pipelines after osConfigurations resolves each host). Phase 2 in the spec was orchestration-layer distribution into target configs. The implementation routes trait data through fxResolve's crossEntityTraits argument — callers must run phase 1 on all hosts, collect provideTo, call distributeCrossEntityTraits, then pass the result when building each target. No osConfigurations.nix changes were found to wire this up end-to-end.

**forward.nix migration: NOT done**
- `nix/lib/aspects/fx/handlers/forward.nix` uses the forwardSpecs / post-processing mechanism (`applyForwardSpecs` in pipeline.nix) — a fresh sub-pipeline run per forward spec.
- The spec said the forward result should be emitted as a provide-to for orchestration routing. This was not implemented; forward remains a post-processing step in runSubPipeline, not a provide-to emission.
- forward's provideTo is collected in `applyForwardSpecs` and bubbled up through runSubPipeline, so any provide-to from forward sub-pipelines does reach the caller.

**Tests: Unit coverage strong, end-to-end absent**
- `templates/ci/modules/features/provide-to.nix` has: no-emissions baseline, sibling routes to provide-to, fleet /etc/hosts distribution (simulated emissions), haproxy backend distribution (simulated), distribute produces handlers, map collection merges, map duplicate throws, fxResolve cross-entity list/map injection, map duplicate via fxResolve, empty crossEntityTraits no-op.
- Fleet and haproxy tests use manually constructed emission lists — they do not run a full den pipeline with real host configs and verify the NixOS outputs.

## Current Status

- Pre-work (scope.stateful): Not done. Different mechanism chosen (__scopeHandlers-as-data).
- provide-to handler: Done.
- provideToHandler in defaultHandlers: Done.
- Sibling routing: Done (routing.from == routing.to gating).
- distribute-cross-entity: Done (trait-keyed, renamed from spec's distributeProvideTo).
- fxResolve crossEntityTraits injection: Done.
- osConfigurations.nix phase 2 wiring: Not done — no end-to-end orchestration glue found.
- forward.nix migration: Not done.
- __parentCtx/__parentCtxId removal: Not done.

## Supersession

The spec's "dendritic class-key content" model for provide-to was superseded by the traits system. Instead of injecting arbitrary NixOS module content across entity boundaries, cross-entity data flows through named traits (den.traits schema). The provide-to effect carries `{ targetEntity, traits }` rather than aspect attrsets with class keys. This is a deliberate design evolution, not a gap — the traits architecture provides schema-driven collection strategies (list/map), duplicate detection, and a clean NixOS injection point via `_den.traits`.

The mutual-provider compat shim (`modules/compat/mutual-provider-shim.nix`) handles migration from the old `den._.mutual-provider` pattern; the spec's note about leaving provides.to-users/to-hosts in place was also honored (mutual-provider replaced by aspect-included policies, with a compat shim for removed API).

## Gaps

1. **osConfigurations.nix phase 2 wiring absent.** The library functions exist but no module wires them into nixosConfigurations construction. Cross-entity trait injection (crossEntityTraits arg to fxResolve) requires the caller to: run fxFullResolve on all hosts, collect state.provideTo null from each, call distributeCrossEntityTraits, pass per-host results as crossEntityTraits when building each host. This orchestration is not automated.

2. **forward.nix not migrated to provide-to.** Forward still runs a fresh sub-pipeline synchronously in applyForwardSpecs. The spec's intent was to route forward results via provide-to for cross-entity distribution. Not done.

3. **No end-to-end fleet test.** The fleet /etc/hosts and haproxy tests in provide-to.nix simulate distribution manually rather than exercising the full pipeline path from host resolution through distribution to NixOS option values.

4. **__parentCtxId not removed.** The spec listed __parentCtxId as eliminated by scoped handlers. It is still live plumbing used by deferred includes for fan-out identity propagation.

## Drift

- **Emission shape**: Spec `{ target, content, emitterCtx }` → Implementation `{ targetEntity, traits }`. Target is now the entity attrset (not a ctx-name string), and content changed from dendritic class-key attrsets to trait-keyed data. emitterCtx is absent.
- **distributeProvideTo → distributeCrossEntity**: Renamed and restructured to be trait-schema-aware (list/map strategies). The docs-fixes worktree still references the old name; main branch uses `den.lib.aspects.fx.distributeCrossEntity`.
- **Phase 2 mechanism**: Spec described re-running pipelines at orchestration level after host resolution. Implementation uses a fxResolve parameter (crossEntityTraits) injected into the traitModule. No pipeline re-run occurs.
- **pre-work mechanism**: Spec said restore scope.stateful for transition context propagation. Implementation uses __scopeHandlers-as-data + constantHandler-as-lookup instead of nix-effects scope handlers. This is functionally equivalent for the transition propagation problem but leaves nix-effects scope.stateful unused in production code. `scope.provide` IS used in `aspect.nix:915` for parametric context (wrapping `bind.fn` calls for aspects with `__scopeHandlers`), but not for transition context propagation or policy handler injection.
- **Graph-based multi-hop routing**: Spec described `target = ["peer", "user"]` multi-hop list support. Implementation only shows single-hop (routing.from == routing.to string comparison). No multi-hop traversal found.

## Cross-Reference Notes

- **`scope.provide` misattribution corrected** (2026-04-28): The original analysis stated `scope.provide` was in `transition.nix line 306` for "policy handler injection." This was wrong on both counts. The only `scope.provide` call in the entire fx pipeline is `aspect.nix:915`, wrapping `fx.bind.fn` for parametric aspects that carry `__scopeHandlers`. This matches the account in `2026-04-17-fixedto-ctx-propagation.analysis.md`, `2026-04-18-ctx-as-data-design.analysis.md`, `2026-04-20-context-guard-design.analysis.md`, and the session-summary analysis — all of which confirm `scope.provide` is the parametric context mechanism, not a policy dispatch mechanism.
- **Supersession chain**: This analysis sits at the legacy-removal end of the chain. The `__scopeHandlers`-as-data approach it describes feeds directly into the policies branch, where `constantHandler` built on top of it is the sole context-propagation mechanism. The policies analyses (`2026-04-27-unified-policy-effects-design.analysis.md`, `2026-04-28-policy-pipeline-simplification.analysis.md`) confirm no transition-level `scope.provide` for policies was ever introduced.
