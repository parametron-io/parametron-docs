# DSL Overview

The Parametron DSL is a declarative language for defining parametric models. A DSL file describes products, their parameters, constants, and output profiles. The engine parses this file, validates it, converts it to an IR, and generates an execution plan.

## File Structure

A DSL file contains the following top-level constructs, in this order:

1. **DSL version directive** (required, must be first meaningful token)
2. **Constants** (optional, global scope)
3. **Profiles** (optional)
4. **`use profile`** declaration (required when multiple profiles are defined)
5. **Products** with parameters

## DSL Version Directive

Every DSL file must declare a version. Supported version: `v1.0`.

Accepted forms:
```dsl
dsl v1.0
dsl 1.0
```

The directive must appear as the first meaningful token. BOM, whitespace, and comments may appear before it.

Errors: missing directive, unsupported version, directive not at top of file.

## Products and Parameters

```dsl
product <name> {
    let <name> = <expression>
    param <name>: <type> = <expression>
}
```

Products are the top-level model entities. Inside a product, `let` introduces
a product-local binding and `param` declares a typed parameter. See
[DSL grammar](dsl-grammar.md) for the full syntax reference.

`let` bindings are internal to product evaluation. `param` bindings are the exported/public product bindings that appear on release-facing plan and manifest surfaces.

## Global Constants

```dsl
const NAME = expression
```

Constants are declared at global scope. They are evaluated at validation time (compile-time) and must be fully evaluable without runtime data. They may reference previously defined constants but cannot reference parameters.

Supported constant types: `number`, `string`, `boolean`, enum value.

Constants and parameters share a single namespace. A constant name cannot be redefined. Resolved constants are available to the planner as literal values; changing a constant value changes the plan hash.

## Profiles

```dsl
profile <Name> {
    key = <literal-or-const>
}
use profile <Name>
```

Profiles are configuration containers that control output behavior. A DSL file can define multiple profiles. When multiple profiles are present, a `use profile` declaration is required. When exactly one profile is defined, it is auto-selected.

Profile settings:
- `output_dir` (string): output directory, must be a safe relative path
- `metadata_enabled` (boolean)
- `file_pattern` (string): supports placeholders `{product}`, `{profile}`, `{plan_hash}`, `{param:<name>}`, `{const:<name>}`

## Product-Level Execution Declaration

Products may declare execution intent and target actions directly inside the `product {}` block:

```dsl
product <name> {
    adapter = "<adapter-id>"
    source_model = "<model-id-or-path>"
    outputs = ["<format>", ...]

    let <name> = <expression>
    param <name>: <type> = <expression>

    target <semantic-target>: action = <expression>
}
```

Execution declaration fields:
- `adapter` (string): the adapter to use for this product. Use `"freecad"` to activate the FreeCAD adapter pipeline. Use `"none"` to declare a product with no execution adapter.
- `source_model` (string): required when `adapter != "none"`. In project-mode flows, interpreted as a logical model ID and resolved to a physical path by the project mapping layer before planning; an unresolvable ID fails deterministically after DSL parse/validation and before plan generation. In standalone DSL flows, used as a physical path directly. Not validated for file existence at DSL parse time.
- `outputs` (list of strings): the output formats to produce. Must be a subset of the selected adapter's supported outputs. For FreeCAD: derived formats `"step"`, `"csv"`, `"pdf"`, or the mutually-exclusive native-only sentinel `["none"]`. For `adapter = "none"`: must be empty or absent. Unknown values produce a validation error.

These fields are defined per product. Profile settings (`output_dir`, `metadata_enabled`, `file_pattern`) remain at the profile level and are not affected.

`let` declarations are also allowed inside product blocks using `let <name> = <expression>`. They participate in evaluation like `param` declarations, but remain internal-only and are not exported on public plan or manifest surfaces.

## Target Action Declarations

```dsl
target <semantic-target>: action = <expression>
```

Examples:

```dsl
target Pad:
    action = suppress

target Pocket:
    action = removeHole ? suppress : unsuppress

target Chamfer: action = table_cell("variants", variant, "chamferAction")
```

Target-action declarations express lifecycle, presentation, or structural intent for named CAD targets:

- `suppress` / `unsuppress`: lifecycle intent
- `hide` / `unhide`: presentation intent
- `delete`: structural intent
- `keep`: no mutation intent (retains target existence)

Syntax and authoring rules:

- **Product-local scope**: Target actions are declared inside `product` blocks. Duplicate exact target declarations within the same product are rejected deterministically by the parser.
- **Allowed action expressions**: The `action` expression accepts canonical action literals (`keep`, `suppress`, `unsuppress`, `hide`, `unhide`, `delete`), typed ternary expressions (`<bool-cond> ? <action> : <action>`), or ordinary string-backed table lookups via `table_cell(...)`.
- **Strict typing**: Action validation enforces the exact canonical action domain. No implicit coercion is allowed from numbers, booleans, direct strings, constants, or parameter bindings.
- **Ordinary parameters**: Prefix-shaped parameter names (such as `suppress_Pad`) remain ordinary parameters; only `target <name>: action = <expr>` authors target actions. Action-valued `let` bindings and `param ...: action` parameter types are unsupported.

For the authoritative specification of semantic target resolution, targetability capability gating, mutation lowering, destination routing, canonical ordering, and manifest projection, see the [Target-Action Contract](../reference/target-action-contract.md).

## Supported Parameter Types

| Type | Description |
|------|-------------|
| `number` | Floating-point numeric value |
| `string` | Text value, supports interpolation and concatenation |
| `boolean` | `true` or `false` |
| `enum { A, B, C }` | Closed set of named values |

## Comments

- Line comment: `//` until end of line
- Block comment: `/* ... */` (non-nestable)
- Comment markers inside string literals are not treated as comments.
- Unclosed block comment is a lex error.

## Related Documents

- [DSL grammar](dsl-grammar.md) — full syntax: operators, types, expressions, built-in functions
- [DSL semantics](dsl-semantics.md) — evaluation model, type rules, identifier resolution
- [IR and planning](ir-and-planning.md) — what happens after parsing: IR and execution plan
- [Target-action contract](../reference/target-action-contract.md) — semantic target resolution, capability gating, mutation lowering, and execution contract
- [Validate](../cli/validate.md) — how to validate a DSL file without executing it
