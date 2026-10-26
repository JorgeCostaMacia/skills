---
name: validation-net
description: >-
  The .NET realization of the two-level validation model (see the `ddd` skill
  for the principles, `ddd-net` for the factories that call these
  validators). Use when writing or reviewing FluentValidation validators —
  for a value object, an aggregate, a request or a query criterion —, when
  deciding where a rule lives (type invariant vs use-case precondition),
  wiring validators into DI, raising the typed ValidationException, or
  mapping validation failures to ProblemDetails. It encodes: per-call
  validators assembled with static Create() chains, composition with
  Include/SetValidator, every rule evaluated (no Cascade), the rule and
  message conventions, the factory-vs-DI rule, and level-2 preconditions in
  Application with DI.
---

# Validation in .NET (FluentValidation)

The .NET *how* of the two-level model defined in `ddd` (load it for the *why*). **FluentValidation is the accepted core coupling** for validation — deliberate, evaluated, not to be wrapped or replaced. *When* a type validates — its `Create`, through a private `Validate()` — is `ddd-net`'s; this skill is about the validators themselves.

## The two levels, realized

| Level | Question | Lives in | DI |
| --- | --- | --- | --- |
| **1 — Invariants** | "Is this object correct in itself?" | The type: its validator, invoked by its `Create` | **Never** — type rules are pure (no services) |
| **2 — Preconditions** | "Is this correct object admissible *here, now*?" | The use case (Application): checks with injected dependencies | **Always** — that's what marks a rule as level 2 |

Litmus test: **if the rule needs a service, it's level 2.** (Format, ranges, cross-field coherence → the type. Uniqueness, referenced-thing-exists, state conflicts → Application.)

## The validator pattern

```csharp
internal sealed class PedidosMetaOccurredAtValidator : AbstractValidator<PedidosMetaOccurredAt>
{
    // The ONLY constructor: receives its composition — usable via DI when needed.
    public PedidosMetaOccurredAtValidator(IValidator<DateTimeUtcValueObject> validator)
    {
        Include(validator);

        RuleFor(v => v.Value)
            .LessThan(DateTime.UtcNow.AddMinutes(15));
    }

    // The self-contained assembly: chains the OTHER validators' Create() —
    // never `new`s another family's validator (its chain is its own knowledge).
    public static PedidosMetaOccurredAtValidator Create() => new PedidosMetaOccurredAtValidator(DateTimeUtcValueObjectValidator.Create());

    // Failures raise the FAMILY's exception, never a generic one.
    protected override void RaiseValidationException(
        ValidationContext<PedidosMetaOccurredAt> context,
        ValidationResult result)
        => throw new PedidosMetaOccurredAtValidationException(result.Errors);
}
```

- **Ctor receives ingredients; `Create` fabricates complete.** Base validators with no includes keep a parameterless ctor and a trivial `Create() => new XValidator();`.
- **Per-call, no caching**: types validate with `XValidator.Create().ValidateAndThrow(x)` — a fresh validator each time. Construction costs a few gen0 allocations (FluentValidation caches compiled accessors globally); a static instance is a later, profiler-driven optimization. It also keeps time bounds honest: a bound on "now" is evaluated when the validator is built, so a cached one would freeze it at process start.
- **Where this pattern applies — validators invoked by static factories** (value objects, aggregates): assembled with `Create()`, container-free, always. **Every other validator** — requests, query criteria — follows standard FluentValidation: constructor injection, normal DI registration (see *Boundary*).
- Lambda style: **`v =>`** (nested scopes `v2`), not `value =>`. Explicit types throughout: `new XValidator(...)`, never `new(...)`.
- No `new` modifier on a validator's `Create` (validators don't inherit each other; they `Include`).

## Writing the rules

- **No `Cascade(CascadeMode.Stop)`.** Every rule runs, so an empty value may report two failures (`NotEmpty` and the `Must`) — accepted. A validator is extended by others through `Include`; the base must not decide how a chain it is part of stops.
- **A base validator rejects only what is not the type** (not empty, a real email, a real IPv4); policy — a range, a format, a closed list — lives in the derived validator.
- **Bounds are literals in the rule**, never a public constant to clamp against, and they are pinned by two tests either side.
- **Messages**: a custom check names its rule and says what was expected:

  ```csharp
  RuleFor(v => v.Value)
      .Must(v => MailAddress.TryCreate(v, out MailAddress? address) && address.Address == v)
      .WithErrorCode("EmailValidator")
      .WithMessage("{PropertyName} must be an email");
  ```

  A closed list needs no code: `.Must(_types.Contains).WithMessage("{PropertyName} must be one of " + string.Join(", ", _types))`. Built-in rules (`NotEmpty`, `LessThan`…) keep FluentValidation's own messages.

## Composition — the aggregate's validator

The aggregate's validator composes its parts' rules and owns its **cross-field rules**: `SetValidator` for each value-object property, `Include` for a validator hierarchy (works via contravariance).

```csharp
public PedidosValidator(
    IValidator<PedidosReferencia> referencia,
    IValidator<PedidosMetaActive> metaActive)
{
    RuleFor(v => v.Referencia!)                 // an optional part: validated when present
        .SetValidator(referencia)
        .When(v => v.Referencia is not null);

    RuleFor(v => v.MetaActive)                  // a required part
        .SetValidator(metaActive);

    RuleFor(v => v.MetaDeletedAt)               // the aggregate's OWN rule (cross-field)
        .NotNull()
        .When(v => !v.MetaActive.Value);
}
```

- One `ValidateAndThrow` reports **every** failure with its property path (`Referencia.Value`, `MetaDeletedAt`) in one exception — the full report `ddd` asks for.
- The pattern recurses: a root with child entities composes their validators the same way.

## The ValidationException

Each validator raises its own typed `<X>ValidationException`, deriving the base's (`DateTimeValueObjectValidationException`, `ValidationException`…), with a **fixed GUID code** and the failure list (`ImmutableList<ValidationFailure>`). At an HTTP boundary that list becomes per-field ProblemDetails errors (`JorgeCostaMacia.Http.ProblemDetails` renders it). The rest of the exception family (`NotFound`, `Exist`) is `ddd-net`'s.

## Boundary — standard FluentValidation, when there is one

When a boundary validates its input (an HTTP request, a query criterion), its validator runs **first** in the handler (`validator.ValidateAndThrow(request)`), so the caller gets the complete per-field report in one response. It speaks the framework's idiom: constructor injection, normal DI registration (`AddScoped<IValidator<XRequest>, XRequestValidator>()`), the container composes the graph. Same shape as the rest: `<X>Request` + `<X>RequestValidator` + `<X>RequestValidationException`.

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
- Uniqueness gets the **double defense**: the application check gives the friendly domain error; the **unique index** gives the concurrency-proof guarantee (translate its violation to the same family exception).

## Testing this model

- Per rule, its **red case**: `Create_OnInvalidX_ThrowsFamilyValidationException`.
- A bound from both sides: the last accepted value and the first rejected one.
- One `ValidateAndThrow` reports ALL failing rules — assert the failure list, not just the throw.
- A validator is never cached in a test helper either: the time bounds would freeze with it.
