# UI — MVC / Razor Pages

> Comprehensive guide to ABP's server-rendered UI stack on ASP.NET Core MVC and Razor Pages, covering tag helpers, navigation/menus, forms, modals, data tables, JS proxies, widgets, layout hooks, theming, customization, security headers, and integration testing.

## When to load this reference

- User is building or customizing a server-rendered UI in an ABP MVC or Razor Pages application
- User asks how to add a menu item, contribute to navigation, or implement IMenuContributor
- User asks about abp-modal, abp-dynamic-form, abp-input, or any abp-* tag helper
- User wants to call a server application service from JavaScript using the dynamic or static JS proxy
- User is configuring DataTables.Net with ABP's normalizeConfiguration and createAjax helpers
- User wants to build a widget with [Widget], AbpWidgetOptions, or refresh widgets via JavaScript
- User asks how to inject content via LayoutHooks.Head.Last or other layout hook points
- User is creating a custom theme or replacing the default Basic/LeptonX theme
- User wants to override a Razor page or view component shipped by an ABP module (Virtual File System override)
- User is configuring security headers (CSP, X-Frame-Options, HSTS, nonce) via UseAbpSecurityHeaders
- User asks how to write integration tests against Razor pages and controllers in an ABP solution

**Audience:** ABP developers building server-rendered UI on the MVC/Razor Pages stack — including those customizing pre-built module UIs, authoring themes, or wiring server APIs to client-side JavaScript through ABP's tag helpers and JS proxies.

## Key concepts

- IMenuContributor (Volo.Abp.UI.Navigation): Interface implemented by modules and apps to contribute items to StandardMenus.Main or StandardMenus.User; reach for it whenever you need to add navigation entries.
- ApplicationMenuItem (Volo.Abp.UI.Navigation): Represents a single menu node (name, displayName, url, icon, order, groupName); reach for it to construct items inside ConfigureMenuAsync.
- AbpNavigationOptions (Volo.Abp.UI.Navigation): Options class registering menu contributors via options.MenuContributors.Add(...).
- StandardMenus (Volo.Abp.UI.Navigation): Constants identifying the framework's two well-known menus: Main and User.
- IMenuManager (Volo.Abp.UI.Navigation): Runtime service used by themes/layouts to obtain the rendered menu tree.
- abp-dynamic-form / abp-input / abp-select / abp-radio (Volo.Abp.AspNetCore.Mvc.UI.Bootstrap.TagHelpers.Form): Tag helpers that produce Bootstrap-styled form controls and wire client/server validation, with automatic display-name localization based on property name conventions.
- abp-modal / abp-modal-header / abp-modal-body / abp-modal-footer (Volo.Abp.AspNetCore.Mvc.UI.Bootstrap.TagHelpers.Modal): Tag helpers that render a Bootstrap modal; reach for them on Razor Pages with Layout = null intended to be opened via abp.ModalManager.
- abp.ModalManager (JavaScript): Client-side helper that opens a server-rendered Razor Page in a modal, handles AJAX form submission, validation, unsaved-change detection, and exposes onOpen/onClose/onResult callbacks.
- abp.libs.datatables (JavaScript): Wrapper around DataTables.Net providing normalizeConfiguration, createAjax (bridges ABP request/response to DataTables format), defaultRenderers, and dataFormat shorthands (boolean/date/datetime).
- /Abp/ServiceProxyScript (Dynamic JS Proxy endpoint): Runtime-generated JS that exposes every application service as namespace.service.method(...) returning a jQuery Deferred backed by abp.ajax.
- abp generate-proxy -t js (CLI / Static JS Proxy): Generates pre-built ClientProxies/*.js files at development time; combine with DynamicJavaScriptProxyOptions.DisableModule(...) to enforce static-only consumption.
- [Widget] attribute & AbpWidgetOptions (Volo.Abp.AspNetCore.Mvc.UI.Widgets): Declares a ViewComponent as a refreshable widget with style/script files, RefreshUrl, RequiresAuthentication, RequiredPolicies, and AutoInitialize.
- AbpViewComponent (Volo.Abp.AspNetCore.Mvc): Recommended base class for ABP view components and widgets.
- LayoutHooks & AbpLayoutHookOptions (Volo.Abp.AspNetCore.Mvc.UI.LayoutHooks): Hook points (Head.First/Last, Body.First/Last, PageContent.First/Last) where view components are injected; reach for it to inject analytics, banners, or chat widgets without forking themes.
- StandardLayouts (Volo.Abp.AspNetCore.Mvc.UI.Theming): Constants identifying the three well-known layouts: Application, Account, Empty.
- ITheme + [ThemeName] (Volo.Abp.AspNetCore.Mvc.UI.Theming): Interface and attribute used to register a theme; GetLayout(name, fallback) maps StandardLayouts to .cshtml paths.
- AbpThemingOptions (Volo.Abp.AspNetCore.Mvc.UI.Theming): Options for registering themes and setting DefaultThemeName.
- IThemeSelector (Volo.Abp.AspNetCore.Mvc.UI.Theming): Service that picks the active theme per request; replace for dynamic per-tenant or per-user theme selection.
- IBrandingProvider, IToolbarManager, IAlertManager, IPageLayout (Volo.Abp.AspNetCore.Mvc.UI.*): Theme-side services for application name/logo, top-bar items, page-level alerts, and breadcrumbs/title.
- Virtual File System override (Volo.Abp.VirtualFileSystem): Mechanism that lets you replace any module-shipped .cshtml or static asset by placing a file with the same path in your project.
- [Dependency(ReplaceServices = true)] + [ExposeServices(typeof(OriginalPageModel))] (Volo.Abp.DependencyInjection): Attribute pair used to substitute a page model class shipped by a module without touching its view.
- UseAbpSecurityHeaders + AbpSecurityHeadersOptions + [IgnoreAbpSecurityHeaderAttribute] (Volo.Abp.AspNetCore.Security): Middleware and options that inject X-Content-Type-Options, X-XSS-Protection, X-Frame-Options, Content-Security-Policy (with optional script nonce), and arbitrary headers; the attribute opts a controller/page out.
- AbpWebApplicationFactoryIntegratedTest (Volo.Abp.AspNetCore.TestBase): Base test class extending WebApplicationFactory; helpers GetResponseAsStringAsync and GetResponseAsObjectAsync<T> drive HTTP integration tests against pages and controllers.

## Configuration pattern

Wiring is split between AbpModule.ConfigureServices (options) and OnApplicationInitialization (middleware). In ConfigureServices, configure the relevant options classes — AbpNavigationOptions, AbpToolbarOptions, AbpThemingOptions, AbpLayoutHookOptions, AbpWidgetOptions, AbpBundlingOptions, DynamicJavaScriptProxyOptions, AbpSecurityHeadersOptions. Example: 

```csharp
public override void ConfigureServices(ServiceConfigurationContext context)
{
    Configure<AbpNavigationOptions>(options =>
    {
        options.MenuContributors.Add(new MyProjectMenuContributor());
    });

    Configure<AbpThemingOptions>(options =>
    {
        options.Themes.Add<BasicTheme>();
        options.DefaultThemeName = BasicTheme.Name;
    });

    Configure<AbpLayoutHookOptions>(options =>
    {
        options.Add(
            LayoutHooks.Head.Last,
            typeof(GoogleAnalyticsViewComponent),
            layout: StandardLayouts.Application);
    });

    Configure<AbpWidgetOptions>(options =>
    {
        options.Widgets
            .Add<MySalesWidgetViewComponent>()
            .WithStyles("/Widgets/MySales/Default.css")
            .WithScripts("/Widgets/MySales/Default.js");
    });

    Configure<AbpSecurityHeadersOptions>(options =>
    {
        options.UseContentSecurityPolicyHeader = true;
        options.ContentSecurityPolicyValue =
            "object-src 'none'; form-action 'self'; frame-ancestors 'none'";
        options.UseContentSecurityPolicyScriptNonce = true;
    });

    Configure<DynamicJavaScriptProxyOptions>(options =>
    {
        options.DisableModule("app"); // when shipping static proxies only
    });
}

public override void OnApplicationInitialization(ApplicationInitializationContext context)
{
    var app = context.GetApplicationBuilder();
    app.UseRouting();
    app.UseAbpSecurityHeaders();      // after UseRouting, before endpoints
    app.UseAuthentication();
    app.UseAuthorization();
    app.UseConfiguredEndpoints();
}
```

Routing/menu/widget contributions are typically registered as classes implementing IMenuContributor, IToolbarContributor, or BundleContributor, then added through their respective options. PreConfigureServices is rarely needed for UI; it is used only when you must configure something before another module's ConfigureServices runs (e.g., adding a tag helper assembly).

## Code examples

### Contribute a menu item with permission gating

_Add a 'Books' entry under StandardMenus.Main only when the current user has the BookStore.Books permission._

```csharp
using System.Threading.Tasks;
using Volo.Abp.UI.Navigation;

public class BookStoreMenuContributor : IMenuContributor
{
    public async Task ConfigureMenuAsync(MenuConfigurationContext context)
    {
        if (context.Menu.Name != StandardMenus.Main) return;

        if (await context.IsGrantedAsync("BookStore.Books"))
        {
            context.Menu.AddItem(
                new ApplicationMenuItem(
                    name: "BookStore.Books",
                    displayName: "Books",
                    url: "/Books",
                    icon: "fa fa-book",
                    order: 2000));
        }
    }
}

// In your module:
Configure<AbpNavigationOptions>(options =>
{
    options.MenuContributors.Add(new BookStoreMenuContributor());
});
```

**Key lines:** context.Menu.Name == StandardMenus.Main scopes the contribution; IsGrantedAsync checks the permission; ApplicationMenuItem.order controls sort position (default 1000).

### Open a Razor Page in a modal with abp.ModalManager

_Edit form rendered in /Books/EditModal.cshtml is opened from the index page, validated, posted as AJAX, and refreshes the list on success._

```csharp
// Books/EditModal.cshtml.cs (page model)
public class EditModalModel : BookStorePageModel
{
    [BindProperty] public BookDto Book { get; set; }
    private readonly IBookAppService _service;
    public EditModalModel(IBookAppService service) => _service = service;
    public async Task OnGetAsync(Guid id) => Book = await _service.GetAsync(id);
    public async Task<NoContentResult> OnPostAsync()
    {
        await _service.UpdateAsync(Book.Id, ObjectMapper.Map<BookDto, UpdateBookDto>(Book));
        return NoContent();
    }
}

@* Books/EditModal.cshtml *@
@page
@model Acme.BookStore.Web.Pages.Books.EditModalModel
@{ Layout = null; }
<form asp-page="/Books/EditModal" data-ajaxForm="true">
    <abp-modal>
        <abp-modal-header title="Edit Book" />
        <abp-modal-body>
            <abp-input asp-for="Book.Id" />
            <abp-input asp-for="Book.Name" />
        </abp-modal-body>
        <abp-modal-footer buttons="@(AbpModalButtons.Save|AbpModalButtons.Cancel)" />
    </abp-modal>
</form>

// books.js (caller)
var editModal = new abp.ModalManager('/Books/EditModal');
editModal.onResult(function () { dataTable.ajax.reload(); });
$('#EditButton').click(function () { editModal.open({ id: '...' }); });
```

**Key lines:** Layout = null is required for modal pages; the surrounding <form> + ModalManager triggers AJAX submission instead of a full POST; OnPostAsync returns NoContent() to signal success and trigger onResult.

### Wire a DataTable to a dynamic JS proxy

_Server-side paged table over IBookAppService.GetListAsync, using ABP's createAjax bridge and default formatters._

```csharp
$(function () {
    var service = acme.bookStore.book;
    var dataTable = $('#BooksTable').DataTable(abp.libs.datatables.normalizeConfiguration({
        serverSide: true,
        paging: true,
        order: [[1, 'asc']],
        searching: false,
        ajax: abp.libs.datatables.createAjax(service.getList),
        columnDefs: [
            { rowAction: { items: [
                { text: 'Edit', visible: abp.auth.isGranted('BookStore.Books.Edit'),
                  action: function (data) { editModal.open({ id: data.record.id }); } },
                { text: 'Delete', confirmMessage: function () { return 'Are you sure?'; },
                  action: function (data) { service.delete(data.record.id).then(function () { dataTable.ajax.reload(); }); } }
            ] } },
            { title: 'Name', data: 'name' },
            { title: 'Release Date', data: 'releaseDate', dataFormat: 'date' }
        ]
    }));
});
```

**Key lines:** normalizeConfiguration applies ABP defaults; createAjax(service.getList) translates DataTables draw/start/length params into ABP's PagedAndSortedResultRequestDto and back; dataFormat: 'date' uses the registered defaultRenderer.

### Register a widget with refresh URL and required policy

_Sales widget visible only to users with the Dashboard.Sales policy; pulls fresh markup from /Widgets/Sales without a full page reload._

```csharp
using Volo.Abp.AspNetCore.Mvc;
using Volo.Abp.AspNetCore.Mvc.UI.Widgets;

[Widget(
    RefreshUrl = "/Widgets/Sales",
    ScriptFiles = new[] { "/Widgets/Sales/Default.js" },
    StyleFiles  = new[] { "/Widgets/Sales/Default.css" },
    AutoInitialize = true,
    RequiresAuthentication = true,
    RequiredPolicies = new[] { "Dashboard.Sales" })]
public class SalesWidgetViewComponent : AbpViewComponent
{
    public IViewComponentResult Invoke() => View(new SalesViewModel { /* ... */ });
}

// In your module ConfigureServices:
Configure<AbpWidgetOptions>(options =>
{
    options.Widgets.Add<SalesWidgetViewComponent>();
});

// In a Razor page:
@await Component.InvokeAsync("SalesWidget")
```

**Key lines:** [Widget] turns an AbpViewComponent into a first-class widget; RefreshUrl enables abp.widgets.refresh on the client; RequiredPolicies blocks the render entirely for unauthorized users.

### Inject a script via Layout Hook and harden CSP

_Add Google Analytics to every Application-layout page and require a nonce on any inline script._

```csharp
// View component
public class GoogleAnalyticsViewComponent : AbpViewComponent
{
    public IViewComponentResult Invoke() => View();
}

@* Default.cshtml *@
@inject IHtmlHelper Html
<script nonce="@Html.GetScriptNonce()">
  // GA snippet
</script>

// Module ConfigureServices
Configure<AbpLayoutHookOptions>(options =>
{
    options.Add(
        LayoutHooks.Head.Last,
        typeof(GoogleAnalyticsViewComponent),
        layout: StandardLayouts.Application);
});

Configure<AbpSecurityHeadersOptions>(options =>
{
    options.UseContentSecurityPolicyHeader = true;
    options.UseContentSecurityPolicyScriptNonce = true;
    options.ContentSecurityPolicyValue =
        "default-src 'self'; script-src 'self' https://www.googletagmanager.com 'nonce-{NONCE}'; object-src 'none'; frame-ancestors 'none'";
});

// Module OnApplicationInitialization
app.UseRouting();
app.UseAbpSecurityHeaders();
```

**Key lines:** LayoutHooks.Head.Last + StandardLayouts.Application limits injection to the main app layout; Html.GetScriptNonce() pairs with UseContentSecurityPolicyScriptNonce so inline scripts pass strict CSP.

### Override a module-shipped Razor Page

_Replace the Account/Login page from the Account module with a custom design without forking the module._

```csharp
// 1. Place a file at the same virtual path:
//    Pages/Account/Login.cshtml in your .Web project
// 2. (Optional) Replace the page model:
[ExposeServices(typeof(LoginModel))]
[Dependency(ReplaceServices = true)]
public class CustomLoginModel : LoginModel
{
    public CustomLoginModel(/* deps */) : base(/* deps */) { }

    public override async Task<IActionResult> OnPostAsync(string action)
    {
        // pre-logic
        var result = await base.OnPostAsync(action);
        // post-logic
        return result;
    }
}
```

**Key lines:** The Virtual File System lets your project's path win over the module's embedded resource; [Dependency(ReplaceServices = true)] + [ExposeServices] swaps the page model class while preserving the original cshtml binding.

## Common mistakes

### Forgetting Layout = null on a modal Razor Page

**Why wrong:** Without Layout = null the page renders the entire site chrome inside the modal, producing a nested layout, broken styles, and double scrollbars.

**Correct pattern:** Add @{ Layout = null; } at the top of every .cshtml that is opened via abp.ModalManager.

```csharp
@page
@model EditModalModel
@{ Layout = null; }
```

### Returning RedirectToPage or a full ViewResult from a modal OnPostAsync

**Why wrong:** ModalManager expects a 2xx with no body for a successful save; redirects cause it to navigate the parent page instead of closing the modal.

**Correct pattern:** Return NoContent() (or a small JSON DTO) on success; ModalManager will fire onResult and the caller closes the modal.

```csharp
public async Task<NoContentResult> OnPostAsync() { await _service.UpdateAsync(...); return NoContent(); }
```

### Wrapping abp-modal in a div instead of a form when you need Save submission

**Why wrong:** ModalManager's automatic AJAX submission and validation only kick in when there is a <form> ancestor; the Save button becomes a no-op otherwise.

**Correct pattern:** Wrap the abp-modal element in a <form> with the asp-page (or asp-action) attribute.

```xml
<form asp-page="/Books/EditModal">
  <abp-modal>...</abp-modal>
</form>
```

### Adding menu items unconditionally without a permission check

**Why wrong:** Items appear for users who cannot reach the underlying page, leading to 403 / Access Denied at click time.

**Correct pattern:** Guard with await context.IsGrantedAsync(...) or use the .RequirePermissions(...) shortcut on ApplicationMenuItem.

```csharp
if (await context.IsGrantedAsync("BookStore.Books")) { context.Menu.AddItem(...); }
```

### Calling app.UseAbpSecurityHeaders() before app.UseRouting()

**Why wrong:** Pipeline ordering matters; placing it before routing can cause headers to be missing on endpoints and breaks CSP nonce generation tied to the request.

**Correct pattern:** Call UseAbpSecurityHeaders() after UseRouting() and before UseAuthentication()/UseAuthorization() in OnApplicationInitialization.

```csharp
app.UseRouting();
app.UseAbpSecurityHeaders();
app.UseAuthentication();
app.UseAuthorization();
```

### Hard-coding inline scripts after enabling UseContentSecurityPolicyScriptNonce

**Why wrong:** Browsers block any <script> without the matching nonce, breaking analytics, modals, or page-level scripts that worked previously.

**Correct pattern:** Add nonce="@Html.GetScriptNonce()" to every inline <script>, or move the script to an external file under /libs.

```xml
<script nonce="@Html.GetScriptNonce()">/* inline */</script>
```

### Generating static JS proxies but leaving dynamic proxies enabled for the same module

**Why wrong:** The page ends up with two definitions for the same namespace; the last one wins silently and surprises developers with stale signatures.

**Correct pattern:** Configure DynamicJavaScriptProxyOptions to DisableModule("<name>") for any module shipped via static proxies, and re-run abp generate-proxy after API changes.

```csharp
Configure<DynamicJavaScriptProxyOptions>(o => o.DisableModule("app"));
```

### Using ApplicationConfigurationScript / ServiceProxyScript output-cached at the CDN

**Why wrong:** Both endpoints are user/tenant-aware (granted permissions, current culture, settings); CDN caching leaks one user's view of the world to another.

**Correct pattern:** Mark these endpoints as no-cache or vary by user/tenant; never put them behind a public CDN cache.

```csharp
// reverse-proxy: proxy_no_cache 1; proxy_cache_bypass 1;
```

### Overriding a module's .cshtml without copying _ViewImports.cshtml tag-helper directives

**Why wrong:** The override file loses access to abp-* tag helpers and renders them as literal HTML attributes on the page.

**Correct pattern:** Either place _ViewImports.cshtml alongside the override or rely on the project-level _ViewImports that already references the abp tag-helper assemblies.

```csharp
@addTagHelper *, Volo.Abp.AspNetCore.Mvc.UI.Bootstrap
```

### Calling new BasicTheme() or instantiating themes manually

**Why wrong:** Themes must be resolved via DI (ITransientDependency) and ThemeManager/IThemeSelector — manual instantiation bypasses options and breaks layout resolution.

**Correct pattern:** Register via Configure<AbpThemingOptions>(o => o.Themes.Add<MyTheme>()) and resolve through IThemeManager/IThemeSelector.

```csharp
Configure<AbpThemingOptions>(o => { o.Themes.Add<BasicTheme>(); o.DefaultThemeName = BasicTheme.Name; });
```

## Version pins (ABP 10.3)

- ABP 10.3 ships three official MVC themes: Basic (free OSS), LeptonX Lite (free), and LeptonX (commercial under ABP license). The legacy Lepton theme has been superseded by LeptonX in modern templates.
- 10.3 keeps the foundational client-side stack on Bootstrap 5, jQuery, jQuery Validation, DataTables.Net, FontAwesome, SweetAlert2, Toastr, Lodash, Luxon, and Select2 — bundled through @abp/aspnetcore.mvc.ui.theme.shared.
- abp generate-proxy CLI must target a running server URL (-u) and writes to a ClientProxies/ folder; regeneration is required after any application-service signature change. [uncertain] whether 10.3 supports incremental generation per service.
- AbpSecurityHeadersOptions.UseContentSecurityPolicyScriptNonce is opt-in (default false). Turning it on without auditing every inline script is a known regression source.
- Layout-hook names are static fields on LayoutHooks (Head.First/Last, Body.First/Last, PageContent.First/Last). [uncertain] if any additional hook points (e.g., for sidebar) are exposed in 10.3 beyond these six.
- StandardMenus exposes only Main and User in 10.3; the LeptonX theme surfaces additional rendering of menu groups but the menu graph itself is still those two roots.
- Group-name rendering for menu items is theme-dependent — only LeptonX honors ApplicationMenuItem.GroupName; in Basic theme groupName is effectively ignored.
- Dynamic JS proxies are served from /Abp/ServiceProxyScript and are user/tenant-aware. The endpoint path has been stable across recent ABP versions but may move if Volo restructures the framework module routes.
- Widget [Widget] attribute properties — RefreshUrl, RequiresAuthentication, RequiredPolicies, AutoInitialize, StyleFiles, ScriptFiles — are confirmed in 10.3. [uncertain] whether RequiredFeatures was added/renamed in this version.
- Test base helpers GetResponseAsStringAsync and GetResponseAsObjectAsync<T> are stable; the suggested HTML-parsing dependency (HtmlAgilityPack) and assertion library (Shouldly) are conventions, not pinned by the framework.

## Cross-references

**Sibling UI references (in this skill):**
- [references/ui/blazor.md](./blazor.md) — cross-link when comparing server-rendered MVC/Razor Pages with the Blazor UI option for an ABP solution
- [references/ui/angular.md](./angular.md) — cross-link when comparing the SPA Angular UI with the MVC/Razor Pages UI
- [references/ui/themes.md](./themes.md) — Basic / LeptonX Lite / LeptonX theme details, AbpThemingOptions, layout overrides under Themes/<Theme>/
- [references/ui/overview.md](./overview.md) — cross-link when picking among UI stacks before drilling into MVC

**Cross-skill references (in the abp-backend skill):**
- See **authorization** in the abp-backend skill (`framework/authorization.md`) — triggered when discussing IsGrantedAsync, RequiredPolicies on widgets, or permission-gated menu items
- See **localization / IStringLocalizer** in the abp-backend skill (`framework/fundamentals.md`) — triggered when explaining automatic display-name and validation-message localization in forms
- See **virtual file system** in the abp-backend skill (`framework/architecture-modularity.md`) — triggered when overriding module-shipped pages, views, or static assets
- See **modularity / DependsOn** in the abp-backend skill (`framework/architecture-modularity.md`) — triggered when contributing menus/toolbars/themes from a module
- See **DI / [Dependency] / [ExposeServices]** in the abp-backend skill (`framework/fundamentals.md`) — triggered when using `[Dependency(ReplaceServices)]` / `[ExposeServices]` to replace a page model
- See **bundling** in the abp-backend skill (`framework/fundamentals.md`) — triggered when configuring AbpBundlingOptions for theme or page-specific styles/scripts
- See **application services** in the abp-backend skill (`framework/architecture-ddd.md`) — triggered when discussing dynamic/static JS proxies that mirror application service signatures
- See **integration testing** in the abp-backend skill (`testing.md`) — triggered when writing AbpWebApplicationFactoryIntegratedTest-based tests for Razor pages or controllers

**External docs:**
- [ASP.NET Core Razor Pages](https://learn.microsoft.com/en-us/aspnet/core/razor-pages/) — ABP recommends Razor Pages for UI; the underlying page lifecycle, [BindProperty], and Tag Helpers come from ASP.NET Core.
- [ASP.NET Core MVC](https://learn.microsoft.com/en-us/aspnet/core/mvc/overview) — ABP recommends MVC for HTTP API endpoints; controllers and ViewComponents (the basis for widgets) are ASP.NET Core primitives.
- [ASP.NET Core View Components](https://learn.microsoft.com/en-us/aspnet/core/mvc/views/view-components) — Widgets, layout-hook injectables, and AbpViewComponent all derive from ASP.NET Core ViewComponent.
- [ASP.NET Core Integration Tests](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests) — Background for AbpWebApplicationFactoryIntegratedTest, which extends WebApplicationFactory<TStartup>.
- [DataTables.Net](https://datatables.net/) — Underlying grid library wrapped by abp.libs.datatables.normalizeConfiguration / createAjax.
- [Bootstrap](https://getbootstrap.com/) — Tag helpers (abp-modal, abp-input, etc.) emit Bootstrap-compatible markup; themes are Bootstrap-based.
- [jQuery](https://jquery.com/) — abp.ajax, ModalManager, and JS proxies return jQuery Deferred objects and use $.extend internally.
- [jQuery Validation](https://jqueryvalidation.org/) — Powers client-side validation in abp-dynamic-form and abp-input tag helpers.
- [Content Security Policy (MDN)](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) — Reference for ContentSecurityPolicyValue and script-nonce semantics enforced by AbpSecurityHeadersOptions.
- [HtmlAgilityPack](https://html-agility-pack.net/) — Recommended HTML parser used in Razor Pages integration tests with GetResponseAsStringAsync.
- [Shouldly](https://docs.shouldly.org/) — Assertion library typically paired with xUnit in the .Web.Tests project.

## Sources (ABP 10.3)

- [https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/overall](https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/overall) — Top-level overview of ABP's MVC/Razor Pages UI: themes, foundational JS libraries, tag helpers, dynamic JS proxies, bundling, modularity
- [https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/navigation-menu](https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/navigation-menu) — IMenuContributor, ApplicationMenuItem, AbpNavigationOptions, StandardMenus (Main/User), permission-based menu visibility
- [https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/forms-validation](https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/forms-validation) — abp-dynamic-form, abp-input/abp-select/abp-radio tag helpers, automatic localization of display names and validation messages
- [https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/modals](https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/modals) — abp-modal/abp-modal-header/body/footer tag helpers, abp.ModalManager JS API, automatic form validation and AJAX submission
- [https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/data-tables](https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/data-tables) — DataTables.Net integration, abp.libs.datatables.normalizeConfiguration, createAjax adapter, default renderers and dataFormat options
- [https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/dynamic-javascript-proxies](https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/dynamic-javascript-proxies) — /Abp/ServiceProxyScript runtime-generated JS clients, namespace.service.method() pattern, abp.ajax integration, jQuery Deferred promises
- [https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/static-javascript-proxies](https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/static-javascript-proxies) — abp generate-proxy -t js CLI command, ClientProxies folder, DynamicJavaScriptProxyOptions.DisableModule
- [https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/widgets](https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/widgets) — [Widget] attribute, AbpViewComponent base, AbpWidgetOptions, RefreshUrl, RequiredPolicies, AutoInitialize, WidgetManager
- [https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/layout-hooks](https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/layout-hooks) — LayoutHooks (Head.First/Last, Body.First/Last, PageContent.First/Last), AbpLayoutHookOptions, StandardLayouts targeting
- [https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/theming](https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/theming) — ITheme interface, [ThemeName] attribute, AbpThemingOptions, StandardLayouts (Application/Account/Empty), IThemeSelector, IBrandingProvider
- [https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/customization-user-interface](https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/customization-user-interface) — Page model override via [Dependency(ReplaceServices)], Razor view override via Virtual File System, view component replacement
- [https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/security-headers](https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/security-headers) — UseAbpSecurityHeaders middleware, AbpSecurityHeadersOptions, CSP nonce, X-Frame-Options, [IgnoreAbpSecurityHeaderAttribute]
- [https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/testing](https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/testing) — AbpWebApplicationFactoryIntegratedTest, GetResponseAsStringAsync/GetResponseAsObjectAsync, HtmlAgilityPack-based assertions

Last verified: 2026-05-10
