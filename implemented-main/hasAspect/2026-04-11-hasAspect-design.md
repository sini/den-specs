# Design: `entity.hasAspect <ref>` on context entities

**Date:** 2026-04-11
**Status:** Draft — pending spec review and user approval
**Author:** sini
**Related:** Tasks #8 (`oneOfAspects` adapter), #9 (template examples), #10 (docs: when to make aspects non-parametric)

---

## 1. Summary

Give context entities (`host`, `user`, `home`, and any user-defined entity kind that imports `den.schema.conf`) a `.hasAspect` method that answers *"is aspect X present in my resolved aspect tree?"* from inside class-config module bodies and deferred aspect functor bodies. The method has three shapes:

```nix
entity.hasAspect <ref>                         # (B) — entity's primary class
entity.hasAspect.forClass "<class>" <ref>      # (C) — specific class
entity.hasAspect.forAnyClass <ref>             # (D) — union across all classes the entity participates in
```

Identity is compared by `aspectPath ref` (`= (meta.provider or []) ++ [name]`). Refs are accepted as raw aspect values or via `<angle-bracket>` sugar (same value post-`__findFile`). "Present" means *visible in the post-adapter resolved tree* — i.e., after `filterIncludes` composition, `meta.adapter` chains, `excludeAspect`, `substituteAspect` all take effect. Tombstones are invisible.

The feature is additive, introduces no changes to the parametric / resolution / ctxApply core, and attaches at the shared `den.schema.conf` extension point so user-defined entity kinds inherit it automatically.

## 2. Motivation

Users of Den frequently want to make configuration decisions *based on what else is installed* — canonical examples:

- `<impermanence>` aspect wants different nixos behavior depending on whether `<zfs-root>` or `<btrfs-root>` is also present on the host.
- A forward definition wants to pick a secret provider based on whether `<agenix-rekey>` or `<sops-nix>` is configured.
- A library aspect wants to gate opt-in behavior on the presence of a companion aspect.

Today, users have no first-class way to express these queries inside aspect bodies or class-config modules. They work around it with `config.*` lookups, manual `lib.elem` on lists they maintain themselves, or by restructuring via adapters — which is the right tool for some of these cases but overkill for pure read-only gating.

This design adds a query primitive that reads the same post-adapter structural tree the final module is built from, with guarantees that `hasAspect` and the built `mainModule` agree: if an aspect is in the built module, `hasAspect` returns true; if it's not, `hasAspect` returns false.

## 3. Goals and Non-goals

### Goals

1. **Query-time primitive** for reading structural membership of aspects in an entity's resolved tree.
2. **Identity via `aspectPath`** — compare by `(meta.provider or []) ++ [name]`, same model the rest of Den uses.
3. **Three variants**: bare (`hasAspect <X>`), per-class (`forClass "c" <X>`), any-class (`forAnyClass <X>`).
4. **Cycle-safe usage** from inside class-module bodies (deferred modules) and inside aspect functor bodies' lazy positions.
5. **Tombstone and adapter respect** — reports the post-`filterIncludes` view, so `excludeAspect` / `substituteAspect` / any `meta.adapter` is reflected in the answer.
6. **Extensibility to user-defined entity kinds** — attaches at `den.schema.conf`, not per-type in `types.nix`, so custom entity schemas that import `conf` inherit `hasAspect` automatically.
7. **Reusable lib primitive** — `den.lib.aspects.hasAspectIn { tree; class; ref }` usable outside entity contexts (tests, introspection, future tooling).
8. **First-class public adapter** (`collectPaths`) that any other consumer can use to walk structural trees, consistent with sini's direction to treat `adapters.nix` as a utility library.

### Non-goals (explicitly out of scope)

1. **Using `hasAspect` to decide an aspect's `includes = [...]` list** (option iii from brainstorming). Cyclic by construction — the tree-to-query depends on an include decision that depends on the tree-to-query. Redirected to `meta.adapter` + `oneOfAspects` / `excludeAspect` / `substituteAspect`, which run during the tree walk and have full structural visibility.
2. **`hasAspectDeclared` or any pre-resolution variant.** Silent false negatives on `perHost`/functor-wrapped aspects; creates a footgun that compounds with the existing unclear-docs issue around parametric vs static aspect forms. Rejected.
3. **Cycle detection or pretty errors for (iii) misuse.** Nix's infinite-recursion error is the feedback channel; detecting cycles requires evaluation-state introspection that Nix doesn't expose cheaply.
4. **Fixed-point iteration or two-phase resolution.** Would solve (iii) but is a major pipeline rewrite with nontrivial convergence concerns. Out of scope indefinitely.
5. **Matching aspects by name only.** Breaks the "distinct providers → distinct aspects" property that `aspectPath` exists to enforce. See §8.1 for factory-identity caveat.
6. **Cross-entity convenience sugar** (e.g. `host.anyUserHasAspect <X>`). Users can already write `lib.any (u: u.hasAspect <X>) (attrValues host.users)`. Defer until a pattern demands it.

## 4. Design

### 4.1 Semantics

**Identity.** Two aspect values are the same aspect iff `aspectPath a == aspectPath b`, where `aspectPath = a: (a.meta.provider or []) ++ [a.name]`. The primitive exists in `nix/lib/aspects/adapters.nix:45`. Valid inputs to `hasAspect`: any value for which `aspectPath` is defined — i.e., merged aspect attrsets with `name` and `meta.provider`. Raw strings, bare functions, or non-aspect attrsets produce an evaluation error from a wrapped validator in `has-aspect.nix`.

The `<angle-bracket>` sugar round-trips to the same aspect reference via `nix/lib/den-brackets.nix`'s `__findFile`, so `host.hasAspect <facter>` and `host.hasAspect den.aspects.facter` are equivalent.

**Present (B).** Aspect `X` is present in entity `E` iff, running `resolve.withAdapter collectPaths E.primaryClass E.resolved`, the returned `paths` list contains `aspectPath X`. The `collectPaths` adapter is wrapped in `filterIncludes`, so every `meta.adapter` along the way composes and tombstones are excluded by checking `meta.excluded`.

**Primary class (p).** For the bare `hasAspect <X>` form, the traversal runs under the entity's primary class:

- `host.class` for hosts
- `home.class` for homes
- `lib.head user.classes` for users (defaults to `"user"`; multi-class users who want union semantics reach for `.forAnyClass`)
- For user-defined entity kinds, see the precedence rule below.

**Class-protocol precedence.** When an entity kind exposes both `classes` (list) and `class` (single string), `classes` takes precedence — primary class is `lib.head classes`. This matches how users currently work (`classes` is the list form, the authoritative multi-class source), and means a user-defined kind that sets `classes = [ "foo" ]` will behave identically to one that sets `class = "foo"`. If only `class` is set, it's treated as `classes = [ class ]`. If neither is set, `hasAspect` fails at option-default time with a clear error (§6). No existing entity has both; the precedence is documented so future authors don't need to guess.

Primary-class was chosen over union-of-classes (option q from brainstorming) because:
- Include-tree traversal is class-invariant in practice (adapter composition can branch on class but rarely does)
- (p) and (q) give the same answer in all common configurations
- (p) is cheaper (one traversal per call) and matches the entity's main resolution path
- Users who need class-branching semantics explicitly have `.forClass` / `.forAnyClass`

**ForClass (C).** `entity.hasAspect.forClass "<class>" <X>` runs the same traversal under an explicit class. Passing a class not in `classes` returns `false` silently (documented — no pre-check or error).

**ForAnyClass (D).** `entity.hasAspect.forAnyClass <X>` returns true iff `X` is present under any of the classes the entity participates in. For hosts/homes (single-class), this is identical to the bare form. For multi-class users, it's meaningful only when a `meta.adapter` actively branches on class.

**Cycle-safety boundary.** `hasAspect` reads from `config.resolved` — the ctxApply output — without forcing any aspect's class-config module bodies. Nix's laziness means:

- ✅ **(i) Class-config module bodies** (`nixos = { config, host, ... }: { services.x.enable = host.hasAspect <y>; }`) — safe. The module body runs at final `evalModules` time, long after the tree is frozen.
- ✅ **(ii) Aspect functor bodies, lazy attribute positions** (`{host, ...}: { nixos.x.y = host.hasAspect <z>; }`) — safe. The `nixos` field is a thunk; `hasAspect` is forced only when the class module merges.
- ❌ **(iii) Aspect functor bodies, eager positions** (`{host, ...}: { includes = lib.optional (host.hasAspect <z>) [...]; }`) — cyclic. Produces Nix's infinite-recursion error. **Not supported.** Users are directed to `meta.adapter` + `oneOfAspects` in documentation.

### 4.2 API surface

#### Entity method (the `α` shape)

```nix
# (B) — bare, uses entity's primary class
host.hasAspect <facter>                  # :: bool
user.hasAspect <agenix-rekey>
home.hasAspect <nixvim>

# (C) — explicit class
host.hasAspect.forClass "nixos" <facter>
user.hasAspect.forClass "homeManager" <agenix-rekey>

# (D) — union across all classes the entity participates in
user.hasAspect.forAnyClass <agenix-rekey>
```

Implementation: the value is an attrset with `__functor`, `forClass`, and `forAnyClass`. Calling `host.hasAspect <X>` triggers the functor (returns bool for primary class); `.forClass` and `.forAnyClass` are direct attribute accesses to nested functions.

#### Lib primitives (`den.lib.aspects`)

```nix
# Low-level query against any resolved tree.
den.lib.aspects.hasAspectIn :: {
  tree  :: resolved-aspect;     # typically an entity's config.resolved
  class :: string;
  ref   :: aspect-ref;
} -> bool

# Set extraction — useful for tests, debugging, custom queries.
den.lib.aspects.collectPathSet :: {
  tree  :: resolved-aspect;
  class :: string;
} -> { "provider/.../name" = true; ... }

# Entity-facing constructor — builds the fn + attrs shape.
den.lib.aspects.mkEntityHasAspect :: {
  tree         :: resolved-aspect;
  primaryClass :: string;
  classes      :: [string];
} -> { __functor; forClass; forAnyClass; }
```

#### New public adapter (`den.lib.aspects.adapters`)

```nix
# Walks via filterIncludes; produces a flat list of non-tombstone
# aspectPaths. Terminal adapter — pass to resolve.withAdapter.
adapters.collectPaths :: adapter
  # result: { paths = [ [providerSeg..., name], ... ]; }
```

The companion adapter `adapters.oneOfAspects` (task #8) lands in the same file as part of this change — it's the structural-decision primitive `hasAspect` redirects users to for (iii)-style problems, and colocating the two helps discoverability.

### 4.3 Architecture

#### File layout

| File | Change |
|---|---|
| `nix/lib/aspects/adapters.nix` | Add `collectPaths`, `oneOfAspects`. Export from public attrset. |
| `nix/lib/aspects/has-aspect.nix` | **NEW.** `hasAspectIn`, `collectPathSet`, `mkEntityHasAspect`. |
| `nix/lib/aspects/default.nix` | Import `has-aspect.nix`, re-export its members. |
| `modules/context/has-aspect.nix` | **NEW.** Module defining `options.hasAspect` on `den.schema.conf`. Handles class-protocol fallback and missing-resolved errors. |
| `modules/options.nix` | `conf = { }` → `conf.imports = [ ../context/has-aspect.nix ]`. |
| `nix/lib/types.nix` | **Zero changes.** |
| `templates/ci/modules/features/collect-paths.nix` | **NEW.** Tests for `collectPaths` adapter. |
| `templates/ci/modules/features/one-of-aspects.nix` | **NEW.** Tests for `oneOfAspects` adapter. |
| `templates/ci/modules/features/has-aspect.nix` | **NEW.** End-to-end tests for `hasAspect` entity method and lib primitive. |

#### Dependency graph

```
modules/options.nix
  └─ modules/context/has-aspect.nix   (NEW)
       └─ den.lib.aspects.mkEntityHasAspect
            └─ nix/lib/aspects/has-aspect.nix   (NEW)
                 ├─ den.lib.aspects.resolve.withAdapter   (existing)
                 └─ den.lib.aspects.adapters.collectPaths   (NEW)
                      └─ nix/lib/aspects/adapters.nix
                           ├─ filterIncludes   (existing)
                           └─ aspectPath       (existing)
```

No circularity. No existing consumer imports the new module.

#### Placement in `den.schema.conf`

`modules/options.nix:69-74` already establishes `den.schema.conf` as the shared base module imported by `host`, `user`, `home`:

```nix
config.den.schema = {
  conf = { };
  host.imports = [ den.schema.conf ];
  user.imports = [ den.schema.conf ];
  home.imports = [ den.schema.conf ];
};
```

This change becomes:

```nix
config.den.schema = {
  conf.imports = [ ../context/has-aspect.nix ];
  host.imports = [ den.schema.conf ];
  user.imports = [ den.schema.conf ];
  home.imports = [ den.schema.conf ];
};
```

Any user-defined entity kind that imports `den.schema.conf` — e.g. `den.schema.vm.imports = [ den.schema.conf ]` — inherits `hasAspect` automatically, with zero changes to library code or `types.nix`. The only requirement is that the custom entity kind:

1. Exposes either `config.class :: string` or `config.classes :: [string]`
2. Has a corresponding `den.ctx.vm` registration (so `config.resolved` gets auto-injected by the schemaEntryType merge logic at `modules/options.nix:18-54`)

If either is missing, `hasAspect` produces a clear error (see §6).

## 5. Implementation sketch

### 5.1 `adapters.nix:collectPaths`

```nix
# Terminal adapter — pass to resolve.withAdapter. Walks the tree via
# filterIncludes and collects the aspectPath of every non-tombstone
# aspect visited, in depth-first order.
collectPaths = filterIncludes (
  { aspect, recurse, ... }:
  {
    paths =
      (lib.optional (!(aspect.meta.excluded or false)) (aspectPath aspect))
      ++ lib.concatMap (i: (recurse i).paths or [ ]) (aspect.includes or [ ]);
  }
);
```

### 5.2 `adapters.nix:oneOfAspects` (task #8, colocated)

```nix
# Keep the first aspect in `candidates` that's present in the subtree;
# tombstone the rest. Used as a meta.adapter to express "prefer A over
# B when both are present" without needing hasAspect in a cyclic position.
#
# Implementation note: needs to walk the subtree rooted at `aspect` to
# find which candidates are structurally present, then fold
# excludeAspect over the losers. The walk is NOT done via resolveChild
# (which returns an applied aspect, not a path list) — it's done by
# running `resolve.withAdapter adapters.collectPaths class aspect`
# inside the adapter body, reusing the class from args. The exact
# shape will be finalized during step 2 of the implementation order;
# the sketch below is illustrative and elides some wiring details.
oneOfAspects = candidates: inherited:
  args@{ aspect, class, ... }:
  let
    # Collect structural paths in the subtree under this class.
    subtreePaths = (resolve.withAdapter collectPaths class aspect).paths or [ ];
    subtreeKeys  = builtins.listToAttrs
      (map (p: { name = lib.concatStringsSep "/" p; value = true; }) subtreePaths);
    keyOf = c: lib.concatStringsSep "/" (aspectPath c);
    present = builtins.filter (c: subtreeKeys ? ${keyOf c}) candidates;
    winner  = if present == [ ] then null else builtins.head present;
    losers  = builtins.filter (c: c != winner) present;
    excluded = lib.foldl' (inner: loser: excludeAspect loser inner) inherited losers;
  in
  excluded args;
```

The sketch elides the fact that `oneOfAspects` imports `collectPaths` and `resolve` from the same file — these are co-located in `nix/lib/aspects/` so the dependency is internal. Final wiring will be written against the real exports during implementation.

### 5.3 `has-aspect.nix`

```nix
{ lib, den, ... }:
let
  inherit (den.lib.aspects) adapters resolve;

  # Join aspectPath into a string key. Slash matches the convention used
  # elsewhere in the codebase (adapters.nix structuredTrace, template tests).
  pathKey = path: lib.concatStringsSep "/" path;

  # Shared ref validator so error text is consistent across entry points.
  # Requires BOTH name and meta so a malformed attrset like
  # `{ meta.provider = ["x"]; }` (no name) is rejected loudly, rather
  # than silently falling back to `aspectPath`'s "<anon>" handling.
  refKey = ref:
    if builtins.isAttrs ref && (ref ? name) && (ref ? meta) then
      pathKey (adapters.aspectPath ref)
    else
      throw (
        "hasAspect: expected an aspect reference (got ${builtins.typeOf ref}). "
        + "Pass `den.aspects.<name>`, `<name>` angle-bracket sugar, or any "
        + "value satisfying `aspectPath` (must have both `name` and `meta`)."
      );

  # Run collectPaths under the given class on the given tree, return
  # the set of visible aspect paths keyed by their string form.
  collectPathSet =
    { tree, class }:
    let
      result = resolve.withAdapter adapters.collectPaths class tree;
      paths = result.paths or [ ];
    in
    builtins.listToAttrs (map (p: { name = pathKey p; value = true; }) paths);

  # Query primitive — usable outside any entity context.
  hasAspectIn =
    { tree, class, ref }:
    (collectPathSet { inherit tree class; }) ? ${refKey ref};

  # Entity-wrapping constructor. Builds the functor+attrs shape.
  # Per-class path sets are computed lazily via `let` binding — Nix's
  # attribute-value caching memoizes them across calls on the same entity.
  mkEntityHasAspect =
    { tree, primaryClass, classes }:
    let
      setFor = builtins.listToAttrs (map (c: {
        name = c;
        value = collectPathSet { inherit tree; class = c; };
      }) (lib.unique ([ primaryClass ] ++ classes)));

      check = class: ref:
        (setFor.${class} or { }) ? ${refKey ref};

      bareFn = check primaryClass;
    in
    {
      __functor = _: bareFn;
      forClass = check;
      forAnyClass = ref: lib.any (c: check c ref) classes;
    };

in
{
  inherit hasAspectIn collectPathSet mkEntityHasAspect;
}
```

### 5.4 `modules/context/has-aspect.nix`

```nix
# Defines `hasAspect` on any context entity that imports den.schema.conf.
# Entities must expose either `classes` (list) or `class` (string).
# If no matching den.ctx.<kind> exists, config.resolved is absent and
# any hasAspect call throws a clear error at call time.
{ den, lib, config, ... }:
{
  options.hasAspect = lib.mkOption {
    internal = true;
    visible = false;
    readOnly = true;
    type = lib.types.raw;
    defaultText = lib.literalExpression
      "den.lib.aspects.mkEntityHasAspect { tree = config.resolved; ... }";
    default =
      let
        classes =
          if config ? classes then config.classes
          else if config ? class then [ config.class ]
          else throw "den.schema.conf.hasAspect: entity has no `class` or `classes` — cannot build hasAspect";

        primaryClass =
          if classes == [ ]
          then throw "den.schema.conf.hasAspect: entity has empty classes list"
          else lib.head classes;

        missingResolvedError = _:
          throw "hasAspect: ${config.name or "<unnamed entity>"} has no config.resolved (no matching den.ctx.<kind> defined)";
      in
      if config ? resolved then
        den.lib.aspects.mkEntityHasAspect {
          tree = config.resolved;
          inherit primaryClass classes;
        }
      else
        {
          __functor = _: missingResolvedError;
          forClass = _: missingResolvedError;
          forAnyClass = missingResolvedError;
        };
  };
}
```

The missing-resolved fallback preserves the functor-plus-attrs shape so `entity.hasAspect.forClass "x" <y>` throws the same error as `entity.hasAspect <y>`, instead of erroring on `.forClass` attribute access before reaching the real problem.

### 5.5 `default.nix` aggregator update

```nix
{ lib, den, ... }:
let
  rawTypes = import ./types.nix { inherit den lib; };
  adapters = import ./adapters.nix { inherit den lib; };
  resolve = import ./resolve.nix { inherit den lib; };
  hasAspect = import ./has-aspect.nix { inherit den lib; };

  defaultFunctor = (den.lib.parametric { }).__functor;
  typesConf = { inherit defaultFunctor; };
  types = lib.mapAttrs (_: v: v typesConf) rawTypes;
in
{
  inherit types adapters resolve;
  inherit (hasAspect) hasAspectIn collectPathSet mkEntityHasAspect;
  mkAspectsType = cnf': lib.mapAttrs (_: v: v (typesConf // cnf')) rawTypes;
}
```

### 5.6 Memoization

The `let setFor = ...` binding in `mkEntityHasAspect` builds an attrset keyed by class name with thunk values. Nix's attribute evaluation caches thunks on first force, so:

- First `host.hasAspect <X>` forces `setFor.${primaryClass}` — one traversal.
- Subsequent `host.hasAspect <Y>` reuses the cached set — no retraversal.
- First `host.hasAspect.forClass "other" <Z>` forces `setFor.${"other"}` — one traversal for the new class, independent of the primary's cached set.
- `host.hasAspect.forAnyClass <Z>` touches up to N thunks (one per class), each cached independently.

Memoization is per-entity-instance (`mkEntityHasAspect` is called once per entity at option-default time), shared across all `hasAspect` calls on that entity within the same evaluation, and not shared across entities.

## 6. Error handling and edge cases

### 6.1 Error matrix

| Trigger | Site | Error form |
|---|---|---|
| `ref` is not aspect-shaped | `refKey` helper in `has-aspect.nix` | `throw "hasAspect: expected an aspect reference (got <type>). Pass den.aspects.<name>, <name> sugar, or any value satisfying aspectPath."` |
| `tree` is not a resolved aspect | `resolve.withAdapter` native | Unwrapped resolver error. Not user-facing — public API always passes `config.resolved`. |
| Entity has no `config.resolved` (no `den.ctx.<kind>`) | Module fallback in `modules/context/has-aspect.nix` | `throw "hasAspect: <name> has no config.resolved (no matching den.ctx.<kind> defined)"`. Thrown at call time, not at option-default time. All three entry points (`bare`, `forClass`, `forAnyClass`) throw the same error. |
| Entity has neither `class` nor `classes` | Module fallback | `throw "den.schema.conf.hasAspect: entity has no \`class\` or \`classes\` — cannot build hasAspect"`. Thrown at option-default evaluation (fires when `.hasAspect` is first read). |
| Entity has empty `classes = [ ]` | Module fallback | `throw "den.schema.conf.hasAspect: entity has empty classes list"`. Same timing as above. |
| `forClass "bogus" <X>` | `check "bogus"` in `mkEntityHasAspect` | Returns `false`. No throw. Documented behavior. |
| Cyclic use in (iii) position | Nix evaluator | Native `infinite recursion encountered`. Not wrapped. Documented redirect to `meta.adapter` / `oneOfAspects`. |

### 6.2 Call-time vs reference-time errors

Reading `entity.hasAspect` as a value (without calling) does NOT throw, even in the missing-resolved case. Only calling (`entity.hasAspect <X>` or `entity.hasAspect.forClass ...`) triggers evaluation. This matches user expectations for a method that's safe to reference but not safe to invoke without context — and keeps tools that introspect options (`nix eval`, schema documenters, flake checks) from choking on entities that happen to lack context pipelines.

Exception: the class-protocol errors (`no class/classes`, `empty classes`) fire at option-default evaluation, because those are fundamental schema misconfigurations — an entity kind without a class concept is structurally broken for this feature and should fail loudly as soon as anyone touches `.hasAspect`.

### 6.3 Purity

Both `hasAspectIn` and `entity.hasAspect` are pure functions of their inputs. No side effects, no global state, no evaluation-order dependence. This is what guarantees memoization is safe — Nix's thunk caching is only valid for pure computations.

## 7. Testing strategy

### 7.1 File organization

Three new test files in `templates/ci/modules/features/`:

| File | Tests | Purpose |
|---|---|---|
| `collect-paths.nix` | ~6 | `adapters.collectPaths` in isolation via `resolve.withAdapter`. |
| `one-of-aspects.nix` | ~6 | `adapters.oneOfAspects` in isolation. |
| `has-aspect.nix` | ~25 | End-to-end `entity.hasAspect` and `den.lib.aspects.hasAspectIn`. |

The existing `aspect-adapter.nix` is left untouched. The three new files follow the `denTest` harness convention used throughout `templates/ci/modules/features/`.

### 7.2 Regression-class coverage

`hasAspect` reads `config.resolved`, which is the output of the ctxApply + parametric resolution pipeline. That pipeline has seen four regressions in recent months (#408, #413, #423, #429), each in a different aspect-construction shape. The test suite is structured so that `has-aspect.nix` exercises `hasAspect` against each of those shapes, making it a **free integration canary** for the pipeline itself: a regression in `parametric.nix` or `aspects/types.nix` that affects any of these shapes would fail a `hasAspect` test before it gets anywhere near user code.

The shapes covered:

- Static attrset aspects (baseline)
- Parametric-parent + static-sub (#423 shape)
- Parametric-parent + parametric-sub / bare function sub (#413 shape)
- Factory-function aspects with static sibling (#408, #429 shapes)
- Provider-chain reachability (`foo._.sub`, mutual-provider, cross-provider)
- `meta.adapter` composition (tombstones, substitution, nesting, `oneOfAspects` integration)
- Multi-class users with class-branching adapters
- User-defined entity kinds via `den.schema.conf` extensibility
- Error paths (bad ref, missing resolved, missing class, unknown forClass)
- Lib primitive usage (`hasAspectIn` against handcrafted trees)
- Real-world call pattern (inside class-module body, confirming cycle-safety)

### 7.3 `has-aspect.nix` test matrix

Tests grouped by shape category. Tests in **bold** are the ones locking in prior-regression-class fixes.

**Group A — Basic shape (sanity).**
- `test-A-hosts-hasAspect-present-static` — static aspect directly in `host.includes` → present.
- `test-A-hosts-hasAspect-absent` — declared elsewhere but not reachable → absent.
- `test-A-hosts-hasAspect-self` — host's own root aspect reported as present.
- `test-A-hosts-hasAspect-chained-transitively` — reachable through 3+ static include levels.

**Group B — Parametric contexts (forced functors).**
- **`test-B-present-via-parametric-parent`** — `{host,...}: {includes = [<bar>]}`. Exercises #419 applyDeep path.
- **`test-B-present-via-static-sub-aspect-in-parametric-parent`** — reproduces #423's shape: `role = {host,...}: {includes = [role._.sub]}; role._.sub.nixos.x = true`. Locks in #423 fix.
- **`test-B-present-via-bare-function-sub-aspect`** — reproduces #413's shape: `foo._.sub = {host,...}: {nixos = ...}`. Locks in #413 fix.
- `test-B-absent-when-parametric-parent-omits` — parent conditionally omits based on host.name.

**Group C — Factory functions.** *(See §8.1 open question — test matrix subject to investigation outcome.)*
- **`test-C-named-factory-instance-present`** — user-named instance (`den.aspects.my-facter = facter "/x"`) visible via `hasAspect <my-facter>`.
- **`test-C-factory-merged-with-static-sibling`** — exercises #408/#429 shape.
- `test-C-factory-itself-absent-when-only-instances-present` — documents the factory-vs-instance identity behavior (see §8.1).

**Group D — Provider sub-aspects.**
- `test-D-static-provider-sub-present` — `foo._.sub` reached via parent's includes.
- `test-D-parametric-provider-sub-present` — exercises `providerFnType` merge path.
- `test-D-provider-sub-identity-distinct-from-homonym` — `den.aspects.foo` and `den.aspects.bar._.foo` distinct aspectPaths.

**Group E — Mutual-provider / provides chains.**
- `test-E-present-via-provides-to-users` — confirms `config.resolved` is the ctx-composed tree.
- `test-E-present-via-provides-specific-user` — per-user scoping via `provides.<name>.includes`.
- `test-E-present-via-cross-provider` — cross-aspect `_.${from.name}` chain.

**Group F — Meta.adapter interactions.**
- `test-F-respects-excludeAspect-tombstone` — confirms `filterIncludes` composition.
- `test-F-respects-substituteAspect` — substituted aspect visible, original absent.
- `test-F-composes-with-nested-adapter` — inner + outer adapters both take effect.
- `test-F-respects-oneOfAspects` — integrates with the new `oneOfAspects` adapter.

**Group G — Multi-class users.**
- `test-G-user-hasAspect-primary-class` — bare form uses `lib.head classes`.
- `test-G-user-hasAspect-forClass-explicit` — specific class traversal.
- `test-G-user-hasAspect-forAnyClass-union` — class-branching adapter differs from primary.
- `test-G-user-hasAspect-forClass-unknown-class-false` — no error, returns false.

**Group H — Extensibility.**
- `test-H-custom-entity-kind-has-hasAspect` — user-defined `den.schema.vm` + `den.ctx.vm` → `vm.hasAspect <X>` works without touching `types.nix`.
- `test-H-conf-option-exists` — `den.schema.conf` declares `options.hasAspect`.

**Group I — Error cases.**
- `test-I-bad-ref-throws-with-pretty-message` — throws mentioning "expected an aspect reference" and the bad type.
- `test-I-missing-resolved-throws` — custom entity kind without matching `den.ctx.<kind>`.
- `test-I-missing-class-throws-at-option-eval` — custom kind exposing neither `class` nor `classes`.
- `test-I-empty-classes-throws` — entity with `classes = [ ]`.

**Group J — Lib primitive usage.**
- `test-J-hasAspectIn-against-handcrafted-tree` — direct `den.lib.aspects.hasAspectIn` call, no entity.
- `test-J-collectPathSet-returns-expected-keys` — direct inspection of the set shape.
- `test-J-in-class-module-body` — real-world call from inside `nixos = {config, host, ...}: ...` deferred module. Confirms cycle-safety under the intended use pattern.

### 7.4 `collect-paths.nix` matrix

- `test-basic-static-tree` — 3-level static tree, DFS path list matches expectation.
- `test-empty-tree` — tree with no includes returns just the root.
- `test-forces-parametric-functors` — parametric aspect reached via functor is in the result.
- `test-skips-tombstones` — `excludeAspect` → excluded aspect's path absent.
- `test-no-duplicates-for-shared-subtree` — shared aspect appears twice in `paths` (documents: `collectPaths` does NOT dedupe; downstream `collectPathSet` does).
- `test-provider-path-included` — `foo._.sub` has path `["foo", "sub"]`.

### 7.5 `one-of-aspects.nix` matrix

- `test-prefers-first-present` — both present → first survives, second tombstoned.
- `test-falls-through-to-second` — only second present → second survives, no tombstone on absent one.
- `test-both-absent-no-effect` — neither present → no tombstones.
- `test-with-hasAspect-integration` — `meta.adapter = oneOfAspects [...]` + hasAspect query cross-validates.
- `test-composes-with-outer-adapter` — nested `meta.adapter`s both take effect.
- `test-works-on-sub-aspects` — `oneOfAspects [<foo._.a> <foo._.b>]` at provider-sub level.

### 7.6 Tests explicitly NOT written

- Infinite recursion on (iii) misuse — asserting "this produces a Nix error" is awkward in `nix-unit`; behavior is the evaluator's, not ours.
- Performance / memoization timing — Nix-internal, not observable without introspection.
- Tests requiring a running `nixosSystem` build — we use `denTest` + module-level assertions throughout.

## 8. Open questions

### 8.1 Factory-function aspect identity

When a user writes:

```nix
den.aspects.facter = reportPath: {
  nixos = { pkgs, ... }: { hardware.facter.reportPath = reportPath; };
};

den.aspects.x1c.includes = [ (den.aspects.facter "/etc/facter.json") ];
```

…the value `(den.aspects.facter "/etc/facter.json")` is a plain attrset `{nixos = ...}` — not a merged aspect. When it goes into the `includes` list and gets typed as `providerType`, the aspectSubmodule merge assigns it a `name` and `meta.provider` based on where it's declared (the module location), NOT based on the factory it came from. As a result:

- `aspectPath (den.aspects.facter)` = `["facter"]` (the factory's own path)
- `aspectPath (den.aspects.facter "/x")` = some loc-derived path, not `["facter"]`

This means `host.hasAspect <facter>` on a host that uses `(facter "/x")` but not `den.aspects.facter` directly will return **false**, which is likely not what users expect.

**Resolution path during implementation:** verify this behavior empirically by writing the test and observing `aspectPath` values. Two possible outcomes:

1. **Instances get the factory's name.** Either because `providerFnType.merge` propagates the name through, or because the aspectSubmodule's `name` default falls back to the factory's key. In this case the concern is moot — factory-pattern users can query naturally and the test passes.

2. **Instances get loc-derived names distinct from the factory.** In this case, document the limitation and direct users to the "name your instances" pattern:

    ```nix
    den.aspects.my-facter = facter "/etc/facter.json";
    den.aspects.igloo.includes = [ den.aspects.my-facter ];
    # Query:
    host.hasAspect <my-facter>   # works
    host.hasAspect <facter>      # false — factory itself isn't in the tree
    ```

The design doesn't require either outcome to hold — it's consistent either way, just with different user-facing ergonomics. Investigation happens during step 3 of the implementation order (§9).

### 8.2 Test: custom entity kind as schema-only

`test-H-custom-entity-kind-has-hasAspect` requires setting up a full custom entity kind with a matching `den.ctx.vm` within a test module. This is substantial setup for one assertion. Alternative: `test-H-conf-option-exists` proves the schema-level wiring works with less boilerplate. Recommendation: write both — the full-setup test validates the end-to-end extensibility claim, the minimal test is the smoke check.

## 9. Implementation order

Roughly five landable steps, each independently greenable against `just ci`:

1. **`collectPaths` adapter + `collect-paths.nix` tests.** Isolated feature, no dependencies on the rest. Exportable from `adapters.nix` immediately.

2. **`oneOfAspects` adapter + `one-of-aspects.nix` tests.** Also landable independently. Ordered after (1) because the `test-with-hasAspect-integration` test depends on (4).

3. **`has-aspect.nix` lib + default.nix wiring + Group J tests.** Exercises `hasAspectIn` / `collectPathSet` / `mkEntityHasAspect` without entity plumbing. Resolves §8.1 via empirical observation during this step.

4. **`modules/context/has-aspect.nix` + `modules/options.nix` wiring + Groups A–I tests.** Wires the entity method, runs full test matrix. Minus any Group C rows still blocked by §8.1 resolution.

5. **Template examples (task #9).** Separate PR, after this one lands. Not blocking.

Tasks #8 (oneOfAspects) and #9 (templates) are coupled to this design but landed as noted. Task #10 (parametric-vs-static docs) is independent and unblocked.

## 10. Related work and future considerations

- **`meta.adapter` as the correct tool for (iii)-style structural decisions.** This design explicitly directs users there via documentation. `oneOfAspects` is the structural-decision primitive most directly complementary to `hasAspect`'s read-only role.

- **Future: cached "all aspects touched" list in ctx.** If `den.ctx` ever grows a generic "all aspect paths visited during resolution" cache (for other purposes — tracing, diag, debugging), `mkEntityHasAspect` can be rewritten to read from it instead of running its own collector. Zero API change. Leaving the collector-based approach in place now is the right call — that cache doesn't exist and building it here would be scope creep.

- **Future: aspect database / type-level registry.** If Den ever reifies aspects as a formal database (rather than a freeform module attrset), `hasAspect` can upgrade from "structural tree walk" to "set-lookup in the database". Same user-facing API, cheaper implementation. Today's approach is the natural baseline and doesn't prevent this.

- **Future: adapter-presence queries.** If users start wanting "does any adapter along my path transform X?", that's a separate primitive on top of `collectPaths` / `filterIncludes`. Possible follow-up if demand emerges.

- **Unrelated but adjacent (task #10):** Documentation work on when to use parametric vs static aspect forms. Users currently reach for `perHost` / `perUser` / functor wrappers even when the aspect has no ctx dependence, which compounds silently with any structural-query primitive that can only see static declarations. This work is docs-only and independent of `hasAspect`, but mentioned here for context — it's the backdrop against which sini's "hasAspectDeclared would be a liability" argument was made.

## 11. Migration / breaking changes

**None.** The change is purely additive:

- `hasAspect` is a new option on existing entities; no attribute-name conflict.
- No changes to function signatures in `parametric.nix`, `resolve.nix`, `ctx-apply.nix`, or elsewhere in the existing library.
- `modules/options.nix:69`'s `conf = { }` → `conf.imports = [...]` is opt-in-transparent: `conf` was previously empty, so adding an option to it doesn't break any existing user of the import chain.
- New adapter exports (`collectPaths`, `oneOfAspects`) don't conflict with existing `adapters.nix` exports.
- No changes to `types.nix`, `den.schema.conf` consumers, or any user-facing attribute outside the additions.

## Appendix A — Quick user-facing summary

For users, once this lands:

```nix
# Inside a class-config body — the primary intended use.
den.aspects.impermanence.nixos = { config, host, ... }: lib.mkMerge [
  (lib.mkIf (host.hasAspect <zfs-root>) {
    /* zfs-flavored impermanence */
  })
  (lib.mkIf (host.hasAspect <btrfs-root>) {
    /* btrfs-flavored impermanence */
  })
];

# Inside a functor body, in a lazy position — also fine.
den.aspects.alice = { user, ... }: {
  homeManager.programs.fish.shellInit =
    lib.mkIf (user.hasAspect <direnv>) "direnv hook fish | source";
};

# Class-specific query — rare but available.
den.aspects.observability.nixos = { host, ... }: {
  services.prometheus.enable = host.hasAspect.forClass "nixos" <metrics-exporter>;
};

# Structural decision — NOT hasAspect, use a meta.adapter instead.
den.aspects.secrets-bundle = {
  includes = [ <agenix-rekey> <sops-nix> ];
  meta.adapter = den.lib.aspects.adapters.oneOfAspects [
    <agenix-rekey>
    <sops-nix>
  ];
};

# What will NOT work (don't do this — will cycle):
# den.aspects.foo = { host, ... }: {
#   includes = lib.optional (host.hasAspect <bar>) [ /* ... */ ];
# };
# Use meta.adapter + excludeAspect / substituteAspect / oneOfAspects instead.
```
