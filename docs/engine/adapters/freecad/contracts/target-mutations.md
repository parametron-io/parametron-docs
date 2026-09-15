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
Layer 1 — Contract/Type Metadata:  implemented
Layer 2 — Manifest Validation:     implemented
Layer 3 — Runtime Execution:       not implemented
```

### Layer 1 — Contract metadata (implemented)

FreeCAD's manifest contract metadata defines manifest schema `2.0`'s optional
`assemblyMutations`/`partMutations` sections and their
`suppression`/`visibility`/`deletion` entry shapes. See
[manifest.md](manifest.md#schema-20-contract-metadata-and-validation-implemented-runtime-execution-not-implemented)
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

### Layer 3 — Runtime execution (not implemented)

FreeCAD's runtime still loads and validates every manifest exclusively
through the schema `1.0` path. A schema `2.0` manifest — mutation-bearing or
not — is rejected before document opening, parameter assignment, recompute,
save, export, or success-result writing. Handled validation failures still
attempt a failed `prm.result.json` when a safe destination is supplied. There is
no native suppression, unsuppression, visibility, or deletion execution, and
no post-mutation validity or target-observation behavior.

Engine-side planning, capability validation, lowering, routing, and schema
2.0 manifest projection for target actions must not be interpreted as proof
of end-to-end native mutation support: Engine handoff tests exercise Engine's
own planning and projection, not FreeCAD's native execution, which does not
yet exist for this capability.

## Native structure precedent

A permanent real-FreeCAD fixture demonstrates that the native object
structure a future runtime-execution layer would need already exists and is
tested against real FreeCAD: a dependency chain of features with native
suppression state, an app-level visibility flag, a safe-delete candidate with
no dependents, and a dependency-sensitive unsafe-delete candidate still
referenced elsewhere in the document. This fixture proves the fixture's own
structure and preconditions; it does not itself implement or exercise any
mutation execution.

## Ownership

FreeCAD owns the CAD-native runtime boundary described above. Engine owns
semantic intent, planning, verification decisions, and normalization — see
[Target-Action Contract](../../../reference/target-action-contract.md).
