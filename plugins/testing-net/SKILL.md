---
name: testing-net
description: >-
  The .NET realization of the testing principles (see the `testing` skill for
  the why). Use when creating or reviewing .NET test projects or tests:
  xUnit v3 on Microsoft.Testing.Platform, test project layout and shared
  test props, InternalsVisibleTo, test naming, cancellation tokens in tests
  (xUnit1051), Testcontainers fixtures and collections, building the test
  database from the repo's own scripts, test-data helpers, the host
  composition test, WebApplicationFactory host tests with hand-written fakes,
  and running the suites. It encodes the stack, the layout, the fixture
  patterns and the conventions that keep suites deterministic.
---

# Testing in .NET

The .NET *how* of `testing` (load it for the *why*). xUnit v3 is the framework; doubles are hand-written fakes — no mocking library.

## The stack

- **xUnit v3** (`xunit.v3`, plus `xunit.runner.visualstudio` for IDE discovery) on **Microsoft.Testing.Platform**. The runner is selected by a root `global.json`:

  ```json
  {
    "test": {
      "runner": "Microsoft.Testing.Platform"
    }
  }
  ```

  A test project is an executable (`<OutputType>Exe</OutputType>`), so every suite can also run on its own.
- **Testcontainers** for real infrastructure (`Testcontainers.PostgreSql`, `.MariaDb`, `.Kafka`…), **`Microsoft.AspNetCore.Mvc.Testing`** for HTTP hosts.
- Versions through Central Package Management like everything else; a test `.csproj` lists packages without versions.

## Layout

- `test/<Project>.Tests/` — one test project per production project, its folders mirroring the production tree (`Domain/`, `Application/`…). An API whose modules are endpoints only has instead one suite for the host, with a file per endpoint (`Endpoints/<Module>EndpointTests.cs`). **One file per unit**: `<Unit>Tests.cs`, a `public sealed class <Unit>Tests`.
- `test/<Project>.Tests/Support/` — fixtures, test-data helpers and fakes, next to the tests that use them.
- `test/Directory.Build.props` — the configuration every test project shares: it imports the root props, sets `IsPackable`/`IsPublishable` false and `OutputType` `Exe`, references `xunit.v3` and the runner, adds `<Using Include="Xunit" />`, and copies whatever the suites run against (the schema scripts) to the output. A helper every suite needs is compiled in with `<Compile Include="…\Shared\X.cs" Link="Shared\X.cs" />` rather than copied into each project.
- The production project exposes its internals to its own suite only: `<InternalsVisibleTo Include="<Project>.Tests" />`. The domain stays `internal`; nothing is made public for a test's sake.

## Names

`Method_StateUnderTest_ExpectedBehavior`, the method being the unit's public entry point: `Extract_WithoutADate_ReturnsTheCountOfTheTable`, `Create_OnAnEmptyCorrelationId_ThrowsValidationException`. One act per test: an "And" joining two acts is two tests; one joining facets of a single outcome (`Load_WithNoIds_ReturnsEmptyAndDeactivatesNothing`) is one.

## Cancellation

Pass **`TestContext.Current.CancellationToken`** to every async call a test makes — the run's own token, so an aborted run stops its queries. Analyzer **xUnit1051** flags a call that omits it (an error under `TreatWarningsAsErrors`).

xUnit1051 only sees calls written in a test method. A helper that calls the code under test takes the token as a **required** parameter (no `= default`) and passes it on — otherwise propagation stops silently at exactly the seam every test goes through.

## Integration tests on real infrastructure — Testcontainers

```csharp
public sealed class PostgreSqlFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _container = new PostgreSqlBuilder("postgres:16").Build();

    public async ValueTask InitializeAsync()
    {
        await _container.StartAsync(TestContext.Current.CancellationToken);

        await using XContext context = NewContext();
        foreach (string script in new string[] { "schema.sql", "tables.sql", "indexes.sql" })
        {
            await context.Database.ExecuteSqlRawAsync(
                await File.ReadAllTextAsync(Path.Combine(AppContext.BaseDirectory, "database", script), TestContext.Current.CancellationToken),
                TestContext.Current.CancellationToken);
        }
    }

    public async ValueTask DisposeAsync() => await _container.DisposeAsync();

    internal XContext NewContext()
        => new XContext(new DbContextOptionsBuilder<XContext>().UseNpgsql(_container.GetConnectionString()).Options);

    internal async Task ResetAsync()
    {
        await using XContext context = NewContext();
        await context.XRepository.ExecuteDeleteAsync(TestContext.Current.CancellationToken);
    }
}

[CollectionDefinition(nameof(PostgreSqlCollection))]
public sealed class PostgreSqlCollection : ICollectionFixture<PostgreSqlFixture>;
```

- **The schema comes from the repo's own scripts** (copied to the output by `test/Directory.Build.props`), never from `EnsureCreated` — that would build whatever the model says, so a mapping drifted from the real tables would still pass. A new column goes into the model and the script together.
- **One container per collection**: the fixture is an `ICollectionFixture`, each test class carries `[Collection(nameof(PostgreSqlCollection))]` and takes the fixture in its constructor. xUnit runs a collection's classes one at a time, so the container is started once and shared safely.
- **Each test resets what it uses** (`ResetAsync`, through `ExecuteDeleteAsync` — no raw SQL) and seeds its own data, so counts and slices are exact.
- **A fresh context per step** (`NewContext()`): seeding, acting and asserting each use their own, so the change tracker never answers a question the database should.
- **Mirror the host's process-wide switches** in `InitializeAsync` (an `AppContext.SetSwitch` the host's `Program` sets), or the suite runs under different rules than production.
- Docker must be running; the suites start their own containers.

## Test data

A static `<Unit>TestDatabase` (or mother) in `Support/` gives the Arrange its vocabulary:

- the **canonical values** every seeded row carries (an instant, a seeded and a changed value of the field under test), so a test can place its input before or after them;
- **`Entity(id, …)`** — one valid row, with optional parameters only for the axes tests vary (those parameters are the builder `testing` describes, without a fluent type) — and **`Rows(count)`** for slices asserted by id;
- built through **`From`**, not `Create`, so seeding never trips a field rule unrelated to the test;
- a helper that holds a **long argument list once** (a load with fifty fields), so tests pass only what they vary.

## Time

When the code reads the clock itself, assert against a window: capture `DateTime.UtcNow` before and after the call and check the stamped instant falls inside. To prove *ordering* against the store — "stamped before it read" — record when each query reached the server with a `DbCommandInterceptor` (override `ReaderExecutingAsync` and `ScalarExecutingAsync`, add it in `NewContext()`), and compare with the last read.

## The composition test

The host has one test that builds its container the way production does, against fake settings, with both checks on:

```csharp
[Fact]
public void AddBuilder_ComposesAValidContainer()
{
    HostApplicationBuilderSettings settings = new HostApplicationBuilderSettings { DisableDefaults = true };
    HostApplicationBuilder builder = Host.CreateApplicationBuilder(settings);
    builder.Configuration.AddInMemoryCollection(_settings);

    builder.AddBuilder();

    ServiceProviderOptions options = new ServiceProviderOptions { ValidateOnBuild = true, ValidateScopes = true };
    using ServiceProvider provider = builder.Services.BuildServiceProvider(options);
}
```

It is what catches a registration that does not resolve, or a scoped service captured by a singleton, before a deploy — and it is what lets thin adapters go without tests of their own (see `testing`).

## HTTP hosts — through the real pipeline

- A `WebApplicationFactory<Program>` subclass, shared through an `ICollectionFixture`: one host for every endpoint suite, not one per class.
- Outbound ports are replaced with **hand-written fakes** in `ConfigureTestServices` (`services.AddSingleton<IBus>(Bus)`): the fake records what was published, the test asserts on it.
- Endpoints are tested through the host — routing, binding, authentication, the exception handler and the request log all run as in production.

## Running

`dotnet test` runs the suites through Microsoft.Testing.Platform. Each built suite is also an executable: when a machine's `dotnet test` discovers nothing, run `test/<Project>.Tests/bin/<Configuration>/<tfm>/<Project>.Tests.exe` directly — same tests, same exit code.
