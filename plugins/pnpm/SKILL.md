---
name: pnpm
description: >-
  Package-management baseline for ANY Node.js work, frontend or backend — use
  when creating or configuring a Node project, adding a package.json, choosing
  a package manager, installing or updating dependencies, scaffolding frontend
  apps or JS/TS tooling, or setting up CI for a Node repo. It mandates pnpm
  with supply-chain hardening: dependency lifecycle scripts blocked (explicit
  allowlist), a minimum release age for new versions, pinned package manager,
  and frozen lockfile in CI.
---

# pnpm — supply-chain hardened baseline

Any Node project or tooling — front or back — uses **pnpm** (v10+) with this configuration. Rationale: the npm supply-chain attacks execute via install scripts of freshly published versions; this setup cuts both vectors.

## Rules

1. **pnpm, not npm/yarn** — strict `node_modules` (no phantom dependencies) and, since v10, dependency lifecycle scripts are **not executed by default**.
2. **Dependency `preinstall`/`postinstall` stay blocked.** Your own package.json scripts still run. Anything that genuinely needs a build step (esbuild, sharp…) goes in an explicit `onlyBuiltDependencies` allowlist — empty until something proves it needs to be there.
3. **`minimumReleaseAge: 4320`** (3 days, in minutes; pnpm ≥ 10.16) — never install versions published less than 3 days ago; trojanized releases are typically detected and pulled within that window. Use `minimumReleaseAgeExclude` for own/trusted packages that must land immediately.
4. **Pin the package manager**: `"packageManager": "pnpm@10.x"` in package.json (corepack) so every dev and CI run the same version.
5. **Lockfile committed; CI installs with `--frozen-lockfile`** (pnpm's default on CI) — deterministic installs, no drift.
6. **Transitive pins via pnpm `overrides`** when a vulnerable transitive needs forcing above a bad version.

## Config

`pnpm-workspace.yaml`:

```yaml
minimumReleaseAge: 4320        # 3 days, in minutes
onlyBuiltDependencies: []      # explicit allowlist; add only what proves it needs build scripts
```

`package.json`:

```json
{
  "packageManager": "pnpm@10.20.0"
}
```

## When scaffolding

- New Node project → this setup from the first commit, before any `pnpm add`.
- If an install warns about skipped build scripts, do NOT blanket-enable: check the package, then either add it to `onlyBuiltDependencies` (it really compiles something) or ignore the warning (`ignoredBuiltDependencies`).
- Never work around a blocked script with `pnpm rebuild` / manual script runs without the same scrutiny.
