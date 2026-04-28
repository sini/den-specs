# Analysis: nix-effects-powered aspect resolution

## Verdict
implemented — core freer monad pipeline fully shipped; some surface API details diverged from spec

## Delivery Target
(B) feat/fx-pipeline (landed as commit 68c26555 "refactor: den-fx (#462)", then extensively built out through current HEAD on feat/fx-pipeline)

## Evidence

**Foundational commit:**
- `68c26555` — "refactor: den-fx (#462)" — 158 files, 7420 insertions. Ships `aspectToEffect`, handler-owned recursion, constraint system, chain provenance, `den.lib.fx` access, and `den.fxPipeline` option. This is the spec landing.

**nix-effects dependency:**
- `nix/lib/fx.nix` — fetches `nix-effects` via `builtins.fetchTarball` from `templates/ci/flake.lock` pin; falls back to `inputs.nix-effects.lib`. Pinned at `sini/nix-effects` rev `cf958815`. Not a flake input on the root `flake.nix` (which is a 3-line delegation to `./nix`); it's a CI-template flake input in `templates/ci/flake.lock`.

**bind.fn in production:**
- `nix/lib/aspects/fx/aspect.nix:928` — `resolveFn = if scopeFn != null then scopeFn (fx.bind.fn userArgs fn) else fx.bind.fn userArgs fn;`
- Every parametric aspect resolved via `fx.bind.fn userArgs fn` — exactly as spec proposed.

**FTCQueue / genericClosure trampoline:**
- Not visible in den source directly; these are internals of the nix-effects library. Den uses `fx.handle`, `fx.send`, `fx.bind`, `fx.pure`, `fx.rotate`, `fx.seq`, `fx.effects.scope.provide` — all nix-effects API surface. The trampoline runs inside nix-effects.

**Handler set in pipeline.nix:**
- `constantHandler` — provides ctx args (host, user, class, aspect-chain) to `bind.fn`
- `chainHandler` — `chain-push` / `chain-pop` for includes-chain provenance
- `classCollectorHandler` — `emit-class` collects per-class modules into `state.classImports`
- `constraintRegistryHandler` — `register-constraint` / `check-constraint`
- `includeHandler` — `emit-include` / `include-unseen` / `defer-include` / `drain-deferred`
- `transitionHandler` — `into-transition` / `ctx-seen` / `resolve-entity`
- `traitCollectorHandler` — `emit-trait`
- `traitArgHandler` — trait names as effects for parametric consumers
- `forwardHandler` — `emit-forward`
- `provideToHandler` — `provide-to`
- `fx.effects.state.handler` — state management (always last)

**Key files:**
- `nix/lib/aspects/fx/pipeline.nix` — `fxResolve`, `fxFullResolve`, `mkPipeline`, `runSubPipeline`, `applyForwardSpecs`
- `nix/lib/aspects/fx/aspect.nix` — `aspectToEffect`, `compileStatic`, `emitClasses`, `emitTraits`, `emitIncludes`, `emitTransitions`, `wrapClassModule`
- `nix/lib/aspects/fx/handlers/` — ctx, include, tree, transition, trait, forward, provide-to, provides-compat
- `nix/lib/fx.nix` — nix-effects import

**Legacy files status:**
- `nix/lib/take.nix` — still exists but fully deprecated; all variants emit `lib.warn` directing to `bind.fn`
- `nix/lib/can-take.nix` — still exists; used internally only for `isSubmoduleFn` detection in `types.nix` (NixOS module function detection, not context-guessing)
- `nix/lib/parametric.nix` — still exists but all methods deprecated (warn); `fixedTo` rewritten to use `__scopeHandlers` + `constantHandler` instead of manual context threading

## Current Status

Pipeline fully operational on `feat/fx-pipeline`. The spec's replacement goal is achieved:

- `take.nix` — retained as deprecated shim; not deleted as spec planned
- `can-take.nix` — retained; `isSubmoduleFn` still uses `canTake.upTo` for NixOS module detection (not the same use case spec was eliminating)
- `parametric.nix` — retained as deprecated shim wrapping `__scopeHandlers`
- `statics.nix` — does not exist in current tree; folded into handler as spec planned
- `aspects/adapters.nix` — does not exist; adapter pattern replaced by constraint system (`meta.handleWith`, `meta.excludes`)
- `ctx-apply.nix` — does not exist; context stages replaced by `into-transition` handler + `scope.provide`
- `coercedProviderType` — still exists in `nix/lib/aspects/types.nix:436` but scoped to a narrow case (parametric fn coercion to `{ includes = [fn]; }` so `bind.fn` can resolve it — different from spec's "not needed" claim)

## Supersession

This spec supersedes `2026-04-12-nfx-resolution-spec.md` (nfx primitives, not implemented). No later spec supersedes this one — it is the implemented pipeline.

Subsequent specs built on this foundation:
- `2026-04-25-traits-and-structural-nesting-design.md` — traits added atop the pipeline
- `2026-04-27-unified-policy-effects-design.md` — policy effects system built on `into-transition`
- `2026-04-28-policy-pipeline-simplification.md` — simplified policies to plain functions

## Gaps

**Not implemented as specced:**

1. **Conditions system** — spec described Common Lisp-style resumable errors via `fx.send "condition" { type = ...; }` with `strictHandlers.condition = ... abort = ...`. Actual strict mode (`nix/lib/strict.nix`) is a NixOS freeform type that `throw`s directly — no condition effects, no handler-swap for lenient/diagnostic modes.

2. **`has-aspect` as named effect** — spec proposed `send "has-aspect" ref` as an in-computation effect. Actual implementation (`nix/lib/aspects/has-aspect.nix`) runs a full sub-pipeline (`fxFullResolve`) to extract `pathSet`, then queries that set. It is NOT an effect sent during aspect computation — there is no `"has-aspect"` handler in `defaultHandlers`. The `get-path-set` effect exists (answered by `pathSetHandler` with the accumulated pathSet), but `hasAspect` is a host-level entity attribute, not an inline effect.

3. **User-facing `fx.bind.fn` in aspects** — spec showed users writing `aspect = fx.bind.fn ({ host, user }: fx.pure { ... })`. Actual API: users write plain `{ host, user }: { ... }` functions which are coerced via `coercedProviderType` to `{ includes = [fn]; }` and then `bind.fn` resolves them internally. Users never call `fx.bind.fn` directly. Internal plumbing is as specced; the user-facing surface is simpler.

4. **`resolve-child`, `class-module`, `structural-presence` effect names** — spec proposed these specific names in the `fx.run` example. Actual effect names are different: `emit-include` (not `resolve-child`), `emit-class` (not `class-module`), `get-path-set` (not `structural-presence`). The structure is the same but names diverged.

5. **`excludeAspectHandler` / `substituteHandler` / `oneOfAspectsHandler` combinator API** — spec proposed adapters as handler combinator functions returning modified outer handlers. Actual implementation: `meta.handleWith` / `meta.excludes` declare constraints on the aspect itself; the `constraintRegistryHandler` evaluates them during `emit-include`. No public handler-combinator API.

6. **Root flake.nix as nix-effects input** — spec file plan lists `flake.nix: Add nix-effects input`. The main `flake.nix` (3 lines) has no inputs; nix-effects is accessed via `builtins.fetchTarball` in `nix/lib/fx.nix` using the CI template's lock file. In `templates/ci/flake.nix` nix-effects IS a proper flake input.

## Drift

1. **Effect naming** — spec used intuitive names (`resolve-child`, `class-module`) from a resolution-centric framing. Implementation uses emission-centric names (`emit-include`, `emit-class`) reflecting that the aspect compiler emits effects and the handler drives recursion. Semantically equivalent, architecturally cleaner.

2. **`has-aspect` lifecycle** — spec proposed it as an inline effect in the computation (cycle-safe because handler has precomputed structural set). Actual: `hasAspect` runs a separate sub-pipeline before the main resolve and is exposed as a host attribute, not an inline send. This is simpler to reason about but loses the spec's "safe in conditional includes" property for case (iii).

3. **Adapter vs. constraint** — spec proposed functional adapter combinators (`excludeAspectHandler`, `substituteHandler`) that compose by overriding effect names. Actual: declarative constraint objects on aspects (`meta.handleWith`, `meta.excludes`) interpreted by a single constraint registry handler. More composable and less error-prone than the functional combinator approach.

4. **`statics.nix` folding** — spec said fold into "a handler that provides `{ class, aspect-chain }`". Actual: `constantHandler` in `defaultHandlers` provides class; `aspect-chain` is in the same `constantHandler` invocation. Not a separate "statics handler" but the intent is identical.

5. **File plan fidelity** — spec's file deletion plan (`take.nix`, `can-take.nix`, `parametric.nix` deleted) was not fully executed; they remain as deprecated shims for backward compat. The core algorithmic replacement happened; the file cleanup was deferred.

6. **`coercedProviderType` retained** — spec said "gone". It survived in a narrower role: coercing bare parametric functions into `{ includes = [fn]; }` so `bind.fn` can resolve them inside the pipeline. The bug-prone coercion use cases (factory function detection, `functionArgs` guessing) were eliminated; this residual coercion is clean.
