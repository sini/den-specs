# Analysis: Provider-Aware Excludes and Trace

## Verdict
implemented

## Delivery Target
(A) main

## Evidence

- Commit `d267c458` ("fix: excludeAspect cascades to provider sub-aspects (#411)") on main:
  - Modified `nix/lib/aspects/adapters.nix` — `excludeAspect` changed from exact path equality to prefix matching via `lib.take (builtins.length refPath) ap != refPath`.
  - Added `templates/ci/modules/features/aspect-path.nix` — two new tests:
    `test-excludeAspect-by-provider` (exclude single provider) and
    `test-excludeAspect-cascades-to-providers` (parent exclude cascades to all providers).

- `nix/lib/aspects/types.nix` (main + feat/fx-pipeline): `__provider` stored at `providerPrefix` level; provider path composed as `(typeCfg.providerPrefix or []) ++ [keyName]` and propagated via `meta.provider` on aspects.

- `nix/lib/aspects/fx/aspect.nix` (feat/fx-pipeline): `meta.provider` set at lines 717 and 804 using same `(aspect.meta.provider or []) ++ [name]` accumulation; `chainIdentity` at line 823 uses `identity.pathKey` on the composed path.

- `nix/lib/aspects/fx/handlers/tree.nix` (feat/fx-pipeline): `check-constraint` handler (lines 83–94) performs slash-joined prefix decomposition — splits `nodeIdentity` by `/`, generates all ancestor prefixes, looks each up in `constraintRegistry`. An exclude constraint registered on `monitoring` is found when checking `monitoring/node-exporter` because `monitoring` is a prefix entry. This is the fx-pipeline equivalent of the legacy `isPrefix` list-prefix check.

- `nix/lib/aspects/fx/handlers/transition.nix`: `registerExcludes` converts exclude aspect refs to `register-constraint` effects with `type = "exclude"` and `scope = "subtree"`, keyed by `pathKey (aspectPath ref)`. The constraint key is the slash-joined identity, which the prefix check in `tree.nix` matches against.

- The spec's `wrapProvider`/`carryAttrs`/`withIdentity` propagation mechanism is not literally present; provider origin is tracked via `meta.provider` on the aspect attrset rather than a separate `wrapProvider` wrapper function.

## Current Status

- **main**: `excludeAspect` with prefix cascade exists in `nix/lib/aspects/adapters.nix` (legacy pipeline path). `__provider`/`providerPrefix`/`meta.provider` tracking active in `nix/lib/aspects/types.nix`.
- **feat/fx-pipeline**: Legacy `adapters.nix` is removed from the active source tree (fx pipeline replaced it). The equivalent functionality lives in `nix/lib/aspects/fx/handlers/tree.nix` (`check-constraint` prefix lookup) and `nix/lib/aspects/fx/handlers/transition.nix` (`registerExcludes`). Both branches have the cascade behavior active.

## Supersession

- Superseded in part by `2026-04-04-deep-provider-cascade.md` which changed `__provider` from `nullOr str` to `listOf str` and added provider-aware substitute and deep cascade. The cascade behavior described here was the initial single-level fix; the deep spec extended it to nested providers and substitutes.
- The fx-pipeline refactor (`68c26555`, "refactor: den-fx") replaced the legacy filter-based `excludeAspect` with a constraint/registry system in `tree.nix`/`transition.nix`, but preserved the prefix-cascade semantics.

## Gaps

- Spec described `wrapProvider` in `types.nix` as the tagging mechanism and `apply = lib.mapAttrs (_: wrapProvider parentPath)` on the `provides` option. Neither `wrapProvider` nor `carryAttrs` exist; tagging is done inline via `meta.provider` assignment in `fx/aspect.nix`. Functionally equivalent, architecturally different.
- Spec described `__provider` as an option declared in `aspectSubmodule` (type `listOf str`). Current code uses `meta.provider` as a free attrset key, not a declared option. No `options.__provider` declaration found.
- Angle-bracket excludes (`<monitoring/node-exporter>`) mentioned in spec — not verified as still active in fx-pipeline; `den-brackets.nix` would need separate check.
- Trace `provider` field in Mermaid renderer mentioned in spec — trace visualization is handled by `nix/lib/diag/` which is out of scope here; not verified.

## Drift

- Spec used list-based `isPrefix` comparison on `__provider ++ [name]` paths. Implementation uses slash-joined string prefix lookup (`lib.splitString "/" nodeIdentity` → prefix array → registry lookup). Same semantics, different representation: lists in the old pipeline, slash-joined strings in the fx pipeline.
- The spec's `__provider` tag was intended to travel through `withIdentity` and `carryAttrs` propagation helpers. In the fx pipeline, `meta.provider` is set at construction time in `aspect.nix` and does not need to be carried through — the effectful pipeline constructs each include's metadata at emit time rather than threading it through transformations.
- Provider-aware substitute (from the deep-cascade follow-on spec) is implemented via the same constraint registry (`type = "substitute"`) and not via a separate `substituteAspect` prefix variant; the prefix check applies uniformly to both exclude and substitute lookups.
