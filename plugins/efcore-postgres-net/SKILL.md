---
name: efcore-postgres-net
description: >-
  EF Core on PostgreSQL with Npgsql — the provider specifics on top of
  `efcore-net`. Use when mapping or querying a Postgres store from EF Core:
  optimistic concurrency, date and time columns, the legacy timestamp
  switch, precision, collation, schemas, indexes, or the Npgsql
  registration. It encodes: xmin as a shadow row version, the legacy
  timestamp switch with `timestamp without time zone` kept on purpose,
  microsecond precision, the database collation, one schema per layer, and
  indexes only on foreign keys.
---

# EF Core on PostgreSQL (Npgsql)

The Postgres specifics of `efcore-net`, through `Npgsql.EntityFrameworkCore.PostgreSQL`: `options.UseNpgsql(connectionString)`, the connection string read where the context is registered and required at startup.

## Optimistic concurrency — xmin

When writes to a row can race (concurrent consumers, see `messaging`), guard them with Postgres's own row version:

```csharp
entity
    .Property<uint>("Version")
    .IsRowVersion();
```

- **A shadow property over `xmin`**: every Postgres row has it, so there is no column to add, nothing in the aggregate and no migration.
- A save over a row written since it was read throws `DbUpdateConcurrencyException`. Let it propagate — the message is retried and the use case re-decides against the row as it now is. No retry loop in the service.
- It works because the operation reads and writes through one scoped context, so the tracker holds the original `xmin`.
- Only where writes can actually race. A table written by one sequential process doesn't need it.

## Dates and times

- **The legacy timestamp switch stays**: `AppContext.SetSwitch("Npgsql.EnableLegacyTimestampBehavior", true)` in `Program.cs`, before the host is built, and every date column mapped as **`HasColumnType("timestamp without time zone")`**. Moving to `timestamptz` means migrating live columns — the end state, deliberately deferred. Don't mix models: new date columns follow the same convention.
- Without the switch Npgsql refuses to write a `DateTime` of `Kind.Utc` into a `timestamp without time zone` column; with it, a read comes back `Unspecified` — which is why the columns this layer writes in UTC get the kind converter on the mapping (see `efcore-net`).
- **Postgres keeps microseconds**; .NET carries 100 ns. A date written and read back, or sent as a parameter, is truncated — so dates that are compared (a version guard, a sweep) are truncated to microseconds before they are stored, or a redelivered `…4567` looks newer than the stored `…4560`. The truncation lives in the value object's conversion (see `ddd-net`).
- Test fixtures set the same switch before the first connection, or the suite runs under different rules than production.

## Collation, schemas, names

- The model declares the database collation — `modelBuilder.UseCollation("es_ES.UTF8")` — so ordering and comparisons agree with the store's.
- **One schema per layer** (`raw`, `std`, `ods`…): `ToTable("pedidos", "raw")`. Tables and columns in snake_case, constraints named (`pk_pedidos`).
- The schema scripts are idempotent (`CREATE … IF NOT EXISTS`) and run in order: schema, tables, indexes — then `migrations/`, in name order (`ALTER TABLE … ADD COLUMN IF NOT EXISTS …`; see `efcore-net`).
- **String columns are `text`.** Postgres stores `text` and `varchar(n)` the same way; a bound in the type is a rule enforced by the store, with a `22001` instead of the typed `ValidationException`, and changing it is an `ALTER`. The rule lives in the value object's validator, and there is no `HasMaxLength` on a Postgres mapping — with an explicit `text` it does nothing.

## Indexes

**Only the foreign keys are indexed**, as plain ascending `(col)` indexes. Flag columns (`meta_active`) were measured and are useless — a value shared by most rows makes the planner scan anyway — so don't add them. Any other index needs a measured query plan that uses it.
