# Provides Backwards-Compatibility Shims

**Date:** 2026-04-28
**Branch:** feat/fx-pipeline
**Status:** Implemented

Shipped on `feat/fx-pipeline` (commits 19738452..2e40b024). Pipeline handler `provides-compat.nix` and mutual-provider shim both landed.
**Prerequisite for:** Provides removal (2026-04-27-provides-removal-post-unified-effects.md)

## Problem

Users migrating from `main` to the merged `feat/fx-pipeline` branch hit hard errors and silent behavioral breakage:

```
error: attribute 'mutual-provider' missing
  at /home/pol/Code/drupol/infra/modules/dendritic.nix:18:29:
      17|
      18|   den.ctx.user.includes = [ den._.mutual-provider ];
         |                             ^
      19|   den.schema.user.classes = lib.mkDefault [ "homeManager" ];
```

```
trace: den: aspect 'x1c' uses 'provides.{to-users}' — migrate to direct nesting
```

On `main`, cross-entity routing was handled by the `mutual-provider` battery: users included it via `den.ctx.user.includes = [ den._.mutual-provider ]`, and it auto-wired `provides.to-users`, `provides.to-hosts`, and named-target `provides.alice` patterns. On this branch, `mutual-provider.nix` was deleted without a translation layer. The `provides.X` cross-provide patterns still parse (the `provides` option exists on the aspect submodule) but their routing mechanism is gone — config silently drops.

**Note:** `emitCrossProvider` in `transition.nix` is a **branch-era addition** (commit `ed1301a4`, part of the provide-to policy implementation). It handles `provides.${entityKind}` patterns (e.g., `provides.user`) during transitions — a different key space from the `mutual-provider` patterns (`to-users`, `to-hosts`, named targets). It does not exist on `main` and does not overlap with the compat shims designed here.

## Goal

Ship compatibility shims that make all `main`-era patterns work with deprecation warnings and migration advice. No silent breakage. Users' existing configs evaluate correctly and produce the same outputs while warnings guide them to the new patterns.

## Compat Surface Inventory

| Surface | Status before this spec | Action |
|---|---|---|
| `den.ctx.*` | Shimmed (`modules/compat/ctx-shim.nix`) | None |
| `den.stages` | Shimmed (`modules/removed-stages.nix`) | None |
| `den.lib.take` / `den.lib.parametric` | Already shimmed to coerce to correct shape | None |
| `den.provides.host-aspects` | Behavior unchanged (verified) | None |
| `den.provides.*` factory namespace | Unchanged | None |
| `den-brackets.nix` provides fallback | Still present, still works | None |
| `_` → `provides` alias on aspects | Still present | None |
| `aspect.into` manual transitions | `emitTransitions` still reads it | None |
| **`den._.mutual-provider`** | **Hard error** — file deleted | **New shim** |
| **`provides.X` cross-provides** | **Silent drop** — routing deleted | **New handler** |

## Design

### 1. Pipeline handler: `provides-compat.nix`

**File:** `nix/lib/aspects/fx/handlers/provides-compat.nix`

**Entry point:** `emitCrossProvideShims : aspect → fx computation`

During tree-walk of each aspect, detects cross-provide keys (`provides.X` where `X != aspect.name`) and synthesizes `register-aspect-policy` effects that replicate the old `mutual-provider` routing.

**Detection:**

```nix
crossKeys = builtins.filter (k: k != aspectName)
  (builtins.attrNames (aspect.provides or {}));
```

Self-provide (`provides.${self.name}`) is untouched — `emitSelfProvide` handles that path and it still works.

Entity-kind keys (e.g., `provides.user`, `provides.host`) are handled by `emitCrossProvider` in `transition.nix` during transition resolution. The compat handler skips keys that are registered schema kinds to avoid double emission:

```nix
schemaKinds = builtins.attrNames (den.schema or {});
compatKeys = builtins.filter (k: k != aspectName && !builtins.elem k schemaKinds) crossKeys;
```

**Value resolution:**

Cross-provide values can be static attrsets, plain functions, `__fn` attrsets (parametric wrappers or partially processed), or functor attrsets. The helper mirrors the shape detection from `emitSelfProvide` (aspect.nix lines 696-703):

```nix
applyProvide = value: ctx:
  if builtins.isAttrs value && value ? __fn then value.__fn ctx
  else if builtins.isAttrs value && value ? __functor then (value.__functor value) ctx
  else if lib.isFunction value then value ctx
  else value;
```

The first branch covers both full parametric wrappers (`{ __fn, __args }`) and bare `__fn`-only attrsets — both resolve the same way for cross-provides.

**Policy synthesis per pattern:**

Synthesized policies use **named argument signatures** so the dispatch system's `functionArgs`-based context matching works correctly. On `main`, `to-users` only fired when both `host` and `user` were in context; `to-hosts` fired when `host` was in context. The signatures replicate this scoping:

| Pattern | Signature | Guard | Body |
|---|---|---|---|
| `to-users` | `{ host, user, ... }:` | None (fires for all users) | `policy.include` the value |
| `to-hosts` | `{ host, user, ... }:` | None (fires for all hosts) | `policy.include` the value |
| Named target | `{ host, user, ... }:` | Entity name match | `policy.include` the value |

All three use `{ host, user, ... }:` — on `main`, mutual-provider ran in the user pipeline where both host and user were always in context. This prevents premature dispatch at host-only level.

```nix
# to-users / to-hosts: fires for every entity pair
mkWildcardPolicy = value: { host, user, ... }:
  [ (policy.include (applyProvide value { inherit host user; })) ];

# Named target: fires only when entity name matches
mkNamedTargetPolicy = key: value: { host, user, ... }:
  lib.optional
    (host.name == key || user.name == key)
    (policy.include (applyProvide value { inherit host user; }));
```

**Behavioral note:** `policy.include` feeds into `aspectToEffect`, the same resolution pipeline as the old mechanism. Deep nesting (provides values with their own `includes`) is an edge case that needs test coverage.

**Named target guard:** User names and host names do not overlap (existing restriction on `main`). The guard checks both `host.name` and `user.name`.

**Effect emitted per cross-provide key:**

```nix
fx.send "register-aspect-policy" {
  name = "${aspectName}/compat:${key}";
  fn = warnedPolicyFn;
  ownerIdentity = nodeIdentity;  # supports exclusion rollback
}
```

The `ownerIdentity` comes from `identity.pathKey (identity.aspectPath aspect)`, same as real aspect policies. This ensures that if the aspect is excluded via `policy.exclude`, its compat-synthesized policies are also rolled back.

**Exclusion rollback verification needed:** The `registerAspectPolicyHandler` in `tree.nix` stores policies keyed by `param.name`. Verify that the exclusion mechanism (`registerExcludes` in `transition.nix`) actually filters `state.aspectPolicies` by `ownerIdentity`, not just by name. If exclusion only matches by name, the `ownerIdentity` field provides no rollback and the spec's claim is incorrect. In that case, compat policies need their names to include the owner identity for exclusion to work.

**Deprecation warning:**

Each synthesized policy wraps `fn` with `lib.warn`:

```
den: aspect 'igloo' uses provides.to-users — migrate to:
  den.aspects.igloo.policies.to-users = { host, user, ... }:
    [ (policy.include { <config> }) ];
```

The warning fires once per policy evaluation (Nix deduplicates identical `lib.warn` strings).

**Integration with `resolveChildren`:**

Single `fx.bind` call, inserted before `emitAspectPolicies`. Replaces the existing deprecation-only trace block (aspect.nix lines 757-761):

```nix
# Before (current):
childResolution = fx.bind (builtins.seq _ (emitSelfProvide aspect)) (
  selfProvResults:
  fx.bind (emitAspectPolicies aspect) (

# After:
childResolution = fx.bind (emitSelfProvide aspect) (
  selfProvResults:
  fx.bind (providesCompat.emitCrossProvideShims aspect) (
    _:
    fx.bind (emitAspectPolicies aspect) (
```

The `builtins.seq _` deprecation trace block is deleted — the handler owns warnings now.

### 2. Module shim: `mutual-provider-shim.nix`

**File:** `modules/compat/mutual-provider-shim.nix`

Makes `den.provides.mutual-provider` (and `den._.mutual-provider` via the `_` alias) evaluate to a valid but inert aspect:

```nix
{ lib, ... }:
{
  den.provides.mutual-provider = lib.warn
    "den.provides.mutual-provider is deprecated — cross-entity routing is now built-in via policies. Remove from includes."
    {
      name = "mutual-provider";
      description = "Deprecated compat shim — remove from includes.";
      # On main, mutual-provider was a parametric aspect. Some users may apply it
      # with arguments (den._.mutual-provider { ... }). The __functor accepts and
      # ignores any arguments to prevent hard errors.
      __functor = _: _:
        { name = "mutual-provider"; description = "Deprecated compat shim."; };
    };
}
```

On `main`, `mutual-provider` was a parametric aspect included in `den.ctx.user.includes`. Its job was cross-entity routing via `provides.X`. That routing is now handled by the `provides-compat` pipeline handler, so the include is a no-op. The shim produces a valid aspect shape (attrset with `name`) so the include mechanism doesn't error. The `__functor` ensures that applying it as a function (`den._.mutual-provider { ... }`) also doesn't error.

**Import wiring:** This module must be added to the compat module import list (alongside `ctx-shim.nix` and `removed-stages.nix`). Check `modules/default.nix` or wherever compat modules are aggregated and add the import.

## What This Does NOT Cover

- **Self-provide removal** (`provides.${self.name}`) — still works via `emitSelfProvide`, removal is Phase 2
- **`provides` option removal** from `types.nix` — the option must remain for the shim to read `aspect.provides`
- **`provides` in `structuralKeysSet`** — must remain so provides keys aren't treated as freeform content
- **Template migration** — users migrate on their own schedule guided by warnings
- **`_` alias removal** — must remain so `den._.X` continues to work
- **`den.fxPipeline` option** — this option was introduced on `feat/fx-pipeline` and later removed on the same branch. It never existed on `main`. Users migrating from `main` cannot have set it. No shim needed.

All of the above (except fxPipeline) are Phase 2 (provides removal spec) and blocked until the compat layer has been in place for a release cycle.

## Removal Path

When migration period ends:

1. Delete `nix/lib/aspects/fx/handlers/provides-compat.nix`
2. Remove the `emitCrossProvideShims` call from `resolveChildren` in `aspect.nix`
3. Delete `modules/compat/mutual-provider-shim.nix`
4. Proceed with Phase 2 of provides removal spec

Two files deleted, one line removed. Clean cut.

## Testing

### provides-compat handler

- Aspect with `provides.to-users = { homeManager... }` (static) → config reaches user entity
- Aspect with `provides.to-users = { host, ... }: { homeManager... }` (parametric) → config reaches user entity with correct host context
- Aspect with `provides.alice = { homeManager... }` (named target) → config reaches only user `alice`
- Aspect with `provides.igloo = { nixos... }` (named target, reverse direction) → config reaches only host `igloo`
- Aspect with `provides.to-users` where value is `{ __fn = ...; __args = ...; }` (parametric wrapper) → resolved correctly
- Aspect with `provides.to-users` where value is `{ __fn = ...; }` (bare __fn, no __args) → resolved correctly
- `provides.${entityKind}` keys (e.g., `provides.user`) are NOT handled by compat (deferred to `emitCrossProvider` in transition.nix)
- Excluded aspect's compat policies are rolled back (verify ownerIdentity linkage — see note above)
- `to-users` policy does NOT fire at host-only level (named args `{ host, user, ... }` prevent premature dispatch)
- All patterns emit deprecation warnings

### mutual-provider shim

- `den.ctx.user.includes = [ den._.mutual-provider ]` evaluates without error
- `den.ctx.user.includes = [ den.provides.mutual-provider ]` evaluates without error
- `den._.mutual-provider { }` (applied as function) evaluates without error
- Deprecation warning fires

## Risks

- **Parametric wrapper shapes**: The `applyProvide` helper must handle all value shapes that the aspect submodule's freeform type can produce (`__fn` attrsets, functor attrsets, plain functions, static attrsets). If a shape is missed, the provide value passes through unevaluated and produces a type error downstream. Mitigated by mirroring the shape detection from `emitSelfProvide` (aspect.nix lines 696-703).
- **Named target ambiguity**: The guard checks both `host.name` and `user.name`. If a future entity kind (e.g., `home`) has a `.name` field that could match, the guard needs extending. Currently safe — only host and user entities have named targets in `provides.X` patterns on main.
- **Evaluation order**: `emitCrossProvideShims` runs before `emitAspectPolicies` in `resolveChildren`. If an aspect has both `provides.X` compat policies and real `policies.X` entries for the same routing, both fire. This is correct (the user is mid-migration) but could produce duplicate config. The deprecation warning should make this visible.
- **Deep nesting**: Cross-provide values that themselves contain nested sub-aspects with their own `includes` are an edge case. The `policy.include` path feeds into `aspectToEffect` recursion, which should handle this, but it's the least-tested path. Add explicit test coverage for nested provides values.
- **Exclusion rollback**: Verify that `ownerIdentity` on compat policies actually enables rollback on exclusion. If the exclusion mechanism doesn't filter by owner, compat policies for excluded aspects would continue firing. See verification note in design section.
