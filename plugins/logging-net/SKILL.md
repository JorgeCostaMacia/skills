---
name: logging-net
description: >-
  Application logging style for Jorge's .NET code — Serilog is the accepted
  coupling. Use when writing or reviewing ANY log statement (ILogger calls,
  LogInformation/Warning/Error), choosing a log message or level, adding
  logging to handlers, services, consumers, jobs or listeners, pushing
  context/correlation into logs, or configuring structured logging. It
  encodes: fixed low-cardinality messages that act as grouping keys (never
  interpolation, never placeholders, never failure details in the message),
  all variable data via LogContext.PushProperty, same message across levels
  for outcome pairs, and correlation ids in every scope.
---

# Logging in .NET (Serilog)

**Serilog is the accepted core coupling** for logging — not wrapped, not replaced. Logs are consumed as **structured data** (Loki/Grafana): the design goal is *grouping and querying*, not prose.

## The rule: the message is a grouping key

The message is a **fixed, low-cardinality constant** — an event name or a semi-generic outcome:

```csharp
Logger.LogInformation("UserLoad");
Logger.LogError(exception, "UserLoad fail");
Logger.LogError(exception, "Handler failed.");
```

- **Never interpolate**: `$"User {id} loaded in {ms}ms"` destroys grouping — every occurrence becomes a distinct message; dashboards and alerts on counts die.
- **No `{Placeholder}` templates in the message either** — stricter than default Serilog guidance: placeholders render the values into the message string, reintroducing cardinality. The message groups; the data travels separately.
- **The message never exposes the failure itself** — no exception text, no offending value. The detail lives in the exception argument and the context properties; the message stays opaque and stable.

## All variable data via context

`LogContext.PushProperty` in stacked `using` blocks — every property becomes queryable/filterable structured data:

```csharp
using (LogContext.PushProperty("AggregateId", aggregate.Id))
using (LogContext.PushProperty("CorrelationId", correlationId))
using (LogContext.PushProperty("UserId", request.UserId))
using (LogContext.PushProperty("Request", request, destructureObjects: true))
using (LogContext.PushProperty("OccurredAt", request.MetaOccurredAt))
{
    Logger.LogInformation("UserLoad");
}
```

- `destructureObjects: true` for complex payloads — structured, not `ToString()`.
- Query by property (`UserId="42"` in Loki/Grafana), never by message text.
- Gotcha: alongside MassTransit, alias the import — `using LogContext = Serilog.Context.LogContext;` (MassTransit ships its own `LogContext`).

## Levels split the series; the message stays

Outcome pairs share the SAME message and differ by **level** — in Grafana the series splits by level while keeping the grouping key:

- `LogInformation("UserLoad")` — success path.
- `LogError(exception, "UserLoad")` — failure path, **exception as the first argument** (Serilog captures it structured; it never goes into the message).
- `LogWarning(...)` — anomalies that aren't failures (skipped, retried, ignored).

Alert recipe for free: rate of `message="UserLoad" AND level=Error` over total.

## Correlation in every scope

`AggregateId` and `CorrelationId` are pushed in **every** logging scope — they tie the log line to the end-to-end trace (the same correlation that flows bus → services → persistence in the Meta* model). A log line you can't correlate is a log line you can't debug.

## Pipeline vs application logs

Cross-cutting enrichment (request logging, exception logging, environment/version enrichers) comes from the packages (`JorgeCostaMacia.Serilog`, `Http.Serilog`, `Http.Exception.Serilog`) wired in the host — don't hand-roll it per service. This style governs the **application-level** log statements you write in handlers, services, consumers, jobs.

## Anti-patterns

- ✘ `LogInformation($"User {id} loaded")` — interpolation; cardinality bomb.
- ✘ `LogInformation("Loaded user {UserId}", id)` — placeholder renders into the message; use a fixed message + `PushProperty`.
- ✘ `LogError($"Failed: {ex.Message}")` — failure detail in the message AND the exception lost as structured data.
- ✘ Serializing objects into the message — that's what `destructureObjects` is for.
- ✘ One-off phrase messages per call site ("about to start processing the thing…") — unqueryable noise; if it deserves a log, it deserves a stable event name.
- ✘ Levels as decoration — level encodes outcome severity (Information/Warning/Error), not emphasis.
