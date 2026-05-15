# Project-specific overrides for the `abp-backend` skill

This skill ships ABP-framework backend defaults (DDD, modules, EF Core, infrastructure, tooling). Real projects often need to layer their own conventions on top — naming, base classes, Pro-module assumptions, deployment-specific tweaks. There are two ways to do that, and they're complementary.

The peer skill **`abp-frontend`** ships UI-flavoured defaults (Razor / Blazor / Angular / MAUI / RN, theming, navigation, replaceable components). Override files dropped at the project root are loaded by *both* skills, so a single `.abp-overrides.md` covers backend and UI rules in one place — but UI-specific conventions (component swaps, theme variables, menu contributors) are interpreted by `abp-frontend`. Cross-link to it whenever an override touches the UI tier.

## Mechanism 1: project `CLAUDE.md` (default, free)

Claude Code already auto-loads `CLAUDE.md` at the project root and any subfolders as project context. The `abp-backend` skill defers to anything in `CLAUDE.md` where it conflicts with the skill's defaults — no marker, no parsing, no special section needed.

**Use this** when your project has a few targeted rules ("we use `MyAppController` as the base class, not `AbpController`") that fit naturally inside the project's main CLAUDE.md.

The skill does not parse CLAUDE.md or look for an "ABP overrides" section. The model already integrates CLAUDE.md as project context; this skill just acknowledges that and steps back.

## Mechanism 2: `.abp-overrides.md` sidecar (opt-in, scales)

For substantial overrides — a project with 5+ ABP-specific rules, code examples, complete worked patterns — drop a `.abp-overrides.md` file at the project root. The skill will discover and load it.

### Discovery rules

- Walk **up** from the current working directory toward the filesystem root.
- **Stop** at any `.git/` or `.hg/` boundary, or at the filesystem root.
- Read **every** `.abp-overrides.md` found between cwd and that boundary.
- **Deeper files override shallower** — a sidecar in `apps/web/` beats one at the repo root for the same topic.

This mirrors how hierarchical `CLAUDE.md` works and supports mono-repos with multiple ABP solutions in sibling folders.

### Discovery cadence

The lookup happens **once per session** at the first ABP question. The model retains the loaded content across turns, so subsequent questions don't trigger re-discovery. If your sidecar changes mid-session, the user can re-invoke the skill to refresh.

### Precedence

```
.abp-overrides.md  >  CLAUDE.md  >  skill defaults
```

A sidecar rule beats both CLAUDE.md and the skill. CLAUDE.md beats the skill. The skill applies when the others are silent.

### Optional frontmatter

The skill **does not parse** the sidecar's frontmatter, but it's recommended for human bookkeeping — to track when overrides may need refreshing as ABP evolves:

```yaml
---
project: AcmeBookStore
abp-version: "10.3"
template: layered
---
```

Fields are free-form; treat the example as a starting point.

### Sidecar structure (recommended)

The body is free-form Markdown. A common layout that the model can navigate easily:

```markdown
---
project: AcmeBookStore
abp-version: "10.3"
template: layered
---

# AcmeBookStore — ABP overrides

## Base classes
- Controllers extend `AcmeBookStoreController` (in `Acme.BookStore.HttpApi`), not `AbpController`.
- Application services extend `AcmeBookStoreAppService`.
- The localization resource is `BookStoreResource`; exception code namespace is `"BookStore"`; table prefix is `"App"`.

## Auth and tenancy
- OpenIddict, multi-tenancy enabled, PostgreSQL via EF Core.
- Some Pro modules in use: SaaS, File Management. Allow guidance to assume them.

## Project-specific rules layered on top of the skill's framework guardrails

### Override of skill's Rule 4 (controllers vs auto-API)
We use **Auto API Controllers** for CRUD application services and only hand-write controllers for endpoints that need streaming/file responses or non-standard verbs.

### Additional rule (project-specific)
All entities have a `Tags` extra-property using ABP's Object Extension System (see framework/architecture-modularity.md).

## UI overrides (interpreted by the abp-frontend skill)

- Theme: LeptonX Lite, SideMenu layout.
- Brand component swapped via `AbpComponentReplacementOptions` — see the abp-frontend skill for the replaceable-components contract.
- Angular SPA at `angular/` consumes proxies generated via `abp generate-proxy -t ng`.

## Deployment

The DbMigrator runs as a Kubernetes Job before any host pods. CI publishes Docker images to Acme's private registry; Helm chart customisation lives in `etc/helm/acme/`.
```

### What to put in the sidecar

Put rules that are stable, project-specific, and not derivable from the code:

- **Base classes and naming conventions** — what your `MyAppController`, `MyAppAppService`, `MyAppDbContext`, exception namespace, table prefix, resource class are called.
- **License-tier assumptions** — which Pro modules the project uses; the skill should assume those are available.
- **Architecture overrides** — places where the project explicitly differs from the skill's framework guardrails (e.g., "we use auto-API throughout, not explicit controllers"). Be concrete about which rule is overridden.
- **Project-specific patterns** — e.g., "all aggregates emit a `<EntityName>Created` distributed event" — that the skill couldn't infer from the framework alone.
- **UI conventions** — theme choice, layout, replaceable-component swaps, menu / branding / toolbar contributors. These are interpreted by the **`abp-frontend`** skill; keep them in the same sidecar so backend and UI guidance stay in sync.

### What NOT to put in the sidecar

- Code-level facts the skill can derive from the project — file paths, project names, what's in `appsettings.json`. The model can read those directly.
- Information that lives in `CLAUDE.md` already — don't duplicate. Pick one home.
- Time-sensitive scratch notes ("last week we changed X") — those belong in commit history or a project journal.
- Secrets, connection strings, API keys.

## When to use which mechanism

| Project shape | Use |
|---|---|
| 1-3 ABP-specific rules, no code samples | CLAUDE.md |
| 5+ rules, code samples, multiple base classes, Pro-module-specific patterns | `.abp-overrides.md` |
| Multiple ABP solutions in one mono-repo with different conventions | `.abp-overrides.md` per solution folder |

## Reference example: `example.md`

A worked example sidecar ships alongside this README at [`example.md`](./example.md) — a fictional `Acme.BookStore` project's full ABP rules in the sidecar contract. It demonstrates:

- A complete frontmatter (`project`, `abp-version`, `template`)
- Project naming substitutions (base controller, exception namespace, table prefix, localization resource)
- Pro-module assumptions called out explicitly
- Per-rule interpretation showing how each framework guardrail manifests project-specifically
- Pre-existing technical debt the skill should know about
- ABP Studio MCP server configuration

Use it as a copy-paste starting point for your own `.abp-overrides.md`. The example is marked `example_only: true` in its frontmatter so it can never be mistaken for an actual project's overrides if a consumer drops the skill into a real repo.
