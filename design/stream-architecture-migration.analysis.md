# Analysis: Stream Architecture Migration

## Verdict

not-implemented

The spec is a forward-looking design document written 2026-05-07 (today). Zero stream primitives
(ST, ctxD, scopeD, run, hostsT, usersT, mkEntityDriver) exist in the implementation. The current
codebase is the exact "Current Model (Shared Pipeline)" the spec proposes to eliminate.

## Delivery Target

not delivered

No branch or commit contains stream-based implementation. Two companion specs created the same
day (`2026-05-07-den2-stream-rebuild.md`, `2026-05-07-den2-incremental-delivery.md`) describe
a ground-up rebuild strategy and phased delivery plan — those are also unimplemented.

## Evidence

### Stream primitives — absent

No files in `nix/` match `ST\b`, `ctxD`, `scopeD`, `ned\.run`, `hostsT`, `usersT`,
`mkEntityDriver`, `mainC`, `distinctByIdentity`, or `Cycle`. Grep across all of
`/home/sini/Documents/repos/den/nix/` returns zero hits for every proposed primitive.

### Old pipeline — fully present

All components the spec targets for elimination are live:

| File | Lines | Spec target |
|------|-------|-------------|
| `nix/lib/aspects/fx/assemble-pipes.nix` | 632 | Eliminate entirely |
| `nix/lib/aspects/fx/resolve.nix` | 557 | Replace 4-phase post-pipeline with stream terminals |
| `nix/lib/aspects/fx/class-module.nix` | 254 | `wrapClassModule`, `mkCollisionValidator`, `resolveCollisionPolicy` |
| `nix/lib/aspects/fx/route/apply.nix` | 245 | `applyRoutes`/`applySimpleRoute`/`applyComplexRoute` |
| `nix/lib/aspects/fx/handlers/push-scope.nix` | 88 | Eliminate `push-scope` |
| `nix/lib/aspects/fx/handlers/restore-scope.nix` | 27 | Eliminate `restore-scope` |
| `nix/lib/aspects/fx/handlers/compile-parametric.nix` | 72 | Replace with `ctxD` |
| `nix/lib/aspects/fx/handlers/bind.nix` | 96 | Eliminate bind/defer/drain machinery |
| `nix/lib/aspects/fx/handlers/drain.nix` | 39 | Eliminate |
| `nix/lib/aspects/fx/handlers/propagate-routes.nix` | 34 | Eliminate |
| `nix/lib/aspects/fx/handlers/constraint.nix` | 153 | Replace with stream filter/map |
| `nix/lib/aspects/fx/pipeline.nix` | 235 | 35+ handlers, 22 state fields |

`assemble-pipes.nix` contains `findMatchingSiblings` (line 123), `collectFromPeers` (153),
`collectAllExposed` (425), `buildTargetedData` (377) — all named in the spec's elimination list.

`resolve.nix` implements all four post-pipeline phases verbatim:
- Phase 1 `wrapPerScope` (line 52)
- Phase 2 `applyProvides` (line 109)
- Phase 3 `applyRoutes` (line 150)
- Phase 4 `applyInstantiates` (line 243)

`pipeline.nix` `defaultState` has 22 fields (line 131-170) including `inLateDispatch` (line 169),
`scopeContexts`, `scopeParent`, `firedPolicyNames` — all targeted for elimination.

`pipeline.nix` `defaultHandlers` composes 35+ handlers (lines 34-79).

`class-module.nix` has `mkCollisionValidator` (line 13), `resolveCollisionPolicy` (line 42),
`wrapClassModule` (line 240).

### Spec exists only as doc

The spec copy in the repo lives at
`docs/superpowers/specs/2026-05-07-stream-architecture-migration.md` (docs-only, not in `nix/`).

## Current Status

Still exists — the shared-pipeline model is the entire current implementation on
`feat/fx-pipeline`. Total handler infrastructure is ~5,700 lines across
`nix/lib/aspects/fx/` alone. `resolve2` entry point: absent.

## Supersession

This spec supersedes the prior incremental approach (see
`docs/superpowers/specs/2026-05-02-fx-pipeline-consolidated.md` and the long series of
`2026-04-*` plans that refactored handlers one by one). Those plans added scope partitioning,
unified policy effects, pipe assembly, and route unification to the existing trampoline.
This spec proposes abandoning that trampoline entirely.

Companion: `2026-05-07-den2-stream-rebuild.md` refines the proposal to a ground-up rebuild
(not migration); `2026-05-07-den2-incremental-delivery.md` adds a phased delivery strategy
with parallel engines.

## Gaps

Everything. No stream primitive landed. Specific proposals not yet started:

- `ST` / `st` / fluent stream API
- `ctxD` / `scopeD` / `run` (Cycle fixed-point)
- `hostsT` / `usersT` / `mkEntityDriver` topology drivers
- `dispatchD` / `converge` policy dispatch driver (~15 lines replacing ~100)
- `distinctByIdentity` include dedup
- Class sink routing (tagged elements + filter)
- Pipe combinators (`pipe.collect` → `.filter`, `pipe.expose` → upstream concat, etc.)
- Elimination of 4-phase post-pipeline assembly
- Elimination of scope push/pop/widen machinery

## Drift

No implementation drift — nothing shipped. The design is aspirational. The gap between
spec and code is the entire ~5,700-line pipeline infrastructure in `nix/lib/aspects/fx/`.
The spec's claimed new total of ~700-900 lines has not been validated against a running
implementation.
