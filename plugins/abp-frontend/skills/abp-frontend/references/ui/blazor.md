# UI — Blazor (Server / WebApp / WebAssembly)

> How to build, configure, and customize Blazor UIs in ABP across Server, WebApp, WebAssembly, and MAUI Blazor hosting models, including navigation, forms, localization, theming, layout, auth, notifications, error handling, component overrides, global features, routing, and PWA setup.

## When to load this reference

Load when the user is: building or modifying any *.Blazor / *.Blazor.Server / *.Blazor.WebApp / *.Blazor.Client / *.Blazor.WebAssembly project; choosing a Blazor hosting model for an ABP solution; defining or extending a navigation menu via IMenuContributor; wiring forms with Blazorise validators; configuring IStringLocalizer in Razor components or AbpComponentBase usage; selecting/customizing the LeptonX, LeptonX Lite, or Basic theme on Blazor; setting PageLayout.Title / BreadcrumbItems / ToolbarItems / MenuItemName; debugging OIDC sign-in/sign-out, AddAbpOpenIdConnect, cookie auth in Blazor Server; using [Authorize] / <AuthorizeView> / IAuthorizationService.IsGrantedAsync; reading ICurrentUser in a component; raising toasts via IUiNotificationService; throwing UserFriendlyException for client-visible errors; replacing a default ABP component with [ExposeServices] + [Dependency(ReplaceServices=true)]; gating UI on GlobalFeatureManager; registering module assemblies with AbpRouterOptions.AdditionalAssemblies; or scaffolding a Blazor WASM PWA via `abp new -t blazor --pwa`.

**Audience:** ABP developers building Blazor UIs (Server, WebApp, WebAssembly, or MAUI Blazor) on top of an ABP 10.3 solution, including module authors who ship reusable Razor class libraries.

## Key concepts

- Hosting models: Blazor Server (server-side, SignalR), Blazor WebAssembly (browser WASM), Blazor WebApp (.NET 8 unified Server+WASM), MAUI Blazor (cross-platform native). Reach for Server when latency is low and you want full server access; WebAssembly for offline/PWA scenarios; WebApp when you want per-component render-mode choice; MAUI Blazor for native cross-platform.
- AbpComponentBase (Volo.Abp.AspNetCore.Components) - Recommended base class for Blazor components. Pre-injects L (IStringLocalizer), CurrentUser (ICurrentUser), Notify (IUiNotificationService), AuthorizationService (IAuthorizationService). Inherit when a component needs any of these services.
- Blazorise - The 3rd-party component library ABP standardizes on (pre-installed in all themes alongside Bootstrap, FontAwesome, Flag Icon). Use Blazorise components (TextEdit, Validation, DataGrid, Modal, etc.) instead of inventing your own.
- IMenuContributor (Volo.Abp.UI.Navigation) - Module-level menu builder. Implement ConfigureMenuAsync(MenuConfigurationContext) and register via Configure<AbpNavigationOptions>(o => o.MenuContributors.Add(new MyContributor())). Use StandardMenus.Main and StandardMenus.User to target the main and user (top-right) menus.
- ApplicationMenu / ApplicationMenuItem / ApplicationMenuGroup (Volo.Abp.UI.Navigation) - Tree-shaped menu model. ApplicationMenuItem properties: name, displayName, url, icon, order, target, elementId, cssClass, groupName, requiredPermissionName. Reach for it inside an IMenuContributor.
- PageLayout (Volo.Abp.AspNetCore.Components.Web.Theming.PageToolbars / Layout) - Per-page layout state holder injected into Razor pages. Set PageLayout.Title, PageLayout.MenuItemName, PageLayout.BreadcrumbItems, PageLayout.ToolbarItems in OnInitializedAsync.
- BlazoriseUI.BreadcrumbItem - Breadcrumb DTO (text, url) added to PageLayout.BreadcrumbItems.
- PageToolbars.PageToolbarItem - Toolbar item that wraps a Blazor component type and is rendered in the page header toolbar.
- IStringLocalizer<TResource> (Microsoft.Extensions.Localization) - Localization service. Inject via @inject IStringLocalizer<MyResource> L or use the L property from AbpComponentBase. Resources defined on the server are reused on the Blazor client automatically.
- IBrandingProvider (Volo.Abp.AspNetCore.Components.Web.Theming) - Returns app name and logo URLs (LogoUrl, LogoReverseUrl). Override to customize branding.
- IMenuManager - Theme/runtime service that retrieves a built menu by name (e.g., await menuManager.GetAsync(StandardMenus.Main)).
- IToolbarManager / IToolbarContributor - Builds the global toolbar (top-right user/notifications/theme switcher area). Themes implement IToolbarContributor.
- ILanguageProvider - Lists configured languages, used by language switcher components.
- AuthenticationStateProvider (Microsoft.AspNetCore.Components.Authorization) - Standard ASP.NET Core type ABP plugs into. Used by [Authorize], <AuthorizeView>, and CascadingAuthenticationState.
- ICurrentUser (Volo.Abp.Users) - Authenticated-user accessor. Properties: Id, UserName, Email, IsAuthenticated, Name, SurName, TenantId, Roles, plus FindClaim/FindClaims/GetAllClaims helpers. Use anywhere you need the current principal.
- IAuthorizationService (Microsoft.AspNetCore.Authorization) - ASP.NET Core service used for IsGrantedAsync("PolicyName") / CheckAsync. ABP routes permission names through this same API.
- IUiNotificationService (Volo.Abp.AspNetCore.Components.Web) - Toast notifications: Info / Success / Warn / Error (each accepts message, title, options action). AbpComponentBase exposes it as Notify.
- IUiMessageService - Modal message dialogs (Info / Success / Warning / Error / Confirm) returning a Task<bool> for confirmations.
- IAlertManager - In-page alert region; manages a collection of inline alerts (Add, Clear).
- UserFriendlyException (Volo.Abp) - Throw to display a localized message dialog to the end user; non-friendly exceptions surface as a generic message with details in the browser console.
- GlobalFeatureManager (Volo.Abp.GlobalFeatures) - Static-instance feature gate. Use GlobalFeatureManager.Instance.IsEnabled<TFeature>() / IsEnabled("Module.Feature") to gate components and pages.
- AbpRouterOptions (Volo.Abp.AspNetCore.Components.Web) - Options for the ABP-extended Blazor Router. AdditionalAssemblies registers module assemblies that contain Blazor components/routes; AppAssembly is the host app assembly.
- Component override attributes: [ExposeServices(typeof(TBase))] (Volo.Abp.DependencyInjection) and [Dependency(ReplaceServices = true)] together with @inherits TBase replace the default component in the DI container.
- PWA scaffolding: `abp new <Name> -t blazor --pwa` generates manifest.json, service-worker.js, service-worker.published.js, icons, and ServiceWorkerAssetsManifest in the WASM project.

## Configuration pattern

Blazor-side configuration is concentrated in your Blazor module's `ConfigureServices(ServiceConfigurationContext)`. Typical wiring:

```csharp
[DependsOn(
    typeof(AbpAspNetCoreComponentsWebModule),
    typeof(AbpAutofacModule)
)]
public class MyProjectBlazorModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        // 1. Register a navigation menu contributor
        Configure<AbpNavigationOptions>(options =>
        {
            options.MenuContributors.Add(new MyProjectMenuContributor());
        });

        // 2. Register module assemblies with the Blazor router
        Configure<AbpRouterOptions>(options =>
        {
            options.AdditionalAssemblies.Add(typeof(MyProjectBlazorModule).Assembly);
        });

        // 3. Set global notification defaults
        Configure<UiNotificationOptions>(options =>
        {
            options.OkButtonText = LocalizableString.Create<MyProjectResource>("Ok");
        });

        // 4. Replace a default component (alternative to attribute-based override)
        Configure<AbpComponentReplacementOptions>(options =>
        {
            options.Replace<Branding, MyBranding>();
        });

        // 5. Add a page-toolbar contributor
        Configure<AbpPageToolbarOptions>(options =>
        {
            options.Contributors.Add(new MyToolbarContributor());
        });
    }
}
```

Notes on the options:
- `AbpNavigationOptions.MenuContributors` is an ordered list; later contributors can mutate items added by earlier ones.
- `AbpRouterOptions.AdditionalAssemblies` MUST include every Razor class library that ships @page components, otherwise routes 404.
- `UiNotificationOptions` currently exposes `OkButtonText` and `OkButtonIcon` only; per-call overrides take precedence.
- `AbpComponentReplacementOptions.Replace<TBase, TReplacement>()` is equivalent to applying [ExposeServices(typeof(TBase))] + [Dependency(ReplaceServices=true)] on TReplacement. [uncertain] - exact options class name in 10.3 may differ; the attribute-based override is the documented canonical form.

## Code examples

### Define a navigation menu contributor

_Add a CRM section with Customers and Orders items to the main menu, gated on a permission._

```csharp
using System.Threading.Tasks;
using MyProject.Localization;
using Volo.Abp.UI.Navigation;

namespace MyProject.Blazor.Menus;

public class MyProjectMenuContributor : IMenuContributor
{
    public async Task ConfigureMenuAsync(MenuConfigurationContext context)
    {
        if (context.Menu.Name != StandardMenus.Main) return;

        var l = context.GetLocalizer<MyProjectResource>();

        var crm = new ApplicationMenuItem("MyProject.Crm", l["Menu:CRM"], icon: "fa fa-briefcase");
        crm.AddItem(new ApplicationMenuItem(
            "MyProject.Crm.Customers", l["Menu:Customers"], "/crm/customers"));

        if (await context.IsGrantedAsync("MyProject.Crm.Orders"))
        {
            crm.AddItem(new ApplicationMenuItem(
                "MyProject.Crm.Orders", l["Menu:Orders"], "/crm/orders"));
        }

        context.Menu.AddItem(crm);
    }
}
```

**Key lines:** context.Menu.Name == StandardMenus.Main filters which menu is being built; context.IsGrantedAsync gates a child item on a permission; ApplicationMenuItem (name, displayName, url, icon) is the canonical constructor.

### Register the menu contributor in the Blazor module

_Wire a contributor and the router in ConfigureServices._

```csharp
public override void ConfigureServices(ServiceConfigurationContext context)
{
    Configure<AbpNavigationOptions>(options =>
    {
        options.MenuContributors.Add(new MyProjectMenuContributor());
    });

    Configure<AbpRouterOptions>(options =>
    {
        options.AdditionalAssemblies.Add(typeof(MyProjectBlazorModule).Assembly);
    });
}
```

**Key lines:** MenuContributors.Add registers the contributor singleton-style; AdditionalAssemblies makes module-hosted @page routes discoverable to the ABP-enhanced Blazor Router.

### Page using PageLayout, AbpComponentBase, and authorization

_A Customers page that sets title/breadcrumbs, requires a permission, shows a toast, and reads the current user._

```csharp
@page "/crm/customers"
@attribute [Authorize("MyProject.Crm.Customers")]
@inherits AbpComponentBase
@inject PageLayout PageLayout

<h3>@L["Menu:Customers"]</h3>
<AuthorizeView Policy="MyProject.Crm.Customers.Create">
    <Button Color="Color.Primary" Clicked="OnCreateAsync">@L["NewCustomer"]</Button>
</AuthorizeView>

@code {
    protected override Task OnInitializedAsync()
    {
        PageLayout.Title = L["Menu:Customers"];
        PageLayout.MenuItemName = "MyProject.Crm.Customers";
        PageLayout.BreadcrumbItems.Add(new BlazoriseUI.BreadcrumbItem(L["Menu:CRM"], null));
        PageLayout.BreadcrumbItems.Add(new BlazoriseUI.BreadcrumbItem(L["Menu:Customers"], "/crm/customers"));
        return Task.CompletedTask;
    }

    private async Task OnCreateAsync()
    {
        if (!await AuthorizationService.IsGrantedAsync("MyProject.Crm.Customers.Create"))
        {
            await Notify.Warning(L["NotAllowed"]);
            return;
        }
        await Notify.Success(L["Created", CurrentUser.UserName!]);
    }
}
```

**Key lines:** @inherits AbpComponentBase grants L / Notify / AuthorizationService / CurrentUser; [Authorize("...")] on @page gates the route; PageLayout sets header title, active menu, breadcrumbs.

### Throw a UserFriendlyException for a client-visible error

_Convert a server-side validation problem into a localized message dialog without writing UI code._

```csharp
if (input.Quantity <= 0)
{
    throw new UserFriendlyException(
        message: L["QuantityMustBePositive"],
        code: "MyProject:010001");
}
```

**Key lines:** ABP's exception pipeline catches UserFriendlyException, looks up its message via the localization system, and shows a modal in the Blazor UI; non-friendly exceptions surface as a generic message and the details go to the browser console.

### Replace a default component (Branding override)

_Swap the theme's Branding component for one with a custom logo and link._

```csharp
@*** File: MyBranding.razor ***@
@inherits Volo.Abp.AspNetCore.Components.Web.Theming.Themes.Basic.Components.Branding
@attribute [ExposeServices(typeof(Volo.Abp.AspNetCore.Components.Web.Theming.Themes.Basic.Components.Branding))]
@attribute [Dependency(ReplaceServices = true)]

<a href="/" class="navbar-brand">
    <img src="/images/logo.svg" alt="MyProject" height="36" />
</a>
```

**Key lines:** @inherits points to the original component type; [ExposeServices(typeof(BaseComponent))] registers the override against the base service; [Dependency(ReplaceServices = true)] tells ABP DI to swap out the original.

### PWA-enabled Blazor WebAssembly setup

_Scaffold a new Blazor WASM solution with PWA support and confirm the wired files._

```bash
abp new Acme.BookStore -t blazor --pwa

<!-- wwwroot/index.html (excerpt) -->
<link href="manifest.json" rel="manifest" />
<script>navigator.serviceWorker.register('service-worker.js');</script>

<!-- *.Blazor.csproj (excerpt) -->
<PropertyGroup>
  <ServiceWorkerAssetsManifest>service-worker-assets.js</ServiceWorkerAssetsManifest>
</PropertyGroup>
```

**Key lines:** --pwa generates manifest.json, service-worker.js (no-op in dev), and service-worker.published.js (cache-first, requires online first visit); ServiceWorkerAssetsManifest plumbs the asset hash list at publish time.

## Common mistakes

- Mistake: Forgetting to add a module's assembly to AbpRouterOptions.AdditionalAssemblies. Why wrong: Blazor's Router only scans the AppAssembly by default, so @page routes inside a Razor class library 404 silently. Fix: in the module's ConfigureServices, call Configure<AbpRouterOptions>(o => o.AdditionalAssemblies.Add(typeof(MyBlazorModule).Assembly)).
- Mistake: Building a custom EditForm + DataAnnotationsValidator stack and expecting ABP-style validation. Why wrong: ABP Blazor UI explicitly does NOT ship a built-in form validation infrastructure - it standardizes on Blazorise's <Validation> / <Feedback> components and ValidationRule.IsNotEmpty / custom ValidatorEventArgs handlers. Fix: use Blazorise validators or rely on server-side FluentValidation/DataAnnotations and surface errors via UserFriendlyException; do not duplicate validation infrastructure on the client.
- Mistake: Throwing a generic `Exception` (or `InvalidOperationException`) when you want the user to see the message. Why wrong: Only `UserFriendlyException` (and its derivatives) is rendered as a localized message dialog; everything else falls back to a generic 'An error occurred' message and the details go to the browser console. Fix: throw new UserFriendlyException(L["..."], code: "YourModule:errCode").
- Mistake: Reading ICurrentUser before the authentication state has resolved (e.g., synchronously in OnInitialized for a WebAssembly app right after navigation). Why wrong: ICurrentUser is hydrated from AuthenticationStateProvider; on first render in WASM it may report IsAuthenticated == false until the state task completes. Fix: use AuthorizeView/CascadingAuthenticationState, or override OnInitializedAsync and await an AuthenticationState before consuming CurrentUser.
- Mistake: Using @inject IStringLocalizer<MyResource> L plus an inherited L from AbpComponentBase. Why wrong: name collision - the @inject hides the base property, and partial-class code-behinds may reference a different L than the markup. Fix: pick one - either inherit AbpComponentBase and use its L, or omit the inheritance and inject explicitly; do not mix.
- Mistake: Replacing a component by editing the framework component's file or copying its source. Why wrong: it diverges on every ABP upgrade and breaks DI semantics. Fix: create a new component with @inherits TheOriginal + [ExposeServices(typeof(TheOriginal))] + [Dependency(ReplaceServices = true)].
- Mistake: Calling GlobalFeatureManager checks inside hot paths (per-render). Why wrong: it works but defeats the purpose of compile-time gating and adds noise. Fix: check once in OnInitializedAsync and store the result, or guard at the page level using a layout/parent component.
- Mistake: Setting PageLayout.MenuItemName to a name that is not registered in any IMenuContributor. Why wrong: themes that highlight the selected item (LeptonX) silently fail to highlight; no exception is thrown. Fix: keep the menu item `name` (e.g., 'MyProject.Crm.Customers') and PageLayout.MenuItemName in lockstep, ideally as a shared const.
- Mistake: Treating Basic Theme behavior as canonical when targeting LeptonX (or vice versa). Why wrong: PageLayout.MenuItemName highlighting and toolbar capabilities differ - Basic Theme uses top-level navigation and won't highlight nested items the way LeptonX does. Fix: pin theme expectations in test plans and verify against the actual production theme.
- Mistake: Forgetting to register the Blazor WASM PWA service worker in index.html, or expecting offline mode in development. Why wrong: service-worker.js is a no-op in dev; only service-worker.published.js does cache-first offline, and only after the user has visited online once. Fix: test PWA behavior against a Release publish, not `dotnet run`.

## Version pins (ABP 10.3)

- ABP 10.3 supports four Blazor hosting models: Blazor Server, Blazor WebAssembly, Blazor WebApp (.NET 8 unified), and MAUI Blazor. The WebApp template is the recommended default for new solutions on .NET 8/9. [uncertain] - exact default per 10.3 templates may evolve.
- Blazorise is bundled and pre-configured by all ABP themes; it became dual-licensed (open source + commercial) in June 2021 - check Blazorise's license terms when shipping commercial software.
- AbpRouterOptions exposes AdditionalAssemblies and AppAssembly. The Microsoft Blazor Router AdditionalAssemblies parameter still works but ABP standardizes on AbpRouterOptions for modular registration.
- GlobalFeatureManager.Instance is a static singleton in Volo.Abp.GlobalFeatures - tests that mutate global features must reset state to avoid cross-test bleed.
- Component override via [ExposeServices(typeof(TBase))] + [Dependency(ReplaceServices = true)] is the documented canonical pattern in 10.3. The `AbpComponentReplacementOptions.Replace<,>()` options-based equivalent is referenced in some module docs but the exact options-class name may shift across minor versions. [uncertain]
- UiNotificationOptions in 10.3 exposes OkButtonText and OkButtonIcon; additional global properties (toast position, duration) are not documented as configurable on the options object and must be set via Blazorise's NotificationProvider configuration. [uncertain]
- Blazor authentication: Blazor Server uses cookie auth; Blazor WebAssembly uses OIDC via AddAbpOpenIdConnect (or the standard Microsoft AddOpenIdConnect). The exact extension method name (`AddAbpOpenIdConnect` vs. `AddOpenIdConnect`) and which assembly hosts it can shift between hosting models. [uncertain]
- PWA scaffolding flag is `--pwa` on `abp new -t blazor`. The generated `service-worker.published.js` uses a cache-first strategy out of the box; updates require a forced reload after deployment.
- Form validation is intentionally NOT wrapped by ABP - this is by design in 10.3 and not expected to change; do not assume future versions will introduce ABP-specific validators.

## Cross-references

**Sibling UI references (in this skill):**
- [references/ui/overview.md](./overview.md) — load when the user is choosing among MVC/Blazor/Angular/RN/MAUI before committing to Blazor.
- [references/ui/themes.md](./themes.md) — LeptonX / LeptonX Lite / Basic theme details, CSS variables, and per-theme PageLayout support.
- [references/ui/mvc-razor-pages.md](./mvc-razor-pages.md) — load when the user mixes Razor Pages and Blazor in the same solution; menu / theme contracts are shared.
- [references/ui/mobile.md](./mobile.md) — load when the user pairs a Blazor host with a MAUI or React Native mobile companion (Blazor Hybrid via MAUI Blazor, or shared OpenIddict auth setup).

**Cross-skill references (in the abp-backend skill):**
- See **localization / IStringLocalizer** in the abp-backend skill (`framework/fundamentals.md`) — server-side resource configuration that Blazor reuses on the client.
- See **authorization** in the abp-backend skill (`framework/authorization.md`) — permission definition providers consumed by [Authorize], <AuthorizeView>, IAuthorizationService.
- See **exceptions / UserFriendlyException** in the abp-backend skill (`framework/fundamentals.md`) — UserFriendlyException, error codes, and remote-error mapping that Blazor surfaces.
- See **modularity / DependsOn** in the abp-backend skill (`framework/architecture-modularity.md`) — module wiring and `[ExposeServices]` / `[Dependency(ReplaceServices=true)]` semantics used for component overrides.
- See **identity and OpenIddict** in the abp-backend skill (`modules/identity-and-auth.md`) — OpenIddict / Account modules that back Blazor's OIDC sign-in.
- See **CLI / `abp new`** in the abp-backend skill (`tools/cli.md`) — `abp new -t blazor --pwa` and other Blazor scaffolding commands.

**External docs:**
- Blazorise - https://blazorise.com/docs - Component library ABP Blazor UI is built on (TextEdit, Validation, DataGrid, Modal, etc.).
- ASP.NET Core Blazor - https://learn.microsoft.com/aspnet/core/blazor - Hosting models, lifecycle, routing fundamentals that ABP layers on top of.
- ASP.NET Core Blazor Authentication and Authorization - https://learn.microsoft.com/aspnet/core/blazor/security - AuthenticationStateProvider, CascadingAuthenticationState, AddOpenIdConnect.
- Microsoft.Extensions.Localization - https://learn.microsoft.com/aspnet/core/fundamentals/localization - IStringLocalizer<T> contract used by AbpComponentBase.L.
- OpenIddict - https://documentation.openiddict.com - Identity provider used by ABP for the Blazor OIDC flow.
- Bootstrap 5 - https://getbootstrap.com/docs/5.0 - CSS framework underlying Blazorise components in ABP themes.
- Progressive Web Apps (Blazor WASM) - https://learn.microsoft.com/aspnet/core/blazor/progressive-web-app - Manifest, service worker, and offline guidance referenced by ABP's --pwa scaffolding.

## Sources (ABP 10.3)

- [https://abp.io/docs/10.3/framework/ui/blazor/overall](https://abp.io/docs/10.3/framework/ui/blazor/overall) — Overview of Blazor UI in ABP, supported hosting models (Blazor Server, WebAssembly, WebApp, MAUI Blazor), Blazorise integration, key services list (IUiMessageService, IUiNotificationService, IAlertManager, ISettingProvider).
- [https://abp.io/docs/10.3/framework/ui/blazor/navigation-menu](https://abp.io/docs/10.3/framework/ui/blazor/navigation-menu) — IMenuContributor / MenuConfigurationContext / ApplicationMenu / ApplicationMenuItem usage, AbpNavigationOptions registration, StandardMenus.Main / User constants, authorization-aware menu items.
- [https://abp.io/docs/10.3/framework/ui/blazor/forms-validation](https://abp.io/docs/10.3/framework/ui/blazor/forms-validation) — Documents that ABP Blazor UI relies on Blazorise's <Validation>, <TextEdit>, <Feedback>, <ValidationSuccess>, <ValidationError> components and ValidationRule.IsNotEmpty / custom ValidatorEventArgs validators (no built-in ABP form validation infrastructure).
- [https://abp.io/docs/10.3/framework/ui/blazor/localization](https://abp.io/docs/10.3/framework/ui/blazor/localization) — IStringLocalizer<T> usage in Razor (@inject + L["Key"]), AbpComponentBase.L property, server-side resources reused on Blazor client.
- [https://abp.io/docs/10.3/framework/ui/blazor/theming](https://abp.io/docs/10.3/framework/ui/blazor/theming) — Theming model: themes are Razor Class Libraries; IBrandingProvider, IMenuManager, IToolbarManager, ILanguageProvider, ICurrentUser, ICurrentTenant, IAlertManager; IBundleContributor, IToolbarContributor; Basic / Lepton / LeptonX themes.
- [https://abp.io/docs/10.3/framework/ui/blazor/page-layout](https://abp.io/docs/10.3/framework/ui/blazor/page-layout) — PageLayout service: Title, MenuItemName, BreadcrumbItems (BlazoriseUI.BreadcrumbItem), ToolbarItems (PageToolbars.PageToolbarItem). LeptonX supports all features; Basic Theme has limited menu item highlighting.
- [https://abp.io/docs/10.3/framework/ui/blazor/authentication](https://abp.io/docs/10.3/framework/ui/blazor/authentication) — OIDC-based authentication; Blazor Server uses cookies, WebAssembly uses AddAbpOpenIdConnect / AddOpenIdConnect; redirect to server for sign-in, OpenIddict integration.
- [https://abp.io/docs/10.3/framework/ui/blazor/authorization](https://abp.io/docs/10.3/framework/ui/blazor/authorization) — [Authorize] attribute on @page, <AuthorizeView Policy="...">, IAuthorizationService.IsGrantedAsync / CheckAsync, AbpComponentBase.AuthorizationService property; reuses server-defined permissions.
- [https://abp.io/docs/10.3/framework/ui/blazor/current-user](https://abp.io/docs/10.3/framework/ui/blazor/current-user) — ICurrentUser (Volo.Abp.Users) properties Id, UserName, Email, IsAuthenticated, Name, SurName, TenantId, Roles; injection in components or via AbpComponentBase.CurrentUser.
- [https://abp.io/docs/10.3/framework/ui/blazor/notification](https://abp.io/docs/10.3/framework/ui/blazor/notification) — IUiNotificationService methods Info/Success/Warn/Error; AbpComponentBase.Notify property; UiNotificationOptions for global defaults (OkButtonText, OkButtonIcon).
- [https://abp.io/docs/10.3/framework/ui/blazor/error-handling](https://abp.io/docs/10.3/framework/ui/blazor/error-handling) — Automatic exception handling pipeline; UserFriendlyException for end-user messages; integrates with localization; server-originated business/validation/authorization exceptions surfaced on the client.
- [https://abp.io/docs/10.3/framework/ui/blazor/customization-overriding-components](https://abp.io/docs/10.3/framework/ui/blazor/customization-overriding-components) — Component override pattern using @inherits + [ExposeServices(typeof(BaseComponent))] + [Dependency(ReplaceServices = true)]; example replaces theme Branding component.
- [https://abp.io/docs/10.3/framework/ui/blazor/global-features](https://abp.io/docs/10.3/framework/ui/blazor/global-features) — GlobalFeatureManager.Instance.IsEnabled(string) and IsEnabled<TFeature>(); namespace Volo.Abp.GlobalFeatures; conditional rendering based on enabled features.
- [https://abp.io/docs/10.3/framework/ui/blazor/routing](https://abp.io/docs/10.3/framework/ui/blazor/routing) — AbpRouterOptions.AdditionalAssemblies.Add(...) and AppAssembly to register module assemblies with the Blazor Router; configured in module's ConfigureServices.
- [https://abp.io/docs/10.3/framework/ui/blazor/pwa-configuration](https://abp.io/docs/10.3/framework/ui/blazor/pwa-configuration) — `abp new ... -t blazor --pwa`; manifest.json, service-worker.js (dev), service-worker.published.js (cache-first); .csproj ServiceWorkerAssetsManifest entry; index.html manifest link and SW registration.

Last verified: 2026-05-10
