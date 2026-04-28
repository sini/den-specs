# Context as Data: Remove scope.run from Transitions

**Date:** 2026-04-18
**Status:** Active
**Goal:** Fix scoped state isolation (C1) by propagating context as data (`__ctx`) instead of handler scoping

## Problem

`transition.nix` uses `scope.run` with `includeHandler` in the scope so that parametric children can resolve their args via the scoped `constantHandler`. But this isolates state — `emit-class`, `chain-push/pop`, `register-constraint`, etc. handled inside the scope don't share state with the root pipeline.

## Design

Context flows as **data on nodes** (`__ctx`), not as ambient handler state. This matches the legacy pipeline's approach and aligns with the capabilities direction.

### Key changes

1. **transition.nix** — Remove `scope.run` entirely. Tag target aspects with `__ctx = scopedCtx` and call `aspectToEffect` directly. Remove `mkScopedTransitionHandler`.

2. **aspect.nix / aspectToEffect** — When an aspect has `__ctx`, scope ONLY the `bind.fn` call:
   ```nix
   scope.run { handlers = constantHandler __ctx; } (fx.bind.fn {} fn)
   ```
   This mini-scope handles context args (`host`, `user`), lets `class`/`aspect-chain` rotate to root. Resume is a plain value — no state isolation.

3. **aspect.nix / resolveChildren** — Propagate `__ctx` to children via `emit-include` payload (`parentCtx` field).

4. **include.nix / includeHandler** — After `wrapChild`, merge `parentCtx` from the emit-include payload into the wrapped child's `__ctx`.

5. **include.nix / keepChild** — Only `probe-arg` for args NOT already in `child.__ctx`. Args in `__ctx` are known-available.

### Why this works

- `emit-class` → root `classCollectorHandler` → shared state ✓
- `chain-push/pop` → root `chainHandler` → shared state ✓
- `register-constraint` / `check-constraint` → root handlers → shared state ✓
- `bind.fn` args → mini-scoped `constantHandler __ctx` → correct context ✓
- `emit-include` → root `includeHandler` → shared state ✓

### What gets removed

- `scope.run` in `resolveContextValue` and `emitCrossProvider`
- `mkScopedTransitionHandler` (nested transitions just tag `__ctx` recursively)
- `includeHandler` import from `transition.nix`
