# Analysis: Policy Pipeline Simplification

## Verdict

**FULLY IMPLEMENTED** — all phases A–E shipped on `feat/fx-pipeline`. The spec's delivery target was "Phases A–D shipped, Phase E deferred". Reality: Phase E shipped too, including all seven deferred steps (16–22). The spec's own tracking section reflects only partial delivery; the codebase is ahead of spec's recorded status.

## Delivery Target

Spec recorded as: Phases A–D shipped, Phase E deferred pending pipeline architecture work.

Actual state: Phases A–E fully complete. Every step in the spec (0–22) has a corresponding commit on `feat/fx-pipeline`. The spec's Phase E section describes steps as future work, but all have shipped.

## Evidence

**Metadata fields removed:**

- `_core` — stripped from core policies (`73ac2f50`), flake policies (`82864411`), all test fixtures (`424888b7`). Zero occurrences in `modules/policies/` or `templates/ci/modules/features/`.
- `isolateFanOut` — stripped from policy definitions (`73ac2f50`). Now computed internally in `transition.nix` from resolve effect `__shared` flag. No policy declares it.
- `from` — removed from all policies (`4d06b4be`). Replaced by `__entityKind` body guards (`{ __entityKind ? null, host, ... }: if __entityKind != "host" then [] else ...`).
- `to` — removed from all policies (`269ee2c8`). Replaced by `resolve.to "kind" { ... }` effect constructor in `policy-effects.nix`.
- `__functor` wrappers — removed from all policies (`de80246b`). All policies are now plain functions.

**Dispatch infrastructure deleted:**

- `policy-dispatch.nix` — deleted (`30ba636e`). Confirmed absent: no file at `nix/lib/aspects/fx/handlers/policy-dispatch.nix`. Exists only in the unrelated `docs-fixes` worktree (branched from main).
- `collectPolicyHandlers` — removed (`72b34735`). Zero occurrences in `nix/` or `modules/`.
- `policyTraceHandlers` — removed (`72b34735`). Zero occurrences.
- `compilePolicyHandlers`, `policyEffectNamesFor`, `activePoliciesFor`, `ctxSatisfies` — all absent from codebase.
- `dualPolicyType`, `isOldStylePolicy` — absent. `policy-types.nix` now contains only `policyFnArgs = policy: lib.functionArgs policy` (9 lines total).
- `entity-policies.nix` — deleted. Absent from `modules/context/`.

**Activation mechanism removed (`29156fe0`):** `den.default.policies`, `den.schema.host.policies`, `entity.policies` option all gone.

**Phase E steps — shipped:**

- Step 16 multi-class collection: `classCollectorHandler` collects into `classImports = { nixos = [...]; homeManager = [...]; }` buckets (`4c58d6d6`).
- Step 17 forward as post-processing: `forward.nix` is now a post-processing step on pipeline results, not a sub-pipeline (`1c077a06`).
- Step 18 `*-to-default` replacement: replaced with `den.default` schema include injection (`7e26712e`).
- Step 19 `from` removal: done (`4d06b4be`). Body guards (`__entityKind`) handle scoping.
- Step 20 `to` removal: done (`269ee2c8`). `resolve.to "kind"` carries `__targetKind`.
- Step 21 `__functor` removal: done (`de80246b`). All policies are plain functions.
- Step 22 class keys as lists: done (`2ccdd1ad`). `emitClasses` coerces to list, enables mixed-scope aspects.

**Template policy conversion:** `f5282ac8` converted all remaining template fixture policies to plain functions.

**Current policy shape** (confirmed from `modules/policies/core.nix`, `modules/policies/flake.nix`):

```nix
# core.nix — plain function, body guard for scoping
host-to-users = { __entityKind ? null, host, ... }:
  if __entityKind != "host" then []
  else map (user: resolve.shared { inherit user; }) (lib.attrValues host.users);

# flake.nix — plain function, resolve.to for explicit target kind
flake-to-flake-system = { __entityKind ? null, ... }:
  if __entityKind != "flake" then []
  else map (system: resolve.to "flake-system" { inherit system; }) den.systems;
```

**hasPolicies simplification:** `aspect.nix:585` now reads `hasPolicies = isEntityRoot && (den.policies or {}) != {}`. No per-policy `from == entityKind` check needed — body guards handle scoping within the policy function itself.

**`synthesize-policies.nix`:** Reduced to `resolveArgsSatisfied` only (function arg matching). `ctxSatisfies` is gone.

## Current Status

All spec steps complete. `policy-dispatch.nix` deleted. `policy-types.nix` is 9 lines. `synthesize-policies.nix` is 29 lines. `transition.nix` dispatches policies by direct iteration with `resolveArgsSatisfied` gating. No activation model. No named policy effects. No dual dispatch.

`policy-inspect.nix` retains `from`/`to` as output fields (inspection result shape, not policy metadata) — this is correct per spec.

## Supersession

This spec supersedes:

- `2026-04-27-unified-policy-effects-design.md` — effects model was prerequisite; fully consumed.
- Implicit supersession of the activation model from `den.schema.host.policies` and `entity.policies` options.

The spec itself is superseded in scope by Phase E shipping earlier than predicted. The "Dependency Order" section's Phase E block is now historical record, not a roadmap.

## Gaps

**Spec-recorded open items, still deferred:**

1. `emitCrossProvider` in `transition.nix` (line 101) — still present. Spec deferred this to provides-removal plan. Confirmed: `emitSelfProvide` also still in `aspect.nix`. Provides removal is tracked separately (`2026-04-27-provides-removal-post-unified-effects.md`).
2. `into`/`intoFn` — still used by `ctx-shim.nix` compat and entity definitions. Spec noted "kept if used by ctx-shim + tests". Still in use; correct to retain.
3. `host-aspects.nix` — still present at `modules/aspects/provides/host-aspects.nix`. Spec deferred to provides removal plan.
4. `policy.exclude` rollback (include-unseen coverage) — spec flagged as unverified. No evidence of explicit rollback implementation added; risk noted but unresolved.
5. Wildcard policy scoping (Option 4, schema-based scope derivation) — spec chose this option but noted it as future implementation work. Current code uses `__entityKind` body guards instead (a pragmatic Option 2 variant). Option 4's lazy pre-analysis or speculative-run approach is not implemented.
6. `policy-inspect.nix` uses `ctxSatisfies` comment mentions it was "kept" for scope filtering — confirmed: `policy-inspect.nix` does NOT use `ctxSatisfies` (removed). It uses `resolveArgsSatisfied` directly. No gap.

## Drift

**Spec vs implementation deviations:**

1. **`hasPolicies` check** — spec Step 7 noted "hasPolicies must check from == entityKind per-policy to avoid user→default→user recursion". Implementation chose body guards inside policies instead, simplifying `hasPolicies` to `den.policies != {}`. The `*-to-default` infinite recursion risk is eliminated differently: `*-to-default` policies were removed entirely (Step 18), and `den.default` is injected as a schema include. The hasPolicies + body guard approach is cleaner than per-policy filtering in `hasPolicies`.

2. **Wildcard scoping** — spec recommended Option 4 (schema-based scope derivation from resolve binding keys). Implemented as Option 2 variant (`__entityKind` body guards). Every policy on the branch uses explicit `__entityKind` guards rather than wildcard `_:` signatures. This is simpler and explicit; Option 4 remains a future improvement if wildcard policies become needed.

3. **`from` still appears inside transition.nix routing struct** — `routing.from` and `routing.to` are internal transition routing metadata computed by the dispatch loop, not policy-declared metadata. Spec's removal target was policy-level fields; internal routing struct fields are implementation detail and not in scope.

4. **`resolve.to` effect constructor** — spec described flake policies converting to `{ system, ... }:` with binding key derivation from schema. Implementation uses explicit `resolve.to "flake-os" { inherit host; }` carrying `__targetKind`. This is the approach spec labeled for Step 20; it shipped.

5. **Compat shims added post-spec** — commits `19738452`, `c52a3779`, `2e40b024` added `provides-compat` handler and mutual-provider shim. These are backward-compatibility infrastructure not mentioned in the spec. They support main-era `provides`-based aspects on the new pipeline. Not a spec gap — out of scope.
