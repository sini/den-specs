# ~~Provides Removal~~ — SUPERSEDED

**Date:** 2026-04-27 (amended 2026-05-02, **superseded 2026-05-04**)
**Branch:** feat/fx-pipeline
**Status:** **CANCELLED** — provides is a permanent user-facing API
**Consolidated spec:** `2026-05-02-fx-pipeline-consolidated.md` Section 5 (updated)

## Why This Spec Is Superseded

`provides.to-users`, `provides.to-hosts`, and `provides.<name>` are established user-facing APIs for cross-entity content delivery. Users depend on them. The `provides` structural key has a remaining purpose: it IS the user API.

The legacy transition system that powered provides has been eliminated. The `provides-compat` handler translates provides into `policy.include` effects — this is the correct **permanent implementation**, not a temporary shim to delete.

The aspect keyspace is being flattened so `provides.` indirection is no longer required for new code, but the syntax continues to work as first-class API.

## What This Means for Downstream Work

The dependency chain that gated constraint cleanup, providerType dispatch, and aspect key type on provides removal no longer applies. Those items are independently implementable.

## Preserved Insights

These observations from the original spec remain useful for future work:

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
