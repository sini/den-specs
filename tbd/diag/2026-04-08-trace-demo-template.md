# Trace Demo Template

**Date:** 2026-04-08
**Status:** Ready for implementation
**Depends on:** All SF1-SF5 merged (adapter accumulation, identity preservation, provider provenance, context adapters, forward consumption)

## Goal

Recreate the `templates/transform-trace-demo` from the `feat/aspect-rewrites` POC using the current adapter-based architecture. Demonstrates excludes, substitutions, and Mermaid trace visualization — all expressed as adapter composition rather than the POC's bespoke `resolve'`/`transforms` system.

## What the POC had

The POC's demo template (`feat/aspect-rewrites:templates/transform-trace-demo/`) included:

- **6 hosts**: laptop, desktop-gdm, web-server, mail-relay, devbox, angle-brackets-demo
- **3 roles**: workstation, server, relay (composing feature aspects)
- **7 features**: networking, tailscale, desktop, regreet/gdm/sddm (greeters), monitoring (with providers: node-exporter, nginx-exporter, alerting), mail, metrics-forward
- **2 users**: alice, deploy
- **Entity-level excludes**: `den.hosts.*.excludes = [ <monitoring> ]`
- **Aspect-level excludes**: `den.aspects.devbox.excludes = [ tailscale ]`
- **Substitution transforms**: `den.aspects.desktop-gdm.transforms = [ (substitute regreet gdm) ]`
- **Mermaid trace rendering**: A `trace.nix` module that built traces via `resolve'` with `trace = true`, rendered to Mermaid `graph TD` diagrams with styling for excluded/replaced nodes
- **`nix run .#write-files`**: Generated `traces/*.md` and `README.md`

## How to express this with adapters

The POC used `host.excludes`, `aspect.excludes`, `aspect.transforms`, and `resolve'` with options. Our architecture uses `meta.adapter` and composable adapters. The mapping:

| POC mechanism | Adapter equivalent |
|---|---|
| `host.excludes = [ <monitoring> ]` | `den.ctx.host.meta.adapter = inherited: adapters.filter (a: a.name != "monitoring") inherited;` — or per-host via aspect |
| `aspect.excludes = [ tailscale ]` | `den.aspects.devbox.meta.adapter = inherited: adapters.filter (a: a.name != "tailscale") inherited;` |
| `aspect.transforms = [ (substitute regreet gdm) ]` | `den.aspects.desktop-gdm.meta.adapter = inherited: adapters.mapAspect (a: if a.name == "regreet" then den.aspects.gdm else a) inherited;` |
| `resolve' class { trace = true; } aspect` | `resolve.withAdapter traceAdapter class aspect` with a custom trace adapter |

### Trace adapter design

The trace adapter replaces the POC's built-in `trace = true` option. It wraps any inner adapter to collect trace entries:

```nix
traceAdapter = inner: { aspect, recurse, class, ... } @ args:
  let
    result = inner args;
    entry = {
      name = aspect.name or "<anon>";
      provider = aspect.meta.provider or [];
      inherit class;
      depth = builtins.length (args.aspect-chain or []);
      chain = map (a: a.name or "<anon>") (args.aspect-chain or []);
    };
    childTraces = lib.concatMap (i: (recurse i).trace or []) (aspect.includes or []);
  in
  result // { trace = [ entry ] ++ childTraces; };
```

The Mermaid renderer reads `trace` entries and produces the graph. Excluded nodes (those filtered by `adapters.filter`) don't appear in the trace at all since `filterIncludes` removes them before the trace adapter sees them. To show excluded nodes with dashed styling (like the POC did), the trace adapter would need to run BEFORE `filterIncludes` and check `meta.adapter` manually — or we accept that excluded nodes simply don't appear (cleaner, but different from POC).

**Decision needed**: Show excluded nodes as dashed (requires custom adapter composition) or omit them (simpler, aligns with how filterIncludes works)?

### Exclude visualization approach

To show excluded nodes with dashed lines (like the POC), the trace adapter can probe includes before filtering:

```nix
traceWithExcludes = inner: args @ { aspect, resolveChild, ... }:
  let
    metaAdapter = aspect.meta.adapter or null;
    allIncludes = aspect.includes or [];
    # Probe each include to determine if it would be filtered
    probeResult = i:
      let child = resolveChild i;
      in { inherit child; excluded = metaAdapter != null && /* check if filtered */; };
  in
  ...;
```

This is more complex but matches the POC's visual output. Alternatively, we can run two resolve passes: one with `filterIncludes` (for the actual module) and one with a pure `traceName`-style adapter (for the full tree including would-be-excluded nodes), then diff them.

## Template structure

```
templates/trace-demo/
  flake.nix                    # inputs: den, nixpkgs
  flake.lock
  modules/
    den.nix                    # den.schema.user.classes = ["user"]
    flake-parts.nix            # import-tree, perSystem for write-files
    trace.nix                  # Mermaid trace adapter and rendering
    aspects/
      defaults.nix             # den.default with stateVersion, hostname, define-user stubs
      hosts/
        laptop.nix             # workstation role, user alice
        desktop-gdm.nix        # workstation + substitute regreet → gdm via meta.adapter
        web-server.nix          # server role + extra monitoring providers
        mail-relay.nix          # relay role + exclude monitoring via meta.adapter
        devbox.nix              # workstation + server, exclude tailscale via meta.adapter
      roles/
        workstation.nix         # includes: networking, tailscale, desktop
        server.nix              # includes: networking, monitoring, monitoring._.node-exporter, tailscale
        relay.nix               # includes: server, mail
      features/
        networking.nix
        tailscale.nix
        desktop.nix             # includes: regreet
        greeters.nix            # regreet, gdm, sddm aspects
        monitoring.nix          # with providers: node-exporter, nginx-exporter, alerting
        mail.nix                # parametric: uses host.hostName
      users/
        alice.nix               # includes primary-user battery
        deploy.nix              # minimal deploy user
  traces/                       # Generated by nix run .#write-files
    trace-laptop.md
    trace-desktop-gdm.md
    trace-web-server.md
    trace-mail-relay.md
    trace-devbox.md
  README.md                     # Generated overview with embedded traces
```

## Key differences from POC

1. **No `excludes` option on hosts or aspects** — use `meta.adapter` instead
2. **No `transforms` option** — use `meta.adapter` with `mapAspect`
3. **No `resolve'`** — use `resolve.withAdapter` with composed trace adapter
4. **No `transforms.nix`** — all transform logic is expressed as adapter composition
5. **`meta.provider`** instead of `__provider` — provider provenance is in meta
6. **`entity.resolved`** — hosts expose `.resolved` for cross-context patterns

## Adapter patterns demonstrated

The demo should showcase these adapter patterns for users:

### Exclude by name
```nix
den.aspects.mail-relay.meta.adapter = inherited:
  den.lib.aspects.adapters.filter (a: a.name != "monitoring") inherited;
```

### Exclude by provider
```nix
# Exclude all aspects provided by "monitoring" (node-exporter, nginx-exporter, alerting)
den.aspects.devbox.meta.adapter = inherited:
  den.lib.aspects.adapters.filter
    (a: !(lib.hasPrefix ["monitoring"] (a.meta.provider or [])))
    inherited;
```

### Substitute
```nix
den.aspects.desktop-gdm.meta.adapter = inherited:
  den.lib.aspects.adapters.mapAspect
    (a: if a.name == "regreet" then den.aspects.gdm else a)
    inherited;
```

### Context-level exclude
```nix
den.ctx.host.meta.adapter = inherited:
  den.lib.aspects.adapters.filter (a: a.name != "debug-tools") inherited;
```

### Trace via custom adapter
```nix
traceAdapter = { aspect, recurse, class, ... }:
{
  trace = [ { name = aspect.name; provider = aspect.meta.provider or []; } ]
    ++ lib.concatMap (i: (recurse i).trace or []) (aspect.includes or []);
};

result = den.lib.aspects.resolve.withAdapter traceAdapter "nixos" hostAspect;
# result.trace is a flat list of entries for Mermaid rendering
```

## Implementation notes

- The demo flake needs `den` and `nixpkgs` inputs. Use `den.url = "path:../..";` to reference the local checkout.
- `import-tree` for auto-importing modules from the directory tree.
- The Mermaid renderer (trace.nix) should be ~150 lines — significantly simpler than the POC's 309-line version since we don't need the exclude/substitute tracking logic (adapters handle that).
- `nix run .#write-files` writes traces and README using `vic/checkmate`'s `files` integration or a simple `writeShellScriptBin`.
- Tests: the CI template should have at least one test verifying the trace adapter produces expected output for a known aspect tree.

## Scope

- ~15 new files (template modules)
- ~1 new file (trace adapter, possibly in adapters.nix or template-local)
- Generated trace outputs
- No changes to core library — everything expressed as adapter composition
