# Verification Standard

This document defines the canonical definition of done for all phases and issues in the Parametron ecosystem.

Deterministic behavior must be demonstrated, not assumed.

---

## Core Rule

Implementation alone is not sufficient for closure.

A phase or issue may be closed only when:

- the behavior is implemented

- relevant tests pass

- deterministic expectations are verified

- documentation is synchronized


---

## Verification Layers

### 1. Standard Tests

Each implementation unit must be covered by the owning repository's normal automated test suite.

These tests validate correctness of individual components in isolation.

---

### 2. Smoke Tests

Smoke tests validate that expected valid workflows execute successfully.

They ensure that normal usage paths remain stable.

---

### 3. Break Tests

Break tests intentionally introduce invalid or adversarial inputs.

They ensure that:

- failures occur deterministically

- failures occur at the correct stage

- failures are classified correctly


---

### 4. Expected Failure Matrix

Expected outcomes for break tests must be explicitly defined.

These expectations are recorded in structured files (e.g. `_expectations.json`) and validated by a matrix test harness.

Each entry must define:

- expected result (`pass`, `fail`, or `timeout`)

- failure stage (e.g. `parse`, `validate`, `plan`)

- failure class (machine-readable classification)


This ensures that failure behavior is:

- stable

- reproducible

- regression-safe


---

### 5. Documentation Synchronization

Verification is incomplete without documentation alignment.

When behavior, interfaces, contracts, commands, examples, or guarantees change,
identify the documentation they affect and review and update the relevant
repository-local documentation.

Repository-local verification documentation or structured expectation data
must also be updated when those artifacts exist and the change affects them.

Before closure, the implementation, tests, verification expectations, and
documentation must describe the same behavior.


---

## Closure Rule

A phase or issue is not complete until:

- implementation is finished

- required verification layers have passed

- expected failures are pinned where applicable

- documentation is aligned

- applicable exit criteria are satisfied


An appropriate GitHub closing reference may be used on the final change when
automatic issue closure is intended and all applicable completion conditions
are satisfied.

---

## Determinism Requirement

Any claim of deterministic behavior must be backed by repeatable tests and explicit expected outcomes.

Determinism is a verified property, not a descriptive label.

---

## Goal

This standard ensures:

- reproducible system behavior

- explicit and test-backed guarantees

- prevention of silent regressions

- consistent definition of "done" across the ecosystem
