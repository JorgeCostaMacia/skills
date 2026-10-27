---
name: validation-net
description: >-
  The .NET realization of validation with FluentValidation, for anything that
  gets validated (the two-level model's principles are in `ddd`). Use when
  writing or reviewing FluentValidation validators — for a request, a query
  criterion, a value object or an aggregate —, when deciding where a rule
  lives (type invariant vs use-case precondition), wiring validators into DI
  or assembling them without a container, raising the typed
  ValidationException, or mapping validation failures to ProblemDetails. It
  encodes: one validator and one exception per validated thing, per-call
  validators from DI or static Create() chains, composition with
  Include/SetValidator, every rule evaluated (no Cascade), the rule and
  message conventions, and level-2 preconditions in Application with DI.
---

# Validation in .NET (FluentValidation)

The .NET *how* of the two-level model defined in `ddd` (load it for the *why*). **FluentValidation is the accepted core coupling** for validation — deliberate, evaluated, not to be wrapped or replaced. This skill covers validators for anything — a request, a query criterion, a domain type; how a domain model applies it (which validator each type builds, and when) is `ddd-net`'s.

## The two levels, realized

| Level | Question | Lives in | DI |
| --- | --- | --- | --- |
| **1 — Invariants** | "Is this object correct in itself?" | The type: its validator, invoked by its `Create` | **Never** — type rules are pure (no services) |
| **2 — Preconditions** | "Is this correct object admissible *here, now*?" | The use case (Application): checks with injected dependencies | **Always** — that's what marks a rule as level 2 |

Litmus test: **if the rule needs a service, it's level 2.** (Format, ranges, cross-field coherence → the type. Uniqueness, referenced-thing-exists, state conflicts → Application.)

## One validator and one exception per validated thing

Everything that gets validated — a value object, an aggregate, an entity, a request, a query criterion — has **its own validator** (`<X>Validator`) holding all its rules, and **its own exception** (`<X>ValidationException`) that the validator raises. Nothing validates through a generic validator, and nothing throws a generic exception.

## The validator pattern

```csharp
internal sealed class FacturaValidator : AbstractValidator<Factura>
{
    // The ONLY constructor: receives its composition — usable via DI when needed.
    public FacturaValidator(IValidator<FacturaEmail> email)
    {
        RuleFor(v => v.Email)                   // a part: its own validator's rules
            .SetValidator(email);

        RuleFor(v => v.Total)                   // this validator's own rule
            .GreaterThanOrEqualTo(0);
    }

    // The self-contained assembly: chains the OTHER validators' Create() —
    // never `new`s another validator (its chain is its own knowledge).
    public static FacturaValidator Create() => new FacturaValidator(FacturaEmailValidator.Create());

    // Failures raise the thing's OWN exception, never a generic one.
    protected override void RaiseValidationException(
        ValidationContext<Factura> context,
        ValidationResult result)
        => throw new FacturaValidationException(result.Errors);
}
```

- **Two ways to obtain a validator, one shape.** Resolved from DI — the standard FluentValidation way, for anything validated where a container exists (requests, query criteria; see *Boundary*). Or assembled with `XValidator.Create()` — for validation that runs **where there is no container**, such as inside a static factory: the chain builds itself, nothing is injected. The constructor taking its ingredients is what lets the same validator serve both.
- **Ctor receives ingredients; `Create` fabricates complete.** A validator with nothing to compose keeps a parameterless ctor and a trivial `Create() => new XValidator();`.
- **`RaiseValidationException` is not optional.** Without the override FluentValidation throws its own `ValidationException`, which is *not* `JorgeCostaMacia.Exception`'s base — the one a bus retry policy excludes — so a malformed message would climb the whole retry ladder instead of being parked.
- **Compose, don't repeat**: `SetValidator` brings a part's rules under its property (an optional part adds `.When(v => v.Part is not null)`); `Include` brings a validator hierarchy's rules onto the same object (a derived type including its base's: the documentation only promises `Include` for the same type; across the hierarchy it works through `IValidator<in T>`'s contravariance, and the suites pin it). Cross-field rules are the composing validator's own.
- One `ValidateAndThrow` reports **every** failure with its property path (`Email.Value`, `Total`) in one exception — the full report `ddd` asks for. The pattern recurses through any depth of parts.
- **Per-call, no caching**: a validator is built for each validation (`XValidator.Create().ValidateAndThrow(x)`). Construction costs a few gen0 allocations (FluentValidation caches compiled accessors globally), so a cached static instance is at most a later, profiler-driven optimization — and never for a validator with a bound on "now": that bound is evaluated when the validator is built, and a cached one would freeze it at process start.
- Lambda style: **`v =>`** (nested scopes `v2`), not `value =>`. Explicit types throughout: `new XValidator(...)`, never `new(...)`.
- No `new` modifier on a validator's `Create` (validators don't inherit each other; they `Include`).

## Writing the rules

- **No `Cascade(CascadeMode.Stop)`.** Every rule runs, so an empty value may report two failures (`NotEmpty` and the `Must`) — accepted. A validator can be included by others; it must not decide how a chain it is part of stops.
- **A base validator rejects only what is not the type — and most check nothing at all.** `StringValueObjectValidator`, `Int`, `Long`, `Decimal`, `Double`, `Float`, `Bool` and `Byte` carry no rule (an empty string passes); `Uuid` is not empty; `DateTime` is after 1900 and at most a century ahead; `PageNumber` / `PageSize` are above zero; `Email`, `Ipv4`, `Json` and `Uri` check their format. Policy — not empty, a range, a closed list — lives in the validator of the thing that has it, never assumed from the base.
- **Bounds are literals in the rule**, never a public constant to clamp against, and they are pinned by two tests either side.
- **Messages**: a custom check names its rule and says what was expected:

  ```csharp
  RuleFor(v => v.Value)
      .Must(v => MailAddress.TryCreate(v, out MailAddress? address) && address.Address == v)
      .WithErrorCode("EmailValidator")
      .WithMessage("{PropertyName} must be an email");
  ```

  A closed list needs no code: `.Must(_types.Contains).WithMessage("{PropertyName} must be one of " + string.Join(", ", _types))`. Built-in rules (`NotEmpty`, `LessThan`…) keep FluentValidation's own messages.

## The ValidationException

`<X>ValidationException` derives the shared base — `JorgeCostaMacia.Exception`'s `ValidationException`, or the one its base type already extends (`DateTimeValueObjectValidationException`…) — and carries a **fixed GUID code** (a UUIDv7 minted for that type, never copied — `ddd-net` has the rule and names the test that guards it) and the failure list (`ImmutableList<ValidationFailure>`). At an HTTP boundary that list becomes per-field ProblemDetails errors (`JorgeCostaMacia.Http.ProblemDetails` renders it). The rest of a domain's exceptions (`NotFound`, `Exist`) belong to `ddd-net`.

## Boundary — standard FluentValidation, when there is one

When a boundary validates its input (an HTTP request, a query criterion), its validator runs **first** in the handler (`validator.ValidateAndThrow(request)`), so the caller gets the complete per-field report in one response. It speaks the framework's idiom: constructor injection, normal DI registration (`AddScoped<IValidator<XRequest>, XRequestValidator>()`), the container composes the graph. Same shape as everything else: `<X>Request` + `<X>RequestValidator` + `<X>RequestValidationException`.

A boundary validator is **optional**. Where the content is validated once, in the layer that owns it, a boundary that only passes data on may validate nothing but what its own infrastructure needs — a missing key is still rejected by deserialization (`RespectRequiredConstructorParameters`). Don't add a request validator that duplicates a rule another layer owns.

## Level 2 — preconditions in Application

```csharp
// In the use-case service — dependencies injected, rejects with the family's domain error:
Factura? existing = await _context.FacturaRepository.FindAsync(aggregate.Id);
if (existing is not null)
{
    throw new FacturaExistException(...);
}
```

- Level-2 checks stay **in the use case** — usually glued to the load or insert they guard (a "validator with a repository inside" doubles the query and blurs the vocabulary: a validator constructor asking for a repository is the red flag). If the same check repeats across use cases, extract a helper or domain service — not a FluentValidation validator.
- Uniqueness gets the **double defense**: the application check gives the friendly domain error; the **unique index** gives the concurrency-proof guarantee. What its violation becomes depends on the write. Where the write is **idempotent** — a version travels with the data and a guard turns a second attempt into an update or a no-op — a duplicate key is a race between two writers: let it propagate, the retry resolves it (see `messaging`). Where it is not, the duplicate is a business fact: translate it to the family's `<X>ExistException` — a 409 at an HTTP boundary, and in a consumer an exception its retry policy excludes, since repeating the write cannot change the answer.

## Testing this model

- Per rule, its **red case**: `Create_OnInvalidX_ThrowsFamilyValidationException`.
- A bound from both sides: the last accepted value and the first rejected one.
- One `ValidateAndThrow` reports ALL failing rules — assert the failure list, not just the throw.
- A validator is never cached in a test helper either: the time bounds would freeze with it.
