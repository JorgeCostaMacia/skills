<p align="center">
  <img src="https://raw.githubusercontent.com/JorgeCostaMacia/skills/main/assets/social-preview.png" width="100%" alt="skills" />
</p>

# skills

Curated agent skills and workflows for AI coding agents — installable as a Claude Code plugin marketplace ([agentskills.io](https://agentskills.io) standard).

Each skill lives as its own plugin under `plugins/`, so you can install only what you need. The stack-agnostic skills (`ddd`, `testing`, `logging`, `messaging`…) hold the principles; each `-net` skill is their .NET realization and declares a `dependencies` entry on them, so installing a `-net` skill enables the principles it builds on. The stack-agnostic skills (`ddd`, `testing`, `logging`, `messaging`…) hold the principles; each `-net` skill is their .NET realization and declares a `dependencies` entry on them, so installing a `-net` skill enables the principles it builds on.

## Available plugins

| Plugin | Description |
| --- | --- |
| [gitflow](plugins/gitflow) | A GitFlow workflow for branches and releases: feature/bugfix/release/hotfix start and finish, SemVer tags, default merge messages, commit identity, and the phased-publish caveat. |
| [solid](plugins/solid) | Transversal software-engineering principles: SOLID, simple design, clean code, code smells and complexity management — applied with judgment, not dogma. |
| [pnpm](plugins/pnpm) | Package-management baseline for any Node.js work: pnpm 12 with dependency build scripts denied unless reviewed (allowBuilds), a strict 3-day minimum release age, no exotic transitive sources, pinned package manager and Node 26, and frozen lockfile in CI. |
| [clean-architecture](plugins/clean-architecture) | Clean architecture layering with hexagonal (ports & adapters) boundaries, stack-agnostic and canon-anchored: the four layers inside each bounded context, the inward dependency rule, ports for external capabilities, and the name mapping to Evans, Uncle Bob, Microsoft and Jason Taylor. |
| [clean-architecture-net](plugins/clean-architecture-net) | The .NET realization of clean architecture: one project per module with its four layer folders, internal by default with one public *Context per module, the static Add<Module>Context / Map<Module>…Context composition, the host as composition only (ProgramBuilder / ProgramApplication), the published Contracts and Query and the internal Shared kernels, and the composition test. *(depends on `clean-architecture`)* |
| [ddd](plugins/ddd) | Tactical Domain-Driven Design, stack-agnostic and canon-anchored: ubiquitous language, aggregates, factories & hydration, value objects, validation principles, per-aggregate data accessors, domain events and domain errors. |
| [ddd-net](plugins/ddd-net) | The .NET realization of tactical DDD: the JorgeCostaMacia.* base types, value objects with their four factories and Convert, no coercion or defaults, aggregates (From cascades, Create seals), the exception family with fixed codes, events collected on the root, and one DbContext per aggregate as its accessor. *(depends on `ddd`, `validation-net`)* |
| [validation-net](plugins/validation-net) | The .NET realization of validation with FluentValidation, for anything that gets validated: one validator and one exception per validated thing, per-call validators from DI or static Create() chains, composition with Include/SetValidator, every rule evaluated (no Cascade), rule and message conventions, optional boundary validators, and use-case preconditions in Application. *(depends on `ddd`)* |
| [testing](plugins/testing) | Testing principles and the pragmatic TDD loop, stack-agnostic and canon-anchored: done means tested, the test pyramid, one test file per unit, names as specification, AAA and independence, classicist test doubles, behavior over implementation, and rule coverage over line coverage. |
| [testing-net](plugins/testing-net) | The .NET realization of the testing principles: xUnit v3 on Microsoft.Testing.Platform, test project layout and shared props, InternalsVisibleTo, naming, TestContext cancellation (xUnit1051), Testcontainers fixtures built from the repo's own scripts, test-data helpers, the clock, the host composition test, WebApplicationFactory host tests with hand-written fakes, and running the suites. *(depends on `testing`)* |
| [logging](plugins/logging) | Application logging principles for any stack: the message as a fixed grouping key, all variable data as structured properties, the level as the outcome, correlation in every scope, a failure logged once where it is handled, and observability over log volume. |
| [logging-net](plugins/logging-net) | The .NET realization of the logging principles with Serilog: GetType().Name as the message, LogContext properties (a using per property for one statement, a PushProperties helper per class for a reused set), destructured payloads, the exception first in LogError, correlation properties in every scope, the pipeline packages, and minimum levels in configuration. *(depends on `logging`)* |
| [messaging](plugins/messaging) | Asynchronous messaging principles for any stack and broker: commands vs events, contracts with the major in the topic, every message assumed duplicated and out of order, the version in the data with a strict newer-wins guard, optimistic concurrency on the write, retries owned by the broker, and malformed messages parked, not retried. |
| [bus-net](plugins/bus-net) | The .NET realization of the messaging principles with the JorgeCostaMacia.Bus family, for every transport: Command/Event records with their two constructors, thin CommandHandler/EventSubscriber adapters, Send/Publish with the incoming transport, the standard retry ladder with ValidationException excluded, retries parked in Quartz and run by bus-jobs, routes and the producer/admin/consumer maps. *(depends on `messaging`)* |
| [bus-kafka-net](plugins/bus-kafka-net) | How the services use the Kafka transport of the JorgeCostaMacia.Bus family: the topic as the route with the contracts major, partitions declared per service in the admin map, no message key on purpose, the consumer group as a stable contract named after the handler, parked messages in .error, and one broker user per service. *(depends on `bus-net`)* |
| [efcore-net](plugins/efcore-net) | EF Core conventions for any provider: value objects as ComplexProperty over .Value, keys, explicit column names, a column date kind stated in the mapping, != null in expression trees, sargable half-open date ranges, the token on every call and the FindAsync trap, scoped contexts, and the schema built from the real scripts. |
| [efcore-postgres-net](plugins/efcore-postgres-net) | EF Core on PostgreSQL with Npgsql: xmin as a shadow row version for optimistic concurrency, the legacy timestamp switch with timestamp without time zone kept on purpose, microsecond precision, the database collation, one schema per layer, and indexes only on foreign keys. *(depends on `efcore-net`)* |
| [code-style-net](plugins/code-style-net) | How C# is written, enforced as build errors: explicit types (no var, no target-typed new(), no collection expressions) as a stated departure from Microsoft's defaults, braces always, a 120-column width with all-or-nothing wrapping and growing lists kept vertical, naming and modifier conventions, is null outside expression trees, comments only where the code cannot speak, CRLF, dotnet format before committing, and no NoWarn or pragma. |
| [build-net](plugins/build-net) | How a .NET repository is built, versioned and restored: the three-tier Directory.Build.props, one VersionPrefix per repo and the separately released piece that keeps its own, Central Package Management, the NuGet vulnerability audit as a build gate, global.json, packing metadata for libraries, and a NuGet configuration without credentials. |

## Install

In Claude Code:

```
/plugin marketplace add JorgeCostaMacia/skills
/plugin install gitflow@jorgecostamacia-agent-skills
```

Then `/skills` to view, and invoke with `/gitflow`.

Update on demand:

```
/plugin marketplace update jorgecostamacia-agent-skills
```

## Layout

```
.claude-plugin/marketplace.json   # marketplace manifest (lists the plugins)
plugins/
  gitflow/
    .claude-plugin/plugin.json    # plugin manifest: name, description, version (lockstep), author, dependencies
    SKILL.md                      # the skill
```

To add a new skill: create `plugins/<name>/` with its `SKILL.md` and its `plugin.json` (every plugin carries the release's version, and a `-net` skill declares `dependencies` on the generic skills it builds on), add an entry to `marketplace.json`, and a row to the table above. See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

[MIT](LICENSE).
