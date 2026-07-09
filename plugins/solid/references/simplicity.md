# Simplicity — managing complexity deliberately

## Essential vs accidental complexity

- **Essential**: the domain is genuinely like that (tax rules, retry semantics, regulatory branching). You can't remove it — only *organize* it: name it, isolate it, test it.
- **Accidental**: we introduced it — indirection without a seam, premature configuration, clever abstractions, framework ceremony, defensive code for impossible states.

Every simplification effort targets the accidental kind. Confusing the two produces either naive designs (denying essential complexity) or baroque ones (accepting accidental complexity as inevitable).

## The three levers

- **YAGNI** — don't build for imagined futures. A feature, parameter, or extension point earns its existence with a *user today*. The future you designed for rarely arrives in the shape you guessed, and the speculative structure then fights the real requirement.
- **KISS** — among designs that work, pick the one a new reader understands fastest. "Clever" is a cost: every trick must buy something measurable (performance, correctness), otherwise write the boring version.
- **DRY — for knowledge, on the third occurrence** — DRY is about *facts and rules*, not text. Two fragments that look alike but encode different decisions must NOT be merged; two different-looking fragments encoding the same rule MUST be. Extract when the third occurrence appears and the shape has stabilized: **a minor duplication beats a wrong abstraction**, because a wrong abstraction spreads — every new case bends it further, and unwinding it costs more than the duplication ever did.

## Four elements of simple design (priority order)

1. **Works** — passes its tests.
2. **Expresses intent** — reveals what and why.
3. **No duplicated knowledge.**
4. **Minimal** — fewest moving parts that satisfy 1–3.

Order matters: never trade a lower number for a higher one (e.g. don't deduplicate — #3 — in a way that obscures intent — #2).

## Abstraction budget

Treat each layer of indirection as spending from a budget:

- An abstraction pays rent when it **hides a decision that can change independently** (storage engine, transport, algorithm) or **provides a needed seam** (tests, module boundary).
- An abstraction that merely *renames* a call, delegates 1:1, or exists "for symmetry" is debt.
- Prefer **fewer, deeper modules** (simple interface, substantial implementation) over many shallow ones — shallow modules push their complexity onto every caller.

## Heuristics

- If explaining the design takes longer than explaining the problem, the design is too big.
- Count the files a plausible next requirement would touch — the design minimizing that honest count wins.
- Deleting code is the highest-value refactor. Cultivate it: dead flags, unused options, expired experiments.
- When stuck between two designs, pick the one that's **easier to delete** later — reversibility beats theoretical perfection.
