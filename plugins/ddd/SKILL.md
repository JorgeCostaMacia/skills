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
  [our convention]; the concrete realization (base types, factories, mapping,
  error vehicle) lives in the stack skills.
---

# Domain-Driven Design (tactical)

Canon-anchored (Evans' *Domain-Driven Design*, Vernon's *Implementing DDD*): sections marked **[canon]** state the standard; **[our convention]** marks deliberate additions, each with its why. The concrete realization (base classes, validation library, error vehicle) belongs to the stack skill. **Rules get revised, not forced: if one fights the reality of a project, question the rule.**

## Ubiquitous language — [canon]

- Domain concepts — modules, aggregates, properties, events — are named in the **business's own language**, the words the domain experts use. Technical suffixes stay conventional English (`Validator`, `Service`, `Handler`, `Event`).
- One concept, one name, everywhere: code, contracts, storage, logs, conversation. If the business renames a concept, the code follows.
- If you can't name it in domain terms, you don't understand it yet — ask first, model later.

## Entities — [canon]

- An entity is defined by its **identity**, not its attributes: it stays the same entity through every change of state, and two entities are the same when their identities are, whatever their other values.
- The identity is assigned once and never changes.
- State changes through **behaviour that keeps the invariants** — not through setters that let a caller break them.

## Aggregates — [canon]

- An aggregate is the **consistency boundary**: loaded, validated and persisted as a unit; the thing whose invariants must hold at every save.
- Keep them **small**: if two clusters of fields change for different business reasons, that's two aggregates. Reference other aggregates by identity, not by object graph.
- The **root** is the one entity outsiders hold: the entities inside are reached through it, and only the root is loaded, saved and referenced from outside.
- Aggregates announce what happened through **domain events** (see below); no events escape a failed operation.

## Creation & hydration — [canon: the Factory pattern]

- **Creation goes through a factory** (`Create`…) that converts the input and **validates before returning** — an object cannot be *created* invalid. Factories are DDD's answer to guarded construction; Evans keeps a plain constructor for the simple cases. **[our convention]** Here it is *always* the factory, so every type has the same surface.
- **Converting is not normalizing** — [our convention]: a factory converts the input to **the type's own representation** (a trimmed text, an instant in UTC, a date at the store's precision) — that is what the type *is*, not a rule of the domain — and **never coerces** an invalid value into a valid one. A layer whose types represent the source's values as they came keeps every value as it came: there the conversion is a pass-through.
- **Hydration is a separate path**: a plain constructor reserved for **reconstitution** — infrastructure (ORM, deserializer) restoring already-validated state. Evans distinguishes creation from reconstitution explicitly, and only says a violated invariant is handled *differently* there; **[our convention]** hydration is *recovery*, not *entry*, and doesn't re-validate at all.

## Value objects — [canon]

- **Every attribute of an entity is a value object** — [our convention]: a primitive that carries a rule (email, money, quantity, identifier) is the canonical case — the rule belongs to the type, not scattered across callers — and a field with no rule today is wrapped all the same, because the type names the concept in the ubiquitous language and gives the rule a home the day it appears. The identity is the one named exception: it may stay a primitive where the persistence technology makes a wrapped key costly.
- Immutable; equality **by value within the type** (two Emails with equal value are equal; an Email never equals a Name with the same string).
- Conversion is one-directional: VO → primitive is cheap; **primitive → VO only through the factory**.
- **A value object never coerces and carries no defaults** — [our convention]: an out-of-range value is rejected, not pulled into range (a page size of 5000 silently served as 1000 hides the caller's mistake), and a default is policy — it belongs at the call site, where two callers can choose differently.

## Domain services — [canon]

- A **domain service** holds a business operation that belongs to no entity or value object — typically a rule that spans several aggregates. It is named in the ubiquitous language, holds no state, and lives in Domain — of its own context, or of the **shared kernel** when several contexts apply the same rule (a strategic pattern: see `clean-architecture`).
- **Rare by design**: if the rule fits an aggregate, it goes in the aggregate. A model whose rules have drifted into services is anemic.
- **Not an application service**: the application service orchestrates a use case (load, validate, execute, persist, publish) and holds no business rules; the domain service holds a rule and orchestrates nothing.

## Validation — two levels

**Level 1 — intrinsic validity (invariants): the type owns it — [canon]**

- *"Is this object correct in itself?"* Rules answerable by looking only at the object's values — **pure**: no services, no IO.
- Enforced by the **factory**: it runs the type's validator before returning — **nothing invalid escapes a factory**. (Instance-based validators may build-then-validate internally; the guarantee is at the factory boundary, not the micro-order inside it.) Being pure, the validator needs no injection.
- Structural, so it holds on **every** path — HTTP, bus, backfill — nobody has to remember it.

**Level 2 — use-case validity (preconditions) — [canon-adjacent]**

- *"Is this correct object admissible **here, now**?"* Rules that need services or state: uniqueness against a store, referenced things existing, business-state conflicts. Not properties of the object — properties of the **system state at a moment** (the same email is valid today and a duplicate tomorrow).
- **[our convention]** They live in **Application**, with injected dependencies, and reject with **domain errors** ("already exists"…). The canon puts them in a domain service (Vernon) or a Specification (Evans); we keep them in the use case, next to the load or insert they guard, and extract a domain service only when the same check repeats. Where a data constraint backs the rule (a unique index), the application check gives the friendly error and the constraint gives the concurrency-proof guarantee.
- The litmus test: **if the rule needs a service, it's level 2** — a type's validator must stay pure.

**How the rules are held — [our convention]**

- Each rule is defined **once**, in the type's own **composable validator**; the factory invokes it (level 1), and boundary validators (request/aggregate) **compose** the validators of their parts to return the **complete list of failures, per field, in one pass** — nothing invalid travels past the boundary, and errors never drip one construction at a time.
- **[our convention] A content rule lives once, in the layer that owns the content.** In a pipeline of layers — a landing layer, a standardizing layer, a store — the standardizing layer holds the content rules; a landing layer validates only what its own infrastructure needs (the identity, its own metadata) and keeps the content as it came. The two-level table says what *kind* of rule something is; this says *which layer* holds it.
- Multiple checkpoints, one source of rules — defense in depth, not duplication. *How* rules compose (declared constraints with collected violations, external composable validators…) is the **stack skill's decision** — judge designs against these principles, not against a particular mechanism.

## Data access — [canon: the repository role]

- **Each aggregate has its own data accessor**: one dedicated access point per aggregate, **internal to its module** — no module reaches into another aggregate's storage, and no shared god-context exposing everyone's tables to everyone. That boundary is what the repository pattern actually protects.
- *What the accessor is* — a repository class (port in Domain, adapter in Infrastructure) or a narrow per-aggregate ORM context — is the **stack skill's decision**.

## Domain events — [canon]

- Named **past tense**, after the business fact: `<X>Created`, `<X>Loaded`, `<X>Deleted`. An event states something that **already happened** — subscribers react, they can't veto.
- Microsoft's microservices guidance separates **domain events** (inside the model) from **integration events** (the contract that crosses contexts), with an explicit mapping between the two; Vernon instead publishes the domain event itself through messaging, as part of the context's published language, and lets each consumer translate on its own side. **[our convention, with Vernon]** The context's published events *are* its domain events: the root raises the event declared in its contract, so what the model announces is exactly what the outside sees — no second type, no mapping, no in-process dispatcher whose only job would be to copy fields. A separate internal event earns its place only when the model needs to raise something it does not publish, or the two shapes diverge.
- Collection/dispatch mechanics (how aggregates accumulate events, when they're published) → the stack skill.

## Domain errors — [canon-adjacent]

- A domain error states a **business problem** — not found, already exists, rule violated — and is **typed and identifiable by a fixed code**, so occurrences trace across logs and API responses regardless of message wording.
- Infrastructure failures (timeouts, connectivity) are **not** domain errors — they surface separately, and resilience policy (retries, redelivery) handles them; retrying a business error is pointless.
- *How* errors travel — an exception family per concept, a Result/Either type, error returns — is the **stack skill's decision**.

## Bounded contexts — [canon]

- A bounded context owns its model: its aggregates, its language, its rules. Other contexts integrate through its **published contracts** (events, commands), never by reaching into its internals. (Evans' MODULE is a smaller thing — a package inside one model; here "module" is the runtime's name for a bounded context.)
- The same business word may mean different things in different contexts — that's two concepts; don't force one shared model.
