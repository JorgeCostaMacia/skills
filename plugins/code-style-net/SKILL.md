---
name: code-style-net
description: >-
  How C# is written — the house style every .NET repo enforces. Use when
  writing, generating or reviewing any C# code: declarations, object
  creation, collections, braces, parameter and argument lists, naming,
  modifiers, null checks, comments, file encoding and line endings, or when
  a build fails on a style rule. It encodes: explicit types (no var, no
  target-typed new(), no collection expressions), braces always, one
  parameter or argument per line from two up, the naming and modifier
  conventions, `is null` outside expression trees, comments only where the
  code cannot speak, CRLF, `dotnet format` before committing, and the strict
  gates that make all of it a build error.
---

# C# style

The style is **enforced, not suggested**: `.editorconfig` + `EnforceCodeStyleInBuild` + `TreatWarningsAsErrors` turn a violation into a build error, and `dotnet format --verify-no-changes` must exit 0. What no analyzer catches is spelled out below — that part is on whoever writes the code, generators included.

## Explicit types

- **No `var`.** `PedidosAggregate aggregate = …`, `List<Command> commands = …`.
- **No target-typed `new()`** — `new PedidosExtractService(context)`, never `new(context)`. The analyzer can't flag it; a CI grep guard does:

  ```bash
  grep -rnE '\bnew\(|new\[\]' --include='*.cs' --exclude-dir=obj --exclude-dir=bin src test | grep -vE ': new\(\)'
  ```

  (`: new()` — the generic constraint — is the one allowed form.)
- **No collection expressions** `[]`, no implicit `new[] { … }`: `new List<string>()`, `new string[] { "a", "b" }`, `Array.Empty<T>()`, `ImmutableList.Create(...)`.

## Shape

- **Braces always**, opening brace on its own line.
- **A blank line before `return`** (unless it is the block's only statement).
- **One parameter or argument per line from two up** — method and constructor declarations, calls, `base(…)` (with `) : base(…)` on its own line), factory calls, every `new …Event(…)`. A one-parameter list stays on its line. No analyzer enforces it, and `dotnet format` won't wrap a long line — a generator that emits one-liners is how a repo drifts.
- Expression-bodied members (`=>`) for one-liners; a block when there is more than one statement.
- Prefer a `using` over full qualification; `global using`s only for what every file needs.

## Names

- PascalCase for types and members, **camelCase** for locals and parameters, **`_camelCase`** for private fields, `I`-prefixed interfaces.
- Lambdas: `v =>` in validators (nested `v2`), `e =>` in query predicates.
- Application locals are **`aggregate` / `aggregates`** — the word the domain models — and the arriving data is `payload`.
- Domain concepts in the business's language, technical suffixes in English (`PedidosLoadService`, `PedidosValidator`).

## Modifiers

- Messages: `public sealed record`. Value objects: `internal sealed record`. Aggregates, services, handlers, validators: `internal sealed class`. Composition contexts: `public static class` (the module's) / `internal static class` (its layers').
- Modifier order follows the `.editorconfig` (`public, private, protected, internal, file, static, … new, …, override, readonly, …, async`).

## Null and booleans

- **`is null` / `is not null`** and **`!flag`** — not `== null`, `!= null`, `== false`.
- The exception: inside an **expression tree** (an EF Core predicate) a pattern is `CS8122` — there it is `!= null`.
- Nullable reference types are on; `!` (null-forgiving) only where the code proves non-null and the compiler can't see it — and the reason is visible in the surrounding lines.

## Comments and documentation

- A comment says what the code **cannot**: a non-obvious constraint, why the obvious approach was rejected, what breaks if this changes. A comment restating the line above it is worse than none — it reads as authoritative and rots.
- No commented-out code — except a deliberately disabled registration that an operator uncomments by hand, with the comment saying when and why.
- **Doc XML only where there are outside consumers**: a library ships it (`GenerateDocumentationFile`, `CS1591` an error); an application doesn't.
- Comments, docs and commit messages in English.

## Files

- **CRLF** line endings, UTF-8, final newline. A file generated or rewritten by a shell tool is normalized before committing, or `dotnet format` flags every line.
- C# indented with 4 spaces; `.csproj` / `.props` / `.targets` with **tabs**.
- **Run `dotnet format <solution>` before every commit** — using order, whitespace, charset; `--verify-no-changes` is the check-only form CI runs.

## The gates

- `EnforceCodeStyleInBuild`, `TreatWarningsAsErrors`, `AnalysisLevel latest`, `Nullable enable` — the build has **zero warnings**, and stays that way.
- **No `NoWarn`, no `#pragma warning disable`.** A warning is fixed, or the rule is changed for the whole repo in `.editorconfig` with a reason — never silenced at one site.
