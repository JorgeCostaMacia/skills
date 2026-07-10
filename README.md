<p align="center">
  <img src="https://raw.githubusercontent.com/JorgeCostaMacia/skills/main/assets/social-preview.png" width="100%" alt="skills" />
</p>

# skills

Curated agent skills and workflows for AI coding agents — installable as a Claude Code plugin marketplace ([agentskills.io](https://agentskills.io) standard).

Each skill lives as its own plugin under `plugins/`, so you can install only what you need.

## Available plugins

| Plugin | Description |
| --- | --- |
| [gitflow](plugins/gitflow) | GitFlow workflow — feature / bugfix / release / hotfix start & finish, SemVer tags, default merge messages, commit identity, and the phased-publish caveat. |
| [solid](plugins/solid) | Transversal engineering principles for any code, front or back — SOLID, simple design, clean code, code smells and complexity management, applied with judgment, not dogma. |
| [pnpm](plugins/pnpm) | Package-management baseline for any Node.js work — pnpm with dependency lifecycle scripts blocked, minimum release age for new versions, pinned package manager and frozen lockfile in CI. |
| [clean-architecture](plugins/clean-architecture) | Clean architecture layering with hexagonal (ports & adapters) boundaries, stack-agnostic and canon-anchored: the four layers inside each bounded context, the inward dependency rule, ports for external capabilities, and the name mapping to Evans, Uncle Bob, Microsoft and Jason Taylor. |
| [ddd](plugins/ddd) | Tactical Domain-Driven Design, stack-agnostic and canon-anchored: ubiquitous language, aggregates, factories & hydration, value objects, validation principles, per-aggregate data accessors, domain events and domain errors. |
| [testing](plugins/testing) | Testing principles and the pragmatic TDD loop, stack-agnostic and canon-anchored: done means tested, the test pyramid, one test file per unit, names as specification, AAA and independence, classicist test doubles, behavior over implementation, and rule coverage over line coverage. |
| [validation-net](plugins/validation-net) | The .NET realization of the two-level validation model: FluentValidation, the three-verb creation surface (ctor hydrates, From converts, Create validates), validator Create() chains, the composite full-report rule, family exceptions with fixed codes, and use-case preconditions in Application. |
| [logging-net](plugins/logging-net) | Application logging style for .NET with Serilog: fixed low-cardinality messages as grouping keys (never interpolation or placeholders, never failure details in the message), all data via LogContext.PushProperty, same message across levels for outcome pairs, and correlation ids in every scope. |

## Install

In Claude Code:

```
/plugin marketplace add JorgeCostaMacia/skills
/plugin install gitflow@jorgecostamacia-agent-skills
```

Then `/skills` to view, and invoke with `/gitflow`.

Update on demand:

```
/plugin marketplace update jorgecostamacia-agent-skills
```

## Layout

```
.claude-plugin/marketplace.json   # marketplace manifest (lists the plugins)
plugins/
  gitflow/
    .claude-plugin/plugin.json    # plugin manifest (name, version, author)
    SKILL.md                      # the skill
```

To add a new skill: create `plugins/<name>/` with its `SKILL.md` (and optional `plugin.json`), then add an entry to `marketplace.json`.

## License

[MIT](LICENSE).
