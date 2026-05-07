# Effect Vocabulary Reference

**Branch:** feat/fx-pipeline (753/753 tests)
**Scope:** Complete catalog of pipeline effects, their payloads, and handler locations.

---

## Design Principles

1. Every decision point and observable state transition is an effect.
2. Pure data transformation remains as functions.
3. No handler contains logic for multiple shapes — shape dispatch is a pure router.
4. Handlers are ≤15 lines.
5. Scope entry/exit are atomic effects encapsulating their full responsibility.

## 1. Resolution Chain

| Effect | Handler File | Payload | Resume |
|--------|-------------|---------|--------|
| `resolve` | `resolve.nix` | `{ aspect, identity, ctx, gated }` | resolved list |
| `compile` | `compile.nix` | `{ aspect, identity, ctx }` | compiled result |
| `compile-forward` | `compile-forward.nix` | `{ aspect, identity, ctx }` | `[]` |
| `compile-conditional` | `compile-conditional.nix` | `{ aspect, identity, ctx }` | results list |
| `compile-parametric` | `compile-parametric.nix` | `{ aspect, identity, ctx }` | results list |
| `compile-static` | `compile-static.nix` | `{ aspect, identity, ctx }` | resolved aspect |

### Shape routing (`compile`)

```
if aspect.meta ? __forward         → compile-forward
if aspect.meta ? guard             → compile-conditional
if (aspect.__args or {}) != {}     → compile-parametric
else                               → compile-static
```

Forwards and conditionals bypass dedup/constraints. Only parametric and static go through gating.

## 2. Gating

| Effect | Handler File | Payload | Resume |
|--------|-------------|---------|--------|
| `gate` | `gate.nix` | `{ aspect, identity, ctx }` | `{ blocked, result }` or `{ passed, owner? }` |
| `check-dedup` | `check-dedup.nix` | aspect | `{ isDuplicate, dedupKey }` |
| `check-constraint` | `constraint.nix` | `{ identity, aspect }` | action |

`gate` is the composite gating effect used by `compile-static` and `compile-parametric` (via `gate-tag.nix`). It sequences dedup then constraint checks and resumes `{ blocked, result }` if either rejects the aspect, or `{ passed, owner? }` on clean pass-through. The `gate-tag` utility (not an effect) calls `gate` and tags `constraintOwner` onto the aspect before continuing.

## 3. Bind Subsystem

| Effect | Handler File | Payload | Resume |
|--------|-------------|---------|--------|
| `bind` | `bind.nix` | `{ fn, args, aspect }` | `{ value }` or `{ deferred: true }` |
| `defer` | `defer.nix` | `{ child, requiredArgs }` | `[]` |
| `drain` | `drain.nix` | ctx attrset | satisfiable list |
| `scope-widened` | `scope-widen.nix` | `{ ctx }` | null |

The `bind` handler probes scope handlers for required args. If all satisfied, applies the function. If any missing, sends `defer`. `drain` partitions the deferred queue by satisfiability. `scope-widened` triggers automatic drain on scope entry via `enterScope`.

## 4. Static Compilation

| Effect | Handler File | Payload | Resume |
|--------|-------------|---------|--------|
| `classify` | `classify.nix` | `{ aspect, targetClass }` | `{ classKeys, nestedKeys }` |
| `emit-classes` | `emit-classes.nix` | `{ aspect, classKeys, identity }` | null (via fx.seq) |
| `emit-class` | `class-collector.nix` | `{ class, identity, module, ctx, ... }` | null |
| `resolve-children` | `resolve-children.nix` | `{ aspect, isMeaningful, chainIdentity }` | children list |

### resolve-children sequence

```
emitAspectPolicies → emitIncludes → installPolicies (if entity)
```

## 5. Policy Iteration

| Effect | Handler File | Payload | Resume |
|--------|-------------|---------|--------|
| `dispatch-policies` | `dispatch-policies.nix` | `{ directPolicies, aspectPolicies, firedPolicies, resolveCtx }` | dispatch result |
| `record-fired` | `record-fired.nix` | `{ entityKind, firedPolicies }` | null |
| `emit-policy-effects` | `emit-policy-effects.nix` | `{ effects, entityKind, enrichedCtx }` | resolved results |
| `widen-context` | `widen-context.nix` | `{ enrichment, currentCtx }` | null |

### Iterate loop (policy/iterate.nix)

```
go iteration accEnrichment accEffects firedPolicies resolveCtx:
  dispatch-policies → dispatched
  if converged (no new enrichment keys):
    record-fired → emit-policy-effects → done
  if not converged:
    widen-context → enterScope → go (iteration + 1)
```

### emit-policy-effects flow

1. Emit excludes (`register-constraint` per exclude)
2. Emit routes/instantiates/provides (`register-*`)
3. If schema resolves exist: `processSchemaResolves` (fan-out to child entities)
4. Independent includes: `policyEmitIncludes` (indexed names prevent dedup collision)

## 6. Entity Resolution / Scope

| Effect | Handler File | Payload | Resume |
|--------|-------------|---------|--------|
| `push-scope` | `push-scope.nix` | `{ scopedCtx, entityClass, parentScope }` | `{ scopeId, scopeHandlers }` |
| `restore-scope` | `restore-scope.nix` | `{ parentScope }` | null |
| `propagate-routes` | `propagate-routes.nix` | `{ scopeId }` | null |
| `resolve-entity` | `pipeline.nix` | `{ kind }` | raw entity |
| `resolve-schema-entity` | `resolve-schema-entity.nix` | `{ scopedCtx, entityClass, ... }` | resolved list |

### resolve-schema-entity sequence

```
push-scope → scope.provide scopeHandlers (
  resolve-entity → merge includes → resolve → drain → walkDeferred → propagate-routes
) → restore-scope
```

### push-scope atomicity

Single handler performs all scope entry:
- Set `currentScope`
- Register scope context
- Initialize scope's policy registry (no parent inheritance — scope-local dispatch)
- Record scope parent
- Copy parent deferred (fan-out)

## 7. Include/Policy Registration

| Effect | Handler File | Payload | Resume |
|--------|-------------|---------|--------|
| `emit-include` | `include.nix` | `{ child, idx, __parentScopeHandlers?, __parentCtxId? }` | emitted results |
| `include-unseen` | `include.nix` | key string | null |
| `register-aspect-policy` | `policy.nix` | `{ name, fn, ownerIdentity }` | null |
| `register-constraint` | `constraint.nix` | `{ type, scope, identity, owner }` | null |
| `register-route` | `route.nix` | route spec | null |
| `register-provide` | `provide.nix` | provide spec | null |
| `register-instantiate` | `instantiate.nix` | instantiate spec | null |
| `register-pipe-effect` | `register-pipe-effect.nix` | pipe effect spec | null |

## 8. Chain / Identity

| Effect | Handler File | Payload | Resume |
|--------|-------------|---------|--------|
| `chain-push` | `chain.nix` | `{ identity }` | null |
| `chain-pop` | `chain.nix` | null | null |
| `resolve-complete` | `identity.nix` | resolved aspect | the aspect (pass-through) |
| `get-path-set` | `identity.nix` | null | path set attrset |
| `ctx-seen` | `ctx.nix` | `{ key, aspects, aspectValues }` | `{ isFirst, newAspectValues }` |

`resolve-complete` is handled by `identity.collectPathsHandler`: it records the aspect into `state.pathSet` (for `hasAspect` queries) and resumes with the aspect unchanged. `get-path-set` reads that accumulated set back out. Both handlers live in `identity.nix`, not in the handlers/ directory.

## 9. Scope Primitives

| Primitive | Location | Purpose |
|-----------|----------|---------|
| `enterScope` | `aspect.nix` | `scope.provide handlers` + send `scope-widened` (auto-drain) |
| `ctxFromHandlers` | `aspect.nix` | Probe handlers to extract ctx values |
| `constantHandler` | `handlers/ctx.nix` | Create handler that always resumes with a fixed value |
| `mkScopeId` | `pipeline.nix` | Deterministic scope identity from ctx keys |

## File Map

```
nix/lib/aspects/fx/
├── identity.nix             # resolve-complete + get-path-set handlers (collectPathsHandler, pathSetHandler)
├── pipeline.nix             # resolve-entity handler + defaultHandlers composition + mkScopeId
└── handlers/
    ├── default.nix          # handler composition (merges all handler sets)
    ├── resolve.nix          # resolve → compile delegation
    ├── compile.nix          # shape router
    ├── compile-forward.nix  # forward → route registration
    ├── compile-conditional.nix  # guard → include/tombstone
    ├── compile-parametric.nix   # bind → tag → re-resolve
    ├── compile-static.nix       # classify → emit → children
    ├── gate.nix             # composite dedup + constraint gate effect
    ├── gate-tag.nix         # utility: calls gate, tags constraintOwner (not a handler set)
    ├── bind.nix             # scope probe + apply
    ├── defer.nix            # stub + queue
    ├── drain.nix            # partition satisfiable
    ├── scope-widen.nix      # auto-drain trigger
    ├── classify.nix         # key classification
    ├── emit-classes.nix     # batch class emission
    ├── class-collector.nix  # emit-class → scope-partitioned state
    ├── resolve-children.nix # policies + includes orchestration
    ├── dispatch-policies.nix    # run + classify policies
    ├── record-fired.nix     # record fired policy names
    ├── emit-policy-effects.nix  # emit converged effects
    ├── widen-context.nix    # update scope context
    ├── push-scope.nix       # atomic scope entry
    ├── restore-scope.nix    # scope exit
    ├── propagate-routes.nix # root route propagation
    ├── resolve-schema-entity.nix # entity fan-out orchestration
    ├── include.nix          # emit-include + include-unseen
    ├── chain.nix            # chain-push + chain-pop
    ├── constraint.nix       # register-constraint + check-constraint
    ├── check-dedup.nix      # dedup check
    ├── ctx.nix              # ctx-seen + constantHandler
    ├── forward.nix          # buildForwardAspect utility (no effects — used by compile-forward)
    ├── policy.nix           # register-aspect-policy
    ├── provide.nix          # register-provide
    ├── route.nix            # register-route
    ├── instantiate.nix      # register-instantiate
    ├── register-pipe-effect.nix # register-pipe-effect
    └── state-util.nix       # scopedMerge / scopedAppend helpers
```
