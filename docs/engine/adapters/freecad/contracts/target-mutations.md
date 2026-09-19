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
  suppression / unsuppression:     standalone consumer implemented and tested
  visibility hide / unhide:        standalone consumer implemented and tested
  post-mutation validity / native
    dependency evidence:           standalone infrastructure implemented and tested
  conservative native deletion:    standalone consumer implemented and tested
  schema 2.0 execute integration:  not implemented
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

#### Visibility hide and unhide (implemented and tested)

FreeCAD has a focused native consumer for already validated `{object,
visible}` entries. As with suppression, Engine retains ownership of semantic
target resolution, captured capability validation, routing, ordering, and
schema 2.0 projection. The consumer checks native operation support at the
execution boundary; that check does not replace Engine capability validation.

For each supplied entry, in supplied order, the consumer:

1. resolves the exact `object` name through `document.getObject(object)`;
2. requires an existing App-level native `Visibility` property;
3. requires `getTypeIdOfProperty("Visibility") == "App::PropertyBool"`; and
4. writes the requested boolean directly to `target.Visibility`.

Both `visible: false` (hide) and `visible: true` (unhide) are supported.
Processing stops at the first failure, retaining any earlier successful writes
and not applying later entries. Missing targets, unsupported native
capabilities, and native lookup, inspection, or write failures produce
controlled consumer failures; underlying native exceptions are preserved
where applicable. Empty visibility collections are accepted, and caller input
is not mutated.

Lookup is by exact native object name only. The consumer does not perform
Label, alias, fuzzy, or case-normalized lookup, and it does not synthesize
`Visibility`. It uses neither `FreeCADGui` nor `ViewObject`. Visibility remains
independent of suppression; the consumer does not use suppression as a
visibility substitute.

#### Conservative native deletion (implemented and tested)

FreeCAD has a focused native consumer for already validated `{object}` entries.
Engine owns semantic policy, target resolution, captured capability validation,
routing, canonical ordering before handoff, and schema 2.0 projection. The
consumer starts with the exact native object identity supplied by its caller and
applies conservative FreeCAD-native deletion safety.

For each supplied entry, in supplied order, the consumer:

1. resolves the exact `object` name through `document.getObject(object)`;
2. inspects native dependency evidence and rejects deletion when `InList`
   reports surviving dependents;
3. removes the target through `document.removeObject(object)`;
4. confirms through exact lookup that the requested object no longer resolves;
5. recomputes the document; and
6. inspects the supported surviving native `PartDesign::Body` objects for
   post-delete validity.

Required dependency and validity evidence is fail-closed. Missing exact
targets, failed native lookup/inspection/removal, surviving-dependent conflicts,
failed absence verification, failed recompute, unavailable supported Body
evidence, and proven invalid supported Body state produce controlled deletion
failures. Underlying native causes are preserved where applicable. Successful
native removal alone is therefore insufficient for standalone deletion success.

The consumer preserves caller-provided order and does not rewrite the supplied
entries. It stops at the first failure, does not execute later entries, and does
not automatically roll back an earlier completed deletion. Lookup has no Label,
alias, fuzzy, or case-normalized fallback. `OutList` remains raw native
dependency evidence; it does not define cascade or repair behavior. The consumer
provides no force deletion, cascade, recursive deletion, dependency repair, or
general deletion-policy input.

#### Deterministic post-mutation validity and dependency evidence (implemented and tested)

FreeCAD has focused, read-only native inspection infrastructure for a caller
that has already performed any mutation and recompute. It resolves the exact
requested native object through `document.getObject(object)` with no Label,
alias, fuzzy, or case-normalized fallback. Shape-health inspection is bounded
to objects with positive native `PartDesign::Body` type evidence.

For a supported Body, validity inspection observes native `Shape.isNull()`,
then `Shape.isValid()`, then the number of native `Shape.Solids`. A positively
proven null shape is classified as invalid before later validity or solid-count
evidence is required. A non-null shape for which `isValid()` is false is also
classified as invalid before solid-count evidence is required. Solid count is
reported as native evidence; no particular count is a universal acceptance
rule.

The controlled failure boundary distinguishes unavailable required native
evidence, failure while invoking native inspection, and native CAD state
positively proven invalid. Diagnostics are deterministic, and underlying
native causes are preserved where applicable.

Dependency inspection reports raw native relationships: `InList` objects are
dependents and `OutList` objects are dependencies. Related objects must expose
stable native `Name` values; duplicate names are removed and the results are
ordered lexically. This evidence makes no deletion-safety or general dependency
policy decision by itself. The deletion consumer composes that raw evidence into
its conservative surviving-dependent rejection.

Document-level composition through
`inspect_document_post_mutation_validity(document)` deterministically discovers
the surviving supported native `PartDesign::Body` objects, orders them lexically
by stable native `Name`, and applies the same bounded shape-health inspection to
each Body. Discovery does not depend on incidental `document.Objects`
enumeration order. The absence of any supported Body evidence fails closed at
this standalone boundary.

All inspection APIs are import-safe and read-only. They do not mutate objects,
recompute, save, export, close, or delete. Mutation and recompute are owned by
the caller. The standalone deletion consumer owns removal and recompute around
these helpers; normal execute lifecycle placement remains issue #6 work.

#### Execute integration (not implemented)

FreeCAD's runtime still loads and validates every manifest exclusively
through the schema `1.0` path. A schema `2.0` manifest — mutation-bearing or
not — is rejected before document opening, parameter assignment, recompute,
save, export, or success-result writing. Handled validation failures still
attempt a failed `prm.result.json` when a safe destination is supplied. The
standalone suppression, visibility, and deletion consumers and the standalone
validity/dependency inspection infrastructure are therefore not invoked by the
normal external `parametron-freecad execute` lifecycle.

Schema 2.0 execute-entrypoint integration, runtime-stage integration, final
recompute/save sequencing around target mutations, structured execute failure
mapping, and compatibility integration are not implemented. Issue #6 owns final
mutation → recompute → validity →
save/etc. ordering and translation of validity failures through the structured
runtime failure boundary. Target-state observation remains separate issue #5
scope.

Engine-side planning, capability validation, lowering, routing, and schema
2.0 manifest projection for target actions must not be interpreted as proof
of end-to-end native mutation support: Engine handoff tests exercise Engine's
own planning and projection. The standalone native suppression, visibility, and
deletion consumers, standalone native inspection infrastructure, and Engine
handoff still do not provide schema 2.0 execution through the normal external
lifecycle.

## Native structure precedent

A permanent real-FreeCAD fixture demonstrates stable starting structure and
preconditions: a dependency chain of features with native suppression state,
an app-level visibility flag, a safe-delete candidate with no dependents, and
a dependency-sensitive unsafe-delete candidate still referenced elsewhere in
the document. Fixture prerequisite tests also establish a healthy
`MutationBody` and its shape-health preconditions. They prove starting
structure only; they do not perform mutations or exercise the standalone
consumers or inspections.

Separate permanent real-FreeCAD tests invoke the production suppression
consumer against temporary copies of that fixture. They demonstrate an
initially suppressed terminal feature becoming unsuppressed and initially
unsuppressed intermediate features becoming suppressed. They also demonstrate
that `Suppressed` is an `App::PropertyBool`, relevant visibility state remains
unchanged, unsupported native targets are rejected without synthetic
properties, and the committed fixture remains unchanged. This is evidence for
the standalone suppression consumer.

Permanent real-FreeCAD tests separately invoke the production visibility
consumer against temporary fixture copies. They demonstrate hiding the
initially visible `MutationBody` and unhiding the initially hidden
`BaseSketch`, with App-level `Visibility` exposed as `App::PropertyBool`. They
also demonstrate that suppression remains unchanged, missing exact native
targets fail through the controlled consumer, and the committed fixture
remains byte-identical.

Permanent standalone validity/dependency tests separately invoke the production
read-only inspections. On temporary copies of the committed fixture they prove
healthy, non-null, valid `MutationBody` evidence and the fixture-specific fact
that its shape contains one solid. They prove `SafeDeleteMarker` has no native
relationships and cover a representative chain in which `BaseSketch` reports
`IntermediatePad` as a dependent, while `IntermediatePad` reports
`IntermediatePocket` as a dependent and `BaseSketch` as a dependency.

The invalid case uses a fresh, test-owned empty `PartDesign::Body` with a
genuinely null native shape. Real FreeCAD raises if `isValid()` is queried on
that null shape; the production inspection proves nullity first and therefore
deterministically raises `InvalidNativeCadStateError`. Separate FreeCAD
processes produce equal normalized healthy evidence, relationship ordering,
and invalid diagnostics. Before/after snapshots prove read-only behavior, and
the committed fixture remains byte-identical.

Permanent real-FreeCAD deletion tests separately invoke the production deletion
consumer against temporary fixture copies. They prove supported removal of
`SafeDeleteMarker`; conservative rejection of `BaseSketch` because native
`InList` evidence includes the surviving dependent `IntermediatePad`; controlled
failure for a missing exact target; and failure when a genuine invalid
post-delete Body state is produced. Independent FreeCAD processes produce
identical safe, unsafe, and invalid-state outcomes and diagnostics, and the
committed fixture remains byte-identical. These object names are representative
fixture evidence, not general semantic rules.

These consumer tests are native execution proof for the standalone suppression
visibility, and deletion consumers. The validity/dependency tests are standalone
native inspection proof. The fixture prerequisite tests establish only starting
structure and preconditions. None of these forms of proof establishes schema
2.0 execute integration, final lifecycle orchestration, target-state
observation, or persistence through normal execute.

## Ownership

FreeCAD owns exact native object lookup, native CAD mutation including
conservative deletion, native dependent safety at the deletion boundary,
post-delete supported Body validity inspection, raw native dependency evidence,
and native runtime failures at the boundary described above. When target
mutations are integrated into execute, FreeCAD also owns its CAD lifecycle
operations around them.
Engine owns semantic intent, semantic target resolution, captured capability
validation, lowering, routing, canonical ordering, schema 2.0 projection,
expected-versus-observed comparison, tolerance evaluation, engineering
verification decisions, and normalized durable records — see [Target-Action
Contract](../../../reference/target-action-contract.md).
