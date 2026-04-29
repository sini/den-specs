# Analysis: Provides Backwards-Compatibility Shims

## Verdict
implemented
Both components shipped on `feat/fx-pipeline` in commits `19738452` (provides-compat handler) and `2e40b024` (mutual-provider shim), matching the spec's stated range. Integration is wired correctly. One spec risk item (ownerIdentity exclusion rollback) was not resolved — it ships as a known gap.

## Delivery Target
feat/fx-pipeline

## Evidence

### 1. Pipeline handler: `provides-compat.nix`

- **File:** `/home/sini/Documents/repos/den/nix/lib/aspects/fx/handlers/provides-compat.nix`
- **Commit:** `19738452 feat: add provides-compat handler for main-era cross-provide shims`
- **Exports:** `emitCrossProvideShims` at line 65
- **schemaKinds filter:** line 14 — filters `builtins.attrNames (den.schema or {})` to only `.isEntity or false` entries (diverges from spec; see Drift)
- **applyProvide helper:** lines 19–28 — `__fn`, `__functor`, `lib.isFunction`, plain attrset; matches spec exactly
- **mkWildcardPolicy:** lines 31–45 — `{ host, user, ... }:` signature, `lib.warn` with migration message
- **mkNamedTargetPolicy:** lines 48–62 — `host.name == key || user.name == key` guard, `lib.optional`
- **fx.send "register-aspect-policy":** lines 89–94 — `name = "${aspectName}/compat:${key}"`, `ownerIdentity = nodeIdentity`
- **compatKeys exclusion of schemaKinds:** line 71 — cross-kind keys excluded from compat path

### 2. Module shim: `mutual-provider-shim.nix`

- **File:** `/home/sini/Documents/repos/den/modules/compat/mutual-provider-shim.nix`
- **Commit:** `2e40b024 feat: add mutual-provider compat shim`
- **Pattern:** `den.provides.mutual-provider = lib.warn "..." { name = "mutual-provider"; __functor = _: _: {...}; };` — lines 8–22
- Matches spec design verbatim

### 3. Integration in `resolveChildren`

- **File:** `/home/sini/Documents/repos/den/nix/lib/aspects/fx/aspect.nix` lines 886–900
- `emitCrossProvideShims` imported at line 9 from `den.lib.aspects.fx.handlers`
- Placed between `emitSelfProvide` and `emitAspectPolicies` exactly as spec prescribed
- Old `builtins.seq _ (emitSelfProvide ...)` trace block removed; handler owns warnings

### 4. Handler wiring

- **File:** `/home/sini/Documents/repos/den/nix/lib/aspects/fx/handlers/default.nix` line 12
- `// (import ./provides-compat.nix args)` — active (not commented out)

### 5. Module import via flakeModule

- **File:** `/home/sini/Documents/repos/den/nix/flakeModule.nix` lines 3–5
- `lib.filesystem.listFilesRecursive ../modules` — picks up `modules/compat/mutual-provider-shim.nix` automatically alongside `ctx-shim.nix` and `removed-stages.nix`
- No explicit import entry needed; spec's instruction to "check modules/default.nix" was moot — there is no `modules/default.nix`

### 6. `provides` in structuralKeysSet

- **File:** `/home/sini/Documents/repos/den/nix/lib/aspects/fx/aspect.nix` lines 12–17
- `provides` present in `structuralKeysSet` — spec requirement satisfied

### 7. Tests

- `/home/sini/Documents/repos/den/templates/ci/modules/features/mutual-provider.nix` — tests for the **new** policy syntax (`policies.to-users`, `policies.to-hosts`, named target), not for the compat shim
- `/home/sini/Documents/repos/den/templates/ci/modules/features/nested-aspects.nix` line 143 — self-provide backward compat test only
- No test file for `provides-compat` handler (old-syntax `provides.to-users`, `provides.to-hosts`, named cross-target, `__fn`/`__functor` value shapes, mutual-provider shim scenarios)

## Current Status
Still exists — both files are present and active on `feat/fx-pipeline`. Removal path deferred to Phase 2 (provides-removal spec).

## Supersession
Prerequisite for: `2026-04-27-provides-removal-post-unified-effects.md` (provides removal Phase 2).
Supersedes: none (fills a gap left by the deleted `mutual-provider.nix` battery).

## Gaps

1. **No compat-specific tests.** The spec listed 10 test cases for `provides-compat.nix` and 4 for the mutual-provider shim. None shipped. The `mutual-provider.nix` CI file tests the new routing syntax, not the old `provides.X` compat path.

2. **ownerIdentity exclusion rollback unresolved.** The spec flagged this as a verification item at lines 130 and 241. The field is stored (`tree.nix:304`) but never read back during policy dispatch (`dispatchPolicyIncludesHandler` at `tree.nix:362-385` filters only by functionArgs matching). The constraint registry (`register-constraint`) keys on the excluded aspect's identity, not on `ownerIdentity` of registered policies. If an aspect with `provides.X` compat policies is excluded, its synthesized policies continue firing.

## Drift

1. **schemaKinds filter stricter than spec.** Spec (`line 71`): `schemaKinds = builtins.attrNames (den.schema or {})`. Implementation (`provides-compat.nix:14`): filters to only entries where `den.schema.${n}.isEntity or false`. This means non-entity schema kinds (e.g., `conf`) are not in the exclusion list. In practice `conf` keys wouldn't appear as `provides.X` targets, and all user-facing schema kinds have `isEntity = true`, so this is a safer/more precise filter rather than a breaking divergence.

2. **No `builtins.seq _` removal comment.** The spec described replacing a `builtins.seq _` deprecation trace block (spec lines 148–159). The actual `resolveChildren` at `aspect.nix:886` has `fx.bind (emitSelfProvide aspect) (selfProvResults:` without any seq wrapping — the trace block was already gone before this commit. The `emitCrossProvideShims` call was simply inserted at line 888.
