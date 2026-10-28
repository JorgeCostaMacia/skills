---
name: efcore-net
description: >-
  EF Core conventions for any provider — how the domain model maps to a
  relational store and how it is queried and saved. Use when writing or
  reviewing a DbContext mapping, mapping value objects, keys or dates,
  writing LINQ predicates over value objects, filtering by date, passing
  cancellation tokens to queries and saves, or testing a mapping against the
  real schema. It encodes: value objects as ComplexProperty over .Value,
  keys as primitives or value-converted VOs, explicit column names, what a
  column's DateTimeKind means stated in the mapping, `!= null` inside
  expression trees, sargable half-open date ranges, the token on every call
  (and the FindAsync trap), writes through the tracked entity, a fresh context per step, and the schema built
  from the real scripts. Provider specifics live in the provider's skill.
---

# EF Core in .NET

How the model meets the store, for any provider. The per-aggregate `DbContext` as the aggregate's accessor is `ddd-net`'s; the provider's types, concurrency token and quirks are its own skill's (`efcore-postgres-net`). For query **performance** — N+1, tracking, keyset paging, plans — use `optimizing-ef-core-queries` from the dotnet/skills marketplace; two of its remedies yield to the rules here: no `ExecuteUpdate` / `ExecuteDelete` in production code, and indexes only where the provider skill says (foreign keys).

## Mapping

- **The context's shape is the same everywhere**: `internal class` (not sealed), `virtual` `DbSet`s, and both a parameterless constructor and the `DbContextOptions<T>` one — the shape EF's design-time tooling expects, kept uniform across the family.
- **One `DbContext` per aggregate**, configured in `OnModelCreating` with one `Entity<X>(entity => …)` block: the table (`ToTable("x", "<schema>")`), the key with a named constraint (`HasKey(e => e.Id).HasName("pk_x")`), then every property — **every column named explicitly** (`HasColumnName("snake_case")`), so a renamed property never renames a column.
- **Value objects map as `ComplexProperty` over their `.Value`**, `IsRequired(false)` on the optional ones (an optional complex type needs EF Core 10 or later):

  ```csharp
  entity
      .ComplexProperty(e => e.Referencia, c => c
          .IsRequired(false)
          .Property(v => v.Value)
          .ValueGeneratedNever()
          .HasColumnName("referencia"));
  ```

  A complex property has value semantics — the instance is never tracked, so one VO can be assigned to many entities. That breaks if the mapping becomes an owned type, which has identity.
- **`ValueGeneratedNever()`** on values the application assigns — the key and every column — so EF never expects the store to generate them.
- **Keys.** A complex property cannot be a key. The identity stays a primitive, or it is a VO mapped with a value converter (`HasConversion`), in which case queries compare the whole VO (`e.Id == id`) and `FindAsync` takes the VO.
- **What a column means belongs to the mapping**, not to the value object: a converter such as `HasConversion(v => v, v => DateTime.SpecifyKind(v, DateTimeKind.Utc))` tags the dates this layer itself wrote in UTC. Never put it on a column whose zone this layer doesn't know — a source's dates keep the kind they have.
- A new column goes into the model **and** the schema scripts together — and, for a table that already exists somewhere, into a migration (see *Against the real schema*).
- **The model declares no index.** `HasIndex` would only matter to `EnsureCreated`, which is never used; indexes live in the schema scripts, and which ones exist is the provider skill's rule.
- **A column type describes the store; a rule lives in the domain.** The database is a store: a length, a range or a format is the value object's validator's (`ddd`), never a `HasMaxLength` or a sized column type, which would enforce the rule in the wrong layer with the wrong error. A read-only legacy source keeps the sized types it has — the mapping describes what is there. The provider skill names the house types.

## Querying

- **Predicates over value objects filter on `.Value`**: `e => e.MetaActive.Value`.
- Inside an expression tree a null check is **`!= null`**, not `is not null` — a pattern there is `CS8122`.
- **Filter a date with a half-open range**, never with `.Date` on the column:

  ```csharp
  e.MetaUpdatedAt != null
      && e.MetaUpdatedAt.Value >= day
      && e.MetaUpdatedAt.Value < day.AddDays(1)
  ```

  A function on the column is computed per row and can't use an index; the range can.
- **Paging** skips with the page size's own arithmetic (`size.Offset(page)`), over a stable order.
- **Writes go through the tracked entity** — load, change or `Remove`, `SaveChangesAsync`. `ExecuteUpdateAsync` / `ExecuteDeleteAsync` bypass the change tracker, the concurrency check and the aggregate's events, so production code doesn't use them; test fixtures do, to empty a table between tests (see `testing-net`). No raw SQL for what LINQ can say.

## Cancellation

- Every call that touches the store takes the token: `CountAsync(…, cancellationToken)`, `ToListAsync(cancellationToken)`, `AddAsync(x, cancellationToken)`, `SaveChangesAsync(cancellationToken)`. Public entry points take `CancellationToken cancellationToken = default`; **private helpers take it without a default**, so a forgotten argument does not compile.
- **The `FindAsync` trap**: its overloads are `FindAsync(params object?[]? keyValues)` and `FindAsync(object?[]? keyValues, CancellationToken)`. `FindAsync(id, cancellationToken)` binds to the first and sends the token **as a second key value**. Write `FindAsync(new object?[] { id }, cancellationToken)` — or call it without a token where a single by-id lookup leaves nothing to cancel.
- Cancelling a save is safe: the save runs in a transaction and rolls back whole, and a scoped context is disposed with its half-applied tracker.

## Contexts and the tracker

- The context is **scoped to one operation** (one request, one message) — never shared, never a singleton.
- An operation reads and writes through **one** context, so the change tracker keeps the original values a concurrency check compares against.
- Tests and tools take **a fresh context per step** (seed, act, assert), so the tracker never answers a question the store should.

## Against the real schema

The schema is the repository's own scripts, the ones production runs — never `EnsureCreated`, which builds whatever the model says and so agrees with any mapping. Integration tests build their database from those scripts (see `testing-net`); a column the mapping names but the table lacks fails there, not in production.

The scripts are the **state** — what a new environment gets — and `CREATE … IF NOT EXISTS` never alters a table that exists, so a change to a live table is a **migration**: an idempotent `ALTER` in `database/migrations/<version>.sql`, named after the release that needs it and listed in its notes, run by hand on every environment **before** that version deploys (nothing runs it for you; the service's user has no DDL). Being idempotent, the folder can always be run whole, in **version** order — not name order, which puts `6.10.0.sql` before `6.9.0.sql` (the fixture sorts by `Version.Parse` of the file name). The test fixture runs it after the state scripts, so a wrong migration fails the suite; a missing one cannot fail there — the tables are born complete — so a change to `tables.sql` is reviewed with its migration beside it. EF migrations are not used: one history per context, times a context per aggregate, for an event that is rare and would end as a hand-run script anyway; `dotnet ef migrations script --idempotent` may *generate* the file.
