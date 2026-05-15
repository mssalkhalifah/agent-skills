# UI Frameworks — Overview & Selection

> Survey of the five UI options ABP 10.3 officially supports — MVC/Razor Pages, Blazor (Server/WASM/WebApp/MAUI Blazor), Angular, React Native, and .NET MAUI — and the cross-cutting UI infrastructure (theming, menus, layouts, dynamic client proxies) that all five share.

## When to load this reference

- User is starting a new ABP solution and asks 'which UI should I pick?' or 'MVC vs Blazor vs Angular for ABP'.
- User asks what UI frameworks ABP supports or whether a given UI (React, Vue, MAUI, Flutter, mobile) is supported in ABP 10.3.
- User asks for differences between Blazor Server, Blazor WebAssembly, Blazor WebApp, and MAUI Blazor inside ABP.
- User wants to add a mobile companion app to an existing ABP backend and is choosing between React Native and .NET MAUI.
- User wants to understand what ABP's theming system is and how the same theme abstractions work across UI stacks.
- User asks about replacing or customizing ABP UI components/menus/toolbars at a conceptual level (specifics live in per-UI references).
- Routing question: deciding which deeper UI reference (mvc-overview, blazor-overview, angular-overview, react-native-overview, maui-overview) to load next.

**Audience:** ABP developers and architects choosing or comparing UI stacks for an ABP 10.3 solution — backend-leaning developers who need a map of the UI options and the per-stack trade-offs before drilling into a specific UI's reference.

## Key concepts

- MVC / Razor Pages UI — Volo.Abp.AspNetCore.Mvc.UI.* — Server-rendered UI built on ASP.NET Core MVC and Razor Pages; ABP recommends Razor Pages for UI and MVC controllers for HTTP APIs; reach for it when you want a server-first, Bootstrap-themed admin app with minimal JS toolchain.
- Blazor Server hosting — Volo.Abp.AspNetCore.Components.Server / Web.* — Renders Blazor components on the server and uses a SignalR circuit; reach for it when you want full C#, real-time UI, and a single deployable host with sticky connections.
- Blazor WebAssembly (WASM) hosting — Volo.Abp.AspNetCore.Components.WebAssembly.* — Ships .NET runtime + app to the browser, calls APIs over HTTP; reach for it when you want a SPA-style C# client decoupled from the server.
- Blazor WebApp hosting (.NET 8+) — Volo.Abp.AspNetCore.Components.WebApp.* — .NET 8 unified Blazor model that combines Server and WebAssembly render modes; reach for it on new 10.3 solutions targeting .NET 8/9 that want per-component render-mode control.
- MAUI Blazor — uses Blazor components inside a .NET MAUI shell — Reach for it when you want to reuse Blazor UI as a native cross-platform desktop/mobile app shell.
- Angular UI — @abp/ng.core, @abp/ng.theme.shared, @abp/ng.account, @abp/ng.identity, @abp/ng.tenant-management, @abp/ng.setting-management, @abp/ng.feature-management, @abp/ng.permission-management, @abp/ng.components — TypeScript SPA stack with NgRx-style ConfigStateService, replaceable components, dynamic forms, and ABP service-proxy generation; reach for it when the team is Angular-skilled and wants a cleanly decoupled SPA against ABP HTTP APIs.
- React Native template — Expo-based RN app under react-native/ folder of solution — Mobile (iOS/Android) startup template with OpenIddict auth wired in; available on Team license and above; scaffolded via ABP Studio or `abp new ... -m react-native`.
- .NET MAUI template — Volo.Abp.Maui.* and HttpApi.Client.* packages inside a MAUI head project — Native cross-platform (Android/iOS/Mac Catalyst/Windows) C# client; uses MVVM with ValidatableObject<T>, C# dynamic client proxies, and Secure Storage for tokens.
- Theming abstraction — Volo.Abp.AspNetCore.Mvc.UI.Theming.ITheme / [ThemeName] / AbpThemingOptions / IThemeSelector — UI-stack-agnostic concept where modules ship theme-independent UI and the final app picks one of three official themes (Basic OSS, Lepton commercial, LeptonX commercial+lite); reach for it whenever you brand or restyle an ABP app.
- Standard layouts — Volo.Abp.AspNetCore.Mvc.UI.Theming.StandardLayouts (Application, Account, Empty) — Named layout slots that every theme is expected to provide; reach for them when overriding a layout or implementing a new theme.
- Branding / menu / toolbar / alert services — IBrandingProvider, IMenuManager (StandardMenus.Main), IToolbarManager + IToolbarContributor, IAlertManager — Cross-UI services exposed in MVC and Blazor for app name/logo, menus, top-bar items, and page-level alerts; reach for them to inject UI elements without forking the theme.
- Dynamic client proxies — JS proxies (MVC), C# proxies (Blazor/MAUI), generated TS service-proxies (Angular), JS proxies (React Native) — Abstraction that lets every UI stack consume ABP application-service interfaces remotely as if they were local; reach for it instead of hand-writing HTTP/serialization code.
- UI message/notification services — IUiMessageService, IUiNotificationService, IAlertManager (Blazor); abp.message / abp.notify / Toaster (MVC); ToasterService / ConfirmationService (Angular) — Common service surface across UI stacks for confirmations, info/error toasts, and inline alerts.

## Configuration pattern

UI selection is primarily a *template* decision (chosen at `abp new` time) rather than runtime configuration; once chosen, each UI stack is wired up by depending on the matching ABP module in a DependsOn attribute and configuring its options inside AbpModule.ConfigureServices. Example for MVC/Razor Pages with the Basic theme:

```csharp
[DependsOn(
    typeof(AbpAspNetCoreMvcUiBasicThemeModule),
    typeof(AbpAspNetCoreMvcUiThemeSharedModule)
)]
public class MyWebModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        Configure<AbpThemingOptions>(options =>
        {
            options.Themes.Add<BasicTheme>();
            options.DefaultThemeName = BasicTheme.Name;
        });

        Configure<AbpBundlingOptions>(options =>
        {
            // global bundles here
        });
    }
}
```

For Blazor (Server/WASM/WebApp), the host project depends on the matching components module (AbpAspNetCoreComponentsServerModule, AbpAspNetCoreComponentsWebAssemblyModule, or the WebApp module) plus the chosen theme module (e.g., AbpAspNetCoreComponentsWebAssemblyBasicThemeModule). For Angular, configuration is in `app.module.ts` via `CoreModule.forRoot({...})` and `ThemeSharedModule.forRoot({...})` plus environment.ts. For React Native and MAUI the template ships with a pre-wired backend URL/Authority that you edit in a single config file (Environment.js or appsettings.json). Theme-name and default-theme are the two knobs every server-rendered UI exposes through AbpThemingOptions; UI module dependencies (which packages you reference) are how you select the actual stack.

## Code examples

### Register an MVC UI module and the Basic theme

_Wiring the MVC/Razor Pages UI with the open-source Basic theme in the Web layer's AbpModule._

```json
[DependsOn(
    typeof(AbpAspNetCoreMvcUiBasicThemeModule),
    typeof(AbpAspNetCoreMvcUiThemeSharedModule)
)]
public class MyProjectWebModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        Configure<AbpThemingOptions>(options =>
        {
            options.Themes.Add<BasicTheme>();
            options.DefaultThemeName = BasicTheme.Name;
        });
    }
}
```

**Key lines:** AbpAspNetCoreMvcUiBasicThemeModule pulls in the Basic theme assets; AbpThemingOptions.Themes.Add<BasicTheme>() registers it; DefaultThemeName tells the IThemeSelector which one to activate when no explicit selection has been made.

### Register the Blazor WebAssembly UI with the Basic theme

_Wiring a Blazor WASM client to render with ABP's Basic theme._

```json
[DependsOn(
    typeof(AbpAspNetCoreComponentsWebAssemblyBasicThemeModule)
)]
public class MyProjectBlazorModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        Configure<AbpThemingOptions>(options =>
        {
            options.Themes.Add<BasicTheme>();
            options.DefaultThemeName = BasicTheme.Name;
        });
    }
}
```

**Key lines:** Blazor mirrors the MVC theming API — same AbpThemingOptions, same theme classes — but pulls them from the WebAssembly-flavored package. Swap the BasicTheme module for the LeptonXTheme module to switch themes; the ConfigureServices code is otherwise unchanged.

### Bootstrap an Angular SPA against an ABP backend

_Configuring @abp/ng.core in app.module.ts so the Angular client knows the ABP backend's API URL and OpenIddict authority._

```csharp
@NgModule({
  imports: [
    CoreModule.forRoot({
      environment,
      registerLocaleFn: registerLocale(),
    }),
    ThemeSharedModule.forRoot({
      httpErrorConfig: { errorScreen: { component: HttpErrorComponent } },
    }),
    AccountConfigModule.forRoot({ redirectUrl: '/' }),
    IdentityConfigModule.forRoot(),
    TenantManagementConfigModule.forRoot(),
    SettingManagementConfigModule.forRoot(),
  ],
  bootstrap: [AppComponent],
})
export class AppModule {}
```

**Key lines:** CoreModule.forRoot wires the ConfigStateService, HTTP interceptors, and OAuth client using the environment object (apis.default.url, oAuthConfig.issuer, etc.); ThemeSharedModule.forRoot installs the shared theme/error UI; the *ConfigModule.forRoot calls register the ABP feature modules' lazy-loaded routes.

### Scaffold a React Native mobile companion against an existing ABP backend

_Creating an RN app that authenticates against the ABP backend via OpenIddict._

```bash
abp new Acme.BookStore -csf -u mvc -m react-native
# then in react-native/Environment.js point apiUrl/issuer at the backend
# during dev, run Kestrel on http://0.0.0.0:44300 and disable HTTPS-only OpenIddict
```

**Key lines:** `-m react-native` adds an Expo-based RN folder alongside the chosen web UI (-u mvc here). Environment.js holds apiUrl and oAuthConfig.issuer; the backend must accept HTTP from emulators (set OpenIddictServerOptions to allow non-HTTPS in Development) and bind Kestrel to 0.0.0.0 so Android/iOS simulators can reach it.

### Switch the default theme without forking layouts

_Changing an existing MVC app from Basic theme to LeptonX Lite by editing one DependsOn line and one option._

```json
[DependsOn(
    typeof(AbpAspNetCoreMvcUiLeptonXLiteThemeModule),
    typeof(AbpAspNetCoreMvcUiThemeSharedModule)
)]
public class MyProjectWebModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        Configure<AbpThemingOptions>(options =>
        {
            options.Themes.Add<LeptonXLiteTheme>();
            options.DefaultThemeName = LeptonXLiteTheme.Name;
        });
    }
}
```

**Key lines:** Because all themes implement ITheme + StandardLayouts, swapping the theme module reference and the registered theme type is sufficient — page code does not change. The same swap-style customization works for Blazor with the corresponding *Components.WebAssembly.LeptonXLiteTheme module.

## Common mistakes

### Treating the UI choice as a runtime switch and trying to add a second UI later by hand.

**Why wrong:** ABP wires the UI through template-time package selection, MainModule DependsOn graph, theme module references, and (for Angular/RN/MAUI) sibling project folders. There is no runtime 'pick a UI' API.

**Correct pattern:** Pick the primary UI at `abp new` (or via ABP Studio's Add-Module / Add-Mobile-App). To add a second UI, run `abp new` with the desired -u/-m to scaffold a sibling project, then point it at the existing HttpApi.Host. Do not hand-port files between solutions.

### Mixing MVC controllers with Razor Pages for the *UI layer* on the rationale that 'they are both ASP.NET Core'.

**Why wrong:** ABP's pre-built modules and docs assume Razor Pages for UI and MVC controllers for HTTP APIs; mixing leads to inconsistent layout/theming wiring and broken module overrides.

**Correct pattern:** Use Razor Pages for end-user UI and MVC controllers (or auto-generated dynamic API controllers from application services) for HTTP APIs.

### Choosing Blazor Server for an offline/PWA-style mobile-friendly app.

**Why wrong:** Blazor Server requires an always-on SignalR circuit; loss of connectivity tears down state and degrades UX dramatically on mobile networks. It also doesn't deploy as a static SPA.

**Correct pattern:** Pick Blazor WebAssembly or Blazor WebApp (.NET 8+) for SPA-style behavior, or Angular for a battle-tested SPA. Use Blazor Server only when you can guarantee a stable LAN/WAN connection and want fast initial loads.

### Pointing a React Native or MAUI emulator at the backend's HTTPS dev endpoint and being surprised when login fails.

**Why wrong:** Mobile emulators don't trust the dev HTTPS certificate, and OpenIddict refuses non-HTTPS by default. The result is silent token-endpoint failures.

**Correct pattern:** In Development, bind Kestrel to `http://0.0.0.0:<port>`, set OpenIddict's RequireHttpsMetadata/disable-https-only flags as documented, and put the same HTTP URL in Environment.js (RN) or appsettings.json (MAUI).

### Forking the theme to add a logo/menu item or hide a toolbar item.

**Why wrong:** Forking divorces you from upstream theme upgrades and bug fixes. ABP exposes extension points for exactly these cases.

**Correct pattern:** Implement IBrandingProvider for app name/logo, IMenuContributor (registered via AbpNavigationOptions) for menu items, and IToolbarContributor for toolbar items. Override only the theme's Razor file when no extension point exists, and use ABP's virtual-file overrides instead of editing the package source.

### Calling raw HTTP from JS/TS/C# clients instead of using ABP's dynamic client proxies.

**Why wrong:** You re-invent serialization, error envelopes, auth-header attachment, current-tenant header, and break with every backend rename.

**Correct pattern:** Use the dynamic JS proxies (MVC), C# proxies via IRemoteService (Blazor/MAUI), or generate TS service-proxies (Angular) so the client tracks the application-service contract automatically.

## Version pins (ABP 10.3)

- ABP 10.3 documents Blazor WebApp (the .NET 8 unified hosting model) alongside Blazor Server and Blazor WebAssembly; older 6.x/7.x docs do not list it. New 10.3 solutions targeting .NET 8/9 should prefer Blazor WebApp over standalone Server or WASM.
- All three official themes (Basic / Lepton / LeptonX) are present in 10.3; LeptonX is the current default in commercial templates while Basic is the only OSS option. [uncertain] — exact default per template version.
- React Native template is gated to Team license or higher in 10.3; the OSS Community license does not include it.
- React Native template targets Node.js v20.11+ in 10.3 docs; older Node versions fail Expo's checks.
- .NET MAUI template ships C# client proxies and MVVM with ValidatableObject<T>; no Blazor Hybrid is bundled in the default MAUI template (MAUI Blazor is documented separately under Blazor). [uncertain] — whether MAUI template in 10.3 has been promoted out of preview status.
- Angular packages are published as @abp/ng.* on npm and are versioned to track the .NET ABP version; mixing @abp/ng.* 10.3 with a non-10.3 backend is unsupported.
- AbpThemingOptions.Themes / DefaultThemeName / IThemeSelector / [ThemeName] / StandardLayouts surface is stable in 10.3 and identical in MVC and Blazor; expect parity to continue but treat any future cross-stack theme APIs as [uncertain] for forward compatibility.

## Cross-references

**Sibling UI references (in this skill):**
- [references/ui/mvc-razor-pages.md](./mvc-razor-pages.md) — Drill-down for MVC/Razor Pages specifics (tag helpers, abp-dynamic-form, bundling). Cross-link triggered when the user picks MVC or asks about Razor Pages, tag helpers, or server-rendered ABP UI.
- [references/ui/blazor.md](./blazor.md) — Drill-down for Blazor hosting models, Blazorise components, IUiMessageService/IUiNotificationService. Triggered by any Blazor-Server-vs-WASM or component question.
- [references/ui/angular.md](./angular.md) — Drill-down for @abp/ng.* packages, ConfigStateService, replaceable components, service proxy generation. Triggered by Angular SPA questions.
- [references/ui/mobile.md](./mobile.md) — Drill-down for the React Native (Expo) and .NET MAUI mobile templates, mobile auth, Environment.js / appsettings.json. Triggered by RN/MAUI/mobile-companion questions.
- [references/ui/themes.md](./themes.md) — LeptonX / LeptonX Lite / Basic theme details, --lpx-* CSS variables, layout overrides, per-stack provider wiring.

**Cross-skill references (in the abp-backend skill):**
- See **modularity / DependsOn** in the abp-backend skill (`framework/architecture-modularity.md`) — Theming and UI modules are ABP modules; cross-link when explaining DependsOn-based UI module wiring.
- See **authorization** in the abp-backend skill (`framework/authorization.md`) — UI stacks consume the same permission system; cross-link when questions touch UI-side authorization (abp-tags / *abpPermission / PermissionService).
- See **localization / IStringLocalizer** in the abp-backend skill (`framework/fundamentals.md`) — All UI stacks consume ABP localization; cross-link for language-switcher / IStringLocalizer questions.
- See **multi-tenancy** in the abp-backend skill (`framework/multi-tenancy.md`) — Tenant resolution backs the per-tenant theme/branding paths surfaced in UI.

**External docs:**
- ASP.NET Core MVC / Razor Pages — https://learn.microsoft.com/aspnet/core/mvc/overview — Underlying framework for ABP's MVC UI; referenced for routing, tag helpers, model binding.
- ASP.NET Core Blazor — https://learn.microsoft.com/aspnet/core/blazor/ — Underlying framework for Blazor Server/WASM/WebApp hosting models that ABP layers over.
- Blazorise — https://blazorise.com/docs — Component library used as the base UI kit in ABP Blazor; ABP wraps it for theming.
- Bootstrap 5 — https://getbootstrap.com/docs/5.0/ — CSS framework underlying Basic, Lepton, and LeptonX themes across MVC and Blazor.
- Angular — https://angular.dev — Framework underlying @abp/ng.* packages; relevant for module wiring, routing, RxJS.
- Expo (React Native) — https://docs.expo.dev — Toolchain ABP's React Native template is built on; referenced for emulator setup, build, EAS.
- .NET MAUI — https://learn.microsoft.com/dotnet/maui/ — Underlying framework for the ABP MAUI template; referenced for platform-specific setup (Android ADB, iOS provisioning).
- OpenIddict — https://documentation.openiddict.com — Auth server every UI stack authenticates against; relevant when configuring HTTPS-only flags for mobile emulators.

## Sources (ABP 10.3)

- [https://abp.io/docs/10.3/framework/ui — Top-level ABP UI Options page enumerating supported UI frameworks (MVC/Razor Pages, Blazor, Angular, React Native, MAUI) and cross-cutting UI infrastructure (theming, navigation, forms, localization).](https://abp.io/docs/10.3/framework/ui)
- [https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/overall — MVC/Razor Pages UI overview: when to use Razor Pages vs MVC, dynamic JS proxies, tag helpers, abp-dynamic-form, bundling, modal/widget systems.](https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/overall)
- [https://abp.io/docs/10.3/framework/ui/blazor/overall — Blazor UI overview: Blazor Server, Blazor WebAssembly, Blazor WebApp (.NET 8 unified), MAUI Blazor; dynamic C# client proxies; UI services (IUiMessageService, IUiNotificationService, IAlertManager).](https://abp.io/docs/10.3/framework/ui/blazor/overall)
- [https://abp.io/docs/10.3/framework/ui/angular/overview — Angular UI overview: @abp/ng.* package family, configuration services, authorization, HTTP, localization, dynamic forms, replaceable components.](https://abp.io/docs/10.3/framework/ui/angular/overview)
- [https://abp.io/docs/10.3/framework/ui/react-native — React Native template overview: Expo-based startup template, OpenIddict integration, ABP Studio/CLI scaffolding, available with Team or higher license.](https://abp.io/docs/10.3/framework/ui/react-native)
- [https://abp.io/docs/10.3/framework/ui/maui — .NET MAUI template overview: MVVM with ValidatableObject<T>, secure token storage, C# client proxies, prebuilt Home/Users/Tenants/Settings pages.](https://abp.io/docs/10.3/framework/ui/maui)
- [https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/theming — ITheme interface, [ThemeName] attribute, AbpThemingOptions, IThemeSelector, StandardLayouts (Application/Account/Empty).](https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/theming)
- [https://abp.io/docs/10.3/framework/ui/blazor/theming — Blazor theme building blocks: Volo.Abp.AspNetCore.Components.WebAssembly.Theming, Blazorise + Bootstrap base, IBrandingProvider, IMenuManager, IToolbarManager, IToolbarContributor, IAlertManager, IBundleContributor.](https://abp.io/docs/10.3/framework/ui/blazor/theming)

Last verified: 2026-05-10
