# Provides Cleanup: From Compat Shim to First-Class Feature

**Date:** 2026-05-04
**Branch:** feat/fx-pipeline
**Status:** Design approved, implementation pending
**CI:** 629/629
**Prerequisite:** None — independent of all other remaining work items

## 1. Context

The `provides`/`_` namespace on aspects is an established user-facing API for:

1. **Cross-entity content delivery** — `_.to-users`, `_.to-hosts` route content between hosts and users
2. **Sub-aspect organization** — `_.enable`, `_.config`, `_.gaming` define selectable sub-components
3. **Named targeting** — `_.alice`, `_.igloo` deliver content to specific entities
4. **Self-referencing** — `den.aspects.steam.includes = with den.aspects.steam._; [ enable ];`

On the main branch, `mutual-provider` was a parametric aspect that handled cross-entity routing by reading `provides.*` keys and including them with the right entity context. Users included `den._.mutual-provider` in their defaults to enable this.

On feat/fx-pipeline, the transition system was eliminated and cross-entity routing was reimplemented as `provides-compat.nix` — a pipeline handler that translates `provides.*` keys into `policy.include` effects during the aspect walk. This handler was written as a temporary shim under the assumption that `provides` would be removed entirely. That assumption was wrong: `provides`/`_` is a permanent API.

### What's wrong with the current implementation

1. **Named "compat" with deprecation warnings** — `lib.warn` messages tell users to migrate away from a permanent feature
2. **Duplicated shape detection** — `applyProvide` in `provides-compat.nix` and `emitSelfProvide` in `include-emit.nix` independently detect `__fn`/`__functor`/bare function shapes
3. **`mkToHostsPolicy` and `mkToUsersPolicy` are identical** — zero behavioral distinction despite different names
4. **`emitSelfProvide` is a separate phase** — `provides.${name}` (self-provide) is used in the `provider` template and CI fixtures; it should fold into `emitAspectPolicies` as an auto-include rather than being a standalone pipeline phase
5. **`"compat:"` prefix on policy names** — persists in traces and debugging
6. **`resolveChildren` has 5 phases** where 3 suffice — two phases exist solely for provides "compat"
7. **`mutual-provider-shim.nix` references a cancelled spec** — warns users to remove a feature that's built-in

## 2. Design Principles

- **`provides`/`_` is a permanent user-facing API** — no deprecation, no migration required
- **New users should use policies and direct nesting** — `provides` is supported but not promoted in new documentation
- **Cross-entity routing is a pipeline concern discovered during the walk** — each aspect contributes its provides-derived policies as the tree is walked; policies are dispatched at entity resolution time when full context is available
- **The pipeline stays generic** — no `to-users`/`to-hosts`-specific structural keys; these remain freeform keys under `_`/`provides`

## 3. The `_`/`provides` Namespace

`_`/`provides` is a **virtual sub-aspect namespace** — a transparent container whose children are real sub-aspects. It exists for:

- **Addressability** — `den.aspects.steam._` gives you the sub-aspect collection
- **Enumerability** — `lib.attrValues den.aspects.security._` includes all children
- **Merge target** — `den.aspects.gwen-t1._.to-users.niri.settings` from separate files
- **Collision isolation** — sub-aspect names can't clash with class keys or structural keys

The module system keeps `provides` as a declared submodule option with `providerType` freeform children. `_` aliases to `provides` via `mkAliasOptionModule`. Both remain in `structuralKeysSet` so the pipeline skips them during key classification.

No changes to the type system or alias mechanism.

## 4. Folding Provides Into `emitAspectPolicies`

### Current `resolveChildren` flow (5 phases)

```
emitSelfProvide → emitCrossProvideShims → emitAspectPolicies → emitIncludes → installPolicies
```

### New `resolveChildren` flow (3 phases)

```
emitAspectPolicies → emitIncludes → installPolicies
```

`emitAspectPolicies` (in `include-emit.nix`) currently registers `aspect.policies.*` entries. After this change, it also scans `aspect.provides or {}` and registers policies for cross-entity keys.

### Policy registration logic

For each key in `aspect.provides` (excluding the aspect's own name and schema entity kinds):

| Key | Policy behavior |
|-----|----------------|
| `to-users` | Fires when `{ host, user }` in context → `policy.include` with the value applied to context |
| `to-hosts` | Fires when `{ host, user }` in context → `policy.include` with the value applied to context |
| `<name>` (named target) | Fires when `host.name == key \|\| user.name == key` → `policy.include` with the value |

Note: `to-users` and `to-hosts` have identical dispatch behavior — both fire when host+user context is available. The names are conventional (documenting intent for the user), not behavioral.

### Shape detection

The value under a provides key can be:
- A plain attrset (`{ homeManager.programs.vim.enable = true; }`)
- A function (`{ host, user, ... }: { ... }`)
- A parametric wrapper (`__fn`/`__functor`)
- An aspect with `includes`

A single `applyProvide` function handles all shapes, extracted into `content-util.nix` to eliminate the current duplication between `provides-compat.nix` and `emitSelfProvide`.

### Why this must be in the pipeline (not a schema policy)

Provides keys live on aspects scattered throughout the include tree, not just the entity root. When the pipeline resolves `igloo` and walks its includes into `desktop`, `desktop`'s `_.to-users` must be discovered and registered as a policy function.

`emitAspectPolicies` runs during `resolveChildren` for each aspect as it is visited. It **registers** policy functions into `state.scopedAspectPolicies` — it does not dispatch them. Later, `installPolicies` **dispatches** all accumulated policies at entity boundaries when full context (`{ host, user }`) is available.

A schema-level policy only sees `host.aspect._` (the root), missing provides from included aspects. You can't dispatch provides-derived policies from aspects that haven't been visited yet.

This is a fundamental constraint of tree-structured discovery: walk to discover, then dispatch. The policy scoping redesign (consolidated spec Section 9) changes dispatch behavior (where/when policies fire), not registration timing (when policies become known). The two are orthogonal.

## 5. Concrete Changes

### Delete

| Component | File | Reason |
|-----------|------|--------|
| `provides-compat.nix` | `handlers/provides-compat.nix` | Entire file — functionality moves into `emitAspectPolicies` |
| `emitCrossProvideShims` import | `handlers/default.nix` | No longer exported |
| `emitCrossProvideShims` import + call | `aspect.nix` | Phase removed from `resolveChildren` |
| `emitSelfProvide` call + bind | `aspect.nix` resolveChildren | Phase removed (self-provide folded into `emitAspectPolicies`) |
| `mkPositionalInclude` | `include-emit.nix` | Replaced by unified `applyProvide` in `emitAspectPolicies` |
| `mkNamedInclude` | `include-emit.nix` | Replaced by unified `applyProvide` in `emitAspectPolicies` |
| `emitSelfProvide` export | `aspect.nix` export block (lines 310-317) | No longer exists |

### Modify

| Component | File | Change |
|-----------|------|--------|
| `emitAspectPolicies` | `include-emit.nix` | After registering `policies.*`, scan `provides or {}` and: (a) for `provides.${name}` (self-provide), emit as auto-include via `emit-include`; (b) for cross-entity/named keys, register as aspect policies |
| `resolveChildren` | `aspect.nix` | Remove selfProvide and crossProvideShims bind phases; simplify to 3-phase chain |
| `mutual-provider-shim.nix` | `modules/compat/` | Remove deprecation warning; keep as inert no-op for users who include it |

Note: `emitAspectPolicies` must import `schemaEntityKinds` from `den.lib.schemaUtil` for the schema-kinds filter (currently done in `provides-compat.nix`).

### Extract

| Component | From | To | Purpose |
|-----------|------|----|---------|
| `applyProvide` | `provides-compat.nix` | `content-util.nix` | Shared shape detection for provides values |

### No changes

| Component | Reason |
|-----------|--------|
| `provides` option in `types.nix` | Permanent user API |
| `_` alias in `types.nix` | Permanent user API |
| `structuralKeysSet` | `provides` and `_` stay structural |
| `aspectKeyType` | Separate work item (Section 7.3/8) |
| Template files | No migration — provides syntax unchanged |
| `den.provides.*` factory namespace | Unrelated — these are pre-built aspect factories |

## 6. Test Impact

### Tests that should pass unchanged

All existing provides tests (`provides-parametric`, `cross-provider`, `deadbugs-cybolic-routes`, `user-host-mutual-config`, etc.) should pass unchanged — the behavior is identical, only the internal path changes.

Self-provide is folded into `emitAspectPolicies` (not deleted), so tests exercising self-provide behavior should also pass. However, tests that assert on internal implementation details (import counts, policy names in traces) will need updating.

### Tests to update

| File | Test | Impact |
|------|------|--------|
| `fx-trace.nix` | `test-self-provide-traced` | Expects `importCount = 3` — verify count still matches after self-provide moves to `emitAspectPolicies` |
| `fx-ctx-apply.nix` | `test-self-provide`, `test-self-provide-absent` | May reference `emitSelfProvide` internals — verify or update |
| `fx-e2e.nix` | `test-self-provider` | Synthetic aspects with self-provide — verify passes |
| `nested-aspects.nix` | `test-provides-backward-compat` | Self-provide on `igloo` — should pass unchanged since self-provide behavior is preserved |
| `fx-coverage.nix` | `test-self-provide-host-provider` | Reads `provides.igloo` from resolved aspect — likely passes unchanged (type system untouched) |
| `provider/modules/den.nix` | `provider.simple.provides.simple` | Self-provide in provider template — must continue working |

### Policy name changes

The `"compat:"` prefix on policy names (e.g., `"igloo/compat:to-users"`) changes to a clean name (e.g., `"igloo/to-users"`). Any test assertions matching on policy names in trace output will need updating.

### Verification

Full CI (629/629) must pass after the change. The two test failures noted in the old provides-removal spec (`deadbugs-cybolic-routes.test-has-no-dups`, `user-host-mutual-config.test-host-parametric-unidirectional`) may be affected — investigate if they were caused by provides-compat interaction bugs.

## 7. Real-World Compatibility

Verified against `gwenodai-nixos` user config (representative real-world usage):

| Pattern | Status |
|---------|--------|
| `_.to-users = { includes = [...]; }` | Works — `emitAspectPolicies` reads it |
| `den.aspects.gwen-t1._.to-users.niri.settings = { ... }` (deep-set from separate file) | Works — module system merges, pipeline reads merged value |
| `_.default`, `_.config`, `_.impermanence` (sub-aspects) | Unaffected — sub-aspects are not cross-entity keys |
| `includes = with den.aspects.steam._; [ enable ];` (self-reference) | Unaffected — `_` submodule stays |
| `den._.mutual-provider` in global includes | Harmless no-op — routing is built-in |
| `lib.attrValues den.aspects.security._` | Unaffected — `_` submodule stays enumerable |

## 8. Future Direction

- **New users** should use `policies.*` for cross-entity delivery and direct nesting for sub-aspects
- **Policy scoping redesign** (consolidated spec Section 9) is independent and could eventually allow cross-entity routing to move from pipeline-native to policy-installed-handler
- **Traits** will supersede the sub-aspect nesting pattern (`_.<sub-aspect>`) for semantic data channels
- **`den.provides.*` factory namespace** is unrelated and unaffected

## 9. Dependency

This work is independent of:
- Forward elimination (consolidated spec Section 6)
- Constraint handler simplification (Section 7.2)
- providerType dispatch / aspect key type (Sections 7.3, 8)
- Policy scoping redesign (Section 9)

No blocking dependencies in either direction.
