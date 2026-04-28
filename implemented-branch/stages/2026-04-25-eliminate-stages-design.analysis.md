# Analysis: Eliminate `den.stages` — Three-Layer Aspect Model

## Verdict
FULLY IMPLEMENTED. Every structural commitment in the spec is present on feat/fx-pipeline. A few architectural decisions diverged from spec (noted in Drift), but the outcome is equivalent or superior.

## Delivery Target
feat/fx-pipeline branch. Relevant commits (oldest to newest):
- `e9ea41b0` — policy effect constructors (resolve/include/exclude)
- `e325fc2c` — aspect-included policies / policyFns plumbing
- `db189e17` — rename stage → entityKind, derive entity kind in trace
- `d8b538f8` — `den.stages` removal shim (throw on access)
- `399a3ddd` — register flake + default routing kinds in `den.schema`
- `30ba36e` — delete `policy-dispatch.nix`, simplify `den.policies`
- `29156fe0` — remove policy activation mechanism
- `4d06b4be` — remove `from` field, inject `__entityKind`
- `269ee2c8` — remove `to` field, use `resolve.to` effect

## Evidence

**Deleted files confirmed absent:**
- `nix/nixModule/stages.nix` — gone
- `nix/lib/stage-types.nix` — gone
- `nix/lib/resolve-stage.nix` — gone (only exists in `.worktrees/docs-fixes`, a different branch)

**Shim in place:**
- `modules/removed-stages.nix` — `den.stages` option with `visible = false`, `type = raw`, `apply = _ -> throw "den.stages has been removed..."` with migration instructions

**New `resolve-entity.nix` exists:**
- `nix/lib/resolve-entity.nix` — implements `resolveEntity name ctx`, produces `{ name, meta, includes, __entityKind, __scopeHandlers }`
- Derives `schemaIncludes` from `den.schema.${name}.includes`
- Self-provide via `__fn`/`__args` wrapper for entity-kind aspects
- Exported from `nix/lib/default.nix` as `resolveEntity`

**Three-layer model active:**
1. `den.aspects` registry — used in `modules/outputs/`, `nix/lib/entities/_types.nix`, `nix/lib/den-brackets.nix`
2. Policy aspects — transition handler injects `policyAspects` from `transition.routing.aspects` into `effectiveTarget.includes` (`transition.nix:301-304`)
3. Entity schema includes — `den.schema.<kind>.includes` used in `resolve-entity.nix:45` and across `modules/aspects/provides/`

**`resolveTargetHandler` replaced:**
- `pipeline.nix` has `resolveEntityHandler` at line 100, handles `"resolve-entity"` effect
- `transition.nix` sends `fx.send "resolve-entity" { kind = ... }` at line 295

**Per-aspect-set merge key implemented:**
- `transition.nix:564-584` — `mergeByPath` uses `"${path}|${aspectsKey}"` as `mergeKey`, sorting aspects canonically

**ctx-seen accumulation implemented:**
- `nix/lib/aspects/fx/handlers/ctx.nix` — `ctxSeenHandler` returns `{ isFirst, newAspectValues }`, tracks accumulated aspect IDs per key
- `transition.nix:352-372` — on subsequent visits, emits supplemental `emit-include` effects for `newAspectValues`

**Trait args in `resolveArgsSatisfied`:**
- `synthesize-policies.nix:20` — `requiredArgSatisfied = k: ctx ? ${k} || traitNames ? ${k}`

**Traits merged into policy resolve context:**
- `transition.nix:402-403` — `resolveCtx = traits // currentCtx // { __entityKind = ... }`

**`makeHomeEnv` rewritten as self-contained battery:**
- `home-env.nix:60-125` — single `policyFn` with `policy.resolve` + `policy.include` effects; returns `{ battery, hostConf }`
- `host-to-${ctxName}-users` policy via `battery.policies` key
- Batteries deployed via `den.schema.host.includes = [ result.battery ]` in hm/maid/hjem provides

**Flake entity kinds registered:**
- `modules/context/flake-schema.nix` — registers `flake`, `flake-system`, `flake-os`, `flake-hm`, `flake-packages`, `flake-apps`, `flake-checks`, `flake-devShells`, `flake-legacyPackages`, `default` in `den.schema` (all `isEntity = false` — routing kinds)
- `modules/policies/flake.nix` — `flake-to-flake-system` + `flake-system-to-flake-{packages,apps,checks,devShells,legacyPackages,os,hm}` all as plain functions using `__entityKind` guards and `resolve.to` / `policy.include` effects

**Compat shim updated:**
- `modules/compat/ctx-shim.nix` — forwards `den.ctx.*` to `den.schema.${name}.includes` with deprecation warning; `den.ctx.*.into` note still present in option description

**Diagnostics updated:**
- `diag/context.nix` — calls `den.lib.resolveEntity` for host, user, home
- `diag/fleet.nix` — calls `den.lib.resolveEntity "host"`

**`entityKeyFor` removed:**
- `synthesize-policies.nix` contains only `resolveArgsSatisfied` — no `entityKeyFor`, no suffix matching, no `ctxSatisfies` per old pattern

**`policy-dispatch.nix` deleted entirely:**
- Commit `30ba36e` — direct policy iteration now happens inline in `transition.nix`

## Current Status
Fully shipped. CI branch is `feat/fx-pipeline`. All core mechanisms operational. Template tests reference `den.schema.*` instead of `den.stages.*`.

## Supersession
Supersedes:
- `nix/lib/stage-types.nix` (deleted)
- `nix/lib/resolve-stage.nix` (deleted)
- `nix/nixModule/stages.nix` (deleted, shim in `modules/removed-stages.nix`)
- Old `resolveTargetHandler` + `resolve-target` effect (replaced by `resolveEntityHandler` + `resolve-entity`)
- `policy-dispatch.nix` (deleted entirely — dispatch collapsed into transition handler)
- `makeHomeEnv` two-stage-two-policy shape (replaced by battery pattern)
- `userEnvAspect` function (eliminated — subsumed by policy aspect inclusion)

## Gaps
None blocking. All spec-required deletions, new files, and modifications are present.

Minor incomplete items (non-blocking):
- `policy-types.nix` no longer defines a policy submodule with an `aspects` field — policies are plain functions; `aspects` travels in `routing` metadata assembled dynamically in `transition.nix`. The spec described `policy.aspects` as a submodule field; the implementation encodes it in routing derived from policy effects (`resolve.to` + `include` effects carry the aspect list). Functionally equivalent — see Drift.
- `modules/removed-stages.nix` throw message says "use policy.aspects with direct aspect references" — slightly stale wording since there is no `policy.aspects` field in the final design, but the concept is correct.

## Drift
Three significant architectural divergences from spec, all deliberate improvements:

**1. No `policy.aspects` submodule field.** The spec defined `aspects = lib.mkOption { type = listOf str; }` on the policy submodule. The implementation went further: eliminated the policy submodule entirely. Policies are plain functions returning `policy.include`/`policy.resolve`/`policy.exclude` effects. Aspect inclusion uses `policy.include` effects — raw aspect values, not registered names. The `routing.aspects` list in `transition.nix` is assembled from `includeEffects` per policy invocation. This is more flexible: aspects don't need to be pre-registered by name; any aspect value works.

**2. `from`/`to` replaced by `__entityKind` guard + `resolve.to` effect.** The spec kept `from`/`to` as explicit policy fields directing entity kind routing. The implementation removed both: `from` is replaced by an `__entityKind` destructuring guard in the policy function body; `to` is replaced by `resolve.to "kind" ctx` effect. This eliminates the policy type entirely and makes routing purely dynamic.

**3. `makeHomeEnv` uses aspect-included policyFns, not top-level `den.policies`.** The spec's new `makeHomeEnv` produces `{ aspects = {...}; policies = { "host-to-${ctxName}-users" = ...; } }` registered at top level. The implementation uses a `battery` aspect (submodule with `policies` key) included in `den.schema.host.includes`. The policy fires during host tree-walk via aspect-included dispatch rather than global `den.policies`. Equivalent routing, no global registration needed.
