# skills — working in this repo

This repo is a Claude Code **plugin marketplace** of agent skills. Each skill is a plugin under `plugins/`.

## Branching & releases — follow GitFlow

This repo **ships** the GitFlow skill at [`plugins/gitflow/SKILL.md`](plugins/gitflow/SKILL.md) — and it follows it. That file is the **source of truth**; use it for any branch/release work here. The essentials:

- New skill or edit → `feature/<name>-<ts>` from `develop` → finish `--no-ff` into `develop`.
- New version → `release/<version>` from `develop` → bump the relevant `version` (plugin.json / marketplace) on the branch → Release Finish (merge into `develop` then `main`, annotated tag `v<version>`, atomic push).
- Use git's **default merge message** (`--no-ff --no-edit`, never `-m`).
- Commits use Jorge's **personal** identity; **no** `Co-Authored-By` / Claude / Anthropic trailer.
- Branch prefixes only: `feature` / `bugfix` / `release` / `hotfix`.

## Adding a skill

See [CONTRIBUTING.md](CONTRIBUTING.md): create `plugins/<plugin>/SKILL.md` (+ optional `plugin.json`), register it in `.claude-plugin/marketplace.json`, add a row to `README.md`.
