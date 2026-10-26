---
name: logging-net
description: >-
  The .NET realization of the logging principles (see the `logging` skill for
  the why) — Serilog is the accepted coupling. Use when writing or reviewing
  ANY log statement in .NET (ILogger calls, LogInformation/Warning/Error),
  pushing properties into the log context, adding logging to handlers,
  services, consumers or jobs, or configuring Serilog. It encodes: the
  message as GetType().Name, LogContext.PushProperty for single statements
  and one PushProperties helper per class when the same set serves several,
  destructureObjects for payloads, the exception as LogError's first
  argument, correlation properties in every scope, the JorgeCostaMacia.Serilog
  pipeline packages, and minimum levels set in configuration.
---

# Logging in .NET (Serilog)

The .NET *how* of `logging` (load it for the *why*). **Serilog is the accepted core coupling** — not wrapped, not replaced. Code logs through an injected `ILogger<T>`; Serilog is the provider behind it.

## The message

```csharp
_logger.LogInformation(GetType().Name);
```

The class's own name is the grouping key: fixed, unique, and never typed by hand. Use a literal (`"UserLoad"`) only where one class logs more than one event.

- No interpolation (`$"…"`), no `{Placeholder}` templates — Serilog renders those into the message.
- Failure: `_logger.LogError(exception, GetType().Name)` — **the exception is the first argument**, so Serilog captures it structured; the message is the same as the success path's, the level tells them apart.
- Levels keep `logging`'s names through `ILogger` (`Information`, `Warning`, `Error`, `Critical`); Serilog writes `Critical` as `Fatal` — the same level, so a `MinimumLevel` of `Fatal` in configuration is `Critical` in code.

## Properties — through the log context

Every property is pushed into `Serilog.Context.LogContext` around the statement; the line carries whatever is pushed when it is written.

**One log statement — push each property in its own `using`:**

```csharp
using (LogContext.PushProperty("PageSize", context.Message.PageSize))
using (LogContext.PushProperty("Commands", commands.Count))
using (LogContext.PushProperty("AggregateId", context.Message.AggregateId))
using (LogContext.PushProperty("CorrelationId", context.Message.AggregateCorrelationId))
{
    _logger.LogInformation(GetType().Name);
}
```

It reads at a glance, and a generated handler stays uniform with every other.

**The same set in several statements — one `PushProperties` helper per class**, so the list lives once and the statements cannot drift apart (today only the libraries' own loggers log several lines with one set — the bus's does it this way; an application class logs one):

```csharp
private static IDisposable PushProperties(
    UserLoadRequest request,
    Guid correlationId)
    => LogContext.Push(
        new PropertyEnricher("AggregateId", request.AggregateId),
        new PropertyEnricher("CorrelationId", correlationId),
        new PropertyEnricher("UserId", request.UserId),
        new PropertyEnricher("Request", request, destructureObjects: true));
```

```csharp
using (PushProperties(request, correlationId))
{
    _logger.LogInformation(GetType().Name);
}
```

- `PropertyEnricher` lives in `Serilog.Core.Enrichers`; `LogContext.Push` takes them all and returns one disposable.
- **`destructureObjects: true`** for a complex payload — logged as a structured object, not its `ToString()`.
- **`AggregateId` and `CorrelationId` in every scope**, under those names, so every service's lines join on them.
- If another library in the project ships its own `LogContext`, alias Serilog's: `using LogContext = Serilog.Context.LogContext;`.

## Wiring — the host

Every host registers Serilog the same way, in its `ProgramBuilder`:

```csharp
if (builder.Environment.IsDevelopment())
{
    Serilog.Debugging.SelfLog.Enable(Console.Error);   // Serilog's own errors (a broken sink, a bad config) to the console
}

builder.Services.AddSerilog((services, config) => config
    .ReadFrom.Configuration(builder.Configuration)      // levels, overrides and sinks from appsettings
    .WithDefaults());                                   // JorgeCostaMacia.Serilog
```

`WithDefaults()` (`JorgeCostaMacia.Serilog.Infrastructure`) adds what every line needs: `FromLogContext` — without it no pushed property reaches the line —, the thread and process ids, structured exception details, and the entry assembly's `Version` and `Application`. Don't re-add those enrichers by hand; anything a service needs on top is configuration.

### HTTP hosts

The request and exception logging come from `JorgeCostaMacia.Http.Serilog` and `JorgeCostaMacia.Http.Exception.Serilog`:

```csharp
// ProgramBuilder
builder.Services.AddExceptionHandler<ExceptionHandler>();      // logs the exception once, maps it to ProblemDetails

// ProgramApplication — order matters
app.UseBodyBufferMiddleware();                                  // lets the request body be read for the log
app.UseEnrichRequestMiddleware();                               // pushes the request's properties around the rest
app.UseSerilogRequestLogging(options => options.WithDefaults());
app.UseExceptionHandler(new ExceptionHandlerOptions().WithDefaultStatusCodes());
app.UseStatusCodePages();
app.UseRouting();
app.UseAuthentication();
app.UseEnrichAuthenticationMiddleware();                        // pushes the authenticated user
app.UseAuthorization();
```

**The request log wraps the exception handler and authorization**: a failed request is then logged with the status it really got (400, 500), and a refused one (401, 403) is logged too. Put the request log inside the exception handler and every failure is logged as a 500. An endpoint handler logs nothing of its own: the request log is its line, with the status it answered and the properties the enrichers pushed.

### Bus hosts

The bus pipeline logs a failed message and its retry or parking, and its own lines already carry the message `Body` (the bus's logger pushes it into the context). A handler therefore logs only its own success line — pushing its parameters and counts, never the payload again — and lets an exception propagate: logging it there too would log one failure twice.

## Configuration

Serilog is configured from `appsettings` (`Serilog:MinimumLevel`), and a source is turned up or down with **`MinimumLevel:Override`** — never by deleting statements. When the default is stricter than a package's own logs (a `Default` of `Error` hiding a library's expected-failure `Warning`s), override that package's namespace instead of lowering the default.
