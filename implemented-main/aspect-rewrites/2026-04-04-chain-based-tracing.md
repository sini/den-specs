# Chain-Based Tracing for Den Aspect Resolution

**Date:** 2026-04-04
**Status:** Implemented
**Branch:** `feat/aspect-rewrites`

## Problem

The original trace implementation wrapped every aspect with a trace functor
during resolution. This added call depth at every node, causing stack
overflow when forwards invoke `resolve'` with `trace = true` on deep
aspect trees.

## Solution

Build trace entries directly in `resolveWith` instead of functor wrapping.
When `opts.trace` is true, each aspect produces a trace entry with
`{ name, class, decision, depth, chain }` plus optional `provider`,
`placedBy`, `replacedBy` fields. Zero additional call depth.

### Key changes

- `resolveWith` builds trace entries inline when `doTrace` is true
- `trace` transform function removed from `transforms.nix` and public API
- `normalizeResult` and `compose` simplified — trace accumulation removed
  from the transform protocol. `compose` returns raw value or null.
- Transforms are pure aspect→aspect functions with no trace plumbing

### What didn't change

- `opts.trace` on `resolve'` still controls trace collection
- Forward traces still use `__forwardTrace` mechanism
- Forward `resolve'` does not enable `trace = true` — the
  `lib.isFunction` overflow in `applyStatics`/`can-take.nix` on
  deep parametric chains is a separate issue in the statics pipeline

## Files Changed

| File | Change |
|------|--------|
| `nix/lib/aspects/resolve.nix` | Direct trace entry building in resolveWith |
| `nix/lib/aspects/transforms.nix` | Removed `trace` function, simplified `compose` return type |
| `nix/lib/aspects/default.nix` | Removed `trace` from public transforms export |
