# abp-skills

Claude Code skills for working in [ABP Framework](https://abp.io) 10.3 .NET solutions, published as a marketplace of plugins. Installable via the Claude Code marketplace or the [`vercel-labs/skills`](https://github.com/vercel-labs/skills) CLI — both channels read the same source in `plugins/`.

## Skills

| Plugin | What it covers |
| --- | --- |
| [`abp-backend`](plugins/abp-backend) | DDD layering, application services, repositories, EF Core, modules, multi-tenancy, authorization, validation, infrastructure, and the `abp` CLI / Studio / Suite. |
| [`abp-frontend`](plugins/abp-frontend) | MVC/Razor, Blazor (Server/WASM/WebApp/MAUI Blazor), Angular, React Native (Expo), .NET MAUI, LeptonX themes, virtual-file-system overrides, and dynamic vs static service proxies. |
| [`writing-commit-messages`](plugins/writing-commit-messages) | Conventional Commits 1.0.0 for commit messages and squash-merge PR titles. |

## Install

**Claude Code marketplace.** Add this repo as a marketplace and pick the plugins you want from the picker:

```
/plugin marketplace add mssalkhalifah/abp-skills
/plugin install abp-backend
```

**`npx skills` (vercel-labs/skills).** Copies the skill files into your project under `.claude/skills/` (or another supported discovery path):

```
npx skills add mssalkhalifah/abp-skills                 # interactive picker
npx skills add mssalkhalifah/abp-skills abp-backend     # install one by name
```

There is no build or publish step — both channels install by copying Markdown.

## How the skills behave

- **Trigger routing lives in the frontmatter `description`.** Each skill's `description` lists concrete primitives (`IRepository`, `AbpThemingOptions`, `LeptonX`, …) and ends with a negative trigger so plain ASP.NET Core / EF Core / Angular / Blazor questions that disclaim ABP don't load it.
- **Narrow `allowed-tools`.** Backend allows `Read, Glob, Bash(abp:*), Bash(dotnet --version|--info), Bash(dotnet ef:*)`. Frontend is the same minus `dotnet ef`. A single denied Bash call disables Bash for the rest of the session — the skill falls back to static references.
- **SKILL.md is the index.** Deep content lives in `references/`; SKILL.md ends with a routing table mapping topics to reference files. The model is told to load the smallest reference that answers the question.
- **Docs pinned to `https://abp.io/docs/10.3/...`** with an explicit fallback to `/docs/latest/` on 404 (flagged to the user).
- **Customization sidecar.** Both ABP skills read `.abp-overrides.md` walking from `cwd` up to the first `.git/` / `.hg/` boundary, deeper overrides shallower. `CLAUDE.md` beats skill defaults; `.abp-overrides.md` beats `CLAUDE.md`.

## Repo layout

```
.claude-plugin/marketplace.json   marketplace manifest consumed by both channels
plugins/<plugin>/                 source of truth for each shipped skill
  .claude-plugin/plugin.json
  skills/<skill>/
    SKILL.md                      entry point — guardrails + routing table
    references/                   focused topic pages, lazily loaded
    customization/                sidecar contract docs
    evals/evals.json              eval suite (abp-* skills only)
benchmarks/<skill>/iteration-N/   append-only history of eval runs
```

See [CLAUDE.md](CLAUDE.md) for authoring conventions, the seven backend framework rules, and the editing flow.

## License

MIT — see [LICENSE](LICENSE).
