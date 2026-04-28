# Rename FX Adapters to Constraints — Design Spec

**Date:** 2026-04-14
**Branch:** `feat/includes-chain`
**Status:** Draft

## Problem

The fx pipeline reuses the name "adapters" from the legacy pipeline, but the concepts have diverged:

- **Legacy adapters** (`aspects/adapters.nix`): GOF adapter pattern — functions that wrap `resolve.withAdapter` to transform resolution output format (`module`, `structuredTrace`, `filterIncludes`).
- **FX "adapters"** (`aspects/fx/adapters.nix`): A grab-bag containing resolution constraint constructors (`excludeAspect`, `substituteAspect`, `filterAspect`), include-list constructs (`includeIf`), trace handlers, and identity utilities. None of these are adapters in the legacy sense.

The `meta.adapter` field was relaxed to accept both legacy function adapters and fx attrset records. These are different types serving different pipelines and should not share a field.

The fx `adapters.nix` file has grown to contain four unrelated concerns: constraints, includes, tracing, and identity.

## Design

### New `meta.handleWith` field

Replace `meta.adapter` usage in the fx pipeline with `meta.handleWith`. This is the extension point where aspect authors provide handlers that govern resolution of their subtree.

```nix
# Built-in constraint constructors
meta.handleWith = exclude foo;
meta.handleWith = [ (exclude foo) (substitute bar baz) (filterBy pred) ];

# Custom user-defined handlers (future extensibility)
meta.handleWith = myCustomHandler;
```

Accepts a single record, a list of records, or null. The type system enforces this separately from `meta.adapter`.

### `meta.excludes` sugar

Convenience field that expands into `meta.handleWith`:

```nix
# Sugar
meta.excludes = [ foo bar ];

# Equivalent to
meta.handleWith = [ (exclude foo) (exclude bar) ];
```

When both `meta.excludes` and `meta.handleWith` are set, `excludes` appends and takes final say:

```nix
# Both set
meta.handleWith = filterBy pred;
meta.excludes = [ foo ];

# Effective result
handleWith = [ (filterBy pred) (exclude foo) ];
```

### Revert `meta.adapter` typing

Revert `meta.adapter` to accept only legacy function adapters. Remove the relaxed `adapterValue` type that accepts attrsets/lists. The fx pipeline reads `meta.handleWith`, not `meta.adapter`.

### Split `aspects/fx/adapters.nix` into four files

**`nix/lib/aspects/fx/constraints.nix`** — constraint constructors (go in `meta.handleWith`):
- `exclude` (was `excludeAspect`) — with `.global` variant
- `substitute` (was `substituteAspect`) — with `.global` variant
- `filterBy` (was `filterAspect`) — with `.global` variant

**`nix/lib/aspects/fx/includes.nix`** — include-list constructs (go in `includes = [...]`):
- `includeIf` — conditional inclusion (existing)
- `includeAll` — stub with comment, transitive includes of all provides (future, separate spec)

**`nix/lib/aspects/fx/trace.nix`** — observation handlers:
- `structuredTraceHandler`
- `tracingHandler`

**`nix/lib/aspects/fx/identity.nix`** — path and identity utilities:
- `aspectPath`
- `pathKey`
- `toPathSet`
- `tombstone`
- `pathSetHandler`
- `collectPathsHandler`

### Naming renames

| Current | New |
|---------|-----|
| `adapters.excludeAspect` | `constraints.exclude` |
| `adapters.substituteAspect` | `constraints.substitute` |
| `adapters.filterAspect` | `constraints.filterBy` |
| `adapters.excludeAspect.global` | `constraints.exclude.global` |
| `adapters.substituteAspect.global` | `constraints.substitute.global` |
| `adapters.filterAspect.global` | `constraints.filterBy.global` |
| `adapters.includeIf` | `includes.includeIf` |
| `adapterRegistryHandler` | `constraintRegistryHandler` |
| `register-adapter` effect | `register-constraint` effect |
| `check-exclusion` effect | `check-constraint` effect |
| `state.adapterRegistry` | `state.constraintRegistry` |
| `state.adapterFilters` | `state.constraintFilters` |
| `meta.adapter` (fx usage) | `meta.handleWith` |
| `hasAdapter` (trace entry) | `handlers` (trace entry, carries data not boolean) |

### Trace entry `handlers` field

Replace the boolean `hasAdapter` field with `handlers` that carries the actual handler data:

```nix
entry = {
  name = ...;
  handlers = param.meta.handleWith or [];
  # consumers check handlers != [] for boolean
};
```

This gives diagram generators access to *what* handlers an aspect declares, not just whether it has any.

### Changes by file

#### `nix/lib/aspects/types.nix`

**Revert `meta.adapter` to function-only:**

```nix
options.adapter = lib.mkOption {
  description = "Legacy adapter function for resolution";
  type = lib.types.nullOr (lib.types.mkOptionType {
    name = "adapterFunction";
    description = "function adapter";
    check = lib.isFunction;
    merge = _: defs: (lib.last defs).value;
  });
  default = null;
};
```

**Add `meta.handleWith`:**

```nix
options.handleWith = lib.mkOption {
  description = "Resolution handlers for this aspect's subtree";
  type = lib.types.nullOr (lib.types.mkOptionType {
    name = "handlerValue";
    description = "handler record or list of handler records";
    check = v: builtins.isAttrs v || builtins.isList v;
    merge = _: defs: (lib.last defs).value;
  });
  default = null;
};
```

**Add `meta.excludes` sugar:**

```nix
options.excludes = lib.mkOption {
  description = "Aspects to exclude from this subtree (sugar for handleWith)";
  type = lib.types.listOf lib.types.unspecified;
  default = [];
};
```

Config merge: when `excludes` is non-empty, append `map exclude excludes` to `handleWith`. `excludes` entries come last (final say):

```nix
config.meta.handleWith = lib.mkIf (config.meta.excludes != []) (
  let
    existing = config.meta.handleWith or [];
    existingList = if builtins.isList existing then existing
                   else if existing != null then [ existing ]
                   else [];
  in
  existingList ++ map exclude config.meta.excludes
);
```

#### `nix/lib/aspects/fx/constraints.nix` (new, from adapters.nix)

Contains:
```nix
exclude = {
  __functor = _: ref: { type = "exclude"; scope = "subtree"; identity = pathKey (aspectPath ref); };
  global = ref: { type = "exclude"; scope = "global"; identity = pathKey (aspectPath ref); };
};

substitute = {
  __functor = _: ref: replacement: { type = "substitute"; scope = "subtree"; ... };
  global = ref: replacement: { type = "substitute"; scope = "global"; ... };
};

filterBy = {
  __functor = _: pred: { type = "filter"; scope = "subtree"; predicate = pred; };
  global = pred: { type = "filter"; scope = "global"; predicate = pred; };
};
```

Imports `identity.nix` for `aspectPath` and `pathKey`.

#### `nix/lib/aspects/fx/includes.nix` (new, from adapters.nix)

Contains `includeIf` (existing) and a stub for `includeAll`:

```nix
includeIf = guardFn: aspects: {
  name = "<includeIf>";
  meta = {
    conditional = true;
    guard = guardFn;
    aspects = aspects;
  };
  includes = [];
};

# TODO: Transitive includes — include all provides from an aspect.
# Design spec pending. Usage: includes = [ (includeAll foo.provides) ];
# includeAll = provides: { ... };
```

#### `nix/lib/aspects/fx/trace.nix` (new, from adapters.nix)

Contains `structuredTraceHandler` and `tracingHandler`. Both updated:
- `hasAdapter` field in trace entries replaced with `handlers = param.meta.handleWith or []`

#### `nix/lib/aspects/fx/identity.nix` (new, from adapters.nix)

Contains: `aspectPath`, `pathKey`, `toPathSet`, `tombstone`, `pathSetHandler`, `collectPathsHandler`.

Pure utilities — no dependency on constraints, includes, or trace.

#### `nix/lib/aspects/fx/handlers.nix`

- Rename `adapterRegistryHandler` to `constraintRegistryHandler`
- Rename effect names: `register-adapter` → `register-constraint`, `check-exclusion` → `check-constraint`
- Rename state keys: `adapterRegistry` → `constraintRegistry`, `adapterFilters` → `constraintFilters`

#### `nix/lib/aspects/fx/resolve.nix`

- Read `meta.handleWith` instead of `meta.adapter` (line ~173)
- `wrapIdentity` (line ~41): carry both `adapter = meta.adapter or null` AND `handleWith = meta.handleWith or null` in the identity envelope. Legacy pipeline reads `adapter`, fx pipeline reads `handleWith`. Both fields preserved through `withIdentity`/`wrapIdentity` until legacy removal.
- Emit `register-constraint` instead of `register-adapter` (line ~187)
- Emit `check-constraint` instead of `check-exclusion` (line ~247)
- Update `defaultHandlers` to use `constraintRegistryHandler`
- Update `defaultState`: `constraintRegistry`, `constraintFilters`
- Update all `adapters.*` references to use `identity.*` for path/identity utilities (`adapters.pathKey` → `identity.pathKey`, `adapters.aspectPath` → `identity.aspectPath`, `adapters.tombstone` → `identity.tombstone`)

#### `nix/lib/parametric.nix`

- `withIdentity` (line ~17): carry both `adapter = meta.adapter or null` AND `handleWith = meta.handleWith or null`. Same dual-carry as `wrapIdentity`.

#### `nix/lib/aspects/fx/default.nix`

Update imports and exports:
```nix
# Pure exports (no nix-effects)
inherit (pureConstraints) exclude substitute filterBy;
inherit (pureIdentity) aspectPath pathKey toPathSet tombstone;
inherit (pureIncludes) includeIf;

# init exports
inherit (constraints) exclude substitute filterBy;
inherit (identity) aspectPath pathKey toPathSet tombstone;
inherit (identity) pathSetHandler collectPathsHandler;
inherit (includes) includeIf;
inherit (trace) structuredTraceHandler tracingHandler;
inherit (handlers) constraintRegistryHandler;
```

Access path for consumers: `fxLib.exclude`, `fxLib.substitute`, `fxLib.filterBy`, `fxLib.includeIf`, `fxLib.aspectPath`, etc. (flat namespace, not nested under module names).

The sub-modules are also accessible for consumers that prefer namespaced access: `fxLib.constraints.exclude`, `fxLib.includes.includeIf`, etc.

#### `nix/lib/aspects/adapters.nix` (legacy)

- Revert any changes that accepted fx attrset records in `filterIncludes`
- `filterIncludes`: update comment to clarify legacy function adapters only
- `structuredTrace`: `hasAdapter` field stays as-is — it checks `meta.adapter` (legacy function) which is correct for the legacy pipeline. No rename needed here.
- Legacy file retains its own `aspectPath`, `pathKey`, `toPathSet`, `collectPaths` — these are independent implementations used by the legacy pipeline. Not affected by the fx split.

#### All test files

Files that need updating (exhaustive list):

**Rename file:**
- `templates/ci/modules/features/fx-adapters.nix` → `fx-constraints.nix`

**Update constructor/handler/effect/state names:**
- `templates/ci/modules/features/fx-constraints.nix` (renamed from fx-adapters.nix)
- `templates/ci/modules/features/fx-adapter-integration.nix`
- `templates/ci/modules/features/fx-effectful-resolve.nix`
- `templates/ci/modules/features/fx-includeIf.nix`
- `templates/ci/modules/features/fx-parametric-meta.nix`
- `templates/ci/modules/features/fx-trace.nix`
- `templates/ci/modules/features/fx-e2e.nix`
- `templates/ci/modules/features/fx-full-pipeline.nix`
- `templates/ci/modules/features/fx-identity.nix`

**Update `fxLib.adapters.*` access paths:**
All above files that reference `fxLib.adapters.excludeAspect`, `fxLib.adapters.includeIf`, `fxLib.adapters.pathSetHandler`, etc. → update to flat `fxLib.exclude`, `fxLib.includeIf`, `fxLib.identity.pathSetHandler`, etc.

**Unaffected test files** (legacy only, use `meta.adapter` with function adapters):
- `templates/ci/modules/features/adapter-propagation.nix`
- `templates/ci/modules/features/adapter-owner.nix`
- `templates/ci/modules/features/identity-preservation.nix` (uses `meta.adapter` but for legacy function — verify and leave unchanged)

#### Template/consumer files using `meta.adapter` with fx records

Any aspect definitions that use `meta.adapter = excludeAspect ref` must change to `meta.handleWith = exclude ref`. Grep for all `meta.adapter` usages that pass attrset records (not functions) and update.

### What stays unchanged

- **Legacy `adapters.nix`** (`aspects/adapters.nix`) — untouched except reverting the relaxed typing
- **`resolve.nix` structure** — same logic, renamed effect names and state keys
- **`chainHandler`** — unaffected (chain effects are orthogonal)
- **`ctx-apply.nix`** — unaffected
- **`has-aspect.nix`** — uses legacy `adapters.aspectPath`/`pathKey`/`toPathSet`, which remain in legacy `adapters.nix`
- **All resolution behavior** — purely a rename + split, no behavioral changes

### Migration path

1. Create `identity.nix`, `constraints.nix`, `includes.nix`, `trace.nix` from `fx/adapters.nix`
2. Update `fx/default.nix` imports and exports
3. Rename handler, effect names, state keys in `handlers.nix`
4. Update `resolve.nix` to use new names and import `identity` instead of `adapters`
5. Update `wrapIdentity` and `parametric.nix` `withIdentity` to carry both `adapter` and `handleWith`
6. Add `meta.handleWith` and `meta.excludes` to type system
7. Revert `meta.adapter` to function-only
8. Update all consumers of `meta.adapter` that use fx records to `meta.handleWith`
9. Rename/update all test files
10. Delete `fx/adapters.nix`
11. Verify 476/476 tests pass

### Verification

```
nix develop -c just ci ""
```

All tests must pass. No behavioral changes — this is a pure rename + split + type separation.
