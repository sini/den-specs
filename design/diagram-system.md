# Den Diagram Generation System

Design spec for the diagram subsystem on `feat/fx-pipeline`, documented from the implementation.

## Overview

Den's diagram system is a composable Nix pipeline that captures structured trace events from the fx-pipeline's aspect resolution, builds a format-agnostic graph intermediate representation (IR), applies filters and reshaping transforms, and renders the result into multiple diagram formats. The entire system is pure Nix -- no external build tools are needed until the final SVG conversion step.

The pipeline has four stages:

1. **Trace capture** -- collect structured trace entries from aspect resolution
2. **Graph construction** -- build a format-agnostic IR from those entries
3. **Filtering** -- prune, fold, reshape, diff the IR
4. **Rendering** -- emit Mermaid, Graphviz DOT, PlantUML, or C4 strings

## 1. Structured Trace: How Events Are Captured

### Trace Handler (`nix/lib/aspects/fx/trace.nix`)

The fx-pipeline uses algebraic effect handlers for aspect resolution. The trace system hooks into this by providing additional handlers that intercept `resolve-complete` effects -- the event fired when an aspect finishes resolution.

Two handler variants exist:

- **`structuredTraceHandler`** -- Minimal handler that accumulates entries without disambiguating anonymous nodes. Used in tests that verify basic parent/entry structure.

- **`tracingHandler`** -- Full handler used for diagram generation. Handles anonymous entry disambiguation by deriving human-readable names from entity kind, context aspect, and provider path. For example, an anonymous resolution node might be named `host/resolve(desktop):provider` instead of `<anon>`.

Both handlers intercept `resolve-complete` and produce entry records containing:

| Field | Source | Description |
|-------|--------|-------------|
| `name` | `param.name` or derived | Aspect name (disambiguated for anon entries) |
| `class` | handler arg | Which class trace this belongs to (nixos, homeManager, etc.) |
| `parent` | includes chain | Nearest meaningful ancestor in the chain |
| `provider` | `param.meta.provider` | Provider path segments |
| `excluded` | `param.meta.excluded` | Whether this aspect was excluded by a constraint |
| `excludedFrom` | `param.meta.excludedFrom` | Which adapter caused the exclusion |
| `replacedBy` | `param.meta.replacedBy` | What replaced this aspect |
| `isProvider` | derived | Whether `provider != []` |
| `handlers` | `param.meta.handleWith` | Resolution handlers attached to this aspect |
| `hasClass` | `param ? ${class}` | Whether this aspect has content for the traced class |
| `isParametric` | `param.meta.isParametric` | Whether the aspect takes function arguments |
| `fnArgNames` | `param.meta.fnArgNames` | Names of the function parameters |
| `entityKind` | `param.__entityKind` or derived | Entity kind (host, user, etc.) |

Key implementation details:

- **Parent derivation**: Uses the scoped includes chain, filtering out self-references. Finds the nearest meaningful ancestor, skipping anonymous intermediates.

- **Entity kind derivation**: Walks the includes chain upward through accumulated entries to find the nearest ancestor with a non-null `entityKind`. O(chain x entries) -- acceptable for diagnostic-only paths.

- **Anonymous disambiguation** in `tracingHandler`: Constraint-owner nodes become `filter:<owner>`. Other anonymous nodes become `<entityKind>/resolve(<ctxAspect>):<provPath>`.

### Trace Utilities (`nix/lib/aspects/fx/trace-util.nix`)

Two-tier trace output gated by the `DEN_TRACE` environment variable:

- `traceSummary` -- Always-on pipeline structure and count traces (scope inventory, module counts)
- `traceDetail` -- Gated behind `DEN_TRACE=1`, per-aspect/handler granularity (emit-class, check-dedup, ctx-seen, installPolicies)

These are runtime debug traces, not the structured capture used for diagrams.

## 2. Trace Event Types

The trace system captures events at the `resolve-complete` boundary -- every aspect that completes resolution generates an entry. The entry metadata encodes what happened during resolution:

| Metadata pattern | Meaning |
|-----------------|---------|
| `hasClass = true` | Aspect defines content for this class |
| `excluded = true` | Aspect was excluded by a constraint handler |
| `replacedBy != null` | Aspect was replaced by another |
| `handlers != []` | Aspect has resolution constraint handlers (adapters) |
| `isProvider = true` | Aspect is a provider sub-aspect |
| `isParametric = true` | Aspect takes function arguments (parametric resolution) |
| `isPolicyDispatch = true` | Node represents a policy dispatch point |
| `entityKind != null` | Aspect belongs to a specific entity kind (host, user, etc.) |

Cross-entity-kind relationships are captured through the includes chain and parent derivation. Entity kind transitions (e.g., host -> user) are detected when a child entry's entity kind differs from its parent's.

## 3. Diag Library Architecture

All files live under `nix/lib/diag/`. The entry point is `default.nix`.

### Core Files

| File | Purpose |
|------|---------|
| `default.nix` | Library entry point; imports all modules, composes the public API |
| `capture.nix` | Trace capture: runs the fx-pipeline with tracingHandler to collect entries |
| `graph.nix` | Graph IR construction: transforms entries into nodes, edges, entity kinds |
| `util.nix` | Pure data manipulation primitives shared across the library |
| `json.nix` | JSON serialization of the graph IR |

### Filter Pipeline (`filters/`)

| File | Purpose |
|------|---------|
| `filters/default.nix` | Barrel module; imports sub-modules, exports composed filters |
| `filters/predicate.nix` | Simple predicate filters: `userDeclaredOnly`, `pipelineOnly`, `crossClassOnly`, `orphansAndLeaves` |
| `filters/fold.nix` | Structural folds: `foldWrappers` (collapse pipeline plumbing), `foldProviders` (collapse sub-aspects into parents), `flattenEntityKinds` |
| `filters/reshape.nix` | Alternative graph structures: `contextOnly`, `aspectsOnly`, `providersOnly`, `decisionsView`, `providersResolved` |
| `filters/closure.nix` | Closure-based filters: `classSlice`, `neighborhoodOf`, `adaptersOnly`, `parametricOnly` |
| `filters/presence.nix` | `hasAspect` presence filters for per-class materialization analysis |
| `filters/diff.nix` | Graph diff: merge two graphs with origin tags (a-only, b-only, both) |

### Renderers

| File | Purpose |
|------|---------|
| `mermaid.nix` | Mermaid flowchart renderer (`graph LR`/`graph TD`) |
| `dot.nix` | Graphviz DOT renderer |
| `plantuml.nix` | PlantUML renderer |
| `c4.nix` | C4 model renderers (both PlantUML and Mermaid backends) |
| `sequence.nix` | Sequence diagram renderers (stage sequence, expanded, policy, stage topology) |
| `sankey.nix` | Sankey flow diagrams (per-host inclusion flow, fleet user->host, fan metrics) |
| `treemap.nix` | Treemap diagrams (provider groups, fleet provider usage, fleet provider matrix) |
| `mindmap.nix` | Mermaid mindmap (radial tree for provider hierarchy) |
| `state.nix` | State diagram (`stateDiagram-v2`) for context resolution pipeline |

### Context and Theming

| File | Purpose |
|------|---------|
| `context.nix` | Entity-agnostic context constructors: `context`, `hostContext`, `userContext`, `homeContext` |
| `fleet.nix` | Fleet-level data capture: iterates host registry, produces C4-compatible records |
| `namespace.nix` | Static namespace graph from `den.aspects` declarations (no host resolution) |
| `colors.nix` | Per-node color selection via MD5 hash into base16 accent pool |
| `themes.nix` | Theme records from base16 palettes; YAML->JSON conversion; mermaid frontmatter generation |
| `render-util.nix` | Shared renderer primitives: `renderMermaid`, `skinparamFor`, `visualFor`, `mkRenderer` |
| `render-context.nix` | Render context factory: bundles renderers, SVG builders, view definitions |
| `render-infra.nix` | Nix derivation builders for SVG conversion (mermaid-cli, plantuml, graphviz) |
| `export.nix` | Export helpers: turn views into derivation entries, build gallery markdown, write scripts |
| `views.nix` | Standard view definitions: core, extended, per-class, fleet |

## 4. Graph Construction

### Capture (`capture.nix`)

Capture runs the fx-pipeline with the tracing handler installed alongside the default handlers:

```
captureRaw = class: root:
  nxFx.handle {
    handlers = composeHandlers (defaultHandlers { class; ctx = {}; })
                               (tracingHandler class);
    state = defaultState // { entries = []; ctxTrace = []; };
  } (aspectToEffect root);
```

- `capture class root` -- returns entries for a single class
- `captureAll classes root` -- concatenates entries across multiple classes
- `captureWithPaths classes root` -- returns `{ entries, pathsByClass, ctxTrace }` with per-class path sets for presence analysis

### Graph IR (`graph.nix`)

`buildGraph { entries; rootName; ctxTrace?; direction?; }` transforms raw trace entries into a graph IR with this shape:

```nix
{
  rootName;    # Entity name (e.g., "laptop")
  rootId;      # Sanitized identifier
  direction;   # "LR" or "TD"
  nodes;       # [ { id, label, fullLabel, pathKey, shape, style, entityKind, classes, class, perClass, ... } ]
  edges;       # [ { from, to, style, label } ]
  entityKinds; # [ { id, name, ctxKeys } ]
  entityEdges; # [ { from, to, style, label } ] -- cross-entity-kind transitions
}
```

Key construction steps:

1. **Group entries by fullName** (provider-qualified path) to merge class info across multiple class traces
2. **Build per-class metadata** (`perClass` attrset): `hasClass`, `excluded`, `replacedBy`, `excludedFrom` per class
3. **Dedup nodes** by fullName; dedup edges by from->to
4. **Classify node shape**: hexagon (parametric), trapezoid (provider), rect (default)
5. **Classify node style**: replaced, excluded, adapter, policy, terminal, default
6. **Build display labels**: short form when unique, full path when colliding
7. **Build provider-provenance edges**: dotted links from sub-aspects back to provider source
8. **Build policy dispatch edges**: connect policy nodes to target entity kind subgraph
9. **Synthesize phantom provider stubs**: for providers referenced by chains but missing from the trace
10. **Detect entity kind transitions**: parent->child relationships crossing entity kind boundaries

### Node Schema

Every node carries the full `emptyNode` schema from `graph.nix`:

- `id` -- sanitized identifier safe for all renderers
- `label` -- short display label (auto-disambiguated)
- `fullLabel` -- full `provider/sub/.../name` path
- `pathKey` -- canonical key matching `identity.pathKey`
- `shape` -- `rect`, `hexagon`, `trapezoid`
- `style` -- `default`, `excluded`, `replaced`, `adapter`, `policy`, `terminal`
- `entityKind` -- entity kind name or null
- `classes` -- list of classes this aspect contributes to
- `perClass` -- per-class metadata attrset
- `isParametric`, `fnArgNames` -- parametric aspect info
- `isProvider`, `providerPath` -- provider chain info
- `isExcluded`, `isReplaced` -- structural booleans for filters
- `isPolicyDispatch`, `policyName`, `from`, `to` -- policy dispatch info

### Edge Styles

- `normal` -- standard inclusion edge
- `excluded` -- edge to/from an excluded node
- `replaced` -- edge with replacement annotation
- `provide` -- provider-provenance (dotted)
- `policy` -- policy dispatch arrow

## 5. Views

Views are defined in `views.nix` as records with `{ view, title, altText, mdLang, svgInfix, svgFn, compute }`. Templates compose view lists and pass them through the export system.

### Core Views (default for all entity types)

| View ID | Name | Description |
|---------|------|-------------|
| `aspects` | Aspect Hierarchy | User aspects with pipeline plumbing folded out, stage subgraphs retained |
| `stage-seq` | Stage Sequence | Compact sequence diagram showing entity kind resolution order |
| `stage-seq-full` | Stage Sequence (expanded) | Untruncated per-kind aspect lists with cross-kind bridges |
| `policy-seq` | Policy Sequence | Policies as participants, showing dispatch and chaining |
| `providers` | Provider Tree | Provider hierarchy as a proper multi-level tree |
| `ir` | Graph IR (JSON) | Raw graph IR serialized to JSON |

### Extended Views (opt-in)

| View ID | Name | Description |
|---------|------|-------------|
| `ctx` | Context Hierarchy | Entity kinds become nodes; aspect content discarded |
| `simple` | Simplified View | Providers folded, entity kinds flattened, aspects only |
| `stage-edges` | Stage Topology | Focused flowchart of entity kinds and transitions only |
| `providers-resolved` | Providers Resolved | Each provider alongside its resolved output nodes |
| `adapters` | Adapter Impact | Nodes with constraint handlers plus their neighbors |
| `decisions` | Structural Decisions | Excluded nodes grouped by constraint-owner attribution |
| `declared` | User-Declared Aspects | Only aspects where `hasClass = true` |
| `diff-classes` | Class Diff | Origin-tagged merge of two class slices |

### Dynamic Views

- **Per-class views** (`class-<className>`) -- Generated from the graph's available classes. Each gets a slice showing only aspects contributing to that class, with ancestor closure.

### Fleet Views

| View ID | Name | Description |
|---------|------|-------------|
| `namespace` | Aspect Namespace | Static declarations from `den.aspects` (no resolution) |
| `c4context` | Fleet C4 Context | PlantUML C4 Context across all hosts/users |
| `provider-matrix` | Fleet Provider Matrix | Bipartite graph of providers to hosts |

### DAG View

Every entity also gets a full DAG view (added by `export.entityEntries`) rendered as a Mermaid flowchart.

## 6. C4 Model Integration

The C4 model is fully integrated via `c4.nix` with three zoom levels:

### C4 Component (`toC4Component` / `toC4ComponentMermaid`)
Per-host view. The host becomes a `System_Boundary`. Each entity kind becomes a `Container_Boundary` holding aspects as `Component` elements. Inclusion relationships become `Rel` arrows.

### C4 Container (`toC4Container` / `toC4ContainerMermaid`)
Per-host view at a higher abstraction. Each entity kind becomes a `Container` with an aspect count. Entity kind transitions become `Rel` arrows.

### C4 Context (`toC4Context` / `toC4ContextMermaid`)
Fleet-wide view. Hosts become `System` elements, users become `Person` elements, and user-to-host class relationships become `Rel` arrows. Consumes fleet data (from `fleet.nix`), not per-host graph IR.

Both PlantUML (via C4 stdlib `!include <C4/C4_*>`) and Mermaid (native C4Context/C4Container/C4Component diagram types) backends share the same body-builder functions. Only the framing wrapper differs.

## 7. Output Formats

### Diagram Source Formats

| Format | Renderer | Used For |
|--------|----------|----------|
| Mermaid flowchart | `mermaid.nix` | Main graph visualization, all filter views |
| Mermaid sequenceDiagram | `sequence.nix` | Stage sequence, policy sequence |
| Mermaid sankey-beta | `sankey.nix` | Flow weight visualization |
| Mermaid treemap-beta | `treemap.nix` | Provider group hierarchy |
| Mermaid mindmap | `mindmap.nix` | Radial provider tree |
| Mermaid stateDiagram-v2 | `state.nix` | Context resolution state machine |
| Mermaid C4Component/C4Container/C4Context | `c4.nix` | C4 diagrams in Mermaid |
| Graphviz DOT | `dot.nix` | Alternative graph layout engine |
| PlantUML | `plantuml.nix` | Alternative diagram renderer |
| PlantUML C4 | `c4.nix` | C4 diagrams in PlantUML |
| JSON | `json.nix` | Machine-readable graph IR |

### SVG Conversion (`render-infra.nix`)

Three Nix derivation builders convert source to SVG:

- `mmdSourceToSvg` -- Uses `mermaid-cli` (mmdc) with Puppeteer. Produces a placeholder SVG on failure rather than failing the build.
- `pumlSourceToSvg` -- Uses `plantuml -tsvg -pipe`
- `dotSourceToSvg` -- Uses `graphviz dot -Tsvg`

All three configure font rendering via `makeFontsConf` with JetBrains Mono, Fira Code, DejaVu, Liberation, and Noto fonts.

### Theming

Themes are built from base16 color schemes (`pkgs.base16-schemes` ships ~300 schemes as YAML). The pipeline:

1. `paletteFromBase16 { pkgs; scheme; }` -- YAML to JSON via `yj`, parse to palette attrset
2. `themeFromPalette palette` -- Maps base16 slots to theme record fields
3. Theme record consumed by all renderers uniformly

A default github-light theme is hardcoded for zero-config use without `pkgs`.

Per-node colors are selected by hashing the node name and category (entity kind) into the theme's 8-slot accent pool (`base08`-`base0F`). Category biases the starting offset (multiplied by 7, coprime with pool size 8); name adds a small perturbation. This keeps related nodes in similar hues while ensuring distinction.

Mermaid theming uses the `%%{init: {...}}%%` directive with full `themeVariables` mapping from the theme record.

### Export and File Layout (`export.nix`)

The export system converts view definitions into derivation entries:

```
diagrams/
  hosts/<host>/         -- per-host views + DAG
  users/<host>-<user>/  -- per-user views
  homes/<home>/         -- per-home views
  fleet/                -- fleet-wide views
```

Each view produces:
- A `.md` file with heading, optional SVG embed, and fenced source code block
- An `.svg` file (when SVG conversion is available for the format)
- Gallery pages that collect all SVG embeds for a directory

`mkWriteScript` produces a shell script that copies all derivation outputs to a target directory.

## 8. How to Run Diagram Generation

The `templates/diagram-demo/` flake template demonstrates the full system:

```bash
# Generate and write all diagrams to the template directory
nix run --override-input den . ./templates/diagram-demo#write-diagrams

# Build a specific diagram package
nix build --override-input den . ./templates/diagram-demo#laptop-aspects-svg
```

The template (`modules/diagrams.nix`) does the following:

1. Configures a theme (`catppuccin-mocha` via base16)
2. Optionally patches mermaid-cli to a newer version for recent diagram types
3. Creates a render context with ELK layout for dense graphs
4. Iterates all hosts, users, and homes in the den configuration
5. Generates view definitions (core + class views) for each entity
6. Produces fleet-level views
7. Assembles all entries into packages and a `write-diagrams` script
8. Generates gallery markdown pages and a README

The render context (`diag.renderContext`) bundles theme, renderers, SVG builders, and view definitions into a single record that templates thread through the export pipeline.

## 9. Future Improvements

Three visualization capabilities are planned but not yet implemented:

### Trait Flow Visualization
Deferred until traits are reimplemented. Would show how trait definitions flow through the aspect resolution tree -- which aspects provide traits, which consume them, and how trait values are transformed by the resolution pipeline.

### Fleet Policy Topology
Would visualize the policy dispatch graph across an entire fleet -- which policies fire on which hosts, how policy routing decisions differ between entity kinds, and where policy chains create cross-host dependencies.

### Parametric Annotation
Would annotate diagram nodes with their parametric function signatures and show how parametric resolution creates entity-specific branches in the aspect tree. The infrastructure exists (nodes carry `fnArgNames` and `isParametric`; sequence diagrams already show parametric self-arrows) but a dedicated view focusing on parametric flow is not yet built.
