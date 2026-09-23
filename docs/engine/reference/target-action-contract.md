# Target-action and mutation contract

Engine evaluates target actions, resolves semantic targets, checks captured
capabilities, and projects deterministic runtime mutation intent. FreeCAD owns
native lookup, mutation, recompute, native post-mutation/post-delete validity
inspection, raw native dependency evidence, save/reopen persistence,
observation, and native failures. Engine owns expected-versus-observed
comparison, tolerance evaluation, engineering verification decisions, and
record normalization. Engine handoff tests do not prove native mutation
correctness.

Permanent Engine tests now prove the complete Engine-owned runtime contract
package through the normal CLI path. Supported project and DSL target actions
pass through semantic lowering, planning, scheduling, mutation-preserving
executor handoff, isolated attempt preparation with staged source, canonical
schema `1.0` request materialization, actual external `runtimecap` process
invocation, canonical raw evidence intake, Engine verification, outcome
consumption, and normal normalized/raw record-package emission. A
repository-controlled, contract-compliant external runtime supplies the
responses. The proof covers `suppress`, `unsuppress`, `hide`, `unhide`, `delete`,
and combined mutation families, including Part and Assembly destinations.
It does not establish real FreeCAD-native mutation correctness or cover HTTP API
target-mutation submission. See [Execution Runtime](../runtime/execution-runtime.md)
for the execution and evidence boundaries.

Current FreeCAD runtime limitation: Engine provides planning, capability
validation, lowering, routing, and canonical schema 1.0 manifest projection for
target actions, but the current `parametron-freecad` execution runtime has not yet
been aligned to consume mutation-bearing canonical schema 1.0 manifests. FreeCAD
has a tested native suppression/unsuppression consumer for validated native
object/state entries, and a tested native visibility consumer for validated
`{object, visible}` entries. The visibility consumer supports both projected
mappings (`hide` to `visible: false` and `unhide` to `visible: true`). FreeCAD also
has implemented and tested bounded, read-only native post-mutation PartDesign
Body validity inspection and deterministic native `InList`/`OutList` dependency
evidence. FreeCAD also has a tested standalone conservative native deletion
consumer for validated `{object}` entries: it resolves the exact projected
native object, rejects surviving native dependents, performs native removal,
recomputes, and requires bounded supported post-delete Body validity. These
standalone capabilities are not connected to normal execute. Final lifecycle
ordering and production of FreeCAD-native lifecycle failures remain downstream
FreeCAD work; Engine's generic CAD runtime failure normalization is implemented.
While Engine defines the canonical schema 1.0 target-state observation contract,
FreeCAD-native target-state observation in normal execute is not yet
implemented. Engine independently derives expected target state in memory,
compares valid canonical observed evidence, classifies target-state verification
failures, and maps target-state observation and verification material through the
existing normalized record families. When valid canonical evidence is supplied,
Engine emits those normalized observation and verification records through the
normal package path.
Engine handoff plus these standalone capabilities therefore does not provide
end-to-end target-mutation execution in real FreeCAD. Handoff tests prove
Engine planning and projection; the complete controlled-runtime normal-CLI
proof exercises Engine orchestration beyond handoff; real FreeCAD-native
execution proof remains separate. FreeCAD remains the owner of native lookup,
mutation, recompute, validity inspection, raw dependency evidence, persistence,
observation, and native failure behavior; Engine retains the comparison,
tolerance, verification, and normalization responsibilities stated above.
See [FreeCAD Target Mutations](../adapters/freecad/contracts/target-mutations.md)
for the detailed native deletion and evidence boundary.

## Authoring and action evaluation

```dsl
target Pad: action = suppress
target Pocket: action = removeHole ? suppress : unsuppress
target Chamfer: action = table_cell("variants", variant, "chamferAction")
```

Declarations are product-local: `target <semantic-target>: action = <expression>`.
`TargetActionNode` carries `SemanticTarget` and `Action`; ordered declarations
live in `ProductNode.TargetActions`, separately from `ExecutionDeclarationOrder`.
The parser rejects exact duplicate target spellings within a product. Case variants
are distinct; the same name can occur in different products. Kind qualifiers such
as `target feature Pad` are invalid.

`target` and `action` are reserved grammar keywords. The action vocabulary is
exactly `keep`, `suppress`, `unsuppress`, `hide`, `unhide`, `delete`, in that
diagnostic order. These case-sensitive words are contextual literals, not globally
reserved words; in action context they take precedence over same-named bindings.
`ParamTypeAction` is an internal expected type, not an authored parameter type.

The expression parser accepts structurally valid expressions; validation enforces
the action domain. Literals and nested ternaries with boolean conditions and
action-valued branches are supported. Strings, numbers, booleans, interpolated
strings, ordinary parameter/enum/constant bindings, numeric functions, arithmetic,
and logical operators do not coerce to actions. Action-valued `let` and
`param …: action` are unsupported. Invalid action values report the product,
target, supplied value and ordered allowed domain before semantic target lookup.

`table_cell` accepts an ordinary string column in action context. Numeric and
boolean columns fail type validation; the table schema has no action-specific
column type. Only the selected cell is checked against the exact action vocabulary,
without trimming, case folding or aliases. Unselected non-action strings remain
valid table data. Lookup follows parameter/let evaluation and `--set` overrides;
dynamic keys are not frozen at default values. Invalid selected-cell diagnostics
identify target, table, row, column, value and allowed actions. This contextual
lookup does not allow general string-to-action coercion.

Parameter names such as `suppress_Pad` and `unsuppress_Pad` are ordinary names.
They never trigger target lookup or lifecycle actions. An exact captured metadata
key can map them to ordinary `set_property`; an unmapped name produces no mutation
or special diagnostic. Prefix-shaped names have no compatibility alias or flag.

## Resolution and capability validation

Semantic-model planning searches one equal-priority pool of Features and
Components by exact case-sensitive semantic `Name`. Parameters, parameter groups,
metadata, relationships and outputs are excluded. There is no DisplayName, label,
alias, native-name, whitespace-trimming or case-folding fallback.

Zero candidates fails as not found; one resolves; multiple candidates fail as
ambiguous, including cross-kind matches. Ambiguity descriptors sort by semantic
entity kind then semantic ID, independently of model enumeration. Resolved state
carries semantic entity kind, target entity kind, semantic ID, scope, and captured
`Targetability` without rewriting them. Every action, including `keep`, requires
target existence.

Captured per-target `Targetability` is the sole capability authority:

| Action | Requirement | Lowered `OperationKind` | Runtime family entry |
| --- | --- | --- | --- |
| keep | Existence only | No intent | No entry |
| suppress | `Suppress` | `suppress` | suppression: `{object, suppressed: true}` |
| unsuppress | `Unsuppress` | `unsuppress` | suppression: `{object, suppressed: false}` |
| hide | `Hide` | `hide` | visibility: `{object, visible: false}` |
| unhide | `Unhide` | `unhide` | visibility: `{object, visible: true}` |
| delete | `Delete` | `delete` | deletion: `{object}` |

Capability bits are independent. No inverse action, other axis, entity kind, or
scope grants a missing bit. A zero-valued capability struct permits only `keep`.
One immutable validation gate handles Features and Components; it neither consults
semantic-map `OperationCapabilities` nor infers capabilities from native CAD types
or properties. Denials identify action, required captured field and semantic kind/ID,
with product/target context from the planner.

Evaluation precedes resolution, which precedes capability validation. Invalid
actions therefore take precedence over missing targets, and resolution failures
precede capability failures. Target checks retain authored order and fail fast.
`CreatePlanWithTablesAndSemanticModel` runs the semantic stages when a model is
supplied. Model-free `CreatePlan`/`CreatePlanWithTables` do not fabricate target
resolution, capability checks or mutation intents. No target-action IR v1 form exists.

## Lowering, routing, and family projection

`semantic.LowerTargetActionMutationIntent` returns no intent for `keep` and exactly
one existing `MutationIntent` for each other action. It copies target kind, semantic
ID and scope; `TargetField` is empty. `ValueSource.Kind = "target_action"` is
provenance only: parameter names/IDs and scalar fields are zero and `BooleanState`
is nil. Action meaning resides in the distinct `OperationKind`, never an inverse
action encoded as a boolean. Lowering does not repeat resolution or capability
checks. Existing `set_parameter`/`set_property` paths remain separate.

Lowered intents are private planner state, not direct plan JSON, IR or runtime
payloads, and are not appended to `ProductIntent.Mutations`. Their final projection
participates in normal identity. Private lowering filters `keep` in place and
preserves encounter order without sorting, merging or deduplicating.

`semanticmap.RouteTargetMutations` requires a model, identity linkage, the
`target_action` provenance kind, and one of the five mutation operations. Routing
uses the shared destination policy: explicit target kind `part`/`assembly` takes
precedence; otherwise scope `part`/`assembly` supplies the destination. Missing
destination fails. Features use scope fallback; Components route by target kind.

Features resolve native object names from `Feature.Name`; Components use captured
identity linkage and `Component.Name`. Missing entities/linkage or empty native
names fail. Semantic IDs are lookup keys, never substitutes for native object
names. Routed entries contain only `OperationKind` and `Object`, partitioned into
Part and Assembly, preserving relative order within each bucket.

`ProjectTargetMutationRouting` maps those entries to the three families in the
table above, preserving native objects and destinations without further lookup,
capability checks or rerouting. It rejects `keep` and unrelated scalar operations.
Suppression and visibility are independent axes; same-object entries in different
families coexist. Deletion has only `Object`, without force/cascade/recursive or
dependency policy. Private projection preserves encounter order and duplicates.

The broader internal `ManifestMutationCollection` retains Parameters, Properties,
Suppression, Visibility and Deletion, including through reduction and defensive
handoff cloning. Visibility mappings require `targetField = object`,
`valueField = visible`, and a boolean value, and forbid `propertyField`. Deletion
mappings require `targetField = object` and forbid value/property fields. The
operation registry includes suppress, unsuppress, hide, unhide, delete,
write_parameter and write_metadata. Destination routing does not derive policy
from that registry or semantic-map family capability metadata.

## Aligned runtime manifest

The canonical active filename is `prm.export-manifest.json` across all
Engine-authored FreeCAD manifests. All Engine-authored FreeCAD runtime manifests
select canonical schema `1.0`, whether mutation-free or mutation-bearing.
Mutation-less, keep-only, scalar-assignment-only, and internal
Parameters/Properties-only plans emit schema `1.0`. Any Suppression,
Visibility, or Deletion in either Part or Assembly destination also emits schema
`1.0`.

Core top-level fields under schema `1.0` are `schemaVersion`, `sourceDocument`,
`parameterAssignments`, and `outputs`. When target mutations are present, the
manifest additionally carries optional `assemblyMutations` and/or
`partMutations`. Absent or empty mutation sections and families are omitted.
Runtime mutation entries have closed key sets: `{object, suppressed}`,
`{object, visible}`, and `{object}`. Booleans, including false, are explicit.

Nested Parameters and Properties are filtered from runtime mutation sections.
Top-level `parameterAssignments` is the sole executable scalar-write surface;
internal parameter metadata still supports resolving `Object.Property` targets.
Runtime entries exclude semantic IDs, target kinds, scope, force, cascade,
recursive, and dependency policy. Native object names and Part/Assembly
destinations are preserved.

The projector validates schemas strictly: Engine rejects schema `2.0` as
unsupported, along with any other unsupported schema version or empty schema
string. Attempt-local materialization reports typed `manifest_validation`
errors before writing files.

Product-level and attempt-local emission share
`ProjectFreeCADRuntimeExportManifest` and `MarshalFreeCADRuntimeExportManifestJSON`
in the Engine-owned FreeCAD contract projection. They preserve scalar
assignments, canonical schema `1.0`, mutation families, and outputs; only
`sourceDocument` is adjusted to the prepared working-copy relative path. Root
`--json-plan` exposes the same normal planner `WriteExportManifestPayload`, not
private lowered intents or a separate serializer.

## Canonical ordering and identity

The planner composes mature base mutations first and target-action entries second,
then `canonicalizeRuntimeTargetMutationOrdering` sorts the final composed
collection before mapping validation, draft hashing for `file_pattern`, final
plan hashing, job derivation, handoff and runtime projection.

Part and Assembly sort independently using Go lexical order:

| Family | Sort key |
| --- | --- |
| Suppression | Object, then Suppressed (`false` before `true`) |
| Visibility | Object, then Visible (`false` before `true`) |
| Deletion | Object |

Sorting does not merge, deduplicate, reject conflicts, apply first/last-write-wins,
flatten families or move entries between destinations. Parameters/Properties are
untouched. Upstream private stages preserve encounter order; adapter, job, handoff
and record emission preserve final planner order without independently sorting.

For otherwise equivalent inputs, explicit `keep` and omission produce identical
plan JSON/hash, job and handoff identity, runtime manifest bytes, and record package
key `engine-run:<planHash>`. Each mutation action, opposite state and family changes
normal serialized identity material without a special hash salt. Equivalent target
declaration permutations converge, including draft hashes and resolved file-pattern
filenames. Repeated table-selected actions are deterministic; changed selected
actions change identity through their payloads. Consolidating to canonical schema
`1.0` preserves this normal serialized identity behavior without introducing a new
hashing algorithm or hash salt.

Nil collections remain nil; empty collections need no allocation; nil target-family
slices remain nil. Empty families are semantically empty, but individual non-nil
empty slice allocations need not survive copying alongside populated families.
No artificial mutation section or identity is created for empty or keep-only input.

Native-only `outputs = ["none"]` retains aligned execution with runtime
`outputs: []`, observation, traversal, verification and result handling. The sentinel
is not an export format or native artifact declaration. See
[DSL semantics](../authoring/dsl-semantics.md) for authoring intent and
[execution runtime](../runtime/execution-runtime.md) for zero-artifact acceptance.

Aligned handoffs accept an empty `Manifest.Outputs` when `CADRuntime` is present
and preserve the non-nil empty output array through cloning and JSON round trips.
`ExpectedArtifacts()` lists CSV, manifest and runtime result filenames, with no
derived filenames for native-only execution. The native working document is not
included. Empty-output non-CADRuntime packages retain `ErrManifestOutputsEmpty`.
Native-only plan/hash/cache/job identity is stable and distinct from derived-output
runs; parameter and mutation intent survives unchanged.

## Target-state observation request bridge

Canonical target mutation intent bridges deterministically to Engine-owned
target-state observation requests in `prm.verification.json`:

```text
suppression mutation -> suppression observation request
visibility mutation  -> visibility observation request
deletion mutation    -> existence observation request
```

### Observation identity model

Observation requests reuse the established native target boundary rather than
introducing a competing identity system.

In `prm.export-manifest.json`, mutations are partitioned by destination into
`partMutations` and `assemblyMutations`, with entries containing `{object,
suppressed}`, `{object, visible}`, or `{object}`.

In contrast, target-state observation requests in `prm.verification.json`
represent destination explicitly within each target identity:

```json
{
  "destination": "part",
  "object": "Pad"
}
```

- `destination`: closed vocabulary of `"assembly"` or `"part"`.
- `object`: exact native CAD object name (non-blank, case-sensitive, matching
  the routed native entity).

This structure preserves exact native object names (`Feature.Name` /
`Component.Name`) and destination boundaries (`assembly` vs. `part`) without
relying on semantic IDs as runtime observation keys.

### Raw evidence independence and limitations

The observation request specifies which native facts to inspect post-execution;
it does not copy requested mutation booleans as observed values:

```text
requested mutation != observed native state
```

For example, a requested mutation of `visible: false` leads to a visibility
observation request for that object, but does not dictate whether the raw
evidence will report `status: "observed"` with `value: true` or `value: false`,
`status: "target_missing"`, or `status: "unavailable"`.
The raw result contract faithfully records live native state.

The permanent normal-CLI proof demonstrates this independence across the actual
runtime boundary: the controlled external process supplies its own canonical
`prm.observed.json`, and Engine consumes that returned evidence. Different
observations for the same requested mutation produce different Engine
verification outcomes. This establishes the Engine evidence and comparison
boundary, not FreeCAD-native target-state observation.

For complete schema details, status vocabularies, and canonical ordering
rules, see the canonical
[Observation Contract](../adapters/freecad/contracts/observation.md).

Current implementation boundaries remain clearly separated:

1. **Engine observation contract**: Canonical schema `1.0` target-state
   request and result contracts are defined.
2. **FreeCAD runtime observation**: Current `parametron-freecad` normal
   `execute` does not yet support target-mutation manifest execution or produce
   native target-state evidence.
3. **Engine verification and normalization**: Engine derives suppression and
   visibility expectations from canonical mutation booleans and derives an
   `absent` expectation from deletion. Those values remain in the in-memory
   verification model; the serialized schema `1.0` observation request stays
   identity-only. Engine validates canonical evidence before comparing the exact
   `(destination, object)` target, classifies target-state outcomes, and maps
   accepted observation and verification material through the existing record
   families. Normal execution emits those mappings through the existing
   observation and verification record families when valid canonical evidence
   is supplied.
