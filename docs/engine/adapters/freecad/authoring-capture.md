# FreeCAD Capture Authoring

This document describes the implemented FreeCAD capture authoring surface:
how an existing FreeCAD project source is turned into Engine-consumable
`prm.cad.json` and `prm.semantic-map.json` files, and what it deliberately does
not do. It covers the current implemented behavior only.

Capture is separate from runtime execution. It does not run Engine, execute
engineering work, mutate the source document, or create a project.

## Ownership Boundary

Engine owns semantic validation, planning, verification decisions, and
normalized engineering records. FreeCAD owns native inspection and the raw
capture files.

| Area | Engine | FreeCAD |
| --- | --- | --- |
| Semantic-map schema, capture schema, and their interpretation | Owns | — |
| Project planning, verification decisions, normalized records | Owns | — |
| `prm.project.json` selection semantics and project resource identity | Owns | Reads |
| Native discovery of components, features, groups, parameters, metadata, and technical drawing pages | — | Owns |
| Drawing grouping, sheet order, drawing component linkage, and drawing resource topology | — | Owns |
| Semantic-name and stable-ID policy for captured entities | — | Owns |
| `prm.cad.json` generation, canonical ordering, and writing | — | Owns |
| `prm.semantic-map.json` generation, merge, and writing | — | Owns |
| Drift detection and deterministic capture previews | — | Owns |
| Optional Engine CLI validation preflight invocation | — | Invokes |

FreeCAD validates capture invariants and produces contract-shaped raw files.
It does not duplicate Engine planning, semantic resolution, verification, or
normalized-record generation.

## Project Context and Selection

Capture requires an existing `prm.project.json`. The mapping is input-only:
capture never creates, repairs, or modifies it.

- The mapping is validated against the Engine-owned project mapping contract
  (closed fields, version `1.0`, logical IDs, project-relative paths).
- A model logical ID must be selected explicitly. Capture does not guess a
  model from the active document.
- A coordinate-only lookup by project-relative path fails when more than one
  logical ID aliases that path.
- Declared paths must stay within the project root; absolute paths, current-
  directory paths, and `..` escapes are rejected.

The project root is the directory containing `prm.project.json`. Both capture
outputs are written there.

## Source Verification and Fingerprint

Capture operates on an already-open FreeCAD document and verifies that it is
the declared source before any discovery:

- the document must have a saved file path;
- the saved path must resolve to the selected model's declared source path;
- the document must not have unsaved changes.

The captured `sourceDocument.fingerprint` is a deterministic `sha256:<hex>`
over the saved source bytes. Capture never saves, recomputes, or otherwise
mutates the source document, so the fingerprint always describes persisted
native state.

## Native Discovery

Discovery is read-only and top-down from the document root. It produces raw
candidates without semantic names or stable IDs:

- components from supported `App::Part`, `Assembly::AssemblyObject`, and
  `PartDesign::Body` containers, with a synthetic document-level root assembly
  when no unambiguous assembly root exists;
- features from supported PartDesign features and sketches;
- parameter groups from `App::VarSet`;
- parameters from supported native scalar properties on captured groups;
- metadata from existing properties grouped under `Metadata` with a supported
  native type;
- technical drawing pages from active-document `TechDraw::DrawPage` objects,
  with exact native page selectors, native `cadType`, descriptive page labels,
  and a one-based page-order signal;
- drawing targets: captured components and features resolved from supported
  `Source` and `XSource` evidence on page views, traversing
  `TechDraw::DrawProjGroup` and `TechDraw::DrawViewClip` containers;
- captured targetability (suppress, unsuppress, hide, unhide, delete) from
  native property evidence, and conservative deletion dependency evidence from
  native `InList`/`OutList`.

Only pages owned by the active document become drawing candidates. A page
reached through a link from a separate `.FCStd` resource is diagnosed instead
of captured, because active-document capture cannot prove its contents.

Links, assembly members that are not direct supported objects, joints, object
groups, spreadsheets, unsupported TechDraw view subtypes, orphan TechDraw
objects, linked or separate-document drawing resources, unresolvable drawing
sources, and unknown native types produce diagnostics. They are not silently
represented as captured entities.

## Top-Down Capture Model

The FreeCAD workbench presents capture as five top-down tabs. The same state
model is available GUI-free for headless callers and tests.

| Tab | Captured content |
| --- | --- |
| Part/Assembly | Root component, parts, and assemblies; subset selection; synthetic root naming; explicit semantic names. |
| Features | Features owned by captured components; captured target actions; authorized-action narrowing; capability and delete-dependency evidence. |
| Parameters | Supported native properties on captured owners; explicit semantic names; semantic type and unit selection; observable/writable state. |
| Metadata | Existing supported native properties on captured owners; supported-type filtering; read-only and unsupported states. Metadata is never created. |
| Drawings | Discovered technical drawing pages grouped into user-authored logical drawings; sheet inclusion, grouping, and order; explicit semantic names; drawing and sheet ID previews; captured-target diagnostics. |

The Features, Parameters, Metadata, and Drawings tabs remain unavailable until a
part or assembly is captured and named. Disabled rows carry a user-facing
reason.

## Semantic Names

Semantic names are user-authored and validated as DSL identifiers:

- nonblank, no leading or trailing whitespace, no NUL;
- first character is a letter; remaining characters are letters, decimal
  digits, or `_`;
- exact and case-sensitive; no trimming, case folding, or normalization.

Capture never derives a semantic name from a native object name, label, CAD
type, geometry, or native reference. Names are unique across captured
components and features, and separately unique across captured parameters.
Drawing names are unique across captured drawings, and sheet names are unique
within their drawing; the same sheet name may appear in different drawings. A
collision in any namespace is a hard failure; no automatic suffix is added.

## Stable Identifiers

Stable IDs are generated deterministically from the semantic name and the
structural scope:

| Entity | Identifier |
| --- | --- |
| Component | `cmp.<nameSlug>` |
| Feature | `fea.<componentSlug>.<nameSlug>` |
| Parameter group | `grp.<ownerSlug>.<nameSlug>` |
| Component/feature parameter | `par.<ownerSlug>.<nameSlug>` |
| Group-owned parameter | `par.<ownerSlug>.<groupSlug>.<nameSlug>` |
| Metadata | `meta.<ownerSlug>.<keySlug>` |
| Drawing | `drw.<nameSlug>` |
| Drawing sheet | `sht.<drawingSlug>.<nameSlug>` |

For a drawing sheet, `<drawingSlug>` is the final dot-delimited segment of the
owning drawing ID, so `drw.manufacturing` gives
`sht.manufacturing.<nameSlug>`.

An explicit ID override is accepted instead of a generated ID. It must be exact
and unique, but it need not follow the generation convention. Generated or
explicit ID collisions are hard failures. IDs are never derived from native
references.

## Technical Drawings

Technical drawings are captured as first-class `entities.drawings` entries. A
drawing is a logical grouping, not a native object:

- each drawing has an explicit semantic name, an optional display name, and a
  stable ID;
- each drawing carries an ordered `sheets` array, where one sheet represents
  one captured native page with `name`, `displayName`, a positive unique
  `order`, the native `cadType`, and a required `identitySource`;
- `componentIds` is the deterministic union of captured components referenced
  by the drawing's sheets. A feature view target resolves to its owning
  captured component.

Sheets bind to native evidence through `identitySource.kind: "native_ref"` and a
nonblank `identitySource.nativeRef` that holds the exact native page selector.
FreeCAD does not emit a drawing-level `identitySource`, and it does not emit
drawing or sheet `annotations`, `targetability`, or `stabilityClass`.

The Engine contract models storage topology with
`resource.topology: "embedded" | "separate"`. `"embedded"` forbids a non-empty
`resource.path`; `"separate"` requires a canonical project-relative `path`.

FreeCAD v1.0 emits embedded drawings only:

- every discovered page candidate is an active-document page, so a captured
  drawing always declares `resource: {"topology": "embedded"}`;
- selecting a page whose topology is not `embedded` is rejected;
- pages linked from a separate `.FCStd` resource remain discovery diagnostics
  and are never converted into candidates or invented into `prm.cad.json`.

The native page label is descriptive only. It becomes the default sheet
`displayName` and is never used as identity, a lookup key, or a `nativeRef`
substitute. The native page-order signal only defaults the sheet `order`; the
captured `order` is user-authored and must stay positive and unique within its
drawing.

Capture enforces these drawing invariants:

- a discovered page is selected by at most one drawing;
- every drawing selects at least one sheet and references at least one captured
  component;
- every referenced component target must be captured, and a feature target
  requires its owning component to be captured;
- unknown or stale page selectors, duplicate sheet selection, invalid or
  duplicate order, and name or ID collisions fail deterministically;
- canonical output sorts drawings by ID, `componentIds` lexicographically, and
  sheets by `order` then `id`.

Zero drawings are emitted as `"drawings": []`; the registry is required and is
never omitted.

Drawings remain capture and semantic relationship state. They are not DSL
targets, planner inputs, execution targets, or reference-graph edges.

## Top-Down Scope and Uncaptured Objects

Capture is top-down and owner-closed:

- the root part/assembly must be captured;
- a captured child component requires its captured parent;
- a captured feature, parameter group, parameter, or metadata entry requires
  its captured owner;
- a captured technical drawing requires every referenced component target to be
  captured, and a feature view target requires its owning component to be
  captured;
- selecting an entity whose owner is not captured fails deterministically.

Uncaptured native objects are simply absent from the capture. Capture does not
emit placeholder entities, invent dependencies, or fill gaps for uncaptured
owners.

## Parameters and Metadata

Parameters are limited to supported native properties. Sketcher constraints
are not captured as parameters.

- Parameter semantic types and default units come from the project semantic
  map. When no project map exists, a conservative default covers only native
  scalar types with a safe semantic mapping; other native types remain
  unmapped and are shown as disabled.
- Captured parameters retain native observable/writable evidence, value type,
  current value, and native unit when present.

Metadata must already exist in the source document:

- only `App::PropertyString`, `App::PropertyBool`, `App::PropertyFloat`,
  `App::PropertyFloatConstraint`, `App::PropertyInteger`, and
  `App::PropertyIntegerConstraint` are authorable;
- unsupported native metadata types are shown as disabled and are rejected if
  selected;
- capture never creates missing metadata, property groups, aliases, or
  fallback properties.

## Semantic Map

Capture generates a versioned FreeCAD `prm.semantic-map.json` from an explicit
project selection:

- mapping identity and adapter/CAD-system identity;
- semantic parameter types with value kind, default unit, and supported
  coercion;
- native-type to semantic-type parameter mappings;
- resolution policy, projections, outputs, mutation collections, operation
  capabilities, override policy, and determinism sections required by the
  Engine contract.

The generated map is validated against the Engine semantic-map schema. When an
existing map is present, capture validates it, checks adapter identity, and
merges by replacing the project selection sections while preserving user edits
in adapter-policy sections. A differing existing map is never replaced without
an explicit confirmation decision.

## Preview, Drift, and Writing

Before writing, capture builds a deterministic preview for both output files.
The preview reports create, unchanged, or merge status and a stable structural
diff.

Capture fails closed on drift against retained capture:

- a retained native selector, including a drawing sheet page selector, that no
  longer resolves;
- a changed native type, including a changed page `cadType`;
- a changed ownership or hierarchy dependency, including changed sheet grouping
  or changed drawing component linkage;
- a semantic-name change that alters a generated drawing or sheet stable ID.

Drift requires rebind or review; it is never repaired silently. An intentional
sheet regroup or re-order is an authored decision rather than drift: it appears
as a structural diff in the preview and requires an explicit confirmation
before writing.

`prm.cad.json` and `prm.semantic-map.json` are written to the project root.
Each file is written atomically to a temporary file in the destination
directory, flushed and synced, then installed by atomic rename or no-replace
link. Existing differing files require an explicit confirmation decision; they
are never silently overwritten. Capture does not mutate the source document or
`prm.project.json`.

Equivalent unchanged captures are semantically deterministic: canonical
ordering, deterministic IDs, and byte-stable repeated output.

## Engine Acceptance

An opt-in real-FreeCAD proof runs the production capture path on temporary
copies of representative projects and feeds the generated `prm.project.json`,
`prm.cad.json`, and `prm.semantic-map.json` to the Engine CLI. The covered
capture shapes include zero drawings, a single drawing with one sheet, a single
drawing with multiple sheets, and multiple drawings, all produced from real
`TechDraw::DrawPage` fixtures. Engine accepts every result through normal
project-mode validation (`Validation succeeded`) without manual structural
repair, and repeated capture produces byte-identical `prm.cad.json` and
`prm.semantic-map.json` files. The proof also verifies that the source `.FCStd`
document and `prm.project.json` are unchanged, and it operates on temporary
copies so committed fixtures are never modified. Repository-local commands,
fixture digests, and the accepted Engine outcome are recorded in
`parametron-freecad`'s `docs/test-matrix.md`.

## Current v0 Limitations

Not implemented:

- drawing layout automation and PDF/SVG/PNG drawing export;
- separate `.FCStd` drawing resources and external linked documents;
- relationships/mates;
- a GUI execution trigger for Engine runs;
- project bootstrap or `prm.project.json` creation/modification;
- the `parametron/` folder layout;
- metadata creation;
- automatic semantic names.

Metadata authoring policy, semantic validation, planning, verification, and
normalized records remain Engine-owned.

## Related Documents

- [FreeCAD Adapter Architecture](architecture.md) — role, ownership boundary,
  and capability areas.
- [FreeCAD Runtime](runtime.md) — headless execution lifecycle and invocation.
- [CAD Capture Contract](../../authoring/cad-contract.md) — Engine-owned
  `prm.cad.json` schema.
- [Project Mapping](../../authoring/project-mapping.md) — Engine-owned
  `prm.project.json` contract.
- [Target-Action Contract](../../reference/target-action-contract.md) —
  Engine's target-action and runtime-handoff semantics.
