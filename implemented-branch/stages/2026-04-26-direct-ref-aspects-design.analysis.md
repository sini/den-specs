# Analysis: Direct-Ref Policy Aspects + Entity Simplification

## Verdict
Implemented — all core structural changes are present on feat/fx-pipeline. No dedicated commit label, delivered as part of the pipeline simplification work stream.

## Delivery Target
feat/fx-pipeline branch. No single labelled commit; changes landed across the pipeline refactor series (f5282ac8 and predecessors).

## Evidence

**entityIncludes deleted:**
- `nix/nixModule/entities.nix` absent — only `aspects.nix`, `default.nix`, `lib.nix`, `policies.nix` in nixModule/
- `modules/context/host.nix` absent — only `perHost-perUser.nix`, `has-aspect.nix`, `flake-schema.nix` in modules/context/
- `modules/context/user.nix` absent — same
- `modules/outputs/osConfigurations.nix` writes `den.aspects.flake-os`, no entityIncludes
- `modules/outputs/hmConfigurations.nix` writes `den.aspects.flake-hm`, no entityIncludes
- `modules/outputs/flakeSystemOutputs.nix` registers forwarders via `den.aspects`, no entityIncludes probe
- `modules/aspects/defaults.nix` not found (superseded)
- Grep for `entityIncludes` across entire repo: only hit is a deadbugs test fixture (`dup-functors-issue-216.nix`)

**resolveEntityHandler always succeeds:**
- pipeline.nix comment: "Always returns a valid entity — existence gating removed."
- Handler body calls `den.lib.resolveEntity kind currentCtx` unconditionally, always returns `entity // { includes = strippedIncludes; }`
- No null check, no tombstone path, no existence guard

**Tombstone/crossProvider dead code removed:**
- Grep for `tombstone`, `crossProvider`, `effectiveTarget == null`, `emitCrossProvider` in handlers/: no matches

**rootIncludes removed:**
- `structuralKeysSet` in `aspect.nix` contains no `rootIncludes` entry
- Grep for `rootIncludes` across nix/ and modules/: no matches

**String indirection removed from policy.aspects:**
- `nix/lib/policy-types.nix` contains only `policyFnArgs` extraction — no `aspects` option declaration at all (aspects are carried inline, not typed via NixOS options)
- `transition.nix` line 302: `policyAspects = (transition.routing or {}).aspects or []` — direct value injection, no `map (name: den.aspects.${name})` lookup
- `transition.nix` line 348-349: `aspects = map (a: pathKey (aspectPath a)) policyAspects; aspectValues = policyAspects;` — path-keyed identity with direct ref values

**ctx-seen stores direct refs:**
- `transition.nix` sends both `aspects` (path ids) and `aspectValues` (direct refs) on ctx-seen effect — matches spec Section 8

**policy.aspects type:**
- No `listOf str` type declaration found in nix/lib/ — the spec's `listOf providerType` option was not implemented as a NixOS option; instead aspects are untyped `routing.aspects` lists carrying direct refs inline
- `providerType` exists in `types.nix` but is not used to type-constrain the aspects field on policies

**resolveEntity — self-provide from schema:**
- `resolve-entity.nix` derives `selfProvide` from `aspectKindSet` (schema kinds with `isEntity = true`) using a parametric wrapper `{ __fn = c: c.${name}.aspect; }` — equivalent to spec's ctx-based self-provide
- `default` special case handled: `if name == "default" && den ? default then [ den.default ]`
- `hostFramework` injects `os-host-fwd` directly (host-level framework aspect, no policy needed)

**ctx-shim.nix:**
- Forwards `den.ctx.*` to `den.schema.${name}.includes` (not `den.aspects` as spec Section 10 proposed)
- Functionally equivalent: schema includes participate in entity resolution via the same path
- `into` field deprecated with warning; no `"ctx:"` prefix naming

**flakeSystemOutputs.nix:**
- No runtime probe against entityIncludes — forwarders registered as named aspects, consumed via `include den.aspects."flake-${output}"` in flake policies
- Not "unconditional entity resolution" per spec Section 9 — instead aspect-based approach, equivalent outcome

**has-aspect.nix error message:**
- Error string: `"(no matching den.schema.<kind> defined)."` — already references `den.schema`, matches spec Section 12

**makeHomeEnv:**
- `home-env.nix` uses `den.lib.policy.include` with direct aspect refs (`den.aspects.os-user-fwd`, `den.aspects.os-user-class-fwd`) — matches spec Section 11 intent
- No `policy.aspects` field on the battery policy struct; includes are emitted as `include` effects directly from the policy function body

## Current Status
All deletions confirmed. Core structural outcomes delivered. The implementation diverges in approach on several sections (see Drift) while preserving equivalent semantics.

## Supersession
Supersedes the entityIncludes/rootIncludes/string-indirection design. The ctx-shim spec (Section 10) is partially superseded — ctx-shim forwards to `den.schema` includes, not `den.aspects` as specified, but the shim's deprecation path is live.

## Gaps
- `policy.aspects` is not a typed NixOS option (`listOf providerType`); typing was not added. Spec Section 1 is unimplemented as a formal option. Aspects on policies are untyped freeform lists on `routing.aspects`.
- Spec Section 9 (unconditional entity resolution in flakeSystemOutputs) was not implemented as written; instead forwarders use the aspect-include path via `include den.aspects."flake-${output}"` in flake policies.

## Drift
- **ctx-shim target:** Spec Section 10 says forward to `den.aspects` with `"ctx:"` prefix. Implementation forwards to `den.schema.${name}.includes`. Same outcome (schema includes enter entity resolution), different mechanism and no `"ctx:"` prefix namespace.
- **makeHomeEnv policy.aspects field:** Spec Section 11 adds `aspects = [...]` to the battery policy struct. Implementation uses `den.lib.policy.include` calls inside the policy function body. No `aspects` field on the policy record.
- **flake policies aspects field:** `modules/policies/flake.nix` uses `include den.aspects."flake-${output}"` as inline include effects, not `aspects = [...]` on a policy record. The spec envisioned `policy.aspects` as the delivery mechanism; the implementation uses inline policy effects instead.
- **hostFramework hardcoded in resolveEntity:** `os-host-fwd` is injected directly in `resolve-entity.nix` rather than via `policy.aspects` on a host policy. Spec Section 7 moved host-level os-class forward to `policy.aspects` on core policies; implementation bakes it into resolveEntity.
