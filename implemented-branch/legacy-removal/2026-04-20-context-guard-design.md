# Remove context guards — handler-based resolution makes them unnecessary

## Problem

Context-level guards (`perHost`, `perUser`, `take.exactly`, `forwardWrap`) exist
because the old manual-context-forwarding model called functions directly with a
context attrset. A function `{ host }: expr` called at user level received
`{ host, user }` — extra keys leaked through. Guards prevented firing at the
wrong level.

Under handler-based resolution, `bind.fn` compiles `{ host, user }: expr` into
chained effect requests. Each arg is resolved independently from scoped handlers.
A function asking for `{ host }` gets exactly `{ host }` — never `user`, even
if `user` is in scope. The `exactly/atLeast/upTo` distinction is meaningless.

Additionally, `__ctx` data-stamping and `__scope` closures are redundant with
`__scopeHandlers`, which is the single inspectable source of truth.

## Design

### No guard mechanism in the pipeline

The pipeline is agnostic to legacy guard patterns. No `meta.contextGuard`, no
context-key-set matching, no `exactly/atLeast/upTo` awareness. Resolution
relies entirely on:

1. `keepChild` defers until required args have handlers (`has-handler` probing)
2. `bind.fn` resolves exactly the declared args from scoped handlers
3. First context level to satisfy all args wins (natural from deferral)

### Deprecation shims at the boundary

Legacy APIs become identity wrappers with deprecation warnings. They reshape
user code to pass through the pipeline unchanged.

#### perCtx (perHost-perUser.nix)

```nix
perCtx = requiredKeys: aspect:
  lib.warn "den.lib.perCtx is deprecated — handler-based resolution makes context guards unnecessary"
    aspect;

perHost = perCtx [ "host" ];
perUser = perCtx [ "host" "user" ];
perHome = perCtx [ "home" ];
```

`perHost myAspect` → warns, returns `myAspect` unchanged. The pipeline resolves
it when `host` is available in handlers. No guard check needed.

#### take.nix

```nix
take.exactly = fn:
  lib.warn "den.lib.take.exactly is deprecated — bind.fn resolves args from handlers"
    fn;

take.atLeast = fn:
  lib.warn "den.lib.take.atLeast is deprecated — bind.fn resolves args from handlers"
    fn;

take.upTo = fn:
  lib.warn "den.lib.take.upTo is deprecated — bind.fn resolves args from handlers"
    fn;

take.__functor = _: _pred: _adapter: fn:
  lib.warn "den.lib.take custom predicate is deprecated"
    fn;
```

All forms become identity + warning. The inner function passes through as a
normal parametric aspect.

#### forwardWrap (aspect.nix)

```nix
forwardWrap = child: child;
```

Identity. Under handler-based resolution, the pipeline's deferral mechanism
handles context-level gating. No `__functor` guard or `meta.contextGuard`
needed.

#### parametric.nix (fixedTo/expands)

```nix
parametric.fixedTo.__functor = _: ctx: aspect:
  warn "fixedTo is deprecated" (
    aspect // { __scopeHandlers = constantHandler ctx; }
  );

parametric.expands = attrs: aspect:
  let
    existingHandlers = aspect.__scopeHandlers or {};
    merged = existingHandlers // constantHandler attrs;
  in
  warn "expands is deprecated" (
    aspect // { __scopeHandlers = merged; }
  );
```

These reshape the legacy `__ctx` stamping into `__scopeHandlers` so the
pipeline can consume them.

### Default functor (types.nix)

Changes to identity:

```nix
default = self: _: self;
```

No `__ctx` stamping. Context flows through `__scopeHandlers`.

### __scope removal — single source of truth

`__scope` is `scope.stateful __scopeHandlers` pre-applied. Every site that
reads `__scope` can derive it from `__scopeHandlers` at point of use.

**Producers (stop creating `__scope`):**

- `ctx-apply.nix`: only stamp `__scopeHandlers = constantHandler ctx`
- `transition.nix`: only stamp `__scopeHandlers` on target aspects
- `aspectToEffect` tagged block: only propagate `__scopeHandlers`

**Consumers (derive scope from handlers):**

- `aspectToEffect` (aspect.nix): wrap `bind.fn` in scope at point of use:
  ```nix
  scopeHandlers = aspect.__scopeHandlers or null;
  resolveFn =
    if scopeHandlers != null
    then fx.effects.scope.stateful scopeHandlers (fx.bind.fn {} fn)
    else fx.bind.fn {} fn;
  ```

- `emitSelfProvide` (aspect.nix): same pattern for provider resolution

- `resolveChildren` (aspect.nix): derive scopeFn from `__scopeHandlers`,
  pass both `__parentScope` (derived) and `__parentScopeHandlers` to
  `emitIncludes`

- `includeHandler` (include.nix): propagate `parentScopeHandlers` only.
  Stop propagating `parentScope`.

- `resolveConditional` (include.nix): stop passing `__parentScope` to
  `emitIncludes`.

- `emitSelfProvide` (aspect.nix): stop stamping `__parentScope` on
  provider includes.

- `default.nix`: drop `__scope` preservation during wrapping

**Structural keys:** Remove `"__scope"` and `"__parentScope"` from
`structuralKeys` in `aspect.nix`.

### resolvedCtx removal (aspect.nix)

The `resolvedCtx` block extracted `resolved.__ctx` from the functor's return
value and composed it into the scope chain. This served two purposes:

1. **Default functor echo** — redundant: parent `__scopeHandlers` already
   provides these args to children.

2. **Deprecated `fixedTo`/`expands`** — now handled by shims stamping
   `__scopeHandlers` directly.

The block is removed entirely. The `tagged` result simplifies to:

```nix
tagged =
  next
  // lib.optionalAttrs (scopeHandlers != null) { __scopeHandlers = scopeHandlers; }
  // lib.optionalAttrs (aspect ? __ctxId) { inherit (aspect) __ctxId; }
  // { __parametricResolved = true; };
```

### What stays

- `__ctx` on ctxApply results — seeds `state.currentCtx` for `into` functions
- `__ctx` stamps from transition handler — entry-point seeding when results
  re-enter `fxResolveTree`
- `__scopeHandlers` propagation — the single source of truth for context

### What's removed

- `__scope` (opaque pre-applied closure) — derived from `__scopeHandlers`
- `__functor`-based guards in `take.nix`, `perCtx`, `forwardWrap`
- `self.__ctx` reads in guard code
- `__ctx` stamping in the default functor
- `resolvedCtx` extraction block in `aspectToEffect`
- The `__ctx` module option in `types.nix` (already removed)
- `meta.contextGuard` mechanism (unnecessary — pipeline uses deferral + handlers)
- `exactly/atLeast/upTo` context-key matching (artifact of manual forwarding)

## Rationale

Under manual context forwarding, `{ host, user }: expr` was called directly
with a context attrset. The function received all keys at once, so guards were
needed to differentiate context levels.

Under handler-based resolution, `bind.fn` compiles the function into chained
effect requests — each arg resolved independently. `{ host, user }: expr` means
"I need both host AND user handlers." If user isn't available, the child is
deferred. The pipeline's deferral + handler probing naturally provides the
correct semantics without any guard mechanism.

The `exactly` distinction ("fire only when scope has these keys and no more")
is unnecessary because `bind.fn` never passes extra args. A function asking for
`{ host }` gets `{ host }` at both host and user levels — the presence of
`user` in the handler scope is invisible to the function.

### Static aspects and the optional-arg pattern

Some user code wraps static attrsets in `perHost` (e.g., `perHost { nixos = ...; }`).
For cases where transition structure alone doesn't prevent wrong-level resolution
(e.g., perHost aspects in shared includes lists), the shim uses the optional-arg
pattern: declare deeper context keys as optional, check which resolved.

nix-effects `bind.fn` now skips optional args when no handler exists (commit
c7931d7), letting Nix defaults kick in. The shim functor checks whether deeper
keys resolved — if so, returns `{}` (wrong level).

### Remove `__functor`/`__functionArgs` from aspect submodule type (next step)

The aspect submodule in `types.nix` currently declares `__functor` and
`__functionArgs` as options with defaults. This causes every submodule-evaluated
aspect to carry these attributes, which creates two problems:

1. **Infinite re-entry in forwardWrap.** When a parametric aspect resolves
   via `bind.fn`, the result carries `__functor`/`__functionArgs` from the
   submodule defaults. `aspectToEffect` sees it as parametric again → loops.
   Current workaround: `forwardWrap` strips them.

2. **`__ctx` stamping in the default functor.** The default
   `self: ctx: self // { __ctx = ctx; }` is the old context-forwarding
   mechanism. Removing it eliminates `__ctx` from the parametric resolution
   path entirely.

Neither option serves a purpose in the handler-based model:

- **`__functor`** is only needed on EXPLICITLY parametric aspects. Bare
  functions get wrapped by `fxResolveTree`. Functor attrsets carry their
  own. The submodule default adds it to every aspect unnecessarily.

- **`__functionArgs`** defaults to `{}`. Only meaningful when explicitly
  set. The submodule default is a no-op that pollutes the attrset.

Removing both options makes:
- `forwardWrap` true identity (no `__functor` to strip)
- `isParametric` false for all submodule-evaluated non-parametric aspects
- Resolved results naturally route to `compileStatic`
- The infinite re-entry class of bugs eliminated structurally

**This requires cascading changes** across the pipeline. Sites that assume
aspects are callable via `__functor` (wrapChild, emitSelfProvide,
fxResolveTree wrapping, include.nix normalization) need updating. This is
the next implementation phase.

## Files changed

### Already implemented

| File | Change |
|------|--------|
| `nix/lib/aspects/fx/handlers/include.nix` | `keepChild`: removed `meta.contextGuard` branch (not needed) |
| `nix/lib/aspects/fx/aspect.nix` | `forwardWrap`: strips `__functor`/`__functionArgs`; derive scope from `__scopeHandlers` |
| `modules/context/perHost-perUser.nix` | Optional-arg pattern shim + deprecation warning |
| `nix/lib/take.nix` | Identity shims + deprecation warnings |
| `templates/ci/flake.lock` | Updated nix-effects (optional arg support in bind.fn) |

### Remaining (next phase)

| File | Change |
|------|--------|
| `nix/lib/aspects/types.nix` | Remove `__functor` and `__functionArgs` options from submodule |
| `nix/lib/aspects/fx/aspect.nix` | `forwardWrap`: true identity; cascade fixes for missing `__functor` |
| `nix/lib/aspects/fx/handlers/include.nix` | `wrapChild`: stop assuming aspects have `__functor`; stop propagating `__parentScope` |
| `nix/lib/aspects/fx/handlers/transition.nix` | Stop stamping `__scope`, only stamp `__scopeHandlers` |
| `nix/lib/ctx-apply.nix` | Stop stamping `__scope`, only stamp `__scopeHandlers` |
| `nix/lib/aspects/default.nix` | Drop `__scope` preservation during wrapping |
| `nix/lib/parametric.nix` | Update shims to stamp `__scopeHandlers` instead of `__ctx` |
