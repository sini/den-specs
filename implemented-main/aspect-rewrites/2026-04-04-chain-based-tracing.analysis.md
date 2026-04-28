# Analysis: Chain-Based Tracing for Den Aspect Resolution

## Verdict
superseded

## Delivery Target
(A) main — the spec's direct intent (inline trace in `resolveWith`, remove trace functor) landed as `adapters.structuredTrace` in the legacy adapter pipeline. The idea fully superseded by (B) feat/fx-pipeline where trace is a first-class handler (`tracingHandler`, `structuredTraceHandler`) in the fx pipeline.

## Evidence

**Spec's immediate successors on main (before den-fx):**
- `f9ef7e5d` — `adapters.traceName` — basic name-tree adapter, precursor to structured trace
- `ca9f7e4f` — `adapters.trace = filterIncludes traceName` — convenience alias
- `eb74bdaf` — `adapterOwner` tracking in `filterIncludes` + `collectSelfPath` helper; sets up structured trace provenance

**Spec's direct implementation in den-fx (68c26555):**
- Added `adapters.structuredTrace` to `nix/lib/aspects/adapters.nix` — flat entry list with `name`, `class`, `parent`, `provider`, exclusion info, parametric flags, entity-kind tags. This is the spec's "trace entries with `{ name, class, decision, depth, chain }`" translated into the adapter protocol.
- Added `nix/lib/aspects/fx/trace.nix` — `structuredTraceHandler` and `tracingHandler` as effect handlers observing `resolve-complete` events. Zero additional call depth; trace accumulation entirely in handler state. This is the spec's core: build trace entries inline, not functor-per-node.
- `chain-push`/`chain-pop` effects replace `__parent` string tracking — corresponds to spec's "chain" field on trace entries.

**Current state on feat/fx-pipeline:**
- `nix/lib/aspects/fx/trace.nix` — `structuredTraceHandler` + `tracingHandler` both present
- `adapters.nix` deleted — legacy adapter pipeline is gone; all tracing now via handlers
- `72b34735` — removed `policyTraceHandlers` from `trace.nix` as policy-effect intercept mechanism became dead code after dispatch collapse

## Current Status

The spec's goal is fully realized and then extended:
- `nix/lib/aspects/fx/trace.nix` exists on both main and feat/fx-pipeline
- `structuredTraceHandler` — minimal, for tests; accumulates entries without disambiguation
- `tracingHandler` — full production handler; disambiguates anonymous entries using entity kind tags, matching legacy `structuredTrace` adapter naming
- `adapters.nix` (legacy) deleted on feat/fx-pipeline; `trace`/`traceName`/`structuredTrace` adapters gone with it

## Supersession

This spec was a design note for what became:
1. `adapters.structuredTrace` (interim, in legacy pipeline, commit `68c26555`)
2. `fx/trace.nix` `tracingHandler` / `structuredTraceHandler` (final, fx pipeline, commit `68c26555`)

The spec itself superseded the earlier functor-wrapping approach (`adapters.trace` / `adapters.traceName` from `ca9f7e4f`/`f9ef7e5d`), which built trace as nested lists via adapter recursion rather than inline handler state.

## Gaps

- **`opts.trace` on `resolve'`** — the spec mentions `opts.trace` on `resolve'` still controlling trace collection. The fx pipeline does not have a `resolve'` call-site flag; tracing is enabled by passing extra handlers (`tracingHandler`) to `mkPipeline`. The opt-in mechanism changed from a flag to handler composition.
- **`__forwardTrace` mechanism** — the spec says "forward traces still use `__forwardTrace` mechanism." No `__forwardTrace` exists anywhere in the current codebase. Forward resolution was later restructured as post-processing on pipeline results (`1c077a06`); forward trace is not separately tracked.
- **`transforms.nix` removal** — spec says `trace` function removed from `transforms.nix` and public API. No `transforms.nix` file ever appeared in the repo; the spec was describing an intended refactor of the adapter module that shipped differently (as `adapters.nix` restructuring, then its deletion).
- **`normalizeResult` and `compose` simplification** — spec says these were simplified. Neither function name appears in the codebase; the resolver was rewritten wholesale in den-fx, so the simplification happened via replacement rather than targeted edits.

## Drift

- **Spec describes surgical edits; implementation was wholesale rewrite.** The spec assumes `resolve.nix`, `transforms.nix`, `default.nix` as stable files receiving targeted changes. Instead the entire resolution engine was replaced by `nix/lib/aspects/fx/` in `68c26555`. The spec's problem statement (functor wrapping adds call depth causing stack overflow on deep chains) was solved, but via effects-based pipeline rather than inline edits to `resolveWith`.
- **`depth` field dropped.** Spec specifies `{ name, class, decision, depth, chain }` per entry. Current `tracingHandler` entries have `name`, `class`, `parent`, `entityKind`, `provider`, `excluded`, `excludedFrom`, `replacedBy`, `isProvider`, `handlers`, `hasClass`, `isParametric`, `fnArgNames`. No `depth` or `decision` field.
- **`decision` field dropped.** The spec implies a per-node resolution decision enum. Not present; exclusion state is captured via `excluded`/`excludedFrom`/`replacedBy` fields instead.
- **Chain as parent pointer, not full path.** Spec's `chain` field implied the full includes-path per entry. Implementation uses `chain-push`/`chain-pop` to maintain `includesChain` in handler state, and each entry gets a single `parent` string (nearest meaningful ancestor) rather than the full chain list.
