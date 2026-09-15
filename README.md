# Parametron Docs

Public engineering documentation for Parametron, an early-stage project for
deterministic engineering automation and CAD runtime integration.

Parametron is early-stage. The current public engineering focus is **Engine +
FreeCAD**: a deterministic, PDM-independent local/headless runtime pipeline.

- `parametron-engine` — deterministic engineering validation, planning,
  execution, adapter contracts, and normalized engineering record production.
- `parametron-freecad` — FreeCAD runtime operations behind Engine-owned
  contracts.

## Scope of This Repository

This repository owns the **public cross-repository engineering architecture,
conventions, and workflow documentation** for the current implementation focus.

- Implementation-local documentation stays in the repository that owns the
  behavior.
- Other Parametron product names and product lines exist, but their
  architecture and plans are outside the scope of this repository.
- A reference here does not imply that a component is available or
  production-ready.

## Where to Start

| Guide | Purpose |
| --- | --- |
| [Product Family](docs/architecture/product-family.md) | Minimum naming and product-boundary context |
| [System Overview](docs/architecture/system-overview.md) | Current Engine + FreeCAD engineering flow |
| [Repository Boundaries](docs/architecture/repo-boundaries.md) | Current public engineering ownership rules |
| [Engine Documentation](docs/engine/README.md) | Canonical current Engine engineering documentation |
| [Development Workflow](docs/development/workflow.md) | GitHub-native work state and cross-repository coordination |
| [Phase System](docs/conventions/phases.md) | Phase definitions and characteristics |
| [Naming Conventions](docs/conventions/naming.md) | Shared terminology and naming rules |
| [Commit Messages](docs/conventions/commit-messages.md) | Commit format and issue references |
| [Verification Standard](docs/development/verification-standard.md) | Validation and completion evidence |

For the documentation inventory, see [Project Structure](project-structure.md).

Live engineering work is tracked in GitHub Issues and coordinated in the public
organization GitHub Project **Parametron Engineering**, not in this repository.

## Contributing

Documentation corrections, clarifications, and focused proposals are welcome.
Read [Contributing](CONTRIBUTING.md) before preparing a change.

## License

Documentation in this repository is licensed under the
[Creative Commons Attribution-ShareAlike 4.0 International License](https://creativecommons.org/licenses/by-sa/4.0/).
