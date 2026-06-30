---
name: gitflow
description: >-
  Jorge's GitFlow workflow for branches and releases. Use this whenever the work
  touches branching or releasing in any of Jorge's repos — starting or finishing a
  feature, bugfix, release, or hotfix; creating a release/hotfix branch; bumping a
  version; tagging (`vX.Y.Z`); merging into develop or main; or publishing a new
  version — even when the user doesn't say "gitflow" explicitly (e.g. "cut a release",
  "ship 2.1.0", "merge this to main", "tag the version", "start a feature for X").
  It encodes branch naming and validation, the transactional Release/Hotfix Finish
  (merge develop+main, annotated tag, atomic push), default merge messages, commit
  identity, and the phased-publish caveat — so releases follow the flow and are
  never improvised.
---

# GitFlow (Jorge's flow)

This mirrors Jorge's `gitflow-vscode` extension. Follow it **from the start** of any branching/release work. Never improvise direct `develop→main` merges or bolt a tag on at the end.

## Branch model

| Type | Branch | Starts from | Merges into | Tagged |
|---|---|---|---|---|
| Feature | `feature/<name>-<timestamp>` | develop | develop | no |
| Bugfix | `bugfix/<name>-<timestamp>` | develop | develop | no |
| Release | `release/<version>` | develop | develop **and** main | `v<version>` |
| Hotfix | `hotfix/<version>` | main | develop **and** main | `v<version>` |

Branch prefixes are **only** `feature` / `bugfix` / `release` / `hotfix`. Never use `chore/` or anything else as a branch prefix (`chore` is a Conventional-Commits *message* type, not a GitFlow branch).

## Validation (reject, don't sanitize)

- **Names** (feature/bugfix): must start with a lowercase letter/digit, then lowercase word chars and `. _ - /`. Reject uppercase, spaces, accents — don't rewrite them.
- **Versions** (release/hotfix): strict 3-part SemVer (`1.0.0`, optionally `1.0.0-rc1`). The version bump lives **on the `release/X` branch**, not after merging.
- **Tag** = `v` + version (`2.0.4` → `v2.0.4`). Annotated tags. If the tag message contains markdown `#` headings, create it with `--cleanup=verbatim` (git strips `#` lines as comments otherwise).

## Procedures

Use `git merge --no-ff --no-edit` so the **default** merge message is kept (`Merge branch '<x>'`) — never pass `-m`. This keeps GitHub Actions run titles uniform.

### Feature / Bugfix start
1. Validate `<name>`.
2. `git checkout -b feature/<name>-<ts> develop` (timestamp `<ts>` = `YYYYMMDDHHmm`; `bugfix/` for bugfix).
3. Publish for backup: `git push -u origin <branch>` (see "Pushing" below).

### Feature / Bugfix finish
1. Working tree clean; branch exists.
2. `git checkout develop`
3. `git merge --no-ff --no-edit <branch>`
4. Delete: `git branch -d <branch>` and remote `git push origin --delete <branch>`.
5. Push `develop`.

### Release start `<version>` / Hotfix start `<version>`
1. Validate SemVer.
2. Release: `git checkout -b release/<version> develop`. Hotfix: `git checkout -b hotfix/<version> main`.
3. **Bump the package/project version(s) on this branch** and commit.
4. `git push -u origin <branch>`.

### Release / Hotfix finish (transactional — develop first)
1. Working tree clean. Capture the current commits of `main` and `develop` first (for rollback).
2. `git checkout develop && git merge --no-ff --no-edit <branch>`
3. `git checkout main && git merge --no-ff --no-edit <branch>`
4. `git tag -a v<version> main -m "<notes>"` (add `--cleanup=verbatim` if notes have `#` headings).
5. **Atomic push** so the remote updates all refs or none: `git push --atomic origin main develop --tags` (keep the tag on the remote).
6. Delete the branch local + remote.
7. **On merge conflict**: abort and **revert** main/develop to the commits captured in step 1. Never leave a half-merged main.

## Pushing

Jorge's `gitflow-vscode` tool does the pushes itself (start: `push -u`; finish: atomic). When **you (the agent) cannot push** (push is blocked in some of his repos — e.g. shared-net), do every **local** step, then **stop before the push** and give Jorge the exact command(s) to run (`git push -u origin <branch>`, or `git push --atomic origin main develop --tags`). Do the local work; he pushes.

## Commit identity

Commits use Jorge's **personal** identity (`Jorge Costa Maciá <costamaciajorge@gmail.com>`), **never** a work account. **No** `Co-Authored-By` / Claude / Anthropic trailer in any commit or message.

## Phased-publish caveat (the one real reason to deviate)

A release sometimes **cannot** fit a single `release/X` branch: when packages reference each other via `PackageReference` (NuGet), a dependent can only build against an **already-published** version. So a tiered library (e.g. tier0 → tier1 → top) must publish in **separate pushes**, waiting for each tier to index on the registry before the next references it — which one release branch can't span in time.

If you hit this, **raise it with Jorge BEFORE proceeding** and agree the plan. Do **not** silently switch to ad-hoc per-tier `develop→main` merges (that exact deviation, in a real NuGet rollout, is what this skill exists to prevent). When phased, the milestone tag still lands on `main` at the end via a normal annotated tag + atomic push.
