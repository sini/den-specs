# Analysis: Pipeline Simplification Targets (2026-04-26)

Spec date: 2026-04-26. Analysis date: 2026-04-28. Branch: feat/fx-pipeline.

---

## Target 1: Delete the `provides` API

### Verdict
NOT STARTED — live code, compat shim ADDED

### Delivery Target
Unscheduled. Requires direct-ref-aspects Task 4 (entityProvides removal) as prerequisite — that task is not yet complete.

### Evidence
- `emitSelfProvide` present in `nix/lib/aspects/fx/aspect.nix` (line 687, called at line 760)
- `mkPositionalInclude` / `mkNamedInclude` present in `aspect.nix` (lines 603, 634)
- `emitCrossProvider` present in `nix/lib/aspects/fx/handlers/transition.nix` (line 101, called at line 287)
- NEW: `nix/lib/aspects/fx/handlers/provides-compat.nix` added AFTER spec was written — a compat shim for main-era `provides.X` cross-provide patterns. Commits: `c52a3779`, `2e40b024`
- The provides-compat shim emits deprecation warnings and re-routes to `policy.include`, but does NOT remove the underlying `emitSelfProvide` / `emitCrossProvider` infrastructure

### Current Status
The spec's prerequisite (entityProvides removal) is incomplete. The branch has moved in the opposite direction for compat: a new handler was added to preserve cross-provide behavior during a migration period. `emitSelfProvide` and `emitCrossProvider` remain fully live.

### Supersession
None. The spec's migration path is intact — the compat shim is step 1 of the migration (audit + deprecation trace). The spec's step 2 onwards (remove emitSelfProvide, emitCrossProvider, mkPositionalInclude, mkNamedInclude, rootIncludes) has not begun.

### Gaps
- `entityProvides` removal (direct-ref-aspects Task 4) is the gating prerequisite and is not done
- The compat shim references in provides-compat.nix mirror `emitSelfProvide`'s shape detection (comment at line 18) — this coupling will need to be severed when the shim outlives the original

### Drift
Spec says `emitCrossProvider` "remains live until entityProvides is fully removed" — accurate, still the case. Compat shim addition is consistent with the spec's migration path but was not anticipated as a distinct step.

---

## Target 2: Remove `wrapClassModule` Collision Detection

### Verdict
NOT STARTED — all symbols live

### Delivery Target
Spec places this last in recommended order (breaking change). No commits targeting it on this branch.

### Evidence
- `wrapClassModule`, `resolveCollisionPolicy`, `mkCollisionDetector` all present in `nix/lib/aspects/fx/aspect.nix` (lines 97, 38, exported at line 969)
- `advertisedArgs` / `lib.setFunctionArgs` still used at lines 117, 262, 265
- `collisionPolicy` option present in `modules/options.nix` (lines 119, 372)
- `collisionPolicy` referenced in test fixtures: `aspect-meta.nix`, `class-module-partial-apply.nix`, `traits.nix`, `aspect-content-type.nix`

### Current Status
Zero progress. The collision detection system is unchanged from before the spec was written.

### Supersession
None.

### Gaps
The spec's scope note is still unresolved: `advertisedArgs` / `lib.setFunctionArgs` does dual duty (collision detection + making den args optional in NixOS module resolution). Removal requires a replacement mechanism before the collision detection path can be deleted.

### Drift
None. Spec accurately describes current state.

---

## Target 3: Extract Sub-Pipeline Patterns

### Verdict
IMPLEMENTED — `runSubPipeline` extracted and all three call sites use it

### Delivery Target
Landed. Commit `1c077a06` ("forward as post-processing on pipeline results") is the key delivery.

### Evidence
- `runSubPipeline` defined in `nix/lib/aspects/fx/pipeline.nix` (line 242)
- Three call sites confirmed:
  1. `resolveFanOut` in `transition.nix` (line 160) — uses `runSubPipeline`, then `state.modify` to merge classImports
  2. `resolveSiblingTransition` in `transition.nix` (line 216) — uses `runSubPipeline`, extracts `sub.traits`, sends `provide-to` effect
  3. `applyForwardSpecs` in `pipeline.nix` (line 212) — uses `runSubPipeline`, wraps in adapter aspect (forward post-processing)
- `runSubPipeline` materializes state thunks and returns `{ classImports, traits, provideTo }` — matches spec's description of shared setup with per-site post-processing

### Current Status
Complete. The spec's intent is fully realized. The three call sites do genuinely different post-processing as specified. State materialization is correct (thunks evaluated, forwarded specs applied inside runSubPipeline itself).

### Supersession
The spec anticipated `forwardHandler` as one of the three call sites. The implementation moved forward to post-processing (`applyForwardSpecs`), which is an improvement — forward is no longer a live handler but is applied after the main pipeline resolves. The third call site is therefore `applyForwardSpecs` rather than `forwardHandler`.

### Gaps
None.

### Drift
Minor: spec described `forwardHandler` as a call site; implementation uses `applyForwardSpecs` (same logical site, refactored shape). The post-processing approach is cleaner than the spec anticipated.

---

## Target 4: Replace `classifyKeys` with Declared Schemas

### Verdict
PARTIALLY IMPLEMENTED — registry dispatch added, depth probe simplified, but `classifyKeys` and heuristic remain

### Delivery Target
Partial. Commit `e790dc8e` ("unified aspectKeyType — three-branch dispatch by class/trait registry") introduced registry-based dispatch. The heuristic is reduced but not removed.

### Evidence
- `classifyKeys` still present in `nix/lib/aspects/fx/aspect.nix` (line 296), still called at line 832
- The function now dispatches: classRegistry lookup → traitRegistry lookup → depth-limited sub-key scan (depth 3, reduced from 4) → unregisteredClassKeys fallback
- No `__keyTypes` annotation mechanism added — static pre-classification at `den.aspects` boundary was not implemented
- `freeformIgnoreSet` is NOT present in current code (searched: no matches) — the spec's `freeformIgnoreSet` reference may refer to structural keys handled via `structuralKeysSet` instead
- The `targetClass` dynamic check remains (line 315: `k == targetClass`) as the spec's Option 1 predicted
- Backward compat path: when both registries are empty, all non-structural keys treated as class keys (line 300-308)

### Current Status
The spec's full migration path (add `__keyTypes`, classify at construction time, remove `classifyKeys` entirely) was not implemented. The implementation chose a middle path: registry lookup at classify time rather than construction time. This reduces the heuristic surface (fewer false-positive nested aspect detections possible when registries are populated) without the complexity of static annotation.

### Supersession
`freeformIgnoreSet` does not appear in the codebase — the spec may have been describing planned machinery that was superseded by the `structuralKeysSet` approach that was already present.

### Gaps
- `__keyTypes` static annotation: not implemented
- `classifyKeys` bulk classification: still present (~68 lines)
- Depth probe (now 3-level): still present and still the most fragile path
- Full removal blocked on Target 1 (provides removal shrinks `classifyKeys` input surface)

### Drift
Spec proposed pre-classification at `den.aspects` boundary. Implementation chose runtime registry lookup. The practical effect is similar for registered keys; unregistered key handling (the fragile path) is unchanged.

---

## Target 5: Generalize Flake Fan-Out

### Verdict
IMPLEMENTED — `isolateFanOut` property on routing, hardcoded `targetClass == "flake"` check removed

### Delivery Target
Landed. Commit `73ac2f50` ("strip _core and isolateFanOut from core policies, use resolve.shared") is the key delivery.

### Evidence
- No `targetClass == "flake"` check present anywhere in transition.nix
- `isolateFanOut` is computed from policy effects: `if resolve.__shared then false else true` (transition.nix lines 445-446, 539-540)
- `resolveFanOut` is called when `isFanOut && routing.isolateFanOut` (line 331)
- `resolve.shared` effect variant added for non-isolated fan-out (commit `8b999f35`)
- The "flake-level" behavior is now generalized: any fan-out without `__shared` gets sub-pipeline isolation

### Current Status
Complete. The spec's migration path was followed: the `targetClass == "flake"` special case is gone. Fan-out isolation is a first-class property derived from the `resolve.shared` effect on the policy. The spec anticipated this as "low risk if Target 3 is done first" — and Target 3 (runSubPipeline) was done first.

### Supersession
The spec described `policy.isolateFanOut = true` as the generalized property. The implementation inverts the flag and derives it from `resolve.shared` (false = isolated). This is a cleaner design — opt-in sharing rather than opt-in isolation.

### Gaps
None.

### Drift
Flag polarity inverted from spec (`isolateFanOut = true` → `__shared = false` → `isolateFanOut = true` in routing). Functionally equivalent, semantically cleaner.

---

## Target 6: Normalize Child Shapes at Source

### Verdict
NOT STARTED — wrapChild infrastructure still present

### Delivery Target
Spec places this last (depends on Target 2 namespace change). Target 2 is not started.

### Evidence
- `wrapChild`, `wrapFunctorChild`, `wrapBareFn`, `normalizeModuleFn` all present in `nix/lib/aspects/fx/handlers/include.nix` (lines 69, 29, 55, 18)
- `wrapChild` called at line 259
- `nix/lib/aspects/types.nix` also contains these symbols (shared type machinery)

### Current Status
Zero progress. The include handler's multi-shape normalization is unchanged.

### Supersession
None.

### Gaps
The spec's scope note about internal factories (`den.provides.forward`, `den.provides.mutual-provider`) producing functor shapes: these factories are now replaced/shimmed (mutual-provider replaced by aspect-included policyFns per commit `11625e47`; provides.forward now handled by `applyForwardSpecs` post-processing). The scope of Target 6 may have shrunk — fewer internal functor producers means fewer call sites that must produce canonical form before hitting `wrapChild`.

### Drift
Mutual-provider removal (commit `11625e47`) and forward-as-post-processing (commit `1c077a06`) reduced the spec's "internal factories are a hidden scope increase" risk. The internal factory surface is smaller than when the spec was written.

---

## Summary Table

| Target | Status | Notes |
|---|---|---|
| 1: Delete provides API | NOT STARTED | Compat shim added; gating prerequisite (entityProvides removal) incomplete |
| 2: Collision detection removal | NOT STARTED | Breaking change; scheduled last |
| 3: Sub-pipeline extraction | COMPLETE | runSubPipeline landed; forward moved to post-processing |
| 4: classifyKeys → declared schemas | PARTIAL | Registry dispatch added; full static annotation not implemented |
| 5: Generalize flake fan-out | COMPLETE | isolateFanOut property derived from resolve.shared |
| 6: Child shape normalization | NOT STARTED | Blocked on Target 2; internal factory scope reduced |

Recommended order from spec: Targets 3 and 5 are the "immediate wins" — both are now done. Next in spec order: Target 1 (blocked on prerequisite), then Target 4, then Target 2, then Target 6.
