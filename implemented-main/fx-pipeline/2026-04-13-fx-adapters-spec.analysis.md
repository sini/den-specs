# Analysis: Effects-Based Adapters

## Verdict
superseded

## Delivery Target
(A) main (partial, via den-fx refactor commit 68c26555)
(B) feat/fx-pipeline (further evolution, not full spec)

## Evidence

**Commits implementing spec concepts:**
- `68c26555` — `refactor: den-fx (#462)` — primary delivery; replaced legacy adapter pipeline with effects pipeline; adapters.nix marked legacy-only
- `eb74bdaf` — `adapterOwner tracking in filterIncludes + collectSelfPath helper` — pre-fx adapterOwner spec concept (landed before fx)
- `3a9cea99` — `effectful pipeline bootstrap` — extended fx.handle lifecycle
- `961463b7` — `fix: edge cases — fan-out dedup, cross-providers, perCtx, constraints`

**Key files on feat/fx-pipeline (branch):**
- `nix/lib/aspects/fx/identity.nix` — `aspectPath`, `pathKey`, `toPathSet`, `tombstone`, `collectPathsHandler` all shipped
- `nix/lib/aspects/fx/constraints.nix` — `exclude`, `substitute`, `filterBy` (constraint API replacing adapter operations)
- `nix/lib/aspects/fx/includes.nix` — `includeIf` shipped
- `nix/lib/aspects/fx/handlers/include.nix` — `emit-include` handler, tombstoning, substitution, conditional resolution via `resolveConditional`
- `nix/lib/aspects/fx/handlers/tree.nix` — `constraintRegistryHandler`, `classCollectorHandler`
- `nix/lib/aspects/fx/aspect.nix` — `resolveChildren` emits `resolve-complete` (line 783); `aspectToEffect` compiles tree
- `nix/lib/aspects/fx/pipeline.nix` — `defaultHandlers`, `mkPipeline`, `fxResolve`
- `nix/lib/aspects/fx/trace.nix` — structured trace handler collecting flat entries

**adapters.nix status:**
- On `main`: exists but marked "Legacy pipeline only — GOF adapters for resolve.withAdapter. The fx pipeline uses meta.handleWith + constraint handlers instead. Remove when the legacy pipeline is removed."
- On `feat/fx-pipeline`: **deleted** (fatal: path does not exist)

## Current Status

Partially replaced. The spec's goal (adapter support for include resolution via effects) was implemented but the architecture diverged significantly from the spec design. Key delivered pieces:

- `resolve-complete` effect: shipped exactly as specced — emitted in `resolveChildren` (aspect.nix:783) and in tombstone paths in include.nix
- `resolve-include` renamed to `emit-include` effect: the per-child dispatch effect exists but under a different name
- `tombstone`: shipped verbatim from spec in identity.nix
- `aspectPath` / `pathKey` / `toPathSet`: shipped in identity.nix (with one addition: `__ctxId` suffix for fan-out dedup)
- `collectPathsHandler`: shipped in identity.nix; state uses thunk-wrapped `pathSet` field (not `paths` list)
- `includeIf`: shipped in includes.nix with `meta.guard` shape; guard evaluates via `get-path-set` effect (live path set, not pre-computed raw set)
- `substituteAspect` / `excludeAspect`: shipped as `constraints.exclude` / `constraints.substitute` in constraints.nix (different API)
- `structuredTrace`: shipped in trace.nix with matching flat-entry shape
- `moduleHandler` (`classCollectorHandler`): shipped in tree.nix

## Supersession

This spec was superseded by the constraint-registry design. The adapter system proposed here used `meta.adapter` attrsets with `fx.rotate`-scoped handler stacking. The implementation replaced this with:

- `meta.handleWith` (list/single constraint records) — declarative, type-safe constraint declarations
- `meta.excludes` sugar — shorthand for exclude lists
- `constraintRegistryHandler` — handles `register-constraint` and `check-constraint` effects
- Scoping via `ownerChain` ancestry check (not `fx.rotate` subtree scoping)

The `fx.rotate` primitive exists in nix-effects and is tested in `fx-handlers.nix` (test-rotate-unknown-to-outer), but is **not used** in the production pipeline. The constraint registry approach superseded the `fx.rotate`-per-aspect design.

No dedicated spec file for the constraint-registry design was found; it appears to have emerged during den-fx implementation.

## Gaps

**Proposed features that did not ship:**
- `meta.adapter` as user-facing API — replaced by `meta.handleWith`; `meta.adapter` is never set by the production code (only referenced in legacy/docs contexts)
- `fx.rotate` scoped handler stacking for per-aspect adapters — not used in production pipeline
- `probeTransform` elimination (explicitly mentioned as dropped) — confirmed dropped, no probing in fx pipeline
- `filter`, `map`, `mapAspect`, `mapIncludes` adapter combinators — not present anywhere; constraint attrsets compose via `//` as predicted
- `mkAdapter` / `adapterOwner` injection wrapper — not shipped; `meta.constraintOwner` used instead for diagnostic attribution
- Context-as-root-adapter pattern — not directly applicable; context is threaded as `constantHandler` values, not as a root `meta.adapter`
- `resolve-include` effect name — effect named `emit-include` in implementation

**Test plan partially delivered:**
- `templates/ci/modules/features/fx-constraints.nix` covers exclude, substitute, filterBy, includeIf, transitive exclusion
- `templates/ci/modules/features/fx-effectful-resolve.nix` covers resolve-complete, collectPaths
- No `fx-adapters.nix` test file (spec proposed this exact name); tests split across constraint and effectful-resolve suites

## Drift

**Architecture divergence:**

1. **`resolve-include` → `emit-include`**: Spec defined a `resolve-include` effect that adapters handle. Implementation uses `emit-include`. The `resolve-include` / `resolve-complete` pair from the spec are both present but `resolve-include` was renamed; semantics remain close.

2. **`fx.rotate` per-adapter → constraint registry**: Spec: each `meta.adapter` installs a `fx.rotate` layer scoping that subtree. Implementation: constraints registered into a global registry keyed by aspect identity path; scoping via `ownerChain` ancestry rather than handler stack depth. This is simpler — no need to rotate handler stacks per-aspect — but loses dynamic handler composition.

3. **`meta.adapter` API → `meta.handleWith` / `meta.excludes`**: Spec: user writes `meta.adapter = { "resolve-include" = ...; }`. Implementation: user writes `meta.handleWith = constraints.exclude ref` (or list). Constraint objects are pure records, not handler functions.

4. **`substituteAspect` list resume → `getReplacement` thunk**: Spec: substitute handler resumes `resolve-include` with `[ tombstone replacement ]`. Implementation: constraint record carries `getReplacement = _: replacement` thunk; `substituteChild` in include.nix calls it and assembles the list. Same observable behavior, different internal path.

5. **`includeIf` guard sees live path set, not raw pre-filter set**: Spec stated guard should run against the raw (pre-filter) subtree via `collectPathsInner` semantics. Implementation uses `get-path-set` effect returning the path set accumulated so far during sequential resolution. Guard can only see aspects resolved before it in the tree. This is documented in includes.nix: "guards can only see aspects resolved BEFORE them in the tree."

6. **`collectPathsHandler` state shape**: Spec: `{ paths = []; }` accumulating list. Implementation: `{ pathSet = _: {}; }` thunk-wrapped attrset for O(1) lookup (avoids re-materializing on deepSeq). Also stores `baseKey` (without ctxId) alongside full key.

7. **`defaultResolve` shape**: Spec proposed `resolveDeepEffectful` + `moduleHandler` as the pipeline entry. Implementation uses `mkPipeline` / `fxResolve` with `defaultHandlers` composing all handlers at root scope — no separate `resolveDeepEffectful` function; the tree walk is implicit in `aspectToEffect` + `emit-include` handler recursion.

8. **`adapterOwner` tracking**: Spec: closure captures declaring aspect identity, injected into tombstone `meta.adapterOwner`. Implementation: `meta.constraintOwner` on the child (set in keepChild), `meta.excludedFrom` on tombstone carries owner string. `adapterOwner` as a field name never materialized in the fx code.
