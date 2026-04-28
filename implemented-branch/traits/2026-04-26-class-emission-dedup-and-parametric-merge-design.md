# Class Emission Dedup & Parametric Aspect Merge

**Target branch:** `feat/traits`
**Date:** 2026-04-26
**Status:** Draft
**Prerequisites:** Traits and structural nesting (complete), stages elimination (complete)

## Problem

Two bugs in the aspect merge and resolution pipeline:

### 1. Silent last-wins on parametric aspect merge (`types.nix`)

When two modules define the same aspect path as parametric wrappers (`__fn`/`__args`), the module system merge silently drops all but the last definition.

**Important context:** At the top-level `den.aspects.X` path, `coercedProviderType` already coerces bare parametric functions to `{ includes = [fn]; }` *before* `providerType.merge` runs. So `den.aspects.X = { user }: ...` from two modules already merges correctly (both coerced to attrsets with includes, merged additively).

The actual bug affects:
- **Parametric wrappers (`__fn`/`__args`)** in `providerType.merge` (lines 171–178): pre-wrapped values (e.g., from programmatic construction or internal factories) bypass `coercedProviderType` and hit the wrapper detection path → `lib.last`.
- **`mergeFunctions`** (lines 130–134): bare parametric functions inside `includes` list element types or nested paths where `providerType` is used without the outer coercion layer → `lib.last`.

**Note on `provides`:** The pipeline simplification spec (2026-04-26) is landing a provides deprecation shim that rewrites `provides.X` to direct nesting at the type level. This reduces but does not eliminate the parametric wrapper path — programmatic construction and internal factories (e.g., `den.provides.unfree`) still produce `__fn`/`__args` wrappers that hit `providerType.merge` directly.

**Not affected (already correct):**
- Top-level `den.aspects.X = fn` → coerced by `coercedProviderType` before merge
- Mixed fn + attrset → coerces fns to `{ includes = [fn]; }`
- All attrsets → submodule merge
- All submodule fns → `aspectType` merge
- Class-key-level (`den.aspects.X.nixos`) → `aspectContentType` merge (additive via `__contentValues`)

### 2. Duplicate resolution of aspects included via multiple parents

When the same aspect is included by two different parents within the same pipeline run, the pipeline resolves the full subtree twice. There is no include-level dedup — only `ctx-seen` for transitions. This causes duplicate class emissions, duplicate trait emissions, and duplicate nested aspect processing.

**Root cause:** `includeHandler` processes each `emit-include` independently. When `security.includes = [ trusted-user ]` and `base.includes = [ trusted-user ]` are both in the same entity's include tree, that's two `emit-include` effects → two full `aspectToEffect` → `compileStatic` → `resolveChildren` passes for the same aspect with the same context.

```nix
den.aspects.trusted-user = {
  nixos = { user, ... }: { nix.settings.trusted-users = [ user.userName ]; };
};
den.aspects.security.includes = [ den.aspects.trusted-user ];
den.aspects.base.includes = [ den.aspects.trusted-user ];
den.aspects.pol.includes = [
  den.provides.define-user
  den.aspects.security
  den.aspects.base
];
# trusted-user resolved twice (once per parent), producing:
# - duplicate emit-class for nixos (trusted-users = ["root" "pol" "pol"])
# - duplicate emit-trait if trusted-user had trait keys
# - duplicate nested aspect resolution if trusted-user had sub-aspects
```

The `classCollectorHandler` doesn't catch this because resolved parametric includes get anonymous names (from `nameAnon`, containing `/<anon>:`), and the `isAnon` check routes them to `setDefaultModuleLocation` — no NixOS `key`, no dedup.

### 3. Missing guard for unsatisfied context args

When a class module function on a static (non-parametric) aspect requests den schema args (e.g., `{ user, ... }:`) that aren't in the current pipeline context, the module passes through to NixOS where it fails with a missing attribute error. Should skip emission and defer to a wider context.

`wrapClassModule` already detects missing schema args and emits `lib.warn` (lines 172–179 on feat/traits), but does not suppress emission — the module still flows through to NixOS where it fails.

```nix
den.aspects.tools.nix-trusted-user = {
  nixos = { user, ... }: { nix.settings.trusted-users = [ user.userName ]; };
};

# benix includes nix-trusted-user but has no user context
den.aspects.benix = {
  includes = [ den.aspects.tools.nix-trusted-user ];
  nixos.users.users.benix.isNormalUser = true;
};
# → error: attribute 'user' missing (in NixOS eval)
```

## Design

### Fix 1: Parametric merge — coerce to includes

Make the parametric-only paths consistent with the existing mixed path.

**`providerType.merge` (lines 171–178):** When all defs are parametric wrappers (`__fn`/`__args`), unwrap each to its `__fn`, coerce to `{ includes = [fn]; }`, merge through `aspectType`.

**`mergeFunctions` (lines 130–134):** When all defs are bare parametric functions (no submodule fns), coerce each to `{ includes = [fn]; }`, merge through `aspectType` (same as `mergeMixed` already does).

After coercion, each function becomes an independent include. They resolve separately via `bind.fn` in the pipeline, each getting its own `wrapClassModule` pass. Anonymous identity (`<parent>/<anon>:0`, `<parent>/<anon>:1`) keeps them distinct in traces and bypasses include-level dedup (correct — they are genuinely different functions producing different class modules).

**`__functor` conflict:** When multiple defs at the same path define `__functor` (callable aspect factories like `den.provides.unfree`), error instead of last-wins. Two factories at the same path is conflicting intent that can't be mechanically composed. Users who genuinely need override semantics can use `lib.mkForce`.

**Summary of merge behavior after fix:**

| Case | Before | After |
|---|---|---|
| All parametric wrappers (`__fn`/`__args`) | last-wins | unwrap to fns, coerce to includes, merge |
| All bare parametric fns (no submodule fns) | last-wins | coerce to includes, merge |
| Multiple `__functor` defs | last-wins on functor | **error** |
| Top-level `den.aspects.X = fn` | coerced by `coercedProviderType` | *(unchanged, already correct)* |
| Mixed fn + attrset | coerce fns to includes | *(unchanged)* |
| All attrsets | submodule merge | *(unchanged)* |
| All submodule fns | `aspectType` merge | *(unchanged)* |

### Fix 2: Include-level dedup in `includeHandler`

The fix prevents duplicate resolution at the source — the `includeHandler` skips re-resolution of aspects already processed in the current pipeline run. This eliminates duplicate class emissions, trait emissions, and nested aspect processing in one shot. No changes needed to `classCollectorHandler` or `traitCollectorHandler`.

#### Semantic identity

The dedup key is the child's full `childIdentity` — already computed at line 247 via `identity.pathKey(identity.aspectPath(child))`. This includes `meta.provider ++ [name] ++ optional ctxId`, preserving full provenance including the `<anon>:N` position index for parametric includes.

`childIdentity` is stable across different include paths because:
- `meta.provider` comes from the aspect definition (type system), not the runtime include chain
- `name` for anonymous includes is assigned by `nameAnon` using `lib.last chain` — the direct parent, not the grandparent. `trusted-user/<anon>:0/{pol,x1c}` is the same whether reached via `security` or `base`
- `__ctxId` comes from the resolution context (same user transition in both paths)

```nix
# In includeHandler, after childIdentity is computed (line 247):
dedupKey =
  if isMeaningfulName (child.name or "<anon>") then
    childIdentity  # full path with provenance: "trusted-user/<anon>:0/{pol,x1c}"
  else
    null;  # bare "<anon>" with no meaningful parent — skip dedup
```

- `"trusted-user/<anon>:0/{pol,x1c}"` → meaningful name → `dedupKey = "trusted-user/<anon>:0/{pol,x1c}"`
- `"foo/bar/shared-setting"` → meaningful → `dedupKey = "foo/bar/shared-setting"`
- `"<anon>"` → not meaningful → `dedupKey = null` → no dedup

#### State tracking

Add `includeSeen` to pipeline state — a thunk-wrapped set of resolved dedup keys:

```nix
# In includeHandler, before resolution dispatch:
seen = (state.includeSeen or (_: { })) null;
alreadyResolved = dedupKey != null && seen ? ${dedupKey};
```

#### Handler changes

In `includeHandler` (include.nix), after computing `childIdentity` and before the `isForward`/`isConditional`/constraint dispatch:

```nix
{
  resume =
    if alreadyResolved then
      fx.pure [ ]  # skip — already resolved in this pipeline run
    else if isForward then
      ...  # existing forward path
    else if isConditional then
      ...  # existing conditional path
    else
      fx.bind (fx.send "check-constraint" { ... }) ( ... );

  state =
    if alreadyResolved || dedupKey == null then
      state
    else
      state // {
        includeSeen = _: seen // { ${dedupKey} = true; };
      };
}
```

The `includeSeen` set is recorded on first visit (before resolution, not after) and wrapped as a thunk for `deepSeq` safety, following the same pattern as `state.traits`, `state.imports`, etc.

**Note:** `state` is returned from the handler alongside `resume`. Recording `includeSeen` in the state return (not inside the resume continuation) ensures the key is registered even if the resolution is complex (deferred, conditional, etc.).

#### What gets skipped

When `alreadyResolved` is true:
- `aspectToEffect` is not called — no `compileStatic`, no `emitClasses`, no `emitTraits`, no `resolveChildren`
- The aspect's `resolve-complete` does not fire a second time
- The `chainHandler` does not record a duplicate chain entry
- `fx.pure []` returns empty results to the parent's includes list

The first resolution produced all the class emissions, trait emissions, and nested aspect content. The second visit contributes nothing.

#### Interaction with excludes and constraints

The `check-constraint` dispatch (lines 260–272) handles `excludes` and `meta.handleWith`. Dedup check runs BEFORE constraint check — if an aspect was already resolved, it's skipped regardless of whether a new parent would have excluded it. This is correct: the exclusion system prevents an aspect from being included in a subtree, but once the aspect has been resolved (producing emissions), excluding it from a second parent's subtree shouldn't un-resolve it.

If an aspect is excluded on its FIRST visit (constraint returns `exclude`), the `excludeChild` path fires and no `includeSeen` entry is recorded (the state update is conditional on `!alreadyResolved && dedupKey != null`). A subsequent visit from a non-excluding parent resolves normally.

#### Interaction with deferred includes

Deferred includes (`defer-include` effect, lines 196–211) are parametric includes whose required args aren't yet available. They're re-emitted when context widens (during transitions). The deferred include stores the original child — when it re-emits, it goes through `includeHandler` again with a new context.

Deferred includes should NOT be blocked by `includeSeen` from a previous failed attempt. The key is context-dependent: `dedupKey` includes `__ctxId`. A deferred include that failed in host context (no `__ctxId` for user) and re-emits in user context (with `__ctxId = "{pol,x1c}"`) gets a different `dedupKey` — no collision.

For deferred includes that re-emit with the SAME context (rare — would mean the context didn't actually widen), `alreadyResolved` correctly skips them.

#### Interaction with Fix 1

Functions coerced to includes by Fix 1 become anonymous (`<parent>/<anon>:N`). `semanticName` for these is the parent aspect name (before `/<anon>:`), but they don't have meaningful standalone identity — they're position-indexed children of a specific parent. Two coerced functions from two different `providerType.merge` defs would have the same `semanticName` but different content.

These bypass dedup because their `child.name` is bare `"<anon>"` (from `wrapBareFn`), so `isMeaningfulName` returns false and `dedupKey = null`. Each coerced function resolves independently (correct — they are different functions). Note that `nameAnon` only runs for includes with `idx != null` and non-meaningful names — coerced functions from Fix 1 land in includes lists and DO get `nameAnon` names. But the parent aspect itself is only resolved once (via its own dedup), so its children are only emitted once.

### Fix 3: Guard unsatisfied class module args

**`wrapClassModule` (aspect.nix):** The function already has the detection infrastructure on feat/traits (lines 169–179): `schemaKinds` filtering, `missingDenArgNames` computation, and `lib.warn` for missing args. The change: when `missingDenArgNames` contains args that have no default (`!(allArgs.${k} or false)`), return `{ unsatisfied = true; }` early instead of proceeding to wrap.

This guard must run **before** the existing `denArgNames == [] && traitArgNames == []` early-return (line 181). Trait args (e.g., `firewall`) are NOT affected — they're in `den.traits`, not `den.schema`, so they don't appear in `schemaKinds`.

```nix
# Insert before existing line 181:
hasMissingDenArgs = missingDenArgNames != [];
# ...
if hasMissingDenArgs then
  { inherit module; wrapped = false; unsatisfied = true; missingArgs = missingDenArgNames; }
else if denArgNames == [] && traitArgNames == [] then
  # ... existing path
```

**`emitClasses` (aspect.nix):** When `result.unsatisfied or false` is true, return `[]` from the `concatMap` — skip both `mainEmit` and `validatorEmit`. Includes still resolve normally via `resolveChildren` — only class key emission is suppressed. The module will emit when the aspect resolves in a context that provides the required args (e.g., user transition).

```nix
# Replace current unconditional emission:
in
if result.unsatisfied or false then
  []
else
  [ mainEmit ] ++ lib.optional (result ? validator) validatorEmit
```

**Note on `__contentValues` unwrapping:** `emitClasses` already unwraps `aspectContentType` wrappers (lines 398–408 on feat/traits), recovering the original module value before passing to `wrapClassModule`. The guard operates on the unwrapped value, so `aspectContentType` is transparent to Fix 3.

**Note on traitCollectorHandler ctx scoping (review finding #2):** The unsatisfied guard uses `ctx` from `ctxFromHandlers(aspect.__scopeHandlers)`, which captures the scoped context at the point of class emission. This has the same root-ctx limitation noted in the traits branch review: after a transition introduces new scoped context, the guard may not see it. In practice, parametric resolution typically resolves such functions before they reach `emitClasses`, but this is an architectural limitation shared with `traitCollectorHandler`.

## Files Changed

| File | Change |
|---|---|
| `nix/lib/aspects/types.nix` | Fix 1: parametric wrapper and bare-fn merge paths, `__functor` conflict error |
| `nix/lib/aspects/fx/handlers/include.nix` | Fix 2: include-level dedup via `includeSeen` state |
| `nix/lib/aspects/fx/aspect.nix` | Fix 3: unsatisfied guard in `wrapClassModule`, skip in `emitClasses` |
| `templates/ci/modules/features/nested-class-module-args.nix` | Tests for all three fixes |

## Test Cases

### Parametric merge (Fix 1)

- **test-parametric-wrapper-merge**: Two modules produce `__fn`/`__args` wrappers at same provides path — both coerced to includes, both contribute.
- **test-mixed-parametric-and-attrset**: One module sets fn, another sets attrset at same path — existing mixed path works (regression guard).
- **test-functor-conflict-errors**: Two modules both define `__functor` at same aspect path — should error.

### Include-level dedup (Fix 2)

- **test-dedup-static-aspect-two-parents**: Static aspect included via two bundles — resolved once, class emitted once.
- **test-dedup-parametric-class-two-parents**: Aspect with parametric class module, same context, included via two parents — resolved once, class emitted once.
- **test-no-dedup-different-contexts**: Same aspect resolved with different `__ctxId` values — resolved twice (correct, different contexts).
- **test-dedup-trait-two-parents**: Aspect with trait emission included via two parents — trait collected once (not doubled).
- **test-excluded-then-included**: Aspect excluded by first parent, included by second — resolves on second visit (exclusion doesn't pollute `includeSeen`).

### Unsatisfied guard (Fix 3)

- **test-guard-skips-without-context**: Static aspect with `{ user, ... }:` class module included by host-only aspect — no error, emission skipped.
- **test-guard-defers-then-emits**: Same aspect also included via user transition — emits correctly in the wider context, proving end-to-end deferral.

### Class-key merge (existing behavior, new coverage)

- **test-same-class-key-same-signature-merges**: Two modules set `den.aspects.X.nixos = { user, ... }: <module>` — both contribute via `aspectContentType` merge (values collected in `__contentValues`, unwrapped by `emitClasses`).
- **test-same-class-key-different-signatures-merges**: Two modules set `den.aspects.X.nixos` with `{ host }` and `{ user }` — both contribute independently.

## Non-goals

- Removing `builtins.trace` debug statements (cleanup task, not part of this fix)
- Changing `provides` deprecation path (handled by pipeline simplification spec)
- Modifying sub-pipeline forward isolation
- `mkForce` override escape hatch for `__functor` conflict (future work if needed)
- Fixing traitCollectorHandler root-ctx limitation (documented in traits review findings #2)

## Appendix: Alternative — emission-level dedup via `dedupIdentity`

An earlier iteration of this spec proposed dedup at the emission level rather than the include level. This approach is documented here as a rejected alternative.

### Approach

Add a `dedupIdentity` field to `emit-class` payloads — the aspect's full identity path + ctxId, independent of the include chain. The `classCollectorHandler` would use `dedupIdentity` for the NixOS module `key`, and override the `isAnon` check to force keyed dedup when `dedupIdentity` is present.

### Why it was rejected

1. **Only fixes class emissions.** Trait emissions (`emit-trait`) and nested aspect resolution have the same duplication problem. Emission-level dedup would require parallel dedup logic in `traitCollectorHandler` and nested aspect handling — three dedup mechanisms for one root cause.

2. **Treats the symptom, not the cause.** The pipeline resolves the same aspect's full subtree twice. Emission-level dedup allows the duplicate resolution to run (wasting computation), then discards the duplicate outputs. Include-level dedup prevents the duplicate work entirely.

3. **NixOS `key` semantics are fragile.** The approach relies on NixOS's module dedup behavior: same `key` → first kept, second silently dropped. This is undocumented internal behavior of the module system. The anonymous path (`setDefaultModuleLocation` without `key`) exists precisely because NixOS key-based dedup was unreliable for computed modules.

### When emission-level dedup might be preferred

If include-level dedup proves too aggressive (e.g., an aspect intentionally produces different results based on which parent's constraint context it resolves under), emission-level dedup could serve as a fallback. The `dedupIdentity` approach is mechanically sound for class emissions and could be added to `classCollectorHandler` as defense-in-depth without conflicting with include-level dedup.
