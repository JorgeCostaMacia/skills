# Clean code — naming, functions, structure, comments

## Naming

Priority order when they conflict: **consistency > understandability > specificity > searchability > brevity**.

- **Consistency**: one concept, one word, everywhere. If the codebase says `Fetch`, don't introduce `Retrieve`/`Get` for the same idea. Match the surrounding style before inventing.
- **Understandability**: the name states what the thing *is* or *does* in domain terms. A reader shouldn't need the body to trust the name.
- **Specificity**: `pendingInvoices` over `data`; `retryLimitReached` over `flag`. Booleans read as assertions (`isActive`, `hasChanges`); functions as verbs; values as nouns.
- **Searchability**: grep-ability matters — avoid abbreviations that collide (`res`, `tmp`, `mgr`) and unnamed magic literals.
- **Ubiquitous language**: domain concepts use the business's own vocabulary (in the business's language); technical suffixes stay conventional English (`Validator`, `Handler`, `Service`, `Context`).
- Rename **the moment** a better name appears. A name that lies is worse than a TODO.

## Functions / methods

- **One job, stated by the name.** If describing it needs "and", split — by responsibility, never by line count.
- **Early returns over `else` pyramids**: handle guards and edge cases first; the happy path reads straight down.
- **Parameters**: a long parameter list usually means a missing concept — group what travels together into a type (data clump → object). A boolean parameter that switches behavior is two functions wearing one name.
- **Command/query separation**: a function either *does* something or *answers* something; doing both hides effects.
- **One level of abstraction per body**: don't mix "orchestrate the workflow" lines with "bit-shift the header" lines — push detail down into named helpers.

## Structure

- **Tell, don't ask**: move behavior to where the data lives instead of pulling data out to decide elsewhere. Chains like `a.getB().getC().doThing()` couple the caller to the whole path (Law of Demeter — talk to friends, not strangers).
- **Wrap primitives that carry rules.** An email, a money amount, a page size — if it has validation or invariants, it deserves a type (value object) so invalid values become unrepresentable. Plain primitives are fine for values with no rules.
- **First-class collections** when a collection has behavior of its own (invariants, filtering rules, aggregation); if it's just a list, keep it a list.
- **Fail fast**: validate at the boundary/construction and throw with a message that says *what* and *which value*; don't limp along with bad state.
- **Immutability by default** where the language makes it cheap; mutation is an optimization that needs a reason.

## Comments

- A comment is a **last resort after renaming and extracting failed**. Comments that restate the code rot into lies.
- Good comments say what the code *cannot*: a non-obvious constraint, a link to the spec/issue, why the obvious approach was rejected.
- Delete commented-out code — version control remembers it.

## Errors

- Handle errors **where something can be done about them**; elsewhere, let them propagate.
- Never swallow silently: log-and-continue is a decision — make it visible and justified.
- Errors carry context: what operation, on what input. Re-wrapping without adding information is noise.
