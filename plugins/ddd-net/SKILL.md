---
name: ddd-net
description: >-
  The .NET realization of tactical DDD (see the `ddd` skill for the
  principles). Use when writing or reviewing C# domain code: value objects
  and their factories, aggregates and their From/Create, the domain exception
  family, collecting events on the root, or giving an aggregate its DbContext. It
  encodes the JorgeCostaMacia.* base types, the four-factory surface
  (From, FromOrNull, Create, CreateOrNull) and why every one is declared,
  Convert as the one place a value is altered, no coercion and no defaults on
  a value object, the aggregate rule (From cascades, Create seals), and one
  DbContext per aggregate as its accessor.
---

# DDD in .NET

The .NET *how* of the model defined in `ddd` (load it for the *why*). Validators — how rules are written and composed — are `validation-net`'s; this skill only says where a type calls its own.

## The base types

The building blocks come from the `JorgeCostaMacia.*` packages; derive from them, don't redeclare them.

| Package | Gives |
| --- | --- |
| `JorgeCostaMacia.ValueObject` | `StringValueObject`, `IntValueObject`, `LongValueObject`, `DecimalValueObject`, `DateTimeValueObject`, `DateTimeUtcValueObject`, `BoolValueObject`, `UuidValueObject`… — each with its validator and validation exception |
| `JorgeCostaMacia.Aggregate` | `Aggregate`: the root's base, with `AddEvent` / `PullEvents` |
| `JorgeCostaMacia.Entity` / `.DomainEvent` | `IEntity`, `IDomainEvent` |
| `JorgeCostaMacia.Exception` | `NotFoundException`, `ExistException`, `ValidationException`, `ErrorException` — the bases of every family exception, each with a fixed code |

## Value objects

A value object is a `sealed record` deriving from its base, one folder per field — `Domain/<Field>/` holding `<Aggregate><Field>`, its `<Aggregate><Field>Validator` and its `<Aggregate><Field>ValidationException`. All of them flat at the root of `Domain/`: no `ValueObjects/` grouping.

```csharp
internal sealed record PedidosReferencia : LongValueObject
{
    // HYDRATE — raw assignment, no conversion, no validation: ORMs and deserializers only.
    public PedidosReferencia(long value) : base(value) { }

    // CONVERT — materializes through Convert, unvalidated: the path composites build their parts with.
    public static new PedidosReferencia From(long value) => new PedidosReferencia(Convert(value));
    public static new PedidosReferencia? FromOrNull(long? value) => value is null ? null : From(value.Value);

    // CREATE — nothing invalid escapes it.
    public static new PedidosReferencia Create(long value)
    {
        PedidosReferencia vo = From(value);
        vo.Validate();

        return vo;
    }

    public static new PedidosReferencia? CreateOrNull(long? value)
    {
        PedidosReferencia? vo = FromOrNull(value);
        vo?.Validate();

        return vo;
    }

    private void Validate() => PedidosReferenciaValidator.Create().ValidateAndThrow(this);
}
```

- **Declare all four factories, always** — whatever the field's nature. The bases declare all four as public statics, so an undeclared one is still reachable and answers wrongly: `XId.CreateOrNull(5L)` without its own declaration returns the *base* type, validated by the base validator — the field's own rule skipped in silence. `new` is mandatory on each, and the compiler proves it both ways: `CS0108` without it, `CS0109` where nothing is hidden.
- **Optionality lives in the `OrNull` sibling, never in the return type**: `From`/`Create` always return non-null. The `OrNull` ones branch on **what `From` returned** (`vo?.Validate()`), not on the input — correct even if a `From` ever maps a sentinel to absence. String VOs write `FromOrNull` as `value is null ? null : From(value)` (`string?` has no `.Value`).
- **One input type per VO**, the base's natural primitive. A second suffixed overload (`FromOrNull(string?)` and `FromOrNull(int?)`) makes `FromOrNull(null)` ambiguous (`CS0121`); other primitives are converted at the call site.
- **The validator is reached through one `private void Validate()`**, the local is `vo`.

### Convert — the one place a value is altered

The base's `protected static Convert` is the only place a value changes on its way in (`StringValueObject` trims). A derived VO that needs another rule **hides it** — `private static new` — and every factory picks it up:

```csharp
// fidelity: a landing layer keeps the source's text as it came
private static new string Convert(string value) => value;

// the store keeps microseconds: one value in memory, on the wire and in the table
private static new DateTime Convert(DateTime value)
    => new DateTime(value.Ticks - (value.Ticks % TimeSpan.TicksPerMicrosecond), value.Kind);
```

- Hiding `Convert` is how a layer states its policy; keep it uniform across the layer (every string VO of a landing layer passes through, not one of them).
- Calls *inside* the base still bind to the base's overloads — hiding only affects the derived type's factories.
- A conversion **never alters a value silently** beyond its rule: no comma dropped as a thousands separator, no fraction truncated into an integer. When the input is not the type, it throws.

### No coercion, no defaults

- **A VO never pulls a value into range.** Out of range is a caller's mistake and its `Create` throws the typed exception.
- **No `Default` and no public `Max`.** A default is call-site policy; a public bound is an invitation to clamp against it. The bound lives as a literal in the validator, pinned by two tests either side (`Create(1000)` accepted, `Create(1001)` rejected) so it cannot drift.
- **A base rejects only what is not the type.** Policy (a range, a format, a closed list) lives in the derived VO's validator.
- A bound on "now" is captured when the validator is constructed — per `Create` today. Cache a validator in a static and the clock freezes at process start.

### A composite value object

A VO made of other VOs (a page range of two page numbers) is a plain `IValueObject` record holding its parts, not a derived base. Its validator runs each part's own validator plus the cross rule (`start <= end`), which inheriting could not do.

## Aggregates

```csharp
internal sealed class Pedidos : Aggregate
{
    public long Id { get; init; }                          // the identity (see ddd: it may stay a primitive)
    public PedidosReferencia? Referencia { get; set; }     // optional field, nullable VO
    public PedidosMetaActive MetaActive { get; set; } = null!;

    private Pedidos() { }                                   // ORM materialization

    public Pedidos(long id, PedidosReferencia? referencia, PedidosMetaActive metaActive)   // hydration
    {
        Id = id;
        Referencia = referencia;
        MetaActive = metaActive;
    }

    // FROM cascades: every part through ITS From — nothing validated, nothing thrown yet.
    public static Pedidos From(long id, long? referencia, bool metaActive)
        => new Pedidos(id, PedidosReferencia.FromOrNull(referencia), PedidosMetaActive.From(metaActive));

    // CREATE seals: one composed validation, one exception with every failure.
    public static Pedidos Create(long id, long? referencia, bool metaActive)
    {
        Pedidos aggregate = From(id, referencia, metaActive);
        aggregate.Validate();

        return aggregate;
    }

    private void Validate() => PedidosValidator.Create().ValidateAndThrow(this);
}
```

- The root is an `internal sealed class` deriving `Aggregate` (identity, not value semantics — so a class, not a record). Fields that change are `set`; the identity and what never changes after creation are `init`.
- **The aggregate never calls its parts' `Create`** — that would throw on the first invalid part alone. `From` cascades through the parts' `From`/`FromOrNull`; `Create` validates the whole once (`validation-net` covers how the aggregate's validator composes its parts').
- **The nullability of each factory call matches its property**: `XField.FromOrNull(...)` for an optional field, `XField.From(...)` for a required one. Add a field, keep both sides in step.
- Long parameter lists go one per line (the examples above are compressed).

## The exception family

Each concept ships its typed exceptions, one file each, deriving the `JorgeCostaMacia.Exception` bases: `<X>ValidationException` (raised by its validator), `<X>NotFoundException`, `<X>ExistException`.

- Each carries a **fixed GUID code**, a literal in its convenience constructor, so occurrences trace across logs and responses whatever the wording.
- Two constructors: the full tuple (transport) and the convenience one producers use.

## Events

The root collects what happened as `IDomainEvent`s through `AddEvent(...)`, and whoever persists it takes them with `PullEvents()` **after** a successful save — so no event escapes a failed operation. How they are dispatched or published belongs to the messaging side, not to this skill.

## The aggregate's accessor

**One `DbContext` per aggregate**, internal to its module, its `DbSet` named `<X>Repository`: the per-aggregate accessor `ddd` asks for. Application uses it as-is — an accepted core coupling, with no repository port around it. How the model maps to the tables (value objects, keys, converters) is persistence, not this skill's.

## Testing this model

- Composition tests use `From` (it skips the validators and keeps the test free of unrelated field rules); rule tests use `Create`.
- Every VO with a rule of its own: its red case on the typed exception, and the bound from both sides.
- `From`/`FromOrNull` both directions on the aggregate: every optional part materialized when supplied, every one null when not.
- A `Convert` that does something (truncation, pass-through, a kind) has its own test: without it, moving the VO to another base passes everything else.
