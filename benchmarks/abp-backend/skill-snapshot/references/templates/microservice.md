# Microservice Solution Template

> ABP Studio's premium pre-built solution template for distributed .NET systems, shipping authentication (OpenIddict + JWT), Identity/Administration microservices, YARP-based BFF gateways, RabbitMQ event bus, Redis cache/lock, Serilog+ELK logging, Prometheus+Grafana monitoring, Helm charts, and optional .NET Aspire orchestration so teams can start coding business microservices instead of building distributed-systems plumbing.

## When to load this reference

Load this reference when the user asks: 'How do I create a microservice solution with ABP?'; 'What's in the microservice startup template?'; 'How do ABP gateways/YARP routes work?'; 'How do I add a new microservice / application / gateway to my ABP solution?'; 'Where does the AuthServer live and how is OpenIddict configured?'; 'How do services talk to each other in ABP (HTTP / gRPC / distributed events)?'; 'How do I deploy ABP microservices to Kubernetes / Helm / Aspire?'; 'When should I pick the microservice template over single-layer or layered?'; 'How does ABP do per-service databases / migrations / multi-tenancy across services?'; 'How is permission/feature/localization data centralized across microservices?'; 'How do I run unit/integration tests for an ABP microservice?'; 'How do I use ABP Suite to scaffold CRUD inside a specific microservice?'.

**Audience:** Senior .NET developers, architects, and platform teams building distributed enterprise applications on ABP Business/Enterprise tier; DevOps engineers wiring CI/CD, Helm, and observability for ABP-based systems; teams migrating from a layered monolith into a service-oriented architecture; tech leads evaluating ABP templates for SaaS workloads needing tenant isolation, audit logs, and horizontal scale.

## Key concepts

- ABP Studio Module (conceptual; ABP Studio): Each leaf in the solution tree (e.g. Acme.CloudCrm.IdentityService, Acme.CloudCrm.WebGateway) is an ABP Studio module with its own independent .NET solution, enabling per-service build/run/test. Reach for it when reasoning about boundaries, code ownership, or how ABP Suite scopes its operations.
- AuthServer (project under apps/; OpenIddict + Account + Identity modules): Central single-sign-on authority that issues tokens and hosts login/registration/2FA/social UI. Reach for it whenever discussing token issuance, login flows, social providers, or password reset.
- Identity Microservice (Volo.Abp.Identity + optional Volo.Abp.OpenIddict): Owns user/role/permission data, runs OpenIddictDataSeeder to bootstrap clients and scopes, optionally hosts the OpenIddict management UI. Reach for it when adding a scope for a new service or configuring SwaggerTestUI/web-app clients.
- Administration Microservice (Volo.Abp.PermissionManagement + Volo.Abp.FeatureManagement + Volo.Abp.SettingManagement + Volo.Abp.LanguageManagement): Central authority that aggregates permission/feature/setting/language definitions across all services. Reach for it when /api/abp/application-configuration or /api/abp/application-localization behavior is in question.
- SaaS Microservice (Volo.Abp.Saas; optional, license-gated): Manages tenants, editions, and per-tenant connection strings; only present if SaaS option selected at creation. Reach for it when implementing multi-tenancy or tenant DB isolation.
- API Gateway / YARP (Microsoft.ReverseProxy via the ABP gateway template): Backend-for-Frontend gateways (WebGateway, PublicWebGateway, MobileGateway) that terminate auth, aggregate Swagger, and reverse-proxy /api/{service}/{**catch-all} into clusters. Reach for it whenever wiring a new microservice into the public surface.
- IDistributedEventBus (Volo.Abp.EventBus.Distributed; RabbitMQ implementation Volo.Abp.EventBus.RabbitMQ): Asynchronous, broker-backed event channel with outbox/inbox reliability. Reach for it for cross-service notifications that don't need a synchronous response.
- Integration Services (Volo.Abp.AspNetCore.Mvc; ExposeIntegrationServices=true): ABP's preferred synchronous channel, exposing only-internal HTTP endpoints with auto-generated dynamic clients. Reach for it whenever Service A needs to call Service B without exposing the API to public callers.
- IAbpDistributedLock (Volo.Abp.DistributedLocking; Medallion-on-Redis): Global mutex backed by Redis for serializing work across replicas. Reach for it inside background workers/jobs or migrators that must run once cluster-wide.
- IDistributedCache<TItem, TKey> (Volo.Abp.Caching; Redis backend): Typed distributed cache with prefix isolation and tenant segregation. Reach for it for read-heavy lookup data shared across services.
- AsyncPeriodicBackgroundWorkerBase (Volo.Abp.BackgroundWorkers): Base class for scheduled in-process work, registered via context.AddBackgroundWorkerAsync<T>(). Reach for it whenever a microservice needs a periodic in-process task.
- IBackgroundJobManager (Volo.Abp.BackgroundJobs.RabbitMQ by default in this template): Queue-based, durable job execution. Reach for it when work must survive restarts or be load-balanced across replicas.
- EfCoreRuntimeDatabaseMigratorBase (Volo.Abp.EntityFrameworkCore): Per-service migrator that runs at startup with retry + distributed-lock + AppliedDatabaseMigrationsEto. Reach for it when explaining auto-migration on first run or extending SeedAsync.
- ReplaceDbContext attribute (Volo.Abp.EntityFrameworkCore): Lets one service's DbContext absorb a module's tables (e.g. Identity service absorbing OpenIddict tables). Reach for it when consolidating module schemas into a single physical DB per service.
- IBlobContainer / IBlobContainer<T> (Volo.Abp.BlobStoring; Database provider by default): Pluggable BLOB API stored in a dedicated _BlobStoring database out-of-the-box. Reach for it whenever uploads/downloads cross service boundaries.
- ICurrentTenant + IDataFilter<IMultiTenant> (Volo.Abp.MultiTenancy): Ambient tenant context and a filter switch to query across tenants. Reach for it inside admin/cross-tenant operations.
- AppHost / ServiceDefaults (.NET Aspire; optional): Code-first orchestrator declaring all containers/services and a shared cloud-native config library (OpenTelemetry, /health, /alive, service discovery). Reach for it when running locally with Aspire dashboard at https://localhost:15105.

## Configuration pattern

Each microservice is its own ABP module class wired in three lifecycle hooks. Typical PreConfigureServices/ConfigureServices/OnApplicationInitializationAsync wiring:

[DependsOn(typeof(AbpAspNetCoreMvcModule), typeof(AbpAutofacModule), typeof(AbpEventBusRabbitMqModule), typeof(AbpCachingStackExchangeRedisModule), typeof(AbpDistributedLockingModule), typeof(AbpBackgroundJobsRabbitMqModule), typeof(AbpEntityFrameworkCoreSqlServerModule))]
public class ProductServiceHttpApiHostModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        var configuration = context.Services.GetConfiguration();

        // 1) Hand internal endpoints over to Integration Services
        Configure<AbpAspNetCoreMvcOptions>(options =>
        {
            options.ExposeIntegrationServices = true;
        });

        // 2) Default DBMS for all module-owned DbContexts
        Configure<AbpDbContextOptions>(options => { options.UseSqlServer(); });

        // 3) Map module schemas to physical connection strings
        Configure<AbpDbConnectionOptions>(options =>
        {
            options.Databases.Configure("Identity", db =>
            {
                db.MappedConnections.Add(AbpIdentityDbProperties.ConnectionStringName);
                db.MappedConnections.Add(AbpOpenIddictDbProperties.ConnectionStringName);
            });
        });

        // 4) JWT bearer trusts the AuthServer
        context.Services.AddAuthentication("Bearer")
            .AddJwtBearer("Bearer", options =>
            {
                options.Authority = configuration["AuthServer:Authority"]; // e.g. https://localhost:44322
                options.MetadataAddress = configuration["AuthServer:MetadataAddress"];
                options.RequireHttpsMetadata = configuration.GetValue<bool>("AuthServer:RequireHttpsMetadata");
                options.Audience = "ProductService";
            });

        // 5) Distributed cache & lock prefixes (avoid key collisions across services)
        Configure<AbpDistributedCacheOptions>(o => o.KeyPrefix = "ProductService:");
        Configure<AbpDistributedLockOptions>(o => o.KeyPrefix = "ProductService");

        // 6) Centralize Permission/Feature dynamic stores ONLY in Administration service
        Configure<PermissionManagementOptions>(o => o.SaveStaticPermissionsToDatabase = true);
        // (Administration only) o.IsDynamicPermissionStoreEnabled = true;
        Configure<FeatureManagementOptions>(o => o.SaveStaticFeaturesToDatabase = true);
        // (Administration only) o.IsDynamicFeatureStoreEnabled = true;
    }

    public override async Task OnApplicationInitializationAsync(ApplicationInitializationContext context)
    {
        var app = context.GetApplicationBuilder();
        app.UseAbpSerilogEnrichers();      // tenant/user/correlation enrichment
        app.UseHttpMetrics();              // Prometheus /metrics
        app.UseAuthentication();
        app.UseAuthorization();
        await context.AddBackgroundWorkerAsync<PassiveUserCheckerWorker>();
    }
}

Key options recap:
- AbpAspNetCoreMvcOptions.ExposeIntegrationServices: opens Integration Service endpoints (must be reachable only inside the cluster).
- AbpDbContextOptions.UseSqlServer/UsePostgreSql/...: default provider per service.
- AbpDbConnectionOptions.Databases.Configure(name, db => db.MappedConnections.Add(...)): consolidate multiple module schemas into one physical DB.
- AbpDistributedCacheOptions.KeyPrefix / AbpDistributedLockOptions.KeyPrefix: required to keep services separated on a shared Redis.
- PermissionManagementOptions.IsDynamicPermissionStoreEnabled and FeatureManagementOptions.IsDynamicFeatureStoreEnabled: enabled only in the Administration service.
- AuthServer:Authority + AuthServer:MetadataAddress in appsettings.json drive JWT validation.

## Code examples

1. Title: Publishing and handling a distributed event between two services
Scenario: ProductService notifies StockService when stock changes; StockService reacts asynchronously through RabbitMQ.
Snippet:
// Shared ETO (lives in a Contracts project both services reference)
public class StockCountChangedEto
{
public Guid ProductId { get; set; }
public int NewCount { get; set; }
}
// Publisher (ProductService)
public class ProductAppService : ApplicationService
{
private readonly IDistributedEventBus _bus;
public ProductAppService(IDistributedEventBus bus) { _bus = bus; }
public async Task ChangeStockCountAsync(Guid productId, int newCount)
{
// ... persist change ...
await _bus.PublishAsync(new StockCountChangedEto { ProductId = productId, NewCount = newCount });
}
}
// Subscriber (StockService)
public class StockChangedHandler
: IDistributedEventHandler<StockCountChangedEto>, ITransientDependency
{
public async Task HandleEventAsync(StockCountChangedEto e)
{
// idempotent reaction; outbox/inbox de-dupes redeliveries
}
}
Key lines: PublishAsync writes through the outbox table before RabbitMQ delivery; IDistributedEventHandler<TEto> auto-subscribes via DI and is wrapped in the inbox table for deduplication.
2. Title: Wiring a new YARP route + cluster on the WebGateway
Scenario: Surface a newly added ProductService at /api/product/* via the WebGateway.
Snippet (gateways/web/appsettings.json):
{
"ReverseProxy": {
"Routes": {
"Product": {
"ClusterId": "Product",
"AuthorizationPolicy": "default",
"Match": { "Path": "/api/product/{**catch-all}" }
},
"ProductSwagger": {
"ClusterId": "ProductSwagger",
"Match": { "Path": "/swagger-json/Product/swagger/v1/swagger.json" },
"Transforms": [ { "PathRemovePrefix": "/swagger-json/Product" } ]
}
},
"Clusters": {
"Product":        { "Destinations": { "d1": { "Address": "http://localhost:44361/" } } },
"ProductSwagger": { "Destinations": { "d1": { "Address": "http://localhost:44361/" } } }
}
}
}
Key lines: each microservice gets a paired API + Swagger route; PathRemovePrefix strips the gateway prefix before forwarding so the underlying Swagger JSON path stays canonical; AuthorizationPolicy="default" forces the gateway to validate the JWT before forwarding.
3. Title: Calling another microservice via Integration Services (preferred over raw HttpClient)
Scenario: ProductService asks IdentityService for a user without exposing IdentityService publicly.
Snippet (caller, ProductService appsettings.json):
{
"RemoteServices": {
"AbpIdentity": { "BaseUrl": "http://localhost:44388/" }
}
}
// Caller code
public class ProductService : ApplicationService
{
private readonly IIdentityUserIntegrationService _identityUsers; // dynamic proxy generated
public ProductService(IIdentityUserIntegrationService identityUsers) { _identityUsers = identityUsers; }
public async Task<string> WhoOwnsAsync(Guid userId)
=> (await _identityUsers.FindByIdAsync(userId))?.UserName;
}
// Callee (IdentityService module): ExposeIntegrationServices = true
Key lines: ExposeIntegrationServices flips the endpoint type so it's invisible to public clients; the dynamic C# proxy reads RemoteServices.{Module}.BaseUrl to find the callee.
4. Title: Periodic background worker guarded by a distributed lock
Scenario: A maintenance job must run on exactly one replica every 10 minutes.
Snippet:
public class PassiveUserCheckerWorker : AsyncPeriodicBackgroundWorkerBase
{
private readonly IAbpDistributedLock _lock;
public PassiveUserCheckerWorker(
AbpAsyncTimer timer,
IServiceScopeFactory scopeFactory,
IAbpDistributedLock distributedLock)
: base(timer, scopeFactory)
{
_lock = distributedLock;
Timer.Period = 600_000; // 10 minutes
}
protected override async Task DoWorkAsync(PeriodicBackgroundWorkerContext ctx)
{
await using var handle = await _lock.TryAcquireAsync("PassiveUserChecker");
if (handle == null) return; // another replica owns it this tick
// ... do the work ...
}
}
// module:
public override async Task OnApplicationInitializationAsync(ApplicationInitializationContext c)
=> await c.AddBackgroundWorkerAsync<PassiveUserCheckerWorker>();
Key lines: TryAcquireAsync returns null when another replica holds the lock, making the worker safe to deploy with replicas > 1; Timer.Period is in milliseconds.
5. Title: Adding a permission and exposing it through the gateway
Scenario: ProductService introduces "Product.Create" so admins can grant it via the Administration service UI.
Snippet:
// ProductService.Contracts/Permissions/ProductServicePermissionDefinitionProvider.cs
public class ProductServicePermissionDefinitionProvider : PermissionDefinitionProvider
{
public override void Define(IPermissionDefinitionContext context)
{
var group = context.AddGroup(ProductServicePermissions.GroupName, L("Permission:Products"));
group.AddPermission(ProductServicePermissions.Products.Default, L("Permission:Products"));
group.AddPermission(ProductServicePermissions.Products.Create,  L("Permission:Create"));
}
private static LocalizableString L(string n) => LocalizableString.Create<ProductServiceResource>(n);
}
// Endpoint usage
[Authorize(ProductServicePermissions.Products.Create)]
public Task<ProductDto> CreateAsync(ProductCreateDto input) => ...;
Key lines: defining the provider in the Contracts project lets every consumer (including the Administration UI) discover the permission; on startup the service writes definitions to the dynamic store (SaveStaticPermissionsToDatabase=true), and Administration picks them up because IsDynamicPermissionStoreEnabled=true is set only there.
## Solution structure

```
Acme.CloudCrm/                                      # solution root
├── .abpstudio/                                     # personal ABP Studio prefs (gitignored)
├── apps/                                           # end-user-facing applications (each its own .NET solution)
│   ├── auth-server/
│   │   └── src/
│   │       └── Acme.CloudCrm.AuthServer/           # OpenIddict + Account + Identity UI host (.csproj)
│   ├── web/                                        # MVC/Razor option
│   │   └── src/Acme.CloudCrm.Web/                  # ASP.NET Core MVC host
│   ├── angular/                                    # Angular SPA option (Node project, no .csproj)
│   ├── blazor/                                     # Blazor WASM or Blazor Server option
│   │   └── src/Acme.CloudCrm.Blazor/
│   ├── public-web/                                 # optional public marketing site
│   ├── maui/                                       # optional MAUI Blazor hybrid
│   │   └── src/Acme.CloudCrm.MauiBlazor/
│   └── react-native/                               # optional React Native (Expo) project
├── gateways/                                       # YARP-based BFF gateways (one per UI family)
│   ├── web/    src/Acme.CloudCrm.WebGateway/       # for MVC/Angular/Blazor admin UI
│   ├── public-web/ src/Acme.CloudCrm.PublicWebGateway/  # if public site selected
│   └── mobile/ src/Acme.CloudCrm.MobileGateway/    # if a mobile app selected
├── services/                                       # microservices (each its own .NET solution)
│   ├── identity/
│   │   └── src/
│   │       ├── Acme.CloudCrm.IdentityService/             # main host (HttpApi.Host-style, single-project)
│   │       ├── Acme.CloudCrm.IdentityService.Contracts/   # DTOs, permissions, localization, integration-service interfaces
│   │       └── Acme.CloudCrm.IdentityService.Tests/       # xUnit + Shouldly tests
│   ├── administration/
│   │   └── src/
│   │       ├── Acme.CloudCrm.AdministrationService/
│   │       ├── Acme.CloudCrm.AdministrationService.Contracts/
│   │       └── Acme.CloudCrm.AdministrationService.Tests/
│   ├── saas/                                       # only if multi-tenancy selected
│   ├── audit-logging/                              # optional
│   ├── gdpr/                                       # optional
│   ├── file-management/                            # optional
│   ├── chat/                                       # optional
│   └── product/                                    # example/business microservice
├── shared/                                         # cross-cutting projects (Hosting.Shared, ServiceDefaults for Aspire) [uncertain]
│   ├── Acme.CloudCrm.Shared.Hosting/               # shared host bootstrap helpers [uncertain]
│   └── Acme.CloudCrm.ServiceDefaults/              # Aspire shared config (OpenTelemetry, /health, /alive)
└── etc/                                            # solution-level resources
    ├── abp-studio/                                 # shared ABP Studio settings
    ├── docker/                                     # docker-compose for SQL Server, Redis, RabbitMQ, Elasticsearch, Kibana, Prometheus, Grafana
    ├── helm/                                       # Helm chart for k8s
    │   ├── Chart.yaml
    │   ├── values.yaml
    │   ├── values.acme-cloudcrm-local.yaml         # env-specific overrides
    │   ├── charts/                                 # one sub-chart per service/gateway/app/infra component
    │   ├── templates/                              # ingress hosts, secrets stubs
    │   ├── build-all-images.ps1                    # build all Docker images
    │   ├── create-tls-secrets.ps1                  # mkcert + secret creation for ingress
    │   ├── install.ps1 / uninstall.ps1             # helm install/uninstall wrappers
    └── k8s/                                        # Kubernetes Dashboard + monitoring extras

Notes per node:
- Each service folder is its own .sln so teams can open just one service in their IDE; ABP Studio glues them together at the workspace level.
- The microservice host project is single-layered intentionally — the docs explicitly call out: "Layering hasn't been applied for the pre-built microservices because they don't include much code." Only the {Service}.Contracts and {Service}.Tests companion projects exist alongside the host.
- Migrations live inside each service host project (per-service DbContext + per-service Migrations folder); the EfCoreRuntimeDatabaseMigratorBase runs them automatically on startup, so there is no separate DbMigrator console app like in the Layered template [uncertain].
- The AuthServer is in apps/, not services/, because it's an interactive web UI rather than a headless microservice.
- The shared/ folder name and exact contents may vary between template options selected at creation time; the ServiceDefaults project only appears if Aspire was enabled.
```

## When to use

Choose the microservice template when the situation matches multiple of these concrete signals:
- License: organization holds an ABP Business or higher license — the template is gated behind it.
- Team size: at least 2-3 backend teams (8+ engineers) that need to ship independently; the per-microservice .NET solution model becomes a productivity boost rather than overhead at this scale.
- Project lifetime: 3+ years with a roadmap that anticipates organizational growth (more teams, more bounded contexts).
- Deployment target: Kubernetes (Helm charts shipped) or .NET Aspire / Docker Compose; you are willing to operate Redis, RabbitMQ, Elasticsearch, Prometheus, Grafana as part of the stack.
- Scaling needs: independent horizontal scaling per service expected (e.g., a high-RPS catalog service vs a low-RPS billing service).
- Multi-tenancy at SaaS scale: separate tenant databases or strict tenant isolation are firm requirements.
- Cross-system integration: heavy event-driven flows (saga-style workflows, audit pipelines) where RabbitMQ + outbox/inbox is a natural fit.
- Observability is a first-class need: you must demo Prometheus dashboards, Kibana logs, distributed traces from day one.
- Mobile + web + public site: multiple client surfaces that benefit from per-client BFF gateways.
If only one or two of these apply, the layered template is usually a better fit and can be split into microservices later.

## When to avoid

Avoid the microservice template when any of these anti-fit signals are present — use the named alternative instead:
- Single-team CRUD product, 1-3 developers, MVP timeline: distributed plumbing (RabbitMQ, Redis, gateways, Helm) becomes pure overhead. Use the single-layer template (references/templates/single-layer.md).
- Domain not yet decomposed: if you cannot name 3+ stable bounded contexts today, you will end up with a distributed monolith. Use the layered template (references/templates/layered.md) and refactor outward later.
- Team without distributed-systems experience: eventual consistency, distributed locks, gateway routing, and OpenIddict scope wiring are non-trivial. Use the layered template until the team is ready, or hire/contract for the gap.
- No Kubernetes / no Docker tooling in production: while docker-compose works locally, fully exploiting the template (Helm, ingress, mkcert, NGINX) assumes container orchestration. Use the layered template if the deployment target is IIS / single VM.
- Hard real-time / single-process latency budget: chaining gateway -> service -> service over JWT-validated HTTP adds latency that may violate sub-millisecond budgets. Stay layered (single process) or move that hot path to gRPC inside the cluster.
- No ABP Business license: the template literally cannot be created. Fall back to the layered or single-layer templates which ship in the free tier.
- Tight regulatory monolith: when audits demand a single deployable artifact and a single DB, microservices add friction without payoff. Use the layered template.
Migration paths: single-layer -> layered -> microservice is the documented progression. There is no automated migration; expect to rebuild the host(s) and copy domain code into per-service host projects, replacing direct DbContext access with HTTP/Integration Service calls or distributed events at the new boundaries.

## Built-in features

- Authentication: OpenIddict-based AuthServer (apps/auth-server) plus JWT bearer validation in every microservice; supports MVC hybrid flow, SPA/Swagger authorization-code flow, mobile native flow, and social providers via the Account module. Default config: AuthServer:Authority + AuthServer:MetadataAddress in each appsettings.json. Deep dive: solution-templates/microservice/authentication.
- Database Configurations: per-service DbContext, ConnectionStringName, ReplaceDbContext, AbpDbConnectionOptions.MappedConnections, AbpDbContextOptions.UseSqlServer (default) or UsePostgreSql/UseMySql/UseOracle. Auto-migrations via EfCoreRuntimeDatabaseMigratorBase with retry (3 tries, 5-15s) and distributed lock. Deep dive: solution-templates/microservice/database-configurations.
- Logging: Serilog out of the box with Console + File (Logs/logs.txt in DEBUG) + Elasticsearch + Kibana + ABP Studio sinks; app.UseAbpSerilogEnrichers() injects tenant/user/correlation IDs; IApplicationInfoAccessor adds the assembly name. Deep dive: solution-templates/microservice/logging.
- Monitoring: prometheus-net.AspNetCore middleware exposes /metrics on every service; Prometheus dashboard at http://localhost:9090; Grafana at http://localhost:3001 (admin/admin); IMeterFactory for custom metrics; auto-configured for Helm when k8s is selected. Deep dive: solution-templates/microservice/monitoring.
- Swagger Integration: per-service Swagger UI plus gateway-aggregated Swagger with a service dropdown; SwaggerTestUI OAuth client auto-seeded by OpenIddictDataSeeder. Deep dive: solution-templates/microservice/swagger-integration.
- Permission Management: PermissionDefinitionProvider in each service's Contracts project; SaveStaticPermissionsToDatabase=true everywhere; IsDynamicPermissionStoreEnabled=true only in Administration; cross-service grants surfaced through /api/abp/application-configuration. Deep dive: solution-templates/microservice/permission-management.
- Feature Management: FeatureDefinitionProvider per service; SaveStaticFeaturesToDatabase=true; dynamic store enabled only in Administration; supports edition/tenant feature toggles. Deep dive: solution-templates/microservice/feature-management.
- Localization System: per-service localization resource class registered automatically on startup; Administration aggregates all resources at /api/abp/application-localization; optional Language Management module gives a UI; optional dedicated LanguageService for runtime edits. Deep dive: solution-templates/microservice/localization-system.
- Background Jobs: Volo.Abp.BackgroundJobs.RabbitMQ wired by default (RabbitMQ:Connections:Default:HostName=localhost); Hangfire/Quartz/TickerQ available as alternatives; durable across restarts. Deep dive: solution-templates/microservice/background-jobs.
- Background Workers: AsyncPeriodicBackgroundWorkerBase + AddBackgroundWorkerAsync<T>(); guarded with IAbpDistributedLock when replicas > 1. Deep dive: solution-templates/microservice/background-workers.
- Distributed Locking: Redis-backed IAbpDistributedLock; AbpDistributedLockOptions.KeyPrefix per service; Redis:Configuration in appsettings.json. Deep dive: solution-templates/microservice/distributed-locking.
- Distributed Cache: Redis-backed IDistributedCache<TItem,TKey>; AbpDistributedCacheOptions.KeyPrefix per service; Entity Cache feature for auto-invalidation. Deep dive: solution-templates/microservice/distributed-cache.
- Multi-Tenancy: optional SaaS module; per-tenant DB or shared DB strategies; ICurrentTenant ambient context; IDataFilter<IMultiTenant> escape hatch; tenant DBs auto-migrated via AppliedDatabaseMigrationsEto. Deep dive: solution-templates/microservice/multi-tenancy.
- BLOB Storing: Database provider by default with a dedicated {ProjectName}_BlobStoring database; IBlobContainer/IBlobContainer<T>; optional File Management module ships a UI. Deep dive: solution-templates/microservice/blob-storing.
- CORS Configuration: App:CorsOrigins (comma-separated) and App:RedirectAllowedUrls in AuthServer + each gateway's appsettings.json. Deep dive: solution-templates/microservice/cors-configuration.
- Inter-service communication: synchronous via Integration Services (preferred) with ExposeIntegrationServices=true and RemoteServices BaseUrl, plus dynamic/static C# proxies; asynchronous via IDistributedEventBus on RabbitMQ with outbox/inbox; gRPC NOT pre-configured (developer-managed). Deep dive: solution-templates/microservice/communication, http-api-calls, grpc-calls, distributed-events.
- Observability for Aspire (optional): ServiceDefaults adds OpenTelemetry tracing/metrics/logging, /health, /alive, service discovery, and HTTP resilience; Aspire dashboard at https://localhost:15105.

## Deployment options

- Local Development with Docker Compose (etc/docker): brings up SQL Server (default password myPassw@rd), Redis, RabbitMQ, Elasticsearch, Kibana, Prometheus, and Grafana so the host services can be debugged from Visual Studio while infra runs in containers. Best for day-to-day coding.
- ABP Studio Solution Runner: starts/stops services individually or as profiles, supports watch-changes (auto-restart on file change), launches an integrated browser, and surfaces logs in the Application Monitor. Best for inner-loop development across multiple services without manual dotnet run commands.
- Kubernetes via Helm (etc/helm): the template ships a top-level chart with sub-charts per service/gateway/app/infra component plus build-all-images.ps1, create-tls-secrets.ps1 (uses mkcert), install.ps1 / uninstall.ps1, and values.{project}-local.yaml overrides. Standard production target; ABP Studio's Kubernetes tab can build images and install/uninstall charts from the UI and supports service interception for hybrid local-debugging-against-cluster.
- .NET Aspire (optional, AppHost project): code-first orchestrator that boots all containers and services in dependency order via WaitFor(), wires connection strings/env vars, and exposes the Aspire Dashboard at https://localhost:15105 with structured logs, traces, and metrics. ServiceDefaults shared library provides OpenTelemetry, /health, /alive, and HTTP resilience to every service. Best for richer local observability.
- Cloud platforms (Azure App Service, AKS, EKS, GKE, on-prem k8s): work as long as the cluster supports NGINX Ingress and the container registry holds the images built by build-all-images.ps1. There is no built-in Azure-specific deployment helper beyond the standard Helm charts [uncertain].

## Differences from other templates

- vs Single-Layer template (references/templates/single-layer.md):
   - Project count: ~1 .csproj for single-layer vs dozens (one Acme.X.Service host + Contracts + Tests per microservice, plus AuthServer, gateways, Aspire AppHost/ServiceDefaults, etc.).
   - Layout: single-layer has src/{App}/ only; microservice has apps/, gateways/, services/, etc/, optionally shared/.
   - Complexity: single-layer = one DbContext, one process, no inter-service comms, no broker, no Helm; microservice = one DbContext per service, RabbitMQ + Redis + Elasticsearch required, OpenIddict scopes per service, YARP gateways.
   - Gained: independent scaling, independent deploys, per-service tech choice, multi-team scalability.
   - Lost: in-process speed, single-DB simplicity, single transaction guarantee, fast project setup.
   - Migration: no automated path; you re-host pieces of the single-layer app into individual services and replace direct method calls with Integration Services or distributed events.
- vs Layered (DDD) template (references/templates/layered.md):
   - Project count: layered = Domain.Shared, Domain, Application.Contracts, Application, EntityFrameworkCore, HttpApi, HttpApi.Host, Web/Blazor/etc + DbMigrator (typically 8-12 .csproj); microservice = many of these collapsed into a single per-service host project (the docs note layering is intentionally NOT applied to microservices because they should stay small).
   - Layout: layered concentrates all layers around one bounded context; microservice replicates the host pattern N times across services/.
   - Complexity: layered uses a single solution and a single DB; microservice splits into N solutions and N DBs with a YARP gateway in front.
   - Gained: independent service evolution, polyglot persistence, per-service deploy cadence.
   - Lost: cross-aggregate transactions, single-solution refactoring (Find All References across the system), fewer moving infra parts.
   - Migration: identify bounded contexts in the layered solution, move each into its own services/{name}/ host using the microservice template's per-service skeleton, replace direct repository calls with HTTP/Integration Service calls or events, and add the service to the gateway YARP config + OpenIddict scopes/clients. There is no scripted converter.
- vs Application (modular monolith) variant of the layered template [uncertain]: shares the same DDD layering inside a single host but with multiple modules; microservices template takes the next step by splitting modules into separate processes.

## Common mistakes

- - Mistake: enabling IsDynamicPermissionStoreEnabled (or IsDynamicFeatureStoreEnabled) on every microservice instead of only Administration. Why wrong: each service then tries to be the source of truth, leading to inconsistent grants and circular calls. Correct pattern: leave SaveStaticPermissionsToDatabase=true everywhere, but only set IsDynamicPermissionStoreEnabled=true / IsDynamicFeatureStoreEnabled=true on the Administration service. Fix: search ConfigureServices for those flags and remove from non-Administration services.
- - Mistake: forgetting to add the new microservice's scope in OpenIddictDataSeeder + the gateway's authentication options. Why wrong: tokens minted for the web app won't include the new audience, so the new service returns 401. Correct pattern: in OpenIddictDataSeeder call await CreateScopesAsync("ProductService"); add "ProductService" to each application's RequiredScopes; add the scope to the gateway's JwtBearer.Audience or AuthorizationPolicy. Fix: re-run the data-seeder, then re-login from the SPA/MVC app.
- - Mistake: skipping the AbpDistributedCacheOptions.KeyPrefix / AbpDistributedLockOptions.KeyPrefix per service when sharing a single Redis. Why wrong: services collide on cache keys and lock names, causing mysterious cache poisoning and missed lock acquisitions. Correct pattern: set distinct prefixes (e.g. "Identity:", "Product:"). Fix: add Configure<AbpDistributedCacheOptions>(o => o.KeyPrefix = "Product:") and the matching Lock options in each service module.
- - Mistake: exposing Integration Services without firewalling the endpoints to the cluster. Why wrong: ExposeIntegrationServices=true publishes endpoints intended for inter-service calls; if the gateway or a public ingress reaches them, internal admin operations leak. Correct pattern: keep Integration Services off the public gateway routes; restrict cluster ingress to the Web/Mobile gateway hostnames only. Docs explicitly warn: 'When you enable the Integration Services, ensure only microservices within the cluster can access the endpoints.'
- - Mistake: deploying replicas > 1 of a service that contains an unguarded AsyncPeriodicBackgroundWorkerBase. Why wrong: every replica fires the timer simultaneously, causing duplicate work (e.g., emailing users twice). Correct pattern: wrap DoWorkAsync in IAbpDistributedLock.TryAcquireAsync; bail out early if the handle is null. Fix: see the PassiveUserCheckerWorker example.
- - Mistake: assuming gRPC works out of the box. Why wrong: the docs explicitly state 'The microservice startup template hasn't any configuration related to gRPC communication.' Correct pattern: follow the .NET gRPC docs and the ABP community article 'Using gRPC with the ABP Framework' to add gRPC manually if needed; otherwise prefer Integration Services or distributed events.
- - Mistake: hand-running EF Core migrations via dotnet ef instead of letting the service migrate itself. Why wrong: the EfCoreRuntimeDatabaseMigratorBase is designed to run on startup with retries and a distributed lock, and SaaS-tenant DBs are migrated via AppliedDatabaseMigrationsEto; running migrations manually skips the event publication and leaves tenant DBs out of sync. Correct pattern: deploy the new service version and let the runtime migrator do the work. Fix: roll back manual migrations, redeploy.
- - Mistake: editing DTOs/permissions only in the host project and not in {Service}.Contracts. Why wrong: other services and the Administration UI consume the Contracts project; if it's not updated, dynamic clients won't find the new endpoints and the Administration UI won't list new permissions. Correct pattern: keep DTOs, permission definitions, integration-service interfaces, and localization resources in {Service}.Contracts.
- - Mistake: not running ABP Suite with services stopped. Why wrong: the docs warn 'ABP Suite requires you to stop all running instances in the related solution/project to be able to generate codes properly.' File locks and stale assemblies otherwise corrupt generation. Correct pattern: stop the Solution Runner, then Save and generate, then Build & Restart.
- - Mistake: ignoring CORS on the AuthServer when adding a new gateway or app. Why wrong: the browser-side OAuth handshake fails because the AuthServer rejects the unknown origin / redirect URI. Correct pattern: add the new origin to App:CorsOrigins and App:RedirectAllowedUrls on the AuthServer, and update OpenIddictDataSeeder to add the redirect URI to the relevant client.
## Version pins (ABP 10.3)

> **Note:** items marked `[uncertain]` need verification against ABP 10.3 source. Treat as best-effort guidance.

- ABP 10.3 default DBMS in the template is SQL Server (AbpDbContextOptions.UseSqlServer); switching to PostgreSQL / MySQL / Oracle requires changing both the AbpDbContextOptions call and the AbpEntityFrameworkCore* module reference. Future versions may shift the default — verify before assuming.
- The Get-Started page lists prerequisites pinned to .NET 10.0+, Node v22.11+, Yarn v1.22+ (or npm v10+), Visual Studio 2026 — these will rotate with each ABP release; treat them as 10.3-specific.
- Background jobs default to RabbitMQ via Volo.Abp.BackgroundJobs.RabbitMQ; the docs name TickerQ as a supported alternative which is a relatively recent addition — confirm it is still listed in your ABP version. [uncertain]
- The Administration service's IsDynamicPermissionStoreEnabled / IsDynamicFeatureStoreEnabled flags are the documented switch in 10.3; older docs used different option names — do not copy snippets from pre-10.x posts without verifying. [uncertain]
- AbpDbConnectionOptions.Databases.Configure(name, db => db.MappedConnections.Add(...)) is the 10.3 idiom; earlier templates used a different consolidation pattern (direct ReplaceDbContext-only). Both still work but the new API is preferred. [uncertain]
- Aspire integration is opt-in at solution-creation time; if Aspire was not selected, AppHost and ServiceDefaults projects do not exist in the solution and adding them later requires manual scaffolding.
- gRPC is explicitly NOT preconfigured in the template; this has been true across recent versions and is unlikely to change soon, but check the page before claiming the template ships gRPC.
- The OpenIddict UI module appears only in the Identity microservice and only when selected; older docs may show it elsewhere.
- Default service ports shown in docs (e.g. 44388 for Identity, 44322 for AuthServer, 44361 for ProductService, 9090 Prometheus, 3001 Grafana, 15105 Aspire) are illustrative; the wizard assigns them at creation and your local solution may differ. Always read launchSettings.json / appsettings.json.
- Default password 'myPassw@rd' for SQL Server in docker-compose and 'admin / 1q2w3E*' for the seeded admin user are development-only — rotate before any non-local deployment.

## Cross-references

**Phase 1 references:**
- - references/templates/single-layer.md: trigger when the user asks 'do I really need microservices?' or 'should I start simpler?' — point them at single-layer for MVPs.
- - references/templates/layered.md: trigger when migration is being discussed ('how do I move from a monolith to microservices?') and when the user wants per-bounded-context layering inside a single host before going distributed.
- - references/framework/architecture-ddd.md: trigger whenever the user asks about bounded contexts, aggregates, domain events vs distributed events, or how DDD maps onto a microservice host (each microservice is essentially a single bounded context plus its API surface).
- - references/framework/authentication-openiddict.md [uncertain]: trigger on any OpenIddict configuration question (clients, scopes, hybrid vs code flow) — the microservice template adds scope-per-service nuances on top of the framework basics.
- - references/framework/distributed-events.md [uncertain]: trigger on any IDistributedEventBus / RabbitMQ / outbox-inbox question — the microservice template is the canonical user of these primitives.
- - references/framework/multi-tenancy.md [uncertain]: trigger on tenant resolution, ICurrentTenant, or per-tenant DB questions — microservice + SaaS module is the multi-tenant flagship.
- - references/framework/background-jobs-workers.md [uncertain]: trigger on Hangfire/Quartz/RabbitMQ/TickerQ choice and AsyncPeriodicBackgroundWorkerBase questions.
- - references/framework/permissions-features-settings.md [uncertain]: trigger on PermissionDefinitionProvider/FeatureDefinitionProvider and how the Administration service centralizes them.
- - references/framework/blob-storing.md [uncertain]: trigger on file uploads, IBlobContainer providers, or moving from the default Database BLOB provider to S3/Azure/MinIO.
- - references/framework/abp-suite.md [uncertain]: trigger on CRUD scaffolding questions — Suite has microservice-specific behavior (must stop running services, abp generate-proxy command).

**External docs:**
- - Microsoft YARP (https://microsoft.github.io/reverse-proxy/): referenced because every API gateway uses YARP for ReverseProxy routes/clusters/transforms; consult for advanced transform syntax, load balancing, and session affinity.
- - OpenIddict (https://documentation.openiddict.com/): referenced because the AuthServer is built on OpenIddict; consult for client/scope models, grant-type details, and migration notes between OpenIddict majors.
- - Microsoft ASP.NET Core Authentication (https://learn.microsoft.com/aspnet/core/security/authentication/): referenced for JwtBearer options used in each microservice (Authority, Audience, MetadataAddress).
- - Entity Framework Core (https://learn.microsoft.com/ef/core/): referenced for DbContext, migrations, and provider packages (SQL Server / PostgreSQL / MySQL / Oracle) plumbed via AbpDbContextOptions.
- - Serilog (https://serilog.net/) + Serilog.Sinks.Elasticsearch / Serilog.Sinks.Console / Serilog.Sinks.File: referenced as the logging pipeline used out of the box.
- - Elasticsearch + Kibana (https://www.elastic.co/): referenced as the default centralized log store/visualizer.
- - Prometheus (https://prometheus.io/) + Grafana (https://grafana.com/): referenced for metrics scraping and dashboards (defaults: Prom :9090, Grafana :3001 admin/admin).
- - prometheus-net (https://github.com/prometheus-net/prometheus-net): referenced because the template uses prometheus-net.AspNetCore for /metrics middleware.
- - RabbitMQ (https://www.rabbitmq.com/): referenced as the default broker for both background jobs and distributed events.
- - Redis (https://redis.io/): referenced as the default backend for distributed cache and distributed locks.
- - .NET Aspire (https://learn.microsoft.com/dotnet/aspire/): referenced for the optional AppHost/ServiceDefaults orchestration, dashboard at https://localhost:15105.
- - Helm (https://helm.sh/) + NGINX Ingress Controller (https://kubernetes.github.io/ingress-nginx/) + mkcert (https://github.com/FiloSottile/mkcert): referenced for the etc/helm deployment scripts.
- - Microsoft .NET Aspire Diagnostics (https://learn.microsoft.com/dotnet/core/diagnostics/metrics): referenced as the canonical guide for the IMeterFactory custom-metrics example.
- - Microsoft gRPC on .NET (https://learn.microsoft.com/aspnet/core/grpc/): referenced because the microservice template does NOT pre-configure gRPC; developers follow the .NET gRPC docs to add it manually.

## Sources (ABP 10.3)

- [https://abp.io/docs/10.3/solution-templates/microservice/overview](https://abp.io/docs/10.3/solution-templates/microservice/overview) — High-level overview, license requirements, target audience, deployment options
- [https://abp.io/docs/10.3/solution-templates/microservice/solution-structure](https://abp.io/docs/10.3/solution-templates/microservice/solution-structure) — Folder layout (apps/gateways/services/etc), ABP Studio module concept, per-microservice .NET solutions
- [https://abp.io/docs/10.3/solution-templates/microservice/main-components](https://abp.io/docs/10.3/solution-templates/microservice/main-components) — Index page linking to microservices, gateways, web apps, and mobile apps subsections
- [https://abp.io/docs/10.3/solution-templates/microservice/microservices](https://abp.io/docs/10.3/solution-templates/microservice/microservices) — Identity, Administration, optional SaaS/Audit Logging/GDPR/File Management/Chat services and their three-project structure (main/contracts/tests)
- [https://abp.io/docs/10.3/solution-templates/microservice/api-gateways](https://abp.io/docs/10.3/solution-templates/microservice/api-gateways) — YARP-based BFF gateways, ReverseProxy config in appsettings.json, SwaggerTestUI client, IProxyConfigFilter
- [https://abp.io/docs/10.3/solution-templates/microservice/web-applications](https://abp.io/docs/10.3/solution-templates/microservice/web-applications) — AuthServer plus MVC/Angular/Blazor WASM/Blazor Server/MAUI Blazor app variants and where they live under apps/
- [https://abp.io/docs/10.3/solution-templates/microservice/mobile-applications](https://abp.io/docs/10.3/solution-templates/microservice/mobile-applications) — MAUI and React Native options, MobileGateway, OIDC flow, JWT bearer to gateway
- [https://abp.io/docs/10.3/solution-templates/microservice/built-in-features](https://abp.io/docs/10.3/solution-templates/microservice/built-in-features) — Index of pre-wired features (auth, logging, monitoring, swagger, BG jobs/workers, locking, cache, multi-tenancy, blob, CORS)
- [https://abp.io/docs/10.3/solution-templates/microservice/authentication](https://abp.io/docs/10.3/solution-templates/microservice/authentication) — OpenIddict-based AuthServer, hybrid flow for MVC, code flow for SPA/Swagger, JWT bearer between services, OpenIddictDataSeeder
- [https://abp.io/docs/10.3/solution-templates/microservice/database-configurations](https://abp.io/docs/10.3/solution-templates/microservice/database-configurations) — Per-service DbContext, ConnectionStringName, ReplaceDbContext, AbpDbConnectionOptions, AbpDbContextOptions, EfCoreRuntimeDatabaseMigratorBase auto-migrate with retry+distributed lock
- [https://abp.io/docs/10.3/solution-templates/microservice/logging](https://abp.io/docs/10.3/solution-templates/microservice/logging) — Serilog with Console/File/Elasticsearch/Kibana/ABP Studio sinks, IApplicationInfoAccessor enrichment, app.UseAbpSerilogEnrichers()
- [https://abp.io/docs/10.3/solution-templates/microservice/monitoring](https://abp.io/docs/10.3/solution-templates/microservice/monitoring) — prometheus-net.AspNetCore middleware, /metrics endpoint, Grafana at :3001 admin/admin, IMeterFactory custom metrics, K8s-aware
- [https://abp.io/docs/10.3/solution-templates/microservice/swagger-integration](https://abp.io/docs/10.3/solution-templates/microservice/swagger-integration) — Per-service Swagger aggregated by gateway, SwaggerTestUI OAuth client, dropdown UI to switch services
- [https://abp.io/docs/10.3/solution-templates/microservice/permission-management](https://abp.io/docs/10.3/solution-templates/microservice/permission-management) — Administration microservice as central authority, PermissionDefinitionProvider in Contracts project, SaveStaticPermissionsToDatabase, IsDynamicPermissionStoreEnabled
- [https://abp.io/docs/10.3/solution-templates/microservice/feature-management](https://abp.io/docs/10.3/solution-templates/microservice/feature-management) — FeatureDefinitionProvider, SaveStaticFeaturesToDatabase, IsDynamicFeatureStoreEnabled in Administration only
- [https://abp.io/docs/10.3/solution-templates/microservice/localization-system](https://abp.io/docs/10.3/solution-templates/microservice/localization-system) — Per-service resource classes, /api/abp/application-localization aggregated through Administration, optional Language Management module
- [https://abp.io/docs/10.3/solution-templates/microservice/background-jobs](https://abp.io/docs/10.3/solution-templates/microservice/background-jobs) — RabbitMQ as default broker (Volo.Abp.BackgroundJobs.RabbitMQ), alternative providers Hangfire/Quartz/TickerQ
- [https://abp.io/docs/10.3/solution-templates/microservice/background-workers](https://abp.io/docs/10.3/solution-templates/microservice/background-workers) — AsyncPeriodicBackgroundWorkerBase, AddBackgroundWorkerAsync registration, distributed-lock guidance for multi-instance
- [https://abp.io/docs/10.3/solution-templates/microservice/distributed-locking](https://abp.io/docs/10.3/solution-templates/microservice/distributed-locking) — Redis-backed IAbpDistributedLock, AbpDistributedLockOptions.KeyPrefix, TryAcquireAsync pattern
- [https://abp.io/docs/10.3/solution-templates/microservice/distributed-cache](https://abp.io/docs/10.3/solution-templates/microservice/distributed-cache) — Redis Configuration, AbpDistributedCache.KeyPrefix, IDistributedCache<TItem,TKey> with GetOrAddAsync, Entity Cache feature
- [https://abp.io/docs/10.3/solution-templates/microservice/multi-tenancy](https://abp.io/docs/10.3/solution-templates/microservice/multi-tenancy) — SaaS module (license-gated), separate vs shared tenant DB strategy, ICurrentTenant, IDataFilter<IMultiTenant>, Connection Strings Management Modal
- [https://abp.io/docs/10.3/solution-templates/microservice/blob-storing](https://abp.io/docs/10.3/solution-templates/microservice/blob-storing) — Database BLOB provider by default, separate MyProjectName_BlobStoring DB, IBlobContainer, optional File Management module
- [https://abp.io/docs/10.3/solution-templates/microservice/cors-configuration](https://abp.io/docs/10.3/solution-templates/microservice/cors-configuration) — App:CorsOrigins comma-separated list in appsettings.json on AuthServer and gateways
- [https://abp.io/docs/10.3/solution-templates/microservice/communication](https://abp.io/docs/10.3/solution-templates/microservice/communication) — Three patterns: HTTP API calls, gRPC calls, distributed events (sync vs async guidance)
- [https://abp.io/docs/10.3/solution-templates/microservice/http-api-calls](https://abp.io/docs/10.3/solution-templates/microservice/http-api-calls) — Integration Services as preferred channel, ExposeIntegrationServices flag, RemoteServices BaseUrl, static/dynamic C# clients
- [https://abp.io/docs/10.3/solution-templates/microservice/grpc-calls](https://abp.io/docs/10.3/solution-templates/microservice/grpc-calls) — Not pre-configured in template; requires manual setup using .NET gRPC and community guide
- [https://abp.io/docs/10.3/solution-templates/microservice/distributed-events](https://abp.io/docs/10.3/solution-templates/microservice/distributed-events) — Volo.Abp.EventBus.RabbitMQ, IDistributedEventBus.PublishAsync, IDistributedEventHandler<TEto>, outbox/inbox pattern
- [https://abp.io/docs/10.3/solution-templates/microservice/helm-charts-and-kubernetes](https://abp.io/docs/10.3/solution-templates/microservice/helm-charts-and-kubernetes) — etc/helm chart with sub-charts per component, build-all-images.ps1, create-tls-secrets.ps1, install.ps1
- [https://abp.io/docs/10.3/solution-templates/microservice/aspire-integration](https://abp.io/docs/10.3/solution-templates/microservice/aspire-integration) — AppHost orchestrator, ServiceDefaults shared lib (OpenTelemetry, /health, /alive, service discovery), Aspire dashboard at https://localhost:15105
- [https://abp.io/docs/10.3/solution-templates/microservice/guides](https://abp.io/docs/10.3/solution-templates/microservice/guides) — Guide index (add microservice/app/gateway, tests, mono- vs multi-repo)
- [https://abp.io/docs/10.3/solution-templates/microservice/adding-new-microservices](https://abp.io/docs/10.3/solution-templates/microservice/adding-new-microservices) — ABP Studio Add > New Module > Microservice; update gateway YARP, OpenIddictDataSeeder scopes, AuthServer CORS, Prometheus, Helm
- [https://abp.io/docs/10.3/solution-templates/microservice/adding-new-applications](https://abp.io/docs/10.3/solution-templates/microservice/adding-new-applications) — Add > New Module > Web; configure Authority/ClientId/ClientSecret, register OpenIddict client, Prometheus and Helm updates
- [https://abp.io/docs/10.3/solution-templates/microservice/adding-new-api-gateways](https://abp.io/docs/10.3/solution-templates/microservice/adding-new-api-gateways) — Add > New Module > Gateway; YARP routes/clusters/transforms, OpenIddict redirect URIs, AuthServer CORS
- [https://abp.io/docs/10.3/solution-templates/microservice/mono-repo-vs-multiple-repository-approaches](https://abp.io/docs/10.3/solution-templates/microservice/mono-repo-vs-multiple-repository-approaches) — Trade-off analysis: shared CI/CD vs independent teams; centralized deps vs fine-grained access
- [https://abp.io/docs/10.3/solution-templates/microservice/authoring-unit-and-integration-tests](https://abp.io/docs/10.3/solution-templates/microservice/authoring-unit-and-integration-tests) — xUnit + Shouldly, MicroServiceNameTestsModule + TestBase, AdditionalAssembly pattern to swap in-memory infra
- [https://abp.io/docs/10.3/solution-templates/microservice/how-to-use-with-abp-suite](https://abp.io/docs/10.3/solution-templates/microservice/how-to-use-with-abp-suite) — ABP Suite at http://localhost:3000, Save and generate, abp generate-proxy command, must stop running instances
- [https://abp.io/docs/10.3/get-started/microservice](https://abp.io/docs/10.3/get-started/microservice) — Prerequisites (.NET 10, Node 22.11, Yarn 1.22, Docker Desktop, Helm, NGINX Ingress, mkcert), wizard steps, Solution Runner, default admin/1q2w3E*

Last verified: 2026-05-10
