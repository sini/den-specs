# Analysis: Flake-Scope Args in Aspect Pipeline Functions

## Verdict
implemented
Both proposed additions shipped on `feat/fx-pipeline` in three commits (abb1bfdd, 6942ad5d, c19e8ca6). `pipelineOnly` utility and `den.provides.flake-scope` battery match the spec exactly. All 9 spec test cases are present in the test file.

## Delivery Target
feat/fx-pipeline

## Evidence

### `pipelineOnly` utility
- `/home/sini/Documents/repos/den/nix/lib/policy-effects.nix` lines 48-60: exact impl from spec — attrset path uses `//`, non-attrset path uses `__functor` wrapper; both emit `collisionPolicy = "class-wins"`.
- Exported as `den.lib.policy.pipelineOnly` via `/home/sini/Documents/repos/den/nix/lib/default.nix` line 31: `policy = ./policy-effects.nix;`.
- Commit abb1bfdd introduced it.

### `resolveCollisionPolicy` pre-existing hook
- `/home/sini/Documents/repos/den/nix/lib/aspects/fx/aspect.nix` lines 38-54: `resolveCollisionPolicy` checks `ctx.${name} ? collisionPolicy` and returns `ctx.${name}.collisionPolicy`. Both `__functor` and attrset forms produce an attrset with `collisionPolicy`, so both paths flow correctly without pipeline changes — matches spec claim.
- Lines 115, 136, 212, 217, 226, 256: `class-wins` branch drops den value silently; error thrown for null policy — matches spec collision semantics.

### `den.provides.flake-scope` battery
- `/home/sini/Documents/repos/den/modules/aspects/provides/flake-scope.nix`: exact battery from spec — `den.provides.flake-scope`, policy key `den-flake-scope`, `resolve { lib = pipelineOnly lib; inputs = pipelineOnly inputs; den = pipelineOnly den; }`.
- Commit 6942ad5d introduced it.

### Test suite
- `/home/sini/Documents/repos/den/templates/ci/modules/features/flake-scope-pipeline-args.nix`: all 9 spec tests present.
  - `test-pipeline-only-preserves-attrs` (line 6)
  - `test-pipeline-only-non-attrset` (line 30)
  - `test-aspect-receives-lib` (line 51)
  - `test-aspect-receives-inputs` (line 73)
  - `test-aspect-receives-den` (line 93)
  - `test-class-module-lib-collision-silent` (line 114)
  - `test-optional-lib-arg` (line 136)
  - `test-mixed-collision-policies` (line 160)
  - `test-forward-sub-pipeline-receives-enrichment` (line 199)
  - `test-enrichment-stripping-at-class-boundary` (line 241)
  - (10 tests total — spec listed 9 named; `test-mixed-collision-policies` covers spec's "independent per-arg policies" bullet)

### Sub-pipeline policy propagation
- `/home/sini/Documents/repos/den/nix/lib/aspects/fx/pipeline.nix` lines 279-285: `runSubPipeline` receives `extraState = { aspectPolicies = spec.__aspectPolicies; }` — matches spec's "Forward sub-pipelines receive parent aspectPolicies via extraState" claim.

### Enrichment mechanism
- `/home/sini/Documents/repos/den/nix/lib/aspects/fx/handlers/transition.nix` lines 21-39: `classifyResolve` separates schema transitions from enrichment — non-schema keys (lib, inputs, den) route as enrichment, installed via `scope.provide`.
- `/home/sini/Documents/repos/den/nix/lib/aspects/fx/pipeline.nix` lines 383-438: enrichment keys stripped at class boundary so module system receives native values.

## Current Status
Still exists. Both files remain on `feat/fx-pipeline` HEAD.

## Supersession
None. This spec introduced the `pipelineOnly` utility and `flake-scope` battery as new additions building on the policy context enrichment mechanism (2026-04-28-policy-context-enrichment-design.md).

## Gaps
None identified. All spec proposals shipped:
- `pipelineOnly` utility — shipped
- `den.provides.flake-scope` battery — shipped
- All 9 test cases — shipped (10 present, one additional edge case added)
- Collision check via existing `resolveCollisionPolicy` — confirmed working without pipeline changes

## Drift
Minor: spec describes `resolveCollisionPolicy` as being at "line 48-53 of `aspect.nix`"; actual location is `/home/sini/Documents/repos/den/nix/lib/aspects/fx/aspect.nix` lines 38-54. Functional behavior identical — spec was describing the mechanism accurately, line numbers off due to file path difference (spec said `aspect.nix`, file lives in `fx/aspect.nix`).

Test count: spec listed 8 named tests plus one note about `test-mixed-collision-policies`; implementation has 10 tests total including `test-enrichment-stripping-at-class-boundary` as a distinct case. The additional test covers spec prose ("enrichment-only keys are correctly handled by post-pipeline `wrapCollectedClasses`") — substance matches, just promoted to its own test.
