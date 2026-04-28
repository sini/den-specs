# Prior Art: Entity-Type Registries with Policies and Resolution

**Date:** 2026-04-21 (revised)
**Companion to:** `2026-04-21-ctx-as-classes-design.md`

## The Problem Pattern

A framework needs to define entity types (Host, User, Home), their relationships (host has many users, user belongs to host), and how entity instances flow through a resolution pipeline that produces configuration output. How do existing systems separate these concerns?

## Den Terminology

Before drawing parallels, note Den's terminology (see main spec for full definitions):
- **Entity** — a tangible infrastructure thing (host, user, home)
- **Schema** — data structure defining what an entity IS (`den.schema.*` + submodule types)
- **Class** — a Nix configuration class (behavior): `nixos`, `darwin`, `homeManager`
- **Aspect** — container of behavior (Nix config classes)
- **Policy** — how entities transition into each other
- **`den.ctx`** — deprecated; conflated policies and behavior in a single construct. Being removed.

## Comparison Table

| System | Entity Definition | Relationships | Data vs Behavior | Resolution/Pipeline |
|---|---|---|---|---|
| **Rails AR** | Model class inherits `ApplicationRecord`; DB schema in migrations | Declarative macros: `has_many :users`, `belongs_to :host` | Schema = migrations (DDL), Behavior = model class | Lazy query builder; callbacks form lifecycle pipeline |
| **GraphQL** | `type Host { name: String! }` — pure data shape | Fields reference other types: `users: [User!]!` — graph edges are typed fields | Types = schema (SDL), Behavior = resolver functions wired separately | Resolver tree walks the query; each field has independent resolver |
| **Terraform** | `resource "aws_instance" "web" { ami = "..." }` — typed block with schema attributes | Implicit via attribute references (`vpc_id = aws_vpc.main.id`), explicit via `depends_on` | Schema = provider's resource schema, Behavior = provider's CRUD methods | DAG-based: builds dependency graph, walks topological order |
| **K8s CRD+Controller** | CRD defines `spec` (OpenAPI schema); instances are YAML docs | Owner references, label selectors — conventions, not schema-level | Spec = desired state (data), Controller = reconciliation loop (behavior) | Level-triggered: controller watches resources, diffs spec vs status, converges |
| **ECS** | Entity = bare ID. That's it. | No first-class relationships; entities store other entity IDs in components | Components = pure data structs, Systems = functions querying component sets | Systems iterate over entities matching component signature |
| **Django** | Model class with field declarations inline; `class Meta` for structural metadata | `ForeignKey(Host, on_delete=CASCADE)`, `ManyToManyField(User)` — on fields | Schema + relationships + behavior all in one class | Manager/QuerySet pipeline; middleware-like signals for lifecycle |
| **Haskell TC** | `data Host = Host { hostname :: Text }` — algebraic data type | Encoded in types: `[User]` field in Host | Data = type declaration, Behavior = typeclass instances | Typeclass dispatch; composable via constraints |

## Detailed Analysis

### Haskell Typeclasses — Closest Analog to Den's Three-Way Split

A `data` declaration defines structure. A `typeclass instance` declares capabilities separately. `instance Resolvable Host where resolve = ...` adds behavior without modifying the Host definition. Open for extension — new typeclasses can be defined and instances added.

**Den mapping:**
- Entity schema (`den.schema.host` + entity submodule) = `data` type declaration (what a host IS)
- Aspects with Nix config class keys (nixos, darwin) = typeclass instances (what a host CAN DO — behavior)
- Policies = typeclass constraints (`Resolvable a => Deployable a` — pipeline ordering)

**Takeaway:** Validates the three-way separation: entity schemas are pure data, aspects declare behavioral capabilities, policies constrain ordering. None of these belong in a single conflated construct (which is why `den.ctx` — merging policies and behavior — is being removed).

**Anti-pattern:** Orphan instances (defining behavior for a type in a third-party module) cause coherence problems. Den should ensure capability declarations are co-located or explicitly imported, never ambient.

### Kubernetes CRDs + Controllers — Spec/Status Split

CRDs define pure schema (what fields a resource has). Controllers provide reconciliation behavior (converge actual state toward desired state). They're separate processes — the API server stores data, controllers add behavior.

**Den mapping:**
- Entity schema options = CRD spec (declared shape)
- Resolution pipeline = controller (converges aspects into NixOS config)
- `config.resolved` = status (observed output)

**Takeaway:** Validates keeping entity schemas as pure data with no behavioral functor. Level-triggered reconciliation ("converge to desired state") is the right mental model for the resolution pipeline.

**Anti-pattern:** Relationships are an afterthought. Owner references are primitive. No way to declare "a Database requires a Cluster" at the schema level — controllers just fail at runtime if dependencies are missing. Den should make policies first-class.

### Rails ActiveRecord — Relationship Vocabulary

`has_many :users`, `belongs_to :host` — declarative macros that read as prose. Relationships are metadata, not behavior. Rails also separates schema (migrations) from the model class.

**Den mapping:**
- `den.policies.host-to-users = { from = "host"; to = "user"; ... }` follows the same declarative pattern
- The policy spec is metadata that the pipeline walks

**Takeaway:** The declarative relationship macros are the gold standard for readability. Rails' `through:` for indirect relationships maps to den's "host has homes through users" pattern.

**Anti-pattern:** Rails merges too much into model classes — god-object models with 500+ lines mixing query logic, validation, and callbacks. Den should resist adding behavior to policy declarations (the same trap `den.ctx` fell into).

### Terraform Providers/Resources — Implicit Relationship Detection

Terraform infers its dependency DAG from attribute references (`vpc_id = aws_vpc.main.id`). Explicit `depends_on` is the escape hatch. Most relationships are discovered, not declared.

**Den mapping:**
- Den could infer relationships from how entities reference each other (a home-manager config references a user which references a host)
- Explicit policies are the primary mechanism, structural inference complements them

**Takeaway:** Implicit detection from references is powerful. Den should support both implicit detection (per capabilities design) and explicit declaration (policies).

**Anti-pattern:** `depends_on` is error-prone when implicit detection fails. Implicit-only relationship detection can be too magical.

### Entity-Component-System (ECS) — Composition over Classification

Entity is NOTHING — just an ID. All meaning comes from which components are attached. A "Host" is an entity with `{HostConfig, NetworkConfig, StorageConfig}` components. Systems query by component signature.

**Den mapping:**
- An entity gains capabilities by which aspects are attached, not by its entity type
- Entity schemas define structural shape; actual configuration comes from aspect composition
- Entity types are constraints on valid compositions, not behavior prescriptions

**Takeaway:** Validates den's aspect model. The ECS framing confirms: identity comes from composition, not classification.

**Anti-pattern:** No schema enforcement — any component can be attached to any entity. Den needs structural constraints (a Home must have a User context).

### GraphQL Schema — Relationships as Typed Fields

Types declare shape. Relationships are just fields that return other types — no special `has_many` keyword. A `Host` type with field `users: [User!]!` IS the relationship declaration. The schema is a graph by construction.

**Den mapping:**
- The idea that relationships are "fields that return other types" is elegant
- Resolver-per-field means resolution logic is granular and independently testable

**Takeaway:** No special relationship DSL needed if the type system is expressive enough. But GraphQL's resolver explosion (every field needs a resolver) is excessive — den's pipeline-based resolution avoids this by resolving in defined phases.

### Django Models — The Meta Class Pattern

Everything in one class: fields, relationships, validators, managers, custom methods, and structural metadata (`class Meta`). The `Meta` inner class is interesting — metadata ABOUT the model (ordering, constraints, db_table name) structurally separate from fields and methods.

**Den mapping:**
- Entity definitions could have a "meta" section for structural metadata (capabilities, pipeline ordering) separate from the fields/aspects
- Django's Manager pattern (customizing how entities are queried/resolved) maps to den's adapter concept

**Anti-pattern:** The monolithic model. Django models become god-objects even faster than Rails. `den.ctx` exhibited the same problem — conflating policies and behavior into one construct.

## Universal Insight

Every system that ages well separates three concerns:

1. **What an entity IS** (structure/schema) — `den.schema.*` + entity type definitions
2. **How entities RELATE** (transitions) — `den.policies`
3. **How entities RESOLVE** (behavior) — `den.aspects` + fx pipeline handlers

Systems that merge any two develop god-object problems. Den's `den.ctx` merged policies and behavior — the redesign separates them.

The closest formal model is Haskell typeclasses:
- `data` type = entity schema (pure structure)
- typeclass instance = aspect with Nix config class keys (behavioral capability)
- typeclass constraint = policy (ordering/dependency)
