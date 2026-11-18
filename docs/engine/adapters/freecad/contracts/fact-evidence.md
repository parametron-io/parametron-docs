# Fact Evidence Contract

This contract defines the Engine-owned bounded engineering-fact request and raw
fact evidence consumed by Engine normalization. It is part of the FreeCAD
observation handoff; the current FreeCAD runtime does not yet emit this
sidecar.

## Transport

- Engine writes `prm.fact-request.json` into the prepared working copy. No
  additional process argument is introduced.
- FreeCAD writes optional `prm.fact-evidence.json` under the validated
  `--output-dir`.
- Engine reads only that confined evidence path, validates it against the
  explicit request, and preserves its exact bytes as raw evidence.
- The request sidecar is runtime-local attempt material and is not packaged.

## Request — schema 1.0

```json
{
  "schemaVersion": "1.0",
  "bindings": [
    {
      "kind": "bound_property",
      "subject": {
        "resource": {"projectId": "...", "kind": "model", "logicalId": "..."},
        "nativeKind": "object",
        "name": "...",
        "semanticId": "..."
      },
      "context": [
        {"parent": {"resource": {}, "nativeKind": "object", "name": "...", "semanticId": "..."}, "member": "..."}
      ],
      "property": "Length",
      "valueKind": "number | integer | string | boolean",
      "unit": "mm",
      "requirement": "optional | required"
    }
  ]
}
```

- `bound_property` is the only supported fact kind. The adapter observes exactly
  the explicitly named native property; it does not infer facts from property
  names or material names.
- `subject` is resource-qualified. `context` is an optional bounded occurrence
  context of at most 16 frames and does not authorize recursive expansion.
- `unit` is an opaque bounded token. It is required for meaningful numeric
  observations and forbidden for string/boolean values. Engine performs no unit
  conversion or unit algebra.
- A required binding that is unavailable or unsupported fails deterministically.

## Evidence — schema 1.0

```json
{
  "schemaVersion": "1.0",
  "kind": "native_fact_evidence",
  "freecadVersion": "1.1.4",
  "facts": [
    {
      "kind": "bound_property",
      "subject": {
        "entity": {"resource": {}, "nativeKind": "object", "name": "...", "semanticId": "..."},
        "document": "attempt-relative/materialized/coordinate.FCStd",
        "nativeType": "PartDesign::Body"
      },
      "context": [],
      "property": "Length",
      "valueKind": "number",
      "value": 42.5,
      "unit": "mm",
      "availability": "observed"
    },
    {
      "kind": "bound_property",
      "subject": {"entity": {}, "document": "...", "nativeType": "..."},
      "context": [],
      "property": "Density",
      "availability": "unavailable",
      "diagnostic": "controlled unavailable density observation"
    }
  ]
}
```

- Every requested binding must be covered exactly once. An evidence file that
  omits a requested binding is invalid; an omitted evidence file instead makes
  every requested optional binding explicitly unavailable.
- `availability` is `observed`, `unavailable`, or `unsupported`.
  - `observed` requires `valueKind`, a canonical JSON scalar `value`, and the
    matching explicit `unit` when the binding declares one.
  - `unavailable` and `unsupported` require a non-empty diagnostic and must not
    carry `value` or `valueKind`.
- `subject.document` is an attempt-relative declared-resource materialized
  coordinate and must join the same resource identity as `subject.entity`.
- Equivalent duplicate evidence canonicalizes; conflicting evidence for one
  binding fails deterministically.

## Engine-derived facts and boundaries

Engine normalizes accepted facts into `ObservationRecord.observation.facts[]` of
the bounded `fact` kind, preserving subject qualification, context, property,
value kind, value, unit, availability, and exact-evidence provenance. Facts do
not become mutation verification, quantity, BOM, material inference, or mass.
Unavailable or unsupported facts never become implicit zero, empty, or false.

## Raw evidence and provenance

The exact evidence bytes are preserved at
`raw/fact/prm.fact-evidence.json` for a single-outcome package, or under
`raw/outcomes/<outcomeKey>/fact/prm.fact-evidence.json` for a multi-outcome
package. Their SHA-256 supplies normalized fact provenance. The adapter does not
write normalized records.

## Validation boundary

Engine rejects unsupported versions, unknown fields, duplicate JSON keys,
trailing content, unsolicited evidence, uncovered requested bindings,
undeclared resource subjects, context or unit disagreement, non-canonical
scalars, invalid availability/value presence, and required unavailable facts
deterministically with the fact contract classification.

See [record contracts — bounded engineering fact observation](../../../reference/record-contracts.md#bounded-engineering-fact-observation)
for normalized fact semantics.
