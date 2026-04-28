# Unified Aspect Key Type

**Target branch:** `feat/traits`
**Date:** 2026-04-27
**Status:** Draft
**Prerequisites:** Class emission dedup (complete), traits (complete), stages elimination (complete)
**Prerequisite investigation:** `den.classes`/`den.traits` must be accessible from `types.nix` — see "Registry Access" section

## Problem

The aspect submodule uses two distinct freeform types for different roles:

- **`aspectContentType`** — wraps values with `__contentValues` provenance. Used as the freeform type on `aspectSubmodule`. Handles class modules and trait data.
- **`providerType`** — produces proper aspect shapes (`name`, `meta`, `__functor`, `includes`). Used inside the `provides` submodule for nested aspects.

With the `provides` removal, all freeform keys live at the same level. But `aspectContentType` wraps everything uniformly — nested aspects lose their aspect shape, breaking `wrapChild`, `hasAspect`, and include references like `den.aspects.foo.sub`.

## Design

### Core insight: labeled heterogeneous record

Borrowing from Shapeless's labeled HList pattern: an aspect is a flat record where each key's **label determines its type**. The key name — not the nesting level — tells the system what kind of value it is.

```nix
den.aspects.nginx = {
  # Key "nixos" → ClassModule: provenance-wrapped content
  nixos = { config, ... }: { services.nginx.enable = true; };

  # Key "firewall" → SemanticData: provenance-wrapped trait emission
  firewall = { port = 80; };

  # Key "to-users" → Aspect: proper aspect shape (name, meta, __functor)
  to-users = { user, ... }: { homeManager.programs.nginx-tools.enable = true; };
};
```

The registry (`den.classes`, `den.traits`) acts as the witness — it tells the merge function which shape to produce for each key.

### Unified type: `aspectKeyType`

One type replaces both `aspectContentType` and `providerType` as the aspect submodule's freeform type. The merge function dispatches on the key name:

```
aspectKeyType.merge(loc, defs) =
  let key = lib.last loc; in
  if classes ? key  → ClassModule merge (provenance, preserves parametric)
  if traits ? key   → SemanticData merge (provenance, preserves parametric)
  otherwise         → Aspect merge (providerType: proper shape, coerces parametric)
```

All three branches handle parametric functions — they differ in HOW.

### ClassModule branch (registered class keys)

For keys in `den.classes` (`nixos`, `homeManager`, `darwin`, `os`, `hjem`, etc.):

**Merge:** Provenance-wrapped `__contentValues` with file attribution. Multi-site defs preserved as list. Already-wrapped values (`__contentValues` from cross-submodule propagation) flattened to prevent double-wrapping.

**Parametric handling:** Functions preserved as-is in `__contentValues`. The pipeline resolves den args via `wrapClassModule` at emission time. **No coercion to includes** — class module functions stay as functions, the pipeline's `emitClasses` unwraps and passes each to `wrapClassModule`.

**Produced shape:**
```nix
{ __contentValues = [ { value = fn-or-module; file = "..."; } ... ]; __provider = [...]; }
```

This is the existing `aspectContentType` merge logic, unchanged.

### SemanticData branch (registered trait keys)

For keys in `den.traits` (`firewall`, `persistence`, `xdg-mime`, etc.):

**Merge:** Same as ClassModule — provenance-wrapped `__contentValues`.

**Parametric handling:** Functions preserved as-is. The `traitCollectorHandler` detects Tier 2 (pipeline-parametric) functions and resolves them at emission time. **No coercion to includes.**

**Produced shape:** Identical to ClassModule. `emitTraits` unwraps `__contentValues` and sends each value as a separate `emit-trait` effect.

### Aspect branch (everything else)

For keys NOT in either registry — nested aspects, mutual-provider children (`to-users`, `to-hosts`), sub-aspects:

**Merge:** Full `providerType` merge dispatch:
- Single def, attrset → return as-is (cheap, no submodule eval)
- Single def, bare parametric fn → raw wrapper `{ __fn, __args, name, meta }` (single-def guard avoids OOM from submodule eval)
- Single def, parametric wrapper (`__fn`/`__args`) → return wrapper directly
- Multi def → coerce parametric fns to `{ includes = [fn]; }`, merge through `aspectType` submodule
- Multiple `__functor` defs → error (ambiguous)

**Parametric handling:** **Coerced to `{ includes = [fn]; }` and merged.** Resolution happens in the fx pipeline via `aspectToEffect` → `bind.fn`. This is fundamentally different from class/trait — parametric nested aspects become includes that the pipeline resolves with context args.

**Provenance:** Parent tracking via `meta.provider = typeCfg.providerPrefix ++ [name]`, injected by `mergeWithAspectMeta`. The `typeCfg.providerPrefix` propagates the parent aspect's provider path, so `den.aspects.igloo.to-users` gets `meta.provider = ["igloo"]`.

**Produced shape:** Proper aspect attrset with `name`, `meta`, `__functor`, `includes` (from `mergeWithAspectMeta` via `aspectType`). For single-def bare fns, raw wrapper with `__fn`, `__args`. Both are valid inputs to `wrapChild` in the include handler.

### Why parametric handling differs by branch

| Branch | Parametric function | Pipeline resolution |
|--------|-------------------|-------------------|
| ClassModule | `{ user, config, ... }: { networking... }` — flat-form class module | `wrapClassModule` pre-applies den args, NixOS evaluates module-system args |
| SemanticData | `{ host }: { port = 80; }` — Tier 2 trait | `traitCollectorHandler` detects Tier 2, calls with ctx |
| Aspect | `{ user, ... }: { homeManager...; includes = [...] }` — parametric nested aspect | Coerced to include, `bind.fn` resolves args, `aspectToEffect` recurses |

The key distinction: class/trait functions are **terminal** — they produce final values consumed by NixOS or the trait system. Aspect functions are **structural** — they produce more aspect tree to recurse into. Coercing class modules to includes would break `wrapClassModule`; keeping aspect functions as content would lose the aspect shape.

## Registry access

### Prerequisite: `den.classes`/`den.traits` visibility from `types.nix` — RESOLVED

The merge function needs `den.classes` and `den.traits` (where `den` = `config.den`, passed as a parameter to `types.nix`).

**Investigation result:** No issue. The registries are properly declared in `modules/options.nix` and fully accessible. Confirmed by:
- Schema-registry tests `test-has-classes` (`den ? classes` → true) and `test-has-traits` (`den ? traits` → true)
- `types.nix` already accesses them at lines 179 and 200 for collision checks: `(den.traits or {}) ? ${name}` and `(den.classes or {}) ? ${name}`
- The earlier `builtins.attrNames` discrepancy was a stale evaluation context artifact, not a module system issue

**Safe access pattern:** `den.classes or {}` and `den.traits or {}` — the `or {}` fallback prevents evaluation-order issues. This is the standard pattern throughout the codebase.

### Circular dependency analysis

Assuming registry access works, the merge function accesses registries lazily:

```nix
classReg = den.classes or {};
traitReg = den.traits or {};
```

`den.classes` is populated from:
1. **Battery modules** (`den.classes.nixos = { ... }` in options.nix, home-manager.nix, os-class.nix, etc.) — no dependency on `den.aspects`
2. **Aspect-level declarations** (`den.aspects.*.classes`) collected by `aspect-schema.nix`

For (2), `aspect-schema.nix` does:
- `builtins.attrNames (config.den.aspects)` — forces key SET but not VALUES
- `aspects.${name}.classes or {}` — accesses declared option (default `{}`), NOT freeform keys
- Declared option access does NOT trigger freeform `aspectKeyType.merge`

Therefore: `classReg ? nixos` evaluates without triggering aspect freeform merges → no cycle.

**Risk:** If `aspect-schema.nix` is refactored to access freeform keys (e.g., for structural detection), the cycle would re-emerge. The separation between declared options (`.classes`, `.traits`) and freeform keys (class modules, trait data, nested aspects) is load-bearing.

## Performance

| Branch | Cost | Why |
|--------|------|-----|
| ClassModule | Cheap | `__contentValues` wrapper, no submodule eval |
| SemanticData | Cheap | Same as ClassModule |
| Aspect (single def) | Cheap | Raw wrapper return, single-def guard skips submodule eval |
| Aspect (multi def) | Moderate | Full `aspectType` submodule merge (necessary for correctness) |

The single-def guard is critical: without it, every nested aspect key triggers a full `aspectSubmodule` evaluation (module system fixed-point + `den.schema.aspect` import), causing OOM on large aspect trees.

## Files changed

| File | Change |
|---|---|
| `nix/lib/aspects/types.nix` | New `aspectKeyType`; replace `aspectContentType` as freeform type; integrate class/trait/aspect branches |
| `nix/lib/aspects/types.nix` | Export `aspectKeyType`; `aspectContentType` becomes internal helper for class/trait branch |
| `nix/lib/aspects/fx/aspect.nix` | Keep `__contentValues` unwrapping in `emitClasses`/`emitTraits` (backward compat during migration) |

## Migration path

1. ~~**Prerequisite:** Ensure `den.classes`/`den.traits` accessible from `types.nix`~~ — **RESOLVED**, no changes needed
2. **Implement `aspectKeyType`** with three-branch dispatch
3. **Replace freeform type** on `aspectSubmodule`: `lazyAttrsOf (aspectContentType typeCfg)` → `lazyAttrsOf (aspectKeyType typeCfg)`
4. **Verify:** All existing tests pass (class modules, traits, nested aspects all produce correct shapes)
5. **Then:** Provides removal becomes safe — `den.aspects.foo.sub` produces aspect shape via the Aspect branch

## Non-goals

- Removing `provides` option (depends on this, separate task)
- Changing `classifyKeys` dispatch logic (still classifies at pipeline time)
- Changing `includes` list element type (stays `providerType`)
- Removing `aspectContentType` (kept as internal helper for class/trait branch merge)

## Test cases

- Class key with single def: `den.aspects.foo.nixos = { config, ... }: ...` → `__contentValues` shape
- Class key with multi def: two modules set same `nixos` → both in `__contentValues`
- Trait key: `den.aspects.foo.firewall = { port = 80; }` → `__contentValues` shape
- Parametric class: `den.aspects.foo.nixos = { user, config, ... }: ...` → function preserved in `__contentValues`
- Parametric trait: `den.aspects.foo.firewall = { host }: { port = 80; }` → function preserved
- Nested aspect (single def): `den.aspects.foo.sub = { nixos = ...; includes = []; }` → proper aspect shape
- Nested aspect (parametric): `den.aspects.foo.to-users = { user, ... }: { homeManager... }` → raw wrapper with `__fn`, `__args`
- Nested aspect (multi def): two modules set same path → coerced to includes, merged
- Nested aspect as ref: `den.aspects.bar.includes = [ den.aspects.foo.sub ]` → `wrapChild` handles aspect shape
- `hasAspect` on nested: `igloo.hasAspect den.aspects.foo.sub` → `aspectPath` computes from `meta.provider` + `name`
- Forward class (`os`): registered in `den.classes` → ClassModule branch, not Aspect
