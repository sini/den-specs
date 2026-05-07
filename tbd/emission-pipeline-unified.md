# Unified Emission Pipeline: Classes and Traits

## Problem

Class module emission (`emit-class`) and future trait emission (`emit-trait`) share a common pipeline:
1. Content unwrapping (from `aspectContentType` wrapper)
2. Per-element iteration (multi-site definitions)
3. Identity computation
4. Scope-partitioned collection with dedup

Currently this logic lives in `emitClasses` (a function) and will be duplicated for traits. Additionally, `wrapClassModule` (class-specific wrapping) is tangled with emission — it happens post-pipeline in `wrapCollectedClasses` but its concerns (arg detection, collision policy, partial application) are mixed in a 48-line function with multiple branches.

## Design Principles

1. **Shared unwrap → iterate → dispatch.** Classes and traits share the iteration pipeline. Only the collection target differs.
2. **Lazy values are just values.** The collector doesn't distinguish resolved from unresolved. Thunks flow through collection and resolve on consumer access via Nix laziness.
3. **Wrapping is class-specific and isolated.** `wrapClassModule` logic stays separate from emission — it's a post-pipeline concern that doesn't affect traits.
4. **Each step ≤15 lines.** No function handles multiple concerns.

## Emission Pipeline (Shared)

### Current: emit-classes handler

From the unified resolve spec, `emit-classes` is a batch effect that iterates class keys and sends `emit-class` per element. This becomes the shared pattern:

```
emit-content { aspect, keys, identity, ctx, target }
  for each key:
    unwrap contentType wrapper → list of { value, file }
    for each element at index:
      compute element identity (append [idx] if multi)
      send "emit-${target}" { key, identity, value, ctx, ... }
```

### Generalized Effects

| Effect | Payload | Resume | Purpose |
|--------|---------|--------|---------|
| `emit-content` | `{ aspect, keys, identity, ctx, target, meta }` | null | Shared iteration: unwrap → index → dispatch |
| `emit-class` | `{ class, identity, module, ctx, ... }` | null | Class-specific collection (existing, unchanged) |
| `emit-trait` | `{ trait, identity, value, ctx, chain }` | null | Trait-specific collection (new) |

`emit-content` is the batch orchestrator. It sends `emit-class` or `emit-trait` per element based on `target`. The `compile-static` handler sends `emit-content` twice — once for class keys, once for trait keys:

```
send "classify" → { classKeys, traitKeys, nestedKeys }
send "emit-content" { keys = classKeys, target = "class", ... }
send "emit-content" { keys = traitKeys, target = "trait", ... }
```

### emit-content handler (~15 lines)

Shared iteration logic:

```
for each key in keys:
  values = unwrapContentValuesList aspect.${key}
  isMulti = length values > 1
  for each (idx, value) in indexed values:
    elemIdentity = if isMulti then "${identity}[${idx}]" else identity
    send "emit-${target}" { key, identity = elemIdentity, value, ctx, meta }
resume null
```

Pure iteration — no collection logic, no dedup. Those live in the target-specific handlers.

### emit-class handler (existing, unchanged)

Receives per-element emission. Computes dedup key, checks `scopedEmittedLocs`, writes to `scopedClassImports`. No changes from current `class-collector.nix`.

### emit-trait handler (~12 lines)

```
scope = state.currentScope
entry = { value = param.value, chain = param.chain or [], identity = param.identity }
append entry to state.scopedTraits.${scope}.${param.key}
resume null
```

No dedup for traits (collection semantics handle duplicates at consumption). Just scope-partitioned append.

## Lazy Trait Values

### The insight

The collector stores VALUES — it doesn't evaluate them. A trait emission that's a function (needs `config`) is stored as-is:

```nix
# This aspect emits a lazy trait:
den.aspects.postgres = {
  backup-targets = { config, ... }: {
    user = config.services.postgresql.user;
    paths = [ config.services.postgresql.dataDir ];
  };
};
```

At pipeline-time, `backup-targets` is classified as a trait key. The value `{ config, ... }: ...` is collected into `scopedTraits` as a thunk. No resolution happens.

### Resolution paths

**Module-time consumers** (class modules with `{ backup-targets, ... }:`):

The `_den.traits` injection module resolves thunks against the entity's own config:

```nix
config._den.traits.${traitName} = applyCollectionStrategy schema (
  map (entry:
    if builtins.isFunction entry.value
    then entry.value { inherit config lib; }
    else entry.value
  ) collectedEntries
);
```

**Cross-entity consumers** (via scope inheritance + fleet):

A sibling entity's trait access forces the source entity's NixOS eval via Nix laziness. The pipeline collected the thunk — the consumer's access triggers the eval chain:

```
Host B accesses backup-targets trait
→ trait entry points to Host A's collected thunk
→ accessing thunk forces Host A's _den.traits resolution
→ which forces Host A's NixOS evalModules
→ which evaluates config.services.postgresql.user
→ value returned to Host B's consumer
```

No custom resolution machinery. Nix's native lazy evaluation handles the ordering.

**Pipeline-time discriminators** (parametric `{ backup-targets, ... }:` in includes):

CAN access lazy trait values — forcing triggers source entity eval via Nix laziness. This creates an evaluation dependency (source pipeline + instantiation must complete). Safe as long as no cycle exists (same constraint as fleet).

### What this enables

Pipeline-time aggregation of config-dependent data:

```nix
# The pipeline collects backup-targets from ALL hosts with postgres
# Collection semantics: list concatenation (automatic)
# The backup-server's class module receives the aggregated list:

den.aspects.borgbackup-server = {
  nixos = { backup-targets, lib, ... }: {
    services.borgbackup.repos = lib.listToAttrs (map (target:
      lib.nameValuePair target.user {
        path = "/var/lib/borg/${target.user}";
        paths = target.paths;
      }
    ) backup-targets);
  };
};
```

Each element in `backup-targets` is forced on access — Nix evaluates the source entity's config lazily. The pipeline did the collection; Nix does the resolution.

## Class Module Wrapping (Isolated Concern)

### Current state

`wrapClassModule` (48 lines) does:
1. Detect which function args match den context keys
2. Warn on missing schema-matching args
3. Partially apply den args
4. Build collision-policy wrapper for remaining args
5. Generate validator module

### Decomposition

| Function | Responsibility | Lines |
|----------|---------------|-------|
| `detectDenArgs` | Partition function args into den-matched vs remaining | ~8 |
| `checkMissingArgs` | Warn on schema-matching args not in context | ~6 |
| `applyDenArgs` | Partially apply matched args with collision policy | ~12 |
| `buildValidator` | Generate collision-check validator module | ~8 |
| `wrapFunctionModule` | Compose: detect → check → apply → validate | ~10 |

### detectDenArgs (~8 lines)

```
allArgs = functionArgs module
argNames = attrNames allArgs
denArgNames = filter (k: ctx ? k) argNames
remainingArgs = removeAttrs allArgs denArgNames
→ { denArgNames, remainingArgs, allArgs }
```

### checkMissingArgs (~6 lines)

```
schemaKinds = schemaArgKinds
missingDenArgNames = filter (k: elem k schemaKinds && !allArgs.${k}) (filter (k: !(ctx ? k)) argNames)
if missingDenArgNames != []:
  warn each → { module = warned, wrapped = false, unsatisfied = true, missingArgs }
```

### applyDenArgs (~12 lines)

Three cases (no branching — dispatch by shape):
- `remainingArgs == {}` → full application: `module denArgs`
- `remainingArgs != {}` → partial application with collision policy wrapper

```
denArgs = genAttrs denArgNames (k: ctx.${k})
if remainingArgs == {}: { module = module denArgs; wrapped = true; }
else:
  policy = resolveCollisionPolicy { ... }
  classWinsNames = filter (name: policy name == "class-wins") denArgNames
  wrapper = moduleArgs: module (classWinsDen // moduleArgs // denWinsDen)
  { module = setFunctionArgs wrapper advertisedArgs; wrapped = true; validator; }
```

### buildValidator (unchanged from current mkCollisionValidator)

Probes `config._module.args` for each den arg name, emits warnings/errors per collision policy.

### wrapFunctionModule (composition, ~10 lines)

```
detected = detectDenArgs module ctx
checked = checkMissingArgs detected module
if checked.unsatisfied: return checked
if detected.denArgNames == []: { module; wrapped = false; }
else: applyDenArgs { inherit module ctx detected aspectPolicy globalPolicy; }
```

### Trait arg detection (future addition)

When traits land, `detectDenArgs` expands to also detect trait arg names:

```
traitArgNames = filter (k: traitSchemas ? k) argNames
```

These get lazy thunk pre-application (reading from `config._den.traits`) rather than den ctx pre-application. Same pipeline, different source.

## classify Handler (Updated)

From the unified resolve spec, `classify` returns key categories. With traits:

```
classify { aspect, targetClass }:
  allKeys = filter non-structural (attrNames aspect)
  if registries empty: { classKeys = allKeys; traitKeys = []; nestedKeys = []; }
  else:
    classKeys = filter (isClass or isTargetClass) allKeys
    nonClassKeys = filter (not class) allKeys
    traitKeys = filter (k: traitRegistry ? k) nonClassKeys
    remainingKeys = filter (not trait) nonClassKeys
    nestedKeys = filter isNested remainingKeys
    unregisteredClassKeys = filter (not nested) remainingKeys
  resume { classKeys, traitKeys, nestedKeys, unregisteredClassKeys }
```

Four categories: class (registered), trait (registered), nested (detected), unregistered (fallback to class).

## Integration with Unified Resolve Spec

The `compile-static` handler from the unified resolve spec becomes:

```
send "gate" → if blocked, return
probe "class" handler → targetClass
send "classify" { aspect, targetClass }
→ { classKeys, traitKeys, nestedKeys, unregisteredClassKeys }
send "emit-content" { keys = classKeys ++ unregisteredClassKeys, target = "class", ... }
send "emit-content" { keys = traitKeys, target = "trait", ... }
registerConstraints aspect
for nestedKeys: send "resolve" { subAspect }
send "resolve-children" { aspect, chainIdentity, isMeaningful }
```

Two `emit-content` sends — one for classes, one for traits. Same handler, different target.

## File Structure

```
nix/lib/aspects/fx/
├── handlers/
│   ├── emit-content.nix         # NEW: shared unwrap → iterate → dispatch
│   ├── emit-trait.nix           # NEW: trait collection (scope-partitioned append)
│   ├── class-collector.nix      # RENAMED from emit-class logic (stays)
│   ├── classify.nix             # UPDATED: 4-category classification
│   └── ...
├── module/
│   ├── detect-args.nix          # NEW: den arg + trait arg detection
│   ├── apply-args.nix           # NEW: partial application with collision policy
│   ├── validator.nix            # EXTRACTED from class-module.nix
│   ├── wrap.nix                 # SIMPLIFIED: composition of detect → check → apply
│   └── default.nix
```

## Migration Strategy

### Phase 1: Extract class-module helpers

1. Extract `detectDenArgs` from `wrapFunctionModule`
2. Extract `checkMissingArgs`
3. Extract `applyDenArgs`
4. Move to `module/` directory
5. `wrapFunctionModule` becomes composition

### Phase 2: Generalize emission

6. Implement `emit-content` handler (shared iteration)
7. Update `emit-classes` (from unified resolve spec) to use `emit-content` internally, or replace with direct `emit-content` sends from compile-static
8. Existing `emit-class` handler unchanged (already narrow)

### Phase 3: Add trait emission (when traits land)

9. Add `emit-trait` handler
10. Update `classify` to return `traitKeys`
11. `compile-static` sends second `emit-content` for trait keys
12. Add `_den.traits` injection module (resolves lazy thunks)
13. Update `detectDenArgs` to detect trait arg names
14. Add lazy thunk pre-application in `applyDenArgs`

### Phase 4: Trait collection semantics

15. Add `traitCollectorHandler` (scope-partitioned state)
16. Add `traitArgHandler` (pipeline-time consumption)
17. Add scope inheritance (via scopeParent walk)
18. Add `register-trait-schema` effect

## Design Invariants

1. **emit-content is target-agnostic** — unwrap + iterate is the same for classes and traits
2. **Collector doesn't evaluate** — stores whatever it receives (resolved or thunk)
3. **Lazy resolution via Nix** — no custom thunk management, no deferred state
4. **Wrapping is class-specific** — traits don't go through wrapClassModule
5. **classify is the single classification point** — 4 categories, no branching elsewhere
6. **Cross-entity lazy access is safe** — same cycle constraints as fleet (no mutual config deps)
7. **Pipeline-time trait access forces eval** — discriminators can read lazy traits, triggering source eval

## Testing Strategy

- Phase 1: existing tests pass (pure extraction, no behavior change)
- Phase 2: emit-content produces same `emit-class` calls as current `emitClasses`
- Phase 3: new trait tests verify collection, scope partitioning, lazy resolution
- Phase 4: trait consumption tests verify aggregation, inheritance, collection strategies
- Lazy trait tests: verify source entity eval triggered on consumer access without cycle
