# Parametron System Overview

This document describes the current public engineering system: the deterministic
**Engine + FreeCAD** local/headless pipeline.

## Primary Flow

```text
project / DSL / tables
  -> Engine (validate / plan / execute)
      -> CAD adapter
          -> FreeCAD where selected
      -> runtime results
  -> Engine-normalized records / local outputs
```

Engine is the entry point. A caller provides a project (DSL, tables, CAD
targets); Engine validates it, plans the work, and executes it. When CAD
execution is required, Engine invokes a CAD adapter behind an Engine-owned
contract. FreeCAD is the adapter/runtime used where it is the configured
runtime. The adapter returns raw runtime results; Engine normalizes them into
engineering records and writes a local record/artifact package.

## Roles

### Engine

Engine is the deterministic engineering execution layer. It owns:

- DSL parsing, DSL/semantic/table validation
- IR generation and planning
- simulation and deterministic execution
- adapter contracts and CAD capability invocation
- observation, mutation, export, and reference graph capabilities
- normalization of runtime results
- normalized engineering record production and deterministic failure records
- local record/artifact package production for standalone use

Engine runs standalone, locally and headlessly. It must remain usable without
any external service and must not require one to parse, validate, plan,
simulate, or execute local projects.

### CAD Adapter

A CAD adapter is a runtime-specific execution layer behind Engine contracts. It
may start CAD runtimes, open documents, observe parameters and metadata,
traverse references, mutate documents, recompute models, export geometry and
drawings, and capture runtime errors. It returns raw runtime results to Engine
and does not perform normalization, workflow, or durable storage.

### FreeCAD

FreeCAD is one CAD adapter/runtime implementation. It performs CAD-native
operations behind Engine-owned contracts and returns deterministic raw runtime
results. Engine + FreeCAD are the immediate public engineering implementation
focus.

## Standalone Local/Headless Usage

Engine is used as a standalone CLI/API tool for:

- local DSL authoring, validation, and simulation
- local deterministic execution and CAD automation
- development, debugging, and CI experiments

Standalone execution produces a local record package and any declared derived
artifacts. Those local outputs are not durable history in any external system
unless that system later ingests them through its own workflow, which is outside
the scope of this repository.

## Isolated Runtime Working Copy

CAD execution may mutate documents, generate outputs, or fail partway through.
Engine therefore runs CAD work against an isolated working copy:

```text
source document
  -> Engine prepares an isolated working copy
  -> FreeCAD runtime operates on the working copy
  -> configured native CAD working document is persisted
  -> original source document remains unchanged
```

Failed runs leave the source document unchanged. Successful runs produce
records, any declared derived artifacts, and the persisted native working
document.

## Deterministic Records and Results

Determinism is a core requirement and a tested property, not a descriptive
label. It applies to:

- Engine planning and execution records
- reference graph records
- artifact identity
- validation and failure records
- local record packages produced by standalone Engine

Engine preserves determinism by validating inputs strictly, using stable
planning and execution rules, normalizing runtime results, producing explicit
failure records, avoiding adapter-specific nondeterminism in normalized
outputs, and recording provenance. CAD adapters preserve determinism by
returning stable runtime result contracts, reporting explicit runtime errors,
and not letting nondeterministic runtime behavior leak into normalized records
without classification.

Determinism must be demonstrated through repeatable tests with explicit
expected outcomes.

## Summary

```text
Engine       = deterministic engineering validation, planning, execution, and
               normalized record production
CAD adapters = runtime-specific CAD operations behind Engine contracts
FreeCAD      = one CAD adapter/runtime implementation

Engine runs standalone, locally and headlessly.
CAD work runs against an isolated working copy; the source document is never
modified.
Determinism is verified through tests.
```
