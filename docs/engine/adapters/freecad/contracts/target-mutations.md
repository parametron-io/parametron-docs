# Target Mutations

"Target mutation" means suppressing/unsuppressing, changing visibility of, or
deleting a named CAD feature or object, as opposed to the scalar property
writes already handled by `parameterAssignments` (see
[manifest.md](manifest.md)).

## Ownership split

Engine owns semantic target-action intent: authoring, semantic-target
resolution, capability validation, lowering to mutation intent, routing to
Part/Assembly destinations, and schema 2.0 manifest projection. That
ownership, including the current runtime limitation this document describes,
is canonical in [Target-Action
Contract](../../../reference/target-action-contract.md).

This document owns the FreeCAD adapter's transport and native-capability
boundary for the same mutations: what schema 2.0 declares, what FreeCAD
validates, and what FreeCAD currently executes.

## Current implementation status

Three layers exist for this capability, at different levels of completeness:

```text
Layer 1 — Contract/Type Metadata:   implemented
Layer 2 — Manifest Validation:      implemented
Layer 3 — Native Execution:         partial
  suppression / unsuppression:     implemented and tested
  schema 2.0 execute integration:  not implemented
  visibility:                      not implemented
  deletion:                        not implemented
```

### Layer 1 — Contract metadata (implemented)

FreeCAD's manifest contract metadata defines manifest schema `2.0`'s optional
`assemblyMutations`/`partMutations` sections and their
`suppression`/`visibility`/`deletion` entry shapes. See
[manifest.md](manifest.md#schema-20-contract-metadata-and-validation-implemented-execute-integration-not-implemented)
for the exact field surface. This metadata is frozen, tuple-based,
import-safe, and free of FreeCAD dependencies or filesystem/runtime side
effects.

### Layer 2 — Manifest validation (implemented)

FreeCAD strictly validates schema `2.0` manifests: exact version dispatch,
closed mutation collections, strict entry shapes and JSON-boolean typing,
duplicate-target rejection, within-scope and cross-scope conflict rejection,
and deterministic diagnostic ordering. This validator can accept and cleanly
reject a schema `2.0` manifest today — but validating a manifest is not the
same as executing its mutations.

### Layer 3 — Native execution (partial)

#### Suppression and unsuppression (implemented and tested)

FreeCAD has a focused native consumer for already validated `{object,
suppressed}` entries. Engine resolves semantic targets, validates captured
capabilities, selects Part or Assembly routing, establishes canonical order,
and projects native object identities into schema 2.0. The consumer starts at
the downstream native boundary; it does not repeat those Engine decisions.

For each supplied entry, in supplied order, the consumer:

1. resolves the exact `object` name through `document.getObject(object)`;
2. requires an existing native `Suppressed` property;
3. requires `getTypeIdOfProperty("Suppressed") == "App::PropertyBool"`; and
4. writes the requested boolean directly to `target.Suppressed`.

Both `suppressed: true` and `suppressed: false` are supported. Processing
stops at the first failure, retaining any earlier successful writes and not
applying later entries. Missing targets, unsupported native capabilities, and
native lookup, inspection, or write failures produce controlled consumer
failures; underlying native exceptions are preserved where applicable.

Lookup is by exact native object name only. The consumer does not perform
semantic resolution or Label, alias, fuzzy, or case-normalized lookup. It does
not synthesize `Suppressed`, and it does not use or change visibility as a
substitute for suppression.

#### Execute integration, visibility, and deletion (not implemented)

FreeCAD's runtime still loads and validates every manifest exclusively
through the schema `1.0` path. A schema `2.0` manifest — mutation-bearing or
not — is rejected before document opening, parameter assignment, recompute,
save, export, or success-result writing. Handled validation failures still
attempt a failed `prm.result.json` when a safe destination is supplied. The
standalone suppression consumer is therefore not invoked by the normal
external `parametron-freecad execute` lifecycle.

Schema 2.0 execute-entrypoint integration, native visibility and deletion
consumers, runtime-stage integration, recompute/save sequencing around target
mutations, structured execute failure mapping, compatibility integration,
post-mutation validity, and target observation are not implemented.

Engine-side planning, capability validation, lowering, routing, and schema
2.0 manifest projection for target actions must not be interpreted as proof
of end-to-end native mutation support: Engine handoff tests exercise Engine's
own planning and projection. The standalone native suppression consumer plus
Engine handoff still does not provide schema 2.0 execution through the normal
external lifecycle.

## Native structure precedent

A permanent real-FreeCAD fixture demonstrates stable starting structure and
preconditions: a dependency chain of features with native suppression state,
an app-level visibility flag, a safe-delete candidate with no dependents, and
a dependency-sensitive unsafe-delete candidate still referenced elsewhere in
the document. Fixture contract tests prove those starting conditions; they do
not perform mutations.

Separate permanent real-FreeCAD tests invoke the production suppression
consumer against temporary copies of that fixture. They demonstrate an
initially suppressed terminal feature becoming unsuppressed and initially
unsuppressed intermediate features becoming suppressed. They also demonstrate
that `Suppressed` is an `App::PropertyBool`, relevant visibility state remains
unchanged, unsupported native targets are rejected without synthetic
properties, and the committed fixture remains unchanged. This is evidence for
the standalone native consumer, not for schema 2.0 execute integration,
recompute/save orchestration, observation, or persistence through normal
execute.

## Ownership

FreeCAD owns exact native object lookup, native CAD mutation, and native
runtime failures at the boundary described above. When target mutations are
integrated into execute, FreeCAD also owns its CAD lifecycle operations around
them. Engine owns semantic intent, semantic target resolution, captured
capability validation, lowering, routing, canonical ordering, schema 2.0
projection, verification decisions, and normalization — see [Target-Action
Contract](../../../reference/target-action-contract.md).
