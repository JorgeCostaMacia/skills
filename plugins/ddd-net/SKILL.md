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

The .NET *how* of the model defined in `ddd` (load it for the *why*). How a validator is written — rules, messages, composition — is `validation-net`'s; this skill says which validator each domain type has and how it runs it.

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

- **Declare all four factories, always** — whatever the field's nature. The bases declare all four as public statics, so an undeclared one is still reachable and answers wrongly: `XId.CreateOrNull(5L)` without its own declaration returns the *base* type, validated by the base validator — the field's own rule skipped in silence. `new` is mandatory on each, and the compiler proves the *modifier* both ways: `CS0108` without it, `CS0109` where nothing is hidden. It cannot prove a factory is *declared*: an undeclared one resolves to the base's at any call site that doesn't demand the derived type — `XEstado.Create(x).Value` in a predicate compiles and skips the rule. `From`/`FromOrNull` are pinned by the aggregate, whose typed properties refuse the base (`CS0266`); `Create`/`CreateOrNull` on a payload VO have no production caller, so a test pins them — see *Testing this model*.
- **Optionality lives in the `OrNull` sibling, never in the return type**: `From`/`Create` always return non-null. `FromOrNull` is the one place that looks at the input (`value is null ? null : From(value.Value)`; a string VO writes `From(value)`, since `string?` has no `.Value`). `CreateOrNull` branches on **what `FromOrNull` returned** (`vo?.Validate()`), never on the input again — so it stays correct if a `From` ever maps a sentinel to absence, and never hands the validator a null.
- **One input type per VO**, the base's natural primitive. A second suffixed overload (`FromOrNull(string?)` and `FromOrNull(int?)`) makes `FromOrNull(null)` ambiguous (`CS0121`); other primitives are converted at the call site.
- **The validator is reached through one `private void Validate()`**, the local is `vo`.

### Convert — the one place a value is altered

The base's `protected static Convert` is the only place a value changes on its way in, and what it does is convert the input to **the base's representation**: `StringValueObject` trims, `DateTimeUtcValueObject` yields an instant in UTC. A derived VO whose type represents something else **hides it** — `private static new` — and every factory picks it up:

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
- **A base rejects only what is not the type** — and most bases check nothing (`validation-net` lists what each one does). Policy (not empty, a range, a format, a closed list) lives in the derived VO's validator.
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

    public Pedidos(                                        // hydration
        long id,
        PedidosReferencia? referencia,
        PedidosMetaActive metaActive)
    {
        Id = id;
        Referencia = referencia;
        MetaActive = metaActive;
    }

    // FROM cascades: every part through ITS From — nothing validated, nothing thrown yet.
    public static Pedidos From(
        long id,
        long? referencia,
        bool metaActive)
        => new Pedidos(
            id,
            PedidosReferencia.FromOrNull(referencia),
            PedidosMetaActive.From(metaActive));

    // CREATE seals: one composed validation, one exception with every failure.
    public static Pedidos Create(
        long id,
        long? referencia,
        bool metaActive)
    {
        Pedidos aggregate = From(
            id,
            referencia,
            metaActive);
        aggregate.Validate();

        return aggregate;
    }

    private void Validate() => PedidosValidator.Create().ValidateAndThrow(this);
}
```

- The root is an `internal sealed class` deriving `Aggregate` (identity, not value semantics — so a class, not a record). Fields that change are `set`; the identity and what never changes after creation are `init`.
- **The aggregate never calls its parts' `Create`** — that would throw on the first invalid part alone. `From` cascades through the parts' `From`/`FromOrNull`; `Create` validates the whole once (see *Validating the model*).
- **The nullability of each factory call matches its property**: `XField.FromOrNull(...)` for an optional field, `XField.From(...)` for a required one. Add a field, keep both sides in step.

## Validating the model

Every value object and every aggregate has its own `<X>Validator` and `<X>ValidationException`, next to it in its folder.

- **The validator is built with `Create()`, never resolved from DI.** A type validates inside its own static `Create`, where there is no container, and its rules are pure — there is nothing to inject. `XValidator.Create()` assembles the whole chain, and the type reaches it through one `private void Validate() => XValidator.Create().ValidateAndThrow(this);`.
- **Only `Create` validates.** `From` and the hydration constructor never do: a composite is built from unvalidated parts, and rehydrating N rows must not re-validate them.
- **A value object's validator `Include`s its base's** (`Include(DateTimeUtcValueObjectValidator.Create())` through the constructor) and adds the field's own rule, so the base's type checks always run. Its exception derives the base's (`DateTimeValueObjectValidationException`…).
- **The aggregate's validator composes its parts**: one `SetValidator` per value-object property — an optional one guarded with `.When(v => v.Part is not null)` and selected as `v.Part!` — plus the aggregate's own cross-field rules. One `ValidateAndThrow` in `Create` reports every failure, with its property path, in one `<X>ValidationException`.

  ```csharp
  RuleFor(v => v.Referencia!)
      .SetValidator(referencia)
      .When(v => v.Referencia is not null);

  RuleFor(v => v.MetaDeletedAt)                // the aggregate's own rule
      .NotNull()
      .When(v => !v.MetaActive.Value);
  ```

- A composite value object (two page numbers making a range) validates each part with the part's own validator plus the cross rule (`start <= end`).
- **Which layer holds a content rule** follows `ddd`: a landing layer's payload VO has a validator that only `Include`s its base's, and the field's rule — a length, a format — lives in the standardizing layer's VO; the landing layer's own rules are on its identity and `Meta*` fields.

## The exception family

Each concept ships its typed exceptions, one file each, deriving the `JorgeCostaMacia.Exception` bases: `<X>ValidationException` (raised by its validator), `<X>NotFoundException`, `<X>ExistException`.

- Each carries a **fixed GUID code**, a literal in its convenience constructor, so occurrences trace across logs and responses whatever the wording. The code is a **UUIDv7** (`Guid.CreateVersion7()`), minted fresh for each exception type by whoever writes the file — never copied from a neighbour: a generator that stamps one GUID into every file it writes leaves seventy exceptions sharing a code that then identifies none of them. Nothing in the language checks it, so the host suite does: one convention test asserts every exception's code is distinct across the solution (see `testing-net`).
- Two constructors: the full tuple (transport) and the convenience one producers use.

## Events

The root collects what happened as `IDomainEvent`s through `AddEvent(...)` — in this family the contract's own event, since the bus's `Event` is an `IDomainEvent` (see `ddd`: the published events *are* the domain events) — and whoever persists it takes them with `PullEvents()` **after** a successful save, so no event escapes a failed operation. How they reach the bus is `bus-net`'s.

## The aggregate's accessor

**One `DbContext` per aggregate**, internal to its module, its `DbSet` named `<X>Repository`: the per-aggregate accessor `ddd` asks for. Application uses it as-is — an accepted core coupling, with no repository port around it. How the model maps to the tables (value objects, keys, converters) is persistence, not this skill's.

## Testing this model

- Composition tests use `From` (it skips the validators and keeps the test free of unrelated field rules); rule tests use `Create`.
- Every VO with a rule of its own: its red case on the typed exception, and the bound from both sides.
- `From`/`FromOrNull` both directions on the aggregate: every optional part materialized when supplied, every one null when not.
- A `Convert` of its own (a truncation, a kind) has its own test: without it, moving the VO to another base passes everything else.
- **A VO with no rule and no `Convert` of its own has no test file of its own.** What it has to pin is the same for every VO of its kind — the four factories declared on the type and returning it, and the layer's `Convert` policy (a landing layer's strings pass through, a standardizing layer's are trimmed) — so one **convention test per module** enumerates the module's value objects and pins it for all of them (`testing-net` has the realization). The day it gains a rule or a `Convert`, it gains its file.
