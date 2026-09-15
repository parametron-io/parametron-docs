# Engine Adapter Domain

The Engine adapter boundary translates planned engineering intent into calls to
a CAD runtime. Engine remains responsible for validating inputs, building the
plan and handoff, selecting and invoking the runtime capability, validating
returned evidence, making expected-versus-observed verification decisions, and
normalizing accepted results into Engine-owned records.

An adapter performs CAD-native work behind those Engine contracts. It opens and
observes documents, traverses references, applies requested mutations,
recomputes models, saves native documents, exports requested artifacts, and
returns raw runtime results and errors. It does not redefine Engine planning,
verification, failure interpretation, or normalized record semantics. FreeCAD
is the only currently implemented CAD adapter.

## Current Engine Interfaces

The generic adapter package in `parametron-engine` defines two current
interfaces:

- `Adapter.Run(context.Context, planner.Step)` provides ordinary planned-step
  dispatch.
- `CADRuntimeOrchestrator.OrchestrateCADRuntime(context.Context,
  CADRuntimeOrchestrationRequest)` executes one validated CAD-runtime attempt.

The orchestration request keeps logical planning data separate from operational
runtime selection. It carries Engine-owned job, product, step, and attempt
context; the planned CSV, manifest, and CAD-runtime payloads; the selected
runtime executable; and any Engine-authoritative external target identities.
Engine validates the request and the consistency of its adapter, product, and
manifest identities before runtime work.

The `runtimecap.Capability` interface is the Engine-owned external invocation
boundary. It passes explicit working-copy, manifest, result, output, observation,
and reference-traversal paths to a file-based runtime command and captures the
resolved command, standard output, and standard error. Runtime-specific code
performs the CAD API operations and returns raw evidence through that contract;
Engine owns its acceptance, verification, and normalized interpretation.

## Adding an Adapter Implementation

Within the current Engine architecture, an adapter implementation plugs in by:

1. implementing the generic `Adapter` interface in
   `internal/engine/adapter/`;
2. placing runtime-specific manifest projection and path adaptation in an
   adapter-specific package below that boundary;
3. registering its profile target with Engine authoring validation; and
4. when it uses an external process, implementing the Engine-side
   `CADRuntimeOrchestrator` handoff and `runtimecap` invocation contracts.

These extension points satisfy existing Engine contracts. Adding an
implementation does not give it ownership of Engine planning, orchestration
policy, evidence verification, or normalized records. Source code, schemas,
tests, package layout, and repository-local build instructions remain
authoritative in the repository that owns each implementation.

## FreeCAD Adapter

FreeCAD is the currently implemented Engine CAD adapter. See [FreeCAD Adapter
Architecture](freecad/architecture.md) for its role, ownership boundary, and
capability areas, and [FreeCAD Runtime](freecad/runtime.md) for its headless
invocation and execution lifecycle. Manifest, observation, reference-
traversal, artifact, result/failure, target-mutation, and in-process
invocation contracts are documented under
[freecad/contracts/](freecad/contracts/).
