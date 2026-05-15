# Adding `Acme.Billing` EF Core to a Layered Modular Monolith (ABP 10.3, PostgreSQL)

This walkthrough is grounded in the official `abp-ef-core`, `abp-module`, `abp-development-flow`, and `abp-dependency-rules` skills. The golden rule for a modular monolith: **each module owns its own `DbContext`, its own `Migrations` folder, and its own `__EFMigrationsHistory_<Module>` table**. Sharing a `DbContext` across modules destroys module isolation and produces conflicting migrations.

---

## 1. Project layout for the new module

Following the layered template rules from `abp-dependency-rules`, add these projects (only the EF-relevant ones are listed; you also need `Domain.Shared`, `Domain`, `Application.Contracts`, `Application`, `HttpApi`):

```
src/
  Acme.Billing.Domain.Shared/
  Acme.Billing.Domain/
  Acme.Billing.EntityFrameworkCore/          <-- new EF Core project for the module
    BillingDbContext.cs
    BillingDbContextModelCreatingExtensions.cs
    BillingDbContextFactory.cs               <-- IDesignTimeDbContextFactory
    EntityFrameworkCoreBillingDbSchemaMigrator.cs
    BillingEntityFrameworkCoreModule.cs
    Migrations/                              <-- module-scoped migrations
  Acme.Billing.Application.Contracts/
  Acme.Billing.Application/
```

Dependency direction (`abp-dependency-rules`): `Billing.EntityFrameworkCore` references `Billing.Domain` only. The Host references each module's `EntityFrameworkCore` project and depends on `AbpEntityFrameworkCorePostgreSqlModule` once globally.

---

## 2. Constants (`Acme.Billing.Domain.Shared`)

Per `abp-module`, expose a table prefix, schema, and a **dedicated connection-string name** so the module is database-relocatable later.

```csharp
// BillingDbProperties.cs
public static class BillingDbProperties
{
    public const string DbTablePrefix = "Bil";   // unique per module
    public const string DbSchema = null;          // or "billing" if you want a Postgres schema
    public const string ConnectionStringName = "Billing";
}
```

---

## 3. `BillingDbContext` (`Acme.Billing.EntityFrameworkCore/BillingDbContext.cs`)

Follow the `abp-ef-core` DbContext pattern. Note `[ConnectionStringName]` matches `BillingDbProperties.ConnectionStringName` so ABP can resolve a module-specific connection string from `appsettings.json` and fall back to `"Default"` if not present.

```csharp
[ConnectionStringName(BillingDbProperties.ConnectionStringName)]
public class BillingDbContext : AbpDbContext<BillingDbContext>, IBillingDbContext
{
    public DbSet<Invoice> Invoices { get; set; }
    public DbSet<InvoiceLine> InvoiceLines { get; set; }
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

`IBillingDbContext` (in `Domain` or `EntityFrameworkCore`) lets repositories from other modules read - but never write - your context if absolutely required. Prefer not exposing it at all.

---

## 4. Entity model configuration

`BillingDbContextModelCreatingExtensions.cs`:

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
            b.ConfigureByConvention();   // mandatory (abp-ef-core)

            b.Property(x => x.Number).IsRequired().HasMaxLength(64);
            b.Property(x => x.TotalAmount).HasColumnType("numeric(18,2)"); // Postgres-native decimal
            b.HasIndex(x => x.Number).IsUnique();
        });

        builder.Entity<InvoiceLine>(b =>
        {
            b.ToTable(BillingDbProperties.DbTablePrefix + "InvoiceLines",
                      BillingDbProperties.DbSchema);
            b.ConfigureByConvention();
        });
    }
}
```

The unique `Bil` prefix (and optional Postgres schema) prevents collisions with the other two modules' tables.

---

## 5. `BillingEntityFrameworkCoreModule` - registration + Npgsql + per-module history table

This is the most important file. The `MigrationsHistoryTable` call is what isolates this module's migration history from the other modules in the same database.

```csharp
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
            // Aggregate-root-only repositories (abp-ef-core best practice)
            options.AddDefaultRepositories<IBillingDbContext>();
        });

        Configure<AbpDbContextOptions>(options =>
        {
            // Per-DbContext UseNpgsql with its own __EFMigrationsHistory table.
            options.Configure<BillingDbContext>(c =>
            {
                c.UseNpgsql(b =>
                {
                    b.MigrationsHistoryTable(
                        "__EFMigrationsHistory_Billing",
                        BillingDbProperties.DbSchema);
                    b.MigrationsAssembly(typeof(BillingDbContext).Assembly.FullName);
                });
            });
        });
    }
}
```

Why this matters in a modular monolith on Postgres:

- All modules share the same physical database, but each `DbContext` has its **own** `__EFMigrationsHistory_<Module>` table. EF Core looks at this table to decide which migrations have been applied. If you reused the default `__EFMigrationsHistory`, EF would think this module's migrations are "missing" and try to re-apply them, or it would refuse to apply because of unknown rows - undefined behavior.
- `MigrationsAssembly(...)` is defensive but useful: it pins migrations to the `Acme.Billing.EntityFrameworkCore` assembly so `dotnet ef` cannot accidentally generate them somewhere else.
- The global `Configure<AbpDbContextOptions>(options => options.UseNpgsql())` in the Host stays; the per-DbContext overload above only adds Billing-specific knobs.

---

## 6. `IDesignTimeDbContextFactory` (`BillingDbContextFactory.cs`)

Required so `dotnet ef migrations add` works from the EF Core project without spinning up the Host. Mirrors the pattern documented in `abp-ef-core` ("ABP templates include `IDesignTimeDbContextFactory` in the EF Core project, so `-s` is not needed").

```csharp
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
                    b.MigrationsAssembly(typeof(BillingDbContext).Assembly.FullName);
                });

        return new BillingDbContext(builder.Options);
    }

    private static IConfigurationRoot BuildConfiguration()
    {
        // Point at the DbMigrator project so the design-time tool reads the right appsettings.
        var configurationBuilder = new ConfigurationBuilder()
            .SetBasePath(Path.Combine(Directory.GetCurrentDirectory(),
                "../../src/Acme.YourSolution.DbMigrator/"))
            .AddJsonFile("appsettings.json", optional: false);

        return configurationBuilder.Build();
    }
}
```

Two non-negotiables here:

1. The Npgsql options inside the factory must mirror the runtime module config (same `MigrationsHistoryTable`, same `MigrationsAssembly`). Otherwise `dotnet ef migrations add` will generate migrations targeting the wrong history table.
2. Resolve the connection string named `"Billing"` first, then fall back to `"Default"`. This is what makes the module relocatable.

---

## 7. Per-module `Migrations/` folder

Run migrations from inside the module's EF Core project so EF places artifacts in the right place:

```bash
cd src/Acme.Billing.EntityFrameworkCore
dotnet ef migrations add Created_Billing_Initial -o Migrations
```

This produces:

```
Acme.Billing.EntityFrameworkCore/
  Migrations/
    20260511_Created_Billing_Initial.cs
    20260511_Created_Billing_Initial.Designer.cs
    BillingDbContextModelSnapshot.cs
```

Each existing module also has its own `Migrations/` folder under its own EF Core project; nothing in Billing's folder ever references the other modules' DbContexts, snapshots, or migrations. That is the structural guarantee against migration conflicts.

---

## 8. `IBillingDbSchemaMigrator` + EF Core implementation

ABP's solution template uses an `IDbSchemaMigrator` per module so the central `DbMigrator` project (or first-run logic in the Host) can apply pending migrations for every module in one pass, without the host project taking a hard reference to each module's `DbContext`.

**Interface in `Acme.Billing.Domain` (no EF Core dependency):**

```csharp
public interface IBillingDbSchemaMigrator
{
    Task MigrateAsync();
}
```

**EF Core implementation in `Acme.Billing.EntityFrameworkCore`:**

```csharp
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
        // Resolve the concrete DbContext (not via repository) so we can call Database.MigrateAsync().
        await _serviceProvider
            .GetRequiredService<BillingDbContext>()
            .Database
            .MigrateAsync();
    }
}
```

**Wire it into the central migrator** (typical `*DbMigrationService` in the `DbMigrator` host):

```csharp
private readonly IEnumerable<IBillingDbSchemaMigrator> _billingMigrators;
// ...alongside the other modules' IXxxDbSchemaMigrator collections

foreach (var migrator in _billingMigrators)
{
    await migrator.MigrateAsync();
}
```

Because each module owns its own `IXxxDbSchemaMigrator` and each `DbContext` writes to its own `__EFMigrationsHistory_<Module>` table, the three modules can be migrated independently and in any order during deploy.

---

## 9. Host wiring

In the Web/Host module:

```csharp
[DependsOn(
    typeof(BillingEntityFrameworkCoreModule),
    typeof(ModuleAEntityFrameworkCoreModule),
    typeof(ModuleBEntityFrameworkCoreModule),
    typeof(AbpEntityFrameworkCorePostgreSqlModule)
)]
public class AcmeWebModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        // Global default - each module overrides its own DbContext above with its own
        // MigrationsHistoryTable, so this is just a safe fallback.
        Configure<AbpDbContextOptions>(options =>
        {
            options.UseNpgsql();
        });
    }
}
```

`appsettings.json` (Postgres connection strings - separate entries make future split-out trivial):

```json
{
  "ConnectionStrings": {
    "Default":   "Host=localhost;Port=5432;Database=AcmeApp;Username=acme;Password=...",
    "Billing":   "Host=localhost;Port=5432;Database=AcmeApp;Username=acme;Password=..."
  }
}
```

You can point `Billing` at a totally different database later without changing any code.

---

## 10. Warning: do NOT share a DbContext across modules

It is tempting to add `DbSet<Invoice>` to an existing module's `DbContext` "for convenience". Don't. The official `abp-ef-core` and `abp-module` skills are explicit that each module is database-agnostic and self-contained. Concrete consequences when you share a `DbContext`:

- **Migrations collide.** Two modules adding migrations against the same context produce two divergent `ModelSnapshot.cs` files in two assemblies. EF Core only supports one snapshot per context. The next `dotnet ef migrations add` in either module emits a migration that drops the other module's tables.
- **One `__EFMigrationsHistory` table for two owners.** You lose the ability to deploy/roll back modules independently. A failed Billing migration blocks Module A and vice versa.
- **Cross-module coupling.** Module A's `DbContext` ends up referencing `Acme.Billing.Domain`, breaking the dependency rule that EF Core projects only reference their own Domain (`abp-dependency-rules`).
- **Repository scope leaks.** `EfCoreRepository<SharedDbContext, Invoice, Guid>` from Module A could now mutate Billing aggregates outside any Billing application service, defeating DDD invariants (`abp-ef-core`: "Inject `DbContext` in application/domain services" is a Don't).
- **Connection-string lock-in.** A shared context can only have one `[ConnectionStringName]`, so you can never relocate Billing to its own Postgres database.

The correct pattern is exactly what is described above: **one `DbContext` per module, one `Migrations/` folder per module, one `__EFMigrationsHistory_<Module>` table per module, one `IXxxDbSchemaMigrator` per module, with the central `DbMigrator` calling them all.**

---

## Files you will add or touch (summary)

| File | Purpose |
|------|---------|
| `Acme.Billing.Domain.Shared/BillingDbProperties.cs` | Table prefix, schema, connection-string name |
| `Acme.Billing.Domain/Data/IBillingDbSchemaMigrator.cs` | Migrator abstraction used by DbMigrator |
| `Acme.Billing.EntityFrameworkCore/BillingDbContext.cs` | The module's `AbpDbContext` |
| `Acme.Billing.EntityFrameworkCore/BillingDbContextModelCreatingExtensions.cs` | Fluent entity configuration |
| `Acme.Billing.EntityFrameworkCore/BillingDbContextFactory.cs` | `IDesignTimeDbContextFactory<BillingDbContext>` |
| `Acme.Billing.EntityFrameworkCore/EntityFrameworkCoreBillingDbSchemaMigrator.cs` | EF implementation of the migrator |
| `Acme.Billing.EntityFrameworkCore/BillingEntityFrameworkCoreModule.cs` | `AddAbpDbContext` + `UseNpgsql` + `MigrationsHistoryTable("__EFMigrationsHistory_Billing")` |
| `Acme.Billing.EntityFrameworkCore/Migrations/*` | Module-scoped EF migrations |
| Host `AcmeWebModule.cs` | `[DependsOn(typeof(BillingEntityFrameworkCoreModule))]` |
| `DbMigrator/*DbMigrationService.cs` | Inject and run `IBillingDbSchemaMigrator` alongside existing migrators |
| `appsettings.json` (Host + DbMigrator) | Add a `"Billing"` connection string |
