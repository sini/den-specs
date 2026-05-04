# Entity-Level Class Evaluation — Eliminating Forwards

## Status: Draft

## Problem

The forward mechanism exists to wire entity class content into flake outputs. But entity schemas already contain all the information needed: `class`, `instantiate`, `intoAttr`, `mainModule`. The forward sub-pipeline redundantly re-resolves content that `mainModule` already produces.

Three output forward modules (`osConfigurations.nix`, `hmConfigurations.nix`, `flakeSystemOutputs.nix`) exist solely to:
1. Re-resolve an entity's aspect tree (redundant — `mainModule` does this)
2. Call `instantiate` on the result
3. Place the evaluated config at the entity's `intoAttr` in the flake output

This is plumbing, not logic. The entity schema already declares how to evaluate (`instantiate`) and where to place (`intoAttr`). The forward mechanism is an indirection layer between "entity has class content" and "flake has output."

## Design

### Core Change

Entity output wiring becomes a direct consequence of entity schema metadata, not a forward spec. The flake policy chain (`flake → flake-system → flake-os/flake-hm`) emits policy effects that reference entity metadata. Post-pipeline, `fxResolve` reads scope partitions and produces flake outputs using entity `instantiate` + `intoAttr`.

### What `mainModule` already provides

`host.mainModule` (host.nix:108) = `den.lib.aspects.resolve config.class config.resolved`

This runs `fxResolveTree` which:
1. Walks the host's aspect tree
2. Collects all nixos class modules
3. Returns `{ imports = [...all modules...] }`

The forward in `osConfigurations.nix` calls `host.instantiate { modules = [module] }` where `module` is the same content `mainModule` produces. The forward's sub-pipeline is a redundant re-resolution.

### Prerequisite: Schema-scoped policies

The current flake policies use `__entityKind` guards to restrict which scope they fire in. This pattern is ugly and propagates. Fix: allow policies to be registered on schema entries via `den.schema.<kind>.policies`. These fire only in scopes matching their schema kind — entity kind discrimination is structural, not a runtime guard.

**Implementation:** `schemaEntryType` (modules/options.nix:32) already extracts `includes` from defs and stores them alongside the merged module. Add `policies` with the same pattern — extract from defs, strip before merge, store in the result. The transition handler's `mkDispatch` reads `den.schema.${sourceEntityKind}.policies or {}` and dispatches them alongside global policies.

```nix
# Current — ugly __entityKind guard:
den.policies.flake-system-to-flake-os = { __entityKind ? null, system, ... }:
  if __entityKind != "flake-system" then [ ]
  else ...;

# New — schema-scoped, clean signature:
den.schema.flake-system.policies.to-os-outputs = { system, ... }:
  map (host: policy.instantiate host)
    (builtins.attrValues (den.hosts.${system} or {}));
```

### Replacement: `policy.instantiate`

New policy effect that says "evaluate this entity's class content and place the result in the flake output":

```nix
policy.instantiate = entity: {
  __policyEffect = "instantiate";
  value = entity;
};
```

The entity carries: `class`, `instantiate`, `intoAttr`, `mainModule`. The pipeline doesn't need to re-resolve anything — it reads `mainModule` (which is lazily evaluated on demand).

**`flake-system` policies become (with schema-scoped registration):**

```nix
den.schema.flake.policies.to-systems = _:
  map (system: policy.resolve.to "flake-system" { inherit system; }) den.systems;

den.schema.flake-system.policies = {
  to-os-outputs = { system, ... }:
    map (host: policy.instantiate host)
      (builtins.attrValues (den.hosts.${system} or {}));

  to-hm-outputs = { system, ... }:
    map (home: policy.instantiate home)
      (builtins.attrValues (den.homes.${system} or {}));
};
```

No `__entityKind` guards, no `resolve.to "flake-os"`, no `include den.aspects.flake-os`. The policy just says "instantiate this entity."

### How `policy.instantiate` works

During policy dispatch, `instantiate` effects are collected (like route effects). Post-pipeline, for each entity:

```nix
# entity has: class, instantiate, intoAttr, mainModule
result = entity.instantiate { modules = [ entity.mainModule ]; };
flakeModule = lib.setAttrByPath entity.intoAttr result;
```

The `flakeModule` is injected into the `flake` class bucket. `fxResolve` returns it as part of `{ imports = [...] }`.

### What this eliminates

| Deleted | Lines |
|---------|-------|
| `modules/outputs/osConfigurations.nix` | 26 |
| `modules/outputs/hmConfigurations.nix` | 24 |
| `den.aspects.flake-os` registration | — |
| `den.aspects.flake-hm` registration | — |
| Forward sub-pipeline for OS/HM resolution | ~100 in pipeline.nix |
| `flake-os` and `flake-hm` schema kinds | — |

### `flakeSystemOutputs.nix` — the remaining forward

Per-system outputs (packages, apps, checks, devShells, legacyPackages) are different. They don't have entity schemas with `instantiate` — they're raw class content forwarded into the flake output with `adaptArgs` injecting `pkgs`.

These can become `policy.route`:

```nix
# Current: den.provides.forward { fromClass = output; intoClass = "flake"; intoPath = ["flake" output system]; adaptArgs = { pkgs = ... }; }
# New:
policy.route {
  fromClass = output;
  intoClass = "flake";
  path = [ "flake" output system ];
  adaptArgs = _: { pkgs = inputs.nixpkgs.legacyPackages.${system}; };
  guard = _: has-flake-output output;
}
```

This is already Tier 1 `policy.route` — no new mechanism needed.

**NestModule path:** `path = ["flake" output system]` targets the flake output namespace (e.g., `flake.packages.x86_64-linux`). This uses `wrapRouteModules`'s plain nesting path (no submodule type required) — `evalModules` with `freeformType = lazyAttrsOf unspecified` evaluates the source module, then `lib.setAttrByPath` places the config at the target path. The `adaptArgs` injection of `pkgs` flows through `evalModules`'s `specialArgs`. This matches the existing `wrapRouteModules` with `adaptArgs != null && path != []` branch (route.nix:37-58).

### Forward elimination path

With `policy.instantiate` handling entity outputs and `policy.route` handling per-system outputs:

1. `osConfigurations.nix` → deleted, replaced by `policy.instantiate host`
2. `hmConfigurations.nix` → deleted, replaced by `policy.instantiate home`
3. `flakeSystemOutputs.nix` → replaced by `policy.route` effects in existing flake policies
4. `den.aspects.flake-os`, `den.aspects.flake-hm`, `den.aspects.flake-<output>` → deleted

After these conversions, the only remaining forward callers are user-defined Tier 2 forwards (adapter modules, dynamic paths). If none remain:

5. `nix/lib/forward.nix` → deleted
6. `nix/lib/aspects/fx/handlers/forward.nix` → deleted (or reduced to just `buildForwardAspect` if adapters are needed)
7. `modules/aspects/provides/forward.nix` → deleted
8. `resolveForwardSource` in pipeline.nix → deleted
9. `applyForwardSpecs` in pipeline.nix → deleted

### User-defined Tier 2 forwards

From gwen's configs, the Tier 2 forwards that use `adapterModule`:
- `niri` with list-merging adapter
- `persist*` with dedup adapter

These need `adapterModule` which `policy.route` doesn't support. Options:
1. **`policy.route` with `adapter` field** — extend route to support adapter modules
2. **Keep simplified `den.provides.forward`** — auto-detected as Tier 2, handler reads scope partitions
3. **Express adapters as trait schemas** — the dedup/list-merge behavior is really a collection strategy

Option 3 is the cleanest: the adapter's `apply = lib.unique` is equivalent to a trait with `collection = "set"` strategy. The list-merging adapter is equivalent to `collection = "list"`. The adapter exists because NixOS module merging doesn't do what the user wants — but trait collection strategies already solve this.

If we add a `collection` field to class schemas (not just trait schemas):
```nix
den.classes.niri = {
  description = "Niri";
  collection = "merge";  # default NixOS module merge
};
den.classes.persist = {
  description = "Persist paths";
  collection = "list-unique";  # collect as list, dedup with lib.unique
};
```

Then `policy.route` can apply the class's collection strategy during wrapping. No adapter module needed.

This is a larger change but eliminates the ENTIRE Tier 2 concept. Every forward becomes a `policy.route` or `policy.instantiate`.

### Revised scope for this spec

The full forward elimination (including class collection strategies) is ambitious. The minimum viable change that eliminates sub-pipelines from the flake output path:

1. Add `policy.instantiate` effect + handler
2. Convert `osConfigurations.nix` and `hmConfigurations.nix` to `policy.instantiate`
3. Convert `flakeSystemOutputs.nix` to `policy.route`
4. Delete the three output forward modules
5. Delete `flake-os`, `flake-hm`, `flake-<output>` schema kinds and aspects

User-defined Tier 2 forwards stay via simplified `den.provides.forward` (auto-detect routes Tier 1 cases, keep sub-pipeline for adapters). Full adapter elimination via class collection strategies is a follow-up — no, not a follow-up. We push onward. But the class collection strategy change requires understanding how adapter modules actually work in `buildForwardAspect` and whether collection strategies can fully replace them.

### Adapter analysis

Looking at gwen's adapters:

**Niri adapter** (`niri [HU]/class.nix`):
```nix
adapterModule = { ... }: {
  options.settings.spawn-at-startup = lib.mkOption { type = listOf ...; };
  options.settings.window-rules = lib.mkOption { type = listOf ...; };
  options.settings.layer-rules = lib.mkOption { type = listOf ...; };
};
```
This declares OPTIONS so that multiple niri class modules can merge their list-type settings. Without the adapter, NixOS would error on duplicate definitions. The adapter creates the merge point.

**Persist adapter** (`persist [HU]/class/classes.nix`):
```nix
adapterModule = dedupModule;  # where dedupModule has apply = lib.unique
```
This wraps the option with `apply = lib.unique` to deduplicate list entries.

Both adapters solve the same problem: NixOS module system merge semantics don't match the desired collection behavior. The adapter adds/modifies option definitions to get the right merge.

A class collection strategy could handle this IF the route wrapping creates the option definition. For `path = ["programs" "niri"]`, the `programs.niri` option already exists (defined by the niri HM module). The adapter adds sub-options under it. This is more than just collection — it's option schema extension.

**Verdict:** Class collection strategies can handle the dedup case (persist) but not the option-extension case (niri). The niri adapter adds OPTIONS that don't otherwise exist. This is genuinely module-system-level customization that `policy.route` shouldn't try to absorb.

For this spec: keep simplified `den.provides.forward` for the option-extension case. The auto-detect mechanism (from `project_forward_autodetect_route.md`) routes simple forwards through `policy.route` and keeps adapter forwards as Tier 2.

### Changes per file

| File | Change |
|------|--------|
| `nix/lib/policy-effects.nix` | Add `instantiate` effect constructor |
| `nix/lib/aspects/fx/handlers/tree.nix` | Add `registerInstantiateHandler` |
| `nix/lib/aspects/fx/pipeline.nix` | Add `scopedInstantiates` state field. Post-pipeline instantiation in `fxResolve`. |
| `nix/lib/aspects/fx/handlers/transition.nix` | Classify `instantiate` effects during policy dispatch |
| `modules/policies/flake.nix` | Replace `resolve.to "flake-os"` + `include flake-os` with `policy.instantiate host`. Same for flake-hm. Convert flake-system-outputs to `policy.route`. |
| `modules/outputs/osConfigurations.nix` | Delete |
| `modules/outputs/hmConfigurations.nix` | Delete |
| `modules/outputs/flakeSystemOutputs.nix` | Delete (logic moves to flake.nix policies) |
| `modules/context/flake-schema.nix` | Remove `flake-os`, `flake-hm` schema kinds (if registered there) |

### Migration strategy

1. Add `policy.instantiate` effect + handler + post-pipeline instantiation
2. Convert flake policies to use `policy.instantiate` for hosts/homes
3. Convert flake policies to use `policy.route` for system outputs (packages, apps, etc.)
4. Delete `osConfigurations.nix`, `hmConfigurations.nix`, `flakeSystemOutputs.nix`
5. Delete orphaned schema kinds and aspects
6. Implement forward auto-detect (Tier 1 → route, Tier 2 stays)
7. Verify no remaining forward callers beyond user Tier 2
8. If no Tier 2 callers: delete forward infrastructure entirely
9. If Tier 2 callers remain: keep simplified forward with scope partition reads
