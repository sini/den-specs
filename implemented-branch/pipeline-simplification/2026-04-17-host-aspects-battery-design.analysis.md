# Analysis: Battery: `host-aspects`

## Verdict

Fully implemented and shipped. Core spec delivered in PR #466 (main); three follow-up fixes/extensions landed in PRs #467, #468, #470. All spec tests pass and the test suite has been significantly expanded beyond what the spec required.

## Delivery Target

Merged to **main** (PRs #466–#468, #470). Also present on `feat/fx-pipeline` via rebase/merge (commits fed2e347, f002e70b, 815076ec, 483f5ec6, 8c1fcd9f).

## Evidence

- `modules/aspects/provides/host-aspects.nix` — battery implementation present
- `templates/ci/modules/features/host-aspects.nix` — test file present with 9 tests (spec required 4)
- Git log: `fed2e347 feat: host-aspects battery for projecting host homeManager to users (#466)`
- Subsequent fixes: `f002e70b` (#467), `815076ec` (#468), `483f5ec6` (#470), `8c1fcd9f` (cleanup)

## Current Status

Shipped. No open items. The battery is live on both main and feat/fx-pipeline.

## Supersession

Not superseded. The battery is a stable, self-contained feature.

## Gaps

None relative to spec. All four spec-required tests are present. The implementation diverges from the spec's pseudocode in two ways that are improvements, not gaps:

1. **Mechanism**: spec used `parametric.fixedTo` + `parametric.exactly`; implementation uses `constantHandler` scopeHandlers injected via `__scopeHandlers` on `host.aspect`. This is the correct approach after the parametric type refactor landed in the fx pipeline.

2. **Class scope**: spec hardcoded `class = "homeManager"` resolution; PR #470 generalised to `lib.genAttrs (user.classes or [ "homeManager" ])` so the battery works with any user class (e.g. `hjem`). This is a strict superset.

## Drift

Minor implementation drift from spec pseudocode — all intentional and sound:

| Spec | Implementation |
|------|----------------|
| `parametric.exactly { includes = [ from-host ]; }` | Plain attrset `{ inherit description; includes = [ from-host ]; }` — parametric wrapping removed after pipeline refactor |
| `parametric.fixedTo { host, user } host.aspect` | `constantHandler ctx` injected as `__scopeHandlers` — same semantics, new API |
| Resolves only `"homeManager"` class | Resolves all `user.classes` (defaults to `["homeManager"]`) |
| 4 tests specified | 9 tests implemented (extra: no-duplication, shared-sub-aspects, overlap-no-conflict, multi-user-distinct, hjem class) |

## Cross-Reference Notes

Cross-referenced 2026-04-28 against pipeline-simplification targets, design, and provides-removal analyses.

- **No inconsistencies found.** This battery is a self-contained feature with no dependency on provides removal or the pipeline simplification targets. Status (shipped, live on main and feat/fx-pipeline) is not contradicted by any other analysis.
- **Not a pipeline simplification target.** The six pipeline simplification targets (provides API, collision detection, sub-pipeline, classifyKeys, fan-out, child shapes) do not include host-aspects. The battery's presence on both branches is independent of that work.
