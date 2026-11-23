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
| Native discovery of components, features, groups, parameters, metadata | — | Owns |
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
- captured targetability (suppress, unsuppress, hide, unhide, delete) from
  native property evidence, and conservative deletion dependency evidence from
  native `InList`/`OutList`.

Links, assembly members that are not direct supported objects, joints, object
groups, spreadsheets, technical drawings, and unknown native types produce
diagnostics. They are not silently represented as captured entities.

## Four-Tab Capture Model

The FreeCAD workbench presents capture as four top-down tabs. The same state
model is available GUI-free for headless callers and tests.

| Tab | Captured content |
| --- | --- |
| Part/Assembly | Root component, parts, and assemblies; subset selection; synthetic root naming; explicit semantic names. |
| Features | Features owned by captured components; captured target actions; authorized-action narrowing; capability and delete-dependency evidence. |
| Parameters | Supported native properties on captured owners; explicit semantic names; semantic type and unit selection; observable/writable state. |
| Metadata | Existing supported native properties on captured owners; supported-type filtering; read-only and unsupported states. Metadata is never created. |

The Features, Parameters, and Metadata tabs remain unavailable until a part or
assembly is captured and named. Disabled rows carry a user-facing reason.

## Semantic Names

Semantic names are user-authored and validated as DSL identifiers:

- nonblank, no leading or trailing whitespace, no NUL;
- first character is a letter; remaining characters are letters, decimal
  digits, or `_`;
- exact and case-sensitive; no trimming, case folding, or normalization.

Capture never derives a semantic name from a native object name, label, CAD
type, geometry, or native reference. Names are unique across captured
components and features, and separately unique across captured parameters. A
collision is a hard failure; no automatic suffix is added.

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

An explicit ID override is accepted instead of a generated ID. It must be exact
and unique, but it need not follow the generation convention. Generated or
explicit ID collisions are hard failures. IDs are never derived from native
references.

## Top-Down Scope and Uncaptured Objects

Capture is top-down and owner-closed:

- the root part/assembly must be captured;
- a captured child component requires its captured parent;
- a captured feature, parameter group, parameter, or metadata entry requires
  its captured owner;
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

- a retained native selector that no longer resolves;
- a changed native type;
- a changed ownership or hierarchy dependency;
- a semantic-name change that alters a generated stable ID.

Drift requires rebind or review; it is never repaired silently.

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
`prm.cad.json`, and `prm.semantic-map.json` to the Engine CLI. Engine accepts
the result through normal project-mode validation without structural repair,
and repeated capture is byte-identical. Repository-local commands, fixture
digests, and the accepted Engine outcome are recorded in
`parametron-freecad`'s `docs/test-matrix.md`.

## Current v0 Limitations

Not implemented:

- technical drawings and drawing capture;
- relationships/mates;
- external linked documents;
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
