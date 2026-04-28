# Analysis: Prior Art — Entity-Type Registries with Policies and Resolution

## Verdict

Accurate reference document. The four core terminology concepts (Entity, Schema, Class, Policy) are all implemented in the codebase with matching names. The `den.ctx` deprecation described in the doc is live — ctx-shim.nix exists with deprecation warnings. The doc is still directionally correct as a conceptual reference, but two specific code examples it contains have drifted from the current implementation.

## Delivery Target

Reference/comparison document only — no delivery target. Companion to `2026-04-21-ctx-as-classes-design.md`. Validates the three-concern separation that motivated the den redesign.

## Evidence

**Entity** — term used throughout. `nix/lib/resolve-entity.nix` injects `__entityKind = name` into context. Comments in `nix/lib/aspects/fx/handlers/transition.nix` and `nix/lib/policy-inspect.nix` use "Schema entity kinds" and "entity kind" consistently.

**Schema** — `den.schema.*` is the live option namespace. `modules/options.nix` defines the schema entry type. `modules/context/has-aspect.nix`, `modules/aspects/provides/wsl.nix`, `modules/aspects/provides/home-manager.nix`, `modules/aspects/provides/import-tree.nix` all use `den.schema.host.imports` / `den.schema.host.includes` directly. `modules/compat/ctx-shim.nix` forwards `den.ctx.*` to `den.schema.*`.

**Class** — term live on entity definitions. `nix/lib/entities/host.nix` declares `class = strOpt "os-configuration nix class for host"` defaulting to `"nixos"` or `"darwin"`. `nix/lib/entities/home.nix` declares `class = strOpt "home management nix class" "homeManager"`. Aspect class keys (`nixos`, `darwin`, `homeManager`) appear throughout CI tests and provides modules.

**Policy** — `den.policies` is the live option namespace (`nix/nixModule/policies.nix`). `modules/policies/core.nix` and `modules/policies/flake.nix` define core policies. All policies are now plain functions — the `from`/`to` field pattern in the doc has been superseded (see Drift).

**Aspect** — `den.aspects` is the live option namespace, confirmed by `modules/aspects/definition.nix`, `modules/outputs/osConfigurations.nix`, and extensive CI test coverage.

**`den.ctx` deprecation** — `modules/compat/ctx-shim.nix` has `options.den.ctx` marked `description = "DEPRECATED: use den.aspects instead."` and `den.policies` described as the replacement for `den.ctx.*.into`. The ctx shim forwards to `den.schema.*` with `lib.warn` on each entry.

**`config.resolved`** — the `.resolved` option on entity schema entries is live. `modules/options.nix` defines `options.resolved` on every schema kind, calling `den.lib.resolveEntity kind ctx`. The K8s "status" analogy holds.

## Current Status

All four terminology concepts match current codebase. The three-concern separation the doc argues for is fully realized:

1. What an entity IS → `den.schema.*` (schema entry type in `modules/options.nix`)
2. How entities RELATE → `den.policies` plain functions with `__entityKind` body guards
3. How entities RESOLVE → `den.aspects` + fx pipeline handlers in `nix/lib/aspects/fx/`

The `den.ctx` removal is complete at the source level (only a compatibility shim remains with deprecation warnings); the doc's description of it as "being removed" reflects a transitional state that has now landed.

## Supersession

No supersession. This is a reference/rationale document, not a design spec. The design decisions it validates (three-concern separation, `den.ctx` removal) have been implemented. The document can be retained as-is for historical rationale.

## Gaps

The doc mentions `den.schema.*` + submodule types as the entity schema mechanism — accurate. It does not mention `den.schema.conf` (the shared base module, `modules/context/has-aspect.nix` injects it), but that is an implementation detail below this doc's scope.

The doc describes `den.policies` as "declarative relationship macros" — the current implementation uses plain functions (not attrset declarations), which is closer to behavior than metadata. This is a real conceptual gap: the doc frames policies as static metadata (Rails `has_many` analogy), but the implementation uses functions that return typed effect constructors (`resolve`, `include`, `exclude` from `nix/lib/policy-effects.nix`).

## Drift

Two concrete code examples in the doc no longer match the current implementation:

**1. Policy `from`/`to` field pattern (line 65):**
```nix
# doc claims this is the pattern:
den.policies.host-to-users = { from = "host"; to = "user"; ... };
```
Current implementation (as of refactor removing `from`/`to` fields) uses plain functions with `__entityKind` body guards:
```nix
den.policies.host-to-users = { __entityKind ? null, host, ... }:
  if __entityKind != "host" then [ ]
  else map (user: resolve.shared { inherit user; }) (lib.attrValues host.users);
```
The `from`/`to` field attrset format appears only in `policyInspect` output (as derived metadata), not in policy definitions. The `doc-examples.nix` test confirms `from = "host"` and `to = "user"` in inspect output, but the policy itself is a function.

**2. Haskell typeclass mapping for policies (line 129):**
Doc maps "typeclass constraint" to policy (ordering/dependency). In the current design, policies are more like runtime transition rules (effect-returning functions dispatched by `__entityKind`) than static typeclass constraints. The constraint analogy still holds loosely but the implementation is more dynamic than a constraint system.

These drifts do not invalidate the document's conceptual argument — they are implementation-level details that evolved after the doc was written.
