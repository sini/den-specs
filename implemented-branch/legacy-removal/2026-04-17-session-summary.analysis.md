# Analysis: Session Summary — Legacy Resolve Pipeline Removal

## Verdict

Fully implemented. All removals, rewrites, and architectural decisions described in the
session summary are reflected in the current state of `feat/fx-pipeline`. The work was
absorbed via commit `2e0fc9e1` ("refactor: remove legacy resolver, stabilize fx pipeline")
on 2026-04-23, and the branch has continued to evolve well past the concerns in this
document.

## Delivery Target

`feat/rm-legacy` (documented branch) → absorbed into `feat/fx-pipeline` via commit
`2e0fc9e1`. No separate `feat/rm-legacy` branch exists in the current repo.

## Evidence

### Files listed as "successfully removed" — confirmed gone

| File | Status |
|------|--------|
| `nix/lib/statics.nix` | Absent — not present anywhere in `nix/` |
| `nix/lib/aspects/resolve.nix` | Absent |
| `nix/lib/aspects/adapters.nix` | Absent |
| `modules/fxPipeline.nix` | Absent |
| `defaultFunctor` in `aspects/default.nix` | No match in file |
| `typesConf` in `aspects/default.nix` | No match in file |
| Legacy parametric symbols (`withOwn`, `deepRecurse`, `applyDeep`, `applyIncludes`, `mapIncludes`) | No matches anywhere in `nix/` |
| `den.fxPipeline = false` in test files | Only reference is a `docs/superpowers/` working doc |

`nix/lib/aspects/` now contains only: `default.nix`, `fx/`, `has-aspect.nix`, `types.nix`.

### Files listed as "successfully rewritten" — confirmed present and correct

**`nix/lib/aspects/has-aspect.nix`** — uses `fxFullResolve` + `state.pathSet` exactly as
described. No reference to `resolve.withAdapter` or `adapters.collectPaths`.

**`nix/lib/parametric.nix`** — completely rewritten. No `bindCtx`, no `resolveIncludes`,
no `pipelineArgs`. Current form exposes only deprecation shims (`fixedTo`, `atLeast`,
`exactly`, `expands`) that warn and delegate to `mkFixedTo` / `constantHandler`. The
"simplified to `bindCtx` with recursive `resolveIncludes`" intermediate design was itself
superseded.

**`nix/denTest.nix`** — trace utility present. Uses `pipeline.mkPipeline` with
`fxTrace.tracingHandler`; builds legacy tree shape from flat `entries`. Includes the
`builtins.intersectAttrs` partial-matching fix (Session 2 item 1) and the
`[displayName]` leaf wrapping fix (Session 2 item 2).

**`templates/ci/modules/features/aspect-path.nix`** — uses `fx.identity.aspectPath`
(verified by git diff stat in `2e0fc9e1`).

### Architectural decisions — confirmed in code

1. **`__functor = lib.const` default in `aspectSubmodule`** — `compileStatic` path in
   `nix/lib/aspects/fx/aspect.nix` handles non-callable attrsets without a `__fn`.
   `lib.const` as default routes correctly.

2. **`providerFnType.merge` with functor wrapping** — `compileFunctor` still present in
   `aspect.nix`; `wrapChild` / inner-function extraction intact.

3. **`aspect-chain` kept in pipeline** — confirmed. `pipeline.nix` initialises
   `"aspect-chain" = [ ]` in `defaultHandlers` and `"aspect-chain" = [ self ]` in
   `rootHandlers`. Still used by `fromAspect = _: lib.head aspect-chain` in multiple
   template providers (`wsl.nix`, `nvf-integration.nix`, `guarded-forward.nix`, etc.).

4. **`forward.nix` `fromAspect` hook preserved** — confirmed. `forward.nix` reads
   `fwd.fromAspect item`. `osConfigurations.nix`, `hmConfigurations.nix`, `os-user.nix`
   all use `fromAspect` with proper context objects, not raw `aspect-chain`.

5. **`scope.run` instead of `scope.stateful`** — confirmed. No call to `scope.stateful`
   exists anywhere in `nix/lib/aspects/fx/`. `scope.provide` is used in `aspect.nix`
   for parametric children (the Session 2 refinement from `scope.run` to `scope.provide`
   from nix-effects).

## Current Status

All 5 "key architectural decisions" are in effect. The three remaining Session 2 blockers
(parametric `atLeast`/`exactly` include loss, inline class-key dedup, gutted test stubs)
were resolved in subsequent commits on `feat/fx-pipeline`:

- Parametric include loss → addressed by `8f9f283d` (parametric type separation,
  `scope.provide`); `parametric.nix` now purely shims, concern removed.
- Duplicate module declarations → `52f0acc4` (include-level dedup) and `4c58d6d6`
  (multi-class collection in `classCollectorHandler`).
- Identity/provenance test stubs → rebuilt as part of the broader pipeline stabilization
  commits.

The branch is well past the 433/461 test state described in "Session 2 progress". The
`feat/fx-pipeline` branch has ~737 commits total with extensive post-legacy-removal work
(policies, traits, stages elimination, unified type, provides-compat shims, etc.).

## Supersession

Several intermediate designs described in this document are superseded:

| Document describes | Current reality |
|---|---|
| `bindCtx` + recursive `resolveIncludes` in `parametric.nix` | Removed; `parametric.nix` is now shim-only |
| `scope.run` for `transitionHandler` + `keepChild` | `scope.provide` (nix-effects) used instead |
| Filter-unresolvable-includes approach | Superseded by proper effects dispatch — parametric functions handled via `__scopeHandlers` + `constantHandler` |
| `__ctx` conditional on callable attrsets | Removed; `__scopeHandlers` is the only mechanism |
| `isBareArgFn` + `takeFn` special-case | Not present; eliminated with parametric simplification |

## Gaps

None with respect to declared goals. The session document's "Recommended next steps"
items (unresolvable parametric handling, dedup, test stubs, stale comments) are all
resolved in later commits.

## Drift

- The session summary describes `feat/rm-legacy` at HEAD `39493107`. This commit hash does
  not appear in `feat/fx-pipeline` history, indicating the work was likely squashed or
  cherry-picked into `2e0fc9e1` rather than merged with history preserved.
- The `aspect-chain` footprint has grown since the session summary predicted it could be
  removed "once providers are rewritten." As of branch HEAD, 15+ `fromAspect` callsites
  still use `lib.head aspect-chain` (templates primarily). This is an open follow-up, not
  tracked in any current plan visible in the repo.
- `templates/ci/modules/features/cross-context-forward.nix` was listed as deleted in
  `2e0fc9e1` stat but `parametric.fixedTo` is still used in one remaining callsite in
  that template (the spec noted aspect-chain-dependent `fromAspect` removal was
  selective).
