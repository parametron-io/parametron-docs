# Fact Evidence Contract

This contract defines the Engine-owned bounded engineering-fact request and raw
fact evidence consumed by Engine normalization. `parametron-freecad` implements
the producer described below; Engine remains the owner of the contract and of
all normalized fact semantics.

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

## FreeCAD runtime support

The runtime consumes the request and emits the evidence without a new CLI
argument:

- `prm.fact-request.json` is discovered only at the working-copy root as a
  plain, non-symlink regular file and is strictly decoded before the document
  is opened. The request is bounded to 32 bindings, property names up to 128
  characters, and unit tokens up to 32 characters.
- `prm.fact-evidence.json` is written atomically under the validated
  `--output-dir`, which is required and must resolve inside the working copy.
  A stale sidecar for an enabled fact request is removed before request
  loading.
- Observation runs on the same open document that was mutated, recomputed, and
  persisted, after optional observation and before document close. Emission
  failures use runtime failure stage `fact_evidence`; request-discovery and
  decoding failures use `fact_request` and occur before document open. A
  `required` binding that is not observed fails deterministically at
  `fact_evidence`.

Evidence is canonical schema `1.0`. Entries are ordered deterministically by
binding identity, so equivalent requests produce byte-identical evidence
regardless of request ordering.

### Subject and document access

The subject resource is joined to the attempt's integrity inventory by declared
`logicalId` and `kind`. With no integrity request, or when the subject resource
is not declared, every binding is reported as `unavailable` with an explicit
diagnostic; the runtime does not guess a resource.

A subject in the selected source document is read from the already-open
document. A subject in another declared resource is read from that resource's
declared materialized coordinate: the runtime opens that document read-only,
verifies that its native `FileName` resolves to exactly the declared path, and
closes it before the selected source document closes. It never mutates, saves,
rebinds, or recursively discovers another document.

### Typed values and explicit units

The runtime observes exactly the named native property; it does not enumerate
properties or infer a property from names.

| Requested `valueKind` | Accepted native value | Observed material |
| --- | --- | --- |
| `boolean` | Native boolean | Boolean, no unit |
| `string` | Native canonical UTF-8 string | String, no unit |
| `number` | Finite native number or unit-bearing quantity | Numeric value plus the matching explicit unit token, if declared |
| `integer` | Finite native number with no fractional part | Integer value plus the matching explicit unit token, if declared |

Unit handling is deliberately opaque and exact:

- a FreeCAD quantity's `Unit` is read as the bounded token in its `Unit: <token>`
  textual form;
- a declared request unit must equal the native unit token exactly, otherwise the
  fact is `unsupported`;
- a declared request unit combined with a unitless native value is `unsupported`;
- a unitless request combined with a unit-bearing native value is `unsupported`;
- no conversion, normalization, or unit algebra is performed.

Non-finite numeric values, boolean native values requested as numeric facts,
non-exact integers, non-canonical strings, and unsupported native shapes are
reported as `unsupported` rather than being coerced. A missing property, a
missing value, or a missing subject object is `unavailable`. Unobserved entries
never carry `value` or `valueKind` material and never become zero, empty, or
false.

The runtime implements no material database, material inference, density, or
physical mass. It never derives mass from geometry, density, or material
metadata. If Engine binds an existing native property that happens to represent
mass, the emitted fact is raw native property evidence only.

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
