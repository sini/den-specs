# Analysis: Effects-Based Context Application

## Verdict
superseded

## Delivery Target
(A) main — landed in den-fx PR #462 (68c26555), merged to main

## Evidence

- **68c26555** `refactor: den-fx (#462)` — the commit that implemented the fx pipeline. Created `nix/lib/aspects/fx/handlers/ctx.nix` (ctxSeenHandler, constantHandler), `nix/lib/aspects/fx/handlers/transition.nix` (into-transition handler), `nix/lib/aspects/fx/pipeline.nix` (mkPipeline, fxFullResolve, fxResolve). Deleted/replaced `nix/lib/ctx-apply.nix` (still exists on main as a legacy shim — see below).
- **ed1301a4** `feat: ctx-as-data transitions, deferred includes, fan-out identity` — subsequent commit hardening transitions.
- **ctx-seen** effect: lives in `nix/lib/aspects/fx/handlers/ctx.nix` line 29 (`ctxSeenHandler`), wired into `pipeline.nix` `defaultHandlers` at line 78.
- **ctx-traverse** effect: emitted in `templates/ci/modules/features/fx-trace.nix` as a test hook; `structuredTraceHandler` handles it. Not in production defaultHandlers — treated as an optional tracing hook only.
- **ctx-provider** effect: tested in `templates/ci/modules/features/fx-ctx-apply.nix` (line 318) but no production handler registered in defaultHandlers. Provider resolution absorbed into transition.nix.
- `ctxSeenHandler` exported and used in `pipeline.nix`; `assembleIncludes` does not exist anywhere on feat/fx-pipeline (grep confirms zero matches); it was part of the legacy `ctx-apply.nix` which was deleted.
- `ctxApplyEffectful` function name: never appeared. The spec proposed this name but the implementation used `aspectToEffect` (aspect → computation compiler) + `mkPipeline` instead.
- `nix/lib/ctx-apply.nix` still exists on **main** as legacy fallback (gated by `den.fxPipeline = false`). Deleted on **feat/fx-pipeline** (HEAD returns `fatal: path does not exist`).

## Current Status

The three spec effects have divergent fates:

- **ctx-seen**: Fully implemented and active in production pipeline. Signature evolved — accepts `{ key, aspects, aspectValues }` attrset (not just a string key) since commit `000c77b5`. Still backward-compatible with plain string param (line 33 in ctx.nix: `if builtins.isString param then param else param.key`).
- **ctx-traverse**: Exists as a named effect but is *not* handled in `defaultHandlers`. It is a hook point for trace handlers only. Tests in `fx-trace.nix` confirm the effect fires and is interceptable.
- **ctx-provider**: Exists as a named effect used in test fixtures (`fx-ctx-apply.nix`). No production handler in `defaultHandlers`. Provider resolution is embedded inline in `transition.nix` rather than delegated through an effect.

## Supersession

This spec was the design input for **den-fx PR #462** (`68c26555`). The spec was authored 2026-04-13 and implemented 2026-04-17. The spec's proposed architecture (ctxApplyEffectful as a separate traversal feeding into resolveDeepEffectful) was merged into a single unified pipeline: `aspectToEffect` compiles any aspect into a computation, and `into-transition` handler owns context traversal — there is no separate ctx-apply stage.

Superseded design notes: `docs/design/fx-pipeline-spec.md` (created in same commit) describes the as-built architecture.

## Gaps

1. **ctxApplyEffectful function**: Never shipped by that name. Replaced by `aspectToEffect` + `mkPipeline` architecture. The spec's two-stage design (ctxApply → resolveDeep) became a single unified pipeline.
2. **ctx-provider in production handlers**: Spec defined `ctxProviderHandler` as a default handler. Implementation chose to inline provider logic in `transition.nix` instead. The effect exists as a test surface but has no registered default handler.
3. **ctx-traverse in production handlers**: Spec defined `ctxTraverseHandler` as a default no-op with tracing hook point. Implementation omits it from `defaultHandlers` entirely — the effect is only meaningful when a trace handler is composed in via `mkPipeline { extraHandlers }`.
4. **fxFullResolve signature**: Spec showed `{ ctxNs, class, self, ctx }` with an explicit context namespace parameter. Shipped as `{ class, self, ctx }` — `ctxNs` was dropped (see spec open question #1, resolved by keeping ctxNs as a closure at a higher level, then eliminated when the two-stage design merged).
5. **Root resolve-complete emission**: Spec included an explicit `fx.send "resolve-complete" resolved` for the root aspect. Implementation handles this via `aspectToEffect` which emits `resolve-complete` for every aspect including root — no special-case needed.
6. **Test plan coverage**: All 9 proposed test cases have analogues in `fx-ctx-apply.nix` and `fx-handlers.nix`, though matched to the as-built API rather than spec API.

## Drift

1. **Architecture**: Spec proposed `ctxApplyEffectful` as a standalone traversal returning `Computation [aspect]` that feeds into `resolveDeepEffectful`. Implementation unified both into a single `aspectToEffect` pipeline — context transitions are `into-transition` effects handled inline, not a separate pre-pass.

2. **ctx-seen param shape**: Spec specified `param = key` (plain string). As-built accepts `{ key, aspects, aspectValues }` to support per-aspect-set merge key tracking (commit `000c77b5`). Legacy string form still works.

3. **Provider resolution**: Spec made provider resolution observable via `ctx-provider` effect + dedicated handler. Implementation embedded it in `transition.nix`'s `resolveContextValue` function. Effect is used in tests but not wired into the default handler set.

4. **ctxTraverseHandler**: Spec included this as a default no-op. Not in production `defaultHandlers`. The effect fires from test code only; production tracing uses `composeHandlers` to attach a trace handler to `mkPipeline`.

5. **fxResolve return shape**: Spec returned `{ imports = result.state.imports }`. As-built returns `{ imports = (forwarded.classImports.${class} or []) ++ lib.optional hasTraitSchemas traitModule }` — multi-class state, trait injection, and forward post-processing all added beyond the spec scope.
