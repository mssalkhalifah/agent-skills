# Adding `Acme.Billing` to a Layered modular monolith — EF Core wiring (ABP 10.3, PostgreSQL)

This is the framework guardrail at stake: in a multi-module ABP solution, **every module owns its own `DbContext`, its own `Migrations/` folder, its own `IDesignTimeDbContextFactory<T>`, its own `IXxxDbSchemaMigrator` implementation, and — critically — its own EF migrations history table.** Sharing the default `__EFMigrationsHistory` across modules makes EF think the other modules' migrations are "missing" and re-apply them; sharing a `DbContext` across modules destroys the modular boundary and lets one module trip another's migrations on every `dotnet ef migrations add`.

The layout below assumes your Billing module sits alongside the existing two — either inside `modules/acme.billing/` (typical for a modular monolith) or as a sibling solution folder pulled in via project reference. The shape is the same either way.

## 1. `BillingDbContext` — module-scoped, one connection-string name, only Billing aggregates

`src/Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/BillingDbContext.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Volo.Abp.Data;
using Volo.Abp.EntityFrameworkCore;

namespace Acme.Billing.EntityFrameworkCore;

[ConnectionStringName(BillingDbProperties.ConnectionStringName)]
public class BillingDbContext :
    AbpDbContext<BillingDbContext>,
    IBillingDbContext
{
    public DbSet<Invoice> Invoices { get; set; }
    public DbSet<Subscription> Subscriptions { get; set; }
    // ...only Billing aggregate roots — NEVER expose other modules' DbSets here

    public BillingDbContext(DbContextOptions<BillingDbContext> options)
        : base(options) { }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);
        builder.ConfigureBilling();   // extension method that maps Billing entities
    }
}
```

And the per-module DbContext interface (best practice — repositories bind to the interface, not the concrete type), in the same folder:

```csharp
[ConnectionStringName(BillingDbProperties.ConnectionStringName)]
public interface IBillingDbContext : IEfCoreDbContext
{
    DbSet<Invoice> Invoices { get; }
    DbSet<Subscription> Subscriptions { get; }
}
```

`BillingDbProperties` (in `Acme.Billing.Domain`) carries the constants:

```csharp
public static class BillingDbProperties
{
    public const string ConnectionStringName = "Billing"; // or "Default" if you want one DB
    public const string DbTablePrefix = "App";
    public const string DbSchema = null; // PostgreSQL uses search_path; leave null
}
```

Entity mappings go in a `ConfigureBilling` extension method (not in `OnModelCreating` directly — that pattern is forbidden by the EF-core best-practice doc; it makes the DbContext un-shareable and forces every consumer to copy your config). Every `Entity<T>(...)` block must call `b.ConfigureByConvention()` so audit columns, soft-delete, multi-tenancy filter, and `ExtraProperties` map correctly.

## 2. `IDesignTimeDbContextFactory<BillingDbContext>` — required for `dotnet ef`

Next to `BillingDbContext.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Design;
using Microsoft.Extensions.Configuration;

namespace Acme.Billing.EntityFrameworkCore;

public class BillingDbContextFactory : IDesignTimeDbContextFactory<BillingDbContext>
{
    public BillingDbContext CreateDbContext(string[] args)
    {
        // PostgreSQL + ABP audit columns store DateTime, not DateTimeOffset.
        // Npgsql 6+ requires this switch or you'll get UTC-Kind errors at runtime.
        AppContext.SetSwitch("Npgsql.EnableLegacyTimestampBehavior", true);

        var configuration = BuildConfiguration();

        var builder = new DbContextOptionsBuilder<BillingDbContext>()
            .UseNpgsql(
                configuration.GetConnectionString("Billing")
                    ?? configuration.GetConnectionString("Default"),
                b => b.MigrationsHistoryTable("__EFMigrationsHistory_Billing")
            );

        return new BillingDbContext(builder.Options);
    }

    private static IConfigurationRoot BuildConfiguration() =>
        new ConfigurationBuilder()
            // Climb to the host's appsettings.json — adjust the relative path
            // to wherever the DbMigrator or Web host lives.
            .SetBasePath(Path.Combine(
                Directory.GetCurrentDirectory(),
                "../../../../host/Acme.YourSolution.DbMigrator/"))
            .AddJsonFile("appsettings.json", optional: false)
            .Build();
}
```

The `MigrationsHistoryTable("__EFMigrationsHistory_Billing")` here **must match** the runtime registration (next section). Mismatch is a silent foot-gun: `dotnet ef migrations add` writes to one history table while the host reads from another, so EF re-applies migrations on every startup.

Without this factory, `dotnet ef migrations add ...` falls back to host-level DbContext discovery, which doesn't see the module's private options — you'll generate the wrong DbContext (or none).

## 3. `AddAbpDbContext` + `UseNpgsql` with the per-module history table

`src/Acme.Billing.EntityFrameworkCore/BillingEntityFrameworkCoreModule.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Volo.Abp.EntityFrameworkCore;
using Volo.Abp.EntityFrameworkCore.PostgreSql;
using Volo.Abp.Modularity;

namespace Acme.Billing;

[DependsOn(
    typeof(AbpEntityFrameworkCoreModule),
    typeof(AbpEntityFrameworkCorePostgreSqlModule), // <-- Postgres provider module
    typeof(BillingDomainModule)
)]
public class BillingEntityFrameworkCoreModule : AbpModule
{
    public override void PreConfigureServices(ServiceConfigurationContext context)
    {
        // Required by Npgsql 6+ for ABP audit columns (DateTime, not DateTimeOffset).
        AppContext.SetSwitch("Npgsql.EnableLegacyTimestampBehavior", true);

        // Map any object-extension extra properties here, before model build.
        BillingEfCoreEntityExtensionMappings.Configure();
    }

    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        context.Services.AddAbpDbContext<BillingDbContext>(options =>
        {
            // Bind repositories to the interface so a consolidated host DbContext
            // can [ReplaceDbContext(typeof(IBillingDbContext))] later.
            options.AddDefaultRepositories<IBillingDbContext>(includeAllEntities: true);
            // options.AddRepository<Invoice, InvoiceRepository>(); // custom repo, if any
        });

        Configure<AbpDbContextOptions>(options =>
        {
            options.Configure<BillingDbContext>(c =>
            {
                c.UseNpgsql(b =>
                    b.MigrationsHistoryTable("__EFMigrationsHistory_Billing"));
            });
        });
    }
}
```

Key points:

- **`AddAbpDbContext` — never `AddDbContext`.** The plain EF Core registration skips ABP's UoW pipeline, audit setters, change-event publishing, and connection-string resolver — soft-delete, multi-tenancy, and audit silently break.
- **`AbpEntityFrameworkCorePostgreSqlModule` in `[DependsOn]`.** Don't carry the SqlServer module — the NuGet package is `Volo.Abp.EntityFrameworkCore.PostgreSql`.
- **`AddDefaultRepositories<IBillingDbContext>`** binds the repository registrations to the *interface*, so a consolidated host DbContext can later `[ReplaceDbContext(typeof(IBillingDbContext))]` without code changes.
- **`MigrationsHistoryTable("__EFMigrationsHistory_Billing")`** matches the design-time factory exactly. Pick a unique suffix per module (`_Billing`, `_Forms`, `_Sales`, etc.).

## 4. `IBillingDbSchemaMigrator` + EF Core implementation using `IDbContextProvider`

The schema-migrator pair is how the host's `DbMigratorHostedService` (in `Acme.YourSolution.DbMigrator`) applies each module's migrations: it resolves every `IXxxDbSchemaMigrator` registered in DI and calls `MigrateAsync()` on each. Adding a new module = shipping a new pair.

Provider-agnostic contract in **`Acme.Billing.Domain/Data/IBillingDbSchemaMigrator.cs`**:

```csharp
namespace Acme.Billing.Data;

public interface IBillingDbSchemaMigrator
{
    Task MigrateAsync();
}
```

EF Core implementation in **`Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/EntityFrameworkCoreBillingDbSchemaMigrator.cs`**:

```csharp
using Microsoft.EntityFrameworkCore;
using Volo.Abp.DependencyInjection;
using Volo.Abp.EntityFrameworkCore;
using Acme.Billing.Data;

namespace Acme.Billing.EntityFrameworkCore;

public class EntityFrameworkCoreBillingDbSchemaMigrator
    : IBillingDbSchemaMigrator, ITransientDependency
{
    private readonly IDbContextProvider<BillingDbContext> _dbContextProvider;

    public EntityFrameworkCoreBillingDbSchemaMigrator(
        IDbContextProvider<BillingDbContext> dbContextProvider)
    {
        _dbContextProvider = dbContextProvider;
    }

    public async Task MigrateAsync()
    {
        var dbContext = await _dbContextProvider.GetDbContextAsync();
        await dbContext.Database.MigrateAsync();
    }
}
```

Why `IDbContextProvider<BillingDbContext>` rather than `BillingDbContext` directly: the provider honours connection-string resolution (per-tenant connection strings, `[ReplaceDbContext]` consolidation) and runs inside the active Unit of Work — so the same migrator works whether the module runs as its own DbContext today or gets consolidated into a host DbContext tomorrow.

`ITransientDependency` ensures the migrator is auto-registered; the host's existing migration service discovers it by injecting `IEnumerable<I*DbSchemaMigrator>` or by resolving each one by name from the migration loop. If your host pattern is "one migration call per module," add `await _billingDbSchemaMigrator.MigrateAsync();` to it; if it's the iterate-everything pattern, no host code changes are needed.

## 5. Separate `Migrations/` folder per module

The `dotnet ef migrations add` command for Billing **must** target the Billing EF Core project as `--project` and (typically) the host `Acme.YourSolution.DbMigrator` as `--startup-project`. The generated files land under `src/Acme.Billing.EntityFrameworkCore/Migrations/` — completely separate from `Acme.Sales.EntityFrameworkCore/Migrations/` and `Acme.Identity.EntityFrameworkCore/Migrations/`.

```bash
# Run from the solution root:
dotnet ef migrations add Initial_Billing \
    --project src/Acme.Billing.EntityFrameworkCore \
    --startup-project host/Acme.YourSolution.DbMigrator \
    --context BillingDbContext \
    --output-dir Migrations
```

Notes:

- `--context BillingDbContext` is mandatory once the host references multiple DbContexts; otherwise EF picks one at random (or errors out).
- `--output-dir Migrations` keeps each module's snapshots inside its own EF Core project — never let two modules write into a shared folder; their snapshots will overwrite each other.
- Apply via the host's `DbMigrator` console app (`dotnet run --project host/Acme.YourSolution.DbMigrator`), not `dotnet ef database update`. The DbMigrator runs every module's `IXxxDbSchemaMigrator` in order and triggers `IDataSeedContributor` afterwards — a raw `database update` skips the seeders.

## 6. Connection-string entry

In `host/Acme.YourSolution.DbMigrator/appsettings.json` and the Web/HttpApi.Host's `appsettings.json` (keep them in lockstep):

```json
{
  "ConnectionStrings": {
    "Default":  "Host=localhost;Port=5432;Database=AcmeApp;Username=postgres;Password=...",
    "Billing":  "Host=localhost;Port=5432;Database=AcmeApp;Username=postgres;Password=..."
  }
}
```

You have two valid choices:

- **One physical database, one connection string.** Set `BillingDbProperties.ConnectionStringName = "Default"` (or omit the `"Billing"` entry — ABP's `IConnectionStringResolver` falls back to `"Default"` when the named string is missing). All three modules share a Postgres database; the `__EFMigrationsHistory_<Module>` table per module keeps their migration trails separate. **This is the common modular-monolith choice.**
- **Per-module physical databases.** Give Billing its own `"Billing"` connection string pointing at a different DB. The history-table naming still matters but no longer prevents the cross-module collision (different DBs entirely).

## 7. Warning — do NOT share a DbContext across modules

It is tempting in a modular monolith to "just put the Billing entities in the existing host DbContext" and skip steps 1–4. **Do not.** Concrete consequences:

- **Migration cross-contamination.** A `dotnet ef migrations add` for any module's entity changes the *shared* DbContext's snapshot. The next module to add a migration regenerates the world, so you ship Billing changes inside a Sales migration (and vice versa). Code review can't catch this — the diff just looks "right."
- **History-table conflict.** A single `__EFMigrationsHistory` table records every module's migrations together. Deploying just Billing means EF sees Sales's history rows referencing a snapshot that no longer matches the running model and treats Sales as "out of date" — it will try to re-apply or re-revert Sales migrations against a database it shouldn't be touching.
- **Boundary collapse.** A shared DbContext means every module's repositories inject the same physical type. Cross-module reads stop going through `[IntegrationService]` (Rule 5) because the DbContext is right there — `IRepository<Invoice, Guid>` resolves cleanly inside `Sales.Application`, and the modular seam exists only on paper. Future extraction to microservices becomes a rewrite, not a refactor.
- **`[ReplaceDbContext]` is the *only* legitimate consolidation path.** If you genuinely want a single physical DbContext at runtime (typical for the layered template's *host* DbContext consuming pre-built ABP modules like Identity / OpenIddict / SaaS), define each module's `IXxxDbContext` interface as in Step 1, then put `[ReplaceDbContext(typeof(IBillingDbContext))]` on the host DbContext and have it implement `IBillingDbContext`. Repositories injected against `IBillingDbContext` resolve to the host DbContext at runtime, but at design time each module still owns its own `BillingDbContext` + `Migrations/` + history table. The boundary stays intact; the runtime tables coalesce.

## Summary — files you create per module

| Path (inside the Billing module) | Purpose |
|----------------------------------|---------|
| `Acme.Billing.Domain/Data/IBillingDbSchemaMigrator.cs` | Provider-agnostic migration contract |
| `Acme.Billing.Domain/Data/BillingDbProperties.cs` | `ConnectionStringName`, `DbTablePrefix`, `DbSchema` constants |
| `Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/BillingDbContext.cs` | `AbpDbContext<BillingDbContext>` + `[ConnectionStringName]`, Billing aggregates only |
| `Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/IBillingDbContext.cs` | Per-module interface (`IEfCoreDbContext`) for `[ReplaceDbContext]`-friendly repository binding |
| `Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/BillingDbContextModelCreatingExtensions.cs` | `public static void ConfigureBilling(this ModelBuilder builder)` — entity mappings live here, not in `OnModelCreating` |
| `Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/BillingDbContextFactory.cs` | `IDesignTimeDbContextFactory<BillingDbContext>` with `UseNpgsql(... MigrationsHistoryTable("__EFMigrationsHistory_Billing"))` |
| `Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/EntityFrameworkCoreBillingDbSchemaMigrator.cs` | `IBillingDbSchemaMigrator` impl using `IDbContextProvider<BillingDbContext>` |
| `Acme.Billing.EntityFrameworkCore/BillingEntityFrameworkCoreModule.cs` | `[DependsOn(... AbpEntityFrameworkCorePostgreSqlModule)]`, `AddAbpDbContext`, `UseNpgsql` + matching `MigrationsHistoryTable` |
| `Acme.Billing.EntityFrameworkCore/Migrations/` | EF-generated migrations (one folder per module, never shared) |

With this in place, `dotnet ef migrations add` for Billing only touches `Acme.Billing.EntityFrameworkCore/Migrations/` and writes only to `__EFMigrationsHistory_Billing` at apply-time — the other two modules' migrations and history stay completely untouched.
