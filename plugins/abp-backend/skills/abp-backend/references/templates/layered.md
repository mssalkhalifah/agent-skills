# Layered Solution Template

> The Layered Solution Template generates a monolithic, DDD-structured ABP solution split into Domain/Application/Infrastructure/Presentation projects, with pre-wired authentication, multi-tenancy, BLOB storage, background jobs/workers, distributed locking, logging, Swagger, and Docker/Kubernetes/IIS/Azure deployment paths.

## When to load this reference

- User asks 'what projects should a Layered ABP solution have?' or 'what does .Domain vs .Application vs .Application.Contracts contain?'
- User asks how to add a new entity, repository, application service, or controller to a Layered template solution.
- User asks about the .DbMigrator console app — when to run it, how data seeding works, how to use it in CI/CD or Docker Compose.
- User asks about the tiered (separate AuthServer + HttpApi.Host) variant and when it is needed.
- User asks how to configure OpenIddict, certificates, CORS, RedirectAllowedUrls in a Layered solution for production.
- User asks how to deploy a Layered ABP solution to Docker Compose, Kubernetes/Helm, Azure App Service, or IIS.
- User asks how multi-tenancy is wired in a Layered solution (per-tenant DB vs shared DB, ICurrentTenant, SaaS module).
- User asks how to compare the Layered template against Single-Layer or Microservice templates and pick one.
- User asks where to put EF Core migrations, DbContext, and connection strings in a Layered solution.

**Audience:** Teams building monolithic, production-grade ABP web applications that want DDD-aligned project layering, optional multi-tenancy, and standard enterprise infrastructure (auth server, BG jobs, BLOB store) without the operational overhead of microservices.

## Key concepts

- Domain.Shared project — cross-layer constants/enums/localization; namespace `<Project>.<Project>DomainSharedModule`. Reach for it when a value (enum, error code, resource key) must be referenced from BOTH the Domain layer and the Application.Contracts/HttpApi.Client layers without leaking domain types.
- Domain project — entities, aggregate roots, value objects, domain services, repository interfaces; namespace `<Project>.<Project>DomainModule`. Reach for it when expressing business invariants that should not depend on EF Core, DTOs, or HTTP concerns.
- Application.Contracts project — application service interfaces, DTOs, permission definition providers; namespace `<Project>.<Project>ApplicationContractsModule`. Reach for it when defining the public surface of the application that HttpApi.Client/Web layers will consume.
- Application project — application service implementations, AutoMapper profiles, authorization-aware orchestration of repositories and domain services; namespace `<Project>.<Project>ApplicationModule`. Reach for it to implement use cases that coordinate domain logic and persistence.
- EntityFrameworkCore project — `AbpDbContext`-derived DbContext, EF Core entity configurations, repository implementations, EF Core migrations, `IDesignTimeDbContextFactory<TDbContext>`, `EntityFrameworkCore<Project>DbSchemaMigrator`; namespace `<Project>.EntityFrameworkCore`. Reach for it for anything persistence-specific.
- HttpApi project — auto-API or hand-written ASP.NET Core controllers that thinly delegate to application services; namespace `<Project>.<Project>HttpApiModule`. Reach for it when you need a custom controller or a non-default route shape.
- HttpApi.Client project — typed C# proxies (Volo.Abp.Http.Client) that other .NET clients (e.g., the MAUI app, console tests) consume; namespace `<Project>.<Project>HttpApiClientModule`.
- Web project (and AuthServer/HttpApi.Host in tiered) — ASP.NET Core host that wires the module dependency graph in Program.cs and invokes `AbpApplicationFactory`/`UseAbpApplication`. In non-tiered solutions Web also serves OpenIddict; in tiered the AuthServer project does it.
- DbMigrator console — `DbMigratorHostedService` runs `<Project>DbMigrationService`, applies pending EF Core migrations, and invokes `IDataSeedContributor` implementations including `OpenIddictDataSeedContributor` (and `IdentityServerDataSeedContributor` if using IdentityServer); namespace `<Project>.DbMigrator`.
- Module class pattern — every project has a `[DependsOn(...)]`-decorated class inheriting `AbpModule` (e.g., `<Project>WebModule : AbpModule`). The PreConfigureServices/ConfigureServices/OnApplicationInitialization lifecycle is the mandatory wiring point for every framework option in the layered template.
- ICurrentTenant — `Volo.Abp.MultiTenancy.ICurrentTenant`. Inject to read the active tenant id/name; use `using (CurrentTenant.Change(tenantId)) { ... }` to switch tenant context (e.g., from a background worker).
- IBlobContainer / IBlobContainer<TContainer> — `Volo.Abp.BlobStoring.IBlobContainer`. The default Layered template uses the Database BLOB Provider; SaveAsync/GetAllBytesOrNullAsync/DeleteAsync are the canonical entry points.
- IAbpDistributedLock — `Volo.Abp.DistributedLocking.IAbpDistributedLock`. Backed by Redis when `Volo.Abp.DistributedLocking` is added; use `await _lock.TryAcquireAsync('name')` and dispose the handle to release.
- AsyncPeriodicBackgroundWorkerBase — `Volo.Abp.BackgroundWorkers.AsyncPeriodicBackgroundWorkerBase`. Base class for periodic workers; register with `context.AddBackgroundWorkerAsync<T>()` in `OnApplicationInitializationAsync`.

## Configuration pattern

Configuration in a Layered solution is split across module classes, with each project owning the wiring for its layer. The four most load-bearing patterns:

1) DB provider (in `<Project>EntityFrameworkCoreModule.ConfigureServices`):
```csharp
public override void ConfigureServices(ServiceConfigurationContext context)
{
    context.Services.AddAbpDbContext<BookStoreDbContext>(options =>
    {
        options.AddDefaultRepositories(includeAllEntities: true);
    });

    Configure<AbpDbContextOptions>(options =>
    {
        options.UseSqlServer(); // or UseNpgsql/UseMySQL/UseSqlite
    });
}
```

2) Connection string + ReplaceDbContext (on the DbContext itself):
```csharp
[ConnectionStringName('Default')]
[ReplaceDbContext(typeof(IIdentityProDbContext))]
[ReplaceDbContext(typeof(ISaasDbContext))]
public class BookStoreDbContext : AbpDbContext<BookStoreDbContext>, IIdentityProDbContext, ISaasDbContext { ... }
```
The `ConnectionStringName` matches an entry under `ConnectionStrings` in `appsettings.json` (default: `Default`). `ReplaceDbContext` collapses module DbContexts into the host DB so a single migration set covers Identity/SaaS.

3) Serilog (in `Program.cs` of Web/AuthServer/DbMigrator):
```csharp
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .Enrich.FromLogContext()
    .WriteTo.Async(c => c.Console())
#if DEBUG
    .WriteTo.Async(c => c.File('Logs/logs.txt'))
#endif
    .WriteTo.Async(c => c.AbpStudio(services))
    .CreateLogger();
```
And in the module's `OnApplicationInitialization`:
```csharp
app.UseAbpSerilogEnrichers(); // adds tenant/user/client/correlation
```

4) Background worker registration (in `OnApplicationInitializationAsync` of any module):
```csharp
public override async Task OnApplicationInitializationAsync(ApplicationInitializationContext context)
{
    await context.AddBackgroundWorkerAsync<PassiveUserCheckerWorker>();
}
```
Options read from `appsettings.json` (CorsOrigins, ConnectionStrings, App:SelfUrl, AuthServer:Authority, OpenIddict:Applications:*) are bound via `Configure<AppOptions>(configuration.GetSection('App'))`-style calls in the host's module.

## Code examples

### Define a DbContext in the EntityFrameworkCore project

_Wire a new aggregate root into the host DbContext with a ReplaceDbContext-aware configuration._

```csharp
// src/Acme.BookStore.EntityFrameworkCore/EntityFrameworkCore/BookStoreDbContext.cs
using Microsoft.EntityFrameworkCore;
using Volo.Abp.Data;
using Volo.Abp.EntityFrameworkCore;
using Volo.Abp.EntityFrameworkCore.Modeling;
using Volo.Abp.Identity.EntityFrameworkCore;
using Volo.Abp.OpenIddict.EntityFrameworkCore;

[ConnectionStringName('Default')]
[ReplaceDbContext(typeof(IIdentityDbContext))]
[ReplaceDbContext(typeof(IOpenIddictDbContext))]
public class BookStoreDbContext :
    AbpDbContext<BookStoreDbContext>,
    IIdentityDbContext,
    IOpenIddictDbContext
{
    public DbSet<Book> Books { get; set; }

    public BookStoreDbContext(DbContextOptions<BookStoreDbContext> options)
        : base(options) { }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);
        builder.ConfigureIdentity();
        builder.ConfigureOpenIddict();
        builder.Entity<Book>(b =>
        {
            b.ToTable('AppBooks');
            b.ConfigureByConvention();
            b.Property(x => x.Name).HasMaxLength(128).IsRequired();
        });
    }
}
```

**Key lines:** `[ConnectionStringName]` ties the context to `ConnectionStrings:Default`. `[ReplaceDbContext]` lets module migrations (Identity, OpenIddict) live inside the host DB. `ConfigureByConvention()` applies ABP's audit/multi-tenancy/soft-delete conventions automatically.

### Run the DbMigrator from CLI / Docker Compose

_Apply pending migrations and seed OpenIddict clients before the Web app starts._

```bash
# Local
cd src/Acme.BookStore.DbMigrator
dotnet run --configuration Release

# In docker-compose.yml
db-migrator:
  image: acme/bookstore-db-migrator:latest
  build:
    context: ../..
    dockerfile: src/Acme.BookStore.DbMigrator/Dockerfile.local
  environment:
    - ConnectionStrings__Default=Server=sqlserver;Database=BookStore;User Id=sa;Password=${SA_PASSWORD};TrustServerCertificate=True
    - OpenIddict__Applications__BookStore_Web__RootUrl=https://bookstore-web
  depends_on:
    sqlserver:
      condition: service_healthy
```

**Key lines:** DbMigrator owns its own appsettings.json, so its connection string and OpenIddict RootUrl values must match the production targets — the Web app will not re-seed OpenIddict applications, only DbMigrator does.

### Periodic background worker with distributed-lock guard

_Run a recurring task safely across multiple Web instances behind a load balancer._

```csharp
public class PassiveUserCheckerWorker : AsyncPeriodicBackgroundWorkerBase
{
    private readonly IAbpDistributedLock _lock;

    public PassiveUserCheckerWorker(
        AbpAsyncTimer timer,
        IServiceScopeFactory serviceScopeFactory,
        IAbpDistributedLock distributedLock)
        : base(timer, serviceScopeFactory)
    {
        Timer.Period = 600_000; // 10 minutes
        _lock = distributedLock;
    }

    protected override async Task DoWorkAsync(PeriodicBackgroundWorkerContext ctx)
    {
        await using var handle = await _lock.TryAcquireAsync('PassiveUserChecker');
        if (handle == null) return; // another instance is running

        var users = ctx.ServiceProvider.GetRequiredService<IIdentityUserRepository>();
        // ... do work ...
    }
}

// In <Project>DomainModule.OnApplicationInitializationAsync:
await context.AddBackgroundWorkerAsync<PassiveUserCheckerWorker>();
```

**Key lines:** Without the lock, an N-replica deployment will run the worker N times per period. `TryAcquireAsync` returns null when another holder owns the lock, so the worker silently skips that tick.

### appsettings.json for production CORS + OpenIddict + connection

_Promote a Layered solution from localhost dev to a real domain._

```json
{
  "App": {
    "SelfUrl": "https://bookstore.acme.com",
    "CorsOrigins": "https://*.acme.com",
    "RedirectAllowedUrls": "https://bookstore.acme.com,https://admin.acme.com"
  },
  "AuthServer": {
    "Authority": "https://auth.acme.com",
    "RequireHttpsMetadata": "true"
  },
  "ConnectionStrings": {
    "Default": "Server=tcp:bookstore.database.windows.net,1433;Database=BookStore;User ID=...;Password=...;Encrypt=True"
  },
  "OpenIddict": {
    "Applications": {
      "BookStore_Web": {
        "ClientId": "BookStore_Web",
        "RootUrl": "https://bookstore.acme.com"
      }
    }
  }
}
```

**Key lines:** Both DbMigrator and Web/AuthServer must see the same values — DbMigrator seeds OpenIddict from these, Web/AuthServer validates against them at runtime. Mismatch produces 'invalid_client' or redirect-uri errors.

### Helm chart values override for Kubernetes

_Deploy the Layered solution to a Kubernetes cluster (Business+ license)._

```bash
# etc/helm/bookstore/values.bookstore-local.yaml
bookstore-web:
  image:
    repository: acme/bookstore-web
    tag: latest
  ingress:
    enabled: true
    host: bookstore.local
    tls:
      secretName: bookstore-tls
bookstore-db-migrator:
  image:
    repository: acme/bookstore-db-migrator
    tag: latest
# Then:
# ./build-all-images.ps1
# ./create-tls-secrets.ps1
# ./install.ps1
```

**Key lines:** The chart in `etc/helm/<project>` ships sub-charts per app and infra component (Redis, RabbitMQ, SQL). `install.ps1` runs `helm install` with the values override; `create-tls-secrets.ps1` provisions self-signed certs for local clusters.

## Solution structure

```
<solution-root>/
├── .abpstudio/                                  # personal preferences (git-ignored)
├── etc/
│   ├── abp-studio/                              # solution-shared ABP Studio config (committed)
│   ├── docker/                                  # Dockerfiles per app
│   ├── docker-compose/                          # docker-compose.yml + run-docker.ps1 / .sh
│   └── helm/<project>/                          # Helm chart (Business+ license)
│       ├── Chart.yaml
│       ├── values.yaml
│       ├── values.<project>-local.yaml
│       ├── charts/                              # sub-charts per app + Redis/RabbitMQ
│       └── templates/                           # ingress, configmaps, secrets
├── src/                                         # all production code
│   ├── Acme.BookStore.Domain.Shared/            # constants, enums, localization keys
│   │   └── BookStoreDomainSharedModule.cs
│   ├── Acme.BookStore.Domain/                   # entities, aggregates, domain services, repo interfaces
│   │   ├── Data/IBookStoreDbSchemaMigrator.cs   # provider-agnostic migration contract
│   │   └── BookStoreDomainModule.cs
│   ├── Acme.BookStore.Application.Contracts/    # service interfaces, DTOs, permission definitions
│   │   └── BookStoreApplicationContractsModule.cs
│   ├── Acme.BookStore.Application/              # service implementations, AutoMapper profiles
│   │   └── BookStoreApplicationModule.cs
│   ├── Acme.BookStore.EntityFrameworkCore/      # DbContext, EF migrations, repo implementations
│   │   ├── EntityFrameworkCore/BookStoreDbContext.cs
│   │   ├── EntityFrameworkCore/BookStoreDbContextFactory.cs   # IDesignTimeDbContextFactory
│   │   ├── EntityFrameworkCore/EntityFrameworkCoreBookStoreDbSchemaMigrator.cs
│   │   ├── Migrations/                          # dotnet ef output
│   │   └── BookStoreEntityFrameworkCoreModule.cs
│   ├── Acme.BookStore.HttpApi/                  # ASP.NET Core controllers (or auto-API)
│   │   └── BookStoreHttpApiModule.cs
│   ├── Acme.BookStore.HttpApi.Client/           # typed C# client proxies
│   │   └── BookStoreHttpApiClientModule.cs
│   ├── Acme.BookStore.HttpApi.Client.ConsoleTestApp/  # demo console using HttpApi.Client
│   ├── Acme.BookStore.Web/                      # ASP.NET Core MVC/Razor host (or Blazor Server host)
│   │   ├── Program.cs                           # Serilog bootstrap, AbpApplicationFactory
│   │   ├── appsettings.json                     # CorsOrigins, ConnectionStrings, OpenIddict, AuthServer
│   │   └── BookStoreWebModule.cs
│   ├── Acme.BookStore.AuthServer/               # ONLY in tiered: separate OpenIddict host
│   │   ├── Program.cs
│   │   ├── appsettings.json
│   │   └── BookStoreAuthServerModule.cs
│   ├── Acme.BookStore.HttpApi.Host/             # ONLY in tiered: separate API host
│   │   └── BookStoreHttpApiHostModule.cs
│   ├── Acme.BookStore.Blazor (or .Blazor.Server / .Blazor.WebApp)/  # if Blazor UI selected
│   ├── Acme.BookStore.Maui/ or Acme.BookStore.ReactNative/          # optional mobile
│   └── Acme.BookStore.DbMigrator/               # console app: DbMigratorHostedService + data seeders
│       ├── Program.cs
│       └── appsettings.json                     # MUST mirror Web's values for OpenIddict + ConnectionStrings
└── test/
    ├── Acme.BookStore.TestBase/                 # shared in-memory test infrastructure
    ├── Acme.BookStore.Domain.Tests/
    ├── Acme.BookStore.Application.Tests/
    ├── Acme.BookStore.EntityFrameworkCore.Tests/
    └── Acme.BookStore.Web.Tests/

```

## When to use

Pick the Layered template when ALL of the following are true: (1) team has 3+ developers and expects the codebase to live 2+ years, (2) the system has identifiable business subdomains that benefit from explicit Domain/Application/Contracts separation (DDD aggregates, domain services, value objects), (3) you want pre-wired enterprise features (OpenIddict auth server, Identity, multi-tenancy, BLOB store, BG jobs, distributed locking) without assembling them yourself, (4) deployment is monolithic — a single Web host (or tiered = Web + AuthServer + HttpApi.Host) rather than dozens of services, (5) you may need to introduce a separate auth server (tiered switch) or microservices later and want a project layout that already isolates HttpApi/Application.Contracts so that move is feasible. Concrete green-light signals: planned multi-tenancy, planned mobile clients (MAUI/React Native that consume HttpApi.Client), planned Kubernetes/Helm deployment, regulatory/audit requirements that need clean layer boundaries.

## When to avoid

Avoid the Layered template when: (1) the project is a small internal tool, MVP, or single-developer side project where the 8-15 csproj overhead and DDD ceremony slow down iteration — pick `references/templates/single-layer.md` (Single-Layer template) instead. (2) The system is already known to require independent scaling per bounded context, polyglot persistence per service, or per-service deployment cadence — pick `references/templates/microservice.md` (Microservice template) instead. (3) You only need a hosted API gateway or BFF in front of existing services with no domain logic of its own — Layered's Domain/Application split becomes dead weight. (4) You need React or Vue as the primary UI — the Layered template officially supports MVC/Razor, Angular, and Blazor variants only; React/React-Native is available only as the optional mobile client (not a primary web UI). Migration paths: Single-Layer → Layered means lifting entity/service code into Domain and Application projects (mostly mechanical); Layered → Microservice means promoting bounded contexts into independent solutions and adding gateways/messaging — non-trivial.

## Built-in features

- Authentication — OpenIddict-based auth server (free tier) with optional OpenIddict UI module (Pro+); MVC uses Hybrid flow, SPA/Swagger uses Authorization Code + PKCE, APIs use JWT Bearer; `OpenIddictDataSeedContributor` seeds clients/scopes from appsettings.json; tiered solutions get a dedicated `<Project>.AuthServer` host. Deep-dive: https://abp.io/docs/10.3/solution-templates/layered-web-application/authentication. Default config: `openiddict.pfx` self-signed cert with random password (replace for production).
- Database configurations — SQL Server default via `Configure<AbpDbContextOptions>(o => o.UseSqlServer())`; alternative providers PostgreSQL/MySQL/SQLite/Oracle/MongoDB; connection string under `ConnectionStrings:Default` in appsettings.json; `IDesignTimeDbContextFactory` for `dotnet ef` tooling; per-tenant connection strings via SaaS module. Deep-dive: https://abp.io/docs/10.3/solution-templates/layered-web-application/database-configurations.
- Logging (Serilog) — initialized in Program.cs with Console + ABP Studio sinks always, File sink (`Logs/logs.txt`) under `#if DEBUG` only; `app.UseAbpSerilogEnrichers()` adds tenant/user/client/correlation properties to every log event; supports any Serilog sink for production (Elastic, Seq, Application Insights). Deep-dive: https://abp.io/docs/10.3/solution-templates/layered-web-application/logging.
- Swagger integration — pre-wired Swagger UI on the Web host (and HttpApi.Host in tiered) with OpenIddict-aware OAuth2 flow so operators can authenticate inside the UI; configured via `AddAbpSwaggerGen` in the Web/HttpApi.Host module. Deep-dive: https://abp.io/docs/10.3/solution-templates/layered-web-application/swagger-integration (cross-references the framework-level https://abp.io/docs/10.3/framework/api-development/swagger).
- Background jobs — default in-memory/DB provider from `Volo.Abp.BackgroundJobs`; replaceable with Hangfire, Quartz, RabbitMQ, or TickerQ integrations; jobs implement `IAsyncBackgroundJob<TArgs>` and are queued via `IBackgroundJobManager.EnqueueAsync`. Deep-dive: https://abp.io/docs/10.3/solution-templates/layered-web-application/background-jobs.
- Background workers — periodic in-process workers via `AsyncPeriodicBackgroundWorkerBase`; registered with `context.AddBackgroundWorkerAsync<T>()` in `OnApplicationInitializationAsync`; combine with distributed locking when running multi-instance. Deep-dive: https://abp.io/docs/10.3/solution-templates/layered-web-application/background-workers.
- Distributed locking — `Volo.Abp.DistributedLocking` package with Redis backend; surface via `IAbpDistributedLock.TryAcquireAsync(name)`; included by default ONLY in Tiered or Public-Website variants — non-tiered solutions must add the package + Redis registration manually. Deep-dive: https://abp.io/docs/10.3/solution-templates/layered-web-application/distributed-locking.
- Multi-tenancy — opt-in at solution-creation; `ICurrentTenant` for tenant context, `[MultiTenant]`/`IMultiTenant` on entities, automatic data filter, `using (CurrentTenant.Change(id)) { ... }` for explicit switching, `IDataFilter<IMultiTenant>.Disable()` to bypass; SaaS module (Team+) provides per-tenant connection-string management and admin UI; cache, BG jobs, and events all carry tenant id. Deep-dive: https://abp.io/docs/10.3/solution-templates/layered-web-application/multi-tenancy.
- BLOB storing — Database provider is the layered template default (BLOBs stored in SQL/Mongo); replaceable with FileSystem, Azure Blob, AWS S3, MinIO, Aliyun OSS, Google Cloud Storage, Bunny.Net via `Configure<AbpBlobStoringOptions>`; `IBlobContainer.SaveAsync/GetAllBytesOrNullAsync/DeleteAsync` API; optional File Management module adds folder/file UI. Deep-dive: https://abp.io/docs/10.3/solution-templates/layered-web-application/blob-storing.
- CORS configuration — `App:CorsOrigins` setting in appsettings.json, comma-separated, wildcard subdomain support (`https://*.acme.com`); required for Tiered, Angular, Blazor WebAssembly, and No-UI deployments where the SPA/API live on a different origin from the AuthServer. Deep-dive: https://abp.io/docs/10.3/solution-templates/layered-web-application/cors-configuration.

## Deployment options

- Docker Compose — `etc/docker-compose/docker-compose.yml` with services for Web (or Web + AuthServer + HttpApi.Host in tiered), DbMigrator (waits for SQL health), and SQL Server/Redis/RabbitMQ; PowerShell `build-images-locally.ps1` + `run-docker.ps1` (and `.sh` siblings on Linux/Mac); self-signed certs mounted from `./certs`. Best for local integration testing and small single-host production setups. Source: https://abp.io/docs/10.3/solution-templates/layered-web-application/deployment/deployment-docker-compose.

- Kubernetes / Helm — `etc/helm/<project>` chart (Chart.yaml, values.yaml, per-env values.<project>-local.yaml) with sub-charts for each app and Redis/RabbitMQ; `build-all-images.ps1`, `install.ps1`, `uninstall.ps1`, `create-tls-secrets.ps1` automate the full lifecycle; ABP Studio adds a GUI. REQUIRES ABP Business or higher license. Best for multi-instance horizontally scaled deployments. Source: https://abp.io/docs/10.3/solution-templates/layered-web-application/helm-charts-and-kubernetes.

- Azure App Service — three-step flow: (1) provision Azure Web App + Azure SQL or Cosmos DB, (2) update appsettings/JSON for production URLs and connection strings, (3) deploy via GitHub Actions; supports MVC, Angular, Blazor variants with EF Core or MongoDB. Best when the team is already in the Azure ecosystem and wants managed PaaS. Source: https://abp.io/docs/10.3/solution-templates/layered-web-application/deployment/azure-deployment/azure-deployment.

- IIS (on-prem Windows) — install .NET hosting bundle on the IIS server, prepare EF Core target DB, generate OpenIddict signing cert via `dotnet dev-certs https -v -ep authserver.pfx -p <pwd>`, `dotnet publish -c Release` for DbMigrator + Web, configure IIS site with 'Load User Profile' enabled, remove WebDAV module from Web.config, optionally enable stdout logging for troubleshooting 502.5/500.3x. Best for regulated/on-prem environments where cloud is not allowed. Source: https://abp.io/docs/10.3/solution-templates/layered-web-application/deployment/deployment-iis.

- OpenIddict-specific production guidance — replace dev signing/encryption certs with real ones (or `IDX10501` errors at runtime); store certs in Windows cert store on IIS, or use `WEBSITE_LOAD_CERTIFICATES` on Azure; configure `ForwardedHeadersOptions` for `X-Forwarded-For` / `X-Forwarded-Proto` when behind Nginx/Kubernetes ingress; keep DbMigrator's appsettings.json in sync with production OpenIddict client URLs. Source: https://abp.io/docs/10.3/solution-templates/layered-web-application/deployment/openiddict-deployment.

- IdentityServer-specific production guidance — applies to legacy/Pro variants where IdentityServer is selected instead of OpenIddict: update `CorsOrigins`, `RedirectAllowedUrls`, replace dev signing cert, enable HTTPS+HSTS, use HTTP internally + HTTPS via ingress in Kubernetes; `IdentityServerDataSeedContributor` reads client config from appsettings.json. Source: https://abp.io/docs/10.3/solution-templates/layered-web-application/deployment/identityserver-deployment.

## Differences from other templates

vs Single-Layer (`references/templates/single-layer.md`):
- Project count: Layered ships 8-15 csproj (Domain.Shared, Domain, Application.Contracts, Application, EntityFrameworkCore, HttpApi, HttpApi.Client, Web, DbMigrator, plus tests; +AuthServer/HttpApi.Host if tiered) vs Single-Layer's 1-2 csproj (everything in one project, optional .Tests).
- Layout: Layered enforces DDD layer boundaries through project references (Application can't reference EntityFrameworkCore directly, only via repository interfaces in Domain) — Single-Layer puts Entities/Services/DbContext in a single namespace.
- Gained: testability per layer, ability to swap persistence (EF Core ↔ MongoDB) without touching application/domain code, cleaner refactoring path to microservices.
- Lost: setup speed, cognitive overhead of cross-project navigation, more nuget/build time.
- Migration Single-Layer → Layered: roughly mechanical move of entities into Domain, DbContext into EntityFrameworkCore, services split across Application/Application.Contracts; ABP Studio has tooling that helps but result still needs review.

vs Microservice (`references/templates/microservice.md`):
- Process model: Layered = single deployable (or 3 in tiered) sharing one DB schema; Microservice = N independent services + API gateway(s) + service-to-service auth + per-service DBs.
- Communication: Layered uses in-process method calls between Application and Domain; Microservice uses HTTP/gRPC + a distributed event bus (RabbitMQ/Kafka) plus IDistributedEventBus.
- Infra ceiling: Layered tops out at Helm/K8s with a small chart; Microservice ships gateways, service discovery, per-service Helm sub-charts, multi-database orchestration, and a microservice-specific Tye/dotnet-aspire wiring.
- Gained by Microservice: independent scaling per bounded context, polyglot persistence, per-service deploy cadence, blast-radius isolation.
- Lost by Microservice: simplicity, debuggability, and the ability for a single F5 to launch the system — Layered is dramatically faster to onboard and to run locally.
- Migration Layered → Microservice: promote each bounded context's Domain/Application/HttpApi triplet into a standalone solution, replace direct DbContext access with HTTP/event-bus calls, introduce gateway and shared identity service. Significant work but Layered's project boundaries make it tractable; Single-Layer first needs Layered-style refactor before this is possible.

## Common mistakes

- Putting EF Core code in the Domain or Application project. Why it's wrong: it inverts the DDD direction the template enforces, breaks `Domain.Tests`/`Application.Tests` (they'd suddenly need a real DB), and prevents swapping the persistence provider. Correct pattern: keep `DbContext`, EF entity configurations, and repository implementations in `<Project>.EntityFrameworkCore`; expose only `IRepository<TEntity, TKey>` (or custom repository interfaces) from the Domain project.
- Editing only the Web project's appsettings.json when changing ConnectionStrings or OpenIddict settings. Why it's wrong: DbMigrator owns its own appsettings and seeds OpenIddict applications independently — if Web has the production URL but DbMigrator still has localhost, OpenIddict client redirect URIs will be seeded incorrectly and login will fail with `invalid_redirect_uri`. Correct pattern: update Web/AuthServer/HttpApi.Host AND DbMigrator appsettings.json in lockstep (or use a shared environment-variable strategy).
- Running a periodic background worker on a horizontally scaled Web app without `IAbpDistributedLock`. Why it's wrong: every replica fires `DoWorkAsync` on the same period, leading to N-fold side effects (duplicate emails, double-charged jobs, race conditions). Correct pattern: gate the work with `await using var handle = await _lock.TryAcquireAsync('worker-name'); if (handle == null) return;`. Note: in the non-tiered Layered template, `Volo.Abp.DistributedLocking` is NOT included by default — you must add the NuGet package and configure Redis.
- Shipping the dev OpenIddict signing certificate (`openiddict.pfx` with auto-generated random password) to production. Why it's wrong: causes `IDX10501: Signature validation failed` once the cert rotates, and stores a generated key inside the image which leaks across deployments. Correct pattern: provision a real cert (e.g., via Azure Key Vault or Windows cert store) and load it through the AuthServer module's signing-credential configuration; on Azure expose it via `WEBSITE_LOAD_CERTIFICATES`.
- Forgetting `app.UseForwardedHeaders` or `ForwardedHeadersOptions` when deploying behind Nginx/Kubernetes ingress. Why it's wrong: AuthServer sees the inbound request as `http://internal-host` and emits issuer/redirect URLs with the wrong scheme/host, breaking discovery and PKCE. Correct pattern: add `app.UseForwardedHeaders(new ForwardedHeadersOptions { ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto });` early in the AuthServer/Web pipeline.
- Bypassing the `IDataFilter<IMultiTenant>` carelessly to 'see all tenants' in a production query. Why it's wrong: the filter is the only thing keeping tenant A from reading tenant B's data; disabling it in shared code paths leaks data across tenants. Correct pattern: only disable inside an admin-only host service that explicitly requires cross-tenant scope, and always wrap in `using (...) { }` so the filter re-enables on exception.
- Adding controllers/services to the wrong project. Why it's wrong: putting business logic in HttpApi means tests can't run without ASP.NET Core; putting DTOs in Application breaks HttpApi.Client (which only references Application.Contracts). Correct pattern: DTOs + service interfaces in Application.Contracts, implementations in Application, controllers (only when overriding auto-API) in HttpApi.
- Removing the `[ReplaceDbContext]` attribute from the host DbContext to 'clean it up'. Why it's wrong: doing so makes Identity/SaaS/OpenIddict create their own DbContexts, which need their own connection strings and migrations, splitting the database into multiple schemas unintentionally. Correct pattern: leave the attributes in place; if you genuinely want separate databases, also add separate `ConnectionStrings` entries and configure each module's options accordingly.
## Version pins (ABP 10.3)

> **Note:** items marked `[uncertain]` need verification against ABP 10.3 source. Treat as best-effort guidance.

- ABP 10.3 prerequisites: Visual Studio 2026, .NET 10.0+, Node 22.11+, Docker Desktop. Earlier ABP versions targeted .NET 8/9; pinning to 10.3 means the get-started commands assume `dotnet --version` reports 10.x.
- Default admin credentials remain `admin` / `1q2w3E*` in 10.3 (per get-started doc); ensure these are rotated before any non-local deployment.
- Distributed locking is NOT included by default in non-tiered Layered solutions in 10.3 — only Tiered and Public-Website variants ship `Volo.Abp.DistributedLocking` pre-wired. [uncertain] whether this default flips in a future minor version.
- Helm charts and on-the-fly database migration require an ABP Business+ (Helm) or Team+ (auto-migration) license in 10.3. License-tier coupling is ABP commercial policy and may shift in future versions.
- Default BLOB provider is the Database provider in 10.3 Layered template. [uncertain] whether the default changes to FileSystem in future templates — the doc page calls Database the default explicitly today.
- OpenIddict is the default auth server in 10.3; IdentityServer is still documented as a deployment path but is treated as legacy. [uncertain] when IdentityServer support will be removed entirely.
- Web UI options officially supported by the Layered template in 10.3: MVC/Razor Pages, Angular, Blazor WebAssembly, Blazor Server, Blazor WebApp, MAUI Blazor (Hybrid), No-UI. React/Vue are NOT first-class web UI options (React Native is mobile-only).
- The `etc/helm` chart structure (Chart.yaml, values.yaml, values.<project>-local.yaml, charts/, templates/) and the helper scripts (build-all-images.ps1, install.ps1, uninstall.ps1, create-tls-secrets.ps1) are documented for 10.3; script names have been stable across recent versions but are [uncertain] for forward compatibility.
- AbpDbContextOptions extension methods used in the EntityFrameworkCore module include `UseSqlServer`, `UseNpgsql`, `UseMySQL`, `UseSqlite`, `UseOracle`. [uncertain] whether `UseOracle` requires the commercial Oracle.EntityFrameworkCore package or ships under a different option name in 10.3 — verify against the EF Core docs page if Oracle is the target.

## Cross-references

**Phase 1 references:**
- [references/templates/single-layer.md](../templates/single-layer.md) — triggered when the user is comparing complexity tiers ('do I really need all these projects?') or asks 'should I move from single-layer to layered?'
- [references/templates/microservice.md](../templates/microservice.md) — triggered when the user asks about scaling beyond a monolith, splitting bounded contexts, or comparing Layered vs Microservice for a new project.
- [references/framework/architecture-ddd.md](../framework/architecture-ddd.md) — triggered whenever the user asks about Domain vs Application responsibilities, aggregate roots, domain services, or what belongs in which Layered project.
- [references/framework/modularity.md](../framework/modularity.md) — triggered whenever the user discusses AbpModule, [DependsOn], PreConfigureServices/ConfigureServices/OnApplicationInitialization lifecycle, or adds a new module to a Layered solution.
- [references/framework/multi-tenancy.md](../framework/multi-tenancy.md) — triggered when the user asks about ICurrentTenant, tenant data filter, per-tenant connection strings, or the SaaS module.
- [references/framework/authorization.md](../framework/authorization.md) — triggered when the user defines permissions, applies [Authorize], or wires the OpenIddict client list (these live in Application.Contracts in a Layered solution).
- [references/framework/ef-core.md](../framework/ef-core.md) — triggered when the user asks about AbpDbContext, repositories, ReplaceDbContext, IDesignTimeDbContextFactory, or `dotnet ef` migrations specifically inside the Layered EntityFrameworkCore project.
- [references/framework/api-development.md](../framework/api-development.md) — triggered when the user designs HttpApi controllers, auto-API conventions, or HttpApi.Client proxy generation.

**External docs:**
- OpenIddict documentation (https://documentation.openiddict.com/) — referenced by the layered template's authentication and openiddict-deployment pages; used for token endpoints, flow configuration, and certificate management beyond what ABP wraps.
- Serilog documentation (https://serilog.net/) — referenced by the logging page; consult for sink configuration (Elastic, Seq, Application Insights) when extending the layered template's default Console + File + ABP Studio setup.
- Entity Framework Core documentation (https://learn.microsoft.com/ef/core/) — referenced by database-configurations and DbMigrator pages; consult for `dotnet ef migrations`, `IDesignTimeDbContextFactory`, and provider-specific connection-string formats.
- ASP.NET Core hosting / IIS documentation (https://learn.microsoft.com/aspnet/core/host-and-deploy/iis/) — referenced by the IIS deployment page; required for .NET hosting bundle install and Web.config tuning.
- Microsoft .NET MAUI documentation (https://learn.microsoft.com/dotnet/maui/) — referenced by the mobile-applications page when MAUI is the chosen mobile framework.
- React Native + Expo documentation (https://reactnative.dev/, https://expo.dev/) — referenced by the mobile-applications page when React Native is chosen; ABP does not run RN inside its Solution Runner so the standard RN toolchain applies.
- Helm documentation (https://helm.sh/docs/) — referenced by the Helm Charts and Kubernetes page; values.yaml override mechanics and chart templating live there, not in ABP docs.
- Azure App Service documentation (https://learn.microsoft.com/azure/app-service/) — referenced by the Azure deployment page for App Service provisioning, GitHub Actions deploy actions, and `WEBSITE_LOAD_CERTIFICATES` configuration.
- Microsoft.AspNetCore.Authentication / Microsoft Identity (https://learn.microsoft.com/aspnet/core/security/) — referenced indirectly by the OpenIddict + Identity wiring; consult when customizing claims or adding external login providers.

## Sources (ABP 10.3)

- [https://abp.io/docs/10.3/solution-templates/layered-web-application/overview](https://abp.io/docs/10.3/solution-templates/layered-web-application/overview) — High-level intro: what the layered template is, included libraries (Autofac, Serilog, Redis, Swagger, OpenIddict), pre-configured features, fundamental modules (Account, Identity, OpenIddict, Feature/Permission/Setting Management), optional modules, supported DB providers, UI frameworks, mobile options, multi-tenancy.
- [https://abp.io/docs/10.3/solution-templates/layered-web-application/solution-structure](https://abp.io/docs/10.3/solution-templates/layered-web-application/solution-structure) — Project layout: Domain.Shared/Domain/Application.Contracts/Application/EntityFrameworkCore/HttpApi/HttpApi.Client/Web/DbMigrator plus tests; tiered adds AuthServer + HttpApi.Host; folders etc/abp-studio, etc/docker, etc/helm, src, test.
- [https://abp.io/docs/10.3/solution-templates/layered-web-application/main-components](https://abp.io/docs/10.3/solution-templates/layered-web-application/main-components) — The three primary application categories: web applications, mobile applications, and the DbMigrator console.
- [https://abp.io/docs/10.3/solution-templates/layered-web-application/web-applications](https://abp.io/docs/10.3/solution-templates/layered-web-application/web-applications) — Supported web UIs: MVC/Razor Pages, Angular, Blazor WebAssembly, Blazor Server, Blazor WebApp, MAUI Blazor (Hybrid), and No-UI option.
- [https://abp.io/docs/10.3/solution-templates/layered-web-application/db-migrator](https://abp.io/docs/10.3/solution-templates/layered-web-application/db-migrator) — DbMigrator console app, DbMigratorHostedService, separation of Domain abstraction vs EF Core implementation, independent appsettings.
- [https://abp.io/docs/10.3/solution-templates/layered-web-application/mobile-applications](https://abp.io/docs/10.3/solution-templates/layered-web-application/mobile-applications) — Optional MAUI and React Native projects, target platforms, AuthServer/MobileGateway prerequisites.
- [https://abp.io/docs/10.3/solution-templates/layered-web-application/built-in-features](https://abp.io/docs/10.3/solution-templates/layered-web-application/built-in-features) — Roster of pre-wired features: Auth, DB config, Logging, Swagger, BG Jobs, BG Workers, Distributed Locking, Multi-Tenancy, BLOB Storing, CORS.
- [https://abp.io/docs/10.3/solution-templates/layered-web-application/authentication](https://abp.io/docs/10.3/solution-templates/layered-web-application/authentication) — OpenIddict as default auth server; AuthServer project under tiered option; Hybrid/Authorization Code/JWT Bearer flows; OpenIddictDataSeedContributor.
- [https://abp.io/docs/10.3/solution-templates/layered-web-application/database-configurations](https://abp.io/docs/10.3/solution-templates/layered-web-application/database-configurations) — Connection strings in appsettings.json, AbpDbContext + ConnectionStringName + ReplaceDbContext attributes, AbpDbContextOptions.UseSqlServer(), IDesignTimeDbContextFactory, per-tenant connection strings via SaaS.
- [https://abp.io/docs/10.3/solution-templates/layered-web-application/logging](https://abp.io/docs/10.3/solution-templates/layered-web-application/logging) — Serilog initialized in Program.cs, Console + File (logs.txt under Logs/) + ABP Studio sinks, UseAbpSerilogEnrichers for tenant/user/correlation enrichment.
- [https://abp.io/docs/10.3/solution-templates/layered-web-application/swagger-integration](https://abp.io/docs/10.3/solution-templates/layered-web-application/swagger-integration) — Swagger UI shipped by default; pointer to the framework-level Swagger Integration guide for AddAbpSwaggerGen specifics.
- [https://abp.io/docs/10.3/solution-templates/layered-web-application/background-jobs](https://abp.io/docs/10.3/solution-templates/layered-web-application/background-jobs) — Default BG Jobs module + integrations: Hangfire, Quartz, RabbitMQ, TickerQ.
- [https://abp.io/docs/10.3/solution-templates/layered-web-application/background-workers](https://abp.io/docs/10.3/solution-templates/layered-web-application/background-workers) — AsyncPeriodicBackgroundWorkerBase pattern, AddBackgroundWorkerAsync registration in OnApplicationInitializationAsync, distributed-locking caveat.
- [https://abp.io/docs/10.3/solution-templates/layered-web-application/distributed-locking](https://abp.io/docs/10.3/solution-templates/layered-web-application/distributed-locking) — Volo.Abp.DistributedLocking (Redis backend), IAbpDistributedLock.TryAcquireAsync usage; included by default only in Tiered/Public-Website variants.
- [https://abp.io/docs/10.3/solution-templates/layered-web-application/multi-tenancy](https://abp.io/docs/10.3/solution-templates/layered-web-application/multi-tenancy) — Optional multi-tenancy, ICurrentTenant, separate-DB or shared-DB-with-filter, IDataFilter<IMultiTenant> bypass, SaaS module integration (Team+).
- [https://abp.io/docs/10.3/solution-templates/layered-web-application/blob-storing](https://abp.io/docs/10.3/solution-templates/layered-web-application/blob-storing) — Database provider as default; FileSystem/Azure/AWS/MinIO/Aliyun/Google/Bunny.Net providers; IBlobContainer.SaveAsync/GetAllBytesOrNullAsync.
- [https://abp.io/docs/10.3/solution-templates/layered-web-application/cors-configuration](https://abp.io/docs/10.3/solution-templates/layered-web-application/cors-configuration) — App:CorsOrigins setting in appsettings.json; required for Tiered/Angular/Blazor-WASM/No-UI scenarios.
- [https://abp.io/docs/10.3/solution-templates/layered-web-application/helm-charts-and-kubernetes](https://abp.io/docs/10.3/solution-templates/layered-web-application/helm-charts-and-kubernetes) — etc/helm directory, Chart.yaml/values.yaml, build-all-images.ps1/install.ps1/uninstall.ps1 scripts; requires Business+ license.
- [https://abp.io/docs/10.3/solution-templates/layered-web-application/deployment/deployment-docker-compose](https://abp.io/docs/10.3/solution-templates/layered-web-application/deployment/deployment-docker-compose) — etc/docker-compose with docker-compose.yml, build-images-locally.ps1, run-docker.ps1; bookstore-web service + db-migrator service + SQL Server.
- [https://abp.io/docs/10.3/solution-templates/layered-web-application/deployment/azure-deployment/azure-deployment](https://abp.io/docs/10.3/solution-templates/layered-web-application/deployment/azure-deployment/azure-deployment) — Azure App Service deployment via GitHub Actions; Azure SQL or Cosmos DB prerequisite.
- [https://abp.io/docs/10.3/solution-templates/layered-web-application/deployment/deployment-iis](https://abp.io/docs/10.3/solution-templates/layered-web-application/deployment/deployment-iis) — IIS prerequisites (.NET hosting bundle, EF Core DB), dotnet publish for DbMigrator/Web, OpenIddict cert generation, Web.config WebDAV removal, stdout logging for diagnostics.
- [https://abp.io/docs/10.3/solution-templates/layered-web-application/deployment/identityserver-deployment](https://abp.io/docs/10.3/solution-templates/layered-web-application/deployment/identityserver-deployment) — Production IdentityServer config: CorsOrigins, RedirectAllowedUrls, IdentityServerDataSeedContributor, signing cert replacement, HTTPS/HSTS, K8s ingress.
- [https://abp.io/docs/10.3/solution-templates/layered-web-application/deployment/openiddict-deployment](https://abp.io/docs/10.3/solution-templates/layered-web-application/deployment/openiddict-deployment) — OpenIddict production: signing/encryption certificates, openiddict.pfx, ForwardedHeadersOptions behind Nginx/K8s, DbMigrator appsettings sync.
- [https://abp.io/docs/10.3/get-started/layered-web-application](https://abp.io/docs/10.3/get-started/layered-web-application) — Get-started: Visual Studio 2026/.NET 10.0+/Node 22.11+/Docker prerequisites; ABP Studio template selection 'Application (Layered)'; Solution Runner; default admin/1q2w3E* credentials.
- [https://abp.io/docs/10.3/solution-templates/layered-web-application](https://abp.io/docs/10.3/solution-templates/layered-web-application) — Index page with TOC and brief intro: monolithic, DDD-based architecture.
- [https://abp.io/docs/10.3/solution-templates](https://abp.io/docs/10.3/solution-templates) — Solution-template comparison: when to pick Single-Layer vs Layered vs Microservice.

Last verified: 2026-05-10
