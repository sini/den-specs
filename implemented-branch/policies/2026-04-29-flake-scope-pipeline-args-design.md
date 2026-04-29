# Flake-Scope Args in Aspect Pipeline Functions

**Date:** 2026-04-29
**Branch:** `feat/fx-pipeline`
**Status:** Implemented

Shipped on `feat/fx-pipeline` (commits abb1bfdd..c19e8ca6). `pipelineOnly` collision policy utility and `den.provides.flake-scope` battery both landed with edge case tests.

## Problem

Aspect functions at the pipeline level can only destructure entity context args (`host`, `user`, `home`) and enrichment args. Flake-scope values like `lib`, `inputs`, and `den` are available as module-level closure args but cannot be destructured in the aspect function signature:

```nix
# Works — lib from module closure
{ lib, den, ... }:
{
  den.aspects.foo = { host, ... }: {
    nixos = lib.mkIf (host.class == "darwin") { ... };
  };
}

# Does NOT work — lib is not a pipeline handler
den.aspects.foo = { host, lib, ... }: {
  nixos = lib.mkIf (host.class == "darwin") { ... };
};
```

The second form defers forever because no handler provides `lib`. This is a usability gap — users expect `lib` to be available in the same way `host` is.

## Design

Two additions. No pipeline changes required — the enrichment mechanism from the policy context enrichment spec handles everything.

### 1. Utility: `den.lib.policy.pipelineOnly`

**File:** `nix/lib/policy-effects.nix`

```nix
pipelineOnly = value:
  if builtins.isAttrs value then
    value // { collisionPolicy = "class-wins"; }
  else
    { __functor = _: value; collisionPolicy = "class-wins"; };
```

Tags a value with `collisionPolicy = "class-wins"`. When this value reaches a class module that also receives the same arg from the module system (e.g., NixOS provides `lib` via `_module.args`), the class module's native value wins silently — no collision error.

**Attrset values** use `//` to merge the `collisionPolicy` key directly. This preserves all original attributes — `(pipelineOnly lib).trace` works identically to `lib.trace`.

**Non-attrset values** (functions, strings, etc.) are wrapped with `__functor` so the value remains callable while carrying the collision policy. The aspect function receives the wrapper, which is callable via Nix's functor protocol (`wrapper arg` invokes `__functor _ arg`).

The existing per-arg collision policy check in `wrapClassModule` (`resolveCollisionPolicy`, line 48-53 of `aspect.nix`) already supports this: it checks `ctx.${name}.collisionPolicy` for attrset values. Both forms produce an attrset with `collisionPolicy`. No pipeline changes needed.

**Trade-off:** For attrset values, the `collisionPolicy` key is visible inside aspect functions (`lib ? collisionPolicy` is `true`). This is unlikely to cause issues in practice — `collisionPolicy` is a reserved name for this purpose. For `inputs`, a flake input literally named `collisionPolicy` would be shadowed — an edge case with negligible probability.

**Why not `__collisionPolicy`?** Double-underscore would follow Nix's metadata convention and avoid polluting the namespace. However, `resolveCollisionPolicy` in `aspect.nix` (line 44) already checks `ctx.${name} ? collisionPolicy` — adopting `__collisionPolicy` would require changing both the utility and the collision resolver. Since `collisionPolicy` is already the established convention used by the existing per-arg collision mechanism, we preserve consistency. If namespace pollution proves problematic, a future rename to `__collisionPolicy` across all collision-aware code is straightforward.

**`__functor` wrapper and collision check:** The `resolveCollisionPolicy` check (`builtins.isAttrs (ctx.${name} or null) && (ctx.${name} ? collisionPolicy)`) works for any attrset with a `collisionPolicy` key. The `__functor` wrapper produces `{ __functor = ...; collisionPolicy = "class-wins"; }` — an attrset, so the check passes. No special-casing needed.

### 2. Battery: `den.policies.den-flake-scope`

**File:** `modules/aspects/provides/flake-scope.nix`

```nix
{ den, lib, inputs, ... }:
let
  inherit (den.lib.policy) resolve pipelineOnly;
in
{
  den.provides.flake-scope = {
    description = "Expose lib, inputs, and den to aspect pipeline functions.";
    policies.den-flake-scope = _: [
      (resolve {
        lib = pipelineOnly lib;
        inputs = pipelineOnly inputs;
        den = pipelineOnly den;
      })
    ];
  };
}
```

Follows the same pattern as other batteries (`den.provides.define-user`, `den.provides.hostname`, etc.). Users opt in via `den.default.includes`:

```nix
den.default.includes = [
  den.provides.flake-scope
];
```

This makes `lib`, `inputs`, and `den` available to all aspect pipeline functions across all entity kinds.

**Values provided:**

| Arg | Value | Why |
|-----|-------|-----|
| `lib` | nixpkgs `lib` | `lib.mkIf`, `lib.optional`, general Nix utilities |
| `inputs` | flake `inputs` | Access to flake inputs in aspect logic |
| `den` | `config.den` fixed-point | Den API access without closure scoping |

**Why not `config`?** Flake-parts `config` is the raw module system config — too low-level for aspect logic and risks circular evaluation.

**`den` circular evaluation caveat:** The `den` value is the `config.den` fixed-point. Accessing `den.lib`, `den.schema`, `den.classes`, `den.traits`, and `den.config` from inside aspect functions is safe. Accessing `den.aspects`, `den.policies`, or `den.default` would trigger circular evaluation — the pipeline is resolving aspects which are defined under those paths. This is the same constraint that exists for closure-scoped `den` access today.

**Circular access mitigation:** No runtime guard is added. The failure mode is Nix's standard infinite recursion error, which is already the behavior for closure-scoped `den.aspects` access. Adding a `builtins.throw` guard would require wrapping the `den` attrset in a proxy that intercepts attribute access — Nix has no such mechanism without `__functor`-based indirection, which would break `den.lib.foo` syntax. The risk is documented, and the safe/unsafe paths are identical to the closure-scoped case users already navigate.

## How it works end-to-end

1. Pipeline resolves entity root (e.g., `host`)
2. Transition handler dispatches `den-flake-scope` policy
3. `classifyResolve` classifies `lib`/`inputs`/`den` as enrichment (non-schema keys)
4. Enrichment installs constant handlers via `scope.provide`
5. `drain-deferred` resolves deferred aspects requesting these args (e.g., `{ host, lib, ... }:`)
6. At class module wrapping (post-pipeline), `wrapClassModule` sees `lib` in ctx
7. NixOS also provides `lib` via `_module.args` — collision check finds `ctx.lib.collisionPolicy == "class-wins"`
8. Class module receives NixOS's `lib` (class wins), no error

## User extensibility

Users add their own flake-scope args using the same pattern:

```nix
{ den, myCustomLib, ... }:
let
  inherit (den.lib.policy) resolve pipelineOnly;
in
{
  den.policies.my-custom-scope = _: [
    (resolve { myCustomLib = pipelineOnly myCustomLib; })
  ];
}
```

## Files changed

| File | Change |
|---|---|
| `nix/lib/policy-effects.nix` | Add `pipelineOnly` utility |
| `modules/aspects/provides/flake-scope.nix` | New battery: enrichment policy for `lib`, `inputs`, `den` |

## Test cases

- **test-aspect-receives-lib** — `den.aspects.foo = { host, lib, ... }: { nixos = lib.mkIf true { }; }` evaluates without error
- **test-aspect-receives-inputs** — `den.aspects.foo = { inputs, ... }: { nixos.networking.hostName = inputs.self.rev or "dirty"; }` evaluates
- **test-aspect-receives-den** — `den.aspects.foo = { den, ... }: { nixos = den.lib.something; }` evaluates
- **test-class-module-lib-collision-silent** — Class module `{ lib, config, ... }: { ... }` receives NixOS lib, no collision error
- **test-pipeline-only-preserves-attrs** — `(pipelineOnly lib).mkIf true { }` works identically to `lib.mkIf true { }`
- **test-optional-lib-arg** — `{ lib ? null, ... }: ...` with optional `lib` resolves. Pipeline deferral only blocks on required args (`builtins.functionArgs` returns `false`). Optional args (`true`) are resolved if a handler exists but don't block — so `lib ? null` receives the enrichment value when available, falls back to `null` otherwise
- **test-mixed-collision-policies** — Aspect uses `{ lib, myArg, ... }:` where `myArg` has `collisionPolicy = "den-wins"`. `lib` gets class-wins, `myArg` gets den-wins — independent per-arg policies
- **test-forward-sub-pipeline-receives-enrichment** — Forward sub-pipelines receive parent `aspectPolicies` via `extraState` in `runSubPipeline`, so policies (including `den-flake-scope`) are re-dispatched. This is independent of context inheritance — policy re-dispatch is structural, not dependent on sub-pipeline scoping fixes
- **test-enrichment-stripping-at-class-boundary** — `lib`/`inputs`/`den` enrichment keys are correctly handled by post-pipeline `wrapCollectedClasses` — class modules receive native module-system values
- **test-pipeline-only-non-attrset** — `pipelineOnly someFunction` produces a `__functor` wrapper that is callable (`(pipelineOnly (x: x + 1)) 5 == 6`) and carries `collisionPolicy = "class-wins"`. Verifies the non-attrset path of `pipelineOnly` works for collision resolution
