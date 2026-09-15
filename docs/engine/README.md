# Parametron Engine

Engine is Parametron's standalone, local/headless execution and planning layer.
This area owns canonical current Engine engineering documentation that is useful
across repository boundaries. Repository-specific contributor instructions,
build and test commands, package layout, and maintenance details remain in
`parametron-engine`.

Documentation follows one ownership rule:

> one fact -> one canonical documentation owner -> references elsewhere

The cross-repository architecture remains canonical in the
[System Overview](../architecture/system-overview.md) and
[Repository Boundaries](../architecture/repo-boundaries.md). This area links to
those documents instead of repeating the full system architecture.

## Authoring

Start with the [DSL overview](authoring/dsl-overview.md), then use the
[grammar](authoring/dsl-grammar.md) and [semantics](authoring/dsl-semantics.md)
references. The authoring area also defines [IR and planning](authoring/ir-and-planning.md),
the [project mapping contract](authoring/project-mapping.md), and the
[CAD capture contract](authoring/cad-contract.md).

## CLI

See [CLI command families](cli/command-families.md) for command selection and
[CLI runtime behavior](cli/runtime-behavior.md) for shared configuration and
output rules. Command references cover [validate](cli/validate.md),
[simulate](cli/simulate.md), [sweep](cli/sweep.md),
[snapshot](cli/snapshot.md), and [diff](cli/diff.md).

## Adapters

Adapter documentation belongs to the Engine adapter domain. See the
[Adapter Domain](adapters/README.md) for the current Engine-owned boundary
between planning and CAD-native execution. FreeCAD is the currently implemented
Engine CAD adapter.

Centralizing engineering documentation does not transfer implementation
ownership. Implementation repositories remain authoritative for their source
code, schemas, tests, and repository-local development details. Engine
implementation and Engine-owned schema/contract sources remain in
`parametron-engine`; CAD-native FreeCAD implementation and FreeCAD-owned
schema/contract sources remain in `parametron-freecad`.
