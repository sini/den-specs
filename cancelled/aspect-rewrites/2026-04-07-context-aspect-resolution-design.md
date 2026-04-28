# Context-Level Aspect Resolution via Adapters

**Date:** 2026-04-07
**Branch:** `feat/context-aspect-resolution`
**Status:** Design approved, pending implementation
**Prior art:** `feat/aspect-rewrites` (POC — concepts accepted, approach rejected as too invasive)

## Summary

Den's `resolve.withAdapter` and `aspect.meta` provide composable, pluggable
aspect resolution. This design extends that foundation so adapters can be
declared at the **context level** (hosts, users) and on **individual aspects**
(subtree filtering), with proper identity and provider provenance flowing
through the entire pipeline — including forwards.

The design is delivered as 5 independent sub-features, each touching 1-3 files,
reviewable and testable in isolation.

## Goals

- Context stages (`den.ctx.host`, `den.ctx.user`) carry an adapter that
  controls how aspects are resolved within that context.
- Aspects can declare a subtree adapter via `meta.adapter` that composes with
  the inherited adapter during resolution.
- Aspect identity (`name`, `__provider`) survives parametric functor evaluation
  so adapters can filter reliably.
- Provider provenance is structural — adapters can distinguish `bar` included
  directly from `bar` provided by `foo`.
- Forward resolution respects context adapters.
- All changes are minimal and fit within each component's existing responsibility.

## Non-Goals

- No `resolve'` — all functionality integrates into the existing `resolve` /
  `withAdapter` interface.
- No `excludes` or `transforms` options on aspects or entities — these are
  expressed as adapter composition in userland.
- No `collectExcludes` or `collectAspectTransforms` in ctx-apply — the context
  adapter replaces this machinery.

## Sub-Features

### SF1: Identity Preservation

**Files:** `nix/lib/parametric.nix`

**Problem:** When parametric functors evaluate (`applyIncludes`, `withOwn`,
`deepRecurse`), they return `{ includes = [...]; }` — a plain attrset with no
`name` or `__provider`. Resolve's `apply` returns non-function results as-is,
skipping `aspectType.merge`, so the identity is lost. Adapters filtering on
`aspect.name` see incorrect defaults.

**Solution:** Each functor result carries identity from `self`:

```nix
__functor = self: ctx: {
  name = self.name;
  __provider = self.__provider or [];
  includes = ...;
};
```

Applied to all functor-producing functions in `parametric.nix`:
- `applyIncludes` (used by `atLeast`, `exactly`)
- `withOwn` (used by default parametric, `expands`)
- `deepRecurse` (used by `fixedTo`, `deep`)

**What this does NOT change:**
- `take.nix` — no `carryAttrs` needed. Identity is on the functor *output*,
  not carried across take boundaries.
- `statics.nix` — static includes don't go through functor evaluation.

**Testing:** A test using `resolve.withAdapter` with a custom adapter that
reads `aspect.name` on a parametric aspect, asserting the correct name
survives functor evaluation.

---

### SF2: Per-Aspect Adapter Accumulation in Resolve

**Files:** `nix/lib/aspects/resolve.nix`, `nix/lib/aspects/types.nix`

**Problem:** `resolve.nix`'s `go` function uses a single `adapter` closed over
the entire resolution tree. An aspect cannot declare "filter my subtree
differently."

**Solution:** Add a typed `adapter` option to the `meta` submodule in
`aspects/types.nix`:

```nix
# Inside the meta submodule options
options.adapter = lib.mkOption {
  description = "Adapter to compose into resolution for this aspect's subtree";
  type = lib.types.nullOr (lib.types.functionTo lib.types.raw);
  default = null;
};
```

Type `nullOr (functionTo raw)` enforces that the value is either `null` (no
subtree adapter) or a function receiving the inherited adapter and returning a
new adapter. Setting `meta.adapter = "foo"` produces a proper type error.

The `meta.adapter` signature is `inheritedAdapter -> adapter`, enabling
composition:

```nix
den.aspects.foo.meta.adapter = inherited:
  adapters.filter (a: a.name != "bar") inherited;
```

Modify `go` in `resolve.nix` to accumulate:

```nix
go = currentAdapter: prevChain: provided:
  let
    ...
    subtreeAdapter =
      if aspect.meta.adapter != null
      then aspect.meta.adapter currentAdapter
      else currentAdapter;
    recurse = go subtreeAdapter aspect-chain;
  in
  currentAdapter { inherit aspect class classModule recurse aspect-chain; };
```

- The **current node** is processed with `currentAdapter` (inherited).
- The **subtree** gets `subtreeAdapter` (current composed with aspect's own).
- `withAdapter` seeds: `go adapter []`.
- `resolve` (default) becomes: `go adapters.module []`.

**Composition order:** context adapter (outermost) > parent aspect adapter >
child aspect adapter (innermost). Each layer wraps the next.

**Backward compatibility:** The `withAdapter` signature is unchanged. The
`adapter` contract is unchanged. Existing adapters and tests work as-is.

**Testing:** An aspect sets `meta.adapter = inherited: adapters.filter ...`
and its subtree is filtered. Verified via `resolve.withAdapter` directly.

---

### SF3: Provider Provenance

**Files:** `nix/lib/aspects/types.nix`, `nix/lib/aspects/default.nix`,
`nix/lib/namespace-types.nix`

**Problem:** When `den.aspects.foo.provides.bar` enters resolution, the
result has `name = "bar"` (from SF1) but nothing indicating it was provided
by `foo`. An adapter cannot distinguish it from a top-level `den.aspects.bar`.

**Solution:** Thread `providerPrefix` through the type configuration (`cnf`)
that already flows through `aspectType -> aspectSubmodule -> providerType`.

1. `aspectSubmodule` sets `__provider = cnf.providerPrefix or []` as a
   default on each aspect.

2. The `provides` option passes an extended `cnf` to its children:
   ```
   cnf // { providerPrefix = (cnf.providerPrefix or []) ++ [config.name] }
   ```
   Each provided value's type is constructed with the extended prefix.

3. SF1 already carries `__provider = self.__provider or []` through functor
   evaluation, so the provenance survives the parametric pipeline.

**Namespace support:** Export `mkAspectsType` from `aspects/default.nix`:

```nix
mkAspectsType = cnf': rawTypes.aspectsType (typesConf // cnf');
```

`namespace-types.nix` changes its import from `den.lib.aspects.types.aspectsType`
to `den.lib.aspects.mkAspectsType` and calls it with the namespace prefix:

```nix
inherit (den.lib.aspects) mkAspectsType;
# ...
freeformType = mkAspectsType { providerPrefix = [ nsArgs.name ]; };
```

Top-level `den.aspects` continues using `aspectsType` (which is
`mkAspectsType {}`).

**Identity values by location:**

| Location | `name` | `__provider` |
|----------|--------|-------------|
| `den.aspects.foo` | `"foo"` | `[]` |
| `den.aspects.foo.provides.bar` | `"bar"` | `["foo"]` |
| `den.aspects.foo._.bar._.baz` | `"baz"` | `["foo", "bar"]` |
| `den.ns.bar` (namespace "ns") | `"bar"` | `["ns"]` |
| `den.ns.bar.provides.qux` | `"qux"` | `["ns", "bar"]` |

**Testing:** An adapter reads `aspect.__provider` and filters based on
provider origin. Verified with `resolve.withAdapter`.

---

### SF4: Context-Level Adapter

**Files:** `nix/lib/ctx-types.nix`, `nix/lib/ctx-apply.nix`

**Problem:** Users need to declare "use this adapter when resolving aspects
for this context." There is no adapter option on context nodes.

**Solution:** Add `adapter` option to `ctxSubmodule` in `ctx-types.nix`:

```nix
options.adapter = lib.mkOption {
  description = "Base adapter for aspect resolution in this context";
  type = lib.types.nullOr (lib.types.functionTo lib.types.raw);
  default = null;
};
```

Type is `nullOr (functionTo raw)`. `null` means "no override — use the
inherited adapter or `adapters.module` at the final consumption point."
Using `null` as default avoids unnecessary composition wrapping — a context
that doesn't set an adapter doesn't add a nesting layer.

**Propagation across context transitions:** Adapters compose across context
stage transitions. A host adapter wraps the user adapter — host-level filtering
cannot be overridden by child contexts.

In `ctx-apply.nix`, `traverse` threads an accumulated adapter:

```nix
traverse = args@{ prev, prevCtx, self, ctx, key, adapter }:
  let
    selfAdapter = self.adapter or null;
    composedAdapter =
      if selfAdapter != null && adapter != null
      then args: selfAdapter (adapter args)
      else selfAdapter or adapter;
    ...
    expandOne = { path, into }:
      ...
      traverse {
        prev = self;
        prevCtx = ctx;
        self = aspect;
        ctx = c;
        key = aspectKey;
        adapter = composedAdapter;
      };
  in
  ...
```

`ctxApply` seeds it and carries the composed adapter on the result:

```nix
ctxApply = self: ctx: {
  adapter = <composed adapter from traversal>;
  includes = assembleIncludes (traverse {
    prev = null;
    prevCtx = null;
    key = self.name;
    adapter = self.adapter or null;
    inherit self ctx;
  });
};
```

**Usage:**

```nix
# Exclude aspect "foo" from host igloo's resolution
den.ctx.host.adapter = adapters.filter (a: a.name != "foo") adapters.module;
```

**Composition order:**
```
host.adapter (outermost) > user.adapter > deeper contexts
  > aspect.meta.adapter (subtree, from SF2)
    > child.meta.adapter (deeper subtree)
```

**Testing:** A template feature example in `templates/ci/modules/features/`
demonstrating context-level adapter with cross-stage composition — a host
adapter excludes an aspect and the exclusion applies to user-level resolution.

---

### SF5: Forward and Output Consumption

**Files:** `nix/lib/forward.nix`, `modules/outputs.nix`,
`modules/aspects/provides/os-user.nix`

**Problem:** `forward.nix` and `outputs.nix` call `den.lib.aspects.resolve`
directly, ignoring the adapter carried on the `ctxApply` result.

**Solution:** Both callsites use `resolve.withAdapter` with the adapter from
the aspect/context they are resolving.

`forward.nix` line 22 changes from:
```nix
sourceModule = mapModule (den.lib.aspects.resolve fromClass asp);
```
To:
```nix
sourceModule = mapModule (
  den.lib.aspects.resolve.withAdapter
    (asp.adapter or den.lib.aspects.adapters.module)
    fromClass
    asp
);
```

`outputs.nix` line 9 changes similarly:
```nix
flakeModule =
  let ctx = den.ctx.flake { };
  in den.lib.aspects.resolve.withAdapter
    (ctx.adapter or den.lib.aspects.adapters.module)
    "flake"
    ctx;
```

**Routing internal forwards through context:** `os-user.nix` currently
bypasses `ctxApply`:
```nix
fromAspect = _: den.lib.parametric.fixedTo { inherit host user; } den.aspects.${user.aspect};
```

Changed to route through the context stage:
```nix
fromAspect = _: den.ctx.user { inherit host user; };
```

This ensures the context's composed adapter reaches the forward's resolve
call. `home-env.nix` already routes through `den.ctx` via `userEnvAspect`.

**Note on behavioral change:** Routing `os-user.nix` through `den.ctx.user`
instead of raw `parametric.fixedTo` means `fromAspect` now resolves the entire
user context pipeline (including all `den.ctx.user.includes`), not just the
single user aspect. This is a semantic broadening — all user-level context
includes (like `mutual-provider`, other batteries) will be included in the
forward resolution. This is believed to be correct behavior (the forward should
see the same aspects the context produces), but existing tests should be
verified during implementation. If any tests break, the behavioral difference
should be investigated before adjusting.

**Testing:** A context-level adapter filters an aspect that would have been
forwarded into home-manager. Verify it is excluded from the forwarded output.

## Dependency Chain

```
SF1 (identity) ─┬─→ SF2 (resolve accumulation) ─→ SF4 (context adapter) ─→ SF5 (forward/outputs)
                 └─→ SF3 (provider provenance)
```

SF2 and SF3 are independent of each other. Both require SF1. SF4 requires SF2.
SF5 requires SF4.

Each sub-feature is a single PR-sized change touching 1-3 files with its own
tests.

## Files Changed (Total)

| File | Sub-feature | Change |
|------|-------------|--------|
| `nix/lib/parametric.nix` | SF1 | Carry `name` and `__provider` in functor results |
| `nix/lib/aspects/types.nix` | SF2, SF3 | Typed `meta.adapter` option; `providerPrefix` in cnf; `__provider` default |
| `nix/lib/aspects/resolve.nix` | SF2 | Adapter accumulation in `go` loop |
| `nix/lib/aspects/default.nix` | SF3 | Export `mkAspectsType` |
| `nix/lib/namespace-types.nix` | SF3 | Use `mkAspectsType` with namespace prefix |
| `nix/lib/ctx-types.nix` | SF4 | `adapter` option on context submodule |
| `nix/lib/ctx-apply.nix` | SF4 | Thread and compose adapter in traverse; carry on result |
| `nix/lib/forward.nix` | SF5 | Use `resolve.withAdapter` with `asp.adapter` |
| `modules/outputs.nix` | SF5 | Use `resolve.withAdapter` with `ctx.adapter` |
| `modules/aspects/provides/os-user.nix` | SF5 | Route `fromAspect` through `den.ctx.user` |
| `templates/ci/modules/features/` | SF1-SF5 | Test modules for each sub-feature |

## Relationship to Prior Work

The `feat/aspect-rewrites` POC implemented excludes, substitutes, and traces
by deeply modifying 47 files across the core pipeline (ctx-apply, parametric,
take, forward, types, definition). The concepts were accepted but the approach
was rejected as too invasive.

This design achieves the same capabilities through Vic's `resolve.withAdapter`
and `aspect.meta` primitives:

| POC mechanism | This design |
|---------------|-------------|
| `den.lib.aspects.transforms.exclude` | `adapters.filter (a: a.name != "x")` |
| `den.lib.aspects.transforms.substitute` | `adapters.mapAspect` |
| `den.lib.aspects.transforms.trace` | Custom trace adapter (see resolve-adapters.nix) |
| `resolve'` with opts | `resolve.withAdapter` (already on main) |
| `aspect.excludes` option | `meta.adapter` or context adapter with `adapters.filter` |
| `aspect.transforms` option | `meta.adapter` with composed adapters |
| Entity excludes (`host.excludes`) | `den.ctx.host.adapter` |
| `collectExcludes` in ctx-apply | Adapter composition across context transitions |
| `__provider` / `wrapProvider` | `providerPrefix` in type cnf |
| `carryAttrs` in take.nix | Not needed — identity on functor output |
| `withIdentity` in parametric.nix | Same concept, carries `name` + `__provider` |
