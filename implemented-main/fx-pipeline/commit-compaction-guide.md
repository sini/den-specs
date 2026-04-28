# Commit Compaction Guide — feat/fx-resolution

**Branch:** 131 commits on `feat/fx-resolution` since `main`
**Target:** ~25-30 clean logical commits

## Compaction strategy

Group commits into logical units matching the final architecture. Drop noise (WIP, reverts, abandoned approaches, intermediate docs). Squash fixup chains into their parent feature.

## Proposed logical commit sequence

After compaction, the branch should read as this sequence of clean commits:

1. **feat(fx): effects-based resolution prototype** — scaffold, bind.fn, initial effects
2. **feat(fx): provide-class effect + handler** — class emission, moduleHandler→classCollectorHandler
3. **feat(fx): constraint registry (exclude/substitute/filterBy)** — stateful query effects, constraint constructors
4. **feat(fx): includes chain provenance (chain-push/chain-pop)** — replaces __parent
5. **feat(fx): trace handlers derive parent from chain state** — structuredTraceHandler, tracingHandler
6. **refactor(fx): split adapters.nix into identity, constraints, includes, trace** — module decomposition
7. **feat(fx): meta.handleWith + meta.excludes** — new constraint API, revert meta.adapter to function-only
8. **refactor(fx): rename adapter→constraint terminology** — effects, handlers, state keys, tests
9. **feat(fx): aspectToEffect compiler** — the unified aspect compiler (compileStatic, compileFunctor)
10. **feat(fx): emit-include handler with effectful resume** — handler-owned recursion
11. **feat(fx): into-transition handler with scope.stateful** — context transitions
12. **feat(fx): constantHandler replaces parametric/static/context handlers**
13. **refactor(fx): module wiring — { lib, den } only, no init, barrel default.nix**
14. **refactor(fx): remove old resolve code** — delete ctx-apply, ctx-stage, resolve-deep, resolve-handler, resolve-one, wrap, resolve-legacy
15. **feat(fx): mkPipeline + fxResolve + fxResolveTree** — pipeline edge, defaultHandlers, composeHandlers
16. **feat(fx): fxPipeline option + flakeModules.fxPipeline** — A/B gate
17. **fix(fx): close pipeline gaps** — wire effects, dedup, aspect-chain compat shim
18. **fix(fx): wrapChild normalization** — type-system-merged aspects, parametric includes, bare lambdas
19. **fix(fx): restore fx test suites** — move from fxCrash to features, adapter tests on legacy
20. **fix(fx): scope.stateful + nix-effects updates** — state preservation on effect rotation
21. **refactor(fx): review feedback** — conditional guard, curried parametrics, fx.seq, provides. alias
22. **chore: nix-effects loading fallback** — locked fetchTarball when no flake input
23. **feat: ci-fast recipe** — nix-eval-jobs parallel test eval
24. **docs: consolidated fx pipeline spec** — unified spec, spec corrections
25. **refactor(fx): code review cleanup** — dead code removal, deduplication, readability, remaining suggestions
26. **fix(fx): deep review fixes** — transition handler bugs, compiler scope, constraint accumulation, incremental pathSet, boundary hardening

---

## Commits to DROP (remove entirely)

These are reverted, WIP, abandoned, or doc-only commits whose content no longer exists in the tree.

| Hash | Subject | Reason |
|------|---------|--------|
| `e71917f` | WIP(fx): option E with hasLegacyArgs filter + shim | Reverted by a9b07fb |
| `a9b07fb` | Revert "WIP(fx): option E..." | Revert of above |
| `5824889` | WIP(fx): option E — aspects never functors | Abandoned approach, superseded by 033b46e |
| `a893c8c` | docs: gate ctxApply at entry point | Documents abandoned option E approach |
| `0c9be04` | enable fxPipeline by default | Superseded by 5b2541d which re-enables properly |
| `df801bd` | docs: fx v2 architectural direction | Consolidated into spec |
| `c72f58d` | docs: spec for review | Intermediate spec draft, consolidated |
| `4f6318b` | docs: spec fx include-chain | Consolidated into spec |
| `309c45d` | docs: spec adapter -> handleWith | Consolidated into spec |
| `60d8415` | docs: spec v3 | Consolidated into spec |
| `6eed743` | docs: specs for closing gaps and diag rebase | Consolidated into spec |
| `f37a2d0` | docs: gate aspect-chain removal | Consolidated into spec |
| `33938c6` | docs: update close-gaps spec | Consolidated into spec |
| `b8a8eed` | docs: resume for v3 pipeline session | Session notes, not in final tree |
| `57d5037` | docs: fx test restoration plan | Triage plan, consolidated into spec |
| `abe4321` | chore: collect docs | Housekeeping, superseded by spec consolidation |

**Total drops: 16 commits**

## Commits to SQUASH (fold into parent)

### Chain 1: Early prototype → resolveOne returns Computation
Squash into one "effects-based resolution" commit.

| Hash | Subject | Squash into |
|------|---------|-------------|
| `a2dae036` | feat(fx): scaffold effects-based resolution prototype | → keep as base |
| `d467c6bd` | feat(fx): aspect → computation translation via bind.fn | → squash into a2dae036 |
| `b42c2ca6` | feat(fx): parametric and static context handlers | → squash into a2dae036 |
| `de21d328` | feat(fx): effects-based resolve with deep recursion | → squash into a2dae036 |
| `32bbd213` | feat(fx): rotate-based strict resolution | → squash into a2dae036 |
| `47e0c714` | refactor(fx): wire missingArgError, add strict mode | → squash into a2dae036 |
| `f28d82ee` | refactor(fx): resolveOne returns Computation | → squash into a2dae036 |
| `eccb6dba` | refactor(fx): replace chain threading with scoped bind chains | → squash into a2dae036 |
| `a632149e` | fix(fx): add context handlers to tests | → squash into a2dae036 |

### Chain 2: provide-class + module handler
| Hash | Subject | Squash into |
|------|---------|-------------|
| `3ffb3642` | feat(fx): provide-class effect replaces moduleHandler | → keep as base |
| `1f1c2c43` | feat(fx): moduleHandler and collectPathsHandler | → squash into 3ffb3642 |

### Chain 3: Constraint effects (adapter→constraint)
| Hash | Subject | Squash into |
|------|---------|-------------|
| `a09c0f11` | feat(fx): exclude/replace as stateful query effects | → keep as base |
| `7b0567b8` | feat(fx): excludeAspect and substituteAspect handlers | → squash into a09c0f11 |
| `fba0a8d3` | feat(fx): includeIf conditional includes | → squash into a09c0f11 |
| `4bdf3603` | test(fx): adapter integration tests | → squash into a09c0f11 |
| `eda995e8` | fix(fx): exact test assertions, nested adapter test | → squash into a09c0f11 |
| `44638e87` | feat(fx): filterAspect, pure adapter exports | → squash into a09c0f11 |
| `4eb2a156` | fix: adapter type accepts v2 records | → squash into a09c0f11 |

### Chain 4: ctx-emit + review fixes
| Hash | Subject | Squash into |
|------|---------|-------------|
| `2534caa7` | feat(fx): ctx-emit effect | → keep as base |
| `fe2c1c2c` | fix(fx): review fixes — export ctxEmitHandler | → squash into 2534caa7 |

### Chain 5: Includes chain (parentPath → chain-push/chain-pop)
| Hash | Subject | Squash into |
|------|---------|-------------|
| `55e61b16` | feat(fx): replace parentPath with chain-push/chain-pop | → keep as base |
| `72597668` | feat(fx): trace handlers derive parent from includesChain | → squash into 55e61b16 |
| `7b7c7cbe` | fix(fx): remove last __parent comment reference | → squash into 55e61b16 |
| `f3fb8276` | feat(fx): parametric detection in wrapIdentity, __parent tracking | → squash into 55e61b16 |

### Chain 6: Trace handlers
| Hash | Subject | Squash into |
|------|---------|-------------|
| `8b7c6649` | feat(fx): structuredTraceHandler and ctxTraceHandler | → keep as base |
| `8d9d4992` | fix(fx): tighten trace test assertions | → squash into 8b7c6649 |
| `6fcf0671` | feat(fx): tracingHandler, composeHandlers docs | → squash into 8b7c6649 |

### Chain 7: adapter→constraint rename
| Hash | Subject | Squash into |
|------|---------|-------------|
| `8867dd0d` | refactor(fx): rename adapter constructors to constraints | → keep as base |
| `c8f3fa8e` | refactor(fx): rename adapter handler/effects/state | → squash into 8867dd0d |
| `aaf3f289` | refactor(fx): update all fx tests to constraint terminology | → squash into 8867dd0d |

### Chain 8: meta.handleWith + meta.excludes
| Hash | Subject | Squash into |
|------|---------|-------------|
| `e4b88fc4` | feat(fx): add meta.handleWith and meta.excludes | → keep as base |
| `0a01fd46` | feat(fx): carry handleWith in identity envelope | → squash into e4b88fc4 |
| `44981119` | fix: update filterIncludes comment for handleWith | → squash into e4b88fc4 |
| `91b70150` | chore(fx): clean up stale adapter terminology | → squash into e4b88fc4 |

### Chain 9: Module split + review fixes
| Hash | Subject | Squash into |
|------|---------|-------------|
| `a6148228` | refactor(fx): extract trace.nix, hasAdapter → handlers field | → keep as base |
| `7dc0b48c` | refactor(fx): extract includes.nix with includeIf | → squash into a6148228 |
| `d6d0ac06` | refactor(fx): split handlers.nix into ctx.nix and tree.nix | → squash into a6148228 |
| `725968d7` | refactor(fx): split resolve.nix into wrap, resolve-one, etc. | → squash into a6148228 |
| `522a844b` | refactor(fx): split ctx-apply and resolve-deep | → squash into a6148228 |
| `eeee78ae` | refactor(fx): decompose large functions into helpers | → squash into a6148228 |
| `2d2840fc` | fix(fx): review fixes — dedup, DRY helpers | → squash into a6148228 |
| `60269db3` | refactor(fx): clarify trace handler roles, consolidate exports | → squash into a6148228 |

### Chain 10: Unified pipeline (aspectToEffect)
| Hash | Subject | Squash into |
|------|---------|-------------|
| `de8567c9` | feat(fx): aspectToEffect — the aspect compiler | → keep as base |
| `b8ba6b22` | feat(fx): aspectToEffect handles into-transitions and self-provide | → squash into de8567c9 |
| `28949943` | feat(fx): emit-include + into-transition handlers | → squash into de8567c9 |
| `0004a50d` | refactor(fx): rename handlers and effects to final names | → squash into de8567c9 |
| `7f0ee80a` | refactor(fx): rewrite module wiring — { lib, den } only | → squash into de8567c9 |
| `74ee1c14` | feat(fx): unified pipeline via aspectToEffect — remove old resolve/ctx-apply | → squash into de8567c9 |
| `9383fa91` | refactor(fx): update all tests for unified effects pipeline | → squash into de8567c9 |
| `579cd795` | chore(fx): cleanup — remove broken resolve-legacy | → squash into de8567c9 |

### Chain 11: Close gaps + aspect-chain shim
| Hash | Subject | Squash into |
|------|---------|-------------|
| `36bed4f8` | fix(fx): close remaining pipeline gaps | → keep as base |
| `033b46ec` | fix(fx): provide aspect-chain handler for legacy provider functions | → squash into 36bed4f8 |
| `7a9d1f23` | refactor(fx): gate aspect-chain on den.fxPipeline | → squash into 36bed4f8 |
| `6c68cb52` | refactor(fx): remove dead ctx handlers | → squash into 36bed4f8 |

### Chain 12: scope.stateful + nix-effects
| Hash | Subject | Squash into |
|------|---------|-------------|
| `48e8e6f9` | use scope.stateful to preserve state on effect rotation | → keep as base |
| `faf04e87` | fix: scope.stateful takes handlers directly | → squash into 48e8e6f9 |
| `9daeba64` | update nix-effects | → squash into 48e8e6f9 |
| `59a5fc13` | flake update | → squash into 48e8e6f9 |

### Chain 13: wrapChild normalization
| Hash | Subject | Squash into |
|------|---------|-------------|
| `fc252d90` | fix: restore 61 fx test suites | → keep as base |
| `6e77b0b5` | fix: preserve class keys in wrapChild | → squash into fc252d90 |
| `9a99d691` | fix: filter unresolved parametric includes in wrapChild | → squash into fc252d90 |
| `5e05fd0d` | fix: refine wrapChild include filter | → squash into fc252d90 |
| `a39f09e6` | fix: filter coerced __functionArgs in wrapChild | → squash into fc252d90 |
| `488b6133` | fix: filter non-pipeline args from bare lambda wrapChild | → squash into fc252d90 |
| `34c172e5` | fix: re-merge bare module functions through aspect type system | → squash into fc252d90 |
| `5b2541d4` | fix: resolve functor attrsets in fxResolveTree | → squash into fc252d90 |

### Chain 14: Test restoration + opt-outs
| Hash | Subject | Squash into |
|------|---------|-------------|
| `d330b80e` | fix: restore remaining fxCrash tests | → keep as base |
| `5703bfa0` | fix: add fxPipeline=false for flake-parts tests | → squash into d330b80e |
| `1e3498a1` | failing tests at _fxCrash | → squash into d330b80e |
| `cb65de12` | move deadbugs to features | → squash into d330b80e |
| `c0c921d0` | enable couple more of tests | → squash into d330b80e |
| `136ef4c0` | test: move performance.namespace to features/ | → squash into d330b80e |

### Chain 15: Spec consolidation
| Hash | Subject | Squash into |
|------|---------|-------------|
| `28b5a580` | docs: consolidate fx pipeline design docs into unified spec | → keep as base |
| `ea856062` | docs: fix spec inaccuracies found during code review | → squash into 28b5a580 |

## Standalone commits to KEEP as-is

| Hash | Subject | Reason |
|------|---------|--------|
| `d3a8a735` | test(fx): regression reproductions for #413/#423, #426, #437, meta carryover | Standalone regression tests |
| `4f4d9ef7` | test(fx): end-to-end pipeline — host/user fanout, providers, adapters, includeIf | Standalone e2e test |
| `e39dac38` | refactor(fx): remove nix-effects API tests, keep den behavior tests | Clean standalone removal |
| `20dab3e8` | feat(fx): context stage tagging on provider includes | May fold into chain 10 |
| `bbcfe550` | feat(fx): den.fxPipeline option + flakeModules.fxPipeline | Standalone feature |
| `b5514af6` | feat(fx): wire fxPipeline flag into resolution | Standalone feature |
| `3ef414d3` | refactor: apply review feedback — conditional guard, curried parametrics, fx.seq | Clean standalone |
| `f0d69004` | refactor: replace _. alias with provides. across codebase | Clean standalone |
| `ae474a6b` | use locked nix-effects when no input exist | Standalone (or fold into chain 12) |
| `0897f988` | use fx name | Standalone (or fold into chain 12) |
| `5971321f` | fix(fx): review fixes — excludes in withIdentity, dead imports, handlerList rename | Fold into chain 7 or 8 |
| `5d43060c` | fix(fx): filter self-references from includesChain | Fold into chain 5 |
| `3d4cbe4b` | chore: update nix-effects with effectful handler support | Standalone dep update |
| `e94a83b0` | refactor(fx): update all tests for resolveAspect API | Fold into chain 1 |
| `8a7f7409` | feat(fx): switch to resolveAspect | Fold into chain 1 |
| `10931703` | feat(fx): add resolveAspect + resolveIncludeHandler | Fold into chain 1 |
| `dec451bb` | feat(fx): add chainHandler for includes-path tracking | Fold into chain 5 |
| `bf95f284` | feat(fx): add scope field to adapter API | Fold into chain 3 |
| `0979f0a2` | feat(fx): scope-aware adapter registration | Fold into chain 3 |
| `f2e1969a` | fix(fx): deduplicate rawSelfPath, add adapterFilters | Fold into chain 3 |
| `7b7c7cbe` | fix(fx): remove last __parent comment ref | Fold into chain 5 |
| `72e4901f` | fix: correct relative path in import-tree test | Fold into chain 14 |
| `79813600` | feat: add ci-fast recipe | Standalone |
| `2d1e4b5f` | feat(fx): effectful ctxApply | Fold into chain 1 or 10 |
| `001fae70` | feat(fx): fxFullResolve | Fold into chain 10 |
| `390ceba9` | fix(fx): moduleHandler location info | Fold into chain 2 |
| `c64008b5` | fix(fx): DRY fxFullResolve via mkPipeline | Fold into chain 10 |
| `a088a06a` | feat(fx): port aspectPath, pathKey, toPathSet, tombstone | Fold into chain 1 |

## Summary

| Action | Count |
|--------|-------|
| Drop entirely | 16 |
| Squash (chains 1-18) | ~89 commits → ~18 |
| Keep standalone | ~12 |
| **Result** | **~26 clean commits** |

131 commits → ~26 clean commits, organized to match the final spec's logical structure.

## Code review commits (2026-04-16)

These 6 commits should squash into one or two logical commits at the end of the sequence.

### Chain 16: Spec consolidation + corrections

| Hash | Subject | Action |
|------|---------|--------|
| `28b5a580` | docs: consolidate fx pipeline design docs into unified spec | → keep as base |
| `ea856062` | docs: fix spec inaccuracies found during code review | → squash into 28b5a580 |
| `b6e0b701` | docs: add wrapChild rules, pipeline entry points, fxPipeline gating to spec | → squash into 28b5a580 |

### Chain 17: Code review refactoring

| Hash | Subject | Action |
|------|---------|--------|
| `6eeeec7e` | refactor(fx): remove dead code — unused args, exports, fallbacks | → keep as base |
| `def9e947` | refactor(fx): deduplicate trace entry construction and constraint constructors | → squash into 6eeeec7e |
| `b9f62a77` | refactor(fx): improve readability — extract helpers, add protocol headers | → squash into 6eeeec7e |
| `b078250e` | refactor(fx): use fx.seq + map for side-effect emissions, extract chainWrap | → squash into 6eeeec7e |
| `0e7d34e1` | refactor(fx): batch 4 — cleanup suggestions and legacy markers | → squash into 6eeeec7e |

### Chain 18: Deep review fixes

| Hash | Subject | Action |
|------|---------|--------|
| `7a6724ca` | fix(fx): fix transition handler — foldl accumulator, parent ctx threading, diagnostics | → keep as base |
| `641320c5` | fix(fx): chainWrap scope encloses all resolution phases, merge parametric meta | → squash into 7a6724ca |
| `9699cd4f` | refactor(fx): boundary hardening — extract normalizeModuleFn, validate refs, document semantics | → squash into 7a6724ca |
| `524b8868` | refactor(fx): maintain pathSet incrementally in collectPathsHandler | → squash into 7a6724ca |
| `c15415c7` | fix(fx): accumulate constraints per identity, assert chain-pop, document composeHandlers | → squash into 7a6724ca |
| `2ebe6e9e` | docs: sync spec with review fix implementations | → squash into chain 16 base (28b5a580) |
