# Analysis: Effectful Pipeline Bootstrap

## Verdict

NOT IMPLEMENTED. The spec describes an approach that was partially scaffolded, then superseded by a fundamentally different architecture. None of the three primary deliverables (resolve-policy effect, resolve-target effect, policy.handlers field) exist in the codebase.

## Delivery Target

Commit 3a9cea99 referenced in the task prompt does not exist on feat/fx-pipeline. No commit in the log references "bootstrap" in its message. The spec predates the pipeline simplification work that eliminated the foundational pieces this spec was built on (synthesizePolicies.mergePolicyInto, policyType submodule with handlers field, buildTarget).

## Evidence

**resolve-policy effect:** grep across all .nix files — zero matches. No handler, no sender, no registration in defaultHandlers.

**resolve-target effect:** grep across all .nix files — zero matches. No handler, no sender, no registration in defaultHandlers.

**mergePolicyInto:** grep finds only a deletion commit (12fed6aa "remove dead synthesize/mergePolicyInto"). The function no longer exists. synthesize-policies.nix now exports only `resolveArgsSatisfied`.

**policy.handlers field:** policy-types.nix has been replaced entirely. It now contains only `policyFnArgs = policy: lib.functionArgs policy;` — a single line. The `policyType` submodule with `from`, `to`, `resolve`, `handlers` options is gone.

**buildTarget:** not present anywhere in the codebase. The spec's §3c ("buildTarget deleted, logic moves into resolveTargetHandler") is accurate in that buildTarget is gone, but the logic did not move into a resolveTargetHandler — it was replaced by a `resolve-entity` effect + `resolveEntityHandler` in pipeline.nix.

**fx.handle bootstrap shape:** pipeline.nix mkPipeline does wrap `aspectToEffect self` inside `fx.handle`, which matches the spec's desired post-change shape. However, `bootstrapAndResolve` is just `aspectToEffect self` with no `fx.bind` chain for resolve-policy. The pre-computation mergePolicyInto call the spec wanted to eliminate was already deleted (commit 12fed6aa) — but it was deleted by removing the feature, not by moving it inside fx.handle.

**scope.provide for policy handlers:** scope.provide is used in aspect.nix (line 915) for `__scopeHandlers` (constant context values), not for policy-declared handlers. The per-policy named handler installation from §3b is absent.

**defaultHandlers:** contains resolveEntityHandler (not in the spec), but does NOT contain resolvePolicyHandler or resolveTargetHandler. The handler set is: traitArgHandler, constantHandler, traitCollectorHandler, classCollectorHandler, constraintRegistryHandler, chainHandler, includeHandler, transitionHandler, ctxSeenHandler, pathSetHandler, collectPathsHandler, registerAspectPolicyHandler, dispatchPolicyIncludesHandler, deferredIncludeHandler, drainDeferredHandler, resolveEntityHandler, forwardHandler, provideToHandler, state.handler.

## Current Status

The transition handler (transition.nix) dispatches policies by direct iteration over `den.policies` — plain functions, not submodules. Policy dispatch checks argument satisfaction via `resolveArgsSatisfied`, calls the policy function, and collects policy effect constructors (resolve/include/exclude from policy-effects.nix). Target resolution uses `fx.send "resolve-entity"` handled by `resolveEntityHandler` in pipeline.nix, which calls `den.lib.resolveEntity`. This is the actual current architecture; the spec's architecture was never built.

## Supersession

Three commits obsolete this spec's foundation:

- **12fed6aa** — "remove dead synthesize/mergePolicyInto": deletes the exact function this spec wanted to move inside fx.handle
- **30ba636e** — "delete policy-dispatch.nix, remove dead types": eliminates policyType submodule; policies are now plain functions, making the handlers field extension impossible
- **de80246b** — "convert all policies to plain functions, remove __functor wrappers": completes the policy model change; no policy can carry a handlers attrset in a typed submodule sense

The `resolve-entity` effect (present in transition.nix, line 295) partially satisfies the interceptability goal of `resolve-target` but has a different name, different protocol, and omits the policy-merge logic the spec included.

## Gaps

Relative to spec intent:

1. **Policy-scoped named handlers** — no mechanism for a policy to install effect handlers scoped to its transition's sub-computation. The `den.policies.host-to-user.handlers.peer` pattern from the spec's Layer 2 example is not possible.
2. **Interceptable policy merge** — policy resolution order / override via `resolve-policy` handler is not possible.
3. **Interceptable target assembly** — `resolve-entity` handler exists but is not the same contract as `resolveTargetHandler` (no policy-merge in handler, no mergedInto assembly).
4. **`coreEffects` guard list** — the safety filter preventing policy handlers from shadowing pipeline internals was never needed since policy handler installation was never implemented.

## Drift

The spec's design assumed:
- Policies are submodule-typed with from/to/resolve/handlers fields — FALSE, they are plain functions
- `mergePolicyInto` exists and routes policies into stages — FALSE, deleted
- `den.stages` exists for target lookup — FALSE, deleted (commit d8b538f8 added a shim warning)
- Bootstrap involves pre-computing `mergedInto` outside fx.handle — FALSE, that code path was deleted entirely rather than moved

The current architecture replaces static stage-based routing with dynamic entity-kind resolution (`resolve-entity`) and schema-driven includes. The spec's Layer 2 goal (per-policy named handlers for peer queries) remains unbuilt and would require a new design compatible with plain-function policies.

## Cross-Reference Notes

- **Consistent with all policies analyses**: This "NOT IMPLEMENTED" verdict is confirmed independently by `2026-04-20-policies-design.analysis.md` ("no named effect per policy"), the simplified-review-guide legacy-removal analysis ("resolve-policy / resolve-target effects: Not present"), and `2026-04-28-policy-pipeline-simplification.analysis.md` (direct iteration dispatch with no named bootstrap effects). No analysis contradicts this.
- **Supersession chain position**: This spec sat between the `from/to` submodule design (`2026-04-20-policies-design.md`) and the plain-function simplification path (`2026-04-27-unified-policy-effects-design.md`). The simplification path was chosen instead — the `den.stages` removal and `policy-dispatch.nix` deletion made the bootstrap approach structurally impossible before it could be built.
- **`scope.provide` clarification**: The spec proposed `scope.provide` per-policy for named handler injection (Layer 2). This never shipped. The only `scope.provide` in the codebase is `aspect.nix:915` for parametric context (`__scopeHandlers`), unrelated to policy dispatch. See `2026-04-19-provide-to-effects-design.analysis.md` Cross-Reference Notes for full account.
