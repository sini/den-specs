# Analysis: Policy Route — Unified Class and Trait Delivery

## Verdict
implemented (core) / not-implemented (fromTrait)

The `policy.route` effect, `registerRouteHandler`, `wrapRouteModules`, `applyRoutes`, scope-prefixed dedup keys, and all three built-in forward conversions (os-class, os-user, wsl) shipped on feat/fx-pipeline. The `fromTrait` route variant (trait-to-module synthesis via `inheritTraits`) was never implemented — the field is unrecognised in `applySimpleRoute`.

## Delivery Target
feat/fx-pipeline

## Evidence

**policy.route constructor** — `nix/lib/policy-effects.nix:70-73`: `route = spec: { __policyEffect = "route"; value = spec; };` shipped in commit `8ab1009f`.

**registerRouteHandler** — `nix/lib/aspects/fx/handlers/route.nix:7-36`: handler processes `"register-route"`, stamps `sourceScopeId = param.sourceScopeId or scope`, deduplicates via `routeKey` format `fromClass>intoClass@sourceScopeId/path`, writes to `scopedRoutes` via `scopedAppend`.

**scopedRoutes state field** — `nix/lib/aspects/fx/pipeline.nix:148`: `scopedRoutes = _: { };` in `defaultState`. Handler wired at line 52 (`// handlers.registerRouteHandler`).

**wrapRouteModules** — `nix/lib/aspects/fx/route/wrap.nix:76-83`: full `adaptModule → nestModule → guardModule` pipeline for each source module. Handles `adaptArgs`, `path`, `guard`. Matches spec design with minor divergence (see Drift).

**applyRoutes** — `nix/lib/aspects/fx/route/apply.nix:205-241`: folds over deduped routes dispatching simple (`applySimpleRoute`) vs complex (`applyComplexRoute`). Called from `nix/lib/aspects/fx/resolve.nix:150-163` as Phase 3, after wrapping and before instantiates.

**Policy classification of route effects** — `nix/lib/aspects/fx/policy/classify.nix:64`: `routeEffects = filterEffect "route" r.effects;` and line 134 tags each with `__routePolicyName`. `nix/lib/aspects/fx/policy/apply.nix:64`: `sendEach "register-route" (e: e.value) routeEffects` in `policyEmitEffects`.

**Scope-prefixed ctxSeen dedup keys** — `nix/lib/aspects/fx/handlers/ctx.nix:28-32`: `scopedKey = if scope == null || scope == "__unscoped" then key else "${scope}/${key}"` — prerequisite spec item shipped.

**Scope-prefixed includeSeen dedup keys** — `nix/lib/aspects/fx/handlers/check-dedup.nix:21`: `dedupKey = if rawDedupKey != null then "${scope}/${rawDedupKey}" else null` — prerequisite shipped.

**Built-in policy: os-class** — `modules/aspects/provides/os-class.nix:27-39`: `den.policies.os-to-host` emits `den.lib.policy.route { fromClass = "os"; intoClass = host.class; path = []; }`. Commit `a48e2efc`.

**Built-in policy: os-user** — `modules/aspects/provides/os-user.nix:43-56`: `den.policies.user-to-host` emits route with `adaptArgs = args: args // { osConfig = args.config; }` at `users.users.<userName>`. Commit `5a1cd0d2`.

**Built-in policy: wsl** — `modules/aspects/provides/wsl.nix:78-85`: `den.policies.wsl-to-host` emits route `fromClass = "wsl"; intoClass = host.class; path = ["wsl"]; guard = ...`. Commit `b31b4179`.

**Forward unification** — commit `3e064032`: deleted `scopedForwardSpecs`, `applyForwardSpecs`, `resolveForwardSource`. All forwards now register in `scopedRoutes` via `compile-forward` handler (`nix/lib/aspects/fx/handlers/compile-forward.nix`). Simple forwards become `simpleRoute`; complex (adapter/guard) become `__complexForward = true` route specs handled by `applyComplexRoute`.

**Route test coverage** — `templates/ci/modules/features/route.nix`: tests for class-toplevel, class-into-subpath, guarded-false, empty-source, guarded-with-path. No `route-builtin.nix` file exists; builtin regression covered by existing `os-class.nix`, `os-user-class.nix`, `wsl-class.nix` test suites.

**fromTrait — NOT implemented**: no `fromTrait` field handling in `applySimpleRoute` (`route/apply.nix:114-158`). No `traitRouteModule` synthesizer. No call to `inheritTraits` from route apply. The spec's `fromTrait` branch is absent.

## Current Status

Core `policy.route` (class routes) exists and is the primary delivery mechanism. The two-tier model from the spec collapsed into a unified route table: simple policy routes and former Tier 2 forwards both write to `scopedRoutes`, with `__complexForward` as the dispatch flag. `den.provides.forward` user API still exists (`modules/aspects/provides/forward.nix:40`) backed by `forwardEach`/`forwardItem` in `nix/lib/forward.nix` which compile to `__complexForward` routes.

## Supersession

The spec's inline-walk for unreachable source entities was superseded by `applyComplexRoute`'s `resolveSourceFallback` function (`route/apply.nix:24-38`), which calls `fxResolve` with the source aspect directly rather than inline-walking via `aspectToEffect`. The Tier 1/Tier 2 distinction from the spec was further collapsed: commit `3e064032` unified all forwards into a single route table, eliminating the two-tier framing entirely.

Succeeded by: `2026-05-02-pipeline-simplification.md` (route unification) and `2026-05-04-fx-simplification.md` (further consolidation).

## Gaps

- **`fromTrait` not implemented**: `policy.route { fromTrait = "persist"; ... }` is not handled. `applySimpleRoute` reads only `route.fromClass`, never `route.fromTrait`. No trait-to-module synthesis (`inheritTraits` call, `lib.setAttrByPath` wrapping). The spec's trait route examples are non-functional.
- **`route-builtin.nix` test file not created**: spec called for a dedicated `templates/ci/modules/features/route-builtin.nix`. Regression coverage exists but spread across existing test files.
- **`adaptArgs` on nested path differs from spec**: spec's `wrapRouteModules` applies `adaptModule` before nesting regardless of path. Implementation only applies `adaptModule` at `path == []` (top-level); nested paths use `nestWithAdaptArgs` via `evalModules/specialArgs` instead of the spec's simpler `args: mod (adaptArgs args)` approach.

## Drift

- **Route table unification**: spec describes explicit Tier 1 / Tier 2 separation throughout pipeline. Implementation merged both into `scopedRoutes` with `__complexForward` flag, eliminating the tier framing. Routes and forwards are now the same state entity with different apply paths.
- **`wrapRouteModules` nesting strategy**: spec uses `lib.setAttrByPath path { imports = [ mod ]; }` for all nested paths, relying on the target option being a submodule type. Implementation uses `evalModules` with `specialArgs` for `adaptArgs` paths and plain config assignment for plain paths — avoids requiring a submodule type at the target path.
- **`registerRouteHandler` dedup**: spec shows simple `scopedAppend`; implementation adds a `routeKey`-based dedup guard (`registeredRouteKeys` state field, `route.nix:15-29`) to suppress duplicate registrations across policy re-fires.
- **sourceScopeId in handler**: spec stamps `sourceScopeId = scope` unconditionally. Implementation uses `param.sourceScopeId or scope`, allowing callers (e.g., `compile-forward`) to pre-compute cross-entity scope IDs before registration.
- **`guard` function signature**: spec describes `guard = { options, ... }: bool` applied via `lib.mkIf`. Implementation passes full module args (including `config`, `options`) to the guard and wraps the entire module result in `lib.mkIf (guard args)` — matches semantics but args set is richer than spec example suggests.
