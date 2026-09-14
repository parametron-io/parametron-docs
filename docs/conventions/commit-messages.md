# Commit Message Convention

This document defines the commit message style used across Parametron repositories.

The goal is to keep commits readable, searchable, and explicit about scope, intent, and impact.

---

## Format

Parametron commits use a structured format:

```text
<type>(<scope>): <summary>

- <change detail>
- <change detail>
- <change detail>

<optional closing note>
```

---

## Rules

### 1. Subject line

The first line must follow this format:

```text
<type>(<scope>): <summary>
```

Examples:

```text
feat(engine): introduce integration manifest for FreeCAD runner resolution
fix(api): reject invalid job IDs before submission lookup
docs(engine): align FreeCAD runner documentation with integration manifest model
test(cli): add project-based execution rehearsal coverage for root CLI
```

---

### 2. Supported types

Common commit types include:

- `feat` — new functionality
- `fix` — bug fix or behavior correction
- `docs` — documentation-only changes
- `test` — test additions or test corrections
- `refactor` — structural change without intended behavior change
- `chore` — repository or maintenance work

---

### 3. Scope

The scope identifies the primary area affected.

Examples:

- `engine`
- `cli`
- `api`
- `adapter`
- `docs`
- `repo`

Use a single clear scope whenever possible.

---

### 4. Summary line

The summary should:

- be specific
- describe the main effect of the change
- avoid vague phrases like "update stuff" or "fix issues"

Good:

- `test(cli): add project-based execution rehearsal coverage for root CLI`

Bad:

- `test(cli): update tests`

---

### 5. Body

The body is written as a bullet list.

Each bullet should describe a concrete change, guarantee, or verification point.

Preferred style:

- implementation changes
- contract effects
- determinism guarantees
- validation added
- compatibility notes

---

### 6. Closing note

Use a short closing note when it adds clarity.

Examples:

- `no production code changes`
- `test-only change`
- `documentation-only update`

This is especially useful for docs-only or test-only commits.

---

## Issue Referencing

Commits are linked directly to GitHub Issues. These rules apply to Phase, Task,
Bug, and Feature issues.

### 1. Referencing an issue

To associate an intermediate commit with its owning issue in the same
repository without closing it:

```text
#<issue-number>
```

Example:

```text
test(cli): extend execution rehearsal coverage

- add additional failure path validation
- improve report.json inspection

#6
```

This links the commit to the issue for traceability.

For an issue in another repository, use GitHub's owner/repository-qualified
form:

```text
parametron-io/<repository>#<issue-number>
```

Example:

```text
parametron-io/parametron-engine#45
```

---
### 2. Closing an issue

On the final change intended to close the owning issue, use a closing keyword
when automatic closure is appropriate:

```text
closes #<issue-number>
```

Example:

```text
test(cli): finalize project-based execution rehearsal coverage

- complete validation coverage for success and failure paths
- confirm deterministic cache behavior
- finalize report.json verification

closes #6
```

The issue will be closed automatically when the commit is merged.

---

### 3. Usage rules

- Intermediate commits may use `#<issue-number>` to reference the owning issue.
- Use `closes #<issue-number>` only on the final change intended to close the
  owning issue, when automatic closure is appropriate.
- Do not use a closing keyword before all applicable tests, documentation,
  verification, and exit criteria are complete.

---

### 4. Multiple issues

If a commit relates to multiple issues:

```text
#6 #7
```

If closing multiple issues:

```text
closes #6
closes #7
```

---

## Example

```text
test(cli): add project-based execution rehearsal coverage for root CLI

- add TestCLI_ProjectBasedExecution_RehearsalPass
- add TestCLI_ProjectBasedExecution_RehearsalFailurePath
- prove end-to-end root CLI execution via `--project` entrypoints
  (directory and parametron.project.json)
- assert successful runs produce report.json, metadata.json,
  manifest.json, and declared artifacts
- verify metadata includes project-captured inputs and table participation
- confirm deterministic cache reuse on identical runs
- confirm cache invalidation on model and table changes
- verify project-mode execution does not require parametron.lock.json
- assert deterministic failure reporting without requiring real FreeCAD
- extend test-only helper to inspect run-root state and report.json

no production code changes
#5
```

---

## Guidance

A Parametron commit should answer:

- What changed?
- Where did it change?
- Why does it matter?
- Did behavior change, or only tests/docs?

A reader should understand the intent of the commit without opening the diff first.

---

## Notes

This convention is preferred across all Parametron repositories unless a repository defines a stricter local rule.
