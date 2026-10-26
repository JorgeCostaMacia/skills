# Contributing

This repo is a Claude Code **plugin marketplace** that hosts agent skills. Each skill is its own plugin under `plugins/`.

## Add a new skill

1. Create the plugin folder and skill:

   ```
   plugins/<plugin>/
     .claude-plugin/plugin.json     # name, description, version (the release's — every plugin carries it), author, dependencies
     SKILL.md                       # the skill (frontmatter + instructions)
   ```

   A `-net` skill declares `"dependencies": ["<generic skill>"]` so installing it enables the principles it builds on; a generic skill declares none and never names a `-net` one.

2. `SKILL.md` frontmatter — at minimum a `description` (and `name` for a plugin-root skill). The `description` is what tells the agent *when* to use the skill, so make it specific. Keep it under ~1,500 chars.

3. Register the plugin in `.claude-plugin/marketplace.json`, with the same `description` as its `plugin.json` (the generic skill first, then its `-net` realization):

   ```json
   {
     "name": "<plugin>",
     "source": "./plugins/<plugin>",
     "description": "..."
   }
   ```

4. Add a row to the table in `README.md`.

## Conventions

- Identifiers (`name`) in **kebab-case**.
- Markdown/JSON: UTF-8, LF, 2-space indent (see `.editorconfig`).
- Keep each `SKILL.md` focused and actionable; prefer concrete steps over prose.

## Test locally

```
/plugin marketplace add JorgeCostaMacia/skills
/plugin install <plugin>@jorgecostamacia-agent-skills
```
