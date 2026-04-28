# Analysis: Context as Data — Remove scope.run from Transitions

## Verdict

Implemented. Core design landed and extended beyond original spec.

## Delivery Target

Initial delivery: commit `cd1301a4` ("ctx-as-data transitions, deferred includes, fan-out identity")
Key follow-ups: `a7f101cc`, `3d206058`, `e668458e`, `4a9e3453`

## Evidence

**scope.run in transition.nix — REMOVED**
Grep finds zero `scope.run` calls anywhere in `nix/lib/aspects/fx/`. The `resolveContextValue` and `emitCrossProvider` functions no longer use `scope.run`. `mkScopedTransitionHandler` does not exist in the codebase.

**__ctx replaced by __scopeHandlers**
The spec's `__ctx` field was implemented then superseded. Commits `a7f101cc` and `3d206058` replaced direct `__ctx` propagation with `__scopeHandlers`-based context reconstruction. `ctxFromHandlers` (aspect.nix:276) reconstructs context by probing the constant handler map. `__ctx` references in aspect.nix are all commented out (lines 624, 632, 662, 810-811, 898) — tombstones of the migration.

**bind.fn mini-scope — IMPLEMENTED (variant)**
`aspectToEffect` (aspect.nix:915-928) uses `fx.effects.scope.provide scopeHandlers` wrapping only the `fx.bind.fn userArgs fn` call — matching the spec's mini-scope intent. Implementation uses `scope.provide` not `scope.run`, which is the correct nix-effects API for handler injection without state isolation.

**__parentCtxId propagation — IMPLEMENTED**
`resolveChildren` / `emitIncludes` propagates `__parentCtxId` and `__parentScopeHandlers` on `emit-include` payloads (aspect.nix:508, 550, transition.nix:269). `includeHandler` merges `parentScopeHandlers` and `parentCtxId` onto wrapped children (include.nix:264-267).

**keepChild probe-arg optimization — IMPLEMENTED**
`keepChild` (include.nix:183-186) filters `keysToProbe` by subtracting keys already in `childScopeHandlers`. Required args present in `__scopeHandlers` skip the `has-handler` probe entirely, matching the spec's "args in `__ctx` are known-available" intent.

## Current Status

Fully implemented. The codebase diverged from the exact field names in the spec (`__scopeHandlers` instead of `__ctx`, `scope.provide` instead of `scope.run`) but the architectural contract is identical: context flows as data on nodes, not as ambient handler scope, and only `bind.fn` argument resolution is scoped.

## Supersession

The spec described `__ctx` as the data carrier. The implementation evolved to `__scopeHandlers` (a handler map rather than a plain attrset) as the canonical carrier, reconstructed to ctx on demand via `ctxFromHandlers`. This is strictly superior — it avoids a second representation and keeps the handler map as the single source of truth for context. Commits documenting this evolution: `a7f101cc`, `3d206058`.

## Gaps

None functional. The `includeHandler` import in `transition.nix` mentioned in "What gets removed" is still absent from transition.nix's imports (pipeline.nix line 76 wires `includeHandler` separately into the pipeline, not through transition). This matches spec intent.

## Drift

- Field name drift: `__ctx` → `__scopeHandlers`. Intentional improvement, not a regression.
- API drift: `scope.run` → `scope.provide`. Correct nix-effects API; `scope.run` was a legacy wrapper.
- Spec described `parentCtx` as a field in `emit-include` payload. Implementation uses `__parentScopeHandlers` and `__parentCtxId` as separate fields. Semantically equivalent, more precise decomposition.
- Deferred includes (not in spec) were added alongside this work, leveraging the new `__scopeHandlers` propagation for drain-deferred resolution.
