# Analysis: Handler File Cleanup — aspect.nix decomposition + transition.nix dedup

## Verdict
partially-implemented
The aspect.nix decomposition shipped almost completely via a different structural approach than specified (no `fx/wrap.nix` or `handlers/dead-letter.nix` — instead `class-module.nix`, `wrap-classes.nix`, `include-emit.nix`). The transition.nix dedup proposals are moot: `transition.nix` was deleted entirely in e561d530, eliminating the problem rather than refactoring it.

## Delivery Target
feat/fx-pipeline

## Evidence

### aspect.nix decomposition

**wrapClassModule extraction** — SHIPPED, different target file.
Spec proposed `nix/lib/aspects/fx/wrap.nix`. Actual: extracted to `nix/lib/aspects/fx/class-module.nix` in commit d3e09dad ("refactor: extract class-module.nix from aspect.nix"), removing ~231 lines from aspect.nix. `wrapClassModule` is at class-module.nix:240. `resolveCollisionPolicy` at class-module.nix:42 and `wrapDeferredImports` at class-module.nix:69 are co-located as spec required. aspect.nix now re-exports via `inherit (import ./class-module.nix { ... }) wrapClassModule` at line 10.

**DLQ cluster extraction** — NOT SHIPPED as specified.
Spec proposed `handlers/dead-letter.nix` with `drainDeadLettersHandler`, `emitClassFromDLQ`, `emitTraitFromDLQ`. None of these symbols exist in the codebase. The `deadLetterQueue` state field is gone; the drain mechanism is now a generic deferred-include drain in `handlers/drain.nix` (added 7b39278c). No `dead-letter.nix` was ever created (git log confirms no history for that path).

**Include normalization extraction** — NOT SHIPPED as specified.
Spec proposed moving `mkPositionalInclude` and `mkNamedInclude` to `handlers/include.nix`. Neither symbol exists anywhere in the codebase. `handlers/include.nix` (line 1–42) handles `emit-include` and `include-unseen` effects — no normalization helpers. Include logic was restructured via the include-emit.nix extraction (commit 270bc9e8), then further decomposed (4e52a117), but under different function names.

**dispatchPolicyIncludes deletion** — SHIPPED (superseded by full transition.nix removal).
`dispatchPolicyIncludes` does not exist in the codebase. The entire handler was deleted when transition.nix was removed in e561d530.

**emitSelfProvide + emitCrossProvideShims deletion** — SHIPPED.
Neither symbol exists in the codebase. `provides-compat.nix` was deleted in commit aa32f77e ("refactor: unify provides into emitAspectPolicies, delete provides-compat"). No provides-era compat shim files remain in `handlers/`.

**wrapCollectedClasses extraction** — SHIPPED (bonus, not in spec).
Extracted from pipeline.nix to `wrap-classes.nix` in 9387fa6b. Current file is nix/lib/aspects/fx/wrap-classes.nix (171 lines at extraction, still present).

**aspect.nix size** — Spec targeted ~595 lines from 1119. Current: 123 lines (nix/lib/aspects/fx/aspect.nix). Far exceeded the target via deeper decomposition across multiple files.

### transition.nix dedup

**transition.nix** — DELETED ENTIRELY, not refactored.
git log shows transition.nix existed in handlers/ and was removed in e561d530 ("refactor: delete transition handler and scope stack machinery, Remove into-transition handler (~900 lines), dispatchPolicyIncludesHandler, scopeStack/scopeChildren/scopeProvenance state fields"). All four transition.nix proposals in the spec are moot.

**classifyPolicyEffects extraction** — SHIPPED AS DIFFERENT SHAPE.
Spec proposed a single `classifyPolicyEffects` function deduplicating global vs aspect dispatch. Actual: `policy/classify.nix` (nix/lib/aspects/fx/policy/classify.nix) implements `classifyPolicyResult` (line 48) and `extractTaggedEffects` (line 125) — same intent, different names and split further. Created in commit 5d3c3d79.

**emitCrossProvider deletion** — MOOT (transition.nix deleted).
Symbol never appears in current codebase.

**processIncludeOnly deletion** — MOOT (transition.nix deleted).
Symbol never appears in current codebase.

**resolveFanOut simplification** — NOT APPLICABLE.
`resolveFanOut` does not exist in current codebase. The sub-pipeline / fanout architecture was replaced by scope-partitioned state (push-scope/restore-scope handlers).

## Current Status
The functional goals of the spec shipped, but via a more thorough cleanup than proposed. aspect.nix is 123 lines (not ~595), transition.nix is gone entirely (not refactored), and the handler directory grew from a monolith to 37 focused files. The spec's proposed file names (`wrap.nix`, `dead-letter.nix`) were not used.

## Supersession
Partially superseded by:
- `2026-05-01-pipeline-architecture-cleanup` — drove the transition.nix deletion (e561d530) and push-scope/restore-scope replacement
- `2026-05-02-pipeline-simplification` — further decomposition of handlers/ directory
- `2026-05-04-provides-cleanup` — drove emitSelfProvide/emitCrossProvideShims removal (aa32f77e)

## Gaps
- `handlers/dead-letter.nix` never created; DLQ concept replaced by generic drain mechanism
- `handlers/wrap.nix` (spec name) never created; shipped as `class-module.nix` instead
- `mkPositionalInclude` / `mkNamedInclude` normalization helpers not present under any name
- `resolveFanOut` simplification not applicable (architecture replaced)

## Drift
- Spec assumed transition.nix would be refactored; it was deleted entirely (stronger outcome)
- Spec proposed specific file names (`wrap.nix`, `dead-letter.nix`); actual extractions used different names (`class-module.nix`, `wrap-classes.nix`, `include-emit.nix`)
- Spec's DLQ model (class/trait re-emission from a queue) was superseded by deferred-include drain without a separate dead-letter subsystem
- aspect.nix reduction exceeded spec target: 1119 → 123 lines (spec expected ~595)
- `classifyPolicyEffects` shipped as two functions (`classifyPolicyResult` + `extractTaggedEffects`) in `policy/classify.nix`, not one function in transition.nix
