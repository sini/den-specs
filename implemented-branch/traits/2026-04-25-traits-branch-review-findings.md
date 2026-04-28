# Traits Branch Review Findings

**Date:** 2026-04-25
**Status:** Pre-merge review
**Branch:** feat/traits (19 commits, 633 tests passing)
**Base:** feat/fx-pipeline
**Re-check:** After stages elimination completes

## Important (fix before merge)

### 1. Unregistered-key trace warnings never fire

`nix/lib/aspects/fx/aspect.nix:725-728`

`builtins.seq` forces the list to WHNF (confirms it's a list) but does NOT force individual elements. The `builtins.trace` calls inside each mapped element are never evaluated. Users with unregistered keys silently get no feedback.

**Fix:** Use `builtins.deepSeq` or `builtins.foldl'`:

```nix
_warn = builtins.foldl' (acc: k:
  builtins.seq (builtins.trace "den: ignoring unregistered key '${k}' in aspect '${rawName}'" null) acc
) null unregisteredClassKeys;
```

### 2. `traitCollectorHandler` uses root ctx for tier detection, not scoped context

`nix/lib/aspects/fx/handlers/trait.nix:113`

The `ctx` is captured at handler construction time. After a transition introduces new scoped context (e.g., adding `user`), Tier 2 detection for `{ user }: ...` still checks against the root context. A function whose args are satisfied by scoped context but not root context would be incorrectly classified as Tier 3.

Not a bug in current flow (parametric resolution typically resolves such functions before they reach `emit-trait`), but an architectural limitation. Consider reading `state.currentCtx` inside the handler body rather than captured `ctx`.

**Action:** Document the limitation. Consider fixing during stages elimination if the scoped-context path becomes more common.

## Suggestions (address during or after stages elimination)

### 3. `emitNestedAspect` drops function-valued nested keys

`aspect.nix:682-683` — When `innerValue` is a function, it's silently lost. Add trace warning.

### 4. `hasRecognizedSubKeysAt` forces nested attrset evaluation

`aspect.nix:328-336` — Depth-limited recursive descent forces thunks. Depth limit of 3 is reasonable. Add comment noting the trade-off.

### 5. `partialOk` violation test doesn't actually test the throw

`traits.nix:719-766` — Test verifies deferred data exists but doesn't exercise `fxResolve`'s throw path. Add focused test that triggers the actual throw.

### 6. `traitModule` destructures `modulesPath` unnecessarily

`pipeline.nix:200` — Not used in body. Remove from destructuring, rely on `...`.

### 7. Handler priority comment could be more explicit

`pipeline.nix:48-56` — Add note that trait name colliding with context arg name means the trait is unreachable via `bind.fn`.

### 8. `collectTrait` map fallback wraps non-attrset silently

`trait.nix:85-86` — Plain value emitted into map-collection trait becomes `{ traitName = value; }`. Add trace warning for misuse.

## What was done well

- Three-tier system maps cleanly onto existing effect pipeline
- Empty-registry fallback preserves backward compat
- 633 tests across 8 new test files
- Collision check via `apply` avoids eval cycles
- `distribute-cross-entity.nix` is a clean replacement
- `aspectContentType` double-wrap flattening is good defensive measure

## Verdict

Ready for review with Vic. Item 1 (silent trace) is a one-line fix. Item 2 is an architectural note. Everything else is polish.
