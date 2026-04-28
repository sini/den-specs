# Analysis: Diag Branch Rebase onto Unified Effects Pipeline

## Verdict

SUPERSEDED — fully implemented, by a different path than specced. The spec anticipated rebasing two old branches (`feat/diag-api`, `feat/diag-templates`) onto `feat/fx-resolution`. Instead, the diag library was written fresh onto the already-unified pipeline on `feat/fx-pipeline` (2026-04-24, after legacy removal landed 2026-04-23). No rebase was needed; no old API references survive.

## Delivery Target

`feat/fx-pipeline` — commit `8ba78575` (2026-04-24). The diag library (`nix/lib/diag/`) was introduced in a single large commit that already used the new unified API throughout.

## Evidence

- `nix/lib/diag/capture.nix:15` — `fxLib = den.lib.aspects.fx;` (no `init` call)
- `nix/lib/diag/capture.nix:20` — `comp = fxLib.aspect.aspectToEffect root;` (new API)
- `nix/lib/diag/capture.nix:23` — `fxLib.pipeline.composeHandlers`, `fxLib.pipeline.defaultHandlers` (new module)
- `nix/lib/diag/default.nix:52` — `capture = import ./capture.nix { inherit den lib; };` (no `inputs` / `nix-effects` conditional, no `fxEnabled` guard)
- `nix/lib/aspects/fx/default.nix` — no `init` export, no `resolve` module; modules are `aspect`, `pipeline`, `handlers`, `trace`, `identity`, `constraints`, `includes`, `distributeCrossEntity`
- Branches `feat/diag-api` and `feat/diag-templates` do not exist locally or as remotes (only `feat/diagram-demo-renders` exists)
- `git merge-base 8ba78575 2e0fc9e1` returns `2e0fc9e1` — the diag commit is a descendant of legacy removal; it was born onto the unified pipeline
- Grep for all old API patterns (`fxLib.resolve.*`, `.init`, `resolveDeepEffectful`, `provide-class`, `resolve-include`, `provideClassHandler`, `parametricHandler`, `resolveIncludeHandler`, `wrapAspect`) across `nix/lib/diag/` and `templates/` returns no matches

## Current Status

All spec verification criteria are met:

- No references to `fxLib.resolve.*` in diag code
- No `init` call in diag code
- No `resolveDeepEffectful` reference
- `fxLib = den.lib.aspects.fx` (direct, unconditional)
- `aspectToEffect` used in `captureRaw` and `captureWithPathsWith`
- `pipeline.composeHandlers`, `pipeline.defaultHandlers`, `pipeline.defaultState` used throughout
- No separate `resolveIncludeHandler` composition — `defaultHandlers` already includes it
- CI test fixtures `fx-diag-capture.nix` and `fx-diag-context.nix` present and use `den.lib.aspects.fx` directly
- `templates/diagram-demo/` (not `diag-fx-demo`) exists as the template counterpart; no old API usage

The spec mentioned `diag-fx-demo` template and an `fx-debug.nix` module — neither exists. The shipped template is `diagram-demo` with `modules/diagrams.nix`.

## Supersession

The spec was written 2026-04-15, targeting a rebase of branches that were pre-unified. By 2026-04-24, those branches were bypassed entirely: the diag library was authored fresh on top of the already-unified `feat/fx-pipeline`. The spec's rebase strategy (three phases: diag-api rebase, fix callers, diag-templates rebase) was never executed; the outcome it aimed for was achieved more directly.

## Gaps

None remaining against spec intent. One naming difference: the shipped template is `diagram-demo`, not `diag-fx-demo` as the spec assumed. This is cosmetic — the template serves the same purpose.

The spec's `fxEnabled && inputs ? nix-effects` conditional is gone entirely; diag now assumes fx is always available (no optional gate). This is a design improvement — simplifies the module, no longer needs the `inputs` argument in `default.nix`.

## Drift

- `nix/lib/diag/default.nix` takes `{ lib, den, inputs, ... }` per spec but the `inputs` arg is present in the signature yet not used for `fxLib` init — it may be used by other downstream imports or is vestigial. The spec's conditional `fxEnabled` path was dropped.
- `capture.nix` does not emit a manual `resolve-complete` at the root (spec suggested this). The pipeline's `defaultHandlers` handles `resolve-complete` internally via `transition.nix`.
- `context.nix` has no `fxLib.*` references at all — confirms spec prediction that it needed no changes once `capture.nix` was fixed.
