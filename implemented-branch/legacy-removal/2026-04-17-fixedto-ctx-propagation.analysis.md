# Analysis: fixedTo __ctx Propagation

## Verdict

Superseded and fully resolved via a different mechanism. The spec's `__ctx` inline approach was implemented transiently then replaced by `__scopeHandlers` + `scope.provide` — a cleaner solution that generalises to all tree depths without special-casing roots. All three spec problems (context gap, duplicate modules, `applyCtxToIncludes`) are resolved. `fixedTo` itself is now deprecated with deprecation warnings.

## Delivery Target

`feat/fx-pipeline` branch. Core changes landed in commits spanning 2026-04-23 through 2026-04-26.

## Evidence

Key commits (chronological):

- `8f9f283d` — `refactor: parametric type separation, scope.provide` — major commit introducing `__scopeHandlers`, `scope.provide`, and revising `constantHandler`; replaced `__ctx`-based context threading with handler-closure mechanism
- `a7f101cc` — `feat: derive ctx from __scopeHandlers instead of __ctx in emitClasses`
- `e668458e` — `cleanup: remove redundant __ctx, use __scopeHandlers only`
- `3d206058` — `cleanup: replace __ctx reads with ctxFromHandlers, remove __ctx from propagation` — final cleanup; deleted `__ctx` from 9 files, replaced reads with `ctxFromHandlers` helper
- `4a9e3453` — `fix: update checkmate functor tests — use ctxFromHandlers instead of removed __ctx`
- `77885717` — `fix: update parametric-fixedTo host-context expected outputs` — test expectations updated for new deferred-drain behaviour

## Current Status

**`fixedTo`**: All variants (`.__functor`, `.exactly`, `.atLeast`, `.upTo`) exist in `nix/lib/parametric.nix` but are wrappers that emit `lib.warn "fixedTo is deprecated"`. Underlying `mkFixedTo` still works via `__scopeHandlers`.

**`__ctx` field**: Does not exist in the codebase. Only occurrence is a comment in `aspect.nix:275` noting the historical distinction between `__ctx` (old, roots-only) and `__ctxId` (current, propagates to children). `__ctx` is NOT in `structuralKeysSet` — the spec's requirement was made moot by removing `__ctx` entirely.

**`__scopeHandlers`**: IS in `structuralKeysSet` (line 27 of `aspect.nix`). This is the live mechanism.

**`applyCtxToIncludes`**: Does not exist anywhere. Never implemented — the spec's design was superseded before this helper was needed.

**`constantHandler`**: Exists in `nix/lib/aspects/fx/handlers/ctx.nix`. Pure map: each ctx key becomes a handler `{param, state}: {resume = value; inherit state;}`. Used in `pipeline.nix`, `parametric.nix`, `forward.nix`, `resolve-entity.nix`, `transition.nix`, `distribute-cross-entity.nix`.

**Context propagation path**: `fxResolveTree` in `aspects/default.nix` calls `ctxFromHandlers (resolved.__scopeHandlers or {})` to reconstruct ctx at the root, then passes it to `pipeline.fxResolve`. Inside the pipeline, `scope.provide scopeHandlers` is called in `aspectToEffect` (line 928) for any aspect carrying `__scopeHandlers`. Parametric children inherit scope handlers via `emit-include` propagating `__parentScopeHandlers` (include.nix line 264–265).

**`keepChild` in include.nix**: Does NOT contain the spec's `scope.stateful (constantHandler child.__ctx)` wrapping. Instead uses `has-handler` effect probing + deferred-include for missing context. The scoped handler is applied at `aspectToEffect` time via `scope.provide`, not in `keepChild`.

**Duplicate module problem**: Resolved because `__scopeHandlers` is a structural key (not emitted as a class key by `compileStatic`), and no `owned` child include was ever introduced — `mkFixedTo` produces a single-level wrapper.

## Supersession

The spec proposed:
1. `__ctx` inline on aspects + `applyCtxToIncludes` for child includes
2. `keepChild` wrapping with `scope.stateful (constantHandler child.__ctx)`
3. `fxResolveTree` extracts `__ctx` from resolved aspect

Implemented instead:
1. `__scopeHandlers` on aspects — handler closures carry context; no special field extraction needed
2. `scope.provide scopeHandlers` in `aspectToEffect` — applies scoped handlers at resolution time for all parametric aspects, not just children observed in `keepChild`
3. `ctxFromHandlers` reconstructs ctx from `__scopeHandlers` — works for any node in the tree, not just roots

The `scope.provide` mechanism is more general: it applies the scoped handlers for the entire subtree beneath a parametric aspect, without needing per-child wrapping logic in `keepChild`.

## Gaps

None — all three problem areas from the spec are addressed:
- Context gap: `scope.provide` installs handlers for entire subtrees, not one level
- Duplicate modules: `__scopeHandlers` is structural; no `owned` child was ever used
- `applyCtxToIncludes`: unnecessary; child includes inherit scope via `__parentScopeHandlers` propagation in `emit-include`

## Drift

Significant design drift from spec, in a positive direction:

| Spec | Implemented |
|------|-------------|
| `__ctx` field inline on aspects | `__scopeHandlers` closure (no `__ctx`) |
| `applyCtxToIncludes` helper | `emit-include` propagates `__parentScopeHandlers` |
| `keepChild` wraps with `scope.stateful` | `aspectToEffect` uses `scope.provide` per-aspect |
| `fxResolveTree` reads `.__ctx` | `ctxFromHandlers` reads `.__scopeHandlers` |
| `__ctx` added to `structuralKeys` | `__scopeHandlers` already in `structuralKeys`; `__ctx` removed |
| `fixedTo` produces new shape | `fixedTo` deprecated; `mkFixedTo` is a shim via `__scopeHandlers` |

The spec was written as a targeted patch for the 141-failure regression. The implemented solution is architecturally cleaner: context is carried as handler closures rather than data fields, which aligns with the effect-handler model throughout the rest of the pipeline.
