---
name: build-net
description: >-
  How a .NET repository is built, versioned and restored — the house
  conventions on top of MSBuild and NuGet. Use when creating a repo or a
  project, editing Directory.Build.props, Directory.Packages.props or a
  .csproj, adding or bumping a package, versioning a release, packing a
  library, configuring the private feed, or when a restore or a build fails
  on the audit gate. It encodes: the three-tier Directory.Build.props, one
  VersionPrefix per repo (and the separately released piece that keeps its
  own), Central Package Management, the NuGet vulnerability audit as a build
  gate, global.json, packing metadata for libraries, and a nuget.config that
  holds no credentials.
---

# Building .NET repositories

The conventions every repo follows; how MSBuild itself works is the `dotnet-msbuild` skill's. Code style and its gates are `code-style-net`'s.

## Directory.Build.props — three tiers

- **Root** — what every project shares:
  - the repo's identity (`Authors`, `Company`, `Copyright`, `RepositoryUrl`, `RepositoryType`) and the **`VersionPrefix`**;
  - `TargetFramework`, `ImplicitUsings`, `Nullable`, `AnalysisLevel latest`, `EnforceCodeStyleInBuild`, `TreatWarningsAsErrors`;
  - `IsPackable false` by default — a project packs only by saying so;
  - the vulnerability audit (below).
- **`src/`** — only where `src` packs (a library): imports the root, then the package output — `GenerateDocumentationFile`, icon, readme, license, SourceLink (`Microsoft.SourceLink.GitHub`), symbols (`snupkg`), `Deterministic`, `ContinuousIntegrationBuild` on CI, and the files packed into every package.
- **`test/`** — imports the root, then the test settings (see `testing-net`).
- Each lower tier starts with `<Import Project="$([MSBuild]::GetPathOfFileAbove('Directory.Build.props', '$(MSBuildThisFileDirectory)../'))" />` — a `Directory.Build.props` does not inherit the one above on its own.
- **A `.csproj` declares only what is its own**: references, and for a package its `Description` / `PackageTags`. Everything else is inherited — a property repeated in a project is a property that will drift.
- Comment each `PropertyGroup` with what it is for; `.props` and `.csproj` are indented with tabs.

## Versions

- **One `VersionPrefix`, in the root props**, versions what the repo delivers — the service's image, or every package of a library at once (lockstep). No project sets `<Version>`: it would override the prefix.
- **The exception: a piece released on its own cycle** — a read model packed beside a service (`<Service>.Query`) — declares its own `VersionPrefix` in its `.csproj`, so a release that only changes the service doesn't republish an unchanged package. CI guards it: a change under that project (or what it is built from) since the last tag, without a moved version, fails the build — otherwise a `--skip-duplicate` push would skip it in silence.
- SemVer: a feature is a minor, anything a consumer cannot see (tests, docs, a dependency bump) a patch, a breaking change a major. A version on the feed is never published twice.

## Central Package Management

- `Directory.Packages.props` with `ManagePackageVersionsCentrally=true`: every version — third-party, the house libraries, the contracts — is a `<PackageVersion>` there, **alphabetical**. A `.csproj` references packages **without** a version.
- Projects inside the repo reference each other with `ProjectReference`, never through CPM.
- Keep the house libraries' versions in step across repos; a stale transitive dependency is caught by the audit, not by a reviewer.

## The audit gate

```xml
<NuGetAudit>true</NuGetAudit>
<NuGetAuditMode>all</NuGetAuditMode>
<NuGetAuditLevel>low</NuGetAuditLevel>
<WarningsAsErrors>$(WarningsAsErrors);NU1903;NU1904</WarningsAsErrors>
```

Every restore audits direct **and** transitive packages and reports any severity; high and critical (`NU1903` / `NU1904`) fail the build instead of hiding in a log. The fix is a version bump (or an override of the transitive package in CPM), never `NoWarn`.

## global.json

At the root, selecting the test runner — `{ "test": { "runner": "Microsoft.Testing.Platform" } }` — which xUnit v3 on the current SDK needs for `dotnet test` to run at all (see `testing-net`). Pin an SDK there only with a reason.

## Feeds

- `nuget.config` lists the feeds — `nuget.org` and the organization's private feed — with **`packageSourceMapping`** (the organization's own package prefix to its feed, everything else to nuget.org), so a package can only come from the feed it belongs to.
- **It holds no credentials.** A developer's live in the user-level NuGet config; CI's come from a read-only token in an organization secret, passed as `NuGetPackageSourceCredentials_<FeedName>`.
- `nuget.config`, `appsettings*.json` and user secrets are never read or rewritten by tooling: hand changes to them to the user.
- Right after a package is published, a restore that can't find it is usually the HTTP cache — restore with `--no-http-cache` before suspecting the feed.
