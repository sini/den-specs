# Analysis: FX Pipeline v2 Refactor Design

## Verdict

**Implemented — collapsed, not sequential.** All six logical goals reached production but were delivered as one large commit (PR #462, `68c26555`, merged 2026-04-17) rather than the spec's prescribed step-by-step sequence. Naming diverged from spec: `resolveOne`/`go`-loop/`provide-class`/`register-adapter` never appeared by those names; the canonical implementation uses `aspectToEffect`/`resolveChildren`/`emit-class`/`register-constraint`.

## Delivery Target

`main` — merged via PR #462 (`refactor: den-fx`) on 2026-04-17. All subsequent work on `feat/fx-pipeline` builds on top of this foundation.

## Evidence

- `68c26555` (`refactor: den-fx (#462)`) — single commit delivering the full rewrite. PR body confirms: "Replace den's legacy recursive tree-walking resolution with an effects-based pipeline. Aspects compile into effectful computations via `aspectToEffect`."
- `nix/lib/aspects/fx/aspect.nix` — `aspectToEffect` is the compiler function (spec called it `resolveOne`). Returns a `Computation`; callers (`includeHandler`, `transitionHandler`) handle it via `fx.bind`.
- `nix/lib/aspects/fx/handlers/include.nix` — dense `fx.bind` chains replace the spec's `go`/`resolveChild`/`processApproved` loop. `resolveChildren` in `aspect.nix:748` uses `fx.bind` iteration.
- `emit-class` effect in `aspect.nix:437,444` and `handlers/tree.nix:151` — spec Step 3 landed as `emit-class` not `provide-class`; `classCollectorHandler` is the accumulator.
- `register-constraint`/`check-constraint` in `handlers/tree.nix` and `handlers/include.nix` — spec Step 4's `register-adapter`/`check-exclusion` landed under constraint naming.
- `get-path-set` effect in `identity.nix:68` — spec's `is-present` effect landed as `get-path-set` (path-set handler returns the accumulated set rather than answering a boolean query).
- `include-unseen`/`defer-include`/`drain-deferred` effects exist in handlers — spec's dedup and deferred-include mechanisms are present under different names.
- `flattenInto` in `handlers/transition.nix:41` — still present but scoped to transition flattening (not ctx-apply dedup as spec Step 5 described). `ctxSeenHandler` is the dedup authority.
- `nix/lib/aspects/default.nix` — `resolve = fxResolveTree` delegates directly to `fx.pipeline.fxResolve`; no legacy `resolve.nix:73-76` internal handle remains.
- `fxPipeline` option introduced in this PR, then removed on `feat/fx-pipeline` before merge — never reached `main`.

## Current Status

Fully operational on `main` and `feat/fx-pipeline`. All 488+ tests pass (as of PR merge). `feat/fx-pipeline` has 30+ follow-up commits (policy simplification, stage→entityKind rename, default routing, provides compat shims) layered on the v2 foundation.

## Supersession

This spec is superseded by:
- `docs/design/fx-pipeline-spec.md` (added in same PR #462) — the as-built canonical reference
- `docs/superpowers/specs/2026-04-28-provides-compat-shims-design.md` — documents what was added/removed relative to the `main`-era API

## Gaps

- **Steps 1–6 ordering not followed** — all steps collapsed into one PR. The spec prescribed testing each step against 459 tests; the PR delivered all at once at 488 tests.
- **`resolveOne` never existed** — compiler function is `aspectToEffect` throughout.
- **`provide-class` not used** — effect is `emit-class`; naming reflects "emit" pattern uniform across the protocol.
- **`register-adapter`/`check-exclusion` not used** — constraint system uses `register-constraint`/`check-constraint`; adapters became a compatibility layer (`adapters.nix`) not the primary exclusion mechanism.
- **`is-present` not used** — path-set query is `get-path-set` returning the full set, not a boolean probe per ref.
- **`buildStageIncludes` → computation chain (Step 6)** — `buildStageIncludes` never appeared; context transitions are handled by `transitionHandler` directly via `into-transition`/`drain-deferred` effects.
- **`ctx-apply.nix` not mentioned in final architecture** — `ctxApplyEffectful` was never created; `into-transition` handler owns that role.

## Drift

Minor — goals fully met, naming and structure diverged from spec but in coherent directions:
- "emit-X" pattern unifies all side-channel effects (`emit-class`, `emit-include`, `emit-forward`, `emit-trait`) vs spec's mixed naming
- Constraint system is more general than spec's exclude/substitute: `filterBy` added, scoped vs global variants, `ownerChain` provenance
- Step 4's lazy adapter registration dropped entirely — constraints register during resolution via `register-constraint`, no pre-scan or default-to-keep fallback needed because `check-constraint` returns `{ action = "keep"; }` for unknown identities by design
