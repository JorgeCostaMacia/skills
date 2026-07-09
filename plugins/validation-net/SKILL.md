---
name: validation-net
description: >-
  The .NET realization of the two-level validation model (see the `ddd` skill
  for the principles). Use when working with validation in Jorge's .NET code:
  writing or reviewing FluentValidation validators, value objects and their
  From/Create factories, aggregate factories, request validators, deciding
  where a rule lives (type invariant vs use-case precondition), wiring
  validators into DI, mapping validation failures to ProblemDetails, or
  implementing uniqueness/existence checks. It encodes: the three-verb
  creation surface (ctor hydrates, From converts, Create validates),
  per-call validators assembled with static Create() chains, the
  factory-vs-DI rule, the aggregate composition rule (From cascades, Create
  seals — never the parts' Create), the family exceptions with fixed codes,
  and level-2 preconditions in Application with DI.
---

# Validation in .NET (FluentValidation)

The .NET *how* of the two-level model defined in `ddd` (load it for the *why*). **FluentValidation is the accepted core coupling** for validation — deliberate, evaluated, not to be wrapped or replaced.

## The two levels, realized

| Level | Question | Lives in | DI |
| --- | --- | --- | --- |
| **1 — Invariants** | "Is this object correct in itself?" | The type: its validator, invoked by its `Create` factory | **Never** — type rules are pure (no services) |
| **2 — Preconditions** | "Is this correct object admissible *here, now*?" | The use case (Application): checks with injected dependencies | **Always** — that's what marks a rule as level 2 |

Litmus test: **if the rule needs a service, it's level 2.** (Format, ranges, cross-field coherence → the type. Uniqueness, referenced-thing-exists, state conflicts → Application.)

## The three-verb creation surface (value objects)

Lean by design: **one `From` and one `Create` per class**, on the type's natural primitive. Other input types are converted at the call site (`From(value.ToString(CultureInfo.InvariantCulture))`) — an extra overload is added only when real usage proves it (additive, non-breaking; e.g. an int VO's parse-from-string twin if ETL consumes it).

```csharp
public record EmailValueObject : StringValueObject
{
    // 1. HYDRATE — raw assignment, zero work. Reserved for ORMs/deserializers
    //    (the EfConverter uses it; rehydrating N records must not re-convert or validate).
    public EmailValueObject(string value) : base(value) { }

    // 2. CONVERT — normalizes and materializes, UNVALIDATED. The composites' path.
    //    Conversion vocabulary: promises materialization, not validity.
    public new static EmailValueObject From(string value) => new(value.Trim());

    // 3. CREATE — the guarantee: nothing invalid escapes Create.
    public new static EmailValueObject Create(string value)
    {
        EmailValueObject vo = From(value);
        EmailValueObjectValidator.Create().ValidateAndThrow(vo);

        return vo;
    }
}
```

Mental convention: **ctor hydrates · `From` converts · `Create` fabricates validated.** Derived VOs redefine `From`/`Create` with the `new` modifier — only the overloads they actually need.

## The validator pattern

```csharp
public class OrderTypeValueObjectValidator : AbstractValidator<OrderTypeValueObject>
{
    // The ONLY constructor: receives its composition — usable via DI when needed.
    public OrderTypeValueObjectValidator(IValidator<StringValueObject> validator)
    {
        Include(validator);

        RuleFor(v => v.Value)
            .NotEmpty()
            .Must(v2 => v2 == "ASC" || v2 == "DESC")
            .WithMessage("{PropertyName} must be 'ASC' or 'DESC'");
    }

    // The self-contained assembly: chains the OTHER validators' Create() —
    // never `new`s another family's validator (its chain is its own knowledge).
    public static OrderTypeValueObjectValidator Create() => new(StringValueObjectValidator.Create());

    // Failures raise the FAMILY's exception, never a generic one.
    protected override void RaiseValidationException(ValidationContext<OrderTypeValueObject> context, ValidationResult result)
        => throw new OrderTypeValueObjectValidationException(result.Errors);
}
```

- **Ctor receives ingredients; `Create` fabricates complete.** Base validators with no includes keep a parameterless ctor and a trivial `Create() => new();`.
- **Per-call, no caching**: factories validate with `XValidator.Create().ValidateAndThrow(x)` — a fresh, short-lived validator each time. Construction costs a few gen0 allocations (FluentValidation caches compiled accessors globally); caching a static instance is a later, internal, profiler-driven optimization — don't pre-pay it. Bonus: constant time arguments in rules (`LessThanOrEqualTo(DateTime.UtcNow…)`) are evaluated per call, so there's no frozen-clock footgun; lambdas for "now" remain good practice.
- **Where this pattern applies — validators invoked by static factories** (VOs, aggregates): assembled with `Create()`, container-free, always. **Every other validator** — requests, query/criteria checks, pre-select entity validations — **follows standard FluentValidation**: constructor injection, normal DI registration, the container composes (see *Boundary* below).
- Lambda style: **`v =>`** (nested scopes `v2`), not `value =>`.
- No `new` modifier on validator `Create` (validators don't inherit each other); VOs' `From`/`Create` DO hide the base's.

## The aggregate rule — From cascades, Create seals

**An aggregate never calls its parts' `Create`.** Its `From` materializes the parts through *their* `From`; its `Create` validates the whole aggregate **composed, once**:

```csharp
// FROM cascades: the whole aggregate materialized unvalidated — no VO throws individually.
public static Factura From(int id, string? email, ...) =>
    new(id, EmailValueObject.From(email), ...);

// CREATE seals: ONE composed pass — one exception, the COMPLETE per-field failure list.
public static Factura Create(int id, string? email, ...)
{
    Factura aggregate = From(id, email, ...);
    FacturaValidator.Create().ValidateAndThrow(aggregate);

    return aggregate;
}
```

The aggregate's validator owns its **cross-field rules** and composes its parts' rules — `SetValidator` for VO properties, `Include` for validator hierarchies (works via contravariance):

```csharp
public FacturaValidator(IValidator<EmailValueObject> emailValidator)
{
    RuleFor(f => f.Email).SetValidator(emailValidator);   // the VO's rules, owned by the VO's family

    RuleFor(f => f.MetaDeletedAt)                          // the aggregate's OWN rules (cross-field)
        .NotEmpty().When(f2 => !f2.MetaActive, ApplyConditionTo.CurrentValidator)
        .Null().When(f2 => f2.MetaActive, ApplyConditionTo.CurrentValidator);
}

public static FacturaValidator Create() => new(EmailValueObjectValidator.Create());
```

- Calling parts' `Create` inside an aggregate factory reintroduces the drip (first invalid VO throws alone) — the exact problem composition solves.
- The invalid aggregate exists only *inside* its factory — nothing invalid escapes it.
- One `ValidateAndThrow` reports **every** failure with property paths (`Email.Value`, `MetaDeletedAt`) in one exception.
- The pattern recurses: a root with child entities builds them through *their* `From`; `Create`'s body is identical at every level.

## The family pattern (errors with fixed codes)

A domain concept ships with its complete family — same prefix, one file each:

| Member | Role |
| --- | --- |
| `<X>` | the type |
| `<X>Validator` | ALL its level-1 rules |
| `<X>ValidationException` | raised by the validator on rule violations |
| `<X>NotFoundException` | requested but doesn't exist (level 2) |
| `<X>ExistException` | created but already exists (level 2) |

- Each exception carries a **fixed unique code (GUID constant)** — occurrences trace across logs and API responses regardless of message wording.
- The `ValidationException` carries the failure list (`ImmutableList<ValidationFailure>`) — which flows to per-field ProblemDetails errors at the HTTP boundary (the `JorgeCostaMacia.Http.ProblemDetails` package renders it).
- Requests get the same shape: `<X>Request` + `<X>RequestValidator` + `<X>RequestValidationException`.

## Boundary (requests) — standard FluentValidation

Request validators run **before anything else** in the handler (`validator.ValidateAndThrow(request)` as the first line) — the user gets the complete per-field report in one response.

At the boundary, **standard FluentValidation applies** — no `Create()`, no ceremony of ours: constructor injection, normal DI registration (`AddScoped<IValidator<XRequest>, XRequestValidator>()`), the container resolves the composition graph (`[FromServices] IValidator<XRequest>`). The strict Create discipline exists to serve static factories; the boundary lives in the framework's world and speaks its idiom.

## Level 2 — preconditions in Application

```csharp
// In the use-case service — dependencies injected, rejects with the family's domain error:
Factura? existing = await Context.FacturaRepository.FindAsync(aggregate.Id);
if (existing != null) throw new FacturaExistException(...);
```

- Level-2 checks stay **in the use case** — they usually come glued to the load/insert they guard (extracting them to a "validator with a repository inside" doubles the query and blurs the vocabulary: a validator constructor asking for a repository is the red flag). If the same check repeats across many use cases, extract a helper/domain service — not a FluentValidation validator.
- Uniqueness gets the **double defense**: the application check gives the friendly domain error; the **unique index** gives the concurrency-proof guarantee (translate its violation to the same family exception).

## Testing this model

- Per validation rule, its **red case**: `Create_OnInvalidX_ThrowsFamilyValidationException`.
- `From` returns normalized-but-unvalidated instances (no throw on invalid input) — test that too.
- One `ValidateAndThrow` reports ALL failing rules — assert the failure list, not just the throw.
- The hydration ctor never validates — the EfConverter tests assert the converter never triggers `Create`-validation on rehydration; keep that guarantee green.
