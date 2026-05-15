# Project-specific overrides for the `abp-frontend` skill

This skill ships ABP-frontend defaults (UI stacks, theming, customization-by-virtual-file-system, dynamic and static service proxies). Real projects often need to layer their own conventions on top — base layouts, theme picks, component-replacement maps, Pro-module assumptions, deployment-specific tweaks. There are two ways to do that, and they're complementary.

The same two mechanisms are shared with the peer **`abp-backend`** skill — a single `.abp-overrides.md` and a single `CLAUDE.md` cover both UI and server defaults. Don't duplicate one project's overrides into two sidecars; both skills read the same file.

## Mechanism 1: project `CLAUDE.md` (default, free)

Claude Code already auto-loads `CLAUDE.md` at the project root and any subfolders as project context. The `abp-frontend` skill defers to anything in `CLAUDE.md` where it conflicts with the skill's defaults — no marker, no parsing, no special section needed.

**Use this** when your project has a few targeted rules ("we use the LeptonX theme everywhere, with `--lpx-brand: #0E7A4B`") that fit naturally inside the project's main CLAUDE.md.

The skill does not parse CLAUDE.md or look for an "ABP overrides" section. The model already integrates CLAUDE.md as project context; this skill just acknowledges that and steps back.

## Mechanism 2: `.abp-overrides.md` sidecar (opt-in, scales)

For substantial overrides — a project with 5+ ABP-specific rules, code examples, complete worked patterns spanning UI and backend — drop a `.abp-overrides.md` file at the project root. Both `abp-frontend` and `abp-backend` will discover and load the same file.

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
ui: angular
theme: leptonx
---
```

Fields are free-form; treat the example as a starting point.

### Sidecar structure (recommended)

The body is free-form Markdown. Because the same file is read by `abp-frontend` and `abp-backend`, group rules by topic so each skill can pick out what's relevant. A common layout:

```markdown
---
project: AcmeBookStore
abp-version: "10.3"
template: layered
ui: angular
theme: leptonx
---

# AcmeBookStore — ABP overrides

## UI / theme conventions
- Angular SPA against the layered backend; `@volosoft/abp.ng.theme.lepton-x` (commercial LeptonX) with side-menu layout.
- Brand color is `#0E7A4B`; override `--lpx-brand` in `src/styles.scss`.
- Custom logo lives at `src/assets/logo.svg`; we replace `eThemeLeptonXComponents.Logo` via `ReplaceableComponentsService`.
- Generated proxies regenerated via `yarn proxy:gen`; never hand-edit `src/app/proxy/**`.

## Auth and tenancy
- OpenIddict, multi-tenancy enabled, PostgreSQL via EF Core.
- Tenant resolution by subdomain in production (`https://{0}.acme.com`); on localhost we use the tenant box switcher.
- Some Pro modules in use: SaaS, File Management. Allow guidance to assume them.

## Backend conventions (read by abp-backend)
- Controllers extend `AcmeBookStoreController` (in `Acme.BookStore.HttpApi`), not `AbpController`.
- Application services extend `AcmeBookStoreAppService`.
- The localization resource is `BookStoreResource`; exception code namespace is `"BookStore"`.

## Project-specific UI rules
- All forms use the `abp-extensible-form` directive — never raw `<form>`.
- Page-level breadcrumbs are set via `PageLayoutService.setBreadcrumbs(...)` in `ngOnInit`, not from the route data.

## Deployment
- Angular bundle ships from `dist/AcmeBookStore/browser` to S3 + CloudFront.
- Backend Helm chart customisation lives in `etc/helm/acme/`.
```

### What to put in the sidecar (UI side)

Put rules that are stable, project-specific, and not derivable from the code:

- **Theme pick and brand tokens** — which official theme (Basic / LeptonX Lite / LeptonX), brand color values, where global SCSS overrides live, which `--lpx-*` variables are project-set.
- **Layout conventions** — Side menu vs Top menu (LeptonX), which custom layout components have replaced ABP's defaults.
- **Component-replacement map** — concrete components your project has substituted (e.g., "we replace `eThemeLeptonXComponents.Logo` and `eIdentityComponents.Roles`").
- **Service-proxy workflow** — how the team regenerates Angular proxies / TS proxies / static C# clients, which folder they live in, whether dynamic proxies are also enabled or disabled per module.
- **License-tier UI assumptions** — which Pro UI modules the project uses (LeptonX commercial, ABP Suite generators); the skill should assume those are available.
- **Project-specific UI patterns** — e.g., "every page sets `PageLayout.MenuItemName` to a shared const matching its menu contributor's `name`".

### What NOT to put in the sidecar

- Code-level facts the skill can derive from the project — file paths, project names, what's in `appsettings.json`. The model can read those directly.
- Information that lives in `CLAUDE.md` already — don't duplicate. Pick one home.
- Time-sensitive scratch notes ("last week we changed X") — those belong in commit history or a project journal.
- Secrets, connection strings, API keys, OAuth client secrets.

## When to use which mechanism

| Project shape | Use |
|---|---|
| 1-3 ABP-specific rules, no code samples | CLAUDE.md |
| 5+ rules, code samples, multiple base classes, theme + module customizations | `.abp-overrides.md` |
| Multiple ABP solutions in one mono-repo with different conventions | `.abp-overrides.md` per solution folder |

## Reference example sidecar

A worked example of an `.abp-overrides.md` ships alongside the peer **`abp-backend`** skill at `abp-backend/customization/example.md` (built around a fictional `Acme.BookStore` project). It demonstrates:

- A complete frontmatter (`project`, `abp-version`, `template`, plus example fields)
- Project naming substitutions (base controller, exception namespace, table prefix, localization resource)
- Pro-module assumptions called out explicitly
- Per-rule interpretation showing how each guardrail manifests project-specifically
- Pre-existing technical debt the skill should know about
- ABP Studio MCP server configuration

Use it as a copy-paste starting point for your own `.abp-overrides.md`. The example is marked `example_only: true` in its frontmatter so it can never be mistaken for an actual project's overrides if a consumer drops the skill into a real repo.
