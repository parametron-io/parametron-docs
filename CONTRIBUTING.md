# Contributing to Parametron Docs

Contributions are welcome. Parametron is early-stage, and focused corrections, clearer explanations, and well-scoped proposals help improve its documentation.

## Understand the Scope

Before proposing a change, read the [Product Family](docs/architecture/product-family.md), [System Overview](docs/architecture/system-overview.md), and [Repository Boundaries](docs/architecture/repo-boundaries.md).

This repository owns cross-repository architecture, conventions, and coordination documentation. Implementation changes and code-specific documentation belong in the repository that owns the behavior. Preserve the documented ownership boundaries and use the current [Naming Conventions](docs/conventions/naming.md).

## Keep Changes Focused

Use issues or available discussion channels to coordinate proposed changes, especially work that affects multiple repositories. Explain the problem, intended scope, and affected documentation before substantial work.

Substantial implementation work should follow the [Development Workflow](docs/development/workflow.md) and [Phase System](docs/conventions/phases.md), with a bounded objective, explicit dependencies, and verifiable exit criteria in the owning repository. Keep unrelated cleanup separate from the change being proposed.

## Validate Your Change

For documentation changes:

- Check that relative links resolve and examples agree with the canonical documentation.
- Preserve the distinction between current implementation, future design, and historical material.
- Run `git diff --check` and review the diff for unintended changes.

For implementation changes in the owning repository, include applicable tests and validation evidence following its local instructions and the [Verification Standard](docs/development/verification-standard.md). Synchronize affected documentation when behavior or contracts change.

Follow the [Commit Message Convention](docs/conventions/commit-messages.md) for subjects, scopes, and issue references. Describe what changed and how it was validated when submitting the change.

Before committing, review the files being included. Do not commit secrets, credentials, local environment data, generated artifacts, or unrelated files.
