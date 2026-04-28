# Diag lib + structuredTrace follow-ups

**Date:** 2026-04-12
**Branch at time of writing:** `feat/new-trace-demo` (single commit,
amended on top of `feat/has-aspect`)
**Related:** `docs/design/resume-diag-lib.md`,
`docs/design/resume-2026-04-11-hasAspect-followups.md`

Items below surfaced during a code review of the diag-lib updates that
added `hasAspectPresent`, `decisionsView`, `ofNamespace`, and
`hostContext` to the library. Each item is scoped for a future PR — none
of them block the current branch.

---

## 1. Upstream: `filterIncludes` attribution drift (HIGH value)

**Root cause:** in `nix/lib/aspects/adapters.nix::filterIncludes`, when
a user-declared aspect carries `meta.adapter`, the `tag` helper copies
the adapter function onto every surviving child of the first pass. The
recursive `composed = metaAdapter (filterIncludes inner)` chain then
re-invokes the adapter at each level with `args.aspect` rebound to the
current node — not to the user-declared aspect that originally owned
the adapter. When a tombstone is produced several levels down,
`excludedFrom = aspect.meta.originalName or aspect.name or "<anon>"`
reaches for the current-level aspect, which is usually an anonymous
wrapper introduced by `parametric.fixedTo` / `parametric.atLeast`.
Attribution collapses to `"<anon>"`.

**Symptoms observed:** every excluded entry in `structuredTrace` output
gets `excludedFrom = "<anon>"` regardless of which user aspect owned
the `excludeAspect` call. Confirmed via
`templates/diag-demo/modules/aspects/hosts/mail-relay.nix`
(`mail-relay.meta.adapter = ... excludeAspect monitoring ...`) — the
three monitoring provider sub-aspects all tombstone with
`excludedFrom = "<anon>"`.

**Impact on diag lib:** forced `decisionsView` in `filters.nix` to use a
structural "decision parent has both excluded and surviving children"
heuristic instead of an attribution-based lookup. The heuristic is
correct for the common case but has two known weaknesses:

1. **False negatives when only one sibling exists.** An aspect that
   excludes its single include (`foo.meta.adapter = excludeAspect <bar>
   foo.includes = [ <bar> ]`) produces a parent with zero surviving
   children, so it doesn't register as a decision.
2. **Structural-only reading.** A user reading the "structural
   decisions" view can't tell *which* user aspect owned the adapter
   that caused the exclusion — just where the exclusion manifests.

**Proposed fix:** capture the adapter-owner's name once at
`processInclude` time and stash it into the tagged children's
`meta.adapterOwner`. Then `tombstone` reads `adapterOwner` (or falls
back to `aspect.name` for compatibility). Roughly:

```nix
# in adapters.nix::filterIncludes, inside the `tag` helper:
tag =
  i:
  if builtins.isAttrs i && i.meta.adapter or null == null && !(i.meta.excluded or false) then
    i
    // {
      meta = (i.meta or { }) // {
        adapter = metaAdapter;
        # NEW: preserve the original adapter-owning aspect's name
        # through the recursive tag chain so tombstones downstream
        # report it instead of the anonymous wrapper they see.
        adapterOwner = aspect.meta.adapterOwner
          or aspect.meta.originalName
          or aspect.name
          or "<anon>";
      };
    }
  else
    i;

# and in processInclude's tombstone call:
[ (tombstone resolved { excludedFrom = aspect.meta.adapterOwner or aspect.meta.originalName or aspect.name or "<anon>"; }) ]
```

Maybe ~5 lines in `adapters.nix`. Matching CI test: add a case to
`templates/ci/modules/features/aspect-path.nix` asserting that
`structuredTrace` emits `excludedFrom = "mail-relay"` for the monitoring
tombstones.

**Downstream benefits:**
- `decisionsView` can be rewritten to use
  `node.perClass.<class>.excludedFrom` for attribution-based grouping,
  matching the reviewer's original expectation.
- Single-sibling exclusions become visible in the view.
- The `excludedFrom` field in the diag graph IR becomes meaningful — it
  was intentionally reverted in this session as dead code; it can be
  re-added once attribution works.
- Future query "which aspects did X's adapter kill?" becomes a
  one-liner.

**Not urgent:** the structural heuristic in `decisionsView` is correct
for all current demo hosts, and the view's doc comment names the
limitation. Schedule when the ergonomic win justifies the upstream
churn.

---

## 2. Expose `structuredTrace` path-set as a side channel (MEDIUM)

**Observation:** `hasAspectPresent` currently requires a second walk of
the aspect tree via `den.lib.aspects.collectPathSet { tree; class }`,
which internally runs `resolve.withAdapter adapters.collectPaths`
against the same host aspect that `captureAll` just walked for
`structuredTrace`. Two full traversals per class per host.

**Proposed fix:** have `structuredTrace` return both its existing
`{ trace = [...] }` AND a flat `paths = [[...], ...]` field (same
shape as `collectPaths` output). The walker already visits every
non-tombstone aspect to compute `parent`/`provider`/`name`; emitting an
`aspectPath` alongside the entry is a free operation.

```nix
structuredTrace = filterIncludes (
  { aspect, recurse, class, classModule, aspect-chain, ... }:
  let
    entry = { ... as today ... };
    # NEW: pre-compute this aspect's path for downstream path-set
    # consumers. Non-tombstone only; tombstones already filter
    # themselves out upstream.
    path = aspect.meta.provider or [ ] ++ [ entry.name ];
    childResults = map recurse (aspect.includes or [ ]);
  in
  {
    trace = [ entry ] ++ lib.concatMap (r: r.trace or [ ]) childResults;
    paths = lib.optional (!(aspect.meta.excluded or false)) path
         ++ lib.concatMap (r: r.paths or [ ]) childResults;
  }
);
```

Then `diag.capture` / `captureAll` return `{ entries; pathsByClass }`
and `diag.graph.hostContext` grabs both with a single walk.
`hasAspectPresent` reads `graph.pathSets.<class>` just like today, but
the pathSet was computed as a side effect of the trace walk, not from a
separate `collectPathSet` call.

**Cost:** ~10 lines in `adapters.nix`, ~5 lines in
`nix/lib/diag/capture.nix`, ~5 in `default.nix::hostContext`. Maybe
halves the evaluation time for a host that renders hasAspect views.

**Caveat:** changes `structuredTrace`'s return shape. Any external
caller that reads `.trace` by attribute position would break — but
structuredTrace is a diag-internal adapter. Safe-ish.

---

## 3. Canonical `pathKey` field on graph nodes (LOW)

**Observation:** `hasAspectPresent` matches `pathSet ? ${n.fullLabel}`,
which implicitly couples the filter to `graph.nix::fullName`'s
slash-joined string format. If `fullName` ever switches format (dot-
joined, different separator, uses `meta.originalName` differently), the
filter silently starts returning empty sets.

**Proposed fix:** add `pathKey` as a distinct node field (built from
`adapters.pathKey (entry.provider ++ [ entry.name ])` to match
`collectPathSet`'s key format exactly), and have
`hasAspectPresent` match on `pathKey` rather than `fullLabel`. Node
record grows by one string per aspect.

**Cost:** 2 lines in `graph.nix::mkNode`, 1 line in
`filters.nix::hasAspectPresentWith`. No behavior change today; it just
makes the filter format-independent.

**When to do it:** only if `fullName`/`displayLabel` gets touched again
for label-shortening reasons. Not urgent on its own.

---

## 4. `structuredTrace.fnArgs` could be a name list (LOW)

**Observation:** `structuredTrace` emits `entry.fnArgs` as the full
attrset from `lib.functionArgs rawProvided`. Every downstream consumer
(`graph.nix::mkNode`, `diag.toJSON`, parametric-view renderers) then
reduces it to `builtins.attrNames`. The attrset itself is never
inspected.

**Proposed fix:** emit `fnArgNames = builtins.attrNames fnArgs` instead
of `fnArgs`, saving an allocation per parametric aspect.

**Cost:** 1 line in `adapters.nix`, 1 line of renames in `graph.nix`
and `default.nix::toJSON`.

**Why wait:** genuinely trivial performance improvement, but the diff is
a distraction from more valuable work. Bundle with item 2 or another
trace-side change.

---

## 5. `decisionsView` parameterized by predicate (LOW, E13 in review)

**Observation:** the current `decisionsView` hard-codes "decision
parent = both excluded and surviving direct children". A second caller
(e.g. "show me only structural-decision parents owned by adapter X",
or "show one-of-aspects picks where the non-winner is a provider
sub-aspect") would re-implement most of the filter.

**Proposed fix:** split into `decisionParentIds` (id set) and
`decisionsViewBy predicate graph` that builds the keep set by applying
`predicate` over the candidate id set. Then `decisionsView = graph:
decisionsViewBy (id: isDecisionParentStructural id) graph`.

**Cost:** ~15 lines, no existing callers affected.

**When to do it:** when a second caller appears. Speculative until
then.

---

## 6. `diag.graph.queryByClass` / `queryByAnyClass` helpers (LOW, E2 in review)

**Observation:** once `hasAspectPresent { class }` lands on a
graph-shaped record, the natural next primitives are
`hasAspectForClass class g` and `hasAspectForAnyClass classes g` —
the diag-level mirror of `mkEntityHasAspect`'s functor surface.

**Proposed fix:** add to `filters.nix`:

```nix
hasAspectForClass = class: hasAspectPresent { inherit class; };
hasAspectForAnyClass = classes: graph:
  let
    perClass = map (c: hasAspectPresent { class = c; } graph) classes;
    # merge: keep nodes present in ANY class
    keepIds = lib.foldl' (acc: g:
      lib.foldl' (acc': n: acc' // { ${n.id} = true; }) acc g.nodes
    ) { } perClass;
  in
  filterByNodes (n: keepIds ? ${n.id}) graph;
```

Five lines per helper. Unblocks multi-class presence views without a
second walker.

**When to do it:** when a view needs multi-class union semantics. One
currently exists (`cross-class`) but it uses `perClass.hasClass`
directly, not hasAspect.

---

## 7. `filterByStyle` shortcut in `util.nix` (LOW, E6 in review)

**Observation:** `adaptersOnly`, the tombstone-detection side of
`decisionsView`, and several future filters ("only replaced", "only
parametric") all reduce to "keep nodes with style S" or "style in
{S1,S2}". Adding:

```nix
filterByStyle = styles: graph:
  let
    styleSet = lib.listToAttrs (map (s: { name = s; value = true; }) styles);
  in
  filterByNodes (n: styleSet ? ${n.style or "default"}) graph;
```

…would collapse `adaptersOnly`, `tombstonesOnly` (new), etc. to
single-line definitions.

**Cost:** 6 lines in `util.nix`. Immediate savings: shrinks
`adaptersOnly` / `neighborhoodOf` refactor by a few lines.

**When to do it:** when adding a second style-based filter. Not
urgent.

---

## Not doing (review items intentionally deferred)

- **E1** (hostContext → graph-shape record): **DONE** in this session
  as part of the review fixes.
- **E3** (ofNamespace filter param): **DONE** in this session.
- **E5** (hoist `isTombstone`): **DONE** in this session.
- **E8** (ofNamespace root-to-top edges): **DONE** in this session.
- **E4** (pathSetOfHost wrapper): subsumed by item 2 above (pathSets
  are now pre-computed on the graph record).

## Task list anchor

If this doc gets picked up in a future session, the task order should
be:

1. File upstream `filterIncludes` adapterOwner fix (item 1)
2. Add CI test asserting `structuredTrace` emits correct `excludedFrom`
3. Rewrite `decisionsView` to use attribution-based grouping
4. Evaluate item 2 (trace-emits-pathset side channel) as a performance
   pass
5. Remaining items as they become useful
