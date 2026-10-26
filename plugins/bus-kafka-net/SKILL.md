---
name: bus-kafka-net
description: >-
  How the services use the Kafka transport of the JorgeCostaMacia.Bus family
  — the decisions on top of the library (see `bus-net` for what every
  transport shares; the library's own mechanics are in its README). Use when
  naming or creating topics, choosing partitions, touching consumer groups,
  thinking about message keys or ordering on Kafka, reading parked messages,
  or giving a service its broker credentials. It encodes: the topic is the
  route with the contracts' major, partitions declared per service in the
  admin map, no message key on purpose, the consumer group as a stable
  contract named after the handler, parked messages in .error, and one
  broker user per service.
---

# The bus on Kafka — how we use it

The decisions the services make on top of `JorgeCostaMacia.Bus.Kafka`. How the library works inside — its error handling, its throttling defaults, its health checks — is its own README's; this skill is what a service decides.

## Topics

- **A topic is a route**: `<service>.v{major}.<module>.<action>.{command|event}`, the module in kebab case. The major is read from the contracts assembly — `typeof(XBusPresentationContext).Assembly.GetName().Version?.Major` — so it is the contracts package's major and moves only with it.
- **Topics are created by the admin map, in the contracts package**, with their partition count; the producer map registers the same messages without one. Both are composed in the host, the service's own first, then every consumed service's.
- **Partitions are the unit of parallelism**, declared per service as two literals — events and commands. A command that fans out into many (one per page) needs as many command partitions as the consumers meant to share them; a service whose commands don't fan out can keep fewer.

## No message key

Messages are produced **without a key**: two messages for one aggregate can land on different partitions and be handled at once. That is deliberate — the services' ids are too heterogeneous to key by — and the consumer defends the order instead (version in the data, strict guard, optimistic concurrency; see `messaging`). Don't add a key to "fix" ordering.

## Consumer groups

- **The group id is the handler's or subscriber's route** (`{command route}.handler`, `{…}.on.{event route}.subscriber`). It is a **stable contract**: Kafka keeps the group's offsets under it, so renaming a route starts a new group from scratch and abandons the old one's position.
- Each handler and subscriber is its own group, so every service — and every subscriber within one — receives every event of a topic it subscribes to.

## Parked messages

A message that is not retried — a `ValidationException`, or the ladder exhausted — ends up in **`<topic>.error`**, with the exception and the failing group; one that can't even be read, in `<topic>.fault`. Both are topics: a parked message is read and replayed from there, never lost.

## Credentials

- `Bus:Producer`, `Bus:Consumer` and `Bus:Admin` each carry `BootstrapServers`, `SaslUsername`, `SaslPassword`. The passwords are secrets — user secrets locally, the deployment's secret in production — never in `appsettings*.json`.
- **One broker user per service**: a service never connects with another's credentials, so ACLs and the broker's logs name the real client.
