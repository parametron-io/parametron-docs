# FreeCAD Adapter Architecture

FreeCAD is the currently implemented Engine CAD adapter. This document
describes FreeCAD's architecture behind the [Engine adapter
boundary](../README.md) at a conceptual level: role, ownership split, runtime
boundary, and capability areas. Detailed lifecycle steps live in
[runtime.md](runtime.md); detailed transport/schema semantics live under
[contracts/](contracts/).

## Role

`parametron-freecad` is the FreeCAD runtime adapter behind Engine-owned
contracts. It opens CAD documents, applies requested mutations, recomputes,
persists the native working document, exports declared artifacts, observes
requested state, and returns raw runtime evidence. Engine normalizes that
evidence into Engine-produced records; see [Record
Contracts](../../reference/record-contracts.md).

## Ownership Boundary

Engine owns input preparation, invocation policy, evidence validation,
expected-versus-observed verification decisions, and normalization. FreeCAD
owns CAD-native execution and raw evidence capture:

| Area | Engine | FreeCAD |
| --- | --- | --- |
| Planning, manifest projection, working-copy preparation | Owns | — |
| Runtime invocation, evidence intake, verification decisions | Owns | — |
| Document lifecycle (open, mutate, recompute, save, close) | — | Owns |
| Requested observation and reference discovery | — | Owns |
| Artifact export (STEP/CSV/PDF) | — | Owns |
| Normalized engineering records, durable storage | Owns | — |

FreeCAD does not own Engine planning, verification decisions, normalization,
higher-level orchestration, or durable product storage and indexing. See the
[Engine Adapter Domain](../README.md) and [Repository
Boundaries](../../../architecture/repo-boundaries.md) for the cross-repository
version of this rule.

## Runtime Boundary

Engine and local callers invoke the installable `parametron-freecad`
executable with explicit working-copy, manifest, and result paths. FreeCAD
validates those paths, performs FreeCAD-specific execution or observation, and
returns raw runtime evidence through deterministic file-based contracts:

```text
Engine / local caller
  -> parametron-freecad (launcher)
      -> FreeCAD host
          -> parametron_freecad.runtime.headless.main
              -> CAD-native execution and observation
```

The launcher forwards arguments as individual `--pass=<argument>` elements
rather than an interpolated shell command, and removes checkout/cwd knowledge
from callers. The full argument contract, host resolution order, and
containment rules are owned by [runtime.md](runtime.md).

## Attempt Identity and Prepared Working Copy

CAD execution can mutate documents, generate outputs, or fail partway through,
so Engine runs every FreeCAD invocation against an isolated attempt working
copy rather than the source document directly. Engine computes a deterministic
attempt identity from contract version, job, product, step, adapter, plan
hash, source path, declared output filenames, and a 1-based attempt index, and
prepares an isolated workspace at `<product-dir>/_working/<attempt-id>/` before
invoking FreeCAD.

The complete working-copy directory layout, request materialization files, and
runtime output paths are canonically defined and owned by [Execution Runtime —
Attempt Identity and Layout](../../runtime/execution-runtime.md#attempt-identity-and-layout).
Individual contract file semantics, including traversal request and output files,
are owned by their respective contract documents (such as
[contracts/reference-traversal.md](contracts/reference-traversal.md)).

This preparation and identity computation is Engine-owned, adapter-specific
work. FreeCAD's side of this boundary is validation, not construction: it
treats the supplied `--working-copy` directory as the sole authoritative,
exactly-contained execution-instance root for one invocation, and rejects
paths that resolve outside it, including sibling attempt directories under a
shared parent. See [runtime.md — Working-copy / execution-root
contract](runtime.md#working-copy--execution-root-contract) for FreeCAD's
containment rules.

The original source document is never the mutation target; Engine's isolated
working copy is. Failed runs leave the source document unchanged.

## Capability Areas

FreeCAD's implementation is organized around four conceptual capability areas
behind the same launcher entry point:

- **Launcher/CLI**: process bootstrap, host resolution, structured argument
  forwarding, and deterministic exit/stdout/stderr propagation.
- **Execution**: manifest loading and validation, parameter assignment,
  recompute, native document persistence, and STEP/CSV/PDF export. See
  [contracts/manifest.md](contracts/manifest.md) and
  [contracts/artifacts.md](contracts/artifacts.md).
- **Observation**: requested parameter, metadata, and reference-existence
  reads from the already-mutated, recomputed, and persisted document. See
  [contracts/observation.md](contracts/observation.md).
- **Reference traversal**: relationship-driven discovery of internal object
  references and Engine-mapped external targets, returned as raw dependency-
  graph evidence. See
  [contracts/reference-traversal.md](contracts/reference-traversal.md).

Manifest schema 2.0 also defines target-mutation (suppress/unsuppress/hide/
unhide/delete) contract metadata and strict validation, but the current
runtime does not execute schema 2.0 manifests. A focused FreeCAD-native
suppression/unsuppression consumer is implemented and tested independently,
but it is not connected to the schema 2.0 execute path. Native visibility and
deletion consumers remain unimplemented. See
[contracts/target-mutations.md](contracts/target-mutations.md) for the exact
boundary between validation, native capability, and execute integration, and
[Target-Action Contract](../../reference/target-action-contract.md) for
Engine's semantic ownership of target actions.

## Document Lifecycle

FreeCAD executes one ordered lifecycle per `execute` call: validate arguments,
resolve the manifest and source document, open the document, apply parameter
assignments, recompute, persist the native document, export declared
artifacts, run reference traversal, run observation, close the document, and
write a deterministic result. [runtime.md](runtime.md) owns the full ordered
lifecycle and failure-stage detail; this document does not repeat it.

## Observation and Reference-Traversal Ownership

FreeCAD gathers native observed facts and raw reference-graph evidence from
the live document; it does not evaluate checks, compare expected-versus-
observed state, or make verification decisions. Engine owns those decisions
and the normalization of accepted evidence into records. Requested reference
*observation* (existence-checking of explicitly named objects) is distinct
from reference *traversal* (relationship-driven dependency-graph discovery);
[contracts/observation.md](contracts/observation.md) and
[contracts/reference-traversal.md](contracts/reference-traversal.md) are the
canonical detailed owners of that distinction and its request/output shapes.

## GUI and Capture Boundary

GUI/capture functionality is not implemented in the current FreeCAD adapter.

## C++ Boundary

There is no active C++ runtime, native extension, or C++ build integration in
the current FreeCAD adapter.

## Related Documents

- [Engine Adapter Domain](../README.md) — generic adapter contract and
  ownership rules.
- [Execution Runtime](../../runtime/execution-runtime.md) — Engine's CAD
  runtime processing pipeline, attempt isolation, and verification.
- [Record Contracts](../../reference/record-contracts.md) — normalized record
  families and packaging.
- [Target-Action Contract](../../reference/target-action-contract.md) —
  Engine's semantic target-action ownership.
