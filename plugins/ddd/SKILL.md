---
name: ddd
description: >-
  Tactical Domain-Driven Design — stack-agnostic and canon-anchored
  (Evans/Vernon). Use when modeling a domain: creating or reviewing entities,
  aggregates, value objects, factories, domain events, domain errors or
  validators; naming domain concepts; deciding where a business rule or
  invariant lives; designing how aggregates are created, hydrated, validated
  and persisted; or defining what a module/bounded context owns — in any
  backend language or framework. Every section is marked [canon] or
  [our convention]; stack-specific mechanics (validation library, error
  vehicle, event dispatching, accessor form) live in the stack skills.
---

# Domain-Driven Design (tactical)

Canon-anchored (Evans' *Domain-Driven Design*, Vernon's *Implementing DDD*): sections marked **[canon]** state the standard; **[our convention]** marks deliberate additions, each with its why. The concrete realization (base classes, validation library, error vehicle) belongs to the stack skill. **Rules get revised, not forced: if one fights the reality of a project, question the rule.**

## Ubiquitous language — [canon]

- Domain concepts — modules, aggregates, properties, events — are named in the **business's own language**, the words the domain experts use. Technical suffixes stay conventional English (`Validator`, `Service`, `Handler`, `Event`).
- One concept, one name, everywhere: code, contracts, storage, logs, conversation. If the business renames a concept, the code follows.
- If you can't name it in domain terms, you don't understand it yet — ask first, model later.

## Aggregates — [canon]

- An aggregate is the **consistency boundary**: loaded, validated and persisted as a unit; the thing whose invariants must hold at every save.
- Keep them **small**: if two clusters of fields change for different business reasons, that's two aggregates. Reference other aggregates by identity, not by object graph.
- Aggregates announce what happened through **domain events** (see below); no events escape a failed operation.

## Creation & hydration — [canon: the Factory pattern]

- **Creation always goes through a factory** (`Create`…) that normalizes input and **validates before returning** — an object cannot be *created* invalid. Factories are DDD's answer to guarded construction.
- **Hydration is a separate path**: a plain constructor reserved for **reconstitution** — infrastructure (ORM, deserializer) restoring already-validated state. Hydration is *recovery*, not *entry*; it doesn't re-validate. (Evans distinguishes creation from reconstitution explicitly.)

## Value objects — [canon]

- **Wrap primitives that carry rules** — email, money, quantity, identifier: the rule belongs to the type, not scattered across callers.
- Immutable; equality **by value within the type** (two Emails with equal value are equal; an Email never equals a Name with the same string).
- Conversion is one-directional: VO → primitive is cheap; **primitive → VO only through the factory**.
- Values with no rules stay plain primitives — don't wrap for ceremony.

## Validation — two levels

**Level 1 — intrinsic validity (invariants): the type owns it — [canon]**

- *"Is this object correct in itself?"* Rules answerable by looking only at the object's values — **pure**: no services, no IO.
- Enforced by the **factory**: it runs the type's validator before returning — **nothing invalid escapes a factory**. (Instance-based validators may build-then-validate internally; the guarantee is at the factory boundary, not the micro-order inside it.) Being pure, the validator needs no injection.
- Structural, so it holds on **every** path — HTTP, bus, backfill — nobody has to remember it.

**Level 2 — use-case validity (preconditions): the use case owns it — [canon]**

- *"Is this correct object admissible **here, now**?"* Rules that need services or state: uniqueness against a store, referenced things existing, business-state conflicts. Not properties of the object — properties of the **system state at a moment** (the same email is valid today and a duplicate tomorrow).
- They live in **Application**, with injected dependencies, and reject with **domain errors** ("already exists"…). Where a data constraint backs the rule (a unique index), the application check gives the friendly error and the constraint gives the concurrency-proof guarantee.
- The litmus test: **if the rule needs a service, it's level 2** — a type's validator must stay pure.

**How the rules are held — [our convention]**

- Each rule is defined **once**, in the type's own **composable validator**; the factory invokes it (level 1), and boundary validators (request/aggregate) **compose** the validators of their parts to return the **complete list of failures, per field, in one pass** — nothing invalid travels past the boundary, and errors never drip one construction at a time.
- Multiple checkpoints, one source of rules — defense in depth, not duplication. *How* rules compose (declared constraints with collected violations, external composable validators…) is the **stack skill's decision** — judge designs against these principles, not against a particular mechanism.

## Data access — [canon: the repository role]

- **Each aggregate has its own data accessor**: one dedicated access point per aggregate, **internal to its module** — no module reaches into another aggregate's storage, and no shared god-context exposing everyone's tables to everyone. That boundary is what the repository pattern actually protects.
- *What the accessor is* — a repository class (port in Domain, adapter in Infrastructure) or a narrow per-aggregate ORM context — is the **stack skill's decision**.

## Domain events — [canon]

- Named **past tense**, after the business fact: `<X>Created`, `<X>Loaded`, `<X>Deleted`. An event states something that **already happened** — subscribers react, they can't veto.
- **Domain events ≠ integration events**: the domain event belongs to the model, *inside* the context; the integration event is a **contract** (primitives only) that crosses contexts. They are different types, and the mapping between them is explicit.
- Collection/dispatch mechanics (how aggregates accumulate events, when they're published) → the stack skill.

## Domain errors — [canon-adjacent]

- A domain error states a **business problem** — not found, already exists, rule violated — and is **typed and identifiable by a fixed code**, so occurrences trace across logs and API responses regardless of message wording.
- Infrastructure failures (timeouts, connectivity) are **not** domain errors — they surface separately, and resilience policy (retries, redelivery) handles them; retrying a business error is pointless.
- *How* errors travel — an exception family per concept, a Result/Either type, error returns — is the **stack skill's decision**.

## Modules / bounded contexts — [canon]

- A module owns its model: its aggregates, its language, its rules. Other modules integrate through its **published contracts** (events, commands), never by reaching into its internals.
- The same business word may mean different things in different contexts — that's two concepts; don't force one shared model.
