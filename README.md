# agent-skills

A personal collection of Claude Code skills, published as a marketplace of plugins. Installable via the Claude Code marketplace or the [`vercel-labs/skills`](https://github.com/vercel-labs/skills) CLI — both channels read the same source in `plugins/`.

## Skills

| Plugin | What it covers |
| --- | --- |
| [`abp-backend`](plugins/abp-backend) | [ABP Framework](https://abp.io) 10.3 backend: DDD layering, application services, repositories, EF Core, modules, multi-tenancy, authorization, validation, infrastructure, and the `abp` CLI / Studio / Suite. |
| [`abp-frontend`](plugins/abp-frontend) | [ABP Framework](https://abp.io) 10.3 UI: MVC/Razor, Blazor (Server/WASM/WebApp/MAUI Blazor), Angular, React Native (Expo), .NET MAUI, LeptonX themes, virtual-file-system overrides, and dynamic vs static service proxies. |
| [`writing-commit-messages`](plugins/writing-commit-messages) | Conventional Commits 1.0.0 for commit messages and squash-merge PR titles. |

## Install

**Claude Code marketplace.** Add this repo as a marketplace and pick the plugins you want from the picker:

```
/plugin marketplace add mssalkhalifah/agent-skills
/plugin install abp-backend
```

**`npx skills` (vercel-labs/skills).** Copies the skill files into your project under `.claude/skills/` (or another supported discovery path):

```
npx skills add mssalkhalifah/agent-skills                 # interactive picker
npx skills add mssalkhalifah/agent-skills abp-backend     # install one by name
```

There is no build or publish step — both channels install by copying Markdown.

## Repo layout

```
.claude-plugin/marketplace.json   marketplace manifest consumed by both channels
plugins/<plugin>/                 source of truth for each shipped skill
  .claude-plugin/plugin.json
  skills/<skill>/
    SKILL.md                      entry point — guardrails + routing table
    references/                   focused topic pages, lazily loaded
    customization/                sidecar contract docs (where applicable)
    evals/evals.json              eval suite (where applicable)
benchmarks/<skill>/iteration-N/   append-only history of eval runs
```

See [CLAUDE.md](CLAUDE.md) for authoring conventions and the editing flow.

## License

MIT — see [LICENSE](LICENSE).
