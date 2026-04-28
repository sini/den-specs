# Analysis: Remove context guards — handler-based resolution makes them unnecessary

## Verdict

Partially implemented. The "already implemented" section of the spec is fully delivered. The "remaining (next phase)" section — removing `__functor`/`__functionArgs` options from the aspect submodule and eliminating `__scope`/`__parentScope` — has not been executed. The branch is in the correct intermediate state described by the spec.

## Delivery Target

`feat/fx-pipeline` branch. The spec predates the pipeline simplification work that landed on this branch; the already-implemented items are present. No guard-removal commit is visible in the recent log — these landed earlier (pre-`d2ace76d`).

## Evidence

### Implemented: deprecation shims

**`nix/lib/take.nix`** — all three forms are identity wrappers with deprecation warnings:
- `take.exactly` emits a `lib.warn` and uses the optional-arg pattern (`__fn`/`__args`/`meta.exactMatch = true`) to detect wrong-level invocation
- `take.atLeast` and `take.upTo` are plain `lib.warn fn` (identity)
- `take.__functor` custom-predicate form warns and returns `fn` unchanged

**`modules/context/perHost-perUser.nix`** — optional-arg pattern shim:
- `perCtx` wraps aspect in `{ __fn; __args }` with extra context keys (user/home) declared optional
- If any extra key resolves, returns `{}` (wrong level, no-op)
- `perHost`, `perUser`, `perHome` all delegate to `perCtx`
- Deprecation warning fired via `lib.warn`

**`nix/lib/parametric.nix`** — shims stamp `__scopeHandlers` instead of `__ctx`:
- `fixedTo` / `fixedTo.exactly` / `.atLeast` / `.upTo` all delegate to `mkFixedTo` which stamps `__scopeHandlers = constantHandler ctx`
- `expands` stamps `__scopeHandlers = (existing) // constantHandler attrs`
- Both return `{ __fn; __args; __scopeHandlers; }` parametric wrappers

**`nix/lib/aspects/fx/handlers/include.nix`** — `keepChild` implements has-handler probing:
- No `meta.contextGuard` branch present anywhere in the file
- `keepChild` probes required args via `fx.effects.hasHandler key` (has-handler chain)
- Optional args (`__args.${k} == true`) are not probed — they handle the perCtx/take.exactly optional-arg gate
- `probeArgs` chains `has-handler` effects recursively; unresolvable children are deferred via `defer-include`

**`nix/lib/aspects/fx/aspect.nix`** — `forwardWrap` is gone; `__scope` absent:
- No `forwardWrap` function exists in the file
- No `__scope` or `__parentScope` keys referenced in the pipeline core
- `structuralKeysSet` does not include `__scope` or `__parentScope`
- `tagParametricResult` merges `__scopeHandlers` from parent + resolved, stamps `__ctxId`, `__parametricResolved` — no `resolvedCtx` block
- `aspectToEffect` uses `scope.provide scopeHandlers` (not `scope.stateful`) and `fx.bind.fn userArgs fn`
- `meta.exactMatch` path injects `__scopeKeys` into args for `take.exactly` detection
- `resolveChildren` passes `__parentScopeHandlers` only (not `__parentScope`) to `emitIncludes`

**`modules/options.nix`** — `resolvedCtx` present but unrelated:
- `resolvedCtx` in `options.nix` (line 59) is a NixOS module that adds `id_hash`, `resolved`, `collisionPolicy` options to entity schemas — not the `aspectToEffect` resolvedCtx extraction block from the spec. Different concern entirely.

**`nix/lib/aspects/default.nix`** — no `__scope` preservation:
- `normalizeRoot` propagates `__scopeHandlers` only (line 50)
- `fxResolveTree` derives `ctx` from `ctxFromHandlers` on `__scopeHandlers`
- No `__scope` key touched

**`nix/lib/aspects/fx/handlers/transition.nix`** — stamps `__scopeHandlers` only:
- All tagged-aspect construction uses `__scopeHandlers = scopeHandlers` (constantHandler)
- No `__scope` stamp anywhere in file

### Not implemented: remaining (next phase)

**`nix/lib/aspects/types.nix`** — `__functor` and `__functionArgs` options are NOT removed from submodule:
- `aspectSubmodule` does not declare `__functor` or `__functionArgs` as NixOS options (they are not in the options block at lines 328–429)
- However: `mergeWithAspectMeta` explicitly attaches `__functor = resolveAspectWith` on merged aspects (line 95)
- Single bare parametric fn path in `mergeFunctions` attaches `__functor = self: self.__fn` (line 162)
- `wrapChild` in `include.nix` still calls `child.__functor child` (line 32) to unwrap functor children
- This means `forwardWrap` would still be needed as non-identity if `__functor` were on submodule defaults — but since `__functor`/`__functionArgs` are NOT submodule options (they're attached post-merge), the infinite re-entry class of bugs the spec describes for this phase may not materialize. The spec's concern was the submodule default adding them to every aspect; the current code only attaches them on explicitly merged aspects.

**`__scope` / `__parentScope` in struturalKeys** — already absent:
- `structuralKeysSet` (aspect.nix line 12–33) does not list `__scope` or `__parentScope`
- These were either never added or already removed; no gap here

## Current Status

The guard mechanism is fully eliminated from the pipeline core. The deprecation shims (`perHost`, `perUser`, `take.exactly`, `take.atLeast`, `take.upTo`, `fixedTo`, `expands`) are in place with the optional-arg detection pattern. `has-handler` probing and `keepChild` deferral work as specified.

The "remaining next phase" items from the spec's files-changed table are partially irrelevant to the current state:
- `__functor`/`__functionArgs` were never added as submodule options, so removing them is a no-op
- `forwardWrap` was removed (not reduced to identity) — the spec describes an intermediate identity step; the final state skips it
- `__scope` and `__parentScope` are already absent from all pipeline files

## Supersession

The spec's "remaining" work on `ctx-apply.nix` is moot — `ctx-apply.nix` does not exist on this branch. Its role is handled by `transition.nix`'s `resolveContextValue` which already stamps `__scopeHandlers` only. The spec was written against an older file layout.

## Gaps

1. `take.exactly` uses `scope.provide` (checked as `scope.stateful` in the spec's pseudocode) — actual implementation uses `fx.effects.scope.provide` in `aspectToEffect`, which is the current API. Not a gap, just API name drift from spec writing.

2. `aspectToEffect` injects `__scopeKeys` for `meta.exactMatch` aspects (the `take.exactly` shim) — this was not in the original spec design section but is present in implementation. The spec's "already implemented" table predates this mechanism; it was added as the shim evolved.

3. The spec mentions `emitSelfProvide` should stop stamping `__parentScope` — confirmed absent; `emitSelfProvide` uses `__scopeHandlers` only.

## Drift

- The spec's `parametric.fixedTo`/`expands` shims were described as stamping `__scopeHandlers` and using `constantHandler` — implemented exactly as described, but returns a `__fn/__args` parametric wrapper rather than a plain aspect with `__scopeHandlers` directly. This is correct: the wrapper survives `providerType` submodule merge.
- `scope.provide` is used instead of `scope.stateful` in `aspectToEffect` — `scope.provide` is the current nix-effects API for installing handlers without stateful tracking; functionally equivalent for this use case.
- `forwardWrap` was not reduced to identity and then removed in a second commit — it was eliminated entirely. The spec described a two-phase approach; the branch went directly to removal.
