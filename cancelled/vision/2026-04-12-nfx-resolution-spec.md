# Spec: nfx-powered aspect resolution

**Date:** 2026-04-12
**Status:** Vision document for discussion with Vic
**Depends on:** [nfx](https://github.com/vic/nfx) v0.1+, PR #449 (strict mode / class registration)

## Goal

Replace den's hand-rolled resolution pipeline (`resolve.withAdapter`,
`filterIncludes`, `ctx-apply`) with nfx effects and handlers. The
user-facing API stays attrset-based. The internal machinery gets
smaller, more composable, and opens doors that the current architecture
can't.

## Why

Den's resolution pipeline is already doing algebraic effects informally:

- A parametric aspect `{ host, ... }: { nixos = ...; }` is a suspended
  computation awaiting context. That's `nfx.pending`.
- `parametric.fixedTo { host } aspect` provides context to a suspended
  aspect. That's `nfx.provide`.
- `resolve.withAdapter adapter class tree` interprets the tree by
  walking it and calling the adapter at each node. That's `nfx.runFx`
  with a composed handler.
- `meta.adapter` intercepts resolution of child aspects and can
  tombstone, substitute, or transform them. That's an effect handler
  (`nfx.via`).
- `filterIncludes` composes parent and child adapters into a chain.
  That's handler composition.

The informal version works but has limitations:

1. Adapters that need to query the tree (`oneOfAspects`, `onlyIf`)
   have to bypass `filterIncludes` with a raw walker to avoid
   re-entering themselves. With nfx, the handler controls evaluation
   order and can answer queries without re-entry.

2. `hasAspect` can't be used inside `includes` (cyclic). With nfx, it
   becomes a `request` that the handler resolves in a separate phase.

3. Adding a new adapter type requires understanding `filterIncludes`
   internals. With nfx, it's a new handler function.

## The mapping, concretely

### Aspects as effects

```nix
# Today: a parametric aspect is a Nix function
aspect = { host, ... }: {
  nixos = { ... };
  includes = [ child1 child2 ];
};

# With nfx: an aspect is a pending effect
aspect = nfx.pending (ctx:
  nfx.immediate ctx {
    nixos = { ... };
    includes = [ child1 child2 ];
  }
);

# Static aspects are immediate effects
aspect = nfx.immediate {} {
  nixos = { ... };
  includes = [ child1 child2 ];
};
```

The conversion is mechanical. Static attrsets become `immediate`.
Functions become `pending`. The rest of the aspect structure (class
config, includes, provides, meta) stays the same inside the effect
value.

### Resolution as runFx

```nix
# Today
resolve = withAdapter adapters.default;
mainModule = resolve class aspect;

# With nfx
resolve = class: aspect:
  nfx.runFx (
    nfx.provide { inherit class; aspect-chain = []; }
      (resolveEffect class aspect)
  );
```

Where `resolveEffect` walks the aspect tree as an effect chain:

```nix
resolveEffect = class: aspect:
  nfx.do [
    # Get the current aspect's value (forces pending effects)
    (_: toEffect aspect)

    # Extract class module
    (resolved: nfx.pure {
      classModule = resolved.${class} or null;
      includes = resolved.includes or [];
      meta = resolved.meta or {};
    })

    # Recurse into includes
    (info:
      let
        childModules = map (child:
          resolveEffect class child
        ) info.includes;
      in
      nfx.pure {
        imports =
          (lib.optional (info.classModule != null) info.classModule)
          ++ map (c: nfx.runFx (nfx.provide ctx c)) childModules;
      }
    )
  ];
```

### Adapters as handlers

```nix
# Today: filterIncludes wraps an inner adapter and processes meta.adapter
filterIncludes = inner: args@{ aspect, ... }:
  let metaAdapter = aspect.meta.adapter or null; in
  if metaAdapter != null then
    # compose, processInclude, tag...
  else
    inner args;

# With nfx: a handler that intercepts "resolve-child" requests
filterIncludesHandler = inner:
  nfx.handle "resolve-child" (child:
    let metaAdapter = child.meta.adapter or null; in
    if metaAdapter != null
    then metaAdapter (filterIncludesHandler inner) child
    else inner child
  );
```

Each adapter type becomes a handler function:

```nix
# excludeAspect: handle resolve-child, tombstone if path matches
excludeAspectHandler = ref: inner:
  nfx.handle "resolve-child" (child:
    if aspectPath child == aspectPath ref
    then nfx.pure (tombstone child {})
    else inner child
  );

# substituteAspect: handle resolve-child, swap if path matches
substituteHandler = ref: replacement: inner:
  nfx.handle "resolve-child" (child:
    if aspectPath child == aspectPath ref
    then inner replacement
    else inner child
  );

# oneOfAspects: handle resolve-child with structural query
oneOfAspectsHandler = candidates: inner:
  nfx.do [
    # Request the structural presence set (a new ability)
    (_: nfx.request "structural-presence" {})

    (presence:
      let
        present = filter (c: presence ? ${pathKey (aspectPath c)}) candidates;
        losers = if present == [] then [] else tail present;
      in
      # Compose excludeAspect handlers for each loser
      foldl' (h: loser: excludeAspectHandler loser h) inner losers
    )
  ];
```

The `request "structural-presence"` is the key change. Instead of
each adapter doing its own tree walk (with `collectPathsInner`
bypassing `filterIncludes`), the request goes to a handler that
knows the tree's structural state. The handler answers from a
precomputed set. No re-entry.

### The structural-presence handler

```nix
# Provided at the resolve root, answers structural queries
structuralPresenceHandler = tree:
  let
    # Walk the raw tree once, collect all paths
    allPaths = collectAllPaths tree;
    pathSet = toPathSet allPaths;
  in
  nfx.handle "structural-presence" (_:
    nfx.pure pathSet
  );
```

This handler is established ONCE at the root of resolution. Every
`request "structural-presence"` in any adapter gets the same
precomputed set. No redundant walks. The set is computed lazily (Nix
thunk) so it's only forced when someone actually asks.

### hasAspect as a request

```nix
# Today: post-resolution query on frozen structure
host.hasAspect <facter>  # reads config.resolved

# With nfx: a request during resolution
hasAspect = ref: nfx.request "has-aspect" ref;
# Handler at the resolve root:
hasAspectHandler = pathSet:
  nfx.handle "has-aspect" (ref:
    nfx.pure (pathSet ? ${pathKey (aspectPath ref)})
  );
```

This is the same semantics as our shipped `hasAspect`, but the query
can happen during resolution (not just after). The handler answers from
the same precomputed structural set.

### ctx-apply as effect composition

```nix
# Today: ctx-apply.nix traverses entity contexts, builds includes
ctxApply = self: ctx:
  parametric.withIdentity self {
    includes = assembleIncludes (traverse { ... });
  };

# With nfx: context stages are handler layers
ctxApply = self: ctx:
  nfx.do [
    (_: nfx.provide ctx (toEffect self))
    (resolved: nfx.pure (
      nfx.via (hostHandler ctx)
        (nfx.via (userHandler ctx)
          (resolveEffect resolved))
    ))
  ];
```

Each context stage (host, user, home) becomes a handler that
provides its context and composes with the next. The traversal
(`ctx.host.into.user`, `ctx.user.into.default`) becomes handler
composition via `nfx.via`.

## What stays the same

- **Aspect declaration.** Users still write `den.aspects.foo = { ... }`
  or `den.aspects.foo = { host, ... }: { ... }`. The module system
  still handles merging. nfx is internal.

- **Type system.** `aspectType`, `providerType`, `coercedProviderType`
  stay. The submodule structure is unchanged. nfx replaces what
  happens AFTER the types have merged the declarations.

- **adapters.nix public API.** `excludeAspect`, `substituteAspect`,
  `oneOfAspects`, `collectPaths`, `trace` stay as user-facing names.
  Their implementations change from filterIncludes-wrapped functions
  to nfx handler constructors, but the call-site syntax is the same
  (they're still `meta.adapter = excludeAspect <ref>`).

- **Entity options.** `host.hasAspect`, `host.mainModule`,
  `host.resolved`, `config.class`, `config.classes` all stay.

## What changes internally

| Today | With nfx |
|---|---|
| `resolve.withAdapter adapter class tree` | `nfx.runFx (nfx.provide ctx (resolveEffect class tree))` |
| `filterIncludes inner` | Handler that intercepts `resolve-child` requests |
| `adapters.default = filterIncludes module` | Handler composition: `filterIncludesHandler moduleHandler` |
| `meta.adapter = excludeAspect ref` | Handler: `excludeAspectHandler ref` |
| `collectPaths` (raw tree walk) | `request "structural-presence"` answered by root handler |
| `ctx-apply.nix assembleIncludes` | `nfx.do` sequencing context stages as handler layers |
| `parametric.applyDeep takeFn ctx` | `nfx.provide ctx (toEffect aspect)` |
| `parametric.withOwn` | `nfx.adapt` with context-dependent branching |

## File changes

| File | Change |
|---|---|
| `nix/lib/aspects/resolve.nix` | Rewrite: `withAdapter` → `nfx.runFx` with handler composition |
| `nix/lib/aspects/adapters.nix` | Rewrite: adapters become nfx handler constructors. Public names stay. |
| `nix/lib/parametric.nix` | Simplify: `applyDeep`, `withOwn`, `applyIncludes` → nfx `provide`/`adapt` |
| `nix/lib/ctx-apply.nix` | Rewrite: traversal → nfx `do` sequencing with handler layers |
| `nix/lib/statics.nix` | Fold into resolve handlers (static = provide {class, aspect-chain}) |
| `nix/lib/aspects/has-aspect.nix` | Simplify: query becomes a request handler, collectPathSet uses root set |
| `modules/context/has-aspect.nix` | Unchanged (entity wiring stays) |
| `modules/context/host.nix` | Rewrite: context stage → nfx handler constructor |
| `modules/context/user.nix` | Same |
| `nix/lib/aspects/types.nix` | Unchanged (declaration types stay) |
| `flake.nix` | Add nfx input |

## Estimated scope

The rewrite touches ~6 internal files. The user-facing API is preserved.
Test suite stays — tests exercise the API, not the internals. Some
adapter tests that inspect internal structure (trace output format) may
need adjustment.

Core implementation: ~300 lines of nfx-based resolve/adapt/ctx-apply
replacing ~400 lines of hand-rolled pipeline code. Net reduction plus
cleaner composition.

## Open questions

1. **nfx as a flake input or vendored?** If den depends on nfx, it's
   a new input. Users get it transitively. Alternatively, vendor the
   kernel (~30 lines) and only the modules den needs.

2. **Performance.** nfx wraps every computation in `pending`/`immediate`
   attrsets. The extra allocations may be measurable on large configs.
   Profile before optimizing — Nix's lazy evaluation may avoid most of
   the overhead since unused branches aren't forced.

3. **Error reporting.** nfx's condition system could provide better
   error messages during resolution (e.g., "aspect X needs host
   context but was resolved without it" as a resumable condition
   rather than a Nix evaluation error). Worth exploring but not
   required for the initial integration.

4. **Migration path.** Can the nfx-based resolve coexist with the
   current resolve during transition? Probably yes — `resolve.withAdapter`
   can delegate to the nfx implementation internally while keeping the
   same function signature. Adapters that are already function values
   (`meta.adapter = excludeAspect ref`) can be detected and wrapped
   as nfx handlers automatically.
