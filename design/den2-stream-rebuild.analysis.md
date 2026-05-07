# Analysis: Den 2 Stream Rebuild

## Verdict

not-implemented

The spec proposes replacing Den's handler/trampoline pipeline with stream composition (Ned's ST/ctxD/run Cycle fixed-point) targeting ~1,000–1,400 lines. The `feat/fx-pipeline` branch still runs the shared-trampoline walk with 35+ handlers, ~14,500 lines total in `nix/`. None of the stream primitives (ST, ctxD, scopeD, mkEntityDriver, buildDriverChain, dispatchD, pipeDataCycle) appear in the codebase. The spec is a forward design document; implementation has not started.

## Delivery Target

not delivered

No branch exists for Den 2. The spec references a companion `den2-incremental-delivery.md` with a 10-phase plan; that doc exists in `design/` but only as a plan file, not evidence of delivery.

## Evidence

**Core stream primitives — absent:**
- `ST`, `ctxD`, `scopeD`, `run` — grep finds zero hits in `nix/` (confirmed: no `ned.nix` import, no `fx.rotate`/`fx.stream`/`fx.bind` stream constructor usage outside existing fx kernel).
- `mkEntityDriver`, `buildDriverChain`, `discoverNestingOrder`, `applyTopology`, `converge`, `dispatchD`, `tagPipeEntry`, `pipeDataCycle`, `routeClasses`, `enrichedSinks` — all absent.

**Existing engine still present:**
- Handler/trampoline: `nix/lib/aspects/fx/pipeline.nix` (235 lines), 35 handlers composed in `defaultHandlers` (lines 34–79).
- Handlers directory: `nix/lib/aspects/fx/handlers/` — 2,082 lines across 35 files including `compile-parametric.nix`, `push-scope.nix`, `drain.nix`, `scope-widen.nix`.
- Post-pipeline assembly: `nix/lib/aspects/fx/resolve.nix` (557 lines), `nix/lib/aspects/fx/assemble-pipes.nix` (632 lines).
- Total nix: ~14,500 lines (spec target: ~1,400).

**Spec API proposals — implementation status per change:**

| Change | Spec proposal | Status |
|--------|--------------|--------|
| 1 | `den.lib.parametric` eliminated | NOT done — still present at `nix/lib/parametric.nix` (46 lines), still used in tests (`templates/ci/modules/features/parametric-context.nix:8`, `templates/ci/provider/modules/den.nix:23`) |
| 2 | `excludes`/`substitutes` top-level syntax | PARTIAL — `excludes` works via `meta.excludes` (`templates/ci/modules/features/fx-constraints.nix:643`, `include-dedup.nix:308`), not top-level; `meta.handleWith` still live in tests (`templates/ci/modules/features/aspect-path.nix:37,60,75,110`) |
| 3 | Aspect-scoped policy auto-activate | NOT done — `autoActivate` absent from codebase |
| 4 | Policy effect constructors injected as context args (`{ host, include, provide, ... }`) | NOT done — no context injection; effects accessed via `den.lib.policy.*` namespace |
| 5 | `den.quirks` → `den.pipes` rename | INVERTED — commit `aa6677e7` (2026-05-06) renamed from `den.pipes` → `den.quirks`; `modules/options.nix:221` declares `options.den.quirks`; spec says rename back to `den.pipes` |
| 6 | `den.provides.forward` retired | NOT done — still used extensively in `templates/ci/modules/features/forward*.nix`; `home-env.nix:89` uses it |
| 7 | Hosts — dropped (no change) | N/A — confirmed no change needed |
| 8 | `den.schema.X.options` / `den.schema.X.includes` split | PARTIAL — `den.schema.X.includes` exists (`modules/removed-stages.nix:11`, `modules/aspects/provides/*.nix`); `den.schema.X.options` appears only in deadbug test (`templates/ci/modules/features/deadbugs/issue-413-provider-bare-function.nix:23`), not as a general pattern |

**Collision system — still present:**
- `nix/lib/aspects/fx/class-module.nix` (254 lines): `mkCollisionValidator`, `classWinsDen`, `denWinsDen` all at lines 13–197. Spec says this is eliminated.

**Enrichment stripping — still present:**
- `nix/lib/aspects/fx/wrap-classes.nix` (190 lines): `stripEnrichmentArgs` at line 29. Spec says this dies.

**Pipes implementation — exists but different model:**
- `assemble-pipes.nix` (632 lines): post-pipeline assembly approach, NOT Cycle fixed-point.
- `pipe.collect`, `pipe.expose`, `pipe.withProvenance`, `pipe.to` — effect constructors defined in `nix/lib/policy-effects.nix:121–135`. Implementation is handler-based (`nix/lib/aspects/fx/handlers/register-pipe-effect.nix`), not stream combinators.
- `scopeParent` tree exists (`resolve.nix:14–23`, `pipeline.nix:163`) for sibling collection — this is the same conceptual mechanism as spec's `parentScopeId`, but implemented via handler state not stream tags.

**forwardTo — partially exists:**
- `nix/lib/forward.nix:14` reads `den.classes.${fromClass}.forwardTo` — the class-level forward declaration exists. `policy.route` also exists (`policy-effects.nix:70`). But `den.provides.forward` remains the primary API in tests.

## Current Status

Still exists — `feat/fx-pipeline` branch implements the handler/trampoline architecture described in `stream-architecture-migration.md` as "Current Model (Shared Pipeline)." The Den 2 stream rebuild spec is a design proposal for a future ground-up replacement that has not been started.

## Supersession

This spec supersedes `design/stream-architecture-migration.md`, which describes the same stream-composition migration but as a migration path. Den 2 proposes a clean-room rebuild instead of in-place migration. Neither has been implemented. The companion `design/den2-incremental-delivery.md` provides the 10-phase delivery plan for this spec.

## Gaps

All eight layers of the resolution engine (mkEntityDriver, dispatchD, expandAspect, pipe stream ops, Cycle fixed-point, wrapModule without collision system, options layer rewrite) are unimplemented.

Specific unimplemented features:
- Ned integration (import or vendor ~210 lines of ST/ctxD/run)
- Schema-driven driver chain construction (`buildDriverChain`, `discoverNestingOrder`)
- Policy effect constructors injected as destructured context args
- Aspect-scoped policy auto-activation (`autoActivate`)
- `excludes`/`substitutes` as top-level aspect keys (not `meta.excludes`/`meta.handleWith`)
- Cycle fixed-point for pipe cross-entity data (current: post-pipeline `assemblePipes`)
- `tagPipeEntry` scope metadata (current: `scopeParent` tree via handler state)
- `wrapModule` without collision detection (~250 lines eliminated per spec)
- `den.provides.forward` retirement

## Drift

The rename in commit `aa6677e7` went the opposite direction from the spec: spec says `den.quirks` → `den.pipes`; the commit renamed `den.pipes` → `den.quirks`. This means the codebase at `aa6677e7` diverges from spec Change 5. The spec likely post-dates this commit and reflects a desired reversal.

The `meta.excludes` pattern in tests (`fx-constraints.nix:643`, `include-dedup.nix:308`) is closer to spec Change 2 than `meta.handleWith`, but the spec wants a top-level `excludes = [...]` key with no `meta.` prefix — this is not implemented.

The `scopeParent` tree in `resolve.nix` is functionally analogous to the spec's `parentScopeId` tagging for `pipe.collect` sibling resolution, but implemented as accumulated handler state (22 state fields) rather than per-element stream tags. The semantics are compatible; the implementation model is not.
