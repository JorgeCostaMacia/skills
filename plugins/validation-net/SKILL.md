---
name: validation-net
description: >-
  The .NET realization of the two-level validation model (see the `ddd` skill
  for the principles). Use when working with validation in Jorge's .NET code:
  writing or reviewing FluentValidation validators, value objects and their
  Create/From factories, aggregate factories, request validators, deciding
  where a rule lives (type invariant vs use-case precondition), wiring
  validators into DI, mapping validation failures to ProblemDetails, or
  implementing uniqueness/existence checks. It encodes: the three-verb
  creation surface (ctor hydrates, From converts, Create validates), the
  validator pattern (injectable constructor + static Create() chain), the
  composite rule (never call the parts' Create — build with From, validate
  composed once), the family exceptions with fixed codes, and level-2
  preconditions in Application with DI.
---

# Validation in .NET (FluentValidation)

The .NET *how* of the two-level model defined in `ddd` (load it for the *why*). **FluentValidation is the accepted core coupling** for validation — deliberate, evaluated, not to be wrapped or replaced.

## The two levels, realized

| Level | Question | Lives in | DI |
| --- | --- | --- | --- |
| **1 — Invariants** | "Is this object correct in itself?" | The type: its validator, invoked by its `Create` factory | **Never** — type rules are pure (no services) |
| **2 — Preconditions** | "Is this correct object admissible *here, now*?" | The use case (Application): checks with injected dependencies | **Always** — that's what marks a rule as level 2 |

Litmus test: **if the rule needs a service, it's level 2.** (Uniqueness, referenced-thing-exists, business-state conflicts → Application. Format, ranges, cross-field coherence → the type.)

## The three-verb creation surface (value objects)

```csharp
public record EmailValueObject : StringValueObject
{
    private static readonly EmailValueObjectValidator Validator = EmailValueObjectValidator.Create();

    // 1. HYDRATE — raw assignment, zero work. Reserved for ORMs/deserializers
    //    (the EfConverter uses it; rehydrating N records must not re-convert).
    public EmailValueObject(string value) : base(value) { }

    // 2. CONVERT — normalizes/converts and materializes, UNVALIDATED.
    //    The composites' path. Conversion vocabulary: promises materialization, not validity.
    public new static EmailValueObject From(string value) => new(StringValueObject.From(value).Value);

    // 3. CREATE — the guarantee: nothing invalid escapes Create.
    public new static EmailValueObject Create(string value)
    {
        EmailValueObject vo = From(value);
        Validator.ValidateAndThrow(vo);

        return vo;
    }
}
```

Mental convention: **ctor hydrates · `From` converts · `Create` fabricates validated.** All conversion logic (trim, string→number via decimal, base64…) lives in `From` — it replaces the old `Convert`, returning the VO instead of the primitive.

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

- **Ctor receives ingredients; `Create` fabricates complete.** Base validators with no includes keep a parameterless ctor and a trivial `Create() => new();` — one uniform convention.
- Lambda style: **`v =>`** (nested scopes `v2`), not `value =>`.
- No `new` modifier on validator `Create` (validators don't inherit each other); VOs' `Create`/`From` DO hide the base's.
- DI registration stays trivial where the pipeline resolves validators: `AddScoped<IValidator<X>, XValidator>()`.

## The composite rule — full report, no drip

**An aggregate never calls its parts' `Create`.** The three-verb surface applies to aggregates too — **`From` cascades, `Create` seals**:

```csharp
// FROM cascades: materializes the whole aggregate unvalidated —
// parts via THEIR From (normalized, no VO throws individually).
public static Factura From(int id, string? email, ...) =>
    new(id, EmailValueObject.From(email), ...);

// CREATE seals: own From + ONE composed pass — the aggregate's validator
// (which Includes its VOs' validators) throws ONE exception carrying
// the COMPLETE per-field failure list.
public static Factura Create(int id, string? email, ...)
{
    Factura aggregate = From(id, email, ...);
    Validator.ValidateAndThrow(aggregate);

    return aggregate;
}
```

The pattern recurses: an aggregate root with child entities builds them through *their* `From` inside its own, and its `Create` validates the whole tree composed, once. `Create`'s body is identical at every level — uniformity is the point.

- Calling parts' `Create` inside an aggregate factory reintroduces the drip (first invalid VO throws alone) — the exact problem the composed validator solves.
- The invalid aggregate exists only *inside* its factory — nothing invalid escapes it (`ddd`: the guarantee is at the factory boundary, not the micro-order inside).
- `ValidateAndThrow` runs ALL the validator's rules and reports every failure in one exception — the full report is native.

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

## Boundary composition (requests)

Request validators compose the same rule sources and run **before anything else** in the handler (`validator.ValidateAndThrow(request)` as the first line) — the user gets the complete per-field report in one response. They're DI-registered for the pipeline (`[FromServices] IValidator<XRequest>`), constructed via ctor or `Create()` — either works, since level-1 chains are self-contained.

## Level 2 — preconditions in Application

```csharp
// In the use-case service — dependencies injected, rejects with the family's domain error:
Factura? existing = await Context.FacturaRepository.FindAsync(aggregate.Id);
if (existing != null) throw new FacturaExistException(...);
```

- Uniqueness gets the **double defense**: the application check gives the friendly domain error; the **unique index** gives the concurrency-proof guarantee (translate its violation to the same family exception).
- Level-2 rules never sneak into type validators — a validator constructor asking for a repository is the red flag.

## Testing this model

- Per validation rule, its **red case**: `Create_OnInvalidX_ThrowsFamilyValidationException`.
- `From` returns normalized-but-unvalidated instances (no throw on invalid input) — test that too.
- One `ValidateAndThrow` reports ALL failing rules — assert the failure list, not just the throw.
- The hydration ctor never validates — the EfConverter tests assert `Create`-that-throws is never called on rehydration; keep that guarantee green.
