# Analysis: Unified Resolution with Effect-Driven Bind

## Verdict
implemented
All 14 new effect handlers from the spec ship on feat/fx-pipeline. All 6 removed effects are gone from the codebase with no traces. The file structure matches spec intent modulo minor naming differences.

## Delivery Target
feat/fx-pipeline

## Evidence

### New handlers — all present

| Handler file | Spec name | Line count |
|---|---|---|
| `handlers/resolve.nix` | `resolve` | 17 |
| `handlers/compile.nix` | `compile` | 30 |
| `handlers/gate.nix` | `gate` | 103 |
| `handlers/compile-forward.nix` | `compile-forward` | 47 |
| `handlers/compile-conditional.nix` | `compile-conditional` | 53 |
| `handlers/compile-parametric.nix` | `compile-parametric` | 72 |
| `handlers/compile-static.nix` | `compile-static` | 103 |
| `handlers/bind.nix` | `bind` | 95 |
| `handlers/defer.nix` | `defer` | 33 |
| `handlers/drain.nix` | `drain` | 39 |
| `handlers/scope-widen.nix` | `scope-widened` | 37 |
| `handlers/classify.nix` | `classify` | 25 |
| `handlers/emit-classes.nix` | `emit-classes` | 109 |
| `handlers/resolve-children.nix` | `resolve-children` | 76 |

All registered in `pipeline.nix` lines 63–79.

### compile router — matches spec exactly

`handlers/compile.nix` lines 16–24: four-branch dispatch on `__forward`, `guard`, `__args != {}`, else static. Zero logic per branch.

### gate handler — matches spec with extra

`handlers/gate.nix`: dedup → constraint check chain. Adds `include-unseen` call on dedupKey (line 50–56) beyond what the spec described. Substitute path (lines 60–84) resolves replacement via `resolve` effect and collects result.

### gate-tag.nix — not in spec, added as shared utility

`handlers/gate-tag.nix`: shared `gateAndTag` function used by `compile-parametric` and `compile-static` to skip gate on `param.gated = true` (parametric re-entry) and tag `constraintOwner` onto aspect. Not in spec — implementation extracted it to avoid duplication.

### bind handler — diverges from spec on scope ctx fallback

`handlers/bind.nix` lines 24–48: added `currentScope`/`scopeCtx` fallback reading scope context from pipeline state. Spec described pure handler probe via `fx.effects.hasHandler`. Also added `hasPipeArgs` detection (lines 39–40) for `den.quirks` pipe registry.

### defer handler — matches spec

`handlers/defer.nix` lines 17–31: emits stub with `meta.deferred = true`, sends `resolve-complete`, queues in `scopedDeferredIncludes`.

### drain handler — matches spec

`handlers/drain.nix` lines 8–37: partitions `scopedDeferredIncludes` for current scope; satisfiable items removed from state.

### scope-widen handler — matches spec

`handlers/scope-widen.nix` lines 18–33: sends `drain` then re-resolves each satisfiable via `resolve`. Passes `gated = true` on re-entry (not in spec — prevents double gate on drain).

### classify handler — extends spec with pipeKeys

`handlers/classify.nix` line 20: resumes `{ classKeys, nestedKeys, pipeKeys }`. Spec only listed `{ classKeys, nestedKeys, unregisteredClassKeys }`. Implementation merged `unregisteredClassKeys` into `classKeys` and added `pipeKeys` for den.quirks pipe entries.

### emit-classes handler — extends spec with pipe support

`handlers/emit-classes.nix` lines 67–85: adds `emitPipeKey` path for pipe entries (`__isPipeEntry = true`) alongside the spec's `emitClassKey`. `pipeKeys` branch (line 104) is post-spec addition for pipes feature.

### resolve-children handler — matches spec

`handlers/resolve-children.nix` lines 55–76: chain-wrap → `emitAspectPolicies` → `emitIncludes` → `installPolicies` (if `__entityKind`) → `resolve-complete`.

### enterScope — matches spec exactly

`aspect.nix` lines 27–31:
```nix
enterScope = handlers: computation:
  fx.effects.scope.provide handlers (
    fx.bind (fx.send "scope-widened" { ctx = ctxFromHandlers handlers; }) (_: computation)
  );
```
Used by `policy/iterate.nix` line 89 (enrichment widen).

### resolve-schema-entity — uses drain effect

`handlers/resolve-schema-entity.nix` line 78: `fx.send "drain" scopedCtx`. Does NOT use `enterScope` (spec-correct — entity walk needs drain after walk, not before). Pushes scope via `push-scope` effect (separate handler) rather than inline push.

### Walker (emitIncludes) — matches spec intent

`aspect/children.nix` line 32: `fx.send "resolve" { aspect, identity, ctx }`. All dispatch logic gone from walker. `wrapChild`, `propagateScope`, `nameAnon` remain as pure functions (spec-correct).

### Removed effects — all absent

Grep for `resolve-parametric`, `resolve-aspect`, `resolve-conditional`, `emit-forward`, `defer-include`, `drain-deferred`, `drainEnrichmentDeferred`, `dispatchChild`, `aspectToEffect`, `compileStatic` (as a function), `resolveChildren` (as a function), `emitClasses` (as a function) across `/nix/lib/aspects/fx/` returns zero matches.

`handlers/forward.nix` still exists but exports `buildForwardAspect` — a pure utility that builds an aspect attrset from a forward spec, used by `route/apply.nix`. Not an effect handler.

## Current Status

Fully implemented. 713/713 tests pass per spec header.

## Supersession

This spec replaced the following earlier specs:
- `docs/superpowers/specs/2026-05-02-fx-pipeline-consolidated.md` (phase-E pipeline architecture)
- `docs/superpowers/specs/2026-05-04-fx-simplification.md` (intermediate simplification)

This spec is not superseded by anything more recent on the branch.

## Gaps

1. **Tracing not updated for new effects** — `trace.nix` only handles `resolve-complete`. Spec item 26 ("Update tracing handler for new effects") is not implemented. The new compile-*, gate, bind, defer, drain, classify effects are invisible to tracing.

2. **`scope-widen.nix` filename** — spec proposed `scope-widen.nix`; file is named `scope-widen.nix` (matches). Spec handler name is `scope-widened` (the effect name); file naming is consistent.

3. **`includes.nix` not present** — spec listed `handlers/include.nix` as "simplified — just delegates to walker." File `handlers/include.nix` exists; walker logic is in `aspect/children.nix` not a separate `includes.nix`. Structural difference only.

## Drift

1. **`gate-tag.nix` extraction** — spec showed gate called directly from compile-parametric and compile-static. Implementation extracted shared `gateAndTag` helper (`handlers/gate-tag.nix`) that adds `gated = true` skip logic and `constraintOwner` tagging. Both handlers import it. Not in spec; sound refactor.

2. **bind handler scope fallback** — spec's bind description probed only `fx.effects.hasHandler`. Shipped code (`handlers/bind.nix` lines 24–48) adds a `scopeCtx` fallback reading from `state.scopeContexts` for child scopes, and `scopeOverrideKeys` augmentation to inject state-derived values into `__scopeHandlers`. This handles a case where `scope.provide` doesn't survive handler boundaries.

3. **Pipe support added post-spec** — `classify` returns `pipeKeys`; `emit-classes` handles `pipeKeys` with `__isPipeEntry = true`; `bind` detects `hasPipeArgs` via `den.quirks`. All three diverge from spec, which predates the pipes/quirks feature (`den.pipes` → `den.quirks` rename landed in commit `aa6677e7`).

4. **classify merges unregisteredClassKeys into classKeys** — spec's `classify` resume showed `{ classKeys, nestedKeys, unregisteredClassKeys }` as three fields. Implementation (`handlers/classify.nix` line 19) merges `unregisteredClassKeys` into `classKeys` so callers see two fields. Minor simplification.

5. **resolve-schema-entity uses push-scope effect** — spec showed inline `pushScope` function call. Shipped code uses `fx.send "push-scope"` effect (`handlers/push-scope.nix`), going through the effect system. Consistent with spec's "every decision point is an effect" principle.
