# Spec: nix-effects-powered aspect resolution

**Date:** 2026-04-12
**Status:** Vision document for discussion with Vic
**Depends on:** [nix-effects](https://github.com/vic/nix-effects), PR #449 (strict mode / class registration)
**Supersedes:** `2026-04-12-nfx-resolution-spec.md` (used nfx primitives; this spec uses nix-effects)

## Goal

Replace den's hand-rolled context-threading and resolution pipeline
with nix-effects' freer monad, named effects, and scoped handlers.
The user-facing API stays attrset-based. Internally, the entire
`take`/`canTake`/`parametric`/`statics`/`applyDeep` stack disappears
and is replaced by effect requests and handler composition.

## Why nix-effects, not nfx

nfx (4 kernel primitives, context-passing model) is the conceptual
sketch. nix-effects is the production-ready version:

- **FTCQueue** for O(1) bind (not O(n^2) naive free monad). Tested
  at 1M operations, constant memory.
- **`builtins.genericClosure` trampoline** for O(1) stack depth.
  Deep aspect trees (100+ levels) don't blow Nix's stack.
- **Named effects via `send`/`handle`** with `resume`/`abort`
  semantics. Richer than nfx's `pending`/`adapt`.
- **Conditions system** (Common Lisp style resumable errors) for
  strict-mode diagnostics.
- **`bind.fn`** (nix-effects#12) decomposes `{ host, user }: body`
  into independent `send "host"` / `send "user"` requests.

## What gets replaced

The entire context-threading stack (~300 lines across 5 files) is
the source of every regression in the last month:

| File | Lines | Bugs it caused | Replaced by |
|---|---|---|---|
| `take.nix` | ~30 | #413 (carryAttrs), meta carryover | `send` + handler `resume` |
| `can-take.nix` | ~25 | #429 (coercion check) | Not needed: effects are named, not guessed |
| `parametric.nix` | ~135 | #419 (applyDeep), #423 (static sub), #426 (canTake.upTo gate) | `handle` + `bind.fn` |
| `statics.nix` | ~25 | — | Folded into a scoped handler |
| `aspects/types.nix` (coercion) | ~15 | #408 (coercedProviderType), #437 (factory coercion) | Not needed: `bind.fn` handles all function shapes |

Total: ~230 lines of bug-prone manual effects machinery replaced by
handler composition on top of nix-effects' proven kernel.

### Why these bugs happened

Vic's diagnosis from PR #446:

> Den currently threads its environment (context in Den parlance)
> `{ host, user }` explicitly, and we had some hard to find / fix
> bugs due to we manually doing it (or forgetting to do it).

Every bug in the list above was the manual context-threading
machinery getting a case wrong:

- #408: `coercedProviderType` wrapped factory functions by accident
- #413: `applyDeep` didn't propagate context to sub-includes
- #419: `applyDeep` was added to fix #413, broke static sub-aspects
- #423: the #419 fix dropped owned class configs on static subs
- #426: narrowed to `canTake.upTo` which missed static subs entirely
- #429: `coercedProviderType` predicate was too broad
- #437: factory functions needed empty-functionArgs exception
- meta carryover: `carryAttrs` only carried `name`, not `meta.provider`

With named effects, the computation TELLS the system what it needs
(`send "host" null`) instead of the system GUESSING from function
argument shapes (`functionArgs`, `canTake`, `isSubmoduleFn`).

## The core mapping

### Aspects as effectful computations

```nix
# Today: a parametric aspect is a Nix function
# The system guesses context requirements from functionArgs
aspect = { host, user, ... }: {
  nixos = { ... };
  includes = [ child1 child2 ];
};

# With nix-effects: each arg is an independent effect request
# The system knows requirements because they're explicit sends
aspect = fx.bind.fn ({ host, user }:
  fx.pure {
    nixos = { ... };
    includes = [ child1 child2 ];
  }
);

# Static aspects are pure (no requests)
aspect = fx.pure {
  nixos = { ... };
  includes = [ child1 child2 ];
};
```

`bind.fn` (nix-effects#12) decomposes the attrset argument into
independent `send` calls. `{ host, user }` becomes `send "host"`
then `send "user"`, each satisfied by whichever handler provides it.
No fixed context shapes. `{ host }`, `{ user }`, `{ host, user }`,
`{ host, user, home }` all work without different code paths.

### Context stages as scoped handlers

```nix
# Today: ctx.host manually threads { host } via fixedTo
ctx.host.provides.host = { host }: fixedTo { inherit host; } host.aspect;

# With nix-effects: host stage installs a scoped handler
resolveHost = host:
  fx.handle {
    host = { param, state }: { resume = host; inherit state; };
  } (resolveAspect host.aspect);

# User stage nests inside, adding "user" to the handler stack
resolveUser = host: user:
  fx.handle {
    user = { param, state }: { resume = user; inherit state; };
  } (resolveHost host);
```

Any computation nested inside the user handler that `send "user"`
gets the current user. Different users see different handlers.
Scoping is automatic: inner handlers shadow outer ones for their
sub-computation only. This is `parametric.fixedTo` with clean
scoping instead of explicit context threading.

### Resolution as `fx.run` with composed handlers

```nix
# Today
mainModule = resolve class aspect;
# Where resolve = withAdapter adapters.default, which walks the tree
# calling the adapter at each node

# With nix-effects: install all handlers, run the computation
mainModule = fx.run
  (fx.handle {
    # "resolve-child": how to process each include
    resolve-child = { param, state }:
      { resume = resolveAspect param; inherit state; };

    # "class-module": extract class config from an aspect
    class-module = { param, state }:
      { resume = param.${class} or null; inherit state; };

    # "structural-presence": the precomputed path set
    structural-presence = { param, state }:
      { resume = structuralSet; inherit state; };

    # "has-aspect": query structural presence by ref
    has-aspect = { param, state }:
      { resume = structuralSet ? ${pathKey (aspectPath param)};
        inherit state; };
  } (resolveAspect aspect))
  initialHandlers
  initialState;
```

### Adapters as handler combinators

```nix
# Today: filterIncludes wraps an inner adapter, processes meta.adapter
# Hand-rolled composition, ~60 lines

# With nix-effects: adapters are handler transforms
excludeAspectHandler = ref: outerHandlers:
  outerHandlers // {
    resolve-child = { param, state }:
      if aspectPath param == aspectPath ref
      then { resume = tombstone param {}; inherit state; }
      else outerHandlers.resolve-child { inherit param state; };
  };

substituteHandler = ref: replacement: outerHandlers:
  outerHandlers // {
    resolve-child = { param, state }:
      if aspectPath param == aspectPath ref
      then outerHandlers.resolve-child { param = replacement; inherit state; }
      else outerHandlers.resolve-child { inherit param state; };
  };

# oneOfAspects: request structural presence, exclude losers
oneOfAspectsHandler = candidates: outerHandlers:
  let
    presenceSet = fx.run (send "structural-presence" null) outerHandlers {};
    present = filter (c: presenceSet ? ${pathKey (aspectPath c)}) candidates;
    losers = if present == [] then [] else tail present;
  in
  foldl' (h: loser: excludeAspectHandler loser h) outerHandlers losers;
```

Handler combinators compose by overriding specific effect names.
No `filterIncludes` machinery. No `processInclude` / `tag` / `probe`.
Each adapter is a function that takes outer handlers and returns
augmented handlers. Composition is `//`.

### hasAspect as a named effect

```nix
# Today: post-resolution query on frozen structure (safe in (i)/(ii))
host.hasAspect <facter>

# With nix-effects: a named effect request (safe everywhere)
# The computation sends "has-aspect", the handler resumes with bool
hasAspect = ref: send "has-aspect" ref;
```

The handler for `"has-aspect"` is installed at the resolve root with
the precomputed structural set. When ANY computation (class body,
functor body, even includes list) sends `"has-aspect"`, the handler
resumes with the answer. No cycle: the handler has the structural set
BEFORE any conditional includes are evaluated.

This enables the (iii) case we rejected during hasAspect design:

```nix
# With bind.fn, hasAspect is just another request in the context
aspect = fx.bind.fn ({ host, hasAspect }:
  fx.pure {
    includes =
      lib.optional (hasAspect <zfs-root>) <zfs-impermanence>
      ++ lib.optional (hasAspect <btrfs-root>) <btrfs-impermanence>;
  }
);
```

The handler answers `"has-aspect"` via `resume`. The computation
continues from where it suspended. No phased resolution, no
re-evaluation, no fixed-point iteration.

### Conditions for diagnostics

```nix
# Today: strict.nix generates error messages via pattern matching
throw "STRICT MODE: ..."

# With nix-effects: signal a condition, handler decides policy
resolveChild = child:
  if isEmpty child
  then fx.bind (fx.send "condition" {
    type = "empty-aspect";
    key = child.name;
    hint = "Did you mean to register a class?";
  }) (_: fx.pure (tombstone child {}))
  else resolveAspect child;

# Strict handler: abort
strictHandlers.condition = { param, state }:
  { abort = throw "STRICT: ${param.hint}"; };

# Lenient handler: warn and resume
lenientHandlers.condition = { param, state }:
  { resume = builtins.trace "warn: ${param.hint}" null;
    inherit state; };

# Diagnostic handler: accumulate for reporting
diagnosticHandlers.condition = { param, state }:
  { resume = null;
    state = state ++ [ param ]; };
```

Same handler-swap pattern nix-effects uses for type-check error
policy. Strict mode, lenient mode, and diagnostic mode are handler
choices, not separate code paths.

## What stays the same

- **Aspect declaration.** `den.aspects.foo = { ... }` or
  `den.aspects.foo = { host, ... }: { ... }`. Module system handles
  merging. Effects are internal to resolution.
- **Type system.** `aspectType`, `providerType` stay. The submodule
  structure is unchanged. Effects replace what happens AFTER merge.
- **User-facing adapter API.** `meta.adapter = excludeAspect <ref>`
  stays. The adapter functions become handler constructors internally
  but the call-site syntax is unchanged.
- **Entity options.** `host.hasAspect`, `host.mainModule`,
  `host.resolved` all stay.
- **Test suite.** Tests exercise the API, not internals. All pass.

## What changes

| Today | With nix-effects |
|---|---|
| `take.nix` (~30 lines) | Deleted. `bind.fn` + `send` replace it. |
| `can-take.nix` (~25 lines) | Deleted. Named effects don't need arg introspection. |
| `parametric.nix` (~135 lines) | Radically simplified. `applyIncludes`, `applyDeep`, `withOwn`, `carryMeta` all gone. `bind.fn` + scoped handlers replace the entire stack. |
| `statics.nix` (~25 lines) | Folded into a handler that provides `{ class, aspect-chain }`. |
| `aspects/types.nix` coercion block (~15 lines) | `coercedProviderType` gone. `bind.fn` handles all function shapes uniformly. |
| `aspects/resolve.nix` (~60 lines) | Rewritten as `fx.run` + handler composition. |
| `aspects/adapters.nix` (~200 lines) | Adapters become handler combinators. `filterIncludes` gone. Public names stay. |
| `ctx-apply.nix` (~120 lines) | Context stages become scoped handler installation. |

Net: ~600 lines of hand-rolled effects replaced by ~200 lines of
handler composition + nix-effects dependency.

## File plan

| File | Change |
|---|---|
| `nix/lib/take.nix` | Delete |
| `nix/lib/can-take.nix` | Delete |
| `nix/lib/statics.nix` | Delete (fold into handler) |
| `nix/lib/parametric.nix` | Rewrite: ~30 lines. `bind.fn` wrapper + identity helpers. |
| `nix/lib/ctx-apply.nix` | Rewrite: stages as handler installation. |
| `nix/lib/aspects/resolve.nix` | Rewrite: `fx.run` + handler composition. |
| `nix/lib/aspects/adapters.nix` | Rewrite: adapters as handler combinators. Public API preserved. |
| `nix/lib/aspects/types.nix` | Remove `coercedProviderType`. Simplify freeform typing. |
| `nix/lib/aspects/has-aspect.nix` | Simplify: query becomes `send "has-aspect"`. |
| `modules/context/host.nix` | Rewrite: handler installation. |
| `modules/context/user.nix` | Rewrite: handler installation. |
| `nix/strict.nix` | Rewrite: conditions + handler swap. |
| `flake.nix` | Add nix-effects input. |

## Open questions

1. **nix-effects as input vs vendored.** Full lib is substantial.
   Den only needs the effects kernel (freer monad + FTCQueue +
   trampoline + handlers). Vendor just that? Or take the full input
   and benefit from conditions, streams, etc.?

2. **`bind.fn` (nix-effects#12) status.** This feature is critical.
   Without it, users still write `fx.bind (fx.send "host" null)
   (host: ...)` which is verbose. With it, `fx.bind.fn ({ host }:
   ...)` is natural. What's the timeline?

3. **Optimized effect rotation.** Vic notes "vic branch is a bit
   slow because I don't have an optimized effect rotation." The
   FTCQueue in nix-effects IS the optimization. Does the version
   pinned in PR #446 have it? If not, is it in nix-effects main?

4. **Migration path.** Can the nix-effects-based resolve coexist
   with the current resolve during transition? The adapter API
   (`meta.adapter = excludeAspect ref`) is a function value.
   Detecting and wrapping legacy adapters as handler combinators
   should be mechanical.

5. **Type kernel integration.** nix-effects has a full MLTT type
   checker. Could aspects gain typed schemas verified by the kernel?
   `refined "Port" Int (x: x >= 1 && x <= 65535)` for host options.
   Future work, but the infrastructure is there.
