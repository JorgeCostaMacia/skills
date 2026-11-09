---
name: changelog
description: >-
  How a repository keeps its changelog: one CHANGELOG.md at the root listing,
  newest first, only what whoever consumes the repo has to act on. Use when
  cutting a release, bumping a major, changing a default, a requirement or a
  name a consumer depends on, deciding whether a change needs an entry, or when
  a README or AGENTS file starts to accumulate version history ("since 3.0…",
  "4.0.0 moved…"). It encodes what goes in and what stays out, the format, that
  the entry is written on the release branch with the version bump, and that the
  other docs describe the present and carry no history.
---

# Changelog — what a consumer has to act on

A repository that others consume — a library, a published contract, a read model, an API — keeps **one `CHANGELOG.md` at its root**. It answers one question for whoever upgrades: *what do I have to do?* Everything else about a version is already in git: the commits, the tag, the release notes the forge generates.

## What goes in

- **Every major**: what breaks and what to do about it — the renamed id, the moved topic or route, the removed field. A major that changes nothing a consumer compiles against (new topics, a new store schema) gets its entry all the more: it is the one that looks safe and is not.
- **A behavior change in a minor**: a new default, a renamed log message or metric someone may query or alert on, a new requirement on the environment (a schema migration, a runtime version), a deprecation.
- **A first release that continues something published elsewhere** (other package ids, another transport, another repository): where the old one went and how to move.

## What stays out

- Patches that touch only the docs, the build, the tests or the internals, and dependency bumps a consumer does not notice.
- Additions a consumer can take or leave without acting: a new optional field, a new optional setting. List them only if the consumers want a what's-new, and never so that what they must act on gets buried.

A version with nothing to act on has no entry. The gap is not a hole: the tag and the commits still say what it was.

## The format

```markdown
# Changelog

What a consumer has to act on, newest first. Versions that only touched the docs, the build or the internals are not listed.

## 5.0.0

Every topic moves to the new major, with fewer partitions; the messages do not change. Every producer and consumer moves to 5.x in the same round, and the 4.x topics are deleted once none is left on them.

## 4.2.0

`QueueMaxKbytes` defaults to 2048 (it was 8192). A consumer that needs the larger buffer sets it back.
```

- One `##` section per listed version, newest first, the bare version number as its heading: the date is the tag's.
- Two or three sentences in the consumer's terms: what changed, then what to do. Name the setting, the id, the topic, the column — never "various improvements".
- Concrete numbers belong here (the old value and the new, the version a thing started): this is the one document that is history on purpose.

## When it is written

- **On the release branch, together with the version bump.** The entry is part of cutting the release, not an afterthought, and a version that needs one does not ship without it.
- A change that will need an entry can say so in its own commit message; the release gathers them.

## The other docs carry no history

README, AGENTS and their kind describe the repository **as it is now**. "Since 3.0…", "4.0.0 moved…", "it used to…" belong in the changelog: a version-history paragraph grows with every release and goes stale beside the present it sits in. The current values those docs need (a default, a requirement) are written as facts, without the version that introduced them, and the docs link to `CHANGELOG.md` at most.
