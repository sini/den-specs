# User and Home Context Coverage

**Date:** 2026-04-12
**Branch:** `feat/new-trace-demo`

## Goal

Extend the diag library and demo template to cover user-rooted and
home-rooted graph generation alongside the existing host-rooted views.
Currently only `den.ctx.host` is exercised by the diag pipeline;
`den.ctx.user` and `den.ctx.home` are untouched.

## Motivation

The diag lib's graph IR, filters, and renderers are already
entity-agnostic (`rootName`/`rootId`, generic `diag.context`). But only
host contexts are wired end-to-end. Users wanting to visualize a single
user's aspect tree, or a standalone home's resolution, have no lib
support or template examples.

## Changes

### 1. Lib: `diag.userContext` and `diag.homeContext`

Add to `nix/lib/diag/default.nix`, alongside the existing `hostContext`.
Export both at the top level of `den.lib.diag`.

```nix
userContext = { host, user, classes ? null, direction ? "LR" }:
  let
    actualClasses = if classes != null then classes
      else lib.unique ([ "homeManager" "user" ] ++ (user.classes or [ "homeManager" ]));
    root = den.ctx.user { inherit host user; };
  in
  context { inherit root direction; name = user.name; classes = actualClasses; };

homeContext = { home, classes ? null, direction ? "LR" }:
  let
    actualClasses = if classes != null then classes
      else lib.unique ([ "homeManager" ] ++ (home.classes or [ "homeManager" ]));
    root = den.ctx.home { inherit home; };
  in
  context { inherit root direction; name = home.name; classes = actualClasses; };
```

Note: `userContext` includes the `"user"` class in its default set
because user-rooted traces still traverse user-stage entries. Without
it, `captureWithPaths` would miss user-class contributions.

### 2. Lib: Decompose `diag.views` into `core` + kind-specific

Refactor `nix/lib/diag/views.nix` so all entity kinds compose from a
shared `core` foundation. Export `core`, `host`, `user`, `home`, and
`fleet` as functions from `views.nix`.

```
diag.views.core  — entity-agnostic views that work on any graph:
                   aspects, simple, providers, adapters, decisions,
                   parametric, declared, orphans, pipeline, ir,
                   ctx, seq, seq-full, state

diag.views.host  — core ++ host-specific:
                   has-aspect-nixos, has-aspect-hm, class-nixos,
                   class-hm, cross-class, diff-classes, sankey, treemap,
                   mindmap, fan, c4container, c4component,
                   c4container-mmd, c4component-mmd

diag.views.user  — core ++ user-specific:
                   has-aspect-hm, class-hm

diag.views.home  — core ++ home-specific:
                   has-aspect-hm, class-hm

diag.views.fleet — fleet-level views (unchanged):
                   namespace, c4context, c4context-mmd, sankey,
                   treemap, provider-matrix
```

Context pipeline views (`ctx`, `seq`, `seq-full`, `state`) are in
`core` because all entity kinds have resolution pipelines (host
pipelines are richer; user/home pipelines are shorter but still
meaningful).

Host-only views (`has-aspect-nixos`, `class-nixos`, `cross-class`,
`diff-classes`) are excluded from user/home sets because those entities
typically don't carry the nixos class. Views that receive an entity
without a matching class degrade gracefully to empty diagrams rather
than erroring.

Templates compose freely:
- `rc.views.host` for the full host set
- `rc.views.core ++ [ myView ]` for a custom minimal set
- `builtins.filter (v: v.view != "pipeline") rc.views.user` to drop one

### 3. Render Context: Pre-Bound View Sets

Update `diag.renderContext` to carry all five view sets:

```nix
rc.views = {
  core  = views.core rc;
  host  = views.host rc;
  user  = views.user rc;
  home  = views.home rc;
  fleet = views.fleet rc;
};
```

### 4. Template: New Demo Aspects

In `templates/diag-demo/modules/aspects/`:

**`den.nix`** — add standalone home registrations:
```nix
den.homes.x86_64-linux = {
  alice = {};              # unbound standalone home
  "alice@laptop" = {};     # host-bound home
};
```

**`users/alice.nix`** — extend with a perHome aspect:
```nix
den.aspects.alice = {
  includes = [
    den._.primary-user
    den.aspects.demo-shell
    den.aspects.hyprland
    den.aspects.dev-tools
    (den.lib.perHome den.aspects.alice-dotfiles)
  ];
  nixos = { ... }: { users.users.alice.isNormalUser = true; };
  homeManager = { pkgs, ... }: { home.packages = [ pkgs.git ]; };
  provides.to-hosts.nixos.users.users.alice.openssh.authorizedKeys.keys = [
    "ssh-ed25519 AAAA... alice@example"
  ];
};

den.aspects.alice-dotfiles = {
  homeManager = { ... }: {
    programs.starship.enable = true;
  };
};
```

The `perHome` wrapper ensures `alice-dotfiles` only fires in home
contexts. In host-rooted views the wrapper is a no-op; in home-rooted
views it materializes.

### 5. Template: Entity Iteration in `diagrams.nix`

Extend the entry generation to iterate users and homes alongside
hosts, using the same `diag.export.entityEntries` helper for all
three:

```nix
# View sets from the render context
hostViewDefs = rc.views.host;
userViewDefs = rc.views.user;
homeViewDefs = rc.views.home;

# Existing: host entries
hostEntries = lib.concatMap (host:
  entityEntries { inherit pkgs rc diag; } {
    entity = host;
    name = host.name;
    viewDefs = hostViewDefs;
    galleryDrv = hostGalleryDrv host.name;
  }
) allHosts;

# New: user entries (one per host x user pair)
allUsers = lib.concatMap (host:
  lib.mapAttrsToList (userName: user: {
    inherit host user;
    name = "${host.name}-${userName}";
  }) (host.users or {})
) allHosts;

userEntries = lib.concatMap (u:
  entityEntries { inherit pkgs rc diag; } {
    entity = diag.userContext { inherit (u) host user; };
    name = u.name;
    viewDefs = userViewDefs;
  }
) allUsers;

# New: standalone home entries (mirrors host iteration pattern)
allHomes = lib.concatMap builtins.attrValues
  (builtins.attrValues (den.homes or {}));

homeEntries = lib.concatMap (home:
  entityEntries { inherit pkgs rc diag; } {
    entity = diag.homeContext { inherit home; };
    name = home.name;
    viewDefs = homeViewDefs;
  }
) allHomes;

everyEntry = hostEntries ++ userEntries ++ homeEntries ++ fleetEntries;
```

The home iteration uses `lib.concatMap builtins.attrValues
(builtins.attrValues ...)` — the same pattern as the existing host
iteration on line 21 of `diagrams.nix`.

### 6. Export: Remove host-assumption from `entityEntries` fallback

`diag.export.entityEntries` currently falls back to `diag.hostContext`
when the entity doesn't look like a pre-computed graph. This couples
the fallback to hosts. Change it to require a pre-computed graph or
throw:

```nix
g = if entity ? nodes then entity
    else throw "entityEntries: entity must be a pre-computed graph (from hostContext, userContext, homeContext, or context).";
```

Callers always pass pre-computed contexts; the host fallback was a
convenience that introduced coupling.

### 7. Gallery and README Updates

- Host galleries: unchanged (existing `galleryDrv`)
- User galleries: omit for now (user views are accessible via
  `nix build .#<host>-<user>-<view>` individually)
- Home galleries: omit for now (same — accessible individually)
- README: add a "User Views" and "Home Views" section listing the
  naming convention and example `nix build` commands

## What This Exercises

| Pattern | Where |
|---------|-------|
| `den.ctx.user` as graph root | user-rooted views |
| `den.ctx.home` as graph root | home-rooted views |
| `den.homes.*` standalone home registry | home registrations in den.nix |
| Host-bound homes (`alice@laptop`) | home registration + host reference |
| `perHome` helper | alice's dotfiles aspect |
| User-level provider chains (`provides.to-hosts`) | existing alice SSH keys |
| homeManager class at home level | standalone home views |
| View set composition (`core` + kind-specific) | all entity kinds |

## Out of Scope

- Cross-host forwarding (future work)
- Mutual providers at user level (future work)
- New filter for "what does this entity provide TO its parent" (future work)
- `perUser`/`perHost` in user-specific aspects (already implicitly exercised)
- Fleet-level incorporation of home entities (future work)
- User/home gallery pages (accessible via individual nix build commands)

## Verification

- All existing host views still build (no regression)
- User-rooted views build for every (host, user) pair in the demo
- Home-rooted views build for every standalone home
- CI tests still pass (aspect-path, has-aspect, one-of-aspects, collect-paths)
- `nix run .#write-diagrams` produces the expanded file set
