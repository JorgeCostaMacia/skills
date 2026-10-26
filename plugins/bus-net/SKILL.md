---
name: bus-net
description: >-
  The .NET realization of the messaging principles with the
  JorgeCostaMacia.Bus family — what is the same whatever the transport (see
  the `messaging` skill for the why; the transport's own mechanics live in
  a transport skill of its own, such as `bus-kafka-net`). Use when writing or reviewing
  message records, command handlers and event subscribers, sending and
  publishing, the retry ladder and its exclude list, the Quartz retry store,
  the contracts package's bus maps, or the host's bus composition. It
  encodes: Command/Event sealed records with their two constructors,
  CommandHandler<T>/EventSubscriber<T> as thin adapters, IBus Send/Publish
  with the incoming transport, the standard backoff with ValidationException
  excluded, retries parked in Quartz and run by bus-jobs, route and handler
  naming, and the producer/admin/consumer maps.
---

# The bus in .NET (JorgeCostaMacia.Bus)

The .NET *how* of `messaging` (load it for the *why*), for what every transport shares. `JorgeCostaMacia.Bus` is the agnostic root; each transport (`Bus.Kafka`, `Bus.RabbitMQ`) ships the **same simple names** — `Command`, `Event`, `CommandHandler<T>`, `EventSubscriber<T>`, `IBus` — in its own namespace, so moving transports is swapping the `using`s. Everything below holds for both; the transport skill adds topics, partitions, groups and dead-letter mechanics.

## Message records

Messages live in the service's **contracts package**, never in the runtime. A command derives `Command`, an event `Event`; both are `sealed record`s carrying the aggregate metadata — `AggregateId`, `AggregateCorrelationId`, `AggregateOccurredAt`, `AggregateConsumers` (the destinations it is meant for, so other subscribers can filter it out).

```csharp
public sealed record PedidosExtractCommand : Command
{
    public DateTime? Date { get; init; }
    public int PageSize { get; init; }

    [JsonConstructor]                                   // the transport's: the full tuple, metadata first
    public PedidosExtractCommand(
        Guid aggregateId,
        Guid aggregateCorrelationId,
        DateTime aggregateOccurredAt,
        ImmutableList<string> aggregateConsumers,
        DateTime? date,
        int pageSize
    ) : base(aggregateId, aggregateCorrelationId, aggregateOccurredAt, aggregateConsumers)
    {
        Date = date;
        PageSize = pageSize;
    }

    public PedidosExtractCommand(                       // the producers': payload first, metadata optional
        DateTime? date,
        int pageSize,
        Guid? aggregateId = null,
        Guid? aggregateCorrelationId = null,
        DateTime? aggregateOccurredAt = null,
        IEnumerable<string>? aggregateConsumers = null
    ) : base(aggregateId, aggregateCorrelationId, aggregateOccurredAt, aggregateConsumers)
    {
        Date = date;
        PageSize = pageSize;
    }
}
```

- **Keep both constructors** and keep them in step; the contracts suite (`MessageContract`) proves every message survives the wire and both constructors agree.
- Payload is primitives and other records — no value objects, no domain types: a contract is a published language, not the model.
- Names: commands `<Module><Action>Command` (imperative), events `<Module><Fact>Event` (past tense: `Extracted`, `Loaded`).
- A `<Module>LoadedEvent` carries the row's previous state as `AggregateLoadedEvent` — the same event type, `null` when the row was created — built before the aggregate is mutated (see `messaging`).

## Handlers and subscribers

```csharp
internal sealed class PedidosExtractCommandHandler : CommandHandler<PedidosExtractCommand>
{
    public override async Task Handle(
        CommandContext<PedidosExtractCommand> context,
        CancellationToken cancellationToken = default)
    {
        List<Command> commands = await _extractService.Extract(
            context.Message.Date,
            context.Message.PageSize,
            context.Message.AggregateCorrelationId,
            context.Message.AggregateConsumers,
            cancellationToken);

        await _bus.Send(commands, context.Transport, cancellationToken);

        // the success line — see logging-net
    }
}
```

- `internal sealed`, in `Presentation/Bus/`, constructor-injected with its use-case service(s), `IBus` and `ILogger<T>`.
- **Thin**: unpack `context.Message` into the service's own parameters (never pass the message into Application), call the service with the **live token**, then `Send` / `Publish` what it returned, then log. No rule, no try/catch, no retry: a failure propagates to the bus.
- **Send and publish with `context.Transport`** — it carries the incoming envelope, so what the handler causes inherits its correlation.
- An event subscriber has the same shape (`EventSubscriber<T>`, `EventContext<T>`); it is named for what it does and what it listens to: `<Module><Action>On<Source><Module><Event>EventSubscriber`.

## Retries

Every handler and subscriber is registered with the **same ladder and the same exclude list**:

```csharp
ImmutableList<TimeSpan> retry = ImmutableList.Create(
    TimeSpan.FromMinutes(5),
    TimeSpan.FromMinutes(15),
    TimeSpan.FromMinutes(30),
    TimeSpan.FromHours(1),
    TimeSpan.FromHours(12),
    TimeSpan.FromDays(1));
ImmutableList<Type> exclude = ImmutableList.Create(typeof(ValidationException));
```

- **`ValidationException` (the shared base) is always excluded**: a malformed message is parked at once, never retried. A concurrency failure is *not* excluded — the retry re-decides against the current state. A consumer whose write is not idempotent also excludes the `ExistException` a retry cannot change (see `messaging`).
- A step of `TimeSpan.Zero` requeues at once; a positive one is **scheduled** through the retry store.
- **The retry store is Quartz** (`JorgeCostaMacia.Bus.<Transport>.Retry.Quartz`): `AddQuartz(q => q.WithPostgresDefaults("retry", "JobsBus", "bus"))` plus `AddRetryContext()` — the `retry` scheduler, the `JobsBus` connection, tables prefixed `bus.`. The worker **only parks** retries: it never adds `AddQuartzHostedService`. **bus-jobs** is the worker that starts that scheduler and fires them. Without a retry store a delayed retry cannot be scheduled and the message is parked as terminal.

## Routes and the maps

- **Routes are declared once, in the contracts package**: a base per module, then its events and its commands, as `static readonly string`s; the public `<Module>Context` re-exposes them as `Bus<Module><Message>Route`. The runtime derives its own: a handler is `{command route}.handler`, a subscriber `{base handler route}.on.{source event route}.subscriber`. A route with no message behind it (a `load.command` that only anchors subscribers) is kept: removing it renames the subscribers' routes.
- **Three maps, composed in the host's `AddBusContext(configuration, producer, consumer, admin)`**:
  - **producer** and **admin** — the service's own contracts maps **first**, then those of every service it consumes;
  - **consumer** — the service's own runtime modules (`Map<Module>BusContext`), which register its handlers and subscribers with their routes, ladder and exclude list.
- The host's **composition test** builds all of it with fake settings (see `testing-net`).

## Logging

Handlers log one success line each; the bus pipeline logs failures, retries and parking — see `logging-net`. Don't log in a handler what the pipeline already logs.
