# Analysis: Separating Data, Policies, and Behavior (ctx-as-classes)

## Verdict
Substantially implemented — core separation delivered, with intentional deviations from spec and one planned-but-deferred item (provides removal).

## Delivery Target
Branch: `feat/fx-pipeline` (not `feat/rm-legacy` as the spec header states — the work landed here).

## Evidence

**Key commits on feat/fx-pipeline:**
- `4819c8ad` plan: pipeline simplification — provides removal, sub-pipeline, fan-out
- `4a01bc53` feat: provides deprecation shim, rewrite to direct nesting
- `9a338b50` feat: delete entityIncludes/entityProvides, schema-based entity guard
- `db189e17` refactor: rename stage → entityKind, derive entity kind in trace
- `e9ea41b0` docs: update stale den.stages references in docstrings and comments
- `d8b538f8` fix: add den.stages removal shim to prevent infinite recursion
- `7e26712e` refactor: replace *-to-default policies with den.default include injection
- `4d06b4be` refactor: remove from field from all policies, inject __entityKind
- `269ee2c8` refactor: remove to field from policies, use resolve.to effect

**den.ctx status:**
- `den.ctx` is NOT fully removed. It survives as a compat shim at `modules/compat/ctx-shim.nix`.
- The shim accepts `den.ctx.X` and forwards entries to `den.schema.X.includes` with a deprecation warning.
- `den.ctx.*.into` slot exists in the shim but is declared `raw` with no forwarding (stub-only).
- Four files reference `den.ctx` in the codebase: the shim, nested-ctx test fixture, doc-examples fixture, and ctx-compat test fixture.
- No non-compat production code writes `den.ctx` directly.

**den.stages status:**
- `den.stages` is fully removed and guarded. `modules/removed-stages.nix` declares it as a tombstone option that throws on use, directing users to `den.schema.<kind>.includes`.
- Three files mention `den.stages` (fx-coverage, doc-examples, removed-stages.nix itself) — all are shim/test/docs contexts.

**Policies:**
- `den.policies` implemented as plain functions (not `{from,to,resolve}` submodules as the spec proposed).
- Policy shape: `{ __entityKind ? null, ... }: [effects]` — entity kind guard is injected by pipeline, not declared as a `from` field.
- `from` and `to` fields were removed (`4d06b4be`, `269ee2c8`). Routing uses `resolve.to "kind"` effect and `__entityKind` injection.
- `den.lib.policy.resolve`, `.include`, `.exclude` effect constructors are in `policy-effects.nix`.
- `nix/lib/policies/` directory does NOT exist — lib-level policy types live in `policy-types.nix`, `policy-effects.nix`, `policy-inspect.nix` at flat lib level.
- `modules/policies/core.nix` and `modules/policies/flake.nix` hold module-level policy declarations.

**Entity types (schema):**
- `nix/lib/entities/` directory EXISTS with `host.nix`, `home.nix`, `_types.nix` — partial file reorganization delivered.
- `nix/lib/types.nix` still exists alongside (re-exports or legacy path).
- Entity ctx nodes have no `__functor` in policy context — policies are plain functions.

**makeHomeEnv:**
- Factory survives but generates `{battery = {policies."host-to-X-users" = policyFn;}, hostConf = ...}` — policy+aspect pairs, not ctx nodes. Matches spec intent.
- `den.provides.forward` is still used inside `makeHomeEnv` for cross-entity forwarding (provides not yet fully removed).

**ctxApply:**
- `ctx-apply.nix` is gone. Scope binding now lives in the fx pipeline handler (`nix/lib/aspects/fx/handlers/ctx.nix` — `constantHandler` builds per-context handlers from the ctx attrset).

**perHost-perUser.nix:**
- Still present at `modules/context/perHost-perUser.nix` — spec called for deletion, not yet done.

**den.stages (scoped behavior binding):**
- Spec proposed `den.stages` as the binding point between topology and behavior.
- Implementation chose a different path: `den.schema.<kind>.includes` replaces `den.stages.<kind>.includes` directly. The binding point concept was absorbed into the schema system rather than a separate `stages` namespace.
- This is a spec deviation — the four-concern model collapsed "stages" into "schema includes." The `den.stages` name was tombstoned, not redirected to a new home.

**default routing:**
- Spec: `*-to-default` transitions are opt-in explicit policies.
- Implementation: `*-to-default` policies were eliminated entirely in favor of `den.default` inject as schema include (`7e26712e`). Functionally equivalent but simpler — no policy needed at all.

## Current Status

| Spec item | Status |
|-----------|--------|
| `den.ctx` removal | Partial — compat shim active, `into` stub only |
| `den.stages` introduction | Deviated — replaced by `den.schema.<kind>.includes` tombstone |
| `den.policies` as plain functions | Delivered (no from/to fields, __entityKind injection) |
| Policy effect constructors (resolve/include/exclude) | Delivered |
| `ctxApply` → pipeline handler | Delivered (`constantHandler` in ctx.nix) |
| `makeHomeEnv` → policy+aspect pairs | Delivered |
| Entity type files into `nix/lib/entities/` | Partial (host, home, _types present; user not confirmed; types.nix still exists) |
| `modules/policies/` directory | Delivered (core.nix, flake.nix) |
| `perHost-perUser.nix` deletion | Not done |
| `provides` cross-entity forwarding cleanup | In progress (provides-compat shim, deprecation path active) |
| Namespace-nested policies/stages | Not implemented (denful future work) |

## Supersession

This spec defined `den.stages` as the fourth concern (scoped behavior binding). The implementation superseded that with a simpler model: `den.schema.<kind>.includes` absorbs the scoped-binding role. The `den.stages` namespace was never populated — it was immediately tombstoned with a migration message pointing to schema includes. The three-way separation (policies / stages / aspects) became two-way (policies inject aspects via schema includes). This is documented in `docs/superpowers/specs/2026-04-25-eliminate-stages-design.md`.

## Gaps

1. `den.ctx.*.into` forwarding is a stub in the compat shim — users who wrote custom `den.ctx.host.into.my-stage` policies get no migration path, only silence (the `into` option exists but does nothing).
2. `modules/context/perHost-perUser.nix` was not deleted per spec.
3. `nix/lib/types.nix` still exists alongside the `entities/` directory — the split is incomplete.
4. Namespace-nested policies (`den.ns.desktop.policies.*`) not implemented — denful batteries use flat `den.policies` with naming conventions instead.
5. `den.schema.<kind>.policies` activation levels (the four-tier Haskell typeclass analogy from the spec appendix) were not implemented as described. Entity-level policy overrides (level 4) are not wired.

## Drift

- **Policy shape:** Spec proposed `{from, to, resolve}` submodule. Landed as plain functions with `__entityKind` injection — simpler, more direct, but syntactically different from the spec examples.
- **stages vs schema.includes:** The spec's `den.stages` concept was eliminated before it was built. The "binding point" insight was valid but the implementation found `den.schema.<kind>.includes` sufficient without a separate namespace.
- **`*-to-default` elimination:** Spec said opt-in explicit policies. Implementation removed the concept entirely — `den.default` is injected as a schema include, not a policy target. Cleaner than the spec proposed.
- **`from`/`to` fields:** Spec's policy syntax had explicit `from` and `to` fields. These were added then removed; routing is now via `__entityKind` body guard and `resolve.to` effect. More composable but less declarative than the spec syntax.
