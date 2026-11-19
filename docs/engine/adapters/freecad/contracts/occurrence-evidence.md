# Occurrence Evidence Contract

This contract defines the Engine-owned occurrence request and raw occurrence
evidence consumed by Engine normalization. `parametron-freecad` implements the
producer described below for the bounded mechanism; Engine remains the owner of
the contract and of all derived occurrence identity and quantity.

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
      "resolution": "resolved | unresolved | unsupported | unavailable",
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
- `resolution` is a closed vocabulary of `resolved`, `unresolved`,
  `unsupported`, and `unavailable`. It is not the generic reference-traversal
  resolution vocabulary; native states such as absent, skipped, or failed are
  not passed through. `classification: "occurrence"` and
  `classification: "auxiliary"` require `resolution: "resolved"`;
  `classification: "unresolved"` requires `unresolved` or `unavailable`; and
  `classification: "unsupported"` requires `unsupported`. Every unresolved or
  unsupported member carries a diagnostic.
- `completeness` is independent of resolution states. Complete zero-member scope
  is a valid inspected empty scope; incomplete, unsupported, or unavailable
  scope must not claim quantity.
- Evidence schema `1.1` adds optional `contents` for an explicitly inspected
  auxiliary empty joint group. Schema `1.0` rejects that field.

## FreeCAD runtime support

The runtime consumes the request and emits the evidence without a new CLI
argument:

- `prm.occurrence-request.json` is discovered only at the working-copy root as a
  plain, non-symlink regular file. The request is strictly decoded before the
  document is opened.
- `prm.occurrence-evidence.json` is written atomically under the validated
  `--output-dir`, which is required and must resolve inside the working copy.
  A stale sidecar for an enabled occurrence request is removed before request
  loading.
- Inspection runs on the same open document that was mutated, recomputed, and
  persisted, after optional observation and before document close. Emission
  failures use runtime failure stage `occurrence_evidence`; request-discovery
  and decoding failures use `occurrence_request` and occur before document
  open.

The runtime always emits evidence schema `1.1`, using `contents` only for an
inspected auxiliary empty joint group. It does not emit a quantity, an
Engine-normalized record, or a generic reference count.

### Supported scope

Inspection is `complete`-capable only when all of the following hold:

- the requested mechanism is
  `freecad-1.1.4-assembly-group-individual-link`;
- the running FreeCAD version token is exactly `1.1.4`;
- the requested occurrence context is empty.

Otherwise the runtime emits `completeness: "unsupported"` with
`inspectionCompleted: false`, empty `group` and `members`, and an explicit
diagnostic; it does not expand a non-empty context or degrade to reference
counting. The request `requirement` is enforced by Engine intake, not by the
adapter.

The parent is resolved by exact native object name. A missing parent yields an
`unavailable` observation; a resolved parent that is not an
`Assembly::AssemblyObject` is reported as `unsupported`. When a declared
resource inventory is present, the requested parent must be the selected source
resource of the attempt.

### Group inventory and member classification

The runtime reads the complete native `Assembly::AssemblyObject.Group`
inventory, orders it by exact native object name, rejects duplicate native names
in one group, and requires each entry to corroborate parent membership through
both `Group` and `InList`. A dangling entry, duplicate identity, or
uncorroborated membership is a native access failure, not a silently omitted
member.

Every member is classified with the bounded rules below:

| Native member state | Classification | Resolution |
| --- | --- | --- |
| `App::Link` with `ElementCount` 0 and a target resolved through a declared resource to `PartDesign::Body`, `Part::Feature`, or `App::Part` | `occurrence` | `resolved` |
| `App::Link` whose target document or object cannot be resolved | `unresolved` | `unresolved` or `unavailable` |
| Empty `Assembly::JointGroup` with inspected empty contents | `auxiliary` | `resolved` |
| Non-empty `Assembly::JointGroup` | `unsupported` | `unsupported` |
| Nested `Assembly::AssemblyObject` or `Assembly::AssemblyLink` | `unsupported` | `unsupported` |
| `App::LinkElement` or `App::Link` with non-zero `ElementCount` | `unsupported` | `unsupported` |
| Unsupported target or member native type | `unsupported` | `unsupported` |

`completeness` is `complete` when every inventory entry is handled and
`incomplete` when any entry is `unresolved` or `unsupported`. A complete
zero-member scope is a valid inspected empty scope. Unhandled members retain an
explicit diagnostic and never become fabricated occurrences.

### Declared target binding

Occurrence targets are joined to declared resource identity through the attempt's
integrity inventory. A native target document joins only when its resolved
`FileName` is absolute, matches exactly one declared materialized coordinate,
and exists as a regular file. A declared file that is missing produces member
`resolution: "unresolved"`; undeclared, ambiguous, or otherwise unavailable
native documents produce `resolution: "unavailable"`. Every such member carries
a diagnostic and classification `unresolved`. The runtime never guesses a
logical ID, rewrites a native link, or derives identity from a path spelling.

Without an integrity request, only the selected source coordinate is known. A
member pointing at any other document is reported as unresolved with the
diagnostic that the declared resource inventory is unavailable.

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
parent evidence, incomplete member coverage, unsupported member resolution
states, contradictory classification/resolution state, unproven mechanisms,
and complete-request noncompleteness deterministically. These
failures carry the occurrence contract classification rather than becoming
generic runtime failures.

See [record contracts — explicit occurrence observation](../../../reference/record-contracts.md#explicit-occurrence-observation)
for normalized occurrence and quantity semantics.
