# Class Emission Dedup — Post Unified Policy Effects

**Date:** 2026-04-27
**Branch:** feat/traits
**Status:** Draft
**Prerequisites:** Unified policy effects (2026-04-27-unified-policy-effects-design.md) implemented
**Supersedes:** Fix 2 and Fix 3 sections of 2026-04-26-class-emission-dedup-and-parametric-merge-design.md

## Context

The class emission dedup spec (2026-04-26) defined three fixes:
1. Parametric merge — coerce to includes in `providerType.merge`
2. Include-level dedup via `includeSeen` in `includeHandler`
3. Unsatisfied guard in `wrapClassModule` for missing den schema args

Fix 1 is type-system-level and unaffected by unified policy effects. Fixes 2 and 3 interact with the policy/constraint model which unified effects replaces. This spec describes how they change.

## Fix 2: Include-Level Dedup

### What stays

The `includeSeen` mechanism in `includeHandler` is the correct dedup point. The principle — "same aspect by identity, same context, resolved once regardless of how many parents include it" — is unchanged. The dedup key (`childIdentity` = `identity.pathKey(identity.aspectPath(child))`) remains correct.

### What changes: exclusion interaction

The original spec (section "Interaction with excludes and constraints") described dedup running BEFORE `check-constraint` dispatch:

```
if alreadyResolved then fx.pure []
else if isForward then ...
else fx.bind (fx.send "check-constraint" { ... }) (...)
```

With unified effects, `policy.exclude` replaces `register-constraint`/`check-constraint` for aspect exclusion. The exclusion is processed during the fixed-point policy iteration — not as a per-include check inside `includeHandler`.

**New interaction model:**

1. Policy fixed-point iterates: `policy.exclude den.aspects.X` registers an exclusion for aspect X
2. When aspect X is later encountered via `policy.include` or `includes`, `includeHandler` checks the exclusion set (a pipeline state field, replacing `constraintRegistry` for excludes)
3. If excluded: `excludeChild` fires, `includeSeen` is NOT recorded (matching the original spec's behavior)
4. If not excluded AND not in `includeSeen`: resolve normally, record in `includeSeen`
5. If not excluded AND in `includeSeen`: skip (dedup)

The ordering guarantee — "dedup runs before exclusion check" — no longer holds because exclusion is not a per-include dispatch anymore. Instead:

- **Exclusion state is set during policy iteration** (before aspect resolution)
- **Dedup state is set during aspect resolution** (as includes are processed)
- An aspect excluded by policy never reaches `includeHandler` via `policy.include` (the pipeline suppresses the injection). An aspect excluded after being included via `includes` hits the exclusion check in `includeHandler`.

The case from the original spec — "aspect excluded on first visit, included by second parent" — works differently:

**Before (constraint model):** First visit → `check-constraint` returns `exclude` → `excludeChild` → no `includeSeen`. Second visit → `check-constraint` returns `keep` → resolve normally.

**After (policy model):** `policy.exclude` from one policy scopes to that policy's context subtree. A different policy in a different context branch can still `policy.include` the same aspect. The `includeSeen` dedup correctly distinguishes these because `childIdentity` includes `__ctxId` — different context branches produce different dedup keys.

### What changes: `policy.include` as an include source

`policy.include den.aspects.X` injects aspect X into the current resolution context. The injected aspect flows through `includeHandler` — the same path as `includes = [X]`. The `includeSeen` dedup catches duplicates from both sources:

- `includes = [X]` on the aspect definition → `emit-include` → `includeHandler` → dedup check
- `policy.include X` from a policy effect → injected into includes → `emit-include` → `includeHandler` → same dedup check

No special handling needed. `policy.include` feeds into the same include pipeline.

### What changes: `substituteChild` removal

The original spec's `substituteChild` path handled `action == "substitute"` from `check-constraint`. Unified effects replaces substitution with `policy.exclude` + `policy.include` (composing existing effects). The `substituteChild` function (~18 lines) and the `"substitute"` branch in the constraint handler are dead.

The `includeSeen` interaction with substitute (original spec didn't address it explicitly) becomes moot.

## Fix 3: Unsatisfied Guard

### What stays

`wrapClassModule` detecting `missingDenArgNames` and returning `unsatisfied = true` is still correct. `emitClasses` skipping emission when `unsatisfied` is still correct. The guard prevents NixOS evaluation failures from class modules requesting den args not in context.

### What changes: reduced frequency

With unified policy effects, the guard fires less often because the policy layer prevents the problematic case upstream:

**Before:** An aspect with `{ user }` class module could be included by any parent regardless of context. If the parent resolves without `user` context, the module reaches NixOS and fails. The guard catches this.

**After:** Aspects are included via `policy.include` effects. A policy function `{ host, user }: [ (policy.include X) ]` only fires when both `host` and `user` are in context (signature matching). So aspect X only enters the include pipeline when its required context is available. The guard rarely fires because the policy layer pre-filters.

**Exception:** Aspects included via static `includes = [X]` on other aspects (not via policies) still need the guard. Example: `den.aspects.benix.includes = [ den.aspects.nix-trusted-user ]` where `benix` is a standalone aspect without user context. The guard correctly skips the `{ user }:` class module emission.

**Recommendation:** Keep the guard as defense-in-depth. It's ~15 lines and catches edge cases that policy signatures don't cover (static includes from aspects without full context).

### What changes: `wrapDeferredImports` propagation

The original class-emission-dedup spec didn't cover `wrapDeferredImports`. The fix/class-emission-dedup branch added `unsatisfied` propagation through `wrapDeferredImports` (filtering out unsatisfied items from deferred import lists). This propagation is still needed — `wrapDeferredImports` processes `{ imports = [fn]; }` attrsets where inner functions may request missing den args.

## `constraintRegistryHandler` Evolution

The `register-constraint`/`check-constraint` pair in `tree.nix` currently serves:

1. `meta.handleWith` — custom handler injection → **survives** (orthogonal to policies)
2. `meta.excludes` — aspect exclusion sugar → **replaced by `policy.exclude`**
3. `type == "filter"` — filter predicates → **survives** (used by `meta.handleWith`)
4. `type == "substitute"` — aspect substitution → **dead** (replaced by exclude + include)

After provides removal + unified effects:
- `register-constraint` keeps `"filter"` type handling, drops `"exclude"` and `"substitute"` types
- `check-constraint` keeps the filter predicate check, drops the exclude/substitute decision branches
- Estimated reduction: ~30 lines from the constraint handler

The `policy.exclude` effect is processed by the new `policyExcludeHandler` which maintains a separate exclusion state. `includeHandler` checks this state before resolving an include — replacing the `check-constraint` dispatch for exclusion.

## Summary of Changes from Original Spec

| Original section | Status | Notes |
|---|---|---|
| Fix 1 (parametric merge) | Unchanged | Type-system concern, orthogonal |
| Fix 2 dedup mechanism | Unchanged | `includeSeen` in `includeHandler` |
| Fix 2 exclusion interaction | Rewritten | `policy.exclude` replaces `check-constraint` for exclusion; scoped to policy context subtree |
| Fix 2 `substituteChild` | Dead | Substitution = exclude + include |
| Fix 3 guard mechanism | Unchanged | `unsatisfied` in `wrapClassModule`, skip in `emitClasses` |
| Fix 3 frequency | Reduced | Policy signatures pre-filter; guard is defense-in-depth |
| Fix 3 `wrapDeferredImports` | Unchanged | Propagation still needed |

## Non-goals

- Modifying `includeSeen` key format (correct as-is)
- Changing `wrapClassModule` collision detection (Target 2, separate spec)
- `classifyKeys` simplification (follows provides removal)
