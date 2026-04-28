# Diag Branch Rebase onto Unified Effects Pipeline

**Date:** 2026-04-15
**Branches:** `feat/diag-api` and `feat/diag-templates`
**Rebase target:** `feat/fx-resolution` (unified pipeline, 486/486 green)
**Status:** Spec

## Context

`feat/diag-api` and `feat/diag-templates` are based on commit `60269db3` (pre-unified-pipeline). `feat/fx-resolution` has 8 new commits implementing the unified effects pipeline:

1. Module wiring: `{ lib, den }` only, no `init` function
2. Handler/effect renames: `constantHandler`, `emit-class`, `emit-include`
3. `aspectToEffect` — the aspect compiler
4. `emit-include` + `into-transition` effectful handlers
5. Context transitions in `aspectToEffect`
6. Pipeline rewrite: `mkPipeline` uses `aspectToEffect`
7. All test updates
8. Cleanup

## Breaking changes the diag code must adapt to

### 1. No `init` function

**Old:** `fxLib = den.lib.aspects.fx.init inputs.nix-effects.lib;`
**New:** Access modules directly via `den.lib.aspects.fx.*`. No `init` call needed.

**Files affected:**
- `nix/lib/diag/default.nix:51` — `den.lib.aspects.fx.init`
- `nix/lib/diag/capture.nix` — all `fxLib.*` references

### 2. No `resolve` module (merged into `pipeline` + `aspect`)

**Old:** `fxLib.resolve.resolveDeepEffectful`, `fxLib.resolve.composeHandlers`, `fxLib.resolve.defaultHandlers`, `fxLib.resolve.defaultState`
**New:**
- `den.lib.aspects.fx.aspect.aspectToEffect` (replaces `resolveDeepEffectful` + `resolveAspect`)
- `den.lib.aspects.fx.pipeline.composeHandlers`
- `den.lib.aspects.fx.pipeline.defaultHandlers`
- `den.lib.aspects.fx.pipeline.defaultState`

### 3. `defaultHandlers` already includes the include handler

**Old:** Had to compose `resolve.resolveIncludeHandler` separately.
**New:** `defaultHandlers` already includes `handlers.includeHandler`. No separate composition needed.

### 4. Effect name changes

| Old | New |
|-----|-----|
| `provide-class` | `emit-class` |
| `resolve-include` | `emit-include` |

### 5. Handler name changes

| Old | New |
|-----|-----|
| `provideClassHandler` | `classCollectorHandler { targetClass ? null }` |
| `parametricHandler` | `constantHandler` |

### 6. `trace.nix` takes `{ lib, den, ... }` not `{ lib, den, fx, identity, ... }`

The trace module's args changed. But since diag imports it indirectly via `den.lib.aspects.fx.trace`, this shouldn't matter — the import is handled by `default.nix`.

## Changes needed per diag file

### `nix/lib/diag/default.nix`

Line 51: Replace
```nix
fxLib = if fxEnabled && inputs ? nix-effects then den.lib.aspects.fx.init inputs.nix-effects.lib else null;
```
with:
```nix
fxLib = if fxEnabled && inputs ? nix-effects then den.lib.aspects.fx else null;
```

Or even simpler — since there's no init, `fxLib` can just be `den.lib.aspects.fx` when enabled. The downstream code accesses `fxLib.trace.tracingHandler`, `fxLib.identity.toPathSet`, etc. — all of which exist at `den.lib.aspects.fx.*` directly.

### `nix/lib/diag/capture.nix`

The `fxCaptureWithPathsWith` function needs a full rewrite of its handler composition:

**Old:**
```nix
resolve = fxLib.resolve.resolveDeepEffectful { ctx = {}; inherit class; };
comp = nxFx.bind (resolve.resolveAspect root) (...);
handlers = fxLib.resolve.composeHandlers
  (fxLib.resolve.defaultHandlers { ... } // resolve.resolveIncludeHandler)
  (fxLib.trace.tracingHandler class // fxLib.handlers.ctxTraceHandler // extraHandlers);
state = fxLib.resolve.defaultState // { ... };
```

**New:**
```nix
comp = nxFx.bind (fxLib.aspect.aspectToEffect root) (
  resolved: nxFx.bind (nxFx.send "resolve-complete" resolved) (_: nxFx.pure resolved)
);
handlers = fxLib.pipeline.composeHandlers
  (fxLib.pipeline.defaultHandlers { inherit class; ctx = {}; })
  (fxLib.trace.tracingHandler class // fxLib.handlers.ctxTraceHandler // extraHandlers);
state = fxLib.pipeline.defaultState // { ... };
```

Key differences:
- `aspectToEffect` replaces `resolveDeepEffectful` + `resolveAspect`
- `pipeline.*` replaces `resolve.*`
- No `resolveIncludeHandler` to compose — `defaultHandlers` already includes it
- Root `resolve-complete` still emitted manually (root has no parent handler)

### `nix/lib/diag/context.nix`

References to `capture.fxCaptureWithPaths` should still work if capture.nix is updated correctly. The `fxLib` variable used in context.nix comes from default.nix — once that's fixed, context.nix may need no changes.

Check: `context.nix` references to `fxLib.*` — these should still resolve since `fxLib = den.lib.aspects.fx` which has all sub-modules.

### `templates/diag-fx-demo/modules/fx-debug.nix`

This file on `feat/diag-templates` uses the old API pattern. Same changes as capture.nix:
- `fxLib.resolve.resolveDeepEffectful` → `fxLib.aspect.aspectToEffect`
- `fxLib.resolve.defaultHandlers` → `fxLib.pipeline.defaultHandlers`
- `fxLib.resolve.defaultState` → `fxLib.pipeline.defaultState`
- Remove `resolve.resolveIncludeHandler` from handler composition

### Test file `fx-aspect.nix` on diag-api

This file still uses `fxLib.aspect.wrapAspect` (old API). On the new `feat/fx-resolution`, `wrapAspect` still exists as legacy compat in `aspect.nix`. But the test references `fxLib = den.lib.aspects.fx.init fx` which no longer exists. Update to `den.lib.aspects.fx`.

## Rebase strategy

### Phase 1: Rebase `feat/diag-api` onto `feat/fx-resolution`

```bash
git checkout feat/diag-api
git rebase feat/fx-resolution
```

Expected conflicts:
- `nix/lib/aspects/fx/trace.nix` — old args vs new args (take ours)
- `nix/lib/aspects/fx/default.nix` — barrel vs old init (take ours)
- `nix/lib/aspects/default.nix` — fxResolveTree changes (take ours)
- `nix/lib/diag/capture.nix` — old API calls (manual fix)
- `nix/lib/diag/default.nix` — fxLib init call (manual fix)
- `templates/ci/modules/features/fx-aspect.nix` — old wrapAspect tests (take ours, tests already rewritten)

For conflicts in fx/ files: take ours (`--ours`) since `feat/fx-resolution` has the canonical versions.

For conflicts in diag/ files: manual fix using the change guide above.

### Phase 2: Fix diag callers post-rebase

After rebase, fix any remaining issues:
1. `capture.nix` — update to `aspectToEffect` API
2. `default.nix` — remove `init` call
3. Run CI: `nix develop -c just ci ""`

### Phase 3: Rebase `feat/diag-templates` onto updated `feat/diag-api`

```bash
git checkout feat/diag-templates
git rebase feat/diag-api
```

Expected conflicts:
- `fx-debug.nix` — old API (manual fix, same pattern as capture.nix)
- Template host files using `meta.handleWith` — should be clean (no API changes there)

### Phase 4: Verify

```bash
nix develop -c just ci ""
nix eval --json --override-input den path:. path:./templates/diag-fx-demo#debug
```

All tests pass, parent parity holds.

## Verification criteria

- [ ] All tests pass on `feat/diag-api`
- [ ] All tests pass on `feat/diag-templates`
- [ ] No references to `fxLib.resolve.*` in diag code
- [ ] No references to `init` in diag code
- [ ] No references to `resolveDeepEffectful` in diag code
- [ ] Parent parity test shows fx matches legacy for all 7 test aspects
- [ ] `grep -r "fxLib\.resolve\.\|\.init " nix/lib/diag/` returns no matches
