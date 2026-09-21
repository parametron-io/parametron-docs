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
| execution | `parametron.execution-record.json` | Execution outcome, jobs, steps, timing and related record links |
| artifact | `artifacts/<identityId>/parametron.artifact-record.json` | Artifact identity, inventory and content metadata |
| observation | `parametron.observation-record.json` | Normalized observed facts and evidence |
| reference | `parametron.reference-record.json` | Normalized reference edges, endpoints, resolution and evidence |
| failure | `parametron.failure-record.json` | Class, message, stage, severity, code, location, retry/timeout/cancellation and evidence |
| verification | `parametron.verification-record.json` | Engine outcome, category results, failure classes, linkage and evidence |

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
  parametron.record-package.json
  records/
    parametron.execution-record.json
    artifacts/
      <artifactRecordIdentity>/
        parametron.artifact-record.json
    parametron.observation-record.json
    parametron.reference-record.json
    parametron.failure-record.json
    parametron.verification-record.json
  artifacts/files/
  raw/
    prm.report.json
    prm.metadata.json
    artifact-store/manifest.json
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

`parametron.record-package.json` indexes records, artifacts and raw evidence and
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
| Artifact store records / `manifest.json` | `MapArtifactStoreRecords` / `MapArtifactStoreManifest`: one artifact record per artifact | All applicable records are emitted at deterministic identity-addressed paths |
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

The pure mapper accepts supplied traversal schema `2.0` evidence and a stable
caller `RecordKey`. It performs no file discovery, I/O, digest computation, or
package writing. `recordcontract.BuildReferenceRecord` owns normalized identity
and edge order; `NormalizeProvenance` owns provenance order.

| Raw value | Normalized value |
| --- | --- |
| Resolution `resolved` | `resolved` |
| Resolution `missing`, `unresolved`, `skipped`, `failed` | `unresolved` |
| Kind `document_internal_reference` | `component` |
| Kind `external_document_reference`, `external_file_reference` | `external` |

Object endpoints use document path and object name; document/external-document/
external-file endpoints use document path with an empty name. Raw node IDs only
support edge lookup; they do not become endpoint IDs, asset/revision IDs, or
storage identity. Labels are not identity. `ReferenceEdge.Role` stays empty.
Property/mechanism/type details, labels, diagnostics, raw sequence, aggregate
status, and raw node IDs stay in raw evidence. Exact normalized duplicate edges
collapse deterministically; distinct normalized edges remain distinct without
invented roles or path suffixes.

Mapper linkage and provenance linkage reconcile independently for `JobID`,
`ProductKey`, and `StepRef`, after trimming: empty plus a value adopts that value,
equal values converge, and conflicting non-empty values fail. One resolved
linkage populates both edge and provenance linkage. Canonical evidence is
`kind = reference-traversal`, `ref = raw/runtime/prm.reference-traversal.json`.
Caller and provenance digests reconcile the same way; duplicate canonical evidence
collapses to one entry. Unrelated provenance is preserved. Context validation
occurs even for zero edges; valid zero-edge input returns no record and no error.

Identity can include caller record key, normalized provenance and linkage,
normalized kind/endpoints/resolution, and supplied canonical evidence digest.
The mapper introduces no additional identity based on raw ordering, labels,
diagnostics, timestamps, process IDs or attempt-local placement.

For normal-run emission, only an overall successful run is eligible. Each candidate
must be a terminal correlated outcome with an explicit verification pass, traversal
bytes, and canonical Engine job/product/step linkage:

| Eligible candidates | Emission |
| --- | --- |
| Zero | No traversal additions |
| Exactly one | Exact raw evidence and optional normalized reference record |
| More than one | Omit traversal additions deterministically |

`recordemit` strictly decodes JSON, rejecting unknown fields and trailing values;
computes lowercase SHA-256 over the exact bytes; supplies
`engine-run:<planHash>:reference` as record key and metadata-derived provenance;
and calls the pure mapper. The exact digest input is emitted unchanged at the
canonical raw path. Non-zero mapped edges produce the registry-backed reference
file through the writer. Zero-edge evidence is indexed without a reference record.
Invalid JSON, mapping, linkage, provenance, or evidence fails package emission
closed. No first/last candidate selection or concatenated traversal JSON occurs.

Normal-planner production coverage is internal/component references. Non-empty
external target integration, multi-product aggregation, multiple records per family,
richer reference fields, and failed-run traversal normalization are not part of
this bounded emission path. Package archiving and remote publishing are not
implemented. Runtime collection is owned by
[execution runtime](../runtime/execution-runtime.md); traversal request structure
is owned by [adapter architecture](../adapters/README.md).

Newly emitted packages use the current `prm.*` raw-evidence paths. No package
migration walker, automatic old-name conversion, or compatibility alias was
introduced for already-produced packages. Captured raw bytes remain unchanged,
and normal package emission retains the overwrite behavior described above.
