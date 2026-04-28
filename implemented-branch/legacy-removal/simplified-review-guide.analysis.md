# Analysis: Simplified Review Guide for feat/rm-legacy (feat/fx-pipeline)

## Verdict

The guide describes an intermediate branch state that has since been substantially superseded by further work on feat/fx-pipeline. Core architecture claims are accurate but several specific implementation details are stale. The "what's left" list is partially complete and partially obsolete. Test claim cannot be independently verified from source.

## Delivery Target

feat/fx-pipeline (formerly described as feat/rm-legacy). The guide was written for an earlier snapshot of the branch; the branch has advanced significantly (30+ additional commits visible in `git log`).

## Evidence

Key commits since the guide was likely written:
- `1c077a06` — forward as post-processing on pipeline results (forward-as-handler landed)
- `30ba836e` — delete policy-dispatch.nix, remove dead types, simplify den.policies option
- `29156fe0` — remove policy activation mechanism
- `7e26712e` — replace *-to-default policies with den.default include injection
- `73ac2f50` — strip `_core` and `isolateFanOut` from core policies, use resolve.shared
- `8b999f35` — add policy.resolve.shared effect variant for shared fan-out
- `de80246b` — convert all policies to plain functions, remove __functor wrappers
- `269ee2c8` — remove `to` field from policies, use resolve.to effect
- `4d06b4be` — remove `from` field from all policies, inject `__entityKind`
- `2e40b024` — add mutual-provider compat shim
- `f5282ac8` — convert template policies to plain functions

## Current Status

### Architecture — matches:
- `aspectToEffect` exists in `/nix/lib/aspects/fx/aspect.nix` with depth guard, `mkParametricNext`, `tagParametricResult`
- `constantHandler`, `ctxSeenHandler` in `handlers/ctx.nix`
- `classCollectorHandler`, `constraintRegistryHandler`, `chainHandler`, `deferredIncludeHandler`, `drainDeferredHandler` all in `handlers/tree.nix`
- `includeHandler` in `handlers/include.nix`
- `transitionHandler` in `handlers/transition.nix`
- `collectPathsHandler` / `pathSetHandler` in `identity.nix`
- `__scopeHandlers` propagation pattern intact
- `scope.provide` used in `aspect.nix:915` for parametric resolution
- `composeHandlers` in `pipeline.nix`
- `emit-forward` effect handler exists at `handlers/forward.nix` (forward-as-handler shipped)

### Architecture — stale/superseded:

**Policy shape**: The guide's policy example using `{ from, to, resolve, handlers? }` struct is gone. Policies are now plain functions with `__entityKind` body guards and typed `policy.resolve.*` effect constructors. `policyType` with `handlers` field was deleted (`nix/lib/policy-types.nix` is now 9 lines, just `policyFnArgs`). No `from`/`to` fields.

**resolve-policy / resolve-target effects**: Not present in current `pipeline.nix`. These bootstrap effects described in the guide were removed (`12fed6aa` — "remove dead synthesize/mergePolicyInto"). `synthesize-policies.nix` now contains only `resolveArgsSatisfied`. No `resolvePolicyHandler` or `resolveTargetHandler` in `defaultHandlers`.

**den.stages**: Fully deleted. `modules/removed-stages.nix` installs a tombstone option that throws a migration error. `resolve-stage.nix` and `stage-types.nix` are absent from `nix/lib/`. The four-concern model diagram in the guide still references `den.stages` as a live concern — this is outdated.

**Per-policy named effect handlers (Layer 2)**: The `handlers` field on policies (with `scope.provide` per-transition) was removed. `coreEffects` filter does not exist. Policy dispatch is now direct iteration (`allPolicies = den.policies or { }`) without scoped handler installation.

**Policy fan-out isolation**: The guide says "current fix is flake-class-scoped". Now resolved generically via `resolve.shared` vs default isolated fan-out. `isolateFanOut` in `transition.nix:445` is derived from `__shared` on the resolve effect — any policy using `resolve.shared` opts into shared (non-isolated) fan-out.

**`den.schema.<kind>.policies`**: Guide lists this as item 3 of "what's left". Actually already implemented in microvm template (`den.schema.host.policies = [...]`). Not confirmed in schema option definition (schemaEntryType in `modules/options.nix` does not explicitly declare a `policies` field — it uses freeform lazyAttrsOf, so it is accepted but may not be formally typed).

**ctx-shim**: Present at `modules/compat/ctx-shim.nix` as described, forwarding `den.ctx.*` to `den.schema.*.includes`. Matches spec.

**mutual-provider-shim**: Added since guide was written — `modules/compat/mutual-provider-shim.nix` and `handlers/provides-compat.nix`. Not mentioned in guide.

**Key files table**: `synthesize-policies.nix` and `resolve-stage.nix` and `stage-types.nix` listed in guide's key files table are either vestigial or gone. `forward.nix` in `nix/lib/` is still present but `handlers/forward.nix` is new and does the real work. `policy-types.nix` is now near-empty. `modules/policies/` no longer has a `batteries.nix`.

## Supersession

The following spec claims are fully superseded:

1. **Per-policy `handlers` + `scope.provide` (Layer 2)** — removed entirely. No policy in the codebase has a `handlers` field. Context is provided via `constantHandler` and `__scopeHandlers` only.
2. **`resolve-policy` / `resolve-target` effectful bootstrap** — removed. Pipeline no longer emits these effects at startup.
3. **`den.stages` as live concern** — tombstoned via `removed-stages.nix`. The four-concern model should drop stages.
4. **`synthesize + mergePolicyInto`** — dead code removed (`12fed6aa`).
5. **Policy struct `{ from, to, resolve, handlers? }`** — replaced by plain function shape with `__entityKind` guard and `policy.resolve.*` typed effects.

## Gaps

1. **`den.schema.<kind>.policies` formal typing**: Used in microvm template but `schemaEntryType` in `modules/options.nix` has no explicit `policies` option — it falls through freeform. May work but is unvalidated. The guide listed this as "what's left" — it is partially addressed but not formally typed.
2. **Vestigial files**: `nix/lib/parametric.nix` and `nix/lib/take.nix` still present (referenced from `nix/lib/default.nix`). Guide noted these as "blocked on downstream migration".
3. **`perHost-perUser.nix`**: Check presence — not verified in this analysis but noted in guide's vestigial list.
4. **Test counts**: Guide claims 483/483 CI + 15/15 checkmate. Cannot verify from source alone — requires running `just ci`.

## Drift

| Guide claim | Current reality |
|---|---|
| Policies: `{ from, to, resolve, handlers? }` struct | Plain functions with `__entityKind` body guard + typed effects |
| `from`/`to` fields on policy | Removed; `__entityKind` injected; `resolve.to "kind"` effect used |
| Per-policy `handlers` field + `scope.provide` per-transition | Deleted; no such mechanism exists |
| `coreEffects` filter protecting core effects | Does not exist |
| `resolve-policy` / `resolve-target` bootstrap effects | Removed |
| `synthesize-policies.nix` owns synthesize + mergePolicyInto | File now only has `resolveArgsSatisfied` |
| `resolve-stage.nix` / `stage-types.nix` in lib | Deleted |
| `den.stages` as live config option | Tombstoned with migration error |
| `forward-as-handler` is "what's left" | Shipped — `handlers/forward.nix` + `emit-forward` effect |
| Fan-out isolation "current fix is flake-class-scoped" | Generalized via `resolve.shared` |
| `modules/policies/batteries.nix` | Does not exist; only `core.nix` and `flake.nix` |
| ctx-shim forwards to `den.stages.*` | Actually forwards to `den.schema.*.includes` |
| PR vic/den#475 open | Status unknown from source |
