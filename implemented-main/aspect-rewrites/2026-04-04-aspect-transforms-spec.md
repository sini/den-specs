# Aspect Transforms for Den

**Date:** 2026-04-04
**Branch:** `feat/aspect-rewrites`
**Status:** Implemented, 276/276 CI tests passing

## Summary

Den's aspect resolution pipeline supports composable transforms that
exclude, substitute, or trace aspects during resolution. Transforms
are declared on aspects or host entities and propagate into nested
include subtrees and forwarded class content.

## Public API

### `den.lib.aspects.transforms`

Three transform constructors, exported from `nix/lib/aspects/default.nix`:

```nix
den.lib.aspects.transforms.exclude    :: [ref] -> transform
den.lib.aspects.transforms.substitute :: ref -> aspect -> transform
den.lib.aspects.transforms.trace      :: transform -> transform
```

Where `ref` is either a string name or an aspect reference with `.name`
(e.g., `den.aspects.foo` or `<foo>` via angle brackets).

A transform is a function `{ provided, class, chain, ... } -> provided | null | { result; trace; }`.
Returning `null` prunes the aspect and its subtree. Returning an attrset
(or `{ result; trace; }`) includes it (possibly modified). The `trace`
functor wraps another transform and emits resolution trace entries.

Internal helpers (`normalizeResult`, `id`, `compose`, `toName`) are not exported.

### `den.lib.aspects.resolve` (unchanged)

```nix
resolve :: class -> aspect -> { imports; }
```

Backwards compatible. Same signature and return type as before.

### `den.lib.aspects.resolve'` (new)

```nix
resolve' :: class -> opts -> aspect -> { module; trace; }
```

Power API for programmatic transforms and trace.

Opts:
```nix
{
  transforms = [];   # list of transform functions
  trace = false;     # enable trace entry collection
}
```

Return:
```nix
{
  module = { imports = [...]; };  # valid Nix module
  trace = [...];                  # trace entries (empty unless traced)
}
```

### Aspect options

```nix
den.aspects.<name> = {
  excludes = [ "name" ];           # strings or aspect refs
  excludes = with den.aspects; [ foo ];  # aspect refs via with
  transforms = [ transform-fn ];   # transform functions for subtree
  includes = [ ... ];              # filtered by excludes via apply
};
```

`excludes` accepts strings and aspect references (attrsets with `.name`).
References are coerced to strings via a lazy `apply = map toAspectName`.
Direct includes are filtered by the `includes` option's apply function.
Deep propagation flows through resolve's `rawExcludes` and ctxApply's
`collectExcludes`.

`transforms` are `listOf raw` functions. They propagate into the resolve
pipeline via ctxApply's `collectAspectTransforms` and resolve's
`rawTransforms`.

### Entity options

```nix
den.hosts.<system>.<name>.excludes = [ "name" ];
den.hosts.<system>.<name>.excludes = with den.aspects; [ foo ];
den.homes.<system>.<name>.excludes = [ "name" ];
# (also on userType)
```

Entity excludes accept the same forms as aspect excludes (strings or
aspect refs). They are collected by ctxApply's `collectExcludes` from
both the entity value (`v.excludes`) and the entity's aspect
(`den.aspects.${v.aspect}.excludes`). Entity excludes do NOT flow
through `definition.nix` — this avoids circular evaluation when
excludes reference other aspects.

## Architecture

### Resolution flow

```
host declaration
  -> ctxApply
    -> collectExcludes (entity + aspect excludes)
    -> collectAspectTransforms (aspect transforms)
    -> traverse (checks excluded set on context transitions)
    -> assembleIncludes (deduplicates by key)
    -> result: { excludes; transforms; includes; }
  -> resolve/forward (walks include tree with transforms)
    -> at each node: apply inherited transforms, check for pruning
    -> rawExcludes + rawTransforms captured before functor evaluation
    -> childOpts accumulates excludeTransforms + rawTransforms + inherited
    -> recurse into includes with childOpts
```

### Identity preservation

Aspect names survive functor evaluation through two mechanisms:

- `parametric.nix`: `withIdentity` carries `self.name` into functor results
  for `applyIncludes`, `deep`, and `withOwn`.
- `take.nix`: `carryAttrs` preserves `fn.name` and `fn.excludes` through
  `take.atLeast` / `take.exactly` call boundaries.

### perHost / perUser / perHome

These return `{ includes = [...]; }` (plain attrsets) instead of raw
lambdas. The module system routes them through `aspectSubmodule`, giving
them proper `name` identity. The context gating logic lives inside the
includes list.

**Breaking change:** code that relied on `den.lib.perHost x` being a
callable function will see a different type.

### Auto-generated aspects

`modules/aspects/definition.nix` creates auto-generated aspects as plain
attrsets (not `parametric` wrapped). The submodule's `defaultFunctor`
provides parametric behavior. This ensures user definitions merge cleanly
with auto-generated ones.

### Forward integration

`nix/lib/forward.nix` reads excludes and transforms from the aspect being
forwarded and passes them to `resolve'`. Forwarded class content respects
the same excludes and transforms as the parent resolution.

Forward trace entries are collected via `__forwardTrace` on forwarded
results but currently disabled (`trace = false` in forward's `resolve'`)
due to stack overflow from functor-based tracing. See the chain-based
tracing spec for the proposed fix.

## Transform composition

Transforms compose via Kleisli composition in the Maybe monad:

```nix
compose :: [transform] -> transform
```

Short-circuits on `null` (pruning). Threads trace entries through the
chain. Used internally by resolve when `opts.transforms` has multiple
entries.

## Trace entries

When `trace = true` (on `resolve'` opts), each visited aspect produces:

```nix
{
  name = "aspect-name";      # or "<anon>" for unnamed
  class = "nixos";           # which class this resolution is for
  depth = 2;                 # depth in resolution tree
  chain = ["parent" "..."];  # ancestor names
  decision = "included";     # "included" | "pruned" | "replaced"
  replacedBy = "new-name";   # only when decision == "replaced"
}
```

## Files changed

| File | Change |
|------|--------|
| `nix/lib/aspects/transforms.nix` | New. Transform primitives with toName normalizer. |
| `nix/lib/aspects/resolve.nix` | Rewritten. resolveWith + resolve/resolve'. class in context. |
| `nix/lib/aspects/default.nix` | Exports transforms, resolve, resolve'. |
| `nix/lib/aspects/types.nix` | excludes (listOf raw + apply), transforms options; includes apply filter; toAspectName. |
| `nix/lib/ctx-apply.nix` | collectExcludes (entity+aspect), collectAspectTransforms, excluded set in traverse. |
| `nix/lib/parametric.nix` | withIdentity for name propagation. |
| `nix/lib/take.nix` | carryAttrs for name/excludes propagation. |
| `nix/lib/types.nix` | excludes option (listOf raw + toAspectName apply) on hostType, userType, homeType. |
| `nix/lib/forward.nix` | Reads excludes/transforms from asp, passes to resolve'. __forwardTrace. |
| `modules/aspects/definition.nix` | Plain attrsets (no parametric, no entityExcludes). |
| `modules/context/perHost-perUser.nix` | perHost/perUser/perHome return plain attrsets. |

## Test coverage (276 total)

- `templates/ci/modules/features/transforms.nix` — 23 unit tests (primitives + aspect ref forms)
- `templates/ci/modules/features/excludes.nix` — 13 integration tests (entity, aspect, nested, diamond, perHost, siblings, aspect refs, forward propagation)
- `templates/ci/modules/features/resolve-prime.nix` — 7 integration tests (exclude, substitute, trace, compose)
- `templates/transform-trace-demo/` — 6 hosts demonstrating all features with Mermaid trace output

## Known limitations

- **Forward trace disabled**: `trace=true` in forward's `resolve'` causes stack overflow. Spec for chain-based tracing addresses this.
- **Angle brackets in excludes**: causes circular eval (`config.den.aspects` access during module eval). Use `with den.aspects;` instead.
- **Provider cascade**: excluding a parent aspect doesn't auto-exclude its independently-included providers.

## Future work

- Chain-based tracing (see `2026-04-04-chain-based-tracing.md`)
- `replaces` sugar for mutual exclusion between aspects
