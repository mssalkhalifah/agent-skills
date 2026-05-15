---
name: writing-commit-messages
description: Use whenever drafting, editing, fixing, amending, or reviewing a git commit message — or a PR title that will be squash-merged into main. Triggers on phrases like "commit this", "what's a good commit message for…", "fix the commit message", "rewrite this commit", as well as before running `git commit`, `git commit --amend`, or `gh pr create`. Produces Conventional Commits 1.0.0 output (type, optional scope, optional !, description, optional body, optional footers) so CHANGELOG generation, SemVer bumps, and commitlint stay correct.
---

# Conventional Commits

Every commit message must follow [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/). The format is machine-parseable so it can drive CHANGELOG generation, SemVer bumps, and release tooling — that's why the rules below exist; they're not arbitrary.

## Read this first

For the common case (one-line summary, no body, no footers), the **Quick start** below + the **Quick reference** table are enough. Skim the rest only when you hit edge cases (breaking changes, scope ambiguity, footers, multi-paragraph bodies).

Before drafting, look at what's actually being committed (`git diff --staged`, not the working tree) so the type and scope match the staged change — not whatever else you happen to be editing.

## Quick start

Skeleton — fill in the brackets, drop the parts you don't need:

```
<type>(<scope>)<!>: <imperative description, lowercase, no period>

<body — explain *why*, wrap ~72 chars, optional>

<Footer-Token: value>
<BREAKING CHANGE: required if you broke something and didn't use !>
```

Worked example:

```
feat(api)!: drop legacy v1 endpoint

The v1 endpoint was deprecated in 2.3 and telemetry shows under 0.1%
of traffic still uses it. Removing it simplifies the rate limiter
and unblocks the v3 migration.

Closes: #482
BREAKING CHANGE: clients on /v1 must migrate to /v2 (see UPGRADE.md).
```

## Quick reference

| Need                                          | Prefix                                            |
|-----------------------------------------------|---------------------------------------------------|
| New user-facing feature                       | `feat:` (MINOR)                                   |
| User-facing bug fix                           | `fix:` (PATCH)                                    |
| Breaking API / DTO / schema / config          | `feat!:` **or** `BREAKING CHANGE:` footer (MAJOR) |
| Performance improvement                       | `perf:` (PATCH)                                   |
| Refactor with no behaviour change             | `refactor:` (no bump)                             |
| Tests only                                    | `test:` (no bump)                                 |
| Docs / README / comments only                 | `docs:` (no bump)                                 |
| Build, packaging, Dockerfiles                 | `build:` (no bump)                                |
| CI config (Actions, pipelines, hooks)         | `ci:` (no bump)                                   |
| Formatting, whitespace                        | `style:` (no bump)                                |
| Reverting a previous commit                   | `revert:` (depends on what was reverted)          |
| Maintenance, deps bumps, repo hygiene         | `chore:` (no bump)                                |

## Format

```
<type>[optional scope][!]: <description>

[optional body]

[optional footer(s)]
```

- **type** — required noun (lowercase). See the type vocabulary below.
- **scope** — optional noun in parentheses naming the section of the codebase, e.g. `feat(api):`, `fix(parser):`, `chore(deps):`.
- **!** — optional marker before the colon to flag a breaking change, e.g. `feat!:` or `feat(api)!:`.
- **description** — required short summary, imperative mood, no trailing period, lowercase first word (unless a proper noun).
- **body** — optional, separated from the description by exactly one blank line. Free-form, may span multiple paragraphs. Explain *why*, not *what*.
- **footers** — optional, separated from the body by one blank line. Each footer is `Token: value` or `Token #value`. Tokens use `-` instead of spaces (e.g. `Reviewed-by`, `Refs`, `Co-authored-by`). `BREAKING CHANGE` is the one allowed exception with a space.

## Type vocabulary (use these)

| Type       | When to use                                                                | SemVer   |
|------------|----------------------------------------------------------------------------|----------|
| `feat`     | A new feature for the user.                                                | MINOR    |
| `fix`      | A bug fix.                                                                 | PATCH    |
| `docs`     | Documentation-only changes (README, CLAUDE.md, code comments, ADRs).       | —        |
| `style`    | Formatting, whitespace, missing semicolons — no code-meaning changes.       | —        |
| `refactor` | Code change that neither fixes a bug nor adds a feature.                   | —        |
| `perf`     | Performance improvement.                                                   | PATCH    |
| `test`     | Adding or correcting tests.                                                | —        |
| `build`    | Build system, packaging, NuGet/CPM, Dockerfiles, npm scripts.              | —        |
| `ci`       | CI configuration (GitHub Actions, pipelines, hooks).                        | —        |
| `chore`    | Maintenance that doesn't fit elsewhere (deps bumps, repo hygiene).         | —        |
| `revert`   | Reverts a previous commit. Body should reference the reverted SHA.          | depends  |

`BREAKING CHANGE` in any commit type bumps **MAJOR** regardless of the prefix.

## Scope conventions

A scope is a single noun in parentheses naming the section of the codebase the change belongs to. Pick the **smallest** unit that still reads cleanly — typically a feature folder, package, or top-level area.

- One word; lowercase; hyphens for multi-word (`feat(user-auth):`, not `feat(userAuth):`).
- Omit the scope when the change is genuinely cross-cutting (e.g. `chore: bump Node 22`).
- The scope vocabulary is **project-specific**. Projects should pin their allowed scopes in `CLAUDE.md` / `AGENTS.md` (or a `commitlint` config); this skill encodes only the format.

## Breaking changes

Two equivalent ways to flag a breaking change — pick **one**, don't double up unless you want both for emphasis:

1. **Bang in the prefix**: `feat(api)!: drop legacy v1 endpoint`. Description alone explains the break.
2. **Footer**: regular prefix plus a `BREAKING CHANGE: <description>` footer. The token MUST be uppercase. `BREAKING-CHANGE:` is also allowed as a synonym.

When the public API, DTO shape, configuration schema, or DB schema migration is non-additive, mark it breaking.

## Description rules

- **Imperative mood**: "add", "fix", "remove" — not "added", "fixes", "removing". Mnemonic: the description should read naturally after "this commit will ___". *"this commit will **add** bulk import endpoint"* ✅ — *"this commit will **added** bulk import endpoint"* ❌.
- Lowercase first letter (proper nouns excepted).
- No trailing period.
- Single line. The description doesn't wrap — multi-line content belongs in the body.
- Keep it short enough to read cleanly in `git log --oneline` (~50 chars is the Tim Pope convention; the Conventional Commits spec itself sets no length limit).
- State the *what* concisely; leave the *why* for the body.

✅ `fix(checkout): prevent duplicate submit on slow networks`
❌ `Fixed bug.` / `fix: Updated the form to handle a case where it submits twice when the network is slow.`

## Body rules

- One blank line between description and body.
- Wrap lines for readability in `git log` (~72 chars is community convention; not part of the Conventional Commits spec).
- Explain motivation, contrast with previous behaviour, link incidents.
- Free-form paragraphs; bullet lists are fine.

## Footer rules

- One blank line between body (or description, if no body) and footers.
- Each footer is `<token>: <value>` **or** `<token> #<value>` (the `#` form is convenient for issue references).
- Token format: a word with hyphens replacing spaces (e.g. `Reviewed-by`, `Co-authored-by`).
- `BREAKING CHANGE` (uppercase, with a space) is the only token allowed to contain whitespace. `BREAKING-CHANGE` is an accepted synonym.
- Multiple `BREAKING CHANGE:` footers are allowed in a single commit.
- Common tokens:
  - `Refs: #123` / `Closes: #123` — issue links.
  - `Reviewed-by: Name <email>` — reviewers.
  - `Co-authored-by: Name <email>` — pair / AI co-authors. Use the same token regardless of whether the co-author is a human or an AI assistant; `git` and GitHub both render it the same way.
  - `Signed-off-by: Name <email>` — for projects using the [Developer Certificate of Origin](https://developercertificate.org/) (`git commit -s`).

## Examples

### Minimal
```
docs: correct spelling of CHANGELOG
```

### With scope
```
feat(api): add bulk import endpoint
```

### With body
```
fix: prevent racing of requests

Introduce a request id and a reference to the latest request.
Dismiss incoming responses other than from the latest request.

Remove timeouts that were used to mitigate the racing issue but are
obsolete now.
```

### Breaking change (via `!` — preferred for short summaries)
```
feat(api)!: drop legacy v1 endpoint
```

### Breaking change (via footer — when you need to explain)
```
feat: allow provided config object to extend other configs

BREAKING CHANGE: `extends` key in config file is now used for extending
other config files.
```

### Multi-paragraph body with footers
```
fix(checkout): prevent racing of submit handlers

Introduce a submission id and a reference to the latest submission.
Dismiss incoming responses other than from the latest submission.

Reviewed-by: Z
Refs: #123
```

### Revert
```
revert: add bulk import endpoint

This reverts commit 9f2e1a7c. The endpoint shipped without the
required permission check; reverting until that lands.

Refs: #456
```

> Conventional Commits doesn't standardise revert syntax — it just lists `revert` as a community-used type. The form above matches what `git revert` produces by default. **Don't** nest the original prefix inside the subject (`revert: feat(api): …`); most parsers won't recognise it.

## Pre-commit checklist

Before writing the commit message:

1. Look at the staged diff (`git diff --staged`) — not the working tree. The message describes what's about to be committed.
2. Pick the **one** type that best describes the dominant change. If you can't pick one, the commit is doing too much — split it.
3. Decide if a scope adds clarity; omit it if not.
4. Write the description in the imperative, short enough for `git log --oneline`.
5. Add a body if the *why* isn't obvious from the diff.
6. Flag breaking changes with `!` **or** a `BREAKING CHANGE:` footer (not both, unless you want emphasis).
7. Add `Refs:` / `Closes:` footers for any tracked issue.

## What to avoid

These are bad outcomes — commit messages that should never ship.

- ❌ `wip`, `update`, `misc`, `stuff` — pick a real type and description.
- ❌ Multiple unrelated changes in one commit (split them).
- ❌ Past tense or sentence-case descriptions.
- ❌ Trailing periods on the description line.
- ❌ Lowercase `breaking change:` — the token MUST be uppercase.
- ❌ Inventing types outside the vocabulary above without a reason.

## Red flags — stop and rewrite

These are warning signs you'll catch *while drafting* — the corresponding bad-outcome lives in **What to avoid** above. If any of these fire, fix the message (or split the commit) before committing.

- The diff doesn't fit one type → split the commit.
- Reaching for `chore` because nothing else "obviously" fits → re-read the diff; usually it's `refactor`, `build`, `ci`, or `docs`.
- Wrote past tense ("added", "fixed") → switch to imperative.
- Description ends in a period or starts with a capital → trim.
- A public API, DTO, config schema, or DB schema went non-additive but no `!` and no `BREAKING CHANGE:` footer → add one.
- Tempted to write `wip:` or `update:` to commit faster → finish the change or split it; never ship `wip` to a shared branch.

## When NOT to use

- **Squash-merge bots** that overwrite the message at merge time — write the **PR title** in conventional format instead; the per-commit messages on the feature branch don't reach the main history.
- **`fixup!` / `squash!` commits** intended for `git rebase --autosquash` — leave git's prefix alone; they get folded into the original commit and disappear.
- **Vendor-generated merge commits** (`Merge pull request #X from …`) — don't reformat them.
