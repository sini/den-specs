# Provides Namespace

The `provides`/`_` namespace is a virtual sub-aspect container declared as a structured option on every aspect. It is the permanent user-facing API for sub-aspect organization, named content delivery, and cross-entity routing. `_` is an alias for `provides` via `lib.mkAliasOptionModule`.

## User-Facing API

### Sub-aspect keys

Any key under `provides` (or `_`) that matches the aspect's own name is the self-provide slot (see Self-Provide below). All other keys are treated as sub-aspects or cross-entity delivery targets.

```nix
den.aspects.steam._ = {
  enable = { ... };    # sub-aspect, selectable by includes
  gaming = { ... };    # sub-aspect
};

# Reference sub-aspects from includes:
den.aspects.tux.includes = with den.aspects.steam._; [ enable gaming ];
# Enumerate all:
lib.attrValues den.aspects.security._
```

### provides.to-hosts

Fires when both `host` and `user` are present in the dispatch context. Intended for content that should flow from a user-scoped aspect to every host.

```nix
den.aspects.alice._.to-hosts = { host, user, ... }: {
  nixos.users.users.${user.userName}.packages = [ ... ];
};
```

### provides.to-users

Fires when both `host` and `user` are present in the dispatch context. Intended for content that should flow from a host-scoped aspect to every user. Dispatch behavior is identical to `to-hosts`; the names document intent, not mechanics.

```nix
den.aspects.desktop._.to-users = { host, user, ... }: {
  homeManager.programs.waybar.enable = true;
};
```

### provides.\<name\> (named targeting)

Fires when `host.name == key || user.name == key`. Delivers content to a specific named entity.

```nix
den.aspects.igloo._.alice = { host, user, ... }: {
  homeManager.programs.ssh.enable = true;
};
```

### _ alias

`_` is identical to `provides` at the type level. Both refer to the same underlying submodule option. Use whichever is more readable; they merge correctly across files.

```nix
# These three are equivalent:
den.aspects.foo._.to-users = { ... };
den.aspects.foo.provides.to-users = { ... };
# Deep-set from a separate file:
den.aspects.foo._.to-users.homeManager.programs.git.enable = true;
```

## Self-Provide

When `provides.${aspect.name}` is set, the aspect uses itself as a provider. This is the mechanism that makes an aspect callable as a dependency without requiring the caller to know its internal structure.

**Module-system side:** `provides` is a `providerType` freeform submodule. The self-provide slot is a first-class key — it merges with the rest of `provides`, is enumerable, and can be deep-set.

**Pipeline side:** `emitAspectPolicies` detects `provides ? ${aspect.name}` after processing cross-entity keys. When present, it emits an `emit-include` effect that feeds the self-provide value back into the include stream as a synthetic include. The include carries `meta.selfProvide = true` and the aspect name for tracing.

**Entity-resolution side:** `resolveEntity` (in `nix/lib/resolve-entity.nix`) adds a bootstrap include for each entity kind that carries an `aspect` field. This include is a parametric wrapper:

```nix
{
  __fn = c: c.${name}.aspect;
  __args = { ${name} = false; };
  name = "<self:${name}>";
  meta = { };
  includes = [ ];
}
```

The `__fn` is deferred until the pipeline provides the entity binding in scope. Once resolved, it injects `host.aspect` (or `user.aspect`, etc.) into the include stream. This is the initial entry point that makes the entity's own aspect subtree reachable. See `entity-resolution.md` for schema-level entity resolution.

**Parametric self-provide:** When the value under `provides.${name}` is a positional function (`lib.functionArgs fn == {}`), `mkSelfProvideInclude` calls it with the current scope context immediately and wraps the result. When it has named args, it produces a deferred parametric wrapper (`__fn`/`__args`) that the pipeline resolves later when the required args are available.

## Internal Routing

Cross-entity provides keys (`to-hosts`, `to-users`, `<name>`) are translated into registered aspect policies during the pipeline walk. This happens in `emitAspectPolicies` (`nix/lib/aspects/fx/aspect/provide.nix`).

For each key `k` in `aspect.provides` where `k != aspect.name` and `k` is not a schema entity kind:

| Key | Policy function | When it fires |
|-----|----------------|---------------|
| `to-hosts` | `{ host, user, ... }: [ (policy.include (applyProvide value ctx)) ]` | Every host×user pair |
| `to-users` | Same as to-hosts | Every host×user pair |
| `<name>` | `lib.optionals (host.name == key || user.name == key) [ (policy.include ...) ]` | When entity name matches |

`applyProvide` (`nix/lib/aspects/fx/contentUtil.nix`) resolves the value shape: parametric wrapper (`__fn`), functor (`__functor`), bare function, or plain attrset.

Each cross-entity key registers a `register-aspect-policy` effect with name `"${aspectName}/${key}"`. The policy is stored in `state.scopedAspectPolicies` and dispatched later by `installPolicies` at entity resolution time, when the full `{ host, user }` context is available.

Schema entity kind keys (`host`, `user`, `home`, etc.) under `provides` are filtered out by `emitAspectPolicies`. They are not registered as cross-entity policies.

See `policy-system.md` for `policy.include`, `policy.provide`, and policy dispatch mechanics.

## Compat Shim

**Status: removed.** The `provides-compat.nix` handler existed in earlier iterations of `feat/fx-pipeline` as a bridge from the main-branch `mutual-provider` approach. It has been removed; its logic was folded directly into `emitAspectPolicies`.

The `mutual-provider-shim.nix` module has also been removed. Users who included `den._.mutual-provider` in older configs no longer need to do so — cross-entity routing is built into the pipeline unconditionally.

There are no deprecation warnings for `provides`/`_` usage. The namespace is permanent.

## Status

Migration of template files to use `aspect.policies.*` directly instead of `provides.<name>` is not in progress. The `provides`/`_` syntax is supported indefinitely. New documentation examples prefer `policies.*` for policy-style delivery and direct `includes` nesting for sub-aspect composition, but both forms are equally valid.

## Key Files

| File | Role |
|------|------|
| `nix/lib/aspects/fx/aspect/provide.nix` | `emitAspectPolicies`: cross-entity policy registration and self-provide emission |
| `nix/lib/aspects/fx/contentUtil.nix` | `applyProvide`: shared shape detection for provides values |
| `nix/lib/aspects/types.nix` | `provides` option declaration (`providerType` freeform submodule); `_` alias via `mkAliasOptionModule` |
| `nix/lib/resolve-entity.nix` | Entity self-provide bootstrap (`<self:${name}>` wrapper) |
| `nix/lib/aspects/fx/handlers/resolve-children.nix` | Pipeline sequence: `emitAspectPolicies` → `emitIncludes` → `installPolicies` |
| `modules/aspects/provides/` | Battery modules: `host-aspects.nix`, `to-hosts.nix`, `to-users.nix`, `forward.nix`, etc. |
