# Parametron Product Family

## Purpose

This document establishes the minimum naming and product-boundary context
needed to read the rest of this repository without ambiguity. It does not
document product architecture.

## Naming

**Parametron** is the family name. Unqualified "Parametron" refers to the
family, not to any single component.

Individual engineering components are named `parametron-<component>`, for
example `parametron-engine` and `parametron-freecad`. See
[Naming Conventions](../conventions/naming.md).

## Current Public Engineering Focus

The current public engineering focus is the **Engine + FreeCAD** deterministic
local/headless runtime pipeline:

- **Engine** (`parametron-engine`) owns deterministic engineering validation,
  planning, execution, adapter contracts, and normalized engineering record
  production.
- **FreeCAD** (`parametron-freecad`) is one CAD adapter/runtime implementation
  operating behind Engine-owned contracts.

## Engine Independence

Engine is deterministic and PDM-independent. It supports standalone
local/headless execution and does not require any other Parametron component to
validate, plan, or execute local projects.

Engine-produced records and runtime results are described as *PDM-ready* where
that reflects existing Engine contracts. Engine-produced output may later be
consumed by other systems; those systems are outside the scope of this
repository.

## Other Product Lines

Other Parametron product lines and product names exist. Their architecture,
roadmaps, and plans are maintained outside this repository and are outside its
scope. This repository documents only the current public engineering
components.
