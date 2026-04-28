# Session Summary: Legacy Resolve Pipeline Removal

**Date:** 2026-04-17
**Branch:** `feat/rm-legacy`
**Starting point:** 493/493 tests on main
**Current state:** 352/452 tests passing (41 legacy-only tests deleted)

## What was accomplished

### Successfully removed
- `nix/lib/statics.nix`, `nix/lib/aspects/resolve.nix`, `nix/lib/aspects/adapters.nix` — legacy pipeline
- `modules/fxPipeline.nix` — the A/B gate (fx is now the only path)
- `defaultFunctor` and `typesConf` from `nix/lib/aspects/default.nix`
- `withOwn`, `deepRecurse`, `deep`, `deepParametrics`, `applyDeep`, `applyIncludes`, `mapIncludes`, `statics` imports from `nix/lib/parametric.nix`
- `den.fxPipeline = false` from all ~22 test files
- 7 legacy-only test files (adapter-owner, adapter-propagation, aspect-adapter, collect-paths, resolve-adapters, one-of-aspects, fx-flag)
- Legacy adapter references from has-aspect tests (rewritten to fx constraints)

### Successfully rewritten
- `nix/lib/aspects/has-aspect.nix` — uses `fxFullResolve` + `state.pathSet` instead of `resolve.withAdapter adapters.collectPaths`
- `nix/lib/parametric.nix` — simplified to `bindCtx` with recursive `resolveIncludes` + inline class keys
- `nix/denTest.nix` — trace utility rebuilt using fx `tracingHandler`
- `templates/ci/modules/features/aspect-path.nix` — uses `fx.identity.aspectPath`
- has-aspect tests Section F — rewritten from `meta.adapter` to `meta.handleWith` + fx constraints

### Key architectural decisions made

1. **`__functor = lib.const` default in `aspectSubmodule`** — needed. Without it, aspects aren't callable and many code paths break. `lib.const` means "ignore context, return self" which routes to `compileStatic` in the pipeline.

2. **`providerFnType.merge` kept with functor wrapping** — needed. The wrapping `{ __functor = _: eth.merge loc defs }` routed through `aspectType.merge` is required for `wrapChild` to extract inner function args correctly.

3. **`aspect-chain` kept in pipeline and providers** — needed for now. Providers use `fromAspect = _: lib.head aspect-chain` to get the root aspect for forwarding. Can't remove until providers are rewritten.

4. **`forward.nix` fromAspect hook preserved** — `osConfigurations.nix`, `hmConfigurations.nix`, `os-user.nix` use `fromAspect` with proper context objects (not aspect-chain). Only aspect-chain-dependent `fromAspect` was removed from specific providers.

5. **`scope.run` instead of `scope.stateful`** — critical discovery. `scope.stateful` uses `state.update` which overwrites outer state after scope exit, losing `emit-class` accumulations from rotated effects. `scope.run` preserves outer state. Applied to both `transitionHandler` and `keepChild`.

## Key discoveries

### `scope.stateful` state loss bug
`scope.stateful` implementation: `state.update (state: rotate { handlers, state } body)`. The `state.update` does `get → rotate → put`. After rotation, `put` overwrites outer state with rotation's internal state, discarding any outer state changes from effects that rotated outward. On main this is latent — `ctxApply` strips `into` so `transitionHandler` never fires and `scope.stateful` never executes. Fix: use `scope.run` for constant-providing handlers.

### Transparent ctxApply doesn't work
Making `ctxApply` return `self // { __ctx = ctx }` causes circular eval. Accessing `self.includes` or `self.into` triggers module system evaluation of context configs, which depends on aspect resolution, which calls ctxApply → cycle. `ctxApply` must eagerly evaluate transitions and produce NEW attrsets (thunk boundary).

### `scope.run` can't propagate context to children via rotation
When `emit-include` effects rotate outward from a `scope.run`, their continuations (including `aspectToEffect` calls) run OUTSIDE the scope. The scoped `constantHandler` is not active when those children are compiled. Only effects that match the scoped handler are caught — the rest leave the scope permanently.

### `bind.fn` attrs are send params, not pre-provided values
`bind.fn { host = val } fn` sends `host` with `val` as the effect PARAMETER. The `constantHandler` handler IGNORES the param and provides its own stored value. `bind.fn` attrs cannot bypass handlers.

## Remaining work (100 test failures)

### Error breakdown (from ci-deep)
- `unhandled effect 'user'` (3 unique, ~40 tests) — user-level parametric functions not resolved at host level
- `unhandled effect 'host'` (2 unique) — host-level parametric not resolved
- `unhandled effect 'hey'` (2 unique) — custom test effects
- `option already declared` / `defined multiple times` (4 unique, ~20 tests) — duplicate module declarations from inline class keys
- `attribute missing` (various, ~20 tests) — specific provider/test issues
- `attribute 'inputs'' missing` (3) — issue-369 tests (known fx crash)
- `expected a set but found null` (2) — null result issues

### Root cause of unhandled effects
`fixedTo` with recursive `resolveIncludes` resolves includes that `canTake.upTo attrs fn` passes. Functions needing args NOT in `attrs` (e.g., `{ user }:` at host level) pass through unchanged to the pipeline. The pipeline wraps them via `wrapChild` and `compileFunctor` sends the missing arg as an unhandled effect.

The old `deepRecurse` handled this by wrapping unresolvable functions in `{ class, aspect-chain }:` functors that returned `{}` on arg mismatch. The `{}` was filtered out, silently skipping the function.

### Approaches tried for context propagation
| Approach | Result | Why |
|----------|--------|-----|
| `scope.stateful` in keepChild | 238 | state.update overwrites outer state |
| `scope.run` in keepChild | 311→311 | emit-include rotates out of scope |
| `scope.run` in resolveChildren | 311→311 | same rotation issue |
| Transparent ctxApply | 190 | circular eval from module config |
| `bind.fn` preProvided | 234 | attrs are send params not values |
| Recursive `resolveIncludes` | **352** | works for sub-includes within resolved results |
| Filter unresolvable bare fns | 323 | too aggressive, removes functions needed at other levels |

### Recommended next steps

1. **Handle unresolvable parametric includes gracefully** — either make `bindCtx` smart about which functions to keep/filter based on which transition level they belong to, or add a pipeline-level fallback for unknown effects in `bind.fn` context.

2. **Fix duplicate module declarations** — the inline class keys approach may emit duplicates when the same class key appears through multiple resolution paths. May need dedup in `classCollectorHandler`.

3. **Fix test-specific issues** — `identity-preservation.nix` and `provider-provenance.nix` were gutted to empty, need fx-equivalent tests. Various deadbug tests need attention.

4. **Clean up stale comments** — references to `deepRecurse`, `defaultFunctor`, `statics` in pipeline code.

## Related specs
- `docs/design/fx-legacy-removal-spec.md` — original removal plan
- `docs/design/fx-pipeline-spec.md` — fx pipeline architecture
- `docs/superpowers/specs/2026-04-17-fixedto-ctx-propagation.md` — fixedTo/ctx design notes
- Capabilities spec (read from `/tmp/legacy-removal-prompt.md` context) — future unification of activation mechanisms

## Session 2 progress (2026-04-18)

**433/461 tests passing** (was 352/452, merged fix/host-aspects-dedup adding ~328 tests)

### Fixes applied
1. **denTest partial matching** — `builtins.intersectAttrs expected expr` when both are attrsets
2. **denTest trace tree** — leaf nodes wrapped `[displayName]` for correct nesting
3. **`__ctx` conditional** — only set when callable attrsets with unresolved args remain
4. **Filter unresolvable includes** — `lib.isFunction i && !canTake.atLeast allAvailable i` drops at wrong level; `allAvailable = attrs // { class; aspect-chain }` preserves pipeline args
5. **Bare-arg function resolution** — detects `isBareArgFn` (empty functionArgs) and calls `takeFn i attrs` directly for `parametric.exactly/atLeast` results

### Remaining 28 failures
- **8 pre-existing** from original removal work
- **8 parametric-fixedTo** — `take.exactly/upTo` should filter non-function includes but `resolveIncludes` passes them through
- **5 parametric regression** — `parametric.atLeast aspect` as bare function loses includes through `providerFnType.merge` submodule (defaults `includes = []`)
- **4 host-aspects** — new tests from merged branch
- **3 other** — fwd-leaf-option, performance, trace

### Key remaining blocker
`parametric.atLeast/exactly` returns `ctx: bindCtx ...` (bare function). Module system wraps via `providerFnType.merge` → submodule → `includes = []` default. Original includes trapped in functor closure. Attempted `aspect // { __functor }` but caused 69 regressions from module system interaction.

## Resume context

To continue this work:
1. Branch is `feat/rm-legacy`, HEAD is `39493107`
2. 433/461 tests passing
3. Key file: `nix/lib/parametric.nix` — `bindCtx`, `resolveIncludes`, `pipelineArgs`
4. The parametric atLeast/exactly regression is highest-impact (~13 tests)
5. Need a way to make `parametric.atLeast aspect` preserve includes that survives module system processing
