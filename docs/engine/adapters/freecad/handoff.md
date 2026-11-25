# FreeCAD Handoff — Engine Observation and Provenance Phase

This document is the canonical handoff from the completed Engine observation
phase to the following `parametron-freecad` phase. It states what the FreeCAD
runtime must implement, what it must not implement, and which Engine-owned
contract answers each adapter implementation question.

## Status

The Engine-owned contracts for declared resources, generic references,
explicit occurrences and quantity, bounded engineering facts, source lock and
configured-result provenance, and outcome-scoped package emission are complete.
The bounded FreeCAD capabilities they authorize are implemented in
`parametron-freecad` and covered by permanent focused, fake-FreeCAD, and
real-FreeCAD evidence:

- occurrence evidence for the supported direct-member mechanism;
- bounded engineering fact evidence;
- pre-mutation baseline and post-persistence configured-result evidence against
  the Engine integrity request;
- mutation of existing native metadata properties and exact-native metadata
  observation.

Generic reference traversal, observation, target mutations, artifacts, and
runtime result/failure keep their implemented FreeCAD contracts under
[contracts/](contracts/). The engineering-evidence surfaces consume the same
working-copy and raw-evidence boundary, introduce no command-line argument, and
do not create a second execution lifecycle. Their implemented transport and
timing are described by [runtime.md](runtime.md) and the contract documents
below.

## Adapter readiness questions

| Question | Canonical answer |
| --- | --- |
| What resource is being observed? | The declared `(projectId, kind=model, logicalId)` resource identity from project mapping, joined to an attempt-relative materialized coordinate. The runtime never invents or changes logical resource identity. |
| Which attempt-relative coordinate should the adapter use? | Coordinates are resolved inside the exact `--working-copy` root. The manifest identifies the selected source document; the integrity request identifies every declared resource coordinate for the attempt. |
| What constitutes a generic reference? | A native relationship discovered by reference traversal, reported through [contracts/reference-traversal.md](contracts/reference-traversal.md). Generic references are not occurrences and do not imply quantity. |
| What constitutes a supported occurrence? | A direct member of the explicitly requested Assembly Group under the supported mechanism `freecad-1.1.4-assembly-group-individual-link`. See [contracts/occurrence-evidence.md](contracts/occurrence-evidence.md). |
| How is occurrence completeness reported? | Explicit `completeness` as `complete`, `incomplete`, `unsupported`, or `unavailable`, with the complete native Group inventory and explicit diagnostics. Omitted optional evidence is unavailable, never an inspected empty scope. |
| What raw evidence is needed for Engine quantity rollup? | The complete evidence `group` plus one classified member entry per group name or auxiliary inventory. Engine derives quantity only from accepted complete occurrence members. |
| How are fact subjects and bindings requested? | Engine writes `prm.fact-request.json` with bounded `bound_property` bindings naming the resource-qualified subject, optional occurrence context, native property, value kind, explicit unit, and requirement. See [contracts/fact-evidence.md](contracts/fact-evidence.md). |
| How are typed values and units represented? | Observed facts carry a canonical JSON scalar in `value`, a closed `valueKind`, and an opaque bounded `unit` token. No unit conversion or unit algebra is implied. |
| How are unavailable or unsupported states reported? | Every requested fact binding receives an evidence entry with `availability` set to `observed`, `unavailable`, or `unsupported`. Unobserved entries carry a diagnostic and no value or value-kind material. |
| What baseline digest must be observed before mutation? | The integrity request's per-resource `baselineSha256`. The runtime reports the digest it actually observed before mutation through `prm.integrity-evidence.json`. See [contracts/integrity-request.md](contracts/integrity-request.md) and [contracts/integrity-evidence.md](contracts/integrity-evidence.md). |
| What digest identifies the persisted configured result? | The post-persistence `persistedResult.sha256` for the resource whose request entry has `expectsPersistedResult: true`, reported with its attempt-relative coordinate. It is never a pre-mutation baseline digest. |
| How is undeclared resource access represented? | Reference traversal may report an undeclared or unsupported target with its contract-relative coordinate; Engine retains explicit qualification and does not invent declared resource identity. Undeclared native access never becomes complete declared-resource coverage. |
| Which raw evidence files are returned? | `prm.result.json`, `prm.observed.json`, optional `prm.reference-traversal.json`, optional `prm.occurrence-evidence.json`, optional `prm.fact-evidence.json`, and optional `prm.integrity-evidence.json`, under the contract-defined working-copy or output-directory locations. |
| Which semantics remain exclusively Engine-owned? | Declared-resource joins, reference normalization, occurrence identity and quantity rollup, fact normalization, source-lock policy, baseline/result validation, verification decisions, deterministic failure classification, and normalized record/package emission. |

## Required FreeCAD responsibilities

The FreeCAD runtime implements bounded native work only:

1. **Declared resource access.** Use only Engine-provided attempt-relative
   coordinates. Report missing, unavailable, unsupported, or undeclared native
   access explicitly.
2. **Generic reference evidence.** Emit accepted native relationships through
   the existing reference-traversal contract, preserving the requested external
   targets, property/mechanism discriminators, and diagnostics.
3. **Occurrence evidence.** For the supported direct-member mechanism, report
   the complete native Group inventory, classify every member, report
   completeness, and return the exact evidence sidecar.
4. **Fact evidence.** Read the fact request, observe only explicitly bound
   native properties, return typed values with explicit unit tokens and
   availability, and never infer material, mass, or unrequested facts.
5. **Baseline integrity.** Read the Engine integrity request, hash each
   materialized resource before native mutation, and report the observed
   baseline digest.
6. **Configured-result integrity.** After persistence, hash the persisted native
   document bytes and report the persisted-result coordinate and digest for the
   expected primary resource.
7. **Diagnostics and raw evidence.** Return deterministic raw evidence bytes
   and explicit diagnostics; never silently drop an unsupported or unavailable
   case.
8. **Document lifecycle.** Open, mutate, recompute, persist, observe, traverse,
   export, and close within the same invocation and supplied working-copy root.

## Forbidden adapter behavior

The FreeCAD phase must not:

- emit Engine-normalized records or local record packages;
- implement Engine verification or decide expected-versus-observed outcomes;
- derive quantity, BOM totals, mass, material properties, or unit conversions;
- invent logical resource identity or promote undeclared native resources into
  declared coverage;
- enforce or rewrite `prm.project-lock.json` or source-lock policy;
- derive plan, job, product, attempt, or normalized record identity;
- persist or query PDM data;
- perform recursive dependency discovery, link rebinding, or recursive
  occurrence expansion beyond the explicit supported scope.

## Raw evidence and timing

| Capability | Engine input | FreeCAD output | Timing |
| --- | --- | --- | --- |
| Generic references | `prm.reference-traversal-request.json` | `prm.reference-traversal.json` | After mutation/recompute/persistence, before close |
| Occurrences | `prm.occurrence-request.json` at the working-copy root | `prm.occurrence-evidence.json` under `--output-dir` | On the same open, mutated, recomputed, persisted document, after observation and before close |
| Facts | `prm.fact-request.json` at the working-copy root | `prm.fact-evidence.json` under `--output-dir` | On the same open, mutated, recomputed, persisted document, after observation and before close |
| Integrity baseline | `PARAMETRON_RUNTIME_INTEGRITY_REQUEST` request document | `prm.integrity-evidence.json` under `--output-dir` | Hash every declared baseline before document open and before native mutation |
| Configured result | Persisted native document | `persistedResult` in the integrity evidence | Hash the post-save bytes immediately after persistence, before exports and observation |

All attempt-scoped paths are contained by the supplied `--working-copy` root or
the validated output directory. Host absolute paths and raw lookup identifiers
are not normalized identity material. Engine preserves returned raw bytes
verbatim and keeps them separate from normalized records.

## Current limitations

The implemented FreeCAD surfaces are deliberately bounded:

- The supported occurrence scope is the direct
  `Assembly::AssemblyObject.Group` member mechanism
  (`freecad-1.1.4-assembly-group-individual-link`).
- Nested assemblies, `Assembly::AssemblyLink`, `LinkArray`/`ElementCount`
  arrays, and non-empty `Assembly::JointGroup` are unsupported or incomplete.
- Non-empty occurrence context is unsupported in this phase; the runtime reports
  an unsupported scope instead of expanding it.
- Generic references are not occurrences and do not imply quantity.
- Facts are bounded to `bound_property`; there is no material database, unit
  algebra, or physical mass.
- Unit tokens are opaque; no conversion is performed.
- The baseline digest is the pre-mutation materialized input identity; the
  persisted-result digest is the post-save configured output identity.
- A baseline mismatch fails before mutation.
- No resource manifest is passed to the adapter; declared coordinates come from
  the integrity request channel.
- FreeCAD never reads, enforces, or rewrites `prm.project-lock.json`.
- Metadata must already exist in the source document. FreeCAD does not create
  missing metadata, property groups, aliases, or fallback properties, and
  metadata authoring policy remains Engine-owned.
- Read-only metadata properties fail at the metadata stage. Unsupported
  Quantity-family native types (`App::PropertyLength`, `App::PropertyDistance`,
  `App::PropertyAngle`, `App::PropertyQuantity`) fail deterministically.
- Parameter observation supports scalar boolean, integer, finite float, and
  string values; native `Quantity` values are not observed.
- In-process `run_engine_invocation` behavior is unchanged: it does not read the
  environment integrity channel and provides no evidence output directory. The
  external `execute` entry point is the supported engineering-evidence
  transport.

## Out of scope

This handoff does not require or authorize:

- recursive dependency or occurrence discovery;
- BOM serialization or adapter-reported quantity authority;
- physical mass, density, material inference, or a material database;
- universal unit algebra or conversion;
- metadata creation, property groups, templates, BOM semantics, metadata
  deletion, or a metadata-authoring GUI beyond selecting existing supported
  properties;
- runtime technical drawing or derived export semantics;
- PDM storage or query models;
- a second schema for any semantic surface already owned by Engine.

See [FreeCAD Adapter Architecture](architecture.md) and
[FreeCAD Runtime](runtime.md) for the implemented runtime boundary, and
[Engine record contracts](../../reference/record-contracts.md) for normalized
record and package semantics.
