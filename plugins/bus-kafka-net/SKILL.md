---
name: bus-kafka-net
description: >-
  The Kafka transport of the JorgeCostaMacia.Bus family
  (JorgeCostaMacia.Bus.Kafka, .Retry.Quartz, .HealthChecks) — see `bus-net`
  for what every transport shares. Use when working with Kafka topics,
  partitions, consumer groups, the admin/producer/consumer configuration,
  the .error and .fault topics, the topic major, or the Bus configuration
  sections. It encodes: the topic name is the route with the contracts'
  major, partitions declared in the admin map, no message key, the consumer
  group as a stable contract named after the handler, the error and fault
  topics and when each is used, startup and topic-creation throttling, and
  per-service credentials.
---

# The bus on Kafka

What `bus-net` leaves to the transport, for `JorgeCostaMacia.Bus.Kafka` (own implementation on Confluent.Kafka, no MassTransit). Types come from `JorgeCostaMacia.Bus.Kafka.Domain.*` (messages, handlers) and `JorgeCostaMacia.Bus.Kafka.Infrastructure.*` (configurators).

## Topics

- **A topic is a route**: `<service>.v{major}.<module>.<action>.{command|event}`, the module in kebab case. The major is read from the contracts assembly — `typeof(XBusPresentationContext).Assembly.GetName().Version?.Major` — so it is the contracts package's major and moves only with it.
- **Topics are created by the admin map**, in the contracts package, with their partition count:

  ```csharp
  admin
      .AddEvent<PedidosExtractedEvent>(PedidosExtractedEventRoute, eventPartitions)
      .AddCommand<PedidosExtractCommand>(PedidosExtractCommandRoute, commandPartitions);
  ```

  The producer map registers the same messages without partitions. Both maps are composed in the host: the service's own first, then every consumed service's.
- **Partitions are the unit of parallelism**, declared per service as two literals: events and commands. A command that fans out into many (one per page) needs as many command partitions as the consumers that should share them; a service whose commands don't fan out can keep fewer.

## No message key

Messages are produced **without a key**: the partitioner spreads them, so two messages for one aggregate can land on different partitions and be handled at once. That is deliberate — the services' ids are too heterogeneous to key by — and the consumer defends the order instead (version in the data, strict guard, optimistic concurrency; see `messaging`). Don't add a key to "fix" ordering.

## Consumer groups

- **The group id is the handler's or subscriber's route** (`{command route}.handler`, `{…}.on.{event route}.subscriber`). It is a **stable contract**: Kafka keeps the group's offsets under it, so renaming a route starts a new group from scratch and abandons the old one's position.
- Each handler and subscriber is its own group, so every service — and every subscriber within one — receives every event of a topic it subscribes to.

## When a message fails

The error handler decides, per failed delivery:

| Situation | Where it goes |
| --- | --- |
| Retryable, next step `TimeSpan.Zero` | requeued at the **tail of the same topic** |
| Retryable, next step > 0 | **scheduled** through the Quartz retry store, produced back to the topic at its time |
| Excluded exception, ladder exhausted, or no retry store | **`<topic>.error`** — a `CommandError` / `EventError` with the message, the exception and the failing group |
| The envelope cannot be read, or the error handling itself breaks | **`<topic>.fault`** |

A parked message is never lost: `.error` and `.fault` are topics, readable and replayable.

## Configuration

- Three sections, `Bus:Producer`, `Bus:Consumer`, `Bus:Admin`, each with `BootstrapServers`, `SaslUsername`, `SaslPassword`. The passwords are secrets — user secrets locally, the deployment's secret in production — never in `appsettings*.json`.
- **One broker user per service**: a service never connects with another's credentials, so ACLs and the broker's logs name the real client.
- Startup is throttled by defaults worth keeping: consumers start a few at a time (`StartupMaxConcurrency`, 2) and the admin creates topics in batches (`TopicsBatchSize`, 25) — a service with hundreds of handlers otherwise opens a SASL handshake storm against the brokers.

## Health

`JorgeCostaMacia.Bus.Kafka.HealthChecks` adds the bus to the host's health checks: `services.AddHealthChecks().AddKafkaBus()` (name `bus-kafka` by default) — wire it where the host exposes health.
