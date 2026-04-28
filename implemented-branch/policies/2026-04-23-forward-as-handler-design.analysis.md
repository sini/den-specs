# Analysis: Forward as Handler

## Verdict

IMPLEMENTED — with a significant post-spec architectural refinement. The `emit-forward` effect and `forwardHandler` landed in commit `c0d9dc56` exactly as specified. The handler was later refactored in `1c077a06` to defer sub-pipeline execution to post-processing rather than running it inline. The effect mechanism, context inheritance, and `buildForwardAspect` result shape are all present. Key spec fields (`fromClass`, `intoClass`, `intoPath`, `source`) are implemented under slightly different naming.

## Delivery Target

- Primary commit: `c0d9dc56` — "feat: forward-as-handler, fix fan-out context dedup" (2026-04-23)
- Follow-up refinement: `1c077a06` — "refactor: forward as post-processing on pipeline results" (2026-04-28)

## Evidence

**`emit-forward` effect:**
- `nix/lib/aspects/fx/handlers/forward.nix` — file exists, header comment: `# Handles: emit-forward`
- `nix/lib/aspects/fx/handlers/include.nix:295` — `fx.bind (fx.send "emit-forward" child.meta.__forward) (_: fx.pure [])`
- `nix/lib/aspects/fx/handlers/forward.nix:203` — `forwardHandler = { "emit-forward" = { param, state }: ...`

**`forwardHandler` registered in default handlers:**
- `nix/lib/aspects/fx/pipeline.nix` — `forwardHandler` imported and used in `defaultHandlers`
- `nix/lib/aspects/fx/handlers/default.nix` — includes forward handler module

**`forwardItem` sends effect instead of calling `aspects.resolve`:**
- `nix/lib/forward.nix:104` — `forwardItem` now returns `meta.__forward = { ... }` marker attrset instead of calling `den.lib.aspects.resolve`
- `includeHandler` in `include.nix:294` detects `isForward` and sends the `emit-forward` effect

**Context propagation via `state.currentCtx`:**
- `forward.nix:211` — `parentCtx = (state.currentCtx or (_: {})) null`
- Inherited context applied via `resolveCtx` with fallback when source has own `__scopeHandlers`

**Sub-pipeline uses `fxFullResolve` with captured context:**
- `pipeline.nix:212-218` — `applyForwardSpecs` calls `runSubPipeline` with `spec.__resolveCtx`
- `runSubPipeline` calls `fxFullResolve { inherit class self ctx extraState; }`

## Current Status

Fully implemented and in production on `feat/fx-pipeline`. The `flake-parts-modules` template regression cited in the spec as motivation is confirmed fixed by the commit message of `c0d9dc56`. The mechanism is:

1. `forwardItem` builds a `meta.__forward` marker (no longer calls `aspects.resolve`)
2. `includeHandler` detects `isForward` and emits `"emit-forward"` effect
3. `forwardHandler` registers the forward spec + captured parent context in `state.forwardSpecs`
4. Post-processing in `applyForwardSpecs` (called from `fxResolve` and `runSubPipeline`) runs each spec's sub-pipeline with the captured context, wraps via `buildForwardAspect`, merges into target class buckets

## Supersession

The spec's `forwardHandler` design had the handler run `fxFullResolve` inline and append the resulting module directly to `state.imports`. The `1c077a06` refactor changed this: the handler only registers the spec in `state.forwardSpecs`; sub-pipeline execution is deferred to post-processing. This is an improvement — it avoids executing sub-pipelines inside the effect handler and allows `provideTo` results from forward sub-pipelines to be spliced into the parent pipeline's `provideTo`.

The spec's state key `state.imports` was replaced by `state.classImports` (the pipeline uses per-class bucketed collection, not a flat imports list). Forward results are merged into `classImports` during post-processing, not inline.

## Gaps

**Spec fields vs. implementation:**
- Spec effect param: `{ fromClass, intoClass, intoPath, source, adaptArgs, guard, evalConfig }`
- Actual `meta.__forward` fields (from `forward.nix:103-126`): `{ fromClass, intoClass, evalConfig, freeformMod, sourceAspect, mapModule, staticIntoPath, needsAdapter, needsTopLevelAdapter, adapterKey, guardFn, guardArgs, intoPathArgs, intoPathFn, adaptArgsFn, adaptArgv, adapterMods, extraArgsFor, canDirectImport }`
- `source` → `sourceAspect` (renamed, same semantics)
- `intoPath` → pre-resolved into `staticIntoPath` + `intoPathFn` split (handles function-valued intoPath)
- `adaptArgs` → pre-processed into `adaptArgsFn` + `adaptArgv`
- `guard` → pre-processed into `guardFn` + `guardArgs`

These are pre-computation differences from `forwardItem`, not missing features.

**Open questions from spec — status:**
1. Sub-pipeline shares no state with parent (independent state). Sub-pipeline `forwardSpecs` are recursively processed via `applyForwardSpecs` in `runSubPipeline`. Not state-shared, but `provideTo` is spliced.
2. `emit-forward` is not user-interceptable — it is only registered in `defaultHandlers`. The question was left open; current implementation chose the non-interceptable path.
3. `forwardItem` is not purely effectful — it still builds the full spec attrset. The simplification to "just send effect with params" did not fully materialize; pre-computation happens in `forwardItem`, effect carries the pre-computed spec.

## Drift

The spec proposed `state.imports` as the accumulator target. The actual implementation uses `state.classImports` (keyed by class name) — this reflects the broader pipeline architecture that evolved toward per-class bucketing. The handler accumulates into `state.forwardSpecs` (deferred), not `state.imports` (immediate). This drift is intentional and architecturally sound.

`forwardHandler` in the spec was designed as the only place sub-pipeline execution occurs. In the implementation, `buildForwardAspect` (also in `forward.nix`) performs the actual module construction, and `applyForwardSpecs` in `pipeline.nix` orchestrates sub-pipeline + construction. The separation of concerns is cleaner than the spec's monolithic handler.
