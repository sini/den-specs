# Effects-Based Resolution Prototype

**Date:** 2026-04-13
**Status:** Approved design
**Branch:** feat/fx-resolution (TBD)
**Location:** `nix/lib/aspects/fx/`

## Goal

Build a parallel effects-based resolution pipeline using nix-effects that proves the core insight: `bind.fn` turns every aspect into an effectful computation where missing context args become effect requests, and handlers supply them. This eliminates `canTake`/`functionArgs`/coercion entirely.

## Motivation

PRs #408, #413, #419, #423, #426, #429, #437 all stem from manual context threading in the parametric pipeline. Each bug was a case the `canTake`/`functionArgs`/`coercion` machinery got wrong. Named effects with `bind.fn` eliminate this entire class of bugs.

## Strategy

**Prototype (this spec):** Build in `nix/lib/aspects/fx/`, isolated from the real pipeline. Prove correctness against existing test fixtures.

**Integration (next):** Wire prototype into den as `resolve.withEffects`, prove equivalence with full test suite, swap.

**Validation:** Reimplement `onlyIf`, recursive freeform children, and `hasAspect`-in-includes as exercises of the new effects API.

## Dependencies

- `nix-effects` as a flake input (or vendored). Needs `bind.fn` (available in v0.5.0+).
- PR #449 class registration (for recursive freeform, not needed for this prototype).

## Architecture

### Core translation: aspect → computation

An aspect function like `{ host }: { services.nginx.enable = true; }` becomes:

```nix
fx.bind.fn {} aspectFn
```

`bind.fn` inspects `functionArgs` of `aspectFn`, finds `host` is not in the provided attrs `{}`, and emits `fx.send "host" false`. The `false` is the default-ness boolean from `lib.functionArgs` — `false` means required, `true` means has a default. A handler intercepts this effect and resumes with the actual host value from the parametric context.

Static aspects (no function args, or only `class`/`aspect-chain`) pass through `fx.pure` or are handled by the static handler.

Factory-function aspects (`den.aspects.facter`) have empty `functionArgs` because they take a bare positional arg (like `reportPath`), not a destructured context attrset. `bind.fn` cannot handle these — it would call `f {}` since there are no declared args. Factory functions are detected by `lib.functionArgs f == {}` and wrapped via a dedicated path:

```nix
wrapAspect = ctx: aspect:
  if !lib.isFunction aspect then fx.pure aspect
  else if lib.functionArgs aspect == {} then fx.pure (aspect ctx)  # factory: pass full context
  else fx.bind.fn {} aspect;  # normal: bind.fn sends per-arg effects
```

### Handlers

Each resolution stage is a handler set.

**Parametric handler:**
```nix
# Built from ctx alone. bind.fn handles the intersection —
# it only sends effects for args the function actually declares.
# Unknown effect names (not in ctx) fall through to fallback.
parametricHandler = ctx:
  builtins.mapAttrs (name: value: { param, state }: {
    resume = value;
    inherit state;
  }) ctx;
```

No `functionArgs` intersection needed at the handler level. `bind.fn` only sends effects for args the function declares. If an effect name isn't in `ctx`, it simply has no handler and falls through.

**Static handler:**
```nix
staticHandler = { class, aspect-chain }:
  {
    "class" = { param, state }: { resume = class; inherit state; };
    "aspect-chain" = { param, state }: { resume = aspect-chain; inherit state; };
  };
```

**Fallback / error handler:**
Unhandled effect names indicate a missing context arg. The handler stack uses a two-layer topology:

1. Inner: `fx.rotate` with parametric + static handlers. Known args are handled; unknown args are re-suspended.
2. Outer: `fx.handle` with a catch-all error handler that intercepts the re-suspended unknowns and produces diagnostic errors (aspect name, missing arg name, available context).

```nix
# Inner layer: handle known context, rotate unknowns outward
inner = fx.rotate {
  handlers = parametricHandler ctx // staticHandler { inherit class aspect-chain; };
  state = {};
} comp;

# Outer layer: catch unknowns, produce diagnostics
result = fx.handle {
  handlers."_fallback" = { param, state }: throw
    "aspect '${aspectName}' requires '${effectName}' but context only provides: ${toString (attrNames ctx)}";
  state = {};
} inner;
```

Note: nix-effects may not support a wildcard/fallback handler. If not, the alternative is to merge an explicit handler for every arg name NOT in ctx that maps to an error. This is determined during implementation.

### Identity envelope

Every resolved aspect needs wrapping in the identity envelope `{ name; meta = { adapter; provider; }; includes; <owned-config> }`. In the current pipeline, `withIdentity` (parametric.nix) constructs this.

In the effects pipeline, the envelope is constructed post-resolution:

```nix
wrapIdentity = { aspect, class, aspect-chain, resolved }:
  let
    owned = removeAttrs resolved [ "includes" "__functor" "__functionArgs" "name" "meta" ];
    includes = resolved.includes or [];
  in {
    inherit (aspect) name;
    meta = {
      adapter = (aspect.meta or {}).adapter or null;
      provider = (aspect.meta or {}).provider or [];
    };
    inherit includes;
  } // owned;
```

This is applied after `fx.handle` returns the raw aspect body. The envelope is structural, not effectful — it doesn't need to be inside the effect computation.

### Owned config stripping

The current `statics.nix` strips `includes`, `__functor`, `__functionArgs` from aspect bodies to isolate "owned" config. The effects pipeline does the same in `wrapIdentity` above — owned config is everything in the resolved body minus the structural keys. This replaces `statics.owned`.

### Deep resolution (replacing `applyDeep`)

`applyDeep` in `parametric.nix` is the most complex part of the current pipeline. It handles:

1. Provider sub-aspects whose result contains function includes needing further context
2. Bare results (`{ includes = [...]; }` without owned config) vs full results
3. Recursive descent through nested include trees
4. The `canTake.upTo` gating that decides whether a sub-aspect consumes context

In the effects pipeline, deep resolution becomes **recursive effect interpretation**:

```nix
resolveDeep = { ctx, class, aspect-chain }:
  let
    resolveOne = aspect:
      let
        # Translate aspect to computation (factory vs normal)
        comp = wrapAspect ctx aspect;

        # Two-layer interpretation: rotate known, catch unknowns
        inner = fx.rotate {
          handlers = parametricHandler ctx // staticHandler { inherit class aspect-chain; };
          state = {};
        } comp;
        result = fx.handle {
          handlers = errorHandler { inherit ctx; aspectName = aspect.name or "<anonymous>"; };
          state = {};
        } inner;
        body = result.value;

        # Recursively resolve any function includes in the result
        resolvedIncludes = map (child:
          if lib.isFunction child
          then resolveOne child
          else child  # static attrset or already-resolved
        ) (body.includes or []);
      in
        wrapIdentity {
          inherit aspect class aspect-chain;
          resolved = body // { includes = resolvedIncludes; };
        };
  in resolveOne;
```

Note: the current pipeline's `isCtxStatic` branching (parametric.withOwn) is now implicit. When context is static-only (class, aspect-chain), `bind.fn` only sends those two effects, which the static handler resolves. When context is parametric, it sends parametric arg effects too. The branching is determined by what the function declares, not by an explicit `if`.

The key insight: **`bind.fn` eliminates the need for `canTake.upTo` gating.** In the current pipeline, `applyDeep` must check whether a sub-aspect can consume the available context before applying it. With effects, every sub-aspect declares what it needs via `bind.fn`, and the handler either provides it or the fallback catches it. There is no "partial application" or "which args were consumed" tracking.

The `isBareResult` check disappears because the effects pipeline treats every function include uniformly — resolve it, wrap its identity, recurse into its includes.

### Deduplication tracking

The current pipeline tracks "seen" aspects in `assembleIncludes` (ctx-apply.nix) to prevent duplicate resolution. The effects prototype uses handler state for this:

```nix
# State tracks seen aspect paths to prevent duplicate resolution
deduplicationHandler = {
  "resolve-include" = { param, state }:
    let key = pathKey (aspectPath param); in
    if state ? ${key}
    then { resume = null; inherit state; }  # skip duplicate
    else { resume = param; state = state // { ${key} = true; }; };
};
```

This is a stretch goal for the prototype — basic resolution works without dedup, and `ctx-apply.nix` integration is Phase B. But the pattern is documented here to validate that handler state supports it.

### Adapters (stretch goal)

Adapters are a stretch goal for the prototype phase. The core resolution must work first.

When implemented, adapters become handler combinators. The current `filterIncludes` walks the include tree and applies `meta.adapter`. In the effects version, each include resolution emits an "include" effect that an adapter handler can intercept:

```nix
excludeHandler = excludeRef: innerHandlers:
  innerHandlers // {
    "include" = { param, state }:
      if aspectPath param == aspectPath excludeRef
      then { resume = tombstone param {}; inherit state; }
      else innerHandlers."include" { inherit param state; };
  };
```

For layered adapter handling (some effects intercepted, others forwarded), `fx.rotate` is the right tool — it handles known effects and re-suspends unknown ones for outer handlers.

### Meta preservation

`meta.provider` and other metadata survive the effects pipeline because:

1. `bind.fn` operates on the function's return value — if the aspect returns an attrset with `meta`, it's in the result
2. `wrapIdentity` copies `aspect.meta` onto the envelope
3. No intermediate functor evaluation can drop meta (unlike `parametric.nix` where `carryMeta` was needed as a workaround)

### `parametric.deep` / `parametric.fixedTo` translation

The current pipeline has several composition patterns in `parametric.nix`:

- `parametric.deep(functor)` — applies functor recursively through includes + statics
- `parametric.deepParametrics(functor)` — applies only to parametric includes
- `parametric.fixedTo(ctx)` — creates fixed-context variants (`.atLeast`, `.exactly`, `.upTo`)

These all exist because the current pipeline must manually choose HOW to apply context (atLeast/exactly/upTo) at each level. With effects, this entire zoo disappears. There is only one resolution path: `bind.fn` sends what the function needs, handlers provide it. The "how much context to apply" question doesn't exist.

## File plan

```
nix/lib/aspects/fx/
  default.nix    — re-exports all public members
  aspect.nix     — aspect → computation translation (bind.fn wrapper, factory detection)
  handlers.nix   — parametricHandler, staticHandler, fallback, dedup handler
  resolve.nix    — resolveOne, resolveDeep, wrapIdentity, includes recursion
  adapters.nix   — adapter → handler combinator translation (stretch goal)
```

## Test plan

Tests in `templates/ci/modules/features/fx/` organized in two tiers.

### Core resolution tests

1. **Static resolution** — aspect with no function args resolves to its body
2. **Single parametric arg** — `{ host }: ...` receives host from handler
3. **Multi parametric args** — `{ host, user }: ...` receives both
4. **Optional args** — `{ host, user ? null }: ...` with `param = true` (has default), handler still provides value
5. **Nested includes** — parent and child aspects resolved through same handler stack
6. **Mixed static/parametric includes** — static children pass through, parametric children get context
7. **Identity envelope** — resolved aspect has `name`, `meta`, `includes`, owned config separated
8. **Owned stripping** — `__functor`, `__functionArgs`, `includes` not in owned config
9. **Meta preservation** — `meta.provider` survives resolution through deep recursion
10. **Missing required arg** — produces diagnostic error with aspect name and missing arg
11. **Factory-function aspects** — empty `functionArgs`, receive full context via dedicated path
12. **Deep resolution** — provider sub-aspect's includes contain functions needing further context

### Targeted regression reproductions

13. **#413/#423 pattern** — provider sub-aspect with parametric includes (the `applyDeep` case). Previously: context dropped during recursive descent. Effects version: each level independently sends what it needs.
14. **#426 pattern** — static sub-aspect inside parametric parent. Previously: `applyDeep` called `takeFn` unconditionally, dropping static subs. Effects version: static subs have no parametric args, `bind.fn` sends nothing, body passes through.
15. **#437 pattern** — factory-function aspect wrapped by coercion. Previously: coercion applied to non-context functions. Effects version: factory detected by empty `functionArgs`, no coercion path exists.
16. **Meta carryover** — parametric functor result preserves `meta.provider`. Previously: intermediate evaluation dropped it. Effects version: no intermediate evaluation.

## What the prototype skips

- `ctx-apply.nix` — integration phase scope. Beyond namespace traversal, ctx-apply handles: cross-provider resolution (`getCrossProvider`), first-seen vs repeat-seen behavior differences (`fixedTo` vs `atLeast`), the `into` callback pattern for nested context trees, and `assembleIncludes` deduplication fold. These are non-trivial behaviors the integration spec must address.
- Performance optimization — correctness first
- Entity wiring (`den.schema.conf`) — library functions only
- Migration shims — no old/new coexistence during prototype
- `onlyIf`, `hasAspect`-in-includes — post-integration exercises
- Deduplication tracking — documented pattern, implemented during integration

## Open questions

1. **nix-effects as flake input vs vendored.** Flake input is cleaner but adds a dependency. The adjacent clone at `../nix-effects` suggests flake input with override for dev.
2. **Handler state shape.** Core resolution uses `state = {}` since parametric/static handlers are stateless. Adapter/dedup handlers need state — compose via `fx.adapt` with lens into parent state.
3. **`fx.effects.reader` for context.** Parametric context is read-only, textbook reader effect. Worth evaluating whether `fx.effects.reader` with `asks` is cleaner than raw handlers. Trade-off: reader is idiomatic but adds a layer; raw handlers are explicit and match the prototype's educational purpose.
4. **Interaction with `den.lib.strict` (#428).** Strict mode disables freeform types. Orthogonal to the prototype.
