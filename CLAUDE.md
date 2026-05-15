# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A home for Claude Code **skills**, structured as a marketplace of plugins that also publishes to npm for `npx` installation. Three skills ship today:

- `abp-backend` — DDD layering, application services, repositories, EF Core, modules, multi-tenancy, authz, validation, infrastructure, CLI/Studio/Suite, integration tests.
- `abp-frontend` — MVC/Razor, Blazor (Server/WASM/WebApp/MAUI Blazor), Angular, React Native (Expo), .NET MAUI, LeptonX themes, virtual-file-system overrides, dynamic vs static service proxies.
- `writing-commit-messages` — Conventional Commits 1.0.0 authoring for commit messages and squash-merge PR titles.

There is no application code in this repo. "Work" here means **authoring or revising the skills themselves**, maintaining the marketplace/npm wrappers, and producing benchmark artefacts.

## Layout

```
abp-skills/
├── .claude-plugin/
│   └── marketplace.json         # marketplace manifest — lists every plugin in plugins/
├── plugins/                     # source of truth for every shipped skill
│   ├── abp-backend/
│   │   ├── .claude-plugin/plugin.json
│   │   └── skills/abp-backend/
│   │       ├── SKILL.md         # entry point — frontmatter + guardrails + routing table
│   │       ├── references/      # focused, lazily loaded topic pages
│   │       │   ├── framework/   # ddd, modularity, multi-tenancy, authz, validation, api, ef-core, fundamentals
│   │       │   ├── infrastructure/ # background processing, blob storing, eventing, caching+locking
│   │       │   ├── modules/     # identity/auth, saas, cms, operations, messaging/payment, ai/misc
│   │       │   ├── templates/   # single-layer, layered, microservice + overview
│   │       │   ├── tools/       # cli, studio, suite
│   │       │   └── testing.md
│   │       ├── customization/   # sidecar contract docs (README.md, example.md)
│   │       └── evals/evals.json # eval suite — prompts + assertions
│   ├── abp-frontend/            # same shape; references/ui/ + customization/
│   └── writing-commit-messages/ # single-file SKILL.md
├── benchmarks/                  # benchmark/iteration artefacts — NOT skills
│   └── abp-backend/
│       ├── skill-snapshot/      # frozen copy of the skill at the time of the run
│       ├── iteration-1/         # per-eval outputs + benchmark.{json,md} + feedback.json
│       ├── iteration-1.5/       # with_skill_plus_competitors comparison
│       ├── iteration-2/
│       └── iteration-2-vs-official/ # head-to-head vs the 18 official ABP skills
└── .claude/settings.json        # enables the superpowers plugin for this repo
```

The `benchmarks/<skill>/` directory mirrors the eval ids in `plugins/<plugin>/skills/<skill>/evals/evals.json`; one folder per eval per run config, each with `outputs/`, `grading.json`, `timing.json`. Treat it as append-only history — don't rewrite past iterations.

## Distribution channels

Both supported channels read from the same source of truth in `plugins/`. There is no per-channel build step — skills are plain Markdown installed by copying.

**Claude Code marketplace.** `.claude-plugin/marketplace.json` declares this repo as a Claude Code marketplace. It sets `metadata.pluginRoot: "./plugins"` and lists each plugin with `source: "<plugin-name>"` (resolved relative to `pluginRoot`) plus a per-plugin `skills: ["./skills/<name>"]` array (resolved relative to the plugin source). Each plugin also has its own `.claude-plugin/plugin.json`. Users add the marketplace by URL and pick the plugins they want.

**`vercel-labs/skills` CLI (npx).** The same `marketplace.json` is consumed by [`vercel-labs/skills`](https://github.com/vercel-labs/skills). The CLI clones this repo, reads `.claude-plugin/marketplace.json`, resolves the `pluginRoot` + `skills[]` entries, and copies the chosen `SKILL.md` (plus its directory tree) into the user's project under one of its discovery paths (`.claude/skills/`, `.agents/skills/`, etc.). Usage:

```
npx skills add mssalkhalifah/abp-skills                    # interactive picker
npx skills add mssalkhalifah/abp-skills abp-backend        # install one by name
```

There is no npm-publish step — `npx skills` is git-based and treats this repo as the registry.

Editing the skill happens in `plugins/<plugin>/skills/<skill>/`. Both channels see those edits the next time a user installs or syncs.

## Skill authoring conventions

These conventions are observed in both shipped skills; preserve them when editing.

**Frontmatter `description` is doing trigger-routing work.** It's intentionally long (multiple sentences, listing concrete primitives like `IRepository`, `AbpThemingOptions`, `LeptonX`, `ConfigStateService`). The model uses it to decide *when to load the skill*, and includes an explicit "Do NOT trigger on plain ASP.NET Core / EF Core / Angular / Blazor questions that disclaim ABP" carve-out. Keep that shape: list the surfaces, name the primitives, end with the negative trigger.

**`allowed-tools` is intentionally narrow.** `Read, Glob, Bash(abp:*), Bash(dotnet --version), Bash(dotnet --info), Bash(dotnet ef:*)` for backend; same minus `dotnet ef` for frontend. Don't broaden it casually — and respect the **deny-sticky rule** documented in `plugins/abp-backend/skills/abp-backend/SKILL.md` (a single denied Bash call disables Bash for the rest of the session and falls back to static refs).

**SKILL.md is the index, references are leaves.** SKILL.md states the framework guardrails and ends with a **routing table** that maps topics → reference files. New deep content goes in a reference; SKILL.md only gains a row in the routing table. The model is told "prefer the smallest reference that answers the question" — keep references focused so that holds up.

**Numbered framework guardrails (Rule N).** `plugins/abp-backend/skills/abp-backend/SKILL.md` codifies seven rules (one-class-per-file, no IQueryable in app service, manager-doesn't-persist, explicit controllers, integration services for cross-module reads, per-module DbContext+history-table, no BusinessException in app service). They are the load-bearing content — the eval suite is built around them, and the iteration-2-vs-official benchmark shows the rule-3 wording (`IReadOnlyRepository<T, Guid>` in managers, Id minted in AppService) is the only place this skill measurably beats the 18 official ABP skills. Phrase rules as: short title, prose statement, code example showing right vs wrong, edge-case rubric.

**ABP docs URLs are pinned to `https://abp.io/docs/10.3/...`** and references include the explicit fallback rule: on 404, retry under `/docs/latest/` and flag the fallback to the user. Don't drop the pin — version drift is what these skills exist to prevent.

**Template detection / UI stack detection.** Both skills open with a "figure out which template / UI stack the project uses" table (signals → likely template). The instruction is "do this once per session via `Glob`; the model retains the result across turns" — keep that cadence guidance; don't push the model to re-detect every turn.

**Customization contract is shared.** Both skills read the same `.abp-overrides.md` sidecar walking up from cwd to the first `.git/` or `.hg/` boundary, deeper-overrides-shallower (like hierarchical CLAUDE.md). `CLAUDE.md` beats skill defaults; `.abp-overrides.md` beats `CLAUDE.md`. Don't fork the sidecar into two files when editing customization docs — the README in each skill's `customization/` cross-references the other.

**Peer-skill split is hard.** Backend = C#, DDD, EF Core, app services, modules, integration services. Frontend = Razor/Blazor/Angular/RN/MAUI, themes, proxies, replaceable components. If a topic spans both (e.g. "expose this app service to Angular"), each skill covers its side and points at the peer.

## Editing flow

When changing a skill:

1. Edit `plugins/<plugin>/skills/<skill>/SKILL.md` and/or the targeted reference. Keep one rule/topic per file; don't bloat SKILL.md.
2. If you introduce a new rule or change a rule's wording, update `evals/evals.json` so an assertion exercises it — the iteration-2-vs-official analysis showed rules without assertions get no credit even when the model follows them.
3. Don't touch `benchmarks/` to "fix" old runs. New runs go in a new iteration folder. Benchmarks are the audit trail of how the skill evolved.
4. If you bump a plugin's content materially, bump the version in `plugins/<plugin>/.claude-plugin/plugin.json` (and optionally `metadata.version` in `.claude-plugin/marketplace.json` for the marketplace-level version).

When changing an eval's assertions, note it as an `assertion_revision` in the next `benchmark.json` (see `benchmarks/abp-backend/iteration-1/benchmark.json` — it carries `"assertion_revision": "tightened-v2"`).

## What's *not* in this repo

No application code, no .NET solution, no lint config, no CI workflow, no build step. The skills themselves are plain Markdown + JSON; both distribution channels (Claude Code marketplace and `vercel-labs/skills`) install by cloning this repo and copying files — there's nothing to compile or pack. The "real" test loop is running the evals externally and dropping artefacts into a new `benchmarks/<skill>/iteration-N/` folder.

Don't invent commands. If a workflow isn't documented above, it doesn't exist yet in this repo.
