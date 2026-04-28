# Deep Provider Cascade, Substitute, and Namespace Qualification

**Date:** 2026-04-04
**Status:** Implemented
**Branch:** `feat/aspect-rewrites`

## Problem

The initial `__provider` was a single-level string. This limited cascade
to one level, created ambiguity between namespace and root aspects with
the same name, and `substitute` had no provider awareness.

## Solution

### 1. `__provider` as list path

`__provider` is `listOf str`, default `cnf.providerPrefix or []`.

Examples:
- Root aspect `monitoring`: `__provider = []`
- Provider `monitoring._.node-exporter`: `__provider = ["monitoring"]`
- Deep provider `m._.n._.c`: `__provider = ["monitoring" "node-exporter"]`
- Namespace aspect `eg.monitoring`: `__provider = ["eg"]`
- Namespace provider `eg.monitoring._.node-exporter`:
  `__provider = ["eg" "monitoring"]`

Using a list avoids ambiguity when aspect names contain `/`.

### 2. `toAspectPath` replaces `toAspectName`

Returns a list path from an aspect ref, string, or existing list:

```nix
toAspectPath = ref:
  if builtins.isList ref then ref
  else if builtins.isString ref then [ ref ]
  else (ref.__provider or []) ++ [ (ref.name or (throw "...")) ];
```

String support is kept for internal compatibility (ctxApply pipeline)
but not exposed to users. Aspect-level `excludes` accept raw refs.

### 3. Prefix-based matching

`isPrefix` with empty-list guard:

```nix
isPrefix = prefix: path:
  prefix != [] && lib.take (builtins.length prefix) path == prefix;
```

The `exclude` transform:

```nix
exclude = refs:
  let
    paths = map toAspectPath refs;
    isExcluded = ap:
      builtins.any (p: p == ap || isPrefix p ap) paths;
  in
  { provided, ... }:
  if isExcluded (aspectPath provided) then null else provided;
```

### 4. Provider-aware substitute

Cascades to providers with auto-match and prune-missing:

```nix
substitute = ref: replacement:
  let refPath = toAspectPath ref;
  in { provided, ... }:
  let ap = aspectPath provided;
  in
  if ap == refPath then
    replacement // { __placedBy = refPath; }
  else if isPrefix refPath ap then
    let sub = (replacement.provides or {}).${provided.name} or null;
    in if sub != null then sub // { __placedBy = refPath; } else null
  else provided;
```

Precedence: transforms apply left-to-right. A specific substitute
placed earlier fires first.

### 5. Deep provider chains via `providerPrefix` threading

The `provides` type threads `parentPath` through `childCnf.providerPrefix`
to child submodules. Each level inherits the full parent chain:

```nix
provides =
  let
    parentPath = base ++ [ name ];
    childCnf = cnf // { providerPrefix = parentPath; };
  in lib.mkOption {
    type = lib.types.submodule ({ config, ... }: {
      freeformType = lib.types.lazyAttrsOf (providerType childCnf);
    });
    apply = lib.mapAttrs (_: wrapProvider parentPath);
  };
```

`__provider` option default is `cnf.providerPrefix or []`, so namespace
aspects inherit their prefix automatically.

### 6. Namespace qualification

`mkAspectsType` exported from `aspects/default.nix`. Namespace types
pass `providerPrefix = [ nsName ]`:

```nix
nsAspectsType = mkAspectsType ({
  defaultFunctor = ...;
} // lib.optionalAttrs (name != null) {
  providerPrefix = [ name ];
});
```

Namespace providers get qualified paths matching angle bracket notation.
Excluding a namespace aspect does not affect same-named root aspects.

### 7. Trace improvements

- `provider` field as list path on trace entries
- `placedBy` field from `__placedBy` annotation on substituted aspects
- `__placedBy` carried through `withIdentity` and `carryAttrs`
- Mermaid renderer: qualified labels, dedup by display name

## Known Asymmetry

Entity-level excludes (host/user) flow through ctxApply which uses
string-based exclude sets. The `toAspectName` in `nix/lib/types.nix`
extracts only the name string, losing `__provider` path info. Provider
cascade still works at the aspect level (via the `includes` apply
filter in `aspectSubmodule`), but ctxApply's `traverse` uses simple
name matching. A future refactor of ctxApply could adopt list-path
matching.

## Files Changed

| File | Change |
|------|--------|
| `nix/lib/aspects/types.nix` | `__provider` as `listOf str`, `wrapProvider` with lists, `toAspectPath`, `isPrefix`, `providerPrefix` threading, `__placedBy` option |
| `nix/lib/aspects/transforms.nix` | `exclude`/`substitute` with prefix matching, `isPrefix`, `toAspectPath`, `aspectPath` helpers |
| `nix/lib/aspects/resolve.nix` | Trace entries with list `provider` and `placedBy` |
| `nix/lib/aspects/default.nix` | Export `mkAspectsType`, `toAspectPath` |
| `nix/lib/namespace-types.nix` | `providerPrefix` threading to `mkAspectsType` |
| `nix/lib/parametric.nix` | `withIdentity` carries `__provider` and `__placedBy` lists |
| `nix/lib/take.nix` | `carryAttrs` carries `__provider` and `__placedBy` |
| `nix/lib/ctx-apply.nix` | `refToName` for raw ref extraction |
| `nix/lib/types.nix` | Entity-level `toAspectName` kept for ctxApply compat |

## Breaking Changes

- `__provider` type: `nullOr str` → `listOf str`
- `toAspectName` replaced by `toAspectPath` (returns list)
- `compose` returns raw value instead of `{ result; trace; }`
- `normalizeResult` no longer preserves `trace` from transform output
- Trace `provider` field: string → list
- `transforms.trace` removed from public API
- String excludes still technically accepted (via `toAspectPath`
  string branch) for internal ctxApply compatibility
