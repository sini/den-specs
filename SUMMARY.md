# Den Spec Implementation Audit — Executive Summary

Generated: 2026-04-28, updated 2026-04-29. Branch under review: `feat/fx-pipeline`.

---

## 1. Overview Table

| Spec | Component | Verdict | Target | Current Status | One-line summary |
|---|---|---|---|---|---|
| 2026-04-04-aspect-transforms-spec | aspect-rewrites | superseded | main→branch | Legacy API never shipped; functionality exists in fx pipeline under different names | `transforms.nix` API replaced wholesale by fx constraint system |
| 2026-04-04-chain-based-tracing | aspect-rewrites | superseded | main→branch | `tracingHandler` / `structuredTraceHandler` in fx/trace.nix; functor wrapping gone | Chain-based tracing fully realized via `chain-push`/`chain-pop` effects |
| 2026-04-04-deep-provider-cascade | aspect-rewrites | partially-implemented | main | Core `meta.provider` list and cascade landed; `toAspectPath` and `__placedBy` absent | Provider path as `listOf str` with prefix-cascade in `tree.nix`; public API weaker than spec |
| 2026-04-04-provider-aware-excludes | aspect-rewrites | implemented | main | Prefix cascade in `tree.nix`; `adapters.nix` deleted on branch | Single-level excludeAspect cascade implemented; replaced by constraint registry on branch |
| 2026-04-11-hasAspect-design | hasAspect | implemented | main | Live on both main (adapter-based) and branch (fx-based); public API unchanged | `entity.hasAspect`, `.forClass`, `.forAnyClass` all shipped; `collectPaths` adapter removed on branch |
| 2026-04-12-nix-effects-resolution-spec | fx-pipeline | implemented | main→branch | Full freer monad pipeline in production; some surface naming diverged | Core fx pipeline fully operational; conditions system and `has-aspect` effect not shipped |
| 2026-04-13-fx-adapters-spec | fx-pipeline | superseded | main→branch | `meta.adapter` API replaced by `meta.handleWith` / constraint registry | Adapter-combinator model replaced by declarative constraint system |
| 2026-04-13-fx-ctx-apply-spec | fx-pipeline | superseded | main | `ctxApplyEffectful` never created; two-stage design collapsed into unified pipeline | ctx-apply design absorbed into `into-transition` handler |
| 2026-04-13-fx-feature-complete-spec | fx-pipeline | partially-implemented | branch | `structuredTraceHandler` present; `ctxTraceHandler`, `__ctxStage` tagging, `flakeModules/` not shipped | Feature-flag removed; pipeline unconditional; ctx-trace and stage tagging not implemented |
| 2026-04-13-fx-resolution-prototype-spec | fx-pipeline | superseded | main | Squash-merged as `68c26555`; all functionality present under different names | Prototype spec superseded by direct full delivery in den-fx PR #462 |
| 2026-04-13-fx-v2-refactor-design | fx-pipeline | implemented | main→branch | All six goals landed in single squash commit; naming diverged per emit-X pattern | v2 architecture delivered wholesale; effect naming (`emit-class` not `provide-class`) evolved |
| 2026-04-14-aspects-as-computations | fx-pipeline | implemented | main | `aspectToEffect`, all three effect types, `includeHandler`, constraint registry live on main | Core spec fully realized; `resolveDeepEffectful` not created as named function |
| 2026-04-14-effectful-handlers | fx-pipeline | implemented | nix-effects | Effectful resume live in pinned nix-effects fork; production handlers use it | Trampoline fix in `sini/nix-effects`; `rotateInterpret` path less exercised |
| 2026-04-14-fx-v2-architectural-direction | fx-pipeline | implemented | branch | All seven changes landed; legacy pipeline removed; no A/B flag | Planning doc superseded by direct implementation; "adapter" → "constraint" rename |
| 2026-04-14-includes-chain-effects-design | fx-pipeline | implemented | main | `chain-push`/`chain-pop` live in `chainHandler`; `__parent` removed | Includes-chain provenance fully shipped; `chainParent` adds robust anonymous filtering |
| 2026-04-14-unified-effects-pipeline | fx-pipeline | implemented | main | All three architectural issues resolved in `68c26555` | Full pipeline architecture (`aspectToEffect`, `emit-class`, `into-transition`) operational |
| 2026-04-15-close-gaps-spec | fx-pipeline | superseded | branch | All mechanical gaps closed; spec's "stays as-is" items accurate | Gap-closing work superseded by subsequent Phase E simplification |
| 2026-04-15-diag-rebase-spec | fx-pipeline | superseded | branch | Diag library written fresh on unified pipeline; no rebase needed | Spec's rebase strategy bypassed; `diagram-demo` template born onto fx pipeline |
| commit-compaction-guide | fx-pipeline | superseded | main | PR squash-merged as single commit; 26-commit plan never executed | Compaction guide superseded by GitHub squash-and-merge of PR #462 |
| 2026-04-17-fixedto-ctx-propagation | legacy-removal | superseded | branch | `__ctx` replaced by `__scopeHandlers` + `scope.provide`; `fixedTo` deprecated | Cleaner `__scopeHandlers` mechanism superseded inline `__ctx` approach |
| 2026-04-17-host-aspects-battery-design | pipeline-simplification | implemented | main+branch | Shipped on main (PR #466) and branch; generalized to `user.classes` list | `host-aspects` battery live; expanded beyond spec (multi-class support) |
| 2026-04-17-session-summary | legacy-removal | implemented | branch | All removals confirmed; `feat/rm-legacy` absorbed into `2e0fc9e1` | Legacy resolver removed; parametric.nix shim-only; denTest.nix rewritten |
| 2026-04-18-ctx-as-data-design | legacy-removal | implemented | branch | `scope.run` removed from transitions; `__scopeHandlers` canonical carrier | Context flows as handler closures, not data fields; `scope.provide` wraps `bind.fn` |
| 2026-04-19-provide-to-effects-design | legacy-removal | partially-implemented | branch | `provideToHandler` and sibling routing shipped; phase 2 orchestration not wired | Cross-entity distribution library exists but `osConfigurations.nix` not plumbed |
| 2026-04-20-context-guard-design | legacy-removal | partially-implemented | branch | Guard mechanism eliminated; `has-handler` probing active; `__functor` cleanup deferred | Deprecation shims in place; "remaining next phase" items mostly pre-resolved |
| 2026-04-20-policies-design | policies | partially-implemented | branch | Plain functions shipped; phase 2 cross-entity distribution not wired for production | Policy layers 1–2 delivered differently than spec; layer 3 library-only |
| 2026-04-21-ctx-as-classes-design | policies | partially-implemented | branch | Three-concern separation delivered; `den.ctx.*.into` stub only; `perHost-perUser.nix` not deleted | Core separation done; `den.stages` tombstoned not redirected; compat gap in `into` slot |
| 2026-04-21-ctx-as-classes-prior-art | policies | implemented | branch | Reference doc; all four terminology concepts (`Entity`, `Schema`, `Class`, `Policy`) live | Accurate conceptual reference; policy `from`/`to` example stale in doc |
| 2026-04-23-effectful-pipeline-bootstrap-design | policies | not-implemented | branch | `resolve-policy`, `resolve-target` effects, `policy.handlers` field — none exist | Abandoned; superseded by plain-function policies before it could be built |
| 2026-04-23-forward-as-handler-design | policies | implemented | branch | `emit-forward` effect and `forwardHandler` shipped; sub-pipeline deferred to post-processing | Forward-as-handler landed; `1c077a06` refined inline execution to deferred post-processing |
| 2026-04-24-class-module-partial-apply-design | class-modules | implemented | branch | Fully shipped with scope extensions (full-application path, trait consumption, deferred-import wrapping) | `wrapClassModule` live with collision policies at three levels |
| 2026-04-25-eliminate-stages-design | stages | implemented | branch | `den.stages` tombstoned; `den.schema.<kind>.includes` is the live interface | Three-layer model active; `resolveEntity` + `resolve-entity` effect are sole path |
| 2026-04-25-freeform-ignore-and-provide-to-unification | traits | partially-implemented | branch | Phase 1 (freeform-ignore) shipped; phase 2 component pieces exist but orchestration not wired | `distribute-cross-entity.nix` exists; `fxResolveTree` not wired for two-phase flow |
| 2026-04-25-traits-and-structural-nesting-design | traits | partially-implemented | branch | Five phases shipped; `hasTrait`, derived traits, trait attenuation deferred | Core trait pipeline complete; `provides` deprecation warning not wired; follow-ups explicitly deferred |
| 2026-04-25-traits-branch-review-findings | traits | partially-implemented | branch | Item 2 partially fixed; Item 1 (trace warning bug) NOT fixed; Items 3–8 open | Silent failure bug in `builtins.seq` at `aspect.nix:842` confirmed unresolved |
| 2026-04-26-class-emission-dedup-and-parametric-merge-design | traits | implemented | branch | All three fixes shipped with full test coverage | Parametric merge, include-level dedup, and unsatisfied-arg guard all live |
| 2026-04-26-direct-ref-aspects-design | stages | implemented | branch | `entityIncludes`/`rootIncludes` deleted; direct aspect refs in policies | Core structural changes present; `policy.aspects` not typed as NixOS option |
| 2026-04-26-eliminate-stages-implementation-notes | stages | superseded | branch | Describes intermediate `den.entityIncludes` state; final is `den.schema.<kind>.includes` | Notes stale; all spec goals met or exceeded on current branch |
| 2026-04-26-pipeline-simplification-design | pipeline-simplification | partially-implemented | branch | Targets 3 and 5 complete; Targets 1, 2, 4, 6 not started or partial | `runSubPipeline` extracted; fan-out generalized; `provides` removal deferred |
| 2026-04-26-pipeline-simplification-targets | pipeline-simplification | partially-implemented | branch | Targets 3+5 done; Targets 1, 2, 6 not started; Target 4 partial | Six-target audit; two immediate wins shipped; four remain |
| 2026-04-26-provides-removal-design | provides-removal | not-implemented | branch | Compat shim added; all deletion steps (Steps 1–7) pending; 114 template occurrences remain | `provides` API still fully live behind deprecation shim; migration not started |
| 2026-04-27-unified-policy-effects-design | policies | partially-implemented | branch | Three effect types, dispatch, aspect-included policies, traits — all shipped; `corePolicies` registry absent | Phase A–D complete; trait cycle detection and battery migration partially done |
| 2026-04-28-dead-letter-queue-unregistered-keys | pipeline-simplification | implemented | branch | DLQ handler, drain, emit helpers, template class registrations all shipped | Unregistered keys queued as dead letters, re-classified on context change |
| 2026-04-28-policy-context-enrichment-design | policies | implemented | branch | Fix 1 (schema/enrichment split) + Fix 2 (post-pipeline class wrapping) shipped | Non-schema resolve keys enrich current entity; class modules wrap post-pipeline |
| 2026-04-28-policy-pipeline-simplification | policies | implemented | branch | All Phases A–E shipped including originally-deferred Phase E | All 23 steps complete; plain functions, no `__functor`, no activation model |
| 2026-04-28-provides-compat-shims-design | provides-removal | implemented | branch | Pipeline handler + mutual-provider shim shipped | Compat layer for main→branch migration; prerequisite for provides deletion |
| 2026-04-29-flake-scope-pipeline-args-design | policies | implemented | branch | `pipelineOnly` utility + `den.provides.flake-scope` battery shipped | Aspect functions can destructure `lib`/`inputs`/`den` via enrichment mechanism |
| 2026-04-29-forward-elimination-trait-unification | traits | partially-implemented | branch | Trait delivery fix shipped; sub-pipeline elimination deferred | fxResolve reorder + subTraitModule; classImports sharing blocks elimination |
| 2026-04-29-scope-partitioned-pipeline-state | pipeline-simplification | in-progress | branch | Spec in progress | Per-entity pipeline state partitioning for forward sub-pipeline elimination |
| simplified-review-guide | legacy-removal | superseded | branch | Core architecture accurate; policy shape, stages, and per-policy handlers all stale | Review guide describes intermediate state; substantially superseded by Phase E |

---

## 2. Verdict Distribution

| Verdict | Count |
|---|---|
| implemented | 21 |
| partially-implemented | 13 |
| superseded | 12 |
| not-implemented | 4 |
| in-progress | 1 |
| **Total** | **51** |

The 4 "not-implemented" verdicts are: `effectful-pipeline-bootstrap-design` (abandoned before build), `provides-removal-design` (deferred, compat shim added instead), and two specs fully absorbed into broader supersession chains before any code landed (`fx-resolution-prototype-spec`, `commit-compaction-guide`). New since 2026-04-28: 5 specs implemented (DLQ, context enrichment, provides compat shims, flake-scope args, policy-pipeline-simplification Phase E), 1 partially implemented (forward-elimination/trait delivery), 1 in progress (scope-partitioned state).

---

## 3. Supersession Chains

### Chain A: aspect-rewrites → fx-pipeline → unified-pipeline

```
2026-04-04-provider-aware-excludes (implemented: single-level cascade on main)
    |
    v
2026-04-04-deep-provider-cascade (partial: listOf str provider paths, cascade extended)
    |
    v
2026-04-04-aspect-transforms-spec (superseded: transforms.nix API never created)
2026-04-04-chain-based-tracing    (superseded: replaced by chain-push/chain-pop effects)
    |
    v
2026-04-12-nix-effects-resolution-spec (implemented: freer monad pipeline)
2026-04-13-fx-resolution-prototype-spec (superseded: squash-merged wholesale)
2026-04-13-fx-adapters-spec            (superseded: meta.adapter → constraint registry)
2026-04-13-fx-ctx-apply-spec           (superseded: two-stage → unified pipeline)
2026-04-14-aspects-as-computations     (implemented: core aspectToEffect model)
2026-04-14-effectful-handlers          (implemented: nix-effects trampoline fix)
2026-04-14-fx-v2-architectural-direction (implemented: legacy removed)
2026-04-14-includes-chain-effects-design (implemented: chain provenance)
2026-04-14-unified-effects-pipeline    (implemented: canonical post-ship description)
    |
    v
2026-04-15-close-gaps-spec   (superseded: gaps closed, then further simplified)
2026-04-15-diag-rebase-spec  (superseded: diag written fresh on unified pipeline)
commit-compaction-guide      (superseded: squash-and-merge)
```

### Chain B: ctx-apply → unified-pipeline

```
2026-04-13-fx-ctx-apply-spec  (superseded: ctxApplyEffectful never created)
    |
    v
2026-04-17-fixedto-ctx-propagation  (superseded: __ctx → __scopeHandlers)
2026-04-18-ctx-as-data-design       (implemented: scope.provide wraps bind.fn)
2026-04-20-context-guard-design     (partially-implemented: guards eliminated; cleanup deferred)
```

### Chain C: ctx-as-data → __scopeHandlers (context carrier evolution)

```
Spec proposed __ctx inline field on aspects
    |
    v (superseded by)
__scopeHandlers as handler-closure map
    |
    v (reconstructed via)
ctxFromHandlers helper in aspect.nix
```

### Chain D: policies-design → unified-policy-effects → policy-pipeline-simplification

```
2026-04-20-policies-design            (partial: core machinery; phase 2 not wired)
    |
    v
2026-04-21-ctx-as-classes-design      (partial: three-concern separation done)
2026-04-23-effectful-pipeline-bootstrap-design  (NOT IMPLEMENTED: abandoned)
    |
    v  (bootstrap path abandoned; plain functions chosen instead)
2026-04-27-unified-policy-effects-design  (partial: A-D done; corePolicies absent)
    |
    v
2026-04-28-policy-pipeline-simplification  (FULLY IMPLEMENTED: all Phases A-E)
```

### Chain E: stages-elimination lineage

```
2026-04-21-ctx-as-classes-design       (den.stages introduced but immediately tombstoned)
    |
    v
2026-04-25-eliminate-stages-design     (FULLY IMPLEMENTED: resolve-entity, den.schema.X.includes)
2026-04-26-direct-ref-aspects-design   (IMPLEMENTED: entityIncludes/rootIncludes deleted)
2026-04-26-eliminate-stages-implementation-notes  (superseded: describes intermediate entityIncludes)
```

### Chain F: provides removal (in-progress)

```
2026-04-19-provide-to-effects-design   (partial: library done; osConfigurations not wired)
    |
    v
2026-04-26-pipeline-simplification-design  (partial: Target 1 not started)
2026-04-26-pipeline-simplification-targets (partial: Targets 1,2,6 not started)
    |
    v
2026-04-28-provides-compat-shims-design   (IMPLEMENTED: migration compat layer)
    |
    v
2026-04-26-provides-removal-design     (NOT IMPLEMENTED: compat shim landed; 114 template occurrences remain)
```

### Chain G: pipeline infrastructure (2026-04-28+)

```
2026-04-28-dead-letter-queue-unregistered-keys  (IMPLEMENTED: DLQ handler + drain)
2026-04-28-policy-context-enrichment-design     (IMPLEMENTED: enrichment routing + post-pipeline wrapping)
    |
    v
2026-04-29-flake-scope-pipeline-args-design     (IMPLEMENTED: rides enrichment mechanism)
2026-04-29-forward-elimination-trait-unification (PARTIAL: trait delivery shipped; elimination deferred)
    |
    v
2026-04-29-scope-partitioned-pipeline-state     (IN PROGRESS: per-entity state partitioning)
```

---

## 4. Gap Inventory

### 4a. Items explicitly deferred by specs

| Item | Deferral location | Blocking condition |
|---|---|---|
| `hasTrait` predicate | traits-and-structural-nesting §Follow-up | No active plan |
| Derived / computed traits (`derivedFrom`) | traits-and-structural-nesting §Follow-up | No active plan |
| Trait attenuation (`traitFilter`) | traits-and-structural-nesting §Follow-up | No active plan |
| Trait type validation (`den.traits.*.type` enforcement) | traits-and-structural-nesting §Follow-up | Field declared but no enforcement implemented |
| Diagram visualization of `emit-trait` effects | traits-and-structural-nesting §Follow-up | No active plan |
| Fleet policy topology diagram view | memory: project_diag_future_views | No active plan |
| Parametric aspect annotation in diagrams | memory: project_diag_future_views | No active plan |
| Unified aspect key type implementation | unified-aspect-key-type-design | Registry access resolved; implementation pending; blocks provides removal |
| Scope-partitioned pipeline state | scope-partitioned-pipeline-state | Spec in progress; blocks forward sub-pipeline elimination |
| `classifyKeys` full removal (Target 4) | pipeline-simplification-targets | Blocked on provides removal (Target 1) |
| `wrapClassModule` collision detection removal (Target 2) | pipeline-simplification-targets | Breaking change; scheduled last |
| Child shape normalization at source (Target 6) | pipeline-simplification-targets | Blocked on Target 2 |
| Wildcard policy scoping via schema analysis (Option 4) | policy-pipeline-simplification | `__entityKind` body guards used instead; Option 4 future improvement |
| `policy.resolve.shared` documentation | unified-policy-effects | "to be specified" clause never closed |
| Phase 2 orchestration wiring (cross-entity traits) | provide-to-effects-design, freeform-ignore | No orchestration in `osConfigurations.nix` |
| aspect-chain removal (fromAspect dependency) | memory: project_phase3_backlog | Deferred; forward-as-handler eliminates dependency |
| Vestigial module deletion (parametric.nix, take.nix, perHost-perUser.nix) | memory: project_phase3_backlog | Shim-only; deferred to post-migration |
| ctx-shim removal (den.ctx compat layer) | memory: project_phase3_backlog | Forwards to `den.schema.X.includes`; deferred |
| `emitCrossProvider` / `emitSelfProvide` deletion | pipeline-simplification-design | Blocked on 114 template migrations |
| `provides` option / `_` alias removal | provides-removal-design | Blocked on template migration |
| `den-brackets.nix` provides fallback cleanup | provides-removal-design | Blocked on provides option removal |

### 4b. Items specced but never shipped

| Item | Spec | Notes |
|---|---|---|
| `resolve-policy` / `resolve-target` effects | effectful-pipeline-bootstrap | Bootstrap approach abandoned; superseded by plain-function dispatch |
| Per-policy `handlers` field with `scope.provide` per-transition | effectful-pipeline-bootstrap, policies-design | Policy submodule removed; mechanism impossible with plain functions |
| `corePolicies` registry in `den.lib` | unified-policy-effects | Core policies merged directly into `den.policies` |
| `den.schema.<kind>.policies` formal typing | ctx-as-classes, simplified-review-guide | Accepted via freeform `lazyAttrsOf raw`; no declared NixOS option |
| `den.ctx.*.into` forwarding | ctx-as-classes | Stub in compat shim; no migration path for existing users |
| `flakeModules/fxPipeline.nix` flake module | fx-feature-complete | Never created; no `den.flakeModules.fxPipeline` consumer API |
| `ctxTraceHandler` / `ctx-traverse` in production pipeline | fx-feature-complete | Effect exists for tests only; `entityKind` derivation used instead |
| `__ctxStage` / `__ctxKind` stage tagging | fx-feature-complete | Not implemented; `entityKind` from `includesChain` ancestry used |
| `deny.hosts.<sys>.<name>.excludes` entity-level option | aspect-transforms-spec | Only policy `routing.excludes` exists; no entity-level option |
| `replaces` sugar (substitute shorthand) | aspect-transforms-spec | Not implemented |
| `resolve'` as a public power API | aspect-transforms-spec | `fxFullResolve` is internal; no verbatim `resolve'` |
| `toAspectPath` as public API | deep-provider-cascade | `identity.aspectPath` is internal; different signature |
| `__placedBy` annotation on substitutions | deep-provider-cascade | Only `meta.replacedBy` name string exists |
| Graph-based multi-hop provide-to routing | provide-to-effects | Only single-hop `routing.from == routing.to` checked |
| `aspectRollback on policy.exclude` | unified-policy-effects | Rollback path not confirmed implemented |
| Aspect-level `provides` deprecation warning | traits-and-structural-nesting | `structuralKeysSet` still lists "provides" silently |
| Conditions system (resumable errors) | nix-effects-resolution-spec | `strict.nix` throws directly; no handler-swap |
| `has-aspect` as inline effect during computation | nix-effects-resolution-spec | Runs separate sub-pipeline; not an in-computation effect |
| `inputs.nix-effects` validation assert | fx-feature-complete | Silent tarball fallback instead |

### 4c. Items partially shipped (compat shim instead of full implementation)

| Item | Current state | Full implementation |
|---|---|---|
| `provides` API removal | `emitCrossProvideShims` compat shim with deprecation warnings; `provides-compat.nix` handler + `mutual-provider-shim.nix` shipped | Delete `emitSelfProvide`, `emitCrossProvider`, `provides` option, remove from `structuralKeysSet` |
| `perHost-perUser.nix` | Deprecated shims with `lib.warn`; logic via `perCtx` / `constantHandler` | Delete file after downstream migration |
| `parametric.nix` | Shim-only (`fixedTo`, `expands` delegate to `mkFixedTo`) | Delete after `aspect-chain` and template migration |
| `take.nix` | Shim-only (all forms emit `lib.warn`) | Delete after all callers removed |
| `den.ctx` | Forwarded to `den.schema.X.includes` via ctx-shim.nix with warnings | Delete shim after migration period |
| Cross-entity trait distribution | `distributeCrossEntityTraits` library function exists | Wire `osConfigurations.nix` to call distribute + pass `crossEntityTraits` |
| `include-level dedup rollback on policy.exclude` | `include-unseen` rollback effect exists for constraint exclusions | Verify rollback fires for policy `exclude` effects |
| `policy.aspects` typing | Untyped `routing.aspects` list in transition handler | Declare as `listOf providerType` NixOS option |
| Forward sub-pipeline isolation | Sub-pipeline per forward spec; `classImports` shared across entities | Per-entity classImports partitioning (scope-partitioned-pipeline-state spec in progress) |
| `ownerIdentity` exclusion rollback for compat policies | Field stored in `tree.nix:304` but never read by `dispatchPolicyIncludesHandler` — compat policies fire even when source aspect excluded | Read `ownerIdentity` during dispatch to gate synthesized compat policies |
| Tests for `provides-compat.nix` compat shim | No tests for old `provides.X` syntax (wildcard, named-target, `__fn`/`__functor` value shapes, mutual-provider shim scenarios) | Add 10+ CI test cases covering compat shim paths |

---

## 5. Known Bugs from Specs

| Bug | Location | Severity | Status |
|---|---|---|---|
| `builtins.seq (map ...) null` never fires traces — unregistered aspect class key warnings are silently swallowed | `nix/lib/aspects/fx/aspect.nix:842-845` | Low (was Medium) — DLQ now handles unregistered keys instead of relying on trace warnings | Partially mitigated by DLQ (2026-04-28); trace fix (`builtins.foldl'`) still not landed |
| `modulesPath` dead destructure in `pipeline.nix:303` | `nix/lib/aspects/fx/pipeline.nix:303` | Low — trivial dead code | Open |
| `emitNestedAspect` silently drops function-valued keys with no trace warning (`aspect.nix:800`) | `nix/lib/aspects/fx/aspect.nix:800` | Low-medium — function-valued nested keys coerced to `{}` without user feedback | Open |
| `traitCollectorHandler` captures root `ctx` at construction time, not `state.currentCtx` — Tier 2 classification can misfire if trait-providing aspects have args satisfied only by transitions | `nix/lib/aspects/fx/handlers/trait.nix:113` | Low in practice (safe fallback to Tier 3 added in `07055169`) — but architectural limitation | Partially addressed; full fix (read `state.currentCtx`) not landed |
| `provide-to.nix` handler comment documents stale "aspects declare `provide-to.${label}`" pattern — actual mechanism is sibling sub-pipeline trait capture | `nix/lib/aspects/fx/handlers/provide-to.nix` header | Low — misleading documentation only | Open |
| `scope.run` limitation (deep handler bug): `scope.run` via `rotateInterpret` drops events in nested scope — den avoids it using `scope.provide`, but `rotateInterpret` path is not production-validated | `sini/nix-effects` | Low (not on production path) — `scope.stateful` effectively disabled | Mitigated by using `scope.provide` throughout |
| `perHost-perUser.nix` `into` compat slot: `den.ctx.*.into` option exists in ctx-shim but is declared `raw` with no forwarding — users who wrote custom `den.ctx.host.into.my-stage` policies get silence, not an error or migration message | `modules/compat/ctx-shim.nix` | Medium — silent migration failure for `into` users | Open |
| `ownerIdentity` stored but never read: `provides-compat.nix` writes `ownerIdentity = nodeIdentity` to registered compat policies, but `dispatchPolicyIncludesHandler` in `tree.nix:362-385` filters only by `functionArgs` — compat policies continue firing when source aspect is excluded | `nix/lib/aspects/fx/handlers/tree.nix:362-385` | Low-medium — excluded aspects leak compat include effects | Open |
| Trait arg consumption via function signature broken: `test-enrichment-with-traits` in policy-context-enrichment tests explicitly notes that `{ greeting, ... }:` destructuring does not receive trait values — only trait *emission* coexists with enrichment, not trait *consumption* via function args | `templates/ci/modules/features/policy-context-enrichment.nix:362` | Low-medium — trait function args silently unfilled | Open (separate tracking) |

---

## 6. Component Summaries

### aspect-rewrites

The original aspect-rewrites work (provider-aware excludes, deep provider cascade) shipped incrementally on main and is fully functional. However, the spec's proposed user-facing API surface (`transforms.nix`, `resolve'`, `transforms.exclude/substitute/trace`) was never created verbatim — the functionality was absorbed into the fx pipeline as handler-level behaviour rather than user-constructed transform functions. The public API now uses `meta.excludes`, `meta.handleWith`, and `tracingHandler`/`structuredTraceHandler`. The entity-level `excludes` option described in the spec was not carried forward; it lives on policies as `routing.excludes` instead. All four specs in this group are effectively closed — either implemented in spirit or explicitly superseded by the fx pipeline.

### hasAspect

Fully implemented on both main (legacy adapter path) and `feat/fx-pipeline` (fx `pathSet` state). The public API (`entity.hasAspect ref`, `.forClass`, `.forAnyClass`) is stable and unchanged. Internal implementation differs significantly between branches: main uses `resolve.withAdapter adapters.collectPaths`; branch uses `fxFullResolve` + `state.pathSet`. The `collectPaths` and `oneOfAspects` adapters were removed on branch when `adapters.nix` was deleted. The `__ctxId` augmentation extends hasAspect to match fan-out instances by base path, a capability not in the spec.

### fx-pipeline

The core fx pipeline is fully shipped and is the sole resolution path on both main and `feat/fx-pipeline`. The freer monad architecture (aspectToEffect compiler, handler-owned recursion, constraint registry, chain provenance, `emit-class`/`emit-include`/`into-transition` effects) matches the specs in spirit with consistent naming evolution (emit-X pattern, "constraint" not "adapter"). Notable gaps are the conditions system (resumable errors), `has-aspect` as an inline effect, and `ctxTraceHandler`/`__ctxStage` tagging — all deferred or replaced by simpler mechanisms. The nix-effects dependency is pinned to `sini/nix-effects` rev `cf958815`; `scope.run` / `rotateInterpret` path is partially exercised and not production-validated. The diag library was written fresh on the unified pipeline, bypassing the planned rebase entirely.

### legacy-removal

The legacy removal work is complete. `ctx-apply.nix`, `resolve.nix`, `adapters.nix`, `statics.nix`, `stage-types.nix`, `resolve-stage.nix`, `policy-dispatch.nix`, `modules/fxPipeline.nix` are all gone. The `parametric.nix` and `take.nix` files survive as deprecation-warning shims. Context propagation via `__ctx` was implemented then superseded by `__scopeHandlers` + `scope.provide`, a cleaner mechanism. The provide-to spec is the notable gap in this group: the library (`provideToHandler`, `distribute-cross-entity.nix`, `fxResolve { crossEntityTraits }`) is complete, but no production orchestration wires phase 2 — `osConfigurations.nix` never calls `distributeCrossEntityTraits`, making cross-entity trait injection dead code for real NixOS configs.

### policies

The policy system went through three design generations in these specs. The `effectful-pipeline-bootstrap` design (resolve-policy / resolve-target effects, per-policy `handlers` field) was abandoned before it was built when the `den.stages` deletion and `policy-dispatch.nix` deletion made its foundation impossible. The final design — plain functions returning typed `policy.resolve/include/exclude` effect constructors, dispatched by direct iteration in `transition.nix`, with `__entityKind` body guards — is fully shipped including the originally-deferred Phase E steps. The `corePolicies` registry was never built; core policies live directly in `den.policies`. Phase 2 cross-entity distribution (the flagship fleet use case) is wired only at the library level; `osConfigurations.nix` is not connected. Aspect-included policies (`policyFns` key on aspects) are live and used by the HM/maid/hjem batteries. Policy rollback on `exclude` effects is unverified.

**New since 2026-04-28:** Policy context enrichment shipped — non-schema resolve keys (e.g., `isNixos`, `isDarwin`) now route as context enrichment of the current entity rather than creating empty child entities. All `wrapClassModule` calls deferred to a post-pipeline pass, ensuring policy-injected context is available regardless of class module form. Flake-scope pipeline args shipped — aspect functions can destructure `lib`/`inputs`/`den` directly via a `den.provides.flake-scope` battery that rides the enrichment mechanism, with `pipelineOnly` collision policy preventing conflicts at NixOS evaluation. Two implementation drift items: (1) enrichment-only arg stripping from `__functionArgs` post-wrap (prevents NixOS infinite recursion probing `_module.args.isNixos`, not in spec); (2) `drain-dead-letters` runs inside each enrichment iteration, not once at the end — more correct but diverges from spec ordering. The `test-enrichment-with-traits` note confirms trait arg consumption via function signature is a separate pre-existing bug.

### class-modules

Fully shipped with scope extensions beyond the original spec. `wrapClassModule` supports partial application (den args pre-applied), full application (when all args are den args), trait consumption via lazy `_den.traits` thunks, and deferred-import nesting. Collision policies at three levels (`aspect.meta.collisionPolicy`, entity `collisionPolicy`, `den.config.classModuleCollisionPolicy`) are implemented with companion validator modules using deferred thunk evaluation for NixOS fixed-point safety. The only divergence from spec is `ctxFromHandlers` replacing `aspect.__ctx or {}` — the correct adaptation given `__ctx` removal.

### stages

The `den.stages` elimination is fully complete. `den.stages` is tombstoned (`modules/removed-stages.nix` throws on use). `resolveEntity` reads from `den.schema.<kind>.includes` with self-provide, framework aspects, and schema includes merged before transitions. The three-layer model (aspects / policy-included aspects / schema includes) is active. `entityIncludes`, `entityProvides`, `rootIncludes`, and `resolve-stage` are all deleted. The `policy.aspects` concept from the spec was not typed as a NixOS option; aspects travel as untyped `routing.aspects` lists carrying direct refs. The `ctx-shim.nix` forwards to `den.schema` includes (not `den.aspects` as one spec section proposed). `hostFramework` is hardcoded in `resolveEntity` rather than via a policy.

### traits

The traits pipeline is substantially complete. All five migration phases (schema registry, structural detection, three-tier trait collection, `_den.traits` injection, cross-entity distribution, forward battery schema integration) are shipped. Three follow-up items (`hasTrait`, derived traits, trait attenuation) are explicitly deferred with no active plan. The known silent-failure bug (`builtins.seq` at `aspect.nix:842` not firing traces for unregistered keys) was flagged before feat/traits merge and is still open. The `provides` structural key has no deprecation warning at the aspect level (only at bracket path level). Cross-entity orchestration has the same gap as the policies component: `fxResolveTree` does not call `distributeCrossEntityTraits`.

**New since 2026-04-28:** Forward trait delivery fix shipped (commit b44fd284). fxResolve reorder ensures `traitModule` is available when forwards run; sub-pipelines synthesize `subTraitModule` from `sub.traits` (Tier 1/2 only). Sub-pipeline elimination was attempted but failed — `classImports` buckets are shared across entities, causing cross-contamination on multi-user hosts. Sub-pipeline isolation is structurally necessary; scope-partitioned pipeline state spec is in progress to address this.

### pipeline-simplification

Of the six simplification targets, Targets 3 (`runSubPipeline` extraction) and 5 (fan-out generalization via `resolve.shared`) are complete. Target 4 (`classifyKeys` → declared schemas) is partial — registry dispatch was added but full static annotation was not. Targets 1, 2, and 6 are not started. The `host-aspects` battery (a separate item in this group) is fully shipped on both main and branch. The overall arc is clear: the "immediate wins" are done; the remaining targets require either a migration pass (Target 1: 114 template occurrences) or are intentionally last due to breaking-change risk (Targets 2 and 6).

**New since 2026-04-28:** Dead letter queue shipped — unregistered aspect keys no longer silently drop. Instead, they queue as dead letter effects and re-classify when context changes (deferred includes registering new classes/traits). Template `den.classes` registrations added to prevent DLQ for known classes. This partially addresses Target 4 at runtime without requiring static annotation. Two drift items worth tracking: drain uses `state.traitSchemas` (dynamic) not the static `traitRegistry` described in spec (more powerful, not a regression); `deadLetterHandler` dual-writes to both `state.deadLetterQueue` and `state.scopedDeadLetterQueue` (scope-partitioned state counterpart, not in original spec).

### provides-removal

This is the largest active gap. The spec's seven deletion steps (migrate templates, delete `emitSelfProvide`, delete `emitCrossProvider`, remove `provides` option, remove from `structuralKeysSet`, clean up `den-brackets.nix`) have not started. The compat shims spec (2026-04-28-provides-compat-shims-design) is now fully implemented — `provides-compat.nix` handler + `mutual-provider-shim.nix` provide the migration bridge. `emitCrossProvideShims` intercepts old `provides.X` patterns, emits deprecation warnings, and re-routes to `policy.include` effects at runtime. The legacy machinery (`emitSelfProvide`, `emitCrossProvider`, `provides` option, `_` alias) remains fully live. Template migration has not begun — approximately 114 occurrences across 30 files remain. Provides removal is now blocked on the unified aspect key type design (registry access prerequisite resolved, implementation pending). This work is the primary blocker for subsequent pipeline cleanup (classifyKeys removal, child shape normalization).

Two known gaps in the compat shim implementation: (1) no tests for the `provides-compat.nix` compat shim itself — the spec listed 10 test cases for old `provides.X` syntax paths and 4 for the mutual-provider shim, none shipped; CI tests cover only new `policies.*` routing syntax; (2) `ownerIdentity` exclusion rollback is inert — the field is stored in `tree.nix:304` but never read during policy dispatch (`dispatchPolicyIncludesHandler` filters only by `functionArgs`), meaning synthesized compat policies continue firing even when their source aspect is excluded.
