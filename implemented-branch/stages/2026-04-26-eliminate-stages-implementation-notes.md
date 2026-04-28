# Eliminate Stages — Implementation Notes

**Date:** 2026-04-26
**Branch:** feat/traits
**Status:** Implemented (pending entityProvides cleanup and code review)

## What Was Done

Implemented the three-layer aspect model from `2026-04-25-eliminate-stages-design.md`:

### Infrastructure Changes

1. **`den.stages` deleted** — option, `resolve-stage.nix`, `stage-types.nix` all removed
2. **`den.entityIncludes`** — per-entity-kind include lists, replacing `den.stages.*.includes` and self-provides
3. **`rootIncludes` pipeline phase** — new phase in `resolveChildren` (aspect.nix) between self-provides and transitions. Required because self-provides resolved before transitions, enabling deferred include drain during context widening. Moving entity root aspects to regular includes broke 9 tests (deferred drain, chain ordering). rootIncludes preserves the ordering.
4. **`resolveEntity`** — sole resolution path, reads from `entityIncludes` only
5. **`resolve-entity` effect** — replaced `resolve-target` in transition handler
6. **Policy `aspects` field** — added to policy type, propagated in routing metadata, injected into target includes during transition resolution
7. **Per-aspect-set merge keys** — transitions with different aspect sets don't merge
8. **ctx-seen accumulation** — tracks aspects per key, emits supplemental aspects on revisit
9. **Trait-aware policy dispatch** — accumulated traits merged into policy resolve context

### Migration: makeHomeEnv

Two-hop chain (`host → hm-host → hm-user`) collapsed to single policy (`host → user` with `aspects = ["hm-host-module", "hm-user-forward"]`). Intermediate entities eliminated. `userEnvAspect` eliminated. Applied to all three batteries (home-manager, hjem, maid).

Key fix: supplemental policy aspects emitted via `emit-include` needed `__parentScopeHandlers` to resolve parametric args. Without scope context, aspects like `hm-user-forward` (needing `{ host, user }`) would defer forever.

### Migration: Flake Forwarders

`SystemOutputFwd`, `osFwd`, `hmFwd` registered as `den.aspects` with flat parametric signatures. The curried form (`{ system, output }: { class, aspect-chain }: ...`) was flattened to `{ system, output, class, aspect-chain }: ...` — the effectful pipeline resolves all named args from scope handlers regardless of currying.

Flake policies use `aspects` field for injection. Empty `entityIncludes` entries register entity kinds (flake-system, flake-packages, etc.) so `resolveEntityHandler` finds them.

### Migration: synthesize-policies

`entityKeyFor` suffix matching removed. `ctxSatisfies` simplified to check schema entity kinds directly. Flake-scope heuristic retained (non-entity contexts).

### Test Impact

~45 test files updated. All `den.stages` references converted to `den.entityIncludes`/`den.entityProvides`. All `resolveStage` calls converted to `resolveEntity`. 4 `parametric-fixedTo` host-context tests updated — rootIncludes correctly drain deferred parametric includes during host→user transitions, producing additional user-context entries.

## Deviations from Spec

1. **`den.entityIncludes` instead of `den.schema.<kind>.includes`** — the spec envisioned entity includes on the schema deferred module. But deferred modules can't be read by `resolveEntity` (it runs outside entity evaluation). `den.entityIncludes` is the practical equivalent — a global per-kind include list that the module system merges.

2. **`den.entityProvides` retained (transitional)** — cross-provides for user-defined test patterns. Being cleaned up in a follow-up commit.

3. **No sibling routes for flake** — spec suggested sibling routes (`from = to = "flake-system"`) for per-output policies. Kept non-sibling routes since sibling resolution runs `resolveSiblingTransition` (trait collection path) which doesn't support aspect injection. Non-sibling routes work correctly with the flat function approach.

4. **`rootIncludes` not in spec** — the spec assumed entity root aspects could be regular includes. The pipeline's ordering (self-provide → transitions → includes) means regular includes resolve after transitions, breaking deferred drain. `rootIncludes` is a minimal pipeline addition that preserves the ordering contract.

## Outstanding Items

- **Compat shim (`ctx-shim.nix`)** — still exists, forwards `den.ctx.*` to `den.entityIncludes`. `den.ctx.*.into` no longer works (stages gone). Could be removed if no downstream users.
- **`emitCrossProvider` in transition.nix** — dead code path. `provides` on resolved entities is always `{}`, `entityProvides` usage removed from all tests. The cross-provider mechanism in `resolveTransition` can be simplified.
- **`provides` deprecation trace** — still fires for user aspects with `provides.X`. This is the traits spec's concern (direct nesting migration), not this spec.
- **Doc string examples** — `mutual-provider.nix` and `import-tree.nix` still reference `den.stages` in description strings (not evaluated code).

## Resolved (from code review)

- **`__ctxStage` added to structuralKeysSet** — eliminates unregistered key warnings.
- **`den.entityProvides` cleaned up** — test cross-provides converted to entityIncludes.
- **Template migrations** — all 7 non-CI templates migrated from den.stages to den.entityIncludes.
- **Stale comments** — resolveStage references updated in diag/default.nix.
- **Redundant mkDetectHost check** — simplified `enabled != false && enabled != []` to `if enabled`.
