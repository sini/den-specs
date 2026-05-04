# Provides Removal (Post Transition Elimination)

**Date:** 2026-04-27 (amended 2026-05-02)
**Branch:** feat/fx-pipeline
**Status:** Unblocked — compat shims shipped, transition elimination complete
**Prerequisites (all met):**
- Unified policy effects design (2026-04-27) — implemented
- Policy pipeline simplification (2026-04-28) — implemented
- Backwards-compat shims (2026-04-28) — shipped
- Transition elimination (2026-05-01) — shipped (612/616 pass)

## Context

The `provides` structural key has no remaining purpose. All `provides.X` patterns are handled by:
- **`policies.X`** — aspect-included policy functions returning typed effects
- **`provides-compat.nix`** — backwards-compat shim translating `provides.X` to policy effects with deprecation warnings

Removing `provides` eliminates the compat shim, which causes 2 of the 4 remaining test failures:
- `deadbugs-cybolic-routes.test-has-no-dups` — duplicate module from provides-compat + routes interaction
- `user-host-mutual-config.test-host-parametric-unidirectional` — mutual-provider compat not working with current pipeline

## What Changed Since Original Spec

| Component | Original status | Current status |
|---|---|---|
| `transition.nix` | Contained `emitCrossProvider` | **Deleted entirely** (-1011 lines) |
| `emitCrossProvider` (branch-era) | Phase 2 removal target | **Already deleted** with transition.nix |
| DLQ | Interacted with provides classification | **Eliminated** — unknown keys emit as classes |
| Trait system | Interacted with provides routing | **Deleted** for fleet/den.exports reimplementation |
| `emitTransitions` | Called after emitCrossProvideShims | **Replaced** by `installPolicies` using scope.provide |
| `dispatch-policies.nix` | N/A (didn't exist yet) | Created then **deleted** — replaced by inline installPolicies |

## What Gets Deleted

### Pipeline machinery

| Component | File | Lines | Status |
|-----------|------|-------|--------|
| `emitSelfProvide` | `aspect.nix` | ~51 | Delete — replaced by `resolveEntity.includes` |
| `mkPositionalInclude` | `aspect.nix` | ~30 | Delete — only called by `emitSelfProvide` |
| `mkNamedInclude` | `aspect.nix` | ~30 | Delete — only called by `emitSelfProvide` |
| `emitCrossProvideShims` call | `aspect.nix` resolveChildren | ~2 | Delete |
| `substituteChild` | `include.nix` | ~18 | Delete — verify no non-provides callers |

### Compat handler

| Component | File | Notes |
|-----------|------|-------|
| `provides-compat.nix` | `handlers/provides-compat.nix` | 96 lines — entire file |
| import line | `handlers/default.nix` | 1 line |
| `emitCrossProvideShims` import | `aspect.nix` | 1 line |

### Type system

| Component | File | Lines |
|-----------|------|-------|
| `provides` option on `aspectSubmodule` | `types.nix` | ~17 |
| `_` alias (`mkAliasOptionModule ["_"] ["provides"]`) | `types.nix` | ~1 |
| `"provides"` in `structuralKeysSet` | `aspect.nix` | 1 |
| duplicate `"policies"` in `structuralKeysSet` | `aspect.nix` | 1 |

### Support files

| Component | File | Notes |
|-----------|------|-------|
| `resolveWithProvidesFallback` | `den-brackets.nix` | Delete fallback, keep direct resolution |

## Template Migration

All `provides.X` writes become aspect-included policies:

```nix
# Before:
den.aspects.igloo.provides.to-users = { user, ... }: {
  homeManager.programs.direnv.enable = true;
};

# After:
den.aspects.igloo.policies.to-users = { host, user, ... }:
  [ (den.lib.policy.include { homeManager.programs.direnv.enable = true; }) ];
```

```nix
# Before: targeted cross-provide
den.aspects.igloo.provides.alice = {
  homeManager.programs.vim.enable = true;
};

# After:
den.aspects.igloo.policies.to-alice = { host, user, ... }:
  lib.optional (user.name == "alice")
    (den.lib.policy.include { homeManager.programs.vim.enable = true; });
```

### `den.provides.*` factory namespace

The `den.provides` top-level namespace (`den.provides.forward`, `den.provides.define-user`, etc.) is the **factory registry** — NOT the aspect-level `provides` structural key. It is unaffected.

## `resolveChildren` Simplification

Current (aspect.nix):
```
emitSelfProvide → emitCrossProvideShims → emitTraitSchemas → emitAspectPolicies → emitIncludes → installPolicies
```

After:
```
emitTraitSchemas → emitAspectPolicies → emitIncludes → installPolicies
```

Two phases removed. `emitSelfProvide` is dead because `resolveEntity.includes` already handles the self-provide pattern. `emitCrossProvideShims` is dead because the compat layer is being removed.

## Preserved Insights (from superseded forward-route-unification spec)

### `adaptArgs` Semantics

Forward `adaptArgs` uses `lib.evalModules { specialArgs = adapted; }`. Route `adaptArgs` uses structural nesting. These have different semantics:
- `specialArgs` bypass option checks — args are available without declaration
- `_module.args` goes through the option system — may conflict with NixOS's own bindings (e.g., `pkgs`)

For any adapter forwards that reference `adaptArgs`, the structural nesting must use `_module.specialArgs` (not `_module.args`) to match the old evaluation semantics. This is already handled in `wrapRouteModules` for Tier 1 routes.

### `fromAspect` Pattern Mapping

The old `fromAspect` field on forwards is eliminated by the unified pipeline. The 4 patterns map to:

| Old pattern | New mechanism |
|---|---|
| `lib.head aspect-chain` | Default — policy fires in current entity's scope |
| `host.aspect` (same pipeline) | Policy fires in host's scope |
| `den.lib.resolveEntity "user" ctx` | Policy fires during user resolution — scope matches |
| `den.lib.parametric.fixedTo { host } aspect` | Policy fires in appropriate scope |

### Guard Semantics

Forward guards are **config transformers** (`lib.optionalAttrs cond`), not boolean gates (`lib.mkIf cond`). Route guards use `lib.mkIf`. These are semantically different:
- `lib.optionalAttrs false { ... }` → `{}` (attrs removed entirely)
- `lib.mkIf false { ... }` → attrs present but conditionally disabled

For the remaining adapter forwards, verify that guard conversion preserves the intended semantics. Most cases work with `lib.mkIf` but edge cases with option presence detection may differ.

## Execution Steps

```
Step 1:  Migrate provides.X patterns in templates → policies.X (~35 files)
Step 2:  Delete provides-compat.nix + emitCrossProvideShims call
Step 3:  Delete emitSelfProvide + mkPositionalInclude + mkNamedInclude
Step 4:  Simplify resolveChildren (remove selfProvide + compat phases)
Step 5:  Remove provides option + _ alias from types.nix
Step 6:  Remove "provides" (and duplicate "policies") from structuralKeysSet
Step 7:  Clean up den-brackets.nix provides fallback
Step 8:  Verify substituteChild in include.nix — delete if unused outside provides
Step 9:  Run full CI — expect 2 of 4 remaining failures to resolve
Step 10: Delete superseded specs (forward-route-unification, forward-simplification-unified-dispatch, dispatch-dedup-investigation)
```

## Impact

| Metric | Value |
|--------|-------|
| Lines deleted | ~210 (pipeline: selfProvide + compat + types) |
| Files deleted | 1 (provides-compat.nix) |
| Files modified | ~5 (aspect.nix, include.nix, types.nix, den-brackets.nix, handlers/default.nix) |
| Template files migrated | ~35 (provides.X → policies.X) |
| Test failures resolved | 2 of 4 remaining (provides-compat interaction bugs) |
| Concepts removed | `provides` structural key, self-provide, cross-provide shims, mutual-provider compat |
