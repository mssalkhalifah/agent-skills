# Single-Layer Solution Template

> ABP single-layer (a.k.a. app-nolayers) template — a one-project, folder-organized monolith preconfigured with Autofac DI, Serilog, Swagger, OpenIddict, EF Core, multi-tenancy, background jobs/workers, BLOB storage, and CORS so DDD-aware teams can ship a small-to-medium ABP app without a layered project graph.

## When to load this reference

- User asks 'should I use the layered or no-layers template?' or 'which ABP template fits a small monolith?'
- User asks about the structure, project list, or .csproj layout of an `abp new ... --template app-nolayers` solution.
- User asks where a specific feature lives in a single-layer ABP app (DbContext location, Pages folder, Permissions class, Migrations folder, etc.).
- User asks how a built-in feature (auth, logging, Swagger, jobs, workers, multi-tenancy, BLOB, CORS, distributed lock) is preconfigured by the single-layer template specifically.
- User asks how to migrate from single-layer to layered, or wants the trade-offs of staying on single-layer.
- Questions referencing OpenIddictDataSeedContributor, app-nolayers, AbpDbContext<T> in a Data folder, or 'no-layers' templating language.
- User wants the canonical list of UI/DB options the single-layer template supports at create time.

**Audience:** Developers and architects evaluating or actively using the ABP single-layer (app-nolayers) template — typically small teams shipping a monolithic web app or API, prototypers, and anyone who wants ABP's modules and DDD building blocks without a multi-project solution graph.

## Key concepts

- Single-project module class (e.g., `BookstoreModule : AbpModule`, namespace `Acme.Bookstore`) — the one ABP module that wires every option in `PreConfigureServices`/`ConfigureServices`/`OnApplicationInitializationAsync`. Reach for it whenever you need to add a dependency, configure options, or register a worker.
- `AbpDbContext<TDbContext>` (Volo.Abp.EntityFrameworkCore) — base EF Core context the template's `XxxDbContext` (in `Data/`) inherits from. Owns `OnModelCreating` and ABP's automatic filters/auditing. Reach for it when adding entities or fluent mappings.
- `IRepository<TEntity, TKey>` / `IRepository<TEntity>` (Volo.Abp.Domain.Repositories) — generic repository pulled from EF Core; default for any aggregate root in `Entities/`. Reach for it from application services in `Services/` instead of touching DbContext directly.
- `ApplicationService` / `IApplicationService` (Volo.Abp.Application.Services) — base class for the auto-controller-exposed services in `Services/`. Reach for it when you want a class to be both DI-resolved and surfaced as a REST endpoint via dynamic-API-controllers.
- `OpenIddictDataSeedContributor` (in the host's `Data/` folder; Volo.Abp.OpenIddict) — runs at startup with `--migrate-database` and seeds default OpenIddict applications/scopes; only present when authentication is not delegated to UI flows. Edit it to add new clients/scopes.
- `ICurrentTenant` (Volo.Abp.MultiTenancy) — the tenant-context accessor when multi-tenancy is enabled. Reach for it to read/scope tenant id, or call `ICurrentTenant.Change(tenantId)` to cross tenant boundaries.
- `IDataFilter<IMultiTenant>` (Volo.Abp.Data) — disables the tenant filter inside a `using` block when you legitimately need cross-tenant queries (host-side admin, reports).
- `IBackgroundJobManager` (Volo.Abp.BackgroundJobs) — fire-and-forget durable jobs. Default storage in single-layer is the EF Core BackgroundJobs table (database-backed); swap to Hangfire/RabbitMQ/Quartz/TickerQ via the matching integration package.
- `AsyncPeriodicBackgroundWorkerBase` (Volo.Abp.BackgroundWorkers) — base class for in-process periodic workers; register via `await context.AddBackgroundWorkerAsync<TWorker>()` in `OnApplicationInitializationAsync`.
- `IAbpDistributedLock` (Volo.Abp.DistributedLocking) — cluster-safe lock. Single-layer ships only the in-process default; for multi-instance deployments add `Volo.Abp.DistributedLocking.Redis` (or another provider) and configure it in the module.
- `IBlobContainer` / `IBlobContainer<TContainer>` (Volo.Abp.BlobStoring) — abstraction over BLOB storage; the single-layer template's default backing store is the Database provider (`Volo.Abp.BlobStoring.Database`).
- ABP `Db Migrator` flag — running the host with `dotnet run -- --migrate-database` creates the database, applies EF migrations, and seeds initial data (OpenIddict, identity admin, tenants). There is no separate DbMigrator console project in this template (unlike the layered template).
- `App:CorsOrigins` (appsettings.json) — comma-separated origin list consumed by ABP's CORS setup; only used when the UI is Angular, Blazor WASM, or 'no UI'.

## Configuration pattern

Everything is wired in the single module class (e.g., `BookstoreModule : AbpModule`) at the project root. Typical sections:

```csharp
public class BookstoreModule : AbpModule
{
    public override void PreConfigureServices(ServiceConfigurationContext context)
    {
        // OpenIddict signing/encryption certificates are pre-configured here
        // for production environments.
    }

    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        var configuration = context.Services.GetConfiguration();
        var hostingEnvironment = context.Services.GetHostingEnvironment();

        // EF Core: pick a single provider here.
        Configure<AbpDbContextOptions>(options =>
        {
            options.UseSqlServer();      // or UseNpgsql() / UseMySql() / UseSqlite()
        });

        // Multi-tenancy toggle (read by AbpMultiTenancyOptions).
        Configure<AbpMultiTenancyOptions>(options =>
        {
            options.IsEnabled = MultiTenancyConsts.IsEnabled;
        });

        // CORS — only when Angular / Blazor WASM / no UI.
        context.Services.AddCors(options =>
        {
            options.AddDefaultPolicy(builder =>
            {
                builder
                    .WithOrigins(
                        configuration["App:CorsOrigins"]!
                            .Split(',', StringSplitOptions.RemoveEmptyEntries)
                            .Select(o => o.RemovePostFix("/"))
                            .ToArray()
                    )
                    .WithAbpExposedHeaders()
                    .SetIsOriginAllowedToAllowWildcardSubdomains()
                    .AllowAnyHeader()
                    .AllowAnyMethod()
                    .AllowCredentials();
            });
        });

        // BLOB storing — single-layer default container goes to the database.
        Configure<AbpBlobStoringOptions>(options =>
        {
            options.Containers.ConfigureDefault(c =>
            {
                c.UseDatabase();
            });
        });
    }

    public override async Task OnApplicationInitializationAsync(
        ApplicationInitializationContext context)
    {
        var app = context.GetApplicationBuilder();
        app.UseAbpRequestLocalization();
        app.UseStaticFiles();
        app.UseRouting();
        app.UseAbpSerilogEnrichers();   // enriches every log with tenant/user/correlation
        app.UseAuthentication();
        app.UseAbpOpenIddictValidation();
        if (MultiTenancyConsts.IsEnabled) { app.UseMultiTenancy(); }
        app.UseUnitOfWork();
        app.UseAuthorization();
        app.UseSwagger();
        app.UseAbpSwaggerUI();
        app.UseAuditing();
        app.UseAbpSerilogEnrichers();
        app.UseConfiguredEndpoints();

        await context.AddBackgroundWorkerAsync<PassiveUserCheckerWorker>();
    }
}
```

Key points:
- One module wires everything; there is no separate `Application` or `EntityFrameworkCore` module class to thread options through.
- `--migrate-database` startup arg triggers a code-path that runs EF migrations and seed contributors (including `OpenIddictDataSeedContributor`) before the host begins serving requests.
- Background workers register in `OnApplicationInitializationAsync`, not in `ConfigureServices`.

## Code examples

Title: Define an aggregate root in `Entities/`
Scenario: First-time entity in a single-layer project — author keeps it next to the DbContext.
```csharp
// Entities/Book.cs
using System;
using Volo.Abp.Domain.Entities.Auditing;
using Volo.Abp.MultiTenancy;

namespace Acme.Bookstore.Entities;

public class Book : FullAuditedAggregateRoot<Guid>, IMultiTenant
{
    public Guid? TenantId { get; set; }
    public string Name { get; set; } = default!;
    public BookType Type { get; set; }
    public DateTime PublishDate { get; set; }
    public float Price { get; set; }
}
```
Key lines: `FullAuditedAggregateRoot<Guid>` gives soft-delete + audit fields automatically; `IMultiTenant` makes the row participate in the tenant filter when multi-tenancy is enabled. The entity sits in `Entities/`, not a separate `*.Domain` project.
Title: Map the entity in `Data/BookstoreDbContext.cs`
Scenario: Configure EF Core mapping inside the single template's only DbContext.
```csharp
// Data/BookstoreDbContext.cs
using Acme.Bookstore.Entities;
using Microsoft.EntityFrameworkCore;
using Volo.Abp.Data;
using Volo.Abp.EntityFrameworkCore;
using Volo.Abp.EntityFrameworkCore.Modeling;

namespace Acme.Bookstore.Data;

[ConnectionStringName("Default")]
public class BookstoreDbContext : AbpDbContext<BookstoreDbContext>
{
    public DbSet<Book> Books => Set<Book>();

    public BookstoreDbContext(DbContextOptions<BookstoreDbContext> options)
        : base(options) { }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);  // wires every ABP module's mappings
        builder.Entity<Book>(b =>
        {
            b.ToTable("AppBooks");
            b.ConfigureByConvention();   // soft-delete, auditing, multi-tenancy
            b.Property(x => x.Name).IsRequired().HasMaxLength(128);
        });
    }
}
```
Key lines: `[ConnectionStringName("Default")]` ties this context to the `ConnectionStrings:Default` entry in `appsettings.json`; `ConfigureByConvention()` is the one call that respects all ABP marker interfaces (audit, soft-delete, `IMultiTenant`).
Title: Register a periodic background worker
Scenario: Run a deactivation sweep every 10 minutes; only one worker class needs to be registered.
```csharp
// Services/PassiveUserCheckerWorker.cs
public class PassiveUserCheckerWorker : AsyncPeriodicBackgroundWorkerBase
{
    public PassiveUserCheckerWorker(
        AbpAsyncTimer timer,
        IServiceScopeFactory serviceScopeFactory)
        : base(timer, serviceScopeFactory)
    {
        Timer.Period = 600_000; // 10 minutes
    }

    protected override async Task DoWorkAsync(
        PeriodicBackgroundWorkerContext workerContext)
    {
        var userRepository = workerContext.ServiceProvider
            .GetRequiredService<IUserRepository>();
        await userRepository.UpdateInactiveUserStatusesAsync();
    }
}

// In BookstoreModule.OnApplicationInitializationAsync:
await context.AddBackgroundWorkerAsync<PassiveUserCheckerWorker>();
```
Key lines: workers must be registered from `OnApplicationInitializationAsync`, not constructor injection; resolve scoped services via `workerContext.ServiceProvider` to get a fresh scope per tick.
Title: Use `IAbpDistributedLock` to make a worker safe across instances
Scenario: When you scale the single-layer host horizontally, periodic workers will run on every replica unless you guard them with a distributed lock.
```csharp
protected override async Task DoWorkAsync(PeriodicBackgroundWorkerContext ctx)
{
    var distributedLock = ctx.ServiceProvider
        .GetRequiredService<IAbpDistributedLock>();

    await using var handle = await distributedLock.TryAcquireAsync(
        name: "PassiveUserCheckerWorker");

    if (handle is null)
    {
        return; // another instance is already running this tick
    }

    var userRepository = ctx.ServiceProvider.GetRequiredService<IUserRepository>();
    await userRepository.UpdateInactiveUserStatusesAsync();
}
```
Key lines: `TryAcquireAsync` returns `null` instead of throwing on contention; the in-process default lock is single-instance-only — add `Volo.Abp.DistributedLocking.Redis` and configure it in the module before relying on this in production.
Title: appsettings.json shape for a single-layer host
Scenario: Wiring connection string, CORS origins, OpenIddict, and Logs.
```json
{
  "App": {
    "SelfUrl": "https://localhost:44300",
    "CorsOrigins": "https://*.MyProjectName.com,https://localhost:4200"
  },
  "ConnectionStrings": {
    "Default": "Server=(LocalDb)\\MSSQLLocalDB;Database=Bookstore;Trusted_Connection=True;TrustServerCertificate=true"
  },
  "AuthServer": {
    "Authority": "https://localhost:44300",
    "RequireHttpsMetadata": true
  },
  "StringEncryption": {
    "DefaultPassPhrase": "REPLACE_WITH_GENERATED_VALUE"
  },
  "Serilog": {
    "MinimumLevel": { "Default": "Information" }
  }
}
```
Key lines: `App:CorsOrigins` is read by the CORS block in the module; `ConnectionStrings:Default` is the single name the template's `[ConnectionStringName("Default")]` DbContext expects; `AuthServer:Authority` is what OpenIddict and the JWT validator point at.
## Solution structure

```
```
Acme.Bookstore.sln
└── Acme.Bookstore/                          # the one project; both ABP module and host
    ├── Acme.Bookstore.csproj                # references ABP modules (Account, Identity,
    │                                          OpenIddict, TenantManagement, FeatureManagement,
    │                                          PermissionManagement, SettingManagement,
    │                                          BackgroundJobs, BlobStoring.Database,
    │                                          AspNetCore.Mvc.UI.Theme.LeptonX, etc.)
    ├── Program.cs                            # WebApplication host, Serilog bootstrap,
    │                                          --migrate-database handler
    ├── BookstoreModule.cs                    # the only AbpModule — wires every option
    ├── appsettings.json                      # ConnectionStrings:Default, App:CorsOrigins,
    │                                          AuthServer, StringEncryption, Serilog
    ├── appsettings.Development.json
    ├── appsettings.secrets.json              # gitignored — OpenIddict cert / encryption keys
    │
    ├── Data/                                 # data layer (no separate EF Core project)
    │   ├── BookstoreDbContext.cs             # AbpDbContext<BookstoreDbContext>
    │   ├── BookstoreDbContextFactory.cs      # IDesignTimeDbContextFactory for EF tooling
    │   ├── BookstoreDbMigrationService.cs    # invoked by --migrate-database
    │   ├── BookstoreEfCoreEntityExtensionMappings.cs
    │   └── OpenIddictDataSeedContributor.cs  # seeds default OpenIddict apps/scopes
    │
    ├── Entities/                             # domain entities and value objects
    │   └── Book.cs                           # AggregateRoot / IMultiTenant types
    │
    ├── Services/                             # application services (auto-exposed as REST)
    │   ├── BookAppService.cs                 # : ApplicationService
    │   └── Dtos/                             # request/response DTOs
    │
    ├── Pages/                                # Razor Pages root (when UI = MVC)
    │   ├── Index.cshtml(.cs)
    │   └── Components/                       # _Layout, NavMenu, etc.
    │
    ├── Permissions/
    │   ├── BookstorePermissions.cs           # const string permission names
    │   └── BookstorePermissionDefinitionProvider.cs
    │
    ├── Localization/
    │   ├── BookstoreResource.cs              # [LocalizationResourceName] marker
    │   └── Resources/Bookstore/en.json
    │
    ├── Menus/
    │   └── BookstoreMenuContributor.cs       # IMenuContributor for the main menu
    │
    ├── ObjectMapping/
    │   └── BookstoreApplicationAutoMapperProfile.cs
    │
    ├── Migrations/                           # EF Core migrations live here, in-project
    │   ├── 20260101000000_Initial.cs
    │   └── BookstoreDbContextModelSnapshot.cs
    │
    ├── Logs/                                 # populated at runtime; logs.txt (DEBUG only)
    └── wwwroot/                              # static assets, libman.json, theme bundles
```

Notes specific to single-layer:
- There is NO separate `*.DbMigrator` console project. Migration + seeding is triggered by running the host with `--migrate-database`.
- There is NO separate test project scaffolded by `app-nolayers` itself (unlike the layered template, which ships `*.Application.Tests` etc.). [uncertain] Test scaffolding may be added via a follow-up `abp add-` command or generated only when specific options are chosen.
- Folder names (`Data/`, `Entities/`, `Services/`, `Pages/`, `Migrations/`, `Permissions/`, `Localization/`, `Menus/`, `ObjectMapping/`) are conventional, not enforced — but ABP Studio's generators target them.
- Module dependencies are declared with `[DependsOn(...)]` directly on `BookstoreModule`, including UI/auth/identity/openiddict modules in one place.
```

## When to use

Choose the single-layer (app-nolayers) template when ALL of these hold:
- Solo dev or small team (~1 to 4 devs) where layer-enforced boundaries are overhead, not value.
- Project lifetime is short-to-medium (prototype, internal tool, MVP, side service) — not a 5+ year flagship.
- Domain complexity ceiling is modest: a handful of aggregate roots, no need for shared `*.Application.Contracts` consumed by other solutions.
- Single deployment target — one web host (with optional Angular/Blazor frontend) — not a fleet of services.
- You still want ABP's modules: Identity, OpenIddict, multi-tenancy, background jobs/workers, BLOB, audit logging, permissions, localization.
- You're comfortable that adding cross-cutting code (e.g., a domain service) means adding a folder/file rather than a project.
- DDD discipline is desirable but you are willing to enforce it through folder hygiene + code review rather than the C# project graph.

## When to avoid

Avoid single-layer when ANY of these are true:
- You expect to extract a microservice, mobile app backend, or shared NuGet client from this codebase later — single-layer has no `*.Application.Contracts` / `*.HttpApi.Client` projects to publish. Use the **layered** template (`references/templates/layered.md`) instead so the contracts surface is reusable from day one.
- The system will be horizontally split into many bounded contexts owned by different teams — pick the **microservice** template (`references/templates/microservice.md`).
- You need strict, compile-time enforcement that domain code cannot reference UI/EF Core types — only the layered template's project graph gives you that. In single-layer it's a folder convention.
- You plan to swap the persistence module wholesale (e.g., EF Core today, MongoDB tomorrow) — having a dedicated `*.EntityFrameworkCore` project (layered) makes the swap cleaner.
- Large codebases (15+ aggregate roots, many feature modules) — folder explosion in one project becomes harder to navigate than a layered solution.
- Migration path: there is no first-class 'convert to layered' tooling. The pragmatic path is to start a fresh layered solution and copy entities/services in. [uncertain] As of ABP 10.3 no automated single-layer-to-layered migrator ships with the CLI.

## Built-in features

- Authentication (OpenIddict): preconfigured OpenID Connect server using Volo.Abp.OpenIddict; `OpenIddictDataSeedContributor` in `Data/` seeds default applications and scopes; flow choice driven by UI (Hybrid for MVC, Authorization Code + PKCE for SPAs/Swagger, JWT bearer for API clients). Account module supplies login UI; Account Pro adds external/social logins. Deep-dive: https://abp.io/docs/10.3/solution-templates/single-layer-web-application/authentication. Default config: dev signing/encryption certs auto-generated; production requires real certs (paths configured in `PreConfigureServices`).
- Database configurations (EF Core): default DBMS SQL Server via `UseSqlServer()`; switch to PostgreSQL/MySQL/SQLite/Oracle by changing the call in `Configure<AbpDbContextOptions>`. Connection string lives at `ConnectionStrings:Default` in `appsettings.json`. Single `XxxDbContext : AbpDbContext<XxxDbContext>` in `Data/`. Migrations live in `Migrations/` inside the same project. Deep-dive: https://abp.io/docs/10.3/solution-templates/single-layer-web-application/database-configurations.
- Logging (Serilog): three sinks wired in `Program.cs` — Console, File (`Logs/logs.txt`, DEBUG builds only), and ABP Studio sink (for live monitoring). `app.UseAbpSerilogEnrichers()` adds tenant/user/correlation properties to every log event. Adjust levels via `Serilog:MinimumLevel` in `appsettings.json` or hard-code in `Program.cs`. Deep-dive: https://abp.io/docs/10.3/solution-templates/single-layer-web-application/logging.
- Swagger/OpenAPI: ABP's Swashbuckle wrapper preconfigured; UI is enabled via `app.UseAbpSwaggerUI()` and OAuth flow is wired so 'Authorize' in Swagger UI hits the local OpenIddict server. Deep-dive: https://abp.io/docs/10.3/solution-templates/single-layer-web-application/swagger-integration. [uncertain] Default Swagger UI URL is conventionally `/swagger` but the template-page summary defers details to the framework Swagger doc.
- Background jobs: `Volo.Abp.BackgroundJobs` is on by default; default store is the EF Core `AbpBackgroundJobs` table, so jobs survive restarts without extra infrastructure. Swap to Hangfire / RabbitMQ / Quartz / TickerQ via the matching `Volo.Abp.BackgroundJobs.*` integration package. Deep-dive: https://abp.io/docs/10.3/solution-templates/single-layer-web-application/background-jobs. Default config: in-process worker host runs in the same process as the web app.
- Background workers: `Volo.Abp.BackgroundWorkers` registered; pattern is `AsyncPeriodicBackgroundWorkerBase` + `await context.AddBackgroundWorkerAsync<T>()` in `OnApplicationInitializationAsync`. Deep-dive: https://abp.io/docs/10.3/solution-templates/single-layer-web-application/background-workers. Default behavior: workers run on every host instance — use distributed locking when scaling out.
- Distributed locking: `IAbpDistributedLock` is available, but the single-layer template ships only the in-process default. Add `Volo.Abp.DistributedLocking.Redis` (or another provider package) and configure it in the module before deploying multiple instances. Deep-dive: https://abp.io/docs/10.3/solution-templates/single-layer-web-application/distributed-locking.
- Multi-tenancy: opt-in at `abp new` time. When enabled, the SaaS module (Team+ license) is included for tenant management UI and per-tenant connection strings; `AbpMultiTenancyOptions.IsEnabled = true` is set via `MultiTenancyConsts`; `app.UseMultiTenancy()` is wired in. `ICurrentTenant` exposes the active tenant; `IDataFilter<IMultiTenant>` toggles the global filter. Deep-dive: https://abp.io/docs/10.3/solution-templates/single-layer-web-application/multi-tenancy.
- BLOB storing: `IBlobContainer` is wired; the default container in single-layer uses the **Database provider** (`Volo.Abp.BlobStoring.Database`), which persists blobs into the same EF Core DB. Swap to FileSystem / Azure / AWS / Aliyun / MinIO / GCS / Bunny.Net / Memory by replacing `c.UseDatabase()` with the matching `Use...()` extension in `AbpBlobStoringOptions.Containers.ConfigureDefault(...)`. Deep-dive: https://abp.io/docs/10.3/solution-templates/single-layer-web-application/blob-storing.
- CORS configuration: comma-separated origins in `App:CorsOrigins` in `appsettings.json`; consumed by the CORS block in the module; only emitted into the template when UI is Angular, Blazor WASM, or no-UI. Wildcard subdomains (`https://*.example.com`) are supported. Deep-dive: https://abp.io/docs/10.3/solution-templates/single-layer-web-application/cors-configuration.
- Db Migrator (built into host): pass `--migrate-database` (CLI: `dotnet run -- --migrate-database`) to apply EF migrations and run all `IDataSeedContributor`s (admin user `admin` / `1q2w3E*`, OpenIddict apps, default tenant). No separate `*.DbMigrator` project — unlike the layered template.
- Identity & Account modules: `Volo.Abp.Identity`, `Volo.Abp.Account` and the OpenIddict UI are referenced by the single module; user/role/permission management UI ships out of the box.
- Permission, Setting, Feature management: ABP's standard management modules are pre-wired; permission constants and providers live in `Permissions/`.
- LeptonX theme (or Basic theme): UI is themed via `Volo.Abp.AspNetCore.Mvc.UI.Theme.LeptonX` (default) — selectable at template-creation time.

## Deployment options

- - Run-from-source / Visual Studio: F5 / Ctrl+F5 inside ABP Studio's Solution Runner or Visual Studio launches the host directly; the easiest dev-time path. Default admin credentials `admin` / `1q2w3E*` after `dotnet run -- --migrate-database`.

- - IIS / Kestrel behind a reverse proxy: the project is a standard ASP.NET Core 10.0+ web app, so any production hosting that supports .NET 10 works (IIS in-process, IIS out-of-process, Nginx + Kestrel, etc.). [uncertain] The `app-nolayers` template doesn't include IIS-specific tooling beyond what `dotnet publish` produces.

- - Docker / Docker Compose: ABP Studio can generate Dockerfiles for the single-layer project, and you can compose the app + database (and Redis when distributed locking is added) in `docker-compose.yml`. [uncertain] Whether Docker assets are scaffolded automatically by `app-nolayers` versus added by ABP Studio's deployment generator depends on the version of ABP Studio in use.

- - Kubernetes / Helm: not pre-generated for single-layer (Helm charts ship with the microservice template). For single-layer, build a Docker image and write Deployment + Service manifests yourself, or use ABP Studio's deployment generator if available in your license.

- - Azure App Service / container hosting: supported as a vanilla .NET 10 web app; no template-specific tooling. Configure connection string and `App:SelfUrl` per environment.

- - .NET Aspire: ABP exposes integration with Aspire orchestration; in single-layer scenarios you typically include the host project in an Aspire AppHost. [uncertain] Whether `app-nolayers` generates an Aspire AppHost project automatically depends on options chosen at create-time and the ABP Studio version.

## Differences from other templates

Vs the **Layered (`app`) template** (`references/templates/layered.md`):
- Project count: 1 vs ~9 (Domain.Shared, Domain, Application.Contracts, Application, EntityFrameworkCore, HttpApi, HttpApi.Client, Web, DbMigrator, plus test projects).
- Boundary enforcement: folder convention vs C# project references — in single-layer, a Razor Page can `using` an entity directly; the layered template would require it to go through `*.HttpApi`/`*.Application.Contracts`.
- Reusable contracts: layered ships `*.Application.Contracts` and `*.HttpApi.Client` packages so other clients (mobile, console, microservice) can consume the API; single-layer has none of this.
- DbMigrator: layered has a dedicated `*.DbMigrator` console project; single-layer folds the same logic into the host with `--migrate-database`.
- Test scaffolding: layered ships per-layer test projects (Domain.Tests, Application.Tests, EntityFrameworkCore.Tests, etc.); single-layer ships [uncertain] little or no test scaffolding by default.
- Migration path single-layer -> layered: no automated converter; copy entities/services into a fresh layered solution. Migration path layered -> single-layer is rarely needed and likewise manual.

Vs the **Microservice template** (`references/templates/microservice.md`):
- Topology: single host vs multiple services + API gateway + identity service + admin app + public web/mobile.
- Infrastructure: microservice ships Kubernetes/Helm/Tye/Aspire orchestration, message bus (RabbitMQ), distributed cache (Redis), per-service databases. Single-layer has none of this baseline.
- Cost of entry: single-layer can be running locally in minutes; microservice template assumes container infra.
- Use single-layer when the answer to 'will this ever need to be split?' is 'no, or if it does we'll re-platform'. Use microservice when the answer is 'yes, soon, and it will be more than 2 services'.
- Migration path single-layer -> microservice: feasible by extracting modules into independent services, but requires re-introducing the module-publishing infrastructure (HttpApi.Client, contracts, distributed events) that single-layer skipped.

## Common mistakes

- Mistake: Treating `Entities/` as a separate Domain layer and importing UI/EF Core types from it.
Why wrong: In single-layer, nothing prevents the import — the compiler will not flag the violation that a layered template's project graph would. It silently turns your domain into a presentation-coupled blob.
Correct pattern: Self-discipline + code review. Keep `Entities/` referencing only `Volo.Abp.Domain.*` and your own value objects. Add a roslyn analyzer or ArchUnit-style test if it matters for your team.
- Mistake: Forgetting `--migrate-database` on first run, or running it against a config that points at a missing database.
Why wrong: The host launches, but Identity tables aren't there, login throws, and OpenIddict scopes haven't been seeded.
Correct pattern: First run is always `dotnet run -- --migrate-database` from the project folder. In CI/CD, run the same command as a deploy step (or run a one-shot pod) before scaling the web hosts.
- Mistake: Scaling the single-layer host to N>1 instances without configuring distributed locking.
Why wrong: Every replica's `BackgroundWorker`s tick on every replica — you get N duplicate runs of the same periodic job. The default `IAbpDistributedLock` is in-process only.
Correct pattern: Add `Volo.Abp.DistributedLocking.Redis` (or DB-backed equivalent), configure it in the module, and wrap each worker's `DoWorkAsync` in `await using var handle = await _lock.TryAcquireAsync(workerName);` with an early-return when `handle` is null.
- Mistake: Storing every BLOB in the default Database provider for a multi-GB upload feature.
Why wrong: Writing large binaries into a SQL Server / PostgreSQL row destroys backup/restore times, replication latency, and cache hit rates.
Correct pattern: Switch the BLOB container to FileSystem (single-host) or an object store (S3 / Azure Blob / MinIO) for large or numerous binaries; reserve the Database provider for small attachments where transactional consistency with relational data matters.
- Mistake: Hard-coding `IsEnabled` for multi-tenancy `true` in `AbpMultiTenancyOptions` after starting with it `false`.
Why wrong: Existing rows have `null` `TenantId` and the global tenant filter will hide them once enabled. Existing OpenIddict apps were seeded for the host, not for tenants.
Correct pattern: Decide multi-tenancy at `abp new` time. If you must add it later, plan a data migration that backfills `TenantId` (or explicitly assigns rows to the host tenant by leaving them null), and re-seed OpenIddict apps per tenant.
- Mistake: Pointing CORS origins at a single hard-coded host while the SPA runs on a different port in dev.
Why wrong: Browser preflight fails silently from the SPA dev server; Swagger 'Try it out' may also fail when run from a different origin.
Correct pattern: `App:CorsOrigins` is comma-separated; include both prod and dev origins (`https://*.example.com,https://localhost:4200`). Wildcard subdomains require ABP's `SetIsOriginAllowedToAllowWildcardSubdomains()` (already wired in the template).
- Mistake: Putting test projects under the host folder and accidentally publishing them.
Why wrong: `dotnet publish` will see them, increase image size, and risk shipping test fixtures with secrets.
Correct pattern: Put any `*.Tests` project at the solution root, next to (not inside) the host project. Reference the host project from the test project, not the other way around. [uncertain] As of 10.3, `app-nolayers` does not scaffold a test project automatically.
- Mistake: Editing `OpenIddictDataSeedContributor` directly to add per-environment scopes without an idempotency check.
Why wrong: `--migrate-database` re-runs all seed contributors; non-idempotent inserts will throw on the second run.
Correct pattern: Look up first (`FindByClientIdAsync`) and only `CreateAsync` when missing — follow the existing patterns in the seeded class.
- Mistake: Trying to host the single-layer app and a microservice from the same module by adding `[DependsOn(...)]` for everything.
Why wrong: Single-layer is intentionally one process; piling unrelated bounded contexts into the same module turns it into a distributed monolith with worse coupling than either template intends.
Correct pattern: When you need >1 bounded context with independent deploys, switch templates rather than overloading single-layer.
## Version pins (ABP 10.3)

- ABP 10.3 templates target .NET 10.0+ and Node 22.11+; older runtimes are unsupported.
- The single-layer template name in the CLI is `app-nolayers` (`abp new MyCompany.MyApp --template app-nolayers`). [uncertain] Earlier ABP versions used different CLI template aliases.
- Default identity provider is OpenIddict (Volo.Abp.OpenIddict). IdentityServer is no longer the default in 10.3-era templates.
- Default BLOB container provider in single-layer is the Database provider (`Volo.Abp.BlobStoring.Database`); other ABP templates may default differently.
- `IAbpDistributedLock` resolves to an in-process provider unless `Volo.Abp.DistributedLocking.<X>` is added explicitly. [uncertain] The 10.3 docs confirm the package must be added; the in-process fallback behavior is implied but not always documented on the template page.
- Background jobs default to EF Core / DB-backed storage for single-layer; Hangfire/RabbitMQ/Quartz/TickerQ are opt-in via integration packages.
- Multi-tenancy can only be cleanly enabled at create time; toggling it post-hoc requires a data migration. [uncertain] Whether ABP 10.3 ships any tooling for the post-hoc enablement is not confirmed in the consulted pages.
- [uncertain] The exact default Swagger UI URL (conventionally `/swagger`) and authentication wiring details are not on the template page; verify against `references/framework/swagger.md` before relying.
- [uncertain] Whether `app-nolayers` 10.3 scaffolds a test project, an Aspire AppHost, or Dockerfiles depends on options chosen during creation and the ABP Studio version; the template documentation pages consulted did not enumerate the artifact list explicitly.
- Default seeded admin credentials remain `admin` / `1q2w3E*`; rotate immediately after first run in any non-throwaway environment.

## Cross-references

**Phase 1 references:**
- - references/templates/layered.md — load when user asks for the layered alternative, or asks about the project graph and reusable contracts that single-layer lacks.
- - references/templates/microservice.md — load when user is debating monolith vs microservices, or asks about K8s/Helm/RabbitMQ scaffolding.
- - references/framework/architecture-ddd.md — single-layer keeps DDD building blocks (entities, repositories, app services); load when explaining how DDD applies inside a folder-organized project.
- - references/framework/modules.md — load when explaining `[DependsOn(...)]`, the AbpModule lifecycle, or how the single module wires every option.
- - references/framework/multi-tenancy.md — load for `ICurrentTenant`, tenant filter, and SaaS module deep-dive; the template page covers only the wiring side.
- - references/framework/auth-openiddict.md — load for OpenIddict configuration, signing certs, scopes, and flow choice details that the template page summarizes.
- - references/framework/background-jobs.md — load for IBackgroundJobManager, queue providers (Hangfire, RabbitMQ, Quartz, TickerQ), and durability semantics.
- - references/framework/background-workers.md — load for AsyncPeriodicBackgroundWorkerBase, IBackgroundWorkerManager, and lifetime hooks.
- - references/framework/distributed-locking.md — load when scaling beyond one instance; covers Redis provider configuration that single-layer omits by default.
- - references/framework/blob-storing.md — load for the full provider matrix (FileSystem/Azure/AWS/Aliyun/MinIO/GCS/Bunny.Net/Memory) and per-container configuration.
- - references/framework/data-seeding.md — load when extending `OpenIddictDataSeedContributor` or adding new `IDataSeedContributor` implementations.
- - references/framework/ef-core.md — load for AbpDbContext<T>, ConfigureByConvention(), repositories, and migration tooling specifics.
- - references/framework/swagger.md — load for Swagger UI URL, OAuth integration, NSwag vs Swashbuckle choice.

**External docs:**
- - ASP.NET Core (Microsoft) — https://learn.microsoft.com/aspnet/core — base host, configuration, middleware pipeline that the single-layer template builds on.
- - Entity Framework Core — https://learn.microsoft.com/ef/core — migrations, fluent mapping, provider list (SQL Server / Npgsql / MySQL / SQLite / Oracle) used in `Configure<AbpDbContextOptions>`.
- - OpenIddict — https://documentation.openiddict.com — underlying OpenID Connect server library; relevant for advanced flow customization, certificate management, and grant types.
- - Microsoft.AspNetCore.Identity — https://learn.microsoft.com/aspnet/core/security/authentication/identity — referenced because ABP's Identity module wraps it; useful when extending user/role schemas.
- - Serilog — https://serilog.net — sink ecosystem (Seq, Elasticsearch, Application Insights, OTLP) you can plug into the template's `Program.cs` Serilog setup.
- - Autofac — https://autofac.readthedocs.io — DI container used in place of the default Microsoft container; relevant when registering with assembly scanning or property injection.
- - Swashbuckle.AspNetCore — https://github.com/domaindrivendev/Swashbuckle.AspNetCore — underlying Swagger generator; relevant when customizing schema filters or operation filters.
- - .NET 10 — https://learn.microsoft.com/dotnet/core/whats-new — runtime baseline for ABP 10.3 templates.
- - StackExchange Redis — https://stackexchange.github.io/StackExchange.Redis — used by Volo.Abp.DistributedLocking.Redis when you scale single-layer horizontally.

## Sources (ABP 10.3)

- [https://abp.io/docs/10.3/solution-templates/single-layer-web-application — main landing page; what the template is, target use cases, included features, and primary disclaimers about when not to use it.](https://abp.io/docs/10.3/solution-templates/single-layer-web-application — main landing page; what the template is, target use cases, included features, and primary disclaimers about when not to use it.)
- [https://abp.io/docs/10.3/solution-templates/single-layer-web-application/overview — overview page describing pre-installed components (Autofac DI, Serilog, Swagger, OpenIddict) and supported UI/database matrix.](https://abp.io/docs/10.3/solution-templates/single-layer-web-application/overview — overview page describing pre-installed components (Autofac DI, Serilog, Swagger, OpenIddict) and supported UI/database matrix.)
- [https://abp.io/docs/10.3/solution-templates/single-layer-web-application/solution-structure — folder-based layout (Data/, Entities/, Services/, Pages/, Migrations/, Permissions/, Localization/, Menus/, ObjectMapping/) and the single-project model.](https://abp.io/docs/10.3/solution-templates/single-layer-web-application/solution-structure — folder-based layout (Data/, Entities/, Services/, Pages/, Migrations/, Permissions/, Localization/, Menus/, ObjectMapping/) and the single-project model.)
- [https://abp.io/docs/10.3/solution-templates/single-layer-web-application/main-components — main components: web app project, Db Migrator command-line option, integrated DDD building blocks.](https://abp.io/docs/10.3/solution-templates/single-layer-web-application/main-components — main components: web app project, Db Migrator command-line option, integrated DDD building blocks.)
- [https://abp.io/docs/10.3/solution-templates/single-layer-web-application/authentication — OpenIddict-based authentication, Account module, OpenIddictDataSeedContributor, flow choices per UI type.](https://abp.io/docs/10.3/solution-templates/single-layer-web-application/authentication — OpenIddict-based authentication, Account module, OpenIddictDataSeedContributor, flow choices per UI type.)
- [https://abp.io/docs/10.3/solution-templates/single-layer-web-application/database-configurations — connection-string location in appsettings.json (Default key), DbContext naming, switching providers via UseSqlServer/UseNpgsql/UseMySql/UseSqlite.](https://abp.io/docs/10.3/solution-templates/single-layer-web-application/database-configurations — connection-string location in appsettings.json (Default key), DbContext naming, switching providers via UseSqlServer/UseNpgsql/UseMySql/UseSqlite.)
- [https://abp.io/docs/10.3/solution-templates/single-layer-web-application/logging — Serilog configuration with Console, File (logs.txt under Logs/), and ABP Studio sinks; UseAbpSerilogEnrichers middleware.](https://abp.io/docs/10.3/solution-templates/single-layer-web-application/logging — Serilog configuration with Console, File (logs.txt under Logs/), and ABP Studio sinks; UseAbpSerilogEnrichers middleware.)
- [https://abp.io/docs/10.3/solution-templates/single-layer-web-application/swagger-integration — confirms Swagger ships built-in; details deferred to framework Swagger doc.](https://abp.io/docs/10.3/solution-templates/single-layer-web-application/swagger-integration — confirms Swagger ships built-in; details deferred to framework Swagger doc.)
- [https://abp.io/docs/10.3/solution-templates/single-layer-web-application/background-jobs — background-jobs module is wired in; Hangfire/RabbitMQ/Quartz/TickerQ available as alternative providers.](https://abp.io/docs/10.3/solution-templates/single-layer-web-application/background-jobs — background-jobs module is wired in; Hangfire/RabbitMQ/Quartz/TickerQ available as alternative providers.)
- [https://abp.io/docs/10.3/solution-templates/single-layer-web-application/background-workers — AsyncPeriodicBackgroundWorkerBase pattern and AddBackgroundWorkerAsync registration in OnApplicationInitializationAsync.](https://abp.io/docs/10.3/solution-templates/single-layer-web-application/background-workers — AsyncPeriodicBackgroundWorkerBase pattern and AddBackgroundWorkerAsync registration in OnApplicationInitializationAsync.)
- [https://abp.io/docs/10.3/solution-templates/single-layer-web-application/distributed-locking — IAbpDistributedLock usage; Volo.Abp.DistributedLocking package must be added explicitly; Redis as default backing store.](https://abp.io/docs/10.3/solution-templates/single-layer-web-application/distributed-locking — IAbpDistributedLock usage; Volo.Abp.DistributedLocking package must be added explicitly; Redis as default backing store.)
- [https://abp.io/docs/10.3/solution-templates/single-layer-web-application/multi-tenancy — SaaS module, ICurrentTenant, DataFilter to bypass tenant isolation, per-tenant connection-string management.](https://abp.io/docs/10.3/solution-templates/single-layer-web-application/multi-tenancy — SaaS module, ICurrentTenant, DataFilter to bypass tenant isolation, per-tenant connection-string management.)
- [https://abp.io/docs/10.3/solution-templates/single-layer-web-application/blob-storing — IBlobContainer; default provider in single-layer is the Database Provider; full provider list (FileSystem, Azure, AWS, Aliyun, MinIO, GCS, Bunny.Net, Memory).](https://abp.io/docs/10.3/solution-templates/single-layer-web-application/blob-storing — IBlobContainer; default provider in single-layer is the Database Provider; full provider list (FileSystem, Azure, AWS, Aliyun, MinIO, GCS, Bunny.Net, Memory).)
- [https://abp.io/docs/10.3/solution-templates/single-layer-web-application/cors-configuration — App:CorsOrigins in appsettings.json; only relevant for Angular, Blazor WASM, or no-UI deployments.](https://abp.io/docs/10.3/solution-templates/single-layer-web-application/cors-configuration — App:CorsOrigins in appsettings.json; only relevant for Angular, Blazor WASM, or no-UI deployments.)
- [https://abp.io/docs/10.3/get-started/single-layer-web-application — getting-started flow; CLI template name 'app-nolayers'; default admin credentials admin / 1q2w3E*; UI/DB matrix; .NET 10.0+ / Node 22.11+ prerequisites.](https://abp.io/docs/10.3/get-started/single-layer-web-application — getting-started flow; CLI template name 'app-nolayers'; default admin credentials admin / 1q2w3E*; UI/DB matrix; .NET 10.0+ / Node 22.11+ prerequisites.)

Last verified: 2026-05-10
