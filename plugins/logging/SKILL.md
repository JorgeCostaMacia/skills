---
name: logging
description: >-
  Application logging principles — stack-agnostic. Use when writing or
  reviewing ANY log statement in any language, choosing a log message or
  level, adding logging to handlers, services, consumers, jobs or endpoints,
  carrying context or correlation into logs, deciding what to log and where,
  or weighing log volume against observability. It encodes: the message as a
  fixed, low-cardinality grouping key (never interpolated, never carrying the
  failure), all variable data as structured properties, the level as the
  outcome with the message unchanged, correlation in every scope, a failure
  logged once where it is handled, and observability over log volume. The
  library and its idioms live in each stack's skill.
---

# Logging (principles)

Logs are consumed as **structured data** — queried, grouped, counted and alerted on (Loki/Grafana or any equivalent). Every rule below serves that: a log line is a record with a fixed name and typed fields, not a sentence. The library, its API and its configuration are each stack skill's.

## The message is a grouping key

The message is a **fixed, low-cardinality constant** — the name of the event or of the unit that logs it:

- **Never interpolate.** `"User 42 loaded in 13ms"` makes every occurrence a distinct message; counts, dashboards and alerts on it die.
- **No templates rendered into the message either.** A placeholder the library substitutes into the text reintroduces the cardinality; the message groups, the data travels beside it.
- **The message never carries the failure** — no exception text, no offending value. Those go in the error and the properties; the message stays stable.
- **[our convention] Name it after the unit that logs it** — the handler's or the job's own name — so the key is unique, stable and written by no one: renaming the unit renames its series.

## All variable data as properties

Every value that varies — ids, counts, inputs, the payload — is a **structured property** of the line, queryable on its own (`UserId="42"`), never part of the text. Complex payloads are logged **structured** (as an object), not as their string form. Query by property, never by message text.

## The level is the outcome; the message stays

An operation's outcomes share **one message** and differ by **level**, so the series splits by level under one key:

| Level | Means |
| --- | --- |
| **Information** | the normal path |
| **Warning** | an anomaly that is not a failure — skipped, retried, ignored |
| **Error** | the **operation** failed; the error travels as structured data, not in the message |
| **Critical** | the **application** is dying — a startup failure, an unrecoverable state, the host going down |

Mental test for Critical: *if the process is still serving after the line, it wasn't Critical.* A failed request or message is Error. An alert falls out for free: the rate of `message=X AND level=Error` over the total.

## Correlation in every scope

The identity of what is being handled and the **correlation id** of the flow it belongs to are on **every** line — they tie it to the end-to-end trace across services. A line you can't correlate is a line you can't debug.

## Who logs what

- **Cross-cutting logging is the pipeline's**: request logging, error logging, environment and version enrichment are wired once in the host, by shared packages — never hand-rolled per service.
- **A failure is logged once, where it is handled** — usually the pipeline that turns it into a response, a retry or a parked message. Logging it at every layer on its way up multiplies one failure into many lines; log-and-rethrow is noise.
- **Application code logs its own events**: the line that says a handler, use case or job did its work, with its properties.
- **Never log secrets** — credentials, tokens, connection strings. Personal data only where the purpose justifies it.

## Observability over volume — [our convention]

Asynchronous pipelines log **one Information line per message handled, with its payload**, on purpose: a slower consumer costs nothing visible, while a lost trail costs the ability to trace a failed or parked message back to what it carried. Disk is managed by **retention and rotation**, not by raising levels or dropping lines. Change a minimum level per source in configuration — never by deleting log statements.

## Anti-patterns

- ✘ An interpolated message — a cardinality bomb.
- ✘ A placeholder template that renders values into the message.
- ✘ The exception's text in the message, and the exception itself not logged as structured data.
- ✘ An object serialized into the message instead of logged as a structured property.
- ✘ A one-off phrase per call site ("about to start processing…") — unqueryable; if it deserves a log, it deserves a stable name.
- ✘ Levels as emphasis — the level is the outcome.
- ✘ Critical for a failed operation.
- ✘ The same failure logged by every layer it passes through.
