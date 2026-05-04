# Unified Resolution with Effect-Driven Bind

## Problem

The current fx-pipeline has inconsistent abstraction levels:
- Some operations are effects (check-dedup, check-constraint, emit-class)
- Others are direct function calls (classifyKeys, emitClasses, resolveChildren)
- Parametric and static resolution are separate effects despite converging to the same path
- Deferral is a separate mechanism from binding, despite being "bind that can't resolve yet"
- Drain is explicit at two call sites rather than reactive to scope changes
- Large handlers contain multiple branches mixing "what kind of thing is this?" with "how to process it"

## Design Principles

1. Every decision point and observable state transition is an effect.
2. Pure data transformation remains as functions.
3. The bind/defer/drain lifecycle is a unified, self-contained subsystem.
4. **No handler contains logic for multiple shapes.** Shape dispatch is a pure router — each compile-* handler knows exactly what it received.
5. Handlers are ≤15 lines.

## Effect Vocabulary

### Resolution Chain

| Effect | Responsibility | Resume |
|--------|---------------|--------|
| `resolve` | Entry point. Observable interception point, delegates to `compile`. | resolved list |
| `compile` | Shape router. Zero logic, pure delegation. | compiled result |
| `compile-forward` | Tier classify + route register (no gate) | `[]` |
| `compile-conditional` | Guard eval + include or tombstone (no gate) | results list |
| `compile-parametric` | Gate + bind + tag + re-resolve (or defer) | results list |
| `compile-static` | Gate + classify + emit-classes + resolve-children | resolved aspect |
| `gate` | Dedup + constraint. Block or pass through. | `{ blocked, ... }` or `{ passed }` |

### Bind Subsystem

| Effect | Responsibility | Resume |
|--------|---------------|--------|
| `bind` | Probe scope handlers + apply fn | `{ value }` or `{ deferred: true }` |
| `defer` | Emit resolve-complete stub + queue in state | `[]` |
| `drain` | Partition deferred by satisfiability, return satisfiable | satisfiable list |
| `scope-widened` | Automatic drain trigger on scope entry | null |

### Static Compilation

| Effect | Responsibility | Resume |
|--------|---------------|--------|
| `classify` | Key classification (class / nested / unregistered) | `{ classKeys, nestedKeys, unregisteredClassKeys }` |
| `emit-classes` | Batch class module emission (sends emit-class per element) | null |
| `resolve-children` | Chain + policies + includes orchestration | children list |

### Unchanged (from current pipeline)

`emit-class`, `register-constraint`, `check-dedup`, `check-constraint`, `chain-push`, `chain-pop`, `resolve-complete`, `resolve-entity`, `resolve-schema-entity`, `register-route`, `register-provide`, `register-instantiate`, `register-aspect-policy`, `emit-include`, `include-unseen`, `get-path-set`, `ctx-seen`

### Removed

| Effect | Replaced By |
|--------|-------------|
| `resolve-parametric` | `compile-parametric` → `bind` |
| `resolve-aspect` | `compile-static` |
| `resolve-conditional` | `compile-conditional` |
| `emit-forward` | `compile-forward` |
| `defer-include` | `defer` |
| `drain-deferred` | `drain` + `scope-widened` |

## Data Flow

### Walker (emitIncludes)

The walker's only job is data preparation. No classification, no dispatch decisions:

```
for each child in includes:
  wrapChild         (normalize to aspect shape)
  propagateScope    (inherit parent scope/ctx)
  nameAnon          (synthetic name if anonymous)
  send "resolve" { aspect, identity, ctx }
```

All decision-making moved to handlers.

### resolve handler (~5 lines)

Pure entry — sends compile, no other logic:

```
send "compile" { aspect, identity, ctx }
```

Note: `resolve` is the public entry point from the walker. It exists as a named effect so tracing can observe "aspect X entered resolution." The handler is trivially small — its value is as an interception/observation point.

### compile handler (~8 lines)

Pure router — zero logic per branch, just shape → effect:

```
if aspect.meta ? __forward         → send "compile-forward" { aspect, identity, ctx }
if aspect.meta ? guard             → send "compile-conditional" { aspect, identity, ctx }
if (aspect.__args or {}) != {}     → send "compile-parametric" { aspect, identity, ctx }
else                               → send "compile-static" { aspect, identity, ctx }
```

**Design decision:** Forwards and conditionals bypass `gate` (dedup + constraints). Rationale:
- Forwards are routing declarations, not aspect inclusions — dedup doesn't apply.
- Conditionals have their own gating mechanism (guard) — constraints on top would double-gate.
- Only `compile-parametric` and `compile-static` go through `gate`.

### compile-forward handler (~15 lines)

Knows it received a forward. No gating, no shape checking:

```
spec = aspect.meta.__forward
classify tier (simple vs complex based on spec flags + state.scopedClassImports)
build route record (simpleRoute or complexRoute // { sourceScopeId, __complexForward })
send "register-route" route
resume []
```

### compile-conditional handler (~12 lines)

Knows it received a conditional. No gating, no shape checking:

```
send "get-path-set" → pathSet
evaluate aspect.meta.guard against { hasAspect = ... }
if pass: emitIncludes aspect.meta.aspects → resume results
if fail: tombstone all aspect.meta.aspects → resume tombstones
```

### gate handler (~12 lines)

Dedup + constraint check. Called by compile-parametric and compile-static before their main logic:

```
send "check-dedup" child
→ if isDuplicate: resume { blocked: true, result: [] }
send "check-constraint" { identity, aspect }
→ if action == "exclude": emit tombstone, resume { blocked: true, result: [tombstone] }
→ if action == "substitute": emit tombstone + resolve replacement, resume { blocked: true, result }
→ resume { passed: true }
```

### compile-parametric handler (~15 lines)

Gates, then binds. No shape checking:

```
send "gate" { aspect, identity }
→ if blocked: resume gate result
if depth >= maxDepth: throw cycle error
send "bind" { fn, args, aspect, requiredKeys }
→ { value }: build base → merge next → tag → send "resolve" (re-entry with depth+1)
→ { deferred: true }: resume [] (defer handler already handled stub + queue)
```

### compile-static handler (~12 lines)

Gates, then classifies + emits + resolves children. No shape checking:

```
send "gate" { aspect, identity }
→ if blocked: resume gate result
probe "class" handler → targetClass (or null)
send "classify" { aspect, targetClass }
send "emit-classes" { aspect, classKeys, identity, ctx, aspectPolicy, globalPolicy, isContextDep }
registerConstraints aspect  ← function call (sends multiple register-constraint effects)
for nestedKeys: send "resolve" { subAspect = emitNestedAspect aspect k, ... }
send "resolve-children" { aspect, chainIdentity, isMeaningful }
```

Note: `registerConstraints` stays as a function call (it's a loop over `register-constraint` effects, not a new decision point). `emitNestedAspect` is a pure helper that builds the sub-aspect attrset.

### bind handler (~15 lines)

Probes scope handlers (via `fx.effects.hasHandler`):

```
requiredKeys = non-defaulted args not already in scopeHandlers
probe each key:
  all available → apply fn (fx.bind.fn) → resume { value }
  any missing → send "defer" { child = aspect, requiredArgs } → resume { deferred: true }
```

### defer handler (~8 lines)

Observable deferral event + resolve-complete stub:

```
emit stub = { name, meta.deferred = true, includes = [] }
send "resolve-complete" stub
queue { child, requiredArgs } in scoped deferred state
resume []
```

### drain handler (~10 lines)

Explicit drain (same semantics as current drain-deferred):

```
read deferred queue for current scope
partition by: all requiredArgs present in ctx
remove satisfiable from state
resume satisfiable list
```

### scope-widened handler (~8 lines)

Automatic drain on scope entry:

```
send "drain" { ctx }
→ satisfiable items
for each satisfiable: send "resolve" { aspect, identity, ctx }
resume null
```

### classify handler (~10 lines)

Key classification made observable:

```
partition aspect keys into classKeys, nestedKeys, unregisteredClassKeys
using schema registry + nested-key depth detection
resume { classKeys, nestedKeys, unregisteredClassKeys }
```

### emit-classes handler (~12 lines)

Batch emission — iterates class keys, unwraps content, delegates to emit-class:

```
for each classKey:
  unwrap content values
  for each module element:
    send "emit-class" { class, identity[idx], module, ctx, ... }
resume null
```

### resolve-children handler (~15 lines)

Child orchestration — chain + policies + includes:

```
chain-push chainIdentity (if isMeaningful)
emitAspectPolicies aspect
emitIncludes (aspect.includes)
installPolicies (if aspect.__entityKind present)
chain-pop (if isMeaningful)
send "resolve-complete" { resolved aspect }
resume children list
```

## Scope Entry

### enterScope (automatic drain)

```nix
enterScope = handlers: computation:
  fx.effects.scope.provide handlers (
    fx.bind (fx.send "scope-widened" { ctx = ctxFromHandlers handlers; }) (
      _: computation
    )
  );
```

Used by:
- `policy/iterate.nix` (enrichment widen)
- `compile-parametric` (scope injection for bind)

### resolve-schema-entity (explicit drain)

Retains precise orchestration:

```
pushScope (updates currentScope)
→ copyDeferredToScope (fan-out: duplicate parent deferred into child)
→ scope.provide scopeHandlers (
    resolve-entity → send "resolve" for entity
    → send "drain" { scopedCtx }  ← explicit, AFTER entity walk
    → walkDeferred (re-resolve satisfiable)
    → propagateRootRoutes
  )
→ restoreScope
```

Entity resolution needs drain AFTER the entity walk (walk may produce new deferred items). Does NOT use `enterScope`.

## Identity + Context in Payload

Computed once by the walker, carried through all effects:

- `identity` = `identity.key aspect` (node identity for dedup + class emission)
- `ctx` = `ctxFromHandlers (aspect.__scopeHandlers or {})` (from scope handlers)

Additional fields computed by `compile-static` for `emit-classes`:
- `chainIdentity` = `identity.pathKey (meta.provider ++ [name])` (for chain-push/pop)
- `aspectPolicy` = `aspect.meta.collisionPolicy or null`
- `globalPolicy` = `den.config.classModuleCollisionPolicy or "error"`
- `isContextDep` = derived from `__parametricResolvedArgs` + `meta.contextDependent`

Handlers receive these as data — never recompute.

## Depth Tracking

Parametric depth tracked on aspect attrset:
- `compile-parametric` checks `aspect.__parametricDepth or 0` before bind
- After successful bind: tagged result gets `__parametricDepth = depth + 1`
- `maxParametricDepth = 10` (unchanged)
- Stripped before static compilation (in `compile` router or `compile-static`)

## Migration Strategy

### Phase 1: New handlers (alongside old)

1. `gate` handler
2. `compile` router
3. `compile-forward` handler
4. `compile-conditional` handler
5. `compile-parametric` handler
6. `compile-static` handler
7. `bind` handler
8. `defer` handler
9. `drain` handler
10. `scope-widened` handler
11. `classify` handler
12. `emit-classes` handler
13. `resolve-children` handler
14. `enterScope` primitive

### Phase 2: Switch walker

15. Walker sends `resolve` only (remove all shape dispatch logic)
16. Walker computes identity + ctx in payload

### Phase 3: Switch scope entry

17. `policy/iterate` uses `enterScope`
18. `resolve-schema-entity` uses `drain` effect (replaces drain-deferred)
19. Remove `drainEnrichmentDeferred`

### Phase 4: Remove old

20. Delete `resolve-parametric`, `resolve-aspect`, `resolve-conditional` handlers
21. Delete `emit-forward` handler (logic now in `compile-forward`)
22. Delete `defer-include`, `drain-deferred` handlers
23. Delete `aspectToEffect`, `compileStatic`, `resolveChildren`, `emitClasses` functions
24. Delete `dispatchChild` from walker

### Phase 5: Cleanup

25. Dead code removal
26. Update tracing handler for new effects
27. Verify test suite at each phase boundary

## File Structure (post-migration)

```
nix/lib/aspects/fx/
├── handlers/
│   ├── resolve.nix              # resolve entry (gate → compile)
│   ├── gate.nix                 # dedup + constraint blocking
│   ├── compile.nix              # shape router (forward/conditional/parametric/static)
│   ├── compile-forward.nix      # forward tier + route registration
│   ├── compile-conditional.nix  # guard eval + include/tombstone
│   ├── compile-parametric.nix   # bind + tag + re-resolve
│   ├── compile-static.nix       # classify + emit + resolve-children
│   ├── bind.nix                 # scope probe + apply fn
│   ├── defer.nix                # stub + queue
│   ├── drain.nix                # partition + return satisfiable
│   ├── scope-widen.nix          # automatic drain trigger
│   ├── classify.nix             # key classification
│   ├── emit-classes.nix         # batch class emission
│   ├── resolve-children.nix     # chain + policies + includes
│   ├── constraint.nix           # (stays)
│   ├── chain.nix                # (stays)
│   ├── class-collector.nix      # (stays — emit-class singular)
│   ├── include.nix              # (simplified — just delegates to walker)
│   ├── provide.nix              # (stays)
│   ├── route.nix                # (stays)
│   ├── instantiate.nix          # (stays)
│   ├── policy.nix               # (stays)
│   ├── resolve-schema-entity.nix # (stays, uses explicit drain)
│   ├── check-dedup.nix          # (stays)
│   ├── ctx.nix                  # (stays)
│   ├── state-util.nix           # (stays)
│   └── default.nix
├── aspect/
│   ├── children.nix             # walker — normalize + send "resolve" only
│   ├── normalize.nix            # (stays)
│   ├── provide.nix              # (stays — emitAspectPolicies)
│   └── default.nix
├── policy/                       # (stays, iterate uses enterScope)
├── route/                        # (stays)
├── pipeline.nix                  # enterScope + handler wiring
├── resolve.nix                   # (stays — post-pipeline assembly)
├── identity.nix                  # (stays)
├── key-classification.nix        # (stays — used by classify handler)
├── content-util.nix              # (stays)
├── constraints.nix               # (stays)
├── trace.nix                     # (updated for new effects)
└── default.nix
```

## Design Invariants

1. **Every handler ≤15 lines** — single purpose, no multi-shape logic
2. **Identity + ctx flow as data** — computed by walker, never recomputed
3. **No handler branches on shape** — `compile` is a pure router (one if-chain, zero logic per branch)
4. **Bind subsystem is self-contained** — bind, defer, drain, scope-widened
5. **Pure functions only transform data** — wrapChild, propagateScope, nameAnon, stripCtxId, tagParametricResult
6. **Tracing observes every decision** — resolve, gate, compile-*, bind, defer, drain, classify
7. **Explicit drain for complex orchestration** — resolve-schema-entity controls timing
8. **resolve-complete always emitted** — success AND deferral produce resolve-complete

## Testing Strategy

- Each phase boundary: full test suite must pass
- Each new handler gets unit tests via `denTest`
- Tracing tests verify new effect names in output
- Deferred tests verify: automatic drain (enterScope), explicit drain (entity resolution), fan-out copy, stub emission
- Depth tests verify parametric cycle detection across resolve → compile-parametric → bind → resolve
- Gate tests verify: dedup blocks, constraint excludes/substitutes, clean pass-through
