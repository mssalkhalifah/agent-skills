# Adding `Acme.Billing` — EF Core wiring on a Layered modular monolith (ABP 10.3, PostgreSQL)

Every internal ABP module in a multi-module Layered solution **owns its persistence stack**. That means `Acme.Billing` ships its own `DbContext`, its own `IDesignTimeDbContextFactory`, its own `Migrations/` folder, its own `__EFMigrationsHistory_*` table, and its own `IBillingDbSchemaMigrator`. Do not extend the host's `DbContext` and do not share a `DbContext` with the other two modules — that's the load-bearing rule that keeps `dotnet ef migrations add` from re-applying or reverting another module's migrations.

The other two existing modules already follow this pattern (assumed). Repeat the same shape for `Acme.Billing` and only the Postgres provider details below.

---

## 1. Project layout for the new module

Mirror the layered conventions for the new module folder (assuming a `modules/` subfolder per module — the exact path depends on your repo, but the project shape is fixed):

```
modules/billing/
  src/
    Acme.Billing.Domain.Shared/
    Acme.Billing.Domain/
      Data/
        IBillingDbSchemaMigrator.cs                ← provider-agnostic contract
    Acme.Billing.Application.Contracts/
    Acme.Billing.Application/
    Acme.Billing.EntityFrameworkCore/
      EntityFrameworkCore/
        BillingDbContext.cs                        ← AbpDbContext<BillingDbContext>
        IBillingDbContext.cs                       ← optional but recommended
        BillingDbContextFactory.cs                 ← IDesignTimeDbContextFactory
        BillingModelBuilderExtensions.cs           ← static ConfigureBilling(ModelBuilder)
        EntityFrameworkCoreBillingDbSchemaMigrator.cs
      Migrations/                                  ← OWNED by this module only
      BillingEntityFrameworkCoreModule.cs
    Acme.Billing.HttpApi/
    Acme.Billing.HttpApi.Client/
  test/...
```

The `Migrations/` folder must be **per module**; never point `dotnet ef migrations add` at a shared output folder. Same rule for the history table.

---

## 2. NuGet / `[DependsOn]` chain

`Acme.Billing.EntityFrameworkCore.csproj` references:

- `Volo.Abp.EntityFrameworkCore` (10.3.x)
- `Volo.Abp.EntityFrameworkCore.PostgreSql` (10.3.x)

`BillingEntityFrameworkCoreModule` depends on:

- `AbpEntityFrameworkCoreModule`
- `AbpEntityFrameworkCorePostgreSqlModule`
- `BillingDomainModule`

---

## 3. `BillingDbContext.cs` — the DbContext

`src/Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/BillingDbContext.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Volo.Abp.Data;
using Volo.Abp.EntityFrameworkCore;

namespace Acme.Billing.EntityFrameworkCore;

[ConnectionStringName(BillingDbProperties.ConnectionStringName)]    // "Billing"
public class BillingDbContext : AbpDbContext<BillingDbContext>, IBillingDbContext
{
    public DbSet<Invoice>      Invoices      { get; set; }
    public DbSet<Subscription> Subscriptions { get; set; }
    // ...aggregate-root DbSets only...

    public BillingDbContext(DbContextOptions<BillingDbContext> options) : base(options) { }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);
        builder.ConfigureBilling();                                 // static extension; see below
    }
}
```

A separate `BillingDbProperties` (in `Acme.Billing.Domain.Shared`) holds `public const string DbTablePrefix = "Bil";`, `DbSchema = null;`, `ConnectionStringName = "Billing";`. The `[ConnectionStringName]` attribute lets ops point this module at a separate database later without code changes — falls back to `"Default"` if no `Billing` entry is configured.

The model configuration lives in a static extension (mandatory pattern in ABP — `OnModelCreating` is forbidden to contain inline config because shared/modular DbContexts would have to copy it):

```csharp
// BillingModelBuilderExtensions.cs
public static class BillingModelBuilderExtensions
{
    public static void ConfigureBilling(this ModelBuilder builder)
    {
        builder.Entity<Invoice>(b =>
        {
            b.ToTable(BillingDbProperties.DbTablePrefix + "Invoices",
                      BillingDbProperties.DbSchema);
            b.ConfigureByConvention();                              // ALWAYS first line
            b.Property(x => x.Number).IsRequired().HasMaxLength(64);
        });
        // ...
    }
}
```

`ConfigureByConvention()` is required on every `Entity<T>(...)` block — it maps `Id`, `ConcurrencyStamp`, audit columns, `ExtraProperties`, `IsDeleted`, `TenantId`, and applies the soft-delete + multi-tenancy global query filters.

---

## 4. `BillingEntityFrameworkCoreModule.cs` — `AddAbpDbContext` + `UseNpgsql` + history table

This is where the Postgres-specific wiring and the unique migrations-history-table name live:

```csharp
using Microsoft.EntityFrameworkCore;
using Volo.Abp.Modularity;
using Volo.Abp.EntityFrameworkCore;
using Volo.Abp.EntityFrameworkCore.PostgreSql;

namespace Acme.Billing.EntityFrameworkCore;

[DependsOn(
    typeof(BillingDomainModule),
    typeof(AbpEntityFrameworkCoreModule),
    typeof(AbpEntityFrameworkCorePostgreSqlModule)
)]
public class BillingEntityFrameworkCoreModule : AbpModule
{
    public override void PreConfigureServices(ServiceConfigurationContext context)
    {
        // Required for ABP audit columns (DateTime, not DateTimeOffset) on Npgsql 6+
        AppContext.SetSwitch("Npgsql.EnableLegacyTimestampBehavior", true);
    }

    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        context.Services.AddAbpDbContext<BillingDbContext>(options =>
        {
            options.AddDefaultRepositories(includeAllEntities: true);
            // options.AddRepository<Invoice, InvoiceRepository>();  // when you write a custom one
        });

        Configure<AbpDbContextOptions>(options =>
        {
            options.Configure<BillingDbContext>(c =>
            {
                c.UseNpgsql(b => b.MigrationsHistoryTable("__EFMigrationsHistory_Billing"));
            });
        });
    }
}
```

Two non-negotiables here:

1. **`MigrationsHistoryTable("__EFMigrationsHistory_Billing")`** — without this, EF writes to the default `__EFMigrationsHistory` table that the other two modules also write to. On the next host start, EF compares the migrations baked into `BillingDbContext` against rows it sees from the *other* modules and treats them as missing — it will try to re-apply them, fail on already-existing tables (or silently revert), and corrupt the migration trail. The naming convention `__EFMigrationsHistory_<Module>` keeps each module's bookkeeping isolated even when all three modules share one physical database.
2. **`AddAbpDbContext` (not `AddDbContext`)** — the plain Microsoft registration skips ABP's UnitOfWork pipeline, audit property setter, entity change events, soft-delete/multi-tenant filters, and connection-string resolver.

`Configure<AbpDbContextOptions>(options.Configure<BillingDbContext>(...))` is the per-context override form — it only affects `BillingDbContext`, not the host's or the other modules' contexts.

---

## 5. `BillingDbContextFactory.cs` — `IDesignTimeDbContextFactory`

`dotnet ef migrations add` does not boot the full ABP host. Without a design-time factory, EF tooling cannot construct `BillingDbContext` and falls back to host-level discovery, which doesn't see this module's private DI configuration. **Every module needs its own factory.**

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Design;
using Microsoft.Extensions.Configuration;

namespace Acme.Billing.EntityFrameworkCore;

public class BillingDbContextFactory : IDesignTimeDbContextFactory<BillingDbContext>
{
    public BillingDbContext CreateDbContext(string[] args)
    {
        // Match the runtime PreConfigureServices switch — design-time runs in its own process
        AppContext.SetSwitch("Npgsql.EnableLegacyTimestampBehavior", true);

        var configuration = BuildConfiguration();

        var builder = new DbContextOptionsBuilder<BillingDbContext>()
            .UseNpgsql(
                configuration.GetConnectionString("Billing")        // falls back to "Default" if you prefer
                    ?? configuration.GetConnectionString("Default"),
                b => b.MigrationsHistoryTable("__EFMigrationsHistory_Billing")    // MUST match the runtime call
            );

        return new BillingDbContext(builder.Options);
    }

    private static IConfigurationRoot BuildConfiguration()
        => new ConfigurationBuilder()
            // Point at whichever appsettings.json holds the dev connection string —
            // typically the host's DbMigrator project. Adjust the relative path
            // to your repo's modules/ layout.
            .SetBasePath(Path.Combine(Directory.GetCurrentDirectory(),
                "../../../../../src/Acme.YourHost.DbMigrator/"))
            .AddJsonFile("appsettings.json", optional: false)
            .Build();
}
```

The `MigrationsHistoryTable` argument **must** match the runtime registration byte-for-byte. A mismatch is a silent foot-gun: `dotnet ef migrations add` writes to one history table, the runtime host reads from another, and on next startup ABP thinks every migration is missing.

Generate the first migration with the project's startup explicitly set to its own module:

```
dotnet ef migrations add Initial_Billing \
  --project    modules/billing/src/Acme.Billing.EntityFrameworkCore \
  --startup-project modules/billing/src/Acme.Billing.EntityFrameworkCore \
  --output-dir Migrations
```

Subsequent migrations land in the same `modules/billing/src/Acme.Billing.EntityFrameworkCore/Migrations/` folder — never mix module migrations.

---

## 6. `IBillingDbSchemaMigrator` + EF implementation

The schema migrator is the provider-agnostic seam the host's `DbMigrator` console app calls to bring each module's schema up to date. The contract lives in the **Domain** project so the EF-Core assembly is not leaked to the host; the implementation lives in the **EntityFrameworkCore** project.

`src/Acme.Billing.Domain/Data/IBillingDbSchemaMigrator.cs`:

```csharp
namespace Acme.Billing.Data;

public interface IBillingDbSchemaMigrator
{
    Task MigrateAsync();
}
```

`src/Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/EntityFrameworkCoreBillingDbSchemaMigrator.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Volo.Abp.DependencyInjection;
using Volo.Abp.EntityFrameworkCore;
using Acme.Billing.Data;

namespace Acme.Billing.EntityFrameworkCore;

public class EntityFrameworkCoreBillingDbSchemaMigrator
    : IBillingDbSchemaMigrator, ITransientDependency
{
    private readonly IServiceProvider _serviceProvider;

    public EntityFrameworkCoreBillingDbSchemaMigrator(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public async Task MigrateAsync()
    {
        // Resolve the DbContext through ABP's IDbContextProvider so the connection
        // string, replace-DbContext, and tenant scoping all flow through correctly.
        var dbContext = await _serviceProvider
            .GetRequiredService<IDbContextProvider<BillingDbContext>>()
            .GetDbContextAsync();

        await dbContext.Database.MigrateAsync();
    }
}
```

Then the host's `DbMigrator` (or `DbMigratorHostedService`) calls `IBillingDbSchemaMigrator.MigrateAsync()` alongside its existing calls for the two other modules. Each module's schema migrator independently runs `Database.MigrateAsync()` against its own context, against its own history table — they never collide.

If you also seed data in this module, implement `IDataSeedContributor` in the same project and ABP's `IDataSeeder` will pick it up; that's separate from schema migration.

---

## 7. Host-side wiring

In your host's `EntityFrameworkCore` module (the layered solution's main `<Project>.EntityFrameworkCore` module), add `BillingEntityFrameworkCoreModule` to the `[DependsOn]` graph. In the host's `appsettings.json` (and the `.DbMigrator` project's `appsettings.json` — **both must stay in lockstep**), add the connection string:

```json
{
  "ConnectionStrings": {
    "Default":  "Host=localhost;Port=5432;Database=AcmeApp;Username=postgres;Password=...",
    "Billing":  "Host=localhost;Port=5432;Database=AcmeApp;Username=postgres;Password=..."
  }
}
```

If `Billing` is absent, ABP's `IConnectionStringResolver` falls back to `Default` — same physical DB, but the history-table isolation still keeps the three module migration trails apart. Later, ops can split `Billing` out to a different database with no code change.

---

## 8. Warning: do not share a DbContext across modules

It is tempting in a modular monolith to point all three modules at one giant `AppDbContext` to "simplify migrations." Don't.

- **Migration ownership collapses.** Every entity change in any module triggers a migration in the shared context, and every team has to coordinate. The whole point of `Acme.Billing` shipping as a module is that its persistence evolves independently.
- **`[ConnectionStringName]` is moot.** A shared DbContext can only point at one connection string at a time, so you lose the ability to split the billing schema onto its own database later.
- **Boundary leaks.** Whoever owns the shared `DbContext` ends up importing every module's entity types into their assembly — the same trap as `[DependsOn]`-ing a producer's full `*.Application` instead of `*.Application.Contracts`. The seam stops being real.
- **The `[ReplaceDbContext]` pattern is the right tool only for pre-built ABP modules** (Identity, OpenIddict, TenantManagement, etc.) where you intentionally want the host to provide a single consolidated DbContext that implements those modules' `IXxxDbContext` interfaces. It is **not** the pattern for your own first-party modules in a modular monolith — those should each own their concrete `DbContext`.

The right setup for `Acme.Billing` plus two existing modules is: three concrete DbContexts, three `Migrations/` folders, three `__EFMigrationsHistory_<Module>` tables, three `IXxxDbSchemaMigrator` implementations, three calls in `DbMigrator`. They can all point at the same physical Postgres database via `Default`; the isolation is at the EF Core bookkeeping level, not the database level.

---

## Checklist — what to verify before running `dotnet ef migrations add Initial_Billing`

- `BillingEntityFrameworkCoreModule` has `AbpEntityFrameworkCorePostgreSqlModule` in `[DependsOn]`.
- `AppContext.SetSwitch("Npgsql.EnableLegacyTimestampBehavior", true)` is set in **both** `PreConfigureServices` and the design-time factory.
- `UseNpgsql(..., b => b.MigrationsHistoryTable("__EFMigrationsHistory_Billing"))` is in **both** the module's `Configure<AbpDbContextOptions>` and `BillingDbContextFactory`.
- The other two modules use a **different** `__EFMigrationsHistory_<Module>` table name each (sanity-check their existing wiring).
- `BillingDbContextFactory.BuildConfiguration()` actually finds an `appsettings.json` with `ConnectionStrings:Default` (or `:Billing`) when run from the module's project directory.
- `EntityFrameworkCoreBillingDbSchemaMigrator` is registered (the `ITransientDependency` marker handles that automatically) and the host's `DbMigrator` resolves and calls `IBillingDbSchemaMigrator.MigrateAsync()`.
- Every entity in `BillingModelBuilderExtensions.ConfigureBilling` calls `b.ConfigureByConvention()` as the first statement of its `Entity<T>(...)` block.
