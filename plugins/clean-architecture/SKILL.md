---
name: clean-architecture
description: >-
  Clean architecture layering with hexagonal (ports & adapters) boundaries —
  stack-agnostic and canon-anchored (Evans/Martin/Cockburn). Use when
  designing or structuring a backend service, bounded context or module,
  deciding which layer some logic belongs to, adding a dependency between
  projects/modules, defining interfaces/ports for external capabilities,
  reviewing architecture, or answering "where does this go?" — in any backend
  language or framework. It encodes the four layers living inside each
  bounded context, the inward dependency rule, ports for genuinely external
  capabilities, and the name mapping to Evans, Uncle Bob, Microsoft and
  Jason Taylor. Sections are marked [canon] or [our convention]; packaging
  and concrete realization live in the stack skills.
---

# Clean architecture (with hexagonal boundaries)

Canon-anchored: sections marked **[canon]** state the standard; **[our convention]** marks deliberate positions, each with its why. Packaging and concrete realization (projects, DI wiring, file layout) belong to the stack skill. **Rules get revised, not forced: if one fights the reality of a project, question the rule.**

## What each canon contributes

No single book describes this architecture — three do, each answering one question:

| Canon | Question it answers | Contribution |
| --- | --- | --- |
| **DDD — strategic** (Evans) | What are the modules? | **Bounded contexts**: one model, one language each. (Tactical DDD — aggregates, value objects, factories — lives in the `ddd` skill.) |
| **Clean / Layered** (Evans, Martin) | How is code layered, and which way do dependencies point? | The **four layers** and the **inward dependency rule**. |
| **Hexagonal** (Cockburn) | How does the boundary with the outside work? | **Ports & adapters**: driving adapters in, driven adapters out. |

## The four layers — [canon]

| Layer | Owns | Never contains |
| --- | --- | --- |
| **Domain** | entities, aggregates, value objects, domain events, domain errors, validation rules | ORM/serializer/transport types, IO. A stack skill MAY declare specific libraries as **accepted core couplings** — deliberate and named, never accidental. |
| **Application** | the use cases — orchestration: build/validate domain objects, execute the change, raise events. Use-case services are **concrete — no interface per use case**: a use case has exactly one implementation by definition, and an interface with a single implementer and no seam requirement is speculative (see `solid`). | presentation or transport concerns; business rules (they belong in Domain) |
| **Infrastructure** | adapters that touch the outside: persistence, external clients, IO. Mapping between persistence shape and domain lives here (the boundary owns its contract). | business logic, use-case orchestration |
| **Presentation** | the delivery mechanisms — HTTP endpoints, bus consumers, CLI, jobs — **thin**: validate/convert input → call an Application service → return/publish. Mapping between transport contracts and domain lives here. | business logic of any kind |

Layer names are the de-facto standard (and nearly Evans' own — see the name mapping below). Evans defines the Presentation layer as interpreting commands from the outside, **"the external actor might be another computer system rather than a human user"** — a bus consumer is this layer as literally as a controller.

## The dependency rule — [canon]

Source dependencies point **inward only**: Presentation → Application → Domain ← Infrastructure.

- Domain imports nothing from the other layers — ever.
- Application knows Domain. It must not know Presentation.
- Infrastructure implements what the inner layers define; nothing inward depends on its concrete types.
- Presentation knows Application (and the bounded context's public contracts) — never another context's internals.

Concrete test: pick any file and read its imports. Every import should be from the same layer or an inner one. An outward import is the defect, wherever it appears.

**[our convention] Exception, by declaration**: capabilities the stack has declared as **accepted core couplings** (see the Domain row above) may be consumed directly by inner layers — e.g. a full ORM's context acting as the aggregate's data accessor (see `ddd`) is used by Application as-is, with no port wrapped around it. The exception is explicit and named by the stack skill — never improvised case by case.

## The unit is the bounded context — [canon-adjacent: composition of two canons]

The four layers live **inside** each bounded context — self-contained, including its Presentation slice (its endpoints, its consumers). **Layers belong to a context, never to "the application"**: an application is one or more bounded contexts behind an **entry point** (API, worker, console, scheduled job…) that composes their public surfaces and the runtime pipeline, holding no logic of its own.

- Why canon-adjacent: Evans gives both ingredients (bounded contexts + the layered architecture) and Bob the rule; composing them — layers repeated per context — is the natural consequence, made explicit by the modular-monolith literature.
- A small app *is* one bounded context: same rule, the layers live inside it, keeping the context's logic separate from the bootstrap.
- How contexts and layers map to projects, packages or folders is **packaging — the stack skill's decision**.

## The hexagonal lens — [canon: Cockburn]

Hexagonal doesn't name layers — it splits the world into **inside** (your model and use cases) and **outside** (everything else), connected through **ports**:

- A **port** is an interface the inside defines for a capability it needs (a clock, a store, a notifier) or exposes (a use case).
- A **driving adapter** calls the inside: HTTP endpoint, bus consumer, CLI command, scheduled job — that's what the Presentation layer contains.
- A **driven adapter** implements a port for the outside: persistence, external API client — that's what Infrastructure contains.

### Ports in moderation — [our convention]

Define a port **only where the inside genuinely needs an outside capability**: the clock, a third-party API, a notifier, storage without a full ORM. Frameworks the stack has declared as accepted core couplings are **not** wrapped in speculative indirection — a port around a library you decided never to swap is cost without benefit (see `solid`: don't abstract for one implementation). Ports exist for *capabilities*, not for libraries you already married.

## One idea, three vocabularies — the name mapping

Clean, hexagonal and onion all enforce the same dependency rule with different words. Don't debate labels — enforce the rule. Orientation map:

| Reference | Their names → these layers |
| --- | --- |
| **Evans** (Layered Architecture) | UI *(or Presentation)* / Application / Domain / Infrastructure — nearly identical; the closest match |
| **Uncle Bob** (rings) | Entities → Domain · Use Cases → Application · Interface Adapters → Presentation + Infrastructure · Frameworks & Drivers → the entry point / composition |
| **Microsoft** (eShopOnWeb) | ApplicationCore ≈ Domain + Application · Web ≈ entry point + Presentation |
| **Jason Taylor** (template) | identical to these |
