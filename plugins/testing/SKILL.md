---
name: testing
description: >-
  Testing principles and the TDD loop — stack-agnostic, any language, front or
  back. Use when writing or reviewing tests, deciding what to test and how to
  name it, judging whether coverage is enough, starting a feature (test-first),
  or when tests are flaky/coupled/unreadable. It encodes: done means tested,
  the pragmatic red-green-refactor loop, test names as specification,
  Arrange-Act-Assert, test independence, asserting behavior over
  implementation, and rule coverage over line coverage. Test file PLACEMENT is
  stack-specific and lives in each stack's skill.
---

# Testing & TDD (principles)

Stack-agnostic. Where test files live (mirrored test project, colocated specs…) and which framework/tooling to use is each stack skill's decision. **These rules get revised, not forced: if one fights the reality of a project, question the rule.**

## Done means tested

A use case, rule, or fix without tests **doesn't exist**. Tests are part of the change, not a follow-up task — same commit, same PR, same definition of done.

## The loop — pragmatic red-green-refactor

1. **Red** — write the failing test that states the expected behavior. Name it first: if you can't name it, you don't know what you're building yet.
2. **Green** — the simplest code that passes. No gold-plating.
3. **Refactor** — with green tests: names, duplication, structure. Tests are code — they refactor too.

Pragmatics over ritual: an exploratory spike may come first — but then the tests are written **before the work is considered done**, and the design is re-checked against them. What's non-negotiable is the end state (tested), not the ceremony.

## One test file per unit under test

Never lump several units' tests into one file — it becomes impossible to see what's covered and what isn't. (Where that file *lives* — mirrored tree, colocated next to the source — is the stack skill's call.)

## Names are the specification

- `Method_Expectation` / `Method_OnCondition_Expectation` (or the stack's idiomatic equivalent — `describe/it` sentences in JS): a reader learns the rule without opening the body.
- If the name needs "and", it's two tests.

## Structure and independence

- **Arrange – Act – Assert**, visually separated. One behavior per test; several asserts are fine when they verify facets of the same outcome.
- Tests are **independent and order-agnostic**: no shared mutable state, no dependence on execution order; each test builds what it needs.
- Shared builders/fixtures stay close to the tests that use them — a test file should read top-to-bottom without hunting elsewhere.
- No timing-based tests: inject the clock / control the scheduler. A sleep in a test is a flake with a countdown.

## Assert behavior, not implementation

- Verify outputs and state changes, not internal call sequences. Interaction-heavy mock tests break on refactors that change nothing observable — that's the test punishing improvement.
- If the type's equality is shallow (collections compared by reference), assert **field by field** in round-trip tests.
- A test that restates its own mock setup asserts nothing.

## Coverage: rules, not lines

- Chase **rule coverage**: every business rule and every validation has a test that fails when the rule breaks. That includes the failing side — each validation rule gets its red case.
- Edge cases are first-class: null/empty, boundaries (0, 1, max), invalid input, duplicates, out-of-order/idempotency where messages are consumed.
- Don't chase 100% line coverage through assertion-free or trivial tests — a covered line with no meaningful assert is worse than an uncovered one: it looks safe.
