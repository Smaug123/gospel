---
name: designing-for-incremental-implementation
description: How to decompose a design into reviewable implementation stages. Use when planning how to implement multi-stage features, breaking down design documents into PRs, or reviewing implementation plans before starting work.
---

# Incremental Implementation Strategy

## Purpose

This skill converts a DESIGN.md (which specifies *what* to build) into an IMPLEMENTATION_PLAN.md (which specifies *how to build it incrementally*).
If you haven't written a design document for the feature, do that first.

The output IMPLEMENTATION_PLAN.md file should be a sequence of stages, each implementable as a PR or commit, suitable for sequential implementation by other Claude instances.

## The Core Criterion

**Each stage must have a correctness oracle**: a way to verify that the stage is complete and correct.

Property-based tests are a powerful and general way to produce correctness oracles (see the `property-based-testing` skill), but they're not the only option.
Integration tests, smoke tests against a running system, or even "this compiles and the existing tests pass" can serve as oracles for appropriate stages.

The key question for each stage: *How will we know when it's done and correct?*

## Stage Boundaries

**Ideal**: A small user-visible feature is implemented.
- "We now log an info line whenever we receive a webhook"
- "The settings page now displays the user's email"

**Acceptable**: Testable infrastructure, even if not yet user-visible.
- "The GitHub client passes an integration test asserting it can authenticate"
- "The rate limiter passes property tests against a naive reference implementation"

This infrastructure may be dead code temporarily, but each stage is still marked by successively more complete testing.

**Avoid**: Multiple parallel components that don't connect to anything.
Prefer a product that does *something* at each stage, even if it's as limited as "the component takes part in service readiness checks", over implementing twenty components that aren't hooked up to each other.

## Ordering Preferences

**Implement simpler correctness oracles first.** Error handling and sad paths are often simpler than the happy path—they make you think about control flow without requiring complex logic. An unimplemented happy path is just another sad path; there's nothing special about it.

**Separate orthogonal concerns into separate stages.** If you need "and" to describe what a stage does, consider splitting it. Each stage should address a single concern.

**Infrastructure before consumption.** When components don't naturally stack into user-visible features, prefer implementing and testing infrastructure first, then consuming it in later stages. Hide staged implementations behind feature flags or unexposed endpoints until complete; the comprehensive testing of stages is still consumption, of a sort.

**Testing infrastructure is infrastructure too.** If verification requires substantial setup—stub services, test harnesses, pipeline changes—treat that setup as its own stage. The oracle might be "the stub service passes its own health check" or "the test harness can run a trivial end-to-end case."

## What Each Stage Should Contain

Each stage in the IMPLEMENTATION_PLAN.md should specify:

1. **Dependencies**: Which prior stages must be complete. (This enables parallel work on orthogonal features.)

2. **DESIGN.md reference**: Which sections of the design document this stage implements.

3. **Correctness oracle**: How we verify completion, with concrete properties where applicable.

## Example

Given a DESIGN.md describing a caching layer, here is a viable IMPLEMENTATION_PLAN.md:

```
## Stage 1: Cache interface and in-memory stub

**Dependencies**: None

**Implements**: DESIGN.md §2.1 (Cache interface)

**Correctness oracle**:
- Property: `get` after `set` returns the set value
- Property: `get` without `set` returns None
- Property: `delete` followed by `get` returns None

---

## Stage 2: TTL expiration

**Dependencies**: Stage 1

**Implements**: DESIGN.md §2.2 (Expiration)

**Correctness oracle**:
- Property: `get` after TTL elapsed returns None
- Property: `get` before TTL elapsed returns value
- Integration test: entries expire under wall-clock time

---

## Stage 3: Redis backend

**Dependencies**: Stage 1

**Implements**: DESIGN.md §2.3 (Redis integration)

**Correctness oracle**:
- All Stage 1 properties pass with Redis backend
- Integration test: cache survives process restart

---

## Stage 4: Cache warming on startup

**Dependencies**: Stage 3

**Implements**: DESIGN.md §2.4 (Startup behaviour)

**Correctness oracle**:
- Integration test: after restart, frequently-accessed keys are pre-populated
- Smoke test: application startup time acceptable with warm cache
```

Note that Stages 2 and 3 can be implemented in parallel since they depend only on Stage 1.
