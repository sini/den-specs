# fixedTo __ctx Propagation — Spec

**Date:** 2026-04-17
**Status:** In Progress
**Goal:** Fix the 141 test failures from the legacy removal by properly propagating context values through the fx pipeline.

## Problem

`ctxApply` pre-processes `into` transitions and calls `fixedTo item.ctx stripped` for each. The current `fixedTo` produces a `withIdentity` wrapper with class keys extracted into a separate `owned` child include. This causes two problems:

1. **Context gap:** Nested parametric includes (inside fixedTo results) need `host`/`user` values but the pipeline's `constantHandler` doesn't have them. The `applyCtxToIncludes` helper only resolves one level — sub-includes that need context hit "unhandled effect."

2. **Duplicate modules:** The `owned` child emits class keys via `compileStatic`, but those same keys may also be emitted through other resolution paths, causing "option defined multiple times."

## Design

### `fixedTo` output shape

`fixedTo attrs aspect` produces the aspect with class keys INLINE (not as a separate child) and tagged with `__ctx`:

```nix
{
  name = aspect.name or "<anon>";
  meta = { adapter, handleWith, excludes, provider };  # from withIdentity
  nixos = aspect.nixos;       # class keys inline
  darwin = aspect.darwin;     # class keys inline
  __ctx = attrs;              # context values for pipeline
  includes = applyCtxToIncludes take.atLeast attrs (aspect.includes or []);
}
```

No `owned` child. `compileStatic` emits class keys directly from the fixedTo result AND processes includes. No duplication.

`__ctx` is in `structuralKeys` (both in `aspect.nix` and `parametric.nix`) so it's not emitted as a class key.

### Pipeline scoped handler

In `handlers/include.nix`, when `keepChild` processes a child with `__ctx`, it wraps the `aspectToEffect` call in `scope.stateful` with a `constantHandler` providing the `__ctx` values:

```nix
keepChild = child:
  let
    comp = aspectToEffect child;
    hasCtx = builtins.isAttrs child && child ? __ctx && child.__ctx != {};
  in
  if hasCtx then
    fx.bind (fx.effects.scope.stateful (handlers.constantHandler child.__ctx) comp) (
      resolved: fx.pure [resolved]
    )
  else
    fx.bind comp (resolved: fx.pure [resolved]);
```

This installs a scoped `constantHandler` that provides `host`/`user`/etc. to ALL nested parametric includes within the fixedTo subtree. The scoped handler only catches effects matching its keys — `emit-class`, `chain-push`, `emit-include`, etc. pass through to outer handlers.

### Root `__ctx` propagation

`fxResolveTree` extracts `__ctx` from the resolved aspect and passes it as `ctx` to the pipeline:

```nix
ctx = resolved.__ctx or {};
```

This provides the top-level context values (e.g., `{ host = config }`) to the root `constantHandler`.

### `structuralKeys` update

Add `__ctx` to `structuralKeys` in `aspect.nix` so `compileStatic` doesn't emit it as a class key.

## Files to modify

- `nix/lib/parametric.nix` — remove `owned`, put class keys inline in `bindCtx`
- `nix/lib/aspects/fx/aspect.nix` — add `__ctx` to `structuralKeys`
- `nix/lib/aspects/fx/handlers/include.nix` — scoped `constantHandler` in `keepChild`
- `nix/lib/aspects/default.nix` — extract `__ctx` in `fxResolveTree` (already done)

## Why this furthers the capabilities spec

The `__ctx` tag is a proto-capability scope marker. The scoped handler effectively says "within this subtree, these capabilities (host, user) are active." This is the same concept as the capabilities model's activation scoping — just using existing handler infrastructure instead of a new `emit-for-capability` effect.

## Verification

```bash
nix develop -c just ci ""
```

Target: all tests passing (452/452 or close — some trace tests may need adjustment).
