# SOLID principles — applied with judgment

Each principle below: what it really claims, how to check it, and when applying it would make things worse.

## Single Responsibility (SRP)

**Claim:** a unit (class/module/component) should have one reason to change — one *actor or concern* whose requirements drive its edits.

**Check:** describe the unit in one sentence without "and"/"or". List who asks for changes to it; more than one distinct source of change pressure → split along that line.

**Don't overdo it:** responsibility ≠ "does one method call". A cohesive unit may have many methods serving one concern. Splitting one concern across five files *creates* shotgun surgery — the exact disease SRP fights.

## Open/Closed (OCP)

**Claim:** where variation is *expected*, design so new variants are added by adding code (new class/strategy/handler), not by editing a growing conditional.

**Check:** find the `switch`/`if-else` chains over a type/kind. If the same chain appears in several places, or new kinds arrive regularly → replace with polymorphism/registration (one new file per new kind).

**Don't overdo it:** a `switch` that exists in ONE place over a stable set is *simpler* than a strategy hierarchy. OCP is an investment — pay it where change actually happens, not everywhere it theoretically could.

## Liskov Substitution (LSP)

**Claim:** anything implementing an abstraction must honor its contract — no stronger preconditions, no weaker postconditions, no surprise exceptions, no "not supported" methods.

**Check:** for each implementation ask "could a caller holding the abstraction be surprised?" A subtype that throws `NotSupportedException`, ignores a parameter, or returns null where the base never did → the hierarchy is wrong.

**Fix direction:** don't patch the caller with type checks; fix the abstraction (split it — see ISP) or use composition instead of inheritance.

## Interface Segregation (ISP)

**Claim:** no client should be forced to depend on members it doesn't use.

**Check:** implementations with empty/throwing members, or callers importing a wide interface to call one method → split into role interfaces (reader vs writer, query vs command).

**Don't overdo it:** one interface per method is fragmentation. Split along *client roles*, not method count.

## Dependency Inversion (DIP)

**Claim:** policy (domain/use cases) depends on abstractions; detail (IO, frameworks, transport) implements them. Dependencies point *inward*.

**Check:** does domain code import a driver, an HTTP client, a serializer, an ORM type? That's an arrow pointing the wrong way — define the port in the domain and implement it outside.

**Don't overdo it:** DIP is about *direction*, not about wrapping everything. Infrastructure code may use infrastructure libraries directly; and a framework the project has deliberately committed to (its ORM, its DI container, its validation library) doesn't need an anticorruption layer around it. Invert where the domain would otherwise depend on detail — not as a reflex.

## How they interact

- SRP and ISP are the same idea at different granularity (unit vs contract).
- LSP violations are usually ISP/abstraction-shape problems.
- OCP is *implemented* through DIP (variation behind an abstraction).
- All five serve one goal: **change stays local**. When in doubt, ask "if requirement X changes, how many files do I touch?" — the design with the smaller honest answer wins.
