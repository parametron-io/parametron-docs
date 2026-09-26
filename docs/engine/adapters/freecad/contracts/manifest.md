# Manifest Contract

The manifest is the file-based description of one execution: source document,
parameter assignments, declared exports, and optional target mutations.

Transport filename:

```text
prm.export-manifest.json
```

The transport filename does not encode a schema generation.

## Canonical Engine-authored manifest contract (Schema 1.0)

Engine authors and projects a single canonical manifest schema: `1.0`. All
Engine-authored manifests use `schemaVersion: "1.0"`, whether mutation-free or
mutation-bearing. FreeCAD validates and consumes the same schema in normal
execution. The experimental pre-release schema `2.0` split is retired and is
not an active runtime compatibility contract.

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

## Current FreeCAD runtime behavior

Normal `parametron-freecad execute` validates and consumes canonical schema
`1.0` directly, including optional mutation sections and their strict mutation
rules. Unsupported schema versions are rejected.

The legacy Engine-manifest normalization helper remains for the rehearsal /
compatibility surface. Normal execute does not import or invoke it.

### Strict mutation validation rules

FreeCAD's canonical schema 1.0 validator applies the following rules in normal
execute:

- **Exact version validation** — `"1.0"` is matched by exact string equality;
  no trimming, case folding, or numeric coercion.
- **Canonical schema 1.0 mutation sections** — normal execute accepts optional
  `assemblyMutations` and `partMutations`; each section is closed to
  `suppression`, `visibility`, and `deletion`.
- **Optional mutation sections** — absent, `{}`, or sparse valid collections
  are accepted.
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

FreeCAD owns manifest loading, strict validation, and native execution. Normal
execute consumes schema 1.0 parameter assignments, outputs, and optional
target-mutation sections directly.

FreeCAD also owns document lifecycle, recompute, native persistence, exports,
reference traversal, observation, and native result/failure evidence — see
[runtime.md](../runtime.md) for the schema 1.0 execution lifecycle.
