# Analysis: Close Gaps — Unified Pipeline Completion

## Verdict

Fully superseded. All six actionable gaps are resolved in feat/fx-pipeline, though via different paths than the spec prescribed. The spec's "stays as-is" items are confirmed correct and still in place. The two commits referenced by the spec (6c68cb52, e71917f1) are not in the current branch history — they were ephemeral exploratory commits on the now-deleted feat/fx-resolution branch.

## Delivery Target

feat/fx-pipeline (current working branch). The spec listed `feat/fx-resolution` as its branch context, but that branch no longer exists. All work landed here.

## Evidence

**constantHandler / aspect-chain shims (spec: Gap 5, stays as-is)**
- `nix/lib/aspects/fx/pipeline.nix:68` — `"aspect-chain" = []` in `defaultHandlers` constantHandler call.
- `nix/lib/aspects/fx/pipeline.nix:161` — `"aspect-chain" = [self]` root override in `mkPipeline`.
- Both shims present exactly as spec documented. The `forward.nix` dependency on `lib.head aspect-chain` was eliminated; `forward` is now post-processing on pipeline results (`1c077a06`), but the shims remain for the type system's provider function path.

**defaultFunctor / providerFnType (spec: circular-eval findings)**
- `nix/lib/aspects/types.nix` — no `defaultFunctor` or `providerFnType` symbols present. The type system was reworked: `parametric.withOwn` style functor removed; `providerFnType.merge` pattern replaced by duck-typed parametric wrappers (`__fn + __args`). The circular-eval constraint the spec describes is therefore no longer relevant in the same form, but `aspect-chain = []` handler was retained anyway.
- `nix/lib/aspects/default.nix:81` — `resolve = fxResolveTree` (no legacy ctxApply path; `options.nix`/`config.resolved` legacy concern is moot).

**Gap 1: emitTransitions + emitSelfProvide wired**
- Resolved. Both are called inside `resolveChildren` (`aspect.nix:760–769`), wired before `emitIncludes` (ordering matches spec Fix 2 intent, with `emitSelfProvide` before `emitIncludes` before `emitTransitions`).

**Gap 2: Dead ctx handlers removed**
- Resolved. `nix/lib/aspects/fx/handlers/ctx.nix` contains only `constantHandler` and `ctxSeenHandler`. The four dead handlers (`ctxProviderHandler`, `ctxTraverseHandler`, `ctxTraceHandler`, `ctxEmitHandler`) are absent. This matches spec Fix 1 but was done independently of the reverted commit 6c68cb52.

**Gap 3: ctx-seen wired in transitionHandler**
- Resolved. `nix/lib/aspects/fx/handlers/transition.nix:346` sends `ctx-seen` effect with structured param `{ key, aspects, aspectValues }` (the ctxSeenHandler accepts both string and attrset forms per ctx.nix:34-36).

**Gap 7: wrapAspect removed**
- Resolved. `wrapAspect` is absent from `nix/lib/aspects/fx/aspect.nix` and from the entire `nix/` tree.

**Gap 8: emitIncludes / foldIncludes deduplication**
- Resolved. `nix/lib/aspects/fx/handlers/include.nix:11` imports `emitIncludes` from `den.lib.aspects.fx.aspect`. No local `foldIncludes` definition present.

**Gap 6: resolve-complete placement**
- Still a design deviation but works. `fx.send "resolve-complete" resolved` is called inside `resolveChildren` (`aspect.nix:783`), which is a nested computation, not a top-level handler boundary. Spec noted this as lower priority.

## Current Status

All mechanical gaps closed. The pipeline is unified with no legacy fallback: `aspects.resolve` routes directly to `fxResolveTree` (no ctxApply path). The spec's "stays as-is" section is accurate: `aspect-chain` shims remain, `resolve-complete`-in-computation remains.

## Supersession

Superseded by subsequent Phase E / pipeline-simplification work on feat/fx-pipeline:
- `1c077a06` — forward as post-processing (eliminates forward.nix's `lib.head aspect-chain` dependency but shims kept)
- `72b34735` — removed collectPolicyHandlers, policyTraceHandlers, dead handler tests
- Policy dispatch overhaul (`a2a7c933`, `30ba636e`, `29156fe0`, `de80246b`, `f5282ac8`)
- Mutual-provider compat shims added (`19738452`, `c52a3779`, `2e40b024`)

The spec's future-work list (type system rework, removing `defaultFunctor`, plain attrset aspects, removing `aspect-chain` handler) remains deferred.

## Gaps

None remaining from this spec. The one residual deviation (resolve-complete in computation vs handler) was explicitly accepted as lower priority and still holds.

## Drift

**Spec said:** `providerFnType.merge` at `types.nix:37` creates `{ class, aspect-chain }` functors; `defaultFunctor = parametric.withOwn` is always baked in.
**Reality:** `providerFnType` and `defaultFunctor` were removed from the type system. Duck-typed parametric wrappers (`__fn + __args`) replaced both. The `aspect-chain = []` handler is still present but the original triggering mechanism (providerFnType.merge creating { class, aspect-chain } args) no longer exists in this form. The shim is effectively benign but retained for safety.

**Spec said:** `options.nix`/`config.resolved` always goes through legacy ctxApply due to circular eval.
**Reality:** ctxApply was removed entirely. `aspects.resolve` is `fxResolveTree` directly. The circular eval concern was resolved by removing the `den.fxPipeline` feature flag entirely — the fx pipeline is now the only path.
