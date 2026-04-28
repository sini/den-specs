# Analysis: Unified Effects Pipeline
## Verdict
**Fully implemented.** All three architectural issues are resolved. Core design (aspectToEffect, emit-class, emit-include, into-transition, constantHandler, { lib, den } wiring) landed in `68c26555` (den-fx, merged 2026-04-17). Subsequent 169 commits on feat/fx-pipeline built further on the spec's foundation without reverting any of it.

## Delivery Target
- **Primary commit:** `68c26555` — "refactor: den-fx (#462)" — landed all core spec items
- **Branch:** feat/fx-pipeline (169 commits ahead of main at analysis time)
- **Spec branch was:** feat/fx-resolution — the work landed on a successor branch

## Evidence

### Module wiring — `{ lib, den }` only
- `nix/lib/aspects/fx/default.nix`: barrel, no init function, all imports take `{ inherit lib den; }` only
- `nix/lib/aspects/fx/aspect.nix`, `pipeline.nix`, `handlers/include.nix`, `handlers/transition.nix`: all open with `{ lib, den, ... }:` and access dependencies via `den.lib.fx`, `den.lib.aspects.fx.*`
- `den.lib.fx` is set once at `nix/lib/fx.nix` (fetches nix-effects from flake lock, falls back to tarball fetch)
- No `init` function found anywhere in `nix/lib/aspects/fx/`
- Spec's barrel example matches production code exactly

### aspectToEffect
- Implemented in `nix/lib/aspects/fx/aspect.nix` (lines 909–959) — parametric path uses `fx.bind.fn userArgs fn` + `scope.provide`, static path calls `compileStatic`
- Exported from `aspect.nix` attrset and re-imported by `pipeline.nix`, `handlers/include.nix`, `handlers/transition.nix`
- `compileStatic` emits `emit-class`, `emit-trait`, `register-constraint`, nested aspects, then calls `resolveChildren` (which emits `emit-include`, `into-transition`, `resolve-complete`)

### emit-class
- Sent in `aspect.nix` (`emitClasses` function) — one `fx.send "emit-class"` per class key + optional validator emit
- Handler in `handlers/tree.nix` — `"emit-class"` handler accumulates modules into `state.classImports`
- Spec deviation: `emit-class` payload now includes `isContextDependent` field (not in spec); spec's `identity` field is present

### emit-include
- Sent in `aspect.nix` (`emitIncludes`, `emitSelfProvide`, `dispatchPolicyIncludes`)
- Handler in `handlers/include.nix` — `"emit-include"` handler owns constraint-check + recursion via `aspectToEffect child`
- Spec's include-level dedup added: `includeSeen` state, `include-unseen` effect for rollback on exclude

### into-transition
- Sent in `aspect.nix` (`emitTransitions`, line ~598: `fx.send "into-transition" {...}`)
- Handler in `handlers/transition.nix` — `"into-transition"` handler; processes `manualTransitions` + `policyTransitions`; uses `constantHandler scopedCtx` for context scoping
- Spec used `scope.run`; production uses `scope.provide` (landed in `8f9f283d` after nix-effects deep handler fix)

### constantHandler
- Implemented in `handlers/ctx.nix` — `constantHandler = ctx: builtins.mapAttrs (_: value: { param, state }: { resume = value; inherit state; }) ctx`
- Used in `pipeline.nix` (defaultHandlers), `handlers/transition.nix`, `nix/lib/parametric.nix`, `nix/lib/resolve-entity.nix`, `nix/lib/forward.nix`
- Spec's `staticHandler` and `contextHandlers` are absent — replaced by `constantHandler` as specified

### Contexts as aspects
- No `ctx-apply.nix` or `ctx-stage.nix` in production lib (confirmed absent from `nix/lib/aspects/fx/`)
- Context resolution goes through `aspectToEffect` via `__scopeHandlers` + `__ctxId` tagging
- `into-transition` handler installs `constantHandler scopedCtx` as scope handlers on target aspects

### Removed files (all confirmed absent from nix/lib)
- `ctx-apply.nix` — gone
- `ctx-stage.nix` — gone
- `resolve-deep.nix` — gone
- `resolve-handler.nix` — gone
- `resolve-one.nix` — gone
- `wrap.nix` — gone
- `resolveAspect`, `resolveOne`, `resolveIncludeHandler`, `foldIncludes`, `buildStageIncludes`, `emitProviders`, `pureIdentity`, `pureConstraints`, `pureIncludes` — all absent
- Old effect names `provide-class`, `resolve-include` — absent

## Current Status
Production. The spec's full pipeline architecture is running in `feat/fx-pipeline`. Post-spec additions (not deviations):
- `emit-trait` effect + trait handlers (traits feature, post-spec)
- `dispatch-policy-includes`, `register-aspect-policy` effects (aspect-included policies, post-spec)
- `provide-to`, `forward` handlers (forward post-processing, post-spec)
- `defer-include`, `drain-deferred` effects (deferred parametric includes, post-spec)
- Include-level dedup via `includeSeen` / `include-unseen` (stability fix, post-spec)
- `emitCrossProvideShims` (compat shim for main-era cross-provides, post-spec)
- 4-step key classification: class → trait → nested → unregistered (unified key type, post-spec)

## Supersession
Spec is fully superseded by production code. The spec describes a design intent from 2026-04-14; `68c26555` (2026-04-17) delivered it with minor adaptations documented below.

## Gaps
None that represent regressions or missing spec obligations. The spec's §8 effect protocol is fully present in production.

One spec item partially generalized: §4 "Self-provide auto-include" — implemented as `emitSelfProvide` in `aspect.nix` (provides.${name} logic), not auto-include of all provides but only self-provide. Broader provides-to-direct-nesting migration was deferred (spec `e865a331`).

## Drift
| Spec | Production | Notes |
|---|---|---|
| `nxFx = den.lib.nxFx` | `fx = den.lib.fx` | Name is `fx` not `nxFx`; same concept |
| `scope.run` in into-transition | `scope.provide` | nix-effects API; `scope.provide` is the stable form |
| `bind.fn` in parametric path | `fx.bind.fn userArgs fn` + `scope.provide scopeHandlers` | scope injection added for handler-closure propagation |
| `constantHandler` docs as "parametricHandler rename" | Implemented as spec named it | No drift |
| `structuralKeys` list in spec | Production adds `traits`, `classes`, `__fn`, `__args`, `__scopeHandlers`, `__ctxId`, `__entityKind`, `__parametricResolved`, `_module`, `_` | Extended for post-spec features |
| `emit-class` payload: `{ class, module, identity }` | Adds `isContextDependent` field | Backward compatible extension |
| No `emit-trait` in spec | `emit-trait` effect added | Post-spec traits feature |
| No policy dispatch in into-transition spec | Full policy dispatch in transition handler | Post-spec policy pipeline |
