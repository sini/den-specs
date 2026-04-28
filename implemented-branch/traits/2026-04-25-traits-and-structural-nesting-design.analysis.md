# Analysis: Traits, Structural Nesting, and Schema-Driven Classes

## Verdict

Largely implemented. All five migration phases (1–5) from the spec are shipped on `feat/fx-pipeline`. Core trait pipeline (collection, tier detection, consumption, cross-entity distribution, `_den.traits` injection, `partialOk` validation) is complete. Three follow-up items from the spec's "Follow-up Work" section (`hasTrait`, derived traits, trait attenuation) are not yet implemented — this is expected and matches the spec's own deferral intent.

## Delivery Target

Branch `feat/fx-pipeline`. No separate traits branch — landed incrementally via 12 commits between `83b28707` (schema registry) and `e790dc8e` (unified `aspectKeyType`).

## Evidence

**Schema registry (Phase 1):**
- `options.den.classes` and `options.den.traits` declared as top-level options in `modules/options.nix` (not under `den.schema` — matches spec decision §1 to avoid module eval cycle).
- `traitSchemaType` submodule with `collection` (`"list"` / `"map"`) and `partialOk` fields.
- Built-in `den.classes.nixos` auto-registered in `config.den.classes` at module import time.
- Collision check via `builtins.intersectAttrs` in `modules/options.nix`.

**Aspect content type (Phase 1):**
- `aspectContentType` in `nix/lib/aspects/types.nix` — generic wrapper with `__contentValues` and `__provider`.
- `aspectKeyType` in `nix/lib/aspects/types.nix` — per-key dispatch using `classReg`/`traitReg` lookups; three-branch dispatch confirmed at `e790dc8e`.
- `freeformType = lib.types.lazyAttrsOf (aspectKeyType typeCfg)` in `aspectSubmodule`.

**Structural detection (Phase 2):**
- `structuralKeysSet` in `nix/lib/aspects/fx/aspect.nix` includes `"traits"` and `"classes"` (lines 21–22), excluding them from freeform classification.
- 4-step classification in `compileStatic` (line 293): class → trait → nested aspect → unregistered (trace warn).
- Trace warning on unregistered keys: `builtins.trace "den: ignoring unregistered key '${k}'..."` (line 844).
- Nested aspect detection via `nestedKeys` accumulation and `emitNestedAspect` recursion (line 854).

**Trait collection — three-tier (Phase 3):**
- `nix/lib/aspects/fx/handlers/trait.nix` implements `traitCollectorHandler` and `traitArgHandler` in full.
- `detectTier` function: Tier 1 (plain value), Tier 2 (all args in `ctx`), Tier 3 (any module-sys arg or no named args).
- `collectTrait` applies `"list"` (concat) or `"map"` (merge with duplicate error) strategy.
- Deferred (Tier 3) stored in `state.deferredTraits`; collected (Tier 1/2) in `state.traits` — both thunk-wrapped.
- `traitArgHandler` maps each registered trait name to a handler reading live `state.traits`.

**`_den.traits` injection (Phase 3):**
- `pipeline.nix` lines 307–340: synthetic module `traitModule` injected into `evalModules` imports.
- `_den.traits` merges Tier 1/2 pipeline data + Tier 3 deferred resolved functions + cross-entity data.
- `partialOk` post-pipeline validation (lines 286–293, 345–346): errors when pipeline-time consumer saw trait AND Tier 3 deferred emissions exist, unless `partialOk = true`.

**`wrapClassModule` trait arg pre-application (Phase 3):**
- `nix/lib/aspects/types.nix` lines 166–262: `traitArgNames` extracted after den context args.
- Trait thunks: `moduleArgs: moduleArgs.config._den.traits.${name} or []` — lazy, resolved at `evalModules` time.
- Den context args take priority (checked first); trait args only for names not already in `ctx`.

**Cross-entity trait distribution (Phase 4):**
- `nix/lib/aspects/fx/distribute-cross-entity.nix` implements `distributeCrossEntityTraits` — groups by target name, merges with collection strategy.
- Transition handler (`handlers/transition.nix` lines 214–224): runs sub-pipeline per sibling peer, captures `sub.traits`, emits `"provide-to"` effect with `{ targetEntity, traits }`.
- Cross-entity trait data injected at `evalModules` time (not pipeline re-run) — matches spec design decision §4 constraint.
- `provide-to` handler (`handlers/provide-to.nix`) accumulates emissions in `state.provideTo`.

**Forward battery schema integration (Phase 5):**
- `nix/lib/forward.nix` line 14: reads `(den.classes.${fromClass} or {}).forwardTo or null`.
- `den.classes.homeManager.forwardTo` with `class` + `path` declared in built-in batteries (`os-user.nix`, `home-manager.nix`, etc.).
- CI tests in `templates/ci/modules/features/forward-to.nix` cover `forwardTo` default routing and path.

**Aspect-level trait/class installation:**
- `modules/aspect-schema.nix` collects `aspect.traits` / `aspect.classes` from all `den.aspects` and `den.ful.*` namespaces, merges into `config.den.traits` / `config.den.classes`.
- Structural key `"traits"` and `"classes"` excluded from namespace freeform scanning.

**Namespace integration:**
- `nix/lib/namespace-types.nix` has `options.traits` and `options.classes` on `namespaceType`.
- `nix/lib/namespace.nix` lines 37–41: merges `denful.traits` and `denful.classes` into `config.den.traits` / `config.den.classes` on import.

**`provides` status:**
- Still a structural key (line 17 of `structuralKeysSet`), not yet aliased or deprecated with a warning.
- `den-brackets.nix` line 23 warns on `provides.${head}` bracket paths: "migrate to direct nesting at key '${head}'".
- `provide-to` as an aspect structural key (emit from aspect body) appears removed — `aspect.nix` has no reference to `"provide-to"` in `structuralKeysSet`; cross-entity routing now goes entirely through sibling sub-pipeline trait capture. However `provide-to.nix` handler comment still says "Aspects declare provide-to.${label} = data" — this is stale documentation.

## Current Status

Shipped in five phases across 12 commits. All major spec sections are implemented:
- Schema registry (`den.classes`, `den.traits`) with collision check
- `aspectContentType` / `aspectKeyType` freeform type
- 4-step structural detection with trace warnings
- Three-tier trait evaluation with `detectTier`
- `traitCollectorHandler` + `traitArgHandler`
- `_den.traits` injection module with deferred resolution
- `partialOk` tier visibility mismatch validation
- `wrapClassModule` lazy trait thunk pre-application
- Cross-entity trait distribution via `distributeCrossEntityTraits`
- `den.classes.*.forwardTo` consumed by forward battery
- Aspect-level and namespace-level trait/class installation

## Supersession

Supersedes the 2026-04-16 Capabilities spec (draft) as stated in the spec header. The `provide-to` labeled-data pattern (aspects declaring `provide-to.http-backends = [...]`) is superseded by direct trait emission — the `provide-to` structural key on aspects is gone, replaced by trait-based cross-entity routing via sibling sub-pipeline capture.

## Gaps

1. **`provides` deprecation warning not wired.** The spec (§5) says `provides` gets a deprecation warning pointing to direct nesting. The `structuralKeysSet` still lists `"provides"` silently. The bracket path warns (`den-brackets.nix`), but the aspect-level `provides` key itself does not.

2. **`provide-to.nix` handler comment is stale.** Still documents the old "aspects declare `provide-to.${label}`" pattern. The actual mechanism is now sibling route sub-pipeline capture. Low severity — implementation is correct, comment is wrong.

3. **Follow-up items not implemented (expected):**
   - `hasTrait` predicate — spec deferred to follow-up
   - Derived/computed traits (`derivedFrom`) — spec deferred
   - Trait attenuation (`traitFilter`) — spec deferred
   - Diagram visualization of `emit-trait` effects — spec deferred

4. **`collection = "map"` cross-entity dedup.** Spec §Design Decision 2 notes that duplicate detection for `"map"` traits from Tier 3 deferred functions happens inside the generated NixOS module (cannot inspect return values at pipeline time). This constraint is correctly implemented but not surfaced with a clear error message for the Tier 3 cross-entity case.

## Drift

1. **`distribute-cross-entity.nix` retains `constantHandler` shim.** The `distribute` export (line 91–98) still builds `constantHandler` bindings from distributed trait data for backward compat. The spec's Phase 4 design intended `state.traits` merging as the sole mechanism. The `constantHandler` path appears to be a compat layer, not the primary flow — primary flow goes through `crossEntityTraits` arg to `pipeline` function and into `_den.traits` injection. No functional regression, but the compat path should eventually be cleaned up.

2. **`provide-to` effect payload.** The spec §Provide-To Unification specifies changing the `"provide-to"` effect payload from "raw labeled data to trait state." This is implemented: `transition.nix` now emits `{ targetEntity, traits }` (trait state). The `provide-to.nix` handler accumulates these trait-carrying payloads. Consistent with spec intent.

3. **`aspectKeyType` three-branch dispatch.** The spec describes `aspectContentType` as a uniform wrapper applied identically to all keys, with classification deferred to `compileStatic`. The implementation uses `aspectKeyType` which does per-key registry lookups at type-merge time (commit `e790dc8e`). This is a minor structural deviation — classification happens slightly earlier (at type merge rather than purely at `compileStatic`) but the semantic result is the same. `compileStatic` still does the authoritative 4-step classification.
