# Manifest Contract

The manifest is the file-based description of one execution: source document,
parameter assignments, declared exports, and optional target mutations.

Transport filename:

```text
prm.export-manifest.json
```

The transport filename and schema versioning are separate concepts; there is
no `export_manifest_v2.json` filename.

## Canonical Engine-authored manifest contract (Schema 1.0)

Engine authors and projects a single canonical manifest schema: `1.0`. All
Engine-authored manifests use `schemaVersion: "1.0"`, whether mutation-free or
mutation-bearing. Schema `2.0` is retired on the Engine side and is not an
Engine-supported transport contract.

Core top-level fields:

- `schemaVersion`
- `sourceDocument`
- `parameterAssignments`
- `outputs`

Optional top-level target-mutation fields:

- `assemblyMutations`
- `partMutations`

Both optional mutation sections use the same target-mutation structure, with
closed collections `suppression`, `visibility`, and `deletion`:

- `suppression[]` → `{object, suppressed}`
- `visibility[]` → `{object, visible}`
- `deletion[]` → `{object}`

Mutation sections never contain `parameters` or `properties`; scalar property
writes remain exclusively the job of `parameterAssignments`. Absent or empty
mutation sections and families are omitted in Engine projection.

Parameter assignment entry fields: `target`, `value`, `valueKind`.

Output declaration entry fields: `id`, `format`, `path`. Supported formats:
`csv`, `pdf`, `step`. See [artifacts.md](artifacts.md) for export behavior.

`outputs: []` is valid and produces zero derived exports; see
[runtime.md](../runtime.md#native-persistence-vs-derived-artifacts).

`sourceDocument` is resolved inside the working copy; relative and absolute
paths are accepted only when they resolve inside it. Missing files and
traversal/symlink escapes are rejected before the FreeCAD document is opened.

Parameter mutation targets are exact `<ObjectName>.<PropertyName>` strings,
resolved only through `document.getObject(name)`. There is no Label lookup,
`document.Objects` fallback, unit conversion, spreadsheet cell semantics,
constraint/expression target support, or capture-backed target projection.

A transitional Engine-shaped compatibility path adapts Engine-authored
dot-form data (`export_manifest.v1.json` shape) into this schema, e.g.
`inputs.sourceModel -> sourceDocument`, `outputs[].type -> format`,
`outputs[].filename -> path`, `outputs[].object -> id`. It intentionally
rejects Engine name-only parameter assignments that lack explicit FreeCAD
target data.

## Current FreeCAD implementation and transitional alignment gap

`parametron-freecad` has not yet been aligned to consume mutation-bearing
canonical schema `1.0` manifests.

Current FreeCAD implementation state:

1. **Normal execute uses closed schema 1.0 validation**: Runtime execution
   loads and validates manifests through `validate_export_manifest_v1`. In that
   validator, top-level fields are closed (`schemaVersion`, `sourceDocument`,
   `parameterAssignments`, `outputs`); `assemblyMutations` and `partMutations`
   are rejected as unknown fields.
2. **Transitional schema 2.0 metadata and validator**: FreeCAD still retains
   standalone schema `2.0` contract metadata and a strict validator
   (`validate_export_manifest_v2`) from an earlier pre-release split.
3. **Normal execute rejects schema 2.0**: The normal external
   `parametron-freecad execute` entrypoint explicitly rejects schema `2.0`
   manifests.
4. **Standalone native capabilities are not connected**: Standalone native
   suppression, visibility, and deletion consumers and post-mutation
   validity/dependency inspection infrastructure are implemented and tested
   independently, but they are not wired into normal `execute`.
5. **Execute alignment is pending**: Normal FreeCAD execute does not gain
   mutation support merely because Engine's schema has been consolidated to
   schema `1.0`. FreeCAD still requires its own follow-up alignment before
   mutation-bearing canonical schema `1.0` can be consumed through normal
   execute.

FreeCAD's existing schema-2 metadata and validator represent a **transitional
implementation artifact of the earlier pre-release split**, not the active
Engine contract. Engine does not author or accept schema `2.0`.

### Transitional FreeCAD validation rules

FreeCAD's standalone manifest validation infrastructure implements the
following rules (currently defined under its transitional schema-2 validator,
awaiting alignment into the normal execute path):

- **Exact version dispatch** — `"1.0"` and `"2.0"` are matched by exact string
  equality in the standalone validator dispatch; no trimming, case folding, or
  numeric coercion. Normal execute enforces schema 1.0.
- **Closed core schema 1.0** — under current FreeCAD schema 1.0 validation,
  `assemblyMutations`/`partMutations` are rejected as unknown fields.
- **Optional mutation sections** — absent, `{}`, or sparse valid collections
  are accepted where mutation sections are supported.
- **Closed mutation collections** — only `suppression`, `visibility`,
  `deletion` are recognized; anything else (`parameters`, `keep`, `actions`,
  `targets`, …) is rejected.
- **Strict entry shape** — unknown/extra entry fields (`force`, `cascade`,
  `recursive`, `dependencyPolicy`, `repair`, `action`, `targetKind`, …) are
  rejected.
- **Strict JSON booleans** — `suppressed`/`visible` must satisfy
  `type(value) is bool`; numeric or string coercions (`0`, `1`, `"true"`) are
  rejected.
- **Duplicate rejection** — the same object twice within one scope/family
  (e.g. `Pad` twice in `partMutations.suppression`) is rejected.
- **Deletion conflicts** — suppression+deletion or visibility+deletion on the
  same object within one scope is rejected.
- **Suppression+visibility is valid** — these are independent semantic axes
  and may coexist on the same object within one scope.
- **Cross-scope conflict rejection** — the same object may not appear in both
  `assemblyMutations` and `partMutations` across any family.
- **No string normalization** — object-name comparisons are exact string
  equality; no trimming, case folding, or CAD lookup.
- **Deterministic diagnostics** — diagnostics sort assembly before part, then
  suppression → visibility → deletion, preserving array order; input is never
  mutated.

## Ownership

Engine owns manifest authoring and canonical schema `1.0` projection, including
the semantic intent behind optional target mutations — see [Target-Action
Contract](../../../reference/target-action-contract.md).

FreeCAD owns manifest loading, strict validation, and native execution. Its
current normal execute path consumes the existing schema-1 core surface,
including parameter assignments and outputs, but does not yet consume the
optional target-mutation sections of the canonical Engine schema-1 contract.

FreeCAD also preserves ownership of its document lifecycle, recompute, native
document persistence, exports, reference traversal, observation, and native
result/failure evidence — see [runtime.md](../runtime.md) for the currently wired
schema 1.0 execution lifecycle. The standalone native suppression, visibility,
and conservative deletion consumers are downstream capabilities, supported by
standalone deterministic post-mutation validity/dependency infrastructure, and
remain outside the normal execute lifecycle until follow-up FreeCAD alignment.
