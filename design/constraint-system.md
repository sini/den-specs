# Constraint System

**Branch:** feat/fx-pipeline (753/753 tests)
**Scope:** How den gates aspect inclusion via exclude and filterBy constraints, covering registration, enforcement, policy integration, and gate composition.

---

## 1. Overview

The constraint system controls which aspects survive include resolution. When an aspect's includes list is walked, each child is checked against registered constraints before compilation proceeds. A constraint can **exclude** an aspect (replace it with a tombstone and block its subtree) or **filter** it (exclude based on a predicate over the aspect attrset). Aspects that pass all constraints are kept and dispatched to a compiler.

Constraints are registered via the `register-constraint` effect and enforced via the `check-constraint` effect, both handled by `constraintRegistryHandler` in `handlers/constraint.nix`. The **gate** handler (`handlers/gate.nix`) composes dedup and constraint checking into a single effect so compilers do not call them separately.

A third type, **substitute**, exists in the code but is vestigial — no active code path produces substitute constraints.

---

## 2. Constraint Types

### 2.1 `exclude` (active)

**Constructor:** `nix/lib/aspects/fx/constraints.nix`

```nix
den.lib.aspects.fx.constraints.exclude ref          # scope = "subtree"
den.lib.aspects.fx.constraints.exclude.global ref   # scope = "global"
```

Produces `{ type = "exclude"; identity = pathKey (aspectPath ref); scope; }`. The `identity` is the `/`-joined provider path of the target aspect. Asserts `ref` is an attrset.

When enforced, the target aspect is tombstoned (`meta.excluded = true`, `meta.excludedFrom = owner`). The entire subtree below the excluded aspect is skipped. A `include-unseen` tombstone is written when a dedup key exists, preventing the slot from being reused by a later include of the same aspect.

### 2.2 `filterBy` (active)

**Constructor:** `nix/lib/aspects/fx/constraints.nix`

```nix
den.lib.aspects.fx.constraints.filterBy pred          # scope = "subtree"
den.lib.aspects.fx.constraints.filterBy.global pred   # scope = "global"
```

Produces `{ type = "filter"; predicate = pred; scope; }`. `pred` is a function `aspect -> bool`; aspects where `pred` returns `false` are excluded with the filter's `owner` as the constraint source.

Filter constraints are stored separately from identity-keyed registry entries. They are checked only when no registry entry matches the aspect's identity.

### 2.3 `substitute` (vestigial)

**Constructor:** `nix/lib/aspects/fx/constraints.nix`

```nix
den.lib.aspects.fx.constraints.substitute ref replacement
```

Produces `{ type = "substitute"; identity; replacementName; getReplacement; scope; }`. The `getReplacement` thunk wraps the replacement aspect to defer evaluation.

This type is present in the handler and gate code but no active code path generates substitute constraints. It originated in the provides era where an aspect could be transparently replaced by a provider's alternative. Removal is blocked on provides removal.

### 2.4 Scope Variants

Both active types use a `scoped` wrapper that exposes `.global`. The default call produces `scope = "subtree"`; the `.global` variant produces `scope = "global"`. Subtree constraints match only within the registering aspect's descendant include tree; global constraints match everywhere in the pipeline run.

---

## 3. Registration

### 3.1 `registerConstraints` (`fx/aspect/children.nix`)

Called from `compileStatic` for every statically-compiled aspect. It reads two fields:

- **`aspect.excludes`** — list of aspect references. Each is converted to `{ type = "exclude"; scope = "subtree"; identity = identity.key ref; }`. References that are policy attrsets (`__isPolicy = true`) use the policy name as identity.
- **`aspect.meta.handleWith`** — a single constraint record or list of constraint records (normalized to list). Passed through directly.

Both are concatenated and sent as `register-constraint` effects, each tagged with `owner = aspect.name`.

### 3.2 `register-constraint` Handler (`handlers/constraint.nix`)

The handler branches on `param.type`:

**Filter (`type == "filter"`):**
- Appends a filter entry `{ predicate, owner, scope, ownerChain }` to:
  - `state.flatConstraintFilters` — flat list across all scopes
  - `state.scopedConstraintFilters.${currentScope}` — scope-partitioned list

**Exclude/substitute:**
- Appends an entry `{ type, getReplacement, owner, scope, ownerChain }` to:
  - `state.flatConstraintRegistry.${param.identity}` — flat lookup by identity
  - `state.scopedConstraintRegistry.${currentScope}.${param.identity}` — scope-partitioned lookup

The `ownerChain` is captured from `state.scopedIncludesChain.${currentScope}` at registration time. This snapshot is what the scope ancestry check compares against during enforcement.

Both scoped state fields are thunk-wrapped (`_: value`) to prevent `deepSeq` from re-materializing growing collections during state propagation.

---

## 4. Enforcement

### 4.1 `check-constraint` Handler (`handlers/constraint.nix`)

Called by the gate handler for each aspect before compilation. The param is either a string identity or `{ identity, aspect }` (the aspect attrset is needed for filter predicates).

**Algorithm:**

1. **Prefix lookup.** `lookupEntries` fetches exact matches for `nodeIdentity` from `flatConstraintRegistry`, then appends entries for all path prefixes (e.g., identity `monitoring/node-exporter` also matches `monitoring`).

2. **Scope filtering.** `filterByScope` applies `inScope` to all candidate entries:
   - `scope == "global"` entries always pass.
   - `scope == "subtree"` entries pass only if `isAncestor ownerChain currentChain`.

3. **First match wins.** The first surviving entry determines the decision:
   - `type == "exclude"` → `{ action = "exclude"; owner }`
   - `type == "substitute"` → `{ action = "substitute"; replacement; owner }`
   - Other → `{ action = "keep" }`

4. **Filter fallback.** If no registry entry matched and the param includes an `aspect` attrset, `filterByScope` is applied to `flatConstraintFilters`. The first filter whose `predicate aspect` returns `false` produces `{ action = "exclude"; owner = failedFilter.owner }`.

5. **Default.** If nothing matched, resumes `{ action = "keep" }`.

### 4.2 Scope Ancestry (`filterByScope` / `isAncestor`)

```nix
isAncestor = ownerChain:
  lib.take (builtins.length ownerChain) currentChain == ownerChain;
```

`currentChain` is the includes chain for the current scope, maintained by `chain-push`/`chain-pop` effects. When an aspect with a meaningful name begins resolution, its identity is pushed; when resolution completes, it is popped. This gives a breadcrumb trail of the current position in the include tree.

A subtree constraint registered when the chain was `["root", "parent"]` matches any check where `currentChain` starts with those elements — i.e., within the registering aspect's descendant subtree. Constraints registered at a higher chain depth have **parent authority**: their excludes apply to children and cannot be overridden by descendants.

---

## 5. `meta.excludes`

Declared on the aspect `meta` option in `types.nix`:

```nix
options.excludes = lib.mkOption {
  description = "Aspects or policies to exclude from this subtree (sugar for handleWith)";
  type = lib.types.listOf lib.types.unspecified;
  default = [ ];
};
```

This is sugar for `meta.handleWith`. `registerConstraints` converts each element to an exclude-type constraint with `scope = "subtree"` and appends it to the `handleWith` list before dispatch. Both participate in the same `register-constraint` effect sequence.

Usage:

```nix
den.aspects.server-host = {
  includes = with den.aspects; [ relay ];
  meta.excludes = [ den.aspects.monitoring._.nginx-exporter ];
};
```

---

## 6. `policy.exclude`

**Constructor:** `nix/lib/policy-effects.nix`

```nix
policy.exclude aspect   # → { __policyEffect = "exclude"; value = aspect; }
```

A policy effect returned by a policy function. The pipeline collects all `exclude` effects from policy dispatch (`classify.nix` → `excludeEffects`) and emits them via `policyEmitExcludes` in `apply.nix`:

```nix
policyEmitExcludes = sendEach "register-constraint" (e: {
  type = "exclude";
  scope = "subtree";
  identity = identity.key e.value;
  owner = "policy";
});
```

This converts each policy exclude effect into a standard `register-constraint` call with `owner = "policy"`. Once registered, enforcement is identical to manually declared excludes — prefix lookup, scope filtering, first-match wins.

`emitPolicyEffectsThen` in `apply.nix` ensures excludes are emitted before route/instantiate/provide effects in the policy application sequence.

**Policy schema path:** `policy.exclude` effects are also checked in `schema.nix` (`isExcluded`) to filter out policies that have themselves been excluded before late dispatch.

---

## 7. Gate Integration

**File:** `nix/lib/aspects/fx/handlers/gate.nix`

The `gate` effect is a composite that sequences dedup and constraint checking so compilers send a single effect rather than two:

```
gate { aspect, identity, ctx }
  → check-dedup aspect
      → isDuplicate? → resume { blocked = true; result = [] }
      → check-constraint { identity, aspect }
          → action = "exclude"
              → identity.tombstone aspect { excludedFrom }
              → resolve-complete tombstone
              → include-unseen dedupKey   (if dedupKey != null)
              → resume { blocked = true; result = [tombstone] }
          → action = "substitute"  (vestigial path)
              → identity.tombstone aspect { excludedFrom, replacedBy }
              → resolve-complete tombstone
              → resolve { aspect = replacement, ... }
              → resume { blocked = true; result = [tombstone] ++ replacementResult }
          → action = "keep"
              → resume { passed = true; owner? }
```

**Dedup precedes constraint.** An already-seen aspect is blocked before constraint evaluation; constraints only run for aspects not yet seen in the current scope.

**Tombstone on exclude.** When the gate produces an exclude decision, `resolve-complete` is sent with the tombstone before the gate resumes. This records the excluded aspect in the trace. If a `dedupKey` exists, `include-unseen` is also sent — this writes the tombstone key into the seen set, preventing any later include of the same aspect from occupying the dedup slot. This is the **tombstone invariant**: an excluded slot cannot be reclaimed.

**Callers:** `compile-static` and `compile-forward` in the static compilation path. See `aspect-compilation.md` for the full gate call context within the compile pipeline.

---

## 8. Invariants

- **Parent authority.** A constraint registered by an ancestor (shallower ownerChain) is authoritative over descendants. Children cannot override an exclude their ancestor has placed — `isAncestor` is unidirectional: ancestors match descendants, not the reverse.

- **Tombstone prevents slot reuse.** When an aspect is excluded, `include-unseen dedupKey` writes the tombstone into the dedup seen set. Any subsequent include of the same identity returns `isDuplicate = true` before even reaching constraint evaluation. The excluded slot is permanently closed for this pipeline run.

- **First match wins.** Registry entries are not merged or composed. The first in-scope entry for an identity determines the action. Registration order is append-only within a scope; earlier registrations take precedence.

- **Filters are secondary.** Filter predicates are only evaluated when no registry entry (exact or prefix) matched and survived scope filtering. A registry exclude always outranks a filter.

- **Scope isolation.** `flatConstraintRegistry` and `flatConstraintFilters` are used at check time (cross-scope visibility), but the `ownerChain` embedded in each entry encodes where it was registered. Scope partitioning in `scopedConstraintRegistry` supports future per-scope constraint queries; the active check path uses the flat versions with `filterByScope`.

---

## 9. Key Files

| File | Role |
|------|------|
| `nix/lib/aspects/fx/constraints.nix` | Constraint constructors: `exclude`, `filterBy`, `substitute` |
| `nix/lib/aspects/fx/handlers/constraint.nix` | `register-constraint` and `check-constraint` handlers; `lookupEntries`, `filterByScope`, `isAncestor` |
| `nix/lib/aspects/fx/handlers/gate.nix` | `gate` composite effect: dedup + constraint + tombstone |
| `nix/lib/aspects/fx/aspect/children.nix` | `registerConstraints`: reads `aspect.excludes` and `meta.handleWith`, emits `register-constraint` |
| `nix/lib/aspects/types.nix` | `meta.excludes` option declaration |
| `nix/lib/policy-effects.nix` | `policy.exclude` effect constructor |
| `nix/lib/aspects/fx/policy/apply.nix` | `policyEmitExcludes`: converts policy exclude effects to `register-constraint` |
| `nix/lib/aspects/fx/policy/schema.nix` | Policy-level `isExcluded` check against `flatConstraintRegistry` |

Cross-reference: `aspect-compilation.md` §2–§4 for the gate call site within the compile pipeline and the full static compile flow.
