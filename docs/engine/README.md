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
