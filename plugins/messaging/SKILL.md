---
name: messaging
description: >-
  Asynchronous messaging principles — stack- and broker-agnostic. Use when
  designing or reviewing message-driven code: commands and events, their
  contracts, consumers and handlers, retries and dead letters, ordering,
  duplicates and idempotency, concurrent writes from consumers, or when a
  message is lost, applied twice, applied out of order or retried forever.
  It encodes: commands vs events, contracts as a published language with the
  version in the topic, every message assumed duplicated and out of order,
  the version travelling with the data and a strict newer-wins guard,
  optimistic concurrency on the write, retries owned by the broker (never a
  loop in the use case), and malformed messages parked, not retried. The
  library and broker specifics live in each stack's skill.
---

# Messaging (principles)

Message-driven systems fail in ways synchronous ones don't: messages arrive twice, late, out of order, or never succeed. These rules make that the normal case instead of a bug. The bus library and the broker's mechanics are each stack skill's.

## Commands and events

- A **command** asks one service to do something (`ExtractPedidos`): one handler, named in the imperative, it may be refused.
- An **event** states that something happened (`PedidosLoaded`): any number of subscribers, named in the past tense, nobody can veto it.
- A service **owns** the commands it accepts and the events it publishes; it never publishes another service's events.

## Contracts are the published language

- Messages are declared in the **service's own contract**, versioned and published on its own; consumers depend on the contract, never on the service.
- **The contract's major is in the topic/route name** (`<service>.v6.<module>.<action>.event`). A breaking change is a new major: new topics, consumers move when they upgrade, nothing breaks in flight. A minor or patch never moves a topic — keep changes additive.
- Routes are names, not code: derived from one base per module, kebab case, `.command` / `.event` suffixes, so a topic, its consumer group and its dead letter can be found from each other.

## Assume duplicates and disorder

Every consumer is written as if each message could arrive **twice, late, or before an older one** — redelivery after a crash, a retry, a reload, two partitions handled at once. Don't lean on broker ordering: it holds only within a partition, and only until something is retried.

- **The version travels with the data.** Each message carries the instant (or sequence) of the state it describes, and the consumer stores it with what it writes.
- **Apply only what is strictly newer** (`stored < incoming`). An equal version is a redelivery — a no-op. An older one is stale — dropped.
- **A newer message always moves the stored version**, even when it changes nothing else; otherwise the row looks older than it is, and a stale message handled later wins.
- **Publish only on a real change**, so identical messages don't ripple downstream — and let the event carry the **previous state** next to the new one (`null` when the row is new), so a consumer can tell what changed without a second read.
- **One instant per operation**: the version stamped on what an operation writes and publishes is taken once, at its start.

## Concurrent writes

Two consumers may handle messages for the same row at the same time. **The write is guarded by optimistic concurrency**: a save over a row changed since it was read fails, the message is retried, and the guard decides again against the row as it now is. A race that surfaces as a duplicate-key failure on insert resolves the same way **when the write is idempotent** — the version travels with the data, so the retry becomes an update or a no-op. Where it is not, a duplicate key is a business fact ("already exists") that no retry will change: it is parked, not retried.

## Retries belong to the broker

- **No retry loop in the use case.** A failure — a conflict, a timeout, a race — escapes the handler; the messaging infrastructure retries the message with a **backoff ladder** (minutes to a day), so a dependency that is down for an hour costs nothing but delay.
- Retries are **durable**: requeued or scheduled outside the process, so a restart loses none.
- **Malformed messages are not retried.** A validation failure is excluded from retries and **parked** in a dead-letter destination at once — retrying it can only fail the same way. Parked messages are kept, readable and replayable; nothing is silently dropped.
- A message that cannot even be read (corrupt envelope) is parked separately from one that failed in the handler.

## Handlers

- A handler is a **thin driving adapter**: unpack the message, call one use case, send or publish what it returns, log its line. No rule lives there.
- A handler that sends commands or publishes events does it **after** the use case succeeded; a failed use case sends and publishes nothing.
- The correlation id of the incoming message travels on everything it causes.
