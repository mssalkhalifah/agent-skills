# Adding `Acme.Billing` to a Layered Modular Monolith — EF Core Wiring on PostgreSQL

A new internal ABP module in a Layered modular monolith is, by ABP's rules, a self-contained .NET solution that ships its own data layer. The single most important rule: **don't extend the host's `DbContext` — give Billing its own**, with its own migrations, its own design-time factory, its own schema migrator, and (this is the silent foot-gun) **its own EF Core migrations history table**. If you skip the last one, every time the host boots, EF reads `__EFMigrationsHistory`, sees the *other* modules' migrations as "missing", and tries to re-apply them — bricking the database in dev and panic-rolling-back in prod.

Below is the complete wiring, file by file. I'm assuming the Billing module follows the standard Layered ABP module shape (`Acme.Billing.Domain`, `Acme.Billing.EntityFrameworkCore`, etc.) and lives under `modules/billing/` in your existing solution.

---

## 1. The DbContext interface — `Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/IBillingDbContext.cs`

ABP's best practice is to declare a per-module DbContext interface that exposes only aggregate-root `DbSet<>`s. Repositories bind to this interface, not to the concrete class — that keeps the door open later if you ever want to consolidate Billing into the host DbContext via `[ReplaceDbContext]`. For now, in a modular monolith, it stays standalone.

```csharp
using Microsoft.EntityFrameworkCore;
using Volo.Abp.Data;
using Volo.Abp.EntityFrameworkCore;

namespace Acme.Billing.EntityFrameworkCore;

[ConnectionStringName(BillingDbProperties.ConnectionStringName)]
public interface IBillingDbContext : IEfCoreDbContext
{
    DbSet<Invoice>      Invoices      { get; }
    DbSet<Subscription> Subscriptions { get; }
    // ...one DbSet<T> per aggregate root in the Billing module
}
```

And a small constants file, `Acme.Billing.Domain/BillingDbProperties.cs`:

```csharp
namespace Acme.Billing;

public static class BillingDbProperties
{
    public const string DbTablePrefix         = "Bil";
    public const string DbSchema              = null;          // PostgreSQL — keep null unless you genuinely need a schema
    public const string ConnectionStringName  = "Billing";     // resolves via AbpDbConnectionOptions
    public const string MigrationsHistoryTable = "__EFMigrationsHistory_Billing";  // ← Rule 6 — unique per module
}
```

Why a separate `ConnectionStringName`? In a modular monolith you can point `Billing` to the same physical Postgres database as the host (just add `"Billing"` alongside `"Default"` in `appsettings.json`, or omit it and let `IConnectionStringResolver` fall back to `Default`). What matters is the **logical name**, because if you ever break Billing into its own service later, only `appsettings.json` needs to change.

---

## 2. The concrete DbContext — `Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/BillingDbContext.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Volo.Abp.Data;
using Volo.Abp.EntityFrameworkCore;

namespace Acme.Billing.EntityFrameworkCore;

[ConnectionStringName(BillingDbProperties.ConnectionStringName)]
public class BillingDbContext : AbpDbContext<BillingDbContext>, IBillingDbContext
{
    public DbSet<Invoice>      Invoices      { get; set; } = null!;
    public DbSet<Subscription> Subscriptions { get; set; } = null!;

    public BillingDbContext(DbContextOptions<BillingDbContext> options) : base(options) { }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);
        builder.ConfigureBilling();    // extension method — see next file
    }
}
```

Two things to note:

- It inherits `AbpDbContext<BillingDbContext>` — never plain `DbContext`. That's what wires Billing into ABP's UoW pipeline, audit-property setter, soft-delete filter, multi-tenancy filter, and entity change events.
- `OnModelCreating` only calls one extension method. Per the EF Core best-practices doc, model configuration **does not** live inline in `OnModelCreating` — it lives in a `ConfigureBilling(this ModelBuilder)` extension method so that the host's DbContext (or some future tenant DbContext) could pick the same configuration up if you ever consolidate.

---

## 3. Model configuration — `Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/BillingDbContextModelCreatingExtensions.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Volo.Abp;
using Volo.Abp.EntityFrameworkCore.Modeling;

namespace Acme.Billing.EntityFrameworkCore;

public static class BillingDbContextModelCreatingExtensions
{
    public static void ConfigureBilling(this ModelBuilder builder)
    {
        Check.NotNull(builder, nameof(builder));

        builder.Entity<Invoice>(b =>
        {
            b.ToTable(BillingDbProperties.DbTablePrefix + "Invoices", BillingDbProperties.DbSchema);
            b.ConfigureByConvention();    // ← ALWAYS first inside the Entity<> block
            b.Property(x => x.Number).IsRequired().HasMaxLength(InvoiceConsts.MaxNumberLength);
            // ...rest of the mapping
        });

        builder.Entity<Subscription>(b =>
        {
            b.ToTable(BillingDbProperties.DbTablePrefix + "Subscriptions", BillingDbProperties.DbSchema);
            b.ConfigureByConvention();
            // ...
        });
    }
}
```

`ConfigureByConvention()` is non-negotiable — without it, soft-delete, audit columns, multi-tenancy filtering, `ExtraProperties`, and concurrency stamps don't get mapped. Forgetting it is one of the most common ABP-EF-Core mistakes; the consequences are silent and painful.

---

## 4. The EntityFrameworkCore module — `Acme.Billing.EntityFrameworkCore/BillingEntityFrameworkCoreModule.cs`

This is the load-bearing file for Rule 6. Both `UseNpgsql` calls (registration + the design-time factory in §5) must specify the same `MigrationsHistoryTable` name.

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Volo.Abp.Data;
using Volo.Abp.EntityFrameworkCore;
using Volo.Abp.EntityFrameworkCore.PostgreSql;
using Volo.Abp.Modularity;

namespace Acme.Billing.EntityFrameworkCore;

[DependsOn(
    typeof(BillingDomainModule),
    typeof(AbpEntityFrameworkCoreModule),
    typeof(AbpEntityFrameworkCorePostgreSqlModule)   // ← Postgres provider, not SqlServer
)]
public class BillingEntityFrameworkCoreModule : AbpModule
{
    public override void PreConfigureServices(ServiceConfigurationContext context)
    {
        // Required by ABP's audit columns (DateTime, not DateTimeOffset) on Npgsql 6+.
        // Set in BOTH PreConfigureServices and the design-time factory — runtime needs it
        // at host startup, dotnet ef tooling needs it when generating migrations.
        AppContext.SetSwitch("Npgsql.EnableLegacyTimestampBehavior", true);

        BillingEfCoreEntityExtensionMappings.Configure();   // optional — only if you map ExtraProperties
    }

    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        context.Services.AddAbpDbContext<BillingDbContext>(options =>
        {
            options.AddDefaultRepositories(includeAllEntities: true);
            // options.AddRepository<Invoice, InvoiceRepository>();   // only when you write a custom repo
        });

        Configure<AbpDbContextOptions>(options =>
        {
            options.Configure<BillingDbContext>(c =>
            {
                c.UseNpgsql(b => b.MigrationsHistoryTable(BillingDbProperties.MigrationsHistoryTable));
            });
        });
    }
}
```

The key call is `options.Configure<BillingDbContext>(c => c.UseNpgsql(b => b.MigrationsHistoryTable("__EFMigrationsHistory_Billing")))`. Without that nested lambda, Postgres writes Billing's migration journal into the default `__EFMigrationsHistory` table — same table the host module and your other two modules write to — and EF will treat the union of all three modules' history rows as "applicable to me", attempt to re-run them, and fail (or worse, succeed and corrupt state).

---

## 5. The design-time factory — `Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/BillingDbContextFactory.cs`

`dotnet ef migrations add` cannot boot the full ABP module graph; it needs an `IDesignTimeDbContextFactory<BillingDbContext>` that constructs the context from a hand-rolled configuration. **Without this file, `dotnet ef migrations add` will fail or fall back to host-level discovery, which doesn't see the module-private DI configuration above.**

The history-table name has to match what's in `BillingEntityFrameworkCoreModule.ConfigureServices` exactly — the runtime registration and the factory are two independent consumers of `UseNpgsql`, and a mismatch means `dotnet ef` writes its bookkeeping to one table while the host reads from another. Silent foot-gun.

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
        // Same legacy-timestamp switch as runtime — Npgsql 6+ needs it for audit DateTime columns
        AppContext.SetSwitch("Npgsql.EnableLegacyTimestampBehavior", true);

        var configuration = BuildConfiguration();
        var connectionString = configuration.GetConnectionString("Billing")
                            ?? configuration.GetConnectionString("Default");

        var builder = new DbContextOptionsBuilder<BillingDbContext>()
            .UseNpgsql(
                connectionString,
                b => b.MigrationsHistoryTable(BillingDbProperties.MigrationsHistoryTable)  // ← MUST MATCH module
            );

        return new BillingDbContext(builder.Options);
    }

    private static IConfigurationRoot BuildConfiguration()
    {
        // Climb up to the host's DbMigrator — adjust the relative path if your module
        // sits at a different depth. The intent is: read whichever appsettings.json
        // already has the production-shaped connection string.
        return new ConfigurationBuilder()
            .SetBasePath(Path.Combine(
                Directory.GetCurrentDirectory(),
                "../../../../../src/Acme.HostName.DbMigrator/"))
            .AddJsonFile("appsettings.json", optional: false)
            .AddJsonFile("appsettings.secrets.json", optional: true)
            .Build();
    }
}
```

You can now run, from `Acme.Billing.EntityFrameworkCore/`:

```bash
dotnet ef migrations add Initial \
    --context BillingDbContext \
    --output-dir Migrations
```

The generated migrations land in `Acme.Billing.EntityFrameworkCore/Migrations/` — *separate* from the host's migrations folder — and the `__EFMigrationsHistory_Billing` table becomes Billing's private journal.

---

## 6. The schema migrator interface — `Acme.Billing.Domain/Data/IBillingDbSchemaMigrator.cs`

This is the abstraction the host's `DbMigrator` uses to apply pending migrations module-by-module without leaking EF Core into the Domain layer. Each module ships a provider-agnostic interface in its Domain project plus one EF Core implementation in its EntityFrameworkCore project. The host's existing `DbMigrationService` resolves all `IXxxDbSchemaMigrator` instances and calls `MigrateAsync` on each — no changes needed in the host once you register Billing's module.

```csharp
using System.Threading.Tasks;

namespace Acme.Billing.Data;

public interface IBillingDbSchemaMigrator
{
    Task MigrateAsync();
}
```

---

## 7. The schema migrator implementation — `Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/EntityFrameworkCoreBillingDbSchemaMigrator.cs`

```csharp
using System;
using System.Threading.Tasks;
using Acme.Billing.Data;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Volo.Abp.DependencyInjection;

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
        // Resolve the DbContext from a fresh scope so it picks up the host's UoW + tenant context
        await _serviceProvider
            .GetRequiredService<BillingDbContext>()
            .Database
            .MigrateAsync();
    }
}
```

A few notes on this shape:

- It registers itself via `ITransientDependency`, so the host's `DbMigrationService` discovers it automatically — no extra wiring in the host.
- For multi-tenancy with per-tenant databases (if Billing has tenant data), the host's `DbMigrationService` typically wraps each tenant in `using (CurrentTenant.Change(tenant.Id)) { ... }` before calling each migrator — your module's migrator doesn't need to know about that, the connection-string resolver handles it via `[ConnectionStringName("Billing")]`.
- This is **not** the place for transient-failure retries; the host-level migration runner owns that policy.

---

## 8. Wiring the module into the host

In your host's main module (`Acme.HostName.Web` or wherever the AbpModule graph is composed), add the Billing dependency:

```csharp
[DependsOn(
    // ...existing modules...
    typeof(BillingApplicationModule),
    typeof(BillingHttpApiModule),
    typeof(BillingEntityFrameworkCoreModule)
)]
public class HostNameWebModule : AbpModule { /* ... */ }
```

And in `appsettings.json` for both the host and the DbMigrator (they need to match — the DbMigrator owns its own appsettings):

```json
{
  "ConnectionStrings": {
    "Default":  "Host=localhost;Port=5432;Database=AcmeApp;Username=postgres;Password=...",
    "Billing":  "Host=localhost;Port=5432;Database=AcmeApp;Username=postgres;Password=..."
  }
}
```

In a modular monolith on a single Postgres database, both names point to the same connection string. The migration history tables stay separated because Rule 6 takes care of that at the EF Core level, not at the connection-string level.

---

## Summary of files to create

| Project | File | Role |
|---|---|---|
| `Acme.Billing.Domain` | `BillingDbProperties.cs` | Constants — table prefix, schema, connection-string name, migrations history table name |
| `Acme.Billing.Domain` | `Data/IBillingDbSchemaMigrator.cs` | Provider-agnostic migration contract |
| `Acme.Billing.EntityFrameworkCore` | `EntityFrameworkCore/IBillingDbContext.cs` | Module's DbContext interface (aggregate-root DbSets) |
| `Acme.Billing.EntityFrameworkCore` | `EntityFrameworkCore/BillingDbContext.cs` | Concrete `AbpDbContext<>` |
| `Acme.Billing.EntityFrameworkCore` | `EntityFrameworkCore/BillingDbContextModelCreatingExtensions.cs` | `ConfigureBilling(this ModelBuilder)` mapping |
| `Acme.Billing.EntityFrameworkCore` | `EntityFrameworkCore/BillingDbContextFactory.cs` | `IDesignTimeDbContextFactory<BillingDbContext>` for `dotnet ef` |
| `Acme.Billing.EntityFrameworkCore` | `EntityFrameworkCore/EntityFrameworkCoreBillingDbSchemaMigrator.cs` | EF Core implementation of `IBillingDbSchemaMigrator` |
| `Acme.Billing.EntityFrameworkCore` | `BillingEntityFrameworkCoreModule.cs` | AbpModule — registers DbContext + Postgres + history table |

---

## Three traps to avoid

1. **Sharing the host's DbContext.** Tempting in a modular monolith ("it's all one database anyway") but it permanently couples Billing's schema lifecycle to the host. You lose the ability to extract Billing into a microservice later, you lose per-module migration isolation, and you couple your release cadence to the host. Keep them separate from day one.
2. **Mismatched `MigrationsHistoryTable` calls.** The runtime registration in `BillingEntityFrameworkCoreModule.ConfigureServices` and the design-time factory in `BillingDbContextFactory.CreateDbContext` are two **independent** consumers of `UseNpgsql(...)`. If one says `__EFMigrationsHistory_Billing` and the other doesn't, `dotnet ef` writes to one table while the host reads from another, and you'll spend an afternoon staring at "pending migrations" that aren't.
3. **Forgetting `Npgsql.EnableLegacyTimestampBehavior` in either place.** Set it in `PreConfigureServices` for runtime *and* at the top of `BillingDbContextFactory.CreateDbContext` for tooling. Skipping it on the runtime side gives you `DateTime values must be UTC` at startup; skipping it on the tooling side fails the migration-add step. Both call sites need it.

Once these eight files are in place and the host's main module `[DependsOn]`s `BillingEntityFrameworkCoreModule`, your existing `DbMigrator` console will pick up `EntityFrameworkCoreBillingDbSchemaMigrator` automatically and apply Billing's migrations alongside the other modules' — each writing to its own history table, each safely independent.
