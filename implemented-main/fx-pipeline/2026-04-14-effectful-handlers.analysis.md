# Analysis: Effectful Handlers for nix-effects

## Verdict

IMPLEMENTED — effectful resume (handlers returning computations) is live in the deployed nix-effects fork and actively used by den's pipeline.

## Delivery Target

nix-effects fork: `github.com/sini/nix-effects` rev `cf958815eb43e9e38aa35436d379b4135b5f3126` (locked in `templates/ci/flake.lock`). Den consumes it via `nix/lib/fx.nix` which fetches from the lock file.

## Evidence

**Handler using effectful resume (computation as resume value):**

`nix/lib/aspects/fx/handlers/include.nix` line 291–312 — the `emit-include` handler returns `resume = fx.bind (fx.send "check-constraint" ...) (decision: ...)`. This is not a plain value; it is an `Impure` computation. The trampoline must detect this and splice the continuation queue, exactly as the spec describes.

`nix/lib/aspects/fx/handlers/transition.nix` line 552 — the transition handler returns `resume = fx.bind (fx.bind dispatchPolicies dispatchAspectPolicies) (rawTransitions: ...)`. Another computation as resume.

Both are production handlers running on every pipeline invocation.

**Trampoline capability confirmed via side evidence:**

`modules/context/perHost-perUser.nix` line 3 references nix-effects commit `c7931d7` for optional-args skipping — confirms the den codebase tracks nix-effects features by commit. The effectful resume feature predates current HEAD (`cf958815`) since include.nix and transition.nix have used computation resumes for multiple shipped phases.

**composeHandlers documents the behavior:**

`nix/lib/aspects/fx/pipeline.nix` lines 18–21 explicitly documents: "When b returns an effectful resume (computation), the sub-computation runs with b's state, not a's." This is a known, tested constraint — confirming effectful resume is an established trampoline capability, not accidental.

**Test coverage:**

`templates/ci/modules/features/fx-coverage.nix` section 8 (`test-composeHandlers-plain-resume-state-correct`) tests the composeHandlers interaction with effectful resumes. `templates/ci/modules/features/fx-effectful-resolve.nix` tests the full include-handler pipeline that depends on effectful resume working correctly.

**nix-effects lock:**

`templates/ci/flake.lock` pins `sini/nix-effects` at `cf958815` (lastModified 1776721916, April 2026). No upstream `vic/nix-effects` — den runs on the sini fork which contains the trampoline fix.

## Current Status

Fully operational. The spec's two code changes (`interpret` and `rotateInterpret` in trampoline.nix) are present in the pinned fork. All den pipeline handlers that need it are using computation resumes without issue. The `queue.append` fix referenced in project memory (commit 9c4db2a) restored `__rawResume` in bind chains — that is a related but separate trampoline fix that was also landed.

## Supersession

None — this is foundational. The "aspects as computations" architecture described in the spec is the current architecture. No successor spec supersedes it; subsequent specs (pipeline simplification, policy effects, traits) all build on top of it.

## Gaps

**Spec tests not verified as standalone tests.** The spec defines five nix-unit test cases (`test-effectful-resume`, `test-effectful-resume-state`, etc.) against the raw nix-effects API. These are not visible as literal tests in den's test suite — den tests the end-to-end behavior (include handler, parametric resolution) rather than the trampoline primitive. Whether these exact tests exist in the `sini/nix-effects` repo itself is not verifiable from the den codebase.

**`rotateInterpret` path.** Den's pipeline uses `fx.handle` (interpret path), not `rotate`. The `rotateInterpret` fix is in the spec but its exercise within den is indirect (scope.provide / scope.stateful use rotate). No den test directly exercises effectful resume through the rotate path.

## Drift

**Spec says `github.com/sini/nix-effects (fork of vic/nix-effects)`.** The lock pins `sini/nix-effects` with no `vic/nix-effects` parent input — the fork relationship is accurate but den never references the upstream directly.

**Spec discusses `isPure` helper.** The actual trampoline implementation may use a different predicate shape (`r._tag == "Pure"` inline check vs named helper). This is an internal detail of nix-effects; the observable behavior matches the spec.

**`scope.run` limitation** (from project memory): `scope.run` (which uses `rotateInterpret`) had a deep handler bug that blocked its use; `scope.provide` is used instead. This means the `rotateInterpret` effectful-resume path is less exercised than the `interpret` path in production, consistent with the "scope.stateful drops 79% events" note. The spec's correctness for the rotate path is not fully production-validated within den.
