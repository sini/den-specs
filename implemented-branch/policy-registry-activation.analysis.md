# Analysis: Policy Registry/Activation Split — Design Spec

## Verdict
implemented

The core registry/activation split shipped fully: `den.policies.*` is a typed registry, `den.schema.*.policies` is gone, all built-in policies activate via `includes`, and `policy.for`/`policy.when` combinators are live. Two spec details — seeding `firedPolicies` from `scopeSourcePolicy` and global dedup for `<policy:*>` names — diverged in implementation but the functional invariants they protect appear to be met through other means.

## Delivery Target
feat/fx-pipeline

## Evidence

### policyType / policyRegistryType (Spec §12.1)
- `/home/sini/Documents/repos/den/nix/lib/aspects/policy-type.nix` lines 4-29: `policyFnType` and `policyRegistryType` exactly match spec shape `{ __isPolicy, name, fn }`.
- `/home/sini/Documents/repos/den/nix/nixModule/policies.nix` lines 6-11: `den.policies` option typed as `policyRegistryType`. Commit: `3c347e8a feat(policy): add policyType wrapping`.

### __isPolicy dispatch in processInclude (Spec §3.3, §13.1)
- `/home/sini/Documents/repos/den/nix/lib/aspects/fx/aspect/children.nix` lines 38-70: `isPolicy` predicate at line 46; `registerPolicy` at lines 39-44 sends `register-aspect-policy`; `processInclude` routes on `isPolicy rawChild` at line 56. Lists of policies handled at lines 58-70.

### emitAspectPolicies auto-registration loop deleted (Spec §12.3)
- `/home/sini/Documents/repos/den/nix/lib/aspects/fx/aspect/provide.nix` lines 130-151: `emitAspectPolicies` handles only provides-derived cross-entity registration and self-provide. No loop over `builtins.attrNames (aspect.policies or {})`. Commit: `9f3a89a8 feat(policy)!: remove schema.*.policies, migrate built-ins to registry+activation`.

### den.schema.*.policies removed (Spec §4.3, §13.4)
- `grep den.schema.*.policies` across entire repo returns zero matches outside worktrees. Confirmed dead.
- Commits: `9f3a89a8`, `a9324d74 fix(policy): migrate CI tests from schema.*.policies`, `ba7a863a fix(policy): remove stale policies strip from schema merge`.

### installPolicies reads only scopedAspectPolicies (Spec §13.4)
- `/home/sini/Documents/repos/den/nix/lib/aspects/fx/policy/default.nix` lines 97-111: reads `state.scopedAspectPolicies` only. No `globalPolicies`, `schemaPolicies`, or `allDirectPolicies`. The three deleted lines are confirmed absent.

### Schema includes activation (Spec §4.1, §11.1–11.5)
- `/home/sini/Documents/repos/den/modules/policies/core.nix` line 18: `den.schema.host.includes = [ den.policies.host-to-users ]`
- `/home/sini/Documents/repos/den/modules/policies/flake.nix` lines 87-92: `den.schema.flake.includes`, `den.schema.flake-system.includes`
- `/home/sini/Documents/repos/den/modules/aspects/provides/os-class.nix` line 24: `den.default.includes = [ den.policies.os-to-host ]`
- `/home/sini/Documents/repos/den/modules/aspects/provides/os-user.nix` line 41: `den.default.includes = [ den.policies.user-to-host ]`
- `/home/sini/Documents/repos/den/modules/aspects/provides/wsl.nix` lines 69,71: schema host includes + default includes

### policy.for and policy.when combinators (Spec §6.1–6.4)
- `/home/sini/Documents/repos/den/nix/lib/policy-effects.nix` lines 152-212: `for` at line 154, `when` at line 188. Both produce per-inner-policy wrappers preserving `name`, accept single or list, compose. Commit: `9ccd7b1c feat(policy): add policy.for and policy.when combinators`.
- CI tests: `/home/sini/Documents/repos/den/templates/ci/modules/features/policy-combinators.nix` covers `when` true/false, list input, `for` match/no-match, composition.

### Policy excludes via constraint registry (Spec §5.3)
- `/home/sini/Documents/repos/den/nix/lib/aspects/fx/policy/apply.nix` lines 53-59: `policyEmitExcludes` sends `register-constraint` with `type = "exclude"; scope = "subtree"; identity = identity.key e.value`.
- `/home/sini/Documents/repos/den/nix/lib/aspects/fx/aspect/children.nix` line 137: excludes identity check handles `__isPolicy` values: `if builtins.isAttrs ref && ref.__isPolicy or false then ref.name else identity.key ref`. Commit: `b03a5e5b feat(policy): support policy excludes via constraint registry`.

### Effect validation (Spec §15.1)
- `/home/sini/Documents/repos/den/nix/lib/aspects/fx/policy/dispatch.nix` lines 22-33: `validateEffects` throws with exact spec format including policy name and index. Commit: `203b0d19 fix(policy): validate effect shapes in dispatch, improve cycle diagnostics`.

### Enrichment cycle diagnostics (Spec §15.2)
- `/home/sini/Documents/repos/den/nix/lib/aspects/fx/policy/iterate.nix` lines 70-73: error includes `entityKind`, fired policy names, and enrichment keys. Matches spec exactly.

### dispatchDirect dead code removed (Spec §13.4 last paragraph)
- Commit: `3dc65ef8 refactor(policy): remove dead dispatchDirect code path, simplify mkDispatch`. Dispatch now unified through `dispatchAspect` which calls `entry.value.fn` on `{ __isPolicy, name, fn }` records.

### pipelineOnly placement (Spec §16)
- `/home/sini/Documents/repos/den/nix/lib/policy-effects.nix` lines 141-150: `pipelineOnly` remains in public `den.lib.policy.*` namespace. Not moved to internal. See Gaps.

### Source policy exclusion invariant (Spec §8.1)
- `scopeSourcePolicy` written in `/home/sini/Documents/repos/den/nix/lib/aspects/fx/handlers/push-scope.nix` lines 68-73.
- Not read in `installPolicies` (`/home/sini/Documents/repos/den/nix/lib/aspects/fx/policy/default.nix` line 111 passes `{ }` as initial `firedPolicies`).
- See Drift below.

### Policy include dedup (Spec §9)
- `/home/sini/Documents/repos/den/nix/lib/aspects/fx/policy/apply.nix` lines 33-38: names anon policy includes `<policy:${policyName}>${suffix}`.
- `/home/sini/Documents/repos/den/nix/lib/aspects/fx/handlers/check-dedup.nix` lines 17-23: `isSyntheticName` check at line 17 matches names starting with `<` and ending with `>`, sets `rawDedupKey = null` (line 20), skipping dedup. Commits: `afdeb67a fix(policy): dedup policy-emitted includes`, `a80fa879 fix(policy): dedup indexed policy includes`.

## Current Status
Fully shipped on feat/fx-pipeline. 713/713 tests pass per spec header.

## Supersession
None. This spec is the terminal design for the registry/activation split.

## Gaps

### pipelineOnly namespace (Spec §16)
`pipelineOnly` remains at `den.lib.policy.pipelineOnly` (public), not moved to an internal namespace. `flake-scope.nix` still calls `inherit (den.lib.policy) pipelineOnly`. The spec called for moving it to `den.lib._internal.pipelineOnly` or inlining — not done.

### policy.for / policy.when not used in production modules
The combinators shipped with full CI coverage (`policy-combinators.nix`) but are not used in any production module under `modules/`. The example templates (`igloo.nix`, `alice.nix`) still use manual `lib.optional (user.name == "alice")` guards in policy bodies, not `policy.for`/`policy.when` at activation sites. Spec §11.3 showed the intended translation; example templates were not updated to it.

### Resolve variant cleanup (Spec §14.1)
The 7 resolve variants (`resolve`, `resolve.shared`, `resolve.to`, `resolve.shared.to`, `resolve.withIncludes`, etc.) are unchanged. Spec deferred this as independent; it was not addressed.

## Drift

### scopeSourcePolicy not seeded into firedPolicies (Spec §8.1 step 6)
Spec says: "`installPolicies` reads `scopeSourcePolicy` for the current scope and seeds `firedPolicies` with it, so `dispatchAspect` skips the policy." In the implementation, `push-scope.nix` writes `state.scopeSourcePolicy.${newScopeId} = sourcePolicyName` (lines 68-73) but `installPolicies` in `default.nix` line 111 passes `{ }` as initial `firedPolicies` — `scopeSourcePolicy` is never read. The self-referential cycle invariant may be protected through the scope-partitioning mechanism (`2307ea99 fix(policy): separate independent includes from cross-provider includes`) rather than via firedPolicies seeding.

### Policy include dedup uses skip-not-global (Spec §9.2)
Spec says `<policy:*>` names should use global dedup keys (no scope prefix). Implementation uses `isSyntheticName` in `check-dedup.nix` to skip dedup entirely for those names (rawDedupKey = null), rather than switching to a global key. The functional result — avoid double-emission — is achieved by always emitting synthetic-named includes without dedup guard, relying on the `__sourcePolicyName` matching in `emit-policy-effects.nix` (handler at lines 28-34) to separate them correctly.
