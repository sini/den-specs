# Pipes and Quirks

**Component Reference** | feat/fx-pipeline (753/753 tests)

---

## 1. Overview

Pipes are Den's named data channels for structured, non-class content. Aspects produce pipe data by setting keys on their attrset (called "quirks"); class modules consume aggregated pipe data as function arguments. Policies attach transform stages and cross-scope operations to pipes, controlling how data flows between scopes. Pipes reuse the existing `scopedClassImports` collection infrastructure -- pipe keys and class keys flow through the same `emit-class` handler, differing only at consumption time.

---

## 2. Declaration

Pipes are declared at `den.quirks` (the option-level name; internally referenced as `pipeRegistry`). Each declaration is minimal -- just metadata:

```nix
den.quirks.firewall = {
  description = "Firewall port declarations";
};
```

The `den.quirks` option is `lazyAttrsOf` a submodule with a single `description` field. The `apply` function on the option:

1. Asserts that `den.classes` and `den.quirks` share no keys (collision = error).
2. Injects a `name` field into each quirk value, matching its attribute name.

### Ref syntax

Because each quirk gets a `name` field, `den.quirks.firewall` can be passed directly to `pipe.from` as a ref instead of the string `"firewall"`. `pipe.from` accepts both:

```nix
pipe.from "firewall" [ ... ]        # string name
pipe.from den.quirks.firewall [ ... ] # ref (uses .name)
```

Implementation in `policy-effects.nix`:

```nix
pipeName = if builtins.isAttrs pipeNameOrRef then pipeNameOrRef.name else pipeNameOrRef;
```

---

## 3. Production

Aspects produce pipe data by setting a key matching a declared pipe name. The aspect does not need to know a pipe exists -- it just sets data on a key:

```nix
den.aspects.nginx = {
  nixos.services.nginx.enable = true;   # class module (key in den.classes)
  firewall = { ports = [ 80 443 ]; };  # quirk (key in den.quirks)
};
```

### Key classification

`classifyKeys` in `key-classification.nix` categorizes aspect keys into four groups:

| Category | Condition | Behavior |
|---|---|---|
| `classKeys` | Key in `den.classes` or matches target class | Emitted as class module |
| `pipeKeys` | Key in `den.quirks` | Emitted via `emit-class` into `scopedClassImports`, but NOT wrapped as class module |
| `nestedKeys` | Contains recognized class sub-keys | Treated as nested aspect |
| `unregisteredClassKeys` | None of the above | Emitted as class module (backwards compatibility) |

Pipe keys flow through `emit-class` into `scopedClassImports.${scope}.${pipeName}` -- the same collection path as class content. The distinction matters only post-pipeline: `assemblePipes` reads pipe keys; `wrapPerScope` reads class keys.

### Value forms

A quirk value can be:

- **Plain value** -- available at pipeline time: `firewall = { ports = [ 80 ]; };`
- **Parametric on den context** -- resolved during walk via `bind`: `firewall = { host, ... }: { ports = [...]; };`
- **Config-dependent thunk** -- collected as-is, resolved post-pipeline: `http-backends = { config, ... }: [{ addr = config.networking.hostName; }];`
- **List** -- auto-flattened: `firewall = [ { ports = [80]; } { ports = [443]; } ];` contributes two entries

---

## 4. Consumption

Consumers receive pipe data as function arguments. The pipe name appears in the function signature:

```nix
den.aspects.networking = {
  nixos = { firewall, lib, ... }: {
    networking.firewall.allowedTCPPorts =
      lib.concatMap (f: f.ports or []) firewall;
  };
};
```

### Delivery mechanism

Pipe data reaches consumers through two paths:

**Class modules (post-pipeline):** `assemblePipes` injects assembled pipe data into `scopeContexts` as keys. `wrapClassModule` detects pipe names via its existing `ctx ? ${k}` check and pre-applies the data -- the same mechanism used for entity context args (`host`, `user`).

**Pipeline-time discriminators (in `includes`):** The `bind` handler in `bind.nix` checks required args against `pipeRegistry`. When a pipe arg is detected, the aspect is unconditionally deferred (`hasPipeArgs = true`). Deferred pipe-arg aspects are drained after `assemblePipes` runs, when pipe data is available in the augmented scope contexts.

```nix
den.aspects.secure-server = {
  includes = [
    ({ firewall, ... }:
      let hasHttps = builtins.any (f: builtins.elem 443 (f.ports or [])) firewall;
      in lib.optionalAttrs hasHttps {
        includes = [ den.aspects.tls-hardening ];
      })
  ];
};
```

### Per-aspect targeting

When `pipe.to` targets specific aspects, `assemblePipes` builds a `__pipeTargeted` map: `{ aspectIdentityKey -> { pipeName -> values } }`. During `wrapPerScope`, `applyPipeTargeting` in `wrap-classes.nix` overrides the scope-wide pipe data with targeted data for matching aspects, keyed by full identity pathkey (e.g., `"provider/postgres"`).

### Empty pipes

Pipes with no emissions produce `[]`. Consumers always receive a value -- never missing or null.

---

## 5. Pipe Policy Effects

Policies attach transform and routing stages to pipes via `pipe.from`. The pipe builder API lives in the `pipe` namespace of `den.lib.policy` (defined in `policy-effects.nix`).

### `pipe.from`

```nix
pipe.from <pipe-name-or-ref> [ <stage> ... ]
```

Returns a single `{ __policyEffect = "pipe"; value = { pipeName, stages }; }` effect. The effect is collected into `scopedPipeEffects.${scope}` during the walk by the `register-pipe-effect` handler.

### Transform stages

Transform stages apply in declared order. Each operates on the current value list:

| Stage | Signature | Behavior |
|---|---|---|
| `pipe.filter` | `(elem -> bool) -> stage` | Remove entries not matching predicate. Config thunk markers pass through unchanged. |
| `pipe.transform` | `(elem -> elem) -> stage` | Map each entry. Config thunk markers pass through unchanged. |
| `pipe.fold` | `(acc -> elem -> acc) -> init -> stage` | Reduce the list. Result is wrapped in a single-element list. Note: arg order is `(acc -> elem -> acc)` using `builtins.foldl'`. |
| `pipe.append` | `value -> stage` | Append a value to the list. |
| `pipe.for` | `(entries -> value) -> stage` | Replace the list with an arbitrary value. At most one per pipe per scope; multiple is an error with provenance (`__pipePolicyName`). |

### Multi-policy composition

Multiple policies can target the same pipe in the same scope:

- **No `pipe.for`:** Each `pipe.from` effect runs independently on the base pool. Results are concatenated.
- **With `pipe.for`:** At most one `pipe.for` effect per pipe per scope. If present, only that effect's stages run. Multiple `pipe.for` effects throw with policy names for diagnostics.
- **Different `pipe.to` targets:** Each aspect receives only its respective effect's output.
- **Same `pipe.to` target:** Results concatenate for that aspect.

---

## 6. Cross-Scope Operations

### `pipe.collect`

```nix
pipe.collect (predicate) -> stage
```

Harvests quirk data from sibling scopes matching the predicate. Non-terminal -- transform stages after `pipe.collect` operate on the combined pool (local + collected).

The predicate receives each sibling scope's context. Entity-kind filtering is implicit in the destructuring:

```nix
# Collect from host scopes only
(pipe.collect ({ host, ... }: host.datacenter == receiver.datacenter))

# Collect from all host scopes
(pipe.collect ({ host, ... }: true))
```

Implementation (`findMatchingSiblings` in `assemble-pipes.nix`):

1. Find all scopes sharing the same parent via `scopeParent`.
2. Exclude the current scope (self-exclusion).
3. Extract entity kinds from the predicate's required function args.
4. Reject scopes with extra entity kinds beyond what the predicate requires. A predicate `{ host, ... }:` matches only host-level scopes, even though user scopes also have `host` in their context -- the bidirectional entity-kind filter rejects user scopes because they have the extra `user` entity kind.
5. Verify all required args are present in the candidate scope's context.
6. Call the predicate; include scope if it returns `true`.

For config-dependent entries in collected scopes, thunks are resolved eagerly using `hostConfigs` -- a set of lazily-instantiated NixOS configs per scope (see Section 7).

### `pipe.expose`

```nix
pipe.expose -> stage
```

Terminal routing stage. Pushes pipe data up one level to the parent scope. Transform stages before `pipe.expose` are applied first.

Assembly processes expose effects bottom-up via `collectAllExposed`:

1. Find root scopes (no parent or self-parented).
2. Recursively process children before parents.
3. For each scope with expose effects, apply transform stages to combined local + child-exposed data.
4. Merge result into the parent scope's exposed pool.
5. During final assembly (pass 2), exposed data merges into the parent scope's combined base alongside local quirks.

Exposed data is visible only to the parent scope, not siblings.

### `pipe.to`

```nix
pipe.to [ aspects ] -> stage
```

Terminal routing stage. Narrows delivery to specific aspects within the current scope. Aspects are identified by their full identity pathkey via `den.lib.aspects.fx.identity.key`.

### `pipe.withProvenance`

```nix
pipe.withProvenance -> stage
```

Wraps each entry as `{ value, source }` where `source` is the scope context of the emitting scope. When present, the entire stage pipeline operates on internally-tagged values (`{ __pv = value; __ps = scopeId; }`). The `withProvenance` stage converts these to the user-visible `{ value, source }` format. Filter and transform stages between `collect` and `withProvenance` operate on the inner `__pv` value transparently.

```nix
(pipe.from den.quirks.http-backends [
  (pipe.collect ({ host, ... }: true))
  (pipe.filter (b: b.port != 8080))     # operates on __pv internally
  pipe.withProvenance                     # converts to { value, source }
  (pipe.transform (e: e.value // { from = e.source.host.name; }))
])
```

Works on local quirks too -- every entry has a source scope.

---

## 7. Config Thunks

A quirk value that is a function with `config` in its args is a config-dependent thunk:

```nix
den.aspects.nginx-backend = {
  http-backends = { config, ... }: [{
    addr = config.networking.hostName;
    port = config.services.nginx.defaultHTTPListenPort;
  }];
};
```

### Detection

`isConfigDependent` in `assemble-pipes.nix`: a value is config-dependent if `builtins.isFunction val && (builtins.functionArgs val) ? config`.

### Resolution: two paths

**Local thunks** (same-host): Marked with `{ __configThunk = true; __fn = v; }` during `assemblePipes`. Resolved inside `wrapClassModule` using the `evalModules` fixpoint `config`. This avoids a circular dependency: `assemblePipes` does not need the host's config to produce local pipe data.

**Collected thunks** (cross-host via `pipe.collect`): Resolved eagerly in `collectFromPeers` using `hostConfigs` -- a set of lazily-instantiated NixOS configs built in `resolve.nix`. These configs use the original (non-augmented) scope contexts, breaking the cycle: `assemblePipes -> hostConfigs -> evalModules -> pipe data`.

### Thunk function args

Config thunks can take both pipeline args and `config`:

```nix
host-info = { host, config, ... }: {
  ${host.name} = config.networking.hostName;
};
```

The resolver provides scope context args (`host`, `user`, etc.) alongside `config` and `lib`.

### Mutual dependencies

Two hosts can read each other's config-dependent quirks as long as they access different attributes. Nix evaluates attributes lazily -- `HOST-A.config.services.X` and `HOST-B.config.networking.Y` are separate thunks. Infinite recursion occurs only when the exact same attribute transitively depends on itself across hosts.

---

## 8. Assembly

`assemblePipes` in `assemble-pipes.nix` runs before the existing post-pipeline phases. It takes `scopeContexts`, `scopedClassImports`, `scopedPipeEffects`, `scopeParent`, and `hostConfigs`, and returns augmented scope contexts with pipe data added as keys.

### Pipeline position

```
assemblePipes       → augmented scope contexts with pipe data
wrapPerScope        → class wrapping (ctx now includes pipe data)
drain deferred      → pipe-arg deferred includes resolved
applyProvides       → policy.provide injection
applyRoutes         → forwards consume from scopedClassImports
applyInstantiates   → entity instantiation
```

### Two-pass algorithm

**Pass 1 (`collectAllExposed`):** Bottom-up traversal collecting exposed data from child scopes. For each scope with `pipe.expose` effects, applies transform stages to combined local + child-exposed data and merges result into parent scope's pool.

**Pass 2 (`mapAttrs`):** For each scope and each declared pipe:

1. Read raw entries from `scopedClassImports.${scope}.${pipeName}`.
2. Flatten and extract values (`flattenAndExtract`).
3. Mark local config thunks for deferred resolution (`markConfigThunks`).
4. Merge exposed data from children.
5. Partition scope effects into untargeted and targeted.
6. Apply untargeted effects via `applyPipeEffects` (or pass through combined base if none).
7. Build `__pipeTargeted` from targeted effects via `buildTargetedData`.
8. Set `__pipeConfigThunks` flags for pipes containing config thunk markers.
9. Merge all pipe data into the scope context attrset.

### Early exit

If `pipeNames == []` (no pipes declared), `assemblePipes` returns `scopeContexts` unchanged.

---

## 9. Invariants

- **Pipes are not classes.** `den.classes` and `den.quirks` must not share keys. Enforced by assertion in the `den.quirks` option `apply` function.
- **Bidirectional entity-kind filter.** `pipe.collect` predicates match scopes by entity kind via function arg destructuring. A `{ host, ... }:` predicate matches only host-level scopes -- user scopes are rejected even though they have `host` in context, because they carry extra entity kinds (`user`) not required by the predicate.
- **Self-exclusion.** `findMatchingSiblings` always excludes the current scope from collection. A host never collects from itself.
- **Sibling-only collection.** `pipe.collect` only finds scopes sharing the same parent. Cross-level collection requires `pipe.expose` to push data up first.
- **Routing kind transparency.** `pipe.to` and `pipe.expose` are terminal stages. `pipe.collect` is non-terminal -- transforms after it operate on the combined pool.
- **`pipe.for` singularity.** At most one `pipe.for` per pipe per scope. Multiple is a hard error with policy name provenance.
- **Empty pipes default to `[]`.** Consumers always receive a list (or a `pipe.for` result), never missing/null.
- **Config thunk cycle avoidance.** Local thunks are marked and resolved in `evalModules`; cross-host thunks use pipe-data-free `hostConfigs`. Neither path creates a circular dependency with `assemblePipes`.

---

## 10. Key Files

| File | Purpose |
|---|---|
| `modules/options.nix` | `den.quirks` option declaration, collision assertion, `name` injection |
| `nix/lib/policy-effects.nix` | `pipe.*` effect constructors (`pipe.from`, `pipe.filter`, etc.) |
| `nix/lib/aspects/fx/assemble-pipes.nix` | Post-pipeline pipe assembly engine (632 lines) |
| `nix/lib/aspects/fx/key-classification.nix` | `classifyKeys` with `pipeKeys` category |
| `nix/lib/aspects/fx/handlers/bind.nix` | Pipe arg detection and deferral in `bind` handler |
| `nix/lib/aspects/fx/handlers/register-pipe-effect.nix` | Collects pipe effects into `scopedPipeEffects` |
| `nix/lib/aspects/fx/class-module.nix` | Config thunk marker resolution in `wrapClassModule` |
| `nix/lib/aspects/fx/wrap-classes.nix` | `applyPipeTargeting` for `pipe.to` delivery |
| `nix/lib/aspects/fx/resolve.nix` | `assemblePipes` integration, `hostConfigs` construction, post-assembly drain |
| `nix/lib/aspects/fx/policy/classify.nix` | Pipe effect classification and `__pipePolicyName` tagging |

### Test suites

| File | Coverage |
|---|---|
| `templates/ci/modules/features/pipes.nix` | Declaration, classification, local consumption, list flattening, discriminator deferral, empty pipes |
| `templates/ci/modules/features/pipe-policy.nix` | `pipe.filter`, `pipe.transform`, `pipe.fold`, `pipe.append`, `pipe.for`, combined stages, multi-policy merge, `pipe.to`, ref syntax |
| `templates/ci/modules/features/pipe-scope.nix` | `pipe.expose`, `pipe.collect`, self-exclusion, entity-kind filtering, fleet collection, `pipe.withProvenance`, config thunks, mutual config dependencies |

---

## 11. Design Decisions

- **Reuse `scopedClassImports`.** Pipe keys flow through `emit-class` into the existing scope-partitioned collection infrastructure. No separate collection bucket or emission handler.
- **`den.quirks` not `den.pipes`.** The option is `den.quirks` (the data declaration); `pipe` is reserved for the policy builder API (`pipe.from`, `pipe.filter`, etc.). Quirk = data emitted by an aspect. Pipe = the policy-mediated channel.
- **No schema validation.** Pipes do not validate quirk shape. Quirks are arbitrary data; consumers and policies transform as needed.
- **Always a list.** Quirks are always collected as a list. No `list`/`map`/`single` collection strategies. `pipe.for` and `pipe.fold` handle reduction when needed.
- **Two-path config thunk resolution.** Local thunks are marked and resolved inside `evalModules` to avoid circular deps. Cross-host thunks use pipe-data-free `hostConfigs` for eager resolution during assembly. This split eliminates the cycle `assemblePipes -> hostConfigs -> evalModules -> pipe data`.
- **Bidirectional entity-kind filter on `pipe.collect`.** Predicate function args determine which entity kinds are accepted. Scopes with extra entity kinds beyond what the predicate requires are rejected. This prevents user scopes from leaking into host-level collection.
- **Non-terminal `pipe.collect`.** Unlike `pipe.to` and `pipe.expose`, `pipe.collect` allows subsequent transform stages. This enables collect-then-filter-then-expose patterns.
- **Provenance is opt-in.** Consumers see flat quirk values by default. `pipe.withProvenance` opts into `{ value, source }` wrapping when source context is needed.
- **No `_den.traits` injection.** Pipe data is delivered via `wrapClassModule` pre-application and `bind` handlers, not through generated NixOS options.
- **Static registration.** All pipes are declared at `den.quirks` before the pipeline starts. No dynamic `register-pipe` effect during the walk.

---

*Cross-references: [scope-partitioning.md](scope-partitioning.md) for scope mechanics and `scopeParent`; [policy-system.md](policy-system.md) for pipe effect dispatch and `scopedPipeEffects` collection.*
