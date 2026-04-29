# Analysis: Policy Context Enrichment & Post-Pipeline Class Wrapping

## Verdict
implemented
Both Fix 1 (non-schema resolve routing as enrichment) and Fix 2 (post-pipeline class module wrapping) shipped on `feat/fx-pipeline`. All spec test cases, including the originally-failing flat-form and darwin-branch cases, are present in the test file; the fixed-point enrichment loop and `wrapCollectedClasses` post-pipeline pass both exist in production code. Implementation went beyond the spec in several places (enrichment-only arg stripping, `__functionArgs` leak guard, fully-applied attrset guard, chained-policy tests).

## Delivery Target
feat/fx-pipeline

## Evidence

### Fix 1 — classifyResolve (transition.nix)

`nix/lib/aspects/fx/handlers/transition.nix` lines 21-51:
- `classifyResolve` function present, matching spec pseudocode exactly.
- `schemaKeys`, `enrichKeys`, `hasTarget` branches all implemented.
- Mixed-resolve split (lines 44-51): schema portion retains `filterAttrs schemaKinds`, enrichment portion filters complement.

Fixed-point enrichment loop — lines 692-757:
- `iterateEnrichment` recursive function with `maxEnrichmentIterations = 10` (line 54).
- `scope.provide enrichHandlers` + `drain-deferred` + `drain-dead-letters` in enrichment apply block (lines 724-756).
- Re-dispatch after enrichment until `newEnrichKeys == []` (line 710).

Commit: `a91eee15` "feat: classify resolve effects into schema transitions vs enrichment" (2026-04-28).

### Fix 2 — raw entry emission (aspect.nix)

`nix/lib/aspects/fx/aspect.nix`:
- `emitClasses` (lines 500-558): emits `__rawEntry = true` with `ctx`, `isContextDependent` from parametric/meta flags. `wrapClassModule` NOT called during tree-walk. Matches spec exactly.
- `emitClassFromDLQ` (lines 404-439): also emits `__rawEntry = true` with `ctx` snapshot. Spec's intent ("simplify to emit raw entries") fully reflected.

Commits: `96d9fdc0` (raw emission), `16a6c439` (tree.nix raw storage).

### Fix 2 — classCollectorHandler (tree.nix)

`nix/lib/aspects/fx/handlers/tree.nix` lines 208-246:
- `isRawEntry = param.__rawEntry or false` (line 213).
- Raw path stores full `param // { __loc = loc; }` for post-pipeline pass (lines 228-230).
- Non-raw legacy path builds module location wrapper inline (lines 231-246).
- Note: `classCollectorEntry` was NOT extracted as a standalone shared helper (spec proposed extraction); instead, the wrapping logic was inlined in `wrapCollectedClasses` in pipeline.nix. Functionally equivalent.

### Fix 2 — wrapCollectedClasses (pipeline.nix)

`nix/lib/aspects/fx/pipeline.nix`:
- `wrapCollectedClasses` defined at line 372.
- Called at lines 354 and 572 (single-entity and multi-entity paths).
- Safety net: `result.unsatisfied` → `builtins.trace ... []` (lines 478-481) — matches spec.
- Enrichment merge: line 387 `lib.filterAttrs (k: _: !(entry.ctx ? ${k})) enrichedCtx` — merges only keys absent from entry's own ctx, preserving fan-out isolation.

Additional deviation from spec: lines 395-450 implement enrichment-only arg stripping from advertised `__functionArgs` — prevents NixOS infinite recursion when enrichment keys (e.g. `isNixos`) are not in `_module.args`. Spec did not specify this; it emerged during implementation (fixed in commits `430df4f9`, `62b8c62b`).

Commits: `36341531` (wrapCollectedClasses), `430df4f9` (setFunctionArgs guard), `62b8c62b` (strip unknown args).

### Test file

`templates/ci/modules/features/policy-context-enrichment.nix` (committed per git status):
- All 3 originally-failing spec cases present: `test-parametric-wrapper-defers-for-policy-context`, `test-flat-form-class-defers-for-policy-context`, `test-policy-context-darwin-branch`.
- All 7 "to add" spec cases present: `test-mixed-resolve-split`, `test-enrichment-chained-policies`, `test-enrichment-fan-out`, `test-enrichment-never-provided`, `test-enrichment-multiple-policies`, `test-static-class-module-unchanged`, `test-enrichment-with-traits`.
- One additional regression test not in spec: `test-fully-applied-hm-no-functionargs-leak` (line 377), guarding the setFunctionArgs attrset crash.

Commit: `27b53fc3` "test: add comprehensive policy context enrichment test suite".

## Current Status
Still exists. Both mechanisms are active on `feat/fx-pipeline`. No supersession or rollback.

## Supersession
none

## Gaps
No spec proposals went unimplemented. The `classCollectorEntry` extraction (spec: "helper extracted from classCollectorHandler into a shared function used by both") was not done as a named helper — the logic is inlined at two sites — but this is a code organization choice, not a missing feature. All behavioral requirements shipped.

## Drift

1. **Enrichment arg stripping**: Spec said "enrichment bindings become available in ctx"; implementation adds a post-pipeline pass that strips enrichment-only keys from wrapped module's `__functionArgs` (pipeline.nix lines 395-450). This was necessary to prevent NixOS from probing `_module.args.isNixos` and crashing. Not in spec.

2. **Unwrapped module arg stripping**: `62b8c62b` added stripping of defaulted args that aren't in ctx even for unwrapped modules (pipeline.nix line 442 `argsToStrip` non-wrapped branch). Spec did not address this case.

3. **`classCollectorEntry` not extracted**: Spec proposed a shared helper to construct module location attrsets, used by both `classCollectorHandler` and the post-pipeline pass. Instead, `classCollectorHandler` stores raw params and pipeline.nix has the construction logic inline. Functionally identical.

4. **`drain-dead-letters` in enrichment loop**: Spec mentions DLQ fires after transitions complete. Implementation runs `drain-dead-letters` inside each enrichment iteration (transition.nix line 730), not just once at the end. This is more correct (DLQ entries from newly-resolved deferred includes are immediately drained) but diverges from the ordering description in the spec.

5. **`test-enrichment-with-traits`** comment (line 362) notes that trait arg access via function signature (`{ greeting, ... }:`) is a pre-existing bug tracked separately — the test only validates that enrichment coexists with trait *emission*, not trait *consumption* via function args. Spec test description implied both would work.
