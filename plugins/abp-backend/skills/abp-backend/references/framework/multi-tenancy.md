# Multi-Tenancy

> How ABP makes a single application serve multiple tenants with automatic data isolation, configurable tenant resolution, and per-tenant database support — driven by IMultiTenant, ICurrentTenant, ITenantStore, and the IMultiTenant data filter.

## When to load this reference

Load this reference when the user asks any of: 'how do I make my app multi-tenant in ABP', 'how do I add TenantId to an entity', 'why is my repository returning only some rows / why are tenant rows missing', 'how do I query across all tenants / disable the tenant filter', 'how do I switch tenant context in a background job or seeder', 'how do I resolve tenant from subdomain / header / cookie / route', 'how do I configure database-per-tenant connection strings', 'what is ICurrentTenant.Change and how do I use it', 'what is the difference between Host and Tenant side', 'how do I restrict an application service to host-only or tenant-only', 'what is the FallbackTenant', 'why is my __tenant header dropped (NGINX)', or anything mentioning IMultiTenant, ITenantResolver, ITenantStore, MultiTenancySides, AbpMultiTenancyOptions, AbpTenantResolveOptions.

**Audience:** ABP developers building any SaaS-style application, plus any developer touching entities, repositories, application services, background jobs, data seeders, or hosting/reverse-proxy configuration in a solution where AbpMultiTenancyOptions.IsEnabled = true.

## Key concepts

- IMultiTenant (Volo.Abp.MultiTenancy): marker interface declaring a nullable Guid? TenantId on an entity. Reach for it on any aggregate root or entity that should be isolated per tenant; null TenantId means the row belongs to the host.
- ICurrentTenant (Volo.Abp.MultiTenancy): ambient service exposing Id (Guid?), Name (string), IsAvailable (bool) and IDisposable Change(Guid? id, string name = null). Pre-injected on ApplicationService, DomainService, AbpController, and AbpPageModel base classes. Reach for it whenever you need to read the active tenant or temporarily switch context (background jobs, seeders, cross-tenant reports).
- ITenantStore (Volo.Abp.MultiTenancy): abstraction that loads TenantConfiguration (Id, Name, NormalizedName, IsActive, ConnectionStrings) by id or normalized name. Reach for it from a custom tenant resolver or when implementing your own tenant source.
- DefaultTenantStore (Volo.Abp.MultiTenancy): the built-in ITenantStore that reads from AbpDefaultTenantStoreOptions (and therefore the 'Tenants' section in IConfiguration). Used when the Tenant Management module is not installed.
- TenantConfiguration / BasicTenantInfo (Volo.Abp.MultiTenancy): DTO-shaped types describing a tenant for resolution; includes ConnectionStrings dictionary keyed by connection-string name.
- ITenantResolver / TenantResolveContributorBase (Volo.Abp.MultiTenancy): pipeline that produces a tenant id-or-name from the current request/scope. Subclass TenantResolveContributorBase and override Name + ResolveAsync(ITenantResolveContext) to add a custom source.
- Built-in resolve contributors (in default order): CurrentUserTenantResolveContributor (claims; first for security), QueryStringTenantResolveContributor (?__tenant=...), RouteTenantResolveContributor ({__tenant} route value), HeaderTenantResolveContributor (__tenant header), CookieTenantResolveContributor (__tenant cookie). DomainTenantResolveContributor is added via options.AddDomainTenantResolver("{0}.example.com").
- AbpMultiTenancyOptions (Volo.Abp.MultiTenancy): top-level on/off switch via IsEnabled. Off by default in the bare framework; turned on in the startup templates through a MultiTenancyConsts class.
- AbpDefaultTenantStoreOptions (Volo.Abp.MultiTenancy): in-memory list of TenantConfiguration consumed by DefaultTenantStore; bound from the 'Tenants' configuration section.
- AbpTenantResolveOptions (Volo.Abp.AspNetCore.MultiTenancy): collection of resolver contributors plus AddDomainTenantResolver helper and FallbackTenant (string id-or-name) to force a specific tenant when none is resolved.
- AbpAspNetCoreMultiTenancyOptions (Volo.Abp.AspNetCore.MultiTenancy): TenantKey (default '__tenant') used by query/route/header/cookie contributors, plus MultiTenancyMiddlewareErrorPageBuilder for inactive/missing tenant errors.
- IDataFilter / IDataFilter<IMultiTenant> (Volo.Abp.Data): the runtime filter ABP installs over IMultiTenant entities. Reach for IDataFilter.Disable<IMultiTenant>() inside a using block when you need a cross-tenant query (host-side reporting, admin tools).
- MultiTenancySides enum (Volo.Abp.MultiTenancy): values Host, Tenant, Both. Used on [MultiTenancySide(...)] attribute, on IMultiTenancyServiceProviderAccessor checks, and on app-service / page-level guards to restrict who can call a feature.
- UseMultiTenancy() middleware (Volo.Abp.AspNetCore.MultiTenancy): the ASP.NET Core middleware that runs the resolver pipeline and sets ICurrentTenant for the request. Must be registered after UseAuthentication() so claims are available; pre-wired in startup templates.
- Tenant Management module (cross-reference): provides the database-backed ITenantStore implementation (Tenant aggregate, ITenantRepository, AbpTenants / AbpTenantConnectionStrings tables, TenantAppService and admin UI). Pulled in by the layered/microservice templates.

## Configuration pattern

Wiring lives across PreConfigureServices and ConfigureServices on your Domain.Shared / AspNetCore module. Typical shape:

```csharp
[DependsOn(
    typeof(AbpMultiTenancyModule),
    typeof(AbpAspNetCoreMultiTenancyModule),
    typeof(AbpTenantManagementDomainModule) // if using the module
)]
public class MyAppModule : AbpModule
{
    public override void PreConfigureServices(ServiceConfigurationContext context)
    {
        // Subdomain wildcard support for OpenIddict (only if using subdomain resolution
        // with OpenIddict-issued tokens):
        PreConfigure<AbpOpenIddictWildcardDomainOptions>(options =>
        {
            options.EnableWildcardDomainSupport = true;
            options.WildcardDomainsFormat.Add("https://{0}.mydomain.com");
        });
    }

    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        // 1. Master switch.
        Configure<AbpMultiTenancyOptions>(options =>
        {
            options.IsEnabled = MultiTenancyConsts.IsEnabled; // typically true
        });

        // 2. (Optional) Static tenant list when not using the Tenant Management module.
        //    Pulled from appsettings.json 'Tenants' section by default.
        Configure<AbpDefaultTenantStoreOptions>(options =>
        {
            options.Tenants = new[]
            {
                new TenantConfiguration(Guid.Parse("446a5211-..."), "acme")
                {
                    ConnectionStrings = new ConnectionStrings
                    {
                        Default = "Server=...;Database=acme_db;..."
                    }
                }
            };
        });

        // 3. Resolver pipeline: rename the tenant key, add a domain resolver,
        //    set a fallback tenant, plug in a custom contributor.
        Configure<AbpAspNetCoreMultiTenancyOptions>(options =>
        {
            options.TenantKey = "__tenant"; // default; change if NGINX/CDN strips it
        });

        Configure<AbpTenantResolveOptions>(options =>
        {
            options.AddDomainTenantResolver("{0}.mydomain.com");
            options.FallbackTenant = null; // or a tenant name/id to force
            options.TenantResolvers.Add(new MyCustomTenantResolveContributor());
        });

        // 4. (Optional) Customize the inactive/unknown-tenant error page.
        Configure<AbpAspNetCoreMultiTenancyOptions>(options =>
        {
            options.MultiTenancyMiddlewareErrorPageBuilder = async (ctx, ex) =>
            {
                ctx.Response.StatusCode = 400;
                await ctx.Response.WriteAsync("Invalid tenant.");
                return true; // true => stop the pipeline
            };
        });
    }

    public override void OnApplicationInitialization(ApplicationInitializationContext context)
    {
        var app = context.GetApplicationBuilder();
        app.UseAbpRequestLocalization();
        app.UseAuthentication();
        app.UseMultiTenancy();   // MUST come after UseAuthentication so the
                                 // CurrentUser-claims contributor sees the principal.
        app.UseAuthorization();
        app.UseConfiguredEndpoints();
    }
}
```

Key options recap:
- AbpMultiTenancyOptions.IsEnabled — global on/off (default false in bare framework, true in templates).
- AbpDefaultTenantStoreOptions.Tenants — in-memory tenant list when no Tenant Management module is used; can also bind from the 'Tenants' section of appsettings.json.
- AbpAspNetCoreMultiTenancyOptions.TenantKey — name of the parameter consumed by query/route/header/cookie contributors (default '__tenant').
- AbpAspNetCoreMultiTenancyOptions.MultiTenancyMiddlewareErrorPageBuilder — last-chance hook for invalid/inactive tenants.
- AbpTenantResolveOptions.TenantResolvers — ordered pipeline; first contributor that yields wins.
- AbpTenantResolveOptions.FallbackTenant — if set, prevents falling back to host context.
- AbpTenantResolveOptions.AddDomainTenantResolver("{0}.host") — registers the domain/subdomain contributor with a format placeholder.

## Code examples

Example 1 — Title: 'Make an entity multi-tenant'. Scenario: shipping a new aggregate root that should be isolated per tenant.
```csharp
using Volo.Abp.Domain.Entities;
using Volo.Abp.MultiTenancy;
public class Order : AggregateRoot<Guid>, IMultiTenant
{
public Guid? TenantId { get; set; } // required by IMultiTenant; null => host
public string OrderNumber { get; set; } = default!;
public decimal TotalAmount { get; set; }
protected Order() { }
public Order(Guid id, string orderNumber, decimal totalAmount) : base(id)
{
OrderNumber = orderNumber;
TotalAmount = totalAmount;
// TenantId is auto-populated by ABP from ICurrentTenant.Id when
// the entity is inserted via a tenant-scoped repository.
}
}
```
Key lines: `IMultiTenant` opts the entity into the framework's data filter; the nullable Guid? `TenantId` is the contract, and ABP fills it automatically on insert based on `ICurrentTenant.Id`.
Example 2 — Title: 'Switch tenant context in a background job or seeder'. Scenario: code that runs outside an HTTP request (no resolver pipeline) but must operate on a specific tenant.
```csharp
public class TenantOrderProcessor : ITransientDependency
{
private readonly ICurrentTenant _currentTenant;
private readonly IRepository<Order, Guid> _orders;
public TenantOrderProcessor(ICurrentTenant currentTenant, IRepository<Order, Guid> orders)
{
_currentTenant = currentTenant;
_orders = orders;
}
public async Task ProcessAsync(Guid tenantId)
{
using (_currentTenant.Change(tenantId))
{
var pending = await _orders.GetListAsync(o => o.TotalAmount > 0);
// ... business logic; queries auto-filter to tenantId
}
// Outside the using: previous ICurrentTenant context is restored.
}
}
```
Key lines: `ICurrentTenant.Change(tenantId)` returns an `IDisposable`; the `using` block is mandatory so the previous tenant scope (often null/host) is restored on exit, even on exceptions.
Example 3 — Title: 'Cross-tenant query via IDataFilter'. Scenario: a host-side admin or reporting endpoint that needs rows from every tenant.
```csharp
public class TenantUsageAppService : ApplicationService
{
private readonly IRepository<Order, Guid> _orders;
private readonly IDataFilter _dataFilter;
public TenantUsageAppService(IRepository<Order, Guid> orders, IDataFilter dataFilter)
{
_orders = orders;
_dataFilter = dataFilter;
}
[Authorize] // host-only endpoint
public async Task<Dictionary<Guid?, int>> CountOrdersPerTenantAsync()
{
using (_dataFilter.Disable<IMultiTenant>())
{
var all = await _orders.GetListAsync();
return all.GroupBy(o => o.TenantId).ToDictionary(g => g.Key, g => g.Count());
}
}
}
```
Key lines: `_dataFilter.Disable<IMultiTenant>()` only works in the single-database model; with database-per-tenant you must instead loop over tenants and use `ICurrentTenant.Change(...)` so each iteration switches the connection string.
Example 4 — Title: 'Resolve tenant from a subdomain'. Scenario: each tenant gets its own URL like `https://acme.mydomain.com`.
```csharp
public override void ConfigureServices(ServiceConfigurationContext context)
{
Configure<AbpTenantResolveOptions>(options =>
{
// {0} is replaced by the tenant's unique name.
options.AddDomainTenantResolver("{0}.mydomain.com");
});
// If you issue tokens with OpenIddict, also enable wildcard domains in PreConfigure:
// PreConfigure<AbpOpenIddictWildcardDomainOptions>(o =>
// {
//     o.EnableWildcardDomainSupport = true;
//     o.WildcardDomainsFormat.Add("https://{0}.mydomain.com");
// });
}
```
Key lines: the format string's `{0}` placeholder is required; the contributor matches the host header against it and looks the tenant up via `ITenantStore`. Subdomain resolution has no effect unless DNS/reverse-proxy actually routes `*.mydomain.com` to the app.
Example 5 — Title: 'Restrict an application service to host or tenant side'. Scenario: a feature that should only ever be callable from the host (e.g., 'create tenant') or only from tenants (e.g., 'submit order').
```csharp
using Volo.Abp.MultiTenancy;
[MultiTenancySide(MultiTenancySides.Host)]
public class TenantProvisioningAppService : ApplicationService
{
public Task ProvisionAsync(CreateTenantDto input) { /* host-only */ return Task.CompletedTask; }
}
[MultiTenancySide(MultiTenancySides.Tenant)]
public class OrderAppService : ApplicationService
{
public Task SubmitAsync(SubmitOrderDto input) { /* tenant-only */ return Task.CompletedTask; }
}
```
Key lines: `[MultiTenancySide(MultiTenancySides.Host|Tenant|Both)]` declares which side may invoke the type; combine with permissions for full access control.
Example 6 — Title: 'Static tenant list in appsettings.json (no Tenant Management module)'. Scenario: small/internal app that doesn't need a database-backed tenant store.
```json
{
"ConnectionStrings": {
"Default": "Server=db;Database=app_main;..."
},
"Tenants": [
{
"Id": "446a5211-3d72-4339-9adc-845151f8ada0",
"Name": "acme",
"NormalizedName": "ACME",
"ConnectionStrings": {
"Default": "Server=db;Database=app_acme;..."
}
},
{
"Id": "25388015-ef1c-4355-9c18-f6b6ddbaf89d",
"Name": "contoso",
"NormalizedName": "CONTOSO"
}
]
}
```
Key lines: this section is bound to `AbpDefaultTenantStoreOptions` and consumed by `DefaultTenantStore`. A tenant without its own `ConnectionStrings` falls back to the root `Default` connection (single-database model).
## Common mistakes

- - Mistake: forgetting to implement IMultiTenant on a tenant-scoped entity. Why wrong: ABP cannot install the data filter, so rows leak across tenants and TenantId is not auto-populated on insert. Fix: declare `: AggregateRoot<Guid>, IMultiTenant` (or the equivalent base) and add `public Guid? TenantId { get; set; }`. Re-run EF migrations to add the column and a TenantId index.
- - Mistake: calling `CurrentTenant.Change(tenantId)` without a `using` block. Why wrong: `Change` returns an `IDisposable` representing a stack frame; if you don't dispose it, the tenant scope leaks to the rest of the request/scope and subsequent queries silently target the wrong tenant. Fix: always wrap in `using (CurrentTenant.Change(tenantId)) { ... }`. The pattern composes (nested `using` blocks restore in LIFO order).
- - Mistake: using `IDataFilter.Disable<IMultiTenant>()` to do a 'global' query in a database-per-tenant deployment. Why wrong: the filter only suppresses the WHERE clause on the active connection — there is no built-in cross-database fan-out. Fix: enumerate tenants via `ITenantStore`/Tenant Management module and loop with `using (CurrentTenant.Change(t.Id))` to switch connection strings, aggregating in memory or via a dedicated reporting database.
- - Mistake: registering the `UseMultiTenancy()` middleware before `UseAuthentication()`. Why wrong: `CurrentUserTenantResolveContributor` runs first in the resolver list and depends on `HttpContext.User`. With the wrong order, claim-based resolution is skipped and the resolver may fall through to a less-trusted source (query string / cookie). Fix: order is `UseAuthentication()` -> `UseMultiTenancy()` -> `UseAuthorization()`.
- - Mistake: putting an admin/host endpoint behind a tenant-resolved URL but not marking it `[MultiTenancySide(MultiTenancySides.Host)]`. Why wrong: a tenant request could otherwise reach it; ABP allows the call because there is no explicit side guard. Fix: annotate host-only services with `[MultiTenancySide(MultiTenancySides.Host)]` (and tenant-only with `MultiTenancySides.Tenant`); the framework throws `AbpAuthorizationException` on mismatched side at runtime.
- - Mistake: relying on the default `__tenant` HTTP header behind NGINX (or other reverse proxies). Why wrong: NGINX strips request headers containing underscores unless `underscores_in_headers on;` is set; the header arrives empty and the resolver falls through. Fix: either enable `underscores_in_headers on;` in NGINX, or change the key via `Configure<AbpAspNetCoreMultiTenancyOptions>(o => o.TenantKey = "X-Tenant")` and align all clients/load balancers.
- - Mistake: mutating `entity.TenantId` after creation to 'move' a record between tenants. Why wrong: related aggregates (with their own TenantId, FK constraints, audit logs, files in BLOB storing) do not move with it; you end up with cross-tenant references that the data filter happily hides. Fix: treat tenant ownership as immutable; if you need to move data, write an explicit migration use-case that re-creates the aggregate (and dependents) under the new tenant inside a single Unit of Work, ideally bracketed by `CurrentTenant.Change`.
- - Mistake: setting `AbpMultiTenancyOptions.IsEnabled = false` in some modules but leaving entities marked `IMultiTenant`. Why wrong: with the master switch off, ABP treats all rows as host (TenantId is forced to null), but pre-existing tenant rows still carry their old TenantId and become invisible/duplicated depending on filters. Fix: keep `IsEnabled` consistent across all dependent modules (the templates expose a single `MultiTenancyConsts.IsEnabled` for this reason) and don't toggle it in production once tenant data exists.
- - Mistake: forgetting that `ICurrentTenant.Change` does not bypass the `IMultiTenant` data filter — it only changes which tenant the filter targets. Why wrong: developers assume `Change(null)` exposes everything, when in fact it switches to host context (TenantId IS NULL) and host rows only. Fix: combine `IDataFilter.Disable<IMultiTenant>()` with the appropriate `CurrentTenant.Change(...)` if you need to look at multiple tenants' data within the single-database model.
## Version pins (ABP 10.3)

- ABP 10.3 ships AbpMultiTenancyOptions.IsEnabled = false by default in the bare framework; the Single-Layer / Layered / Microservice templates flip it on through `MultiTenancyConsts.IsEnabled`. If a future version changes the default, the template code is the single source of truth — don't rely on the framework default.
- The default tenant key (`__tenant`) and the default order of resolve contributors (CurrentUser -> QueryString -> Route -> Header -> Cookie) is documented for 10.3; both are subject to change in major releases. Pin to explicit configuration (`Configure<AbpAspNetCoreMultiTenancyOptions>` and `options.TenantResolvers.Clear()` + add) when ordering matters for security.
- Subdomain wildcard support for OpenIddict requires `PreConfigure<AbpOpenIddictWildcardDomainOptions>` with `EnableWildcardDomainSupport = true` and a `WildcardDomainsFormat` entry. The wildcard mechanism is OpenIddict-specific; if you migrate auth providers (e.g., back to Identity Server, or to an external IdP) the option name and shape will differ. [uncertain] whether the IdentityServer-equivalent option survives in 10.3 — IdentityServer is deprecated upstream, treat any related config as legacy.
- `IDataFilter.Disable<IMultiTenant>()` semantics for 10.3 are documented as 'must be inside a using block; supports nesting'. The default state of the IMultiTenant filter (enabled vs. disabled) is treated as enabled-when-multi-tenancy-is-enabled but is not explicitly called out in the docs as 'enabled by default' the way ISoftDelete is — [uncertain] regarding any future change to make this configurable via `AbpDataFilterOptions.DefaultStates`.
- Database-per-tenant connection strings are supported by the open-source Tenant Management module's `AbpTenantConnectionStrings` table, but the UI for managing them per tenant is part of the SaaS module (PRO). [uncertain] whether the OSS tenant management UI in 10.3 exposes connection-string editing — the docs imply it does not.
- `MultiTenancySides` enum members are stable (Host, Tenant, Both); `[MultiTenancySide(...)]` attribute is in `Volo.Abp.MultiTenancy` and is not expected to change. Behavior on application services is enforced at runtime, not compile time — tests should cover host vs. tenant invocation paths.
- Middleware order: `UseAuthentication()` -> `UseMultiTenancy()` -> `UseAuthorization()` is current guidance for 10.3. Earlier ABP versions occasionally placed `UseMultiTenancy()` before authentication; do not copy older samples blindly.
- The `Configure<AbpDefaultTenantStoreOptions>` shape (in-memory `TenantConfiguration[]`) is stable for 10.3. The `'Tenants'` configuration-section auto-binding is convenience and may be removed if a future version standardizes on the Tenant Management module — keep the explicit `Configure<AbpDefaultTenantStoreOptions>` form for safety. [uncertain] for any pending changes in 11.x.

## Cross-references

**Phase 1 references:**
- - references/framework/architecture-ddd.md — triggered when the user asks how IMultiTenant fits with aggregate roots / entities / repositories / unit of work; tenant-aware repositories rely on the DDD building blocks.
- - references/framework/data-ef-core.md — triggered when the user asks how the IMultiTenant filter is materialized at the SQL level, how DbContext picks up TenantId, how connection-string resolution per tenant works with EF Core, or how migrations behave in database-per-tenant deployments.
- - references/framework/authorization.md — triggered when the user asks about CurrentUserTenantResolveContributor, claim-based tenant resolution, or restricting permissions by host/tenant side.
- - references/framework/fundamentals.md — triggered when the user asks about Configure<TOptions>, AbpModule lifecycle, connection-string management, or [DependsOn] for AbpMultiTenancyModule / AbpAspNetCoreMultiTenancyModule.
- - references/framework/api-development.md — triggered when the user asks how the __tenant query/header/cookie keys interact with auto-generated controllers, dynamic C# clients, or Swagger.
- - references/templates/single-layer.md, references/templates/layered.md, references/templates/microservice.md — triggered when the user asks 'how is multi-tenancy wired in template X' (each template has its own multi-tenancy sub-page describing where MultiTenancyConsts.IsEnabled lives and which middleware order is pre-wired).

**External docs:**
- - ASP.NET Core middleware pipeline — https://learn.microsoft.com/aspnet/core/fundamentals/middleware/ — referenced because UseMultiTenancy() must be ordered after UseAuthentication() and before UseAuthorization() in the request pipeline.
- - ASP.NET Core authentication & claims — https://learn.microsoft.com/aspnet/core/security/authentication/ — referenced because CurrentUserTenantResolveContributor reads the tenant id from the authenticated principal's claims.
- - EF Core global query filters — https://learn.microsoft.com/ef/core/querying/filters — referenced because ABP layers the IMultiTenant filter on top of EF Core's global query filter mechanism; understanding EF Core's filter is essential when debugging unexpected row visibility.
- - NGINX HTTP headers with underscores — https://nginx.org/en/docs/http/ngx_http_core_module.html#underscores_in_headers — referenced because the default '__tenant' header contains underscores and NGINX drops such headers unless 'underscores_in_headers on;' is set.
- - OpenIddict server documentation — https://documentation.openiddict.com/ — referenced because subdomain-based tenant resolution interacts with OpenIddict's allowed redirect/issuer validation; ABP exposes AbpOpenIddictWildcardDomainOptions to bridge them.

## Sources (ABP 10.3)

- [https://abp.io/docs/10.3/framework/architecture/multi-tenancy](https://abp.io/docs/10.3/framework/architecture/multi-tenancy) — Primary source: full Multi-Tenancy architecture page covering IMultiTenant, ICurrentTenant, ITenantStore, AbpMultiTenancyOptions, AbpTenantResolveOptions, AbpAspNetCoreMultiTenancyOptions, default and custom tenant resolvers, domain/subdomain resolution, fallback tenant, middleware error handling, database-per-tenant connection strings.
- [https://abp.io/docs/10.3/framework/infrastructure/data-filtering](https://abp.io/docs/10.3/framework/infrastructure/data-filtering) — Companion source: IDataFilter / IDataFilter<TFilter> Disable/Enable/IsEnabled semantics and AbpDataFilterOptions.DefaultStates used to bypass the IMultiTenant filter for cross-tenant queries.
- [https://abp.io/docs/10.3/modules/tenant-management](https://abp.io/docs/10.3/modules/tenant-management) — Companion source: Tenant Management module (Tenant aggregate, ITenantRepository, TenantAppService, AbpTenants / AbpTenantConnectionStrings tables) — the production ITenantStore implementation referenced from the multi-tenancy page.

Last verified: 2026-05-10
