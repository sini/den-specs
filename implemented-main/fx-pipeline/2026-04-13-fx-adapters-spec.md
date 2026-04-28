# Effects-Based Adapters

**Date:** 2026-04-13
**Status:** Approved design
**Branch:** feat/fx-resolution
**Location:** `nix/lib/aspects/fx/`

## Goal

Add adapter support to the fx resolution pipeline. Adapters are effect handlers that intercept include resolution to exclude, substitute, or select aspects. They compose via handler stacking — per-aspect `meta.adapter` installs scoped handlers, and context (host/user) is just the root of the resolution tree.

## Model

Resolution walks a tree of includes. At each node, if the aspect has `meta.adapter`, a scoped handler is installed for that subtree via `fx.rotate`. Two named effects drive the adapter system:

- **`resolve-include`** — emitted for each child before resolution. `param` is the child aspect (or bare function wrapped in an envelope). The handler decides: keep (resume with child), substitute (resume with replacement + tombstone for original), tombstone (resume with exclusion marker), or drop (resume with null).

- **`resolve-complete`** — emitted after a child is fully resolved (including recursive includes). `param` is the resolved child with identity envelope. Handlers can inspect the result for collection (paths, trace, module imports). This fires at every level of recursion — a handler installed at the root scope will see every resolved descendant in the tree.

### Handler stacking

When entering an aspect with `meta.adapter`:

1. Outer handler stack handles context args (host, user, class, aspect-chain)
2. `meta.adapter` installs a new `fx.rotate` layer for that subtree
3. Inner includes emit `resolve-include` effects
4. The adapter handler intercepts and decides keep/exclude/substitute
5. Unknown effects (context args, resolve-complete) rotate through to outer handlers

Nested `meta.adapter` on a child overrides the parent's — the innermost handler wins for `resolve-include`. This matches existing semantics where nested adapters don't compose through.

**Adapter tag propagation is eliminated.** The existing `filterIncludes` explicitly tags surviving children with `meta.adapter` so the adapter propagates down. With `fx.rotate`, propagation is structural — the rotate layer scopes to the entire subtree automatically, so children without their own adapter still have the parent's handler active. No explicit tagging needed.

### Adapter ownership tracking

For diagnostic attribution, adapters must track which aspect declared them. The existing `filterIncludes` now propagates `meta.adapterOwner` (see `feat/new-trace-demo`). In the fx model, this is natural — the handler closure captures the declaring aspect's identity:

```nix
mkAdapter = declarer: adapterHandlers:
  let ownerKey = pathKey (aspectPath declarer); in
  builtins.mapAttrs (_: handler:
    { param, state }:
    let result = handler { inherit param state; }; in
    result // (lib.optionalAttrs (result ? resume && builtins.isAttrs result.resume) {
      resume = result.resume // { meta = (result.resume.meta or {}) // { adapterOwner = ownerKey; }; };
    })
  ) adapterHandlers;
```

### Composing multiple adapters on one aspect

When `meta.adapter` needs both `excludeAspect A` and `excludeAspect B`, it composes as a single handler set with merged logic:

```nix
meta.adapter = let
  excludeA = excludeAspect aspectA;
  excludeB = excludeAspect aspectB;
in handlers: args:
  # Chain: apply A's filter, then B's filter
  excludeA (excludeB handlers) args;
```

Or in the fx model, since adapter handlers are attrsets, a composite adapter installs a single `resolve-include` handler that checks multiple conditions:

```nix
meta.adapter = {
  "resolve-include" = { param, state }:
    let ap = aspectPath param; in
    if ap == aspectPath refA || ap == aspectPath refB
    then { resume = tombstone param { ... }; inherit state; }
    else { resume = param; inherit state; };
};
```

### Context as root

A host or user carries adapter policies via `meta.adapter` on the root aspect. Since the context is the root of the resolution tree, its adapter applies to all includes. No special "context-level" vs "per-aspect" distinction — the tree is uniform.

## Identity: aspectPath

All adapter operations compare aspects by structural identity, not reference equality. Port from existing `adapters.nix`:

```nix
aspectPath = a: (a.meta.provider or []) ++ [ (a.name or "<anon>") ];
pathKey = path: lib.concatStringsSep "/" path;
toPathSet = paths: builtins.listToAttrs (map (p: { name = pathKey p; value = true; }) paths);
```

These are pure functions with no pipeline coupling.

## Tombstones

Excluded aspects become tombstone markers with metadata for debugging and tracing:

```nix
tombstone = resolved: extra: {
  name = "~${resolved.name or "<anon>"}";
  meta = (resolved.meta or {}) // {
    excluded = true;
    originalName = resolved.name or "<anon>";
  } // extra;
  includes = [];
};
```

Tombstones are visible in trace output and to `collectPaths` (filtered out by default). They carry `meta.excludedFrom` and optionally `meta.replacedBy`. The `meta.adapterOwner` field (from the declaring aspect) is preserved for diagnostic attribution.

## Adapter operations

### excludeAspect

Handler that tombstones aspects matching a reference's aspectPath, including transitive descendants (provider sub-aspects):

```nix
excludeAspect = ref:
  let refPath = aspectPath ref; in
  {
    "resolve-include" = { param, state }:
      let ap = aspectPath param; in
      if ap == refPath || lib.take (builtins.length refPath) ap == refPath
      then { resume = tombstone param { excludedFrom = state.ownerName or "<anon>"; }; inherit state; }
      else { resume = param; inherit state; };
  };
```

### substituteAspect

Handler that replaces matching aspects. `resolveChild` returns a list — tombstone + replacement — matching the existing `concatMap processInclude` pattern:

```nix
substituteAspect = ref: replacement:
  let refPath = aspectPath ref; in
  {
    "resolve-include" = { param, state }:
      if aspectPath param == refPath
      then {
        resume = [
          (tombstone param { excludedFrom = state.ownerName or "<anon>"; replacedBy = replacement.name or "<anon>"; })
          replacement
        ];
        inherit state;
      }
      else { resume = [ param ]; inherit state; };
  };
```

All `resolve-include` handlers resume with a list. `resolveChild` flattens with `concatMap`. This eliminates the `_substitute` sentinel — clean composition, direct port of the existing list-based approach.

### includeIf

Conditional include: adds aspects to the tree only when a guard predicate passes. The guard receives a context object with `hasAspect` for querying structural presence. Replaces `oneOfAspects` with a more general pattern.

```nix
# User-facing API:
includes = [
  (includeIf (ctx: ctx.hasAspect sops-nix) [ sops-config ])
  (includeIf (ctx: !ctx.hasAspect sops-nix) [ age-config ])  # fallback
];
```

The guard runs against the **raw** (pre-filter) subtree, same as existing `oneOfAspects` which pre-scans via `collectPathsInner`. This avoids the chicken-and-egg: guards see what's declared, not what survives filtering.

Implementation as a `resolve-include` handler:

```nix
includeIf = guardFn: aspects:
  let
    # Marker so resolveDeep recognizes conditional includes
    aspectKeys = toPathSet (map aspectPath aspects);
  in
  {
    name = "<includeIf>";
    meta = { conditional = true; guard = guardFn; aspects = aspects; };
    includes = aspects;  # declared but gated
  };
```

At resolution time, when `resolveDeep` encounters an `includeIf` node, it:

1. Builds a context object with `hasAspect` backed by the raw subtree paths
2. Evaluates `guardFn ctx`
3. If true: includes the aspects normally (emit `resolve-include` for each)
4. If false: emits tombstones for each (visible in trace, excluded from modules)

```nix
# In resolveDeep, when processing a child that's an includeIf:
resolveConditional = condNode: rawPathSet:
  let
    ctx = { hasAspect = ref: rawPathSet ? ${pathKey (aspectPath ref)}; };
    pass = condNode.meta.guard ctx;
  in
  if pass then
    map (a: fx.send "resolve-include" a) condNode.includes
  else
    map (a: fx.pure (tombstone a { excludedFrom = "includeIf"; guardFailed = true; })) condNode.includes;
```

This subsumes `oneOfAspects`:

```nix
# oneOfAspects [A, B] becomes:
includes = [
  (includeIf (ctx: ctx.hasAspect A) [ A ])
  (includeIf (ctx: !ctx.hasAspect A && ctx.hasAspect B) [ B ])
];
```

## Collection adapters

### collectPaths

Handler on `resolve-complete` that accumulates aspect paths in state:

```nix
collectPathsHandler = {
  "resolve-complete" = { param, state }:
    let isExcluded = param.meta.excluded or false; in
    {
      resume = param;
      state = state // {
        paths = state.paths ++ (lib.optional (!isExcluded) (aspectPath param));
      };
    };
};
```

Initial state: `{ paths = []; }`. Result accessed from final handler state.

### structuredTrace

Handler on `resolve-complete` producing flat queryable entries for the `feat/new-trace-demo` diagnostic pipeline. Each entry carries:

```nix
{
  name, fullName, class, parent, provider,
  isParametric, fnArgNames, hasClass, hasAdapter,
  excluded, excludedFrom, replacedBy, isProvider,
  adapterOwner
}
```

This replaces the nested `trace` adapter. The handler accumulates entries in state:

```nix
structuredTraceHandler = class: {
  "resolve-complete" = { param, state }:
    let
      entry = {
        name = param.name or "<anon>";
        class = class;
        provider = param.meta.provider or [];
        excluded = param.meta.excluded or false;
        excludedFrom = param.meta.excludedFrom or null;
        replacedBy = param.meta.replacedBy or null;
        hasClass = param ? ${class};
        hasAdapter = (param.meta.adapter or null) != null;
        adapterOwner = param.meta.adapterOwner or null;
        # Parametric detection deferred — requires aspect-chain inspection
      };
    in {
      resume = param;
      state = state // { entries = state.entries ++ [ entry ]; };
    };
};
```

Initial state: `{ entries = []; }`.

Parametric detection (inspecting raw `aspect-chain` elements for `functionArgs`) requires access to the chain at resolve time. The `resolve-complete` param has the resolved envelope, not the raw function. Two options:

1. Carry `isParametric`/`fnArgNames` through the identity envelope from `wrapIdentity` (set during resolution when we know it's a function)
2. Inspect `aspect-chain` in a pre-resolve handler

Recommendation: option 1 — add `meta.isParametric` and `meta.fnArgNames` to the identity envelope when the aspect is a function. Clean, no extra effects needed.

### module

Handler on `resolve-complete` that collects class modules into `{ imports }`:

```nix
moduleHandler = class: {
  "resolve-complete" = { param, state }:
    let
      mods = if param ? ${class} && !(param.meta.excluded or false)
        then [ param.${class} ]
        else [];
    in {
      resume = param;
      state = state // { imports = state.imports ++ mods; };
    };
};
```

## Default adapter pipeline

The existing `default = filterIncludes module` (adapters.nix:159) becomes:

1. `resolveDeep` walks the tree, emitting `resolve-include` and `resolve-complete` effects
2. Per-aspect `meta.adapter` handlers are installed via `fx.rotate` for their subtrees (replacing `filterIncludes`)
3. `moduleHandler class` is installed at the root scope to collect `{ imports }` (replacing `module`)

```nix
# The default resolution pipeline in fx:
defaultResolve = { ctx, class, aspect-chain, aspect }:
  let
    comp = resolveDeepEffectful { inherit ctx class aspect-chain; } aspect;
  in
  fx.handle {
    handlers = moduleHandler class;
    state = { imports = []; };
  } comp;
  # Returns { value = resolvedTree; state = { imports = [...]; }; }
```

## Changes to resolveDeep

`resolveDeep` becomes fully effectful — returns `Computation aspect`. Each child emits `resolve-include` before resolution and `resolve-complete` after. The entire tree walk is a computation so handler state threads through:

```nix
resolveDeepEffectful = { ctx, class, aspect-chain, strict ? false }:
  let
    resolver = if strict then resolveOneStrict else resolveOne;

    go = aspectVal:
      let
        resolved = resolver { inherit ctx class aspect-chain; } aspectVal;
        metaAdapter = resolved.meta.adapter or null;
      in
      # Install scoped adapter handler if present
      resolveChildren metaAdapter (resolved.includes or []) (resolvedIncludes:
        fx.pure (resolved // { includes = resolvedIncludes; })
      );

    # Resolve a single child: emit resolve-include, recurse if approved, emit resolve-complete.
    resolveChild = child:
      let envelope = wrapChildEnvelope child; in
      fx.bind (fx.send "resolve-include" envelope) (approvedList:
        # approvedList is a list (supports substitution returning [tombstone, replacement])
        sequenceChildren approvedList
      );

    sequenceChildren = children:
      builtins.foldl'
        (acc: child: fx.bind acc (results:
          if child == null then fx.pure results
          else if child.meta.excluded or false then
            # Tombstone: emit resolve-complete but don't recurse
            fx.bind (fx.send "resolve-complete" child) (_:
              fx.pure (results ++ [ child ])
            )
          else
            # Live child: recurse, then emit resolve-complete
            fx.bind (go child) (resolvedChild:
              fx.bind (fx.send "resolve-complete" resolvedChild) (_:
                fx.pure (results ++ [ resolvedChild ])
              )
            )
        ))
        (fx.pure [])
        children;

    # Resolve all children, optionally under a scoped adapter handler.
    resolveChildren = metaAdapter: includes: cont:
      let
        childComp = builtins.foldl'
          (acc: child: fx.bind acc (results:
            fx.bind (resolveChild child) (childResults:
              fx.pure (results ++ childResults)
            )
          ))
          (fx.pure [])
          includes;
      in
      if metaAdapter != null then
        # Install scoped adapter via rotate
        fx.bind
          (fx.rotate {
            handlers = metaAdapter;
            state = { ownerName = aspectVal.name or "<anon>"; };
          } childComp)
          (rotatedResult:
            cont rotatedResult.value  # unwrap rotate's { value; state; }
          )
      else
        fx.bind childComp cont;

  in go;
```

The existing non-effectful `resolveDeep` is preserved for backward compatibility with existing tests. The new `resolveDeepEffectful` is the adapter-aware variant.

### Dropped from existing pipeline

- **`probeTransform`** — existed to detect substitutions by probing the adapter with a fake inner. The effects approach makes substitution explicit via the `resolve-include` handler response (list with tombstone + replacement). No probing needed.
- **`filter`, `map`, `mapAspect`, `mapIncludes` combinators** — these are general-purpose adapter combinators. The fx model doesn't need them because handler attrsets compose directly via `//` and `fx.rotate` stacking. If a specific use case emerges, they can be added as handler combinators later.

## Compatibility with feat/new-trace-demo

The `feat/new-trace-demo` branch introduces `structuredTrace`, `adapterOwner` tracking, and context-stage tagging. The fx adapter spec accounts for:

| Trace-demo requirement | Fx coverage |
|----------------------|-------------|
| `meta.adapter` / `meta.adapterOwner` | Handler closure captures declaring aspect identity |
| `meta.excluded` / `meta.excludedFrom` / `meta.replacedBy` | Tombstone fields match exactly |
| `meta.provider` preservation | `wrapIdentity` copies provider from aspectVal |
| Parametric detection (`isParametric`, `fnArgNames`) | Added to identity envelope via `wrapIdentity` |
| `aspect-chain` for raw chain inspection | Preserved as handled effect, accessible to trace handlers |
| Context stage tagging (`__ctxStage`, `__ctxKind`) | Deferred to ctx-apply integration |
| `structuredTrace` adapter | `structuredTraceHandler` collects flat entries via `resolve-complete` |

## File plan

```
nix/lib/aspects/fx/
  adapters.nix   — aspectPath, pathKey, toPathSet, tombstone,
                    excludeAspect, substituteAspect, includeIf,
                    collectPathsHandler, structuredTraceHandler, moduleHandler
  resolve.nix    — resolveDeepEffectful, resolveChild, sequenceChildren, resolveChildren
  default.nix    — updated exports
```

## Test plan

Tests in `templates/ci/modules/features/fx-adapters.nix`:

1. **excludeAspect** — aspect excluded by path, tombstone with metadata
2. **excludeAspect transitive** — excluding parent also excludes provider sub-aspects
3. **substituteAspect** — aspect replaced, tombstone + replacement both in tree
4. **includeIf guard passes** — guarded aspects included when predicate returns true
5. **includeIf guard fails** — guarded aspects tombstoned when predicate returns false
6. **includeIf with hasAspect** — guard queries structural presence of other aspects
7. **includeIf fallback pattern** — mutual exclusion via complementary guards
8. **collectPaths** — all non-tombstone paths collected from resolved tree
8. **structuredTrace** — flat entries with name, provider, excluded, hasClass fields
9. **module** — class modules collected into `{ imports }`
10. **nested meta.adapter** — inner adapter overrides outer, doesn't compose through
11. **context root adapter** — adapter on root aspect applies to all descendants
12. **default pipeline** — resolveDeepEffectful + moduleHandler produces correct imports

## What this spec skips

- `ctx-apply.nix` integration (cross-context namespaces, `__ctxStage`/`__ctxKind` tagging) — separate spec
- Performance optimization — correctness first
- `onlyIf` adapter — exercise for after adapters ship
- Backward compatibility with existing adapter API — fx is a parallel pipeline

## Resolved questions

1. **Substitution representation.** `resolve-include` handlers resume with a list. Substitution returns `[ tombstone replacement ]`. No sentinel values. Matches existing `concatMap processInclude` pattern.
2. **Handler state threading.** `fx.rotate` threads state sequentially through handled effects. State persists across sibling includes within the rotate scope. `oneOfAspects` pre-computes the winner from raw paths to match existing semantics.
3. **resolveDeep return type.** Fully effectful `resolveDeepEffectful` returns `Computation aspect`. Existing `resolveDeep` preserved as non-effectful for backward compatibility.

## Open questions

1. **`collectPathsRaw` for includeIf guards.** The guard needs a raw path set to back `hasAspect`. This is equivalent to the existing `collectPathsInner`. Should it be computed once at the root and threaded through, or computed per-subtree? Root-level is simpler and sufficient since guards query the whole tree.
2. **Performance of foldl' bind chains.** Sequential folding over includes creates O(n) bind nodes. For wide aspect trees (50+ includes), this may be slow. nix-effects' FTCQueue makes bind O(1) amortized, but the total chain length matters for the trampoline. Benchmark during implementation.
3. **includeIf as an aspect vs a marker.** The spec shows `includeIf` returning an aspect-shaped attrset with `meta.conditional`. Alternative: make it a special effect (`fx.send "conditional-include" { guard; aspects; }`) that the resolver handles. The aspect shape is simpler for users but requires `resolveDeep` to detect `meta.conditional`.
