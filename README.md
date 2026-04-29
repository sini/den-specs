# den-specs

Design specs and implementation audit for [den](https://github.com/denful/den), a NixOS configuration framework built on algebraic effects.

## What's here

This repo is the spec archive for den's `feat/fx-pipeline` branch — a ground-up rewrite of the aspect resolution pipeline using a freer monad effect system ([nix-effects](https://github.com/sini/nix-effects)).

- **`SUMMARY.md`** — Executive summary: overview table of all specs with verdicts, supersession chains, gap inventory, known bugs, and component summaries.
- **`DESIGN-incremental-audit.md`** — Repeatable process for analyzing new specs and integrating them into the summary.

## Directory structure

```
implemented-main/       Specs implemented on the main branch
  aspect-rewrites/
  fx-pipeline/
  hasAspect/

implemented-branch/     Specs implemented on feat/fx-pipeline
  class-modules/
  legacy-removal/
  pipeline-simplification/
  policies/
  provides-removal/
  stages/
  traits/

tbd/                    Specs not yet implemented
  diag/
  hasAspect/
  pipeline-state/
  provides-removal/
  unified-effects/
  vision/

cancelled/              Superseded or abandoned specs
  aspect-rewrites/
  fx-pipeline/
  policies/
  vision/
```

Each directory contains spec files (`*.md`) and their corresponding implementation analyses (`*.analysis.md`).

## Verdicts

Each spec is classified as one of:

| Verdict | Meaning |
|---------|---------|
| **implemented** | Shipped as designed (with possible minor drift) |
| **partially-implemented** | Core shipped, some features deferred or diverged |
| **superseded** | Replaced by a later spec before or after implementation |
| **not-implemented** | Abandoned or blocked before any code landed |
| **in-progress** | Actively being worked on |

See `SUMMARY.md` for the full breakdown.
