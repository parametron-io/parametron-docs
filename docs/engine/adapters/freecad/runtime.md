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

Separately from the normal external lifecycle documented below, FreeCAD has
implemented and tested standalone native suppression/unsuppression, visibility,
and conservative deletion consumers, plus deterministic post-mutation validity
and dependency inspection. Normal `execute` does not invoke those consumers or
that infrastructure: schema 2.0 target-mutation execution remains unavailable.
Issue #6 owns their future lifecycle placement and translation through the
structured runtime failure boundary; target-state observation remains separate
issue #5 scope. See [Target Mutations](contracts/target-mutations.md) for the
standalone capability semantics.

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
- `--output-dir` is optional for execution-only calls.
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

## Execution lifecycle

For one `execute` call, in order:

1. Validate CLI arguments before importing FreeCAD.
2. Resolve `--working-copy` as the authoritative execution-instance root.
3. Load and validate the manifest (see
   [contracts/manifest.md](contracts/manifest.md)).
4. Resolve `sourceDocument` inside the working copy.
5. If a reference-traversal request is present, load it exactly once (see
   [contracts/reference-traversal.md](contracts/reference-traversal.md)).
6. If an observation request is present, load it and compute the
   source-document SHA-256 digest before the document is opened (see
   [contracts/observation.md](contracts/observation.md)).
7. Open the FreeCAD document.
8. Apply parameter assignments in manifest order.
9. Recompute once.
10. Persist the configured native working document with `document.save()`.
11. Export declared STEP/CSV/PDF artifacts (see
    [contracts/artifacts.md](contracts/artifacts.md)).
12. If reference traversal is enabled, run it on the recomputed, persisted
    live document and emit the canonical raw traversal output.
13. If observation is enabled, observe the same open live document — after
    persistence, exports, and traversal, before document close.
14. Close the document.
15. Write deterministic success `prm.result.json` (see
    [contracts/result-and-failure.md](contracts/result-and-failure.md)).

Live-document observation and traversal both operate on the same open FreeCAD
document instance that received mutation, recompute, and persistence; the
document is not reopened. The original external/project source document
remains outside the mutation target.

### Native persistence vs. derived artifacts

The runtime distinguishes the configured native document from derived
exported artifacts:

```text
configured native CAD document != derived exported artifact
```

`document.save()` runs unconditionally after recompute, independent of
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
observation, export execution, raw runtime result capture, and raw reference
traversal evidence.

FreeCAD does not own Engine planning, verification decisions, normalization,
durable product storage/indexing, or higher-level orchestration.

Engine owns (outside this document): executable selection/configuration,
working-copy preparation, request construction, subprocess invocation,
verification comparison and decisions, and output normalization into
Engine-produced records. See [Execution
Runtime](../../runtime/execution-runtime.md) for that side of the pipeline.
