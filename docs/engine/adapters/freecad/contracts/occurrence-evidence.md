# Occurrence Evidence Contract

This contract defines the Engine-owned occurrence request and raw occurrence
evidence consumed by Engine normalization. It is part of the FreeCAD observation
handoff; the current FreeCAD runtime does not yet emit this sidecar.

## Transport

- Engine writes `prm.occurrence-request.json` into the prepared working copy.
  No additional process argument is introduced.
- FreeCAD writes optional `prm.occurrence-evidence.json` under the validated
  `--output-dir`.
- Engine reads only that confined evidence path, validates it, and preserves its
  exact bytes as raw evidence.
- Omitted optional evidence means `unavailable`; it never means an inspected
  empty occurrence scope.

## Request — schema 1.0

```json
{
  "schemaVersion": "1.0",
  "requirement": "optional | complete",
  "scope": {
    "parent": {
      "resource": {"projectId": "...", "kind": "model", "logicalId": "..."},
      "nativeKind": "object",
      "name": "...",
      "semanticId": "..."
    },
    "context": [
      {"parent": {"resource": {}, "nativeKind": "object", "name": "...", "semanticId": "..."}, "member": "..."}
    ],
    "mechanism": "freecad-1.1.4-assembly-group-individual-link"
  }
}
```

- `parent` is a resource-qualified native object. It is not a generic
  reference and is not inferred from a semantic name.
- `context` is a bounded enclosing instance coordinate of at most 16 frames.
  It is context only, not a certificate of recursive membership or complete
  recursive expansion.
- The only currently supported mechanism is
  `freecad-1.1.4-assembly-group-individual-link`, a direct member of the
  explicitly requested Assembly Group. Unsupported mechanisms remain explicit;
  they do not degrade into generic reference counts.
- `requirement` is `optional` or `complete`. A complete request requires complete
  accepted evidence or fails deterministically.

## Evidence — schema 1.0 / 1.1

```json
{
  "schemaVersion": "1.0 | 1.1",
  "kind": "native_occurrence_evidence",
  "freecadVersion": "1.1.4",
  "scope": { "...": "echo of the requested scope" },
  "parent": {
    "entity": {"resource": {}, "nativeKind": "object", "name": "...", "semanticId": "..."},
    "document": "attempt-relative/materialized/coordinate.FCStd",
    "nativeType": "Assembly::AssemblyObject"
  },
  "completeness": "complete | incomplete | unsupported | unavailable",
  "inspectionCompleted": true,
  "group": ["Third", "First", "Second"],
  "members": [
    {
      "name": "First",
      "parent": {"resource": {}, "nativeKind": "object", "name": "...", "semanticId": "..."},
      "nativeType": "App::Link",
      "elementCount": 0,
      "membershipCorroborated": true,
      "classification": "occurrence | unresolved | unsupported | auxiliary",
      "resolution": "resolved | unresolved | missing | skipped | failed",
      "nativeStateValid": true,
      "target": {
        "entity": {"resource": {}, "nativeKind": "object", "name": "...", "semanticId": "..."},
        "document": "attempt-relative/materialized/coordinate.FCStd",
        "nativeType": "PartDesign::Body"
      },
      "diagnostic": "required only for unexplained unavailable/unsupported state"
    }
  ],
  "diagnostics": []
}
```

- `group` is the entire native Group inventory, not a filtered list of
  recognized links. Every entry must be represented in `members`; an omitted
  member is invalid. Ordering in the raw file is not semantic; Engine normalizes
  ordering deterministically.
- `members[].target` is required where a native target is resolved and carries
  both explicit logical resource identity and the attempt-relative materialized
  coordinate. Both must join the same declared inventory binding.
- `completeness` is independent of resolution states. Complete zero-member scope
  is a valid inspected empty scope; incomplete, unsupported, or unavailable
  scope must not claim quantity.
- Evidence schema `1.1` adds optional `contents` for an explicitly inspected
  auxiliary empty joint group. Schema `1.0` rejects that field.

## Engine-derived quantity

Engine derives occurrence identity and quantity only from accepted explicit
member evidence in a complete scope. Generic reference cardinality, semantic
component quantities, `elementCount`, and CAD BOM totals are not quantity
sources. The adapter never reports a normalized quantity or BOM.

## Raw evidence and provenance

The exact evidence bytes are preserved at
`raw/runtime/prm.occurrence-evidence.json` for a single-outcome package, or
under `raw/outcomes/<outcomeKey>/runtime/prm.occurrence-evidence.json` for a
multi-outcome package. Their SHA-256 supplies normalized evidence provenance.
The adapter does not write normalized records.

## Validation boundary

Engine rejects unsupported versions, unknown fields, duplicate JSON keys,
trailing content, scope mismatch, undeclared document coordinates, missing
parent evidence, incomplete member coverage, contradictory state, unproven
mechanisms, and complete-request noncompleteness deterministically. These
failures carry the occurrence contract classification rather than becoming
generic runtime failures.

See [record contracts — explicit occurrence observation](../../../reference/record-contracts.md#explicit-occurrence-observation)
for normalized occurrence and quantity semantics.
