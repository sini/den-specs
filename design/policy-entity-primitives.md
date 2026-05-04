# Policy Iteration and Entity Resolution as Primitive Compositions

## Problem

The policy iteration loop (`policy/iterate.nix`) and entity resolution handler (`resolve-schema-entity.nix`) contain multi-step orchestration with inline state mutations. While the iterate loop benefits from staying as a recursive function, its internal steps (dispatch, record fired, emit effects, widen context) are opaque to tracing. Entity resolution's scope management combines multiple concerns into one handler body.

## Design Principles

1. The iterate loop stays as a recursive function — iteration structure is simple recursion.
2. Each **step** inside the loop is a named effect — observable, testable.
3. Scope entry/exit are atomic effects that encapsulate their full responsibility.
4. No unconditional sequence split across multiple effects — if A always follows B, they're one effect.

## New Effects

### Policy Iteration Steps

| Effect | Payload | Resume | Purpose |
|--------|---------|--------|---------|
| `dispatch-policies` | `{ directPolicies, aspectPolicies, firedPolicies, resolveCtx }` | dispatch result (effects + enrichment + firedNames) | Run policies against context, classify results |
| `record-fired` | `{ entityKind, firedPolicies }` | null | Record which policies fired at this scope |
| `emit-policy-effects` | `{ effects, entityKind, currentCtx, enrichedCtx }` | resolved results | Emit final effects when converged (excludes → routes/provides/instantiates → includes or schema resolves) |
| `widen-context` | `{ enrichment, entityKind, currentCtx }` | null | Update scope context with new enrichment keys |

### Entity Resolution Steps

| Effect | Payload | Resume | Purpose |
|--------|---------|--------|---------|
| `push-scope` | `{ scopedCtx, entityClass, parentScope }` | `{ scopeId, scopeHandlers }` | Atomic scope entry: set currentScope, register context, inherit policies, copy parent deferred |
| `restore-scope` | `{ parentScope }` | null | Reset currentScope to parent |
| `propagate-routes` | `{ scopeId }` | null | Copy relevant root complex routes to child scope |

### Effects That Stay

- `resolve-entity` — unchanged (looks up entity from schema)
- `resolve` — from unified resolve spec (walk the entity tree)
- `drain` — from unified resolve spec (explicit drain after entity walk)
- `register-route`, `register-provide`, `register-instantiate` — state mutations (unchanged)
- `register-constraint`, `register-aspect-policy` — state mutations (unchanged)

### Effects Absorbed

| Current | Absorbed Into |
|---------|--------------|
| Inline `state.modify` for scope push (resolve-schema-entity lines 21-41) | `push-scope` |
| Inline `state.modify` for deferred copy (resolve-schema-entity lines 121-138) | `push-scope` |
| Inline `state.modify` for fired policy recording (iterate.nix recordFired) | `record-fired` |
| Inline `state.modify` for scope context widen (iterate.nix widenAndContinue) | `widen-context` |
| `policyEmitExcludes` + `policyEmitEffects` + `policyEmitIncludes` + `processSchemaResolves` | `emit-policy-effects` |

## Data Flow

### Policy Iterate Loop

The `iterate` function stays as a recursive `go`. Each iteration sends effects for its steps:

```
go = iteration: accEnrichment: accEffects: firedPolicies: resolveCtx:

  send "dispatch-policies" { directPolicies, aspectPolicies, firedPolicies, resolveCtx }
  → dispatched = { schemaEffects, includeEffects, excludeEffects, routeEffects,
                    instantiateEffects, provideEffects, enrichment, firedNames }

  newEnrichKeys = check convergence (pure)
  combinedEffects = merge accEffects + dispatched (pure)
  updatedFired = firedPolicies + new (pure)

  if converged (newEnrichKeys == []):
    send "record-fired" { entityKind, updatedFired }
    send "emit-policy-effects" { combinedEffects, entityKind, currentCtx, enrichedCtx }

  if not converged:
    if iteration >= max: throw cycle error
    send "widen-context" { enrichment = dispatched.enrichment, entityKind, currentCtx }
    enterScope enrichHandlers (drain fires automatically via scope-widened)
    go (iteration + 1) combinedEnrichment combinedEffects updatedFired nextResolveCtx
```

### dispatch-policies handler (~15 lines)

Runs policies against context, classifies results:

```
run directPolicies against resolveCtx (filter by arg satisfaction + not already fired)
run aspectPolicies against resolveCtx (same filter)
classify each result (schema / enrichment / include / exclude / route / instantiate / provide)
extract tagged effects
resume combined dispatch result
```

### record-fired handler (~10 lines)

Records which policies fired at this scope (for late-dispatch visibility):

```
dispatchKey = "${entityKind}@${state.currentScope}"
update state.firedPolicyNames.${dispatchKey} = firedPolicies
resume null
```

### emit-policy-effects handler (~12 lines)

Emits converged effects in the correct order:

```
send excludes (register-constraint for each)
send routes/instantiates/provides (register-* for each)
if schemaEffects != []: processSchemaResolves (entity fan-out)
else: emit includes (via emit-include for each)
resume results
```

Note: `processSchemaResolves` is the current schema resolution logic (ctx-seen dedup, fan-out, late-dispatch). It stays as a function that sends `resolve-schema-entity` effects. Future decomposition could make it effect-driven too, but it's not in scope here.

### widen-context handler (~10 lines)

Updates scope context with new enrichment:

```
combinedEnrichment = currentCtx enrichment + new enrichment
update state.scopeContexts.${currentScope} = enrichedCtx
resume null
```

### resolve-schema-entity handler (revised, ~12 lines)

Composition of scope effects + entity walk:

```
send "push-scope" { scopedCtx, entityClass, parentScope = state.currentScope }
→ { scopeId, scopeHandlers }
scope.provide scopeHandlers (
  send "resolve-entity" { kind = targetKind }
  → rawEntity
  merge includes (pure: stripCtxId + concatenate)
  send "resolve" { entity, identity, ctx }
  send "drain" { ctx = scopedCtx }
  → satisfiable
  walkDeferred satisfiable (re-resolve via "resolve")
  send "propagate-routes" { scopeId }
)
send "restore-scope" { parentScope }
```

### push-scope handler (~15 lines)

Atomic scope entry — ALL scope setup in one handler:

```
newScopeId = mkScopeId scopedCtx
scopeHandlers = constantHandler (scopedCtx // optional { class = entityClass })
state updates:
  currentScope = newScopeId
  scopeContexts.${newScopeId} = scopedCtx
  scopeParent.${newScopeId} = parentScope (if different)
  scopedAspectPolicies.${newScopeId} = parent's policies (inheritance)
  scopedDeferredIncludes.${newScopeId} = parent's deferred (fan-out copy)
resume { scopeId = newScopeId, scopeHandlers }
```

### restore-scope handler (~5 lines)

```
state.currentScope = parentScope
resume null
```

### propagate-routes handler (~12 lines)

Copy relevant root complex routes to child scope:

```
rootRoutes = state.scopedRoutes.${rootScopeId}
childClasses = state.scopedClassImports.${scopeId}
relevantRoutes = filter (complex + fromClass exists in childClasses) rootRoutes
tag with sourceScopeId = scopeId
append to state.scopedRoutes.${scopeId}
resume null
```

## Late Dispatch Pass

The `lateDispatchPass` in `policy/schema.nix` currently iterates over sibling scopes and re-dispatches aspect policies. With the new effects:

```
for each sibling:
  send "dispatch-policies" { aspectPolicies filtered, firedPerScope, resolveCtx }
  → late effects
  if hasLateEffects:
    send "push-scope" sibling scope (temporarily)
    scope.provide (emit late effects)
    send "restore-scope"
```

This reuses `dispatch-policies` (same handler, different input). The late-dispatch specific logic (filtering already-fired, targeting sibling scopes) stays in the calling function.

## enterScope Integration

From the unified resolve spec, `enterScope` wraps `scope.provide` + `scope-widened`:

```nix
enterScope = handlers: computation:
  fx.effects.scope.provide handlers (
    fx.bind (fx.send "scope-widened" { ctx }) (_: computation)
  );
```

The iterate loop uses `enterScope` for enrichment widen (automatic drain). Entity resolution does NOT use `enterScope` — it uses `push-scope` + explicit `drain` at the right point.

## File Structure Changes

```
nix/lib/aspects/fx/
├── policy/
│   ├── classify.nix             # (stays)
│   ├── dispatch.nix             # (stays — pure dispatch logic, used by dispatch-policies handler)
│   ├── apply.nix                # (stays — effect emission helpers)
│   ├── schema.nix               # (stays — processSchemaResolves, lateDispatchPass)
│   ├── iterate.nix              # simplified: go loop sends effects, no inline state mutations
│   └── default.nix              # installPolicies entry
├── handlers/
│   ├── dispatch-policies.nix    # NEW: run + classify policies
│   ├── record-fired.nix         # NEW: record fired policy names
│   ├── emit-policy-effects.nix  # NEW: emit converged effects
│   ├── widen-context.nix        # NEW: update scope context
│   ├── push-scope.nix           # NEW: atomic scope entry
│   ├── restore-scope.nix        # NEW: scope exit
│   ├── propagate-routes.nix     # NEW: root route propagation
│   ├── resolve-schema-entity.nix # SIMPLIFIED: composition of new effects
│   └── ...                       # (others stay)
```

## Functions That Become Effects

| Current Function | New Effect | Location |
|-----------------|-----------|----------|
| `mkDispatch` (policy/dispatch.nix) | Internal to `dispatch-policies` handler | handlers/dispatch-policies.nix |
| `recordFired` (policy/iterate.nix) | `record-fired` | handlers/record-fired.nix |
| `emitFinalEffects` (policy/iterate.nix) | `emit-policy-effects` | handlers/emit-policy-effects.nix |
| `widenAndContinue` (policy/iterate.nix) | `widen-context` + `enterScope` | handlers/widen-context.nix |
| `pushScope` (resolve-schema-entity.nix) | `push-scope` | handlers/push-scope.nix |
| `copyDeferredToScope` (resolve-schema-entity.nix) | Part of `push-scope` | handlers/push-scope.nix |
| `propagateRootRoutes` (resolve-schema-entity.nix) | `propagate-routes` | handlers/propagate-routes.nix |
| `mkScopeTransition` (resolve-schema-entity.nix) | Deleted (logic split into push-scope + restore-scope) | — |

## Functions That Stay as Functions

| Function | Reason |
|----------|--------|
| `mergeEffects` | Pure data merge (no state, no decision) |
| `classifyPolicyResult` | Pure classification |
| `extractTaggedEffects` | Pure extraction |
| `decomposeSchemaEffect` | Pure decomposition |
| `processSchemaResolves` | Orchestrator that sends resolve-schema-entity (stays for now) |
| `stripCtxId` | Pure recursive transform |
| `walkDeferred` | Loop that sends "resolve" for each satisfiable (stays as function) |
| `emitLateForSibling` | Per-sibling late dispatch logic |

## Design Invariants

1. **Every handler ≤15 lines** — state mutations are atomic, named effects
2. **iterate stays as recursive function** — loop structure is explicit, steps are effects
3. **push-scope is atomic** — all scope entry concerns (context, policies, deferred copy) in one effect
4. **No inline state.modify in iterate** — all mutations go through named effects
5. **dispatch-policies is reusable** — called by both iterate (main dispatch) and lateDispatchPass (re-dispatch)
6. **Tracing sees everything** — dispatch, convergence decision (implicit in converge/widen branch), fired recording, effect emission, scope push/pop

## Migration Strategy

### Phase 1: New handlers

1. `push-scope` handler
2. `restore-scope` handler
3. `propagate-routes` handler
4. `dispatch-policies` handler
5. `record-fired` handler
6. `emit-policy-effects` handler
7. `widen-context` handler

### Phase 2: Update resolve-schema-entity

8. Rewrite handler to use push-scope + restore-scope + propagate-routes
9. Remove mkScopeTransition, copyDeferredToScope (absorbed into push-scope)

### Phase 3: Update iterate

10. Rewrite `go` to send dispatch-policies + record-fired + emit-policy-effects + widen-context
11. Remove recordFired, emitFinalEffects, widenAndContinue functions
12. Use enterScope for enrichment widen (from unified resolve spec)

### Phase 4: Update late dispatch

13. lateDispatchPass uses dispatch-policies + push-scope/restore-scope
14. Remove emitLateForSibling inline logic (replaced by handler composition)

### Phase 5: Cleanup

15. Remove dead functions
16. Verify test suite at each phase boundary

## Testing Strategy

- Each new handler gets unit tests
- iterate convergence tests verify dispatch-policies → record-fired → emit-policy-effects sequence
- Entity resolution tests verify push-scope → resolve → drain → propagate → restore sequence
- Late-dispatch tests verify cross-sibling policy visibility with new effects
- Tracing tests verify new effect names appear in output
