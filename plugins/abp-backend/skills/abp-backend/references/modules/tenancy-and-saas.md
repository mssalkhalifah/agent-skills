# Modules — Tenant Management & SaaS

> Reference for ABP 10.3's Tenant Management (open-source ITenantStore implementation with tenants/feature UI) and SaaS (commercial superset adding editions, per-tenant connection strings, and subscription/payment integration) modules and how they sit on top of the multi-tenancy framework.

## When to load this reference

- User asks how to enable multi-tenancy in an ABP solution and where tenants are stored.
- User asks the difference between the Tenant Management module and the SaaS module / when to upgrade.
- User asks how to configure per-tenant connection strings or implement database-per-tenant with ABP.
- User asks how Editions work, how to assign features to an edition, or how tenant subscriptions/billing are wired (CreateSubscriptionAsync, AbpSaasPaymentOptions, Volo.Payment).
- User asks how to handle TenantEto / EditionEto distributed events to react to tenant lifecycle changes.
- User asks which permissions guard tenant CRUD and feature management (AbpTenantManagement.Tenants.*).
- User asks how the tenant management UI relates to ICurrentTenant, ITenantResolver, or the __tenant query/header parameter.
- User asks about extending the Tenant entity, replacing TenantAppService, or customizing the SaaS Angular UI.

**Audience:** ABP 10.3 developers building multi-tenant or SaaS applications who need to choose between the open-source Tenant Management module and the commercial SaaS module, configure per-tenant data isolation, manage editions/subscriptions, or react to tenant lifecycle events.

## Key concepts

- Tenant Management module (Volo.Abp.TenantManagement.* NuGet/NPM packages) - Open-source MIT-licensed module that implements ITenantStore against a database and ships an admin UI for tenants and per-tenant features. Reach for this when you only need basic tenant CRUD with shared schema.
- ITenantRepository (Volo.Abp.TenantManagement) - Domain repository for the Tenant aggregate, used by TenantManager and TenantAppService. Reach for it when you need direct tenant queries from custom domain code.
- TenantManager (Volo.Abp.TenantManagement) - Domain service that encapsulates tenant creation/rename/validation rules. Reach for it instead of repository writes whenever you create or rename tenants from custom code so business invariants run.
- TenantAppService / ITenantAppService (Volo.Abp.TenantManagement) - DTO-facing application service backing the admin UI; exposes GetList/Create/Update/Delete and ManageFeatures. Reach for it when scripting tenant provisioning from another service.
- TenantManagementDbContext / ITenantManagementDbContext (Volo.Abp.TenantManagement.EntityFrameworkCore) - EF Core context with AbpTenants and AbpTenantConnectionStrings tables, configured via builder.ConfigureTenantManagement(). Reach for it when adding a navigation from your domain to Tenant or extending the Tenant entity.
- TenantManagementMongoDbContext / ITenantManagementMongoDbContext (Volo.Abp.TenantManagement.MongoDB) - MongoDB equivalent storing AbpTenants documents with embedded connection strings.
- TenantEto (Volo.Abp.TenantManagement) - Distributed event ETO published automatically on tenant entity changes. Reach for it via IDistributedEventHandler<EntityCreatedEto<TenantEto>> to seed per-tenant data or sync downstream services.
- AbpTenantManagementPermissions (Volo.Abp.TenantManagement) - Permission constants: AbpTenantManagement.Tenants and Tenants.Create/Update/Delete/ManageFeatures, plus a host-level ManageHostFeatures option.
- SaaS module (Volo.Saas.* commercial packages) - Commercial superset of Tenant Management that adds Editions, per-tenant per-module connection strings, subscriptions, and payment integration. Reach for it instead of Tenant Management when you need editions or database-per-tenant deployment.
- Edition / IEditionRepository / EditionAppService (Volo.Saas) - Aggregate representing a feature bundle assigned to tenants. Reach for it whenever pricing/feature tiers exist (Free/Standard/Enterprise).
- ISubscriptionAppService (Volo.Saas) - Application service that orchestrates payment flow: CreateSubscriptionAsync(editionId, tenantId) returns a payment request id used to redirect to the gateway selection page. Reach for it from your own sign-up/upgrade pages.
- AbpSaasPaymentOptions (Volo.Saas.Host) - Options class exposing IsPaymentSupported; flip it on to wire subscription UI and Volo.Payment webhook handling.
- SaasTenant / SaasEdition (Volo.Saas) - SaaS-side aggregates persisted to SaasTenants / SaasTenantConnectionStrings / SaasEditions tables (or matching MongoDB collections).
- EditionEto (Volo.Saas) - Distributed event ETO published on edition entity changes, complementing TenantEto.
- ICurrentTenant (Volo.Abp.MultiTenancy) - Ambient service the modules read to scope queries and to bind feature/connection-string lookups to the active tenant; CurrentTenant.Change(tenantId) opens a scope to act as another tenant.
- ITenantStore (Volo.Abp.MultiTenancy) - Abstraction the framework calls to resolve a TenantConfiguration (id, name, connection strings) by id or normalized name. Tenant Management and SaaS modules each provide their own database-backed implementation, replacing the default appsettings-based DefaultTenantStore.
- MultiTenantConnectionStringResolver (Volo.Abp.MultiTenancy) - IConnectionStringResolver implementation that consults ITenantStore for a tenant-specific connection string and falls back to AbpDbConnectionOptions. Both modules persist the strings it reads (AbpTenantConnectionStrings / SaasTenantConnectionStrings).
- TenantFeatureManagementProvider / EditionFeatureManagementProvider (Volo.Abp.FeatureManagement) - Providers that supply feature values per tenant and per edition, executed in reverse order Default -> Configuration -> Edition -> Tenant so Tenant overrides Edition. The Tenant Management module exposes only the Tenant level; the SaaS module activates the Edition level.

## Configuration pattern

Both modules follow ABP's standard module-dependency wiring rather than requiring per-service configuration. Add the appropriate module to your AbpModule via [DependsOn] and configure options inside ConfigureServices.

```csharp
[DependsOn(
    typeof(AbpTenantManagementApplicationModule),
    typeof(AbpTenantManagementHttpApiModule),
    typeof(AbpTenantManagementEntityFrameworkCoreModule),
    typeof(AbpTenantManagementWebModule)
)]
public class MyAppModule : AbpModule
{
    public override void PreConfigureServices(ServiceConfigurationContext context)
    {
        // Optional: extend the tenant entity from another module
        OneTimeRunner.Run(() =>
        {
            ObjectExtensionManager.Instance
                .ConfigureTenantManagement(
                    tenantManagement => tenantManagement.ConfigureTenant(
                        tenant => tenant.AddOrUpdateProperty<string>("BillingContactEmail")
                    ));
        });
    }

    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        // Multi-tenancy is enabled by default in ABP startup templates
        Configure<AbpMultiTenancyOptions>(options =>
        {
            options.IsEnabled = true;
        });

        // Customize the tenant resolution parameter (default is __tenant)
        Configure<AbpAspNetCoreMultiTenancyOptions>(options =>
        {
            options.TenantKey = "__tenant";
        });
    }
}
```

For the SaaS module the only required toggle is enabling payment:

```csharp
[DependsOn(typeof(SaasHostApplicationModule), typeof(PaymentWebModule))]
public class MySaasHostModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        Configure<AbpSaasPaymentOptions>(options =>
        {
            options.IsPaymentSupported = true; // wires subscription UI + webhooks via Volo.Payment
        });
    }
}
```

EF Core wiring uses the standard ConfigureTenantManagement extension on ModelBuilder inside your application's DbContext OnModelCreating, after which AbpTenants/AbpTenantConnectionStrings (or SaasTenants/SaasEditions) are migrated alongside your domain tables. Per-tenant connection strings are configured at runtime from the admin UI; the framework's MultiTenantConnectionStringResolver reads them through ITenantStore and they override AbpDbConnectionOptions for the active tenant scope. The Saas connection string itself defaults to the 'Saas' entry in ConnectionStrings, falling back to 'Default' if not provided.

## Code examples

### Handle TenantEto to seed per-tenant data

_Run setup work whenever a new tenant is created (e.g. seed reference data or notify a downstream service)._

```csharp
using System.Threading.Tasks;
using Volo.Abp.DependencyInjection;
using Volo.Abp.Domain.Entities.Events.Distributed;
using Volo.Abp.EventBus.Distributed;
using Volo.Abp.TenantManagement;

public class TenantCreatedHandler :
    IDistributedEventHandler<EntityCreatedEto<TenantEto>>,
    ITransientDependency
{
    private readonly ITenantSeeder _tenantSeeder;

    public TenantCreatedHandler(ITenantSeeder tenantSeeder)
    {
        _tenantSeeder = tenantSeeder;
    }

    public async Task HandleEventAsync(EntityCreatedEto<TenantEto> eventData)
    {
        var tenant = eventData.Entity;
        await _tenantSeeder.SeedAsync(tenant.Id, tenant.Name);
    }
}
```

**Key lines:** TenantEto only auto-publishes for the Tenant aggregate; for monolith apps subscribe via the local event bus (ILocalEventHandler) for efficiency, and via IDistributedEventHandler in microservice setups so other services receive it through the message broker.

### Run code as a specific tenant or as the host

_Seed data for a tenant from a host-side background job, or query host-level data while a tenant scope is active._

```csharp
using System;
using System.Threading.Tasks;
using Volo.Abp.MultiTenancy;

public class TenantSeeder : ITenantSeeder
{
    private readonly ICurrentTenant _currentTenant;
    private readonly IMyDataSeeder _dataSeeder;

    public TenantSeeder(ICurrentTenant currentTenant, IMyDataSeeder dataSeeder)
    {
        _currentTenant = currentTenant;
        _dataSeeder = dataSeeder;
    }

    public async Task SeedAsync(Guid tenantId, string tenantName)
    {
        using (_currentTenant.Change(tenantId, tenantName))
        {
            // All repository calls here resolve tenant-specific connection strings
            // via MultiTenantConnectionStringResolver and apply IMultiTenant filters.
            await _dataSeeder.SeedAsync();
        }
    }
}
```

**Key lines:** _currentTenant.Change(tenantId, ...) opens an AsyncLocal scope; inside it ICurrentTenant.Id matches the argument so MultiTenantConnectionStringResolver consults ITenantStore for that tenant's connection string. Pass null to act as the host.

### Create a tenant subscription and redirect to payment (SaaS module)

_Customer picks an edition on a sign-up page; you create a SaaS subscription and redirect to the payment gateway selection page provided by Volo.Payment._

```csharp
using System;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;
using Volo.Abp.MultiTenancy;
using Volo.Saas.Host.Subscription;

public class SubscribeModel : PageModel
{
    private readonly ISubscriptionAppService _subscriptionAppService;
    private readonly ICurrentTenant _currentTenant;

    public SubscribeModel(
        ISubscriptionAppService subscriptionAppService,
        ICurrentTenant currentTenant)
    {
        _subscriptionAppService = subscriptionAppService;
        _currentTenant = currentTenant;
    }

    public async Task<IActionResult> OnPostAsync(Guid editionId)
    {
        var paymentRequest = await _subscriptionAppService
            .CreateSubscriptionAsync(editionId, _currentTenant.GetId());

        return Redirect($"/Payment/GatewaySelection?paymentRequestId={paymentRequest.Id}");
    }
}
```

**Key lines:** CreateSubscriptionAsync needs both editionId and an existing tenantId, so you must create the tenant before subscribing. Set AbpSaasPaymentOptions.IsPaymentSupported = true in your host module so the Editions admin page exposes the 'Plan' link and webhooks are registered.

### Use IDataFilter to query across tenants from host code

_An admin/host job that needs to aggregate data across all tenants in a shared database._

```csharp
using System.Linq;
using System.Threading.Tasks;
using Volo.Abp.Data;
using Volo.Abp.Domain.Repositories;
using Volo.Abp.MultiTenancy;

public class CrossTenantReportService
{
    private readonly IRepository<Order, Guid> _orderRepository;
    private readonly IDataFilter _dataFilter;
    private readonly ICurrentTenant _currentTenant;

    public CrossTenantReportService(
        IRepository<Order, Guid> orderRepository,
        IDataFilter dataFilter,
        ICurrentTenant currentTenant)
    {
        _orderRepository = orderRepository;
        _dataFilter = dataFilter;
        _currentTenant = currentTenant;
    }

    public async Task<int> CountAllOrdersAsync()
    {
        using (_currentTenant.Change(null)) // act as host
        using (_dataFilter.Disable<IMultiTenant>()) // bypass TenantId filter
        {
            return await _orderRepository.CountAsync();
        }
    }
}
```

**Key lines:** Disabling IMultiTenant only works against a shared database; if tenants live in their own databases the resolver still routes the query to the current tenant's connection string, so you must iterate tenants and Change() into each one.

### Configure a domain-based tenant resolver

_Resolve tenants from a subdomain like acme.myapp.com instead of the default __tenant query parameter._

```csharp
using Volo.Abp.AspNetCore.MultiTenancy;

public override void ConfigureServices(ServiceConfigurationContext context)
{
    Configure<AbpTenantResolveOptions>(options =>
    {
        options.AddDomainTenantResolver("{0}.myapp.com");
    });

    Configure<AppUrlOptions>(options =>
    {
        options.Applications["MVC"].RootUrl = "https://{0}.myapp.com";
    });
}
```

**Key lines:** Domain resolver replaces the resolution chain order; ABP still walks CurrentUser -> QueryString -> Route -> Header -> Cookie before domain unless you call options.TenantResolvers.Clear() first. The {0} placeholder is the tenant Name (normalized) which Tenant Management/SaaS modules persist.

## Common mistakes

### Using IDataFilter.Disable<IMultiTenant>() to query 'all tenants' in a database-per-tenant setup.

**Why wrong:** IDataFilter only removes the TenantId WHERE clause; it cannot redirect a query to another tenant's database. With per-tenant connection strings the query still hits the current tenant's database and silently misses data.

**Correct pattern:** Iterate tenants from ITenantRepository (or ITenantStore.GetListAsync) and wrap each query in using (_currentTenant.Change(tenant.Id)) so MultiTenantConnectionStringResolver picks the right database. Combine with Disable<IMultiTenant>() only inside a shared-database tenant if you want host visibility.

### Calling new Tenant(...) and saving via repository instead of going through TenantManager / TenantAppService.

**Why wrong:** Bypasses normalization (tenant name uniqueness, normalization key) and skips emitting the TenantEto distributed event, so subscribers don't seed per-tenant data.

**Correct pattern:** Inject ITenantAppService.CreateAsync (or TenantManager.CreateAsync from domain code) and let the module raise events and enforce invariants. Use IDistributedEventHandler<EntityCreatedEto<TenantEto>> for follow-up work.

### Storing a per-tenant connection string in appsettings.json under ConnectionStrings:{TenantName}.

**Why wrong:** DefaultTenantStore reads tenants from the 'Tenants' configuration section, but once you install the Tenant Management or SaaS module the database-backed store replaces it; appsettings entries become dead config.

**Correct pattern:** Set the connection string from the admin UI (Tenants -> Edit -> Connection Strings) which writes AbpTenantConnectionStrings / SaasTenantConnectionStrings. Programmatic alternative: ITenantRepository.UpdateAsync after assigning Tenant.SetDefaultConnectionString(...) (or the SaaS equivalent).

### Treating the SaaS module and Tenant Management module as installable side by side.

**Why wrong:** They both implement ITenantStore; registering both yields a DI conflict at runtime and the wrong module wins. The SaaS module is a superset and is meant to replace Tenant Management.

**Correct pattern:** Pick one. Use Tenant Management for open-source / community projects without editions or billing; use the SaaS (Pro) module when you need editions, per-module per-tenant connection strings, or subscriptions. Remove Volo.Abp.TenantManagement.* package references when adopting Volo.Saas.

### Setting AbpSaasPaymentOptions.IsPaymentSupported = true without referencing Volo.Payment modules.

**Why wrong:** ISubscriptionAppService still creates the payment request, but no IPaymentRequestProvider/gateway is registered, so the redirect 404s and no webhook closes the subscription.

**Correct pattern:** Add Volo.Payment + a gateway-specific module (Volo.Payment.Stripe, Volo.Payment.PayPal, etc.) to your AbpModule [DependsOn] and configure their options before flipping IsPaymentSupported.

### Subscribing to TenantEto via the local event bus in a microservice setup.

**Why wrong:** The auto-publishing happens on the boundary that owns the Tenant aggregate; only that service receives local events. Other services miss tenant creation and never seed.

**Correct pattern:** Use IDistributedEventHandler<EntityCreatedEto<TenantEto>> in remote services (with the configured message broker), and reserve ILocalEventHandler for the host that owns the Tenant Management / SaaS module.

### Calling _currentTenant.GetId() without checking IsAvailable.

**Why wrong:** GetId throws when no tenant is active (host scope). This is a frequent NRE source on host pages that share code with tenant pages, especially on the SaaS subscription page if the user signed up before a tenant was created.

**Correct pattern:** Use ICurrentTenant.Id (nullable) and guard with IsAvailable, or call CreateTenantAsync first and then CreateSubscriptionAsync(editionId, newTenantId). Wrapping the operation in CurrentTenant.Change(newTenantId) sets context explicitly.

## Version pins (ABP 10.3)

- ABP 10.3 ships TenantEto auto-publishing for the Tenant aggregate only; other module ETOs (e.g. EditionEto) are auto-published by their owning modules but custom aggregates still need AbpDistributedEntityEventOptions.AutoEventSelectors registration.
- AbpSaasPaymentOptions lives in Volo.Saas.Host (commercial). Its property surface (IsPaymentSupported plus payment-flow hooks) has been stable across recent 10.x releases but is subject to change with the Volo.Payment module versioning.
- Default tenant resolution parameter remains __tenant in 10.3; AbpAspNetCoreMultiTenancyOptions.TenantKey controls it across query/header/route/cookie consistently.
- Database tables are AbpTenants / AbpTenantConnectionStrings for Tenant Management and SaasTenants / SaasTenantConnectionStrings / SaasEditions for the SaaS module; both use the 'Saas' connection string name with fallback to 'Default'. [uncertain] - exact column-level migrations between 10.2 and 10.3 are not detailed in the public docs.
- Tenant Management module is MIT-licensed; SaaS module requires an ABP Team license or higher in 10.3, and version availability on the SaaS package mirrors the framework version.
- Domain-based tenant resolution (DomainTenantResolver) and the AspNetCore tenant resolvers ship in Volo.Abp.AspNetCore.MultiTenancy; in 10.3 the resolver chain order is CurrentUser -> QueryString -> Route -> Header -> Cookie by default - any custom domain resolver must be inserted explicitly via AbpTenantResolveOptions.TenantResolvers.
- MultiTenantConnectionStringResolver inherits from DefaultConnectionStringResolver; replacing IConnectionStringResolver via DI overrides both modules' resolution paths, so custom resolvers must re-implement the ITenantStore lookup or call base.ResolveAsync.
- [uncertain] - the SaaS module's Angular package (@volo/abp.ng.saas) provideSaasConfig API surface in 10.3 (extensible toolbar/entity-action contributors) is not described in detail in the verified-by-AI docs page.

## Cross-references

**Phase 1 references:**
- [references/framework/multi-tenancy.md](../framework/multi-tenancy.md) — Whenever this reference talks about ICurrentTenant, IMultiTenant, tenant resolvers, AbpMultiTenancyOptions, or IDataFilter usage - those are framework primitives owned by the multi-tenancy reference.
- [references/modules/identity-and-auth.md](../modules/identity-and-auth.md) — When user provisions tenant admin users on tenant creation, or wires per-tenant identity providers - the identity module owns user/role storage that the tenant lifecycle interacts with.
- [references/modules/operations.md](../modules/operations.md) — When user manages tenant or edition features programmatically (TenantFeatureManagementProvider, EditionFeatureManagementProvider) instead of through the UI.
- [references/framework/data-ef-core.md](../framework/data-ef-core.md) — When user customizes TenantManagementDbContext, adds AbpTenants relations, or implements per-tenant database migrations.
- [references/framework/fundamentals.md](../framework/fundamentals.md) — When user configures per-tenant connection strings or implements database-per-tenant - MultiTenantConnectionStringResolver is the framework piece consuming what these modules persist.
- [references/infrastructure/eventing.md](../infrastructure/eventing.md) — When user subscribes to TenantEto / EditionEto distributed events across services.
- [references/modules/messaging-and-payment.md](../modules/messaging-and-payment.md) — Only for the SaaS module - subscription billing routes through Volo.Payment and AbpSaasPaymentOptions.IsPaymentSupported.

**External docs:**
- [ASP.NET Core configuration providers](https://learn.microsoft.com/aspnet/core/fundamentals/configuration/) — Connection strings consumed by AbpDbConnectionOptions and the default appsettings-backed ITenantStore are loaded through the standard ASP.NET Core configuration pipeline.
- [EF Core - Inheritance and shadow properties](https://learn.microsoft.com/ef/core/modeling/inheritance) — Tenant Management ships its own DbContext; extending the Tenant entity uses ABP's ObjectExtensionManager which materializes as EF Core shadow properties.
- [MongoDB .NET Driver](https://www.mongodb.com/docs/drivers/csharp/current/) — MongoDB integration package (Volo.Abp.TenantManagement.MongoDB) maps the AbpTenants collection through the official driver.
- [Stripe Payments / PayPal Subscriptions](https://stripe.com/docs/billing/subscriptions/overview) — Volo.Payment gateways (Stripe, PayPal, etc.) handle the actual checkout invoked by ISubscriptionAppService.CreateSubscriptionAsync.

## Sources (ABP 10.3)

- [https://abp.io/docs/10.3/modules/tenant-management](https://abp.io/docs/10.3/modules/tenant-management)
- [https://abp.io/docs/10.3/modules/saas](https://abp.io/docs/10.3/modules/saas)
- [https://abp.io/docs/10.3/framework/architecture/multi-tenancy](https://abp.io/docs/10.3/framework/architecture/multi-tenancy)
- [https://abp.io/docs/10.3/framework/fundamentals/connection-strings](https://abp.io/docs/10.3/framework/fundamentals/connection-strings)
- [https://abp.io/docs/10.3/modules/feature-management](https://abp.io/docs/10.3/modules/feature-management)

Last verified: 2026-05-10
