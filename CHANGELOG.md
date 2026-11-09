# Changelog

What someone who installs these skills has to act on, newest first. Versions that only refined what a skill says are not listed.

## 2.0.0

Every .NET skill is renamed from `-net` to `-dotnet`, like the library repositories (`build-net` is `build-dotnet`, `ddd-net` is `ddd-dotnet`, `bus-net` is `bus-dotnet`, `bus-kafka-net` is `bus-kafka-dotnet`…; the stack-agnostic skills keep their names). Uninstall the `-net` plugins and install the `-dotnet` ones. The skills now point at the libraries' new repositories and package ids: the domain building blocks are `JorgeCostaMacia.DDD.*` (ddd-dotnet), and Quartz, Serilog and each bus transport have a repository of their own (quartz-dotnet, serilog-dotnet, bus-dotnet, bus-kafka-dotnet, bus-rabbitmq-dotnet).
