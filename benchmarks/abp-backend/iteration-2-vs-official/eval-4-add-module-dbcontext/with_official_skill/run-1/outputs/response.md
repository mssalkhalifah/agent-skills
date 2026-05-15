# Adding `Acme.Billing` Module to a Layered ABP 10.3 Solution (PostgreSQL)

Relevant skills cited: **abp-ef-core**, **abp-development-flow**, **abp-dependency-rules**, **abp-core**.

This walkthrough covers everything you need on the EF Core side so the new `Acme.Billing` module gets its own migrations history, its own `Migrations/` folder, and never steps on the other two modules' migrations.

---

## 1. Why each module needs its own DbContext

Per **abp-dependency-rules** and **abp-ef-core**, the EF Core project is the only place that should know about `DbContext`. The critical rule when running a **modular monolith** is:

> Each module owns its own `DbContext` and its own EF Core migrations history table. Do NOT share a `DbContext` across modules.

Reasons:

- **Migrations conflict.** If two modules share `__EFMigrationsHistory`, the order of `dotnet ef migrations add` from different module teams collides, and one module's `up` may try to recreate the other module's tables.
- **Module independence.** Modules must be installable/upgradable independently. A shared `DbContext` ties their release cycles together.
- **DDD aggregate boundaries.** **abp-dependency-rules** says cross-module calls should go through events or domain abstractions, not shared infrastructure.
- **`AddDefaultRepositories` scope.** Calling `AddDefaultRepositories()` on a shared context registers repositories for entities that don't belong to your module.

The fix: give `Acme.Billing` its own `BillingDbContext` mapped to the **same physical PostgreSQL database** (so transactions across modules still work via ABP's UnitOfWork) but with a **dedicated `__EFMigrationsHistory_Billing` table** so each module's migrations live in isolation.

---

## 2. Project layout

Following **abp-dependency-rules** (Layered Template Structure):

```
src/
  Acme.Billing.Domain.Shared/
  Acme.Billing.Domain/
  Acme.Billing.Application.Contracts/
  Acme.Billing.Application/
  Acme.Billing.EntityFrameworkCore/        <-- everything below lives here
    EntityFrameworkCore/
      BillingDbContext.cs
      BillingDbContextModelCreatingExtensions.cs
      BillingDbContextFactory.cs           <-- IDesignTimeDbContextFactory
      BillingEntityFrameworkCoreModule.cs
      EntityFrameworkCoreBillingDbSchemaMigrator.cs
    Migrations/                            <-- module-private migrations folder
  Acme.Billing.HttpApi/
  Acme.Billing.HttpApi.Client/
```

Then in the host (`Acme.MySolution.Web` or `Acme.MySolution.HttpApi.Host`):

```
Acme.MySolution.DbMigrator/
  BillingDbSchemaMigrator is invoked here through IBillingDbSchemaMigrator
```

---

## 3. Key files

### 3.1 `BillingDbContext.cs`

Per **abp-ef-core** DbContext pattern. Note the `[ConnectionStringName]` uses a **module-specific name** so you can later split it to another DB without code changes (per **abp-core** module conventions).

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

    public BillingDbContext(DbContextOptions<BillingDbContext> options)
        : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);
        builder.ConfigureBilling();
    }
}
```

`BillingDbProperties` lives in `Acme.Billing.Domain` (or `Domain.Shared`):

```csharp
public static class BillingDbProperties
{
    public const string DbTablePrefix    = "Bil";   // table prefix, per abp-ef-core
    public const string DbSchema         = null;    // or "billing" for a Postgres schema
    public const string ConnectionStringName = "Billing";
}
```

### 3.2 `BillingDbContextModelCreatingExtensions.cs`

Per **abp-ef-core** Entity Configuration. Always call `ConfigureByConvention()`:

```csharp
public static class BillingDbContextModelCreatingExtensions
{
    public static void ConfigureBilling(this ModelBuilder builder)
    {
        Check.NotNull(builder, nameof(builder));

        builder.Entity<Invoice>(b =>
        {
            b.ToTable(BillingDbProperties.DbTablePrefix + "Invoices",
                      BillingDbProperties.DbSchema);
            b.ConfigureByConvention();

            b.Property(x => x.Number).IsRequired().HasMaxLength(InvoiceConsts.MaxNumberLength);
            b.Property(x => x.Amount).HasColumnType("numeric(18,2)"); // Postgres
            b.HasIndex(x => x.Number);
        });

        builder.Entity<Subscription>(b =>
        {
            b.ToTable(BillingDbProperties.DbTablePrefix + "Subscriptions",
                      BillingDbProperties.DbSchema);
            b.ConfigureByConvention();
        });
    }
}
```

Postgres-specific tips:
- Use `numeric(p,s)` (not `decimal(p,s)`) in `HasColumnType`.
- Optionally namespace tables with a Postgres schema by setting `DbSchema = "billing"` so `__EFMigrationsHistory_Billing` and module tables can also live in a dedicated schema.
- For JSON columns prefer `jsonb` over `json`.

### 3.3 `BillingEntityFrameworkCoreModule.cs` -- `AddAbpDbContext` + `UseNpgsql` + separate history table

This is the heart of migration isolation. The `MigrationsHistoryTable("__EFMigrationsHistory_Billing")` call is what prevents migration conflicts with the other two modules.

```csharp
using Microsoft.EntityFrameworkCore;
using Volo.Abp.EntityFrameworkCore;
using Volo.Abp.Modularity;

namespace Acme.Billing.EntityFrameworkCore;

[DependsOn(
    typeof(BillingDomainModule),
    typeof(AbpEntityFrameworkCorePostgreSqlModule)
)]
public class BillingEntityFrameworkCoreModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        context.Services.AddAbpDbContext<BillingDbContext>(options =>
        {
            // Per abp-ef-core: aggregate roots only - do NOT pass includeAllEntities: true
            options.AddDefaultRepositories<IBillingDbContext>();

            // Custom repositories
            // options.AddRepository<Invoice, EfCoreInvoiceRepository>();
        });

        Configure<AbpDbContextOptions>(options =>
        {
            // Apply to THIS context only - the other two modules each call
            // Configure<...>(...).UseNpgsql(...) for their own contexts.
            options.Configure<BillingDbContext>(c =>
            {
                c.UseNpgsql(b =>
                {
                    // Critical: per-module migrations history table.
                    // Each module must use a DIFFERENT name here.
                    b.MigrationsHistoryTable(
                        "__EFMigrationsHistory_Billing",
                        BillingDbProperties.DbSchema); // null or "billing"

                    // Keep migrations assembly == this EF Core project
                    b.MigrationsAssembly(
                        typeof(BillingEntityFrameworkCoreModule).Assembly.FullName);
                });
            });
        });
    }
}
```

Key points:
1. **`options.Configure<BillingDbContext>(...)`** is the per-context overload. Without it your `UseNpgsql` settings would leak across every module's context.
2. **`MigrationsHistoryTable("__EFMigrationsHistory_Billing")`** -- give each module a unique name (e.g. `__EFMigrationsHistory_Identity`, `__EFMigrationsHistory_Saas`, `__EFMigrationsHistory_Billing`). This is what guarantees `dotnet ef database update` only sees its own module's migrations.
3. **`MigrationsAssembly(...)`** pins migrations to the Billing EF Core project so EF doesn't try to scan the host.
4. Depend on `AbpEntityFrameworkCorePostgreSqlModule` (the Npgsql package), not the SqlServer one.

### 3.4 `BillingDbContextFactory.cs` -- `IDesignTimeDbContextFactory`

Per **abp-ef-core**: "ABP templates include `IDesignTimeDbContextFactory` in the EF Core project, so `-s` (startup project) parameter is not needed." Add the equivalent for your module so `dotnet ef migrations add` works when run from the module's EF Core project without referencing the host.

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
        BillingEfCoreEntityExtensionMappings.Configure();

        var configuration = BuildConfiguration();

        var builder = new DbContextOptionsBuilder<BillingDbContext>()
            .UseNpgsql(
                configuration.GetConnectionString("Billing")
                    ?? configuration.GetConnectionString("Default"),
                b =>
                {
                    b.MigrationsHistoryTable(
                        "__EFMigrationsHistory_Billing",
                        BillingDbProperties.DbSchema);
                    b.MigrationsAssembly(
                        typeof(BillingDbContextFactory).Assembly.FullName);
                });

        return new BillingDbContext(builder.Options);
    }

    private static IConfigurationRoot BuildConfiguration()
    {
        var builder = new ConfigurationBuilder()
            .SetBasePath(Path.Combine(
                Directory.GetCurrentDirectory(),
                "../../host/Acme.MySolution.DbMigrator/"))
            .AddJsonFile("appsettings.json", optional: false)
            .AddJsonFile("appsettings.secrets.json", optional: true);

        return builder.Build();
    }
}
```

The history-table + assembly settings **must match** those in the module class, otherwise `dotnet ef` design-time will scaffold against a fresh history and re-create existing tables.

### 3.5 `IBillingDbSchemaMigrator` (Domain.Shared / Domain) + EF Core implementation

ABP's `DbMigrator` calls one `IDataSeedContributor` and one `I<Module>DbSchemaMigrator` per module. Define the abstraction in `Acme.Billing.Domain.Shared` so the host (and `DbMigrator`) can reference it without depending on EF Core (per **abp-dependency-rules**: "EntityFrameworkCore/MongoDB -> Referenced by Host only").

**`Acme.Billing.Domain.Shared/Data/IBillingDbSchemaMigrator.cs`:**

```csharp
using System.Threading.Tasks;

namespace Acme.Billing.Data;

public interface IBillingDbSchemaMigrator
{
    Task MigrateAsync();
}
```

**`Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/EntityFrameworkCoreBillingDbSchemaMigrator.cs`:**

```csharp
using System;
using System.Threading.Tasks;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Volo.Abp.DependencyInjection;
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
        await _serviceProvider
            .GetRequiredService<BillingDbContext>()
            .Database
            .MigrateAsync();
    }
}
```

In your `DbMigrator` host (the existing one in your solution), inject **all** schema migrators and call them in order:

```csharp
public class MySolutionDbMigrationService : ITransientDependency
{
    private readonly IIdentityDbSchemaMigrator _identityMigrator;
    private readonly ISaasDbSchemaMigrator _saasMigrator;
    private readonly IBillingDbSchemaMigrator _billingMigrator; // NEW

    // ctor...

    public async Task MigrateAsync()
    {
        await _identityMigrator.MigrateAsync();
        await _saasMigrator.MigrateAsync();
        await _billingMigrator.MigrateAsync();   // NEW
        // then run IDataSeedContributors
    }
}
```

### 3.6 `appsettings.json` (host + DbMigrator)

```json
{
  "ConnectionStrings": {
    "Default":  "Host=localhost;Port=5432;Database=MySolution;Username=postgres;Password=...",
    "Billing":  "Host=localhost;Port=5432;Database=MySolution;Username=postgres;Password=..."
  }
}
```

Same physical DB is fine -- the per-module migrations history table is what keeps things separate. If/when you split `Billing` to its own database later, just change the `"Billing"` connection string and nothing else needs to move (because we used `[ConnectionStringName("Billing")]` in step 3.1).

---

## 4. Separate `Migrations/` folder per module

Because `MigrationsAssembly` is the Billing EF Core project, EF will write migrations into that project. Force them into a per-module folder via the `--output-dir` flag (per **abp-ef-core** migration commands):

```bash
cd src/Acme.Billing.EntityFrameworkCore

dotnet ef migrations add Initial_Billing_Schema \
    --output-dir Migrations \
    --context BillingDbContext
```

Each module's EF Core project gets its OWN `Migrations/` folder under its OWN project -- they never see each other on disk, and because of the per-module `__EFMigrationsHistory_Billing` table they never see each other in the database either.

To apply (recommended by **abp-ef-core**: "use `DbMigrator` project to apply migrations and seed data"):

```bash
cd aspnet-core/host/Acme.MySolution.DbMigrator
dotnet run
```

This will execute every module's `I<Module>DbSchemaMigrator.MigrateAsync()` in turn.

---

## 5. Host module wiring

In your existing `Acme.MySolution.HttpApi.Host` (or `Web`) module's `[DependsOn]` list, add the new EF Core module:

```csharp
[DependsOn(
    // ...existing modules
    typeof(BillingEntityFrameworkCoreModule),
    typeof(BillingApplicationModule),
    typeof(BillingHttpApiModule)
)]
public class MySolutionHttpApiHostModule : AbpModule { }
```

And similarly add `typeof(BillingEntityFrameworkCoreModule)` to the `DbMigrator` module's `[DependsOn]`.

---

## 6. Warnings -- things that will go wrong if you cut corners

| Mistake | What breaks |
|---|---|
| Sharing one `DbContext` across all modules | `dotnet ef migrations add` from one module emits `Drop`/`Create` for the others' tables. Module upgrades become impossible. (**abp-dependency-rules**: each module owns its data.) |
| Forgetting `MigrationsHistoryTable("__EFMigrationsHistory_Billing")` | All modules write to the same `__EFMigrationsHistory`. EF then sees "unknown migrations" from the other modules and tries to apply yours on top, often producing duplicate-table errors in Postgres. |
| Using the global `options.UseNpgsql()` instead of `options.Configure<BillingDbContext>(c => c.UseNpgsql(...))` | The history table name leaks into the other contexts (or your settings get overwritten by the next module loaded). |
| Skipping `IDesignTimeDbContextFactory` | `dotnet ef` design-time falls back to scanning the host, picks up the wrong context, and either fails or writes Billing migrations against the wrong assembly. |
| Skipping `MigrationsAssembly(...)` | EF stores migrations under the host project; you lose per-module isolation on disk. |
| Mixing `decimal(p,s)` with the SqlServer module | Works on Npgsql but is non-idiomatic; prefer `numeric(p,s)` and `jsonb`. |
| Calling `AddDefaultRepositories(includeAllEntities: true)` | Per **abp-ef-core**: creates repositories for child entities, lets callers bypass the aggregate root. |
| Putting `IBillingDbSchemaMigrator` in the EF Core project | Host/DbMigrator would have to reference EF Core directly -- violates **abp-dependency-rules** (the abstraction must live in Domain/Domain.Shared). |
| Forgetting to add `BillingEntityFrameworkCoreModule` to the DbMigrator's `[DependsOn]` | `dotnet run` in DbMigrator silently skips Billing -- no tables get created. |

---

## 7. End-to-end checklist

- [ ] `BillingDbContext` with `[ConnectionStringName("Billing")]` and its own `DbSet`s
- [ ] `BillingDbContextModelCreatingExtensions.ConfigureBilling` calling `ConfigureByConvention()` and using `BillingDbProperties.DbTablePrefix`
- [ ] `BillingEntityFrameworkCoreModule` calls `AddAbpDbContext<BillingDbContext>(o => o.AddDefaultRepositories<IBillingDbContext>())`
- [ ] `Configure<AbpDbContextOptions>(o => o.Configure<BillingDbContext>(c => c.UseNpgsql(b => { b.MigrationsHistoryTable("__EFMigrationsHistory_Billing"); b.MigrationsAssembly(...); })))`
- [ ] `BillingDbContextFactory : IDesignTimeDbContextFactory<BillingDbContext>` with identical history table + assembly settings
- [ ] `IBillingDbSchemaMigrator` in `Domain.Shared/Data/`
- [ ] `EntityFrameworkCoreBillingDbSchemaMigrator` in EF Core project, `ITransientDependency`
- [ ] DbMigrator service updated to call `_billingMigrator.MigrateAsync()`
- [ ] Host `[DependsOn(typeof(BillingEntityFrameworkCoreModule))]`
- [ ] DbMigrator `[DependsOn(typeof(BillingEntityFrameworkCoreModule))]`
- [ ] Connection string `"Billing"` (can equal `"Default"` for shared DB) added to `appsettings.json`
- [ ] First migration generated with `--output-dir Migrations --context BillingDbContext` from the Billing EF Core project
- [ ] `dotnet run` from DbMigrator applies it and creates `__EFMigrationsHistory_Billing` plus `BilInvoices`, `BilSubscriptions`
