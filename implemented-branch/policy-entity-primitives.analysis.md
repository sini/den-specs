# Analysis: Policy Iteration and Entity Resolution as Primitive Compositions

## Verdict
implemented
All seven new effect handlers shipped. The iterate loop sends named effects instead of inline state mutations. resolve-schema-entity composes push-scope + restore-scope + propagate-routes. One minor drift: `directPolicies` in the spec's dispatch-policies payload never materialised — only `aspectPolicies` exists.

## Delivery Target
feat/fx-pipeline

## Evidence

### New handler files — all present in `nix/lib/aspects/fx/handlers/`

| Handler | File | Key evidence |
|---------|------|-------------|
| `push-scope` | `push-scope.nix` | Lines 14–83: handles `"push-scope"`, sets `currentScope`, `scopeContexts`, `scopeParent`, `scopedAspectPolicies`, `scopedDeferredIncludes`; resumes `{ scopeId, scopeHandlers }` |
| `restore-scope` | `restore-scope.nix` | Lines 6–22: handles `"restore-scope"`, resets `currentScope = param.parentScope`; also pops `inLateDispatchStack` (post-spec addition) |
| `propagate-routes` | `propagate-routes.nix` | Lines 9–29: handles `"propagate-routes"`, reads `rootScopeId`, filters `complexForward` routes by `childClasses`, tags with `sourceScopeId` |
| `dispatch-policies` | `dispatch-policies.nix` | Lines 23–34: `mkDispatchPoliciesHandler` wraps `mkDispatch`; filters by `flatConstraintRegistry` exclusions |
| `record-fired` | `record-fired.nix` | Lines 6–17: handles `"record-fired"`, writes `firedPolicyNames[entityKind@currentScope]` |
| `emit-policy-effects` | `emit-policy-effects.nix` | Lines 13–53: `mkEmitPolicyEffectsHandler`; separates cross-provider vs independent includes, delegates schema fan-out to `processSchemaResolves` |
| `widen-context` | `widen-context.nix` | Lines 6–16: handles `"widen-context"`, updates `scopeContexts[currentScope]` with `currentCtx // enrichment` |

All seven registered in `handlers/default.nix` lines 32–37.

### iterate.nix — no inline state mutations

`nix/lib/aspects/fx/policy/iterate.nix` lines 36–93: `go` sends `"dispatch-policies"` (line 36), `"record-fired"` (line 55), `"emit-policy-effects"` (line 64), `"widen-context"` (line 83). Zero `state.modify` / `state.get` calls — confirmed by grep.

### resolve-schema-entity.nix — composition pattern

`handlers/resolve-schema-entity.nix` lines 92–112: sends `"push-scope"`, then calls `resolveEntityInScope` which runs `"resolve-entity"` → `"resolve"` → `"drain"` → `walkDeferred` → `"propagate-routes"` → `"restore-scope"`. Matches spec pseudocode exactly.

### lateDispatchPass uses push-scope / restore-scope

`policy/schema.nix` lines 117–165: `emitLateForSibling` sends `"push-scope"` (line 146) and `"restore-scope"` (line 164). Uses `fx.effects.state.get` for initial read rather than `"dispatch-policies"` effect (see Drift).

### Functions absorbed — confirmed absent

- `mkScopeTransition` — not present anywhere in codebase
- `copyDeferredToScope` — not present anywhere
- `recordFired` function — absorbed; only the handler name survives as `recordFiredHandler` in `record-fired.nix`
- `emitFinalEffects` — not present; absorbed into `emit-policy-effects` handler
- `widenAndContinue` — not present; absorbed into `widen-context` handler

### enterScope integration

`policy/iterate.nix` line 89: uses `enterScope enrichHandlers (fx.pure null)` for enrichment widen. `enterScope` defined in `aspect.nix` line 27.

### policy/ directory unchanged

`policy/classify.nix`, `dispatch.nix`, `apply.nix`, `schema.nix`, `iterate.nix`, `default.nix` — all six files present, matching spec's "stays" list.

## Current Status
Still exists. All handlers are active in the pipeline. 713/713 tests pass per spec header.

## Supersession
none — this spec is a self-contained refactor. No earlier spec describes this design; the spec itself references the "unified resolve spec" for `enterScope` and `drain`, which are implemented in `aspect.nix` and `handlers/drain.nix`.

## Gaps
None significant. One spec table entry never shipped: `directPolicies` in the `dispatch-policies` payload. The spec's effect table shows `{ directPolicies, aspectPolicies, firedPolicies, resolveCtx }` but the handler (`dispatch-policies.nix` line 25–31) receives only `aspectPolicies`, `firedPolicies`, `resolveCtx`. Den appears to have consolidated to aspect-only dispatch, eliminating the direct-policy branch entirely.

## Drift

1. **No `directPolicies` branch.** The spec's `dispatch-policies` effect payload included `directPolicies` as a separate collection. Implementation uses only `aspectPolicies`. All policies appear to be registered as aspect policies.

2. **`inLateDispatchStack` in push-scope / restore-scope.** Spec says `restore-scope` takes `{ parentScope }` and resets `currentScope`. Implementation also pushes/pops `inLateDispatchStack` to track per-scope late-dispatch state (`push-scope.nix` lines 38–39, `restore-scope.nix` lines 8–11). This is a post-spec addition to prevent O(N²) re-dispatch.

3. **`emit-policy-effects` splits includes.** Spec describes a flat `includeEffects` emission. Implementation (`emit-policy-effects.nix` lines 25–50) separates `crossProviderIncludes` (same source policy as a schema effect) from `independentIncludes` (no associated schema effect), routing them differently. This resolves a multi-policy-at-one-scope bug not anticipated in the spec.

4. **`lateDispatchPass` skips `dispatch-policies` effect.** Spec says late dispatch should reuse `dispatch-policies` + `push-scope`/`restore-scope`. `emitLateForSibling` (`schema.nix` lines 118–165) calls `dispatchAspect` directly via `fx.effects.state.get`, bypassing the named effect. The push-scope / restore-scope calls are present, but the dispatch step remains inline rather than going through the observable `"dispatch-policies"` effect.

5. **`emit-policy-effects` handler is a constructor, not a plain handler.** Spec describes it as a direct handler. Implementation exports `mkEmitPolicyEffectsHandler` requiring `processSchemaResolves` to be injected, making it a factory to avoid a circular import (`emit-policy-effects.nix` line 13). Same pattern applies to `dispatch-policies` (`mkDispatchPoliciesHandler`). The observable effect name is unchanged.
