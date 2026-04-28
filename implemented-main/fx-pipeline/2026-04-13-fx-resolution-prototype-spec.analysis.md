# Analysis: Effects-Based Resolution Prototype

## Verdict
superseded

## Delivery Target
(A) main — landed in `refactor: den-fx (#462)` (68c26555), then completed by `refactor: remove legacy resolver, stabilize fx pipeline` (2e0fc9e1).

## Evidence

**Commits:**
- `68c26555` — `refactor: den-fx (#462)` — initial landing of `nix/lib/aspects/fx/` directory with all core files
- `2e0fc9e1` — `refactor: remove legacy resolver, stabilize fx pipeline` — deleted legacy resolver, fx pipeline becomes sole path
- `a97f5810` — `test: static aspect, cross-entity parametric, parenthesized two-layer` — early fx test cases
- `52f0acc4` — `fix: include-level dedup prevents duplicate aspect resolution`
- `6839a020` — `fix: resolve effects carry only new bindings, dedup uses merged context`
- `7b116381` — `refactor: address code review — merge excludeChild, fix stale comments`

**Files (on main):**
- `nix/lib/aspects/fx/aspect.nix` — `aspectToEffect` compiler (analogous to spec's `wrapAspect` + `resolveDeep`)
- `nix/lib/aspects/fx/pipeline.nix` — `fxResolve`, `fxFullResolve`, `runSubPipeline`, `mkPipeline`
- `nix/lib/aspects/fx/identity.nix` — `aspectPath`, `pathKey`, `tombstone`, `collectPathsHandler`, `pathSetHandler`
- `nix/lib/aspects/fx/handlers/ctx.nix` — `constantHandler`, `ctxSeenHandler`
- `nix/lib/aspects/fx/handlers/include.nix` — `includeHandler` (handles `emit-include`, dedup, constraint dispatch)
- `nix/lib/aspects/fx/handlers/tree.nix` — `register-constraint`, `check-constraint`, `emit-class`
- `nix/lib/aspects/fx/handlers/trait.nix` — `emit-trait`, `traitArgHandler`, `traitCollectorHandler`
- `nix/lib/aspects/fx/handlers/transition.nix` — `into-transition`
- `nix/lib/aspects/fx/includes.nix` — `includeIf` (conditional inclusion via guard)
- `nix/lib/aspects/fx/constraints.nix` — constraint registry
- `nix/lib/aspects/fx/trace.nix` — tracing handler

**`bind.fn` is live:** `fx.bind.fn userArgs fn` in `aspect.nix:928`

**Effect names in use (shipped):**
- `emit-include`, `emit-class`, `emit-trait`, `register-constraint`, `check-constraint`, `resolve-complete`, `resolve-entity`, `into-transition`, `chain-push`, `chain-pop`, `ctx-seen`, `get-path-set`, `defer-include`, `emit-forward`, `dispatch-policy-includes`

**Tests (in `templates/ci/modules/features/`):**
- `fx-aspect.nix`, `fx-resolve.nix`, `fx-effectful-resolve.nix`, `fx-integration.nix`, `fx-e2e.nix`, `fx-regressions.nix`, `fx-identity.nix`, `fx-ctx-apply.nix`, `fx-handlers.nix`, `fx-constraints.nix`, `fx-adapter-integration.nix`, `fx-includeIf.nix`, `fx-diag-context.nix`, `fx-diag-capture.nix`, `fx-parametric-meta.nix`
- Regression tests: `deadbugs/issue-413-*`, `deadbugs/issue-423-*`, `deadbugs/issue-442-*`, `deadbugs/issue-448-*`, `deadbugs/issue-460-*`

## Current Status

Fully shipped and is the sole resolution path on main. The legacy resolver (`parametric.nix` recursive path) was removed in `2e0fc9e1`. `fxResolve` is the live entrypoint used by `resolveEntity` / `den.lib.aspects.fx.pipeline`.

## Supersession

This spec was the initial prototype spec. It was superseded in practice by the full integration — there was no separate "integration spec" document; the prototype scope was merged directly with integration work in the `den-fx` PR (#462). The legacy removal spec (`docs/design/fx-legacy-removal-spec.md`, written at `2e0fc9e1`) covers the swap and equivalence validation.

## Gaps

**Proposed but not implemented as-spec'd:**

1. **`adapters.nix` file** — never created. Spec listed it as a "stretch goal". Adapter behavior (constraints, excludes, substitutions) landed in `handlers/tree.nix` and `handlers/include.nix` via `check-constraint` / `exclude` / `substitute` decision actions. The `excludeHandler` combinator pattern from the spec was not used; constraint decisions are centralized instead.

2. **`resolve-include` effect name** — never used. The actual effect emitted for includes is `emit-include`. The spec proposed `resolve-include` for the deduplication handler; shipped impl uses `includeSeen` state inside `includeHandler` directly, with `include-unseen` for rollback on exclusion.

3. **`provide-class` effect name** — never used. Class emission uses `emit-class` effect to a collector handler (`classCollectorHandler`). There is no `provide-class` effect.

4. **`templates/ci/modules/features/fx/` test subdirectory** — spec called for a dedicated `fx/` subdir; tests landed flat in `features/` as `fx-*.nix` files instead.

5. **`resolveOne` / `wrapAspect` / `wrapIdentity` as named top-level functions** — these names do not exist. The shipped decomposition is: `aspectToEffect` (covers both `wrapAspect` and `resolveDeep`'s recursive logic), and identity envelope construction is folded into `compileStatic` via `emitClasses`/`emitTraits`. There is no separate `wrapIdentity` function; owned config is accumulated into `state.classImports` rather than being assembled on a per-aspect struct.

6. **Two-layer `fx.rotate` / `fx.handle` topology** — spec described explicit inner `fx.rotate` + outer catch-all error handler. Shipped impl uses a flat `fx.handle` with a composed handler set and per-arg `fx.effects.hasHandler` probing (`keepChild` in `include.nix:186`). Unknown required args result in `defer-include` rather than a synchronous throw; the error/diagnostic path fires as a warning via `wrapClassModule` for missing den args.

7. **`deduplicationHandler` using handler state via `resolve-include` effect** — documented as a stretch goal, noted as "implemented during integration". Shipped dedup is in-handler state in `includeHandler` (`includeSeen` in `state`), accessed directly rather than via a separate named effect. Functionally equivalent to what the spec described.

## Drift

**Architecture divergence (intent preserved, shape changed):**

1. **`aspectToEffect` replaces `wrapAspect` + `resolveDeep` + `resolveOne`** — the spec's three-function decomposition (translate, recurse, recurse-one) collapsed into a single recursive function. `aspectToEffect` calls itself after `bind.fn` resolves a parametric layer (`compileStatic` / `mkParametricNext` / `tagParametricResult`). The spec's `resolveIncludes` map-over-children is replaced by `emitIncludes` which emits per-child `emit-include` effects, with the `includeHandler` recursing via effectful resume.

2. **Identity envelope not a separate struct** — spec's `wrapIdentity` assembled `{ name, meta, includes, <owned> }`. Shipped pipeline accumulates class modules into `state.classImports` keyed by class name. The "identity" concept maps to `aspectPath` / `pathKey` used for dedup and tracing, not a per-resolved-aspect wrapper.

3. **Factory-function detection evolved** — spec: detect by `functionArgs aspect == {}`, apply full ctx. Shipped: `__fn` / `__args` on the normalized aspect attrset; `isParametric = userArgs != {}` in `aspectToEffect`. Factory-like aspects (empty args) fall through to `compileStatic` which processes their class/trait keys directly. The `wrapAspect ctx` factory path does not exist as described.

4. **`parametricHandler` is `constantHandler`** — the spec's `builtins.mapAttrs` over `ctx` to build handlers is exactly what `constantHandler` does (`handlers/ctx.nix:18`). Name changed, semantics identical.

5. **`staticHandler` for `class` / `aspect-chain`** — in shipped impl, `class` and `aspect-chain` are injected into `constantHandler ctx` directly (pipeline.nix `defaultHandlers` merges `{ inherit class; "aspect-chain" = []; } // ctx`). There is no separate `staticHandler`. Effect is identical.

6. **`onlyIf` / `hasAspect`-in-includes** — spec deferred these as post-integration exercises. Shipped as `includeIf` (in `includes.nix`) with `hasAspect` guard that reads from `state.pathSet` via `get-path-set` effect. Landed in the den-fx PR itself, not post-integration.

7. **State thunks** — spec assumed `state = {}` is adequate for stateless handlers. Shipped impl wraps large state values in thunks (`_: value`) throughout `defaultState` to prevent `deepSeq` forcing at every trampoline step. Documented in `pipeline.nix:122`.

8. **Test org** — spec called for two tiers: "Core resolution tests" (12 cases) and "Targeted regression reproductions" (4 patterns). Shipped test org is flat `fx-*.nix` with regressions in `deadbugs/` alongside all pre-existing regression tests, not a dedicated fx/ hierarchy.
