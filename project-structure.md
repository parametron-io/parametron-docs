# Project Structure

This is a structural index. For conceptual guidance and reading order, see
[README.md](README.md).

## File Index

| File | Purpose |
|------|---------|
| [README.md](README.md) | Repository entry point and navigation hub |
| [CONTRIBUTING.md](CONTRIBUTING.md) | How to propose and validate documentation changes |
| [LICENSE](LICENSE) | CC BY-SA 4.0 license text |
| [project-structure.md](project-structure.md) | This file — structural index |
| [docs/architecture/product-family.md](docs/architecture/product-family.md) | Minimum naming and product-boundary context |
| [docs/architecture/system-overview.md](docs/architecture/system-overview.md) | Current Engine + FreeCAD engineering system overview |
| [docs/architecture/repo-boundaries.md](docs/architecture/repo-boundaries.md) | Current public engineering ownership rules |
| [docs/engine/README.md](docs/engine/README.md) | Canonical current Engine engineering documentation |
| [docs/engine/adapters/README.md](docs/engine/adapters/README.md) | Current Engine adapter-domain contracts and ownership |
| [docs/engine/authoring/cad-contract.md](docs/engine/authoring/cad-contract.md) | CAD capture schema and validation contract |
| [docs/engine/authoring/dsl-grammar.md](docs/engine/authoring/dsl-grammar.md) | Parametron DSL syntax reference |
| [docs/engine/authoring/dsl-overview.md](docs/engine/authoring/dsl-overview.md) | Parametron DSL structure and authoring overview |
| [docs/engine/authoring/dsl-semantics.md](docs/engine/authoring/dsl-semantics.md) | Parametron DSL evaluation and semantic rules |
| [docs/engine/authoring/ir-and-planning.md](docs/engine/authoring/ir-and-planning.md) | Engine intermediate representation and deterministic planning |
| [docs/engine/authoring/project-mapping.md](docs/engine/authoring/project-mapping.md) | Project mapping schema, resolution, and validation contract |
| [docs/engine/cli/command-families.md](docs/engine/cli/command-families.md) | CLI execution modes and command selection |
| [docs/engine/cli/diff.md](docs/engine/cli/diff.md) | Plan and snapshot comparison command reference |
| [docs/engine/cli/runtime-behavior.md](docs/engine/cli/runtime-behavior.md) | Shared CLI configuration, output, and cache behavior |
| [docs/engine/cli/simulate.md](docs/engine/cli/simulate.md) | Multi-case simulation command reference |
| [docs/engine/cli/snapshot.md](docs/engine/cli/snapshot.md) | Snapshot package command reference |
| [docs/engine/cli/sweep.md](docs/engine/cli/sweep.md) | Parameter sweep command reference |
| [docs/engine/cli/validate.md](docs/engine/cli/validate.md) | DSL and project validation command reference |
| [docs/development/workflow.md](docs/development/workflow.md) | GitHub-native engineering workflow and coordination rules |
| [docs/development/verification-standard.md](docs/development/verification-standard.md) | Canonical definition of done for all phases and issues |
| [docs/conventions/phases.md](docs/conventions/phases.md) | What a phase is: definition, characteristics, ownership, naming |
| [docs/conventions/naming.md](docs/conventions/naming.md) | Naming rules for repositories, phases, files, and contracts |
| [docs/conventions/commit-messages.md](docs/conventions/commit-messages.md) | Commit format, types, scopes, and issue referencing |

## Directory Tree

```
.
├── docs
│   ├── architecture
│   │   ├── product-family.md
│   │   ├── repo-boundaries.md
│   │   └── system-overview.md
│   ├── conventions
│   │   ├── commit-messages.md
│   │   ├── naming.md
│   │   └── phases.md
│   ├── development
│   │   ├── verification-standard.md
│   │   └── workflow.md
│   └── engine
│       ├── adapters
│       │   └── README.md
│       ├── authoring
│       │   ├── cad-contract.md
│       │   ├── dsl-grammar.md
│       │   ├── dsl-overview.md
│       │   ├── dsl-semantics.md
│       │   ├── ir-and-planning.md
│       │   └── project-mapping.md
│       ├── cli
│       │   ├── command-families.md
│       │   ├── diff.md
│       │   ├── runtime-behavior.md
│       │   ├── simulate.md
│       │   ├── snapshot.md
│       │   ├── sweep.md
│       │   └── validate.md
│       └── README.md
├── CONTRIBUTING.md
├── LICENSE
├── project-structure.md
└── README.md

9 directories, 27 files
```
