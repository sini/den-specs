# Analysis: Class Module Partial Application

## Verdict

FULLY IMPLEMENTED — with significant scope extension beyond the spec.

## Delivery Target

Branch: `feat/fx-pipeline`
Spec origin branch: `feat/class-module-partial-apply`

## Evidence

Git log (chronological, relevant):

```
0b58dbdc feat: wrapClassModule — pre-apply den context args to class modules
ffb3da3e test: class module partial application integration tests
34afcc56 feat: full application when all class module args are den args
5c2b5955 feat: add meta.collisionPolicy to aspect type
09db1be6 feat: add collisionPolicy option to entity schema
1af28f8a feat: add den.config.classModuleCollisionPolicy option
ff2c0ea8 feat: F(Validate(X),G(X)) collision detection via deferred thunk evaluation
9e90b026 feat: companion module for _module.args collision detection with policy enforcement
9d7505ff fix: validator for deferred-import class modules, ci-deep compatible collision tests
28118e46 feat: trait consumption in class modules via _den.traits injection
6ac38956 fix: guard unsatisfied class module args, skip emission when den schema args missing
```

Implementation file: `nix/lib/aspects/fx/aspect.nix`
Test file: `templates/ci/modules/features/class-module-partial-apply.nix`

## Current Status

Core spec fully shipped:

- `wrapClassModule` exists in `aspect.nix` (lines 97–269), called from `emitClasses` (lines 395–457).
- `builtins.functionArgs` used to detect den args on class modules (line 162).
- Args partitioned dynamically via `ctx ? ${k}` — no hardcoded allowlist (line 164).
- `lib.setFunctionArgs wrapper remainingArgs` used to preserve module-system arg advertising (line 265).
- `isContextDependent = result.wrapped || ...` forced true on wrapped modules (lines 441–443).
- `lib.warn` emitted for missing den args that match schema kinds (lines 177–179).
- Collision policy at three levels: `aspect.meta.collisionPolicy` → `ctx.${name}.collisionPolicy` → `den.config.classModuleCollisionPolicy` (lines 38–54 via `resolveCollisionPolicy`).
- Policy values `"error"` / `"class-wins"` / `"den-wins"` fully implemented (lines 213–258).
- `ctxFromHandlers` reconstructs ctx from `__scopeHandlers` (lines 272–284) — spec said ctx from `aspect.__ctx or {}` but scopeHandlers was the correct mechanism post-fx-pipeline refactor.

Tests cover all 10 spec test cases plus additional regression cases. All test cases in `class-module-partial-apply.nix` are integration tests exercising the full pipeline.

## Supersession

The spec described `ctx = aspect.__ctx or {}` as the source of den context. The implementation uses `ctxFromHandlers (aspect.__scopeHandlers or {})` instead, which reconstructs the context from the scope handler chain. This is a correct adaptation: `__ctx` was removed from the fx-pipeline architecture; `__scopeHandlers` is the canonical context carrier.

## Gaps

None relative to the spec. All specified behaviours are present:
- Flat form with context
- Two-layer form unchanged
- Missing context warning (guarded: only warns for schema-kind args without defaults)
- `functionArgs` preservation via `setFunctionArgs`
- Multiple den args
- Functor module (passthrough — `builtins.isFunction` is false for functors, so no wrapping)
- Collision error/class-wins/den-wins
- Aspect-level collision policy override

The spec noted "missing context → lib.warn, module passes through". The implementation adds a sharper guard: if the missing arg matches a schema kind and has no default, the module is suppressed entirely (`unsatisfied = true`, emission skipped) rather than passed through. This is stricter than the spec but avoids downstream NixOS eval failures.

## Drift

Three significant extensions beyond the spec, all post-dating the spec document:

1. **Full application path** (`34afcc56`): when all `functionArgs` are den args (no module-system args remain), the module is called immediately with den args rather than wrapped. Result is a plain attrset passed to NixOS. Spec only described partial application with `setFunctionArgs`.

2. **Trait consumption in class modules** (`28118e46`): `traitArgNames` partition added alongside `denArgNames`. Trait args (matching `traitRegistry` keys not in ctx) are resolved lazily via `config._den.traits.${name}` thunks at eval time. Spec had no trait-in-class-module path.

3. **Deferred-import (`{ imports = [...]; }`) class module wrapping** (`9d7505ff`, `wrapDeferredImports`): class modules that are attrsets with an `imports` list are recursively descended to find and wrap any inner functions. Spec assumed all class modules are bare functions or attrsets without imports nesting.

4. **Companion collision-detector module** (`9e90b026`): a separate validator module `F(Validate(X), G(X))` is emitted alongside every wrapped module to detect `_module.args` collisions at NixOS fixed-point time (deferred evaluation). The spec described call-time collision detection in `wrapper` itself; the implementation separates the detection concern into a companion to avoid the `_module.args` thunk recursion problem.
