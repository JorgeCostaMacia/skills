---
name: clean-architecture
description: >-
  Clean architecture layering with hexagonal (ports & adapters) boundaries —
  stack-agnostic. Use when designing or structuring a backend service or
  module, deciding which layer some logic belongs to, adding a new dependency
  between projects/modules, defining interfaces/ports for external
  capabilities, reviewing architecture, or answering "where does this go?" —
  in any backend language or framework. It encodes the four layers
  (Domain / Application / Infrastructure / Presentation), the inward
  dependency rule, and ports & adapters at the boundaries.
---

# Clean architecture (with hexagonal boundaries)

Stack-agnostic principles. The concrete realization — project layout, DI wiring, file placement — belongs to the stack skill (e.g. `backend-net`). **These rules get revised, not forced: if one fights the reality of a project, question the rule.**

## The four layers

| Layer | Owns | Never contains |
| --- | --- | --- |
| **Domain** | entities, aggregates, value objects, domain events, domain exceptions, validation rules | ORM/serializer/transport types, framework code, IO |
| **Application** | use cases — one service per use case; orchestration: build/validate domain objects, execute changes, raise events | presentation concerns, transport details, business rules that belong in Domain |
| **Infrastructure** | adapters that touch the outside: persistence, external clients, IO | business logic, use-case orchestration |
| **Presentation** | delivery mechanisms: HTTP endpoints, bus consumers, CLI — **thin**: validate/convert input → call an Application service → return/publish | business logic of any kind |

## The dependency rule

Source dependencies point **inward only**: Presentation → Application → Domain ← Infrastructure.

- Domain imports nothing from the other layers — ever.
- Application knows Domain. It must not know Presentation.
- Infrastructure implements what the inner layers define; nothing inward depends on its concrete types.
- Presentation knows Application (and the module's public contracts) — never another module's internals.

Concrete test: pick any file and read its imports. Every import should be from the same layer or an inner one. An outward import is the defect, wherever it appears.

## The hexagonal lens (ports & adapters)

Hexagonal architecture doesn't name layers — it splits the world into **inside** (your model and use cases) and **outside** (everything else), connected through **ports**:

- A **port** is an interface the inside defines for a capability it needs (a clock, a store, a notifier) or exposes (a use case).
- A **driving adapter** calls the inside: HTTP endpoint, bus consumer, CLI command, scheduled job — that's what the Presentation layer contains.
- A **driven adapter** implements a port for the outside: repository over a raw driver, external API client — that's what Infrastructure contains.

Define a port **only where the inside genuinely needs an outside capability** — not around every library reflexively. Frameworks the stack has deliberately committed to are accepted couplings; the stack skill names them.

## Clean vs hexagonal vs onion — one idea, three vocabularies

All three enforce the same dependency rule. Clean contributes the **layer vocabulary** used here; hexagonal contributes the **boundary mechanics** (ports/adapters); onion is the same picture drawn as rings. Don't debate the labels — enforce the rule.

## Where things go — quick answers

- Input validation of a request/message → Presentation (its validator), before anything else runs.
- A business rule/invariant → Domain (the aggregate or its validator).
- "When X happens, do Y and Z" orchestration → Application.
- Talking to anything with a connection string or URL → Infrastructure.
- Mapping between contracts (DTOs) and domain → the boundary that owns the contract (Presentation for transport contracts, Infrastructure for persistence shape).
- A new module's public surface → one entry point; internals stay private to the module.
