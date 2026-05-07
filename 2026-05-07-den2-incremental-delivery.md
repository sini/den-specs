# Den 2: Incremental Delivery via Parallel Engines

Companion to [Den 2: Ground-Up Rebuild](2026-05-07-den2-stream-rebuild.md) and [Stream Architecture Migration](2026-05-07-stream-architecture-migration.md).

The rebuild spec proposes ~1,400 lines of stream-based resolution engine replacing ~7,300 lines of handler-based pipeline. This spec defines how to get there incrementally — no flag day, no regression window, rollback at every step.

## Strategy: Parallel Engines

Both engines consume the same input (`den.aspects`, `den.policies`, `den.hosts`) and produce the same output (module lists for `nixosSystem`). They're interchangeable black boxes:

```nix
# Old engine (handler-based trampoline)
oldResult = den.lib.aspects.resolve "nixos" rootAspect;

# New engine (stream-based)
newResult = den.lib.resolve2 { inherit aspects policies hosts pipes classes schema default; };

# Same output shape: { imports = [ ... ]; }
```

Build `resolve2` alongside the current engine. Both coexist in the repo. Tests run against both. Switch features one layer at a time. Delete old code when all 753 tests pass on the new engine.

### Why this works

1. **No flag day.** Both engines coexist at every phase. The old engine runs all tests throughout.
2. **Each phase is testable.** Clear test subsets validate each layer. Failures localize to the layer just added.
3. **Rollback at any point.** If Phase 4 (pipes) is harder than expected, the old engine still works.
4. **API changes are separate from engine changes.** Phase 7 changes the surface; Phases 0-6 change internals. Never debug both simultaneously.
5. **Reviewable PRs.** Each phase is a self-contained chunk. Phase 0 is ~280 lines.
6. **Zero regression window.** The old engine handles production. The new engine catches up.

### Test accounting

The CI suite has **753 tests** across the following categories:

| Category | Tests | Notes |
|----------|-------|-------|
| Behavioral integration tests | ~539 | Test observable behavior (declarations in → config out) |
| `fx-*` internal engine tests | ~214 | Test handler internals (scope push/pop, bind, gate, etc.) |

The `fx-*` tests (including `narrow-effects.nix`) drive old-engine internals directly (handlers, pipeline state, scope effects). These **do not port** — they test machinery that won't exist. They get replaced by new engine-specific tests as each phase lands. The behavioral tests port unchanged or with minimal API adaptation.

**Note on per-file test counts:** The per-file counts in phase tables below are approximate (derived from test-key enumeration, which can overcount). The total of 753 is confirmed by CI. Use `nix-unit --flake .#tests.<suite>` to get exact counts per suite before starting each phase.

## Phase 0: Foundation

**Goal:** One host, one static aspect, no policies, no pipes, no forwards. Prove the core architecture works end-to-end.

**Lines added:** ~280 (Ned core import + resolve2 entry point + minimal class routing + module wrapping)

**What works after this phase:**

```nix
den.hosts.x86_64-linux.igloo.users.tux = {};
den.aspects.igloo.nixos.networking.hostName = "igloo";
# → nixosSystem with hostName = "igloo"
```

**Validates:** ST, ctxD, hostsT, usersT, hostUserD, class sink routing, module key/file tagging, the Cycle structure.

**Test files:**

| File | Tests | What it validates |
|------|-------|-------------------|
| `aspect-content-type.nix` | 6 | Content-type classification |
| `aspect-key-type.nix` | 4 | Key type shapes |
| `aspect-meta.nix` | 6 | Meta/name fields |
| `apply.nix` | 1 | Basic apply |
| `apply-non-exact.nix` | 2 | Non-exact apply |
| `empty-aspects.nix` | 1 | No-aspects edge case |
| `host-options.nix` | 5 | Host name, user name |
| `hostname.nix` | 6 | den.provides.hostname |
| `host-propagation.nix` | 1 | Host config propagation |
| `den-as-lib.nix` | 5 | Den as plain library |
| `den-default.nix` | 3 | den.default |
| `homes.nix` | 6 | Standalone homes |
| `home-flat-form.nix` | 2 | Flat-form home |
| `home-managed-home.nix` | 2 | Home class |
| `define-user.nix` | 3 | User definition |
| `primary-user.nix` | 1 | Primary-user include |
| `insecure.nix` | 1 | Insecure packages |
| `unfree.nix` | 3 | Unfree packages |
| `use-global-pkgs.nix` | 3 | Global pkgs |
| `flake-parts.nix` | 2 | Flake-parts |
| `strict.nix` | 10 | Strict mode |

**Test target: ~73 behavioral tests**

**Deliverable:** PR with `nix/lib/resolve2.nix` and Ned primitives in `nix/lib/st/`. Old engine untouched.

## Phase 1: Aspect Resolution

**Goal:** Include tree expansion, nested aspects, identity-based dedup, key classification.

**Lines added:** ~150 (expandAspect, normalizeChild, identity, classifyKeys, distinctByIdentity)

**What works after this phase:**

```nix
den.aspects.igloo = {
  nixos.networking.hostName = "igloo";
  includes = [ den.aspects.shared-tools ];
};
den.aspects.shared-tools.nixos.environment.systemPackages = [ pkgs.git ];

# Nested aspects
den.aspects.igloo.servers.nixos.services.nginx.enable = true;
```

**Validates:** DAG linearization, dedup (diamond includes), nested aspect auto-detection, heterogeneous includes list.

**Test files:**

| File | Tests | What it validates |
|------|-------|-------------------|
| `include-dedup.nix` | 13 | Static/parametric dedup, functor conflict |
| `nested-aspects.nix` | 6 | Nesting, multi-level, backward compat |
| `nested-class-module-args.nix` | 3 | Guard skip, dedup with two parents |
| `default-includes.nix` | 4 | Host service, HM all-users |
| `conditional-config.nix` | 3 | NixOS/HM conditional imports |
| `structural-detection.nix` | 11 | Schema registry class detection |
| `has-aspect.nix` | 31 | Aspect presence/absence |
| `has-aspect-lib.nix` | 10 | hasAspectIn, collectPathSet |
| `aspect-path.nix` | 11 | aspectPath, excludeAspect |
| `angle-brackets.nix` | 6 | Angle-bracket access |

**Test target: ~98 behavioral tests** (cumulative ~171)

## Phase 2: Parametric Aspects

**Goal:** Context-aware aspects resolve through ctxD. `den.lib.parametric` wrapping becomes unnecessary.

**Lines added:** ~30 (ctxD already handles function arg resolution; this phase adds depth-bounded unfolding for parametric-returning-parametric)

**What works after this phase:**

```nix
# Context-aware — just works, no wrapper needed
den.aspects.igloo = { host, ... }: {
  nixos.networking.hostName = host.name;
};

# Dual-function form
den.aspects.foo = { host }: { config, ... }: {
  nixos.networking.hostName = "${host.name}-${config.networking.domain}";
};

# Skip on missing arg
den.aspects.never = { nonexistent, ... }: { nixos.foo = true; };  # silently skipped
```

**Validates:** ctxD function arg resolution, skip-on-missing-arg, dual-function form, depth-bounded recursion, class-module partial application.

**Test files:**

| File | Tests |
|------|-------|
| `parametric.nix` | 4 |
| `parametric-context.nix` | 8 |
| `parametric-fixedTo.nix` | 26 |
| `auto-parametric.nix` | 7 |
| `top-level-parametric.nix` | 3 |
| `provides-parametric.nix` | 6 |
| `ctx-chain.nix` | 4 |
| `ctx-pipeline.nix` | 3 |
| `ctx-user-hm.nix` | 1 |
| `nested-ctx.nix` | 12 |
| `custom-ctx.nix` | 5 |
| `class-module-partial-apply.nix` | 53 |
| `user-host-mutual-config.nix` | 9 |
| `perUser-perHost.nix` | 3 |
| `flake-scope-pipeline-args.nix` | 10 |
| `hjem-class.nix` | 3 |
| `maid-class.nix` | 3 |
| `os-class.nix` | 2 |
| `os-class-host.nix` | 4 |
| `os-user-class.nix` | 4 |
| `user-classes.nix` | 3 |
| `wsl-class.nix` | 2 |
| `tty-autologin.nix` | 2 |
| `user-shell.nix` | 1 |

**Test target: ~178 behavioral tests** (cumulative ~349)

**Note:** This is the largest test phase because parametric aspects touch everything — class-module partial application alone is 53 tests. Getting this phase right is critical.

## Phase 3: Policy Dispatch

**Goal:** Policy enrichment convergence, context-dependent dispatch, constraints (exclude/substitute/filter).

**Lines added:** ~160 (converge, dispatchD, effect classification, constraint operators)

**What works after this phase:**

```nix
den.policies.enrich = { host, ... }: [
  (den.lib.policy.resolve { isNixos = host.class == "nixos"; })
];

den.policies.guard = { isNixos, ... }: [
  (den.lib.policy.include { nixos.security.sudo.enable = true; })
];

den.aspects.igloo.meta.excludes = [ den.aspects.dangerous ];
```

**Validates:** enrichment convergence (fixed-point, max 10 iterations), `{ host, ... }:` dispatch gating, policy.include/exclude/provide, constraint enforcement, `policy.when`/`policy.for` combinators.

**Test files:**

| File | Tests |
|------|-------|
| `policies.nix` | 6 |
| `policy-combinators.nix` | 11 |
| `policy-context-enrichment.nix` | 10 |
| `policy-excludes.nix` | 3 |
| `policy-include-routing.nix` | 4 |
| `policy-inspect.nix` | 15 |
| `policy-provide.nix` | 4 |
| `policy-resolve-shared.nix` | 4 |
| `policy-type.nix` | 2 |
| `route.nix` | 6 |
| `doc-examples.nix` | 7 |
| `stages.nix` | 4 |

Note: `fx-constraints.nix` has ~21 tests but mixes behavioral constraint semantics with handler-internal tests (`fx.send "register-constraint"`, `constraintRegistryHandler`). The behavioral subset ports here; the handler-internal tests are non-porting fx-* tests. Similarly, `narrow-effects.nix` (7 tests) is entirely fx-internal and does not port.

**Test target: ~80 behavioral tests** (cumulative ~429)

## Phase 4: Pipes

**Goal:** Pipe transform stages, cross-host pipe.collect via Cycle fixed-point, pipe.expose, targeted delivery, provenance, config thunks.

**Lines added:** ~130 (pipe combinators, the targeted Cycle fixed-point for pipe data)

**What works after this phase:**

```nix
den.pipes.firewall = {};
den.aspects.nginx.firewall = { ports = [ 80 443 ]; };
den.aspects.networking.nixos = { firewall, ... }: {
  networking.firewall.allowedTCPPorts = concatMap (f: f.ports) firewall;
};

den.policies.fleet-backends = { host, ... }:
  let inherit (den.lib.policy) pipe; in [
    (pipe.from "http-backends" [
      (pipe.collect ({ host, ... }: true))
      pipe.withProvenance
    ])
  ];
```

**Validates:** Local aggregation, pipe.filter/transform/fold/append/for, pipe.collect with predicate + self-exclusion, pipe.expose (child→parent), pipe.to (targeted), pipe.withProvenance, config thunks.

**Test files:**

| File | Tests |
|------|-------|
| `pipes.nix` | 8 |
| `pipe-policy.nix` | 13 |
| `pipe-scope.nix` | 16 |
| `provide-to.nix` | 3 |

**Test target: ~40 behavioral tests** (cumulative ~486)

**This is the Cycle phase.** The targeted fixed-point for pipe data cross-referencing is the only genuinely circular part of the architecture. If this works, the hardest architectural question is answered.

## Phase 5: Forwards

**Goal:** Class-to-class forwarding via stream concat, adapter functor for guards/adaptArgs, forwardTo wiring, policy.route.

**Lines added:** ~110 (adapter functor, forwardItem rewrite, forwardTo, policy.route processing)

**What works after this phase:**

```nix
den.classes.os.forwardTo = { class = "nixos"; };
den.aspects.igloo.os.networking.hostName = "igloo";

# Guarded forward
guard = { config, ... }: _: lib.mkIf config.programs.vim.enable;

# policy.route
den.policies.os-to-host = { host, ... }: [
  (den.lib.policy.route { fromClass = "os"; intoClass = host.class; })
];
```

**Validates:** Simple forwards (stream concat), guarded forwards (lib.mkIf), adaptArgs (specialArgs injection), dynamic intoPath, adapter functor (`den.fwd.${key}`), forwardTo wiring, policy.route, cross-context forwards.

**Test files:**

| File | Tests |
|------|-------|
| `forward.nix` | 2 |
| `forward-to.nix` | 4 |
| `forward-flake-level.nix` | 6 |
| `forward-alias-class.nix` | 3 |
| `forward-from-custom-class.nix` | 5 |
| `guarded-forward.nix` | 3 |
| `cross-context-forward.nix` | 3 |
| `debug-fwd.nix` | 6 |
| `dynamic-intopath.nix` | 2 |

**Test target: ~34 behavioral tests** (cumulative ~520)

## Phase 6: Custom Entities, Schema, Namespaces

**Goal:** Schema-to-driver generation, custom entity kinds (fleet, environment), namespaces, provider system.

**Lines added:** ~80 (mkEntityDriver factory, schema evaluation, namespace identity)

**What works after this phase:**

```nix
den.schema.fleet = {};
den.schema.flake.includes = [ den.policies.to-fleet ];
den.schema.fleet.includes = [ den.policies.fleet-to-hosts ];

# Namespaces
imports = [ (inputs.den.namespace "ns" false) ];
ns.my-aspect.nixos.foo = true;
```

**Validates:** Custom entity kinds, schema-to-driver generation, fleet topology, namespace aspect identity, provider system (named-provider, cross-provider, mutual-provider), schema options/includes split.

**Test files:**

| File | Tests |
|------|-------|
| `schema-registry.nix` | 13 |
| `schema-base-modules.nix` | 8 |
| `namespace.nix` | 3 |
| `namespace-provider.nix` | 2 |
| `namespace-schemas.nix` | 7 |
| `namespaces.nix` | 5 |
| `named-provider.nix` | 15 |
| `cross-provider.nix` | 9 |
| `mutual-provider.nix` | 4 |
| `special-args-custom-instantiate.nix` | 1 |
| `host-aspects.nix` | 10 |
| `hm-host-isolation.nix` | 2 |

**Test target: ~79 behavioral tests** (cumulative ~599)

## Phase 7: Regression Corpus + Edge Cases

**Goal:** All 52 deadbug regression tests pass. Edge cases: depth stress tests, pure-eval, performance benchmarks.

**Lines added:** ~20 (edge case handling discovered during regression testing)

**Test files:**

| Category | Tests |
|----------|-------|
| `deadbugs/` (21 files) | 52 |
| `depth.nix` | 5 |
| `resolve.nix` | 4 |
| `pure-eval.nix` | 4 |
| `ctx-compat.nix` | 4 |

**Test target: ~69 tests** (cumulative ~668)

## Phase 8: API Changes + Compat Shims

**Goal:** Apply the 7 API improvements from the Den 2 spec. Add thin compat shims so old-syntax tests pass on both engines.

**Lines added:** ~50 (compat shims, API surface changes)

**API changes applied:**

1. `den.lib.parametric` → identity shim (wrapping is now automatic)
2. `excludes` as top-level aspect key (+ shim for `meta.handleWith`)
3. Aspect-scoped policy auto-activation (with `autoActivate` opt-out)
4. Policy effect constructors injected via context (`{ host, include, provide, ... }:`)
5. `den.quirks` → `den.pipes` (+ alias)
6. `den.provides.forward` retired (+ shim delegating to `forwardTo`/`policy.route`)
7. ~~Users as list~~ (dropped)
8. `den.schema` split into `.options` and `.includes`

**Compat shims:**

```nix
# Thin shims that let old-syntax tests pass
den.lib.parametric = x: x;  # identity — wrapping is automatic
den.lib.perHost = f: f;     # identity — ctxD handles context
den.quirks = den.pipes;     # alias
```

**Test target: remaining ~85 tests** (cumulative: all 753 behavioral + new engine tests)

## Phase 9: Delete Old Engine

**Goal:** Remove the handler-based pipeline. Rename resolve2 → resolve.

**Lines added:** 0
**Lines deleted:** ~5,500

**What gets deleted:**

```
nix/lib/aspects/fx/handlers/          # 35+ handler files (~2,100 lines)
nix/lib/aspects/fx/pipeline.nix       # trampoline bootstrap (~235 lines)
nix/lib/aspects/fx/resolve.nix        # 4-phase post-pipeline (~557 lines)
nix/lib/aspects/fx/assemble-pipes.nix # pipe assembly (~632 lines)
nix/lib/aspects/fx/wrap-classes.nix   # class wrapping (~190 lines)
nix/lib/aspects/fx/class-module.nix   # collision detection (~254 lines)
nix/lib/aspects/fx/route/             # route application (~300 lines)
nix/lib/aspects/fx/policy/            # policy dispatch (~500 lines)
nix/lib/aspects/fx/aspect/            # aspect processing (~388 lines)
```

Also deleted: compat shims from Phase 8, `fx-*` internal test files (~214 tests that tested old engine internals).

**Post-deletion test count:** 753 - 214 (old engine internals) + N (new engine-specific tests) = ~539+ behavioral tests, all passing on the stream engine.

The `fx-*` internal tests are replaced incrementally during Phases 0-6 with stream-engine-specific tests that validate the new internals (ST operations, ctxD scoping, Cycle fixed-point behavior, etc.).

**Diag subsystem:** The diag subsystem (~3,000 lines, `fx-trace.nix` + `fx-diag-*` = ~19 tests) reads old pipeline state (handler traces, scope contexts). Phase 9 breaks diag. This is accepted — diag is a separate redesign effort using stream tracing (`traceD` driver). Diag tests are excluded from the 753 behavioral target and documented as broken-until-redesigned.

## Summary

| Phase | Lines + | Lines - | Cumulative | Tests added | Milestone |
|-------|---------|---------|------------|-------------|-----------|
| 0: Foundation | ~280 | 0 | 280 | ~73 | Core architecture proven |
| 1: Aspect resolution | ~150 | 0 | 430 | ~98 | Include trees work |
| 2: Parametric | ~30 | 0 | 460 | ~113 | `den.lib.parametric` eliminated |
| 3: Policy dispatch | ~160 | 0 | 620 | ~80 | Enrichment convergence works |
| 4: Pipes | ~130 | 0 | 750 | ~40 | Cycle fixed-point proven |
| 5: Forwards | ~110 | 0 | 860 | ~34 | Class routing complete |
| 6: Custom entities | ~80 | 0 | 940 | ~79 | Full entity system |
| 7: Regressions | ~20 | 0 | 960 | ~69 | Deadbugs pass |
| 8: API changes | ~50 | 0 | 1,010 | remaining | Feature parity (753) |
| 9: Delete old engine | 0 | ~5,500 | 1,010 | — | Done |

Per-file test counts are approximate. Run `nix-unit --flake .#tests.<suite>` for exact counts before each phase.

Phases 4, 5, and 6 are independent — they can run in parallel worktrees after Phase 3 lands.

**Final state:** ~1,400 lines of resolution engine (1,010 new + ~390 preserved types/options/identity), down from ~7,300. All behavioral tests passing. 5x reduction achieved incrementally over 10 reviewable PRs.

## Phase Dependencies

```
Phase 0 (foundation)
  ├── Phase 1 (aspect resolution)
  │     ├── Phase 2 (parametric)
  │     │     ├── Phase 3 (policy dispatch)
  │     │     │     ├── Phase 4 (pipes)
  │     │     │     ├── Phase 5 (forwards)
  │     │     │     └── Phase 6 (custom entities)
  │     │     │           └── Phase 7 (regressions)
  │     │     │                 └── Phase 8 (API changes)
  │     │     │                       └── Phase 9 (delete old)
```

Phases 4, 5, and 6 are independent of each other — they all depend on Phase 3 but can be worked in parallel (separate worktrees).

## Risk Mitigation

### "What if a phase is harder than estimated?"

Each phase is additive. The old engine continues running all 753 tests regardless of new engine progress. If Phase 4 (pipes) takes 200 lines instead of 130, the total shifts from ~1,400 to ~1,470 — still a massive reduction.

### "What if the Cycle fixed-point doesn't work for pipes?"

Phase 4 is the architectural crux. If the Cycle approach fails for pipe.collect, we fall back to a lightweight post-resolution coordination pass (Approach C from the migration spec). This adds ~100 lines but is strictly simpler than the current 632-line assemblePipes. Phases 0-3 and 5-6 are unaffected.

### "What about the fx-* internal tests?"

These test handler internals that won't exist. They're replaced as each phase lands:

| Phase | Old `fx-*` tests replaced | New tests added |
|-------|--------------------------|-----------------|
| 0 | fx-compile-static, fx-aspect, fx-static-effects, fx-identity | ST tests, ctxD tests |
| 1 | fx-compile-conditional, fx-gate, fx-includeIf | expandAspect tests, dedup tests |
| 2 | fx-compile-parametric, fx-ctx-*, fx-parametric-meta, fx-regressions | ctxD arg resolution tests |
| 3 | fx-constraints, fx-dispatch-policies, fx-adapter-integration | converge tests, constraint tests |
| 4 | fx-scope-effects, fx-scope-widen, fx-iterate-effects, fx-bind-subsystem, fx-coverage | Cycle tests, pipe combinator tests |
| 5 | fx-compile-forward | adapter functor tests |
| 6 | fx-full-pipeline, fx-e2e | end-to-end stream tests |

### "What about diagnostics?"

The diag subsystem (~3,000 lines) reads old pipeline state. It's deferred — not part of this delivery plan. Diag continues working against the old engine until Phase 9. After deletion, diag needs a separate redesign effort using stream tracing (a `traceD` driver). The `fx-trace.nix` and `fx-diag-*` tests (~19 tests) are not ported — they're part of the diag redesign scope.

### Performance validation

Before Phase 0 begins, benchmark the current engine on the CI template (5+ hosts, 20+ aspects, 10+ policies). Re-benchmark after each phase lands. If any phase introduces >10% regression, investigate before proceeding. Nix thunk sharing should prevent redundant evaluation, but verify empirically.
