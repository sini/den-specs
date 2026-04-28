# Analysis: Aspect Transforms for Den

## Verdict
superseded

## Delivery Target
(A) main — the spec targeted `feat/aspect-rewrites`, which was merged into main.
The superseding architecture (`feat/fx-pipeline`, (B)) fully replaced the mechanism.

## Evidence

**Partial delivery of spec intent on main:**
- `737843b6` — `feat: tombstones, identity paths, and substitution in filterIncludes (#409)`
  Introduced `excludeAspect`, `substituteAspect`, `aspectPath`, and `tombstone` in `nix/lib/aspects/adapters.nix`.
  These are the spec's `transforms.exclude` / `transforms.substitute` functional equivalents,
  implemented as adapter-level functions rather than as a `transforms` namespace.
- `3828d77a` — `feat: tombstones record excludedBy for provenance tracking (#420)` — added `excludedFrom` field.
- `459bd1df` — `fix: rename excludedBy to excludedFrom on tombstones (#421)` — finalized tombstone schema.
- `d266c3a0` — `fix: perHost/perUser/perHome preserve aspect identity (#410)` — identity preservation shipped.
- `ca9f7e4f` — `feat: adapters.trace = (filterIncludes traceNames) (#407)` — trace functionality via adapters.

**Spec API never landed verbatim (`transforms.exclude / .substitute / .trace`, `resolve'`):**
- `nix/lib/aspects/transforms.nix` has no entry in git history — it was never created.
- `den.lib.aspects.transforms` namespace never existed in `nix/lib/aspects/default.nix`.
- `resolve'` never appeared in `nix/lib/aspects/default.nix`; `default.nix` exports `resolve` and `resolveWithState`.
- `collectExcludes`, `collectAspectTransforms`, `resolveWith` — no commits or code found.

**Full supersession via effectful pipeline (commit `68c26555`, `3a9cea99`):**
- `68c26555` — `refactor: den-fx (#462)` — replaced the entire legacy resolver with an effects-based pipeline.
  `nix/lib/aspects/resolve.nix` (the spec's "rewritten" file) was deleted in `3a9cea99`.
  `nix/lib/ctx-apply.nix` was deleted in `3a9cea99`.
  `nix/lib/aspects/adapters.nix` (tombstones/substitute) was deleted in `3a9cea99`.
- `3a9cea99` — `feat: effectful pipeline bootstrap` — confirmed deletion of `resolve.nix`, `ctx-apply.nix`, `adapters.nix`.
- `2e0fc9e1` — `refactor: remove legacy resolver, stabilize fx pipeline` — completed the transition.

**Current implementation on `feat/fx-pipeline`:**
- `meta.excludes` lives in `nix/lib/aspects/types.nix` (line 360) as aspect-level sugar.
- `meta.handleWith` is the general constraint mechanism replacing `transforms`.
- Exclude logic is in `nix/lib/aspects/fx/aspect.nix:registerConstraints` — reads `meta.excludes`, emits `register-constraint` effects.
- Substitute logic is in `nix/lib/aspects/fx/handlers/include.nix:substituteChild`.
- Exclude in the handler layer: `nix/lib/aspects/fx/handlers/include.nix:excludeChild`.
- Exclude from policy routing: `nix/lib/aspects/fx/handlers/transition.nix:registerExcludes`.
- Tracing: `nix/lib/aspects/fx/trace.nix` — `structuredTraceHandler`, `tracingHandler`.
- `perHost`/`perUser`/`perHome`: deprecated in `modules/context/perHost-perUser.nix`, now emit deprecation warning directing users to plain functions.

## Current Status

The spec's named API (`transforms.exclude`, `transforms.substitute`, `transforms.trace`, `resolve'`, `ctx-apply.nix`) **does not exist** in either main or `feat/fx-pipeline`. The underlying functionality (exclude by reference, substitute, trace entries, identity preservation) is present but restructured:

| Spec API | Current equivalent |
|---|---|
| `transforms.exclude ref` | `meta.excludes = [ ref ]` sugar on aspect; or `routing.excludes` on policy |
| `transforms.substitute ref asp` | `meta.handleWith = { type = "substitute"; ... }` constraint |
| `transforms.trace t` | `tracingHandler` / `structuredTraceHandler` in `fx/trace.nix` |
| `resolve' class opts asp` | `fx.pipeline.fxFullResolve` / `fxResolveTree` |
| `ctx-apply.nix:collectExcludes` | `fx/aspect.nix:registerConstraints` + `fx/handlers/transition.nix:registerExcludes` |
| `ctx-apply.nix:collectAspectTransforms` | Absorbed into effect-based constraint registration |
| `adapters.nix:excludeAspect` | `fx/handlers/include.nix:excludeChild` (internal) |
| `adapters.nix:substituteAspect` | `fx/handlers/include.nix:substituteChild` (internal) |

## Supersession

Superseded by the Phase E effectful pipeline architecture. The design spec for that transition is
`docs/design/fx-legacy-removal-spec.md` (added in `2e0fc9e1`). The fx-pipeline branch's
`2026-04-28-policy-pipeline-simplification.md` spec and related Phase E work are the active successors.

Companion spec `2026-04-04-chain-based-tracing.md` (the "future work" item in this spec) was also superseded —
chain-based tracing shipped via `chain-push`/`chain-pop` effects and `includesChain` state in the fx pipeline.

## Gaps

- `resolve'` as a public power API: never shipped verbatim. Closest is `resolveWithState` (internal) and `fxFullResolve`.
- `transforms` as a composable, user-facing constructor namespace: never existed. The fx pipeline internalised these as handler-level decisions rather than user-constructed transform functions.
- `den.hosts.<sys>.<name>.excludes` entity-level option: not found in entity types (`nix/lib/entities/`). Only `meta.excludes` on aspects and `routing.excludes` on policies are present. Spec described entity-level `excludes` as a first-class option; this was not carried forward.
- `replaces` sugar (spec's future work): not implemented.
- Forward `resolve'` with `trace = true`: the spec acknowledged this was disabled; in the fx pipeline, forward became post-processing (`1c077a06`) with no direct tracing API.

## Drift

1. **Architecture inversion**: spec assumed a recursive walker augmented with transform functions passed at call sites (`resolve'` opts). The fx pipeline inverted this — constraints are registered as effects during resolution, and handlers decide at each node. No user-visible `transforms` API exists.

2. **`perHost`/`perUser`/`perHome` type change**: spec stated these would return `{ includes = [...] }` plain attrsets (breaking change). The actual implementation uses `__fn`/`__args` wrapper inside `perCtx` and emits a deprecation warning. Semantically equivalent but structurally different from the spec.

3. **Entity excludes not wired**: spec described `den.hosts.<sys>.<name>.excludes` as a first-class option. Current entity types show no `excludes` option; entity-level exclusion flows through policy `routing.excludes` instead.

4. **`transforms` on aspect options removed**: spec proposed `options.transforms = listOf raw` on aspect submodule. Current `types.nix` has `meta.excludes` and `meta.handleWith` but no `transforms` option.

5. **Forward integration path changed**: spec proposed `forward.nix` reading `excludes`/`transforms` from asp and passing to `resolve'`. Instead, forward became a post-processor on pipeline results (`1c077a06`), and the constraint system propagates independently through handler state.
