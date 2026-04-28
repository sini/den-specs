# Simplified Review Guide for feat/rm-legacy

A plain-language walkthrough of what this branch does, how the pipeline works now, and what's left. Written for someone who understands programming but hasn't read every commit.

## What is den?

Den is a Nix framework for managing fleets of machines. You declare hosts, users, and "aspects" (reusable config bundles). Den's job is to take these declarations and produce NixOS/home-manager configs for each machine.

```nix
den.hosts.x86_64-linux.igloo.users.tux = {};

den.aspects.igloo.nixos.networking.hostName = "igloo";
den.aspects.tux.homeManager.programs.git.enable = true;
```

The **resolution pipeline** is the engine that turns these declarations into actual NixOS modules. This branch rewrites that engine.

## What did the old pipeline do?

The old pipeline was a recursive tree walker. It visited each aspect, checked what context was available (which host? which user?), called parametric functions with the right args, and collected the results. The recursion was manual — each step knew about the next.

Problems:
- Context threading was manual. Every new feature (constraints, tracing, dedup) meant touching the recursion logic.
- Bugs were easy: forget to thread context, lose a value, get infinite recursion. PRs 408-437 were ALL caused by manual context threading mistakes.

## What does the new pipeline do?

The new pipeline uses **algebraic effects** (via the `nix-effects` library). Instead of recursive tree walking, aspects are compiled into **computations** that emit **effects**. Handlers decide what to do with each effect. The trampoline (interpreter) runs the computation.

```
Old: function calls function calls function (manual plumbing)
New: computation emits event → handler catches it → decides what to do
```

### The key players

**aspectToEffect** — The compiler. Takes an aspect attrset and produces a computation:
```
{ nixos = {...}; includes = [...]; }
  → emit "emit-class" {class="nixos"; module={...}}
  → emit "emit-include" {child=...} for each include
```

**Handlers** — Each handler catches one type of effect:
- `constantHandler` — provides context values (host, user, class) when asked via effects
- `classCollectorHandler` — collects NixOS/HM modules that match the target class
- `includeHandler` — processes child aspects (wraps, resolves, recurses)
- `transitionHandler` — handles `into` transitions (host→user fan-out, cross-providers)
- `constraintRegistryHandler` — manages exclude/substitute rules
- `chainHandler` — tracks parent→child provenance
- `collectPathsHandler` / `pathSetHandler` — records which aspects were resolved (for `hasAspect` and `includeIf`)
- `ctxSeenHandler` — dedup for transitions
- `deferredIncludeHandler` / `drainDeferredHandler` — parks parametric includes whose args aren't available yet, resolves them when context widens
- `resolvePolicyHandler` — resolves policy-to-into merge at pipeline start (interceptable)
- `resolveTargetHandler` — assembles transition targets from stages + policies (interceptable)

**The trampoline** — The interpreter loop from `nix-effects`. Takes a computation and a handler set. Runs the computation step by step. Uses `genericClosure` for stack safety.

### How a host gets resolved

```
1. Pipeline starts: resolve-policy effect merges policies into root aspect's into
2. flake has into.flake-system → fans out per system
3. flake-system has into.flake-os → resolve-target effect assembles host target
4. Each host's aspects compiled via aspectToEffect
5. Parametric aspects ({ host }: ...) resolved via bind.fn effects
   → scope.provide installs __scopeHandlers for the subtree
6. Policy handlers (if any) scoped via scope.provide per-transition
7. classCollectorHandler collects matching modules
8. Result: list of NixOS modules for this host
```

### The effectful bootstrap

The pipeline setup is itself effectful — two effects fire before aspect compilation:

1. **`resolve-policy`** — determines the `into` function for the root aspect by merging synthesized policies with explicit `into` declarations. Default handler calls `mergePolicyInto`. Overridable for custom routing.

2. **`resolve-target`** — assembles a transition target from `den.stages` + policy merge. Default handler does the stage lookup + policy merge. Overridable for virtual stages or custom target assembly.

Both effects are sent from inside `fx.handle`, making them interceptable by `extraHandlers` or handler composition.

### Per-policy named effect handlers (Layer 2)

Policies can declare `handlers` — named effect handlers that get `scope.provide`'d into the transition's sub-computation:

```nix
den.policies.host-to-user = {
  from = "host";
  to = "user";
  resolve = { host }: map (u: { inherit host user; }) (attrValues host.users);
  handlers.peer = { param, state }: {
    resume = /* peer entity lookup */;
    inherit state;
  };
};
```

When `host-to-user` fires, aspects resolved under that transition can `{ peer, ... }: ...` to query the policy's handler. Handlers are scoped — they only exist within that transition, not globally. Core pipeline effects are protected from shadowing via a `coreEffects` filter.

### Context propagation: __scopeHandlers and scope.provide

**scope.provide + __scopeHandlers:** Transitions tag target aspects with `__scopeHandlers` — a plain attrset of handler records built via `constantHandler`. At point of use, `aspectToEffect` calls `scope.provide` to install them for `bind.fn` resolution. State-transparent — class collector events pass through unchanged.

### Defensive guards

- **Parametric depth limit** (max 10): Prevents divergence from curried functions that never bottom out. Throws with clear error message naming the offending aspect.
- **Transition depth limit** (max 50): Detects policy cycles (A→B→A) with clear error message.
- **Core effects protection**: Policy handlers cannot shadow core pipeline effects (`emit-class`, `into-transition`, etc.) — stripped via `coreEffects` filter before `scope.provide`.

### Internal `__`-prefixed tags

- `__ctx` — at entry points only (ctxApply, resolveStage). Not for general context propagation.
- `__scopeHandlers` — single source of truth for context propagation. Transitions stamp this.
- `__fn` / `__args` — parametric wrapper fields. Detected by `isParametricWrapper`.
- `__ctxId` — fan-out identity. Distinguishes `tux/{igloo}` vs `tux/{iceberg}`.
- `__parametricResolved` — tells classCollector to preserve ctxId in module keys.
- `__parametricDepth` — recursion counter for depth limit guard.

## What's different from main

### What was deleted

| File | Lines | What it did |
|---|---|---|
| `adapters.nix` | 349 | Legacy adapter layer |
| `resolve.nix` | 68 | Legacy recursive resolve |
| `statics.nix` | 32 | Static aspect handling |
| `fxPipeline.nix` | 11 | Feature gate |
| `ctx-types.nix` | 78 | ctx type definitions |
| `ctx-apply.nix` | 124 | ctx __functor resolution |
| `nixModule/ctx.nix` | 13 | den.ctx option declaration |

### What was rearchitected

**`include.nix`**: Owns ALL include resolution — wrapping (`wrapFunctorChild`, `wrapBareFn`), parametric detection, constraint checking, deferred includes, context propagation.

**`transition.nix`**: Full transitions — `resolve-target` effect, fan-out, cross-providers, ctx-seen dedup, policy handler installation via `scope.provide`, deferred include draining.

**`aspect.nix`**: Aspect compiler — `aspectToEffect` with depth guard, `mkParametricNext`, `tagParametricResult`, `mkPositionalInclude`, `mkNamedInclude`. Shared predicates exported from `types.nix`.

**`pipeline.nix`**: Effectful bootstrap — `resolve-policy` effect at pipeline start, `resolvePolicyHandler`, `resolveTargetHandler` in `defaultHandlers`. `composeHandlers` with documented disjointness constraint.

**`types.nix`**: `providerType` merge decomposed into `mergeMixed`, `mergeFunctions`. Shared predicates (`isParametricWrapper`, `isSubmoduleFn`, `isMeaningfulName`) exported via `default.nix`.

### The four-concern model

`den.ctx` conflated three concerns. This branch separates them:

```
Data (Schema)    — what entities ARE       — den.schema.* + entity types
Policies         — how entities RELATE     — den.policies (directed edges + handlers)
Stages           — where behavior BINDS    — den.stages (scoped behavior)
Behavior         — how entities RESOLVE    — den.aspects.* + fx pipeline
```

```nix
# Policy — directed edge with optional handlers
den.policies.host-to-users = {
  from = "host";
  to = "user";
  resolve = { host }: map (user: { inherit host user; }) (lib.attrValues host.users);
  handlers.peer = { param, state }: { resume = ...; inherit state; };
};

# Stage — named scope for attaching behavior
den.stages.user.includes = [ user-shell-forward ];

# Aspect — pure behavior
den.aspects.gaming.nixos.programs.steam.enable = true;
```

### Compatibility shim

`modules/compat/ctx-shim.nix` — always-imported, forwards `den.ctx.*` to `den.stages.*` with deprecation warnings. Part 1 (scoped behavior) and Part 2 (into forwarding to `meta.into`). Uses `stageSubmodule` directly (not `stageTreeType`) since `den.ctx` was always flat.

### Key files

```
nix/lib/aspects/fx/
  aspect.nix        — aspectToEffect with depth guard, decomposed helpers
  pipeline.nix      — effectful bootstrap, resolve-policy/resolve-target handlers
  identity.nix      — aspect identity paths, pathSet, tombstones
  handlers/
    include.nix     — emit-include: wraps, resolves, defers
    transition.nix  — into-transition: resolve-target effect, policy scope.provide
    ctx.nix         — constantHandler, ctxSeenHandler
    tree.nix        — classCollector, constraints, chain, deferred includes

nix/lib/
  synthesize-policies.nix — synthesize + mergePolicyInto
  resolve-stage.nix       — stage assembly with policy merge
  policy-types.nix        — policyType with handlers field
  stage-types.nix         — stageTreeType, stageSubmodule
  forward.nix             — mkDirectForward, mkAdapter, mkTopLevelAdapter

modules/
  policies/           — core.nix, batteries.nix, flake.nix
  compat/ctx-shim.nix — den.ctx compatibility shim
  options.nix         — schema + entity arg passing
```

## Current test status

**483/483 CI + 15/15 checkmate tests passing.** PR vic/den#475 open.

Templates: bogus, minimal, microvm, nvf-standalone all pass. flake-parts-modules has a regression (see below).

## Known issue: forward-as-handler

`den.provides.forward` calls `den.lib.aspects.resolve fromClass sourceAspect` which runs a FRESH pipeline without the current pipeline's context. Function-valued class keys like `{ pkgs, ... }: ...` get deferred and lost because `pkgs` has no handler in the fresh pipeline.

**Affects:** flake-parts-modules template — `packages` from den aspects don't reach `perSystem.packages`.

**Root cause:** `forwardItem` in `forward.nix` creates a separate `fxResolve` pipeline. On main this was masked because `aspect-chain` carried a unified ctx tree.

**Fix spec:** `~/Documents/den-docs-backup/superpowers/specs/2026-04-23-forward-as-handler-design.md` — replace with `emit-forward` effect handler that resolves within the current pipeline's handler scope.

## What's left

1. **forward-as-handler** — blocking flake-parts-modules CI; also eliminates aspect-chain dependency
2. **Generic fan-out isolation** — current fix is flake-class-scoped
3. **`den.schema.<kind>.policies`** — entity-kind-scoped policy activation
4. **Vestigial removal** — `parametric.nix`, `take.nix`, `perHost-perUser.nix` (blocked on downstream migration)

## Glossary

- **Aspect** — Named, addressable config bundle. Has class keys, includes, provides, into.
- **Effect** — Value emitted by computation. Pauses until handler provides a resume value.
- **Handler** — Catches a specific effect name, returns `{ resume; state; }`.
- **Trampoline** — Interpreter loop. Runs computations via `genericClosure`.
- **bind.fn** — Resolves parametric function args by sending each as an effect.
- **Policy** — Directed edge between entity kinds: `{ from, to, resolve, handlers? }`. Handlers installed via `scope.provide` per-transition.
- **Stage** — Named scope (`den.stages.X`) for binding behavior to pipeline stages.
- **scope.provide** — nix-effects primitive installing handlers for a subtree without forking state.
- **__scopeHandlers** — Single source of truth for context propagation. Built via `constantHandler`.
- **resolve-policy** — Effect sent at pipeline start. Default handler calls `mergePolicyInto`.
- **resolve-target** — Effect sent per-transition. Default handler does stage lookup + policy merge.
- **constantHandler** — Handler factory: ctx attrset → handler set that resumes with context values.
- **Parametric wrapper** — `{ __fn, __args }` attrset. Detected by `isParametricWrapper`.
- **Deferred include** — Parametric include parked when required args have no handlers. Drained when context widens.
