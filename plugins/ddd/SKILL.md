---
name: ddd
description: >-
  Tactical Domain-Driven Design — stack-agnostic. Use when modeling a domain:
  creating or reviewing entities, aggregates, value objects, domain events,
  domain exceptions or validators; naming domain concepts; deciding where a
  business rule or invariant lives; designing what a module/bounded context
  owns — in any backend language or framework. It encodes ubiquitous language,
  aggregates as consistency boundaries, validate-on-creation, value objects,
  past-tense domain events, and the family pattern (aggregate + validator +
  specific exceptions).
---

# Domain-Driven Design (tactical)

Stack-agnostic principles. The concrete realization (base classes, factory syntax, validation library) belongs to the stack skill. **These rules get revised, not forced: if one fights the reality of a project, question the rule.**

## Ubiquitous language

- Domain concepts — modules, aggregates, properties, events — are named in the **business's own language**, the words the domain experts use. Technical suffixes stay conventional English (`Validator`, `Service`, `Handler`, `Event`).
- One concept, one name, everywhere: code, contracts, storage, logs, conversation. If the business renames a concept, the code follows.
- If you can't name it in domain terms, you don't understand it yet — model later, ask first.

## Aggregates

- An aggregate is the **consistency boundary**: loaded, validated and persisted as a unit; the thing whose invariants must hold at every save.
- **Validate on creation**: construction goes through a factory that normalizes input, and validation runs before the instance travels anywhere. An invalid aggregate must be unrepresentable downstream — fail fast at the boundary.
- Aggregates **collect their domain events** while a use case executes; the events are pulled and published only after the operation succeeds — no events escape a failed operation.
- Scope: if two clusters of fields change for different business reasons, that's two aggregates. Keep them as small as the invariants allow.

## Value objects

- **Wrap primitives that carry rules** — email, money, quantity, identifier: the rule belongs to the type, not scattered across callers.
- Immutable; equality **by value within the type** (two Emails with equal value are equal; an Email never equals a Name with the same string).
- Conversion is one-directional: VO → primitive is cheap; **primitive → VO only through the creating factory** (which normalizes: trim, defaults) — never an implicit reverse conversion.
- Values with no rules stay plain primitives — don't wrap for ceremony.

### Where VO validation runs — a deliberate deviation from the textbook

The textbook says a VO **self-validates at construction** (an invalid VO cannot exist). **Our convention is different, on purpose**: each VO's rules live in its own **composable validator**, and the aggregate/request validator at the boundary **extends the validators of its VOs** — so ONE validation pass returns the **complete list of failures** to the end user (per-field errors in one response), instead of failing at the first invalid value.

- Trade-off accepted: the guarantee shifts from "invalid is unrepresentable" to "**validated at the boundary before it travels**" — which the family pattern enforces (the entry point always runs the validator before anything else).
- Why: user-facing quality. Drip-feeding errors one construction at a time is hostile; a full validation report is what forms and APIs need.
- Don't "fix" external VO validators back into self-validating constructors — the composition is the point.

## The family pattern

A domain concept ships as a **complete family**, never alone:

| Member | Role |
| --- | --- |
| `<X>` | the aggregate/entity |
| `<X>Validator` | ALL validation rules for it |
| `<X>ValidationException` | raised on rule violations |
| `<X>NotFoundException` | requested but doesn't exist |
| `<X>ExistException` | created but already exists |

- Each exception type carries a **fixed unique code** (a constant, e.g. a GUID) so occurrences are identifiable in logs and API responses regardless of message wording or language.
- The validator raises the **family's own exception**, never a generic one — catch sites and error mapping stay precise.
- The same shape applies at boundaries: a request/message gets its own `Request` + `RequestValidator` + `RequestValidationException`.

## Domain events

- Named **past tense**, after the business fact: `<X>Created`, `<X>Loaded`, `<X>Deleted`. An event states something that already happened — subscribers react, they can't veto.
- Events carry **primitives** (ids, values, timestamps), not domain objects — they're contracts that cross boundaries.

## Domain exceptions vs the rest

- Domain exceptions state a **business problem** (not found, already exists, rule violated) — they're part of the model and map cleanly to API errors.
- Infrastructure failures (timeouts, connectivity) are NOT domain exceptions — let them surface as what they are; retry/resilience policy handles them elsewhere.

## Modules / bounded contexts

- A module owns its model: its aggregates, its language, its rules. Other modules integrate through its **published contracts** (events, commands), never by reaching into its internals.
- The same business word may mean different things in different modules — that's two concepts; don't force one shared model.
