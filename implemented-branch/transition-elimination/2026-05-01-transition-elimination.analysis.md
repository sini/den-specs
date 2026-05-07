# Analysis: Transition Elimination — Policies as Context Expansion

## Verdict
substantially-implemented (99.4% test pass rate, 4 edge cases remain open)
Core transition machinery deleted and replaced by policy-driven context expansion. `into-transition`, DLQ, and trait system are gone; `installPolicies` + `scope.provide` delivers the replacement. Four failing tests documented in the spec are still open as of the last status line.

## Delivery Target
feat/fx-pipeline

## Evidence

### transition.nix — deleted
No file at `nix/lib/aspects/fx/handlers/transition*`. No matches for `into-transition`, `emitTransitions`, `resolveContextValue`, `resolveFanOut`, `resolveTransition`, `processIncludeOnly`, `emitCrossProvider`, `iterateEnrichment`, `flattenInto`, or `transitionHandler` anywhere under `nix/lib/aspects/fx/`. Confirmed by grep with zero results.

### DLQ — gone
No matches for `scopedDeadLetterQueue`, `deadLetterQueue`, `DLQ`, `dead-letter`, or `dead_letter` under `nix/lib/aspects/fx/`. The spec's claim of -178 lines for DLQ elimination is confirmed by absence.

### Trait system — gone
No files under `nix/lib/aspects/fx/` match `trait`. State fields `scopedDeferredTraits`, `scopedConsumedTraits`, `scopedTraits` do not appear anywhere in the fx subtree. Spec claimed -2463 lines across 23 files.

### State fields removed per spec
Confirmed absent in `pipeline.nix` defaultState (lines 131–169):
- `scopeStack` — not present
- `scopeChildren` — not present
- `scopeProvenance` — not present
- `scopedForwardSpecs` — not present
- `transitionDepth` — not present

Confirmed present (retained as designed):
- `scopeParent` (`pipeline.nix:163`)
- `dispatchedPolicies` (`pipeline.nix:167`) — the spec said this was removed but it is present; see Gaps below
- `currentScope`, `scopeContexts`, `scopedClassImports`, `scopedRoutes`, `scopedInstantiates` — all present
- `inLateDispatch` (`pipeline.nix:169`) — new field not in spec, added during implementation

### installPolicies — shipped
`policy/default.nix:81` defines `installPolicies`. Called from `handlers/resolve-children.nix:47` inside `resolveChildSequence`, gated on `aspect ? __entityKind`. Uses `fx.effects.scope.provide` for context expansion (`policy/schema.nix:160`).

### dispatchPolicies iteration loop — shipped as iterate + fixed-point
`policy/iterate.nix` implements the fixed-point enrichment loop (lines 31–95) with `maxPolicyIterations = 10` cap (line 9). The `dispatch-policies` effect handler at `handlers/dispatch-policies.nix:23` wraps `mkDispatch`. This is the observable dispatch step called per iteration from `iterate`.

### mkDispatch — shipped (not removed, renamed/refactored)
`policy/dispatch.nix:59` defines `mkDispatch`. The spec said it would be "replaced by simpler dispatch" — in practice it was refactored and retained; it classifies policy results and extracts tagged effects. The handler wraps it for observability.

### push-scope / restore-scope — new mechanism replacing scope stack
`handlers/push-scope.nix` (84 lines): atomically sets `currentScope`, `scopeContexts`, `scopeParent`, resets `inLateDispatch`. `handlers/restore-scope.nix` (27 lines): restores `currentScope` and `inLateDispatch` from stack. No `scopeStack` attrset — the spec's "save/restore" model landed, but implemented with an explicit `inLateDispatchStack` for the late-dispatch flag.

### resolve-schema-entity — new handler implementing scope creation
`handlers/resolve-schema-entity.nix:91`: sends `push-scope` then walks entity tree via `resolve-entity` + `resolve`, then calls `drain` + `propagate-routes` + `restore-scope` (lines 97–112). This is the materialized form of the spec's steps 4–7 (set currentScope, walk entity, restore).

### wrapCollectedClasses / wrappedPerScope — present
`resolve.nix:55–56` produces `wrappedPerScope` by mapping `wrapCollectedClasses` over all scopes. `route/apply.nix:118–123` reads from `wrappedPerScope` for simple routes. Complex (forward-derived) routes fall back to `resolveSourceFallback` which runs a mini-pipeline (`fxResolve`) — the "post-pipeline forward source resolution" from Phase 1b.

### Forward sub-pipelines — partially retained
`handlers/compile-forward.nix` still exists. Tier 1 forwards become simple routes (line 26: `isTier1` check). Complex forwards are stored with `__complexForward = true` (line 37) and resolved via `applyComplexRoute` in `route/apply.nix:51`. The spec predicted full forward elimination but complexity remains for non-simple cases.

### Policy effect constructors — fully shipped
`lib/policy-effects.nix`: `resolve`, `include`, `exclude`, `route`, `instantiate`, `provide`, `pipe` — all defined. Additional constructors beyond spec: `resolve.shared`, `resolve.to`, `resolve.shared.to`, `resolve.withIncludes`, `pipelineOnly`, `for`, `when`.

### Fan-out — shipped
`policy/schema.nix:195–230`: `processSchemaResolvesInner` folds over schema effects with `isFanOut = length > 1`. Includes late-dispatch pass for sibling visibility (`lateDispatchPass`, line 173). The `ctx-seen` dedup check at line 87 matches the spec's dedup requirement.

### `into` key — still structural
`key-classification.nix:16` lists `"into"` in `structuralKeysSet`. The spec said `aspect.into` and `meta.into` would be removed; the key is still recognized as structural (not dispatched as a class key), meaning old code using it won't error — but it is not processed. `aspect.nix:48` propagates `into` through parametric resolution. Whether any user code still depends on it is not visible from this analysis.

### Spec-documented 4 remaining failures — not resolved
The spec's status block (and the live `docs/superpowers/specs/2026-05-01-transition-elimination.md` mirror) lists four open failures. All four test files exist in the repo, confirming they are real tests that were failing at time of writing.

## Current Status
Still exists on `feat/fx-pipeline`. The core architecture — `installPolicies`, `push-scope`/`restore-scope`, `resolve-schema-entity`, fixed-point iterate, `lateDispatchPass` — is the active pipeline machinery. No separate `transition.nix` file.

## Supersession
This spec supersedes `2026-04-30-forward-route-unification.md` (Tasks 2–5 absorbed here per spec). It is preceded in the spec chain by `2026-05-01-policies-as-handlers-redesign.md` which it references as the architectural design for `installPolicies`. The policies-as-handlers redesign was implemented concurrently.

## Gaps

1. **`dispatchedPolicies` state field** — spec claimed it was removed; it is present in `pipeline.nix:167` and used in `policy/default.nix:94,109`. The spec's status block (written 2026-05-02) says it was removed, but the code contradicts this. Either it was re-added after the status note, or the status note was premature.

2. **Trait system reimplementation** — spec deferred traits to fleet/den.exports. No trait files exist under `nix/lib/aspects/fx/`. This remains future work.

3. **`into` removal** — spec said `aspect.into` and `meta.into` would be deleted. `"into"` is still in `structuralKeysSet` and propagated through parametric resolution at `aspect.nix:48`. Partial: it is no longer dispatched but is not fully excised.

4. **Forward sub-pipeline not fully eliminated** — complex forwards still resolve via `applyComplexRoute` + `fxResolve` mini-pipeline. Spec predicted full elimination; reality is Tier 1 / Tier 2 split retained from earlier work.

5. **Four failing tests** (from spec's own status block, written 2026-05-02):
   - `den-as-lib.test-module-can-resolve-custom-domain`
   - `deadbugs-cybolic-routes.test-has-no-dups`
   - `user-host-mutual-config.test-host-parametric-unidirectional`
   - `user-host-mutual-config.test-user-provides-to-all-users`

## Drift

1. **`dispatchPolicies` as a named function was not the final form.** The spec designed a recursive `dispatchPolicies` function (~80 lines) that handles resolve effects as immediate context widening. Implementation split this into: `iterate` (fixed-point loop in `policy/iterate.nix`), `mkDispatch` (effect classification in `policy/dispatch.nix`), `processSchemaResolves` (fan-out + late-dispatch in `policy/schema.nix`), and `emitPolicyEffectsThen` (effect application in `policy/apply.nix`). More modular than spec; the "immediate widening per resolve" model became a fixed-point iteration with batched enrichment.

2. **Late-dispatch mechanism not in spec.** `lateDispatchPass` in `policy/schema.nix:173` and `inLateDispatch` / `inLateDispatchStack` in pipeline state are a new mechanism for sibling cross-visibility that the spec did not anticipate. Added to address the 4th failing test (alphabetical resolution order). Not a regression — it solves a real gap.

3. **`resolve-schema-entity` as explicit handler, not inline code.** Spec described steps 4–7 as inline logic inside `dispatchPolicies`. Implementation materialized them as a dedicated handler (`handlers/resolve-schema-entity.nix`), making the scope push/walk/drain/propagate/restore sequence observable via the effect system.

4. **`dispatchedPolicies` retained for per-entity dispatch dedup.** Spec said it would be removed as part of state simplification. Implementation kept it (`pipeline.nix:167`, `policy/default.nix:94,109`) to prevent double-dispatch at the same `entityKind@scope` key.
