# Dead Letter Queue for Unregistered Aspect Keys

## Status: Implemented

Shipped on `feat/fx-pipeline` (commits 999cd036..5ab9ba79). Dead letter handler, DLQ state, drain handler, emit helpers, and `den.classes` registrations in templates all landed. ~659 tests passing.

## Problem

`classifyKeys` drops aspect keys not in `classRegistry`/`traitRegistry` with a trace warning. This breaks:

1. **Templates with custom classes** (e.g., flake-parts-modules `devshell`, `treefmt`) — the class modules define policies but don't register `den.classes.devshell = {}`. On main, all non-structural keys defaulted to class. After registry-based classification, they're silently dropped.

2. **Deferred includes declaring traits/classes** — aspects resolved only during pipeline time (deferred includes, forward sub-pipelines, policy-injected aspects) can declare `traits.foo = {}` on themselves, which `aspect-schema.nix` folds into `den.traits` at module fixed-point. But if the declaring aspect is only reachable via a deferred include that gets drained mid-pipeline, the registry is already computed. Its keys on OTHER aspects get dropped.

## Design

### Dead Letter Effect and Handler

**New effect: `"dead-letter"`** — emitted by `compileStatic` for each `unregisteredClassKeys` entry instead of the trace warning.

Payload — captures everything needed for deferred re-emit:
```nix
{
  key = "devshell";
  rawValue = aspect.${k};    # aspectContentType wrapper: { __contentValues = [{ value; file; }]; __provider; }
  aspectIdentity = nodeIdentity;
  aspectName = rawName;
  # Context captured at classification time for class re-emit via wrapClassModule:
  ctx = ctxFromHandlers (aspect.__scopeHandlers or { });
  aspectPolicy = aspect.meta.collisionPolicy or null;
  globalPolicy = den.config.classModuleCollisionPolicy or "error";
}
```

Note: `targetClass`-matched keys bypass classification entirely (L315: `targetClass != null && k == targetClass` routes to `classKeys`), so they never enter the DLQ.

**New handler: `deadLetterHandler`** in `tree.nix`:
```nix
"dead-letter" = { param, state }: {
  resume = null;
  state = state // {
    deadLetterQueue = x: (state.deadLetterQueue x) ++ [ param ];
  };
};
```

**New state field** in `defaultState`:
```nix
deadLetterQueue = _: [];
```

### Drain Mechanism

**New effect: `"drain-dead-letters"`** — called at the same points as `"drain-deferred"` (in `resolveContextValue` after new context enters scope).

Handler behavior:
1. Read current `deadLetterQueue` from state
2. For each entry, re-classify the key against `classRegistry` and `traitRegistry`
3. Entries matching a class → re-emit as `"emit-class"` (using `wrapClassModule` on the raw value, same as `emitClasses`)
4. Entries matching a trait → re-emit as `"emit-trait"` (using the raw value, same as `emitTraits`)
5. Entries still unmatched → remain in DLQ
6. Update `state.deadLetterQueue` to only contain remaining unmatched entries

The drain handler needs access to `classRegistry`, `traitRegistry`, and the `wrapClassModule`/`emitTraits` emit logic. Since these are defined in `aspect.nix` (module scope), the drain handler can be defined there alongside `emitClasses`/`emitTraits` and exported to the handler composition.

Alternatively, the drain handler can be simpler: instead of re-running `wrapClassModule` inline, it emits the matched entries as `"emit-class"` or `"emit-trait"` effects. The existing handlers for those effects handle the wrapping. This avoids duplicating emit logic in the drain handler.

```nix
drainDeadLettersHandler = {
  "drain-dead-letters" = { param, state }:
    let
      queue = (state.deadLetterQueue or (_: [])) null;
      classified = builtins.partition (entry:
        classRegistry ? ${entry.key} || traitRegistry ? ${entry.key}
      ) queue;
      matched = classified.right;
      remaining = classified.wrong;

      # Re-emit matched entries as class or trait effects.
      # Each entry carries full context captured at dead-letter time.
      reEmits = lib.concatMap (entry:
        if classRegistry ? ${entry.key} then
          emitClassFromDLQ entry
        else
          emitTraitFromDLQ entry
      ) matched;
    in {
      resume = fx.seq reEmits;
      state = state // {
        deadLetterQueue = _: remaining;
      };
    };
};
```

**`emitClassFromDLQ`** — unwraps `rawValue` (aspectContentType), runs `wrapClassModule` on each content value using captured context, emits `"emit-class"`:

```nix
emitClassFromDLQ = entry:
  let
    rawValue = entry.rawValue;
    modules =
      if builtins.isList rawValue then rawValue
      else if builtins.isAttrs rawValue && rawValue ? __contentValues then
        let vals = map (d: d.value) rawValue.__contentValues;
        in if builtins.length vals == 1 then [ (builtins.head vals) ] else [ { imports = vals; } ]
      else [ rawValue ];
    indexed = lib.imap0 (idx: module: { inherit idx module; }) modules;
    isMulti = builtins.length modules > 1;
  in
  lib.concatMap ({ idx, module }:
    let
      result = wrapClassModule {
        inherit module;
        ctx = entry.ctx;
        aspectPolicy = entry.aspectPolicy;
        globalPolicy = entry.globalPolicy;
        traitNames = traitRegistry;
      };
      elemIdentity = if isMulti then "${entry.aspectIdentity}[${toString idx}]" else entry.aspectIdentity;
    in
    if result.unsatisfied or false then []
    else
      [ (fx.send "emit-class" {
        class = entry.key;
        identity = elemIdentity;
        inherit (result) module;
        isContextDependent = result.wrapped;
      }) ]
      ++ lib.optional (result ? validator) (fx.send "emit-class" {
        class = entry.key;
        identity = "${elemIdentity}/<collision-validator>";
        module = lib.setFunctionArgs result.validator result.advertisedArgs;
        isContextDependent = true;
      })
  ) indexed;
```

**`emitTraitFromDLQ`** — unwraps `rawValue`, emits `"emit-trait"` per content value:

```nix
emitTraitFromDLQ = entry:
  let
    rawValue = entry.rawValue;
    contentValues =
      if builtins.isAttrs rawValue && rawValue ? __contentValues then rawValue.__contentValues
      else [{ value = rawValue; file = "<unknown>"; }];
  in
  map (cv: fx.send "emit-trait" {
    trait = entry.key;
    inherit (cv) value;
    chain = entry.aspectIdentity;
  }) contentValues;
```

These mirror the emit logic in `emitClasses` (L395-457) and `emitTraits` (L366-393) exactly, using the DLQ entry's captured context instead of the live aspect.

### Post-Pipeline Diagnostics

`fxFullResolve` already exposes full state. Callers can inspect `state.deadLetterQueue null` for entries that were never claimed. `fxResolve` doesn't expose state (NixOS deferredModule constraint), but the DLQ is accessible via `fxFullResolve` for debugging.

For user-facing diagnostics: after pipeline completes, if `deadLetterQueue` is non-empty, emit `builtins.trace` warnings with structured info (key name, aspect name, source file from `__contentValues`).

### Template Fix

Separately from the DLQ mechanism, `templates/flake-parts-modules/modules/classes/*.nix` need `den.classes` registrations:

```nix
# devshell.nix
den.classes.devshell = {};

# treefmt.nix
den.classes.treefmt = {};

# packages.nix
den.classes.packages = {};

# files.nix
den.classes.files = {};

# nix-unit.nix
den.classes.tests = {};
```

This ensures these classes are in the static registry at pipeline time. The DLQ is for dynamic cases; known classes should be registered.

## Call Sites for drain-dead-letters

Same as `drain-deferred` — `resolveContextValue` in `transition.nix`. The drain happens when new context enters scope, which is also when deferred includes get drained. If a deferred include brings in an aspect that declares a new trait (via `aspect-schema.nix` → `den.traits`), the static registry already has it. The drain re-checks and reclaims.

```nix
# In resolveContextValue, after drain-deferred:
fx.bind (fx.send "drain-deferred" scopedCtx) (satisfiable:
  fx.bind (fx.send "drain-dead-letters" null) (_:
    # ... process satisfiable ...
  )
)
```

## Impact

- `aspect.nix`: Replace trace warning with `"dead-letter"` effect emission
- `tree.nix`: Add `deadLetterHandler` and `drainDeadLettersHandler`
- `pipeline.nix`: Add `deadLetterQueue` to `defaultState`, add handler to `defaultHandlers`
- `transition.nix`: Add `"drain-dead-letters"` call after `"drain-deferred"` in `resolveContextValue`
- `templates/flake-parts-modules/modules/classes/*.nix`: Add `den.classes` registrations

## Risks

- **Drain re-emit ordering**: DLQ entries re-emitted as `"emit-class"` arrive later than normal class emissions. `classCollectorHandler` just appends to the bucket — ordering within a bucket shouldn't matter for NixOS module merging, but verify with dedup (`include-seen`).
- **Double-emit**: If the same aspect is processed twice (once with unregistered key → DLQ, then drained), ensure the DLQ entry isn't also emitted normally. The DLQ replaces the drop, so no double-emit: the key was never emitted as a class in the first place.
- **Registry timing**: `classRegistry`/`traitRegistry` are computed once at module-system fixed-point. `aspect-schema.nix` folds ALL aspect-declared traits/classes into `den.traits`/`den.classes` at that point — even from aspects only reachable via deferred includes. The fixed-point means the registry is complete from pipeline start; the DLQ drain re-checks against this complete registry. If a trait is NOT in `config.den.aspects` at all (truly dynamic, no module-system path), the drain won't match and it stays in DLQ.
- **Nested key gap**: The drain only checks class/trait registry membership, not the nesting detection logic (`hasRecognizedSubKeys`). An unregistered key whose sub-keys become recognizable after registry population won't be reclaimed as a nested key. This is an acceptable limitation — nested aspect detection is a heuristic, and the DLQ primarily targets class/trait reclamation.
