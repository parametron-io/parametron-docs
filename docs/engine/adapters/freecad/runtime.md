# FreeCAD Runtime

This document describes the current, implemented `parametron-freecad`
headless runtime contract: how it is invoked, how it resolves a FreeCAD host,
how it scopes file access, and what it does during one execution.

For architecture-level ownership and capability boundaries, see
[architecture.md](architecture.md). For manifest, result/failure, observation,
reference-traversal, and artifact schemas, see the documents under
[contracts/](contracts/). For Engine's orchestration of this runtime —
attempt isolation, request materialization, evidence intake validation, and
verification — see [Execution Runtime](../../runtime/execution-runtime.md).

Normal `execute` consumes canonical schema `1.0` mutation sections and invokes
the native metadata, suppression, visibility, and conservative deletion
consumers. Metadata mutation writes only existing native properties and never
creates metadata. Mutation work follows parameter assignment and precedes
persistence and downstream evidence. Recompute and validity behavior depends on
the mutation sequence: mutation-free execution retains its established
recompute behavior; metadata, suppression, or visibility state receives a
required final recompute and supported Body validity inspection; deletion
performs removal, exact absence verification, recompute, and surviving-Body
validity checks for each deletion. Later mutation
state after deletion receives its required final pass, while a final deletion
does not receive a redundant pass. See [Target
Mutations](contracts/target-mutations.md) for the detailed composition.

Before opening the document, the runtime can hash Engine-declared baseline
resources and consume occurrence and fact request sidecars. After successful
mutation processing and applicable recompute/validity, it saves the native
working document, hashes the persisted configured-result bytes when integrity
is enabled, runs declared exports, optional reference traversal, optional
observation, and optional occurrence and bounded fact inspection on the same
live document, then closes it, writes integrity evidence, and writes the success
result. Observation emits raw live post-mutation target state when requested;
Engine alone compares that evidence with expected state and normalizes records.
See [Observation Contract](contracts/observation.md) for request and evidence
shapes.

## Invocation

For the separate in-process `run_engine_invocation` callable, its request
types, and typed errors, see
[Engine invocation](contracts/engine-invocation.md). Its `observe` mode
accepts injected document state; it is not a standalone external command.

The supported external entry point is the installable executable
`parametron-freecad`. Callers should not need to invoke `freecadcmd` directly
or know a Python script path.

Smoke mode:

```text
parametron-freecad smoke
```

Execute mode:

```text
parametron-freecad execute \
  --working-copy <working-copy-dir> \
  --manifest <manifest-path> \
  --result <result-path> \
  [--output-dir <output-dir>] \
  [--observation-request <request-path>] \
  [--reference-traversal-request <request-path>]
```

## FreeCAD host resolution

Host resolution order:

1. `PARAMETRON_FREECAD_BIN`, if configured
2. otherwise `freecadcmd` from `PATH`

`PARAMETRON_FREECAD_BIN` names the underlying FreeCAD host executable, not
`parametron-freecad` itself. A blank or whitespace-only value is rejected. A
value that resolves back to the `parametron-freecad` wrapper (recursively) is
rejected. Invalid configured values do not silently fall back to the default
host. Engine's own external-runtime executable selection is separate and
documented in [CLI Runtime Behavior — External runtime
configuration](../../cli/runtime-behavior.md#external-runtime-configuration).

## Launcher transport

The launcher starts the FreeCAD host without a shell, using a fixed Python
bootstrap expression. Caller arguments are never interpolated into that
bootstrap source and `eval` is never used. Each runtime protocol argument is
forwarded as one structured `--pass=<argument>` element, preserving order and
values containing spaces, Unicode, quotes, shell metacharacters, and `=`.

Process behavior:

- child stdout/stderr are inherited without rewriting
- the child's return code is propagated
- signal termination is mapped to `128 + signal`
- host startup/configuration failures return `127` with diagnostic stderr
- the launcher is independent of the caller's working directory

## Working-copy / execution-root contract

The resolved `--working-copy` directory is the authoritative
execution-instance root for one invocation. Its basename and parent layout are
not prescribed by the runtime.

Accepted examples (must exist and be a directory):

```text
/run/_working
/run/_working/execution-a
/run/execution-a
/tmp/arbitrary-safe-root
```

Rejected: a missing root; a root that is a file; a manifest, result, output
directory, observation request, source document, or artifact path that
resolves outside that exact root; a sibling execution path; a parent/shared
path; a symlink-resolved external path.

Containment is exact, not ancestor-widening. For a supplied root of
`/run/product/_working/execution-a`, a path such as
`/run/product/_working/execution-b/result.json` remains invalid even though it
shares the `_working` parent. `_working` and an execution-id leaf are caller
conventions; the runtime does not infer or select a parent layout, and it does
not create the root or any parent directory automatically.

Root canonicalization uses existing path-resolution semantics; downstream
paths are checked against the same canonical root, and symlink children that
escape the root are rejected.

Ownership:

- Caller/Engine owns: choosing and preparing the root, execution identity,
  copying source material into it, and the manifest/result/output/request
  paths passed to the runtime. See [architecture.md — Attempt Identity and
  Prepared Working Copy](architecture.md#attempt-identity-and-prepared-working-copy)
  for how Engine derives and prepares that root.
- FreeCAD owns: root existence/type validation, canonicalization, child
  containment, traversal/symlink escape rejection, and CAD-native execution
  and observation within that root.

## Execute argument contract

- `--working-copy`, `--manifest`, and `--result` are required.
- `--output-dir` is optional for execution-only calls, but required whenever an
  evidence surface is enabled: reference traversal, observation, an environment
  integrity request, or an occurrence/fact request sidecar.
- `--observation-request` enables CAD-native observation on the same execute
  call and requires `--output-dir`.
- `--reference-traversal-request` enables reference traversal on the same
  execute call and requires `--output-dir`.
- Optional request/output paths must resolve inside the validated working
  copy; the observation and traversal request files must be existing regular
  files; the output directory must not resolve to an existing regular file.
- Empty and whitespace-only optional values are rejected.
- There is no legacy environment-variable fallback for these arguments.
- A standalone `observe` command is not supported and is rejected as an
  unknown command.

### Engineering-evidence sidecars

The runtime consumes three Engine-authored inputs and writes their evidence
under `--output-dir`; no new command-line argument is introduced:

- `prm.occurrence-request.json` at the working-copy root -> optional
  `<output-dir>/prm.occurrence-evidence.json`;
- `prm.fact-request.json` at the working-copy root -> optional
  `<output-dir>/prm.fact-evidence.json`;
- the Engine integrity request named by
  `PARAMETRON_RUNTIME_INTEGRITY_REQUEST` -> optional
  `<output-dir>/prm.integrity-evidence.json`.

Occurrence and fact requests are discovered only as plain, non-symlink regular
files at the exact working-copy root. A directory, symbolic link, or other
non-regular candidate fails deterministically instead of being followed or
ignored. The integrity request is read only from the environment channel named
by Engine; there is no CLI flag and no in-file discovery for it, and an
invocation without that variable reports no integrity evidence.

Enabling any evidence surface requires an existing `--output-dir` inside the
working copy. Stale sidecars for enabled surfaces are removed before request
loading, and each evidence file is written atomically as canonical UTF-8 JSON
with one trailing newline. Requests and evidence shapes remain Engine-owned and
are defined by the [FreeCAD handoff](handoff.md), the
[occurrence evidence contract](contracts/occurrence-evidence.md), the
[fact evidence contract](contracts/fact-evidence.md), and the
[integrity request](contracts/integrity-request.md) /
[integrity evidence](contracts/integrity-evidence.md) contracts.

## Execution lifecycle

For one `execute` call, in order:

1. Validate CLI arguments before importing FreeCAD.
2. Resolve `--working-copy` as the authoritative execution-instance root.
3. Load and validate the manifest (see
   [contracts/manifest.md](contracts/manifest.md)).
4. Resolve `sourceDocument` inside the working copy.
5. Determine the enabled evidence surfaces, require `--output-dir`, and remove
   stale sidecars for those surfaces.
6. If a reference-traversal request is present, load it exactly once (see
   [contracts/reference-traversal.md](contracts/reference-traversal.md)).
7. If an observation request is present, load it (see
   [contracts/observation.md](contracts/observation.md)).
8. If `PARAMETRON_RUNTIME_INTEGRITY_REQUEST` names a request, load the declared
   resource inventory and hash every declared baseline before the document is
   opened or mutated (see
   [contracts/integrity-request.md](contracts/integrity-request.md)).
9. If an occurrence request is present at the working-copy root, load it (see
   [contracts/occurrence-evidence.md](contracts/occurrence-evidence.md)).
10. If a fact request is present at the working-copy root, load it (see
    [contracts/fact-evidence.md](contracts/fact-evidence.md)).
11. If observation or integrity is enabled, compute the source-document
    SHA-256 digest before the document is opened (observation reuses the
    integrity baseline digest when both are enabled).
12. Open the FreeCAD document.
13. Apply parameter assignments in manifest order.
14. Apply target mutations in deterministic section/family order when present
    (`metadata` → `suppression` → `visibility` → `deletion`, Assembly before
    Part).
15. Perform applicable recompute and native validity checks (see
    [contracts/target-mutations.md](contracts/target-mutations.md)).
16. Persist the configured native working document with `document.save()`.
17. If integrity is enabled, hash the post-save bytes of the resource with the
    persisted-result obligation and verify that the saved document still
    matches its declared coordinate.
18. Export declared STEP/CSV/PDF artifacts (see
    [contracts/artifacts.md](contracts/artifacts.md)).
19. If reference traversal is enabled, run it on the recomputed, persisted
    live document and emit the canonical raw traversal output.
20. If observation is enabled, observe the same open live document — after
    persistence, exports, and traversal, before document close.
21. If occurrence evidence is requested, inspect the requested Assembly Group
    and write `prm.occurrence-evidence.json`.
22. If fact evidence is requested, observe every bound property and write
    `prm.fact-evidence.json`.
23. Close the document; any secondary documents opened only for declared fact
    subjects are closed first.
24. If integrity is enabled, write `prm.integrity-evidence.json` for the
    completed run.
25. Write deterministic success `prm.result.json` (see
    [contracts/result-and-failure.md](contracts/result-and-failure.md)).

Live-document observation, traversal, occurrence inspection, and fact inspection
all operate on the same open FreeCAD document instance that received mutation,
applicable recompute/validity, and persistence; the document is not reopened.
The original external/project source document remains outside the mutation
target. Declared secondary fact subjects may be opened read-only from their
declared materialized coordinates; those documents are closed before the
selected source document closes.

### Native persistence vs. derived artifacts

The runtime distinguishes the configured native document from derived
exported artifacts:

```text
configured native CAD document != derived exported artifact
```

`document.save()` runs unconditionally after parameter/mutation processing and
applicable recompute/validity, independent of
whether any STEP/CSV/PDF export is declared. Execution with zero declared
outputs (`outputs: []`) is supported: the native document is persisted, no
derived export runs, and `prm.result.json` reports `artifacts: []`. The persisted
native document is not itself added to `prm.result.json.artifacts`.

## Process outcomes

Deterministic exit codes:

- `0` success
- `2` FreeCAD unavailable
- `64` invalid arguments
- `70` deterministic execute failure

Handled failures produce deterministic single-line stderr with no traceback.

## Ownership boundary

FreeCAD owns: runtime bootstrapping, the launcher transport, headless
execution, CAD document lifecycle, requested mutation, recompute, requested
observation, export execution, raw runtime result capture, raw reference
traversal evidence, native occurrence and bounded fact inspection, pre-mutation
baseline hashing, post-save configured-result hashing, and raw integrity
evidence.

FreeCAD does not own Engine planning, verification decisions, normalization,
durable product storage/indexing, or higher-level orchestration.

Engine owns (outside this document): executable selection/configuration,
working-copy preparation, request construction, subprocess invocation,
verification comparison and decisions, and output normalization into
Engine-produced records. See [Execution
Runtime](../../runtime/execution-runtime.md) for that side of the pipeline.
