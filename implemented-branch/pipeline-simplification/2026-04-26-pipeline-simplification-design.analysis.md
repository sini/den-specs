# Analysis: Pipeline Simplification — Provides Removal + Sub-Pipeline + Fan-Out

## Verdict

Partially delivered. Target 3 (runSubPipeline) and Target 5 (isolateFanOut) are complete and match spec. Target 1 (provides removal) is partially complete — entityIncludes/entityProvides deleted and self-provide inlined, but the pipeline machinery for `provides.X` (emitSelfProvide, emitCrossProvider, structuralKeysSet "provides") still exists, replaced by a compat shim layer rather than deleted outright. The spec's Step 6 (delete provides machinery) and the branch's approach diverge: the branch ships a provides-compat handler and mutual-provider shim instead of removing the machinery.

## Delivery Target

Branch `feat/fx-pipeline`. Spec written 2026-04-26 for `feat/traits` context; executed on fx-pipeline.

## Evidence

### Target 1: Entity Migration + Provides Removal

Steps 1–4 fully executed per commit history:

- `3a15b7b6` — self-provides in resolveEntity, delete context modules (Step 1)
- `7e81ebfc` — framework aspects to policy.aspects, remove entityIncludes writes (Step 2)
- `9a338b50` — delete entityIncludes/entityProvides, schema-based entity guard (Step 3)
- `973ac30b` — remove rootIncludes phase, merge entity includes before transitions (Step 4)

Confirming:
- `nix/nixModule/entities.nix` — deleted; `nixModule/` now has `aspects.nix`, `default.nix`, `lib.nix`, `policies.nix`
- No `entityIncludes`/`entityProvides` references anywhere in `nix/` except one deadbugs fixture
- `rootIncludes` — zero occurrences in `nix/`
- `den.schema.X.includes` used instead of entityIncludes for existence/includes registration

Step 5 (provides deprecation shim) executed: `a01bc53a` adds shim rewriting `provides.X` to direct nesting. However, the shim operates at the type system level only — cross-provide structural keys (`provides.to-users`, named targets) are handled by a separate compat handler.

Steps 5–6 deviated from spec:

- Spec Step 6 says: delete `emitSelfProvide`, `emitCrossProvider`, `emitCross`, `crossProvider`, remove `provides` from `structuralKeysSet`.
- Actual: `emitSelfProvide` still present in `aspect.nix` (line 687), wired at line 760. `emitCrossProvider`/`emitCross` still present in `transition.nix` (lines 101, 288, 355). `"provides"` still in `structuralKeysSet` (line 17).
- Instead: commits `19738452`, `c52a3779`, `2e40b024` added `provides-compat.nix` handler + `emitCrossProvideShims` + mutual-provider compat shim. The compat shim synthesizes aspect policies from `provides.X` keys and emits deprecation warnings.

The result is that provides-path routing still works (via compat handler), not deleted. The `provider/` CI fixture (`templates/ci/provider/modules/den.nix`) still uses `provides.X` patterns and tests the compat path.

### Target 3: Sub-Pipeline Extraction

Fully delivered. Commit `2b2b3cf3` extracted `runSubPipeline` into `pipeline.nix`.

- Defined at `nix/lib/aspects/fx/pipeline.nix` line 242
- Signature matches spec: `{ class, self, ctx, extraState ? {} }` → `{ classImports, traits, provideTo }`
- Three call sites confirmed: `applyForwardSpecs` (forward/pipeline.nix line 212), `resolveFanOut` (transition.nix line 160), `resolveSiblingTransition` (transition.nix line 216)
- Exported as `den.lib.aspects.fx.pipeline.runSubPipeline` (line 371)

Minor spec drift: spec showed `imports` field; actual returns `classImports` (multi-class map) + forwarded specs applied internally. Shape is richer than the spec's simplified view, but semantically equivalent.

### Target 5: Generalize Flake Fan-Out

Fully delivered. Commit `32455189` added `policy.isolateFanOut`.

- `isolateFanOut` computed in transition handler from `resolve.shared` effect: `!(__shared or false)` (line 445–446)
- Guard at line 331: `if isFanOut && ((transition.routing or {}).isolateFanOut or false) then resolveFanOut`
- `targetClass == "flake"` hardcode removed — confirmed zero occurrences in `nix/`
- Commit `73ac2f50` stripped explicit `isolateFanOut` from core policies; it is now derived inline from resolve effects

Spec deviation on policy-types.nix: spec said add `isolateFanOut = lib.mkOption { type = types.bool; ... }` as a declared policy option. Actual: `isolateFanOut` is derived inline from `__shared` flag on resolve effects, never declared as a named policy option. The `policy-types.nix` file has no `isolateFanOut` field. The mechanism is equivalent but the API surface differs — callers use `resolve.shared` (shared fan-out) vs `resolve.to` (isolated fan-out by default), rather than setting `policy.isolateFanOut = true`.

## Current Status

- entityIncludes/entityProvides: **deleted**
- rootIncludes: **deleted**
- self-provide machinery (Steps 1–2): **migrated into resolveEntity**
- emitSelfProvide: **still present** — serves aspects that still use `provides.name` self-provide key
- emitCrossProvider / emitCross: **still present** — provides legacy cross-provide routing for aspects with `provides.X`
- structuralKeysSet "provides": **still present**
- provides-compat handler: **added** — intercepts `provides.X` cross-keys, synthesizes policyFns, emits warnings
- mutual-provider compat shim: **added** — `modules/compat/mutual-provider-shim.nix`
- runSubPipeline: **delivered**
- isolateFanOut: **delivered** (mechanism differs from spec)

## Supersession

Spec Section "Target 1 / Step 6" is superseded by the compat-shim approach. Rather than hard-deleting provides machinery, the branch introduced provides-compat.nix as a migration bridge. The deletion of `emitSelfProvide`, `emitCrossProvider`, and `provides` from structuralKeysSet is deferred to a post-migration cleanup pass.

The `provides-removal` spec referenced in git history (`e865a331`) likely captures this revised approach. The provider CI fixture (`templates/ci/provider/`) was kept and tests the compat path rather than being migrated away.

## Gaps

1. `emitSelfProvide` not deleted — still active for aspects using `provides.${name}` self-provide. Only safe to remove after all CI fixtures verified free of self-provide patterns.
2. `emitCrossProvider` / `emitCross` not deleted — live code path in transition.nix. Active whenever `sourceAspect.provides.${targetKey}` is non-null (i.e. legacy provides.X cross-provide still works via this path in addition to the compat shim).
3. `"provides"` not removed from `structuralKeysSet` — aspect content keys named "provides" still filtered as structural.
4. `templates/ci/provider/modules/den.nix` still uses `provides.X` patterns — not migrated.
5. `policy-types.nix` has no `isolateFanOut` option — spec called for explicit declaration; actual uses derived inline value.

## Drift

| Spec item | Spec intent | Actual |
|-----------|-------------|--------|
| Step 6: delete emitSelfProvide | Hard delete | Still present; deferred |
| Step 6: delete emitCrossProvider | Hard delete | Still present; compat shim added instead |
| Step 6: remove "provides" from structuralKeysSet | Hard delete | Still present |
| Target 5: isolateFanOut in policy-types.nix | Declared mkOption | Derived inline from __shared flag; no option declaration |
| Target 5: set isolateFanOut = true on flake policies | Explicit set | Flake policies use resolve.to (isolated by default); no explicit field |
| runSubPipeline returns { imports, traits, provideTo } | Plain record | Returns { classImports, traits, provideTo }; classImports is multi-class map, not flat imports list |
