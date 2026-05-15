# Adding `Acme.Billing` Module - EF Core / PostgreSQL Setup

For a modular monolith on ABP 10.3, each module **owns its own `DbContext` and its own migrations history table** in the shared PostgreSQL database. This keeps migrations independent so `Acme.Billing` migrations never collide with the two existing modules.

Skills consulted: `abp-ef-core`, `abp-module`, `abp-development-flow`, `abp-dependency-rules`.

---

## 1. Project layout (follow `abp-dependency-rules`)

Add the standard layered set of projects for the new module:

```
src/
├── Acme.Billing.Domain.Shared/
├── Acme.Billing.Domain/
├── Acme.Billing.Application.Contracts/
├── Acme.Billing.Application/
├── Acme.Billing.EntityFrameworkCore/    <-- everything below lives here
├── Acme.Billing.HttpApi/
└── Acme.Billing.HttpApi.Client/
```

The Host project (the modular-monolith Web/API host) references `Acme.Billing.EntityFrameworkCore` and depends on the new module. The `DbMigrator` project should also depend on `Acme.BillingEntityFrameworkCoreModule` so `IBillingDbSchemaMigrator` runs on startup.

---

## 2. `BillingDbContext` (own DbContext per module - never share)

`src/Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/BillingDbContext.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Volo.Abp.Data;
using Volo.Abp.EntityFrameworkCore;

namespace Acme.Billing.EntityFrameworkCore;

[ConnectionStringName(BillingDbProperties.ConnectionStringName)]
public class BillingDbContext : AbpDbContext<BillingDbContext>, IBillingDbContext
{
    public DbSet<Invoice> Invoices { get; set; }
    public DbSet<Payment> Payments { get; set; }

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

`src/Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/BillingDbContextModelCreatingExtensions.cs`

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
            b.ConfigureByConvention();           // critical - audit/soft-delete/multi-tenant cols
            b.Property(x => x.Number).IsRequired().HasMaxLength(64);
            b.Property(x => x.Amount).HasColumnType("numeric(18,2)"); // PG-native decimal
            b.HasIndex(x => x.Number).IsUnique();
        });

        builder.Entity<Payment>(b =>
        {
            b.ToTable(BillingDbProperties.DbTablePrefix + "Payments",
                      BillingDbProperties.DbSchema);
            b.ConfigureByConvention();
        });
    }
}
```

`src/Acme.Billing.Domain.Shared/BillingDbProperties.cs` - the table-prefix / connection-string-name constants from `abp-module`:

```csharp
public static class BillingDbProperties
{
    public const string DbTablePrefix = "Bil";          // avoid clashes with other modules
    public const string DbSchema = null;                // or "billing" if you want a PG schema
    public const string ConnectionStringName = "Billing";
}
```

> Per `abp-ef-core`: **never inject `DbContext` outside the EF Core project**. Application/Domain services use `IRepository<Invoice, Guid>` (or a custom `IInvoiceRepository`).

---

## 3. Module class - `AddAbpDbContext` + `UseNpgsql` + isolated migrations history table

`src/Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/BillingEntityFrameworkCoreModule.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Volo.Abp.EntityFrameworkCore;
using Volo.Abp.EntityFrameworkCore.PostgreSql;
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
            // Aggregate roots only - do NOT pass includeAllEntities: true
            options.AddDefaultRepositories<IBillingDbContext>();

            // Custom repositories
            options.AddRepository<Invoice, EfCoreInvoiceRepository>();
        });

        Configure<AbpDbContextOptions>(options =>
        {
            // Apply Npgsql AND a module-scoped migrations history table.
            // This is the single most important line for "migrations don't conflict".
            options.Configure<BillingDbContext>(ctx =>
            {
                ctx.UseNpgsql(b =>
                {
                    b.MigrationsHistoryTable(
                        "__EFMigrationsHistory_Billing", // unique per module
                        BillingDbProperties.DbSchema);   // null = "public"
                    b.MigrationsAssembly(
                        typeof(BillingEntityFrameworkCoreModule).Assembly.GetName().Name);
                });
            });
        });
    }
}
```

Why the custom `MigrationsHistoryTable`:

- All modules can share the same PostgreSQL database (and even the same schema).
- EF Core tracks applied migrations in `__EFMigrationsHistory` by default. If two modules both write there, `dotnet ef database update` from one module will think the other module's migrations are "unknown" and try to re-apply or refuse.
- Giving every module its own history table (`__EFMigrationsHistory_Billing`, `__EFMigrationsHistory_Catalog`, ...) makes each module's migration history independent.

---

## 4. `IDesignTimeDbContextFactory` for `dotnet ef migrations add`

ABP layered templates always include one in the EF Core project so you don't need `-s` on the CLI (per `abp-ef-core`).

`src/Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/BillingDbContextFactory.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Design;
using Microsoft.Extensions.Configuration;

namespace Acme.Billing.EntityFrameworkCore;

public class BillingDbContextFactory : IDesignTimeDbContextFactory<BillingDbContext>
{
    public BillingDbContext CreateDbContext(string[] args)
    {
        var configuration = BuildConfiguration();

        var builder = new DbContextOptionsBuilder<BillingDbContext>()
            .UseNpgsql(
                configuration.GetConnectionString(BillingDbProperties.ConnectionStringName)
                    ?? configuration.GetConnectionString("Default"),
                b =>
                {
                    b.MigrationsHistoryTable(
                        "__EFMigrationsHistory_Billing",
                        BillingDbProperties.DbSchema);
                    b.MigrationsAssembly(
                        typeof(BillingDbContextFactory).Assembly.GetName().Name);
                });

        return new BillingDbContext(builder.Options);
    }

    private static IConfigurationRoot BuildConfiguration()
    {
        // Point at the Host's appsettings.json so the connection string is found
        var builder = new ConfigurationBuilder()
            .SetBasePath(Path.Combine(Directory.GetCurrentDirectory(),
                "../../host/Acme.Host"))    // adjust to your host project path
            .AddJsonFile("appsettings.json", optional: false);

        return builder.Build();
    }
}
```

The `MigrationsHistoryTable` call here must match the one in the module - otherwise design-time and runtime disagree.

---

## 5. `IBillingDbSchemaMigrator` + EF Core implementation

This is the module-local hook the central `DbMigrator` calls to bring each module's schema up to date.

`src/Acme.Billing.Domain/Data/IBillingDbSchemaMigrator.cs`

```csharp
namespace Acme.Billing.Data;

public interface IBillingDbSchemaMigrator
{
    Task MigrateAsync();
}
```

`src/Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/EntityFrameworkCoreBillingDbSchemaMigrator.cs`

```csharp
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
        // Resolve the DbContext through DI so ABP's UoW + connection string resolution runs.
        await _serviceProvider
            .GetRequiredService<BillingDbContext>()
            .Database
            .MigrateAsync();
    }
}
```

In the `DbMigrator` host (or in `HostDbMigrationService`), inject **every** module's `I{Module}DbSchemaMigrator` and call them in sequence:

```csharp
public class AcmeDbMigrationService : ITransientDependency
{
    private readonly IBillingDbSchemaMigrator _billingMigrator;
    private readonly ICatalogDbSchemaMigrator _catalogMigrator;
    // ... existing module migrators

    public async Task MigrateAsync()
    {
        await _catalogMigrator.MigrateAsync();
        await _billingMigrator.MigrateAsync();   // each writes to its own history table
    }
}
```

---

## 6. Separate `Migrations/` folder per module (do not share)

Each EF Core project keeps its own migrations:

```
src/Acme.Billing.EntityFrameworkCore/
└── Migrations/
    ├── 20260511_InitialBilling.cs
    ├── 20260511_InitialBilling.Designer.cs
    └── BillingDbContextModelSnapshot.cs
```

Run from inside `src/Acme.Billing.EntityFrameworkCore`:

```bash
dotnet ef migrations add Initial_Billing
# then apply via the central DbMigrator (recommended - also runs seeds)
dotnet run --project ../../shared/Acme.DbMigrator
```

Because `MigrationsAssembly` points at `Acme.Billing.EntityFrameworkCore` and `MigrationsHistoryTable` is `__EFMigrationsHistory_Billing`, EF Core will:

- Only scan this assembly for migration classes.
- Only read/write the Billing history table.

No other module's migrations are visible to this DbContext and vice versa.

---

## 7. Connection string

`appsettings.json` in the Host and the `DbMigrator`:

```json
{
  "ConnectionStrings": {
    "Default": "Host=localhost;Database=AcmeMonolith;Username=acme;Password=...",
    "Billing": "Host=localhost;Database=AcmeMonolith;Username=acme;Password=..."
  }
}
```

Same DB is fine for a modular monolith. The `[ConnectionStringName("Billing")]` attribute means you can later split Billing onto its own PG instance just by changing the connection string - no code changes.

---

## 8. Warning: do NOT share a DbContext across modules

It is tempting to add `DbSet<Invoice>` to the existing host DbContext. Do not. From `abp-ef-core` and `abp-module`:

- **Migrations collide.** A single DbContext means a single migrations history table and a single `ModelSnapshot`. Two teams adding migrations in parallel produce merge conflicts in `ModelSnapshot.cs` on every PR.
- **Breaks module independence.** Other modules would have to reference Billing's entities to compile.
- **Breaks DDD aggregate boundaries.** Cross-module navigation properties leak ownership; cross-module updates skip aggregate root invariants.
- **Can't be redeployed separately.** If you ever split a module out (microservice extraction), a shared DbContext makes it nearly impossible.
- **Connection-string flexibility lost.** `[ConnectionStringName("Billing")]` on the module's own DbContext is the only thing that lets you point Billing at a different PG database without touching code.

Cross-module communication should go through **application service interfaces, domain events, or the distributed event bus** (see `abp-dependency-rules` "Cross-module app service call: Tight coupling -> Use events or domain"), never through a shared DbContext.

---

## Key files checklist

| File | Purpose |
|------|---------|
| `Acme.Billing.Domain.Shared/BillingDbProperties.cs` | `DbTablePrefix`, `DbSchema`, `ConnectionStringName` |
| `Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/BillingDbContext.cs` | Own `AbpDbContext<BillingDbContext>` |
| `Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/BillingDbContextModelCreatingExtensions.cs` | `ConfigureByConvention` + entity mappings |
| `Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/BillingEntityFrameworkCoreModule.cs` | `AddAbpDbContext` + `UseNpgsql` + `MigrationsHistoryTable("__EFMigrationsHistory_Billing")` |
| `Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/BillingDbContextFactory.cs` | `IDesignTimeDbContextFactory<BillingDbContext>` for `dotnet ef` CLI |
| `Acme.Billing.Domain/Data/IBillingDbSchemaMigrator.cs` | Module's migrate hook |
| `Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/EntityFrameworkCoreBillingDbSchemaMigrator.cs` | EF Core implementation calling `Database.MigrateAsync()` |
| `Acme.Billing.EntityFrameworkCore/Migrations/` | Module-private migrations folder + `ModelSnapshot` |
| Host `appsettings.json` | `"Billing"` connection string (can equal `"Default"` initially) |
| `Acme.DbMigrator` `AcmeDbMigrationService` | Calls `IBillingDbSchemaMigrator.MigrateAsync()` alongside the other modules |
