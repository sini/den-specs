# Analysis: FX Pipeline — Feature Complete + Integration Flag

## Verdict
partially-implemented

## Delivery Target
(B) feat/fx-pipeline — but the A/B gate itself was removed on this branch; pipeline is unconditional

## Evidence

**Pipeline flag lifecycle:**
- `modules/fxPipeline.nix` created in `68c26555` (den-fx initial commit, 2026-04-17) with `options.den.fxPipeline`, default `true`, path `den.fxPipeline` (not `den.schema.conf.fxPipeline` as specced)
- `nix/lib/aspects/default.nix` used `fxEnabled = den.fxPipeline or true;` and branched `resolve = if fxEnabled then fxResolveTree else legacyResolve;`
- `modules/fxPipeline.nix` deleted and legacy resolver removed in `3a9cea99` (2026-04-23) — pipeline is now unconditional
- `templates/ci/modules/features/fx-flag.nix` created in `3a9cea99` then deleted in same commit (stash-cleanup worktree retains it)

**structuredTraceHandler** (§2): fully shipped
- `nix/lib/aspects/fx/trace.nix` — `structuredTraceHandler` and `tracingHandler` exported
- Uses `includesChain` state (chain-push/pop) for parent tracking — not `__parent` field on param as specced; parent is derived in the handler from `chainParent` helper
- `isParametric` and `fnArgNames` populated via `aspect.nix` compile step, not in a `wrapIdentity` function
- Tests in `templates/ci/modules/features/fx-trace.nix` cover: basic fields, excluded aspects, parent from chain, root parent null, mkPipeline with trace, tracingHandler entries+paths

**ctxTraceHandler** (§3): partially implemented
- No dedicated `ctxTraceHandler` export in `trace.nix`
- `ctx-traverse` effect is not emitted by the production pipeline — `transition.nix` uses `ctx-seen` instead
- `ctxTrace = []` initialised in `capture.nix` `defaultState` and threads through pipeline but never populated by any pipeline handler
- `fx-trace.nix` test `test-ctx-trace-handler` tests the effect construct directly (manual `fx.send "ctx-traverse"`), not a production emission
- Diagram capture (`nix/lib/diag/capture.nix`) reads `state.ctxTrace or []` but gets an empty list in practice

**Context stage tagging** (§4): not implemented
- No `__ctxStage` or `__ctxKind` fields set anywhere in production pipeline (`grep` returns zero results in `nix/lib/aspects/fx/`)
- `entityKind` field on trace entries is derived via `deriveEntityKind` walking `includesChain` and `entries` — a different mechanism than the specced `__ctxStage`/`__ctxKind` tagging

**mkPipeline / defaultHandlers / defaultState exports** (§5 partial): fully shipped
- `pipeline.nix` exports all three at top level
- `den.lib.aspects.fx.pipeline.{mkPipeline,defaultHandlers,defaultState,composeHandlers}` all accessible

**flakeModules/fxPipeline.nix** (§1 FlakeModule): not implemented
- `flakeModules/` directory never created
- No `den.flakeModules.fxPipeline` export
- `inputs.nix-effects` validation assert (§1 input validation) never implemented

**nix-effects wiring** (§1 wiring): landed differently
- `nix/lib/fx.nix` uses `inputs.nix-effects.lib or nfx` with a pinned tarball fallback via `templates/ci/flake.lock`
- No assert on missing `inputs.nix-effects`; fallback handles it silently

**Test plan coverage:**
- §5 item 1 (structuredTraceHandler fields): covered in fx-trace.nix
- §5 item 2 (ctxTraceHandler ctxTrace items): manual construct only, no production emission tested
- §5 item 3 (parent tracking): covered — child.parent == "root" test
- §5 item 4 (ctx stage tagging): not implemented
- §5 item 5 (mkPipeline with extraHandlers): covered
- §5 item 6 (nested adapter override): not found in test files
- §5 item 7 (pipeline flag activation): flag removed; pipeline unconditional
- §5 item 8 (flag without nix-effects error): not implemented (fallback tarball instead)
- §5 item 9 (exact assertions): fx-e2e.nix still has `>= 1` on line 84; fx-full-pipeline.nix assertions not tightened

## Current Status

**Flag:** Removed. `modules/fxPipeline.nix` deleted in `3a9cea99`; no `den.fxPipeline` option exists. `nix/lib/aspects/default.nix` no longer has legacy/fx branch. Pipeline is the only path.

**structuredTraceHandler / tracingHandler:** Present and functional in `nix/lib/aspects/fx/trace.nix`. Used by diagram capture pipeline in `nix/lib/diag/capture.nix`.

**ctxTraceHandler / ctx-traverse:** `ctx-traverse` effect is not emitted by any production handler. `ctxTrace` field in capture state stays `[]`. The spec's idea (track context stage transitions via `ctx-traverse`) was superseded by `entityKind` derivation from `includesChain` + `entries` ancestry walk in `deriveEntityKind`.

**flakeModules/fxPipeline.nix:** Never created.

## Supersession

Superseded by the full pipeline stabilisation arc:
- `68c26555` (den-fx) — initial A/B gate + legacy resolver; flag path `den.fxPipeline` (not `den.schema.conf.fxPipeline`)
- `3a9cea99` (effectful bootstrap) — deleted `den.fxPipeline` option and legacy resolver simultaneously
- `2e0fc9e1` (remove legacy resolver) — removed legacy `resolve.nix` and cleaned `default.nix` to unconditional fx path
- Diagram-side specs (`2026-04-13-fx-feature-complete-spec.md` §3/§4) partially superseded by `entityKind` derivation in trace.nix

## Gaps

1. **`den.schema.conf.fxPipeline` option path** — specced path never used; implementation used `den.fxPipeline`; then removed entirely
2. **`flakeModules/fxPipeline.nix`** — never created; no `den.flakeModules.fxPipeline` consumer API
3. **`inputs.nix-effects` validation assert** — replaced by silent tarball fallback in `fx.nix`
4. **`ctxTraceHandler` as a named export** — no such export; `ctx-traverse` not emitted by pipeline
5. **`__ctxStage`/`__ctxKind` tagging on provider results** (§4) — not implemented; `entityKind` derivation used instead
6. **Nested adapter override test** — not present in any feature test file
7. **Exact assertion tightening** — `fx-e2e.nix` line 84 still uses `>= 1`

## Drift

- **Option path:** Spec says `den.schema.conf.fxPipeline`; implementation used `den.fxPipeline` (module-level, not schema.conf). Then the option was eliminated entirely — the pipeline is unconditional on feat/fx-pipeline.
- **Parent tracking mechanism:** Spec proposed `__parent` field injected onto `resolve-complete` param by the resolver (option A). Implementation uses `includesChain` state (chain-push/chain-pop effects) and derives parent in the trace handler via `chainParent`. Functionally equivalent but architecturally inverted: state-driven rather than param-driven.
- **Parametric detection:** Spec proposed `wrapIdentity` function doing the detection. Implementation does it in `aspectToEffect` compile step inside `aspect.nix`, setting `meta.isParametric` and `meta.fnArgNames` before emission.
- **ctxTraceHandler:** Spec designed it as a handler on `ctx-traverse` effect emitted by `buildStageIncludes` in ctx-apply.nix. The fx pipeline has no ctx-apply.nix equivalent — transitions are handled by `transition.nix` using `ctx-seen`. Entity-kind context is reconstructed at trace time from `includesChain` ancestry, not captured live via a traversal effect.
- **nix-effects requirement:** Spec required a hard assert on missing `inputs.nix-effects`. Implementation uses a lock-pinned tarball fallback (`nix/lib/fx.nix`) — nix-effects is never truly "missing" from the consumer's perspective.
