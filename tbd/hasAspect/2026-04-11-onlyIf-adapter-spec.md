# Spec: Guarded include wrappers (`onlyIf` adapter)

**Date:** 2026-04-11
**Status:** Rough spec, pre-brainstorm. Use as a starting point. Refine via the brainstorming skill if complexity grows beyond the sizing below.
**Parent work:** `feat/has-aspect` branch, see `docs/design/resume-2026-04-11-hasAspect-followups.md`.

## One-sentence summary

Let users write `includes = [ (onlyIf <guard> <target>) ]` to include an aspect only when a structural condition holds in the parent entity's resolved tree, using the same adapter machinery `oneOfAspects` is built on.

## Primary API

One function in `den.lib.aspects.adapters`:

```nix
onlyIf :: aspect-ref -> aspect -> guarded-aspect
# "include target only if guard is structurally present in the tree"
```

Usage:

```nix
den.aspects.impermanence.includes = [
  (den.lib.aspects.adapters.onlyIf <zfs-root> <zfs-impermanence>)
  (den.lib.aspects.adapters.onlyIf <btrfs-root> <btrfs-impermanence>)
];
```

That's the 95% shape: "this aspect belongs in the tree iff that aspect is also in the tree."

## Behavior

- **When guard is structurally present in the tree at resolve time:** target is included as if it were written directly in the parent's `includes` list. Its own `meta.adapter`s, functor context, class configs, everything flows through unchanged.
- **When guard is absent:** the wrapper and its target both tombstone cleanly. Nothing from target contributes to the entity's final module. The tombstone is visible in trace views as a `~onlyIf-<target-name>` entry.
- **When guard is itself a tombstoned aspect** (present in the declared tree but excluded by another adapter): still counts as present. Guard checking is structural-only, matching `oneOfAspects`. This is the only cycle-safe definition.

## Identity rules (carried from `hasAspect`)

- Guard is compared by `aspectPath` (`meta.provider ++ [name]`).
- `<angle-bracket>` sugar and direct `den.aspects.<name>` references are interchangeable.
- Provider sub-aspects (`foo._.sub`) use their full `["foo","sub"]` path.
- Factory-fn aspects have stable identity at the factory reference itself. Invoked factory instances (`factory arg`) are anonymous and won't match. Users who want instance-level guarding name their instances: `den.aspects.my-facter = facter "/x"; ... onlyIf <my-facter> <something>`.
- Malformed refs (missing `name` or `meta`) throw loudly at adapter time, using the same validator pattern `hasAspect`'s `refKey` uses.

## Non-goals

- **Post-adapter semantics.** `onlyIf` sees the structurally-declared tree, not the tree after other adapters have run. If you need "keep X only if Y survives all the filtering", that's a different feature and probably wants a new resolve phase. Not doing that here.
- **Arbitrary predicate guards.** No `onlyIf.when (condition: bool)` or similar. Guards are aspect references. If users want boolean predicates, that's `lib.mkIf` inside a class-config body, not this feature.
- **Two-way dependence (soft presence).** No "include target iff guard is included AND guard says yes". One-way structural check is the whole trick.
- **Auto-deduplication of nested wrappers.** `onlyIf A (onlyIf B C)` is valid and means "include C iff A and B are both present." Each wrapper makes its decision independently. No special handling.

## Implementation sketch

One adapter in `nix/lib/aspects/adapters.nix` next to `oneOfAspects`. Reuses existing helpers (`collectPathsInner`, `pathKey`, `toPathSet`, `aspectPath`) that the `feat/has-aspect` branch already exports.

```nix
onlyIf =
  guard: target:
  {
    name = "onlyIf-${target.name or "<anon>"}";
    includes = [ target ];
    meta.adapter =
      inherited:
      args@{ class, aspect-chain, ... }:
      let
        root = lib.head aspect-chain;
        presence = toPathSet (
          (den.lib.aspects.resolve.withAdapter collectPathsInner class root).paths or [ ]
        );
      in
      if presence ? ${pathKey (aspectPath guard)} then inherited args else { };
  };
```

Size: ~15 lines. No new helpers, no changes outside `adapters.nix`, no changes to `resolve.nix` or `filterIncludes`.

## How it avoids cycles

Same technique `oneOfAspects` already uses:

- The decision runs in a `meta.adapter`, not inside an aspect's own body. The adapter fires during `filterIncludes` processing of the parent, AFTER the parent's includes list is built.
- The presence walk uses `collectPathsInner` passed directly to `resolve.withAdapter`, which bypasses `filterIncludes` entirely. That means the walk doesn't process any `meta.adapter`s, so it can't re-enter the `onlyIf` it's currently evaluating.
- The walk forces functors (via `take.upTo` in `apply`), but only for `includes` traversal. Class-config bodies (`nixos = ...`, etc.) remain deferred. Users who call `hasAspect` inside a class body are cycle-safe for the same reason they were before `onlyIf` existed.

The only new cycle risk is if `guard`'s aspect chain contains another `onlyIf` pointing back, but that's already the case for any mutually-recursive meta.adapter setup and resolves the same way: each wrapper makes its independent structural decision, and the raw walker never re-enters a decision in progress.

## API shape alternatives considered

**A. `onlyIf <guard> <target>`** (recommended). Two positional refs, read as "only include target if guard present." Clean at call site, matches the mental model users will have.

**B. `onlyIf { when, include }`** (attrset form). More verbose at call site but more discoverable via attribute completion. Rejected: the two-positional form is shorter and the argument order is unambiguous because the refs have distinct semantic roles.

**C. `target.when <guard>`** (method-like on the target). Rejected: aspects aren't Lua objects. Pretending they are would require some kind of `den.lib.aspects.withGuard` constructor on every aspect type, which is overkill.

**D. `guard.gates <target>`** (method-like on the guard). Same objection as C, and the reading is backwards: the guard doesn't "gate" anything, it's queried. `onlyIf` reads naturally.

## Testing strategy

One test file: `templates/ci/modules/features/only-if.nix`. Roughly 7-8 tests:

1. `test-guard-present-keeps-target`: guard declared in `host.includes` elsewhere, target survives.
2. `test-guard-absent-drops-target`: guard declared nowhere, target is tombstoned. Verify via `trace` that the wrapper shows as `~onlyIf-<target>`.
3. `test-nested-onlyIfs`: `onlyIf A (onlyIf B C)` requires both A and B present for C to survive.
4. `test-guard-via-provider-sub-aspect`: guard is `foo._.sub`, identity uses full `["foo","sub"]` path.
5. `test-guard-via-factory-reference`: guard is a factory-fn aspect referenced directly, factory identity works.
6. `test-composes-with-oneOfAspects-at-different-levels`: nested onlyIf inside a oneOfAspects parent, both adapters take effect at their own level.
7. `test-composes-with-excludeAspect-at-parent-level`: outer `excludeAspect` excludes a sibling, inner `onlyIf` drops target, both take effect independently.
8. `test-multiple-onlyIfs-at-same-parent-independent`: two `onlyIf`s in the same includes list make independent decisions (one kept, one dropped, for the same parent resolution).

Each test uses the `denTest` harness and the `trace` specialArg for tombstone-visibility assertions, matching the existing `one-of-aspects.nix` patterns.

## Performance notes

Each `onlyIf` wrapper does one `resolve.withAdapter collectPathsInner class root` call when its meta.adapter fires. For a host with N wrappers, that's N walks, each O(tree size). Noticeable if N is large (dozens).

**Fix if it bites:** memoize the presence set at the `(root, class)` granularity. Probably as a thunk threaded through `resolve.withAdapter` or stashed on the entity's `config.resolved`. Same optimization would also benefit `oneOfAspects`. Don't pre-optimize; measure first.

## Template example update

Append a Pattern 4 section to `templates/example/modules/aspects/hasAspect-examples.nix`:

```nix
# Pattern 4: conditional include via onlyIf
#
# For "include X only if Y is also present", `onlyIf` is the
# cleanest shape. It's a wrapper that sits in the includes list
# and decides at resolve time whether to surface its target.
den.aspects.example-impermanence-bundle.includes = [
  (den.lib.aspects.adapters.onlyIf <example-zfs-root> <example-zfs-impermanence>)
  (den.lib.aspects.adapters.onlyIf <example-btrfs-root> <example-btrfs-impermanence>)
];
```

The anti-pattern section in the same file stays. `hasAspect` inside a literal `includes = [...]` is still wrong. `onlyIf` is the correct alternative for the single-gated-include case, `oneOfAspects` is the correct alternative for the pick-one-of-N case. Both go in the "correct tools" list at the bottom of the anti-pattern section.

## Scope check

Stays within `nix/lib/aspects/adapters.nix` + one new test file + one template edit. Zero changes to core lib files (`parametric.nix`, `resolve.nix`, `types.nix`, `ctx-apply.nix`). Zero changes to `modules/context/has-aspect.nix` or the `has-aspect.nix` library. Fully additive, purely a new adapter.

If the scope grows beyond that (memoization plumbing, new resolve phase, anything that requires changing `filterIncludes`), bounce into the brainstorming skill and write a fresh spec. This rough spec is sized for "small follow-up PR," not for a big architectural pass.

## Estimated sizing

- `adapters.nix`: +15 lines (the adapter)
- `templates/ci/modules/features/only-if.nix`: ~150 lines (new file, 7-8 tests)
- `templates/example/modules/aspects/hasAspect-examples.nix`: ~15 lines (Pattern 4 section)
- `docs/` (design + resume updates): untracked, working only

Single commit shape: `feat(adapters): add onlyIf conditional-include adapter`. One PR.

## Open questions (for brainstorm if they come up)

1. **Naming.** `onlyIf` vs `whenPresent` vs `requires` vs `guardedBy`. `onlyIf` has the best read-at-call-site (`onlyIf <guard> <target>` parses as English). The others are either longer or less clear. Not worth bikeshedding unless the team has strong feelings.
2. **Should the wrapper's own aspectPath be queryable via `hasAspect`?** The wrapper has a name (`onlyIf-<target-name>`) so technically yes, but this is likely not useful and could be confusing. Document that the wrapper is an implementation detail and users should query the target or the guard, not the wrapper.
3. **Multiple guards.** `onlyIfAll [<g1> <g2>] <target>` for "require all of these" and `onlyIfAny [<g1> <g2>] <target>` for "require any of these"? Could be useful. Probably wait until someone asks for it and do them as small siblings of `onlyIf` in the same file.
