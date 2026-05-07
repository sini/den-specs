# Analysis: Den 2 Incremental Delivery

## Verdict

not-implemented

The spec is a forward-looking delivery plan written 2026-05-07. No phase has started. `resolve2`
does not exist anywhere in the repo. The old handler-based engine (the thing this plan proposes to
replace incrementally) is fully intact with 37 handler files and 5,500+ lines targeted for eventual
deletion.

## Delivery Target

not delivered

No branch or commit contains any Phase 0 work. The plan has no PR open against it. The companion
specs (`den2-stream-rebuild.md`, `stream-architecture-migration.md`) are also unimplemented.

## Evidence

### resolve2 — absent

`nix/lib/resolve2.nix` does not exist. Grep for `resolve2` across all `*.nix` in the repo returns
zero hits. No `nix/lib/st/` directory. No stream primitives: `ctxD`, `hostsT`, `usersT`,
`mkEntityDriver`, `expandAspect`, `normalizeChild`, `distinctByIdentity`, `converge`, `dispatchD`,
`Cycle`, `traceD`, `Ned` — all absent from `nix/`.

### Old engine — fully intact

Every file the spec targets for Phase 9 deletion exists:

- `nix/lib/aspects/fx/handlers/` — 37 files, 2,082 lines
- `nix/lib/aspects/fx/pipeline.nix` — 235 lines (trampoline bootstrap)
- `nix/lib/aspects/fx/resolve.nix` — 557 lines
- `nix/lib/aspects/fx/assemble-pipes.nix` — 632 lines (the Phase 6 Cycle alternative)
- `nix/lib/aspects/fx/wrap-classes.nix` — 190 lines
- `nix/lib/aspects/fx/class-module.nix` — 254 lines (collision detection the spec calls "dead code")
- `nix/lib/aspects/fx/route/` — 3 files
- `nix/lib/aspects/fx/policy/` — 6 files
- `nix/lib/aspects/fx/aspect/` — 4 files

### Phase 0 test targets — exist (old engine)

All 21 behavioral test files listed for Phase 0 exist in
`templates/ci/modules/features/` (e.g. `apply.nix`, `hostname.nix`, `empty-aspects.nix`,
`host-options.nix`, `homes.nix`). They run against the old engine, not resolve2.

### Phase 1-2 test files — mostly exist

Phase 1 targets (`include-dedup.nix`, `nested-aspects.nix`, `aspect-path.nix`, `has-aspect.nix`,
`structural-detection.nix`, `angle-brackets.nix`) all exist at
`templates/ci/modules/features/`. Phase 2 targets exist too: `class-module-partial-apply.nix`,
`auto-parametric.nix`, `flake-scope-pipeline-args.nix`, `ctx-chain.nix`, `perUser-perHost.nix`,
`user-host-mutual-config.nix`, `parametric-fixedTo.nix`. Missing as separate files:
`ctx-pipeline.nix`, `nested-ctx.nix`, `custom-ctx.nix` — those tests likely live inside
`fx-coverage.nix` or similar.

### Phase 3-4 test files — exist

Policy test files (`policies.nix`, `policy-combinators.nix`, `policy-context-enrichment.nix`,
`policy-include-routing.nix`, `policy-provide.nix`, `route.nix`) all present. Forward test files
(`forward.nix`, `forward-to.nix`, `forward-flake-level.nix`, `guarded-forward.nix`,
`cross-context-forward.nix`, `debug-fwd.nix`, `dynamic-intopath.nix`) all present.

### Phase 5-6 test files — exist

Schema/namespace targets (`schema-registry.nix`, `namespace.nix`, `named-provider.nix`,
`cross-provider.nix`, `mutual-provider.nix`) all present. Pipe test files (`pipes.nix`,
`pipe-policy.nix`, `pipe-scope.nix`, `provide-to.nix`) all present.

### Phase 7 regression corpus — exists

`templates/ci/modules/features/deadbugs/` has 21 files (the spec says 21 deadbug files covering 52
tests). `depth.nix`, `resolve.nix`, `pure-eval.nix` also present.

### Phase 6 Cycle — implemented differently in old engine

`assemble-pipes.nix` (632 lines) implements pipe.collect via sibling scope traversal
(`scopeParent`/`scopeId` in `resolve.nix:242`, `resolve.nix:462`). This is the "Approach C
post-resolution coordination pass" the spec mentions as the Cycle fallback — it already exists in
the current engine, not as the Cycle fixed-point the spec proposes.

### API changes — partial pre-implementation

- `den.lib.parametric` — exists at `nix/lib/parametric.nix`, deprecated with warnings, not yet
  identity shim (`# DEPRECATED: scheduled for removal after first stable release post-fx-pipeline merge.`)
- `den.provides.forward` — still live (`nix/lib/home-env.nix:89`, `nix/lib/aspects/types.nix:69`)
- `den.lib.policy.route` — implemented (`modules/policies/flake.nix:27`,
  `modules/aspects/provides/os-class.nix:34`)
- `den.quirks` — used in engine (`nix/lib/aspects/fx/key-classification.nix:37`,
  `nix/lib/aspects/fx/assemble-pipes.nix:9`); spec Phase 8 calls for `den.pipes` alias
- `den.schema.*.includes` — used throughout modules (`modules/policies/core.nix:18`,
  `modules/policies/flake.nix:87`); the `.options` split is not implemented
- Policy effect constructors via context (`{ host, include, provide, ... }:`) — not implemented;
  current API still uses explicit `den.lib.policy.include/provide/exclude` calls
- Aspect-scoped policy auto-activation — not implemented

### Test count

Spec claims 753 total. Memory says 713. The spec acknowledges per-file counts are approximate and
says 753 is "confirmed by CI." Both are consistent with the current branch adding tests (40 pipe
tests from pipes work). The fx-* internal tests (214 per spec) are still present: 31 fx-* files
visible in `templates/ci/modules/features/`.

## Current Status

Still exists — the spec is a plan document written the same day as this analysis. The old engine
it plans to replace is the current production engine on `feat/fx-pipeline`. No execution has begun.

## Supersession

Supersedes: none directly. The spec supersedes an implicit "flag day rewrite" approach — it was
written specifically to describe the incremental alternative to the ground-up rebuild in
`den2-stream-rebuild.md`. The companion spec `stream-architecture-migration.md` describes the
architectural transition model this delivery plan operationalizes.

## Gaps

Every phase is unstarted:

- Phase 0: `resolve2` + Ned primitives + `nix/lib/st/` — zero lines written
- Phase 1: `expandAspect`, `normalizeChild`, `distinctByIdentity` — absent
- Phase 2: `ctxD` depth-bounded unfolding — absent (current engine does context via scope effects)
- Phase 3: `converge`, `dispatchD` — absent (current engine does fixed-point via `policy/iterate.nix`)
- Phase 4: stream concat for forwards — absent (current engine uses compile-forward handler)
- Phase 5: `mkEntityDriver` factory — absent
- Phase 6: Cycle fixed-point for pipes — absent (current engine uses post-resolution pass in `assemble-pipes.nix`)
- Phase 7-9: no regression porting, no API migration, no deletion

## Drift

The spec was written against the current codebase as target-to-replace, so there is no drift yet.
Two structural observations for when execution begins:

1. **pipe.collect is already solved differently.** The current engine's `assemble-pipes.nix` +
   `resolve.nix` sibling collection works. The spec's Cycle approach would replace it. Phase 6
   risk is lower than described — there is proven prior art in the existing code.

2. **Test count discrepancy.** Spec says 753; memory says 713. ~40 tests likely added by pipe
   work on `feat/fx-pipeline` after the memory file was written. The 753 figure is likely current.

3. **`ctx-pipeline.nix`, `nested-ctx.nix`, `custom-ctx.nix` missing as standalone files.** The
   Phase 2 table lists them, but they do not exist as separate files. Their content is presumably
   inside `fx-coverage.nix` or merged into other test files. Test file audit needed before Phase 2.
