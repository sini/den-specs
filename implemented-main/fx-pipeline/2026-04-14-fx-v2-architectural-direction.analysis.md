# Analysis: FX Pipeline v2: Architectural Direction

## Verdict

SUBSTANTIALLY IMPLEMENTED — all 7 key changes landed, but under different names and with
evolved designs compared to the April 14 spec. The "v2" concept was realized directly:
no v1/v2 flag split occurred. The legacy pipeline was removed outright in commit 2e0fc9e1
("refactor: remove legacy resolver, stabilize fx pipeline"), making the fx-as-effects
pipeline the only resolver.

## Delivery Target

feat/fx-pipeline branch (in progress). Legacy pipeline already removed. Branch is ahead
of main by ~60+ commits. Changes span April 23–28 2026.

## Evidence

**Key commits:**
- `2e0fc9e1` — remove legacy resolver, stabilize fx pipeline
- `3a9cea99` — effectful pipeline bootstrap, defensive guards (pipeline inside fx.handle)
- `8f9f283d` — parametric type separation, scope.provide
- `961463b7` — edge cases: fan-out dedup, cross-providers, perCtx, constraints
- `ed1301a4` — ctx-as-data transitions, deferred includes, fan-out identity
- `e0594bf1` — guard unsatisfied class modules + dedup class emissions by identity
- `4c58d6d6` — multi-class collection in classCollectorHandler
- `1c077a06` — forward as post-processing on pipeline results

## Current Status

### Change 1: `fx.handle` only at edges

**Implemented.** `pipeline.nix:165` — single `fx.handle` call in `mkPipeline`.
`resolveOne` does not exist; `aspectToEffect` returns computations, never resolves
them internally. The entire tree walks inside one `fx.handle` boundary.

### Change 2: `wrapAspect` produces recursive computations

**Implemented under a different name.** `wrapAspect` was never created; instead
`aspectToEffect` in `aspect.nix` is the equivalent. It handles both parametric
(`fx.bind.fn` for named args) and static (`compileStatic`) cases. Each aspect
emits its class/trait/nested keys as effects, then recurses into children via
`resolveChildren` → `emitIncludes`. No manual `go` loop.

### Change 3: `provide-class` effect / stateful class accumulation

**Implemented under a different name.** The spec called it `provide-class`; the
implementation uses `emit-class` effects (sent by `emitClasses` in `aspect.nix`).
`classCollectorHandler` in `handlers/tree.nix` accumulates modules by class key
in `state.classImports` (thunk-wrapped). Dedup by identity is present (`e0594bf1`).
Multi-class collection landed in `4c58d6d6`. The aspectPath-based `aspectIdentity`
dedup maps exactly to what the spec described.

### Change 4: Exclude/replace as query effects

**Implemented, evolved design.** The spec proposed `check-exclusion` querying a
handler populated from `meta.adapter` declarations. The actual implementation uses
`register-constraint` / `check-constraint` effects handled by `constraintRegistryHandler`
in `handlers/tree.nix`. Constraints are registered from `meta.handleWith` and
`meta.excludes`, then queried per child via `emit-include` → `check-constraint`.
Substitute (replace) support also shipped. The term "adapter" was retired; constraints
is the production name.

### Change 5: Presence queries as effects (`hasAspect`)

**Partially implemented, different mechanism.** The spec proposed `fx.send "is-present"
ref` with handler checking accumulated path set. Actual implementation: `includeIf` in
`includes.nix` sends `get-path-set` to the `pathSetHandler`, then evaluates a guard
function `{ hasAspect = ref: pathSet ? key; }`. The path set accumulation exists in
handler state. The effect is `get-path-set` not `is-present`, and `hasAspect` is a
pure Nix check on the returned set, not a separate effect per query. Functionally
equivalent.

### Change 6: Remove `flattenInto` dedup from ctx-apply / `assembleIncludes`

**Fully implemented.** Neither `flattenInto` (in the dedup sense) nor `assembleIncludes`
exist in the current codebase. Dedup is entirely in handler state: `ctxSeenHandler`
(context-key dedup in `handlers/ctx.nix`) and `includeSeen` in `includeHandler`
(include-level dedup, `52f0acc4`). The `flattenInto` function that does exist in
`transition.nix` is unrelated — it flattens attrset transition targets to lists.

### Change 7: Includes as computation chains in ctx-apply / `buildStageIncludes`

**Implemented, evolved.** `buildStageIncludes` was never created as a named function.
Instead, `resolveChildren` in `aspect.nix` produces the computation chain:
`emitSelfProvide` → `emitCrossProvideShims` → `emitAspectPolicies` → `emitIncludes`
→ `emitTransitions` — all bound together via `fx.bind`. This covers main aspect,
self-provider, cross-provider as sequential effect sends, matching the spec intent.

### Migration path / fxPipeline flag

**Not followed.** The spec said v2 would be built alongside v1 with an A/B flag.
What actually happened: v2 was built as the only pipeline (commit 2e0fc9e1, April 23),
with the legacy resolver removed immediately. No `fxPipeline` flag was ever added on
this branch.

### nix-effects loading

**Implemented ahead of spec.** `nix/lib/fx.nix` uses `fetchTarball` locked to
`templates/ci/flake.lock` as fallback when `inputs.nix-effects.lib` is absent — exactly
what the spec described. This was in place before the spec was written (the spec noted
it as a pre-merge requirement; the implementation had it already).

## Supersession

The spec was a planning document written during code review on PR #2. By April 23 the
implementation had leapfrogged it:

- "adapter" terminology → "constraint" (rename happened organically)
- `provide-class` → `emit-class` (name chosen differently)
- `check-exclusion` → `check-constraint` (generalized to support substitute)
- `is-present` / `hasAspect` effect → `get-path-set` + guard closure
- `buildStageIncludes` → `resolveChildren` bind chain
- `wrapAspect` → `aspectToEffect` / `compileStatic`

Later work on feat/fx-pipeline extended the architecture with traits, policies,
deferred includes, class forwarding, and provides-compat shims — none of which appear
in this spec.

## Gaps

One spec item was never realized as designed:

- **`hasAspect` as a dedicated effect.** The spec wanted `fx.send "is-present" ref`
  per query. The actual design batches path-set retrieval into `get-path-set` + guard
  closure. The distinction matters only for composability of handlers; the functional
  result is identical.

## Drift

The spec assumed a v1→v2 migration with the `fxPipeline` flag providing a safe fallback.
The actual approach was bolder: single-pipeline from day one of feat/fx-pipeline. This
eliminated the migration complexity but means the branch cannot be partially reverted to
v1 behavior. This is consistent with the broader project direction (see
project_legacy_removal_state.md memory — legacy was always the target for removal).

The "adapter" → "constraint" rename is the most visible terminology drift. The spec used
"adapter" for both the exclusion mechanism (`meta.adapter`) and the forward/shim layer.
The codebase now uses "constraint" for exclusion/substitution and reserves "adapter"
only inside the forward handler's internal wiring (`adapterKey`, `adapterMods`).
