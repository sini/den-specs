# Traits, Structural Nesting, and Schema-Driven Classes

**Date:** 2026-04-25
**Status:** Draft
**Authors:** Vic, Sini (with agent assist)
**Supersedes:** 2026-04-16 Capabilities spec (draft)

## Problem

Den aspects currently treat all non-structural keys as class modules. There is no distinction between:
- A **class** key (`nixos`, `homeManager`) — a module passed to an evaluation domain
- A **semantic data** key (`firewall`, `persistence`) — structured data collected across aspects and consumed by other modules
- A **nested aspect** key (`tux`, `node-exporter`) — a sub-aspect scoped to a named entity or feature

All three are emitted as class modules today, meaning semantic data cannot be collected, composed, and routed back into consumers. The `.provides.` mechanism handles nested aspects but requires explicit nesting under a structural key rather than direct attrset nesting.

## Core Insight

Aspects emit two fundamentally different things:

- **Class modules** — functions evaluated *by* the NixOS/Darwin/HM module system, becoming part of `config`
- **Traits** — structured data collected across aspects and made available to consumers

Traits exist at three evaluation tiers:

| Tier | Depends on | Available at | Use case |
|------|-----------|-------------|----------|
| **1. Static** | Nothing | Pipeline time | Plain values: `firewall.ports = [ 80 443 ];` |
| **2. Pipeline-parametric** | Den context (`host`, `user`) | Pipeline time | `firewall = { host, ... }: { ports = host.openPorts or []; };` |
| **3. Module-evaluated** | Module system (`config`, `pkgs`) | Inside `evalModules` | `persistence = { config, ... }: { user = config.services.nginx.user; };` |

Tier 1 and 2 traits are resolved during pipeline execution. This means **parametric include discriminators can branch on trait data** — for example, conditionally including a TLS hardening aspect only when port 443 appears in collected firewall traits. Tier 1/2 data accumulates in pipeline state and is accessible via a stateful trait handler that reads live state at each point of use.

Tier 3 traits participate in the `evalModules` fixpoint. They cannot drive pipeline-time decisions but are available to class module consumers via lazy thunk pre-application.

The tier is detected automatically:
- Plain value (not a function) → **Tier 1**
- Function with only den context args (`host`, `user`, etc.) → **Tier 2**, resolve now
- Function with any module-system args (`config`, `pkgs`, `lib`, `options`) → **Tier 3**, defer to `evalModules`

**Mixed args** (`{ host, config, ... }:`): classified as **Tier 3**. The den args (`host`) are pre-applied at pipeline time (same as `wrapClassModule`), reducing the function to `{ config, ... }:` for deferred evaluation. This matches existing behavior — `wrapClassModule` already handles the mixed-arg case by extracting den args and wrapping the remainder.

Detection uses `builtins.functionArgs` and checks each arg name against the pipeline context (`ctx ? ${k}`). Module-system args are the well-known set: `config`, `lib`, `pkgs`, `options`, `modulesPath`, plus any `_module.*` args. If any are present, the function is Tier 3.

## Design

### 1. Schema Registry

Two new **top-level** options declare what classes and traits exist. These live at `den.classes` and `den.traits` (not under `den.schema`) to avoid a module evaluation cycle — `den.schema` contains entity-kind entries that aspects read via `den.schema.aspect`, and nesting class/trait registries inside `den.schema` creates a cycle through aspect submodule evaluation.

```nix
# Classes — evaluation domains (top-level)
den.classes.nixos = {
  description = "NixOS system configuration";
};
den.classes.homeManager = {
  description = "Home Manager user configuration";
  forwardTo = {
    class = "nixos";
    path = { host, user }: [ "home-manager" "users" user.userName ];
  };
};
den.classes.darwin = {
  description = "nix-darwin system configuration";
};

# Traits — semantic data channels (top-level)
den.traits.firewall = {
  description = "Firewall port and interface declarations";
  collection = "list";   # "list" (default) or "map"
  partialOk = true;      # Tier 3 emissions allowed alongside pipeline-time consumers
};
den.traits.persistence = {
  description = "Impermanence path declarations";
  collection = "list";
};
den.traits.service-endpoints = {
  description = "Named service endpoints for discovery";
  collection = "map";   # keyed by top-level attrset keys — duplicates error
  # Emitters provide: service-endpoints = { nginx = { port = 80; }; postgres = { port = 5432; }; };
  # Result merges top-level keys: { nginx = { port = 80; }; postgres = { port = 5432; }; }
};
```

**Classes** declare evaluation domains. Optional `forwardTo` specifies where collected output lands (replaces per-call forward routing for the common case).

**Traits** declare semantic data channels. Optional `type` provides validation of the data shape.

**Class/trait name collision.** A key cannot be both a class and a trait. The schema module asserts `builtins.intersectAttrs classes traits == {}` and errors with full provenance on collision.

Built-in batteries auto-register their classes at module import time. The forward battery auto-registers classes created via `den.provides.forward`.

**Aspect-level trait installation.** Aspects can declare the traits they emit, making them self-contained. `traits` is a structural key on aspects (like `policies`, `meta`, `includes`):

```nix
den.aspects.nginx = {
  # Aspect declares the trait it emits
  traits.firewall = {
    description = "Firewall port declarations";
    collection = "list";
    partialOk = true;
  };

  nixos.services.nginx.enable = true;
  firewall = { ports = [ 80 443 ]; };
};
```

Aspect-level `traits` declarations merge into `den.traits` via the module system (config merging), not via pipeline-time mutation. The `traits` key is an explicit option on `aspectSubmodule` (like `policies`), and its values are merged into `den.traits` during module evaluation — before the pipeline runs. The pipeline reads the already-merged `den.traits` at construction time. This means:
- Aspect authors don't need to tell consumers "also add `den.traits.firewall`"
- Multiple aspects can declare the same trait — declarations must be compatible (same `collection`, same `type`)
- Namespaced aspects carry their trait declarations — importing the namespace brings both aspects and trait schemas
- `den.traits` remains the canonical registry; aspect-level declarations merge into it

The same pattern works for classes: `aspect.classes.myClass = { ... }` merges into `den.classes`. This mirrors how `aspect.policies` already works.

Phase 1 adds `"traits"` and `"classes"` to `structuralKeysSet` so they are excluded from freeform content classification.

### 2. Aspect Content Type

The aspect submodule's freeform type changes from `lazyAttrsOf deferredModule` to `lazyAttrsOf (aspectContentType typeCfg)`. This is a uniform wrapper type that accepts any value and preserves identity information, deferring semantic classification to the pipeline.

```nix
aspectContentType = typeCfg:
  lib.types.mkOptionType {
    name = "aspectContent";
    description = "class module, trait emission, or nested aspect";
    check = _: true;
    merge = loc: defs:
      let
        keyName = lib.last loc;
        values = map (d: d.value) defs;
      in {
        __contentValues = map (d: { inherit (d) value file; }) defs;
        __provider = (typeCfg.providerPrefix or []) ++ [ keyName ];
      };
  };
```

**Design rationale.** The previous `deferredModule` freeform type forced all non-structural keys through module-shaped type checking. Traits (plain data, lists, functions returning data) don't conform to module shape. A schema-aware freeform type that dispatches based on `den.classes` / `den.traits` creates a chicken-and-egg problem: the schema is a config value populated during the same evaluation that runs type checking.

The solution: the type layer is **generic**. It wraps every non-structural value uniformly with identity metadata (`__provider`). The pipeline classifies values against the fully-resolved schema at pipeline time, detecting parametric vs static from the values themselves (`builtins.isFunction`).

**Wrapper fields:**

| Field | Purpose |
|-------|---------|
| `__contentValues` | List of `{ value, file }` records (preserves multi-site merging and file provenance for error messages) |
| `__provider` | Identity path: `typeCfg.providerPrefix ++ [ keyName ]` |

**Content classification** is a product of two axes, resolved by the pipeline:

| | aspect | classModule | trait | recursive parametric |
|---|---|---|---|---|
| **static** | attrset with class/trait sub-keys | attrset module | plain data | — |
| **parametric** | function → aspect | function → module | function → data | function → function (curried) |

**Multi-site definitions.** The merge preserves all definitions with file provenance via `map (d: { inherit (d) value file; }) defs`. For class modules, the values from `__contentValues` become the import list (same semantics as the previous `deferredModule` merge). For traits, the pipeline takes the single value or applies the collection strategy. For nested aspects, recursion applies. The `file` field is **provenance metadata** — useful for debugging, error messages, and tooling (same lineage as `aspect.meta.file`). It is NOT used for dedup or conflict detection; `chainHandler`'s `state.includesChain` is the authoritative identity for those purposes, since it correctly distinguishes parametric dispatches from the same file.

**`provides` unification.** Since aspect-level freeform keys now carry the same `__provider` identity tracking that `provides` used `providerType` for, `provides` becomes structurally redundant. It stays as an alias with a deprecation warning. Nested aspects defined directly on the aspect (`igloo.tux = { ... }`) get identical identity pathing to `igloo.provides.tux = { ... }`.

### 3. Structural Detection

`compileStatic` unwraps `aspectContentType` values and uses the schema registry to classify each non-structural key:

```
For each key k on aspect (excluding structuralKeysSet):
  wrapped = aspect.${k}   # { __contentValues, __provider }

  1. k ∈ den.classes  →  class leaf (emit-class, existing behavior)
  2. k ∈ den.traits   →  trait emission (emit-trait, new effect)
  3. k ∉ registry, value has class/trait sub-keys  →  nested aspect (recurse)
  4. k ∉ registry, no recognized sub-keys  →  freeform data (trace warning, ignored)
```

**Step 4 emits a trace warning** — not silent, not an error. `builtins.trace "aspect '${name}': key '${k}' is not a registered class, trait, or nested aspect — ignored"`. This catches typos (`firwall` instead of `firewall`) without breaking anything.

**Traits are optional.** An aspect can put `firewall = { ports = [80]; }` on itself without any trait declaration existing. The data sits as freeform data (step 4, with trace warning). If a networking aspect is later included that declares `traits.firewall`, the pipeline starts collecting it. This is the duck-typing philosophy: aspects say "here's my data," and the pipeline only acts on it when a trait declaration makes it meaningful.

The trait registry is a **discovery mechanism**, not a gatekeeper. Without a declaration, data passes through with a trace warning. With a declaration, the pipeline collects and routes it.

**Example:**

```nix
# The networking aspect declares the firewall trait
den.aspects.networking = {
  traits.firewall = {
    description = "Firewall port declarations";
    collection = "list";
    partialOk = true;
  };
  nixos.networking.firewall.enable = true;
};

den.aspects.nginx = {
  includes = [ den.aspects.networking ];

  # Step 1: "nixos" is a registered class → emit-class
  nixos = { config, ... }: { services.nginx.enable = true; };

  # Step 2: "firewall" is now a registered trait (via networking) → emit-trait
  firewall = { ports = [ 80 443 ]; };

  # Step 3: "monitoring" is unknown, but contains "nixos" (a class) → nested aspect
  monitoring = {
    nixos.services.prometheus.exporters.nginx.enable = true;
  };

  # Step 4: "metadata" is unknown, no class/trait sub-keys → freeform, trace warning
  metadata.author = "nginx-team";
};
```

If `den.aspects.nginx` is used **without** including `networking`, the `firewall` key is freeform data — trace warning, ignored. Once `networking` is in the include tree and its `traits.firewall` declaration is registered, the pipeline starts collecting `firewall` emissions.

**Nested aspect detection operates on unwrapped values.** For step 3, `compileStatic` reads `wrapped.__contentValues` and inspects the definitions:
- If all defs are attrsets, their keys are checked against the class and trait registries. If any child key is a registered class or trait, the parent is a nested aspect.
- If defs include both functions and attrsets, and the attrset defs contain class/trait sub-keys, the key is treated as a parametric nested aspect (same mechanism as `providerType`). The attrset defs provide evidence of aspect structure; the function defs are parametric variants.
- If ALL defs are functions and the key is unknown, it falls through to step 4 (freeform with trace warning). A bare function on an unknown key cannot be classified as a nested aspect without evidence of aspect structure, since it could equally be an unregistered trait emission.
- If a child is itself an attrset whose children contain registered class/trait keys, the parent is also detected as a nested aspect — this enables recursive nesting like `development.gui.vscode`.

Detection happens per-`compileStatic` pass: when the pipeline recurses into a nested aspect, children are classified by the same rules.

### 4. Trait Collection and Consumption

Trait collection happens in two phases matching the evaluation tiers.

#### Phase A: Pipeline-time collection (Tier 1 + 2)

**Emission.** When `compileStatic` encounters a trait key, it unwraps `__contentValues` and emits one `emit-trait` per definition. This fans out multi-site definitions into individual emissions, each with its own tier detection:

```nix
# For each value in wrapped.__contentValues:
fx.send "emit-trait" { trait = "firewall"; value = singleValue; chain = state.includesChain; }
```

**Tier detection.** The handler inspects the value:
- Plain value (not a function) → Tier 1, collect immediately
- Function with only den context args (`host`, `user`, etc.) → Tier 2, resolve now using pipeline context, collect the result
- Function with module-system args (`config`, `pkgs`, `lib`, `options`) → Tier 3, defer to Phase B

**Collection.** A new `traitCollectorHandler` accumulates Tier 1/2 resolved data in pipeline state. It is constructed with `ctx` as a closure arg (like `classCollectorHandler` takes `{ targetClass }`), since the pipeline context is fixed for the duration of a pipeline run:

```nix
moduleSystemArgs = lib.genAttrs [ "config" "lib" "pkgs" "options" "modulesPath" ] (_: true);

traitCollectorHandler = { ctx }:
{
  "emit-trait" = { param, state }:
    let
      value = param.value;
      # Tier detection reuses wrapClassModule's existing logic:
      # builtins.functionArgs gives { argName = hasDefault; ... }
      # Check each arg against pipeline context (ctx ? ${k}) and moduleSystemArgs.
      # _module.* args are detected via lib.hasPrefix "_module." check.
      fargs = if builtins.isFunction value then builtins.functionArgs value else {};
      hasModuleArgs = builtins.any (k:
        moduleSystemArgs ? ${k} || lib.hasPrefix "_module." k
      ) (builtins.attrNames fargs);
      denArgNames = builtins.filter (k: ctx ? ${k}) (builtins.attrNames fargs);
      isDenArgsOnly = fargs != {} && !hasModuleArgs
        && builtins.length denArgNames == builtins.length (builtins.attrNames fargs);
      resolved =
        if !builtins.isFunction value then value                                     # Tier 1: plain data
        else if isDenArgsOnly then value (lib.getAttrs denArgNames ctx)              # Tier 2: resolve now
        else null;                                                                   # Tier 3: defer
      # Note: { ... }: (no named args, just ellipsis) has fargs == {}, so isDenArgsOnly
      # is false and the function is conservatively classified as Tier 3 (deferred).
      # Recommend using plain values or explicitly naming den args.
    in
    { resume = null;
      state = state // {
        traits = _: ((state.traits or (_: {})) null) // {
          ${param.trait} = ((state.traits or (_: {})) null).${param.trait} or []) ++ (
            if resolved != null then [ { value = resolved; chain = param.chain; } ] else []
          );
        };
        deferredTraits = _: ((state.deferredTraits or (_: {})) null) // {
          ${param.trait} = ((state.deferredTraits or (_: {})) null).${param.trait} or []) ++ (
            if resolved == null then [ { value = value; chain = param.chain; } ] else []
          );
        };
      };
    };
};
```

Both `state.traits` and `state.deferredTraits` are thunk-wrapped (`_: value`) to survive `builtins.deepSeq` on the trampoline, matching the pattern used for `state.provideTo`, `state.imports`, etc.

**Pipeline-time availability.** Tier 1/2 trait data accumulates in pipeline state (in `state.traits`). Downstream parametric functions access it via a **stateful trait handler** — NOT `constantHandler` (which snapshots a value). The trait handler reads live state at each point of use, so consumers see all trait data collected up to that point in the depth-first walk:

```nix
traitArgHandler = traitNames:
  builtins.mapAttrs (traitName: _:
    { param, state }:
    { resume = map (e: e.value) (state.traits.${traitName} or []);
      state = state // {
        consumedTraits = (state.consumedTraits or {}) // { ${traitName} = true; };
      };
    }
  ) traitNames;
```

This follows the `constantHandler` pattern: each registered trait name becomes a handler that reads live state (like `ctxSeenHandler`, not a snapshot). When a parametric function requests `{ firewall, ... }:`, `bind.fn` sends a bare `"firewall"` effect, and the trait handler responds with the current collected list. Priority ordering in `composeHandlers` ensures den context handlers take precedence over trait handlers on name collision.

```nix
# This runs at PIPELINE TIME — firewall trait data is already collected
den.aspects.server = {
  includes = [
    ({ firewall, ... }:
      lib.optionalAttrs (builtins.any (f: builtins.elem 443 (f.ports or [])) firewall) {
        includes = [ den.aspects.tls-hardening ];
      }
    )
  ];
};
```

The `{ firewall, ... }:` discriminator receives the collected list of Tier 1/2 firewall trait emissions from upstream aspects. This enables **conditional inclusion based on semantic data** — a capability the current pipeline lacks.

#### Phase B: Module-time collection (Tier 3)

Deferred trait emissions (functions requiring `config`, `pkgs`, etc.) are injected into the `evalModules` call as a generated NixOS module:

```nix
# Generated by pipeline, invisible to user:
{ config, lib, ... }: {
  options._den.traits = lib.mkOption {
    type = lib.types.attrsOf (lib.types.listOf lib.types.anything);
    default = {};
    internal = true;
  };
  config._den.traits.firewall =
    pipelineResolvedData                     # Tier 1/2: already resolved
    ++ map (fn: fn { inherit config lib; })  # Tier 3: resolve inside fixpoint
       deferredFirewallEmissions;
}
```

The `_den.traits` option merges Tier 1/2 data (already resolved at pipeline time) with Tier 3 data (resolved now inside the fixpoint). Consumers see a single unified list.

**Self-referential constraint.** A Tier 3 trait function that reads its own trait via `config._den.traits.X` creates infinite recursion in the `evalModules` fixpoint. This is a documented constraint — the Nix evaluator's infinite recursion error clearly identifies the offending thunk. The `chain` provenance on emissions is available to implement a structural guard (filtering own-chain emissions from the visible set) if this proves to be a common footgun, but speculative complexity is not warranted.

#### Consumption via unified pre-application

**Key design decision:** Trait args are delivered through `wrapClassModule`, not `_module.args`. This keeps the pre-application model unified — consumers see no difference between den context args (`host`, `user`) and trait args (`firewall`, `persistence`).

**The timing problem:** `wrapClassModule` runs at pipeline time (before `evalModules`), but Tier 3 trait data only exists inside the `evalModules` fixpoint. The solution is **lazy thunk pre-application**: the wrapper pre-applies a thunk that reads from `moduleArgs` at eval time.

```nix
wrapClassModule = { module, ctx, traitNames, aspectPolicy, globalPolicy, ... }:
  let
    moduleArgNames = builtins.functionArgs module;
    denArgs = /* existing: extract from ctx */;
    traitArgNames = lib.filter (k: traitNames ? k) (builtins.attrNames moduleArgNames);
  in
  if traitArgNames == [] && denArgs == {} then
    { inherit module; wrapped = false; }
  # If trait args are present, full application is suppressed even when all
  # other named args are den args — trait thunks require moduleArgs.config
  # at eval time, so the module must always be wrapped.
  else
    let
      wrapper = moduleArgs:
        module (moduleArgs // denArgs // lib.genAttrs traitArgNames (name:
          moduleArgs.config._den.traits.${name} or []  # lazy: resolved inside evalModules
        ));
    in
    { module = lib.setFunctionArgs wrapper (/* remaining non-den non-trait args */);
      wrapped = true; };
```

The thunk `moduleArgs.config._den.traits.firewall` contains **both** Tier 1/2 and Tier 3 data — merged by the injection module. Nix's lazy evaluation means it's only forced when the consumer accesses it, at which point the fixpoint provides all tiers.

**Consumer code (class module — sees all tiers):**

```nix
den.aspects.hardened = {
  nixos = { firewall, config, ... }: {
    networking.firewall.allowedTCPPorts =
      lib.concatMap (f: f.ports) firewall;
  };
};
```

**Consumer code (parametric discriminator — sees Tier 1/2 only):**

```nix
den.aspects.conditional = {
  includes = [
    ({ persistence, ... }:
      if persistence != [] then { includes = [ den.aspects.impermanence ]; }
      else {}
    )
  ];
};
```

Both consumers use the same `{ traitName, ... }:` syntax. The difference is when they run: parametric discriminators run at pipeline time (Tier 1/2 data), class modules run at eval time (all tiers).

**Collision policy.** The existing three-level collision policy (aspect → entity → global) applies to trait args the same way it applies to den context args.

**Handler name collision avoidance.** Entity context args (`host`, `user`) and trait args (`firewall`, `persistence`) both need effect handlers in the pipeline. Trait handlers are registered under bare names (matching the `constantHandler` pattern used for entity context), with **priority ordering**: den context handlers are composed first, trait handlers second. Since `composeHandlers` gives the first handler's state priority on overlap, den context always wins if a trait name collides with an entity context arg (e.g., a trait named `host`). The consumer writes `{ firewall, ... }:` and the pipeline's existing `bind.fn` resolution sends a bare `"firewall"` effect — no translation layer needed.

**`traitNames` propagation.** `wrapClassModule` needs to know which arg names are registered traits. This comes from `den.traits` via the pipeline's handler setup — the set of trait names is available at pipeline construction time. `wrapClassModule` checks trait names AFTER den context args, so den context always takes priority.

### 5. Nested Aspects (provides → direct nesting)

`provides` becomes an alias for direct nesting, like `_` already is:

```nix
# These are identical:
den.aspects.igloo.provides.tux = { homeManager.programs.git.enable = true; };
den.aspects.igloo.tux = { homeManager.programs.git.enable = true; };
```

**Detection:** Unknown keys whose values contain registered class or trait sub-keys are nested aspects. The pipeline recurses into them as sub-aspects with the key as the scope name.

**Nested aspects can contain traits:**

```nix
den.aspects.igloo = {
  nixos.networking.hostName = "igloo";
  firewall = { config, ... }: { ports = [ 22 ]; };

  tux = {
    homeManager.programs.git.enable = true;
    firewall = { config, ... }: { ports = [ 8080 ]; };
  };
};
```

Both `firewall` emissions get collected. The consumer sees `firewall = [ { ports = [22]; } { ports = [8080]; } ]`.

**Self-provide becomes implicit.** An aspect named `igloo` IS the `igloo` scope. The current `provides.${self.name}` self-provide pattern is unnecessary.

**Type-level routing.** With `aspectContentType` as the freeform type, all non-structural values are wrapped uniformly with identity metadata (`__provider`). The pipeline's `compileStatic` reads the wrapper and classifies against the schema registry. The type layer does not need schema knowledge — it preserves values and identity, and the pipeline validates. See **Section 2: Aspect Content Type** for the full type definition.

**Recursive nesting.** Detection is per-`compileStatic` pass, not a single pre-scan. When the pipeline recurses into a nested aspect, children are classified by the same rules. This enables multi-level nesting:

```nix
den.aspects.development = {
  gui = {
    vscode = {
      nixos.programs.vscode.enable = true;
      homeManager.programs.vscode.extensions = [ ... ];
    };
    alacritty = {
      homeManager.programs.alacritty.enable = true;
    };
  };
  cli = {
    nixos.programs.git.enable = true;
  };
};
```

`gui` is detected as a nested aspect because its children (`vscode`, `alacritty`) themselves contain registered class keys. Today this pattern is expressed as `provides.gui.provides.vscode = { ... }` — direct nesting makes it natural. A pure organizational level (containing only nested aspects, no class or trait keys of its own) is valid — the pipeline checks children recursively during each `compileStatic` pass.

**Migration:** `provides` stays as an alias with a deprecation warning. Existing configs work unchanged.

### 6. Forward Battery Integration

The forward battery currently computes class routing per-call via `fromClass`/`intoClass`/`intoPath` functions. With the schema registry, the common case is declarative:

```nix
den.classes.homeManager = {
  description = "Home Manager user configuration";
  forwardTo = {
    class = "nixos";
    path = { host, user }: [ "home-manager" "users" user.userName ];
  };
};
```

Built-in batteries (`home-manager.nix`, `hjem.nix`, `maid.nix`, `os-class.nix`, `os-user.nix`) register their classes at import time. The `forwardTo` on the class schema becomes the source of truth for "where does this class's output land?"

The forward battery's per-call `intoClass`/`intoPath` remains available as an override for advanced patterns. For the common case, the schema-level declaration is sufficient.

### 7. Namespace Integration

Namespaces (`den.ful.<name>`) already export aspects, schema, and stages. Traits become a fourth sub-namespace, following the same pattern.

**Type extension.** `namespaceType` in `namespace-types.nix` gains a `traits` option:

```nix
options.traits = lib.mkOption {
  description = "Trait declarations for this namespace";
  type = lib.types.lazyAttrsOf traitSchemaType;
  default = {};
};
```

**Provider exports traits alongside aspects:**

```nix
imports = [ (inputs.den.namespace "monitoring" true) ];

# Aspects
monitoring.prometheus = {
  nixos.services.prometheus.enable = true;
  firewall = { ports = [ 9090 ]; };
};

# Trait declarations
monitoring.traits.firewall = {
  description = "Firewall declarations";
  collection = "list";
  partialOk = true;
};
```

**Consumer imports both:**

```nix
imports = [ (inputs.den.namespace "monitoring" [ inputs.upstream ]) ];

# Aspects available via namespace
den.aspects.server.includes = [ monitoring.prometheus ];

# Trait declarations auto-merged into den.traits
# monitoring.traits.firewall → den.traits.firewall
```

**Merge on import.** The namespace import module (in `namespace.nix`) merges `upstream.denful.NAME.traits` directly into `config.den.traits`, the same way it merges aspects into `den.ful.NAME`. Trait declarations go through `den.traits` (the canonical registry), not through `namespace.schema` (which handles entity kind schemas). The `_` / `__functor` stripping for dedup applies.

**Cross-namespace trait contributions.** Multiple namespaces can declare the same trait (e.g., both `monitoring.traits.firewall` and `security.traits.firewall`). Declarations must be compatible — same `collection` mode, same `type` (if provided). If incompatible, the module system's merge produces an error. Emissions from all namespaces contribute to the unified `den.traits.firewall` collection.

**Class declarations in namespaces.** The same pattern applies to classes — a namespace can declare custom classes via `monitoring.classes.prometheus-exporter = { ... }`, merged into `den.classes` on import. This enables namespaces to ship complete "class + trait + aspect" bundles.

## End-to-End Example

```nix
# --- TRAIT-DECLARING ASPECTS ---

# The networking aspect declares the firewall trait
den.aspects.networking = {
  traits.firewall = {
    description = "Firewall port declarations";
    collection = "list";
    partialOk = true;
  };
  nixos.networking.firewall.enable = true;
};

# The storage aspect declares the persistence trait
den.aspects.storage = {
  traits.persistence = {
    description = "Impermanence path declarations";
    collection = "list";
  };
};

# --- PRODUCERS (emit trait data, don't need to declare traits themselves) ---

den.aspects.nginx = {
  includes = [ den.aspects.networking den.aspects.storage ];
  nixos.services.nginx.enable = true;

  # Tier 1 traits: available at pipeline time
  firewall = { ports = [ 80 443 ]; };
  persistence = { directories = [ "/var/lib/nginx" "/var/log/nginx" ]; };
};

den.aspects.postgres = {
  includes = [ den.aspects.networking den.aspects.storage ];
  nixos.services.postgresql.enable = true;

  firewall = { ports = [ 5432 ]; };
  persistence = { directories = [ "/var/lib/postgresql" ]; };
};

den.aspects.custom-service = {
  includes = [ den.aspects.networking ];
  nixos.systemd.services.custom.enable = true;

  # Tier 2 trait (pipeline-parametric): resolved at pipeline time
  firewall = { host, ... }: {
    ports = if host.datacenter == "dmz" then [ 8443 ] else [ 8080 ];
  };
};

den.aspects.nginx-monitoring = {
  includes = [ den.aspects.networking ];
  nixos.services.prometheus.exporters.nginx.enable = true;

  # Tier 3 trait (needs config): deferred to evalModules
  # Supplementary metadata — doesn't affect structural decisions
  firewall = { config, ... }: {
    user = config.services.nginx.user;
    interfaces = builtins.attrNames config.networking.interfaces;
  };
};

# --- PIPELINE-TIME CONSUMER (discriminator) ---

den.aspects.secure-server = {
  includes = [
    # Tier 1/2 firewall data is available HERE at pipeline time
    ({ firewall, ... }:
      let hasHttps = builtins.any (f: builtins.elem 443 (f.ports or [])) firewall;
      in lib.optionalAttrs hasHttps {
        includes = [ den.aspects.tls-hardening den.aspects.acme ];
      }
    )
  ];
};

# --- MODULE-TIME CONSUMER (class module) ---

den.aspects.hardened-server = {
  # Receives ALL tiers (including Tier 3) via lazy thunk
  nixos = { firewall, persistence, config, ... }: {
    networking.firewall.allowedTCPPorts =
      lib.concatMap (f: f.ports or []) firewall;
    environment.persistence."/persist".directories =
      lib.concatMap (p: p.directories or []) persistence;
  };
};

# --- NESTED ASPECT ---

den.aspects.igloo = {
  includes = [
    den.aspects.nginx          # brings networking + storage traits via includes
    den.aspects.postgres
    den.aspects.custom-service
    den.aspects.nginx-monitoring
    den.aspects.secure-server
    den.aspects.hardened-server
  ];
  nixos.networking.hostName = "igloo";

  # Nested aspect: detected because it contains homeManager (registered class)
  tux = {
    homeManager.programs.git.enable = true;
    firewall = { ports = [ 8080 ]; };  # Tier 1: contributes to firewall trait
  };
};
```

**Pipeline flow:**

1. Resolve `nginx` (includes `networking` and `storage`):
   - `networking.traits.firewall` → registers `firewall` trait in schema
   - `storage.traits.persistence` → registers `persistence` trait in schema
   - `nginx.nixos` → `emit-class { class = "nixos"; ... }`
   - `nginx.firewall` → `emit-trait` — Tier 1, collected: `{ ports = [80, 443]; }`
   - `nginx.persistence` → `emit-trait` — Tier 1, collected: `{ directories = ["/var/lib/nginx", "/var/log/nginx"]; }`

2. Resolve `postgres` (includes `networking` and `storage`):
   - `postgres.firewall` → Tier 1, collected: `{ ports = [5432]; }`
   - `postgres.persistence` → Tier 1, collected: `{ directories = ["/var/lib/postgresql"]; }`

3. Resolve `custom-service` (includes `networking`):
   - `custom-service.firewall` → Tier 2 (has `host` arg), resolved now: `{ ports = [8443]; }`

4. Resolve `nginx-monitoring` (includes `networking`):
   - `nginx-monitoring.firewall` → Tier 3 (has `config` arg), deferred

5. Resolve `secure-server.includes`:
   - Pipeline-time trait state so far: `firewall = [ { ports=[80,443]; } { ports=[5432]; } { ports=[8443]; } ]`
   - Parametric `{ firewall, ... }:` receives Tier 1+2 data via stateful trait handler
   - Port 443 IS in Tier 1+2 data (from nginx) → `tls-hardening` and `acme` INCLUDED
   - `firewall` has `partialOk = true`, so the Tier 3 emission from `nginx-monitoring` does not error

6. Resolve `hardened-server` — no own trait emissions

7. Resolve `igloo.tux`:
   - Detected as nested aspect (contains `homeManager`)
   - `tux.firewall` → Tier 1, collected: `{ ports = [8080]; }`

8. Emit class modules:
   - `hardened-server.nixos` → `wrapClassModule` pre-applies:
     - `firewall` → lazy thunk reading `config._den.traits.firewall` (all tiers merged)
     - `persistence` → lazy thunk reading `config._den.traits.persistence`

9. `evalModules` runs:
   - `_den.traits.firewall` = Tier 1+2 data ++ Tier 3 deferred emission from `nginx-monitoring` resolved with `{ config; }`
   - Consumer forces `firewall` → gets: `[ { ports=[80,443]; } { ports=[5432]; } { ports=[8443]; } { user="nginx"; interfaces=[...]; } { ports=[8080]; } ]`
   - Consumer forces `persistence` → gets: `[ { directories = ["/var/lib/nginx" "/var/log/nginx"]; } { directories = ["/var/lib/postgresql"]; } ]`
   - `hardened-server.nixos` uses `f.ports or []` so the Tier 3 entry (which has `user`/`interfaces` but no `ports`) contributes `[]` — no crash

## Migration Path

### Phase 1: Schema registry + aspect content type (additive)

Add `den.classes` and `den.traits`. Add collision assertion. Replace the aspect freeform type from `lazyAttrsOf deferredModule` to `lazyAttrsOf (aspectContentType typeCfg)`. Update `compileStatic` to unwrap content values. Pipeline reads the registry but behavior is unchanged for existing configs — all non-structural keys with no trait registration still emit as classes. Registered traits emit via `emit-trait` instead of `emit-class`.

### Phase 2: Structural detection

`compileStatic` uses the registry for key classification. Unknown keys with class/trait sub-keys become nested aspects. Unclassified keys emit trace warnings. `provides` gets deprecation warning pointing to direct nesting.

### Phase 3: Trait collection

`traitCollectorHandler`, `traitArgHandler`, `_den.traits.*` injection into evalModules, `wrapClassModule` trait arg pre-application. Full trait producer → consumer flow within a single entity's pipeline.

### Phase 4: Cross-entity trait distribution

Replace provide-to's labeled data path with trait-based cross-entity routing. Remove `"provide-to"` from `structuralKeysSet`. Refactor `distribute-provide-to.nix` to merge cross-entity trait data into target's initial `state.traits`. Update transition handler's sibling routing to capture sub-pipeline trait state. The `"provide-to"` effect payload changes from raw labeled data to trait state.

### Phase 5: Forward simplification

Forward battery uses `den.classes.*.forwardTo` for routing. Per-call `intoClass`/`intoPath` becomes override-only.

Phase 1 is purely additive. Phase 2 depends on Phase 1 (needs the registry and content type). Phase 3 depends on Phase 2 (needs `emit-trait` routing). Phase 4 depends on Phase 3 (needs trait collection to replace provide-to labeled data). Phase 5 depends on Phase 1 (needs `forwardTo` on class schema). Phases 4 and 5 are independent of each other.

## Design Decisions

These were open questions during design; all are now resolved.

1. **Trait ordering.** Collection order follows the depth-first include walk — deterministic for a given tree, but fragile across tree changes. Consumers should treat trait lists as unordered sets. The `collection` field on the schema controls merge semantics: `"list"` concatenates, `"map"` merges by key with error on duplicates.

2. **Trait merging.** Handled by the `collection` field on `den.traits`. `"list"` (default) concatenates all emissions. `"map"` merges by key, errors on duplicates with full provenance in the error message. Consumers receive the merged result directly. For `"map"` traits, "duplicate" means the same top-level key appears in two emissions — deep merging is not performed. Two emitters providing `service-endpoints.nginx = { ... }` with different sub-keys is a duplicate error, not a deep merge. Emitters should coordinate or use distinct keys. For Tier 3 emissions (deferred functions), duplicate detection for `"map"` traits happens inside the generated NixOS module after function application, not at pipeline time — the pipeline cannot inspect return values before `evalModules` runs.

3. **Trait validation.** When `den.traits.*.type` is provided, it validates each emission at eval time. When `null` (default), no validation — the consumer is responsible for handling the shape. Enforcement when present, optional otherwise.

4. **Cross-entity traits.** Within a single pipeline run, traits are scoped to the current entity. Cross-entity trait aggregation uses the policy system's sibling routing, with traits as the data model and provide-to as the transport. See **Section 8: Provide-To Unification** for the full design.

    The mechanism:

    - A policy fans out from the source entity to target entities (sibling routing: `from == to`)
    - The transition handler runs a sub-pipeline for each target, which collects traits in `state.traits` as usual
    - The transition handler captures the sub-pipeline's trait state and emits a `"provide-to"` effect with the trait data and target identity
    - After phase 1 completes, the distribution phase merges cross-entity trait data into each target's initial `state.traits` before the target's own pipeline runs
    - The target's trait consumption works identically for within-entity and cross-entity data — same `{ traitName, ... }:` syntax

    ```nix
    # Trait declaration (on a shared aspect)
    den.aspects.backup-aware.traits.backup-targets = {
      description = "Paths and schedules for remote backup";
      collection = "list";
    };

    # Client hosts emit backup-targets (Tier 1) — no awareness of cross-entity routing
    den.aspects.postgres-host = {
      includes = [ den.aspects.backup-aware ];
      backup-targets = { paths = [ "/var/lib/postgresql" ]; schedule = "daily"; };
    };

    den.aspects.mail-host = {
      includes = [ den.aspects.backup-aware ];
      backup-targets = { paths = [ "/var/lib/mail" ]; schedule = "hourly"; };
    };

    # Policy: backup server fans out to clients, collects their traits
    den.policies.backup-to-clients = {
      from = "host"; to = "host"; as = "backup-client";
      resolve = { host, ... }:
        lib.optional (host.role == "backup-server")
          (map (h: { backup-client = h; inherit host; })
            (lib.filter (h: h.backup.enable or false)
              (lib.attrValues den.hosts.x86_64-linux)));
    };

    # Backup server consumes aggregated trait data from all clients
    # backup-targets arrives via: sub-pipeline traits → provide-to → target initial state
    den.aspects.backup-server = {
      includes = [ den.aspects.backup-aware ];
      nixos = { backup-targets, config, ... }: {
        services.restic.backups = lib.listToAttrs (lib.imap0 (i: t: {
          name = "backup-${toString i}";
          value = { paths = t.paths; timerConfig.OnCalendar = t.schedule; };
        }) backup-targets);
      };
    };
    ```

    Key difference from the provide-to labeled data pattern: the source aspect declares traits without any awareness of cross-entity routing. The policy controls which entities' traits flow where. The target consumes trait data uniformly.

5. **Tier visibility mismatch.** If a pipeline-time discriminator consumes trait `X` AND any aspect emits a Tier 3 value for `X`, the pipeline **errors by default** — the discriminator made a decision without seeing all data. This check is a **post-pipeline validation**: after the full include tree is walked, if `state.deferredTraits.${trait}` is non-empty AND the trait's handler was invoked during the walk (tracked via a `consumedTraits` set in state), error unless `partialOk` is set on the trait schema. Fixes: (a) rewrite the Tier 3 emitter to Tier 1/2, (b) move the discriminator into a class module, or (c) mark the trait as `partialOk = true` in `den.traits` to acknowledge that pipeline-time consumers accept partial data. With `partialOk`, Tier 3 emissions supplement Tier 1/2 data without invalidating pipeline-time decisions — useful when Tier 3 adds metadata (e.g., resolved usernames) that doesn't affect the discriminator's structural decision (e.g., which ports are open). The `partialOk` flag lives on the trait schema declaration — the trait author decides whether partial consumption is safe for all consumers of that trait.

6. **Top-level `den.classes` / `den.traits`.** Classes and traits are top-level options (`den.classes`, `den.traits`), not nested under `den.schema`. This avoids a module evaluation cycle: `den.schema` contains entity-kind entries that aspects read via `den.schema.aspect`, and nesting class/trait registries inside the same submodule creates a cycle through aspect submodule evaluation. Top-level placement breaks the cycle — the pipeline reads `den.classes`/`den.traits` independently of `den.schema`. A `builtins.intersectAttrs` collision check ensures no key appears in both.

7. **Forward `forwardTo` scope.** Phase 4's schema-level `forwardTo` covers the common case (`class` + `path`). Complex forward patterns (custom `evalConfig`, `adapters`, `extraArgs`) continue using the battery's per-call API.

8. **Self-referential trait recursion.** A Tier 3 trait function reading its own trait (`config._den.traits.X` inside an `X` emission) creates infinite recursion in the `evalModules` fixpoint. This is a documented constraint. The Nix evaluator's infinite recursion error clearly identifies the offending thunk. The `chain` provenance on emissions is available to implement a structural guard (filtering own-chain emissions from the visible set) if this proves to be a common footgun, but speculative complexity is deferred.

9. **`hasAspect` compatibility.** The existing `hasAspect` mechanism continues to work unchanged. Traits are a more powerful superset (check data, not just presence). No migration needed.

10. **Diagram integration.** The trace infrastructure captures `emit-trait` effects alongside `emit-class` via the existing `chain` provenance. Visualization in the `diag` library is deferred follow-up work.

11. **Trait name / entity context collision.** Both entity context handlers and trait handlers use bare effect names, following the `constantHandler` pattern. Priority is enforced via `composeHandlers` ordering: den context handlers are composed first, so they take precedence on name collision. `wrapClassModule` checks trait names AFTER den context args for the same reason. A trait named `host` would be unreachable in contexts where a `host` entity context arg exists — this is unlikely and effectively prohibited by convention.

## Provide-To Unification

Traits subsume the labeled semantic data path of provide-to. Since the policies spec is not yet merged, this unification should be implemented directly rather than incurring tech debt.

### The overlap

The provide-to labeled data pattern (`provide-to.http-backends = [...]`) and traits do the same thing: collect structured data from multiple aspects and make it available to consumers via parametric args. The only difference is scope — provide-to routes cross-entity, traits stay within one entity's pipeline.

| | Traits | Provide-to labeled data |
|---|---|---|
| **Collection** | `traitCollectorHandler` → `state.traits` | `provideToHandler` → `state.provideTo` |
| **Distribution** | Stateful handler reads live state | `distributeProvideTo` → `constantHandler` |
| **Consumer syntax** | `{ firewall, ... }:` | `{ http-backends, ... }:` |
| **Declaration** | `den.traits` with validation | Freeform labels, no validation |
| **Tiering** | 3 tiers | Static only |

Two parallel collection/distribution systems for the same pattern is unnecessary.

### What changes

**`provide-to` structural key on aspects — removed.** Aspects no longer declare `provide-to.http-backends = [...]`. Instead, they emit traits:

```nix
# Before (provide-to labeled data):
den.aspects.example-site = { host, ... }: {
  provide-to.http-backends = [
    { address = host.meta.ip; port = 8080; vhost = "example.com"; }
  ];
};

# After (trait emission):
den.aspects.example-site = { host, ... }: {
  http-backends = [
    { address = host.meta.ip; port = 8080; vhost = "example.com"; }
  ];
};
# With: den.traits.http-backends = { collection = "list"; };
```

The aspect declares what it IS (an http backend) without any awareness of cross-entity routing. The policy controls where the data flows.

**`provide-to` effect — kept but payload changes.** The `provideToHandler` and `"provide-to"` effect remain as the transport mechanism for cross-entity data. But the payload changes from raw labeled data to trait state:

```nix
# Transition handler for sibling routes:
# Runs sub-pipeline for each peer, captures trait state, emits provide-to
fx.send "provide-to" {
  targetEntity = peer;
  traits = subPipelineState.traits;  # captured from sub-pipeline
}
```

**`distributeProvideTo` — becomes trait-aware.** Instead of installing labeled data as `constantHandler` bindings, distribution collects cross-entity trait data per target:

```nix
distributeCrossEntityTraits =
  emissions:
  let
    grouped = groupByTarget emissions;
  in
  lib.mapAttrs (_targetId: traitSets:
    lib.foldl' (acc: traits:
      let allNames = lib.unique (builtins.attrNames acc ++ builtins.attrNames traits);
      in lib.genAttrs allNames (name:
        (acc.${name} or []) ++ (traits.${name} or [])
      )
    ) {} traitSets
  ) grouped;
```

**Cross-entity trait data is injected at module eval time, not pipeline time.** The target's pipeline already ran in phase 1 — phase 2 does NOT re-run it. Cross-entity trait data is merged into the target's `_den.traits` injection module alongside within-entity Tier 1/2 and Tier 3 data:

```nix
config._den.traits.http-backends =
  pipelineResolvedData            # within-entity Tier 1/2
  ++ deferredEmissions             # within-entity Tier 3
  ++ crossEntityData;              # from peers via phase 2
```

This means cross-entity traits are available to class module consumers (`{ http-backends, config, ... }:`) but NOT to pipeline-time discriminators. This is correct: cross-entity data can't drive pipeline-time decisions because the source's pipeline may not have run yet when the target's discriminators fire. The constraint is structural, not a limitation — if you need pipeline-time branching on cross-entity data, use a policy with explicit handling.

**`structuralKeysSet` — remove `"provide-to"`.** Since aspects no longer declare `provide-to.*` keys, it doesn't need to be a structural key. The `"provide-to"` effect is emitted by the transition handler, not by `compileStatic`.

### Cross-entity examples rewritten

**Fleet `/etc/hosts`:**

```nix
# Trait declaration
den.traits.fleet-hosts = {
  description = "Host identity for fleet /etc/hosts generation";
  collection = "list";
};

# Every host emits its identity as a trait — no cross-entity awareness
den.aspects.fleet-member = {
  fleet-hosts = { host, ... }: { ip = host.meta.ip; name = host.name; };
};

# Policy routes fleet-hosts trait data to peers
den.policies.host-to-peers = {
  from = "host"; to = "host"; as = "peer";
  resolve = { host }:
    map (peer: { inherit peer; })
      (filter (h: h.name != host.name) (attrValues den.hosts.${host.system}));
};

# Consumer aspect generates NixOS config from trait data
den.aspects.fleet-hosts-config = {
  nixos = { fleet-hosts, ... }: {
    networking.extraHosts = lib.concatMapStrings
      (h: "${h.ip} ${h.name}\n") fleet-hosts;
  };
};
```

**Haproxy backends:**

```nix
# Trait declaration
den.traits.http-backends = {
  description = "HTTP backend endpoints for load balancing";
  collection = "list";
};

# Source aspects emit trait data
den.aspects.example-site = { host, ... }: {
  nixos.services.nginx.virtualHosts."example.com".locations."/".proxyPass =
    "http://localhost:8080";
  http-backends = [
    { address = host.meta.ip; port = 8080; vhost = "example.com"; }
  ];
};

# Target aspect consumes trait data — same syntax as within-entity
den.aspects.loadbalancer = {
  nixos = { http-backends, config, ... }: {
    services.haproxy.enable = true;
    services.haproxy.frontends = lib.listToAttrs (map (b: {
      name = b.vhost;
      value.backends = [{ inherit (b) address port; }];
    }) http-backends);
  };
};
```

### What stays

- **`provideToHandler`** — transport mechanism, payload carries trait state
- **Phase 1/Phase 2 architecture** — two-phase resolution is fundamental to cross-entity routing
- **Cycle prevention** — phase 2 does not re-run pipelines, preventing cycles
- **Class module cross-entity routing** — unchanged. The transition handler's sibling routing captures sub-pipeline `state.imports` for class module injection into the target's module list. This is separate from trait distribution and continues to work as before

### Migration from provide-to labeled data

Since the policies spec is not yet merged, this is a clean replacement — no deprecation needed. The `provide-to` structural key, `resolveChildren`'s provide-to emission logic (aspect.nix lines 496-515), and the `constantHandler`-based installation in `distribute-provide-to.nix` are replaced by trait-based equivalents.

The `distribute-provide-to.nix` file is renamed/refactored to handle cross-entity trait distribution. Its `groupByTarget` logic is preserved; `mkProvideToHandlers` changes from `constantHandler` installation to `state.traits` merging.

## Follow-up Work

Deferred items that build on this design but are not part of Phase 1–4.

### `hasTrait` introspection

**Problem:** `hasAspect` checks whether an aspect is included but cannot inspect trait data. Users want to conditionally include aspects based on whether a trait has emissions (e.g., "include impermanence setup only if any aspect emits persistence paths").

**Spike:** Add a `hasTrait` predicate to the pipeline that checks `state.traits.${name} != []` at pipeline time (Tier 1/2 only). Expose as a parametric arg or a new effect. Should compose with existing `includeIf` patterns.

### Derived / computed traits

**Problem:** Related traits require redundant emissions. If an aspect emits `persistence` paths, a `bind-mounts` trait should be derivable automatically without every aspect re-emitting.

**Spike:** Add a `den.traits.*.derivedFrom` field that references another trait and a transform function: `derivedFrom = { trait = "persistence"; transform = paths: map mkBindMount paths; }`. The pipeline computes derived traits after collecting the source trait. Tier follows the source — if `persistence` is Tier 1, `bind-mounts` is also Tier 1.

### Trait attenuation

**Problem:** An intermediate aspect may want to expose a restricted view of its dependency's traits — hiding implementation details while forwarding relevant data.

**Spike:** Add an optional `meta.traitFilter` on aspects that intercepts trait emissions from child includes before they propagate upward. The filter is a function `traitName -> emissions -> emissions` applied during the include walk. Uses the existing `chainHandler` scope to determine which aspect owns the filter.

### Diagram visualization of traits

**Problem:** The `diag` library visualizes `emit-class` and `emit-include` effects but not trait emissions. Trait data flow is invisible in generated diagrams.

**Spike:** Extend `structuredTrace` entries to include `emit-trait` events with trait name, tier, and provenance chain. Add a `traitFlow` view to `views.extended` that renders trait-specific edges (producer → trait → consumer) as a separate subgraph or overlay on the aspect hierarchy. The `chain` field on `emit-trait` already carries the provenance needed for edge generation.

---

## Appendix A: Prior Art

### Closest parallels

**Bazel providers and aspects.** Bazel rules return named, typed data bundles (providers) that downstream rules consume by name. Bazel aspects propagate transitively along the dependency graph, running on each target and emitting additional providers. This is the closest prior art to Den's trait model — graph-propagated structured data with declared schemas. Den's tiering adds a dimension Bazel lacks (Bazel has a single analysis phase, no fixpoint evaluation).

**Dagger 2 multibindings** (`@IntoSet`, `@IntoMap`). Multiple Dagger modules contribute elements to a collected `Set<T>` or `Map<K,V>` that consumers inject. Compile-time validation ensures all contributions and consumers are satisfiable. Den's `collection = "list"` vs `"map"` on trait schemas directly borrows this distinction. Dagger's default of error-on-duplicate informed Den's conflict semantics.

**NixOS module system fixpoint.** Den's Tier 3 IS the NixOS fixpoint. The entire `_den.traits` injection module pattern works because `evalModules` computes a lazy fixpoint where all modules see the final `config`. The tiering (Tier 1/2 outside fixpoint, Tier 3 inside) directly addresses the infinite-recursion pitfall that `specialArgs` was created to work around.

**impermanence.** The direct inspiration. Multiple NixOS modules append to `environment.persistence.*.directories` (a list option), and the impermanence module reads the merged list to configure bind mounts. This is trait collection using the NixOS module system's built-in list merging — entirely Tier 3, no graph awareness, no tiering. Den generalizes this pattern with Tier 1/2 pipeline-time availability and cross-aspect graph propagation.

### Relevant patterns from type systems

**Rust traits / Haskell typeclasses.** Both enforce global coherence (at most one implementation per type). Rust's orphan rule prevents implementing a foreign trait for a foreign type. Den's trait model is deliberately less strict — it collects from multiple emitters — but the coherence lesson applies: when `collection = "map"`, duplicate keys should error, and error messages should include full provenance.

**Scala given instances.** Consumers declare what they need by type; the compiler resolves and injects it. The ambiguity-is-an-error rule is a coherence mechanism. Den resolves by name rather than type, but the principle holds: when resolution is ambiguous, fail loudly.

**Swift conditional conformance.** A generic type conforms to a protocol only when its type parameters satisfy constraints. Den's pipeline-time discriminators (`{ firewall, ... }: lib.optionalAttrs (hasHttps) { includes = [...]; }`) are a direct application of this pattern — conditional emission based on available trait data.

### Relevant patterns from DI/IoC

**Spring component scanning.** Discovers beans on the classpath, consumers inject `List<SomeInterface>` to receive all beans of a type. `@Order` controls collection order, `@Conditional` controls registration. Den's trait collection is the same pattern with graph-awareness added.

**Guice multibinder.** `Multibinder.newSetBinder` allows multi-module contribution to a collected set. `permitDuplicates()` is an explicit opt-in to relaxed coherence. Den's `collection = "list"` is implicitly permit-duplicates (list concat); `collection = "map"` defaults to error-on-duplicate.

### Design principles derived from survey

1. **Declare schemas up front** — Bazel providers, Dagger multibindings, and NixOS options all require declaring the shape of contributed data. Den traits should have declared schemas (via `den.traits`).
2. **Error on conflict by default** — Haskell coherence, Dagger, and Guice all default to error when there's ambiguity. Den follows this for `collection = "map"` traits.
3. **Include full provenance in errors** — the #1 frustration with NixOS option merging is "defined in multiple places" without adequate attribution. Den trait emissions carry `state.includesChain` from `chainHandler`, so merge errors say "firewall trait emitted by nginx (included via igloo → server → nginx) conflicts with emission from postgres (included via igloo → database → postgres)."
4. **Make propagation discoverable** — AspectJ's experience shows that "aspects adding things to targets" creates comprehensibility problems. Den should make trait contributions introspectable (via `diag` library and `policyInspect`-style tooling).
5. **Tiering is validated by multiple systems** — Bazel's loading/analysis/execution, NixOS's `specialArgs` vs `config`, Scala's compile-time vs runtime. Den's three-tier model is more explicit and fine-grained, which is an advantage.

### Future extensions informed by prior art

- **Derived traits** (from Rust blanket implementations): "any aspect providing trait X automatically also provides trait Y, computed from X's data." Avoids redundant emissions and ensures consistency across related traits.
- **Trait attenuation** (from capability model): intermediate aspects filtering or restricting trait data as it flows through the graph, supporting encapsulation.

Note: **propagation edge specification** (from Bazel) is not needed — traits fire along all edges, and existing parametric scope filtering already lets consumers ignore irrelevant data. **Tier visibility mismatch** is addressed in Design Decision 5 (it's an error, not a warning).

## Appendix B: Aspect Content Type — Design Rationale

### The chicken-and-egg problem

The previous aspect freeform type (`lazyAttrsOf deferredModule`) forced all non-structural values through module-shaped type checking. Traits (plain data, lists, functions returning data) don't conform to module shape, requiring a type that can accept diverse value forms.

A schema-aware freeform type that dispatches based on `den.classes` / `den.traits` creates a timing problem: both registries are `config` values populated during module evaluation, but the freeform type's `check` and `merge` functions execute during that same evaluation. The type needs the schema to route values, but the schema is part of what's being evaluated.

### Solution: generic wrapper, pipeline classifies

The `aspectContentType` accepts any value and wraps it uniformly with identity metadata (`__provider`). Semantic classification (class vs trait vs nested aspect) and structural classification (parametric vs static) are deferred to the pipeline's `compileStatic`, which runs after the module system has fully resolved the schema registry.

This aligns with the existing architecture: `compileStatic` is already the classification entry point. The type system's role is to **preserve values and identity** (via `__provider`), not to interpret them.

### Relationship to `providerType`

The existing `provides` option uses `lazyAttrsOf (providerType typeCfg)`, where `providerType` handles nested aspects with identity tracking via `typeCfg.providerPrefix`. With `aspectContentType` as the aspect-level freeform type, all non-structural values carry the same `__provider` identity path. This makes `provides` structurally redundant — direct nesting on the aspect gets identical identity tracking.

`providerType`'s dispatch logic (parametric wrapper detection, function vs attrset merge) remains relevant for `includes` and can inform `compileStatic`'s handling of wrapped values. The merge strategies are complementary: `aspectContentType` preserves all definitions for the pipeline; `compileStatic` applies the appropriate merge strategy per classification.

### Multi-site definition preservation

The merge function collects all definitions with file provenance via `map (d: { inherit (d) value file; }) defs` rather than last-wins. This is critical for class modules, where multiple files may contribute to the same class key:

```nix
# file1: den.aspects.foo.nixos.services.bar.enable = true;
# file2: den.aspects.foo.nixos.services.baz.enable = true;
# → freeform key "nixos" gets two defs, both must survive as module imports
```

For traits, the pipeline applies the trait's `collection` strategy. For nested aspects, the pipeline recurses. The type layer preserves all information; the pipeline decides the merge semantics.
