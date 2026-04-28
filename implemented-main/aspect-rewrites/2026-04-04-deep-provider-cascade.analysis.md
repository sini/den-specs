# Analysis: Deep Provider Cascade, Substitute, and Namespace Qualification

## Verdict
partially-implemented

## Delivery Target
(A) main — the core features (providerPrefix threading, meta.provider list, namespace qualification) landed on main. The fx-pipeline branch reimplemented the constraint/cascade machinery with a different architecture.

## Evidence

**Commits (main):**
- `6ca251e7` — "feat: structural provider provenance via meta.provider (#399)" — adds `meta.provider` as list path, providerPrefix threading through provides submodule, `mkAspectsType` factory, namespace providerPrefix = [name]
- `d267c458` — "fix: excludeAspect cascades to provider sub-aspects (#411)" — prefix-based cascade for excludeAspect, landed in `nix/lib/aspects/adapters.nix` and `templates/ci/modules/features/aspect-path.nix`
- `afbe47f1` — "fix: provider sub-aspect functions receive parametric context (#419)" — related provider sub-aspect fix

**Files (current state on feat/fx-pipeline):**
- `nix/lib/aspects/types.nix:158,281,370,410` — `typeCfg.providerPrefix or []` threading; `__provider = (typeCfg.providerPrefix or []) ++ [keyName]` in aspectContentType; `options.provider` default = `providerPrefix`; provides submodule extends providerPrefix with `config.name`
- `nix/lib/aspects/fx/identity.nix:10–12` — `aspectPath = a: (a.meta.provider or []) ++ [a.name] ++ ...` — list-based path derivation consuming `meta.provider`
- `nix/lib/namespace-types.nix:28` — `mkAspectsType { providerPrefix = [ name ]; }` for namespace qualification
- `nix/lib/aspects/fx/handlers/tree.nix:83–94` — prefix-based cascade in check-constraint: splits `nodeIdentity` on "/", generates all prefix keys, collects entries from each — this is the fx-pipeline reimplementation of `isPrefix` cascade

## Current Status

Core features still exist but the surface API has changed significantly:

| Spec Feature | Current State |
|---|---|
| `__provider` as `listOf str` | Exists as `meta.provider` on aspects (not `__provider` option on config output) |
| `providerPrefix` threading | Exists in `types.nix` and `aspectSubmodule.provides` |
| `toAspectPath` function | **Not present** — replaced by `identity.aspectPath` which reads `meta.provider ++ [name]` |
| `isPrefix` + prefix cascade | Reimplemented in `tree.nix` check-constraint via string split on "/" |
| Provider-aware `substitute` | Present in fx pipeline via `substituteChild` in `include.nix` + registry lookup in `tree.nix` |
| `__placedBy` annotation | **Not present** — replaced by `meta.replacedBy` (string name, not path) in trace |
| `wrapProvider` function | **Not present** — functionality absorbed into `providerType` merge and `aspectContentType` |
| `toAspectName` (compat) | **Not present** — ctx-apply.nix deleted; entity-level excludes now use fx pipeline |
| Namespace qualification | Exists — `namespace-types.nix` passes `providerPrefix = [name]` |
| Trace `provider` field as list | Exists in `trace.nix` as `provider = param.meta.provider or []` |
| Mermaid qualified labels/dedup | Exists — `diag/graph.nix` uses `identity.pathKey (identity.aspectPath ...)` for canonical keys |

## Supersession

This spec extended and partially superseded `2026-04-04-provider-aware-excludes.md`, which introduced single-level prefix cascade for `excludeAspect`. This spec added `listOf str` provider paths, deep/nested cascade, provider-aware substitute, and namespace qualification on top of that foundation.

This spec was then partially superseded by the fx-pipeline architecture (feat/fx-pipeline). The spec's `transforms.nix`-based approach (pure functions applied to `{ provided, ... }`) was replaced by algebraic effects with a constraint registry. Key replacements:

- `aspects/transforms.nix` → `aspects/fx/constraints.nix` (exclude/substitute as effect records) + `aspects/fx/handlers/tree.nix` (constraint registry with prefix cascade)
- `aspects/resolve.nix` trace → `aspects/fx/trace.nix` (structuredTraceHandler)
- `ctx-apply.nix` string-based entity excludes → fx transition handler with path-based constraint dispatch

The spec's "known asymmetry" about ctxApply string matching was resolved by deleting ctx-apply.nix entirely and routing all excludes through the fx pipeline.

## Gaps

1. **`toAspectPath` as public API** — spec exports it from `aspects/default.nix` for user-facing use. Current code has `identity.aspectPath` (internal) but it is not the same signature: it reads `meta.provider` from an already-resolved aspect, not a bare ref/string/list. No user-facing `toAspectPath` exists.

2. **`__placedBy` annotation** — spec threads `__placedBy` through `withIdentity` and `carryAttrs` to track which substitute placed a replacement. Current code uses `meta.replacedBy` (a name string, not a path), and there is no `__placedBy` field. Provenance tracking exists but is weaker.

3. **String accepts for internal compat** — spec's `toAspectPath` accepted strings for ctxApply compat. Current `fx.exclude`/`fx.substitute` require attrsets (will throw on strings). The compat path is gone because ctxApply is gone.

4. **`substitute` auto-match and prune-missing** — spec's provider-aware substitute cascades into `replacement.provides.${provided.name}` for sub-aspect replacement. Current `substituteChild` substitutes only the exact matched node; there is no automatic cascade into the replacement's provides sub-tree.

## Drift

1. **Architecture** — spec describes a functional transform pipeline (`compose`, `normalizeResult`, transform functions receiving `{ provided, ... }`). Implementation uses algebraic effects with a stateful constraint registry. The semantics are equivalent but the mechanism is entirely different.

2. **`meta.provider` vs `__provider` option** — spec defines `__provider` as a NixOS option on the aspect submodule (`options.__provider`). Current code tracks provenance via `meta.provider` (set at merge time in `aspectContentType` and the single-def fast path). The `options.provider` in `aspectSubmodule` still exists but is separate from `__provider` in the content wrapper.

3. **Cascade scope** — spec's isPrefix cascade applies at the aspect resolution level (transform layer). Current implementation applies cascade in the constraint registry check (`check-constraint` in `tree.nix`) by splitting the pathKey string on "/" and looking up all ancestor prefixes. Functionally equivalent but implemented one level deeper in the effect handler.

4. **`compose` / `normalizeResult`** — spec notes breaking changes removing these. They are fully absent; no trace.

## Cross-Reference Notes

- **Supersession chain**: `2026-04-04-provider-aware-excludes.md` (single-level prefix cascade, verdict: implemented) → this spec (deep/nested cascade, `listOf str` provider paths, provider-aware substitute; verdict: partially-implemented) → fx-pipeline architecture (constraint registry in `tree.nix`/`transition.nix`). The `provider-aware-excludes` analysis correctly identifies this spec as its partial supersessor.
- **Consistent with `aspect-transforms-spec` analysis**: both note `transforms.nix` never existed as a real file; both confirm `adapters.nix` deleted in `3a9cea99`.
- **Consistent with `provider-aware-excludes` analysis**: both agree `__provider` was never a declared NixOS option; both agree `meta.provider` is the implementation mechanism; both agree cascade semantics are preserved in the fx pipeline via string prefix decomposition in `tree.nix`.
