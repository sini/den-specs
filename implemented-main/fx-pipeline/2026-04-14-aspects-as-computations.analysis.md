# Analysis: Aspects as Computations

## Verdict

**Substantially implemented, with divergence in naming and structure.** The core thesis — aspects compile to effect types, handlers own resolution strategy — is fully realized. The specific API surface (function names, effect names, handler decomposition) diverged during implementation, but the semantic model is intact and operational on `feat/fx-pipeline`.

## Delivery Target

`feat/fx-pipeline` (current branch). Not on `main`. The spec targeted `feat/fx-resolution`, which was absorbed into the pipeline branch.

## Evidence

**Three effect types present:**

| Spec name | Implemented name | File |
|---|---|---|
| `provide-class` | `emit-class` | `nix/lib/aspects/fx/aspect.nix:437,444` |
| `resolve-include` | `emit-include` | `nix/lib/aspects/fx/aspect.nix:502,544` |
| `register-constraint` | `register-constraint` | `nix/lib/aspects/fx/aspect.nix:481` |

`register-constraint` matches exactly. `provide-class` → `emit-class` and `resolve-include` → `emit-include` are name-only divergences; the semantics are identical.

**Effect handlers present:**

- `emit-class` handled by `classCollectorHandler` in `nix/lib/aspects/fx/handlers/tree.nix:152`
- `register-constraint` / `check-constraint` handled by `constraintRegistryHandler` in `nix/lib/aspects/fx/handlers/tree.nix:19`
- `emit-include` handled by `includeHandler` in `nix/lib/aspects/fx/handlers/include.nix:254`

**`aspectToEffect` present and central** (`nix/lib/aspects/fx/aspect.nix:908`). It is the primary entry point: parametric aspects iterate via `fx.bind.fn`, static aspects compile via `compileStatic`. Both paths send `emit-class`, `emit-include`, `register-constraint`.

**Handler-owned recursion implemented** — `includeHandler` intercepts `emit-include`, calls `aspectToEffect` directly for kept children (`include.nix:203,225`), calls it for substitution replacements (`include.nix:140`). This is the effectful resume model: the handler owns recursion, not the computation.

**`emitIncludes` / `foldIncludes` equivalent present** (`aspect.nix:486–513`). Named `emitIncludes` rather than `foldIncludes`. Same fold-left structure, sends `emit-include` per child with index and parent scope.

**`resolveConditional` present** (`include.nix:94–111`). Named function matches the spec design: queries `get-path-set`, evaluates guard, either calls `emitIncludes` (pass) or `tombstoneAll` (fail). Called from `includeHandler` on `isConditional` children.

**`resolveChildren` still exists** (`aspect.nix:748`). This is the primary divergence point — see Drift.

**`scope.provide` in use** (`aspect.nix:915`). Scoped handler injection for parametric context propagation.

## Current Status

All three effect types operational. `includeHandler` handles `emit-include` as an effectful handler with `check-constraint` probe, constraint dispatch, and recursive `aspectToEffect` calls. `classCollectorHandler` accumulates `emit-class` emissions. `constraintRegistryHandler` stores and checks `register-constraint` entries. The pipeline assembles these in `defaultHandlers` (`pipeline.nix:57–88`) and runs via `mkPipeline` (`pipeline.nix:145–175`).

`resolveDeepEffectful` was not created as a separate function. Its responsibilities are distributed across `compileStatic` + `resolveChildren` + `aspectToEffect` in `aspect.nix`, with handlers in `handlers/include.nix` and `handlers/tree.nix`.

## Supersession

The spec's `resolveDeepEffectful` → `{ resolveAspect, resolveIncludeHandler }` return shape was not adopted. Instead:

- `aspectToEffect` IS `resolveAspect` — it is the top-level computation constructor
- `includeHandler` IS `resolveIncludeHandler` — it intercepts `emit-include` and recurses via `aspectToEffect`
- The handler is wired into `defaultHandlers` at module-load time (not returned by a resolver factory)
- `compileStatic` + `resolveChildren` remain as the internal structure of `aspectToEffect`'s static path

The spec proposed exporting `{ resolveAspect, resolveIncludeHandler }` as a pair from `resolveDeepEffectful` so tests could inject the recursive handler. The implementation instead exports `aspectToEffect` from `aspect.nix` and the handler lives in `handlers/include.nix`. Tests compose handlers directly via `defaultHandlers`.

## Gaps

1. **`resolveDeepEffectful` not present** — the named function from the spec was never created. Functionality is present, distributed differently.
2. **`resolve-complete` ownership** — spec says the handler emits `resolve-complete` for children (option A for root). Implementation: `resolveChildren` emits `resolve-complete` for the current node (`aspect.nix:783`), handlers emit it for tombstones and in `resolveConditional`. Partially different from spec but functionally equivalent.
3. **nix-effects dependency** — spec required `sini/nix-effects#feat/effectful-handlers`. The lock (`templates/ci/flake.lock:107`) references `sini/nix-effects` main rev (no feature branch), suggesting the effectful-handler support landed in main before implementation.
4. **`foldIncludes` name** — implemented as `emitIncludes`, not `foldIncludes`. Minor.

## Drift

**`resolveChildren` persists.** The spec's migration table listed `resolveChildren (foldl over children)` → `foldIncludes (sends resolve-include per child)`. In the implementation, `resolveChildren` (`aspect.nix:748`) still exists and calls `emitIncludes`. The spec intended `resolveChildren` to be deleted and replaced by `foldIncludes`; instead `resolveChildren` was refactored to use `emitIncludes` internally and kept as the orchestration point for self-provide, cross-provide shims, policy dispatch, and transition emission.

**`go` inside `emitIncludes`.** The fold helper inside `emitIncludes` is still named `go` (`aspect.nix:495`), an index-incrementing tail-recursive fold, structurally identical to the original manually-recursive walker but now expressed via `fx.bind` chains over `emit-include` sends.

**Handler decomposition diverged.** Spec proposed a single `resolveIncludeHandler` in `resolveDeepEffectful`. Implementation splits into `includeHandler` (emit-include dispatch) in a separate file, composited statically into `defaultHandlers`. The recursive reference to `aspectToEffect` is resolved via the module fixpoint, not via a factory closure.

**Additional handlers added.** `defaultHandlers` includes `traitArgHandler`, `traitCollectorHandler`, `registerAspectPolicyHandler`, `dispatchPolicyIncludesHandler`, `deferredIncludeHandler`, `drainDeferredHandler`, `resolveEntityHandler`, `forwardHandler`, `provideToHandler` — none present in the spec. These reflect post-spec feature additions (traits, policies, provides-compat, forward post-processing) rather than spec violations.
