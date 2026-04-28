# Aspect Rewrites: Excludes, Transformations, and Trace for Den

**Date:** 2026-04-03
**Status:** Design — pending implementation
**Scope:** `den.lib.aspects` API extension, `resolve` internals, type system additions

---

## Motivation

Den identifies aspects for deduplication during resolution, but provides no mechanism to remove an aspect from the DAG. Users migrating from other configuration frameworks (which support `excludes` at both feature and host levels) need the ability to:

1. **Exclude** an aspect entirely — entity-level (host/home/user) or subtree-scoped (aspect-level)
2. **Replace** one aspect with another (e.g., swap `gdm` for `regreet`)
3. **Transform** aspects during resolution (patch includes, strip class configs)
4. **Trace** resolution decisions for debugging

Rather than implementing these as separate features, this design introduces a single composable primitive — **rewrites** — that generalizes all four.

---

## Core Primitive: Rewrites

A **rewrite** is a Kleisli arrow in the Maybe monad over aspects. It receives a context record and returns the aspect (possibly modified), `null` (prune the subtree), or a rich result containing trace entries.

### Rewrite Input

Every rewrite receives a context record:

```nix
{
  provided   # attrset — the evaluated aspect at this node
  chain      # list — ancestor aspects from root to parent
}
```

Rewrites declare what they need via standard Nix function argument destructuring:

```nix
# Simple — only needs the aspect
{ provided, ... }: if condition then null else provided

# Context-aware — needs the ancestor chain
{ provided, chain, ... }: ...
```

### Rewrite Output (polymorphic, normalized by resolver)

```nix
# Prune this aspect and its entire subtree
null

# Include (possibly modified)
provided

# Rich form — include result and emit trace entries
{ result = provided; trace = [{ ... }]; }

# Rich form — prune and emit trace entries
{ result = null; trace = [{ ... }]; }
```

### Normalization

The resolver normalizes all return forms to `{ result; trace; }`:

```nix
normalizeResult = raw:
  if raw == null then            { result = null; trace = []; }
  else if raw ? result then      raw // { trace = raw.trace or []; }
  else if builtins.isAttrs raw then  { result = raw; trace = []; }
  else throw "rewrite must return an aspect attrset, null, or { result; trace; }";
```

---

## Public API

### `den.lib.aspects.resolve` (unchanged)

```nix
resolve :: class -> aspect -> { imports; }
```

Backwards-compatible. Same signature, same return type. Internally calls `resolveWith` with default opts.

### `den.lib.aspects.resolve'` (new)

```nix
resolve' :: class -> opts -> aspect -> { module; trace; }
```

Power API. Accepts an opts record, returns a resolution result (not a module directly).

**Opts record:**

```nix
{
  rewrites = [];    # list of rewrites, composed via Kleisli composition
  trace = false;    # when true, auto-wrap composed rewrite with trace functor
}
```

**Return value:**

```nix
{
  module = { imports = [...]; };    # valid Nix module
  trace = [...];                    # list of trace entries (empty unless traced)
}
```

### Rewrite Constructors (new)

```nix
# Identity rewrite — pass-through
den.lib.aspects.id :: rewrite

# Exclude by name — O(1) set lookup, prunes matching aspects
den.lib.aspects.exclude :: [string] -> rewrite

# Replace by name — substitutes one aspect for another
den.lib.aspects.replace :: string -> aspect -> rewrite
```

### Rewrite Combinators (new)

```nix
# Kleisli composition — chains rewrites, short-circuits on null, threads trace
den.lib.aspects.compose :: [rewrite] -> rewrite

# Trace functor — lifts any rewrite to emit trace entries
den.lib.aspects.trace :: rewrite -> rewrite
```

---

## Semantics

### Two-Tier Exclusion Model

| Level | Scope | Declared at | Mechanism |
|-------|-------|-------------|-----------|
| **Entity-level** | Global for that entity's resolution | `den.hosts.*.excludes`, `den.homes.*.excludes`, or via `den.schema.*` | Desugars to an `exclude` rewrite prepended before resolution |
| **Aspect-level** | Subtree only — propagates to children, not siblings | `den.aspects.*.excludes` | Desugars to an `exclude` rewrite prepended for children during recursion |

### Invariants

1. An excluded aspect contributes nothing: no owned configs, no includes, no provides, no transitive dependencies. It does not exist in the DAG.
2. Entity-level excludes are order-independent (pre-seeded before traversal).
3. Aspect-level excludes are subtree-scoped: they propagate to children but not to siblings.
4. Exclusion is by aspect `name` (string), matching `provided.name` after functor evaluation.
5. Within a subtree scope, exclusion wins over inclusion.
6. The `rewrites` list composes left-to-right. Entity excludes run first, then aspect excludes, then user rewrites.

### Rewrite Evaluation Order Per Node

```
1. Evaluate the aspect (call functor if function)
2. Build effective rewrite for THIS node from opts.rewrites:
   a. Inherited aspect-level excludes from ancestors (already in opts)
   b. Entity-level excludes (already in opts, pre-seeded before resolution)
   c. User rewrites from opts.rewrites
   NOTE: This node's own `excludes` are NOT applied to itself — only to children
3. If trace = true, wrap composed rewrite with trace functor
4. Apply composed rewrite to { provided, chain }
5. Normalize result
6. If result is null, return { imports = []; trace = ...; }
7. Otherwise, build child opts:
   a. Prepend this node's `excludes` (as an exclude rewrite) to opts.rewrites
   b. Carry forward all inherited rewrites
8. Recurse into result.includes with child opts
```

---

## Implementation

### `resolveWith` (internal)

```nix
resolveWith = class: opts: aspect-chain: aspect:
  let
    provided = if lib.isFunction aspect
               then aspect { inherit class aspect-chain; }
               else aspect;

    # Build effective rewrite for this node (user rewrites only —
    # aspect-level excludes apply to children, not the declaring node itself)
    allRewrites = opts.rewrites or [];

    composed = if allRewrites == [] then null else compose allRewrites;
    effectiveRewrite =
      if opts.trace or false
      then trace (if composed != null then composed else id)
      else composed;

    # Apply rewrite
    applied =
      if effectiveRewrite != null
      then normalizeResult (effectiveRewrite { inherit provided; chain = aspect-chain; })
      else { result = provided; trace = []; };

  in
  if applied.result == null then
    { imports = []; inherit (applied) trace; }
  else
    let
      walked = applied.result;

      # Child opts: carry forward user rewrites + this aspect's excludes for its children
      childExcludes =
        lib.optional ((walked.excludes or []) != []) (exclude walked.excludes);
      childOpts = opts // {
        rewrites = childExcludes ++ (opts.rewrites or []);
      };

      next-chain = aspect-chain ++ [ walked ];
      children = map (resolveWith class childOpts next-chain)
                     (walked.includes or []);
    in
    {
      imports = (lib.optional (walked ? ${class}) walked.${class})
             ++ (map (c: { inherit (c) imports; }) children);
      trace = applied.trace ++ (lib.concatMap (c: c.trace) children);
    };
```

### Rewrite Helpers

```nix
id = { provided, ... }: provided;

exclude = names:
  let set = lib.genAttrs names (_: true);
  in { provided, ... }:
    let aspectName = provided.name or null;
    in if aspectName != null && set ? ${aspectName} then null else provided;

# Note: replace substitutes the aspect wholesale. The replacement's own
# `name` governs subsequent rewrite matching — if the replacement has a
# different name, rewrites targeting the original name will not match it.
replace = name: replacement: { provided, ... }:
  let aspectName = provided.name or null;
  in if aspectName != null && aspectName == name then replacement else provided;

compose = rewrites: ctx:
  builtins.foldl' (acc: f:
    if acc.result == null then acc
    else
      let next = normalizeResult (f (ctx // { provided = acc.result; }));
      in {
        result = next.result;
        trace = acc.trace ++ next.trace;
      }
  ) { result = ctx.provided; trace = []; } rewrites;

trace = rewrite: ctx@{ provided, chain, ... }:
  let
    applied = normalizeResult (rewrite ctx);
    aspectName = provided.name or "<anon>";
    resultName =
      if applied.result != null
      then (applied.result.name or "<anon>")
      else null;
    decision =
      if applied.result == null then "pruned"
      else if resultName != aspectName then "replaced"
      else "included";
  in {
    inherit (applied) result;
    trace = applied.trace ++ [{
      name = aspectName;
      depth = builtins.length chain;
      chain = map (a: a.name or "<anon>") chain;
      inherit decision;
    }];
  };
```

### Public API Wrappers

```nix
# aspects/default.nix additions:

defaultOpts = { rewrites = []; trace = false; };

resolve = class: aspect:
  let r = resolveWith class defaultOpts [] aspect;
  in { inherit (r) imports; };

resolve' = class: opts: aspect:
  let r = resolveWith class opts [] aspect;
  in { module = { inherit (r) imports; }; inherit (r) trace; };
```

### Call Site: `mainModule` in `types.nix`

The current `mainModule` calls `den.lib.aspects.resolve` directly. It must switch to `resolve'` to wire entity-level excludes into the resolution pipeline. Without this change, `den.hosts.*.excludes` would have no effect.

```nix
mainModule = from: intent: name:
  let
    asp = intent { ${name} = from; };
    entityExcludes =
      lib.optional ((from.excludes or []) != [])
        (den.lib.aspects.exclude from.excludes);
    result = den.lib.aspects.resolve' (from.class) {
      rewrites = entityExcludes;
    } asp;
  in
  result.module;
```

### Call Site: `forward.nix`

Unchanged. `den.lib.aspects.resolve fromClass asp` continues to work with the backwards-compatible signature.

### Note: Initial Chain

Both `resolve` and `resolve'` pass `[]` as the initial `aspect-chain`. The root aspect appears in the chain only after evaluation, consistent with all other nodes. This means `chain` at depth 0 is empty, and `chain` at depth N contains N evaluated ancestor aspects.

This is a minor behavioral change from the current `resolve.nix` which passes `[ aspect ]` (the unevaluated form) as the initial chain. The current behavior is inconsistent — the root chain element is unevaluated while all subsequent elements are evaluated. Since `aspect-chain` is passed to functors via `aspect { inherit class aspect-chain; }`, any functor inspecting the chain at depth 0 will now see `[]` instead of `[ <unevaluated> ]`. This is unlikely to affect real-world code, but should be noted in the PR.

### Note: Pre-existing Issue in Bogus Template

`templates/bogus/modules/test-base.nix` calls `resolve "funny" [] aspect` using a 3-argument form that does not match the current 2-argument public API. This pre-dates this spec and should be fixed independently.

---

## Type System Changes

### `aspects/types.nix` — Aspect Submodule

Add `excludes` alongside existing `includes`:

```nix
excludes = lib.mkOption {
  description = "Aspect names to exclude from this aspect's include subtree";
  type = lib.types.listOf lib.types.str;
  default = [];
};
```

### Entity Types — Host, Home, User

Add `excludes` as an explicit option on the host and home type definitions in `types.nix` (inside `hostType` and `homeType` submodules), accessible via `from.excludes`:

```nix
excludes = lib.mkOption {
  description = "Aspect names to exclude from resolution for this entity";
  type = lib.types.listOf lib.types.str;
  default = [];
};
```

For user-level excludes, add the option to `den.schema.user` or to the `userType` submodule in `types.nix`. This makes `excludes` available on all entity types that enter the resolution pipeline.

Note: `den.schema.conf` does not currently exist in the codebase. If a shared base schema is added in the future, `excludes` could be moved there to reduce duplication.

---

## Backwards Compatibility

| Surface | Impact |
|---------|--------|
| `den.lib.aspects.resolve` | **Unchanged** — same signature, same return type |
| `den.lib.aspects.types` | **Unchanged** |
| `den.aspects.*.includes` | **Unchanged** |
| `den.aspects.*.excludes` | **New option** — defaults to `[]`, no effect if unused |
| Entity `.excludes` | **New option** — defaults to `[]`, no effect if unused |
| `den.lib.aspects.resolve'` | **New function** |
| `den.lib.aspects.{exclude,replace,compose,trace,id}` | **New functions** |

Zero breaking changes to public API signatures or return types. All new surface area is additive.

**Minor behavioral note:** The initial `aspect-chain` passed to root-level functors changes from `[ <unevaluated-aspect> ]` to `[]`. This fixes an inconsistency in the current implementation (see "Note: Initial Chain" above) but could affect functors that inspect chain length at depth 0.

---

## Usage Examples

### Entity-level exclude

```nix
# Kubernetes nodes: no tailscale
den.hosts.x86_64-linux.k8s-node.excludes = [ "tailscale" ];

# Standalone home: no bloated shell config
den.homes.x86_64-linux.alice.excludes = [ "bloated-shell-config" ];

# All users default: exclude something globally
den.schema.user = { lib, ... }: {
  config.excludes = lib.mkDefault [ "deprecated-tool" ];
};
```

### Aspect-level subtree exclude

```nix
# kubernetes-node is server-minus-tailscale
den.aspects.kubernetes-node = {
  excludes = [ "tailscale" ];
  includes = [ den.aspects.server ];
};

# If a host includes both workstation and kubernetes-node:
den.aspects.myhost.includes = [
  den.aspects.workstation       # tailscale included here (unaffected)
  den.aspects.kubernetes-node   # tailscale excluded in this subtree only
];
# To exclude globally, use entity-level: den.hosts.*.excludes = [ "tailscale" ];
```

### Programmatic rewrites

```nix
# Replace a greeter
den.lib.aspects.resolve' "nixos" {
  rewrites = [ (den.lib.aspects.replace "gdm" den.aspects.regreet) ];
} myAspect

# Compose multiple rewrites
den.lib.aspects.resolve' "nixos" {
  rewrites = [
    (den.lib.aspects.exclude [ "tailscale" "avahi" ])
    (den.lib.aspects.replace "gdm" den.aspects.regreet)
    ({ provided, ... }: builtins.removeAttrs provided [ "darwin" ])
  ];
} myAspect

# Custom rewrite with chain awareness
den.lib.aspects.resolve' "nixos" {
  rewrites = [
    ({ provided, chain, ... }:
      let parentNames = map (a: a.name or "") chain;
      in if builtins.elem "server" parentNames && (provided.name or "") == "tailscale"
         then null
         else provided)
  ];
} myAspect
```

### Trace

```nix
# Quick trace via opts
den.lib.aspects.resolve' "nixos" {
  rewrites = [ (den.lib.aspects.exclude [ "tailscale" ]) ];
  trace = true;
} myAspect
# -> {
#   module = { imports = [...]; };
#   trace = [
#     { name = "myhost"; depth = 0; decision = "included"; chain = []; }
#     { name = "server"; depth = 1; decision = "included"; chain = ["myhost"]; }
#     { name = "tailscale"; depth = 2; decision = "pruned"; chain = ["myhost" "server"]; }
#   ];
# }

# Manual trace functor for fine-grained control
den.lib.aspects.resolve' "nixos" {
  rewrites = [
    (den.lib.aspects.trace (den.lib.aspects.exclude [ "tailscale" ]))
  ];
} myAspect
```

---

## Future Extensions (not implemented, documented for reference)

### Alternative naming

If community feedback prefers alternatives to `resolve'`:

- `den.lib.aspects.resolve.with` — nested attrset style, matches `den.lib.take.atLeast` / `den.lib.parametric.fixedTo`
- `den.lib.aspects.resolve.excluding` — dedicated convenience for the exclude-only case

### `replaces` sugar

Mutual exclusion between aspects providing the same role:

```nix
den.aspects.regreet.replaces = [ "gdm" "sddm" ];
# Desugars to: regreet.excludes = [ "gdm" "sddm" ] + gdm/sddm would exclude regreet if present
```

Requires convention for how mutual exclusion composes. Deferred pending real-world usage of `excludes`.

### Extended trace entries

Trace entries could include additional fields:

- `reason` — which rewrite produced the decision (entity-exclude, subtree-exclude, user rewrite)
- `excludedBy` — name of the aspect or entity whose `excludes` caused pruning
- `originalName` — for replacements, the name of the aspect that was replaced

### Rewrite context extensions

The context record passed to rewrites could include additional fields discovered during implementation:

- `class` — the class being resolved (available at call site, but convenient)
- `depth` — integer depth in the tree (derivable from `chain` length)
