# Class Module Partial Application

**Date:** 2026-04-24
**Branch:** `feat/class-module-partial-apply`
**Status:** Design approved

## Problem

Today, class modules (nixos, darwin, homeManager) are dispatched to the underlying OS module system untouched. Den context args like `host` are only available via the aspect-scope parametric wrapper pattern:

```nix
# Works today — two layers:
{ host }: { nixos = { config, pkgs, ... }: { networking.hostName = host.name; }; }
```

This requires users to write a curried two-layer function. A flat form where den args and module-system args coexist in a single attrset would be more ergonomic:

```nix
# Desired — flat:
{ name = "foo"; nixos = { host, config, pkgs, ... }: { networking.hostName = host.name; }; }
```

## Design

### Approach: Wrap in `emitClasses` (Approach A)

Add a `wrapClassModule` helper in `emitClasses` (`aspect.nix`) that intercepts class modules before they are sent as `emit-class` effects. The wrapping:

1. If `module` is not a function → pass through unchanged. (Note: attrsets with `__functor` are functions per `builtins.isFunction` and are handled by this same path.)
2. If `module` is a function → `builtins.functionArgs module` to get expected args.
3. Partition args: a key is a "den arg" if and only if `ctx ? ${k}` where `ctx = aspect.__ctx or {}`. This is fully dynamic — no hardcoded allowlist.
4. For each den arg present in context → pre-apply from `ctx`.
5. For each den arg NOT in context → `lib.warn "den: class module requests '${k}' but no ${k} context is available"`.
6. Return a wrapped function with preserved `functionArgs`:

```nix
let
  wrapper = moduleArgs: module (moduleArgs // denArgs);
  remainingArgs = removeAttrs (builtins.functionArgs module) (builtins.attrNames denArgs);
in
lib.setFunctionArgs wrapper remainingArgs
```

The wrapper MUST use `lib.setFunctionArgs` to advertise the remaining (module-system) args. Without this, the NixOS module system cannot discover what the module expects and will fail at eval time.

7. When wrapping occurs, force `isContextDependent = true` in the `emit-class` effect payload. Wrapped modules have baked-in den context, so dedup must use the full identity (with ctxId) to avoid collapsing distinct instances that differ only by context values.

**Constraint:** Flat-form class modules must use `...` (ellipsis) in their argument pattern. Without it, the NixOS module system's extra args (beyond what the module declares) will cause an eval error. This is already conventional for NixOS modules but worth noting since the flat form is new.

### Collision handling

When a den context key collides with a module-system arg (e.g., a class system passes `host` via `specialArgs` and `__ctx` also contains `host`), the collision is detected **at call time** — when the module system invokes the wrapped function — not at wrap time, since we can't know what the module system will pass in advance.

**Collision policy resolution** (first match wins):

1. `aspect.meta.collisionPolicy` → single enum, blanket override for this aspect
2. `ctx.${collidingArg}.collisionPolicy` → the colliding entity declares its own policy via schema
3. `den.config.classModuleCollisionPolicy` → global default

**Policy values:**

- `"error"` (global default) — throw on collision, force the user to resolve it
- `"class-wins"` — module-system value wins, den value is dropped, `lib.warn` emitted
- `"den-wins"` — den value wins, module-system value is shadowed, `lib.warn` emitted

**Schema integration:** `collisionPolicy` is defined on `den.schema.<kind>` as an optional enum. It travels with the entity value through `__ctx`, surviving policy renames and forwards. Since the colliding arg name IS the context key, the lookup is simply `ctx.${name}.collisionPolicy or null`.

**`aspect.meta.collisionPolicy`** is a single enum (not per-field). At the aspect level, you know exactly which class module you're writing — rename the arg if only one collision is the issue.

**Call-time wrapper logic:**

```nix
wrapper = moduleArgs:
  let
    collisions = builtins.intersectAttrs moduleArgs denArgs;
    resolvePolicy = name:
      if aspectPolicy != null then aspectPolicy
      else if builtins.isAttrs (ctx.${name} or null) && (ctx.${name} ? collisionPolicy)
        then ctx.${name}.collisionPolicy
      else globalPolicy;
    kept = lib.filterAttrs (name, _:
      let policy = resolvePolicy name;
      in
        if !(collisions ? ${name}) then true
        else if policy == "error" then
          throw "den: class module arg '${name}' collides with module-system arg — set collisionPolicy to resolve"
        else if policy == "class-wins" then
          lib.warn "den: class module arg '${name}' collision — class-wins, den value dropped" false
        else
          lib.warn "den: class module arg '${name}' collision — den-wins, module-system value shadowed" true
    ) denArgs;
  in
  module (moduleArgs // kept);
```

### Future alternative: Approach B — `emit-class-parametric` effect

If complexity grows (e.g., wrapping needs handler-level state or per-class customization), consider introducing a new `emit-class-parametric` effect that carries both the module and the context. This would move the wrapping logic into `classCollectorHandler` in `tree.nix`, separating emission from transformation. Not needed for the current scope.

## Edge Cases

### Two-layer form (unchanged)

```nix
{ host }: { nixos = { config, pkgs, ... }: ...; }
```

After parametric resolution, `nixos` is a closure that already captured `host`. `functionArgs` returns `{ config, pkgs }` — no overlap with `__ctx`. Wrapper is a no-op.

### Flat form with context (new)

```nix
{ name = "foo"; nixos = { host, config, pkgs, ... }: ...; }
```

Aspect is non-parametric but reached via a policy/transition that set `__ctx = { host = <host>; }`. `functionArgs` finds `host` in ctx, pre-applies it. Wrapped function advertises only `{ config, pkgs }` via `setFunctionArgs`. `isContextDependent` forced true.

### Missing context

```nix
{ name = "foo"; nixos = { host, config, pkgs, ... }: ...; }
```

Top-level aspect, no policy. `__ctx` is `{}`. `host` is not in ctx. `lib.warn` fires. Module passes through unwrapped — NixOS module system will fail with its own error if `host` has no default.

### Default values

```nix
{ nixos = { host ? null, config, ... }: ...; }
```

`host` not in ctx → warn, but module still works because the default kicks in. The warn is a hint, not a blocker.

### Multiple den args

```nix
{ nixos = { host, fleet, config, ... }: ...; }
```

Both `host` and `fleet` found in `__ctx` → both pre-applied. `setFunctionArgs` advertises only `{ config }`. Works identically to single-arg case.

### Arg collision with module system

```nix
# Class system also passes `host` via specialArgs
{ nixos = { host, config, pkgs, ... }: ...; }
```

Both `__ctx` and the module system provide `host`. At call time, `builtins.intersectAttrs moduleArgs denArgs` detects the collision. Policy resolved per the collision handling rules above. Default: error.

## Implementation Location

**File:** `nix/lib/aspects/fx/aspect.nix`, in the `emitClasses` function (currently lines 33–46).

The `wrapClassModule` helper sits alongside `emitClasses` and is called per class key before the `fx.send "emit-class"` effect.

## Testing

Tests live alongside existing cross-entity integration tests:

1. **Flat form with context** — aspect with `nixos = { host, config, ... }: ...` reached via a policy. Assert module evaluates and `host` is accessible.
2. **Two-layer form unchanged** — existing pattern continues to work identically.
3. **Missing context warning** — flat form with no context available. Assert warning is emitted and module passes through.
4. **Wrapped module functionArgs** — assert that `builtins.functionArgs` on the wrapped module returns only module-system args (den args removed).
5. **Multiple den args** — aspect with `{ host, fleet, config, ... }:` where both `host` and `fleet` are in context. Assert both are pre-applied.
6. **Functor module** — class module that is an attrset with `__functor`. Assert it is handled by the same wrapping path.
7. **Collision — error (default)** — class system passes an arg that collides with a den context key. Assert eval throws.
8. **Collision — class-wins** — entity sets `collisionPolicy = "class-wins"`. Assert module-system value is used, den value dropped.
9. **Collision — den-wins** — entity sets `collisionPolicy = "den-wins"`. Assert den value wins.
10. **Collision — aspect override** — `meta.collisionPolicy` overrides entity-level policy.
