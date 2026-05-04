# den-specs

Design specs for [den](https://github.com/denful/den), a NixOS configuration framework built on algebraic effects via [nix-effects](https://github.com/sini/nix-effects).

## The Story

Den's `feat/fx-pipeline` branch replaced the legacy aspect resolution pipeline with an algebraic effects architecture. The initial implementation introduced scope-partitioned state, typed policy effects, and entity-schema-driven resolution — but accumulated tech debt as each new mechanism (transitions, DLQ, three-tier traits, forward sub-pipelines) interacted with every other. A cleanup arc deleted ~3000 lines by recognizing that `scope.provide` — the effect system's own scoping primitive — could replace most of the manually-built machinery. Transitions became `installPolicies` with lexical `scope.provide`, the DLQ was eliminated in favor of direct class emission, and the entire trait system was removed for a simpler reimplementation.

The result is a pipeline where policy dispatch, context expansion, and entity resolution are composed from small effect handlers rather than orchestrated by monolithic dispatchers. Scope partitioning gives each entity its own state partition without sub-pipeline isolation. Routes and provides deliver content post-pipeline from those partitions. The remaining work — provides removal, trait reimplementation via a single pipeline tier + fleet/den.exports, and policy scoping — builds on this foundation without requiring further architectural changes.

## Design Specs (Current Architecture)

These describe the pipeline as it exists today (629/629 tests, PR [#475](https://github.com/denful/den/pull/475)). Written from code analysis.

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
| **Routes & Forwards** | [design/routes-and-forwards.md](design/routes-and-forwards.md) | `policy.route` reads from scope partitions and delivers class content to target classes via `wrapRouteModules`. `policy.provide` delivers new content directly. Tier 2 forwards (adapter modules, dynamic paths) survive for advanced cases. Forward scope isolation propagates root specs to child scopes with filtered fallback. |
| **Constraint System** | [design/constraint-system.md](design/constraint-system.md) | `register-constraint` / `check-constraint` handler pair manages excludes, filters, and substitutions. `meta.handleWith` and `meta.excludes` install subtree-scoped constraints. Substitute type is vestigial (provides era). Planned evolution: constraints become policy effects via includes/excludes. |

### Class & Type System

| Component | Spec | Summary |
|-----------|------|---------|
| **Class Module Wrapping** | [design/class-module-wrapping.md](design/class-module-wrapping.md) | `wrapClassModule` enables flat-form class modules by detecting den args via `builtins.functionArgs` and pre-applying them. Three collision policy levels (aspect, entity, global). Post-pipeline `wrapCollectedClasses` wraps per-scope with scope-specific context. |

### Visualization

| Component | Spec | Summary |
|-----------|------|---------|
| **Diagram System** | [design/diagram-system.md](design/diagram-system.md) | `tracingHandler` captures pipeline events into structured traces. 31-file diag library constructs format-agnostic graph IR with nodes, edges, and entity kind metadata. 20+ renderers (Mermaid, C4, Graphviz, PlantUML, JSON). Views system with core, extended, fleet, and DAG perspectives. |

### Planned (Not Yet Implemented)

| Component | Spec | Summary |
|-----------|------|---------|
| **Traits** | [design/traits.md](design/traits.md) | Semantic data channels collected across aspects. Simplified to one pipeline tier (static + parametric collapse under `scope.provide`). Schema registry (`den.traits`) with collection strategies. `{ traitName, ... }:` consumption in discriminators and class modules. ~250 lines estimated, 10x simpler than the deleted implementation. |
| **Fleet + den.exports** | [design/fleet-and-exports.md](design/fleet-and-exports.md) | Cross-host data via lazy `fleet` attrset of evaluated NixOS configs + `den.exports` freeform option. Replaces the original provide-to mechanism and Tier 3 traits. Uses Nix laziness directly — no pipeline involvement, no custom distribution phase. |

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
