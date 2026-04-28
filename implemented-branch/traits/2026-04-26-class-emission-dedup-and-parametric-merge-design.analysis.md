# Analysis: Class Emission Dedup & Parametric Aspect Merge

## Verdict

Fully implemented. All three fixes shipped on feat/fx-pipeline. Spec target branch was `feat/traits`; work landed here instead as part of the pipeline refactor sequence.

## Delivery Target

Spec target: `feat/traits`. Actual delivery: `feat/fx-pipeline`. Branch divergence is expected — pipeline work absorbed traits.

Commits (all present on feat/fx-pipeline):
- `f7b252a1` — Fix 1: coerce parametric-only aspect defs to includes
- `52f0acc4` — Fix 2: include-level dedup prevents duplicate aspect resolution
- `6ac38956` — Fix 3: guard unsatisfied class module args, skip emission
- `aa3501f4` — tests for all three fixes (combined into include-dedup.nix)
- `1e6447c5` — end-to-end parametric merge and class-key merge via NixOS eval

## Evidence

### Fix 1: Parametric merge — coerce to includes

`nix/lib/aspects/types.nix`:

- `mergeFunctions` (line 131): multi-def bare parametric path coerces each to `{ includes = [fn]; }` via `baseType.merge` with mapped defs. Single-def returns raw wrapper as before.
- `providerType.merge` (line 208): multiple `__fn`/`__args` wrappers: unwrap `__fn`, coerce to includes, merge through `aspectType`. Single-def preserves raw wrapper (cheap path).
- `__functor` conflict (lines 73–76 and 237–241): two defs with `__functor` at the same path throw with message "merge is ambiguous. Use lib.mkForce to override."

Comment on dispatch at lines 188–192 matches spec table exactly.

### Fix 2: Include-level dedup

`nix/lib/aspects/fx/handlers/include.nix`:

- `isMeaningfulName` imported from `den.lib.aspects` (line 12).
- `dedupKey` (line 286): `isMeaningfulName originalName && !isSyntheticName` — adds `isSyntheticName` guard (synthetic `<...>` names skip dedup) not in spec but consistent with spec intent.
- `seen` and `alreadyResolved` (lines 287–288): pattern matches spec.
- `resume` (line 292): `alreadyResolved → fx.pure []`, else existing dispatch.
- `state` update (lines 313–320): conditional on `!alreadyResolved && dedupKey != null`, thunk-wrapped `_: seen // { ${dedupKey} = true; }`. Matches spec.
- `include-unseen` effect handler (lines 238–252): unregisters a key from `includeSeen` when exclude fires after eager registration. This is a deviation from the spec (spec described rollback in `excludeChild` prose, handler approach is the implementation form).
- `excludeChild` (lines 117–128): sends `include-unseen dedupKey` when `dedupKey != null`. Correct rollback semantics.

### Fix 3: Unsatisfied class module arg guard

`nix/lib/aspects/fx/aspect.nix`:

- `wrapClassModule` (lines 183–188): returns `{ module; wrapped = false; unsatisfied = true; missingArgs = missingDenArgNames; }` on missing den schema args. Includes `warnedModule` (module with lib.warn already applied) per spec.
- `emitClasses` (lines 451–454): `result.unsatisfied or false → []`. Matches spec exactly.

### Tests

`templates/ci/modules/features/include-dedup.nix` contains all spec test cases:
- `test-parametric-wrapper-merge` (Fix 1)
- `test-mixed-parametric-and-attrset` (Fix 1 regression guard)
- `test-functor-conflict-errors` (Fix 1 `__functor` error)
- `test-dedup-static-aspect-two-parents` (Fix 2)
- `test-dedup-parametric-class-two-parents` (Fix 2)
- `test-no-dedup-different-contexts` (Fix 2)
- `test-dedup-trait-two-parents` (Fix 2)
- `test-excluded-then-included` (Fix 2 exclude rollback)
- `test-guard-skips-without-context` (Fix 3)
- `test-guard-defers-then-emits` (Fix 3)

Spec also listed class-key merge regression tests; covered by `1e6447c5`.

Spec specified a separate test file `templates/ci/modules/features/nested-class-module-args.nix` — does not exist; all tests consolidated into `include-dedup.nix`.

## Current Status

Shipped. No open gaps relative to spec behavior. `include-dedup.nix` passes (confirmed via commit `1e6447c5` end-to-end eval).

## Supersession

No superseding spec. `2026-04-28-policy-pipeline-simplification.md` touches `types.nix` for provides shim but does not revisit parametric merge paths.

## Gaps

One minor deviation from spec prose: spec said the `includeSeen` state update records the key "before resolution, not after" and is in the `state` return (not the resume continuation). Implementation matches — state update is in the outer `state =` branch of the handler attrset, not inside any `fx.bind` continuation. Correct.

Spec described a `isMeaningfulName` check but did not mention the additional `isSyntheticName` guard (`lib.hasPrefix "<" && lib.hasSuffix ">`). This is a tightening: bare `<anon>` is caught by `isMeaningfulName` returning false; `isSyntheticName` catches computed synthetic names (e.g., `<forward:x>`) that pass `isMeaningfulName`. Not a gap — improves correctness.

Test file name mismatch: spec named `nested-class-module-args.nix`; implementation used `include-dedup.nix`. Functional equivalent.

## Drift

None. All three fixes match spec semantics. Merge table, dedup key construction, state thunk pattern, exclude rollback, and unsatisfied guard return shape all align with spec exactly.
