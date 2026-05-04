# Provider-Aware Excludes and Trace

**Date:** 2026-04-04
**Status:** Implemented (superseded by deep-provider-cascade spec for
`__provider` format and substitute behavior)
**Branch:** `feat/aspect-rewrites`

## Problem

Excluding an aspect did not cascade to its providers. If `server`
includes both `monitoring` and `monitoring._.node-exporter` as siblings,
excluding `monitoring` left `node-exporter` alive because the exclude
transform only matched on `provided.name`.

## Solution

### Provider Origin Tagging

`wrapProvider` in `types.nix` tags provider values with `__provider`
(a list path). The `provides` option's `apply` wraps each provider:

```nix
apply = lib.mapAttrs (_: wrapProvider parentPath);
```

Where `parentPath` is composed from `config.__provider ++ [name]`.

The tag propagates through:
- `withIdentity` in `parametric.nix`
- `carryAttrs` in `take.nix`
- Functor re-wrapping for curried providers

`__provider` is declared as an option in `aspectSubmodule` (type
`listOf str`, default `cnf.providerPrefix or []`) to avoid freeform
type conflicts.

### Provider-Aware Exclude

The `exclude` transform uses list prefix matching via `isPrefix`:

```nix
isPrefix = prefix: path:
  prefix != [] && lib.take (builtins.length prefix) path == prefix;
```

Excluding `["monitoring"]` matches `["monitoring"]` (exact) and
`["monitoring" "node-exporter"]` (prefix), but NOT
`["eg" "monitoring"]` (different prefix).

### Angle Bracket Excludes

The trace refactor eliminated the circular eval that previously blocked
angle brackets in excludes. `den-brackets.nix` already rewrites `/`
to `.provides.`, so `<monitoring/node-exporter>` resolves to the
provider aspect ref with `__provider` already set.

### Trace Visualization

Trace entries include `provider` (list path) when `provided.__provider`
is non-empty. The Mermaid renderer shows qualified labels:
`monitoring/node-exporter` for provider nodes.

## See Also

- `2026-04-04-deep-provider-cascade.md` — supersedes the `__provider`
  format (changed from `nullOr str` to `listOf str`) and adds
  provider-aware substitute, deep cascade, and namespace qualification.
Here's the analysis:

  Why extractClassModulesDeep is a unique path:

  The compat shim policy functions are pure functions ({ host, user, ... } -> [effect]) that return a list of policy effects. They run inside installPolicies at entity resolution time. They
  can't use the fx monad directly.

  Can we use policy.include instead?

  Yes — and this is the cleaner approach. Instead of manually walking includes and extracting class modules as policy.provide effects, the compat shim could return policy.include effects for
  each item in the includes list. The pipeline would then process them through policyEmitIncludes → emit-include → emitIncludes — the full walk with dedup, parametric resolution, class
  extraction, etc.

  Let me check if this approach would work for the to-users case. The concern: policy.include injects into the current resolution context. For to-users, we're inside a user entity resolution,
  so the includes would be processed at user scope — which is exactly where we want homeManager modules to land.

  Let me try the refactor:
