# UI Themes (LeptonX, LeptonX Lite, Basic)

> Reference for the three official ABP UI theme modules — LeptonX (commercial), LeptonX Lite (free, default for free templates), and Basic (minimalist Bootstrap baseline) — covering how they plug into the AbpThemingOptions/ITheme system across MVC, Blazor, and Angular.

## When to load this reference

- User asks 'which theme should I use in my ABP solution?' or compares LeptonX vs LeptonX Lite vs Basic
- User wants to switch themes (e.g., 'replace Basic theme with LeptonX' or 'remove LeptonX Lite and use Basic')
- User asks how to customize layout, navbar, footer, sidebar, or brand component for an ABP-rendered page
- User asks how to change the default LeptonX style (Dim/Dark/Light/System) or remove a style from the picker
- User asks about --lpx-* CSS variables, brand color, logo, or how to scope theme overrides to dark vs light
- User asks for the NuGet package or NPM package name of an ABP theme (Volo.Abp.AspNetCore.Mvc.UI.Theme.* / @abp/ng.theme.* / @volosoft/abp.ng.theme.lepton-x)
- User asks about Side Menu vs Top Menu layout, or about LeptonXThemeMvcOptions / LeptonXThemeBlazorOptions
- User asks about ITheme implementation, ThemeName attribute, AbpThemingOptions.Themes.Add, or DefaultThemeName
- User asks how to write their own theme module on top of the Basic theme
- User mentions Volo.Abp.LeptonXTheme or 'abp get-source LeptonXTheme'

**Audience:** ABP developers configuring or customizing the visual layer of an ABP solution — choosing a theme at template-creation time, swapping themes mid-project, restyling brand colors and logos, overriding layouts/components, or authoring a custom theme that integrates with ABP's ITheme/AbpThemingOptions infrastructure.

## Key concepts

- ITheme — Volo.Abp.AspNetCore.Mvc.UI.Theming.ITheme — interface implemented by every ABP theme; its GetLayout(string name, bool fallbackToDefault) returns the .cshtml/.razor path for the requested standard layout. Reach for it when authoring a custom theme or replacing layout resolution.
- ThemeNameAttribute — Volo.Abp.AspNetCore.Mvc.UI.Theming.ThemeNameAttribute — decorates an ITheme implementation with its unique name (e.g., [ThemeName("Basic")]). Required so AbpThemingOptions.DefaultThemeName and IThemeSelector can identify the theme.
- AbpThemingOptions — Volo.Abp.AspNetCore.Mvc.UI.Theming.AbpThemingOptions — module options exposing Themes (TypeList<ITheme>) and DefaultThemeName. Configure inside ConfigureServices to register themes (options.Themes.Add<BasicTheme>()) and pick the default. Each ABP theme module already registers itself; override only when you ship your own ITheme.
- IThemeSelector / DefaultThemeSelector — Volo.Abp.AspNetCore.Mvc.UI.Theming.IThemeSelector and DefaultThemeSelector — runtime service that returns the active ITheme for the current request. Replace via DI to drive theme by tenant, user setting, or feature flag.
- StandardLayouts — Volo.Abp.AspNetCore.Mvc.UI.Theming.StandardLayouts — string constants Application, Account, and Empty. Every theme must resolve these three layout names so framework pages (Login, Settings, error pages) render under any registered theme.
- BasicTheme — Volo.Abp.AspNetCore.Mvc.UI.Theme.Basic.BasicTheme — the open-source minimalist ITheme implementation (no styling beyond Bootstrap 5). Reach for it when you plan to author your own visual design and want to inherit only ABP's wiring, not an opinionated look.
- AbpAspNetCoreMvcUiBasicThemeModule — Volo.Abp.AspNetCore.Mvc.UI.Theme.Basic.AbpAspNetCoreMvcUiBasicThemeModule — module pulled in via NuGet 'Volo.Abp.AspNetCore.Mvc.UI.Theme.Basic'; registers BasicTheme in AbpThemingOptions.
- AbpAspNetCoreMvcUiLeptonXLiteThemeModule — Volo.Abp.AspNetCore.Mvc.UI.Theme.LeptonXLite.AbpAspNetCoreMvcUiLeptonXLiteThemeModule — MVC module from NuGet 'Volo.Abp.AspNetCore.Mvc.UI.Theme.LeptonXLite'. Registers LeptonXLite as an ITheme.
- LeptonXLiteThemeBundles — Volo.Abp.AspNetCore.Mvc.UI.Theme.LeptonXLite.Bundling.LeptonXLiteThemeBundles — strongly-typed bundle names (Styles.Global, Scripts.Global) used to extend the theme's CSS/JS bundle.
- LeptonXLiteToolbars — Volo.Abp.AspNetCore.Mvc.UI.Theme.LeptonXLite.Toolbars.LeptonXLiteToolbars — Main and MainMobile string constants identifying toolbars you target via IToolbarContributor.
- AbpAspNetCoreMvcUiLeptonXThemeModule — Volo.Abp.AspNetCore.Mvc.UI.Theme.LeptonX.AbpAspNetCoreMvcUiLeptonXThemeModule — commercial MVC theme module from NuGet 'Volo.Abp.AspNetCore.Mvc.UI.Theme.LeptonX'; supersedes the legacy AbpAspNetCoreMvcUiLeptonThemeModule and LeptonThemeManagementWebModule.
- LeptonXThemeOptions — Volo.Abp.AspNetCore.Components.Web.LeptonXTheme.Options.LeptonXThemeOptions (also exposed for MVC) — DefaultStyle (LeptonXStyleNames.System default; Dim, Dark, Light, System), and Styles dictionary you can mutate to add/remove available styles.
- LeptonXStyleNames — string constants System, Dim, Dark, Light identifying the bundled visual styles in LeptonX (no styles ship in LeptonX Lite — Lite has a single visual).
- LeptonXThemeMvcOptions — Volo.Abp.AspNetCore.Mvc.UI.Theme.LeptonX.Options.LeptonXThemeMvcOptions — ApplicationLayout (LeptonXMvcLayouts.SideMenu | TopMenu) and AccountLayoutBackgroundStyle inline CSS for login/registration backdrop.
- LeptonXThemeBundles — Volo.Abp.AspNetCore.Mvc.UI.Theme.LeptonX.Bundling.LeptonXThemeBundles — Styles.Global / Scripts.Global bundle name constants paralleling LeptonXLiteThemeBundles.
- AbpAspNetCoreComponentsWebAssemblyLeptonXLiteThemeModule / AbpAspNetCoreComponentsWebAssemblyLeptonXLiteThemeBundlingModule — Blazor WebAssembly modules for LeptonX Lite (ship under Volo.Abp.AspNetCore.Components.WebAssembly.LeptonXLiteTheme.* NuGet IDs); register layouts and bundle assets respectively.
- AbpAspNetCoreComponentsWebAssemblyLeptonXThemeModule / AbpAspNetCoreComponentsWebAssemblyLeptonXThemeBundlingModule — Blazor WebAssembly modules for commercial LeptonX (Volo.Abp.AspNetCore.Components.WebAssembly.LeptonXTheme.* NuGet); equivalent server-side names: AbpAspNetCoreComponentsServerLeptonXThemeModule.
- LeptonXThemeBlazorOptions — Volo.Abp.AspNetCore.Components.Web.LeptonXTheme.Options.LeptonXThemeBlazorOptions — Layout (LeptonXBlazorLayouts.SideMenu | TopMenu) and MobileMenuSelector predicate filtering which menu items render on mobile.
- @abp/ng.theme.basic — Angular NPM package shipping ThemeBasicModule (legacy NgModule.forRoot pattern) and provideThemeBasicConfig() (standalone). Registers ApplicationLayoutComponent, AccountLayoutComponent, EmptyLayoutComponent against @abp/ng.core's layout service.
- @abp/ng.theme.lepton-x — Angular NPM for LeptonX Lite. Exposes provideThemeLeptonX(), provideSideMenuLayout(), provideAccountLayout() and the eThemeLeptonXComponents replacement-key enum (ApplicationLayout, AccountLayout, EmptyLayout, Logo, Navbar, Breadcrumb, Footer, etc.).
- @volosoft/abp.ng.theme.lepton-x — Angular NPM for commercial LeptonX. Exposes provideThemeLeptonX() plus provideSideMenuLayout()/provideTopMenuLayout() under the /layouts subpath. Pairs with bootstrap-icons.
- ReplaceableComponentsService — @abp/ng.core.ReplaceableComponentsService — Angular service used together with eThemeLeptonXComponents (or eThemeBasicComponents) keys to swap any layout subcomponent (Logo, Navbar, Footer, Toolbar) at runtime.
- IToolbarContributor — Volo.Abp.UI.Navigation.Toolbars.IToolbarContributor — implement and register to push items into LeptonXLiteToolbars.Main or .MainMobile (and the equivalent commercial LeptonX toolbar names).
- --lpx-* CSS custom properties — global theming surface for both LeptonX and LeptonX Lite: --lpx-brand, --lpx-brand-text, --lpx-logo, --lpx-logo-icon, --lpx-content-bg, --lpx-content-text, --lpx-card-bg, --lpx-card-title-text-color, --lpx-border-color, --lpx-shadow, --lpx-radius, --lpx-navbar-color, --lpx-navbar-text-color, --lpx-navbar-active-text-color, --lpx-navbar-active-bg-color. Override on :root globally or scope to .lpx-theme-dark / .lpx-theme-light / .lpx-theme-dim for variant-specific tuning.

## Configuration pattern

Themes plug into the framework via two layers: (1) every theme module registers itself in AbpThemingOptions.Themes during its own ConfigureServices, so consumers normally only add the matching [DependsOn(typeof(AbpAspNetCoreMvcUi<X>ThemeModule))] to their web module; (2) theme-specific behavior (default style, layout choice, login background, bundle extensions, toolbar items) is configured via that theme's Options class. Concrete pattern for each stack:

MVC web module (LeptonX example):
```csharp
using Volo.Abp.AspNetCore.Mvc.UI.Theme.LeptonX;
using Volo.Abp.AspNetCore.Mvc.UI.Theme.LeptonX.Bundling;
using Volo.Abp.AspNetCore.Mvc.UI.Theme.LeptonX.Options;
using Volo.Abp.AspNetCore.Components.Web.LeptonXTheme.Options;
using Volo.Abp.Modularity;
using Volo.Abp.AspNetCore.Mvc.UI.Bundling;

[DependsOn(typeof(AbpAspNetCoreMvcUiLeptonXThemeModule))]
public class MyProjectWebModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        Configure<LeptonXThemeOptions>(options =>
        {
            options.DefaultStyle = LeptonXStyleNames.Dark;       // System | Dim | Dark | Light
        });

        Configure<LeptonXThemeMvcOptions>(options =>
        {
            options.ApplicationLayout = LeptonXMvcLayouts.SideMenu;   // or .TopMenu
            options.AccountLayoutBackgroundStyle = "background-image: url('/images/bg.svg') !important;";
        });

        Configure<AbpBundlingOptions>(options =>
        {
            options.StyleBundles.Configure(
                LeptonXThemeBundles.Styles.Global,
                bundle => bundle.AddFiles("/global-styles.css"));
        });
    }
}
```

For LeptonX Lite the depend-on, options, and bundle constants are the AbpAspNetCoreMvcUiLeptonXLiteThemeModule + LeptonXLiteThemeBundles family; LeptonX Lite has no DefaultStyle / SideMenu-vs-TopMenu switch. For Basic, DependsOn(typeof(AbpAspNetCoreMvcUiBasicThemeModule)) is the only required wiring; further customization is by overriding the .cshtml files under Themes/Basic/Layouts in your project's virtual file system.

Blazor web module (LeptonX example):
```csharp
[DependsOn(
    typeof(AbpAspNetCoreComponentsWebAssemblyLeptonXThemeBundlingModule),
    typeof(AbpAspNetCoreComponentsWebAssemblyLeptonXThemeModule))]
public class MyProjectBlazorModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        Configure<LeptonXThemeBlazorOptions>(options =>
        {
            options.Layout = LeptonXBlazorLayouts.SideMenu;
            options.MobileMenuSelector = items =>
                items.Where(x => x.MenuItem.Name == "MyProject.Home");
        });
    }
}
```

Angular standalone app (LeptonX commercial example, src/app/app.config.ts):
```typescript
import { ApplicationConfig } from '@angular/core';
import { provideThemeLeptonX } from '@volosoft/abp.ng.theme.lepton-x';
import { provideSideMenuLayout } from '@volosoft/abp.ng.theme.lepton-x/layouts';

export const appConfig: ApplicationConfig = {
  providers: [
    provideSideMenuLayout(),   // or provideTopMenuLayout()
    provideThemeLeptonX(),
  ],
};
```
For LeptonX Lite, the same shape applies with the @abp/ng.theme.lepton-x package and an additional provideAccountLayout() when using ROPC. For Basic, replace with provideThemeBasicConfig() from @abp/ng.theme.basic (or import ThemeBasicModule.forRoot() in NgModule-based apps).

## Common mistakes

### Leaving the legacy Lepton (v1) module references in [DependsOn] when migrating to LeptonX

**Why wrong:** AbpAspNetCoreMvcUiLeptonThemeModule and LeptonThemeManagementWebModule are deprecated and conflict with AbpAspNetCoreMvcUiLeptonXThemeModule, registering two ITheme implementations and breaking layout resolution.

**Correct pattern:** Remove the legacy types from [DependsOn] and add only typeof(AbpAspNetCoreMvcUiLeptonXThemeModule); rebuild and clear bin/obj before running.

```json
[DependsOn(typeof(AbpAspNetCoreMvcUiLeptonXThemeModule))] // not AbpAspNetCoreMvcUiLeptonThemeModule
```

### Adding only AbpAspNetCoreComponentsWebAssemblyLeptonXThemeModule (without the *Bundling module) in a Blazor WASM client

**Why wrong:** The bundling module is what registers the theme's CSS/JS assets with the WebAssembly bundler; without it pages render unstyled and asset URLs 404.

**Correct pattern:** Always pair the two modules — typeof(AbpAspNetCoreComponentsWebAssemblyLeptonXThemeBundlingModule) AND typeof(AbpAspNetCoreComponentsWebAssemblyLeptonXThemeModule) — same rule applies to LeptonX Lite (its *.Bundling counterpart).

```json
[DependsOn(typeof(AbpAspNetCoreComponentsWebAssemblyLeptonXThemeBundlingModule), typeof(AbpAspNetCoreComponentsWebAssemblyLeptonXThemeModule))]
```

### Editing files inside the NuGet-resolved theme folder (e.g., copying the package contents to disk and editing) instead of using the virtual file system override

**Why wrong:** NuGet restores wipe those edits, and PR/CI builds will produce different output than local; the changes are also invisible to abp-cli source-extract tooling.

**Correct pattern:** Place a same-path .cshtml/.razor under your project's Themes/<ThemeName>/... folder so the ABP virtual file system overlays the theme file at runtime, or run 'abp get-source Volo.Abp.LeptonXTheme' to clone the source into the solution as a local module.

```csharp
// e.g., create MyProject.Web/Themes/LeptonX/Layouts/Account.cshtml to override the LeptonX login layout
```

### Importing styles in @abp/ng.theme.basic without also providing a layout in a standalone Angular app

**Why wrong:** ThemeBasicModule.forRoot() / provideThemeBasicConfig() registers layout components against @abp/ng.core but does not pick a default; without the corresponding layout provider the router falls through to a blank shell.

**Correct pattern:** Provide both the theme and a layout: provideThemeBasicConfig() plus the appropriate layout/component registration; for LeptonX/LeptonX Lite use provideSideMenuLayout() or provideTopMenuLayout() alongside provideThemeLeptonX().

```csharp
providers: [ provideSideMenuLayout(), provideThemeLeptonX() ] // or provideThemeBasicConfig() for Basic
```

### Setting --lpx-* variables on a component instead of on :root or .lpx-theme-* and expecting the theme switcher to honor them

**Why wrong:** LeptonX swaps a class on the html element when the user changes style; if your overrides live on a leaf component selector they do not flip with the active style, leading to inconsistent dark/light branding.

**Correct pattern:** Override on :root for variables that should apply to every style; scope to :root .lpx-theme-dark / .lpx-theme-light / .lpx-theme-dim for variant-specific values, mirroring how LeptonX itself defines them.

```csharp
:root { --lpx-brand: #edae53; } :root .lpx-theme-dark { --lpx-brand: #4dd0e1; }
```

### Hard-coding LeptonXStyleNames.Dim/Dark/Light strings in LeptonX Lite code paths

**Why wrong:** LeptonX Lite ships a single visual style and does not surface a style switcher; LeptonXStyleNames lives in the commercial package namespace, so referencing it from a Lite-only project causes compile errors.

**Correct pattern:** Treat 'multiple styles + LeptonXThemeOptions.DefaultStyle' as a LeptonX-only feature; in Lite, customize the single visual through --lpx-* CSS variables and theme component overrides.

```csharp
// LeptonX Lite: do not Configure<LeptonXThemeOptions>(); brand via :root { --lpx-brand: ... } in global-styles.css
```

### Forgetting to update Routes.razor when swapping between LeptonX and LeptonX Lite in Blazor

**Why wrong:** Each theme ships a distinct MainLayout under its own namespace (Volo.Abp.AspNetCore.Components.Web.LeptonXTheme.Themes.LeptonX vs .LeptonXLiteTheme.Themes.LeptonXLite); leaving the old @using leaves Blazor resolving to the previous layout type and you'll see double headers or unstyled pages.

**Correct pattern:** When switching, update the @using directive in Routes.razor and Imports.razor to the new theme's namespace and confirm DefaultLayout points at typeof(MainLayout).

```csharp
@using Volo.Abp.AspNetCore.Components.Web.LeptonXTheme.Themes.LeptonX // replace .LeptonXLiteTheme.Themes.LeptonXLite
```

## Version pins (ABP 10.3)

- ABP 10.3 ships LeptonX Lite as the default theme for free templates and LeptonX as the default for commercial (Team/Business/Enterprise) templates — verified against the abp.io/docs/10.3/ui-themes index.
- LeptonXThemeOptions.DefaultStyle defaults to LeptonXStyleNames.System (follow OS preference); System falls back to Dim when the OS does not advertise a preference. [uncertain] — Search results phrase this as 'default value of Dim'; the System sentinel is documented but the runtime fallback target may differ across patch releases.
- Volo.Abp.AspNetCore.Components.WebAssembly.LeptonXTheme(.Bundling) packages were referenced in the 10.3 docs with the --prerelease flag in install commands; in stable 10.3 this flag is no longer required for GA versions. [uncertain] — flag presence depends on whether you target a GA or preview release of 10.3.
- Angular package split: @abp/ng.theme.lepton-x is the LeptonX Lite package; @volosoft/abp.ng.theme.lepton-x is the commercial LeptonX package — these are not interchangeable and have separate provider names.
- @abp/ng.theme.basic supports both the legacy ThemeBasicModule.forRoot() NgModule pattern and the standalone provideThemeBasicConfig() pattern; ABP 10.3 templates default to standalone, but module-based codebases still work.
- ThemeName attribute names ('Basic', 'LeptonXLite', 'LeptonX') are stable identifiers stored in user/tenant settings — renaming or duplicating themes invalidates persisted DefaultThemeName values across upgrades.
- BasicTheme is intentionally minimal and is the recommended starting point for custom themes; abp get-source @abp/ng.theme.basic --with-source-code (Angular) and abp get-source Volo.Abp.AspNetCore.Mvc.UI.Theme.Basic (MVC) extract the source for forking.
- LeptonX Lite framework-specific docs were reachable under /docs/10.3/ at verification time; the Basic theme framework-specific subpages (.../basic-theme/asp-net-core, .../blazor, .../angular) returned 404 under /10.3/ on 2026-05-10, so /latest/ content was used for those — paths and module names match historical 10.3 releases.

## Cross-references

**Sibling UI references (in this skill):**
- [references/ui/mvc-razor-pages.md](./mvc-razor-pages.md) — load when the user is on the MVC stack and asks about layout overrides, bundling, or virtual file system theme paths
- [references/ui/blazor.md](./blazor.md) — load for Blazor-specific component replacement, Routes.razor wiring, and bundling-module pairing required by LeptonX/LeptonX Lite
- [references/ui/angular.md](./angular.md) — load for Angular standalone-app provider wiring, ReplaceableComponentsService, and eThemeLeptonXComponents replacement keys
- [references/ui/overview.md](./overview.md) — load when the user is comparing UI stacks and which themes ship per stack

**Cross-skill references (in the abp-backend skill):**
- See **modularity / DependsOn** in the abp-backend skill (`framework/architecture-modularity.md`) — load when explaining how AbpAspNetCoreMvcUi*ThemeModule self-registers via [DependsOn] and AbpThemingOptions.Themes.Add
- See **virtual file system** in the abp-backend skill (`framework/architecture-modularity.md`) — load when the user wants to override a theme's .cshtml/.razor file by shadowing the path inside Themes/<Theme>/...
- See **localization / IStringLocalizer** in the abp-backend skill (`framework/fundamentals.md`) — load when customizing LeptonXThemeOptions.Styles entries that take LocalizableString labels

**External docs:**
- Bootstrap 5 — https://getbootstrap.com/docs/5.3/ — every ABP theme is built on Bootstrap 5; Basic adds nothing on top, LeptonX/LeptonX Lite enhance the same primitives
- Bootstrap Icons — https://icons.getbootstrap.com/ — required when adding LeptonX style entries (icon classes like 'bi bi-circle-fill') and is an explicit npm peer for @abp/ng.theme.lepton-x
- ASP.NET Core MVC Razor Layouts — https://learn.microsoft.com/aspnet/core/mvc/views/layout — backing concept for ITheme.GetLayout returning .cshtml paths
- Blazor Layouts and Routing — https://learn.microsoft.com/aspnet/core/blazor/components/layouts — context for the LeptonX Routes.razor wiring (DefaultLayout, AuthorizeRouteView)
- Angular Standalone Application Configuration — https://angular.dev/guide/standalone-components — context for the provideThemeLeptonX() / provideSideMenuLayout() pattern in app.config.ts
- MDN: CSS Custom Properties — https://developer.mozilla.org/en-US/docs/Web/CSS/Using_CSS_custom_properties — backs the --lpx-* override pattern

## Sources (ABP 10.3)

- [https://abp.io/docs/10.3/ui-themes](https://abp.io/docs/10.3/ui-themes) — UI Themes overview index, listing the three official themes (LeptonX, LeptonX Lite, Basic) and which UI stacks they support
- [https://abp.io/docs/10.3/ui-themes/lepton-x](https://abp.io/docs/10.3/ui-themes/lepton-x) — LeptonX (commercial) theme overview: Bootstrap 5 base, Dim/Dark/Light/System styles, NuGet/NPM package list, abp get-source instructions, settings tab integration
- [https://abp.io/docs/10.3/ui-themes/lepton-x/mvc](https://abp.io/docs/10.3/ui-themes/lepton-x/mvc) — LeptonX MVC/Razor Pages: AbpAspNetCoreMvcUiLeptonXThemeModule dependency, LeptonXThemeOptions, LeptonXThemeMvcOptions, LeptonXThemeBundles, layout overrides under Themes/LeptonX/Layouts
- [https://abp.io/docs/10.3/ui-themes/lepton-x/blazor](https://abp.io/docs/10.3/ui-themes/lepton-x/blazor) — LeptonX Blazor: AbpAspNetCoreComponentsWebAssemblyLeptonXTheme(.Bundling)Module dependencies, LeptonXThemeBlazorOptions, Routes.razor wiring, component replacement attributes
- [https://abp.io/docs/10.3/ui-themes/lepton-x/angular](https://abp.io/docs/10.3/ui-themes/lepton-x/angular) — LeptonX Angular: @volosoft/abp.ng.theme.lepton-x package, provideThemeLeptonX(), provideSideMenuLayout()/provideTopMenuLayout(), CSS variable theming
- [https://abp.io/docs/10.3/ui-themes/lepton-x-lite](https://abp.io/docs/10.3/ui-themes/lepton-x-lite) — LeptonX Lite overview: free/open-source counterpart to commercial LeptonX, default theme of free ABP solutions
- [https://abp.io/docs/10.3/ui-themes/lepton-x-lite/asp-net-core](https://abp.io/docs/10.3/ui-themes/lepton-x-lite/asp-net-core) — LeptonX Lite MVC: Volo.Abp.AspNetCore.Mvc.UI.Theme.LeptonXLite NuGet, AbpAspNetCoreMvcUiLeptonXLiteThemeModule, LeptonXLiteThemeBundles, LeptonXLiteToolbars
- [https://abp.io/docs/10.3/ui-themes/lepton-x-lite/blazor](https://abp.io/docs/10.3/ui-themes/lepton-x-lite/blazor) — LeptonX Lite Blazor: AbpAspNetCoreComponentsWebAssemblyLeptonXLiteTheme(.Bundling)Module dependencies, Routes.razor with Volo.Abp.AspNetCore.Components.Web.LeptonXLiteTheme.Themes.LeptonXLite namespace
- [https://abp.io/docs/10.3/ui-themes/lepton-x-lite/angular](https://abp.io/docs/10.3/ui-themes/lepton-x-lite/angular) — LeptonX Lite Angular: @abp/ng.theme.lepton-x package, provideThemeLeptonX(), provideSideMenuLayout(), provideAccountLayout(), eThemeLeptonXComponents replacement enum
- [https://abp.io/docs/10.3/ui-themes/basic-theme](https://abp.io/docs/10.3/ui-themes/basic-theme) — Basic theme overview: minimalist Bootstrap-only theme; index page only, framework-specific subpages 404 under /10.3/
- [https://abp.io/docs/latest/ui-themes/basic-theme](https://abp.io/docs/latest/ui-themes/basic-theme) — (fetched from /latest/, 10.3 path 404 for framework-specific subpages) Basic theme: foundation for custom theming, supports MVC, Blazor, Angular
- [https://abp.io/docs/10.3/framework/ui/common/leptonx-css-variables](https://abp.io/docs/10.3/framework/ui/common/leptonx-css-variables) — Catalogue of --lpx-* CSS custom properties (brand, content, navbar, card, radius, shadow) and override patterns scoped to .lpx-theme-dark/light/dim
- [https://abp.io/docs/latest/framework/ui/mvc-razor-pages/theming](https://abp.io/docs/latest/framework/ui/mvc-razor-pages/theming) — (fetched from /latest/, complements /10.3/ docs) AbpThemingOptions, ITheme interface, ThemeName attribute, IThemeSelector and DefaultThemeSelector, StandardLayouts constants in Volo.Abp.AspNetCore.Mvc.UI.Theming

Last verified: 2026-05-10
