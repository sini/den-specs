# Analysis: Includes Chain Effects Design

## Verdict

Fully implemented. All core mechanisms from the spec landed in `68c26555` (den-fx, main, 2026-04-17). The implementation is complete and active on both `main` and `feat/fx-pipeline`.

## Delivery Target

`main` — shipped in `68c26555` ("refactor: den-fx (#462)"). The commit message explicitly calls out: "Includes chain provenance — chain-push/chain-pop effects replace __parent string tracking; observable by any handler for diagrams, scope visualization, composable analysis."

## Evidence

**chain-push / chain-pop effects:**
- `nix/lib/aspects/fx/aspect.nix:742` — `chainWrap` emits `chain-push`/`chain-pop` around meaningful node subtrees
- `nix/lib/aspects/fx/handlers/tree.nix:121-148` — `chainHandler` handles both effects, with thunk-wrapped `includesChain` state

**chainHandler wired into pipeline:**
- `nix/lib/aspects/fx/pipeline.nix:75` — `handlers.chainHandler` in `defaultHandlers`
- `nix/lib/aspects/fx/pipeline.nix:135` — `includesChain = _: []` in `defaultState`

**__parent removed from resolve-complete emissions:**
- `nix/lib/aspects/fx/trace.nix:29,44,95,122` — all parent derivation reads from `state.includesChain`, never `param.__parent`
- `nix/lib/diag/default.nix` — no `__parent` references (file contains no resolve-complete emissions with __parent)
- `templates/ci/modules/features/fx-parametric-meta.nix:115-116` — test reads `state.includesChain` directly, not `param.__parent`

**Adapter/constraint scoping via ownerChain:**
- `nix/lib/aspects/fx/handlers/tree.nix:23,37,50` — `ownerChain` stamped from `state.includesChain` at registration time
- `nix/lib/aspects/fx/handlers/tree.nix:73-74` — `isAncestor` prefix check on `currentChain`
- `nix/lib/aspects/fx/constraints.nix` — `exclude`, `substitute`, `filterBy` constructors with `scope = "subtree"` default and `.global` variant

**Effect naming drift (register-adapter → register-constraint):**
The spec used `register-adapter`/`check-exclusion` effect names. The implementation uses `register-constraint`/`check-constraint` throughout. Functionally equivalent.

## Current Status

Fully operational. The `__parentScopeHandlers` / `__parentCtxId` fields still present in `aspect.nix` are unrelated — they propagate scope handler context for sub-pipeline wiring, not the includes-chain parent tracking described in the spec.

The `chainParent` helper in `trace.nix:56-69` adds extra sophistication beyond the spec: it filters out anonymous intermediates and self-references from the chain before deriving parent, using `isMeaningfulName` — addressing the anonymous node edge cases the spec discussed in prose but left as implicit behavior.

Error semantics differ from spec: `chain-pop` on empty chain throws an error (`"fx: chain-pop on empty includesChain — push/pop mismatch"`) rather than the spec's safe degradation path returning `[]`. Stricter and correct.

## Supersession

The spec referenced 5 files for `__parent` removal:
- `nix/lib/aspects/fx/resolve.nix` — does not exist; functionality merged into `aspect.nix` and handler files during den-fx refactor
- `nix/lib/aspects/fx/adapters.nix` — does not exist; split into `constraints.nix` and `trace.nix`
- `nix/lib/diag/default.nix` — exists, no `__parent` references
- `templates/diag-fx-demo/modules/fx-debug.nix` — template renamed/reorganized to `diagram-demo`; no fx-debug.nix present
- `templates/ci/modules/features/fx-parametric-meta.nix` — exists, uses chain state (not __parent)

The spec's file list reflects a pre-den-fx file layout that was fully reorganized during `68c26555`.

## Gaps

None functionally. The spec's `filterAspect` API became `filterBy` in `constraints.nix`. The spec's adapter API shape (`excludeAspect`, `substituteAspect`, `filterAspect`) maps to the implementation's (`exclude`, `substitute`, `filterBy`), all with `.global` variants. The rename is cosmetic and the scoping semantics are identical.

The spec mentioned `graph.nix` reads `parent` from trace entries (no changes needed). This holds — `nix/lib/diag/graph.nix` still reads `parent` from entries, which trace handlers now populate via chain state.

## Drift

**Thunk-wrapped state:** The implementation wraps `includesChain` as `_: []` (a thunk) rather than a plain list, matching the trampoline's deepSeq strategy for all growing state fields. The spec showed a plain list. This is an implementation refinement applied uniformly across all state fields after the spec was written.

**`constraintRegistryHandler` not `adapterRegistryHandler`:** The spec named the combined handler `adapterRegistryHandler`. The implementation split concerns: `constraintRegistryHandler` handles `register-constraint`/`check-constraint`, while `chainHandler` is separate. Structure is cleaner than spec.

**`chainParent` filtering logic:** The spec's anonymous node handling said "read `last chain` as parent." The implementation's `chainParent` walks the chain backward filtering out self-references and non-meaningful entries before taking the last. More robust than the spec draft.
