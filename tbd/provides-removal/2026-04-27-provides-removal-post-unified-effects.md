# Provides Removal (Post Unified Policy Effects)

**Date:** 2026-04-27 (amended 2026-04-28)
**Branch:** feat/fx-pipeline
**Status:** Draft — blocked on compat layer
**Prerequisites:**
- Unified policy effects design (2026-04-27) — implemented
- Policy pipeline simplification (2026-04-28) — implemented
- **Backwards-compat shims (2026-04-28-provides-compat-shims-design.md) — MUST ship first**

## Context

With unified policy effects and pipeline simplification both landed, the `provides` structural key has no remaining purpose. All `provides.X` patterns are replaced by:
- **`policies.X`** — aspect-included policy functions returning typed effects (`policy.include`, `policy.resolve`, `policy.exclude`)
- **`mutual-provider`** — eliminated; bidirectional config expressed as aspect-included policies

This spec covers deleting `provides` from the pipeline, type system, and all consumer code. It also covers collateral simplifications that become possible once `provides` and the old policy dispatch model are gone.

> **IMPORTANT:** No removal work may proceed until the backwards-compat shims spec
> (`2026-04-28-provides-compat-shims-design.md`) is implemented and shipped. Users
> migrating from `main` rely on `provides.X` cross-provide patterns, `den._.mutual-provider`,
> and other surfaces that this spec targets for deletion. The compat layer translates
> these legacy patterns to the new policy mechanism with deprecation warnings, giving
> users a migration path. Removal happens in a future release after the compat layer
> has been in place.

## Current State (post pipeline simplification)

The following items from the original spec are **already deleted** by pipeline simplification work:

| Component | Deleted in |
|---|---|
| `policy-dispatch.nix` (entire file) | commit 30ba636e |
| `mutual-provider.nix` (entire file) | commit 12fed6aa |
| `compilePolicyHandlers` | commit 30ba636e |
| `policyEffectNamesFor` | commit 30ba636e |
| `emitCrossProvider` (main-era) | commit 12fed6aa (mutual-provider deletion) |
| `collectPolicyHandlers` | commit 72b34735 |
| `activePoliciesFor` / `ctxSatisfies` dispatch | commit 29156fe0 / a2a7c933 |
| All `from`/`to`/`__functor`/`_core` on policies | commits through de80246b |

The following remain and are **NOT reduced** from the original spec:
- `synthesize-policies.nix` — trimmed to 29 lines, exports only `resolveArgsSatisfied` (still live)
- `policy-types.nix` — trimmed to 9 lines, exports only `policyFnArgs` (still live)

**Note:** A branch-era `emitCrossProvider` exists in `transition.nix` (commit `ed1301a4`). This is NOT the same as the main-era cross-provide mechanism — it handles `provides.${entityKind}` patterns during transition resolution and is part of the provide-to policy implementation. It is a Phase 2 removal target alongside `emitSelfProvide`.

## What Gets Deleted (remaining)

> Items already deleted by pipeline simplification are listed in "Current State" above.
> This section covers only what remains after compat shims have been in place.

### Pipeline machinery (~120 lines)

| Component | File | Lines | Why dead |
|-----------|------|-------|----------|
| `emitSelfProvide` | `aspect.nix` | ~51 | `provides.${self.name}` pattern replaced by `resolveEntity.includes` |
| `mkPositionalInclude` | `aspect.nix` | ~30 | Only called by `emitSelfProvide` |
| `mkNamedInclude` | `aspect.nix` | ~30 | Only called by `emitSelfProvide` |
| `substituteChild` | `include.nix` | ~18 | `substituteAspect` replaced by `policy.exclude` + `policy.include` |

### Compat handler removal

| Component | File | Notes |
|-----------|------|-------|
| `provides-compat.nix` | `handlers/provides-compat.nix` | Cross-provide→policy translation shim |
| `emitCrossProvideShims` call | `aspect.nix` resolveChildren | Single call site |
| `mutual-provider-shim.nix` | `modules/compat/` | No-op aspect shim |

### Branch-era provides machinery

| Component | File | Lines | Why dead |
|-----------|------|-------|----------|
| `emitCrossProvider` | `transition.nix` | ~43 | `provides.${entityKind}` routing; dead once provides structural key removed |

### Utility files to relocate (not delete)

| File | Current | After |
|------|---------|-------|
| `synthesize-policies.nix` | 29 lines, `resolveArgsSatisfied` | Inline into transition.nix or keep as-is |
| `policy-types.nix` | 9 lines, `policyFnArgs` | Inline into transition.nix or keep as-is |

### Type system cleanup (~20 lines)

| Component | File | Lines |
|-----------|------|-------|
| `provides` option on `aspectSubmodule` | `types.nix` | ~17 |
| `_` alias (`mkAliasOptionModule ["_"] ["provides"]`) | `types.nix` | ~1 |
| `"provides"` in `structuralKeysSet` | `aspect.nix` | 1 |
| duplicate `"policies"` in `structuralKeysSet` | `aspect.nix` | 1 |

### Simplifications

| Component | File | Current | After |
|-----------|------|---------|-------|
| `resolveChildren` | `aspect.nix` | 4-phase (selfProvide → aspectPolicies → includes → transitions) | 3-phase (aspectPolicies → includes → transitions) |
| deprecation trace block | `aspect.nix` | ~7 lines detecting cross-provide keys | removed (compat handler owns this during compat phase; nothing left after) |

## Migration: `provides.X` Patterns

All `provides.X` writes in user/template code become aspect-included policies:

```nix
# Before:
den.aspects.igloo.provides.to-users = { user, ... }: {
  homeManager.programs.direnv.enable = true;
};

# After:
den.aspects.igloo.policies.to-users = { host, user }:
  [ (policy.include { homeManager.programs.direnv.enable = true; }) ];
```

```nix
# Before: targeted cross-provide
den.aspects.igloo.provides.alice = {
  homeManager.programs.vim.enable = true;
};

# After:
den.aspects.igloo.policies.to-alice = { host, user }:
  lib.optional (user.name == "alice")
    (policy.include { homeManager.programs.vim.enable = true; });
```

```nix
# Before: programmatic read
find-mutual = from: to: from.aspect.provides.${to.aspect.name} or {};

# After: eliminated — mutual-provider gone, policies handle routing
```

### `den.provides.*` factory namespace

The `den.provides` top-level namespace (`den.provides.forward`, `den.provides.define-user`, `den.provides.import-tree`, etc.) is the **factory registry** — NOT the aspect-level `provides` structural key. It is unaffected and out of scope.

### `den-brackets.nix` provides fallback

The `resolveWithProvidesFallback` in `den-brackets.nix` falls back to `aspect.provides.X` when direct `aspect.X` lookup fails. With `provides` gone, delete the fallback — direct resolution only.

## `resolveChildren` Simplification

Current state (4-phase): `emitSelfProvide` → `emitCrossProvideShims` (compat) → `emitAspectPolicies` → `emitIncludes` → `emitTransitions`.

After Phase 2 removal of `emitSelfProvide` and compat shims (3-phase):

```nix
childResolution =
  fx.bind (emitAspectPolicies aspect) (_:
    fx.bind (emitIncludes emitCtx (aspect.includes or [])) (includeResults:
      fx.bind (emitTransitions aspect) (transitionResults:
        fx.pure (includeResults ++ transitionResults))));
```

`emitTransitions` remains — it handles `into` transitions and policy dispatch. Only the self-provide and compat phases are removed.

## Dependency Order

### Phase 1: Compat layer (MUST ship first — separate spec)

See `2026-04-28-provides-compat-shims-design.md`. No removal work until this is complete.

### Phase 2: Removal (future release, after migration period)

```
Step 1: Migrate provides.X patterns in templates → policies.X
Step 2: Delete provides-compat.nix handler + emitCrossProvideShims call from resolveChildren
Step 3: Delete mutual-provider-shim.nix from modules/compat/
Step 4: Delete emitSelfProvide + mkPositionalInclude + mkNamedInclude from aspect.nix
Step 5: Delete emitCrossProvider from transition.nix (branch-era provides.${entityKind} routing)
Step 6: Simplify resolveChildren (remove selfProvide phase)
Step 7: Remove provides option + _ alias from types.nix
Step 8: Remove "provides" (and duplicate "policies") from structuralKeysSet
Step 9: Clean up den-brackets.nix provides fallback
Step 10: Assess substituteChild in include.nix — verify if used outside provides patterns
```

Steps 2-3 are the compat teardown. Steps 4-10 are the provides infrastructure removal.

## Impact Summary (revised)

| Metric | Value |
|--------|-------|
| Lines deleted (remaining, Phase 2) | ~190 (pipeline: selfProvide + crossProvider + compat) + ~20 (types) |
| Already deleted by pipeline simplification | ~690+ |
| Files to delete (Phase 2) | 2 (provides-compat.nix, mutual-provider-shim.nix) |
| Files with changes (Phase 2) | 5 (aspect.nix, transition.nix, types.nix, den-brackets.nix, include.nix) |
| Template files to migrate | ~35 (provides.X patterns) |

## Risks

- **Compat layer must ship first**: Users on `main` already hit hard errors (e.g., `den._.mutual-provider` missing). Phase 2 removal is blocked until the compat layer has been in place for a release cycle.
- **Template migration volume**: ~35 files, mechanical but high volume. Each `provides.X` becomes a `policies.X` function returning effects. Compat layer buys time for gradual migration.
- **`den.provides` factory confusion**: Users may confuse the `den.provides.*` factory namespace (kept) with the aspect `provides` structural key (deleted). Clear documentation needed.

## Out of Scope

These remain as future targets:
- **`wrapClassModule` collision detection** (~100 lines) — needs arg namespacing, separate breaking change
- **`classifyKeys` → declared schemas** (~67 lines) — can follow this work since `provides` no longer in the classification set
- **Child shape normalization** (~57 lines) — depends on collision detection removal
