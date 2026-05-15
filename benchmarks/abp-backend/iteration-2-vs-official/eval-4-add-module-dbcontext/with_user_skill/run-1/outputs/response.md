# Adding `Acme.Billing` to a Layered modular monolith (ABP 10.3, PostgreSQL)

Each module in a modular monolith owns its own `DbContext`, its own `Migrations/` folder, its own EF migrations history table, and its own schema migrator. Sharing one `DbContext` across modules looks tempting but defeats the entire point of modularity (see the warning at the bottom).

Below is the minimum file set to drop `Acme.Billing` in cleanly alongside your two existing modules, with PostgreSQL via `UseNpgsql`.

## File layout

```
modules/billing/
└── src/
    ├── Acme.Billing.Domain/
    │   └── Data/IBillingDbSchemaMigrator.cs
    └── Acme.Billing.EntityFrameworkCore/
        ├── EntityFrameworkCore/
        │   ├── BillingDbContext.cs
        │   ├── BillingDbContextFactory.cs
        │   └── EntityFrameworkCoreBillingDbSchemaMigrator.cs
        ├── Migrations/                          # ← module-private, never shared
        └── BillingEntityFrameworkCoreModule.cs
```

The host's `Acme.MyApp.DbMigrator` console app keeps owning the appsettings.json with the connection string and the runtime entry point. Your existing modules' `IXxxDbSchemaMigrator` implementations are already wired up; the host's `DbMigrationService` iterates every `IXxxDbSchemaMigrator` registered in DI, so the new `IBillingDbSchemaMigrator` just gets picked up.

---

## 1. `BillingDbContext.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Volo.Abp.Data;
using Volo.Abp.EntityFrameworkCore;
using Volo.Abp.EntityFrameworkCore.Modeling;

namespace Acme.Billing.EntityFrameworkCore;

[ConnectionStringName(BillingDbProperties.ConnectionStringName)] // "Billing", falls back to "Default"
public class BillingDbContext : AbpDbContext<BillingDbContext>
{
    public DbSet<Invoice> Invoices { get; set; } = null!;
    public DbSet<Subscription> Subscriptions { get; set; } = null!;

    public BillingDbContext(DbContextOptions<BillingDbContext> options)
        : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);
        builder.ConfigureBilling(); // extension method — never put config inline here
    }
}
```

Two important conventions:

- `: AbpDbContext<BillingDbContext>` — inheriting from `AbpDbContext<T>` wires the framework (UoW, audit property setter, soft-delete + multi-tenancy filters, change-event publishing, `IDataFilter`). Plain `DbContext` skips all of that.
- `[ConnectionStringName("Billing")]` — lets ops point Billing at its own database later via `ConnectionStrings:Billing` without touching code. With nothing configured, ABP's resolver falls back to `"Default"`.

`BillingDbContextModelCreatingExtensions.ConfigureBilling(this ModelBuilder builder)` lives in the same project; call `b.ConfigureByConvention()` as the first line of every `Entity<T>(...)` block inside it.

---

## 2. `BillingEntityFrameworkCoreModule.cs` — `AddAbpDbContext` + `UseNpgsql` + per-module history table

```csharp
using Microsoft.EntityFrameworkCore;
using Volo.Abp.EntityFrameworkCore;
using Volo.Abp.Modularity;

namespace Acme.Billing.EntityFrameworkCore;

[DependsOn(
    typeof(BillingDomainModule),
    typeof(AbpEntityFrameworkCoreModule),
    typeof(AbpEntityFrameworkCorePostgreSqlModule) // ← Postgres, not SqlServer
)]
public class BillingEntityFrameworkCoreModule : AbpModule
{
    public override void PreConfigureServices(ServiceConfigurationContext context)
    {
        // ABP audit columns are DateTime (not DateTimeOffset); Npgsql 6+ requires this.
        AppContext.SetSwitch("Npgsql.EnableLegacyTimestampBehavior", true);
    }

    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        context.Services.AddAbpDbContext<BillingDbContext>(options =>
        {
            options.AddDefaultRepositories(includeAllEntities: true);
            // options.AddRepository<Invoice, InvoiceRepository>(); // when you write custom ones
        });

        Configure<AbpDbContextOptions>(options =>
        {
            options.Configure<BillingDbContext>(c =>
            {
                c.UseNpgsql(b =>
                    b.MigrationsHistoryTable("__EFMigrationsHistory_Billing")); // ← critical
            });
        });
    }
}
```

Three things to call out:

- **`AddAbpDbContext`, not `AddDbContext`.** The plain EF Core registration skips ABP's UoW pipeline and the audit/property setter — soft delete, tenancy, and audit columns silently stop working.
- **`UseNpgsql`, not `UseSqlServer`.** The `[DependsOn]` on `AbpEntityFrameworkCorePostgreSqlModule` is the contract change; `UseNpgsql` is what actually runs.
- **`MigrationsHistoryTable("__EFMigrationsHistory_Billing")`.** This is the line that prevents cross-module conflicts. If you leave the default `__EFMigrationsHistory`, EF sees migrations rows from your two other modules and tries to re-apply them (or fails because the rows don't match Billing's migration ids). The naming convention is `__EFMigrationsHistory_<Module>`.

---

## 3. `BillingDbContextFactory.cs` — `IDesignTimeDbContextFactory<TDbContext>`

EF Core tooling (`dotnet ef migrations add`, `Update-Database`) cannot boot the full ABP host, so each module ships its own design-time factory. **Repeat the `MigrationsHistoryTable` call here** — if you don't, `dotnet ef` writes to the default history table at design time while the runtime reads from `__EFMigrationsHistory_Billing`. That mismatch is silent and very annoying.

```csharp
using System.IO;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Design;
using Microsoft.Extensions.Configuration;

namespace Acme.Billing.EntityFrameworkCore;

public class BillingDbContextFactory : IDesignTimeDbContextFactory<BillingDbContext>
{
    public BillingDbContext CreateDbContext(string[] args)
    {
        AppContext.SetSwitch("Npgsql.EnableLegacyTimestampBehavior", true);

        var configuration = BuildConfiguration();
        var builder = new DbContextOptionsBuilder<BillingDbContext>()
            .UseNpgsql(
                configuration.GetConnectionString("Billing")
                    ?? configuration.GetConnectionString("Default"),
                b => b.MigrationsHistoryTable("__EFMigrationsHistory_Billing") // ← MUST match runtime
            );

        return new BillingDbContext(builder.Options);
    }

    private static IConfigurationRoot BuildConfiguration() =>
        new ConfigurationBuilder()
            // Walk up to the host's DbMigrator appsettings.json — adjust depth to your repo layout
            .SetBasePath(Path.Combine(
                Directory.GetCurrentDirectory(),
                "../../../../../src/Acme.MyApp.DbMigrator/"))
            .AddJsonFile("appsettings.json", optional: false)
            .Build();
}
```

Run migrations from the module project, pointing the startup project at the DbMigrator host so the connection string resolves:

```bash
dotnet ef migrations add Initial_Billing \
  --project   modules/billing/src/Acme.Billing.EntityFrameworkCore \
  --startup-project src/Acme.MyApp.DbMigrator \
  --output-dir Migrations
```

The `--output-dir Migrations` is what guarantees Billing's migrations land in its own folder, never mixed with the other modules.

---

## 4. `IBillingDbSchemaMigrator` (Domain layer) + EF Core implementation

The **interface** is provider-agnostic and lives in `Acme.Billing.Domain` so the host's `DbMigrationService` can depend on it without taking an EF Core reference.

```csharp
// modules/billing/src/Acme.Billing.Domain/Data/IBillingDbSchemaMigrator.cs
using System.Threading.Tasks;

namespace Acme.Billing.Data;

public interface IBillingDbSchemaMigrator
{
    Task MigrateAsync();
}
```

The **implementation** lives in `Acme.Billing.EntityFrameworkCore` and uses `IDbContextProvider<BillingDbContext>` so it picks up the active connection string, multi-tenancy resolution, and UoW context:

```csharp
// modules/billing/src/Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/EntityFrameworkCoreBillingDbSchemaMigrator.cs
using System.Threading.Tasks;
using Acme.Billing.Data;
using Microsoft.EntityFrameworkCore;
using Volo.Abp.DependencyInjection;
using Volo.Abp.EntityFrameworkCore;

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
        var ctx = await _dbContextProvider.GetDbContextAsync();
        await ctx.Database.MigrateAsync();
    }
}
```

The `ITransientDependency` marker is what lets the host's existing `DbMigrationService` discover this implementation alongside the other modules' migrators — typically the host injects `IEnumerable<I…DbSchemaMigrator>` (one per module) and awaits each `MigrateAsync()` in turn. No host code change is required when you add Billing; it just shows up in DI.

`IDbContextProvider<TDbContext>` is the ABP-idiomatic dependency here: it routes through ABP's connection-string resolver, honors `[ReplaceDbContext]` if you ever introduce it, and participates in the current Unit of Work — which `new BillingDbContext(...)` or a raw `IServiceProvider.GetRequiredService<BillingDbContext>()` would not.

---

## 5. Wire the module into the host

The host's main `AbpModule` (the one in `Acme.MyApp.Web` or `Acme.MyApp.EntityFrameworkCore`, and likewise the DbMigrator) needs `[DependsOn(typeof(BillingEntityFrameworkCoreModule))]`. The `IBillingDbSchemaMigrator` registration then flows in automatically via the module's `ConfigureServices`.

Add the connection string entry (optional but recommended for future per-module DB splits):

```jsonc
// src/Acme.MyApp.DbMigrator/appsettings.json and Acme.MyApp.Web/appsettings.json
{
  "ConnectionStrings": {
    "Default": "Host=localhost;Port=5432;Database=AcmeMyApp;Username=postgres;Password=...",
    "Billing": "Host=localhost;Port=5432;Database=AcmeMyApp;Username=postgres;Password=..."
    // Identical for now; can be split off to its own database later without code changes.
  }
}
```

---

## Why per-module DbContext (don't share one across modules)

It is technically possible to make `Acme.MyApp.EntityFrameworkCore.MyAppDbContext` implement `IBillingDbContext`, `IOrdersDbContext`, `ICustomersDbContext` and stack `[ReplaceDbContext]` attributes — that pattern exists in the ABP template for **first-party application modules** (Identity, OpenIddict, etc.) consolidated into one physical schema. **Do not extend it to your own business modules** in a modular monolith. Specifically:

- **Migrations become a tangle.** A single `Migrations/` folder mixing Billing, Orders, and Customers means any schema change in one module shows up as a migration that touches the other two, and `dotnet ef migrations add` always rolls up every pending change across modules. Reverting one module's change is no longer local.
- **The history-table fix doesn't apply.** `__EFMigrationsHistory_<Module>` only makes sense with one history table per `DbContext`. Sharing the `DbContext` gives you exactly one history row stream for everything.
- **Module boundaries leak through EF.** A shared `DbContext` knows every `DbSet<T>` in the system; navigation properties, eager-loading, and projections start crossing module seams. The whole `[DependsOn(*.Application.Contracts only)]` discipline that keeps modules isolated falls apart at the persistence layer.
- **The "split this module out" escape hatch closes.** If Billing later needs its own database (regulatory, scaling, polyglot persistence) you just change `ConnectionStrings:Billing` and rerun migrations against the new target. With a shared `DbContext` you'd be rewriting code.

The per-module shape above (own `DbContext`, own `Migrations/`, own history table, own schema migrator) keeps every one of those doors open. Use the shared/`[ReplaceDbContext]` pattern only for the ABP-shipped modules that the template already consolidated for you.

---

## Checklist for the new module

- [ ] `BillingDbContext : AbpDbContext<BillingDbContext>` with `[ConnectionStringName("Billing")]`
- [ ] `BillingEntityFrameworkCoreModule` `[DependsOn]` includes `AbpEntityFrameworkCorePostgreSqlModule`
- [ ] `AddAbpDbContext<BillingDbContext>` (not `AddDbContext`)
- [ ] `UseNpgsql(b => b.MigrationsHistoryTable("__EFMigrationsHistory_Billing"))` in **both** the module's `ConfigureServices` and `BillingDbContextFactory`
- [ ] `AppContext.SetSwitch("Npgsql.EnableLegacyTimestampBehavior", true)` in `PreConfigureServices` and the design-time factory
- [ ] `IBillingDbSchemaMigrator` in `Acme.Billing.Domain/Data/`
- [ ] `EntityFrameworkCoreBillingDbSchemaMigrator : ITransientDependency` using `IDbContextProvider<BillingDbContext>` in `Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/`
- [ ] Host's main module `[DependsOn(typeof(BillingEntityFrameworkCoreModule))]`
- [ ] `dotnet ef migrations add Initial_Billing --output-dir Migrations` against the Billing project, startup-project pointing at DbMigrator
- [ ] Run `Acme.MyApp.DbMigrator`; confirm a fresh `__EFMigrationsHistory_Billing` table appears next to the other modules' tables in PostgreSQL
