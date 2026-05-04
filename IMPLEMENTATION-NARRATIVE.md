# Implementation Narrative: feat/fx-pipeline Branch

**Date:** 2026-05-04
**Purpose:** Accurate intermediate documentation tracing how the fx-pipeline implementation evolved. Designed to be consumed alongside the consolidated spec (`docs/superpowers/specs/2026-05-02-fx-pipeline-consolidated.md`) for producing accurate end-state documentation.

**Reading guide:** This document covers the implemented-branch specs chronologically. For each phase, it describes what was designed, what actually shipped, where the design diverged, and what the lasting consequence is. The consolidated spec describes the current end-state; this document explains how we got there and why.

---

## Phase 1: Legacy Removal (Apr 17-20)

### Specs
- `legacy-removal/2026-04-17-session-summary.md` — session log
- `legacy-removal/2026-04-17-fixedto-ctx-propagation.md` — `__ctx` design
- `legacy-removal/2026-04-18-ctx-as-data-design.md` — context as data
- `legacy-removal/2026-04-19-provide-to-effects-design.md` — cross-entity routing
- `legacy-removal/2026-04-20-context-guard-design.md` — guard elimination
- `legacy-removal/simplified-review-guide.md` — architecture overview

### What was designed

Remove the legacy resolver (`resolve.nix`, `adapters.nix`, `statics.nix`, `parametric.nix` internals) and make the fx pipeline the sole resolution path. Key design decisions:

1. **`__ctx` as context carrier.** Aspects tagged with `__ctx = { host, user }` to propagate context as data rather than ambient handler state.
2. **`scope.stateful` for transitions.** Scoped handlers providing context to subtrees.
3. **`provide-to` for cross-entity routing.** Two-phase resolution: collect cross-entity emissions in phase 1, distribute to target configs in phase 2.
4. **Guard elimination.** `perHost`/`perUser`/`take.exactly` become identity wrappers — handler-based resolution makes guards unnecessary.

### What actually shipped

- Legacy resolver fully deleted. fx pipeline is the only path.
- `__ctx` shipped but was **immediately superseded** by `__scopeHandlers` (a cleaner mechanism using handler closure maps instead of data fields). The `ctxFromHandlers` helper reconstructs context from handlers when needed.
- `scope.stateful` was tried but **failed** — it uses `state.update` which overwrites outer state, losing `emit-class` accumulations from rotated effects. `scope.run` was tried next but also had issues (effects rotating out of scope). Eventually settled on `scope.provide` after nix-effects got deep handler semantics.
- `provide-to` handler and `distribute-cross-entity.nix` library shipped. But phase 2 orchestration was **never wired** — `osConfigurations.nix` never called `distributeCrossEntityTraits`. This entire mechanism was later **deleted** during the cleanup arc and replaced by scope inheritance in the scope-partitioned state.
- Guard deprecation shims shipped (`perCtx`, `perHost`, `perUser` become identity + `lib.warn`).

### Lasting consequence

The key survivor from this phase is the **architectural principle**: context flows via scoped handlers, not data fields. `scope.provide` + `constantHandler` is the universal mechanism for making entity context available to subtrees. Everything else (provide-to, `__ctx`, scope.stateful, scope.run) was tried and discarded.

The session summary's exploration of `scope.stateful` vs `scope.run` vs `scope.provide` is valuable context for understanding why `scope.provide` was chosen — it's the only variant that doesn't interfere with outer state mutations.

### Staleness in these specs

- `__ctx` references throughout — replaced by `__scopeHandlers`
- `provide-to` two-phase architecture — deleted, replaced by scope inheritance
- `scope.stateful`/`scope.run` recommendations — superseded by `scope.provide`
- `distribute-cross-entity.nix` — deleted
- `simplified-review-guide.md` — describes intermediate state with `transitionHandler`, `dispatchPolicyIncludes`, stages. All substantially superseded.

---

## Phase 2: Policy System (Apr 20-24)

### Specs
- `policies/2026-04-20-policies-design.md` — three-layer policy model
- `policies/2026-04-21-ctx-as-classes-design.md` — four-concern separation
- `policies/2026-04-21-ctx-as-classes-prior-art.md` — terminology reference
- `policies/2026-04-23-effectful-pipeline-bootstrap-design.md` — abandoned: effectful bootstrap
- `policies/2026-04-23-forward-as-handler-design.md` — `emit-forward` effect

### What was designed

1. **Three-layer policy model:** Layer 1 (policies as data), Layer 2 (effect-based materialization with per-policy named handlers), Layer 3 (cross-entity routing via provide-to).
2. **Four-concern separation** (`ctx-as-classes`): Data (schema), Policies (topology), Stages (scoping), Behavior (aspects). Replaces the conflated `den.ctx` namespace.
3. **Effectful pipeline bootstrap:** `resolve-policy` and `resolve-target` effects, `policy.handlers` field — policy submodule with per-policy handler installation.
4. **Forward as handler:** `emit-forward` effect replacing fresh pipeline calls.

### What actually shipped

- **Policies as plain functions.** The three-layer model shipped but Layer 2 (per-policy named effect handlers) was **abandoned** before implementation. The `effectful-pipeline-bootstrap` spec was superseded when `den.stages` deletion and `policy-dispatch.nix` deletion made its foundation impossible. Instead, policies became plain functions returning typed effects — simpler, no `__functor`, no activation model.
- **Four-concern separation** shipped with evolution: "Stages" concern was immediately **eliminated** (Phase 3) — it was introduced and deleted in the same branch. The final model is three concerns: Schema, Collection+Routing, Policy. (The consolidated spec Section 2.3 shows the current four-concern model which re-introduces "Collection" and "Routing" as separate concerns.)
- **`den.ctx` removed.** `ctx-shim.nix` provides backwards-compat forwarding.
- **Forward as handler** shipped. `emit-forward` effect and `forwardHandler` deferred forward processing to post-pipeline. The initial attempt at inline resolution failed (fresh pipeline lacks parent context). This was later resolved by scope partitioning.
- **Prior art terminology** (Entity, Schema, Class, Policy) is still accurate and used throughout the codebase.

### Lasting consequence

The **typed policy effects API** (`policy.resolve`, `policy.include`, `policy.exclude`) is the core survivor — it's the user-facing API that all later work built on. The insight that policies should be plain functions (not submodules with metadata) eliminated a huge amount of complexity.

The `emit-forward` effect and deferred forward processing pattern also survived, though forwards were later largely replaced by `policy.route`.

### Staleness in these specs

- `effectful-pipeline-bootstrap-design.md` — entirely abandoned. `resolve-policy`, `resolve-target` effects never created. Per-policy `handlers` field impossible with plain functions.
- Layer 2 named effect handlers — never built.
- Layer 3 cross-entity routing — provide-to deleted, replaced by scope inheritance.
- `den.stages` references in `ctx-as-classes` — stages immediately eliminated.
- `corePolicies` registry — never built. Core policies live directly in `den.policies`.
- `den.ctx.*.into` forwarding — stub in compat shim, no real migration path.

---

## Phase 2b: Class Module Partial Application (Apr 24)

### Spec
- `class-modules/2026-04-24-class-module-partial-apply-design.md`

### What was designed

`wrapClassModule` intercepts class modules in `emitClasses`, detects den context args in function signatures via `builtins.functionArgs`, pre-applies them from scoped handlers, and passes the partially-applied result to NixOS. Enables flat-form class modules: `{ host, config, pkgs, ... }: { networking.hostName = host.name; }`.

### What actually shipped

Fully shipped with scope extensions beyond the spec:
- Partial application (den args pre-applied, NixOS args left)
- Full application (when ALL args are den args — returns a plain attrset)
- Collision policies at three levels (aspect, entity, global `den.config.classModuleCollisionPolicy`)
- Deferred-import wrapping for `{ imports = [fn]; }` patterns
- `unsatisfied` guard for class modules requesting den args not in context

### Lasting consequence

`wrapClassModule` is unchanged and central to the pipeline. The collision policy mechanism (class-wins, pipeline-only) is used by `pipelineOnly` for flake-scope args. The `unsatisfied` guard prevents evaluation failures from context-dependent class modules included at the wrong scope.

### Staleness

Minimal. `ctxFromHandlers` replaced `aspect.__ctx or {}` — already reflected in the code. The spec's approach A was chosen and matches current implementation.

---

## Phase 3: Stages Elimination & Traits (Apr 25-26)

### Specs
- `stages/2026-04-25-eliminate-stages-design.md` — three-layer aspect model
- `stages/2026-04-26-direct-ref-aspects-design.md` — entityIncludes deletion
- `stages/2026-04-26-eliminate-stages-implementation-notes.md` — intermediate state notes
- `traits/2026-04-25-traits-and-structural-nesting-design.md` — traits, three-tier eval
- `traits/2026-04-25-freeform-ignore-and-provide-to-unification.md` — freeform keys + provide-to
- `traits/2026-04-25-traits-branch-review-findings.md` — review findings
- `traits/2026-04-26-class-emission-dedup-and-parametric-merge-design.md` — dedup fixes

### What was designed

1. **Stages elimination.** Replace `den.stages` with three layers: `den.aspects` (registry), policy `aspects` field (inclusion), entity `includes` (static). Intermediate step: `den.entityIncludes`.
2. **Direct-ref aspects.** Delete `entityIncludes` / `rootIncludes`. Policies reference aspects directly (`den.aspects.foo`) not by string name.
3. **Traits.** Semantic data channels with three evaluation tiers (static, pipeline-parametric, module-evaluated). Schema-driven classification (`den.classes` + `den.traits`). Structural nesting (nested aspects at any key, not just under `provides`).
4. **Freeform-ignore + provide-to unification.** Unregistered keys → trait or class via registry lookup. Merge `provide-to` and traits into one collection system.
5. **Class emission dedup.** Fix parametric merge (coerce to includes), include-level dedup (`includeSeen`), unsatisfied guard.

### What actually shipped

- **Stages fully eliminated.** `den.stages` tombstoned. `resolveEntity` + `den.schema.X.includes` is the live interface. The intermediate `entityIncludes` was created then also deleted.
- **Direct-ref aspects** shipped. Policies use `den.aspects.*` refs directly.
- **Traits fully shipped** — then **entirely deleted** during the cleanup arc (Phase 5). The three-tier model (~2463 lines across 23 files) was removed as pre-work for the dispatch redesign because it interacted with every pipeline concern. Reimplementation planned via fleet + den.exports.
- **Freeform-ignore** partially shipped (phase 1: unregistered keys to DLQ). Phase 2 (provide-to unification) shipped but was later **deleted** with provide-to.
- **Class emission dedup** fully shipped. `includeSeen`, parametric merge coercion, and unsatisfied guard all survived to the current state.

### Lasting consequence

**Stages elimination** is permanent — the three-layer model (aspects, policies, schema includes) is the foundation. **Direct-ref aspects** is permanent. **Class emission dedup** mechanisms (`includeSeen`, unsatisfied guard) survived intact.

**Traits** are the major reversal. The entire three-tier model was built, shipped, tested, then deleted. The consolidated spec Section 11 preserves the fleet + den.exports design for reimplementation. Key insight preserved: pipeline-time trait flags (booleans/enums for routing) are separate from NixOS-config-dependent data (fleet + den.exports handles that).

### Staleness in these specs

- **All traits specs** describe a deleted system. The design insights are valuable but the implementation details are entirely gone.
- `entityIncludes` references in stages specs — deleted twice (introduced then removed).
- `rootIncludes` — deleted.
- `provide-to` unification — deleted with provide-to.
- DLQ (`deadLetterQueue`, `drain-dead-letters`) — eliminated during cleanup arc. Unregistered keys now emit as classes directly.
- `structuralKeysSet` still lists `"provides"` — this is a vestigial item flagged in the consolidated spec Section 9.1.

---

## Phase 4: Unified Effects, Scope Partitioning, Routes (Apr 27-29)

### Specs
- `policies/2026-04-27-unified-policy-effects-design.md` — typed effects API
- `policies/2026-04-28-policy-context-enrichment-design.md` — enrichment routing
- `policies/2026-04-28-policy-pipeline-simplification.md` — plain function policies
- `policies/2026-04-29-flake-scope-pipeline-args-design.md` — `pipelineOnly`
- `provides-removal/2026-04-26-provides-removal-design.md` — provides API removal plan
- `provides-removal/2026-04-28-provides-compat-shims-design.md` — compat shims
- `pipeline-simplification/2026-04-26-pipeline-simplification-design.md` — six targets
- `pipeline-simplification/2026-04-26-pipeline-simplification-targets.md` — target audit
- `pipeline-simplification/2026-04-29-entity-class-evaluation.md` — `policy.instantiate`
- `pipeline-simplification/2026-04-29-policy-route-class-delivery.md` — `policy.route`
- `pipeline-simplification/2026-04-29-scope-partitioned-pipeline-state.md` — scope partitioning
- `traits/2026-04-29-forward-elimination-trait-unification.md` — trait delivery in forwards

### What was designed

This is the densest phase — 12 specs covering the core architectural work:

1. **Unified policy effects.** `policy.resolve`, `policy.include`, `policy.exclude` as typed constructors. Policies as plain functions. All `__functor`/metadata removed.
2. **Policy context enrichment.** Non-schema resolve keys (isNixos, isDarwin) enrich current entity context rather than creating empty child entities. All `wrapClassModule` calls moved to post-pipeline pass.
3. **Policy pipeline simplification (Phases A-E).** Remove all policy metadata, convert templates, delete dead dispatch code.
4. **Flake-scope pipeline args.** `pipelineOnly` collision policy + `den.provides.flake-scope` battery for `lib`/`inputs`/`den` in aspect functions.
5. **Provides removal plan.** Seven deletion steps for the `provides` structural key. Migrate 114 template occurrences.
6. **Provides compat shims.** `provides-compat.nix` handler + `mutual-provider-shim.nix` as migration bridge.
7. **Pipeline simplification targets.** Six targets (provides removal, collision detection, sub-pipeline extraction, classifyKeys, fan-out generalization, child shape normalization).
8. **`policy.instantiate`** for entity output wiring (replacing osConfigurations.nix etc.).
9. **`policy.route`** for class/trait delivery from scope partitions.
10. **Scope-partitioned state.** Replace sub-pipeline isolation with scope-partitioned state. `mkScopeId`, scoped emission handlers, scope push/pop, trait inheritance, dynamic trait schema registration.
11. **Forward-trait unification.** `fxResolve` reorder + `subTraitModule` for trait data in forward sub-pipelines.

### What actually shipped

- **Unified policy effects** — fully shipped. The API (`policy.resolve`, `.include`, `.exclude`) is the current user-facing API. Later extended with `.route`, `.provide`, `.instantiate`.
- **Context enrichment** — fully shipped. Enrichment routing and post-pipeline wrapping are core mechanisms.
- **Policy pipeline simplification (A-E)** — fully shipped. All 23 steps complete. Plain functions, no `__functor`.
- **Flake-scope args** — fully shipped. `pipelineOnly` is in current `policy-effects.nix`.
- **Provides removal** — **NOT shipped.** Compat shims landed instead. The 114 template migrations never happened. This is the primary remaining work item (consolidated spec Section 5).
- **Provides compat shims** — shipped. `provides-compat.nix` and `mutual-provider-shim.nix` are live.
- **Pipeline simplification targets** — Targets 3 (sub-pipeline extraction) and 5 (fan-out generalization) shipped. Targets 1 (provides), 2 (collision detection), 4 (classifyKeys), 6 (child shapes) did not ship. Sub-pipeline extraction was later **undone** (sub-pipelines eliminated entirely during cleanup arc).
- **`policy.instantiate`** — shipped. Flake output forwards eliminated.
- **`policy.route`** — shipped. Built-in forwards (os-class, os-user, wsl) converted. Route application in `fxResolve`.
- **Scope-partitioned state** — shipped (9 commits). All scope infrastructure, dual-write handlers, trait inheritance, provide-to removal. Some items were later simplified or deleted during the cleanup arc (scopeStack, scopeChildren eliminated by `scope.provide`; traits deleted).
- **Forward-trait unification** — shipped (fxResolve reorder, subTraitModule). Then traits were **deleted**, making the trait-related parts moot.

### Lasting consequence

This phase built the infrastructure that the cleanup arc then simplified. The key survivors are:
- **Policy effects API** (resolve, include, exclude, route, provide, instantiate) — unchanged
- **Context enrichment** — unchanged
- **Plain function policies** — unchanged
- **`pipelineOnly`** — unchanged
- **Scope-partitioned state** (`mkScopeId`, `scopedClassImports`, `scopeParent`, `scopeContexts`) — unchanged
- **`policy.route`** + `wrapRouteModules` — unchanged
- **`policy.instantiate`** — unchanged
- **Provides compat shims** — still live, removal is next work

### Staleness in these specs

- **Scope-partitioned state spec** references traits extensively (scopedTraits, deferredTraits, traitSchemas, traitModuleForScope, inheritTraits) — all deleted with traits.
- **Scope-partitioned state spec** describes `scopeStack`/`scopeChildren`/`scopeProvenance` — eliminated during cleanup arc.
- **Forward-trait unification** — trait-related parts deleted.
- **Pipeline simplification targets** — Targets 3/5 shipped but Target 3 (`runSubPipeline`) was later undone. Targets 1/2/4/6 unchanged (not started or partial).
- **Entity class evaluation** references `flake-os`/`flake-hm` schema kinds — these may have been eliminated.
- **Provides removal spec** blocked on unified aspect key type — still blocked, but for different reasons than originally stated (traits are gone, so the three-branch type is now two-branch).

---

## Phase 5: Transition Elimination — The Cleanup Arc (May 1-2)

### Spec
- `transition-elimination/2026-05-01-transition-elimination.md`

(Plus 16 specs from `docs/superpowers/specs/` consolidated into `2026-05-02-fx-pipeline-consolidated.md`)

### What was designed

Replace transition boundaries with policy-driven context expansion. Delete `transition.nix`, the DLQ, and the trait system. Redesign dispatch as `installPolicies` using `scope.provide`.

### What actually shipped

- **`transition.nix` deleted** (-1011 lines)
- **Trait system deleted** (-2463 lines across 23 files)
- **DLQ eliminated** (-178 lines)
- **`dispatch-policies.nix`** created as replacement, then **deleted** when it recreated the same complexity. Replaced by inline `installPolicies` in `aspect.nix`.
- **Narrow effect vocabulary** — monolithic `emit-include` decomposed into 4 typed effects.
- **`resolve-schema-entity`** — reusable entity resolution handler.
- **`policy.provide`** — direct content delivery (fixes provides-compat duplicate emissions).
- **`policy.resolve.withIncludes`** — per-resolve scoped includes.
- **Forward scope isolation** — root-to-child propagation + filtered root fallback.
- **Architecture cleanup** — dedup of `schemaEntityKinds`, extraction of `content-util.nix`, `wrap-classes.nix`, `class-module.nix`.
- **629/629 tests passing** (from 647/667 baseline before trait deletion).

### Lasting consequence

This IS the current state. The consolidated spec (`2026-05-02-fx-pipeline-consolidated.md`) is the authoritative description.

### Staleness

The transition-elimination spec in `implemented-branch/` says "612/616 tests pass (99.4%), 4 edge cases remain." All 4 are now fixed — 629/629.

---

## Cross-Phase Design Decisions That Survived

These decisions, made across multiple phases, are load-bearing in the current architecture:

| Decision | Phase | Why it matters |
|----------|-------|---------------|
| `scope.provide` for context, not `scope.stateful`/`scope.run` | 1 | Only variant that doesn't interfere with outer state mutations |
| `__scopeHandlers` as context carrier, not `__ctx` | 1→2 | Handler closures compose naturally; data fields require manual threading |
| Policies as plain functions, not submodules | 2→4 | Eliminated `__functor`, activation model, per-policy handlers — massive simplification |
| `wrapClassModule` partial application with collision policies | 2b | Unchanged mechanism enabling flat-form class modules |
| `den.schema.X.includes` replacing entityIncludes/stages | 3 | Clean three-layer model (aspects, policies, schema includes) |
| `includeSeen` dedup + `unsatisfied` guard | 3 | Survived trait deletion, cleanup arc — fundamental pipeline correctness mechanisms |
| Typed policy effects API | 4 | `policy.resolve/include/exclude/route/provide/instantiate` is the user-facing contract |
| Scope-partitioned state with `mkScopeId` | 4 | Foundation for entity isolation without sub-pipelines |
| `den.default` stripping is load-bearing | 5 | Removing it causes 8+ test failures — complementary with filtered root fallback |
| `installPolicies` via `scope.provide`, not monolithic handler | 5 | The cleanup arc's key insight: use the effect system's primitives, don't reimplement them |

## Cross-Phase Design Decisions That Were Reversed

| Decision | Phase introduced | Phase reversed | What replaced it |
|----------|-----------------|---------------|-----------------|
| `__ctx` as context carrier | 1 | 1-2 | `__scopeHandlers` + `ctxFromHandlers` |
| `scope.stateful` for transitions | 1 | 1 | `scope.provide` |
| `provide-to` two-phase cross-entity routing | 1 | 4-5 | Scope inheritance via `scopeParent` tree |
| `den.stages` scoping concern | 2 | 3 | Eliminated — parametric resolution handles scoping |
| Three-tier trait system | 3 | 5 | Deleted — fleet + den.exports planned |
| DLQ for unregistered keys | 3-4 | 5 | Direct class emission |
| `dispatch-policies.nix` monolithic handler | 5 (early) | 5 (late) | `installPolicies` inline with `scope.provide` |
| `scopeStack`/`scopeChildren`/`scopeProvenance` | 4 | 5 | `scope.provide` lexical scoping eliminates need for explicit stack |
| `runSubPipeline` combinator | 4 | 5 | Scope partitions + inline resolution |
| `entityIncludes` per-kind aspect lists | 3 (early) | 3 (late) | `den.schema.X.includes` with direct refs |

---

## Spec Status Update (as of 2026-05-04)

Quick-reference for each implemented-branch spec against current codebase:

### legacy-removal/
| Spec | SUMMARY verdict | Current reality |
|------|----------------|-----------------|
| session-summary | implemented | Accurate as history; mechanisms described are all superseded |
| fixedto-ctx-propagation | superseded | `__ctx` → `__scopeHandlers`. Correct verdict. |
| ctx-as-data-design | implemented | `scope.provide` replaced ctx-as-data. Verdict should be "superseded." |
| provide-to-effects-design | partially-implemented | Phase 2 never wired, then provide-to **deleted entirely**. Verdict should be "superseded." |
| context-guard-design | partially-implemented | Guards eliminated. Shims survive. Verdict accurate. |
| simplified-review-guide | superseded | Describes intermediate state. Correct verdict. |

### policies/
| Spec | SUMMARY verdict | Current reality |
|------|----------------|-----------------|
| policies-design | partially-implemented | Core shipped, Layer 2/3 abandoned. Verdict accurate. |
| ctx-as-classes-design | partially-implemented | Separation shipped. Stages eliminated. Verdict accurate. |
| ctx-as-classes-prior-art | implemented | Terminology reference. Still accurate. |
| effectful-pipeline-bootstrap | not-implemented | Abandoned. Correct verdict. |
| forward-as-handler-design | implemented | `emit-forward` shipped. Correct verdict. |
| unified-policy-effects-design | partially-implemented | Phases A-D shipped. `corePolicies` absent. Traits deleted. Verdict should be "implemented" (the remaining items were deliberately not pursued). |
| policy-context-enrichment | implemented | Correct verdict. |
| policy-pipeline-simplification | implemented | Correct verdict. |
| flake-scope-pipeline-args | implemented | Correct verdict. |

### provides-removal/
| Spec | SUMMARY verdict | Current reality |
|------|----------------|-----------------|
| provides-removal-design | not-implemented | Correct — migration not started. This spec targets direct nesting, but the actual migration target is now `policies.X` (per consolidated spec Section 5). |
| provides-compat-shims | implemented | Correct verdict. Shims still live. |

### stages/
| Spec | SUMMARY verdict | Current reality |
|------|----------------|-----------------|
| eliminate-stages | implemented | Correct verdict. |
| direct-ref-aspects | implemented | Correct verdict. |
| implementation-notes | superseded | Correct verdict. |

### traits/
| Spec | SUMMARY verdict | Current reality |
|------|----------------|-----------------|
| traits-and-structural-nesting | partially-implemented | **Should be "deleted."** Entire system removed. |
| freeform-ignore-and-provide-to | partially-implemented | **Should be "deleted."** Both provide-to and freeform-ignore via DLQ removed. |
| traits-branch-review-findings | partially-implemented | **Should be "deleted."** Bug references are in deleted code. |
| class-emission-dedup | implemented | Core dedup mechanisms survive (includeSeen, unsatisfied guard). Verdict accurate. |
| forward-elimination-trait-unification | partially-implemented | Trait parts deleted. Forward reorder logic survived briefly then was superseded by scope partitioning. **Should be "superseded."** |

### pipeline-simplification/
| Spec | SUMMARY verdict | Current reality |
|------|----------------|-----------------|
| pipeline-simplification-design | partially-implemented | Targets 3/5 shipped then superseded. Target 1 still pending. Verdict accurate. |
| pipeline-simplification-targets | partially-implemented | Correct verdict. |
| entity-class-evaluation | (not in SUMMARY) | `policy.instantiate` shipped. Flake output forwards eliminated. |
| policy-route-class-delivery | (not in SUMMARY) | `policy.route` shipped. Routes fully functional. |
| scope-partitioned-pipeline-state | in-progress | **Should be "implemented"** (with caveats: traits deleted, some state fields removed). |

### transition-elimination/
| Spec | SUMMARY verdict | Current reality |
|------|----------------|-----------------|
| transition-elimination | (not in SUMMARY) | Substantially shipped. 629/629 tests. |
