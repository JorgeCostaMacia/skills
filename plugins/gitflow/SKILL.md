---
name: gitflow
description: >-
  A GitFlow workflow for branches and releases. Use this whenever the work
  touches branching or releasing in a repo that follows it — starting or finishing a
  feature, bugfix, release, or hotfix; creating a release/hotfix branch; bumping a
  version; tagging (`vX.Y.Z`); merging into develop or main; or publishing a new
  version — even when the user doesn't say "gitflow" explicitly (e.g. "cut a release",
  "ship 2.1.0", "merge this to main", "tag the version", "start a feature for X").
  It encodes branch naming and validation, the transactional Release/Hotfix Finish
  (merge develop+main, annotated tag, atomic push), default merge messages, commit
  identity, and the phased-publish caveat — so releases follow the flow and are
  never improvised.
---

# GitFlow

This mirrors the [`gitflow-vscode`](https://github.com/JorgeCostaMacia/gitflow-vscode) extension; when in doubt, its source is the reference. Follow it **from the start** of any branching/release work. Never improvise direct `develop→main` merges or bolt a tag on at the end.

## Branch model

| Type | Branch | Starts from | Merges into | Tagged |
|---|---|---|---|---|
| Feature | `feature/<name>-<timestamp>` | develop | develop | no |
| Bugfix | `bugfix/<name>-<timestamp>` | develop | develop | no |
| Release | `release/<version>` | develop | develop **and** main | `v<version>` |
| Hotfix | `hotfix/<version>` | main | develop **and** main | `v<version>` |

Branch prefixes are **only** `feature` / `bugfix` / `release` / `hotfix`. Never use `chore/` or anything else as a branch prefix (`chore` is a Conventional-Commits *message* type, not a GitFlow branch). A hotfix merges into `develop` even while a release branch is open (the original model would merge it into that release instead): that is what the extension does; an open release branch that needs the fix merges `develop` into itself by hand.

## Validation (reject, don't sanitize)

- **Names** (feature/bugfix): must start with a lowercase letter/digit, then lowercase word chars and `. _ - /`. Reject uppercase, spaces, accents — don't rewrite them.
- **Versions** (release/hotfix): strict 3-part SemVer (`1.0.0`, optionally `1.0.0-rc1`). The version bump lives **on the `release/X` branch**, not after merging.
- **Tag** = `v` + version (`2.0.4` → `v2.0.4`). Annotated tags, always created with `--cleanup=verbatim` (git would otherwise strip `#` lines — markdown headings — as comments). A tag that already exists, locally or on origin, means the version was used: pick another, never reuse it.

## Procedures

Use `git merge --no-ff --no-edit` so the **default** merge message is kept (`Merge branch '<x>'`) — never pass `-m`. This keeps GitHub Actions run titles uniform. A merge that conflicts is aborted (`git merge --abort`), leaving the repository as it was. The main branch is `main`, or `master` where that is the one that exists.

### Feature / Bugfix start
1. Validate `<name>`.
2. Working tree clean. `git checkout develop && git pull`.
3. `git checkout -b feature/<name>-<ts>` (timestamp `<ts>` = `YYYYMMDDHHmm`, local time; `bugfix/` for bugfix).
4. Publish for backup: `git push -u origin <branch>` (see "Pushing" below).

### Feature / Bugfix finish
1. Working tree clean; branch exists.
2. `git checkout develop && git pull`
3. `git merge --no-ff --no-edit <branch>`
4. Push `develop`.
5. Delete the branch, **remote first** (`git push origin --delete <branch>`), then local (`git branch -d <branch>`): removing the remote first drops the upstream tracking ref, so `-d` does not refuse the local branch as "not merged to its upstream".

### Release start `<version>` / Hotfix start `<version>`
1. Validate SemVer.
2. Working tree clean. Release: `git checkout develop && git pull && git checkout -b release/<version>`. Hotfix: the same from `main`.
3. `git push -u origin <branch>`.
4. **Bump the package/project version(s) on this branch** and commit. The extension does not bump — this step is yours.

### Release / Hotfix finish (transactional — develop first)
1. Working tree clean, and `v<version>` exists neither locally (`git tag -l v<version>`) nor on origin (`git ls-remote --tags origin v<version>`) — checked before anything is merged. `git pull` on `main` and on `develop`, then capture their commits (for rollback).
2. `git checkout develop && git merge --no-ff --no-edit <branch>`
3. `git checkout main && git merge --no-ff --no-edit <branch>`
4. `git tag -a v<version> main --cleanup=verbatim -m "Release v<version>"` (`Hotfix v<version>` for a hotfix; richer notes are fine).
5. **Atomic push** so the remote updates all refs or none: `git push --atomic origin main develop --tags` (keep the tag on the remote).
6. Delete the branch, remote first, then local (as in the feature finish).
7. Leave the repository on `develop`.
8. **On any failure** (a conflict, a rejected push): abort the merge, delete the tag if it was created, and **revert** main/develop to the commits captured in step 1. Never leave a half-merged main.

## Pushing

The `gitflow-vscode` extension does the pushes itself (start: `push -u`; finish: atomic). When **you (the agent) cannot push** (push is blocked in some repos), do every **local** step, then **stop before the push** and give the user the exact command(s) to run (`git push -u origin <branch>`, or `git push --atomic origin main develop --tags`). Do the local work; the user pushes.

## Commit identity

Commit with **the identity git already has** — don't set one in the command. If one has to be given explicitly, take the repository's own (`git config user.name` / `git config user.email` run inside it) and use that — a repository can set its own identity (`git config --local`) over the global one. **No** `Co-Authored-By` / Claude / Anthropic trailer in any commit or message.

## Phased-publish caveat (the one real reason to deviate)

A release sometimes **cannot** fit a single `release/X` branch: when packages depend on each other through a registry (NuGet, npm…), a dependent can only build against an **already-published** version. So a tiered library (e.g. tier0 → tier1 → top) must publish in **separate pushes**, waiting for each tier to index on the registry before the next references it — which one release branch can't span in time.

If you hit this, **raise it with the user BEFORE proceeding** and agree the plan. Do **not** silently switch to ad-hoc per-tier `develop→main` merges (that exact deviation, in a real package rollout, is what this skill exists to prevent). When phased, the milestone tag still lands on `main` at the end via a normal annotated tag + atomic push.
