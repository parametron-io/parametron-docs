# Naming Conventions

This document defines naming rules used across the current public Parametron
engineering repositories.

The goal is consistency, clarity, and predictability across repositories,
phases, issues, contracts, and engineering vocabulary.

---

## General Principles

- Names must be explicit and descriptive
- Avoid abbreviations unless standard, such as API, CLI, DSL, IR, CAD, or BOM
- Prefer clarity over brevity
- Use consistent terminology across all repositories
- Avoid overloading terms with multiple meanings
- Name the owner of a responsibility, not just the mechanism used to implement it

---

## Repository Naming

Repositories follow the format:

```text
parametron-<component>
```

Examples:

- `parametron-docs`
- `parametron-engine`
- `parametron-freecad`

Rules:

- Lowercase only
- Use hyphen-separated words
- Component name must reflect responsibility
- Avoid generic names like `core`, `utils`, or `system`

---

## Phase Naming

A Phase is a GitHub Issue whose issue type is `Phase`. GitHub supplies the
Phase identity through the issue number in the owning repository.

```text
#45
```

Across repositories, use GitHub's owner/repository-qualified form:

```text
parametron-io/parametron-engine#45
```

Rules:

- Use `#<number>` to reference a Phase inside its owning repository.
- Use `parametron-io/<repository>#<number>` when referencing a Phase across
  repositories.
- Titles must describe the bounded engineering outcome or objective.
- Keep titles concise and explicit; avoid vague names such as "Improvements"
  or "Refactor".
- Do not create or maintain a separate Phase-number system.

---

## Cross-Repository Dependency Signalling

Live work items are GitHub Issues owned by the repository that owns the
behavior. There is no separate card-title format.

When an issue depends on work across a repository boundary:

- Use GitHub's native blocked-by / blocking issue relationships to record the
  dependency.
- Apply the `external` label as a visible signal of the boundary crossing.
- Do not create a placeholder issue in a repository that does not own the work.

The `external` label is reserved for cross-repository dependencies. See
[Workflow](../development/workflow.md) for the full coordination rules.

---

## Commit Scope

Commit scope rules are defined in [Commit Message Convention](commit-messages.md).

---

## File Naming

- Use lowercase with hyphens for multi-word files
- Use `.md` for documentation
- Use `.json` for structured contracts
- Use `.yaml` or `.yml` for OpenAPI or other YAML contracts when the owning
  repository uses YAML

Examples:

- `verification-standard.md`
- `workflow.md`
- `prm.verification.json`
- `prm.observed.json`

---

## JSON Contract Naming

Parametron-owned JSON contract filenames MUST follow this canonical form:

```text
prm.<semantic-name>.json
```

Examples:

- `prm.export-manifest.json`
- `prm.result.json`
- `prm.observed.json`
- `prm.verification.json`
- `prm.reference-traversal.json`
- `prm.reference-traversal-request.json`

Rules:

- `prm.` is the reserved **contract filename namespace** for Parametron JSON
  contract files. It is a filename namespace abbreviation, not a general product
  abbreviation, and MUST NOT be reused as the naming basis for unrelated
  identifiers.
- The **semantic name** identifies the contract's purpose. It MUST be lowercase,
  explicit, descriptive, and single-purpose.
- Multi-word semantic names MUST use hyphens. Unnecessary abbreviations such as
  `ref-trav` SHOULD be avoided; use `reference-traversal`.
- Schema versions MUST NOT appear in filenames. Schema versions MUST be carried
  inside the contract, for example through `schemaVersion`.

Names such as `export_manifest_v1.json` are deprecated as active
transport/configuration filenames: they encode a schema version in the filename,
lack the canonical namespace, and use underscore-separated semantic naming.
Generic active contract filenames such as `result.json` are migration candidates
because they lack the Parametron contract namespace.

These rules define canonical naming policy, not completed implementation state.
Current implementation documentation may still describe older names. Migration
of existing active surfaces is future work in the owning repositories; this
policy does not rename files, change schemas or runtime behavior, or revise the
record-package layout.

### Active Contract Filenames and Preserved Raw-Evidence Filenames

An **active contract surface** is a transport or configuration boundary where a
contract file is produced, selected, or consumed for current execution. Its
**active contract filename** identifies the file at that boundary.

**Preserved raw evidence** is received/generated transport material retained as
evidence of an execution or historical event. A **preserved raw-evidence
filename** retains that material's original transport identity for provenance.
Raw runtime output can serve as an active contract before being preserved as
evidence; calling it raw evidence does not exempt its active surface.

| Role | Naming policy |
| --- | --- |
| Active transport/configuration contract surface | MUST use `prm.<semantic-name>.json` as the canonical filename. |
| Preserved historical/raw evidence | MAY retain the original filename when retaining the received/generated transport identity is necessary for provenance. |

New active contract surfaces MUST use canonical naming. Once an existing
surface's relevant migration is complete, newly produced active runtime,
transport, or configuration contracts MUST NOT retain the old filenames.

Names such as `parametron.*.json`, `result.json`, and `export_manifest_v1.json`
MUST NOT automatically be treated as globally forbidden strings. They MAY remain
in preserved raw evidence, historical evidence, migration history, and
tests/fixtures that intentionally prove compatibility or provenance, when that
use is semantically legitimate. Preservation MUST NOT be used to justify
continued production under an old name on a migrated active surface.

Migration audits MUST ask: **"Is the old filename still used by an active
contract surface?"** They MUST NOT use **"Does the old filename exist anywhere?"**
as the migration-completion criterion.

An old contract filename is not automatically a retired planning/product term.
Filename migration policy and [Legacy Terms](#legacy-terms) policy are separate
concerns; preserved evidence filenames MUST NOT automatically be placed in a
generic Legacy Terms category.

---

## Environment Variable Naming

Parametron-owned environment variables MUST retain the canonical product prefix
`PARAMETRON_` and use uppercase, underscore-separated words. The JSON filename
namespace `prm.` MUST NOT be used to derive environment-variable naming or imply
a migration to `PRM_`.

### Component Configuration

**Component configuration** supplies settings for a component, such as its
executable or operating mode. Names MUST follow:

```text
PARAMETRON_<COMPONENT>_<NAME>
```

`<COMPONENT>` identifies the configured component; `<NAME>` describes the setting.
Examples:

- `PARAMETRON_FREECAD_BIN`
- `PARAMETRON_FREECAD_STRICT_SMOKE`
- `PARAMETRON_FREECAD_RUNTIME`

### Behavior, Test, and Execution Toggles

A **behavior/test toggle** controls whether a workflow, test, integration, or
execution path runs, rather than supplying component configuration. Such toggles
MAY use an action-oriented name under `PARAMETRON_` when it communicates that
purpose more clearly than the component configuration form.

`PARAMETRON_RUN_FREECAD_INTEGRATION` is an explicit example of this category:
its action-oriented name expresses whether the FreeCAD integration path runs.
It is not an accidental violation of the component configuration convention and
does not require renaming under this standard. A boolean value alone does not
determine the category; the variable's purpose does. These naming categories
define policy without renaming existing variables or changing their behavior.

---

## Engine Terminology

Use `standalone local/headless Engine` for local CLI/API execution,
diagnostics, validation, plan preview, and deterministic engineering runs.

Engine-produced output may be described as `PDM-ready` / `PDM-independent`
where that reflects existing Engine contracts. Do not use `engine` as
shorthand for all orchestration.

Ownership language must match the architecture:

- Use `validates`, `plans`, `executes`, `normalizes`, `verifies`, or
  `produces records` for Engine engineering behavior
- Use `opens`, `observes`, `mutates`, `exports`, or `returns runtime results`
  for CAD adapter behavior

Avoid wording that makes Engine a global workflow policy owner or a CAD
adapter a direct writer to any external store.

---

## Adapter Terminology

- `adapter` -> runtime-specific CAD execution integration behind Engine
  contracts
- `freecad` / `CAD adapter` -> a runtime-specific CAD execution implementation
- `bridge` -> only for legacy or explicitly named observation bridging code

CAD adapters return raw evidence to Engine. Engine validates and normalizes it.

---

## Data and Record Terminology

- `raw evidence` -> umbrella term for preserved runtime and operational evidence
  under a record package's `raw/` area
- `raw runtime evidence` -> narrower external-runtime evidence subset allowlisted
  under `raw/runtime/`
- `reference traversal` -> FreeCAD-owned runtime process or capability that
  discovers CAD references and emits traversal evidence
- `raw traversal evidence` -> serialized raw evidence produced by reference
  traversal, currently `parametron.reference-traversal.json`
- `reference record` -> Engine-owned normalized record derived from traversal
  evidence
- `engineering facts` -> engineering results produced by Engine
- `Engine-produced records` -> durable records produced by Engine, such as
  execution, artifact, observation, reference graph, BOM/component, failure,
  and verification records
- `baseline reference graph capability` -> Engine capability for producing
  reference graph records
- `DSL-defined or policy-requested artifacts` -> artifacts generated only when
  DSL intent, user action, or CI explicitly requests them

See [Engine-produced record contracts](../engine/reference/record-contracts.md)
for the canonical record-package layout and evidence contract details.

Avoid implying that references are extracted by reading CAD files outside the
Engine + adapter contract, that BOMs are generated by default, or that exports
happen without explicit intent.

---

## Execution and Verification Vocabulary

Use consistent terminology across all repositories:

- "execution" for Engine or CAD-runtime execution work
- "observation" for runtime state captured through Engine-owned contracts
- "verification" for Engine-owned expected-vs-observed decision logic where
  applicable
- "validation" for syntax, schema, semantic, or contract checking
- "adapter" for runtime-specific CAD execution integration
- "reference graph" for Engine-produced dependency/reference records
- "artifact intent" for declared or policy-requested output intent

Do not mix synonyms across documentation and code when a canonical term
exists.

---

## Documentation Classification Labels

Use these labels consistently in documentation notes and review comments:

- `canonical` — current source of truth in `parametron-docs`
- `repo-local` — stays in the owning repository
- `legacy` — superseded but historically useful
- `transitional` — describes a temporary shortcut or prototype path
- `test-only` — supports fixtures or harnesses only
- `generated` — produced from code or tooling

---

## Legacy Terms

The following terms appear in older documents and repository history. They are
not current target vocabulary and should only be used when explicitly marked
legacy or historical:

- `Engine Authoring Core` -> legacy split term; prefer `standalone
  local/headless Engine`
- `Engine Runtime Service` -> legacy split term; prefer `standalone
  local/headless Engine`

---

## Reserved Terms

The following terms have fixed meanings and must not be redefined:

- Engine
- CAD adapter
- FreeCAD
- engineering facts
- Engine-produced records
- baseline reference graph capability
- DSL-defined artifact
- policy-requested artifact
- Phase
- `external` (label)

---

## Goal

These conventions ensure:

- consistent naming across repositories
- reduced ambiguity
- easier onboarding for contributors
- predictable system structure
