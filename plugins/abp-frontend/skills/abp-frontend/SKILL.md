---
name: abp-frontend
description: ABP Framework UI guidance for any ABP 10.3 solution — MVC / Razor Pages, Blazor (Server / WebAssembly / WebApp / MAUI Blazor), Angular SPA, React Native (Expo) mobile, and .NET MAUI mobile, plus the cross-cutting UI infrastructure (LeptonX / LeptonX Lite / Basic theming, navigation menus, layout hooks, dynamic and static service proxies, replaceable components, virtual-file-system overrides, mobile auth via OpenIddict). Use whenever editing Razor / Blazor `.razor` / Angular TS / Expo RN / MAUI XAML files in an ABP solution, choosing a UI stack at `abp new` time, configuring `AbpThemingOptions` / `AbpRouterOptions` / `AbpNavigationOptions` / `AbpLayoutHookOptions` / `AbpComponentReplacementOptions`, picking LeptonX vs LeptonX Lite vs Basic, switching SideMenu / TopMenu layouts, swapping a default ABP component for a custom one, generating service proxies (`abp generate-proxy -t ng`/`-t js`), styling with `--lpx-*` CSS variables, wiring `IMenuContributor` / `IBrandingProvider` / `IToolbarContributor`, configuring multi-tenancy in the Angular tenant box / Blazor tenant switcher, debugging `DisableTransportSecurityRequirement` for mobile auth, or answering questions about ABP UI customization. Trigger broadly — even when the user mentions ABP UI, AbpComponentBase, abp-modal / abp-dynamic-form / abp-input tag helpers, ConfigStateService, RestService, ReplaceableComponentsService, PageLayout, LeptonX / LeptonX Lite, `@abp/ng.*`, `@volosoft/abp.ng.theme.lepton-x`, AbpMauiClientModule, OpenIddict in an ABP UI context, MAUI Blazor / Blazor Hybrid, or React Native auth flow without explicitly naming the framework. Do NOT trigger on plain Angular / Blazor / React Native / MAUI questions that explicitly disclaim ABP ("not using ABP", "vanilla Angular", "stock Blazor WebAssembly").
allowed-tools: Read, Glob, Bash(abp:*), Bash(dotnet --version), Bash(dotnet --info)
---

# ABP Framework — UI guidance (ABP 10.3)

This skill is **template-aware** ABP UI guidance. It covers the five UI options ABP 10.3 ships first-party — MVC / Razor Pages, Blazor (Server / WebAssembly / WebApp / MAUI Blazor), Angular, React Native (Expo), .NET MAUI — plus the cross-cutting UI infrastructure (theming, navigation, layout hooks, dynamic and static service proxies, virtual-file-system overrides, replaceable components, mobile auth wiring). For backend topics (DDD layering, application services, repositories, EF Core, OpenIddict, modules, microservice infrastructure), reach for the peer `abp-backend` skill.

Read this file first; load focused references below only when the topic is in scope.

## How this skill is organised

- **SKILL.md** (this file) — UI stack detection, theme picks, customization-by-virtual-file-system pattern, component-vs-page override decision, dynamic-vs-static service proxies, customization contract, routing table.
- **references/ui/** — per-stack drill-downs (overview, mvc-razor-pages, blazor, angular, mobile, themes).
- **customization/** — sidecar contract documentation; the sidecar is shared with the peer `abp-backend` skill, so a single `.abp-overrides.md` covers both UI and server defaults.

When deciding what to load, prefer the smallest reference that answers the question. The routing table at the end of this file is the source of truth.

## UI stack detection — figure out which UI(s) the project uses

ABP wires the UI through template-time package selection plus sibling project folders; there is no runtime "pick a UI" API. Before answering a UI-shaped question, infer which UI(s) the active project ships:

| Signal in the working directory | Likely UI stack |
|---|---|
| `*.Web/` with `Pages/`, `wwwroot/`, `_ViewImports.cshtml`, `[DependsOn(typeof(AbpAspNetCoreMvcUi*ThemeModule))]` | **MVC / Razor Pages** (`-u mvc`) |
| `*.Blazor/`, `*.Blazor.Server/`, `*.Blazor.Client/`, `*.Blazor.WebApp/` with `App.razor`, `Routes.razor`, `*.razor` files, `[DependsOn(typeof(AbpAspNetCoreComponents*Module))]` | **Blazor** (`-u blazor` / `-u blazor-server` / `-u blazor-webapp`) — drill into hosting model from `Routes.razor` and the `*.csproj` (Server vs WASM vs WebApp) |
| `angular/` folder at solution root with `package.json`, `app.config.ts` or `app.module.ts`, `@abp/ng.core` in dependencies | **Angular** (`-u angular`) |
| `react-native/` folder with `package.json`, `Environment.ts`, `app.json` (Expo) | **React Native** (`-m react-native`) — almost always paired with another `-u` |
| `*.Maui/` project with `MauiProgram.cs`, `[DependsOn(typeof(AbpMauiClientModule))]`, `Platforms/Android` and `Platforms/iOS` | **.NET MAUI** (`-t maui --old`) |

Do this discovery once per session via `Glob`; the model retains the result across turns. **A solution can ship multiple UIs at once** — most commonly an Angular SPA + React Native mobile, or a Blazor host + MAUI client. When you see two, surface both in your interpretation rather than picking one arbitrarily.

If detection is ambiguous and the answer materially depends on which stack the user is editing, ask rather than guess.

## Theme conventions across stacks

ABP ships three official themes; the wiring is parallel across MVC / Blazor / Angular but the package names and provider APIs differ:

| Theme | OSS / Commercial | Default in | NuGet (MVC) | Angular npm |
|---|---|---|---|---|
| **Basic** | OSS | nothing — opt-in only | `Volo.Abp.AspNetCore.Mvc.UI.Theme.Basic` | `@abp/ng.theme.basic` |
| **LeptonX Lite** | OSS | free templates | `Volo.Abp.AspNetCore.Mvc.UI.Theme.LeptonXLite` | `@abp/ng.theme.lepton-x` |
| **LeptonX** | Commercial | commercial templates (Team+) | `Volo.Abp.AspNetCore.Mvc.UI.Theme.LeptonX` | `@volosoft/abp.ng.theme.lepton-x` |

Three patterns hold across all stacks:

1. **Theme = a module reference plus an options class.** Add a `[DependsOn(typeof(AbpAspNetCoreMvcUi*ThemeModule))]` (or the Blazor/WASM variant) and configure the matching `<Theme>ThemeOptions` (e.g., `LeptonXThemeOptions.DefaultStyle`, `LeptonXThemeMvcOptions.ApplicationLayout`). Each theme module already self-registers in `AbpThemingOptions.Themes`; you only touch `AbpThemingOptions` directly when shipping a custom `ITheme`.
2. **Brand and surface tweaks go through `--lpx-*` CSS variables, not forks.** Override on `:root` for theme-wide values, scope to `.lpx-theme-dark / .lpx-theme-light / .lpx-theme-dim` for variant-specific tuning. Forking the theme module guarantees pain at the next ABP upgrade.
3. **Theme switches stay package-level.** Swap the `[DependsOn(...)]` line and the registered theme type (Blazor: also update the `@using` in `Routes.razor`); page code does not change because every theme implements `StandardLayouts.Application / Account / Empty`.

Load `references/ui/themes.md` for the full per-stack wiring (LeptonX Blazor needs the `*Bundling` module too, Lite has no style picker, Angular's `@volosoft/...` vs `@abp/ng.theme.lepton-x` split, etc.).

## Customization-by-virtual-file-system — the canonical override pattern

ABP's virtual file system is what lets a project override a module-shipped Razor page, Razor component, or static asset **without forking the module**. The pattern is the same across MVC, Blazor, and (with a different mechanism) Angular:

- **MVC / Razor Pages:** drop a file at the same virtual path inside your `*.Web` project (`Pages/Account/Login.cshtml`, or `Themes/<ThemeName>/Layouts/Application.cshtml`). The runtime overlays your file over the module's embedded resource. Pair with `_ViewImports.cshtml` so `abp-*` tag helpers still work.
- **Blazor:** create a Razor component that `@inherits` the original and decorate with `[ExposeServices(typeof(BaseComponent))]` + `[Dependency(ReplaceServices = true)]`. ABP DI substitutes your component at injection sites.
- **Angular:** `ReplaceableComponentsService.add({ component, key })` from `@abp/ng.core`, where `key` is one of the well-known enums (`eIdentityComponents`, `eThemeLeptonXComponents`, `eThemeBasicComponents`). Layout replacements **must** keep `<router-outlet>` in the template.
- **Page model swaps (MVC):** decorate your custom `PageModel` with `[ExposeServices(typeof(OriginalPageModel))]` + `[Dependency(ReplaceServices = true)]` to swap behavior without touching the `.cshtml`.

Why this matters: **forking a module's source disconnects you from upstream upgrades.** The override patterns above re-express your project's intent in your own files, so an `abp update` continues to work.

## When to override a component vs replace a page

The rule of thumb: scope your override to the smallest unit that owns the rule.

- **Component override** (Blazor `[ExposeServices]` / Angular `ReplaceableComponentsService`): when you want to change rendering or behavior of a *single subcomponent* — branding, logo, a specific identity grid, a toolbar slot. The shell, routing, and surrounding layout stay ABP-managed.
- **Layout replacement**: when the change spans the whole shell — different navbar layout, sidebar collapsed by default, a corporate banner above every page. Use the per-stack layout slot (LeptonX `LeptonXThemeMvcOptions.ApplicationLayout`, LeptonX Blazor `LeptonXThemeBlazorOptions.Layout`, Angular `provideSideMenuLayout()` / `provideTopMenuLayout()`).
- **Page replacement** (MVC virtual file system / Blazor `[ExposeServices(typeof(OriginalPageModel))]` / Angular per-route component swap): when a single page (Login, Register, Profile) needs a different visual *and* different behavior. Reach for this only when component-level extension points (`IMenuContributor`, `IBrandingProvider`, `IToolbarContributor`, ngx-validate blueprints, page toolbars) cannot express the change.
- **Theme fork** (`abp get-source Volo.Abp.AspNetCore.Mvc.UI.Theme.Basic` or `abp add-package @abp/ng.theme.basic --with-source-code`): last resort. Useful only when authoring a new branded theme on top of Basic. Once you fork, you own upgrades.

## Dynamic vs static service proxies — pick once per UI stack

Every UI stack consumes ABP application services through a generated proxy layer. Two flavours exist:

- **Dynamic proxies** — runtime-served. MVC: `/Abp/ServiceProxyScript` emits JS for every app service. Blazor / MAUI: `IRemoteService` resolves a `DynamicHttpProxyInterceptor`-backed implementation of `IXxxAppService`. Reach for dynamic when you want zero-build-step proxy refresh during dev — proxy follows backend signature changes automatically.
- **Static proxies** — generated at dev time. MVC: `abp generate-proxy -t js` writes `ClientProxies/*.js`. Angular: `abp generate-proxy -t ng` writes TS services under `@proxy/...` (this is the **only** Angular path — there is no dynamic Angular proxy). MAUI: static C# client proxies are the documented default. Reach for static when you want compile-time visibility of breaking changes, AOT-friendliness, or both.

Two operating rules:

1. **Don't run dynamic and static for the same module.** If you ship static JS proxies, call `Configure<DynamicJavaScriptProxyOptions>(o => o.DisableModule("<name>"))`. Otherwise both definitions register and the last one wins silently.
2. **Re-run `abp generate-proxy` after every backend application-service signature change.** Stale static proxies are a primary cause of "method not found" errors at runtime.

Angular is the special case — `abp generate-proxy -t ng` requires `@abp/ng.schematics` installed and `apis.default.rootNamespace` populated in `environment.ts`. See `references/ui/angular.md`.

## How localization, auth, and multi-tenancy surface in each UI stack

These three concerns are owned by the backend (`abp-backend` skill) but each UI stack consumes them differently. A high-level map:

| Concern | MVC | Blazor | Angular | RN | MAUI |
|---|---|---|---|---|---|
| **Localization** | `@inject IHtmlLocalizer<MyResource> L` in `.cshtml`; auto display-name on `abp-input` | `AbpComponentBase.L` or `@inject IStringLocalizer<MyResource> L` | `LocalizationService.instant/get` + `abpLocalization` pipe; `'Resource::Key'` syntax | `Environment.ts.localization.defaultResourceName` + i18n-js | Settings page language switcher; `IStringLocalizer` works in MVVM |
| **Auth** | Cookie auth from MVC host; `[Authorize]` on PageModel | Cookie (Server) or OIDC via `AddAbpOpenIdConnect` (WASM); `[Authorize]` / `<AuthorizeView>` | `angular-oauth2-oidc`; `oAuthConfig.responseType: 'code'` (PKCE); `AuthGuard` | OpenIddict via Expo + `clientId` in `Environment.ts`; **needs** `DisableTransportSecurityRequirement()` in DEBUG | `OidcClientOptions` + `WebAuthenticatorBrowser`; tokens in `SecureStorage` |
| **Multi-tenancy** | Tenant resolved server-side; `[MultiTenantSwitcherTagHelper]` on the login page | `ICurrentTenant` injection; tenant box in account layout | `__tenant` HTTP header; subdomain via `application.baseUrl: 'https://{0}.mydomain.com'` and `{0}` in `oAuthConfig.issuer`; `TENANT_NOT_FOUND_BY_NAME` token | Tenant text field on login screen; sets `__tenant` header | Tenant switcher on login; same `__tenant` header model |

For server-side definitions of permissions, localization resources, tenant resolvers, and OpenIddict clients — defer to the **abp-backend** skill (`framework/authorization.md`, `framework/fundamentals.md`, `framework/multi-tenancy.md`, `modules/identity-and-auth.md`).

## Customization contract — overrides win

This skill provides ABP-frontend defaults. **Project-specific overrides take precedence** in this order:

1. **`.abp-overrides.md`** — opt-in sidecar shared with the peer `abp-backend` skill. Walk up from the current working directory toward the filesystem root; stop at any `.git/` or `.hg/` boundary. Read every `.abp-overrides.md` found between cwd and that boundary; **deeper files override shallower** (mirrors hierarchical CLAUDE.md). Do this lookup once per session at the first ABP question; the model retains the loaded content across turns.
2. **`CLAUDE.md`** — already auto-loaded by Claude Code as project context. The skill does not parse it or look for any "ABP overrides" marker. Defer to it where it conflicts with this skill's defaults.
3. **This skill** — applies when the above are silent.

A consumer sidecar may carry an optional frontmatter for human bookkeeping:

```yaml
---
project: AcmeBookStore
abp-version: "10.3"
template: layered
ui: angular
theme: leptonx
---
```

The skill does not parse this frontmatter; it's only there so the consumer can track when the override file might need refreshing. See `customization/README.md` for the full sidecar template, walk-up rules, and what UI-side rules to put in it.

## Documentation version pin and fallback

ABP docs URLs in references are pinned to **`https://abp.io/docs/10.3/...`**. If a `/10.3/` URL returns 404 or redirects to a generic page, retry the same path under `/docs/latest/` and treat the result as best-effort (newer behaviour may not match 10.3 exactly). Note any such fallback when citing docs to the user. The themes reference already records that the Basic theme's framework-specific subpages currently 404 under `/10.3/` and uses `/latest/` for those.

## Tooling — Bash deny-sticky rule

This skill is allowed to invoke a small set of CLI tools (`abp ...`, `dotnet --version`, `dotnet --info`) to detect versions, generate proxies, or read CLI help — note that `dotnet ef ...` is **not** in the allow-list because migrations are a backend concern; for those, defer to `abp-backend`. **If the user denies a Bash tool call from this skill, set a session flag and stop attempting Bash for the rest of the session unless the user explicitly re-invokes the skill or asks for a CLI action by name.** Fall back to static guidance from the references for the rest of the session. The denial is a strong signal; respect it without re-asking.

## Routing table — when to read which reference

Pull the focused reference only when the topic applies. Each file is a self-contained slice of the UI surface.

### UI stacks

| Topic | File |
|-------|------|
| "Which UI should I pick?" / MVC vs Blazor vs Angular vs RN vs MAUI / cross-stack theme abstraction | [references/ui/overview.md](./references/ui/overview.md) |
| MVC / Razor Pages — tag helpers (`abp-modal`, `abp-dynamic-form`), `abp.ModalManager`, DataTables.Net wiring, dynamic + static JS proxies, widgets (`[Widget]`), layout hooks, virtual-file-system overrides, security headers, Razor Pages integration tests | [references/ui/mvc-razor-pages.md](./references/ui/mvc-razor-pages.md) |
| Blazor (Server / WASM / WebApp / MAUI Blazor) — `AbpComponentBase`, Blazorise, `IMenuContributor`, `PageLayout`, `[ExposeServices]` + `[Dependency(ReplaceServices=true)]` overrides, `AbpRouterOptions.AdditionalAssemblies`, `UserFriendlyException`, `GlobalFeatureManager`, PWA scaffolding | [references/ui/blazor.md](./references/ui/blazor.md) |
| Angular SPA — `@abp/ng.*` packages, `provideAbpCore` + `provide<Feature>Config`, `Config.Environment`, `RestService`, generated TS proxies (`@proxy/...`), `ConfigStateService`, `ReplaceableComponentsService`, ngx-validate `withValidationBluePrint`, ngx-datatable, multi-tenancy `{0}` placeholder, `CoreTestingModule` | [references/ui/angular.md](./references/ui/angular.md) |
| Mobile — React Native (Expo) template + .NET MAUI template; `Environment.ts`, `AbpMauiClientModule`, `OidcClientOptions`, `WebAuthenticatorBrowser`, `SecureStorage`, `DisableTransportSecurityRequirement`, Android `adb reverse`, iOS Simulator quirks | [references/ui/mobile.md](./references/ui/mobile.md) |

### Theming

| Topic | File |
|-------|------|
| Theme picks across stacks (Basic / LeptonX Lite / LeptonX), `AbpThemingOptions`, `LeptonXThemeOptions`, `LeptonXThemeMvcOptions` / `LeptonXThemeBlazorOptions`, Angular `provideThemeLeptonX()`, `--lpx-*` CSS variables, layout overrides, custom theme authoring | [references/ui/themes.md](./references/ui/themes.md) |

### Customization

| Topic | File |
|-------|------|
| How to write `.abp-overrides.md` for a project; sidecar contract; how it's shared with `abp-backend` | [customization/README.md](./customization/README.md) |
| Worked example sidecar | See `abp-backend/customization/example.md` in the peer skill |

## When this skill steps back

If the question is purely backend-shaped — DDD layers, application services, repositories, EF Core, OpenIddict server config, modules (Identity, SaaS, BlobStoring), microservice gateways, integration services — defer to the **`abp-backend`** skill. This skill stays silent on those topics; the cross-references above point at the right `abp-backend` files for follow-up.
