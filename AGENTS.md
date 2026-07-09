# skills — working in this repo

This repo is a Claude Code **plugin marketplace** of agent skills. Each skill is a plugin under `plugins/`.

## Branching & releases — follow GitFlow

This repo **ships** the GitFlow skill at [`plugins/gitflow/SKILL.md`](plugins/gitflow/SKILL.md) — and it follows it. That file is the **source of truth**; use it for any branch/release work here. The essentials:

- New skill or edit → `feature/<name>-<ts>` from `develop` → finish `--no-ff` into `develop`.
- New version → `release/<version>` from `develop` → **bump the `version` in EVERY `plugins/*/.claude-plugin/plugin.json` to the release version** on the branch → Release Finish (merge into `develop` then `main`, annotated tag `v<version>`, atomic push).

## Versioning — lockstep

All plugins version **as a single block** (same model as shared-net's central `VersionPrefix`): there is ONE version — the release's — and every `plugins/*/.claude-plugin/plugin.json` carries it. A release bumps **all** of them, changed or not, so plugin versions == git tag == GitHub Release, and any installed skill traces back to the exact tag. Never bump a single plugin on its own.

Bump one-liner (run on the `release/<version>` branch):

```bash
find plugins -name plugin.json -path "*/.claude-plugin/*" \
  -exec sed -i 's/"version": "[^"]*"/"version": "<version>"/' {} +
```
- Use git's **default merge message** (`--no-ff --no-edit`, never `-m`).
- Commits use Jorge's **personal** identity; **no** `Co-Authored-By` / Claude / Anthropic trailer.
- Branch prefixes only: `feature` / `bugfix` / `release` / `hotfix`.

## Adding a skill

See [CONTRIBUTING.md](CONTRIBUTING.md): create `plugins/<plugin>/SKILL.md` (+ optional `plugin.json`), register it in `.claude-plugin/marketplace.json`, add a row to `README.md`.
