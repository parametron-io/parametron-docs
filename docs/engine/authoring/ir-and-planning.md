# IR and Planning

Engine owns both its intermediate representation (IR) and deterministic
execution plans. IR separates parsed syntax from a serializable authoring model;
planning turns validated expressions and inputs into resolved execution intent.

```text
DSL -> parsing -> semantic validation -> Engine-owned IR -> planning
                                                              |
                                                              v
                                              ExecutionPlan / runtime intent
```

The normal CLI also plans directly from the validated AST. IR is an intentional
Engine authoring boundary with its own conversion, serialization, DSL generation,
and planning APIs; CLI use of the AST planner does not make IR obsolete.

## IR representation

Engine IR defines these structures:

| Type | Content |
| --- | --- |
| `IRProgram` | `version`, resolved `constants`, `profiles`, `activeProfileName`, and `products`. |
| `IRConstant` | Name, type, and concrete value. |
| `IRProfile` | Name and a map of setting expression trees. |
| `IRProduct` | Name, internal `lets`, and exported `parameters`. |
| `IRLet` | Name and expression. |
| `IRParameter` | Name, declared type, and default expression tree. |
| `IRType` | Kind and optional enum values. |

Type kinds are `number`, `string`, `boolean`, and `enum`. Expression kinds
are `literal`, `reference`, `interpolated_string`, `binary`, `unary`,
`ternary`, and `call`. References distinguish `param`, `let`,
`constant`, and `enum`. Conversion emits IR version `"1.0"`.

IR conversion expects a validated AST with resolved constants. It preserves expression trees for product bindings and profile settings. The IR can be serialized to JSON, loaded with structural validation, and rendered as DSL for represented constructs.

IR coverage is explicit: `IRProduct` has no product adapter, source-model,
output-declaration, or target-action fields. Conversion is therefore not a
lossless round trip for every AST construct. Capture-backed target-action
planning uses the AST/semantic-model entry point. The complete action contract
is defined in [target-action contract](../reference/target-action-contract.md).

## Planning inputs

Planning accepts a validated AST or an IR program, plus binding overrides. Table-aware planning also accepts tables keyed by logical ID, and semantic planning accepts a captured semantic model and its mapping.

Both AST and IR planning accept overrides for `param` and `let` and resolve
those values through the same logical binding graph. Lets remain internal;
exported values remain parameter-only. For let overrides, planning derives the
required inferred binding type, including enum domains, without changing the
public/versioned IR representation: `IRLet` remains name plus expression.
Table cells are resolved during evaluation. [DSL semantics](dsl-semantics.md)
owns the evaluation and type rules.

The IR planner builds CSV and manifest steps and can emit `RunCADRuntime` when
its selected IR profile requests FreeCAD. Its manifest construction does not
supply the full source/mapping intent of the normal AST FreeCAD path. A runtime
step in an IR-generated plan alone is not proof of an executable CAD package;
handoff and runtime validation still apply.

## Execution plans and products

`planner.ExecutionPlan` contains an ordered `Steps` slice. Each step has a
type and a typed payload. A product contributes multiple steps, not one step.

| Step type | Planned intent |
| --- | --- |
| `WriteCSV` | Product key, filename, headers, and resolved exported values. |
| `WriteExportManifest` | Product identity, values, assignments, source/output intent, and applicable mutation payloads. |
| `RunCADRuntime` | Product key, logical adapter ID, manifest filename, and required result filename. |

These are the current `StepType` values. Normal AST FreeCAD planning emits
`WriteCSV -> WriteExportManifest -> RunCADRuntime`. Runtime intent remains
separate from executable selection, retry attempts, and workspace preparation.

The AST planner preserves product and parameter declaration order in the
corresponding plan/CSV surfaces. It validates product filenames and rejects
collisions. FreeCAD manifests use `prm.export-manifest.json`, with
`prm.result.json` as the required runtime result. Derived output paths are logical
paths under `outputs/`. Explicit `outputs = ["none"]` retains the runtime
step while declaring an empty export list.

Capture-backed target planning resolves the DSL target by exact, case-sensitive
semantic Name, retains stable semantic identity, validates captured capabilities,
and lowers semantic mutation intent. Engine then maps the selected Feature or
Component through captured `identitySource.nativeRef` to resolved runtime
`Object`. Semantic Name, semantic ID, DisplayName, and native selector remain
distinct. Executable actions require a usable native mapping before dispatch;
`keep` contributes no native mutation. Part/Assembly destination routing remains
independent of Object. Runtime observation and expected-state verification reuse
the projected `(destination, object)` identity. The
[target-action contract](../reference/target-action-contract.md) owns mapping
validation and runtime ownership rules.

## Deterministic ordering and identity

The planner resolves filenames before execution. When `file_pattern` uses
plan identity, a draft plan supplies the naming hash; the final plan includes
the resulting filenames. Canonical target mutation ordering occurs before
naming and hashing and follows final projected runtime Objects, as specified
by the target-action reference.

`ComputePlanHash` hashes serialized execution-plan JSON with SHA-256. With a
selected AST profile, it also includes the profile's resolved signature, whose
setting keys are sorted. Without a selected profile, it hashes plan JSON alone.
The manifest's planning hash is populated during construction; the CLI computes
its final run hash from the returned plan and selected profile.

A let override changes normal plan identity when it changes serialized resolved
plan intent. An internal override that leaves that intent unchanged does not
change identity merely because it was supplied. Raw override history is not a
separate identity input; equivalent override inputs remain deterministic.

Equivalent semantic/native mappings yield equivalent resolved plan JSON and
hashes. Changing a used nativeRef changes the projected runtime Object and
serialized resolved intent, and thus plan identity/hash. DisplayName-only changes
do not retarget or change applicable plan identity. Unused native mappings do not
create artificial identity changes: raw capture mappings are not a separate
unconditional hash input. Native routing uses the existing serialization and
hashing mechanism.

Identity follows serialized values, step order, and included profile settings.
It is not a hash of DSL formatting or of every unused declaration. Host paths
already present in planner payloads can participate; downstream executable
selection, attempt numbers, process logs, and timestamps are operational state
and are not added to plan identity. Existing repeated-run and permutation tests
cover the defined ordering and identity surfaces.

## Execution boundary

Authoring decides what accepted work should happen. The
[execution model](../architecture/execution-model.md) owns product jobs, handoff,
scheduling, and attempts. [Adapter architecture](../adapters/README.md)
owns logical-to-operational contract adaptation and working copies. The
[execution runtime](../runtime/execution-runtime.md) owns the detailed external
invocation and evidence-processing lifecycle. CAD-native execution belongs to
the external runtime.
