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
| [docs/engine/adapters/freecad/architecture.md](docs/engine/adapters/freecad/architecture.md) | FreeCAD adapter architecture, ownership boundary, and capability areas |
| [docs/engine/adapters/freecad/runtime.md](docs/engine/adapters/freecad/runtime.md) | FreeCAD headless runtime invocation and execution lifecycle |
| [docs/engine/adapters/freecad/contracts/artifacts.md](docs/engine/adapters/freecad/contracts/artifacts.md) | FreeCAD STEP/CSV/PDF artifact export contract |
| [docs/engine/adapters/freecad/contracts/engine-invocation.md](docs/engine/adapters/freecad/contracts/engine-invocation.md) | FreeCAD in-process Engine invocation contract |
| [docs/engine/adapters/freecad/contracts/manifest.md](docs/engine/adapters/freecad/contracts/manifest.md) | FreeCAD manifest transport and schema contract |
| [docs/engine/adapters/freecad/contracts/observation.md](docs/engine/adapters/freecad/contracts/observation.md) | FreeCAD observation request/response contract |
| [docs/engine/adapters/freecad/contracts/reference-traversal.md](docs/engine/adapters/freecad/contracts/reference-traversal.md) | FreeCAD reference-traversal request/output contract |
| [docs/engine/adapters/freecad/contracts/result-and-failure.md](docs/engine/adapters/freecad/contracts/result-and-failure.md) | FreeCAD result, failure, and trace contracts |
| [docs/engine/adapters/freecad/contracts/target-mutations.md](docs/engine/adapters/freecad/contracts/target-mutations.md) | FreeCAD target-mutation transport/native-capability boundary |
| [docs/engine/architecture/system-overview.md](docs/engine/architecture/system-overview.md) | Current Engine pipeline and architecture |
| [docs/engine/architecture/execution-model.md](docs/engine/architecture/execution-model.md) | Jobs, handoff, scheduling, attempts, and execution semantics |
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
| [docs/engine/runtime/api-overview.md](docs/engine/runtime/api-overview.md) | Engine HTTP API overview |
| [docs/engine/runtime/artifact-serving.md](docs/engine/runtime/artifact-serving.md) | Artifact listing and byte-serving behavior |
| [docs/engine/runtime/execution-runtime.md](docs/engine/runtime/execution-runtime.md) | CAD runtime processing, verification, and cache behavior |
| [docs/engine/runtime/job-artifacts.md](docs/engine/runtime/job-artifacts.md) | Job-scoped artifact listing contract |
| [docs/engine/runtime/job-lifecycle.md](docs/engine/runtime/job-lifecycle.md) | API job states, transitions, and failure details |
| [docs/engine/runtime/job-submission.md](docs/engine/runtime/job-submission.md) | Job submission payload and validation contract |
| [docs/engine/runtime/reporting.md](docs/engine/runtime/reporting.md) | Execution report schema and deterministic behavior |
| [docs/engine/reference/json-table-resources.md](docs/engine/reference/json-table-resources.md) | JSON table format, validation, fingerprinting, and lookup |
| [docs/engine/reference/record-contracts.md](docs/engine/reference/record-contracts.md) | Normalized Engine record and package contracts |
| [docs/engine/reference/target-action-contract.md](docs/engine/reference/target-action-contract.md) | Semantic target-action validation and runtime projection |
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
│       │   ├── freecad
│       │   │   ├── contracts
│       │   │   │   ├── artifacts.md
│       │   │   │   ├── engine-invocation.md
│       │   │   │   ├── manifest.md
│       │   │   │   ├── observation.md
│       │   │   │   ├── reference-traversal.md
│       │   │   │   ├── result-and-failure.md
│       │   │   │   └── target-mutations.md
│       │   │   ├── architecture.md
│       │   │   └── runtime.md
│       │   └── README.md
│       ├── architecture
│       │   ├── execution-model.md
│       │   └── system-overview.md
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
│       ├── reference
│       │   ├── json-table-resources.md
│       │   ├── record-contracts.md
│       │   └── target-action-contract.md
│       ├── runtime
│       │   ├── api-overview.md
│       │   ├── artifact-serving.md
│       │   ├── execution-runtime.md
│       │   ├── job-artifacts.md
│       │   ├── job-lifecycle.md
│       │   ├── job-submission.md
│       │   └── reporting.md
│       └── README.md
├── CONTRIBUTING.md
├── LICENSE
├── project-structure.md
└── README.md

13 directories, 48 files
```
