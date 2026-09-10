# Parametron Repository Boundaries

This document defines the ownership boundaries for the current public
engineering components: **Engine** and **CAD adapters**, with **FreeCAD** as
the first adapter implementation.

The goal is to prevent responsibility drift: no repository grows into another
repository's role.

## Current Public Engineering Model

```text
project / DSL / tables
  -> Engine
      -> CAD adapter
          -> FreeCAD where selected
  -> Engine-normalized records / local outputs
```

Engine runs standalone, locally and headlessly. It does not require any
external service to validate, plan, or execute local projects.

## Engine

Engine owns:

- DSL and table validation
- planning
- deterministic engineering execution
- adapter contracts
- CAD capability invocation (observation, mutation, export, reference graph)
- normalization of CAD runtime results
- normalized engineering record production
- deterministic failure classification
- local record/artifact package production for standalone use

Engine does not own:

- runtime-specific CAD execution
- durable product storage
- product workflow and policy
- user or permission management

Engine may execute engineering work requested by a standalone local caller. It
does not decide product workflow policy.

## CAD Adapters

CAD adapters are runtime-specific execution layers behind Engine-owned
contracts. `parametron-freecad` is one adapter/runtime implementation; other
adapters may exist in the future.

CAD adapters own:

- CAD runtime bootstrapping and compatibility checks
- document opening, observation, and reference traversal through runtime APIs
- document mutation and recompute
- geometry and drawing export
- runtime-specific error capture
- returning raw runtime results to Engine

CAD adapters do not own:

- Engine planning
- workflow or policy
- durable product storage
- normalized durable record contracts beyond runtime result contracts

CAD adapters return raw runtime results to Engine. Engine normalizes those
results into engineering records.

## FreeCAD

FreeCAD is one CAD adapter/runtime implementation. It executes CAD-native
operations behind Engine-owned contracts and returns raw runtime results.
Engine + FreeCAD are the immediate public engineering implementation focus.

## Normalized Records

Engine is the producer of engineering facts: execution, artifact, observation,
parameter, reference graph, BOM/component, failure, and verification records.

In standalone use, Engine writes a local record/artifact package. This output
is described as *PDM-ready* where that reflects existing Engine contracts.
Engine-produced records may later be consumed by other systems; those systems
are outside the scope of this repository.

## Determinism Boundaries

Determinism is a tested property, not a descriptive claim.

- Engine must ensure deterministic validation, planning, execution records,
  failure classification, normalized adapter outputs, stable artifact identity,
  explicit provenance, and deterministic local record/artifact packages where
  claimed.
- CAD adapters must ensure stable runtime result contracts where possible,
  explicit runtime errors, no hidden direct writes outside the Engine contract,
  and no unclassified nondeterministic output leaking into normalized records.

## Cross-Repository Dependencies

Cross-repository dependencies are explicit. The issue lives in the repository
that owns the work, linked through GitHub's native blocked-by / blocking issue
relationships and marked with the `external` label on the dependent issue.

Do not create a placeholder issue in a repository that does not own the work,
and do not implement another repository's responsibility locally to unblock a
phase. Use fixtures, mocks, or documented contracts instead.

See [Development Workflow](../development/workflow.md) for the full coordination
rules.

## Summary

```text
Engine       = deterministic engineering execution and normalized records
CAD adapters = runtime-specific CAD operations behind Engine contracts
FreeCAD      = one CAD adapter/runtime implementation
```

No repository grows into another repository's role. Engine produces
engineering facts; CAD adapters perform CAD runtime operations; determinism is
verified through tests.
