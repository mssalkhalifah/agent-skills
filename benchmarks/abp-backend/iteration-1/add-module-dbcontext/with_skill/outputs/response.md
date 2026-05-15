# Adding `Acme.Billing` to a modular monolith — EF Core wiring

For a Layered modular monolith, the rule that drives almost every decision here is **Rule 6: each module owns its own `DbContext`, its own EF Core migrations folder, its own `IXxxDbSchemaMigrator`, and crucially its own EF migrations history table.** If you skip the history-table piece, EF reads the default `__EFMigrationsHistory` for the Billing module, sees the other two modules' migration rows there, decides those are "missing" Billing migrations, and tries to re-apply them on the next startup. Don't share that table.

I'll walk through the files end-to-end, then call out the host-side glue.

## 0. Where these files live

In a Layered modular monolith, the existing two modules already live as nested .NET solutions, typically under something like `modules/forms/`, `modules/identity/`, etc. Mirror that. For Billing the layout is the standard ABP module project set:

```
modules/billing/
  src/
    Acme.Billing.Domain.Shared/
    Acme.Billing.Domain/
    Acme.Billing.Application.Contracts/
    Acme.Billing.Application/
    Acme.Billing.EntityFrameworkCore/        <-- everything below lives here
    Acme.Billing.HttpApi/
    Acme.Billing.HttpApi.Client/
  test/
    Acme.Billing.*.Tests/
```

Everything I show below sits in `Acme.Billing.EntityFrameworkCore/` unless I say otherwise.

## 1. The DbContext interface (Domain.Shared or Domain side, depends on convention — usually inside `EntityFrameworkCore`)

ABP's best-practice pattern is to declare a per-module `IXxxDbContext : IEfCoreDbContext` interface that exposes only aggregate-root `DbSet<>`s (no setters) and carries `[ConnectionStringName]`. Custom repositories bind to the interface, not the concrete class — that lets the [ReplaceDbContext] consolidation pattern work later if you ever fold modules together.

```csharp
// Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/IBillingDbContext.cs
using Microsoft.EntityFrameworkCore;
using Volo.Abp.Data;
using Volo.Abp.EntityFrameworkCore;
using Acme.Billing.Invoices;
using Acme.Billing.Subscriptions;

namespace Acme.Billing.EntityFrameworkCore;

[ConnectionStringName(BillingDbProperties.ConnectionStringName)]
public interface IBillingDbContext : IEfCoreDbContext
{
    DbSet<Invoice> Invoices { get; }
    DbSet<Subscription> Subscriptions { get; }
}
```

A `BillingDbProperties` constants class is the conventional home for the connection-string name, the table prefix, and the default schema:

```csharp
// Acme.Billing.Domain.Shared/BillingDbProperties.cs
namespace Acme.Billing;

public static class BillingDbProperties
{
    public const string DbTablePrefix = "Bil";        // produces BilInvoices, BilSubscriptions
    public const string DbSchema = null;              // or "billing" if you want a Postgres schema
    public const string ConnectionStringName = "Billing";
}
```

Setting a unique prefix (and optionally a schema) keeps Billing's tables clearly separate from the other two modules' tables when they share the same physical database.

## 2. The DbContext class

```csharp
// Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/BillingDbContext.cs
using Microsoft.EntityFrameworkCore;
using Volo.Abp.Data;
using Volo.Abp.EntityFrameworkCore;
using Acme.Billing.Invoices;
using Acme.Billing.Subscriptions;

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
        builder.ConfigureBilling();   // see step 3
    }
}
```

Notes:
- Inheriting from `AbpDbContext<T>` is non-negotiable — it wires the `IDataFilter` (soft-delete + multi-tenancy filters), audit-property setter, change-event publishing, and connection-string resolution. Don't fall back to plain `DbContext`.
- `[ConnectionStringName]` on the concrete class **and** the interface is intentional; the ABP best-practices doc recommends both.

## 3. Model-configuration extension method

Don't put entity configuration directly inside `OnModelCreating`. The ABP best-practices page is explicit: write a `ConfigureXxx` extension on `ModelBuilder`. This is what makes module DbContexts composable later.

```csharp
// Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/BillingDbContextModelCreatingExtensions.cs
using Microsoft.EntityFrameworkCore;
using Volo.Abp;
using Volo.Abp.EntityFrameworkCore.Modeling;
using Acme.Billing.Invoices;
using Acme.Billing.Subscriptions;

namespace Acme.Billing.EntityFrameworkCore;

public static class BillingDbContextModelCreatingExtensions
{
    public static void ConfigureBilling(this ModelBuilder builder)
    {
        Check.NotNull(builder, nameof(builder));

        builder.Entity<Invoice>(b =>
        {
            b.ToTable(BillingDbProperties.DbTablePrefix + "Invoices",
                      BillingDbProperties.DbSchema);
            b.ConfigureByConvention();          // ALWAYS — Rule 6 / Rule from EF best-practices
            b.Property(x => x.Number).IsRequired().HasMaxLength(64);
            b.HasIndex(x => x.Number);
            // ... relationships, owned types, etc.
        });

        builder.Entity<Subscription>(b =>
        {
            b.ToTable(BillingDbProperties.DbTablePrefix + "Subscriptions",
                      BillingDbProperties.DbSchema);
            b.ConfigureByConvention();
            b.Property(x => x.PlanCode).IsRequired().HasMaxLength(32);
        });
    }
}
```

`ConfigureByConvention()` on every `Entity<T>` block is mandatory. It maps `ExtraProperties`, `ConcurrencyStamp`, `IsDeleted`, audit columns, and `TenantId`, and it applies the soft-delete + multi-tenancy global filters. Forget it on one entity and that entity is silently broken.

## 4. The AbpModule for the EntityFrameworkCore project — runtime registration with the unique migrations history table

This is the place where Rule 6 actually pays off. The `MigrationsHistoryTable("__EFMigrationsHistory_Billing")` argument is the line that prevents the cross-module migration confusion.

```csharp
// Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/BillingEntityFrameworkCoreModule.cs
using Microsoft.Extensions.DependencyInjection;
using Volo.Abp.Data;
using Volo.Abp.EntityFrameworkCore;
using Volo.Abp.EntityFrameworkCore.PostgreSql;
using Volo.Abp.Modularity;
using Acme.Billing.Invoices;

namespace Acme.Billing.EntityFrameworkCore;

[DependsOn(
    typeof(BillingDomainModule),
    typeof(AbpEntityFrameworkCoreModule),
    typeof(AbpEntityFrameworkCorePostgreSqlModule)
)]
public class BillingEntityFrameworkCoreModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        context.Services.AddAbpDbContext<BillingDbContext>(options =>
        {
            // Bind the interface too so consumers (and ReplaceDbContext later) work
            options.AddDefaultRepositories<IBillingDbContext>(includeAllEntities: true);

            // Custom repositories
            options.AddRepository<Invoice, InvoiceRepository>();
        });

        Configure<AbpDbContextOptions>(options =>
        {
            options.Configure<BillingDbContext>(c =>
            {
                c.UseNpgsql(b =>
                {
                    b.MigrationsHistoryTable(
                        "__EFMigrationsHistory_Billing",       // <-- THE critical line
                        BillingDbProperties.DbSchema);          // null is fine
                    b.MigrationsAssembly(
                        typeof(BillingEntityFrameworkCoreModule).Assembly.GetName().Name);
                });
            });
        });
    }
}
```

A couple of subtle points:

- `AddDefaultRepositories<IBillingDbContext>` (the generic-on-interface overload) is what the EF best-practices doc recommends — it makes the default repositories resolve through the interface, not the concrete `BillingDbContext`. That keeps Rule 5 healthy if you ever consolidate DbContexts.
- `MigrationsAssembly(...)` is only needed if you keep migrations in an assembly other than where the `DbContext` lives. For the standard layout shown here you can omit it, but I include it because module migrations sometimes live in a sibling project.
- Don't use the global `options.UseNpgsql()` from the module-cross-cutting overload — the host might have its own `Configure<AbpDbContextOptions>(options => options.UseNpgsql(...))` that doesn't know to set Billing's history table. The per-DbContext `options.Configure<BillingDbContext>(...)` form is the safe one.

## 5. Design-time DbContextFactory (mandatory)

Without this, `dotnet ef migrations add` cannot construct the DbContext — the production graph (the host with all modules) is too heavy for tooling to boot. **The `MigrationsHistoryTable` call must match the runtime registration exactly**; a mismatch is a silent foot-gun.

```csharp
// Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/BillingDbContextFactory.cs
using System.IO;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Design;
using Microsoft.Extensions.Configuration;

namespace Acme.Billing.EntityFrameworkCore;

public class BillingDbContextFactory : IDesignTimeDbContextFactory<BillingDbContext>
{
    public BillingDbContext CreateDbContext(string[] args)
    {
        // Required by Npgsql 6+ for ABP's DateTime audit columns
        AppContext.SetSwitch("Npgsql.EnableLegacyTimestampBehavior", true);

        var configuration = BuildConfiguration();

        var builder = new DbContextOptionsBuilder<BillingDbContext>()
            .UseNpgsql(
                configuration.GetConnectionString("Billing")
                    ?? configuration.GetConnectionString("Default"),
                b => b.MigrationsHistoryTable(
                    "__EFMigrationsHistory_Billing",   // <-- MUST match step 4
                    BillingDbProperties.DbSchema));

        return new BillingDbContext(builder.Options);
    }

    private static IConfigurationRoot BuildConfiguration()
    {
        return new ConfigurationBuilder()
            // Walk to the host's DbMigrator (or Web) project where appsettings.json lives.
            // Path is relative to the .csproj, so adjust nesting depth to your layout.
            .SetBasePath(Path.Combine(
                Directory.GetCurrentDirectory(),
                "../../../../../host/src/Acme.Host.DbMigrator/"))
            .AddJsonFile("appsettings.json", optional: false)
            .Build();
    }
}
```

The `Npgsql.EnableLegacyTimestampBehavior` switch is set in **both** the runtime module's `PreConfigureServices` (assumed to already be set in your existing host wiring) and in this factory — without it design-time tooling will throw on ABP's `DateTime` audit columns.

The `BuildConfiguration` path needs to climb out to wherever the host's DbMigrator's `appsettings.json` lives — count the `..` segments based on where this `.csproj` sits relative to it.

## 6. Schema migrator (per Rule 6)

Each module owns an `IXxxDbSchemaMigrator` so the host's `DbMigrator` runs all modules' migrations from one entry point.

```csharp
// Acme.Billing.Domain/Data/IBillingDbSchemaMigrator.cs
using System.Threading.Tasks;

namespace Acme.Billing.Data;

public interface IBillingDbSchemaMigrator
{
    Task MigrateAsync();
}
```

```csharp
// Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/EntityFrameworkCoreBillingDbSchemaMigrator.cs
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
        // Resolve out of the host's DI container so the DbContextOptions
        // (including connection string + history-table config) are honored.
        await _serviceProvider
            .GetRequiredService<BillingDbContext>()
            .Database
            .MigrateAsync();
    }
}
```

Then the host's `DbMigrator` console app calls `_billingMigrator.MigrateAsync()` alongside its existing `_formsMigrator.MigrateAsync()` etc.

## 7. Generate the migrations

Run from the Billing EntityFrameworkCore project (or pass `--project`/`--startup-project`):

```bash
dotnet ef migrations add Initial \
  --project    modules/billing/src/Acme.Billing.EntityFrameworkCore \
  --startup-project host/src/Acme.Host.DbMigrator \
  --context    BillingDbContext \
  --output-dir Migrations
```

The `--context BillingDbContext` is important once the host's startup project sees multiple DbContexts — without it, `dotnet ef` complains about ambiguity.

## 8. Host wiring (the glue)

In your host's `*HostModule` (or wherever you wire modules), add the dependency:

```csharp
[DependsOn(
    typeof(ExistingModuleAEntityFrameworkCoreModule),
    typeof(ExistingModuleBEntityFrameworkCoreModule),
    typeof(BillingEntityFrameworkCoreModule)         // <-- new
    // ...
)]
public class AcmeHostModule : AbpModule { }
```

And in `appsettings.json`:

```json
{
  "ConnectionStrings": {
    "Default": "Host=localhost;Port=5432;Database=Acme;Username=postgres;Password=...",
    "Billing": "Host=localhost;Port=5432;Database=Acme;Username=postgres;Password=..."
  }
}
```

Two notes here:

- The `"Billing"` key matches `BillingDbProperties.ConnectionStringName`. If you omit it, ABP's `IConnectionStringResolver` falls back to `"Default"` — which is fine for a modular monolith where every module shares one physical DB. Listing it explicitly makes it trivial to peel Billing onto its own database later.
- If you use the same physical database, you'll see three migration history tables side-by-side: `__EFMigrationsHistory_ModuleA`, `__EFMigrationsHistory_ModuleB`, `__EFMigrationsHistory_Billing`. That's the goal.

## 9. Don't forget the AbpModule chain on the other layers

Each layer's `*Module` declares `[DependsOn]` on the previous layer, exactly as the existing modules do. The EntityFrameworkCore module is the leaf — but it's only useful if `BillingDomainModule` and friends are wired up too. (If your existing two modules are already running, this part is muscle memory.)

## Files-to-write checklist

| File | Purpose |
|------|---------|
| `Acme.Billing.Domain.Shared/BillingDbProperties.cs` | Constants: connection name, table prefix, schema |
| `Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/IBillingDbContext.cs` | Per-module DbContext interface |
| `Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/BillingDbContext.cs` | `AbpDbContext<T>` subclass |
| `Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/BillingDbContextModelCreatingExtensions.cs` | `ConfigureBilling(this ModelBuilder)` |
| `Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/BillingEntityFrameworkCoreModule.cs` | `AddAbpDbContext` + `MigrationsHistoryTable("__EFMigrationsHistory_Billing")` |
| `Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/BillingDbContextFactory.cs` | Design-time factory, **same** `MigrationsHistoryTable` arg |
| `Acme.Billing.Domain/Data/IBillingDbSchemaMigrator.cs` | Migrator interface |
| `Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/EntityFrameworkCoreBillingDbSchemaMigrator.cs` | Concrete migrator |
| `Acme.Billing.EntityFrameworkCore/Migrations/*` | Generated by `dotnet ef migrations add Initial` |
| Host's `appsettings.json` (DbMigrator + Web/HttpApi.Host) | `ConnectionStrings:Billing` (optional but recommended) |
| Host module's `[DependsOn]` list | Add `BillingEntityFrameworkCoreModule` |
| Host's `DbMigrator` runner | Resolve `IBillingDbSchemaMigrator` and call `MigrateAsync()` |

## The "if you only remember one thing" bit

The only line that matters more than `ConfigureByConvention()` for a new module DbContext is:

```csharp
b.MigrationsHistoryTable("__EFMigrationsHistory_Billing", BillingDbProperties.DbSchema);
```

It must appear **identically** in (a) the runtime registration in `BillingEntityFrameworkCoreModule.ConfigureServices`, and (b) the design-time `BillingDbContextFactory`. Any drift between those two — different name, missing schema arg, missing in one — and `dotnet ef` writes to one history table while the host reads from another. That class of bug is silent for weeks until someone runs a migration on a non-trivial environment.

That's the whole EF Core side. Let me know if you want me to dig into the host-level DbMigrator runner shape or the per-tenant database scenario (where Billing might want its own connection string at runtime via `MultiTenantConnectionStringResolver`).
