# API Development (Controllers, Integration Services, Clients, Swagger)

> Canonical reference for ABP 10.3 HTTP API surface — turning application services into REST controllers via Auto API Controllers, distinguishing client-facing from microservice-internal Integration Services, generating dynamic/static C# proxy clients, versioning APIs, wiring Swagger/OpenAPI with OAuth/OIDC, and consuming the standard application-configuration / application-localization endpoints.

## When to load this reference

- User asks how to expose an application service as an HTTP REST endpoint without writing a controller (auto API controllers, ConventionalControllers.Create, [RemoteService])
- User asks why a service shows up at /api/app/<name> or how to change the root path / route segment (RootPath, UrlControllerNameNormalizer, UrlActionNameNormalizer, UseV3UrlStyle)
- User asks how ABP picks the HTTP verb for an action (GetXxx -> GET, CreateXxx -> POST, UpdateXxx -> PUT, DeleteXxx -> DELETE, default POST) and how to override it
- User asks how to hide a service from Swagger or from the API surface entirely ([RemoteService(IsEnabled=false)] / IsMetadataEnabled=false, ControllersToRemove, [ReplaceControllers])
- User asks how to expose internal microservice-to-microservice APIs differently from external client APIs (Integration Services, /integration-api prefix, ExposeIntegrationServices)
- User asks how a .NET client should call ABP services without hand-writing HttpClient code (dynamic vs static C# proxies, AddHttpClientProxies, AddStaticHttpClientProxies, abp generate-proxy)
- User asks where to configure remote service base URLs ('RemoteServices' section in appsettings.json, AbpRemoteServiceOptions, named remote services like 'BookStore')
- User asks how to add Polly retry / circuit breakers to ABP HTTP clients (AbpHttpClientBuilderOptions.ProxyClientBuildActions)
- User asks how to version an ABP API or run multiple versions side-by-side (AddAbpApiVersioning, [ApiVersion], ICurrentApiVersionInfo, ConventionalControllers ApiVersions)
- User asks how to enable Swagger UI in an ABP app, add JWT/OpenIddict OAuth flows, or hide framework endpoints (AddAbpSwaggerGen, AddAbpSwaggerGenWithOAuth, AddAbpSwaggerGenWithOidc, UseAbpSwaggerUI, HideAbpEndpoints)
- User asks what /api/abp/application-configuration returns or how to add custom data to it (IApplicationConfigurationContributor)
- User asks how to fetch localization from the server in a custom UI (/api/abp/application-localization?cultureName=...&onlyDynamics=true)

**Audience:** Developers building or consuming ABP HTTP APIs — anyone exposing application services as REST endpoints, integrating microservices, generating C# client proxies, versioning APIs, or configuring Swagger/OpenAPI.

## Key concepts

- Auto API Controllers (Volo.Abp.AspNetCore.Mvc) — convention-based mechanism that converts every IRemoteService implementation in a target assembly into an MVC controller without writing one. Reach for it whenever you want application services callable over HTTP without boilerplate controllers.
- AbpAspNetCoreMvcOptions (Volo.Abp.AspNetCore.Mvc.AbpAspNetCoreMvcOptions) — main options class for auto controllers, used inside PreConfigureServices via PreConfigure<T>. Holds ConventionalControllers (registration list), ControllersToRemove, and ExposeIntegrationServices.
- ConventionalControllerSetting — per-assembly settings configured in the Create(...) callback: RootPath (default 'app'), UrlControllerNameNormalizer, UrlActionNameNormalizer, TypePredicate, ApplicationServiceTypes (Default | IntegrationServices | All), UseV3UrlStyle, ApiVersions.
- IRemoteService (Volo.Abp.RemoteServices) — marker interface that opts a service into auto-controller exposure. IApplicationService inherits IRemoteService, so any application service is eligible.
- [RemoteService] (Volo.Abp.RemoteServices.RemoteServiceAttribute) — class-level attribute with IsEnabled (default true) and IsMetadataEnabled (default true). Reach for it to hide a service from API/Swagger or from the entire HTTP surface.
- [ReplaceControllers] / ControllersToRemove — hooks to swap a built-in ABP controller for a custom one or drop a built-in controller entirely (e.g. AbpLanguagesController).
- [IntegrationService] (Volo.Abp.Application.Services.IntegrationServiceAttribute) — applied to interface or class. Marks a service as microservice-internal: it's hidden from /api unless ExposeIntegrationServices is true, routes under /integration-api, audit logging is off by default, and authorization is typically not required.
- AbpAuditingOptions.IsEnabledForIntegrationService — flip to true to enable audit logs for integration services (off by default).
- Dynamic C# API Client Proxies (Volo.Abp.Http.Client) — runtime-generated implementations of IApplicationService interfaces. Reach for them in fast-moving development where regenerating proxies is friction.
- AbpHttpClientModule (Volo.Abp.Http.Client.AbpHttpClientModule) — module dependency required by both dynamic and static client setups.
- AddHttpClientProxies(Assembly, [remoteServiceConfigurationName], [asDefaultServices]) — registers dynamic proxies. asDefaultServices=false forces consumers to use IHttpClientProxy<T>.
- AddStaticHttpClientProxies(Assembly, [remoteServiceConfigurationName]) — registers proxies generated by `abp generate-proxy` (Volo.Abp.Http.Client.ClientProxying). Requires AbpVirtualFileSystemModule and embedding the generated app-generate-proxy.json file.
- abp generate-proxy CLI — `abp generate-proxy -t csharp -u <baseUrl> [-m <moduleName>] [--without-contracts]`. Run against a live HTTP API to emit ClientProxies/<Service>.Generated.cs (partial), interfaces, DTOs, and the metadata JSON.
- AbpRemoteServiceOptions / RemoteServiceConfiguration — typed wrapper over the 'RemoteServices' configuration section. Default + named entries (e.g. 'BookStore') let one client target multiple services.
- IRemoteServiceConfigurationProvider — async-resolved configuration accessor used when remote URLs depend on tenant/runtime context.
- IHttpClientProxy<TService> — explicit proxy injection wrapper exposing .Service. Required when AddHttpClientProxies is registered with asDefaultServices:false.
- AbpHttpClientBuilderOptions.ProxyClientBuildActions — list of (remoteServiceName, IHttpClientBuilder) callbacks invoked at proxy registration; canonical hook for Polly retry/circuit-breaker policies (Microsoft.Extensions.Http.Polly).
- AbpApiVersioningOptions / AddAbpApiVersioning — ABP wrapper around ASPNET-API-Versioning. Configure ReportApiVersions, AssumeDefaultVersionWhenUnspecified; chain .AddApiExplorer for Swagger grouping.
- [ApiVersion] (Microsoft.AspNetCore.Mvc.ApiVersionAttribute) — declares supported version(s) on a controller; Deprecated=true tags retired versions.
- ICurrentApiVersionInfo (Volo.Abp.Http.Client.ClientProxying) — runtime version-switch ambient: .Change(new ApiVersionInfo(ParameterBindingSources.Query, '4.0')) inside a using block forces the call to a specific version.
- Volo.Abp.Swashbuckle (AbpSwashbuckleModule) — wraps Swashbuckle.AspNetCore. Provides AddAbpSwaggerGen, AddAbpSwaggerGenWithOAuth, AddAbpSwaggerGenWithOidc, UseAbpSwaggerUI, and HideAbpEndpoints.
- AbpSwaggerOidcFlows enum — AuthorizationCode (default, recommended), Implicit (deprecated), Password (legacy), ClientCredentials (server-to-server). Drives which OIDC flow Swagger UI offers.
- IApplicationConfigurationContributor (Volo.Abp.AspNetCore.Mvc.ApplicationConfigurations) — extension point for /api/abp/application-configuration; register via AbpApplicationConfigurationOptions.Contributors.
- /api/abp/application-configuration — pre-built endpoint returning auth/granted-policies, settings, current user, current tenant, and timing info. Mirrored in MVC by /Abp/ApplicationConfigurationScript.
- /api/abp/application-localization — pre-built endpoint returning localization resources for a culture; takes cultureName (required) and onlyDynamics (optional, default false). Mirrored by /Abp/ApplicationLocalizationScript for MVC.

## Configuration pattern

Auto controllers and HTTP-client proxies are wired from the consuming module's `PreConfigureServices` (for AbpAspNetCoreMvcOptions, which must be set before MVC builds its action descriptors) and `ConfigureServices` (for client-side proxy registration, Swagger, and versioning). Server-side pattern:

```csharp
[DependsOn(
    typeof(AbpAspNetCoreMvcModule),
    typeof(AbpSwashbuckleModule),
    typeof(MyApplicationModule)
)]
public class MyHttpApiHostModule : AbpModule
{
    public override void PreConfigureServices(ServiceConfigurationContext context)
    {
        // Auto API Controllers — register the application module's assembly
        PreConfigure<AbpAspNetCoreMvcOptions>(options =>
        {
            options.ConventionalControllers.Create(
                typeof(MyApplicationModule).Assembly,
                opts =>
                {
                    opts.RootPath = "app";              // /api/app/<service>
                    opts.UseV3UrlStyle = false;          // kebab-case routes (4.x+ default)
                    opts.ApplicationServiceTypes =
                        ApplicationServiceTypes.Default; // exclude integration services
                });
        });
    }

    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        var configuration = context.Services.GetConfiguration();

        // Allow integration services on this host (off by default)
        Configure<AbpAspNetCoreMvcOptions>(options =>
        {
            options.ExposeIntegrationServices = true;
        });

        // Audit logs for integration services (off by default)
        Configure<AbpAuditingOptions>(options =>
        {
            options.IsEnabledForIntegrationService = true;
        });

        // API versioning
        context.Services.AddTransient<IApiControllerFilter, NoControllerFilter>();
        context.Services
            .AddAbpApiVersioning(options =>
            {
                options.ReportApiVersions = true;
                options.AssumeDefaultVersionWhenUnspecified = true;
            })
            .AddApiExplorer(options =>
            {
                options.GroupNameFormat = "'v'VVV";
                options.SubstituteApiVersionInUrl = true;
            });

        // Swagger with OpenIddict OAuth
        context.Services.AddAbpSwaggerGenWithOAuth(
            authority: configuration["AuthServer:Authority"],
            scopes: new Dictionary<string, string> { { "MyApi", "My API" } },
            options =>
            {
                options.SwaggerDoc("v1",
                    new OpenApiInfo { Title = "My API", Version = "v1" });
                options.DocInclusionPredicate((docName, description) => true);
                options.CustomSchemaIds(type => type.FullName);
                options.HideAbpEndpoints();
            });
    }

    public override void OnApplicationInitialization(ApplicationInitializationContext context)
    {
        var app = context.GetApplicationBuilder();

        app.UseStaticFiles();
        app.UseSwagger();
        app.UseAbpSwaggerUI(options =>
        {
            options.SwaggerEndpoint("/swagger/v1/swagger.json", "My API v1");
            options.OAuthClientId("MyApi_Swagger");
            options.OAuthScopes("MyApi");
        });
    }
}
```

Client-side pattern (a console / worker / another service consuming the API):

```csharp
[DependsOn(
    typeof(AbpHttpClientModule),
    typeof(AbpVirtualFileSystemModule),  // only needed for static proxies
    typeof(MyApplicationContractsModule)
)]
public class MyClientModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        // Dynamic proxies (no codegen step)
        context.Services.AddHttpClientProxies(
            typeof(MyApplicationContractsModule).Assembly,
            remoteServiceConfigurationName: "Default");

        // OR static proxies generated by `abp generate-proxy -t csharp -u <url>`
        // context.Services.AddStaticHttpClientProxies(
        //     typeof(MyClientModule).Assembly,
        //     remoteServiceConfigurationName: "Default");
        //
        // Configure<AbpVirtualFileSystemOptions>(options =>
        //     options.FileSets.AddEmbedded<MyClientModule>());
    }

    public override void PreConfigureServices(ServiceConfigurationContext context)
    {
        // Polly retry policy on every proxy HttpClient
        PreConfigure<AbpHttpClientBuilderOptions>(options =>
        {
            options.ProxyClientBuildActions.Add((remoteServiceName, builder) =>
            {
                builder.AddTransientHttpErrorPolicy(p =>
                    p.WaitAndRetryAsync(3, i => TimeSpan.FromSeconds(Math.Pow(2, i))));
            });
        });
    }
}
```

With `appsettings.json`:

```json
{
  "RemoteServices": {
    "Default":   { "BaseUrl": "https://localhost:44345/" },
    "BookStore": { "BaseUrl": "https://localhost:48392/" }
  },
  "AuthServer": {
    "Authority": "https://localhost:44341"
  }
}
```

Key timing rule: AbpAspNetCoreMvcOptions.ConventionalControllers must be configured in PreConfigureServices — by ConfigureServices the MVC application model is already being assembled and additions silently no-op. AbpHttpClientBuilderOptions.ProxyClientBuildActions has the same constraint (PreConfigure only).

## Code examples

### Expose application services as auto API controllers

_Standard server-side wiring: an HttpApi.Host module turns every application service in BookStoreApplicationModule into a REST controller under /api/app/..._

```csharp
using Microsoft.OpenApi.Models;
using Volo.Abp.Application.Services;
using Volo.Abp.AspNetCore.Mvc;
using Volo.Abp.Modularity;
using Volo.Abp.Swashbuckle;

namespace BookStore;

[DependsOn(
    typeof(AbpAspNetCoreMvcModule),
    typeof(AbpSwashbuckleModule),
    typeof(BookStoreApplicationModule)
)]
public class BookStoreHttpApiHostModule : AbpModule
{
    public override void PreConfigureServices(ServiceConfigurationContext context)
    {
        PreConfigure<AbpAspNetCoreMvcOptions>(options =>
        {
            options.ConventionalControllers.Create(
                typeof(BookStoreApplicationModule).Assembly,
                opts =>
                {
                    opts.RootPath = "app";
                    opts.ApplicationServiceTypes = ApplicationServiceTypes.Default;
                });
        });
    }

    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        context.Services.AddAbpSwaggerGen(options =>
        {
            options.SwaggerDoc("v1", new OpenApiInfo { Title = "BookStore", Version = "v1" });
            options.DocInclusionPredicate((_, _) => true);
            options.CustomSchemaIds(t => t.FullName);
        });
    }

    public override void OnApplicationInitialization(ApplicationInitializationContext context)
    {
        var app = context.GetApplicationBuilder();
        app.UseRouting();
        app.UseConfiguredEndpoints();
        app.UseSwagger();
        app.UseAbpSwaggerUI(o => o.SwaggerEndpoint("/swagger/v1/swagger.json", "BookStore v1"));
    }
}
```

**Key lines:** PreConfigure<AbpAspNetCoreMvcOptions>(...).ConventionalControllers.Create(assembly, opts => ...) is the load-bearing line — it must be in PreConfigureServices because MVC's action discovery happens before ConfigureServices completes for downstream modules. RootPath="app" is the default and produces /api/app/<service>/<action> kebab-case routes; ApplicationServiceTypes.Default excludes [IntegrationService] services from this assembly's auto-controllers.

### Hide a service from the HTTP surface and from Swagger

_An internal background-only service should not be reachable over HTTP and should not show up in Swagger._

```csharp
using Volo.Abp.Application.Services;
using Volo.Abp.RemoteServices;

namespace BookStore.Internal;

// Not exposed as an auto controller; not in Swagger.
[RemoteService(IsEnabled = false)]
public class CleanupAppService : ApplicationService, ICleanupAppService
{
    public Task PurgeAsync() => Task.CompletedTask;
}

// Exposed over HTTP, but hidden from Swagger / API Explorer metadata.
[RemoteService(IsMetadataEnabled = false)]
public class TelemetryAppService : ApplicationService, ITelemetryAppService
{
    public Task IngestAsync(EventDto e) => Task.CompletedTask;
}
```

**Key lines:** [RemoteService(IsEnabled=false)] removes the service from auto-controller generation entirely. [RemoteService(IsMetadataEnabled=false)] keeps the route live but suppresses API Explorer descriptors so Swagger UI ignores it. IsMetadataEnabled granularity is per-class, not per-method — split methods into separate services to hide a subset.

### Mark a service as internal microservice-only via [IntegrationService]

_A microservice exposes IInventoryIntegrationService for sister microservices behind the API gateway. The gateway must block external traffic to /integration-api/._

```csharp
using Volo.Abp.Application.Services;

namespace Inventory.Application.Contracts;

// Apply on the contract interface — implementation class doesn't need it.
[IntegrationService]
public interface IInventoryIntegrationService : IApplicationService
{
    Task<int> GetStockLevelAsync(Guid productId);
}

// Host module that needs to expose integration services to other microservices:
[DependsOn(typeof(InventoryApplicationModule))]
public class InventoryIntegrationHostModule : AbpModule
{
    public override void PreConfigureServices(ServiceConfigurationContext context)
    {
        PreConfigure<AbpAspNetCoreMvcOptions>(options =>
        {
            options.ConventionalControllers.Create(
                typeof(InventoryApplicationModule).Assembly,
                opts =>
                {
                    opts.ApplicationServiceTypes =
                        ApplicationServiceTypes.IntegrationServices;
                });
        });
    }

    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        Configure<AbpAspNetCoreMvcOptions>(options =>
        {
            options.ExposeIntegrationServices = true; // off by default
        });
        Configure<AbpAuditingOptions>(options =>
        {
            options.IsEnabledForIntegrationService = true;
        });
    }
}
```

**Key lines:** [IntegrationService] on the interface is enough — the implementation inherits it. The service routes under /integration-api/inventory/... instead of /api/.... ExposeIntegrationServices=true is mandatory; without it, integration services don't even register as MVC actions. ApplicationServiceTypes.IntegrationServices on the controller registration filters the assembly down to integration-only services for this host. Always pair with an API gateway rule that drops external requests to /integration-api/*.

### In-process integration service for a Layered modular monolith

_Two ABP modules in the same host (Forms producer, Reports consumer). Same `[IntegrationService]` contract as the microservice example — DI resolves it to the in-process implementation, no HTTP hop, but the contract stays portable to the eventual microservice split._

```csharp
// === Producer side: Forms module ===
// Forms.Contracts/IFormSubmissionIntegrationService.cs
using Volo.Abp.Application.Services;

namespace Acme.Forms;

[IntegrationService]
public interface IFormSubmissionIntegrationService : IApplicationService
{
    Task<int> GetSubmissionCountForOwnerAsync(Guid ownerId);
}

// Forms/IntegrationServices/FormSubmissionIntegrationService.cs
public class FormSubmissionIntegrationService
    : ApplicationService, IFormSubmissionIntegrationService
{
    private readonly IRepository<FormSubmission, Guid> _submissionRepository;
    public FormSubmissionIntegrationService(IRepository<FormSubmission, Guid> r)
        => _submissionRepository = r;

    public Task<int> GetSubmissionCountForOwnerAsync(Guid ownerId)
        => _submissionRepository.CountAsync(s => s.OwnerId == ownerId);
}

// === Consumer side: Reports module ===
// Reports.Application/Services/ReportAppService.cs
public class ReportAppService : ApplicationService, IReportAppService
{
    private readonly IFormSubmissionIntegrationService _formSubmissions;
    private readonly IRepository<Report, Guid> _reportRepository;

    public ReportAppService(
        IFormSubmissionIntegrationService formSubmissions,    // ← injected as a normal service
        IRepository<Report, Guid> reportRepository)
    {
        _formSubmissions = formSubmissions;
        _reportRepository = reportRepository;
    }

    public async Task<ReportDto> BuildAsync(Guid ownerId)
    {
        var submissionCount = await _formSubmissions.GetSubmissionCountForOwnerAsync(ownerId);
        // ...assemble report...
    }
}

// === Host wiring ===
// Reports module DependsOn the producer's Contracts module (NOT the producer's app module):
[DependsOn(
    typeof(AcmeFormsApplicationContractsModule),
    typeof(AcmeReportsDomainModule)
)]
public class AcmeReportsApplicationModule : AbpModule { }

// The host (or Forms application module) registers the integration service implementation
// alongside the regular application services — typically via [DependsOn] of the Forms application
// module on the host. ExposeIntegrationServices is left at false because no HTTP exposure is needed.
```

**Key lines:** the consumer module depends on the producer's `*.Contracts` module, not on the implementation. DI resolves `IFormSubmissionIntegrationService` to the in-process class registered by the producer's application module. `ExposeIntegrationServices` stays `false` — there is no HTTP hop, so no `/integration-api/...` routes are needed. The same `[IntegrationService]` interface remains portable: when (if) Forms is extracted into its own microservice, swap the in-process registration for a generated `IHttpClientProxy<IFormSubmissionIntegrationService>` and the consumer code does not change.

### Consume an ABP API with dynamic or static C# proxies

_A worker service calls IBookAppService over HTTP. Same client-code, swap one registration line to switch from runtime to compile-time proxies._

```csharp
using Volo.Abp.DependencyInjection;
using Volo.Abp.Http.Client;
using Volo.Abp.Modularity;
using Volo.Abp.VirtualFileSystem;

namespace Reporting;

[DependsOn(
    typeof(AbpHttpClientModule),
    typeof(AbpVirtualFileSystemModule),
    typeof(BookStoreApplicationContractsModule)
)]
public class ReportingClientModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        // (a) Dynamic — discovers endpoints at runtime; no codegen required.
        context.Services.AddHttpClientProxies(
            typeof(BookStoreApplicationContractsModule).Assembly,
            remoteServiceConfigurationName: "BookStore");

        // (b) Static — uses files emitted by `abp generate-proxy -t csharp -u ...`.
        // context.Services.AddStaticHttpClientProxies(
        //     typeof(ReportingClientModule).Assembly,
        //     remoteServiceConfigurationName: "BookStore");
        // Configure<AbpVirtualFileSystemOptions>(o =>
        //     o.FileSets.AddEmbedded<ReportingClientModule>());
    }
}

public class ReportingService : ITransientDependency
{
    private readonly IBookAppService _books;
    public ReportingService(IBookAppService books) => _books = books;

    public async Task RunAsync()
    {
        var list = await _books.GetListAsync(); // HTTP GET /api/app/book
    }
}

// appsettings.json:
// "RemoteServices": { "BookStore": { "BaseUrl": "https://bookstore.local/" } }
```

**Key lines:** AddHttpClientProxies = dynamic, AddStaticHttpClientProxies = static — the only line that changes between approaches. Static requires AbpVirtualFileSystemModule + embedding the generated app-generate-proxy.json so the proxy can resolve endpoint metadata. The CLI command is `abp generate-proxy -t csharp -u <runningApiUrl>`; rerun after every API contract change. remoteServiceConfigurationName binds to the RemoteServices:BookStore key in appsettings.json; omit it (or pass "Default") to use RemoteServices:Default. Consumer code is identical either way — just inject the interface.

### Run two API versions side-by-side via auto controllers

_TodoAppService v1 stays online (deprecated) while v2 ships in a new namespace; clients pass api-version as a query string._

```csharp
using Asp.Versioning;
using Volo.Abp.AspNetCore.Mvc;
using Volo.Abp.Modularity;

public class TodoApiHostModule : AbpModule
{
    public override void PreConfigureServices(ServiceConfigurationContext context)
    {
        PreConfigure<AbpAspNetCoreMvcOptions>(options =>
        {
            // v1
            options.ConventionalControllers.Create(
                typeof(TodoApplicationModule).Assembly,
                opts =>
                {
                    opts.TypePredicate = t =>
                        t.Namespace == typeof(Todo.v1.TodoAppService).Namespace;
                    opts.ApiVersions.Add(new ApiVersion(1, 0));
                });

            // v2
            options.ConventionalControllers.Create(
                typeof(TodoApplicationModule).Assembly,
                opts =>
                {
                    opts.TypePredicate = t =>
                        t.Namespace == typeof(Todo.v2.TodoAppService).Namespace;
                    opts.ApiVersions.Add(new ApiVersion(2, 0));
                });
        });
    }

    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        context.Services.AddTransient<IApiControllerFilter, NoControllerFilter>();
        context.Services
            .AddAbpApiVersioning(o =>
            {
                o.ReportApiVersions = true;
                o.AssumeDefaultVersionWhenUnspecified = true;
            })
            .AddApiExplorer(o =>
            {
                o.GroupNameFormat = "'v'VVV";
                o.SubstituteApiVersionInUrl = true;
            });

        Configure<AbpAspNetCoreMvcOptions>(options =>
        {
            options.ChangeControllerModelApiExplorerGroupName = false;
        });
    }
}
```

**Key lines:** TypePredicate scopes each ConventionalControllers.Create call to a single namespace, and ApiVersions.Add tags every controller it creates. NoControllerFilter is required so ABP doesn't strip the [ApiController] adornment off versioned controllers. ChangeControllerModelApiExplorerGroupName=false keeps version-based grouping intact for Swagger. Clients call with `?api-version=2.0`; ABP recommends query-string versioning over URL-segment versioning because static C# / JS proxies generate cleaner code with it.

### Swagger with OpenIddict OAuth and a contributor on the application-configuration endpoint

_Production API host wires Swagger UI with the OpenIddict authority, hides built-in framework endpoints, and adds a custom property to /api/abp/application-configuration so SPA clients can read a feature flag without a second round-trip._

```csharp
using Microsoft.OpenApi.Models;
using Volo.Abp.AspNetCore.Mvc.ApplicationConfigurations;
using Volo.Abp.Modularity;
using Volo.Abp.Swashbuckle;

public class CustomConfigContributor : IApplicationConfigurationContributor
{
    public Task ContributeAsync(ApplicationConfigurationContributorContext context)
    {
        context.ApplicationConfiguration.SetProperty(
            "betaFeatures",
            new { newReports = true });
        return Task.CompletedTask;
    }
}

[DependsOn(typeof(AbpSwashbuckleModule))]
public class MyApiHostModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        var configuration = context.Services.GetConfiguration();

        Configure<AbpApplicationConfigurationOptions>(options =>
        {
            options.Contributors.AddIfNotContains(new CustomConfigContributor());
        });

        context.Services.AddAbpSwaggerGenWithOAuth(
            authority: configuration["AuthServer:Authority"],
            scopes: new Dictionary<string, string> { { "MyApi", "My API" } },
            options =>
            {
                options.SwaggerDoc("v1", new OpenApiInfo { Title = "MyApi", Version = "v1" });
                options.DocInclusionPredicate((_, _) => true);
                options.CustomSchemaIds(t => t.FullName);
                options.HideAbpEndpoints();
            });
    }

    public override void OnApplicationInitialization(ApplicationInitializationContext context)
    {
        var app = context.GetApplicationBuilder();
        app.UseSwagger();
        app.UseAbpSwaggerUI(opts =>
        {
            opts.SwaggerEndpoint("/swagger/v1/swagger.json", "My API");
            opts.OAuthClientId("MyApi_Swagger");
            opts.OAuthScopes("MyApi");
        });
    }
}
```

**Key lines:** AddAbpSwaggerGenWithOAuth pre-wires the Swagger UI OAuth dialog against an OpenIddict authority — pass the authority URL plus the scope dictionary. HideAbpEndpoints() removes ABP's own framework controllers (account, identity admin, etc.) from the doc. IApplicationConfigurationContributor with SetProperty extends /api/abp/application-configuration; the SPA's config-state service surfaces betaFeatures.newReports without any new endpoint. For Kubernetes deployments where the discovery URL differs from the issuer URL, swap to AddAbpSwaggerGenWithOidc(authority, scopes, flows, discoveryEndpoint, options).

## Common mistakes

### Calling `Configure<AbpAspNetCoreMvcOptions>(o => o.ConventionalControllers.Create(...))` in ConfigureServices instead of PreConfigureServices.

**Why wrong:** By the time ConfigureServices runs for downstream modules, MVC has already started building action descriptors. ConventionalControllers entries added there are silently ignored — your services don't get controllers and there is no error.

**Correct pattern:** Always wrap ConventionalControllers.Create in PreConfigure<AbpAspNetCoreMvcOptions> inside PreConfigureServices.

```csharp
PreConfigure<AbpAspNetCoreMvcOptions>(options => options.ConventionalControllers.Create(typeof(MyAppModule).Assembly));
```

### Forgetting to set ExposeIntegrationServices=true on the host that should expose [IntegrationService] services.

**Why wrong:** Integration services are hidden from /api by default and are also filtered out of MVC action descriptors unless ExposeIntegrationServices is true. The integration host returns 404 for every /integration-api/* call.

**Correct pattern:** On the integration host, set Configure<AbpAspNetCoreMvcOptions>(o => o.ExposeIntegrationServices = true) AND register a ConventionalControllers.Create with ApplicationServiceTypes.IntegrationServices.

```csharp
Configure<AbpAspNetCoreMvcOptions>(options => options.ExposeIntegrationServices = true);
```

### Exposing /integration-api through the public API gateway.

**Why wrong:** Integration services are designed to skip authorization checks — they assume the caller is another trusted microservice in the private network. Letting external traffic reach /integration-api/* allows unauthenticated access to internal contracts.

**Correct pattern:** Add a gateway rule (YARP/Ocelot/whatever) that explicitly drops or 404s any external request whose path starts with /integration-api/, and only expose /api/* to clients.

```csharp
Treat /integration-api as a private-network-only path; deny it at the gateway and document the boundary.
```

### Pinning every action with [HttpGet]/[HttpPost] manually on application services.

**Why wrong:** Auto controllers already infer the HTTP verb from the method name prefix (Get/GetList/GetAll -> GET, Create/Add/Insert/Post -> POST, Update/Put -> PUT, Delete/Remove -> DELETE, Patch -> PATCH, default POST). Hand-applied attributes obscure intent and risk diverging from the convention.

**Correct pattern:** Name methods to match the verb convention. Use HTTP attributes only when you genuinely need a route or verb the convention doesn't produce.

```csharp
Rename `DoCreateAsync` -> `CreateAsync`; rename `FetchAllAsync` -> `GetListAsync`. Drop the [HttpPost]/[HttpGet] attributes.
```

### Forgetting to embed app-generate-proxy.json (or the AddEmbedded fileset) when using static client proxies.

**Why wrong:** Static proxies discover endpoint metadata from this embedded JSON. Without the embedded fileset registration, the proxy throws at startup or at the first call with a missing-resource error.

**Correct pattern:** After `abp generate-proxy -t csharp`, set the generated `app-generate-proxy.json` to EmbeddedResource in the .csproj and add Configure<AbpVirtualFileSystemOptions>(o => o.FileSets.AddEmbedded<MyClientModule>()) in the client module.

```xml
<EmbeddedResource Include="ClientProxies/app-generate-proxy.json" /> + AddEmbedded<MyClientModule>().
```

### Pointing `appsettings.json` RemoteServices.Default at the wrong base URL (missing trailing slash, https/http mismatch, container DNS not resolvable).

**Why wrong:** Both dynamic and static proxies concatenate the BaseUrl with the discovered route. A missing trailing slash collapses paths (`https://api/api/app/book` becomes `https://api` because Uri composition drops the segment); http/https mismatches with HSTS produce 307 loops; in-cluster DNS that only resolves inside a Pod fails locally.

**Correct pattern:** Always include the trailing slash, match the scheme to the actual gateway, and keep RemoteServices configuration per-environment.

```csharp
"BaseUrl": "https://my-service:443/" — note the trailing slash and matching scheme.
```

### Versioning controllers without registering NoControllerFilter.

**Why wrong:** ASPNET-API-Versioning's default IApiControllerFilter strips ABP's auto-generated controllers because they don't carry the [ApiController] adornment in the way the filter expects, leaving versioned routes unreachable.

**Correct pattern:** Register `services.AddTransient<IApiControllerFilter, NoControllerFilter>()` BEFORE AddAbpApiVersioning so ABP's NoControllerFilter wins the DI registration.

```csharp
context.Services.AddTransient<IApiControllerFilter, NoControllerFilter>(); context.Services.AddAbpApiVersioning(...);
```

### Using URL-path versioning (/api/v{version}/...) and then expecting static C#/JS proxies to work.

**Why wrong:** ABP's documentation explicitly recommends query-string versioning because the proxy generators emit cleaner code for `?api-version=` than for path segments, and the JS proxies hard-code the version into a query parameter.

**Correct pattern:** Use query-string versioning with AssumeDefaultVersionWhenUnspecified=true; let proxies pass api-version automatically.

```csharp
options.ApiVersionReader = new QueryStringApiVersionReader("api-version");
```

### Calling AddAbpSwaggerGenWithOAuth without HideAbpEndpoints in a customer-facing API.

**Why wrong:** Swagger then displays every framework endpoint (Identity, Tenant management, Account, Permission management, Feature management, Setting management). Customers see internal admin endpoints they cannot use, and the schema balloons.

**Correct pattern:** Call options.HideAbpEndpoints() inside AddAbpSwaggerGen* for any tenant-facing or customer-facing API host.

```csharp
options.HideAbpEndpoints();
```

### Adding `[Authorize]` to an [IntegrationService] and being surprised it 401s every internal call.

**Why wrong:** Integration services are designed to flow through with the calling microservice's machine identity (or no identity) and are off the authorization happy path by convention. Slapping [Authorize] on them re-enables full token requirements without adjusting the calling side.

**Correct pattern:** If integration services need authorization, use a service-to-service auth strategy (OpenIddict client credentials flow with a dedicated client) and configure the calling proxy to attach the bearer token; otherwise rely on private-network isolation.

```csharp
Add a dedicated OpenIddict client_credentials application for the caller and propagate that token via the proxy's AbpHttpClientBuilderOptions.
```

## Version pins (ABP 10.3)

- ABP 10.3 ships with the kebab-case URL convention by default (4.x+ behavior). UseV3UrlStyle=true forces the legacy camelCase routes; do not flip this on a new project — it's only for migrating old clients.
- Volo.Abp.Swashbuckle wraps Swashbuckle.AspNetCore; the underlying Swashbuckle major version that ships with ABP 10.3 follows the .NET 9 / ASP.NET Core 9 line. [uncertain] — exact pinned Swashbuckle version may change within 10.x patch releases.
- AddAbpSwaggerGenWithOidc with AbpSwaggerOidcFlows.Implicit is documented as deprecated; new deployments should use AuthorizationCode (PKCE) — Implicit may be removed in a future major release.
- ASPNET-API-Versioning packages renamed from `Microsoft.AspNetCore.Mvc.Versioning` to `Asp.Versioning.*` (Asp.Versioning.Mvc.ApiExplorer etc.) — code lifted from older ABP 6.x/7.x samples needs the new namespaces.
- AbpAuditingOptions.IsEnabledForIntegrationService default is false in 10.3; enabling it can substantially increase audit log volume in chatty service-to-service systems.
- AbpAspNetCoreMvcOptions.ExposeIntegrationServices default is false. Microservice templates flip it on per host — single-layer / layered templates leave it off because they don't ship integration services by default. [uncertain] for any specific custom template.
- RemoteServiceConfiguration in ABP 10.3 supports per-tenant base URL resolution via IRemoteServiceConfigurationProvider; older synchronous accessor patterns from 6.x are removed.
- The `abp generate-proxy` CLI requires the target API to be running and reachable when you invoke it; in 10.3 it discovers metadata from the live `/api/abp/api-definition` endpoint. CI pipelines that run codegen need a pre-running service or recorded metadata.
- Application Configuration endpoint (/api/abp/application-configuration) shape is stable through 10.x but additive changes (extra properties) happen between minor versions — clients should treat it as forward-compatible JSON, not as a frozen contract.
- ChangeControllerModelApiExplorerGroupName default behavior interacts with API versioning: leave it false when using Swagger version groups so Swashbuckle keeps the version-derived group name; flipping it true overrides ABP's default grouping logic. [uncertain] — not exhaustively documented in 10.3 release notes.

## Cross-references

**Phase 1 references:**
- [references/framework/fundamentals.md](../framework/fundamentals.md) — triggered when the user asks about PreConfigureServices vs ConfigureServices ordering, IOptions configuration timing, [DependsOn] module wiring, or where AbpHttpClientBuilderOptions / AbpAspNetCoreMvcOptions live in the lifecycle.
- [references/framework/architecture-ddd.md](../framework/architecture-ddd.md) — triggered when the user asks about IApplicationService / ApplicationService, what an application service contract looks like, or DTO shapes; auto controllers and proxies require an IApplicationService surface and matching DTOs.
- [references/framework/architecture-modularity.md](../framework/architecture-modularity.md) — triggered when the user asks about adding integration services to an existing module, replacing a built-in controller via [ReplaceControllers], or extending the ABP Application Configuration response across modules.
- [references/framework/multi-tenancy.md](../framework/multi-tenancy.md) — triggered when the user asks why TenantId is auto-propagated on dynamic/static proxy calls, how the application-configuration endpoint surfaces currentTenant, or how /integration-api requests behave under multi-tenancy.
- [references/framework/authorization.md](../framework/authorization.md) — triggered when the user asks about [Authorize] / [AllowAnonymous] on auto controllers, why integration services bypass authorization by default, or how Swagger's OAuth flows obtain tokens that satisfy granted-policies returned by /api/abp/application-configuration.
- [references/framework/validation.md](../framework/validation.md) — triggered when the user asks about model validation behavior on auto controllers, FluentValidation integration, or DTO validation surfaced through Swagger schemas.
- [references/templates/single-layer.md](../templates/single-layer.md) — triggered when the user asks how Swagger is wired in the single-layer template (out-of-the-box AddAbpSwaggerGen call site).
- [references/templates/layered.md](../templates/layered.md) — triggered when the user asks how the *.HttpApi.Host project hosts auto controllers and Swagger in the layered template.
- [references/templates/microservice.md](../templates/microservice.md) — triggered when the user asks how integration services and gateway routing for /integration-api work in the microservice template, and how static/dynamic proxies interact with the API gateway.

**External docs:**
- [ASP.NET Core MVC controllers and routing](https://learn.microsoft.com/aspnet/core/mvc/controllers/routing) — ABP auto controllers piggyback on standard ASP.NET Core attribute routing, [HttpGet]/[HttpPost], and [ApiController] semantics — overrides flow through the same primitives.
- [Swashbuckle.AspNetCore](https://github.com/domaindrivendev/Swashbuckle.AspNetCore) — Volo.Abp.Swashbuckle wraps Swashbuckle; AddAbpSwaggerGen forwards its options builder to AddSwaggerGen, so all underlying configuration knobs (operation filters, schema filters, document filters) come from the Swashbuckle docs.
- [ASP.NET API Versioning (Asp.Versioning.*)](https://github.com/dotnet/aspnet-api-versioning) — AbpApiVersioning is a thin wrapper. ApiVersion attribute, ApiVersionReader strategies (query/header/URL), and IApiVersionDescriptionProvider all come from this library.
- [Microsoft.Extensions.Http.Polly](https://learn.microsoft.com/dotnet/architecture/microservices/implement-resilient-applications/use-httpclientfactory-to-implement-resilient-http-requests) — Polly's AddTransientHttpErrorPolicy / AddPolicyHandler are the actual extension methods invoked from inside AbpHttpClientBuilderOptions.ProxyClientBuildActions for retry/circuit-breaker/timeout policies.
- [OpenAPI Specification (OAS 3.x)](https://spec.openapis.org/oas/latest.html) — Swagger UI in ABP renders an OAS document; understanding OperationId, schemes, and security scheme objects matters when customizing AddAbpSwaggerGenWithOAuth/Oidc.
- [OpenIddict documentation](https://documentation.openiddict.com/) — AddAbpSwaggerGenWithOAuth/Oidc target an OpenIddict authority by default in ABP templates — Swagger client IDs/scopes must be registered as OpenIddict applications to grant tokens.
- [ASP.NET Core HttpClientFactory](https://learn.microsoft.com/aspnet/core/fundamentals/http-requests) — AddHttpClientProxies / AddStaticHttpClientProxies register typed clients through IHttpClientFactory, so handler lifetime, named clients, and DelegatingHandler patterns from this doc apply directly.

## Sources (ABP 10.3)

- [https://abp.io/docs/10.3/framework/api-development](https://abp.io/docs/10.3/framework/api-development) — Index page listing the API Development sub-topics: ABP Endpoints, Auto API Controllers, Integration Services, Dynamic/Static C# Clients, Versioning, Swagger Integration
- [https://abp.io/docs/10.3/framework/api-development/auto-controllers](https://abp.io/docs/10.3/framework/api-development/auto-controllers) — Auto API Controllers: ConventionalControllers.Create, AbpAspNetCoreMvcOptions, route/HTTP-method conventions, [RemoteService], [ReplaceControllers], UseV3UrlStyle, controller/action name normalizers
- [https://abp.io/docs/10.3/framework/api-development/integration-services](https://abp.io/docs/10.3/framework/api-development/integration-services) — [IntegrationService] attribute, /integration-api/ prefix, ExposeIntegrationServices flag, AbpAuditingOptions.IsEnabledForIntegrationService, ApplicationServiceTypes filter, microservice-to-microservice routing
- [https://abp.io/docs/10.3/framework/api-development/dynamic-csharp-clients](https://abp.io/docs/10.3/framework/api-development/dynamic-csharp-clients) — AddHttpClientProxies, AbpHttpClientModule, RemoteServices section in appsettings.json, AbpRemoteServiceOptions, IRemoteServiceConfigurationProvider, IHttpClientProxy<T>, named remote services, asDefaultServices flag, Polly retry via AbpHttpClientBuilderOptions.ProxyClientBuildActions
- [https://abp.io/docs/10.3/framework/api-development/static-csharp-clients](https://abp.io/docs/10.3/framework/api-development/static-csharp-clients) — AddStaticHttpClientProxies, abp generate-proxy CLI (-t csharp -u <url> [-m <module>] [--without-contracts]), generated ClientProxies/*.Generated.cs partial classes, app-generate-proxy.json embedded resource, AbpVirtualFileSystemModule dependency
- [https://abp.io/docs/10.3/framework/api-development/versioning](https://abp.io/docs/10.3/framework/api-development/versioning) — AddAbpApiVersioning, AbpApiVersioningOptions (ReportApiVersions, AssumeDefaultVersionWhenUnspecified), [ApiVersion], NoControllerFilter, query-string versioning recommendation, ICurrentApiVersionInfo.Change, ConventionalControllers.Create with ApiVersions
- [https://abp.io/docs/10.3/framework/api-development/swagger](https://abp.io/docs/10.3/framework/api-development/swagger) — Volo.Abp.Swashbuckle package, AbpSwashbuckleModule dependency, AddAbpSwaggerGen, AddAbpSwaggerGenWithOAuth, AddAbpSwaggerGenWithOidc, UseAbpSwaggerUI, AbpSwaggerOidcFlows enum, HideAbpEndpoints
- [https://abp.io/docs/10.3/framework/api-development/standard-apis](https://abp.io/docs/10.3/framework/api-development/standard-apis) — Index of pre-built ABP endpoints: Application Configuration and Application Localization
- [https://abp.io/docs/10.3/framework/api-development/standard-apis/configuration](https://abp.io/docs/10.3/framework/api-development/standard-apis/configuration) — /api/abp/application-configuration endpoint payload (auth/permissions, settings, currentUser, currentTenant, timing); IApplicationConfigurationContributor, AbpApplicationConfigurationOptions.Contributors; /Abp/ApplicationConfigurationScript
- [https://abp.io/docs/10.3/framework/api-development/standard-apis/localization](https://abp.io/docs/10.3/framework/api-development/standard-apis/localization) — /api/abp/application-localization?cultureName=<c>&onlyDynamics=<bool> endpoint; /Abp/ApplicationLocalizationScript companion script endpoint for MVC/Razor Pages

Last verified: 2026-05-10
