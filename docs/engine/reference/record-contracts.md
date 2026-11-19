# Engine-produced record contracts

Engine builds normalized records and local record packages independently of an
external storage service. Raw operational outputs are evidence and mapping inputs;
they are not interchangeable with normalized records. Engine does not own durable
PDM persistence or write directly to a PDM store.

## Families, version, identity, and provenance

The Engine implementation owns the payload contracts, builders, normalizers,
validators, family registry, identity derivation, and shared provenance. The
registered version is `1.0`. Version validation rejects missing, malformed, and
unsupported versions, including unsupported versions within the same major.
Concrete JSON schema files and golden examples are not the current authority.

Registry order and filenames are:

| Family | Filename under `records/` | Material |
| --- | --- | --- |
| execution | `prm.execution-record.json` | Execution outcome, jobs, steps, timing and related record links |
| artifact | `artifacts/<identityId>/prm.artifact-record.json` | Artifact identity, inventory and content metadata |
| observation | `prm.observation-record.json` | Normalized observed facts and evidence, with optional typed occurrence summary and optional bounded engineering fact observations |
| reference | `prm.reference-record.json` | Normalized reference edges, endpoints, resolution and evidence |
| failure | `prm.failure-record.json` | Class, message, stage, severity, code, location, retry/timeout/cancellation and evidence |
| verification | `prm.verification-record.json` | Engine outcome, category results, failure classes, linkage and evidence |

Each family has `Build…Record`, `Normalize…Record`, and `Validate…Record`
functions. `Definitions()` returns an independent registry snapshot;
`FileNameForFamily` supplies the package filename. Registration does not imply
that every normal run emits every family.

`DeriveIdentity` uses `sha256-canonical-json-v1` over normalized family, version,
required caller record key, provenance, and deterministic identity parts. Identity
parts sort by key and value. Family builders own payload-specific identity
material; callers must not invent a second record hash or substitute runtime paths
for logical identity.

Shared provenance contains:

- Source revision: revision ID, asset ID, optional SHA-256 digest.
- Inputs: kind, identity, optional digest for inputs such as DSL, model, tables,
  and profile.
- Plan: plan ID/hash.
- Linkage: product key, job ID, step reference.
- Evidence: kind, reference, optional digest.
- Runtime: tool ID, runtime ID, adapter.
- Integrity: optional per-resource source, baseline, runtime-baseline, and
  persisted-result identity chain (see
  [integrity provenance](#integrity-provenance)).

Normalization trims applicable strings, sorts collections deterministically, and
copies caller-owned data. Present source-revision provenance requires a revision
or asset ID; inputs require kind and identity; evidence requires kind and ref;
present plan provenance requires plan ID or hash. Supplied digests must be valid
lowercase SHA-256. Normalization does not fabricate source or revision identities.
Failure and verification records validate their outcome consistency and normalize
evidence, linkage, and category ordering.

## Package layout and writer

The Engine implementation owns layout, path classification, filesystem
materialization, and normal-run emission. Documentation centralization does not
move those sources, validators, or schemas out of `parametron-engine`.

```text
parametron-record-package/
  prm.record-package.json
  records/
    prm.execution-record.json
    artifacts/
      <artifactRecordIdentity>/
        prm.artifact-record.json
    prm.observation-record.json
    prm.reference-record.json
    prm.failure-record.json
    prm.verification-record.json
    outcomes/
      <outcomeKey>/
        prm.observation-record.json
        prm.reference-record.json
        prm.verification-record.json
  artifacts/files/
  raw/
    prm.report.json
    prm.metadata.json
    artifact-store/prm.artifact-store-manifest.json
    handoff/
    observed/prm.observed.json
    verification/prm.verification.json
    runtime/prm.result.json
    runtime/prm.reference-traversal.json
    runtime/prm.occurrence-evidence.json
    runtime/prm.integrity-evidence.json
    fact/prm.fact-evidence.json
    outcomes/
      <outcomeKey>/
        observed/prm.observed.json
        verification/prm.verification.json
        runtime/prm.result.json
        runtime/prm.reference-traversal.json
        runtime/prm.occurrence-evidence.json
        runtime/prm.integrity-evidence.json
        fact/prm.fact-evidence.json
```

This is the allowed layout, not a promise that every listed file exists per run.
Artifact records are plural: a package may contain zero, one, or multiple
artifact records, with one identity-addressed record per artifact even when only
one exists. Engineering-observation families are outcome-scoped when a run has
more than one eligible verified CAD outcome: observation, reference, and
verification records are written under
`records/outcomes/<outcomeKey>/<family file>`, and their attempt-scoped raw
evidence is written under `raw/outcomes/<outcomeKey>/<family-relative path>`.
A single eligible outcome keeps the established singular record and raw-evidence
paths. Execution and failure records remain singular. Duplicate artifact
identities, duplicate family/outcome scopes, or resolved paths fail rather than
overwriting or collapsing records. `PackageInput` requires `PackageRoot`,
`PackageKey`, and one or more validated records; artifact and raw evidence files
are optional. By default it writes directly to the resolved root;
`UseCanonicalDirectoryName` selects the canonical child directory.

`prm.record-package.json` indexes records, artifacts and raw evidence and
includes `schemaVersion`, `packageKey`, `layoutVersion`, and ownership metadata.
Records follow registry family order. Within a family, outcome-scoped entries
sort by normalized outcome key and then canonical record path; singleton entries
keep their established path order. Artifact record entries sort by normalized
record identity; packaged artifact payloads sort by contract path. Raw evidence
follows layout order, with canonical attempt-scoped paths before deterministically
ordered outcome-scoped paths. Safe `raw/handoff/...` files sort lexically at the
handoff layout slot. Writer-owned JSON uses two-space indentation and a trailing
newline. Input payload buffers are copied.

The default writer rejects a non-empty destination or an existing manifest.
`OverwriteExisting` replaces layout-owned files idempotently and preserves
unrelated files. Writes are confined to the package root. Artifact payloads must
be under `artifacts/files/`; raw files must pass `IsRawEvidenceFileContractPath`.
Sentinels include `ErrInvalidPackageInput`, `ErrInvalidRecord`,
`ErrPackageDestinationExists`, `ErrPackageManifestExists`, and `ErrInvalidLayoutPath`.

`Entries`, `RecordEntries`, `RawEvidenceEntries`, `ValidateContractPath`,
`JoinContractPath`, and the write-free `Layout` resolver define paths without
performing record mapping. `EntryForContractPath` classifies static layout
entries and dynamic identity- or outcome-addressed record/raw paths.
`IsNormalizedRecordContractPath` recognizes registered singleton paths and
outcome-scoped record paths.

Raw CAD evidence uses an explicit allowlist:
`raw/runtime/prm.result.json`, `raw/verification/prm.verification.json`,
`raw/observed/prm.observed.json`,
`raw/runtime/prm.reference-traversal.json`, optional
`raw/runtime/prm.occurrence-evidence.json`, optional
`raw/runtime/prm.integrity-evidence.json`, and optional
`raw/fact/prm.fact-evidence.json`. Each attempt-scoped entry also accepts its
safe outcome-scoped form under `raw/outcomes/<outcomeKey>/`, preserving the
family-relative path (for example
`raw/outcomes/<outcomeKey>/runtime/prm.result.json`). The outcome key must be a
single safe path segment; escaping, empty, or multi-segment keys are rejected.
Run-level report, metadata, artifact-store manifest, and handoff evidence do not
have outcome-scoped forms. Raw fact evidence is accepted only through this
explicit allowlist entry; the corresponding `prm.fact-request.json` sidecar is
runtime-local attempt material and is not packaged. Arbitrary descendants under
`raw/runtime/` are not accepted automatically. Safe
descendants of `raw/handoff/` are dynamic handoff evidence files, not
normalized records or raw runtime-result evidence. `raw/handoff` itself is a directory, not an evidence file. Unsafe and
noncanonical paths are rejected.

## Operational mapping and emission

Engine performs deterministic, validated, copy-safe mapping;
it does not write package files. Each mapping family exposes a typed invalid-input
error (`ErrInvalidReportMapping`, `ErrInvalidMetadataMapping`,
`ErrInvalidArtifactMapping`, `ErrInvalidObservedMapping`,
`ErrInvalidVerificationMapping`, `ErrInvalidCADRuntimeFailureMapping`, or
`ErrInvalidReferenceTraversalMapping`).

| Input | Mapping | Normal-run use |
| --- | --- | --- |
| `prm.report.json` | `MapReport`: execution and optional failure record; status, timing, plan, jobs, steps, errors, retry/timeout/cancellation, deterministic linkage and outcome precedence | Execution/failure records and raw report |
| `prm.metadata.json` | `MapMetadata`: provenance and input identities, plan and conservative runtime/toolchain enrichment | Provenance enrichment and available raw metadata |
| Artifact store records / `prm.artifact-store-manifest.json` | `MapArtifactStoreRecords` / `MapArtifactStoreManifest`: one artifact record per artifact | All applicable records are emitted at deterministic identity-addressed paths |
| Typed observed-state input | `MapObserved`: observation and optional reference records, including canonical target-state facts | Applicable observation record emitted; exact `prm.observed.json` bytes remain raw evidence |
| Engine verification-result input | `MapVerification`: summary, categories, failure classes and evidence, including target-state category/failures and observed-evidence provenance when target-state verification is enabled | Applicable verification record emitted; distinct from raw `prm.verification.json` request bytes |
| Generic CAD runtime failure outcome | `MapCADRuntimeFailure`: adapter-neutral semantic class/stage plus native code/message, linkage, retry context and provenance | May replace the report-derived failure for one uniquely correlated terminal failed CAD outcome |
| `prm.reference-traversal.json` | `MapReferenceTraversal`: optional reference record | Bounded verified-candidate integration described below |
| Run integrity identity chain | `MapIntegrityProvenance`: lock verification state and the separate source, baseline, runtime-baseline, and persisted-result identities per declared resource | Optional integrity provenance block for runs that address declared CAD resources; exact `prm.integrity-evidence.json` bytes remain raw evidence |
| Handoff package | Raw runtime/provenance evidence classification | No normalized handoff record mapping or automatic handoff collection |
| Job status | Operational lifecycle state | Not a normalized record or durable storage contract |

`MapCADRuntimeFailure` accepts an Engine-owned generic CAD runtime failure
outcome, not an adapter-native result type. Runtime-specific interpretation
happens before this mapper. Current FreeCAD-native argument/manifest validation
failures map to `validation / validation`; availability/access failures and
native observation/traversal failures map to `adapter / adapter`; artifact
export failure maps to `export / export`; and ordinary execution failures or
otherwise-valid unknown native vocabulary use the deterministic
`runtime / runtime` fallback. The exact native boundary, category, stage, code,
and message remain in raw `prm.result.json`; normalized records contain Engine
interpretation, with native code and message preserved. Engine-owned resource
integrity violations are classified distinctly as failure class/stage
`integrity` with their specific stable, path-free code preserved; see
[integrity provenance](#integrity-provenance).

Verification and CAD runtime failure mappers validate optional evidence digests,
collapse matching evidence references, and reject conflicting digests.
Verification category order is `components`, `metadata`,
`parameters`, `references`, `target_state` under record normalization. This is
normalized-record ordering, not the semantic verifier's first-failure evaluation
order.

### Target-state observation and verification mapping

`MapObserved` represents canonical target-state evidence through the existing
observation family with `kind = target_state`; it does not introduce a separate
record family. The supported fact keys are `suppression`, `visibility`, and
`existence`. Each fact subject retains the exact destination and object identity.
Its value preserves the raw evidence status, and includes boolean material only
when suppression or visibility evidence semantically contains an observed
boolean. In particular, `target_missing` and `unavailable` are statuses, not
boolean `false` values. Existence retains `absent`, `exists`, or `unavailable` as
the raw semantic status.

`MapVerification` supports the existing verification family's
`category = target_state`. Target-state outcomes use
`target_state_mismatch`, `target_missing`, and
`native_evidence_unavailable`; the shared `required_observation_missing` class
also applies when required target-state evidence is omitted. Existing contract
and observed-artifact invalidity classes remain distinct from these semantic
outcomes.

When target-state verification is enabled, normalized verification provenance
can reference both `raw/verification/prm.verification.json`, the serialized
identity-only request and Engine verification input contract, and
`raw/observed/prm.observed.json`, the actual evidence used in comparison.
Optional SHA-256 digests are independent. Matching references are deduplicated;
conflicting supplied digests fail mapping instead of replacing evidence. These
normalized references do not replace either raw source.

Normal non-cached runs emit under `<runRoot>/parametron-record-package/`, with
package key `engine-run:<planHash>`. Execution records are always mapped when
report construction succeeds; report failure material adds a validated failure
record, including on failed runs. Raw report bytes remain unchanged. Metadata and
artifact inventory use their run-level sources. CAD result, verification-request,
and observed evidence comes from authoritative paths retained from actual CAD
attempts rather than guessed run-root locations.

For successful result, verification-request, and observed projection, every
terminal CAD outcome that completed without a job error and passed Engine
verification is eligible. Zero eligible outcomes omit the projection. Multiple
eligible outcomes each retain their own outcome-scoped observation,
verification, reference, occurrence, fact, integrity, and raw evidence; no
attempt is selected arbitrarily and no eligible outcome is silently dropped.
Successful outcomes remain eligible observation evidence even when another
product makes the overall run fail; failure-record selection and projection are
unchanged. Exact `prm.result.json`, `prm.verification.json`, and
`prm.observed.json` bytes are preserved where available. Typed observed evidence
is supplied to `MapObserved`, while the Engine-owned in-memory verification
result is supplied to `MapVerification`; raw bytes are provenance material, not
semantic substitutes for those typed inputs.

On a failed run, runtime-native evidence is eligible only when exactly one CAD
failure outcome correlates with the authoritative terminal report failure by
established job, product, and step linkage. A valid unique candidate is mapped
through `MapCADRuntimeFailure` and replaces the report-derived normalized
failure. Zero or multiple matching candidates, absent or malformed evidence, or
evidence that cannot be uniquely correlated retain the report-derived fallback;
there is no first- or last-match selection. The package contains at most one
failure record. Execution and raw report evidence remain report-derived, and
Engine-owned retry count and plan-hash provenance are retained. A successful
runtime followed by an Engine verification mismatch is not a runtime-native
failure, and malformed, invalid, missing, or unavailable evidence does not
fabricate one. Retry authority remains with the executor and scheduler.

Each available source is read through the existing
working-copy confinement, symlink, and regular-file guards. Optional missing
sources are omitted. The bytes read from each source are passed unchanged to
package emission. In particular, `raw/verification/prm.verification.json` is the
request sent to FreeCAD, not Engine's normalized or in-memory verification result.
For an applicable native failure, SHA-256 provenance is calculated over the exact
preserved `raw/runtime/prm.result.json` bytes, not a reserialized typed value.

Normalized report mapping removes step/runtime timing for normal-run records.
Consequently, normal emitted report-derived and runtime-native failure records
omit `OccurredAt`, although the generic CAD runtime failure mapper can normalize
an explicitly supplied occurrence timestamp. Retry count remains present.
Equivalent normal runs have stable package keys, normalized record and manifest
bytes, file sets, and indexes. Raw evidence preserves timestamps and operational
paths; full package-tree byte equality is not guaranteed. Successful non-cached
runs complete cache only after package emission succeeds. Failed execution retains
its original execution error and does not complete cache. Cached runs neither
create missing packages nor modify existing packages.

## Traversal normalization and bounded reference emission

The pure `MapReferenceTraversal` mapper accepts rich traversal schema `1.0`
evidence, a stable caller `RecordKey`, and the completed finite declared-resource
inventory described in [project mapping](../authoring/project-mapping.md).
Every inventory binding must have a completed materialized coordinate. The mapper
performs no discovery, I/O, digest computation, or package writing.
`recordcontract.BuildReferenceRecord` owns normalized identity and canonical edge
order; `NormalizeProvenance` owns provenance order.

### Resource-qualified native endpoints

Declared resource identity is the tuple `projectId`, `kind`, `logicalId`.
The mapper joins raw endpoint document coordinates against the inventory's
attempt-relative materialized coordinates, requiring a unique match for the
source. Normal execution also supplies `ExpectedSourceDocument`: raw
`sourceDocument` must exactly match that selected prepared source, rather than
merely naming any staged resource. An ambiguous project-source alias cannot
substitute for a completed materialized coordinate.

These coordinates have different purposes:

| Coordinate | Meaning |
| --- | --- |
| Logical resource identity | Declared project/resource identity, independent of working-copy placement |
| Project-source coordinate | Project-relative location of declared source material; multiple logical resources may alias it |
| Materialized coordinate | Canonical attempt-relative location used to join native traversal evidence to a completed binding |
| Operational absolute path | Host filesystem location used by execution; not a native relationship identity coordinate |
| Native entity identity | Native entity kind and, for objects, exact object name scoped by its resource |

Native endpoints preserve `resource`, `nativeKind`, `qualification`, `resolution`,
`name`, and `path` where applicable. `nativeKind` distinguishes `document`,
`object`, `external_document`, and `external_file`. Only objects carry `name`;
document-level entities do not acquire object names. Thus resource A / object
`Part` and resource B / object `Part` are distinct entities.

`qualification` is independent of native resolution:

- `declared`: `resource` carries the declared identity tuple (currently a model
  resource); `path` is absent. Raw document paths are join coordinates, not the
  authoritative identity of qualified endpoints.
- `undeclared`: a target cannot be joined; it retains a canonical contract-relative
  `path` and no invented resource identity.
- `unsupported`: an unjoinable `external_file` target retains its canonical
  contract-relative `path` without a resource identity.

The source must be declared. Accepted unqualified targets retain their native
kind, name where applicable, and resolution. Native endpoints do not substitute
raw lookup IDs, `assetId`, `revisionId`, digests, or semantic target IDs for
resource identity. Absolute, escaping, and noncanonical fallback paths are rejected.

### Native relationships and identity

A normalized edge carries neutral `nativeKind`, nullable `sourceProperty` and
`referenceMechanism`, existing coarse `kind`, `source`, `target`, `resolution`,
`linkage`, and `evidence`. Each nullable discriminator is either unavailable or
an exact non-empty string without surrounding whitespace or NUL; absence differs
from every observed value. Property-only and mechanism-only differences remain
distinct. For example, resource A / `Body01` / `Link` -> resource B / `Part`
and resource A / `Body01` / `Support` -> resource B / `Part` are distinct generic
native relationships when accepted evidence proves the property distinction.

| Native relationship kind | Coarse normalized kind |
| --- | --- |
| `document_internal_reference` | `component` |
| `external_document_reference`, `external_file_reference` | `external` |

`ReferenceEdge.Role` remains empty; property names do not imply domain roles.
Internal relationships remain within one declared resource. External relationships
cannot target the source resource; external-file relationships require an
`external_file` target.

`ReferenceRelationshipKey` is the resource-qualified normalized relationship
identity. Its coordinates are source and target logical resource identity where
qualified, native entity kind and object name, explicit qualification and accepted
contract-relative fallback coordinates, neutral native relationship kind, nullable
source property, and nullable reference mechanism. It excludes resolution state,
raw traversal sequence, raw lookup IDs, raw JSON ordering, raw evidence SHA-256,
absolute host/attempt paths, and execution linkage. Equivalent supported
relationships retain the same keys and canonical ordering across equivalent
relocation and evidence-order changes.

The mapper canonicalizes equivalent duplicate relationship evidence, ignoring
lookup aliases and incidental sequences. Distinct accepted relationships sharing
source and target objects remain distinct. Conflicting evidence claiming the same
relationship identity fails deterministically, rather than merging or selecting
an arbitrary value. Raw node state, object type, labels, and diagnostic material
remain relevant to evidence comparison before projection; contradictory native
node evidence or reused lookup IDs also fail. The record validator rejects
remaining duplicate relationship keys. Legacy generic reference records retain
their existing ordering and duplicate rules; they do not all require native
resource fields or traversal scope. Their endpoint operational paths and
runtime-reported working-copy digests remain in the normalized payload and raw
evidence but do not participate in record identity. Endpoint qualification is
unchanged: the generic mapper does not invent resource identity where none is
declared.

`ReferenceRecord.Identity` serves a different purpose from the relationship key.
It retains the existing evidence-sensitive derivation through the unchanged
shared `DeriveIdentity` algorithm: caller record key, normalized reference
payload, provenance, linkage, and supplied evidence digest participate. Generic
endpoint paths and runtime-reported endpoint digests are excluded from that
identity material, so relocating a working copy or re-reporting its digest does
not change reference-record identity when the remaining reference material is
equal. Accepted contract-relative native traversal coordinates remain identity
material, and the native traversal record path is unchanged. Different exact
raw evidence bytes can still change record identity even when normalized
relationship keys and order are identical. Determinism is relative to the actual
accepted identity material; this is not a promise of identical record bytes
across different raw evidence or provenance.

With multiple eligible verified CAD outcomes, each outcome emits its own
reference record under `records/outcomes/<outcomeKey>/` and its own exact raw
traversal bytes under `raw/outcomes/<outcomeKey>/runtime/`. Reference records
from distinct outcomes are never merged merely because their resource or native
coordinates are equal; the outcome record key, linkage, and evidence provenance
keep them distinct. Zero eligible outcomes add no reference material.

### Resolution, traversal scope, and empty records

Native endpoint and relationship resolution preserves all five states:
`resolved`, `missing`, `unresolved`, `skipped`, and `failed`. Missing evidence
establishes absence or unavailability; unresolved evidence does not establish
absence; skipped denotes an explained non-attempt; failed denotes an attempted
item failure. Skipped/failed items require item or scope diagnostic evidence.
A resolved relationship requires resolved source and target evidence; a missing
relationship cannot have a resolved target. Every relationship source must agree
with the traversal source document and resource.

Aggregate `status` and supported-scope `completeness` are separate:

| Traversal status | Accepted completeness and constraints |
| --- | --- |
| `succeeded` | `complete` or `incomplete`; the source is resolved |
| `partial` | `incomplete`; raw evidence includes an opened, resolved source document and explanatory diagnostics |
| `failed` | `unavailable`; diagnostics are required and no usable edges are accepted; the mapper retains a declared source with unresolved native resolution |

The mapper starts succeeded scope as complete, marks partial scope incomplete,
and marks any unqualified target incomplete. `unsupported` is a scope indicator
set for unsupported targets or an accepted `unsupported_reference_value_shape`
diagnostic. That diagnostic requires partial status, warning severity, and
`reference_discovery` stage. Unsupported scope cannot be complete.

Succeeded status does not guarantee that targets are declared or that all resource
joins succeed. Item resolution does not by itself determine aggregate status or
completeness; a failed item can remain in a partial graph. Completeness describes
only accepted supported traversal scope and resource qualification, not recursive
CAD dependency closure or assembly occurrence completeness. Inventory members
need not each have a native relationship.

Valid zero-edge input produces a scope-bearing reference record: complete empty
traversal, incomplete empty traversal, and unavailable/failed empty traversal
remain distinguishable. Source qualification, status, completeness, and evidence
provenance survive without edges. Pure mapper acceptance is separate from normal
run eligibility: failed traversal status is not synonymous with failed overall
Engine execution, and failed overall executions do not enter the successful-run
traversal package path.

### Exact evidence and bounded emission

The package preserves exact raw bytes at
`raw/runtime/prm.reference-traversal.json`. Lowercase SHA-256 is computed over
the original bytes, never a normalized reserialization. Raw evidence and
normalized records are separate. Lookup IDs, sequences, labels, diagnostic prose,
and other raw-only fields do not automatically become relationship identity,
though relevant evidence still participates in validation and conflict rejection.

Mapper linkage and provenance linkage reconcile independently for `JobID`,
`ProductKey`, and `StepRef`, after trimming: empty plus a value adopts that value,
equal values converge, and conflicting non-empty values fail. The resolved
linkage populates both edge and provenance linkage. Canonical evidence is
`kind = reference-traversal` with `ref = raw/runtime/prm.reference-traversal.json`
or its safe outcome-scoped form. Caller and provenance digests reconcile the same
way; duplicate canonical evidence collapses to one entry and unrelated
provenance is preserved. Context validation also applies to zero-edge records.

For normal-run emission, every terminal correlated CAD outcome with an explicit
verification pass, traversal bytes, and canonical Engine job/product/step
linkage is eligible, regardless of whether another product makes the overall run
fail:

| Eligible verified CAD outcomes | Traversal additions |
| --- | --- |
| Zero | None |
| One | Accepted exact raw traversal evidence and normalized reference record at established singular paths, including zero-edge scope |
| More than one | One exact raw traversal evidence file and normalized reference record per outcome under `raw/outcomes/<outcomeKey>/` and `records/outcomes/<outcomeKey>/`, including zero-edge scope |

Each eligible outcome supplies its completed inventory and prepared
source-document coordinate for Engine-owned correlation and qualification.
`recordemit` strictly decodes JSON, rejecting unknown fields and trailing values;
computes SHA-256 over exact bytes; supplies the established reference record key
(singular or outcome-scoped) and metadata-derived provenance; and calls the
mapper. The writer emits the registry-backed reference file and separately
indexes the unchanged raw bytes. Invalid JSON, mapping, linkage, provenance, or
evidence fails package emission closed. No first/last candidate selection or
concatenated JSON occurs, and equivalent relationships are not aggregated across
outcomes.

Normal-planner production coverage remains internal/component references;
accepting supplied external target evidence does not establish automatic
production population of all external mappings. There is no recursive discovery
or native link rebinding. Runtime collection is owned by
[execution runtime](../runtime/execution-runtime.md); FreeCAD's raw contract is
owned by [reference traversal](../adapters/freecad/contracts/reference-traversal.md).

Generic native reference != assembly/component occurrence. The historical
`component` classification does not prove a physical instance. Distinct property
relationships and repeated evidence do not establish physical instance cardinality.
Generic `App::PropertyXLink` relationships, shared or distinct targets, native
property/mechanism counts, and traversal `succeeded` status cannot certify
occurrence inventory or completeness. The generic relationship identity contract
remains independent of the explicit occurrence contract below.

## Explicit occurrence observation

Engine owns occurrence requests and evidence contracts, native-coordinate and
resource qualification, accepted identity, parent/context validation,
completeness, duplicate/conflict handling, deterministic normalization, quantity
rollup, normalized records, and raw provenance/package emission. FreeCAD owns
actual native Assembly inspection, member inventory discovery, parent-membership
observation, link-target resolution, native status/diagnostics, coverage evidence,
and production occurrence evidence generation. Engine does not inspect native
FreeCAD objects. FreeCAD does not derive Engine logical-resource identity or
quantity. The Engine contract accepts controlled evidence, and
`parametron-freecad` implements the bounded production producer for the
direct-member mechanism.

### Versions and evidence validation

| Contract | Accepted schema versions |
| --- | --- |
| `OccurrenceRequest` | `1.0` |
| Native occurrence evidence | `1.0`, `1.1` |
| `OccurrenceSummary` | `1.0` |

Request `1.0` accepts either supported evidence version and produces summary
`1.0`. These version domains are independent of generic reference contracts.
Evidence `1.0` retains its closed member shape: `contents` is forbidden even
when explicitly null. Evidence `1.1` adds structured auxiliary contents.
Unsupported versions have no automatic compatibility path. Strict request/evidence
decoding rejects unknown fields, duplicate keys including case-alias duplicates,
trailing JSON, invalid types, and contract-invalid values, including forbidden
nulls. Null is permitted only where the contract allows it; it cannot prove a
required inventory, state, or completed inspection.

Evidence has `kind: "native_occurrence_evidence"`, `freecadVersion`, the exact
requested `scope`, observed `parent`, `completeness`, `inspectionCompleted`,
`group`, `members`, and `diagnostics`. Inventory and diagnostic arrays must be
explicit, not missing or null. Member fields are `name`, `parent`,
`nativeType`, `elementCount`, `membershipCorroborated`, `classification`,
`resolution`, `nativeStateValid`, `target`, optional `diagnostic`, and, under
`1.1`, optional `contents`. Classifications are `occurrence`, `auxiliary`
(`1.1` only), `unresolved`, or `unsupported`. Contradictory native evidence fails
validation rather than being repaired or silently dropped.

### Bounded native mechanism

`freecad-1.1.4-assembly-group-individual-link` requires exact
`freecadVersion: "1.1.4"`. The root must be an explicitly identified
`Assembly::AssemblyObject`. Its entire direct `Group` inventory is the scope,
not every link in the document. An accepted member must be an individual
`App::Link` with `ElementCount == 0`, corroborated requested-parent membership,
valid native state, `classification: "occurrence"`, `resolution: "resolved"`,
no diagnostic, and an unambiguously qualified resolved target. Supported target
types are `PartDesign::Body`, `Part::Feature`, and `App::Part`. One accepted link
contributes one occurrence of the exact target; `App::Part` contents are not
expanded. This mechanism carries no FreeCAD 1.1.1 compatibility claim.

Evidence `1.1` permits only the narrow `Assembly::JointGroup` auxiliary exclusion:
exact type, membership in the requested inventory, corroborated parent ownership,
valid native state, `classification: "auxiliary"`, `resolution: "resolved"`,
null `elementCount` and `target`, no diagnostic, and verified contents. The
following illustrates only the member's `contents` value:

```json
{"inspectionCompleted": true, "group": []}
```

Omitted or null contents/inventory do not establish emptiness. A verified empty
joint group contributes no occurrence; nonempty, unverified, or arbitrary
auxiliary objects cannot disappear from completeness accounting.

### Identity and context

`OccurrenceScope` contains `parent`, ordered `context`, and `mechanism`.
`OccurrenceKey(scope, member)` combines the scope and exact native member name,
excluding target and semantic annotations. Thus conflicting targets cannot hide
behind different occurrence keys. Same-named members in different parent scopes
or resources remain distinct; distinct members targeting one entity remain
separate occurrences.

Each `OccurrenceEntity` contains `resource: (projectId, kind, logicalId)`,
`nativeKind: "object"`, exact native `name`, and optional `semanticId` annotation.
Native names/member names must be nonempty, have no surrounding whitespace,
and contain no space, tab, CR, LF, dot, slash, backslash, or NUL. Logical identity, project-source coordinates,
attempt-relative materialized coordinates, parent assembly, occurrence member,
referenced native target, semantic annotation, and raw provenance are separate
identities. Labels, absolute host/attempt paths, generic edge IDs, and raw hashes
cannot substitute for occurrence identity.

Native parent/target evidence carries `entity`, attempt-relative `document`,
and `nativeType`. Document coordinate and explicit logical identity must join
the same unique materialized declared-resource binding. All inventory bindings
must be materialized. The requested parent resource must be the selected source;
the observed parent identity and document must match that request/source.
Undeclared, ambiguous, escaping, or mismatched resource joins fail.

Context is an explicit array of at most 16 ordered `{parent, member}` frames.
Missing/null context, malformed frames, repeated parent entities, and cycles to
the scope parent are invalid. Representation of a valid contextual path does
not prove recursive membership. Accepted occurrences and complete quantity
require empty context for this direct-member mechanism. Nonempty context may
represent unsupported/unavailable scope, never accepted recursive rollup.

### Completeness, duplicates, and quantity

| Completeness | Meaning and quantity |
| --- | --- |
| `complete` | Full direct inventory inspected and every member classified as an accepted occurrence or verified auxiliary exclusion; derived quantities available |
| `incomplete` | Coverage/resolution insufficient; accepted partial members may remain, but `quantities: null` |
| `unsupported` | Scope/member semantics outside the proven boundary; accepted members only within the supported direct scope, with `quantities: null` |
| `unavailable` | No usable scope observation; no accepted members and `quantities: null` |

Complete scope requires completed inspection, no coverage gaps, no unhandled
members, and no scope diagnostics. Duplicate native `group` names are invalid.
An inspected empty inventory, or one containing only verified empty auxiliaries,
produces `occurrences: []` and `quantities: []`. Incomplete, unsupported, and
unavailable scopes require diagnostics. Explicit unavailable evidence has no
observed parent, completed inspection, or inventory members. Omitted optional
evidence becomes unavailable without fabricated raw provenance; absent intent
adds no summary. Neither case establishes complete zero. Request requirement
`complete` rejects every noncomplete result with
`ErrOccurrenceCompletenessRequired`; malformed optional evidence still fails.

Exact repeated member evidence canonicalizes once; differing evidence for one
member identity fails deterministically, including target, parent, classification,
or semantic-annotation conflicts. Normalized summaries reject duplicate members.
Within one scope/context, Engine groups distinct accepted members by
`OccurrenceEntityKey(target)`: referenced `(projectId, kind, logicalId)`,
`nativeKind`, and exact `name`, excluding `semanticId`. Different entities within
one resource stay separate. Distinct members sharing a target each contribute
one; semantic annotations neither split nor merge native groups and are omitted
from quantity targets. Occurrences sort by member name; quantities sort by target
entity key, independently of raw inventory/member order.

Quantity comes only from complete accepted scope. Generic edge counts, semantic
component quantities, native `ElementCount`, CAD BOM totals, and recursively
inferred counts are not quantity sources.

### Records, provenance, and verification boundary

The optional typed summary is `ObservationRecord.observation.occurrences`,
containing `schemaVersion`, `scope`, `completeness`, `occurrences` (`member`,
`target`), `quantities` (`target`, `quantity`), and `evidence`. It extends the
existing observation family while preserving ordinary facts and independent
reference records; there is no seventh record family. Every eligible verified
CAD outcome retains its own occurrence summary and quantity, scoped to that
outcome's complete occurrence scope. Zero eligible outcomes produce no
occurrence material. Quantity is never aggregated across independent CAD
outcomes, and incomplete occurrence state in one outcome does not affect
another. See
[execution runtime](../runtime/execution-runtime.md#execution-scoped-occurrence-intent).

Original evidence bytes are retained exactly at
`raw/runtime/prm.occurrence-evidence.json` for single-outcome packages and under
`raw/outcomes/<outcomeKey>/runtime/prm.occurrence-evidence.json` for
multi-outcome packages, classified as raw runtime evidence. SHA-256 of those
original bytes supplies summary evidence and record provenance with kind
`occurrence-evidence`. Conflicting linkage/digests fail emission.
Equivalent accepted semantics preserve occurrence keys, ordering, and rollup;
different raw bytes can change evidence digest and containing observation record
identity. Shared `DeriveIdentity` is unchanged. Stable semantic identity does not
promise byte-identical full packages when provenance or operational outputs differ.

Permanent Engine contract/runtime tests cover versions, native eligibility,
resource joins, identity/context bounds, verified auxiliary exclusion, scope
availability, quantity, duplicates/conflicts, and semantic ordering. Executor
tests cover request copies, retries, successive execution isolation, and outcome
retention. Emission/projection tests cover exact bytes/digests, independent generic
references, and plural eligible-outcome emission. The continuous controlled proof
calls `ExecutePackageWithOptions` through materialization, runtime invocation,
intake, verification, retention, projection, and package emission; it does not
prove native FreeCAD occurrence production. The committed Engine
[test matrix](https://github.com/parametron-io/parametron-engine/blob/e483de980e7be8c9643e37adc815e22d80ca5723/docs/test-matrix.md#explicit-occurrence-contract-coverage)
identifies the permanent test responsibilities.

Unsupported behavior includes general recursive occurrence discovery/quantity,
complete nested assembly rollup, flexible subassemblies, link arrays,
`App::LinkElement` counting, unproven cyclic expansion, arbitrary auxiliary
exclusions, automatic external dependency discovery, public CLI/DSL/HTTP
occurrence authoring, CAD BOM quantity authority, BOM serialization, physical
mass, density/material inference, material catalogs or an Engine material
database, universal unit frameworks and unit conversion algebra, arbitrary or
universal native property observation, PDM persistence, and cross-outcome
quantity aggregation. Bounded native occurrence emission and bounded
`bound_property` fact observation are implemented by `parametron-freecad`.
Separately supported mutations and generic traversal retain their own contracts.

## Bounded engineering fact observation

Engine owns the requested fact scope, subject identity and resource
qualification, accepted fact kinds, value typing, explicit unit representation,
availability semantics, normalization into the existing observation record
family, deterministic identity and ordering, and deterministic failure
classification. The CAD runtime owns native property access, native value
extraction, native unit evidence, unavailable and unsupported diagnostics, and
raw evidence. Engine performs no heuristic property discovery and invents no
facts the runtime did not observe. Fact observation is separate from mutation
verification: fact observations never serve as target-state evidence, and
target-state verification never decides fact acceptance. Quantity remains the
occurrence-derived semantics described above; it is not a CAD fact and is not
observed through the fact contract.

### Sidecar contracts

| Contract | Accepted schema versions |
| --- | --- |
| `FactRequest` (`prm.fact-request.json`) | `1.0` |
| `FactRuntimeEvidence` (`prm.fact-evidence.json`) | `1.0` |

Intent is opt-in and execution-scoped through
`Executor.ExecutePackageWithOptions` with
`PackageExecutionOptions.FactObservation`; the public CLI, DSL, and HTTP
submission do not author fact requests. Engine materializes the versioned
request sidecar `prm.fact-request.json` (schema `1.0`) in the prepared attempt
working directory, and reads only the versioned evidence sidecar
`prm.fact-evidence.json` (schema `1.0`, evidence kind
`native_fact_evidence`) from its confined output directory. Nil intent requests
no fact observation and forbids unsolicited fact evidence. Strict decoding
rejects unknown fields, duplicate JSON keys, trailing JSON, invalid types, and
contract-invalid values. A present evidence file must cover every requested
binding; unsolicited fact evidence is rejected.

The single supported fact kind is `bound_property`: one explicitly named native
property observed under an explicitly bound, resource-qualified subject. Each
request binding names the native property explicitly and carries a closed
`valueKind`, an optional explicit unit token, and a `requirement` of `optional`
or `required`. Engine never infers native facts from property names.

### Resource-qualified subjects

A fact subject is the occurrence entity described
[above](#explicit-occurrence-observation): the declared resource tuple
`(projectId, kind, logicalId)`, `nativeKind: "object"`, the exact native name,
and an optional `semanticId` annotation. The `semanticId` is annotation-only
and is excluded from fact identity. Occurrence context frames are carried only
when a fact is genuinely occurrence-scoped, under the same occurrence context
validation, including the bounded frame count and cyclicity rules; the context
participates in fact identity only when present. Observed subjects join the
declared logical identity with the materialized document coordinate: identity
and coordinate must identify the same binding, and the same native name in two
declared resources stays distinct. Operational absolute paths are not identity
material.

### Values, units, and availability

Observed values are typed finite scalars: `number`, `integer`, `string`, or
`boolean`. The value kind is cross-checked against the actual JSON scalar at
intake and again during record validation. Numeric values canonicalize
deterministically — `42`, `42.0`, and `4.2e1` are one observed value — while
`NaN`, infinities, non-integral integers, and non-canonical strings are
rejected. Zero is a valid observed value.

Units are optional explicit opaque text tokens carried by both the request
binding and the evidence; an absent unit is an explicit unitless fact, never an
implicit unit. Unit tokens are bounded (at most 32 characters) and have no
semantics in Engine: no unit conversion, conversion algebra, or quantity
arithmetic exists or is implied. Units are incompatible with non-numeric
`string` and `boolean` values.

Availability is explicit: `observed`, `unavailable`, or `unsupported`. Observed
evidence must carry a value and a value kind; `unavailable` and `unsupported`
evidence must carry a diagnostic and no value material. Omission never
substitutes a state. An `optional` `unavailable` or `unsupported` fact is
retained as a normalized fact carrying its explicit state, while a `required`
non-observed binding fails deterministically. An omitted evidence file is
explicit unavailability for every requested binding.

Invalid fact contracts and conflicting duplicates within evidence fail with
`ErrInvalidFact`; required unavailable or unsupported bindings fail with
`ErrRequiredFactUnavailable`; record-level conflicting facts fail under the
existing `ErrInvalidObservationRecord`. Equivalent duplicate evidence
canonicalizes into one fact instead of selecting a value.

### Normalization, provenance, and emission

Fact evidence is mapped at the CAD runtime boundary by `MapFactEvidence` in
Engine's CAD runtime intake, not by a `recordmap` mapper: evidence subject
joins, availability strictness, numeric canonicalization, and duplicate and
conflict handling are decided before record construction. Normalized
engineering facts extend the existing observation record family as
`fact`-kind entries inside `observation.facts[]`, keyed by the fact kind; no
separate record family exists and no new family is registered. Fact emission
merges into that outcome's observation record, preserving ordinary observation
facts and the occurrence summary. Normalized fact ordering follows the
requested bindings; raw evidence ordering does not influence acceptance or
output, and relocation does not change normalized fact identity.

Every fact carries shared provenance covering the observed resource and entity,
the explicitly bound native property, the fact kind, and the exact raw
evidence bytes: source kind `fact-evidence`, source reference
`raw/fact/prm.fact-evidence.json` or its
`raw/outcomes/<outcomeKey>/fact/prm.fact-evidence.json` form, and the SHA-256
digest of the exact evidence bytes. Raw fact evidence is preserved at the
corresponding path, classified as raw runtime evidence; the request sidecar is
not packaged. Every eligible verified CAD outcome retains its own fact material:
equal fact subjects with different observed values remain distinct
outcome-scoped observations, and unavailable or unsupported state remains
outcome-specific. Zero eligible outcomes emit no fact material. Facts are never
merged or aggregated across CAD outcomes, and intake failures surface at the
`outcome_validation` consumption stage.

Permanent Engine contract/runtime tests cover finite typed values and numeric
canonicalization, closed vocabularies, bounded binding and request validation,
resource-qualified subject joins, availability strictness, duplicate and
conflict semantics, deterministic identity and ordering across relocation and
evidence-order changes, request materialization, absent/present/confined
evidence handling, observation-family merge, optional omission versus required
failure, and raw-evidence path classification. The committed Engine
[test matrix](https://github.com/parametron-io/parametron-engine/blob/80d93c1d28efe6e68def896be058b7164cdf2a0c/docs/test-matrix.md#bounded-engineering-fact-observation-contract-coverage)
identifies the permanent test responsibilities.

Unsupported behavior includes physical mass, density/material inference,
material catalogs or an Engine material database, universal unit frameworks and
unit conversion algebra, heuristic or arbitrary/universal native property
observation, target-scoped metadata mutation, multi-outcome fact aggregation,
and public CLI/DSL/HTTP fact authoring. Native fact production is implemented by
`parametron-freecad` for explicit `bound_property` bindings; permanent Engine
verification additionally retains controlled-evidence coverage, and native
runtime version tokens in evidence are provenance only.

Newly emitted packages use the current `prm.*` raw-evidence paths. No package
migration walker, automatic old-name conversion, or compatibility alias exists
for already-produced packages. Package archiving and remote publishing are not
implemented. Normal package emission retains the overwrite behavior described above.

## Integrity provenance

Shared record provenance carries an optional integrity block for runs that
address declared CAD resources. It is normalized provenance material, not a
new record family, and it is omitted from serialized provenance entirely
when absent, so records for runs without integrity material keep their
existing bytes and identities. The shared `DeriveIdentity` algorithm treats
integrity as separate provenance material; the record identity contract is
unchanged.

The block carries the run-level lock verification state — `verified`,
`mismatch`, `missing` (a declared resource absent from the required lock
scope), `malformed` (a non-canonical locked digest), `unavailable` (a lock
that cannot be read or parsed), or `not_performed` for an unlocked workflow,
which never claims lock verification — plus one entry per declared resource:

| Field group | Contents |
| --- | --- |
| Source identity | Logical resource ID, canonical project-source coordinate, accepted source digest, per-resource lock state |
| Materialized baseline identity | Attempt-relative materialized coordinate, materialized baseline digest, materialization status |
| Runtime baseline identity | Runtime-reported baseline digest and status |
| Persisted-result identity | Attempt-relative persisted-result coordinate, reported persisted-result digest, result validation status |

Source, materialized baseline, runtime-baseline, and persisted-result
content identities are separate fields and are never collapsed into one
content identity. Per-stage statuses are `verified`, `mismatch`, `missing`,
`malformed`, `inaccessible`, `unavailable`, and `unsupported`, plus
`not_performed` for stages that did not run and `not_reported` for evidence
the runtime did not provide. Entries are ordered by declared logical
resource identity, so equivalent evidence normalizes identically regardless
of evidence order or host paths; duplicate, contradictory, or undeclared
provenance evidence for one resource identity fails deterministically.

Integrity failures are classified distinctly: failure records for
Engine-owned resource integrity violations carry failure class/stage
`integrity` with their specific stable, path-free code preserved, keeping
lock, materialization, runtime-baseline, and result-content failures
distinguishable; non-integrity failures keep their existing classes.
Configured-result digests never update the source lock and never replace
source identity. In multi-outcome packages the run-level execution record keeps
source/lock identity only, while each outcome's engineering-observation records
retain that outcome's full configured-result chain; separate outcomes may share
source identity while carrying different persisted-result digests. Runtime
integrity evidence is preserved verbatim at
`raw/runtime/prm.integrity-evidence.json` for single-outcome packages and under
`raw/outcomes/<outcomeKey>/runtime/prm.integrity-evidence.json` for
multi-outcome packages, with SHA-256 provenance over the exact original bytes.
The enforced pipeline is documented in
[execution runtime](../runtime/execution-runtime.md#source-lock-and-configured-result-provenance);
the runtime-facing contract surfaces are owned by the
[integrity request](../adapters/freecad/contracts/integrity-request.md) and
[integrity evidence](../adapters/freecad/contracts/integrity-evidence.md)
contracts.
