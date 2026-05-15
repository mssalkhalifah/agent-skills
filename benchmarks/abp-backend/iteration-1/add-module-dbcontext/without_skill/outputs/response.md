# Adding the `Acme.Billing` Module's EF Core Setup in an ABP 10.3 Layered Modular Monolith (PostgreSQL)

In a modular monolith you want each module to own its **own `DbContext`** and its **own migrations folder**, but at runtime the host app composes everything into a **single physical database** via a single "host" `DbContext`. ABP gives you the plumbing for this; you just need to wire it up in the right places.

Below is the full checklist, file-by-file, for adding `Acme.Billing` alongside your two existing modules.

---

## 0. Project layout (recap)

Assuming a standard ABP 10.3 Layered template, you should already have something like:

```
src/
  Acme.MyApp.Domain.Shared/
  Acme.MyApp.Domain/
  Acme.MyApp.Application.Contracts/
  Acme.MyApp.Application/
  Acme.MyApp.EntityFrameworkCore/    <-- HOST DbContext lives here
  Acme.MyApp.HttpApi/
  Acme.MyApp.HttpApi.Host/           <-- runs migrations + connection string
  Acme.MyApp.DbMigrator/             <-- console app that runs migrations
```

For a new module called `Acme.Billing`, create the equivalent layered projects:

```
src/
  Acme.Billing.Domain.Shared/
  Acme.Billing.Domain/
  Acme.Billing.Application.Contracts/
  Acme.Billing.Application/
  Acme.Billing.EntityFrameworkCore/   <-- module DbContext + module migrations
  Acme.Billing.HttpApi/
```

The module's **own** `DbContext` lives in `Acme.Billing.EntityFrameworkCore`. The host's `DbContext` (which already exists) will be told that the Billing entities also belong to it.

---

## 1. The module's entity (Domain layer)

`src/Acme.Billing.Domain/Invoices/Invoice.cs`

```csharp
using System;
using Volo.Abp.Domain.Entities.Auditing;
using Volo.Abp.MultiTenancy;

namespace Acme.Billing.Invoices;

public class Invoice : FullAuditedAggregateRoot<Guid>, IMultiTenant
{
    public Guid? TenantId { get; protected set; }
    public string Number { get; protected set; } = default!;
    public decimal Amount { get; protected set; }
    public DateTime IssuedAt { get; protected set; }

    protected Invoice() { }

    public Invoice(Guid id, Guid? tenantId, string number, decimal amount, DateTime issuedAt)
        : base(id)
    {
        TenantId = tenantId;
        Number   = number;
        Amount   = amount;
        IssuedAt = issuedAt;
    }
}
```

---

## 2. Constants for table prefix and schema (Domain.Shared)

`src/Acme.Billing.Domain.Shared/BillingDbProperties.cs`

```csharp
namespace Acme.Billing;

public static class BillingDbProperties
{
    public const string DbTablePrefix = "Bil";   // tables become BilInvoices, BilInvoiceLines, ...
    public const string DbSchema      = null;    // PostgreSQL: leave null, default schema "public"
    public const string ConnectionStringName = "Billing"; // optional, falls back to "Default"
}
```

Using a unique prefix is what keeps the module's tables from colliding with the other two modules in the shared database.

---

## 3. The module's `IBillingDbContext` interface (EntityFrameworkCore project)

This is the contract the **host** DbContext will implement so that the module's repositories work without depending on the host.

`src/Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/IBillingDbContext.cs`

```csharp
using Acme.Billing.Invoices;
using Microsoft.EntityFrameworkCore;
using Volo.Abp.Data;
using Volo.Abp.EntityFrameworkCore;

namespace Acme.Billing.EntityFrameworkCore;

[ConnectionStringName(BillingDbProperties.ConnectionStringName)]
public interface IBillingDbContext : IEfCoreDbContext
{
    DbSet<Invoice> Invoices { get; }
}
```

---

## 4. The module's own `BillingDbContext` (used for design-time migrations only)

`src/Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/BillingDbContext.cs`

```csharp
using Acme.Billing.Invoices;
using Microsoft.EntityFrameworkCore;
using Volo.Abp.Data;
using Volo.Abp.EntityFrameworkCore;

namespace Acme.Billing.EntityFrameworkCore;

[ConnectionStringName(BillingDbProperties.ConnectionStringName)]
public class BillingDbContext : AbpDbContext<BillingDbContext>, IBillingDbContext
{
    public DbSet<Invoice> Invoices => Set<Invoice>();

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

> Important: this `DbContext` is **only** used at design time (i.e., when running `dotnet ef migrations add`) and lets the module ship its own migrations folder. At runtime the host app uses the host `DbContext` (see step 7), and the module's repositories resolve through `IBillingDbContext`.

---

## 5. Centralized model configuration (extension method)

`src/Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/BillingDbContextModelCreatingExtensions.cs`

```csharp
using Acme.Billing.Invoices;
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
            b.ToTable(BillingDbProperties.DbTablePrefix + "Invoices",
                      BillingDbProperties.DbSchema);
            b.ConfigureByConvention();   // tenant id, audit props, soft-delete, concurrency, etc.

            b.Property(x => x.Number).IsRequired().HasMaxLength(64);
            b.Property(x => x.Amount).HasColumnType("numeric(18,2)");
            b.HasIndex(x => x.Number);
        });
    }
}
```

Putting the configuration in an extension method is the trick that lets **both** the module's design-time `BillingDbContext` and the host's runtime `DbContext` apply the same mapping.

---

## 6. Repository + module class

`src/Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/EfCoreInvoiceRepository.cs`

```csharp
using Acme.Billing.Invoices;
using Volo.Abp.Domain.Repositories.EntityFrameworkCore;
using Volo.Abp.EntityFrameworkCore;
using System;

namespace Acme.Billing.EntityFrameworkCore;

public class EfCoreInvoiceRepository
    : EfCoreRepository<IBillingDbContext, Invoice, Guid>, IInvoiceRepository
{
    public EfCoreInvoiceRepository(IDbContextProvider<IBillingDbContext> provider)
        : base(provider)
    {
    }
}
```

`src/Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/BillingEntityFrameworkCoreModule.cs`

```csharp
using Microsoft.Extensions.DependencyInjection;
using Volo.Abp.EntityFrameworkCore;
using Volo.Abp.Modularity;
using Acme.Billing.Invoices;

namespace Acme.Billing.EntityFrameworkCore;

[DependsOn(
    typeof(BillingDomainModule),
    typeof(AbpEntityFrameworkCoreModule)
)]
public class BillingEntityFrameworkCoreModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        context.Services.AddAbpDbContext<BillingDbContext>(options =>
        {
            options.AddDefaultRepositories(includeAllEntities: true);
            options.AddRepository<Invoice, EfCoreInvoiceRepository>();
        });
    }
}
```

---

## 7. Host project: extend the host DbContext to also own Billing entities

This is the most important step for "no migration conflicts." In `src/Acme.MyApp.EntityFrameworkCore/EntityFrameworkCore/MyAppDbContext.cs`:

```csharp
using Acme.Billing.EntityFrameworkCore;
using Acme.Billing.Invoices;
using Microsoft.EntityFrameworkCore;
// ... existing usings

[ReplaceDbContext(typeof(IBillingDbContext))]            // tell ABP the host satisfies the module's contract
[ConnectionStringName("Default")]
public class MyAppDbContext :
    AbpDbContext<MyAppDbContext>,
    IBillingDbContext            // implement the module interface
    /* , IIdentityDbContext, IOpenIddictDbContext, ... your other modules */
{
    // Billing
    public DbSet<Invoice> Invoices => Set<Invoice>();

    // ... your other DbSets

    public MyAppDbContext(DbContextOptions<MyAppDbContext> options) : base(options) { }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);

        // Existing modules
        builder.ConfigurePermissionManagement();
        builder.ConfigureSettingManagement();
        builder.ConfigureBackgroundJobs();
        builder.ConfigureAuditLogging();
        builder.ConfigureFeatureManagement();
        builder.ConfigureIdentity();
        builder.ConfigureOpenIddict();
        builder.ConfigureTenantManagement();

        // YOUR new module
        builder.ConfigureBilling();
    }
}
```

Two pieces matter here:

1. `[ReplaceDbContext(typeof(IBillingDbContext))]` — at runtime, when any Billing repository asks for `IBillingDbContext`, ABP gives it the host `MyAppDbContext`. So **only one DbContext is opened per request**, and one transaction covers all modules.
2. `builder.ConfigureBilling()` — runs the same model configuration the module would run, so the host migrations include the Billing tables.

---

## 8. Wire the module into the host's `EntityFrameworkCoreModule`

In `src/Acme.MyApp.EntityFrameworkCore/EntityFrameworkCoreModule.cs`:

```csharp
[DependsOn(
    // ... existing module dependencies
    typeof(BillingEntityFrameworkCoreModule)
)]
public class MyAppEntityFrameworkCoreModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        context.Services.AddAbpDbContext<MyAppDbContext>(options =>
        {
            options.AddDefaultRepositories(includeAllEntities: true);
        });

        Configure<AbpDbContextOptions>(options =>
        {
            options.Configure(ctx =>
            {
                ctx.UseNpgsql();   // PostgreSQL
            });
        });
    }
}
```

The `UseNpgsql()` here is what binds the host (and, via `ReplaceDbContext`, every module) to PostgreSQL with `Npgsql.EntityFrameworkCore.PostgreSQL`. Make sure that NuGet package is referenced from `Acme.MyApp.EntityFrameworkCore`.

You also typically depend on the module from upper layers:

- `Acme.MyApp.Domain` → `DependsOn(typeof(BillingDomainModule))`
- `Acme.MyApp.Application` → `DependsOn(typeof(BillingApplicationModule))`
- `Acme.MyApp.HttpApi` → `DependsOn(typeof(BillingHttpApiModule))`

---

## 9. Module-side migrations (design-time only)

You want each module to keep an independent migrations folder so module developers can evolve the module's schema without touching the host. Add the EF design-time factory in `Acme.Billing.EntityFrameworkCore`:

`src/Acme.Billing.EntityFrameworkCore/EntityFrameworkCore/BillingDbContextFactory.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Design;
using Microsoft.Extensions.Configuration;
using System.IO;

namespace Acme.Billing.EntityFrameworkCore;

public class BillingDbContextFactory : IDesignTimeDbContextFactory<BillingDbContext>
{
    public BillingDbContext CreateDbContext(string[] args)
    {
        var configuration = BuildConfiguration();

        var builder = new DbContextOptionsBuilder<BillingDbContext>()
            .UseNpgsql(configuration.GetConnectionString("Billing")
                       ?? configuration.GetConnectionString("Default"));

        return new BillingDbContext(builder.Options);
    }

    private static IConfigurationRoot BuildConfiguration()
    {
        var builder = new ConfigurationBuilder()
            .SetBasePath(Directory.GetCurrentDirectory())
            .AddJsonFile("appsettings.json", optional: false);
        return builder.Build();
    }
}
```

Add a small `appsettings.json` next to the project (or symlink the host's) so a connection string is reachable at design time.

To create the module's own migration:

```bash
cd src/Acme.Billing.EntityFrameworkCore
dotnet ef migrations add Initial_Billing_Migration -o Migrations
```

This produces `src/Acme.Billing.EntityFrameworkCore/Migrations/...` — the module's private migration history.

---

## 10. Host migrations (the ones that actually run)

The host `MyAppDbContext` already contains the Billing entities (step 7), so the **host** also needs a migration to add the Billing tables to the real database:

```bash
cd src/Acme.MyApp.EntityFrameworkCore
dotnet ef migrations add Added_Billing_Module -o Migrations
```

Then run:

```bash
dotnet run --project src/Acme.MyApp.DbMigrator
```

The `DbMigrator` console application (created by the ABP template) will:

1. Apply all pending host migrations.
2. Seed initial data through `IDataSeedContributor`s.

Because there is exactly **one** physical migration history table (`__EFMigrationsHistory`) — owned by the host context — there is **no possibility of conflict** between modules. Each module's design-time migrations are for development reference and packaging only; the host's migrations are the source of truth.

---

## 11. Connection string in the host

`src/Acme.MyApp.HttpApi.Host/appsettings.json`:

```json
{
  "ConnectionStrings": {
    "Default": "Host=localhost;Port=5432;Database=MyApp;Username=postgres;Password=postgres;Include Error Detail=true"
  }
}
```

If you want Billing on a **separate** physical database later, add:

```json
"Billing": "Host=...;Database=BillingDb;..."
```

ABP will pick that up automatically because of `[ConnectionStringName("Billing")]` on `IBillingDbContext`. For a true modular monolith on one DB, leave it pointing at `Default`.

---

## 12. Quick sanity checklist

- [ ] `BillingDbProperties.DbTablePrefix = "Bil"` so no table-name collisions.
- [ ] `IBillingDbContext` defined and implemented by **both** `BillingDbContext` (design-time) and the host `MyAppDbContext` (runtime).
- [ ] Host `MyAppDbContext` decorated with `[ReplaceDbContext(typeof(IBillingDbContext))]`.
- [ ] `builder.ConfigureBilling()` called in the host's `OnModelCreating`.
- [ ] Host `EntityFrameworkCoreModule` depends on `BillingEntityFrameworkCoreModule` and calls `UseNpgsql()`.
- [ ] Migration added in **the host** project, then applied via `DbMigrator`.
- [ ] Module-level `dotnet ef migrations` works independently for module developers (optional but recommended).
- [ ] Repository tests in `Acme.Billing.EntityFrameworkCore.Tests` pass against the in-memory SQLite or PostgreSQL test fixture.

That's the whole loop. After this, adding a third or fourth module follows the same recipe: new prefix, new `IXyzDbContext`, host implements it with `[ReplaceDbContext]`, host calls `ConfigureXyz()`, host adds one migration. No two modules ever touch the same migrations folder, but they all live happily in one PostgreSQL database with one transaction scope.
