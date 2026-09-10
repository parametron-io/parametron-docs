# Phase System

This document defines how phases are structured and used across the Parametron ecosystem.

Phases are the primary unit of development planning and execution. Each phase represents a well-defined, bounded objective with explicit scope and completion criteria.

---

## Purpose

The phase system exists to:

- structure development into clear, incremental steps

- define stable checkpoints across repositories

- enable deterministic progress tracking

- support cross-repository coordination


---

## Phase Definition

A phase must include:

- **Goal** — what the phase achieves

- **Scope** — what is included (and implicitly excluded)

- **Dependencies** — required prior work

- **Exit Criteria** — conditions for completion


Optional:

- **Next Focus** — where development moves after completion

---

## Phase Characteristics

Each phase must be:

- **bounded** — clearly limited in scope

- **deterministic** — produces predictable, repeatable outcomes

- **verifiable** — completion can be objectively checked

- **independent** — does not rely on undefined future work


---

## Phase Ownership

Phases belong to a specific repository.

Examples:

- Engine phases → defined in `parametron-engine`

- FreeCAD phases → defined in `parametron-freecad`


Rules:

- A phase is always defined by its owning repository

- Other repositories may depend on it but do not redefine it

- Phase identifiers must remain stable


---

## Phase Naming

Phases follow this format:

```
Phase <number> — <Title>
```

Examples:

- `Phase 7.30B — CAD Observation Layer`

- `Phase 7.30C — Execution Verification Layer`

- `Phase 4 — Engine Integration Gate Alignment`


Rules:

- Numbers are identifiers, not ordering guarantees across repositories

- Titles describe outcomes, not actions

- Avoid vague names such as "Improvements" or "Cleanup"


For naming format details across all identifiers, see [Naming Conventions](naming.md).

---

## Phase Progression and Project Status

Phases and their child work items move through the GitHub Project status
columns from Backlog to Done.

For live workflow state, status semantics, and the one-active-Phase-per-repository
expectation, see [Workflow](../development/workflow.md).

---

## Phase Closure

Phase closure requirements are defined in [Verification Standard](../development/verification-standard.md).

---

## Phase Granularity

Phases should be:

- large enough to represent meaningful progress

- small enough to be completed without long-term drift


Avoid:

- overly small phases (task-level)

- overly large phases (multi-system scope)


---

## Goal

The phase system ensures:

- structured and predictable development

- clear ownership and responsibility

- explicit dependency management

- consistent progress across the Parametron ecosystem