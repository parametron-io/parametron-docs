# Observation Contract

Observation reads requested CAD-native state from an already-mutated,
recomputed, and persisted document. It never evaluates checks or makes
verification decisions — that is Engine's job.

## Request: `prm.verification.json`

Malformed JSON, duplicate keys, non-JSON constants, and non-object roots are
rejected; decoded requests must also fit the supported observation scope.
Loading and compatibility failures become a structured observation-request
error with the original cause preserved. This validation does not evaluate
checks or compare expected values.

Contract metadata defines top-level fields under canonical schema `1.0`:
`schemaVersion` (required), `observe` (required), `observationContext` (optional
— Engine defaults absent parameter bindings to an empty list), `expected`
(required), `checks` (required).

These canonical Engine contract requirements are not all enforced with the same
strictness by the current `parametron-freecad` loading and compatibility chain:
for example, `{"observe":{"metadata":true}}` is accepted without
`schemaVersion`, `expected`, or `checks`.

`observe` is a set of boolean-ish enable flags per category: `parameters`,
`metadata`, `references`, `components`, and optional `targetState`. `expected`
carries per-category lookup data (e.g. `expected.references[]` entries with
`kind`/`name`). `checks` is recognized contract shape only; FreeCAD does not
evaluate checks.

Of the categories defined in schema `1.0`, only `parameters`, `metadata`, and
`references` have an implemented observation helper in current FreeCAD.
`components` is recognized contract shape only; the external request
compatibility check rejects enabled component observation and non-empty
component expectations.

### Target-state observation request extension

Under canonical schema `1.0`, Engine optionally requests target-state
observation by setting:

```json
"observe": {
  "targetState": true
}
```

`observe.targetState` corresponds directly to the presence of request-scoped
target-state context in `observationContext.targetState`.

When `observationContext.targetState` is present, the contract defines
structural presence rules:

- All three fact family arrays are strictly required:
  - `suppression`
  - `visibility`
  - `existence`
- The request must contain at least one target-state fact across those three
  arrays (an entirely empty `targetState` context object is invalid).

Each requested target identity is an exact native reference:

```json
{
  "destination": "part",
  "object": "Pad"
}
```

- `destination`: closed vocabulary of `"assembly"` or `"part"`.
- `object`: exact native object name (non-blank string, without leading or
  trailing whitespace, and containing no null bytes). Semantic IDs are never
  runtime observation identities.

Target-state requests are strictly request-scoped; they identify specific native
objects to inspect rather than requesting unrestricted native-document
enumeration. Canonical mutation intent maps to observation request families as
follows:

- suppression mutation $\rightarrow$ `suppression` observation request
- visibility mutation $\rightarrow$ `visibility` observation request
- deletion mutation $\rightarrow$ `existence` observation request

The observation request identifies the native fact to inspect; it does not copy
or fabricate requested mutation boolean values as observed evidence.

Entries in each target-state collection (`suppression`, `visibility`,
`existence`) are ordered deterministically by `destination` ascending, then
`object` ascending. Duplicate `{destination, object}` entries within any family
are rejected.

When target-state observation is not requested or unused, `observe.targetState`
and `observationContext.targetState` are omitted, preserving existing schema `1.0`
request bytes unchanged.

## Aligned external observation (`--observation-request` on `execute`)

`--observation-request` is a file-based input confined to the validated
working copy and requires `--output-dir`. The runtime:

1. loads the request and computes the source-document digest before the
   document is opened;
2. observes the same open, already-mutated, recomputed, and persisted
   document, after artifact export and reference traversal, before it is
   closed;
3. writes `<output-dir>/prm.observed.json` atomically.

A standalone external `observe` command does not exist; observation is only
available aligned onto `execute`.

## Response: `prm.observed.json`

The current FreeCAD runtime output emitted by `parametron-freecad` execute is:

```json
{
  "schemaVersion": "1.0",
  "workingCopy": { "path": "...", "sha256": "..." },
  "observation": {
    "parameters": [ { "id": "...", "name": "...", "groupId": "...", "value": 1.0, "valueKind": "number" } ],
    "metadata": [ { "id": "...", "key": "...", "ownerId": "...", "value": "...", "valueKind": "string" } ],
    "references": [ { "kind": "...", "name": "..." } ]
  }
}
```

- `workingCopy.sha256` is the lowercase SHA-256 of the source `.FCStd` file's
  bytes, computed before any in-memory mutation. The field name is
  `workingCopy.sha256` even though the hashed bytes are the source document's
  bytes, not a hash of the whole working-copy directory. Caller request data
  cannot override this digest, and unrelated output-directory contents do not
  participate in it.
- Only enabled, supported categories appear under `observation`; `components`
  is defined in the contract but not currently populated.
- Output is canonical UTF-8 JSON with deterministic requested-item ordering,
  canonical key ordering, and one trailing newline. Writing is atomic and
  never leaves a partial file after a failed write.

### Target-state raw evidence contract extension

Under canonical schema `1.0`, Engine defines the optional raw evidence result
contract `observation.targetState`. When target-state observation is unrequested
or absent, `observation.targetState` is omitted.

Current `parametron-freecad` normal `execute` does not yet emit this contract
extension; the following structure is the canonical Engine-owned target-state
result shape:

```json
{
  "schemaVersion": "1.0",
  "workingCopy": { "path": "...", "sha256": "..." },
  "observation": {
    "parameters": [ { "id": "...", "name": "...", "groupId": "...", "value": 1.0, "valueKind": "number" } ],
    "metadata": [ { "id": "...", "key": "...", "ownerId": "...", "value": "...", "valueKind": "string" } ],
    "references": [ { "kind": "...", "name": "..." } ],
    "targetState": {
      "suppression": [
        { "destination": "part", "object": "Fillet", "status": "unavailable" },
        { "destination": "part", "object": "Pad", "status": "observed", "value": true },
        { "destination": "part", "object": "Pocket", "status": "target_missing" }
      ],
      "visibility": [
        { "destination": "part", "object": "Body", "status": "unavailable" },
        { "destination": "part", "object": "DatumPlane", "status": "observed", "value": false },
        { "destination": "part", "object": "Sketch", "status": "target_missing" }
      ],
      "existence": [
        { "destination": "assembly", "object": "SubAssembly", "status": "exists" },
        { "destination": "part", "object": "Fastener", "status": "unavailable" },
        { "destination": "part", "object": "OldBracket", "status": "absent" }
      ]
    }
  }
}
```

#### Structural presence rules

When `observation.targetState` is present, the contract shape strictly requires
all three fact family arrays:
- `suppression`
- `visibility`
- `existence`

Unlike request context, the contract does not require observed target-state
collections to contain at least one item; empty collections are structurally
valid. Entries within each collection are canonically ordered by `destination`
ascending, then `object` ascending.

#### Suppression and visibility evidence

Boolean target-state evidence represents native suppression and visibility using
a closed status vocabulary:

- `status: "observed"`: The native object was resolved and its boolean state was
  read. The entry **must contain** `value: true | false`.
- `status: "target_missing"`: The requested native object was not found in the
  document. `value` **must be absent**. `target_missing` must not be treated as
  equivalent to `value: false`.
- `status: "unavailable"`: The native object was found, but suppression or
  visibility evidence could not be determined natively. `value` **must be
  absent**. `unavailable` must not be treated as equivalent to `target_missing`
  or `false`.

#### Existence evidence

Existence evidence represents post-execution object presence using a closed
status vocabulary:

- `status: "exists"`: The native object was confirmed to exist in the live
  document post-execution.
- `status: "absent"`: The native object was confirmed absent from the live
  document post-execution. This is valid native evidence (e.g. following
  deletion intent) and is distinct from an observation failure or `unavailable`.
- `status: "unavailable"`: Existence or absence could not be definitively
  determined from native state. `unavailable` must not be treated as equivalent
  to `absent`.

#### Raw evidence independence

Raw observation output represents live native state independently from mutation
intent:

```text
requested mutation != observed native state
```

For example, a visibility mutation of `visible = false` may produce raw evidence
`status: "observed", value: true` if the CAD object remained visible. The raw
contract faithfully records this observed fact; interpreting whether that
mismatch constitutes a verification pass or failure is strictly an Engine-owned
responsibility.

### Requested parameter and metadata observation

Both resolve exact requested items only (owner/key or object/property reads);
neither infers, enumerates, or guesses values that were not explicitly
requested.

### Requested reference observation (not graph traversal)

For each `expected.references[]` entry, FreeCAD resolves the exact `name`
through `document.getObject(name)` to prove existence and echoes back the
requested `kind`/`name` in request order. It:

- does not traverse `document.Objects`
- does not infer `kind`
- does not discover dependencies
- does not preserve missing/unresolved external reference state

This is existence-checking of explicitly named objects, not the reference
graph traversal described in
[reference-traversal.md](reference-traversal.md).

## Current FreeCAD support

Current `parametron-freecad` normal `execute` implementation does not yet
support the target-state request fields or produce the corresponding native
target-state evidence.

When valid canonical target-state evidence is supplied, Engine consumes it
through its expected-versus-observed verifier and the existing normalized
observation and verification mappers. See
[Execution Runtime](../../../runtime/execution-runtime.md) for verification
semantics and [Record Contracts](../../../reference/record-contracts.md) for
normalized mapping semantics. Mapper availability does not imply normal-run
normalized record emission.

Specifically:

- **Engine-owned contract shape**: Canonical schema `1.0` request and result
  contracts for `targetState` are fully defined.
- **FreeCAD runtime observation**: Of the categories defined in schema `1.0`,
  current FreeCAD normal `execute` implements observation helpers only for
  `parameters`, `metadata`, and `references`. `components` is recognized
  contract shape only and rejected if enabled. Target-state request fields
  (`observe.targetState`, `observationContext.targetState`) are not yet consumed
  by the normal `execute` request loader, and native target-state evidence
  (`observation.targetState`) is not yet emitted.

## Failure behavior

Request loading, source hashing, observation generation, serialization, and
output-writing failures all use structured-failure stage `observation` (see
[result-and-failure.md](result-and-failure.md)). The document is still closed
through the normal lifecycle; no success `prm.result.json` is written after an
observation failure; a failed `prm.result.json` is written when its path is safe.

## Ownership

FreeCAD owns: document lifecycle, CAD-native mutation, recompute, native
persistence, export, and reading requested parameter/metadata/reference facts
(and future native target-state facts) into canonical raw observation output.
FreeCAD never evaluates checks or makes verification decisions.

Engine owns: defining requested observation facts and raw result schemas,
expected/observed comparison, tolerance evaluation, category and overall
verification outcomes, failure classification, and acceptance/rejection
decisions. FreeCAD's observation output never contains comparison results,
tolerances, category outcomes, or normalized Engine records. See [Execution
Runtime — Engine-Owned
Verification](../../../runtime/execution-runtime.md#engine-owned-verification)
for how Engine consumes this evidence.
