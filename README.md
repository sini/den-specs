# den-specs

Design specs for [den](https://github.com/denful/den), a NixOS configuration framework built on algebraic effects via [nix-effects](https://github.com/sini/nix-effects).

## The Story

Den's `feat/fx-pipeline` branch replaced the legacy aspect resolution pipeline with an algebraic effects architecture. The initial implementation introduced scope-partitioned state, typed policy effects, and entity-schema-driven resolution — but accumulated tech debt as each new mechanism (transitions, DLQ, traits, forward sub-pipelines) interacted with every other. A cleanup arc deleted ~3000 lines by recognizing that `scope.provide` — the effect system's own scoping primitive — could replace most of the manually-built machinery. Transitions became `installPolicies` with lexical `scope.provide`, the DLQ was eliminated in favor of direct class emission, and the trait system was removed for a simpler reimplementation.

The result is a pipeline where policy dispatch, context expansion, and entity resolution are composed from small effect handlers rather than orchestrated by monolithic dispatchers. Scope partitioning gives each entity its own state partition without sub-pipeline isolation. Routes and provides deliver content post-pipeline from those partitions. The `provides.*` API remains as a first-class user-facing interface, now powered internally by policy effects. The remaining work — forward elimination, aspect key type unification, pipeline-time trait reimplementation + fleet/den.exports for cross-host data, and policy scoping — builds on this foundation without requiring further architectural changes.

## Design Specs (Current Architecture)

These describe the pipeline as it exists today (753/753 tests on feat/fx-pipeline). Written from code analysis.

### Core Pipeline

| Component | Spec | Summary |
|-----------|------|---------|
| **Aspect Compilation** | [design/aspect-compilation.md](design/aspect-compilation.md) | `aspectToEffect` compiles aspects into fx computations. `compileStatic` classifies keys against the class registry and emits effects. `resolveChildren` orchestrates the five-step processing order. Four narrow effect handlers (resolve-aspect, resolve-parametric, resolve-conditional, check-dedup) each handle one include type with a single code path. |
| **Scope Partitioning** | [design/scope-partitioning.md](design/scope-partitioning.md) | `mkScopeId` produces injective scope identities from context key-value pairs. All emission state is partitioned by scope. `resolve-schema-entity` creates scopes via `scope.provide` + `state.modify` working in sync. Post-pipeline, `fxResolve` composes results per-scope: wrap, route, forward, flatten. |
| **Entity Resolution** | [design/entity-resolution.md](design/entity-resolution.md) | `den.schema` declares entity kinds with includes and policies. `resolveEntity` builds root aspect shapes with self-provide and framework aspects. The three-layer model: aspects (registry), policies (inclusion), schema includes (static). Seven entity kinds: host, user, home, flake, flake-system, conf, default. |

### Policy & Routing

| Component | Spec | Summary |
|-----------|------|---------|
| **Policy System** | [design/policy-system.md](design/policy-system.md) | Policies are plain functions from context to typed effects. Seven effect types (resolve, include, exclude, route, provide, instantiate, pipelineOnly). `installPolicies` dispatches via signature matching with enrichment iteration to fixpoint. Three registries: global, schema-scoped, aspect-included. |
| **Routes & Forwards** | [design/routes-and-forwards.md](design/routes-and-forwards.md) | `policy.route` reads from scope partitions and delivers class content to target classes via `wrapRouteModules`. `policy.provide` delivers new content directly. Complex forwards (adapter modules, dynamic paths) survive for advanced cases. Forward scope isolation propagates root specs to child scopes with filtered fallback. |
| **Constraint System** | [design/constraint-system.md](design/constraint-system.md) | `register-constraint` / `check-constraint` handler pair manages excludes, filters, and substitutions. `meta.handleWith` and `meta.excludes` install subtree-scoped constraints. Substitute type is vestigial (provides era). Planned evolution: constraints become policy effects via includes/excludes. |

### Class & Type System

| Component | Spec | Summary |
|-----------|------|---------|
| **Class Module Wrapping** | [design/class-module-wrapping.md](design/class-module-wrapping.md) | `wrapClassModule` enables flat-form class modules by detecting den args via `builtins.functionArgs` and pre-applying them. Three collision policy levels (aspect, entity, global). Post-pipeline `wrapCollectedClasses` wraps per-scope with scope-specific context. |

### Data Flow

| Component | Spec | Summary |
|-----------|------|---------|
| **Pipes and Quirks** | [design/pipes-and-quirks.md](design/pipes-and-quirks.md) | Named data channels (`den.quirks`) with producer/consumer pattern. Aspects emit pipe data as keys; class modules consume via function args. Policy-driven transform stages (filter, transform, fold, append, for). Cross-host collection via `pipe.collect`, child-to-parent via `pipe.expose`, targeted delivery via `pipe.to`. |
| **Effect Vocabulary** | [design/effect-vocabulary.md](design/effect-vocabulary.md) | Complete catalog of all named effects in the fx pipeline with handler locations and semantics. |

### Visualization

| Component | Spec | Summary |
|-----------|------|---------|
| **Diagram System** | [design/diagram-system.md](design/diagram-system.md) | `tracingHandler` captures pipeline events into structured traces. 31-file diag library constructs format-agnostic graph IR with nodes, edges, and entity kind metadata. 20+ renderers (Mermaid, C4, Graphviz, PlantUML, JSON). Views system with core, extended, fleet, and DAG perspectives. |

### Cleanup (Designed, Not Yet Implemented)

| Component | Spec | Summary |
|-----------|------|---------|
| **Provides Cleanup** | [design/provides-cleanup.md](design/provides-cleanup.md) | `provides`/`_` stays as permanent user API (virtual sub-aspect namespace). `provides-compat.nix` deleted; functionality folded into `emitAspectPolicies`. Self-provide and cross-entity routing handled in one pipeline phase. `resolveChildren` simplifies from 5 phases to 3. New users use policies + direct nesting; existing patterns unchanged. |

### Cancelled / Superseded

| Component | Spec | Summary |
|-----------|------|---------|
| **Traits (superseded)** | [cancelled/traits-reimplementation-design.md](cancelled/traits-reimplementation-design.md) | Pre-implementation design for trait system reimplementation. Superseded by [Pipes and Quirks](design/pipes-and-quirks.md) which implements the same data channel concept with a simpler architecture. |
| **Fleet + den.exports (absorbed)** | [cancelled/fleet-and-exports-design.md](cancelled/fleet-and-exports-design.md) | Cross-host data via lazy `fleet` attrset. Core use case absorbed by `pipe.collect` + config thunks. The `fleet` module arg may still land as an orthogonal convenience. |

### Proposed: Stream Architecture (Den 2)

Proposed ground-up rebuild of the resolution engine on [Ned](https://github.com/denful/ned)-style stream primitives (ST, ctxD, Cycle.js fixed-point). Replaces the handler-based trampoline with stream composition. ~1,400 lines targeting feature parity with the current ~7,300 line pipeline.

| Document | Summary |
|----------|---------|
| **[Stream Architecture Migration](design/stream-architecture-migration.md)** | Concern-by-concern analysis mapping each current subsystem (forwards, policies, pipes, DI, dedup) to stream equivalents. Establishes that Approach A (independent entity resolution + Cycle fixed-point) handles all cross-host features. |
| **[Den 2: Stream Rebuild](design/den2-stream-rebuild.md)** | Target architecture spec. 8 resolution engine layers, 7 API changes, Cycle structure (linear chain + targeted pipe fixed-point). Pipe.collect sub-design with scope metadata tagging and bidirectional entity-kind filtering. |
| **[Den 2: Incremental Delivery](design/den2-incremental-delivery.md)** | 10-phase parallel-engines delivery plan. Build resolve2 alongside current engine, port test suites incrementally, delete old engine when all 753 tests pass. Phases 4+5 parallelizable; Phase 6 (pipes) depends on Phase 5 (topology). |

## Reference Documents

| Document | Purpose |
|----------|---------|
| [IMPLEMENTATION-NARRATIVE.md](IMPLEMENTATION-NARRATIVE.md) | Phase-by-phase arc from midpoint to current state. What was designed, what shipped, where it diverged, what survived vs was reversed. |
| [Consolidated Spec](2026-05-02-fx-pipeline-consolidated.md) | Authoritative reference for current pipeline state, design invariants, remaining work, vestigial code audit. [Gist mirror](https://gist.github.com/sini/bdc1f509f87f0ae4869e84f6a310d04b). |
| [SUMMARY.md](SUMMARY.md) | Legacy audit of all 51 original specs with verdicts. Partially stale — design specs above are the current source of truth. |

## Archive

```
implemented-main/       Specs implemented on the main branch (pre-fx-pipeline)
implemented-branch/     Specs implemented on feat/fx-pipeline (historical)
tbd/                    Specs not yet implemented (some absorbed into design/)
cancelled/              Superseded or abandoned specs
```

Each directory contains spec files (`*.md`) and their corresponding implementation analyses (`*.analysis.md`). These are historical — the `design/` specs above reflect the current architecture.
