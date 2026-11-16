---
name: code-style-md
description: >-
  How Markdown is written in every repository — README, AGENTS, CONTRIBUTING,
  CHANGELOG, SECURITY, a skill's SKILL.md, any .md. Use when writing, editing
  or reviewing a Markdown file, or when a doc's lines are wrapped by hand, its
  code blocks carry no language, its links point at a file of the same repo by
  URL, or bold is everywhere. It encodes how a doc must read, whatever the
  stack and its tools: one paragraph per line (never wrapped by hand, no forced
  breaks), no trailing whitespace, code spans for everything literal and bold
  kept rare, fenced code blocks with their language and commands without a
  prompt, relative links inside the repo, tables small and for comparisons
  only, headings, alerts, images and diagrams, HTML only where Markdown cannot,
  and reflowing an existing repo once.
---

# Markdown style

Markdown is what every repository is read through: on the forge, in an editor, and quoted into Releases, issues and pull requests. One way of writing it, the same in every repo, so a doc never has to be reflowed to be read and a diff shows what changed. What a repo's docs hold — which files, and what goes in each — is not this skill's; this one is how any `.md` is written. It says how a doc must read, not which tool gets it there: where the stack runs a formatter, its baseline configures it to keep these rules; where none runs, they are followed by hand.

## One paragraph per line

- **A paragraph, a list item or a table cell is one line**, however long: never wrapped by hand at a column. The editor wraps it on screen.
- Why: a Markdown file renders the same either way (a single line break inside a paragraph is a space), but a hand-wrapped paragraph is reflowed on every edit, so changing one word shows as a whole changed paragraph in the diff. And the same text copied where a line break is kept — a GitHub Release's notes, an issue, a pull request body — breaks mid-sentence. The `changelog` skill's sections are exactly that.
- **No forced line breaks**: no two trailing spaces, no backslash, no `<br>`. A paragraph ends with a blank line; what needs a line of its own is a list item.
- A column limit (120 in C#, TypeScript and YAML) is for code, not for prose.

## Whitespace

- **No trailing whitespace in `.md` either**: with no forced breaks there is nothing to keep, so the repo's `.editorconfig` trims it in Markdown as everywhere, with no `trim_trailing_whitespace = false` section for `*.md`.
- LF line endings and a final newline, as the rest of the repo.

## Inline

- **Code spans for everything literal**: a file or folder name, a command, a variable, a setting, a value, a type or method, a package (`DOCKER_PORTS`, `pnpm test`, `appsettings.json`, `--no-ff`). Whatever the reader would type or search for goes in backticks.
- **Bold is rare**: the one idea of a paragraph or a list item that the reader must not miss, at most once per item. Bold on every line marks nothing. No italics for emphasis, no capitals.
- **Links inside the repo are relative** (`[CONTRIBUTING.md](CONTRIBUTING.md)`, `[the workflow](.github/workflows/x.yml)`): they work on the forge, in a fork, in a clone and in any branch. An absolute URL is for what lives elsewhere — another repo, a badge, an image a package page must show. The link text says where it goes, never "here".

## Blocks

- **Fenced code blocks always carry their language** (`bash`, `powershell`, `yaml`, `json`, `csharp`, `ts`, `text`), so they are highlighted and the reader knows where to run them. A command is written to be copied: no `$` or `>` prompt, and its output, if shown, in a block of its own.
- **Lists**: steps in order are a numbered list, a set of points a bulleted one (`-`). One idea per item.
- **Tables are small and for comparisons** (options against criteria, items against the same fields). A formatter aligns their columns, so one changed cell can re-pad the whole table in the diff: a table that keeps growing becomes a list.
- **Alerts** for what must not be skipped: `> [!NOTE]`, `> [!TIP]`, `> [!IMPORTANT]`, `> [!WARNING]`, `> [!CAUTION]`, which GitHub renders as callouts, instead of bold or capitals.
- **Diagrams in a ` ```mermaid ` block** rather than an image: the forge renders it, and a change to it is a readable diff. An image always has alt text that says what it shows.
- **HTML only where Markdown cannot** (a centered banner, an image's width). Never `<br>`, `<b>`, `<i>` or a `<table>` that Markdown could write.

## Headings

- One `#` heading per file, its title; sections with `##` and below, never skipping a level.
- **Sentence case** (`## What it does`, `## The design, and why`), no final period, and unique within the file: a heading is also its anchor, and a link to `#versioning` must have one target.

## Bringing a repo in line

A repo whose docs are wrapped by hand is reflowed once, in a feature branch of its own, as a commit that does nothing else, with whatever the stack has; its `.editorconfig` loses any `trim_trailing_whitespace = false` section for Markdown in the same commit. Review the diff: only the source's layout may change, never what the doc renders. What no tool judges — bold, code spans, links, headings — is fixed by hand when a file is touched.
