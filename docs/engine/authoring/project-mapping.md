# Project Mapping (`parametron.project.json`)

The project mapping file links a DSL file and logical resource IDs to project-relative physical paths. It is the project-level input declaration contract for Parametron.

---

## Purpose

In project-mode flows, a DSL file's `source_model` and table references are interpreted as logical IDs by the project mapping layer, not as physical paths. The project mapping file declares where those resources live relative to the project directory. This separation keeps the DSL file free of physical path dependencies within a project.

When present, a conventional project-root `parametron.cad.json` is also loaded as a capture contract for an additional project-mode authoring/planning-time validation boundary.

When `parametron validate --project` (or any execution entrypoint with `--project`) receives a project directory or project mapping file, it loads and validates the mapping before any DSL processing begins.

---

## File Location and Name

The file must be named exactly `parametron.project.json`. It lives at the root of the project directory.

When `--project` is given a directory, the CLI looks for `parametron.project.json` inside that directory. When `--project` is given the file path directly, it is used as-is.

---

## Schema

The file is a JSON object with the following fields:

| Field | Type | Required | Description |
|:------|:-----|:---------|:------------|
| `version` | string | Yes | Schema version. Strict `major.minor` numeric string. Currently supported: `"1.0"`. |
| `projectId` | string | Yes | Non-empty project identifier. |
| `dsl` | string | Yes | Project-relative path to the DSL file. |
| `resources` | object | Yes | Resource declarations container. |
| `resources.models` | object | Yes | Map from logical model ID to project-relative model file path. |
| `tables` | object | No | Map from logical table ID to project-relative JSON table file path. |

### Example — minimal

```json
{
  "version": "1.0",
  "projectId": "my-project",
  "dsl": "model.dsl",
  "resources": {
    "models": {
      "box_model": "input/box.FCStd"
    }
  }
}
```

### Example — with table

```json
{
  "version": "1.0",
  "projectId": "my-project",
  "dsl": "model.dsl",
  "resources": {
    "models": {
      "box_model": "input/box.FCStd"
    }
  },
  "tables": {
    "fasteners": "tables/fasteners.json"
  }
}
```

---

## Field Rules

### `version`

The `version` field is required and must be a string in strict `major.minor` numeric format.

- Accepted syntax: digits only, exactly two dot-separated numeric components (e.g. `"1.0"`). No leading-zero forms (e.g. `"01.0"`), trimming, normalization, or coercion are applied.
- Currently supported version: exactly `"1.0"`.
- Missing or non-string `version` is rejected deterministically.
- Malformed version strings are rejected deterministically.
- Unsupported newer same-major versions (e.g. `"1.1"`, `"1.9"`) are rejected deterministically.
- Unsupported major versions (e.g. `"0.9"`, `"2.0"`) are rejected deterministically.
- Version validation occurs before any project resource is accessed or path validation runs.

### `projectId`

Must be a non-empty string. Must not contain leading or trailing whitespace.

### `dsl`

A project-relative path to the DSL file. Subject to path rules below.

### `resources.models`

A map of logical model IDs to project-relative paths. Required; an empty map is valid but `resources.models` itself must be present.

### `tables`

Optional. A map of logical table IDs to project-relative paths. When absent, no project-level tables are loaded.

---

## Unknown Fields

Unknown fields at the document root and within the `resources` object are rejected deterministically. All map values in `resources.models` and `tables` must be strings; non-string values are rejected.

---

## Logical ID Rules

- Logical IDs (all map keys in `resources.models` and `tables`) must be non-empty strings.
- Logical IDs must not contain leading or trailing whitespace.
- Logical ID lookup is case-sensitive. `box_model` and `Box_Model` are distinct IDs.
- No normalization of any kind is applied to logical IDs before lookup.

---

## Path Rules

All declared paths (`dsl`, model paths, table paths) are subject to the following rules:

- **Relative only**: absolute paths are rejected. This includes slash-prefixed paths (`/foo`), UNC paths (`\\server\share`), and Windows drive paths (`C:\foo`).
- **Must stay within project root**: paths containing parent traversal segments that would escape the project directory are rejected (e.g. `../input/box.FCStd`).
- **Must not normalize to `.`**: paths such as `a/..` or `dir/../` that resolve to the current directory are rejected.
- **No null bytes**: paths containing `\x00` are rejected.
- **No surrounding whitespace**: paths with leading or trailing whitespace are rejected.
- **Normalized form**: accepted paths are returned in forward-slash, `path.Clean`-normalized form.

---

## Error Classification

The `projectmap` package exposes three sentinel errors:

| Error | Type wrapper | Condition |
|:------|:-------------|:---------|
| `ErrIO` | `*FileError` | File read failure (OS-level I/O error) |
| `ErrDecode` | `*DecodeError` | JSON parse failure or trailing JSON |
| `ErrValidation` | `*ValidationError` | Schema or path constraint violation |

Use `errors.Is` to distinguish error classes:

```go
if errors.Is(err, projectmap.ErrValidation) { ... }
if errors.Is(err, projectmap.ErrIO) { ... }
if errors.Is(err, projectmap.ErrDecode) { ... }
```

`ValidationError` carries a slice of human-readable problem strings. Multiple problems are collected and reported together. Problem ordering is deterministic.

---

## Determinism

- Repeated calls with identical input produce identical output.
- Validation problem ordering is deterministic.
- Logical ID iteration order is sorted before validation.

---

## Relation to DSL `source_model`

In project-mode flows, the DSL product's `source_model` field is interpreted as a logical model ID via the project mapping layer:

```dsl
product Box {
  adapter = "freecad"
  source_model = "box_model"
  outputs = ["step"]

  param length: number = 35
}
```

The string `"box_model"` is a key in the project mapping's `resources.models`. At CLI validate time:

1. The project mapping is loaded.
2. The DSL is parsed.
3. Before plan generation, `source_model = "box_model"` is looked up in `resources.models`.
4. If the key does not exist, validation fails immediately with an error identifying the unknown ID.
5. If a project-root `parametron.cad.json` is present, `source_model` must also exactly match `capture.sourceDocument.logicalId`.
6. If the key exists and any capture-backed `source_model` check passes, the resolved physical path is used for execution.

The DSL file itself does not contain or reference physical paths. Physical paths live only in the project mapping.

Current capture-backed project-mode validation is intentionally narrow. It covers `source_model` consistency and numeric DSL parameters that feed manifest `parameterAssignments[]`. It does not yet imply that all CAD-backed groups, mutation targets, metadata references, or output scopes are expressible and resolved through the current DSL surface.

---

## Table Loading

Tables declared in the `tables` field are loaded by logical ID when a project entrypoint is used. These tables are made available to the planner by the same logical IDs used in `table_cell` DSL expressions.

`--table` inputs are merged after project tables are loaded. If a flag-specified table shares a logical ID with a project table, validation fails deterministically with a duplicate logical ID error.

## Related Documents

- [Validate](../cli/validate.md) — project directory and project file as CLI entrypoints
- [DSL overview](dsl-overview.md) — DSL file structure, profile settings, and product-level execution declarations including `source_model`
- [JSON table resources](../reference/json-table-resources.md) — JSON table file format
