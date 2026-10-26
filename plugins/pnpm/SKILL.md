---
name: pnpm
description: >-
  Package-management baseline for ANY Node.js work, frontend or backend — use
  when creating or configuring a Node project, adding a package.json, choosing
  a package manager, installing or updating dependencies (pnpm itself
  included), scaffolding frontend apps or JS/TS tooling, or setting up CI for
  a Node repo. It mandates pnpm 12 with supply-chain hardening: dependency
  build scripts denied unless reviewed (allowBuilds), a 3-day minimum release
  age enforced strictly, no exotic transitive sources, pinned package manager,
  and frozen lockfile in CI.
---

# pnpm — supply-chain hardened baseline

Any Node project or tooling — front or back — uses **pnpm 12** with this configuration. Rationale: npm supply-chain attacks run through the install scripts of freshly published versions; this setup cuts both vectors.

## Rules

1. **pnpm, not npm/yarn** — strict `node_modules` (no phantom dependencies), and dependency build scripts **never run unreviewed**.
2. **Build scripts go through `allowBuilds`**, a map of package → `true` (may run its scripts) / `false` (reviewed, denied). With the default `strictDepBuilds: true`, an install that meets an unlisted package with build scripts **fails** — that is the review prompt, not a nuisance. Allow only what genuinely compiles or fetches a binary (esbuild, sharp…); deny the rest explicitly. Your own `package.json` scripts are not affected. (`onlyBuiltDependencies`, `neverBuiltDependencies`, `ignoredBuiltDependencies` and `onlyBuiltDependenciesFile` were removed in v11; `pnpx codemod run pnpm-v10-to-v11` migrates them.)
3. **`minimumReleaseAge: 4320`** (3 days, in minutes) — never install a version published less than 3 days ago; trojanized releases are typically detected and pulled within that window. pnpm's own default since v11 is 1440 (1 day); set it explicitly, which also turns on `minimumReleaseAgeStrict` (no version in range old enough → resolution fails instead of silently taking a newer one). Own/trusted packages that must land immediately go in `minimumReleaseAgeExclude`.
4. **No exotic transitive sources**: keep `blockExoticSubdeps: true` (the default) — only the root `package.json` may point at a git repo or a tarball URL.
5. **Pin the package manager**: `"packageManager": "pnpm@<exact version>"` in package.json, so every dev and CI run the same pnpm. Moving the pin is a dependency update like any other: pick a release at least 3 days old — `pnpm self-update` does **not** apply the repo's `minimumReleaseAge`.
6. **Lockfile committed; CI installs with `--frozen-lockfile`** (the default on CI). pnpm 12 dropped `--frozen-lockfile false`: write `--no-frozen-lockfile`.
7. **Transitive pins via `overrides`** when a vulnerable transitive needs forcing above a bad version.
8. **Settings live in `pnpm-workspace.yaml`**, where pnpm 12 checks them: an unknown key warns with the closest real name and, when the pinned pnpm is the one running, fails with `ERR_PNPM_UNRECOGNIZED_WORKSPACE_SETTINGS` — a typo in a security setting no longer passes in silence.

## Config

`pnpm-workspace.yaml`:

```yaml
minimumReleaseAge: 4320   # 3 days, in minutes; explicit, so minimumReleaseAgeStrict is on
allowBuilds:              # reviewed packages only: true runs its build scripts, false denies them
  esbuild: true           # fetches its native binary
```

`package.json`:

```json
{
  "packageManager": "pnpm@12.9.1"
}
```

Optional, stricter: `trustPolicy: no-downgrade` fails an install when a package's trust evidence (provenance, trusted publishing) drops below what its earlier releases had — the signature of a hijacked maintainer account. Exceptions go in `trustPolicyExclude` (`'pkg@1.2.3'`).

## When scaffolding or updating

- New Node project → this setup from the first commit, before any `pnpm add`.
- An install that stops on unreviewed build scripts: look at what the package's script does, then add it to `allowBuilds` as `true` (it really builds something) or `false` (it does not need to). Never `dangerouslyAllowAllBuilds`.
- Never work around a blocked script with `pnpm rebuild` / manual script runs without the same scrutiny.
- `pnpm outdated` lists what can move; an update still lands only versions older than `minimumReleaseAge`.
