# FX Pipeline Refactor — Consolidated Spec

**Date:** 2026-05-02
**Branch:** feat/fx-pipeline
**PR:** #475
**CI:** 629/629
**Scope:** Refactor narrative, current architecture, remaining work, forward plan

## 1. Introduction

This document consolidates 16 design specs written during the second half of the fx-pipeline refactor — the cleanup arc that eliminated accumulated tech debt and design missteps from the initial implementation. It covers:

- The architectural transformation from the fx-pipeline midpoint to present
- Design invariants and constraints that future work must respect
- A component inventory mapping midpoint → current state
- Provides as permanent API (Section 5, updated 2026-05-04)
- Next work (forward elimination, aspect key type, cleanup)
- A vestigial code audit identifying suspected dead paths
- Policy scoping and activation redesign (registry vs activation, meta.handleWith → policy methods)
- Future traits reimplementation via fleet + den.exports

**What this document does NOT cover:** The initial main→fx-pipeline transformation (replacing the legacy eager resolver with algebraic effects, introducing the four-concern model, etc.) is a larger story documented elsewhere. This spec starts from the point where the fx-pipeline already had effects, scope partitioning, transitions-as-effects, traits, DLQ, and forward sub-pipelines — and documents the cleanup that simplified all of that into the current architecture.

This document supersedes 15 of the 16 prior specs in `docs/superpowers/specs/`. Those files are deleted upon adoption of this document. The 16th — `2026-04-27-provides-removal-post-unified-effects.md` — is retained for historical reference (superseded 2026-05-04: provides is a permanent API).

---

## 2. Architectural Transformation

### 2.1 The Midpoint Architecture (fx-pipeline before cleanup)

By the midpoint of the fx-pipeline branch, the initial implementation had replaced the legacy eager resolver with algebraic effects, introduced the four-concern model (schema, collection, routing, policy), and shipped scope partitioning. But the implementation had accumulated significant tech debt and design missteps:

**What existed at the midpoint:**
- **Algebraic effects** via nix-effects (`fx.send`, `fx.bind`, `scope.provide`, `state.modify`) — the core infrastructure, working correctly.
- **Scope-partitioned state** — `mkScopeId`, `scopedClassImports`, `scopeParent`, `scopeContexts`, dual-write handlers. Shipped and functional.
- **`transition.nix`** (~1011 lines) — the `into-transition` handler managed entity boundary crossings via scope push/pop with explicit stack tracking. Handled fan-out, enrichment iteration, cross-provider emission, and aspect policy inheritance. The largest and most complex file in the pipeline.
- **`dispatch-policies.nix`** — created as a replacement for transition dispatch, but grew to ~524 lines recreating the same complexity with `fx.bind` instead of explicit state threading. Five dedup layers, parent-scope chain walks, forward output accumulators.
- **Three-tier trait system** (~2463 lines across 23 files) — handlers, state fields, deferred evaluation, inheritance walking, three-tier classification. Interacted with every other pipeline concern. Every scope change, forward, and dedup mechanism had trait-specific branches.
- **DLQ (Dead Letter Queue)** (~178 lines) — unregistered freeform keys went to a dead letter queue, re-classified when new trait/class registrations occurred via `drain-dead-letters`. Structural key sniffing was fragile.
- **Forward sub-pipelines** (`resolveForwardSource` / `runSubPipeline`) — post-pipeline re-walks of source entities for class module collection. Redundant since scope partitions already held the data.
- **provide-to mechanism** (~150 lines) — `handler → distribute-cross-entity → crossEntityTraits` for cross-entity trait data routing. Replaced by scope inheritance but not yet deleted.
- **Monolithic `emit-include` handler** — branched into 5 code paths (forward/conditional/parametric/static/dedup). Internal dispatch instead of caller classification.
- **`provides.X` structural key** — still active with `emitSelfProvide`, `emitCrossProvider`, and a provides-compat shim. The original removal plan was blocked on the unified aspect key type.
- **entityIncludes / entityProvides** — already deleted by midpoint, replaced by `den.schema.X.includes` and policies.
- **Stages (`den.stages`)** — already deleted by midpoint, replaced by schema-based policies.
- **`rootIncludes`** — already deleted by midpoint.

**The core problem:** The midpoint architecture had translated the old sub-pipeline model into effect syntax without redesigning for effects. `transition.nix` and `dispatch-policies.nix` reimplemented scope management, dedup, and iteration manually — exactly what the effect system's `scope.provide` and `bind` already provided as primitives. Each fix for one of the 20+ remaining test failures revealed new implicit contracts between these overlapping mechanisms.

### 2.2 The Cleanup Arc (What Changed)

The cleanup arc deleted ~3024 net lines and replaced overlapping mechanisms with effect-system-native patterns. The key insight driving all changes: **use `scope.provide` for scoping, not manual state management.**

**What was deleted:**

| Component | Lines | Why |
|-----------|-------|-----|
| `transition.nix` | -1011 | Replaced by `installPolicies` using `scope.provide` for context expansion |
| Trait system (23 files) | -2463 | Deleted for fleet/den.exports reimplementation — interacted with every pipeline concern |
| `dispatch-policies.nix` | -524 | Monolithic handler that recreated transition complexity. Replaced by inline `installPolicies` |
| DLQ machinery | -178 | Unregistered keys now emit as classes immediately |
| provide-to mechanism | -150 | Replaced by scope inheritance |
| `scopeStack`, `scopeChildren`, `scopeProvenance` state fields | — | Explicit stack replaced by `scope.provide` lexical scoping |
| Forward sub-pipelines | ~200 | `applyForwardSpecs` reads scope partitions directly |

**What was added/redesigned:**

- **`installPolicies`** (aspect.nix ~45 lines + `policy-dispatch.nix` 656 lines). Replaces both `transition.nix` and `dispatch-policies.nix`. Policies are dispatched and their effects processed using `scope.provide` for context expansion. When `policy.resolve { user = tux; }` fires, `scope.provide` creates a lexically-scoped handler frame — the entity tree walks inside it, handler restoration is automatic. No explicit scope push/pop stack.

- **Narrow effect vocabulary.** The monolithic `emit-include` handler (5 code paths) was decomposed into typed effects: `resolve-aspect` (20 lines), `resolve-parametric` (60 lines), `resolve-conditional` (52 lines), `check-dedup` (51 lines). Callers classify, handlers execute. Single code path per handler.

- **`resolve-schema-entity` effect** (192 lines). Reusable entity resolution: scope push/walk/pop as a proper effect handler. Extracted from the depths of `transition.nix` and `dispatch-policies.nix` into a composable handler that policies and future mechanisms can invoke.

- **`policy.provide` effect** (30-line handler). Delivers new content directly to a target class without tree walking. Created to fix duplicate emissions from the provides-compat shim — `policy.include` walked content through the tree, creating duplicates when content matched routes. `policy.provide` bypasses the tree entirely.

- **`policy.resolve.withIncludes`**. Per-resolve scoped includes. The old approach mixed root-scope and entity-scope includes in one flat list — all includes were injected into all entity walks. Now entity-scope includes are carried on their resolve effect and emitted inside `scope.provide` where entity context is live.

- **Forward scope isolation.** Root-scope forward specs propagated to child scopes during `installPolicies`. Each child scope gets its own copy with `sourceScopeId` pointing at the child's partition. Post-pipeline `applyForwardSpecs` reads per-scope sources with filtered root fallback (only `@default` identity modules).

- **DLQ elimination.** Unregistered freeform keys emit as class modules immediately. The DLQ's structural sniffing and re-classification on trait registration added complexity for marginal benefit. With traits deleted, the DLQ had no remaining purpose.

### 2.3 The Current Architecture (Post-Cleanup)

The pipeline is 4570 lines across 27 files. The four-concern model alignment after cleanup:

| Concern | Midpoint mechanism | Post-cleanup mechanism |
|---------|-------------------|----------------------|
| **Schema** (what entities exist) | `den.schema` with `includes`, `policies` (already clean) | Unchanged |
| **Collection** (gather class/trait content) | Monolithic `emit-include` + `emitSelfProvide` + `emitCrossProvider` + DLQ | Narrow effects + direct class emission (no DLQ) |
| **Routing** (move content between classes) | `policy.route` + forward sub-pipelines + `provide-to` | `policy.route` + `policy.provide` + scope partition reads |
| **Policy** (who gets what, context expansion) | `transition.nix` / `dispatch-policies.nix` + enrichment loop | `installPolicies` + `scope.provide` + typed effects |

**What survived unchanged from the midpoint:** Algebraic effects infrastructure, scope-partitioned state (`mkScopeId`, `scopedClassImports`, `scopeParent`, `scopeContexts`), typed policy effects API (`policy.resolve/include/exclude/route/instantiate`), `policy.route` for Tier 1 delivery, constraint registry, forward infrastructure for Tier 2, `provides.X` structural key + compat shim (removal is next work item).

---

## 3. Design Invariants & Constraints

These are non-obvious decisions discovered during implementation that future work must respect. Violating any of these will likely cause test failures.

### 3.1 den.default Stripping Is Load-Bearing

`resolveEntityHandler` (pipeline.nix) strips `den.default` from child entity includes. This prevents duplicate NixOS module definitions when den.default re-walks at child scopes.

**Why:** den.default is walked once at root scope. If child entities also include den.default (via their includes), `includeSeen` doesn't catch the duplicate because child entity resolution tags den.default with different `__scopeHandlers`, changing its `childIdentity`. Without stripping, anonymous modules produce "defined multiple times" errors.

**Tested:** Removing stripping causes 8+ test failures. The filtered root fallback (only `@default` identity modules) exists as the complementary mechanism — child scopes get den.default's shared modules (stateVersion etc.) via root fallback without re-walking.

### 3.2 scope.provide and state.modify Must Stay in Sync

`scope.provide` handles lexical handler scoping (context handlers visible to tree walk). `state.modify` tracks `currentScope` for emission handlers (which scope partition to write to). Both happen in the same `fx.bind` chain with deterministic ordering.

**Invariant:** If `scope.provide` restores parent handlers but `state.modify` hasn't popped the scope (or vice versa), emissions go to the wrong partition. The `resolve-schema-entity` handler maintains this invariant — any new scope-creating mechanism must follow the same pattern.

### 3.3 mkScopeId Injectivity

`mkScopeId` produces `"host=igloo,user=tux"` format — key=value pairs, sorted, comma-separated. The key names are included (unlike the old `mkCtxId` which was values-only). Different contexts must produce different scope IDs.

**Edge case:** Entity names containing `=` or `,` would break injectivity. Safe because entity names are Nix identifiers (alphanumeric + hyphens).

### 3.4 Enrichment Is Downward-Flowing Only

Enrichment policies (isNixos, isDarwin, etc.) fire within a scope and inject enrichment keys into that scope's context. Child scopes inherit enrichment from parent context via scope.provide (parent context already has enrichment keys when child scope is created).

**Invariant:** A child scope's enrichment does not affect the parent or siblings. Violating this would break commutativity — sibling evaluation order would affect results.

### 3.5 Parametric Handling Differs by Type Branch

When the unified aspect key type is implemented (Section 8), parametric functions must be handled differently based on their classification:

| Branch | Functions are... | Resolution |
|--------|-----------------|------------|
| ClassModule (registered class keys) | Terminal — produce final NixOS/HM modules | `wrapClassModule` pre-applies den args |
| Aspect (everything else) | Structural — produce more aspect tree | Coerced to `{ includes = [fn]; }`, pipeline recurses |

Coercing class modules to includes would break `wrapClassModule`. Keeping aspect functions as content would lose aspect shape. This distinction is fundamental.

### 3.6 Sibling Transitions Are Sequential

Within a scope, `resolve-schema-entity` processes sibling transitions one at a time (push scope A, walk, pop, push scope B, walk, pop). Nix's strict evaluation enforces this — no concurrent sibling processing. The scope mechanism is correct because push/pop is balanced within a single `fx.bind` chain.

### 3.7 Forward Source Reads Use Scope + Root Fallback

Post-pipeline `applyForwardSpecs` reads source modules per-scope with a filtered root fallback (only modules with `@default` identity from root scope). This is the correct pattern because:
- Per-scope only: misses den.default's shared modules (stateVersion)
- Merged all scopes: cross-user leakage
- Root fallback with identity filter: gets shared modules without host-specific contamination

### 3.8 Constraint Registry Serves Multiple Purposes

`register-constraint` / `check-constraint` in tree.nix currently handles: exclusion (from `policy.exclude`), substitution (from `meta.excludes` with replacement), and filter predicates (from `meta.handleWith`). These are functionally distinct but share the handler because they all gate include resolution. The substitution path is vestigial (see Section 9).

### 3.9 Declared Options vs Freeform Keys Separation

`aspect-schema.nix` accesses `builtins.attrNames config.den.aspects` (forcing key SET) and `aspects.${name}.classes or {}` (declared option, default `{}`). Declared option access does NOT trigger freeform `aspectKeyType.merge`. This separation is load-bearing — if `aspect-schema.nix` were refactored to access freeform keys, it would create circular evaluation.

### 3.10 Forward Propagation Timing

Root-scope forward specs must be propagated to child scopes **during the pipeline walk** (in `installPolicies`), not as post-pipeline expansion. The pipeline walk is where scope information is live. Post-pipeline, scope context is lost and expansion can't distinguish per-scope vs aggregate forwards.

---

## 4. Component Inventory

### 4.1 Files Deleted (from main or during cleanup arc)

| File | Purpose | Replacement |
|------|---------|-------------|
| `nix/lib/aspects/fx/handlers/transition.nix` | Entity boundary crossing, sub-pipelines, fan-out | `installPolicies` + `resolve-schema-entity` |
| `nix/lib/aspects/resolve.nix` | Legacy aspect resolution | Folded into fx pipeline |
| `nix/lib/aspects/adapters.nix` | Forward adapter utilities | Folded into `route.nix` |
| `nix/lib/ctx-apply.nix` | Context application helpers | Eliminated — context is handler-scoped |
| `nix/lib/ctx-types.nix` | Context type definitions | Eliminated — schema-based |
| `nix/lib/last-function-to.nix` | Legacy utility | Dead code |
| `nix/lib/statics.nix` | Static resolution helpers | Dead code |
| `modules/context/host.nix` | Self-provide for host entities | `resolveEntity.includes` |
| `modules/context/user.nix` | Self-provide for user entities | `resolveEntity.includes` |
| `modules/aspects/provides/mutual-provider.nix` | Cross-entity mutual provide | `mutual-provider-shim.nix` (compat) |
| `modules/outputs/osConfigurations.nix` | NixOS output wiring | Flake policies + `policy.instantiate` |
| `modules/outputs/hmConfigurations.nix` | Home Manager output wiring | Flake policies + `policy.instantiate` |
| `modules/outputs/flakeSystemOutputs.nix` | Per-system flake output wiring | Flake policies + `policy.route` |
| `modules/fxPipeline.nix` | Pipeline module registration | `modules/config.nix` |

### 4.2 Files Added

| File | Lines | Purpose |
|------|-------|---------|
| `handlers/resolve-schema-entity.nix` | 192 | Reusable entity resolution: scope push/walk/pop |
| `handlers/resolve-aspect.nix` | 20 | Static aspect resolution |
| `handlers/resolve-parametric.nix` | 60 | Handler probing + deferral |
| `handlers/resolve-conditional.nix` | 52 | Guard check via pathSet |
| `handlers/check-dedup.nix` | 51 | includeSeen query |
| `handlers/forward.nix` | 262 | Forward spec registration + source scope |
| `handlers/provide.nix` | 30 | policy.provide collection |
| `handlers/provides-compat.nix` | 132 | Backwards-compat shim for provides.X |
| `policy-dispatch.nix` | 656 | Policy matching, enrichment, dispatch |
| `route.nix` | 350 | Route application + wrapRouteModules |
| `include-emit.nix` | ~380 | Include classification + emission (extracted from aspect.nix) |
| `key-classification.nix` | 94 | Key classification (extracted from aspect.nix) |
| `class-module.nix` | 239 | Class module wrapping helpers (extracted) |
| `content-util.nix` | 79 | `unwrapContentValues` shared helper |
| `wrap-classes.nix` | 171 | `wrapCollectedClasses` (extracted from pipeline.nix) |
| `trace-util.nix` | 27 | Trace utilities |
| `nix/lib/policy-effects.nix` | 104 | Typed policy effect constructors |
| `nix/lib/synthesize-policies.nix` | 24 | Policy synthesis from provides-compat |
| `modules/aspect-schema.nix` | — | Dynamic schema collection from aspects |
| `modules/config.nix` | — | Pipeline configuration |
| `modules/policies/core.nix` | — | Core policies (host-to-users, etc.) |
| `modules/policies/flake.nix` | — | Flake output policies |
| `modules/context/flake-schema.nix` | — | Flake entity schema definitions |
| `modules/aspects/provides/flake-scope.nix` | — | Flake-scope provides |
| `modules/compat/ctx-shim.nix` | — | Backwards-compat for den.ctx.* |
| `modules/compat/mutual-provider-shim.nix` | — | Backwards-compat for mutual-provider |
| `modules/removed-stages.nix` | — | Error messages for deleted den.stages |

### 4.3 Significantly Changed Files

| File | Main lines | HEAD lines | Change |
|------|-----------|-----------|--------|
| `aspect.nix` | 221 | 318 | resolveChildren rewritten, installPolicies added. Code extracted to `include-emit.nix` (386), `class-module.nix` (239), `key-classification.nix` (94). |
| `pipeline.nix` | 142 | 531 | Scope state init, fxResolve with per-scope wrapping/routing/forwarding |
| `tree.nix` (handlers) | — | 430 | New: emission handlers, constraint registry, chain tracking. (Replaces role of deleted `transition.nix` (101 lines on main) plus new handler infrastructure.) |
| `include.nix` (handler) | ~50 | 50 | Simplified — delegates to narrow effect handlers |
| `identity.nix` | ~60 | ~60 | Unchanged |
| `constraints.nix` | ~50 | ~50 | Unchanged |
| `types.nix` | ~400 | ~450 | aspectKeyType placeholder, provides option still present |
| `nix/lib/forward.nix` | ~200 | ~200 | API preserved, internals simplified |

### 4.4 Mechanisms Deleted During Cleanup Arc

| Mechanism | Lines | When | Replacement |
|-----------|-------|------|-------------|
| Stages (`den.stages`, stage ordering) | ~100 | Early branch | `den.schema` policies |
| entityIncludes/entityProvides | ~100 | Early branch | `den.schema.X.includes`, policies |
| rootIncludes | ~30 | Early branch | Policies |
| Sub-pipelines (`runSubPipeline` for isolation) | ~200 | Mid branch | Scope partitions |
| provide-to (handler, distribute-cross-entity) | ~150 | Mid branch | Scope inheritance |
| Transitions (`into-transition`, `resolveTransition`, etc.) | ~1011 | Cleanup arc | `installPolicies` + `resolve-schema-entity` |
| Traits (handlers, state, inheritance, deferred, classification) | ~2463 | Cleanup arc | Deleted; fleet/den.exports future |
| DLQ (`deadLetterQueue`, `drain-dead-letters`, structural sniffing) | ~178 | Cleanup arc | Direct class emission |
| dispatch-policies.nix (monolithic handler) | ~524 | Cleanup arc | `installPolicies` + `policy-dispatch.nix` (well-structured) |

### 4.5 Net Statistics

| Metric | Value |
|--------|-------|
| Commits ahead of main | 293 |
| nix/lib/ lines (main → HEAD) | 3335 → 12285 (+8950, includes diagrams) |
| modules/ lines (main → HEAD) | 1462 → 1769 (+307) |
| fx/ pipeline lines (main → HEAD) | 1178 → 4570 (+3392) |
| git diff --stat (nix/lib + modules) | 124 files changed, +11782, -2525 |
| Tests | 629/629 passing |

---

## 5. Provides: Permanent API, New Internals

~~**Previous plan:** Remove `provides` entirely, migrate templates to `policies.X`.~~

**Corrected understanding:** `provides.to-users`, `provides.to-hosts`, and `provides.<name>` are established user-facing APIs for cross-entity content delivery. Users depend on them. They must remain as first-class API targets.

### 5.1 What Changed

The legacy transition system that originally powered provides has been eliminated. The provides→policy translation is now folded into `emitAspectPolicies` as a first-class pipeline feature. New users should use policies and direct nesting; existing `provides`/`_` patterns continue working unchanged.

### 5.2 What Stays (User API)

| Component | File | Status |
|-----------|------|--------|
| `provides` option on `aspectSubmodule` | `types.nix` | **Permanent** — user-facing API |
| `_` alias (`mkAliasOptionModule`) | `types.nix` | **Permanent** — ergonomic shorthand |
| `"provides"` in `structuralKeysSet` | `key-classification.nix` | **Permanent** — prevents dispatch as class key |

### 5.3 What Changes (Internals)

| Component | File | Change |
|-----------|------|--------|
| `provides-compat.nix` | `handlers/provides-compat.nix` | **Deleted** — functionality folded into `emitAspectPolicies` |
| `emitCrossProvideShims` | `aspect.nix` | **Deleted** — separate phase removed from `resolveChildren` |
| `emitSelfProvide` | `include-emit.nix` | **Deleted** — self-provide folded into `emitAspectPolicies` as auto-include |
| `emitAspectPolicies` | `include-emit.nix` | **Modified** — now handles `policies.*` AND `provides.*` in one phase |

See `2026-05-04-provides-cleanup-design.md` for the full design spec.

### 5.4 Impact on Dependency Chain

The original dependency chain gated constraint cleanup, providerType dispatch, and aspect key type on provides removal. Since provides is not being removed, those items are **independently implementable**:

- Constraint handler simplification (Section 7.2) — `substitute` type is vestigial regardless of provides
- providerType dispatch (Section 7.3) — can proceed with provides in place
- Aspect key type (Section 8) — independent of provides

### 5.5 `den.provides.*` Factory Namespace

The `den.provides` top-level namespace (`den.provides.forward`, `den.provides.define-user`, etc.) is the **factory registry** — NOT the aspect-level `provides` structural key. It is unaffected and also permanent.

---

## 6. Forward Elimination — COMPLETE (verified 2026-05-04)

All forward elimination items are implemented. The forward system has two tiers: Tier 1 (simple forwards auto-classified as routes) and Tier 2 (adapter/guard forwards using `buildForwardAspect`). Both are handled uniformly by `applyRoutes` in `route.nix`.

### 6.1 policy.instantiate for Entity Outputs — COMPLETE

Old output modules (`osConfigurations.nix`, `hmConfigurations.nix`, `flakeSystemOutputs.nix`) deleted. Flake policies in `modules/policies/flake.nix` handle all wiring via `policy.resolve.to`, `policy.route`, and `policy.instantiate`.

### 6.2 Forward Auto-Detection — COMPLETE

`forwardHandler` in `handlers/forward.nix` classifies forwards at handler time:
- **Tier 1** (`isTier1`): `canDirectImport && !needsAdapter && !evalConfig && sourceIsLocal && sourceAlreadyCollected` → stored as simple route shape
- **Tier 2**: everything else → stored with `__complexForward = true`

Both tiers go into `scopedRoutes`. Users writing simple `den.provides.forward` get route performance with zero migration.

### 6.3 Scope-Prefixed Dedup Keys — COMPLETE

- `includeSeen`: scope-prefixed in `check-dedup.nix` (`"${scope}/${rawDedupKey}"`)
- `ctxSeen`: scope-prefixed in `ctx.nix` (`"${scope}/${key}"`)

### 6.4 Flat State Field Removal — COMPLETE

`scopedClassImports` is the sole source of truth during the walk. No flat `classImports` state field exists. The `emit-class` handler writes only to `scopedClassImports`. Post-pipeline flattening in `fxResolve` is a local variable, not state. `applyForwardSpecs` no longer exists — logic inlined into `applyRoutes`.

### 6.5 Tier 2 Forward Simplification — STABLE

Tier 2 forwards (adapter modules, dynamic paths) stay as-is. Real-world cases:
- **Niri adapter**: option schema extension via `types.submoduleWith`
- **Persist adapter**: `apply = lib.unique` for dedup
- **home-env.nix** (`makeHomeEnv`): home-manager/hjem/maid integration

`den.provides.forward` remains the API for these cases. The infrastructure works correctly.

### 6.6 Remaining Dependency Chain

```
Constraint handler simplification (Section 7.2) — independent
providerType dispatch (Section 7.3) — independent
  → Aspect key type (Section 8)
```

Forward items no longer block anything.

### 6.7 Vestigial Items — ALL CLEAN

- `__resolveCtx` / `__aspectPolicies`: absent from codebase
- `resolveForwardSource`: absent from codebase
- `applyForwardSpecs`: absent (logic inlined into `applyRoutes`; only a comment reference remains in `route.nix`)

---

## 7. What Comes Next: Remaining Cleanup

### 7.1 Handler File Decomposition

**Shipped:** `aspect.nix` decomposed into `class-module.nix`, `key-classification.nix`, `content-util.nix`, `include-emit.nix`, `wrap-classes.nix`. `transition.nix` deleted entirely.

**Remains:**
- `emitSelfProvide` / `emitCrossProvideShims` / `provides-compat.nix` — all deleted (2026-05-04). Functionality folded into `emitAspectPolicies` in `include-emit.nix`. See Section 5 and `2026-05-04-provides-cleanup-design.md`.
- `policy-dispatch.nix` at 656 lines is the largest file. Consider extraction of `classifyPolicyEffects` and enrichment iteration into focused helpers. Not urgent — the file is well-structured with clear function boundaries.

### 7.2 Constraint Handler Simplification

The constraint registry can be simplified (independent of provides, which remains):

| Type | Current status | After cleanup |
|------|---------------|----------------------|
| `"exclude"` | Active — from `policy.exclude` | Unchanged |
| `"substitute"` | Vestigial — from provides-era `meta.excludes` | Delete |
| `"filter"` | Active — from `meta.handleWith` | Unchanged |

Delete `substituteChild` from `include-emit.nix`, `substitute` type from `constraints.nix`, and the `"substitute"` branch in `check-constraint` (tree.nix).

### 7.3 Architecture Cleanup Phase 3a: providerType Dispatch

`types.nix:308` has a dead conditional — both branches call `contentType.merge`. The comment documents that the else-branch should dispatch to `providerType` for unregistered freeform keys.

**Design:** Two-branch dispatch in `aspectKeyType`:
- Registered class keys → `contentType.merge` (ClassModule: provenance-wrapped `__contentValues`)
- Everything else → `providerType.merge` (Aspect: proper aspect shape with `name`, `meta`, `__functor`, `includes`)

**Note:** The original spec described a three-branch dispatch (ClassModule / SemanticData / Aspect) where SemanticData handled registered trait keys. With traits deleted, this simplifies to two branches. When traits are reimplemented, a SemanticData branch can be re-added.

**Risk:** Medium-high. `providerType` runs multi-def values through full aspect submodule merge (NixOS module system fixed-point) instead of list collection. This changes observable merge semantics for unregistered freeform keys defined from multiple files. Full CI gates the change.

**Dependency:** None — `provides` keys are handled by `structuralKeysSet` classification before reaching the freeform type dispatch. The `provides` option coexists with providerType dispatch.

---

## 8. What Comes Next: Aspect Key Type

The unified aspect key type replaces the current dual-type system (`aspectContentType` for freeform, `providerType` for provides children) with a single type that dispatches on key name.

### 8.1 Current State

- `aspectKeyType` exists in `types.nix:293` as a placeholder — both branches call `contentType.merge`
- `providerType` exists and is used for `includes` list elements and the `provides` submodule
- Comment at line 288 documents the intended switch

### 8.2 Two-Branch Design

```
aspectKeyType.merge(loc, defs) =
  let key = lib.last loc; in
  if classes ? key  → ClassModule merge (provenance, preserves parametric)
  otherwise         → Aspect merge (providerType: proper shape, coerces parametric)
```

### 8.3 Future Three-Branch Design (Post-Traits-Reimplementation)

When traits return:
```
aspectKeyType.merge(loc, defs) =
  let key = lib.last loc; in
  if classes ? key  → ClassModule merge
  if traits ? key   → SemanticData merge (provenance, preserves parametric)
  otherwise         → Aspect merge
```

The SemanticData branch uses the same merge as ClassModule (`__contentValues` provenance wrapping). The distinction matters for pipeline dispatch — trait keys are emitted via `emit-trait`, class keys via `emit-class`.

### 8.4 Execution

Execution spec to be written when ready. The providerType dispatch (Section 7.3) is a prerequisite for the full aspect key type implementation.

### 8.5 Registry Access

No circular dependency. `den.classes or {}` is accessible from `types.nix` — confirmed by existing usage at types.nix:179,200. The `or {}` fallback prevents evaluation-order issues. `aspect-schema.nix` accessing declared options (`.classes`) does NOT trigger freeform merge.

---

## 9. What Comes Next: Policy Scoping & Activation Redesign

### 9.1 Problem

Policy scoping is currently implicit and inconsistent. Policies are defined on a `.policies` field (on aspects, schema entries, or globally) and activated by argument matching — if a policy's function signature is satisfied by the current context, it fires. This creates several problems:

1. **No way to store a policy without activating it.** Defining `den.aspects.igloo.policies.foo = ...` both registers and activates the policy. There's no concept of "here's a policy, but don't use it yet."

2. **Scope is unclear from definition site.** A policy on `den.policies.*` is global. A policy on `den.schema.host.policies.*` is entity-kind-scoped. A policy on `den.aspects.*.policies.*` fires when the aspect is included. But this isn't documented or enforced — it's emergent behavior from how the dispatch code happens to iterate registries.

3. **No fine-grained scoping.** You can't say "this policy applies only to igloo's subtree" or "this policy applies only to this specific entity instance" without encoding guards in the policy function body (`lib.optionals (host.name == "igloo") [...]`).

4. **`meta.handleWith` is a parallel mechanism.** Aspects can declare `meta.handleWith` to install custom constraint handlers (filters, excludes, substitutions) for their subtree. This is semantically a scoped policy — "within my subtree, apply these rules" — but uses completely different machinery (the constraint registry in tree.nix). Similarly, `meta.excludes` is sugar for `handleWith` with exclude-type constraints.

### 9.2 Design Direction: Registry vs Activation

**Core principle:** `.policies` is the **registry** — it stores policy functions. `includes`/`excludes` is the **activation method** — it controls where policies fire.

**Policy scoping levels:**

| Level | Scope | Activation mechanism |
|-------|-------|---------------------|
| **Pipeline** | Entire pipeline run | Pipeline configuration (e.g., `mkPipeline { policies = [...]; }`) |
| **Global** | All entities, all scopes | `den.policies.*` registration (current behavior, preserved) |
| **Entity-kind** | All entities of a kind | `den.schema.host.policies.*` (current behavior, preserved) |
| **Entity-instance** | A specific entity | Include policy on the entity's aspect (e.g., `den.aspects.igloo.includes = [ myPolicy ]`) |
| **Aspect subtree** | Within an aspect's include tree | Include policy in an aspect's includes (fires only during that aspect's subtree resolution) |

**How it works:**

Policies become first-class values that can be placed in `includes` and `excludes`:

```nix
# Registry: define the policy (does NOT activate it)
den.aspects.myBattery.policies.deliver-niri = { host, user, ... }:
  [ (policy.route { fromClass = "niri"; intoClass = "homeManager"; path = [ "programs" "niri" ]; }) ];

# Activation: include the policy at the desired scope
# Entity-instance scope — only igloo gets this policy:
den.aspects.igloo.includes = [ den.aspects.myBattery.policies.deliver-niri ];

# Aspect subtree scope — policy fires within myBattery's subtree:
den.aspects.myBattery.includes = [ den.aspects.myBattery.policies.deliver-niri ];

# Deactivation: exclude a policy from a subtree:
den.aspects.server.excludes = [ den.aspects.myBattery.policies.deliver-niri ];
```

**Global and entity-kind policies** continue to work as today — `den.policies.*` and `den.schema.*.policies.*` are implicitly activated at their respective scopes. The new mechanism adds finer-grained control without breaking the existing API.

### 9.3 meta.handleWith → Policy Methods

`meta.handleWith` currently installs constraint handlers (filters, excludes, substitutions) scoped to an aspect's subtree. This is semantically identical to "activate a policy for this subtree that gates what gets included."

**Current usage:**
```nix
den.aspects.server = {
  meta.handleWith = [
    { type = "exclude"; identity = identity.pathKey (identity.aspectPath den.aspects.desktop); scope = "subtree"; }
  ];
  meta.excludes = [ den.aspects.desktop ];  # sugar for the above
};
```

**Proposed: express as policy methods:**
```nix
# Exclusion becomes a policy in the subtree's includes:
den.aspects.server = {
  includes = [
    (policy.exclude den.aspects.desktop)  # already exists as a policy effect
  ];
};

# Custom filter handler becomes a policy:
den.aspects.server = {
  includes = [
    (policy.filter (aspect: aspect.meta.category or "" != "desktop"))
  ];
};
```

This eliminates `meta.handleWith` and `meta.excludes` as separate mechanisms. The constraint registry (`register-constraint` / `check-constraint` in tree.nix) simplifies — excludes and filters become policy effects processed through the standard policy dispatch path.

**New policy method: `policy.filter`**

```nix
policy.filter = pred: {
  __policyEffect = "filter";
  value = pred;  # aspect -> bool
};
```

A filter policy gates includes within its activation scope. When an aspect is about to be resolved, the filter predicate runs. If it returns false, the aspect is excluded from that scope. This replaces the `"filter"` type in the constraint registry.

### 9.4 What This Enables

- **Battery aspects with self-contained routing.** A battery aspect registers its policies in `.policies` and includes them in its own `includes`. Anyone who includes the battery gets the policies scoped to that subtree — no global pollution.

- **Per-entity policy overrides.** A specific host can exclude a globally-registered policy: `den.aspects.igloo.excludes = [ den.policies.strict-firewall ]`.

- **Declarative policy topology.** The policy activation graph becomes visible from the aspect tree structure — you can see which policies fire where by reading includes/excludes, not by tracing argument matching at runtime.

- **Elimination of handleWith/excludes/constraint parallel path.** One mechanism (policy effects via includes/excludes) replaces three (handleWith, meta.excludes, constraint registry types).

### 9.5 Open Questions

1. **Policy identity for excludes.** When `excludes = [ myPolicy ]`, the system needs to match `myPolicy` against activated policies. Current policies are functions — function identity in Nix is referential (same thunk = same policy). Is this sufficient, or do policies need explicit names/keys for matching?

2. **Ordering between included policies and included aspects.** If `includes = [ policyA, aspectB ]`, does policyA apply to aspectB's subtree? The natural reading is yes — includes are processed in order, policyA activates before aspectB resolves. But this creates ordering sensitivity in the includes list.

3. **Global policy override granularity.** Can an entity-instance exclude override a global policy? If `den.policies.foo` fires globally and `den.aspects.igloo.excludes = [ den.policies.foo ]`, should igloo be exempt? This requires the exclude mechanism to reach into the global dispatch path.

4. **Migration path for meta.handleWith.** The `handleWith` option accepts arbitrary handler records, not just exclude/filter types. Any handler record can be registered. How much of this flexibility do policy methods need to preserve? The `"substitute"` type is vestigial (Section 7.2), but custom handler types may exist in user configs.

### 9.6 Dependency

The constraint registry simplification (Section 7.2, removing substitute type) should happen first, reducing the surface area that the policy scoping redesign needs to address.

---

## 10. Vestigial Code Audit

Suspected dead or unnecessary code paths. Each should be verified (grep for callers, run CI after removal) before deletion.

### 9.1 Provides-Era Mechanisms (Remove with Section 5)

| Item | Location | Why vestigial | Related section |
|------|----------|--------------|-----------------|
| `emitSelfProvide` | `include-emit.nix:326`, `aspect.nix:34,113` | No aspect has `provides.${name}` — always returns `fx.pure []` | Section 5 |
| `mkPositionalInclude` | `include-emit.nix:243` | Only called by `emitSelfProvide` | Section 5 |
| `mkNamedInclude` | `include-emit.nix:274` | Only called by `emitSelfProvide` | Section 5 |
| `emitCrossProvideShims` | `provides-compat.nix:100`, `aspect.nix:9,115` | Entire compat handler | Section 5 |
| `substituteChild` | `include-emit.nix:122,225` | `check-constraint` substitute path is provides-era | Sections 5, 7.2 |
| `substitute` type | `constraints.nix:26-42` | No provides = no substitution source | Section 7.2 |
| `provides` option | `types.nix:395` | Structural key being removed | Section 5 |
| `_` alias | `types.nix:318` | Alias for provides | Section 5 |
| `"provides"` in structuralKeysSet | `key-classification.nix:14` | No longer a structural key | Section 5 |
| `resolveWithProvidesFallback` | `den-brackets.nix:10-23` | Provides namespace gone | Section 5 |
| `provides-compat.nix` (entire file) | `handlers/provides-compat.nix` | 132 lines of compat | Section 5 |

### 9.2 Forward-Era Mechanisms (Verify with Section 6)

| Item | Location | Why suspected vestigial | Related section |
|------|----------|----------------------|-----------------|
| `__resolveCtx` on forward specs | `handlers/forward.nix` | Already removed — no longer captured |
| `__aspectPolicies` on forward specs | `handlers/forward.nix` | Already removed — no longer captured |

### 9.3 Dead Code (Verify independently)

| Item | Location | Why suspected vestigial |
|------|----------|----------------------|
| `aspectKeyType` dead branch | `types.nix:308` | Both branches identical — placeholder | Section 7.3 |
| Duplicate `"policies"` in structuralKeysSet | `key-classification.nix:15` | Single entry present — no duplicate. Mentioned in provides-removal spec was wrong. |
| ~~`hostFramework = []`~~ | `resolve-entity.nix` | **Already removed** (Phase 1d shipped) |
| ~~`policyFnArgs` / `policy-types.nix`~~ | `nix/lib/policy-types.nix` | **Already removed** (Phase 1e shipped, file deleted) |

### 9.4 Dedup Layers (Evaluate with Section 6)

| Layer | Location | Purpose | May be removable? |
|-------|----------|---------|-------------------|
| `includeSeen` | pipeline state | Prevent duplicate include resolution | Keep — scope.provide doesn't eliminate all cases (static includes from shared aspects) |
| `scopedEmittedLocs` | pipeline state | Prevent duplicate class module emission per scope | Evaluate — may be redundant with NixOS key-based dedup |
| `dispatchedPolicies` | pipeline state | Prevent policies from firing twice | Keep — needed for enrichment iteration re-dispatch |
| `registeredRouteKeys` | `tree.nix:368,388` | Prevent duplicate route registration | Evaluate — may be redundant with scope isolation |
| `ctxSeen` | handler state | Prevent duplicate context processing | Keep — essential for fan-out dedup |

---

## 11. Future: Traits Reimplementation

**Status: Unvalidated design sketch.** This section preserves key insights from the refactor for future design work. Traits were deleted (-2463 lines) to simplify the dispatch redesign. They will be reimplemented with a fundamentally different architecture.

### 10.1 Why Traits Were Deleted

The three-tier trait system added ~400 lines of pipeline complexity (handlers, state fields, inheritance walking, deferred evaluation) that interacted with every other pipeline concern. During the transition elimination, traits were the primary source of cross-cutting complexity — every scope change, forward, and dedup mechanism had trait-specific branches.

### 10.2 What Traits Actually Are

Traits span a spectrum from pure pipeline-time flags to NixOS-config-dependent data:

| Trait | Purpose | Pipeline-time? | Needs NixOS config? |
|---|---|---|---|
| impermanence | "does this host have impermanence?" | Yes (gates routing) | Yes (persistence rules need `config.services.*.user`) |
| secrets | "agenix or sops-nix?" | Yes (selects backend) | No |
| firewall | "does this host have firewall enabled?" | Yes (gates inclusion) | Yes (rules need `config.services.*.port`) |
| xdg-mime | "does the user have a desktop?" | Yes (gates DE aspects) | No |
| service-discovery | export data for loadbalancers, backup targets | No | Yes |

The old three-tier model handled all of these but conflated pipeline-time routing decisions with NixOS-config-dependent data aggregation.

### 10.3 The fleet + den.exports Design

**Key insight:** Separate pipeline-time capability flags from NixOS-config-dependent data.

**Pipeline-time trait flags** stay as lightweight capability indicators (booleans, enums, small lists). These gate policy routing decisions during the pipeline walk. Implementation: simple key registry in `den.traits`, `classifyKeys` dispatches to trait branch. No DLQ, no structural sniffing, no three-tier classification.

**NixOS-config-dependent data** moves to `fleet` + `den.exports`:

```nix
# Built into den's flake policy — one line:
_module.args.fleet = lib.mapAttrs (_: sys: sys.config) config.flake.nixosConfigurations;

# Uniform export interface — injected into every host's mainModule:
options.den.exports = lib.mkOption {
  type = lib.types.lazyAttrsOf lib.types.anything;
  default = {};
};
```

Any host reads any other host's evaluated config via the lazy `fleet` attrset:

```nix
# Firewall: collect ports from all fleet members
{ fleet, host, ... }: {
  networking.firewall.allowedTCPPorts =
    lib.concatMap (peer: peer.den.exports.firewall or [])
      (lib.attrValues fleet);
}

# Hosts file
{ fleet, ... }: {
  networking.hosts = lib.mkMerge (lib.mapAttrsToList
    (name: cfg: cfg.den.exports.hostsEntries or {}) fleet);
}
```

### 10.4 Cycle Avoidance

The `fleet` pattern has one structural constraint: no circular evaluation paths. Host A can read host B's config, but host B's value must not depend on host A's read.

In practice this is rarely an issue — service ports, persistence paths, and backup targets are derived from the host's own service definitions. Mutual dependencies (host A's firewall depends on host B's firewall) would infinite-recurse. This is the same constraint as any lazy self-referential structure in Nix.

### 10.5 What Gets Deleted vs Reimplemented

**Deleted permanently** (the old three-tier model):
- `traitModuleForScope`, deferred trait evaluation, `inheritTraits` scope walking
- Three-tier classification in `classifyKeys`
- Trait state fields (`scopedTraits`, `scopedDeferredTraits`, `scopedConsumedTraits`, `traitSchemas`)
- `partialOk` validation, trait module injection
- `traitCollectorHandler`, `traitArgHandler`
- `get-trait-schemas` / `register-trait-schema` effects

**Reimplemented (future)**:
- Pipeline-time trait flags: simple boolean/enum registry, lightweight `classifyKeys` branch
- Cross-host data: `fleet` + `den.exports` (plain NixOS module code, no pipeline involvement)
- Trait-to-class routing: `policy.route { fromTrait = "persist"; intoClass = "nixos"; ... }` (the route infrastructure already supports this — `fromTrait` field is in the spec but implementation is deferred to traits reimplementation)

### 10.6 Dynamic Trait Schema Registration (Preserved Design)

When traits return, the scope-partitioned spec's dynamic registration design should be used:

- `register-trait-schema` effect during tree walk (analogous to `register-aspect-policy`)
- `state.traitSchemas` seeded from `den.traits`, grows during walk
- `classifyKeys` reads from dynamic registry
- DLQ-free: unknown keys emit as classes; if a trait schema registers later, the key was already emitted as a class (no re-classification needed — accept the slight semantic mismatch, or add a warning)

This enables self-contained battery aspects that bring their own trait schema + emission + routing policy via a single `includes` reference.

---

## 12. Specs Superseded by This Document

The following spec files are deleted. Their key design decisions are preserved in Sections 3, 5-10 above.

| File | Summary |
|------|---------|
| `2026-04-26-pipeline-simplification-design.md` | entityIncludes removal, sub-pipeline extraction, isolateFanOut. All shipped. |
| `2026-04-26-provides-removal-design.md` | Early provides removal draft targeting direct nesting. Superseded by 04-27 version targeting policies. |
| `2026-04-27-class-dedup-post-unified-effects.md` | Class emission dedup interaction with unified effects. includeSeen and unsatisfied guard shipped. substituteChild removal folded into Section 7.2. |
| `2026-04-27-unified-aspect-key-type-design.md` | Three-branch type dispatch. Design preserved in Section 8, simplified to two branches (traits deleted). |
| `2026-04-29-entity-class-evaluation.md` | policy.instantiate for forward elimination. Shipped via flake policies. Forward auto-detection remaining work in Section 6.2. |
| `2026-04-29-handler-file-cleanup.md` | aspect.nix/transition.nix decomposition. transition.nix deleted, aspect.nix decomposed. Remaining items in Section 7.1. |
| `2026-04-29-policy-route-class-delivery.md` | Routes, scope-prefixed dedup, forward simplification. Routes shipped. Remaining items in Section 6. |
| `2026-04-29-scope-partitioned-pipeline-state.md` | Scope infrastructure, trait inheritance, provide-to removal. All shipped except flat state removal (Section 6.4). |
| `2026-05-01-entity-ownership-tracking.md` | 15 test failures from entity ownership loss. All fixed — 629/629 passing. |
| `2026-05-01-forward-scope-isolation.md` | Forward spec propagation + per-scope source reading. Shipped. |
| `2026-05-01-pipeline-architecture-cleanup.md` | Narrow effects, policy.provide, installPolicies decomposition. All shipped. |
| `2026-05-01-policies-as-handlers-redesign.md` | installPolicies + scope.provide design. Shipped. fleet/den.exports design preserved in Section 11. |
| `2026-05-01-scope-native-policy-dispatch.md` | policy.resolve.withIncludes API. Shipped. |
| `2026-05-01-transition-elimination.md` | Core transition elimination. Shipped (-3024 lines). |
| `2026-05-02-architecture-cleanup.md` | Mechanical dedup, extraction, providerType dispatch. Phases 1-2 shipped. Phase 3a in Section 7.3. |

---

## Appendix: Policy Effects API Reference

```nix
policy.resolve { user = tux; }                    # Context expansion
policy.resolve.withIncludes [asp] { user = tux; }  # With scoped includes
policy.resolve.to "kind" { bindings }              # Explicit target kind
policy.resolve.shared { system = sys; }            # Shared (non-isolated) fan-out
policy.include aspect                              # Inject aspect into walk
policy.exclude aspect                              # Remove from resolution
policy.route { fromClass; intoClass; path; ... }   # Move content between classes
policy.provide { class; module; path?; }           # Deliver new content
policy.instantiate entity                          # Post-pipeline evaluation
policy.pipelineOnly value                          # Tag: class-wins collision
```
