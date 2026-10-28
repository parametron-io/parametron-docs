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
| observation | `prm.observation-record.json` | Normalized observed facts and evidence |
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
```

This is the allowed layout, not a promise that every listed file exists per run.
The artifact family is the bounded exception to the otherwise-singular family
model: a package may contain zero, one, or multiple artifact records, with one
record per artifact. The identity-addressed path applies even when exactly one
artifact record exists. Execution, observation, reference, failure, and
verification remain singular. Duplicate artifact identities or resolved paths
fail rather than overwriting or collapsing records. `PackageInput` requires
`PackageRoot`, `PackageKey`, and one or more validated records; artifact and raw
evidence files are optional. By default it writes directly to the resolved root;
`UseCanonicalDirectoryName` selects the canonical child directory.

`prm.record-package.json` indexes records, artifacts and raw evidence and
includes `schemaVersion`, `packageKey`, `layoutVersion`, and ownership metadata.
Records follow registry family order. Artifact record entries sort by normalized
record identity; packaged artifact payloads sort by contract path, and raw
evidence follows layout order. Safe `raw/handoff/...` files sort lexically at the
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
performing record mapping. `EntryForContractPath` classifies static layout entries.
`IsNormalizedRecordContractPath` recognizes only registered record paths.

Raw CAD evidence uses an explicit allowlist:
`raw/runtime/prm.result.json`, `raw/verification/prm.verification.json`,
`raw/observed/prm.observed.json`, and
`raw/runtime/prm.reference-traversal.json`. Arbitrary descendants under
`raw/runtime/` are not accepted automatically. Safe descendants of `raw/handoff/`
are dynamic handoff evidence files, not normalized records or raw runtime-result
evidence. `raw/handoff` itself is a directory, not an evidence file. Unsafe and
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
interpretation, with native code and message preserved.

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

For successful result, verification-request, and observed projection, the
overall execution must succeed and exactly one CAD outcome must have completed
without a job error and passed Engine verification. Zero eligible outcomes omit
the projection; multiple eligible outcomes also omit it rather than selecting an
attempt arbitrarily. Exact `prm.result.json`, `prm.verification.json`, and
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
their existing identity, ordering, and duplicate rules; they do not all require
native resource fields or traversal scope.

`ReferenceRecord.Identity` serves a different purpose from the relationship key.
It retains the existing evidence-sensitive derivation through the unchanged
shared `DeriveIdentity` algorithm: caller record key, normalized reference
payload, provenance, linkage, and supplied evidence digest participate. Different
exact raw evidence bytes can therefore change record identity even when
normalized relationship keys and order are identical. Determinism is relative
to the actual accepted identity material; this is not a promise of identical
record bytes across different raw evidence or provenance.

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
`kind = reference-traversal`, `ref = raw/runtime/prm.reference-traversal.json`.
Caller and provenance digests reconcile the same way; duplicate canonical evidence
collapses to one entry and unrelated provenance is preserved. Context validation
also applies to zero-edge records.

For normal-run emission, only an overall successful run is eligible. Each candidate
must be a terminal correlated CAD outcome with an explicit verification pass,
traversal bytes, and canonical Engine job/product/step linkage:

| Eligible verified CAD outcomes | Traversal additions |
| --- | --- |
| Zero | None |
| Exactly one | Accepted exact raw traversal evidence and normalized reference record, including zero-edge scope |
| More than one | Omitted deterministically |

The selected outcome supplies its completed inventory and prepared source-document
coordinate for Engine-owned correlation and qualification. `recordemit` strictly
decodes JSON, rejecting unknown fields and trailing values; computes SHA-256 over
exact bytes; supplies `engine-run:<planHash>:reference` and metadata-derived
provenance; and calls the mapper. The writer emits the registry-backed reference
file and separately indexes the unchanged raw bytes. Invalid JSON, mapping,
linkage, provenance, or evidence fails package emission closed. No first/last
candidate selection, concatenated JSON, or general multi-outcome aggregation occurs.

Normal-planner production coverage remains internal/component references;
accepting supplied external target evidence does not establish automatic
production population of all external mappings. There is no recursive discovery
or native link rebinding. Runtime collection is owned by
[execution runtime](../runtime/execution-runtime.md); FreeCAD's raw contract is
owned by [reference traversal](../adapters/freecad/contracts/reference-traversal.md).

Generic native reference != assembly/component occurrence. The historical
`component` classification does not prove a physical instance. Distinct property
relationships and repeated evidence do not establish physical instance cardinality.
This contract introduces no occurrence IDs, native occurrence paths, quantity
rollup, or BOM inference. The explicit occurrence boundary belongs to
[parametron-io/parametron-engine#45](https://github.com/parametron-io/parametron-engine/issues/45)
and [parametron-io/parametron-engine#50](https://github.com/parametron-io/parametron-engine/issues/50).

Newly emitted packages use the current `prm.*` raw-evidence paths. No package
migration walker, automatic old-name conversion, or compatibility alias exists
for already-produced packages. Package archiving and remote publishing are not
implemented. Normal package emission retains the overwrite behavior described above.
