# Direct-Ref Policy Aspects + Entity Simplification

**Date:** 2026-04-26
**Branch:** feat/traits
**Status:** Draft
**Prerequisite:** Stages elimination (complete)

## Problem

The stages elimination introduced `den.entityIncludes` as a per-kind aspect list with string-keyed lookup. This creates several workarounds:

1. **Empty registrations** — `entityIncludes.flake-system = []` needed solely for entity existence checks
2. **String indirection** — `policy.aspects = ["hm-host-module"]` requires name→value lookup at injection time (`map (name: den.aspects.${name}) policyAspectNames`)
3. **`rootIncludes` pipeline phase** — added to preserve self-provide-before-transitions ordering, but is a spec deviation and extra pipeline complexity
4. **Untyped** — `entityIncludes` uses `listOf raw`, no validation at definition time
5. **`resolveEntityHandler` existence check** — string membership test against entityIncludes keys

## Design

### 1. `policy.aspects` — direct refs, typed

`policy.aspects` changes from `listOf str` to `listOf providerType`:

```nix
# policy-types.nix
options.aspects = lib.mkOption {
  type = lib.types.listOf den.lib.aspects.types.providerType;
  default = [];
  description = "Aspects to include for entities resolved by this policy.";
};
```

Usage:

```nix
# Before
den.policies.host-to-hm-users.aspects = [ "hm-host-module" "hm-user-forward" ];

# After
den.policies.host-to-hm-users.aspects = with den.aspects; [ hm-host-module hm-user-forward ];
```

Injection site in `transition.nix` simplifies:

```nix
# Before
policyAspectNames = (transition.routing or {}).aspects or [];
policyAspects = map (name: den.aspects.${name}) policyAspectNames;

# After
policyAspects = transition.routing.aspects or [];
```

### 2. `resolveEntity` — self-provide from context

`resolveEntity` derives the root self-provide from `ctx.${kind}.aspect`:

```nix
resolveEntity = kind: ctx:
  let
    scopeHandlers = constantHandler ctx;
    entity = ctx.${kind} or null;
    hasAspect = entity != null && entity ? aspect;
  in
  {
    name = kind;
    meta = { handleWith = null; excludes = []; provider = []; into = null; };
    provides = lib.optionalAttrs hasAspect { ${kind} = _: entity.aspect; };
    includes = [];
    __ctxStage = kind;
    __scopeHandlers = scopeHandlers;
  };
```

Self-provide goes through the original `emitSelfProvide` path (resolved before transitions), restoring the pipeline ordering that deferred drain depends on.

**Known empty-shell cases:** `flake` (no ctx), `flake-system` (ctx has `{ system }` not an entity), custom schema kinds without `.aspect`. All produce empty entities — policy aspects populate them. This is correct.

**`default` entity** special case — `den.default` is not a context entity:

```nix
provides =
  if kind == "default" && den ? default then { default = _: den.default; }
  else if hasAspect then { ${kind} = _: entity.aspect; }
  else {};
```

### 3. Remove `rootIncludes`

With self-provides back in `provides.${kind}`, the `rootIncludes` phase in `resolveChildren` has no consumers. Remove it:

```nix
# aspect.nix resolveChildren — revert to original shape
childResolution = fx.bind (emitSelfProvide aspect) (selfProvResults:
  fx.bind (emitTransitions aspect) (transitionResults:
    fx.bind (emitIncludes emitCtx (aspect.includes or [])) (children:
      fx.pure (selfProvResults ++ transitionResults ++ children)
    )
  )
);
```

Remove `rootIncludes` from `structuralKeysSet`.

**Ordering invariant:** the self-provide (`provides.${kind}`) resolves via `emitSelfProvide` before `emitTransitions`. Deferred includes from the root aspect register before transitions widen context, then drain during transitions. This is the same ordering the original pre-stages pipeline used. No aspect migrating from `rootIncludes` to `provides` emits deferred includes that require transition context — os-class/os-user forwarders and mutual-provider move to `policy.aspects` (Section 7), not to provides.

### 4. Remove `entityIncludes`, `entityProvides`, and `entities.nix`

All three deleted. `nixModule/default.nix` removes the import.

Current `entityIncludes` consumers migrate:

| Current | New home |
|---|---|
| `host.nix`: `entityIncludes.host = [({ host }: host.aspect)]` | Deleted — `resolveEntity` derives from ctx |
| `user.nix`: `entityIncludes.user = [({ host, user }: user.aspect)]` | Deleted — same |
| `os-class.nix`: `entityIncludes.{host,user} = [fwd]` | `policy.aspects` on core policies |
| `os-user.nix`: `entityIncludes.user = [fwd]` | `policy.aspects` on core `host-to-users` |
| `defaults.nix`: `entityIncludes.default = [den.default]` | `resolveEntity "default"` special case |
| `home-manager.nix`: `entityIncludes.home = [({ home }: home.aspect)]` | Deleted — `resolveEntity` derives from ctx |
| `osConfigurations.nix`: `entityIncludes.flake-os = []` | Deleted — no existence check needed |
| `hmConfigurations.nix`: `entityIncludes.flake-hm = []` | Deleted — same |
| `wsl.nix`: `entityIncludes.wsl-host = [fn]` | `policy.aspects` on `host-to-wsl-host` |
| `flakeSystemOutputs.nix`: `entityIncludes.{flake-system,...} = []` | Deleted — no existence check needed |

### 5. `resolveEntityHandler` — always succeeds

No existence check. Every `resolve-entity` effect produces a valid (possibly empty) entity:

```nix
resolveEntityHandler = {
  "resolve-entity" = { param, state }:
    let
      kind = param.kind;
      currentCtx = (state.currentCtx or (_: {})) null;
    in
    { resume = den.lib.resolveEntity kind currentCtx; inherit state; };
};
```

The tombstone path in `resolveTransition` (`effectiveTarget == null && crossProvider == null`) becomes dead code for the `resolve-entity` path since the handler never returns null. Remove the null check — `effectiveTarget` is always a valid entity.

### 6. `options.nix` entity guard

Uses `den.schema` for known kinds:

```nix
schemaKinds = builtins.filter (n: n != "conf" && !(lib.hasPrefix "_" n))
  (builtins.attrNames (den.schema or {}));

# Guard: inject resolvedCtx when kind is a schema entity
if builtins.elem kind schemaKinds then { imports = [ merged resolvedCtx ]; } else merged;
```

`knownKinds` derived from `den.schema` only.

Note: `flake`, `flake-system`, `flake-packages`, etc. are NOT in `den.schema`. They don't get `resolvedCtx` injected. This is unchanged from current behavior — these entity kinds are populated entirely via transitions and policy aspects, not via schema entity evaluation.

### 7. Framework aspects on core policies

os-class, os-user, and mutual-provider move to `policy.aspects` on existing core policies:

```nix
# host-to-users (existing core policy from host entity activation)
# Carries user-level framework aspects
den.policies.host-to-users = {
  from = "host";
  to = "user";
  aspects = [
    user-os-fwd       # os-class.nix user forward
    os-user-fwd        # os-user.nix user class forward
    den.provides.mutual-provider
  ];
  ...
};
```

Host-level os-class (`host-os-fwd`) takes `{ host }` and forwards os→nixos/darwin. It fires during user transitions (which provide host context). The forward targets the host's NixOS config, so this is correct — the host OS module receives the forwarded content regardless of which transition carries the aspect.

### 8. `ctx-seen` accumulation — path-based identity

Policy aspects are direct values, not strings. The `ctx-seen` handler must:

1. **Track aspects by full path identity** (not just `.name`) for dedup — uses `identity.pathKey (identity.aspectPath aspect)` consistent with how the pipeline identifies aspects elsewhere
2. **Store direct ref values** in the seen state — supplemental emission replays these values directly instead of re-looking them up by name from `den.aspects`

```nix
# In transition.nix, sending ctx-seen:
aspectIdentities = map (a: identity.pathKey (identity.aspectPath a))
  (transition.routing.aspects or []);

fx.send "ctx-seen" {
  key = ctxKey;
  aspects = aspectIdentities;
  aspectValues = transition.routing.aspects or [];  # carry direct refs
}

# In ctx-seen handler, newAspects returns { newIds, newValues }
# Supplemental emission uses newValues directly — no den.aspects lookup
```

This eliminates the correctness hole where user-defined policy aspects not registered in `den.aspects` would fail the supplemental injection path.

### 9. `systemOutputFwd` runtime probe

`flakeSystemOutputs.nix` currently probes `den.entityIncludes."flake-${output}"` to decide whether to resolve the entity or fall back to `aspect-chain`. With `entityIncludes` deleted, this conditional dispatch changes:

`resolveEntityHandler` always returns a valid entity. If no user aspects exist for `flake-packages`, the entity is an empty shell — `resolveEntity` returns `{ provides = {}; includes = []; ... }`. The forwarder always resolves the entity; an empty entity produces no class content, which is equivalent to the `aspect-chain` fallback (the root aspect has no content for that output class).

Replace the runtime probe with unconditional entity resolution:

```nix
source = den.lib.resolveEntity "flake-${output}" { inherit system; };
```

### 10. `ctx-shim.nix` — update forwarding target

The compat shim currently forwards `den.ctx.*` to `den.entityIncludes`. With `entityIncludes` deleted, the shim must forward to a living target. Since `den.ctx` content is aspect data (class keys, includes), forward to `den.aspects`:

```nix
# den.ctx.foo = { nixos = ...; } → den.aspects."ctx:foo" = value
config.den.aspects = lib.mapAttrs' (name: value: {
  name = "ctx:${name}";
  value = lib.warn "den.ctx.${name} is deprecated — use den.aspects" (
    builtins.removeAttrs value ["into" "_module"]
  );
}) (builtins.removeAttrs config.den.ctx ["_module"]);
```

The `den.ctx.*.into` field is dropped with a deprecation warning — use policies instead. The `ctx:` prefix prevents name collisions with user-defined aspects.

The `den.ctx` option declaration remains (for backward compat). The shim module stays in `modules/compat/ctx-shim.nix`.

### 11. `makeHomeEnv` update

Switch from string aspect names to direct refs:

```nix
makeHomeEnv = { ... }: {
  aspects = { ... };  # unchanged — registered in den.aspects
  policies = {
    "host-to-${ctxName}-users" = {
      from = "host";
      to = "user";
      aspects = [
        result.aspects."${ctxName}-host-module"
        result.aspects."${ctxName}-user-forward"
      ];
      ...
    };
  };
};
```

Battery consumers (`home-manager.nix`, `hjem.nix`, `maid.nix`) pass through aspect values from the result, not string names.

### 12. `has-aspect.nix` error message

Update error message from `"(no matching den.entityIncludes.<kind> defined)."` to reference `den.schema`.

## Deletions

- `nix/nixModule/entities.nix` — `entityIncludes`/`entityProvides` options
- `modules/context/host.nix` — entityIncludes write (delete file)
- `modules/context/user.nix` — same
- `rootIncludes` from `structuralKeysSet` in `aspect.nix`
- `rootIncludes` phase from `resolveChildren` in `aspect.nix`
- Tombstone null-check path in `resolveTransition` (dead code)
- `emitCrossProvider` in `transition.nix` (dead code — `provides` always `{}`)

## Files Affected

| File | Changes |
|------|---------|
| `nix/lib/policy-types.nix` | `aspects` type: `listOf str` → `listOf providerType` |
| `nix/lib/resolve-entity.nix` | Self-provide from ctx, default special case, no entityIncludes |
| `nix/lib/aspects/fx/aspect.nix` | Remove rootIncludes phase and structural key |
| `nix/lib/aspects/fx/pipeline.nix` | Remove existence check from resolveEntityHandler |
| `nix/lib/aspects/fx/handlers/transition.nix` | Direct ref injection, path-based ctx-seen, remove tombstone/crossProvider dead code |
| `nix/lib/aspects/fx/handlers/ctx.nix` | ctx-seen stores path identities + direct ref values |
| `nix/lib/home-env.nix` | Direct refs in policy aspects |
| `modules/options.nix` | Schema-based entity guard, remove entityIncludes refs |
| `modules/context/host.nix` | Delete |
| `modules/context/user.nix` | Delete |
| `modules/context/has-aspect.nix` | Update error message |
| `modules/compat/ctx-shim.nix` | Update — forward `den.ctx` to `den.aspects` instead of entityIncludes |
| `modules/aspects/provides/os-class.nix` | Move to policy.aspects |
| `modules/aspects/provides/os-user.nix` | Move to policy.aspects |
| `modules/aspects/provides/home-manager.nix` | Direct refs, remove entityIncludes |
| `modules/aspects/provides/hjem.nix` | Same |
| `modules/aspects/provides/maid.nix` | Same |
| `modules/aspects/provides/wsl.nix` | Move to policy.aspects |
| `modules/aspects/defaults.nix` | Remove entityIncludes write |
| `modules/outputs/flakeSystemOutputs.nix` | Remove entityIncludes, unconditional entity resolution |
| `modules/outputs/osConfigurations.nix` | Remove entityIncludes |
| `modules/outputs/hmConfigurations.nix` | Remove entityIncludes |
| `modules/policies/flake.nix` | Direct refs in policy.aspects |
| `nix/nixModule/entities.nix` | Delete |
| `nix/nixModule/default.nix` | Remove entities.nix import |
| `templates/ci/**` | Update entityIncludes→policy.aspects, string→ref |
| `templates/default/**` | Migrate entityIncludes |
| `templates/noflake/**` | Migrate entityIncludes |
| `templates/nvf-standalone/**` | Migrate entityIncludes |
| `templates/flake-parts-modules/**` | Migrate entityIncludes |
| `templates/microvm/**` | Migrate entityIncludes |

## Test Impact

- ~45 CI test files that write `den.entityIncludes` need updating
- 4 `parametric-fixedTo` host-context tests revert to pre-rootIncludes expectations
- Tests using `policy.aspects = ["name"]` switch to direct refs
- `ctx-compat` tests updated (ctx-shim forwards to `den.aspects` now)
- Template test suites that use `entityIncludes` for flake content need migration to policies or direct aspect includes
