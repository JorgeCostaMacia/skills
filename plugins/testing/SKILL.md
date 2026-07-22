---
name: testing
description: >-
  Testing principles and the pragmatic TDD loop — stack-agnostic and
  canon-anchored. Use when writing or reviewing tests, deciding what to test
  and how to name it, choosing between unit/integration/e2e, deciding whether
  to mock or use the real thing, judging whether coverage is enough, starting
  a feature test-first, or when tests are flaky, coupled or unreadable — in
  any language, front or back. It encodes: done means tested, the
  red-green-refactor loop, the test pyramid, one test file per unit, names as
  specification, AAA and test independence, classicist test doubles (real
  collaborators and fakes over mocks), behavior over implementation, and rule
  coverage over line coverage. Test file placement and tooling are
  stack-specific and live in each stack's skill.
---

# Testing & TDD (principles)

Canon-anchored: sections marked **[canon]** state the standard; **[our convention]** marks deliberate positions, each with its why. Where test files live and which framework/tooling to use is each stack skill's decision. **Rules get revised, not forced: if one fights the reality of a project, question the rule.**

## Done means tested — [our convention]

A use case, rule, or fix without tests **doesn't exist**. Tests are part of the change — same commit, same PR, same definition of done — never a follow-up task.

## The loop — [canon: red-green-refactor]

1. **Red** — write the failing test that states the expected behavior. Name it first: if you can't name it, you don't know what you're building yet.
2. **Green** — the simplest code that passes. No gold-plating.
3. **Refactor** — with green tests: names, duplication, structure. Tests are code — they refactor too.

**[our convention]** Pragmatics over ritual: an exploratory spike may come first — but the tests are written **before the work is considered done**, and the design is re-checked against them. The non-negotiable is the end state (tested), not the ceremony.

## The test pyramid — [canon]

```
      /  e2e   \       fewest — slowest, most brittle, vaguest diagnosis
    / integration \     some — real infrastructure, slower
  /      unit        \   most — milliseconds, failure points at the exact rule
```

- **Unit**: one unit in isolation — fast, no infrastructure, pinpoint failures.
- **Integration**: pieces collaborating with real infrastructure — a real database, a real broker, a module's wiring.
- **E2E**: the deployed system through its public surface.

The shape follows **cost**: the higher you go, the slower the feedback and the vaguer the diagnosis — so most checks live at the bottom and only critical paths at the top. It's a **guide, not a mandate**: not every project needs all three levels, and the right proportion is each stack/project's call. What the pyramid actually forbids is **inverting it** — thousands of e2e tests over a handful of unit tests, where every change costs a full slow run to validate.

## One test file per unit under test — [canon, via SRP]

Each test file exercises exactly **one** unit. With single-responsibility units (see `solid`), per-unit equals per-behavior — the form every school converges on. What's forbidden is the lumped file testing many units at once: it hides which units are covered, and it's the test-side mirror of the god-class. File *placement* (mirrored tree, colocated specs…) → the stack skill.

## Test names are specification — [canon]

A test name states the behavior it verifies: a reader learns the rule without opening the body, and the suite reads as the unit's specification. If the name needs "and", it's two tests.

The *format* is per school and per stack tooling — all equivalent realizations of the same canon: `MethodName_StateUnderTest_ExpectedBehavior` (Osherove), `Should_X_When_Y`, Given-When-Then (BDD), `describe/it` sentences (JS). Each stack skill fixes its idiomatic one.

## Structure and independence — [canon]

- **Arrange – Act – Assert**, visually separated. One behavior per test; several asserts are fine when they verify facets of the same outcome.
- Tests are **independent and order-agnostic**: no shared mutable state, no dependence on execution order; each test builds what it needs.
- Shared builders/fixtures stay close to the tests that use them — a test file reads top-to-bottom without hunting elsewhere.
- No timing-based tests: inject the clock / control the scheduler. A sleep in a test is a flake with a countdown.

## Test data: Object Mother + Builder — [canon: Meszaros; our default]

Two complementary patterns build the Arrange, with a clear division of labour — reach for them instead of hand-constructing objects inline in every test:

- **Object Mother** — named, canonical instances that model **domain-meaningful states**: `Invoices.Overdue()`, `Customers.Premium()`. The default entry point; most tests read exactly one. **[our convention]** reach for a mother first.
- **Test Data Builder** — fluent, field-level **variation**: `.WithNoEmail()`, `.WithTwoItems()`. For the tweak a mother doesn't name.
- **Compose them — the mother returns a builder**: `Customers.Premium()` yields a builder preloaded with a valid premium customer; `.Build()` for the canonical case, `.WithNoEmail().Build()` for a one-off variation. Canonical default *and* on-demand variation, with no method explosion.

The guardrail: **a mother names a STATE, not a field tweak.** `Invoices.Overdue()` — yes; `Customers.PremiumButNoEmail()` — no, that's the builder's job. Once mother names start carrying `WithX`, the variation belongs in the builder. Well-segregated units (see `solid`) have few meaningful states, so the mother set stays small on its own.

Concrete form and placement — static classes, naming, where the mothers/builders live — are the stack skill's call.

## Behavior over implementation — [canon: classicist school]

- Verify **outputs and state changes**, not internal call sequences. Interaction-heavy tests break on refactors that change nothing observable — a test punishing improvement is failing its purpose.
- If the type's equality is shallow (collections compared by reference), assert **field by field** in round-trip tests.
- A test that restates its own mock setup asserts nothing.

## Test doubles — [canon: classicist school, our pick]

Two canonical schools exist: **classicist** (real collaborators and fakes; verify outcomes) and **mockist** (mocks at every boundary; verify interactions). **We deliberately follow the classicist school** — consistent with behavior-over-implementation, and the position the industry has converged on.

The ladder — **use the most real rung you can afford**:

1. **Real collaborator** whenever it's cheap — the real validator, the real value object.
2. **Fake** (a working lightweight implementation — an in-memory provider/store) when the real thing needs infrastructure.
3. **Mock only for what you genuinely can't control**: a third-party API, the clock — and for verifying an *outbound* contract at an architectural boundary ("the event was published"), where the interaction IS the observable behavior.

Never mock the thing under test — a partially-mocked SUT tests the mock, not the code.

## Coverage: rules, not lines — [our convention]

- Line coverage says a line *executed*; it doesn't say its behavior was *verified*. The question that matters: **"if someone deletes this rule, does a test go red?"** If not, the rule is uncovered — whatever the percentage says. (Mutation testing automates exactly this question.)
- Every business rule and every validation gets its **failing case** — including the red side of each validation rule.
- Edge cases are first-class: null/empty, boundaries (0, 1, max), invalid input, duplicates, out-of-order/idempotency where messages are consumed.
- Don't chase 100% line coverage through assertion-free or trivial tests — a covered line with no meaningful assert is worse than an uncovered one: it looks safe.
