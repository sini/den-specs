# Constraint System

**Branch:** feat/fx-pipeline (629/629 tests)
**Date:** 2026-05-04
**Scope:** How den gates aspect inclusion via constraints: excludes, filters, substitutions

---

## 1. Overview

The constraint system controls which aspects survive include resolution. When an aspect's subtree is resolved, each child aspect is checked against registered constraints before being compiled. A constraint can **exclude** an aspect (replace it with a tombstone), **substitute** it (replace with a different aspect), or **filter** it (exclude based on a predicate over the aspect attrset). Aspects that pass all constraints are **kept** and proceed to class emission or further include resolution.

Constraints are registered as algebraic effects (`register-constraint`) and checked as algebraic effects (`check-constraint`). Both are handled by `constraintRegistryHandler` in `handlers/tree.nix`.

---

## 2. Constraint Constructors (`constraints.nix`)

Three constructors live in `nix/lib/aspects/fx/constraints.nix`. Each returns a record with a `type`, a `scope` field, and type-specific fields.

### 2.1 `exclude`

```nix
den.lib.aspects.fx.constraints.exclude ref
```

- `type = "exclude"`
- `identity = pathKey (aspectPath ref)` -- the `/`-joined provider path of the target
- `scope = "subtree"` (default) or `"global"` via `.global`

Asserts `ref` is an attrset (an aspect reference like `den.aspects.foo`).

### 2.2 `substitute` (vestigial)

```nix
den.lib.aspects.fx.constraints.substitute ref replacement
```

- `type = "substitute"`
- `identity` -- same derivation as exclude
- `replacementName = replacement.name or "<anon>"`
- `getReplacement = _: replacement` -- thunked to avoid eager evaluation

This type exists from the provides era where an aspect could be replaced by its provider's alternative. No live code path generates substitute constraints today. Pending removal with provides (consolidated spec Section 7.2).

### 2.3 `filterBy`

```nix
den.lib.aspects.fx.constraints.filterBy pred
```

- `type = "filter"`
- `predicate = pred` -- a function `aspect -> bool`; aspects where `pred` returns `false` are excluded
- `scope = "subtree"` (default) or `"global"` via `.global`

### 2.4 Scope Variants

All three constructors use a shared `scoped` wrapper that adds a `.global` attribute. The default call produces `scope = "subtree"`; `constructor.global args` produces `scope = "global"`. This controls whether the constraint applies only within the registering aspect's subtree or across the entire pipeline run.

---

## 3. Constraint Registration

### 3.1 `registerConstraints` (include-emit.nix)

Called from `compileStatic` in `aspect.nix` (line 192) for every statically-compiled aspect. It reads two meta fields:

1. **`meta.handleWith`** -- a single constraint record, a list of constraint records, or null. Normalized to a list.
2. **`meta.excludes`** -- a list of aspect references. Each is converted to an exclude-type constraint with `scope = "subtree"`.

Both sources are concatenated (`handleWithList ++ excludeList`), tagged with `owner = aspect.name`, and each sent as a `register-constraint` effect via `fx.seq`.

```
registerConstraints aspect =
  let
    allConstraints = handleWithList ++ excludeList;
  in
  fx.seq (map (c -> fx.send "register-constraint" (c // { owner })) allConstraints);
```

### 3.2 `register-constraint` Handler (tree.nix)

The handler branches on `param.type`:

**Filter type:** Appended to `scopedConstraintFilters` state. Each entry stores `{ predicate, owner, scope, ownerChain }`. The ownerChain is captured from the current `scopedIncludesChain` at registration time.

**Exclude/substitute type:** Appended to `scopedConstraintRegistry` state, keyed by `param.identity`. Each entry stores `{ type, getReplacement, owner, scope, ownerChain }`.

Both state fields are scope-partitioned (keyed by `state.currentScope`, e.g. `"host"` or `"user"`), and thunk-wrapped (`_: value`) to prevent deepSeq from re-materializing growing collections.

---

## 4. Constraint Checking

### 4.1 `check-constraint` Handler (tree.nix)

Called from `emitIncludes` in `include-emit.nix` (line 216) for every non-forward, non-conditional child aspect. The param can be either a string identity or `{ identity, aspect }`.

**Decision algorithm:**

1. **Merge registries across all scopes.** `allScopedRegistry` reads constraints from every scope (host, user, etc.) and merges them with `zipAttrsWith`. This ensures cross-scope constraints are visible.

2. **Exact match.** Look up `registry.${nodeIdentity}` for entries matching the aspect's full identity.

3. **Prefix match.** If the identity contains `/` separators, split into path components and generate all prefixes. Look up each prefix in the registry. This lets a constraint on `monitoring` match `monitoring/node-exporter`.

4. **Scope filtering.** All matched entries (exact + prefix) are filtered by `inScope`:
   - `scope == "global"` entries always match
   - `scope == "subtree"` entries match only if `isAncestor ownerChain` -- i.e., the current includes chain starts with the constraint's ownerChain

5. **First match wins.** If any scoped entry exists:
   - `type == "exclude"` -> decision `{ action = "exclude"; owner }`
   - `type == "substitute"` -> decision `{ action = "substitute"; replacement; owner }`
   - anything else -> `{ action = "keep" }`

6. **Filter fallback.** If no registry entry matched, check all scoped filters. For each in-scope filter, run `f.predicate aspect`. The first filter that returns `false` produces an exclude decision. If all pass (or aspect is null), decision is `keep`.

### 4.2 `isAncestor` -- Subtree Scoping

```nix
isAncestor = ownerChain:
  lib.take (builtins.length ownerChain) currentChain == ownerChain;
```

The includes chain is a stack maintained by `chain-push`/`chain-pop` effects (also in tree.nix). When an aspect with a meaningful name begins resolution, `chain-push` appends its identity. When resolution completes, `chain-pop` removes it. This creates a breadcrumb trail of the current position in the include tree.

A subtree constraint registered at chain `["root", "parent"]` matches any check where the current chain starts with those elements -- i.e., within the registering aspect's descendant subtree.

### 4.3 Decision Outcomes in `emitIncludes`

After `check-constraint` returns, `emitIncludes` (include-emit.nix line 221-230) branches:

- **`"exclude"`**: calls `excludeChild` -- creates a tombstone via `identity.tombstone`, sends `resolve-complete` with the tombstone (marking `meta.excluded = true`), returns `[ tombstone ]`.
- **`"substitute"`**: calls `substituteChild` -- creates a tombstone, then resolves the replacement aspect via `aspectToEffect`, returns `[ tombstone, resolved ]`.
- **`"keep"`**: tags the aspect with `constraintOwner` if applicable, then dispatches to `resolve-parametric` or `resolve-aspect` depending on whether `__args` is non-empty.

---

## 5. `meta.handleWith`

Declared on an aspect's `meta` option in `types.nix` (line 342):

```nix
options.handleWith = lib.mkOption {
  type = lib.types.nullOr (lib.types.mkOptionType {
    name = "handlerValue";
    description = "handler record or list of handler records";
    check = v: builtins.isAttrs v || builtins.isList v;
    merge = _: defs: (lib.last defs).value;
  });
  default = null;
};
```

Accepts a single constraint record, a list of constraint records, or null. The merge strategy is last-writer-wins (`lib.last defs`).

**Usage pattern** (from diagram-demo `server.nix`):

```nix
den.aspects.server-host = {
  includes = with den.aspects; [ relay ];
  meta.handleWith = [
    (den.lib.aspects.fx.constraints.exclude den.aspects.monitoring._.nginx-exporter)
    (den.lib.aspects.fx.constraints.filterBy (
      a: lib.take 1 (a.meta.provider or []) != [ "monitoring" ]
    ))
  ];
};
```

This installs both an exclude constraint (targeting a specific sub-aspect by identity) and a filter constraint (targeting all aspects with `monitoring` as their first provider) scoped to `server-host`'s subtree.

---

## 6. `meta.excludes`

Declared on an aspect's `meta` option in `types.nix` (line 354):

```nix
options.excludes = lib.mkOption {
  description = "Aspects to exclude from this subtree (sugar for handleWith)";
  type = lib.types.listOf lib.types.unspecified;
  default = [];
};
```

Sugar for `handleWith`. Each element is an aspect reference. `registerConstraints` converts each to:

```nix
{ type = "exclude"; scope = "subtree"; identity = pathKey (aspectPath ref); }
```

The `meta.excludes` list and `meta.handleWith` list are concatenated -- both participate in the same `register-constraint` dispatch.

---

## 7. State Shape

Three pipeline state fields support constraints, all scope-partitioned and thunk-wrapped:

| Field | Shape | Purpose |
|-------|-------|---------|
| `scopedConstraintRegistry` | `{ scope -> { identity -> [ entry ] } }` | Exclude and substitute constraints keyed by target identity |
| `scopedConstraintFilters` | `{ scope -> [ { predicate, owner, scope, ownerChain } ] }` | Filter-type constraints as an ordered list |
| `scopedIncludesChain` | `{ scope -> [ identity ] }` | Current position in the include tree; used for subtree scoping |

---

## 8. Vestigial Components

### 8.1 `substitute` Type

The `substitute` constructor in `constraints.nix` and its handler branch in `check-constraint` exist from the provides era, where `provides.${name}` could substitute one aspect with another. With no code path generating substitute constraints today, the type is dead. `substituteChild` in `include-emit.nix` (the function that executes a substitute decision by tombstoning the original and resolving the replacement) is similarly vestigial.

Removal is gated on provides removal (consolidated spec Section 5, Section 7.2).

### 8.2 What Remains Active

- **`exclude`**: Used by `meta.excludes` sugar and directly via `meta.handleWith` with `constraints.exclude`.
- **`filterBy`**: Used via `meta.handleWith` with `constraints.filterBy`. Enables predicate-based subtree pruning (e.g., exclude all aspects from a provider prefix).

---

## 9. Planned Evolution

The consolidated spec (Section 9) outlines replacing the constraint mechanism with policy effects:

**Current:** `meta.handleWith` / `meta.excludes` -> `registerConstraints` -> `register-constraint` effect -> constraint registry state -> `check-constraint` effect during include resolution. A parallel path to policy dispatch.

**Planned:** Constraints become policy effects processed through standard policy dispatch. `meta.handleWith` and `meta.excludes` are eliminated. Instead:

```nix
# Exclusion as a policy in includes:
den.aspects.server.includes = [
  (policy.exclude den.aspects.desktop)
];

# Filter as a policy in includes:
den.aspects.server.includes = [
  (policy.filter (aspect: aspect.meta.category or "" != "desktop"))
];
```

This unifies three mechanisms (handleWith, meta.excludes, constraint registry) into one (policy effects via includes/excludes). The constraint registry handler pair (`register-constraint` / `check-constraint`) simplifies or is eliminated, replaced by policy effects processed through the existing `installPolicies` dispatch path.

**Dependencies:** Provides removal first (eliminates substitute type, reduces constraint surface area), then policy scoping redesign.

**Open questions from consolidated spec Section 9.5:**
1. Policy identity for excludes -- referential identity vs explicit keys
2. Ordering between included policies and included aspects
3. Global policy override granularity
4. Migration path for arbitrary `handleWith` handler records
