# Analysis: Dead Letter Queue for Unregistered Aspect Keys

## Verdict
implemented
All major spec proposals shipped: dead-letter effect, deadLetterHandler, drainDeadLettersHandler, DLQ state field, drain-dead-letters call in resolveContextValue, post-pipeline trace warnings, and template den.classes registrations. One notable deviation: the drain uses dynamic `state.traitSchemas` instead of the static `traitRegistry` the spec described.

## Delivery Target
feat/fx-pipeline

## Evidence

Commits (in order, all on feat/fx-pipeline):
- `bd35b47b` — add deadLetterHandler to tree.nix
- `8338e70f` — add DLQ emit helpers and drain handler in aspect.nix
- `5192f54b` — wire deadLetterQueue state and handlers into pipeline
- `50c3974d` — replace unregistered key trace warning with dead-letter effects
- `b662f9cd` — drain dead letter queue after drain-deferred in resolveContextValue
- `6887c89f` — register den.classes in template flake-parts-modules
- `5ab9ba79` — emit trace warnings for unclaimed dead letter queue entries

### dead-letter effect emission
`nix/lib/aspects/fx/aspect.nix` L947-967: `deadLetterEffects` built via `map (k: fx.send "dead-letter" { key; rawValue; aspectIdentity; aspectName; ctx; aspectPolicy; globalPolicy; }) unregisteredClassKeys`. Appended to effect list in `compileStatic`.

### deadLetterHandler
`nix/lib/aspects/fx/handlers/tree.nix` L436-454: handler for `"dead-letter"` accumulates to `state.deadLetterQueue` and also writes to `state.scopedDeadLetterQueue` (dual-write for scope-partitioned state — not in original spec, added later).

### DLQ state field
`nix/lib/aspects/fx/pipeline.nix` L172: `deadLetterQueue = _: [];`
Also L182: `scopedDeadLetterQueue = _: {};` (scope-partitioned counterpart).

### drainDeadLettersHandler
`nix/lib/aspects/fx/aspect.nix` L465-498. Uses `state.traitSchemas null` (dynamic, not static `traitRegistry`) for trait matching. Exported to `pipeline.nix` L10 via `inherit ... drainDeadLettersHandler` and wired at L88 via `// drainDeadLettersHandler`.

### emitClassFromDLQ / emitTraitFromDLQ
`nix/lib/aspects/fx/aspect.nix` L404-463. Both helpers present. `emitClassFromDLQ` takes `dynamicTraitSchemas` as first arg (divergence from spec — spec showed single-arg; runtime signature now two-arg to thread trait schemas).

### drain-dead-letters call sites
`nix/lib/aspects/fx/handlers/transition.nix` L154: after `drain-deferred` in `resolveContextValue`.
L730: second call site in enrichment path (`resolveContextValue` enrichment branch), matching spec requirement of same call points as drain-deferred.

### post-pipeline diagnostics
`nix/lib/aspects/fx/pipeline.nix` L583-586: `finalDLQ` read from `state.deadLetterQueue null`; trace warning emitted per unclaimed entry. Accessible via `fxFullResolve` as spec described.

### template den.classes registrations
- `templates/flake-parts-modules/modules/classes/devshell.nix` L7: `den.classes.devshell = {};`
- `templates/flake-parts-modules/modules/classes/treefmt.nix` L7: `den.classes.treefmt = {};`
- `templates/flake-parts-modules/modules/classes/packages.nix` L12: `den.classes.packages = {};`
- `templates/flake-parts-modules/modules/classes/files.nix` L7: `den.classes.files = {};`
- `templates/flake-parts-modules/modules/classes/nix-unit.nix` L12: `den.classes.tests = {};`
All five registrations from spec present.

## Current Status
Still exists. All DLQ infrastructure active on feat/fx-pipeline. No supersession.

## Supersession
none

## Gaps
None observed. All spec-listed impact points shipped:
- aspect.nix: dead-letter effect emission replacing trace warning ✓
- tree.nix: deadLetterHandler ✓
- pipeline.nix: deadLetterQueue state + handler wiring ✓
- transition.nix: drain-dead-letters after drain-deferred ✓
- templates: den.classes registrations ✓

## Drift

1. **Static vs dynamic trait registry**: Spec described drain re-checking against `traitRegistry` (static, computed at module fixed-point as `den.traits`). Implementation uses `state.traitSchemas null` — the dynamic trait schema state accumulated during pipeline execution. This is strictly more powerful: traits registered dynamically via `"register-trait-schema"` effects are also visible to the drain.

2. **scopedDeadLetterQueue dual-write**: `deadLetterHandler` in tree.nix also writes `state.scopedDeadLetterQueue` (scope-partitioned counterpart, L443-452). Not in spec. Added as part of scope-partitioned pipeline state work (`671ebf0c`, `892dbf12`).

3. **emitClassFromDLQ signature**: Spec showed `emitClassFromDLQ = entry: ...`. Implementation is `emitClassFromDLQ = dynamicTraitSchemas: entry: ...` (curried, threads trait schemas for wrapClassModule's traitNames arg).

4. **Early-exit optimization**: `drainDeadLettersHandler` short-circuits when `queue == []` (L472-476), returning state unchanged. Not in spec pseudocode but correct behavior.

5. **Drain re-check uses traitSchemas not traitRegistry for trait matching**: `dynamicTraitSchemas ? ${entry.key}` at L480 reads from live state, not the module-system `den.traits` attrset. Spec's risk note about "registry timing" (static fixed-point complete from pipeline start) is rendered moot — the implementation matches against whatever trait schemas are in state at drain time, which may be a superset of the static registry.
