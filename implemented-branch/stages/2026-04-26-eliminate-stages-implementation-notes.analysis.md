# Analysis: Eliminate Stages — Implementation Notes

## Verdict

Fully implemented and then superseded. The spec describes an intermediate state (`den.entityIncludes`) that shipped on feat/traits and was subsequently evolved on feat/fx-pipeline to use `den.schema.<kind>.includes` directly. All described infrastructure is either confirmed in place or confirmed deleted/replaced with a better equivalent. No gaps remain; several outstanding items from the spec were resolved post-authorship.

## Delivery Target

Spec authored 2026-04-26 against feat/traits. Implementation confirmed shipped. feat/fx-pipeline carries forward the completed work and extends it (entityIncludes and entityProvides themselves deleted; rootIncludes phase removed; schema-based entity guard replaces entityIncludes presence check).

## Evidence

### den.stages deleted
- `removed-stages.nix` (`modules/removed-stages.nix`) defines `den.stages` as a `lib.types.raw` option that immediately throws on any write with migration guidance. No `resolve-stage.nix` or `stage-types.nix` present in repo.
- Commit `e8a26848`: "delete den.stages, resolve-stage.nix, stage-types.nix".

### resolveEntity + resolve-entity effect
- `nix/lib/resolve-entity.nix` exists. Reads from `den.schema.<kind>.includes` and `den.schema.<kind>.isEntity`. Self-provides, host framework aspects, and schema includes merged into single `includes` list.
- `nix/lib/aspects/fx/pipeline.nix` lines 85–106: `resolveEntityHandler` handles `resolve-entity` effect via `den.lib.resolveEntity`.
- Commit `7ceea535`: "switch resolve-target effect to resolve-entity".

### den.entityIncludes — intermediate, now deleted
- `den.entityIncludes` was the initial implementation deviation (global per-kind list). It was later replaced with `den.schema.<kind>.includes` as a first-class option on schema entries.
- Commit `03c14319`: "den.schema.X.includes — readable includes on deferred schema entries".
- Commit `9a338b50`: "delete entityIncludes/entityProvides, schema-based entity guard".
- No `entityIncludes` or `entityProvides` references remain in `nix/` or `modules/` (confirmed by search; only `removed-stages.nix` shows the removal shim for stages itself).

### rootIncludes phase
- Spec documents rootIncludes as a new pipeline phase in `resolveChildren`. This shipped, then was removed.
- Commit `973ac30b`: "remove rootIncludes phase, merge entity includes before transitions". Entity self-provide, framework, and schema includes now go directly into the `includes` field; `resolveChildren` processes includes before transitions — same ordering contract, no extra phase needed.
- No `rootIncludes` references remain in main branch source.

### Policy aspects field
- `nix/lib/aspects/fx/handlers/transition.nix` line 254: `aspects = routing.aspects or [ ]`; line 302: `policyAspects = (transition.routing or { }).aspects or [ ]`; lines 347–348, 460, 537, 567: inject/track policy-declared aspects during transition resolution.
- `nix/lib/policy-effects.nix`: `resolve`, `include`, `exclude` effect constructors confirmed; `resolve.to` present.

### makeHomeEnv two-hop collapse
- `nix/lib/home-env.nix`: `makeHomeEnv` constructs a single `battery` attrset with a `policyFn` that emits `resolve.user` + includes (`hostModule`, `userForward`, optional `os-user-fwd`/`os-user-class-fwd`). No intermediate `hm-host` entity. No `userEnvAspect`.
- `modules/aspects/provides/home-manager.nix`, `maid.nix`, `hjem.nix`: all use `makeHomeEnv` → `den.schema.host.includes = [ result.battery ]`.
- Search for `userEnvAspect` returns only worktree stale copies (docs-fixes, stash-cleanup).

### Flake forwarders registered as aspects
- `modules/outputs/flakeSystemOutputs.nix`: `systemOutputFwd` takes flat `{ system, output, class, aspect-chain }` signature (not curried); registered as `den.aspects.flake-${output}` for all five system output kinds.
- `modules/outputs/osConfigurations.nix`: `osFwd` registered as `den.aspects.flake-os`.
- `modules/outputs/hmConfigurations.nix`: `hmFwd` registered as `den.aspects.flake-hm`.
- Flake policies (`modules/policies/flake.nix`) inject these via `include den.aspects."flake-${output}"`.
- Flake kinds registered in `den.schema` via `modules/context/flake-schema.nix` with `isEntity = false`.

## Current Status

All core spec deliverables confirmed present or replaced with a superior equivalent:

| Spec item | Status |
|---|---|
| `den.stages` deleted | Confirmed — removal shim in place |
| `den.entityIncludes` | Superseded by `den.schema.<kind>.includes` |
| `rootIncludes` pipeline phase | Added then removed; ordering preserved without it |
| `resolveEntity` sole resolution path | Confirmed |
| `resolve-entity` effect | Confirmed |
| Policy `aspects` field | Confirmed in transition handler |
| makeHomeEnv two-hop collapsed | Confirmed |
| Flake forwarders as registered aspects | Confirmed |
| `entityProvides` cleaned up | Confirmed deleted |
| `__ctxStage` structural key | Confirmed (prior code review item) |
| Template migrations | Confirmed (7 templates via commit `3346471f`) |

Outstanding items from spec section, current state:
- **ctx-shim.nix**: Still present at `modules/compat/ctx-shim.nix`. Now forwards `den.ctx.*` to `den.schema` includes (updated in commit `9a338b50`). `den.ctx.*.into` path removed.
- **emitCrossProvider in transition.nix**: Still present (lines 101, 288). Spec called it dead code. The mutual-provider compat shim (`2e40b024`) added a `provides-compat` handler path, so this may now be active for compat shims rather than dead.
- **doc-examples.nix**: Two commented-out lines reference `den.stages` in `templates/ci/modules/features/fx-coverage.nix` (line 264–265) and `doc-examples.nix` (line 178) — these are comment strings, not evaluated.

## Supersession

The spec's deviation #1 (`den.entityIncludes` instead of `den.schema.<kind>.includes`) was itself superseded. The subsequent implementation went further and landed the spec's original intent: schema-entry includes as first-class, readable option. The spec's deviations section is therefore partially stale.

Deviation #2 (`den.entityProvides` retained transitional) resolved — deleted in `9a338b50`.

Deviation #3 (no sibling routes for flake) still accurate — flat function approach confirmed in `flakeSystemOutputs.nix`.

Deviation #4 (`rootIncludes` not in spec) resolved — rootIncludes added and then removed; spec's original assumption ultimately vindicated.

## Gaps

None functional. The implementation is complete and has advanced beyond the spec's described state.

## Drift

The spec describes `den.entityIncludes` as the settled resolution source for `resolveEntity`. This is stale — `entityIncludes` was an intermediate and is now deleted. The actual settled interface is `den.schema.<kind>.includes`. Any reader following the spec should substitute `den.entityIncludes` reads with `den.schema.<kind>.includes` throughout.

The spec's "rootIncludes" section describes a phase that no longer exists. Entity includes now merge directly into the includes field before transitions in `resolveEntity` output — same effect without the extra phase.

The spec also predates the provides-compat handler (`19738452`, `2e40b024`), which resurrected the `emitCrossProvider` path for backwards-compatible mutual-provider shims.
