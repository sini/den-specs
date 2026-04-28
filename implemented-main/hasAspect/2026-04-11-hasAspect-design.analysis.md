# Analysis: `entity.hasAspect <ref>` on context entities

## Verdict
implemented

## Delivery Target
(A) main — shipped in PR #439 (commit 326ed590), merged April 13 2026.

## Evidence

- **PR #439** (`326ed590`) — primary delivery commit. Added:
  - `nix/lib/aspects/has-aspect.nix` — lib primitives (`hasAspectIn`, `collectPathSet`, `mkEntityHasAspect`)
  - `modules/context/has-aspect.nix` — `options.hasAspect` on `den.schema.conf`
  - `nix/lib/aspects/adapters.nix` — `collectPaths` and `oneOfAspects` adapters added
  - `nix/lib/aspects/default.nix` — re-exports `hasAspectIn`, `collectPathSet`, `mkEntityHasAspect`
  - Test files: `templates/ci/modules/features/has-aspect.nix`, `has-aspect-lib.nix`, `collect-paths.nix`, `one-of-aspects.nix`
  - Example file: `templates/example/modules/aspects/hasAspect-examples.nix`

- **Inline cleanup** (`bb5dc7f0`, April 23) — `nix/lib/entities/_has-aspect.nix` inlined into `modules/context/has-aspect.nix` and deleted. The spec had no such intermediate lib file; this file appeared in the four-concern refactor (`8fa148bf`) and was immediately cleaned up.

- **Legacy resolver removal** (`2e0fc9e1`, April 23) — `nix/lib/aspects/has-aspect.nix` rewritten to use the fx pipeline (`fxFullResolve` + `state.pathSet`) instead of `resolve.withAdapter adapters.collectPaths`. The `main` branch version still uses the legacy `resolve.withAdapter` path.

- **fx pipeline refactor** (`3a9cea99`, April 23) — `nix/lib/aspects/adapters.nix` deleted entirely (marked legacy-only in `68c26555`); `collect-paths.nix` and `one-of-aspects.nix` test files removed (superseded by fx-coverage.nix).

- **Current files on `feat/fx-pipeline`**:
  - `nix/lib/aspects/has-aspect.nix` — fx-based implementation via `fxFullResolve` + `pathSet` state
  - `modules/context/has-aspect.nix` — entity option wiring (unchanged semantics from main)
  - `templates/ci/modules/features/has-aspect.nix` and `has-aspect-lib.nix` — entity and lib tests, still present
  - `templates/example/modules/aspects/hasAspect-examples.nix` — still present

- **Current files on `main`**:
  - Same `nix/lib/aspects/has-aspect.nix` but using `resolve.withAdapter adapters.collectPaths` (pre-fx version)
  - `nix/lib/aspects/adapters.nix` still exists on main with `collectPaths` and `oneOfAspects`
  - All three test files (`collect-paths.nix`, `one-of-aspects.nix`, `has-aspect.nix`) present on main

## Current Status

On **main**: fully implemented, uses legacy `resolve.withAdapter` + `adapters.collectPaths`. All test files present. `adapters.nix` still exists.

On **feat/fx-pipeline**: reimplemented to use the fx effects pipeline. `pathSet` is accumulated via `collectPathsHandler` (in `identity.nix`) during `fxFullResolve`. The `adapters.nix` file and adapter-based test files (`collect-paths.nix`, `one-of-aspects.nix`) are deleted. The `hasAspect` entity method and lib primitives continue to exist with unchanged public API.

The `identity.nix` fx handler also adds a `baseKey` (path without `__ctxId`) alongside the full key, allowing `hasAspect` to match across fan-out context instances — a capability the spec did not describe, added to handle the fx pipeline's context identity model.

## Supersession

No supersession. This spec is the canonical design for `hasAspect`. The fx pipeline refactor is an internal reimplementation, not a feature replacement.

## Gaps

All three API variants shipped: bare `entity.hasAspect <ref>`, `.forClass`, `.forAnyClass`.

**Spec-proposed `collectPaths` as a public adapter** (`den.lib.aspects.adapters.collectPaths`) — shipped on main; removed on `feat/fx-pipeline` when `adapters.nix` was deleted. The capability is internal to the fx pipeline now, not a public adapter consumers can pass to `resolve.withAdapter`.

**`oneOfAspects` adapter** — shipped on main; deleted on `feat/fx-pipeline`. Its function is subsumed by `meta.handleWith` + constraint handlers (`exclude`, `substitute`) in the fx pipeline, which are the structural-decision primitives at the new layer. The example file on `feat/fx-pipeline` still references `adapters.nix` in a comment but the adapter itself is gone.

**Test groups A–J** — all present in `templates/ci/modules/features/has-aspect.nix` on both branches. On `feat/fx-pipeline`, groups testing `collectPaths` and `oneOfAspects` in isolation (files `collect-paths.nix`, `one-of-aspects.nix`) were removed; adapter coverage migrated into `fx-coverage.nix`.

**`test-H-custom-entity-kind-has-hasAspect`** — spec flagged this as substantial setup (§8.2). Not verified whether this specific test is in the current test file; the option existence test (`test-H-conf-option-exists`) is confirmed present.

**`den.lib.aspects.adapters` public namespace** — on `feat/fx-pipeline`, `den.lib.aspects.adapters` no longer exists (adapters.nix deleted). Code consuming `den.lib.aspects.adapters.collectPaths` or `oneOfAspects` directly would break when the branch lands on main. The `hasAspect-examples.nix` file references `adapters.nix` in a comment that will become stale.

## Drift

**Implementation mechanism diverged significantly on `feat/fx-pipeline`**: the spec described `collectPaths` as a terminal adapter passed to `resolve.withAdapter`, accumulating a `paths` list via `filterIncludes`. The fx pipeline implementation instead accumulates a `pathSet` attrset directly as pipeline state, updated by `collectPathsHandler` on each `resolve-complete` effect. The public API (`hasAspect`, `forClass`, `forAnyClass`) is identical; the internals are completely different.

**`modules/context/has-aspect.nix` wiring**: the spec specified `conf.imports = [ ../context/has-aspect.nix ]` in `modules/options.nix`. What shipped instead is a flake-level module (`modules/context/has-aspect.nix` as a self-contained module file) discovered via the existing import-tree setup, which adds `config.den.schema.conf.imports = [ entityModule ]` inside its own `config` block. Semantically equivalent; structurally different from the spec's proposed `options.nix` edit.

**`_has-aspect.nix` intermediate file**: the spec's `default.nix` aggregator re-exports from `has-aspect.nix` as a lib file. The actual implementation temporarily had a `nix/lib/entities/_has-aspect.nix` containing the module definition (introduced in `8fa148bf`), which was immediately inlined into `modules/context/has-aspect.nix` in `bb5dc7f0`. The spec's proposed file layout (`nix/lib/entities/` as a location) did not match the original PR layout or the final state.

**`ctx` field in `fxFullResolve` call** (feat/fx-pipeline): the `collectPathSet` function in `has-aspect.nix` passes `ctx = den.lib.aspects.fx.aspect.ctxFromHandlers (...)` rather than an empty `ctx = {}`, using scope handlers from `__scopeHandlers` if present. The spec had no concept of scope handlers (predates the fx pipeline).

**`__ctxId` augmentation of aspect paths**: `aspectPath` on `feat/fx-pipeline` appends `{ctxId}` when present, and `collectPathsHandler` stores both the full key and base key (without ctxId) in `pathSet`. This means `hasAspect` can match a fan-out instance by its base path. The spec's `aspectPath` definition (`(meta.provider or []) ++ [name]`) has no ctxId concept; this is a net-additive semantic extension.
