# Trait System Design

**Date:** 2026-05-04
**Status:** Design (pre-implementation)
**Depends on:** Provides removal (consolidated spec Section 5), unified aspect key type (consolidated spec Section 8)
**Supersedes:** `implemented-branch/traits/2026-04-25-traits-and-structural-nesting-design.md` (deleted implementation)

## 1. Problem

Den aspects emit content that falls into three semantic categories:

- **Class modules** (`nixos`, `homeManager`) — functions evaluated by an external module system (`evalModules`), becoming part of `config`
- **Semantic data** (`firewall`, `persistence`) — structured data collected across aspects and consumed by other aspects or class modules
- **Nested aspects** (`tux`, `node-exporter`) — sub-aspects scoped to a named entity or feature

Today, all non-structural keys are emitted as class modules. There is no mechanism to:

1. **Collect structured data across aspects.** An nginx aspect and a postgres aspect both know what firewall ports they need, but there is no way to aggregate `[ { ports = [80 443]; } { ports = [5432]; } ]` and hand that to a firewall-configuration aspect. Each must independently set `networking.firewall.allowedTCPPorts` in a class module, with no coordination and no ability to reason about the aggregate.

2. **Branch on semantic data at pipeline time.** A parametric discriminator cannot currently say "include TLS hardening if any included aspect declares port 443." The data needed for that decision is locked inside class modules that only resolve inside `evalModules`.

3. **Distinguish semantic intent.** `firewall = { ports = [80]; }` is not a NixOS module. Treating it as one forces awkward encoding and prevents the pipeline from applying appropriate collection semantics (list concatenation, map merge, validation).

Traits solve all three by introducing a schema-driven data channel parallel to class emission.

## 2. Schema Registry

### 2.1 `den.traits` Option

A top-level option declaring semantic data channels. Lives at `den.traits` (not under `den.schema`) to avoid a module evaluation cycle — `den.schema` contains entity-kind entries that aspects read, and nesting trait registries inside it creates a cycle through aspect submodule evaluation.

```nix
den.traits.firewall = {
  description = "Firewall port and interface declarations";
  collection = "list";   # "list" (default), "map", or "single"
};
den.traits.persistence = {
  description = "Impermanence path declarations";
  collection = "list";
};
den.traits.service-endpoints = {
  description = "Named service endpoints for discovery";
  collection = "map";   # keyed by top-level attrset keys; duplicates error
};
```

**Collection strategies:**

| Strategy | Merge behavior | Use case |
|----------|---------------|----------|
| `list` (default) | Concatenate all emissions into a list | Firewall ports, persistence paths — additive aggregation |
| `map` | Merge top-level attrset keys; duplicate keys error with full provenance | Service endpoints, DNS records — keyed registries |
| `single` | Exactly one emission permitted; multiple emissions error | Hostname declaration, primary interface |

**Optional fields:**
- `description` — human-readable purpose
- `type` — NixOS-style type for validation of each emission (default: no validation)

**Collision with classes:** A key cannot be both a class and a trait. The schema module asserts `builtins.intersectAttrs den.classes den.traits == {}` and errors with full provenance on collision.

### 2.2 Aspect-Level Declaration

Aspects declare the traits they emit via `aspect.traits`, a structural key on the aspect submodule (like `policies`, `meta`, `includes`):

```nix
den.aspects.networking = {
  traits.firewall = {
    description = "Firewall port declarations";
    collection = "list";
  };

  nixos.networking.firewall.enable = true;
  firewall = { ports = [ 80 443 ]; };
};
```

Aspect-level `traits` declarations merge into `den.traits` via the module system (config merging), not pipeline-time mutation. The `traits` key is an explicit option on `aspectSubmodule`, and its values merge into `den.traits` during module evaluation — before the pipeline runs. Consequences:

- Aspect authors do not need to tell consumers "also add `den.traits.firewall`"
- Multiple aspects can declare the same trait — declarations must be compatible (same `collection`, same `type`)
- Namespaced aspects carry their trait declarations — importing the namespace brings both aspects and trait schemas
- `den.traits` remains the canonical registry; aspect-level declarations merge into it

Phase 1 adds `"traits"` to `structuralKeysSet` so it is excluded from freeform content classification.

## 3. Classification: Three-Branch Dispatch

`key-classification.nix` currently implements two-branch dispatch (registered class vs default-to-class). Traits add a third branch:

```
For each non-structural key k on an aspect:

  1. k in den.classes       -> ClassModule  (emit-class, existing behavior)
  2. k in den.traits        -> SemanticData (emit-trait, new effect)
  3. k not in either        -> ClassModule  (emit as class — backwards-compatible default)
```

**Branch 3 rationale:** Unregistered keys emit as classes, not as errors or trace warnings. This preserves backwards compatibility (all existing aspects continue to work) and eliminates the need for a DLQ. The original implementation's DLQ (dead letter queue for unregistered keys) was built and then deleted during the cleanup arc. The direct-emit approach is simpler and correct: if a key is not declared as a trait, it is a class module.

**Dynamic registration changes branch 3 behavior.** If an aspect registers a trait schema via `register-trait-schema` during the tree walk, subsequent encounters of that key in other aspects will hit branch 2 instead of branch 3. This is how battery aspects work — the first aspect in the include tree that declares `traits.firewall` causes all subsequent `firewall` keys to be collected as trait data.

### 3.1 Integration with `aspectKeyType`

The `aspectKeyType` in `types.nix` (currently a placeholder with dead two-branch dispatch) becomes the type-level support for three-branch classification. The type layer remains generic — it wraps every non-structural value uniformly with identity metadata (`__contentValues`, `__provider`). Semantic classification happens at pipeline time in `classifyKeys`, not in the type system. This avoids the chicken-and-egg problem where the type system needs schema that is itself a config value being evaluated.

## 4. Collection: `emit-trait` Effect and Handler

### 4.1 The `emit-trait` Effect

When `classifyKeys` encounters a trait key, it emits one `emit-trait` per definition:

```nix
fx.send "emit-trait" {
  trait = "firewall";
  value = singleValue;
  chain = state.includesChain;
}
```

Multi-site definitions (same trait key set in multiple files) fan out into individual emissions, each with its own provenance chain.

### 4.2 `traitCollectorHandler`

A handler that accumulates trait data in scope-partitioned pipeline state:

```nix
traitCollectorHandler = {
  "emit-trait" = { param, state }:
    let
      inherit (param) trait value chain;
      schema = state.traitSchemas.${trait} or { collection = "list"; };
      scopeId = state.currentScopeId;
      existing = state.scopedTraits.${scopeId}.${trait} or [];
      entry = { inherit value chain; };
    in
    {
      resume = null;
      state = state // {
        scopedTraits = state.scopedTraits // {
          ${scopeId} = (state.scopedTraits.${scopeId} or {}) // {
            ${trait} = existing ++ [ entry ];
          };
        };
      };
    };
};
```

**Scope partitioning.** Trait data is stored in `scopedTraits.${scopeId}.${traitName}`, mirroring how class imports use `scopedClassImports.${scopeId}`. Each scope (host scope, user scope, forward scope) accumulates trait data independently.

**State field:** `scopedTraits` is thunk-wrapped (`_: value`) to survive `builtins.deepSeq` on the trampoline, matching the pattern used for `scopedClassImports` and other accumulated state.

**No tier detection in the collector.** Unlike the original design's three-tier model, the simplified pipeline has ONE tier: pipeline-time. All trait values are resolved at pipeline time. Static values are collected directly. Parametric values (`{ host, ... }: { ports = [...]; }`) are resolved via `scope.provide` context before reaching the collector — the parametric resolution in `compileStatic` / `bind.fn` handles this before the `emit-trait` effect fires. There is no Tier 3 (module-evaluated) in this design; that use case is handled by fleet + `den.exports` (Section 9).

### 4.3 Collection Strategy Application

Collection strategies are applied when trait data is consumed, not at emission time. The collector always appends to a list. The consumption handlers apply the strategy:

- **`list`**: Return the list of values directly.
- **`map`**: Fold the list into a single attrset. Duplicate top-level keys error with full provenance (chain fields identify the conflicting emitters).
- **`single`**: Assert length == 1. Zero emissions returns `null`. Multiple emissions error with provenance.

## 5. Consumption

### 5.1 Pipeline-Time: `{ traitName, ... }:` Discriminators

Trait data is available to parametric discriminators (in `includes`, in aspect bodies) via the same `bind.fn` mechanism used for entity context. A `traitArgHandler` installs bare effect names matching registered trait names:

```nix
traitArgHandler = traitSchemas:
  lib.mapAttrs (traitName: schema: {
    handler = { param, state }:
      let
        scopeId = state.currentScopeId;
        raw = state.scopedTraits.${scopeId}.${traitName} or [];
        inherited = inheritTraits { inherit state traitName; };
        allEntries = inherited ++ raw;
        values = map (e: e.value) allEntries;
        applied = applyCollectionStrategy schema values;
      in
      {
        resume = applied;
        state = state;
      };
  }) traitSchemas;
```

**Handler reads live state.** Unlike `constantHandler` (which snapshots a value), `traitArgHandler` reads from `state.scopedTraits` at each point of use. Consumers see all trait data collected up to that point in the depth-first walk.

**Priority ordering.** Den context handlers (`host`, `user`) are composed first, trait handlers second. `composeHandlers` gives the first handler priority on name collision. A trait named `host` would be unreachable in contexts where a `host` entity context arg exists — this is effectively prohibited by convention.

**Example — conditional inclusion based on trait data:**

```nix
den.aspects.secure-server = {
  includes = [
    ({ firewall, ... }:
      let hasHttps = builtins.any (f: builtins.elem 443 (f.ports or [])) firewall;
      in lib.optionalAttrs hasHttps {
        includes = [ den.aspects.tls-hardening den.aspects.acme ];
      }
    )
  ];
};
```

### 5.2 Module-Time: `_den.traits` Lazy Thunk

Class modules access trait data via `_den.traits`, a generated NixOS option injected into the `evalModules` call:

```nix
# Generated by pipeline, invisible to user:
{ lib, ... }: {
  options._den.traits = lib.mkOption {
    type = lib.types.attrsOf lib.types.anything;
    default = {};
    internal = true;
  };
  config._den.traits = collectedTraitData;  # pipeline-resolved data per scope
}
```

**Delivery through `wrapClassModule`.** Trait args in class modules are delivered via the existing `wrapClassModule` partial application mechanism. When a class module's function signature includes a registered trait name, `wrapClassModule` pre-applies a lazy thunk that reads from `moduleArgs.config._den.traits`:

```nix
# wrapClassModule detects traitArgNames in the function signature:
traitArgNames = lib.filter (k: traitSchemas ? k) (builtins.attrNames moduleArgNames);

# Pre-applies a thunk for each trait arg:
wrapper = moduleArgs:
  module (moduleArgs // denArgs // lib.genAttrs traitArgNames (name:
    moduleArgs.config._den.traits.${name} or (
      if schema.collection == "list" then []
      else if schema.collection == "map" then {}
      else null
    )
  ));
```

Nix's lazy evaluation means the thunk is only forced when the consumer accesses the trait arg, at which point `config._den.traits` contains the fully-collected pipeline data.

**Consumer syntax — identical to pipeline-time:**

```nix
den.aspects.hardened-server = {
  nixos = { firewall, persistence, config, ... }: {
    networking.firewall.allowedTCPPorts =
      lib.concatMap (f: f.ports or []) firewall;
    environment.persistence."/persist".directories =
      lib.concatMap (p: p.directories or []) persistence;
  };
};
```

Both parametric discriminators and class modules use `{ traitName, ... }:`. The difference is when they run: discriminators at pipeline time, class modules at eval time. Data is the same — there is no tier distinction in the simplified design.

## 6. Dynamic Registration: `register-trait-schema` Effect

Battery aspects that are self-contained (declaring both the trait schema and the aspects that emit/consume it) use dynamic registration:

```nix
den.aspects.networking = {
  # Aspect-level declaration (merged into den.traits at module-eval time)
  traits.firewall = {
    description = "Firewall port declarations";
    collection = "list";
  };

  nixos.networking.firewall.enable = true;
  firewall = { ports = [ 80 443 ]; };
};
```

For cases where the trait schema must be registered dynamically during the tree walk (e.g., a battery aspect included via a deferred include where module-eval-time registration is not available), the `register-trait-schema` effect mutates `state.traitSchemas`:

```nix
registerTraitSchemaHandler = {
  "register-trait-schema" = { param, state }:
    let
      inherit (param) name schema;
      existing = state.traitSchemas.${name} or null;
      # Validate compatibility if already registered
      compatible = existing == null
        || (existing.collection == schema.collection
            && (existing.type or null) == (schema.type or null));
    in
    assert compatible
      || throw "den: trait '${name}' registered with incompatible schema"
               + " (existing: collection=${existing.collection}"
               + ", new: collection=${schema.collection})";
    {
      resume = null;
      state = state // {
        traitSchemas = state.traitSchemas // {
          ${name} = schema;
        };
      };
    };
};
```

**No DLQ interaction.** If `classifyKeys` encounters a key before its trait schema is registered, the key emits as a class (branch 3). Once the schema is registered, subsequent encounters of that key in other aspects hit branch 2 (SemanticData). This is a natural consequence of depth-first tree walk ordering — the declaring aspect should appear in the include tree before emitting aspects. If ordering cannot be guaranteed, use module-eval-time registration (`aspect.traits`) instead.

**Initialization.** `state.traitSchemas` is initialized from `den.traits` at pipeline construction time. Dynamic registrations add to this set during the walk.

## 7. Scope Inheritance

Child scopes see parent trait data via scope inheritance, following the `scopeParent` tree:

```nix
inheritTraits = { state, traitName }:
  let
    scopeId = state.currentScopeId;
    parentId = state.scopeParent.${scopeId} or null;
    parentEntries =
      if parentId == null then []
      else
        let parentData = state.scopedTraits.${parentId}.${traitName} or [];
        in parentData ++ (inheritTraits {
          inherit state traitName;
          # Walk up the scope tree by switching currentScopeId
          state = state // { currentScopeId = parentId; };
        });
  in
  parentEntries;
```

**Collection strategy interaction with inheritance:**

- **`list`**: Parent entries concatenated before child entries. Consumer sees the full aggregated list.
- **`map`**: Parent and child entries merged. A child key shadows a parent key (child wins), matching NixOS module system precedence semantics.
- **`single`**: Child emission shadows parent. If neither child nor parent emits, result is `null`.

**Example:** A host scope declares `firewall = { ports = [22]; }`. A user scope nested within it declares `firewall = { ports = [8080]; }`. A consumer in the user scope requesting `{ firewall, ... }:` receives `[ { ports = [22]; } { ports = [8080]; } ]` (list collection — both entries visible).

This replaces the original design's cross-entity trait distribution via provide-to. Scope inheritance via `scopeParent` is simpler and leverages the existing scope-partitioned state infrastructure.

## 8. Integration with `aspectKeyType`

The `aspectKeyType` placeholder in `types.nix` (~line 293) currently has dead two-branch dispatch. The three-branch classification extends it:

```
aspectKeyType dispatch:
  1. k in classRegistry   -> ClassModule   (existing)
  2. k in traitRegistry   -> SemanticData  (new)
  3. k not in either      -> ClassModule   (default, existing fallback)
```

The `SemanticData` branch in `classifyKeys` emits `emit-trait` instead of `emit-class`. All other pipeline mechanics (include chain tracking, dedup via `includeSeen`, scope partitioning) continue to operate on the class branch unchanged.

**`emitTraitSchemas` placement in `resolveChildren`.** Before `classifyKeys` runs for an aspect's content, `resolveChildren` in `aspect.nix` calls `emitTraitSchemas` to process any `aspect.traits` declarations. This ensures dynamic registrations from the current aspect are visible to the classification of its own keys:

```
resolveChildren order (in aspect.nix):
  1. emitTraitSchemas   -- register trait schemas from aspect.traits
  2. classifyKeys       -- three-branch dispatch using updated traitSchemas
  3. emitClasses        -- emit class modules (branch 1 + 3)
  4. emitTraits         -- emit trait data (branch 2)
  5. resolveIncludes    -- recurse into child aspects
```

## 9. What Fleet + `den.exports` Handles Instead

The original traits spec described three evaluation tiers:

| Tier | Original scope | Simplified design |
|------|---------------|-------------------|
| **1. Static** | Pipeline-time plain values | Traits (this spec) |
| **2. Pipeline-parametric** | Pipeline-time functions of den context | Traits (this spec) — resolved via `scope.provide` before emission |
| **3. Module-evaluated** | Inside `evalModules` fixpoint | Fleet + `den.exports` (separate system) |

**Why Tier 3 is out of scope.** Module-evaluated data (functions requiring `config`, `pkgs`, `lib`) cannot drive pipeline-time decisions. The original design injected Tier 3 data into `_den.traits` via a generated module, creating complexity around:
- Self-referential recursion guards
- `partialOk` validation (pipeline consumer + deferred emitter = error)
- Mixed-arg pre-application in `wrapClassModule`
- Thunk management across the trampoline

Fleet + `den.exports` handles this case more naturally:

```nix
# Host exports NixOS-config-dependent data:
den.exports.nginx-user = { config, ... }: config.services.nginx.user;

# Fleet policy aggregates across hosts:
den.fleet.http-backends = { hosts, ... }:
  lib.concatMap (h: h.exports.http-backends or []) (lib.attrValues hosts);
```

This uses Nix's lazy evaluation directly — each host's `exports` are thunks evaluated inside that host's `evalModules` fixpoint. Fleet aggregation happens outside any single host's fixpoint, avoiding the self-referential recursion problem entirely.

**Cross-host data** (the haproxy backends, fleet `/etc/hosts` examples from the original spec) are also fleet + `den.exports` territory. Traits in this spec are scoped to a single pipeline run. Cross-host aggregation requires a mechanism that sees multiple hosts' resolved configurations, which is what fleet provides.

The boundary is clean: traits handle pipeline-time structured data within a single entity resolution. Fleet handles cross-entity and module-evaluated data.

## 10. Migration

Traits implementation depends on two prerequisite changes and proceeds in three phases.

### Prerequisites

1. **Provides removal** (consolidated spec Section 5). The `provides` structural key must be removed so that direct nesting on aspects works. Without this, `aspectKeyType` cannot dispatch nested aspect keys correctly — the `provides` structural key intercepts them before classification.

2. **Unified aspect key type** (consolidated spec Section 8). The two-branch `aspectKeyType` placeholder must be upgraded to three-branch dispatch. This is a small change to `key-classification.nix` and `types.nix` that can land alongside or immediately before traits.

### Phase 1: Schema Registry (Additive)

- Add `den.traits` option with `collection` field and optional `type`
- Add `traits` structural key to `aspectSubmodule` (merges into `den.traits`)
- Add `"traits"` to `structuralKeysSet`
- Add collision assertion (`den.classes` intersect `den.traits` must be empty)
- Initialize `state.traitSchemas` from `den.traits` at pipeline construction

No behavioral change. All existing keys continue to emit as classes.

### Phase 2: Classification + Collection

- Add `SemanticData` branch to `classifyKeys` in `key-classification.nix`
- Add `emit-trait` effect
- Add `traitCollectorHandler` with scope-partitioned state (`scopedTraits.${scopeId}`)
- Add `register-trait-schema` effect and handler
- Add `emitTraitSchemas` call in `resolveChildren` (aspect.nix), before `classifyKeys`

Aspects that declare `traits.X` and emit key `X` now get trait collection instead of class emission for that key. Aspects without trait declarations are unchanged.

### Phase 3: Consumption

- Add `traitArgHandler` to handler composition (after entity context handlers)
- Update `wrapClassModule` to detect trait arg names and pre-apply lazy thunks
- Add `_den.traits` injection module to `evalModules` calls
- Add scope inheritance via `scopeParent` tree walking

Full producer-to-consumer flow operational. Parametric discriminators can branch on trait data. Class modules receive trait data via the existing partial application mechanism.

### Estimated Scope

Based on the original implementation (which shipped and passed 633 tests before deletion):

- **Phase 1:** ~50 lines (option declarations, structural key, assertion)
- **Phase 2:** ~120 lines (handler, effect, classification branch, dynamic registration)
- **Phase 3:** ~80 lines (traitArgHandler, wrapClassModule update, injection module, inheritance)
- **Total:** ~250 lines, vs ~2463 lines in the original implementation

The reduction comes from: no DLQ (~178 lines), no provide-to unification (~400+ lines), no Tier 3 deferred evaluation (~300+ lines), no transition handler interaction (~200+ lines), no separate dispatch-policies interaction (~300+ lines), simpler scope model (scope.provide eliminates stack management).

## Design Decisions

### D1. One pipeline tier, not three

The original design detected three tiers via `builtins.functionArgs`: static (plain values), pipeline-parametric (den context args only), and module-evaluated (has `config`/`pkgs`/etc.). The simplified pipeline collapses tiers 1 and 2 — `scope.provide` resolves parametric functions before they reach `emit-trait`, so the collector only ever sees resolved values. Tier 3 is handled by fleet + `den.exports` (Section 9).

This eliminates: tier detection logic, `deferredTraits` state, `partialOk` validation, `consumedTraits` tracking, and the `_den.traits` injection module's dual-source merge. The `traitCollectorHandler` becomes ~30 lines instead of ~80.

### D2. No DLQ

Unregistered keys emit as classes (branch 3 of classification). This is the existing behavior and maintains backwards compatibility. The DLQ was implemented, tested, and then deleted because it added complexity without solving a real problem — templates that define custom classes just need `den.classes.X = {}` registrations, which they already have.

### D3. No provide-to for cross-entity traits

The original design routed cross-entity trait data through the provide-to mechanism (sub-pipeline captures traits, distribute to targets). This was deleted because: (a) provide-to itself was deleted, (b) scope inheritance via `scopeParent` handles cross-scope data within a pipeline, (c) fleet + `den.exports` handles cross-host data more naturally.

### D4. Scope-partitioned collection

Trait data is stored in `scopedTraits.${scopeId}.${traitName}`, not a flat `state.traits`. This matches the scope-partitioned state architecture (using `mkScopeId`, `scopeParent`, etc.) and provides natural isolation between entity scopes within a single pipeline run.

### D5. Collection strategies applied at consumption, not emission

The collector always appends to a list. Strategy application (`list` concat, `map` merge, `single` assertion) happens in `traitArgHandler` when a consumer requests the data. This keeps the collector simple and allows the same collected data to be presented differently if the schema is upgraded (though schema changes mid-pipeline are not expected).

### D6. `traitArgHandler` reads live state, not snapshots

The original implementation identified a bug where `traitCollectorHandler` captured root `ctx` at construction time, causing stale context for Tier 2 resolution. The fix — reading from `state.scopeContexts.${currentScope}` inside the handler body — applies equally to `traitArgHandler`. It reads `state.scopedTraits` at each invocation, so consumers see all data collected up to that point in the walk.

### D7. Dynamic registration is optional, not primary

Most trait schemas are declared via `den.traits` (module-eval-time) or `aspect.traits` (also module-eval-time, merged into `den.traits`). The `register-trait-schema` effect exists for edge cases where a battery aspect is only reachable via deferred includes. The primary path is static registration.
