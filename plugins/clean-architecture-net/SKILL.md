---
name: clean-architecture-net
description: >-
  The .NET realization of clean architecture (see the `clean-architecture`
  skill for the principles): how a .NET service is packaged and composed.
  Use when creating or reviewing a .NET solution's structure, adding a
  module (bounded context) or a project, deciding what is public or
  internal, registering a module's services, wiring the host's
  ProgramBuilder/ProgramApplication, or placing contracts, read models and
  shared kernels. It encodes: one project per module with its four layer
  folders, internal by default with one public *Context per module, the
  static Add<Module>Context / Map<Module>…Context composition, the host as
  composition only, the published pieces (Contracts, Query) and the internal
  Shared kernels, the reserved segments, and the composition test.
---

# Clean architecture in .NET

The .NET *how* of `clean-architecture` (load it for the *why*): how a service maps its bounded contexts and layers onto projects, and how the host composes them.

## Projects

```
src/
├─ <Service>/                     the host: Program.cs, ProgramBuilder.cs, ProgramApplication.cs — no logic of its own
├─ <Service>.<Module>/            one project per bounded context
│  ├─ Domain/                     aggregates, value objects, validators, typed exceptions
│  ├─ Application/                the use-case services
│  ├─ Infrastructure/             <Module>InfrastructureContext, the DbContext, external clients
│  ├─ Presentation/               the driving adapters (bus handlers, endpoints) + <Module>…PresentationContext
│  └─ <Module>Context.cs          the module's only public type
├─ <Service>.Shared.<X>/          a shared kernel, one per concern — internal, never packed
└─ <Service>.Query/               the read model, packed with its own version (when the service has one)
test/
└─ <Service>.<Module>.Tests/      one suite per project (an API with endpoint-only modules: one suite for the host)
```

- **The layers are folders inside the module's project**, not projects of their own: the dependency rule of `clean-architecture` (Domain imports nothing from the other folders, Application only Domain, Infrastructure and Presentation only inward and never each other) is held by review and by the namespaces, and a module stays one unit to add, move or delete. A module holds **the layers it needs**: a worker's module has the four; a webhook module that only turns a request into an event has `Presentation/` alone. A module's `.csproj` references contracts packages — its own and those it consumes — and the service's shared kernels it uses, never another module's project.
- **The host references every module project** and nothing else of theirs but their `<Module>Context`.
- The module is named in the ubiquitous language (`Pedidos`, `Expedientes`), and so is its aggregate — which makes `Pedidos` both the namespace segment and the type. Inside the module the type is reached as `Domain.Pedidos` where the namespace would shadow it, and the module's contracts context as `Contracts.Pedidos.PedidosContext`, since the runtime's own `PedidosContext` hides it. Both forms are the convention, not a smell — in `src/`; the `using PedidosAggregate = …` alias is the tests' idiom (see `testing-net`), not production's. `Contracts`, `Query` and `Shared` are **reserved segments**: no module may take one as its name.

## Internal by default

Everything inside a module is `internal`: aggregates, value objects, services (`internal sealed`), handlers, the DbContext, the layer contexts. The one public type is the static `<Module>Context`. The module's suite reaches the rest through `<InternalsVisibleTo Include="<Service>.<Module>.Tests" />` in the module's `.csproj` — nothing is made public for a test's sake.

## The *Context composition pattern

Each module composes itself through static extension methods, layer by layer:

```csharp
public static class PedidosContext
{
    public static IServiceCollection AddPedidosContext(this IServiceCollection services, IConfiguration configuration)
    {
        services
            .AddPedidosInfrastructureContext(configuration);

        return services;
    }

    public static ConsumerConfigurator MapPedidosBusContext(this ConsumerConfigurator consumer)
    {
        consumer
            .MapPedidosBusPresentationContext();

        return consumer;
    }
}
```

- **`Add<Module>Context(services, configuration)`** registers the module's services by calling its layers' own `internal static` contexts — `<Module>InfrastructureContext` registers the DbContext, the clients and the Application services (`AddScoped<XService>()`, concrete, no interface). A module that reads no configuration takes only `services`; one that registers nothing returns them unchanged — the method still exists, so the host composes every module the same way.
- **`Map<Module>…Context(...)`** maps the module's driving adapters: `Map<Module>BusContext(consumer)` for its bus handlers, `Map<Module>HttpContext(endpoints)` for its endpoints (a `MapGroup` per module).
- **Configuration is read where it is used**, and a missing required value fails at startup with a clear message (`GetConnectionString("X") ?? throw new InvalidOperationException("Connection string 'X' is null.")`).
- Each method returns what it received, so the host chains them.

## Application services

A use case is one `internal sealed class <Module><UseCase>Service`, registered scoped, with one public entry point. It takes **primitives** — the values the adapter unpacked from its message or request — plus the correlation id and the consumers, and a `CancellationToken cancellationToken = default` last; it builds the aggregate through `Domain.<X>.Create(…)`, so every path validates the same way; works through the module's `DbContext`; takes one `DateTime now = DateTime.UtcNow` at its start; and **returns** what the adapter then sends or publishes (commands, events, the aggregate). It never receives the transport's object, and never sends or publishes itself.

## The host — composition only

```csharp
// Program.cs — process-wide switches, then build and run
IHost host = Host
    .CreateApplicationBuilder(args)
    .AddBuilder()
    .Build()
    .AddApplication();

host.Run();
```

- **`ProgramBuilder.AddBuilder`** does the whole composition: logging, the cross-cutting infrastructure (bus, scheduler, HTTP pipeline services), then every module — `.AddPedidosContext(builder.Configuration)` and the `Map…Context` calls, one line per module, in one place.
- **`ProgramApplication.AddApplication`** is the post-build seam: the HTTP middleware pipeline and the endpoint maps in an API, a pass-through in a worker — kept as the hook either way.
- `Program.cs` holds only what must run before the builder exists (an `AppContext.SetSwitch`).
- No business logic, no use case, no query in the host. An operator's tool (a one-off hosted service) lives there commented out, never registered by default.

## Published and shared pieces

| Piece | Name | Where | Published |
| --- | --- | --- | --- |
| The module's messages — the Published Language | `<Service>.Contracts.<Module>` | its own repository, its own version | yes, a NuGet package per module |
| The read model | `<Service>.Query` | `src/` of the service, its own `VersionPrefix` | yes, one package |
| A shared kernel | `<Service>.Shared.<X>` | `src/` of the service | **never** — internal to the service |

The service consumes its own contracts as a `PackageReference`, like any other consumer.

## The composition test

The host's suite builds the container exactly as production does, against in-memory fake settings, with `ValidateOnBuild` and `ValidateScopes` on (see `testing-net`). It is the test that proves every `Add<Module>Context` registered something that resolves — the safety net for the whole pattern.
