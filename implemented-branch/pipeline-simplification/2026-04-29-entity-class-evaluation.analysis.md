# Analysis: Entity-Level Class Evaluation — Eliminating Forwards

## Verdict
implemented
The core of this spec shipped in full on the feat/fx-pipeline branch: `policy.instantiate` effect constructor, handler, and post-pipeline resolution were all added; `osConfigurations.nix`, `hmConfigurations.nix`, and `flakeSystemOutputs.nix` were deleted; `flake-os`, `flake-hm`, and all `flake-<output>` schema kinds were removed. Schema-scoped policies via `den.schema.<kind>.policies` landed as specified. The Tier 2 adapter forward path was not deleted but was unified into routes rather than kept as a separate mechanism.

## Delivery Target
feat/fx-pipeline

## Evidence

**`policy.instantiate` effect constructor** — added in commit c2ccb5d6, present at:
- `/home/sini/Documents/repos/den/nix/lib/policy-effects.nix` lines 76-79

**`register-instantiate` handler** — added in c2ccb5d6:
- `/home/sini/Documents/repos/den/nix/lib/aspects/fx/handlers/instantiate.nix` lines 7-16
- `scopedInstantiates` state field: `/home/sini/Documents/repos/den/nix/lib/aspects/fx/pipeline.nix` line 149

**Post-pipeline `applyInstantiates`** — full implementation at:
- `/home/sini/Documents/repos/den/nix/lib/aspects/fx/resolve.nix` lines 238-348
- Collects all instantiate specs, locates host scope by `findHostScopeId` (lines 170-192), runs subtree assembly phases, calls `spec.instantiate instantiateArgs`, places result via `lib.setAttrByPath (["flake"] ++ spec.intoAttr) evaluated` (line 342), appends to `flake` classImports (line 347)

**`osConfigurations.nix`, `hmConfigurations.nix`, `flakeSystemOutputs.nix` deleted** — commit a6bd47c8 (2026-04-29). `modules/outputs/` now contains only `systems.nix`.

**Flake policies converted** — `/home/sini/Documents/repos/den/modules/policies/flake.nix`:
- `to-os-outputs` (lines 54-65): emits `resolve.to "host"` + `den.lib.policy.instantiate host` per host
- `to-hm-outputs` (lines 67-78): same pattern for homes
- `mkOutputPolicy` (lines 23-37): uses `den.lib.policy.route` for packages/apps/checks/devShells/legacyPackages
- No `__entityKind` guards anywhere in the file

**Schema-scoped policy dispatch** — commit ebc0c75d:
- `modules/options.nix`: `schemaEntryType` extracts `policies` from defs alongside `includes`, stores as `policies = allPolicies` in merged result
- `/home/sini/Documents/repos/den/nix/lib/aspects/fx/handlers/dispatch-policies.nix` → transition.nix diff shows: `schemaPolicies = (den.schema.${sourceEntityKind} or {}).policies or {}; allPolicies = (den.policies or {}) // schemaPolicies;`

**`flake-os`, `flake-hm`, `flake-packages`, etc. schema kinds deleted** — commit a6bd47c8:
- `/home/sini/Documents/repos/den/modules/context/flake-schema.nix` now only registers `["flake" "flake-system" "default"]` (line 8-10)

**Output class names registered** — `modules/policies/flake.nix` lines 41-46: `den.classes` gets entries for each of the five system output names so aspect key dispatch works.

**Forward infrastructure status** — NOT deleted as the spec contemplated:
- `/home/sini/Documents/repos/den/nix/lib/forward.nix` still exists (140 lines)
- `/home/sini/Documents/repos/den/nix/lib/aspects/fx/handlers/forward.nix` still exists with `buildForwardAspect`
- `/home/sini/Documents/repos/den/modules/aspects/provides/forward.nix` still exists
- Commit 3e064032 (2026-05-02) unified forwards into routes (all forwards register in `scopedRoutes`; complex/adapter forwards use `__complexForward`-marked routes), eliminating sub-pipelines and the separate `scopedForwardSpecs` state, but keeping the forward API itself

## Current Status
Still exists in the form delivered. `policy.instantiate` is the active path for OS/HM entity outputs. Forward infrastructure survives for user-defined adapter forwards (niri, persist-style cases), having been unified into routes rather than deleted.

## Supersession
This spec is part of the pipeline-simplification series. It was preceded by `2026-04-29-scope-partitioned-pipeline-state.md` (scope partitioning) and `2026-04-29-policy-route-class-delivery.md` (route delivery), all of which landed. The Tier 2 adapter elimination section of this spec was superseded by `2026-05-02-architecture-cleanup.md` or similar (commit 3e064032 took a different path: unify into routes rather than eliminate via class collection strategies).

## Gaps

**Class collection strategies** — the spec's "cleanest option" (section on Tier 2, option 3) for eliminating adapter forwards via `den.classes.<name>.collection = "list-unique"` was not implemented. The spec itself flagged this as a "larger change" and withdrew it to a follow-up.

**Forward infrastructure deletion** — steps 5-9 of the "forward elimination path" (`nix/lib/forward.nix` deleted, handlers deleted, `applyForwardSpecs` / `resolveForwardSource` deleted) did not happen as written. Instead, 3e064032 unified forwards into routes — the forward handler and lib are still present but sub-pipelines and the separate state key are gone.

**`den.schema.<kind>.policies` not actually used by flake.nix** — flake.nix uses `den.schema.flake-system.includes` (lines 88-92), not `den.schema.flake-system.policies`. The `policies` field on schema entries was added to `schemaEntryType` and wired into `mkDispatch`, but the flake module itself uses `includes` for policy injection, which is the older pattern. The schema-scoped dispatch via `policies` is functional but unused by the main flake policy module.

## Drift

**`resolve.to "host"` emitted alongside `policy.instantiate`** — the spec showed `policy.instantiate host` as a standalone replacement for `resolve.to "flake-os"` + `include flake-os`. The actual implementation (flake.nix lines 59-64) emits both `resolve.to "host"` and `policy.instantiate host` per host. The extra resolve is needed because `applyInstantiates` uses `findHostScopeId` to locate the host subtree from the pipeline walk data — the host scope must exist. The spec's description assumed `mainModule` would be used directly without needing a host walk, but the implementation re-runs the full subtree assembly (phases 1-3 via `extractSubtreeModules`) to get correct routing. This is more correct but differs from the spec's simpler "read `entity.mainModule` directly" model.

**`preWalkedModules` path added later** — commits f5972fce and 680a1452 (2026-05-06) extended `applyInstantiates` to re-run assembly phases on the host subtree rather than fall back to `spec.mainModule` directly. The spec described a simple `entity.mainModule` read; the actual implementation is subtree re-assembly for correctness with routes and pipes.
