# Analysis: Policies and Cross-Entity Resolution
## Verdict
Substantially implemented with significant design drift. All three layers are present but each deviates from the spec in meaningful ways. Core machinery shipped; spec's activation model, `from`/`to` field API, and `__functor` style all superseded by leaner approaches.

## Delivery Target
feat/fx-pipeline branch. Policies work shipped across ~30 commits spanning `e325fc2c` (aspect-included policies) through `f5282ac8` (template plain functions).

## Evidence

**den.policies option** — present in `nix/nixModule/policies.nix`. Typed as `lazyAttrsOf raw` (no submodule schema, no policyType record). Plain functions only.

**policy-effects.nix** — `resolve`, `include`, `exclude` constructors exist with spec-matching semantics. `resolve.to "kind" { bindings }` added (commit `269ee2c8`) replacing the spec's `to = "kind"` field on the policy record. `resolve.shared` exists for non-isolated fan-out.

**transition handler** — `nix/lib/aspects/fx/handlers/transition.nix` dispatches `den.policies` directly via iteration (not per-policy named effects as spec describes). `resolveArgsSatisfied` guards on required args. `isSiblingRoute` checks `routing.from == routing.to`, routing to `resolveSiblingTransition`. Child routes call `resolveContextValue` / `resolveFanOut`. Routing decision (child vs sibling) matches spec intent.

**provideToHandler** — implemented in `nix/lib/aspects/fx/handlers/provide-to.nix` with thunk-chain pattern matching spec's `state.provideTo` design exactly.

**provide-to state** — `pipeline.nix` defaultState includes `provideTo = _: []`. `fxFullResolve` collects `pipelineProvideTo` and merges with forwarded provideTo. `runSubPipeline` returns `{ classImports, traits, provideTo }`.

**distribute-cross-entity.nix** — phase 2 distribution implemented but as trait distribution, not raw module injection. Groups by `targetEntity.name`, merges traits via `collectTrait` strategy (list/map). Returns `constantHandler` bindings per target.

**policy-inspect.nix** — `den.lib.policies.inspect { kind, context }` implemented as spec describes. Cheap, no pipeline run.

**policy-types.nix** — minimal: `policyFnArgs = policy: lib.functionArgs policy`. No submodule schema.

**ctx removal** — `den.ctx` fully removed from active code. Compat shim at `modules/compat/ctx-shim.nix` forwards `den.ctx.<kind>` to `den.schema.<kind>.includes` with deprecation warning.

**aspect-included policies** — `state.aspectPolicies`, `registerAspectPolicyHandler`, `dispatchPolicyIncludesHandler` all implemented in `tree.nix`. Policies can live on aspects (`policyFns` key) not just `den.policies`.

## Current Status

**Layer 1 (Policies)** — fully implemented, simplified. Plain functions. No `from`/`to` fields (removed in `4d06b4be`, `269ee2c8`). No activation gating (removed `29156fe0`). Scope determined by `__entityKind` guard in policy body + `resolveArgsSatisfied` on function args.

**Layer 2 (Effect-based materialization)** — implemented but differently than spec. Spec describes per-policy named effects (`fx.send "host-to-users" {...}`). Actual: direct iteration over `den.policies` in transition handler, no named effect per policy. `policy.resolve`/`include`/`exclude` are typed effect constructors returned by policies, not effects sent to handlers. The pipeline dispatches on `__policyEffect` discriminant.

**Layer 3 (Cross-entity routing)** — implemented but pivoted to trait-based distribution, not module injection. `resolveSiblingTransition` sends `"provide-to" { targetEntity, traits }` (not raw modules). `distribute-cross-entity.nix` distributes trait data by target name. `fxResolve` accepts `crossEntityTraits` parameter to inject distributed data. Phase 2 orchestration is caller-responsibility — no orchestration wiring in `osConfigurations.nix` or `fxFullResolve` itself.

## Supersession

| Spec feature | Superseded by |
|---|---|
| `from = "kind"` on policy record | removed; policies guard via `{ __entityKind, ... }:` body check (`4d06b4be`) |
| `to = "kind"` on policy record | moved to `resolve.to "kind" { bindings }` effect constructor (`269ee2c8`) |
| `as = "peer"` context alias | implicit via binding key name in resolve effect |
| Four activation levels (`den.default.policies`, `den.schema.<kind>.policies`, entity-instance) | removed entirely (`29156fe0`); all matching policies fire unconditionally |
| Per-policy named effects (`fx.send "host-to-users"`) | direct iteration with typed `__policyEffect` discriminant in transition handler |
| `__functor` on policy objects | plain functions; `policyFnArgs = lib.functionArgs policy` |
| `distributeProvideTo` orchestration function in `pipeline.nix` | `distributeCrossEntityTraits` in `distribute-cross-entity.nix` (trait-only) |
| Phase 2 module injection into osConfigurations | not wired; `crossEntityTraits` accepted by `fxResolve` but caller must supply |
| `provide-to.http-backends` SemanticData label from aspect body | traits system via `emit-trait` handler (different mechanism, same goal) |

## Gaps

1. **Phase 2 not wired at orchestration level** — `osConfigurations.nix` does not call `distributeCrossEntityTraits` or pass `crossEntityTraits` to `fxResolve`. `resolveWithState` (`fxResolveTreeFull`) exists but has no callers outside tests. Cross-entity trait distribution is dead code for production NixOS configs.

2. **`provide-to` in aspect body not structural** — spec describes `provide-to.http-backends = [...]` in aspect attrset as structural key. Actual: traits system (`emit-trait`) is the mechanism for semantic data. No `provide-to` structural key in aspects; spec's `structuralKeys` note for `provide-to` was not implemented.

3. **Sibling route runs sub-pipeline for source traits** — `resolveSiblingTransition` calls `runSubPipeline` on `stageAspect` to collect traits, then sends `provide-to { targetEntity, traits }`. This is trait collection, not the spec's "collect resolved modules in state.provideTo". Semantics differ for non-trait use cases.

4. **`den.lib.policies.inspect` API shape deviation** — spec shows `den.lib.policies.inspect { kind, context }` returning `{ host-to-users = [{ host, user = tux }, ...]; }`. Actual returns `{ policyName = { targetKey, targets, from, to, as, routing }; }` — more metadata, targets are bindings attrsets not resolved entities.

5. **Cycle prevention via depth counter, not structural** — spec promises structural cycle prevention (phase 2 pipeline runs don't install provideToHandler). Actual: `maxTransitionDepth = 50` hard limit in transition handler. Structurally weaker.

6. **No `den.schema.<kind>.policies` scoped activation** — removal of activation model means there's no way to scope policies to a specific entity kind declaratively. All policies fire for any entity kind where `resolveArgsSatisfied` passes.

## Drift

**Positive drift** (implementations better than spec):
- Plain functions simpler than spec's `from`/`to`/`resolve` submodule; arg introspection replaces declarative scope
- `resolve.to` effect is more expressive than top-level `to` field (supports per-effect routing)
- `resolve.shared` for non-isolated fan-out is a useful addition not in spec
- Aspect-included policies (`policyFns`) not in spec; extends composability
- `drain-deferred` integration in `resolveContextValue` handles deferred includes with new context — not in spec

**Negative drift** (implementations weaker than spec):
- Phase 2 distribution not wired in production path — the flagship cross-entity use case (fleet `/etc/hosts`, haproxy backends) has no working end-to-end path in `osConfigurations.nix`
- Activation levels removed — users cannot scope policies to specific entity kinds without `__entityKind` body guards (which is implicit, not declarative)
- SemanticData/ClassModule duality from spec's capability-labeled provides section not implemented (traits partially substitute but different interface)
