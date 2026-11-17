# Integrity Request Contract

The integrity request is the Engine-authored statement of the per-resource
integrity expectations for one CAD runtime attempt. Engine resolves its own
byte-verified materialized baselines, then publishes this document so the
runtime can check the actual baseline it observes against the identity Engine
assigned. Engine validates the runtime's answer through the
[integrity evidence contract](integrity-evidence.md); the request itself is
not an adapter-authored contract.

Transport filename:

```text
prm.integrity-request.json
```

## Canonical Engine-authored request contract (Schema 1.0)

Schema `1.0` is the only supported version. Top-level fields:

- `schemaVersion`
- `resources[]`

One `resources` entry per declared CAD resource the runtime may report:

| Field | Meaning |
|-------|---------|
| `logicalId` | Declared logical resource identity |
| `kind` | Declared resource kind (currently the model kind only) |
| `materializedCoordinate` | Canonical attempt-relative coordinate Engine materialized for the resource |
| `baselineSha256` | Expected pre-mutation baseline digest of the materialized bytes |
| `expectsPersistedResult` | Whether this resource must report persisted configured-result identity |

Composition is Engine-owned and byte-deterministic: resources are unique and
ordered by declared logical resource identity, carry no host paths, and
equivalent inputs serialize to byte-identical documents. Exactly one resource
carries `expectsPersistedResult: true` — the selected primary source document
for the attempt. Strict decoding rejects unknown fields, trailing content,
non-`1.0` schema versions, empty resource sets, and non-canonical logical IDs,
resource kinds, coordinates, and digests.

## Publication and delivery

The request is published at an attempt-scoped location under the run root,
outside the attempt working copy:

```text
<runRoot>/integrity/<attemptID>/prm.integrity-request.json
```

The location is delivered to the external runtime process only through the
`PARAMETRON_RUNTIME_INTEGRITY_REQUEST` environment variable. The channel is
Engine-owned: when Engine publishes no request for an invocation, an
inherited value of that variable is stripped rather than passed through, and
the runtime command-line arguments never change. There is no CLI flag or
in-file discovery for this contract; a runtime that receives the variable
reads exactly that document, and a runtime invoked without it reports no
integrity evidence — unsolicited evidence for coordinates outside a request
is rejected at intake. The in-process invocation callable does not read this
environment configuration (see
[engine-invocation.md](engine-invocation.md#files-and-environment)).

## Current FreeCAD support

No native FreeCAD integrity producer exists yet. Permanent Engine
verification covers this contract through controlled runtimes and stub
capabilities: the runtime participant reads the published request, reports
[evidence](integrity-evidence.md) against it, and Engine validates that
evidence strictly. Real FreeCAD pre-mutation baseline observation and
post-persistence result hashing are a later adapter phase to be built against
this contract and the evidence contract; they are not implemented.

## Ownership

Engine authors, publishes, and strictly validates this request, and owns the
integrity decisions made from the runtime's response; the runtime owns native
access to the materialized resource, pre-mutation baseline observation,
mutation/recompute/persistence, post-persistence result hashing, and raw
integrity evidence. See
[Execution Runtime — Source Lock and Configured-Result Provenance](../../../runtime/execution-runtime.md#source-lock-and-configured-result-provenance)
for the enforcement pipeline this request belongs to.
