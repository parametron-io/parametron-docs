# Development Workflow

This document describes the current public Parametron engineering workflow. It
defines stable process rules, not current project status.

Live work state is not tracked in this repository. It lives in GitHub Issues and
in the public organization GitHub Project **Parametron Engineering**.

---

## Where Information Lives

- **Stable/current engineering knowledge** belongs in documentation.
- **Live work** belongs in GitHub Issues.
- **Organization-wide coordination and workflow state** belong in the public
  GitHub Project **Parametron Engineering**.

Documentation describes how the system works and how work is done. Issues and
the Project describe what is currently being done.

---

## Issue Types

- **Phase** — a maintainer-owned, bounded engineering objective with related
  work and explicit exit criteria. A Phase groups the work needed to reach one
  engineering boundary.
- **Task** — a maintainer-owned work-item issue type used for planned repository work under a Phase. Task creation is reserved for maintainers.
- **Bug**, **Feature** — the normal child/work-item issue types
  carried out under a Phase.

For what a Phase is conceptually, see [Phases](../conventions/phases.md).

### Phase Rules

- A Phase may have multiple child work items in progress concurrently.
- The default organizational expectation is normally **one active Phase per
  repository**, unless there is an explicit reason otherwise.
- A Phase is not a GitHub Milestone. Milestones are reserved for real
  release/version grouping.

---

## Live Workflow State

GitHub Project status is the single source of truth for live workflow state:

- **Backlog**
- **Ready**
- **In Progress**
- **In Review**
- **Blocked**
- **Done**

GitHub Issues are the source of truth for live engineering work. The
**Parametron Engineering** Project coordinates that work and records its
workflow state.

---

## Cross-Repository Dependencies

- Use GitHub's native blocked-by / blocking issue relationships to express
  dependencies across repositories.
- Apply the `external` label as a visible signal that an issue has a dependency
  across a repository boundary.
- Do not duplicate a cross-repository dependency by creating placeholder issues
  or cards in a repository that does not own the work.
- The repository that owns the behavior owns the issue, its scope, and its
  completion evidence.

---

## Pull Requests and Completion

- A pull request should link its owning issue where applicable, preferably with
  `Closes #<issue>`.
- Merging the pull request and closing the issue is the normal completion path.
- Closure evidence follows the [Verification Standard](verification-standard.md).
- Commit and PR formatting follows the
  [Commit Message Convention](../conventions/commit-messages.md).

---

## Scope Boundary

This workflow documentation and the public GitHub Project describe current,
public engineering only. Unpublished future architecture, product strategy,
commercial plans, and other private strategic planning do not belong in the
public workflow or project documentation.
