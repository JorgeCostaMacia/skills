# Code smells — catalog with fixes and false positives

A smell is a **signal to investigate**, not an order to refactor. For each: what it looks like, the usual fix, and when it's actually fine.

## Bloaters

- **Long method** — does several jobs; you scroll to understand it.
  *Fix:* extract by responsibility (each extraction gets a name that means something).
  *Fine when:* it's one linear job — a declarative mapping, a config/wiring table, an exhaustive fluent configuration. Long-and-boring beats fragmented.
- **Large class** — accumulates unrelated concerns; many fields used by disjoint method groups.
  *Fix:* split along the field-usage clusters (each cluster is a hidden class).
  *Fine when:* it's cohesive — one concern with a wide but uniform surface.
- **Long parameter list** — data that always travels together passed one by one.
  *Fix:* introduce the missing type (parameter object / value object).
- **Data clumps** — the same group of fields repeats across signatures/classes.
  *Fix:* same — name the concept.
- **Primitive obsession** — rules attached to raw strings/ints scattered across callers.
  *Fix:* value object that validates on creation; invalid values become unrepresentable.
  *Fine when:* the value has no rules — don't wrap for ceremony.

## Change amplifiers

- **Divergent change** — one class edited for many unrelated reasons.
  *Fix:* SRP split by reason-to-change.
- **Shotgun surgery** — one logical change forces edits in many files.
  *Fix:* the knowledge is scattered; gather it into one place. (Note: over-splitting causes this — the cure for one smell can be the disease of the other.)
- **Parallel inheritance** — every new subclass in hierarchy A demands one in B.
  *Fix:* merge the hierarchies or replace one with composition.

## Coupling smells

- **Feature envy** — a method reads another object's data more than its own.
  *Fix:* move the method to where the data lives (tell, don't ask).
  *Fine when:* it's a deliberate boundary type whose job IS transformation (mappers, converters, projections).
- **Inappropriate intimacy** — two classes reach into each other's internals.
  *Fix:* narrow the contract between them; hide the internals.
- **Message chains** — `a.b().c().d()`.
  *Fix:* let the first object answer directly.
  *Fine when:* it's a fluent builder/configuration API — chaining is the design there.
- **Middle man** — a class that only delegates.
  *Fix:* remove it and call the target.
  *Fine when:* it's a real boundary (anti-corruption, published facade, seam for testing).

## Abstraction smells

- **Speculative generality** — hooks, parameters, interfaces "for the future" with one concrete user and no seam requirement.
  *Fix:* delete; YAGNI. Re-introduce when the second real user arrives.
- **Switch/if-else on type or kind, repeated** — the same discrimination in several places.
  *Fix:* polymorphism or a registration table (OCP).
  *Fine when:* it appears in exactly one place over a stable, closed set.
- **Refused bequest** — a subtype ignores or throws on inherited members.
  *Fix:* the hierarchy is wrong — split the interface (ISP) or use composition (LSP).
- **Temporary field** — fields only meaningful during one operation.
  *Fix:* the operation is a hidden class; move the fields into it.

## Hygiene

- **Dead code** — unreached branches, unused members, commented-out blocks. *Fix:* delete; git remembers.
- **Duplicate knowledge** — the same rule/constant/format in two places (worse than duplicate *text*: they drift apart silently). *Fix:* single source of truth. Apply the rule of three for incidental code similarity.
- **Comments as deodorant** — a comment explaining what a rename/extraction would make obvious. *Fix:* do the rename/extraction, drop the comment.
