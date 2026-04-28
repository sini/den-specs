# Effectful Handlers for nix-effects

**Date:** 2026-04-14
**Repo:** github.com/sini/nix-effects (fork of vic/nix-effects)
**Status:** Draft

## Problem

nix-effects handlers are currently pure — they return `{ resume = value; state; }` where `resume` is a plain value fed to the continuation. Handlers cannot return computations.

This prevents deep/recursive handling patterns where a handler needs to run a sub-computation before resuming. In den's fx pipeline, this blocks the "aspects as computations" architecture: an `include` handler needs to recursively resolve a child aspect (itself a computation) before resuming with the result.

**Current:**
```nix
"include" = { param, state }: {
  resume = 42;  # plain value only
  inherit state;
};
```

**Needed:**
```nix
"include" = { param, state }: {
  resume = fx.bind (fx.send "check" param) (decision:
    fx.pure decision.value
  );  # computation — handle it, then feed result to continuation
  inherit state;
};
```

## Design

### Change to `interpret` (trampoline.nix line ~107-109)

Current:
```nix
else if result ? resume then
  [{ key = k;
     _comp = resumeWithQueue step._comp.queue result.resume;
     _state = newState; }]
```

The `resumeWithQueue` call feeds `result.resume` directly to the continuation queue. If `resume` is a computation, this breaks — the continuation receives a computation object as a value instead of the computation's result.

**Fix:** Detect whether `resume` is a computation. If so, bind it with the continuation queue instead of feeding it directly.

```nix
else if result ? resume then
  let
    r = result.resume;
    isComp = r ? _tag && (r._tag == "Pure" || r._tag == "Impure");
    nextComp =
      if isComp then
        # Effectful resume: bind the computation with the continuation
        if isPure r then
          resumeWithQueue step._comp.queue r.value
        else
          # Impure: append the continuation queue to the computation's queue
          impure r.effect (queue.append r.queue step._comp.queue)
      else
        # Plain value: existing behavior
        resumeWithQueue step._comp.queue r;
  in
  [{ key = k; _comp = nextComp; _state = newState; }]
```

The key insight: when `resume` is an `Impure` computation, we append the original continuation queue to the computation's own queue. This means:
1. The computation's effects are handled first (by the same handler set)
2. When the computation reaches `Pure`, the original continuation runs with the result

This is equivalent to `fx.bind resumeComp (value: resumeWithQueue queue value)` but avoids creating an intermediate bind — it directly splices the queues.

### Change to `rotateInterpret` (trampoline.nix line ~136-139)

Same pattern for the rotation interpreter:

```nix
else if result ? resume then
  let
    r = result.resume;
    isComp = r ? _tag && (r._tag == "Pure" || r._tag == "Impure");
    nextComp =
      if isComp then
        if isPure r then
          resumeWithQueue comp.queue r.value
        else
          impure r.effect (queue.append r.queue comp.queue)
      else
        resumeWithQueue comp.queue r;
  in
  rotateInterpret {
    comp = nextComp;
    inherit handlers done;
    state = newState;
  }
```

### Backward compatibility

Plain value resumes continue to work unchanged — the `isComp` check is false for non-computation values, so `resumeWithQueue` is called with the raw value as before.

The only risk: a handler that intentionally returns an attrset with `_tag = "Pure"` as a plain value (not a computation). This is extremely unlikely — `_tag` is an internal implementation detail. If needed, a `_isComp = true` sentinel could disambiguate, but YAGNI.

### State threading

When a handler returns an effectful resume, the computation runs with the state returned by the handler (`newState`). The sub-computation's effects are handled by the SAME handler set with this updated state. This is correct — the sub-computation is part of the same handling scope.

If the sub-computation sends effects that modify state, those state changes propagate through to the original continuation. This is the desired behavior for deep handling.

### Tests

```nix
# Effectful resume: handler returns a computation
test-effectful-resume = {
  expr =
    let
      comp = send "outer" null;
      result = handle {
        handlers = {
          "outer" = { param, state }: {
            # Resume with a computation that sends another effect
            resume = bind (send "inner" 10) (x: pure (x * 2));
            inherit state;
          };
          "inner" = { param, state }: {
            resume = param + 1;
            inherit state;
          };
        };
        state = null;
      } comp;
    in result.value;
  expected = 22;  # inner resumes 11, outer computation doubles to 22
};

# Effectful resume with state threading
test-effectful-resume-state = {
  expr =
    let
      comp = send "resolve" null;
      result = handle {
        handlers = {
          "resolve" = { param, state }: {
            resume = bind (send "count" null) (_: send "count" null);
            state = state // { resolved = true; };
          };
          "count" = { param, state }: {
            resume = null;
            state = state // { n = (state.n or 0) + 1; };
          };
        };
        state = {};
      } comp;
    in { value = result.value; n = result.state.n; resolved = result.state.resolved; };
  expected = { value = null; n = 2; resolved = true; };
};

# Effectful resume in rotate
test-effectful-resume-rotate = {
  expr =
    let
      comp = send "a" null;
      rotated = rotate {
        handlers = {
          "a" = { param, state }: {
            resume = bind (send "b" 5) (x: pure (x + 1));
            inherit state;
          };
          "b" = { param, state }: {
            resume = param * 2;
            inherit state;
          };
        };
      } comp;
      result = handle { handlers = {}; } rotated;
    in result.value;
  expected = 11;  # b resumes 10, computation adds 1
};

# Plain resume still works (backward compat)
test-plain-resume-unchanged = {
  expr =
    let
      comp = send "x" 5;
      result = handle {
        handlers.x = { param, state }: { resume = param + 1; inherit state; };
      } comp;
    in result.value;
  expected = 6;
};

# Effectful resume with continuation
test-effectful-resume-with-continuation = {
  expr =
    let
      comp = bind (send "fetch" null) (x: pure (x + 100));
      result = handle {
        handlers = {
          "fetch" = { param, state }: {
            resume = bind (send "db" null) (row: pure row.value);
            inherit state;
          };
          "db" = { param, state }: {
            resume = { value = 42; };
            inherit state;
          };
        };
      } comp;
    in result.value;
  expected = 142;  # db returns {value=42}, fetch extracts 42, continuation adds 100
};
```

## Files to modify

1. `/home/sini/Documents/repos/nix-effects/src/trampoline.nix` — `interpret` and `rotateInterpret`
2. Tests in the same file or a dedicated test file

## Scope

This is a minimal change — two code paths in trampoline.nix, backward compatible, with tests. No changes to `comp.nix`, `queue.nix`, `kernel.nix`, or the public API surface.
