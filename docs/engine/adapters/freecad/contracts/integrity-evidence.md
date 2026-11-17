# Integrity Evidence Contract

The integrity evidence is the runtime-authored report of the per-resource
baseline and persisted configured-result identity the runtime actually
observed for one attempt. It answers the Engine-authored
[integrity request](integrity-request.md): Engine strictly validates this
evidence against its own request and its own independently computed digests;
a runtime never validates its own report.

Transport filename:

```text
prm.integrity-evidence.json
```

The runtime writes this file into its confined output directory
(`<OutputDir>/prm.integrity-evidence.json`). A missing file is absent
evidence, not a failure; Engine continues with the baseline evidence it
already verified itself and records the runtime stages as not reported.

## Canonical runtime-facing evidence contract (Schema 1.0)

Schema `1.0` is the only supported version. Top-level fields:

- `schemaVersion`
- `resources[]`

One `resources` entry per reported declared resource:

| Field | Meaning |
|-------|---------|
| `materializedCoordinate` | Attempt-relative coordinate the runtime accessed; must match a requested coordinate |
| `logicalId` | Optional declared logical ID; when present it must agree with the requested resource for that coordinate |
| `status` | Resource access status; `observed` when omitted |
| `baselineSha256` | Digest of the actual pre-mutation baseline bytes, for observed resources |
| `persistedResult` | Post-persistence configured-result identity for the resource that must report it: `{coordinate, sha256}` |

### Resource access statuses

| Status | Meaning |
|--------|---------|
| `observed` | The runtime accessed the resource and reports its actual baseline content identity |
| `unavailable` | The runtime could not access the declared resource |
| `missing` | The declared resource was absent at the materialized coordinate |
| `unsupported` | The runtime cannot verify a resource of this kind |

A non-observed status states that the runtime could not establish the
resource's baseline; it must not carry baseline or persisted-result content
identity. Non-observed statuses are accepted for secondary resources; the
primary source document must be observed and reported.

## Strict validation against the Engine request

Engine joins each reported entry to declared resource identity through the
requested materialized coordinate and validates strictly:

- Malformed or non-`1.0` documents, unknown fields, trailing content, unknown
  statuses, and duplicate materialized coordinates are rejected.
- Evidence naming a coordinate the request did not ask about is rejected.
- A logical ID that disagrees with the requested resource for its coordinate
  is a provenance conflict, not a reinterpretation.
- An observed resource that omits its baseline digest, or reports a
  non-canonical baseline digest, fails distinctly from one whose reported
  baseline differs from the Engine materialized baseline.
- The resource carrying the persisted-result obligation must report its
  persisted configured-result identity. Engine additionally hashes the
  accessible persisted bytes itself and requires the runtime-reported digest
  to agree; missing, malformed, or mismatched result digests and missing or
  inaccessible persisted files are distinct deterministic failures.

Runtime baseline mismatch therefore stays distinguishable from source-lock,
materialization, and result-content failures. The full intake pipeline, the
path-free failure codes, and the normalized provenance built from this
evidence are documented in
[Execution Runtime — Source Lock and Configured-Result Provenance](../../../runtime/execution-runtime.md#source-lock-and-configured-result-provenance)
and [record contracts](../../../reference/record-contracts.md#integrity-provenance).

## Current FreeCAD support

No native FreeCAD integrity producer exists yet. Permanent Engine
verification uses controlled runtimes and stub capability responses for this
contract. Real FreeCAD pre-mutation baseline observation and post-persistence
result hashing are a later adapter phase to be built against this contract;
they are not implemented.

## Ownership

The runtime author of this evidence owns native access, baseline observation,
post-persistence hashing, and raw diagnostics. Engine owns joining reports to
declared resource identity, strict validation, independent persisted-result
validation, deterministic failure classification, normalized provenance, and
byte-preserving raw evidence storage at
`raw/runtime/prm.integrity-evidence.json`.
