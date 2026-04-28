# Battery: `host-aspects`

**Date:** 2026-04-17
**Status:** Approved

## Purpose

Project all homeManager-class configs from a host's aspect tree onto a user. Opt-in per user via `den._.host-aspects` in their includes.

## Usage

```nix
den.aspects.tux.includes = [ den._.host-aspects ];
```

Host aspects that define `homeManager` keys will have those configs forwarded to the user's homeManager evaluation. `nixos`/`darwin` keys are ignored (pipeline resolves with `class = "homeManager"`). Dedup handled by identity.

## Implementation

Single file: `modules/aspects/provides/host-aspects.nix`

```nix
{ den, ... }:
let
  inherit (den.lib) parametric;

  from-host =
    { host, user }:
    parametric.fixedTo { inherit host user; } host.aspect;
in
{
  den.provides.host-aspects = parametric.exactly {
    includes = [ from-host ];
  };
}
```

### How it works

1. `parametric.exactly` makes this a parametric aspect taking `{ host, user }` context
2. `from-host` wraps `host.aspect` with `fixedTo { host, user }` so parametric children get context
3. Pipeline resolves with target class `homeManager` — only `homeManager` keys collected
4. No standalone home support needed (no host to inherit from)

## Testing

Test file: `templates/ci/modules/features/host-aspects.nix`

Tests:
- Host aspect with `homeManager` key projects to user who includes `den._.host-aspects`
- Host aspect with only `nixos` key does NOT leak into user's homeManager
- Multiple host aspects with homeManager keys all project correctly
- User who does NOT include `den._.host-aspects` does not receive host homeManager configs

## Non-goals

- Per-user filtering at the host level (users opt in by including the battery)
- Standalone home support
- Projection of non-homeManager classes
