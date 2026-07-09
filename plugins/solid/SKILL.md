---
name: solid
description: >-
  Transversal software-engineering principles for any code in any language,
  frontend or backend. Use when writing new code, implementing a feature,
  refactoring, reviewing code, naming things, deciding where logic belongs,
  splitting or merging classes/modules/components, introducing an abstraction,
  or judging whether code is "good enough" — even when the user doesn't name a
  principle explicitly (e.g. "clean this up", "does this look right?", "where
  should this go?"). It encodes SOLID, simple design, clean code, code smells,
  and complexity management — applied with judgment, not dogma: rules are
  signals to weigh, and a minor duplication beats a wrong abstraction.
---

# Engineering principles (transversal)

These apply to **all** code — any language, any layer, front or back. They are lenses for judgment, not laws: when two of them conflict, prefer the one that makes the code easier to change tomorrow.

## The core loop

1. **Understand the behavior** you need before touching code — inputs, outputs, edge cases. If you can't state it in one sentence, you're not ready.
2. **Make the smallest change that delivers it.** No speculative parameters, no "while I'm here" abstractions.
3. **Prove it works** — a test when the code is testable; a real execution when it isn't yet.
4. **Then refactor**: better names, remove duplication (rule of three), simplify structure. Refactor with green tests, never mid-change.

## Four elements of simple design — in priority order

1. **Works** — passes its tests / observably does the job.
2. **Expresses intent** — a reader gets *what* and *why* without archaeology.
3. **No knowledge duplication** — one fact lives in one place (code repetition is only a smell when it duplicates *knowledge*).
4. **Minimal** — nothing present "just in case". YAGNI.

When these conflict, the lower number wins.

## SOLID — the five questions

Ask these when designing or reviewing a type/module (details and examples in [references/solid-principles.md](references/solid-principles.md)):

- **S**RP — does it have **one reason to change**? (One actor/concern per unit.)
- **O**CP — can foreseeable variation be added **without editing** this code? (Only invest where variation is *actually expected*.)
- **L**SP — can any implementation substitute the abstraction **without surprising** the caller?
- **I**SP — does anyone depend on methods it **doesn't use**?
- **D**IP — does policy depend on **abstractions**, not concretions? (Dependencies point toward the domain.)

## Naming

Priority: **consistent** with the codebase > **understandable** > **specific** > **searchable** > short. Rename the moment a better name appears — a misleading name is a bug waiting to be misread. Full guidance: [references/clean-code.md](references/clean-code.md).

## Code smells are signals, not sins

Long method, large class, feature envy, data clump, primitive obsession, shotgun surgery… A smell means *stop and look*, not *mechanically split*. Catalog with the fix — and the false-positive cases — in [references/code-smells.md](references/code-smells.md).

## Complexity

Distinguish **essential** complexity (the domain is genuinely like that) from **accidental** complexity (we introduced it). Fight only the accidental kind: YAGNI, KISS, DRY-after-the-third-time. Deep dive: [references/simplicity.md](references/simplicity.md).

## Anti-dogma — when NOT to apply

- **A minor duplication beats a wrong abstraction.** Extract on the third occurrence, when the shape is stable — not the second.
- **Don't split by size.** A 60-line method with one clear job beats four 15-line methods a reader must reassemble. Split by *responsibility*, never by line count.
- **Don't abstract for one implementation.** An interface with a single implementer and no seam requirement is noise (test seams and module boundaries are valid seam requirements).
- **Declarative/config-like code is exempt from "too long"**: exhaustive mappings, schema definitions, and wiring tables are *meant* to be long and boring.
- **Frameworks the project committed to are not "coupling to fix"** — don't wrap them in speculative indirection.

## Review checklist (before calling it done)

- [ ] Behavior proven (test or real run).
- [ ] Names say what things are; no comment explains what a rename could.
- [ ] Each new/edited unit has one reason to change.
- [ ] No knowledge duplicated; no abstraction with a single caller and no seam need.
- [ ] Nothing speculative: every parameter, branch, and config option has a user today.
- [ ] Errors handled where they can be *acted on* — not swallowed, not re-wrapped noise.

## References

| File | Load when |
| --- | --- |
| [references/solid-principles.md](references/solid-principles.md) | applying/reviewing SOLID in a concrete design |
| [references/clean-code.md](references/clean-code.md) | naming, functions, structure, comments |
| [references/code-smells.md](references/code-smells.md) | something feels off and you want the catalog + fixes |
| [references/simplicity.md](references/simplicity.md) | fighting complexity, deciding YAGNI/DRY/KISS trade-offs |
