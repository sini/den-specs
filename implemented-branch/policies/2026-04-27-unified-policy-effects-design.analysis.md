# Analysis: Unified Policy Effects Design

## Verdict

Substantially implemented. Core effect model (three effect types, `__policyEffect` dispatch, plain functions) is fully shipped. Aspect-included policies, trait collection, cross-entity distribution, and the `provides` compat shim are all present. The `corePolicies` registry described in the spec was never built — core policies merged into `den.policies` directly. Trait consumer deferral via `has-handler` / `drain-deferred` is implemented but the spec's explicit fixed-point iteration language does not match the actual mechanism (transition depth + ctx-seen dedup). Battery migration section is aspirational — HM battery uses aspect-included policies but is not structured exactly as the spec example.

## Delivery Target

`feat/fx-pipeline` branch. Shipped across ~15 commits from `db189e17` (plain function shape) through `2e40b024` (mutual-provider compat shim). All core commits landed; no open PR merging this to main at time of analysis.

## Evidence

**Effect constructors present and match spec exactly:**
- `/home/sini/Documents/repos/den/nix/lib/policy-effects.nix` — `policy.resolve`, `policy.resolve.shared`, `policy.resolve.to`, `policy.include`, `policy.exclude`, all returning `{ __policyEffect = "..."; value = ...; }` tagged attrsets.

**`__policyEffect` dispatch in pipeline:**
- `nix/lib/aspects/fx/handlers/transition.nix` lines 421–429: filters rawEffects by `__policyEffect == "resolve"`, `"include"`, `"exclude"` and routes accordingly.
- `nix/lib/aspects/fx/handlers/tree.nix` lines 293–294: include/exclude effects dispatched during tree-walk (Phase A spec behavior).

**Policies are plain functions:**
- `nix/lib/policy-types.nix`: `policyFnArgs = policy: lib.functionArgs policy` — no `__functor` unwrapping, assumes plain function.
- `nix/nixModule/policies.nix`: `den.policies` option typed as `lazyAttrsOf raw` — no submodule, no `from/to/as/resolve/aspects/handlers` fields.
- Commit `de80246b` removed all `__functor` wrappers from policies; commit `f5282ac8` converted template policies.

**Old `policyType` submodule replaced:**
- No `policyType` submodule exists in `nix/lib/` or `nix/nixModule/`. `nix/lib/policy-types.nix` is a thin utility (arg extraction only), not the old submodule.
- `_core` field: stripped by commits `73ac2f50`, `82864411`, `424888b7`.
- `isolateFanOut`: field removed from policy declarations; pipeline infers from `__shared` flag on resolve effects (transition.nix line 445).

**Aspect-included policies:**
- `nix/lib/aspects/fx/aspect.nix` line 664: emits `register-aspect-policy` for each entry in `aspect.policies`.
- `nix/lib/aspects/fx/handlers/tree.nix` line 204: `register-aspect-policy` handler collects into `state.aspectPolicies`.
- `nix/lib/aspects/fx/handlers/tree.nix` line 245: `dispatch-policy-includes` runs include-only aspect policies during tree-walk.
- `nix/lib/aspects/fx/handlers/transition.nix` line 471: aspect policies dispatched for resolve effects in transitions.

**Trait infrastructure:**
- `nix/lib/aspects/fx/handlers/trait.nix`: `emit-trait` handler, Tier 1/2/3 classification, `state.traits` collection.
- `nix/lib/aspects/fx/handlers/tree.nix` line 303: `drain-deferred` handler.
- `nix/lib/synthesize-policies.nix` line 19: `traitNames = den.traits or {}` used in `resolveArgsSatisfied` — traits are treated as satisfiable context args without requiring handler presence.
- `nix/lib/aspects/fx/distribute-cross-entity.nix`: cross-entity trait distribution (Phase 2 architecture).

**`mutual-provider` replaced:**
- Commit `11625e47`: replaced mutual-provider with aspect-included `policyFns`.
- Commit `2e40b024`: added compat shim for legacy `provides` structural key.
- `nix/lib/aspects/fx/handlers/provides-compat.nix`: wraps old `provides.X` values into `policy.include` effects with deprecation warnings pointing to migration syntax.

**HM battery aspect-included policy:**
- `nix/lib/home-env.nix` line 57: "Self-contained battery: host → user routing via aspect-included policy."
- Lines 101–108: uses `den.lib.policy.resolve`, `den.lib.policy.include` directly.

## Current Status

| Spec Feature | Status |
|---|---|
| `policy.resolve` / `policy.include` / `policy.exclude` constructors | Shipped |
| `__policyEffect` tag dispatch | Shipped |
| Policies as plain functions | Shipped |
| Old `policyType` submodule removed | Shipped |
| `_core` field removed | Shipped |
| `isolateFanOut` removed from declarations | Shipped |
| Aspect-included policies via `policies` key | Shipped |
| `register-aspect-policy` / `dispatch-policy-includes` | Shipped |
| `den.traits` schema + `emit-trait` collection | Shipped |
| Trait consumer deferral (`drain-deferred`) | Shipped |
| Cross-entity trait distribution | Shipped |
| `mutual-provider` replaced | Shipped |
| `provides` compat shim | Shipped |
| `corePolicies` registry in `den.lib` | Not built — core policies live in `den.policies` |
| `policy.resolve.shared` opt-out | Shipped (`__shared` flag) |
| Fixed-point iteration language from spec | Partial — ctx-seen dedup + transition depth limit approximates this, not a formal iterate-until-stable loop |
| Trait cycle detection (spec section) | Partial — `partialOk` validation present; explicit cycle error path not confirmed |

## Supersession

Supersedes the field-based `policyType` submodule from `nix/nixModule/policies.nix` (pre-fx-pipeline). Also supersedes the `__functor`-wrapped policy shape that was an intermediate step (visible in commit `2820ed62` introducing it, then `de80246b` removing it). The `provides` structural key on aspects is superseded by `policies` key — compat shim provides backward compatibility.

## Gaps

1. **`corePolicies` registry absent.** Spec says "Core policies are registered in `den.lib.corePolicies` and always active." In practice, core policies (flake outputs, default routing) are merged into `den.policies` directly with no separate registry. This is a spec/impl divergence but not a functional gap — the behavior is equivalent.

2. **`policy.resolve.shared` spec gap.** Spec notes "Opt-out via `policy.resolve.shared` if needed (to be specified during implementation)." This was implemented (`policy.resolve.shared`, `__shared` flag in transition.nix), but the spec's "to be specified" clause was never closed with documentation of when to use it.

3. **Aspect-included policy rollback on exclude.** Spec says "excluding an aspect removes its included policies too... those effects are rolled back." The registration/dispatch machinery exists but rollback of already-produced resolve effects was not confirmed in the code — `includeSeen` tracks included aspects but policy effect rollback path was not found in transition.nix.

4. **Trait cycle detection.** Spec describes explicit cycle detection (deferred consumers exist, no new emissions → cycle error). The `partialOk` validation in `pipeline.nix` line 346 catches one case, but the general cycle detection described (both A and B defer, drain detects stalemate) was not confirmed implemented.

5. **Battery migration incomplete.** The `batteries` section describes `home-manager-battery` and `maid-battery` as self-contained with inline `policies` key. HM uses aspect-included policy in `home-env.nix` but the structural form differs from the spec example (it's a factory, not a literal `den.aspects.home-manager-battery` attrset). Maid battery migration status not verified.

## Drift

- **`policy.resolve.to`** was added beyond the spec's three base effects. Spec only describes `policy.resolve`, `policy.resolve.shared`; implementation adds `policy.resolve.to "kind" {}` and `policy.resolve.shared.to "kind" {}` for explicit target kind routing. This is an additive extension, not a conflict.

- **`__entityKind` in resolve context.** Transition.nix line 403 injects `__entityKind = sourceEntityKind` into `resolveCtx`. Not mentioned in spec's context model — internal pipeline detail added during implementation (commit `4d06b4be`).

- **`provides` compat shim.** Spec describes `provides` structural key as fully replaced. Implementation added a backward-compat shim (`provides-compat.nix`) that translates old `provides.X` to `policy.include` effects at runtime with deprecation warnings. This is a compatibility layer beyond the spec's clean-cut migration.

- **`den.lib.policy` exposure path.** Spec says "provided via `den.lib.policy`." Confirmed: `nix/lib/default.nix` line 31 maps `policy = ./policy-effects.nix`. Used in `home-env.nix` and `provides-compat.nix` as `den.lib.policy.resolve` / `den.lib.policy.include`. Consistent.

## Cross-Reference Notes

- **Consumed by pipeline-simplification spec**: `2026-04-28-policy-pipeline-simplification.analysis.md` declares this spec as a prerequisite that is "fully consumed." The three effect constructors, `__policyEffect` dispatch, and plain-function shape established here are the foundation the pipeline-simplification spec built on. The pipeline-simplification analysis treats Phases A–E as complete; this spec covers Phase A (effects model) through Phase D (aspect-included policies).
- **Supersession chain**: This spec sits between the effectful-bootstrap approach (which was abandoned — see `2026-04-23-effectful-pipeline-bootstrap-design.analysis.md`) and the pipeline-simplification (which completed the cleanup). The "NOT IMPLEMENTED" bootstrap spec and this spec were parallel explorations; the plain-function path taken here made the bootstrap spec moot.
- **`scope.provide` position**: This spec does not claim `scope.provide` is used for policy dispatch, which is correct. The only `scope.provide` in the codebase is `aspect.nix:915` for parametric context propagation via `__scopeHandlers`. Policy dispatch uses direct iteration only.
