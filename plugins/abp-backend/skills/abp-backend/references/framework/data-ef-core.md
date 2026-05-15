# Data Access — Entity Framework Core

> Canonical reference for ABP 10.3 Entity Framework Core integration - AbpDbContext<T>, repositories, AddAbpDbContext registration, ReplaceDbContext consolidation, AbpDbContextOptions / AbpEntityOptions configuration, the ConfigureByConvention + ConfigureXxx mapping pattern, DBMS provider switching (SQL Server / PostgreSQL / MySQL / SQLite / Oracle), and the migration project layout used by every ABP startup template.

## When to load this reference

- User asks how to define a DbContext in an ABP project ('inherit from AbpDbContext<T>', '[ConnectionStringName]', AddAbpDbContext registration)
- User asks how repositories work in EF Core - default vs custom, IRepository<TEntity, TKey> vs EfCoreRepository<TDbContext, TEntity, TKey>, AddDefaultRepositories vs AddRepository
- User asks how to map an entity in OnModelCreating ('what does ConfigureByConvention do?', 'why are my soft-delete / audit columns missing?', table-prefix/schema setup)
- User asks how to switch the database provider (SQL Server -> PostgreSQL / MySQL / Oracle / SQLite) on an existing ABP solution
- User asks how to add a migration in an ABP solution, why their migration project has a separate IDesignTimeDbContextFactory, or how DbMigrator works
- User hits the 'multiple DbContexts in development, single DbContext in production' question and needs the [ReplaceDbContext] pattern
- User asks how to do bulk insert/update/delete in ABP EF Core (IEfCoreBulkOperationProvider)
- User asks how to configure split queries, query tracking behavior, lazy loading proxies, or other DbContextOptions across all ABP DbContexts (AbpDbContextOptions.Configure / PreConfigure)
- User asks how to use ExtraProperties / object extensions on EF Core entities (ObjectExtensionManager.MapEfCoreProperty)
- User asks how a repository can drop down to raw EF Core (GetDbContextAsync, GetDbSetAsync, ExecuteSqlRawAsync)
- User asks why their entity is automatically scoped to the current tenant or excluded by ISoftDelete - they want the ABP filter integration with EF Core
- User asks how to set the default eager-loading graph for an aggregate (AbpEntityOptions.Entity<T>().DefaultWithDetailsFunc)

**Audience:** Developers building any ABP 10.3 solution that uses Entity Framework Core (the default for the layered, single-layer, and microservice templates). Especially relevant to module authors, anyone designing an aggregate-and-repository pair, anyone switching DBMS, and anyone debugging migrations or OnModelCreating behavior.

## Key concepts

- AbpDbContext<TDbContext> (Volo.Abp.EntityFrameworkCore) - Base class every ABP DbContext derives from. Wires the framework into EF Core: injects LazyServiceProvider, EntityChangeEventHelper (publishes EntityCreated/Updated/Deleted local events on SaveChanges), GuidGenerator, AuditPropertySetter (fills CreationTime/CreatorId/LastModifier/etc.), ICurrentTenant, and IDataFilter (applies ISoftDelete + IMultiTenant filters). Reach for it whenever you create a DbContext.
- IEfCoreDbContext (Volo.Abp.EntityFrameworkCore) - Interface implemented by AbpDbContext<T>. The recommended best-practice is to declare a per-module interface IXxxDbContext : IEfCoreDbContext that exposes only aggregate-root DbSet<>'s (no setters) plus a [ConnectionStringName] attribute, then bind repositories to that interface so any concrete DbContext can satisfy them.
- [ConnectionStringName("Name")] (Volo.Abp.Data) - Decorate a DbContext (or its interface) to declare the logical connection-string name. ABP's IConnectionStringResolver looks up AbpDbConnectionOptions.ConnectionStrings[Name], honors Databases.MappedConnections groupings, and falls back to "Default". Tenant-aware resolution swaps in MultiTenantConnectionStringResolver when AbpMultiTenancyModule is loaded.
- AddAbpDbContext<TDbContext>(Action<IAbpDbContextRegistrationOptionsBuilder>) (Volo.Abp.EntityFrameworkCore) - Replacement for AddDbContext used inside ConfigureServices. Registers the DbContext, plugs it into ABP's UnitOfWork pipeline, and accepts the repository/DbContext-replacement options builder.
- IAbpDbContextRegistrationOptionsBuilder (Volo.Abp.EntityFrameworkCore.DependencyInjection) - Options builder passed to AddAbpDbContext. Methods: AddDefaultRepositories(includeAllEntities: false) to auto-register IRepository<TEntity[, TKey]> for every aggregate root in the DbContext, AddRepository<TEntity, TRepository>() for custom repositories, ReplaceDbContext<TInterfaceOrDbContext>(MultiTenancySides) to swap a module's DbContext at registration time.
- EfCoreRepository<TDbContext, TEntity, TKey> (Volo.Abp.Domain.Repositories.EntityFrameworkCore) - Default implementation of IRepository<TEntity, TKey> for EF Core. Inject IDbContextProvider<TDbContext> via constructor; expose GetDbContextAsync(), GetDbSetAsync(), and the standard CRUD/query methods. Inherit from it when writing a custom repository (e.g., BookRepository : EfCoreRepository<BookStoreDbContext, Book, Guid>, IBookRepository).
- IRepository<TEntity, TKey> / IRepository<TEntity> / IBasicRepository<...> / IReadOnlyRepository<...> (Volo.Abp.Domain.Repositories) - Generic data-access contracts. IRepository extends IQueryable so LINQ works directly. IReadOnlyRepository forces AsNoTracking. IBasicRepository excludes IQueryable for environments where it cannot be exposed (e.g., MongoDB). Inject these by default; only fall back to a custom repository when you have entity-specific behavior.
- IDbContextProvider<TDbContext> (Volo.Abp.EntityFrameworkCore) - Resolves the active DbContext for the current Unit of Work, applying connection-string resolution and DbContext replacement. Repositories receive it via constructor injection; rarely consumed directly.
- ConfigureByConvention() (Volo.Abp.EntityFrameworkCore.Modeling extension) - Extension method on EntityTypeBuilder<T> that maps ABP-base properties (Id, ConcurrencyStamp, ExtraProperties, audit columns, IsDeleted, TenantId) and applies global query filters for ISoftDelete + IMultiTenant. Call it once inside every Entity<T>(...) block in OnModelCreating.
- [ReplaceDbContext(typeof(IXxxDbContext), MultiTenancySides)] (Volo.Abp.EntityFrameworkCore) - Class-level attribute on a concrete AbpDbContext that announces it implements (and replaces) the named DbContext interface. The startup template's app-level DbContext stacks several of these so a single physical DbContext serves Identity, TenantManagement, FeatureManagement, etc. The MultiTenancySides argument (default Both) restricts replacement to host- or tenant-side requests.
- ReplaceDbContext (registration overload, IAbpDbContextRegistrationOptionsBuilder) - Equivalent to the attribute but configured at AddAbpDbContext time: options.ReplaceDbContext<IBookStoreDbContext>(). Use this when you cannot or do not want to put the attribute on the class.
- AbpDbContextOptions (Volo.Abp.EntityFrameworkCore) - Cross-cutting options bag configured inside ConfigureServices. Methods: Configure(opts => opts.UseSqlServer()) for the global default; Configure<TDbContext>(...) for per-context overrides; PreConfigure<TDbContext>(opts => opts.DbContextOptions.UseLazyLoadingProxies()) for native EF Core options that must run before UseXxx; ConfigureDefaultConvention / ConfigureConventions<T> / ConfigureDefaultOnModelCreating / ConfigureOnModelCreating<T> / ConfigureDefaultOnConfiguring / ConfigureOnConfiguring<T> for cross-cutting model and configuration hooks.
- AbpEntityOptions (Volo.Abp.Domain) - Per-entity options. Use options.Entity<TAggregate>(o => o.DefaultWithDetailsFunc = q => q.Include(...)) to centralize the eager-loading graph that WithDetailsAsync() returns for that aggregate, so callers stop hand-maintaining Includes.
- IEfCoreBulkOperationProvider (Volo.Abp.EntityFrameworkCore) - Pluggable provider invoked by InsertManyAsync / UpdateManyAsync / DeleteManyAsync on EfCoreRepository when bulk semantics matter. Register your own ITransientDependency-implementing version (typically wrapping EFCore.BulkExtensions or similar) to swap in real bulk SQL.
- ObjectExtensionManager.Instance.MapEfCoreProperty<TEntity, TProperty>(name, b => ...) (Volo.Abp.EntityFrameworkCore.Modeling) - Registers a strongly-typed extra property on an entity that already implements IHasExtraProperties. The mapping translates the ExtraProperties dictionary entry into a real EF Core column so it is queryable and migrate-able. Call inside the static constructor (or PreConfigureServices) of the EntityFrameworkCore module.
- IDesignTimeDbContextFactory<TDbContext> (Microsoft.EntityFrameworkCore.Design) - Standard EF Core hook the ABP templates implement (XxxDbContextFactory.cs in the .EntityFrameworkCore project). Returns a DbContext built against a hand-rolled appsettings.json so dotnet ef migrations add / Update-Database work without booting the full ABP module graph.
- DbMigrator project (XxxDbMigrator console app) - Convention from every ABP startup template. References the .EntityFrameworkCore module, runs IDataSeeder + EF Core Database.MigrateAsync at startup, and is the recommended way to apply migrations and seed data into dev/staging/prod databases. Replaces the use-it-once Update-Database call in real deployments.
- ConfigureXxx extension method pattern - Best practice from the EF Core best-practices page: never write entity configuration inside OnModelCreating directly. Instead each module ships a static class (e.g., BookStoreDbContextModelCreatingExtensions) with public static void ConfigureBookStore(this ModelBuilder builder) calls; the consuming app calls builder.ConfigureBookStore(); from its own OnModelCreating, alongside builder.ConfigureIdentity(), builder.ConfigureAuditLogging(), builder.ConfigureTenantManagement(), etc.
- AbpEntityFrameworkCoreModule (Volo.Abp.EntityFrameworkCore) - The base EF Core integration module. Every project's .EntityFrameworkCore module class should [DependsOn] this plus exactly one DBMS-provider module: AbpEntityFrameworkCoreSqlServerModule (default), AbpEntityFrameworkCorePostgreSqlModule, AbpEntityFrameworkCoreMySQLModule, AbpEntityFrameworkCoreMySQLPomeloModule, AbpEntityFrameworkCoreSqliteModule, AbpEntityFrameworkCoreOracleModule, or AbpEntityFrameworkCoreOracleDevartModule.
- SetMultiTenancySide(MultiTenancySides) (Volo.Abp.EntityFrameworkCore.Modeling, ModelBuilder extension) - Tells the per-module Configure helpers whether to emit host-only, tenant-only, or both sets of tables. Used in scenarios where the host database and a per-tenant database point to the same DbContext type but should expose different schemas.

## Configuration pattern

An EF Core integration in ABP 10.3 is wired across exactly four places: (a) the DbContext class, (b) the EntityFrameworkCore module, (c) the design-time DbContextFactory, (d) the appsettings.json connection strings.

(a) DbContext class - .EntityFrameworkCore/EntityFrameworkCore/MyAppDbContext.cs:

using Microsoft.EntityFrameworkCore;
using Volo.Abp.Data;
using Volo.Abp.EntityFrameworkCore;

[ConnectionStringName("Default")]
[ReplaceDbContext(typeof(IIdentityDbContext))]
[ReplaceDbContext(typeof(ITenantManagementDbContext))]
public class MyAppDbContext :
    AbpDbContext<MyAppDbContext>,
    IIdentityDbContext,
    ITenantManagementDbContext
{
    public DbSet<Book> Books { get; set; }

    public MyAppDbContext(DbContextOptions<MyAppDbContext> options) : base(options) { }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);

        // Module schemas - via extension methods, NOT direct config
        builder.ConfigurePermissionManagement();
        builder.ConfigureSettingManagement();
        builder.ConfigureBackgroundJobs();
        builder.ConfigureAuditLogging();
        builder.ConfigureFeatureManagement();
        builder.ConfigureIdentity();
        builder.ConfigureOpenIddict();
        builder.ConfigureTenantManagement();
        builder.ConfigureBlobStoring();

        // App entities - one block per aggregate
        builder.Entity<Book>(b =>
        {
            b.ToTable("AppBooks");
            b.ConfigureByConvention(); // ALWAYS
            b.Property(x => x.Name).IsRequired().HasMaxLength(BookConsts.MaxNameLength);
        });
    }
}

(b) EntityFrameworkCore module - .EntityFrameworkCore/EntityFrameworkCore/MyAppEntityFrameworkCoreModule.cs:

[DependsOn(
    typeof(AbpEntityFrameworkCoreModule),
    typeof(AbpEntityFrameworkCoreSqlServerModule), // swap per DBMS
    typeof(MyAppDomainModule)
)]
public class MyAppEntityFrameworkCoreModule : AbpModule
{
    public override void PreConfigureServices(ServiceConfigurationContext context)
    {
        // Register extra properties (object extensions) BEFORE ConfigureServices
        MyAppEfCoreEntityExtensionMappings.Configure();
    }

    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        context.Services.AddAbpDbContext<MyAppDbContext>(options =>
        {
            options.AddDefaultRepositories(includeAllEntities: true);
            options.AddRepository<Book, BookRepository>();
        });

        Configure<AbpDbContextOptions>(options =>
        {
            options.UseSqlServer();
            options.PreConfigure<MyAppDbContext>(opts =>
            {
                // Native EF Core knobs that must run before UseXxx
                // opts.DbContextOptions.UseLazyLoadingProxies();
            });
        });

        Configure<AbpEntityOptions>(options =>
        {
            options.Entity<Book>(o => o.DefaultWithDetailsFunc = q => q.Include(b => b.Authors));
        });
    }
}

(c) Design-time factory - .EntityFrameworkCore/EntityFrameworkCore/MyAppDbContextFactory.cs:

```csharp
public class MyAppDbContextFactory : IDesignTimeDbContextFactory<MyAppDbContext>
{
    public MyAppDbContext CreateDbContext(string[] args)
    {
        // Per provider - e.g., for PostgreSQL also set:
        // AppContext.SetSwitch("Npgsql.EnableLegacyTimestampBehavior", true);

        var configuration = BuildConfiguration();
        var builder = new DbContextOptionsBuilder<MyAppDbContext>()
            .UseSqlServer(configuration.GetConnectionString("Default"));
        return new MyAppDbContext(builder.Options);
    }

    private static IConfigurationRoot BuildConfiguration() =>
        new ConfigurationBuilder()
            .SetBasePath(Path.Combine(Directory.GetCurrentDirectory(), "../MyApp.DbMigrator/"))
            .AddJsonFile("appsettings.json", optional: false)
            .Build();
}
```

**Module DbContexts need a unique migrations history table — repeat the call in the factory.** The factory above is for a host-level `DbContext` and uses the default `__EFMigrationsHistory` table. When the same factory pattern is adapted for an internal ABP module (per Rule 6 in `SKILL.md` — each module owns its own DbContext + migrations + history table), the `UseSqlServer` / `UseNpgsql` call **must** include `MigrationsHistoryTable("__EFMigrationsHistory_<Module>")` in **both** the runtime registration AND the design-time factory. Otherwise `dotnet ef migrations add` writes to the default history table and EF treats other modules' migrations as missing on next startup. Module-DbContext factory shape:

```csharp
// modules/forms/src/Forms/Data/FormsDbContextFactory.cs
public class FormsDbContextFactory : IDesignTimeDbContextFactory<FormsDbContext>
{
    public FormsDbContext CreateDbContext(string[] args)
    {
        var configuration = BuildConfiguration();
        var builder = new DbContextOptionsBuilder<FormsDbContext>()
            .UseSqlServer(                                                    // or UseNpgsql / UseMySQL / etc.
                configuration.GetConnectionString("Default"),
                b => b.MigrationsHistoryTable("__EFMigrationsHistory_Forms")  // ← matches the runtime call
            );
        return new FormsDbContext(builder.Options);
    }
    // BuildConfiguration() climbs to the host's appsettings.json (path depends on module nesting).
}
```

The matching runtime registration in the module's `ConfigureServices` uses the same `MigrationsHistoryTable("__EFMigrationsHistory_Forms")` argument. Mismatch between runtime and factory is a silent foot-gun — `dotnet ef` writes one history table while the host reads from another.

(d) appsettings.json - in .DbMigrator and .Web/.HttpApi.Host:

{
  "ConnectionStrings": {
    "Default": "Server=(LocalDb)\\MSSQLLocalDB;Database=MyApp;Trusted_Connection=True"
  }
}

Key rules:
- Every Entity<T>(...) block must call ConfigureByConvention() so soft-delete, multi-tenancy, audit, ExtraProperties, and concurrency stamps map correctly.
- AddDefaultRepositories(includeAllEntities: false) registers IRepository<T> only for aggregate roots; pass true to include child entities.
- AddRepository overrides the default registration with your custom repository class.
- PreConfigure<TDbContext>(...) is required for EF Core options that must execute before UseXxx (lazy loading, query splitting in some cases).
- The design-time factory is mandatory; without it dotnet ef migrations cannot resolve a DbContext (the production graph is too heavy to boot in tooling).
- Switching DBMS = (1) swap the AbpEntityFrameworkCore.<Provider>Module dependency, (2) replace UseSqlServer with UseNpgsql / UseMySQL / UseMySql / UseSqlite / UseOracle in BOTH the module and the design-time factory, (3) update connection strings, (4) delete + regenerate migrations.

## Code examples

### Custom repository with raw SQL drop-down

_Aggregate-specific data access (DeleteBooksByType) plus a method that needs to break the abstraction and use ExecuteSqlRawAsync._

```csharp
// Domain layer - the contract
public interface IBookRepository : IRepository<Book, Guid>
{
    Task<List<Book>> GetListByTypeAsync(BookType type, CancellationToken cancellationToken = default);
    Task DeleteBooksByTypeAsync(BookType type);
}

// EntityFrameworkCore layer - the implementation
public class BookRepository
    : EfCoreRepository<MyAppDbContext, Book, Guid>, IBookRepository
{
    public BookRepository(IDbContextProvider<MyAppDbContext> dbContextProvider)
        : base(dbContextProvider) { }

    public async Task<List<Book>> GetListByTypeAsync(
        BookType type, CancellationToken cancellationToken = default)
    {
        var dbSet = await GetDbSetAsync();
        return await dbSet
            .Where(b => b.Type == type)
            .ToListAsync(GetCancellationToken(cancellationToken));
    }

    public async Task DeleteBooksByTypeAsync(BookType type)
    {
        var dbContext = await GetDbContextAsync();
        await dbContext.Database.ExecuteSqlRawAsync(
            "DELETE FROM AppBooks WHERE Type = {0}", (int)type);
    }
}

// Registration in EntityFrameworkCore module
context.Services.AddAbpDbContext<MyAppDbContext>(options =>
{
    options.AddDefaultRepositories(includeAllEntities: true);
    options.AddRepository<Book, BookRepository>();
});
```

**Key lines:** Inherit EfCoreRepository<TDbContext, TEntity, TKey>; constructor takes IDbContextProvider<TDbContext>; GetDbSetAsync()/GetDbContextAsync() expose EF Core; GetCancellationToken(token) falls back to ICancellationTokenProvider; AddRepository overrides the default IRepository<Book, Guid> registration so domain code that injects the interface still resolves to BookRepository.

### Switch SQL Server to PostgreSQL

_Existing solution created with -dbms SqlServer needs to run on PostgreSQL._

```csharp
// 1. .EntityFrameworkCore.csproj - replace package
// <PackageReference Include="Volo.Abp.EntityFrameworkCore.SqlServer" Version="10.3.x" />
// becomes
// <PackageReference Include="Volo.Abp.EntityFrameworkCore.PostgreSql" Version="10.3.x" />

// 2. EntityFrameworkCore module
[DependsOn(
    typeof(AbpEntityFrameworkCoreModule),
    typeof(AbpEntityFrameworkCorePostgreSqlModule), // was AbpEntityFrameworkCoreSqlServerModule
    typeof(MyAppDomainModule)
)]
public class MyAppEntityFrameworkCoreModule : AbpModule
{
    public override void PreConfigureServices(ServiceConfigurationContext context)
    {
        AppContext.SetSwitch("Npgsql.EnableLegacyTimestampBehavior", true);
    }

    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        context.Services.AddAbpDbContext<MyAppDbContext>(options =>
        {
            options.AddDefaultRepositories(includeAllEntities: true);
        });

        Configure<AbpDbContextOptions>(options =>
        {
            options.UseNpgsql();
        });
    }
}

// 3. DbContextFactory
public MyAppDbContext CreateDbContext(string[] args)
{
    AppContext.SetSwitch("Npgsql.EnableLegacyTimestampBehavior", true);

    var configuration = BuildConfiguration();
    var builder = new DbContextOptionsBuilder<MyAppDbContext>()
        .UseNpgsql(configuration.GetConnectionString("Default"));
    return new MyAppDbContext(builder.Options);
}

// 4. appsettings.json
// "Default": "Host=localhost;Port=5432;Database=MyApp;Username=postgres;Password=..."

// 5. Delete .EntityFrameworkCore/Migrations and regenerate
// dotnet ef migrations add Initial --project MyApp.EntityFrameworkCore --startup-project MyApp.DbMigrator
```

**Key lines:** [DependsOn] swap is the contract change - AbpEntityFrameworkCorePostgreSqlModule replaces AbpEntityFrameworkCoreSqlServerModule. UseNpgsql() must be set in BOTH the module and the design-time factory or migrations will use the wrong provider. The legacy-timestamp switch is PostgreSQL-specific and required by ABP audit columns that store DateTime (not DateTimeOffset). Migrations are provider-specific and must be regenerated.

### Object-extension extra property mapped as a real EF Core column

_Add a SocialSecurityNumber to IdentityUser without forking the Identity module, and have it persisted as a real column (queryable, indexable)._

```csharp
public static class MyAppEfCoreEntityExtensionMappings
{
    private static readonly OneTimeRunner OneTimeRunner = new OneTimeRunner();

    public static void Configure()
    {
        MyAppModuleExtensionConfigurator.Configure();

        OneTimeRunner.Run(() =>
        {
            ObjectExtensionManager.Instance
                .MapEfCoreProperty<IdentityUser, string>(
                    "SocialSecurityNumber",
                    (entityBuilder, propertyBuilder) =>
                    {
                        propertyBuilder.HasMaxLength(64);
                    });
        });
    }
}

// Called from PreConfigureServices of the EntityFrameworkCore module
public override void PreConfigureServices(ServiceConfigurationContext context)
{
    MyAppEfCoreEntityExtensionMappings.Configure();
}
```

**Key lines:** ObjectExtensionManager.Instance.MapEfCoreProperty<TEntity, TProperty>(name, builderAction) creates a real EF Core property whose value is read from / written to the entity's ExtraProperties dictionary. Call inside a OneTimeRunner from PreConfigureServices so the mapping is registered before AbpEntityFrameworkCoreModule builds the model. After running dotnet ef migrations add the column appears as a normal column (HasMaxLength(64) here).

### Custom IEfCoreBulkOperationProvider

_Replace the default per-row InsertManyAsync with EFCore.BulkExtensions for high-volume inserts._

```csharp
public class BulkExtensionsEfCoreBulkOperationProvider
    : IEfCoreBulkOperationProvider, ITransientDependency
{
    public async Task InsertManyAsync<TDbContext, TEntity>(
        IEfCoreRepository<TEntity> repository,
        IEnumerable<TEntity> entities,
        bool autoSave,
        CancellationToken cancellationToken)
        where TDbContext : IEfCoreDbContext
        where TEntity : class, IEntity
    {
        var dbContext = (DbContext)await repository.GetDbContextAsync();
        await dbContext.BulkInsertAsync(entities.ToList(), cancellationToken: cancellationToken);
        if (autoSave) await dbContext.SaveChangesAsync(cancellationToken);
    }

    public Task UpdateManyAsync<TDbContext, TEntity>(...) where ... { /* analogous */ }
    public Task DeleteManyAsync<TDbContext, TEntity>(...) where ... { /* analogous */ }
}
```

**Key lines:** Implementing IEfCoreBulkOperationProvider + ITransientDependency auto-replaces ABP's default. EfCoreRepository.InsertManyAsync routes through the provider, so application code calling _repository.InsertManyAsync(...) silently picks up bulk semantics.

### AbpEntityOptions for default eager-loading graph

_WithDetailsAsync() on Order should always include Lines and PaymentInfo without each caller hand-writing Includes._

```csharp
Configure<AbpEntityOptions>(options =>
{
    options.Entity<Order>(o =>
    {
        o.DefaultWithDetailsFunc = query => query
            .Include(x => x.Lines)
            .Include(x => x.PaymentInfo);
    });
});

// Application service usage
public async Task<OrderDto> GetAsync(Guid id)
{
    var order = await _orderRepository.GetAsync(id, includeDetails: true);
    // 'includeDetails: true' triggers DefaultWithDetailsFunc, so Lines/PaymentInfo are loaded
    return ObjectMapper.Map<Order, OrderDto>(order);
}
```

**Key lines:** DefaultWithDetailsFunc centralizes the include graph so the repository stays the source of truth. The boolean includeDetails parameter on GetAsync/GetListAsync/WithDetailsAsync drives whether the func runs. Combine with split queries (enabled by default in ABP) to avoid cartesian explosions.

## Common mistakes

- Forgetting ConfigureByConvention() inside an Entity<T>(...) block. Result: soft delete works (Filter is on the base type) but ExtraProperties is not mapped, ConcurrencyStamp is not configured, audit columns get default precision, and TenantId may not be filtered. Fix: call b.ConfigureByConvention() as the first line of every Entity block.
- Putting model configuration directly in OnModelCreating. The best-practices doc explicitly forbids it - it makes shared/modular DbContexts impossible because every consumer has to copy your code. Fix: write a public static void ConfigureXxx(this ModelBuilder builder) extension method and call builder.ConfigureXxx() from your DbContext's OnModelCreating.
- Enabling lazy loading proxies. The best-practices doc says to avoid lazy loading because it produces hidden N+1 queries and breaks across distributed boundaries. Fix: define a per-entity IncludeDetails(this IQueryable<T> q, bool include = true) extension method or use AbpEntityOptions.Entity<T>().DefaultWithDetailsFunc; let WithDetailsAsync(includeDetails: true) drive eager loads explicitly.
- Letting a custom repository depend on the concrete DbContext type instead of the DbContext interface. This couples it to a specific physical schema and breaks the [ReplaceDbContext] consolidation pattern. Fix: declare IXxxDbContext : IEfCoreDbContext with [ConnectionStringName] and bind EfCoreRepository<IXxxDbContext, ...>.
- Calling AddDbContext (Microsoft.EntityFrameworkCore) instead of AddAbpDbContext. The plain registration skips ABP's UoW pipeline, audit/property-setter, change-event publishing, and connection-string resolver - effectively disabling soft delete tenancy and audit. Fix: always use context.Services.AddAbpDbContext<TDbContext>(options => { ... }).
- Forgetting to call AddRepository<TEntity, TRepository>() when you write a custom repository. AddDefaultRepositories registers the default IRepository<T> binding; AddRepository overrides it. Without the override, IBookRepository resolves but the IRepository<Book, Guid> injection still hits the default repo - some methods then bypass your custom logic.
- Switching DBMS but only swapping UseSqlServer in the module and forgetting the design-time factory. dotnet ef tooling silently keeps using the old provider, so generated migration files are SQL-Server-flavored even after the runtime switches. Fix: change UseXxx in BOTH the module and the IDesignTimeDbContextFactory implementation.
- Not regenerating migrations after a DBMS switch. EF Core migrations contain provider-specific SQL fragments; running SQL-Server migrations against PostgreSQL fails on type mismatches (DateTime vs timestamptz, uniqueidentifier vs uuid). Fix: delete the Migrations folder, run dotnet ef migrations add Initial against the new provider, run DbMigrator.
- Switching to PostgreSQL without setting AppContext.SetSwitch("Npgsql.EnableLegacyTimestampBehavior", true) in both the module's PreConfigureServices and the DbContextFactory. Result: runtime errors saying DateTime values must be UTC because Npgsql 6+ rejects 'unspecified' Kind for timestamptz. Fix: set the switch in both places (PreConfigureServices runs at host startup; the design-time factory needs it for migrations).
- Trying to inject the DbContext directly into application services instead of using IRepository<T>. Breaks DDD layering, makes tests harder, and bypasses ABP's UoW transactions. Fix: inject IRepository<TEntity, TKey>, drop down to GetDbContextAsync only inside a custom repository.
- Calling _repository.GetQueryable() (sync). The 10.3 API is async and the sync version is gone in many overloads. Fix: await _repository.GetQueryableAsync(); for AsNoTracking use the IReadOnlyRepository<T> family or call .AsNoTracking() on the queryable result.
- Not putting [ReplaceDbContext] on the consolidated app DbContext when reusing pre-built modules' DbContext interfaces. Result: at runtime ABP resolves IIdentityDbContext to the module's own internal DbContext (separate connection, separate transaction, separate migration trail). Fix: stack the [ReplaceDbContext(typeof(IIdentityDbContext))] / [ReplaceDbContext(typeof(IOpenIddictDbContext))] / etc. attributes on your app DbContext, and have the class implement those interfaces.
## Version pins (ABP 10.3)

- ABP 10.3 specifics that may shift in later versions: (1) AbpDbContextOptions.UseSqlServer / UseNpgsql / UseMySQL / UseSqlite / UseOracle extension methods are stable but live in different Volo.Abp.EntityFrameworkCore.<Provider> packages - earlier minors had several spelling variants. (2) Split queries are enabled by default in 10.3; if a future version flips that default it could change query plans for aggregates with multiple Includes. [uncertain] - the docs describe 10.3 as enabling them globally but do not confirm whether the default survives unchanged into 10.4+. (3) The Volo.Abp.EntityFrameworkCore.MySQL.Pomelo package is the recommended path for new MySQL projects; the official Oracle MySQL connector path (Volo.Abp.EntityFrameworkCore.MySQL) is still supported but receives slower upstream updates. (4) Volo.Abp.EntityFrameworkCore.Oracle.Devart requires a paid Devart license; ABP integrates only and explicitly states it does not provide support for the underlying driver. (5) IEfCoreBulkOperationProvider's method signatures (TDbContext : IEfCoreDbContext, TEntity : class, IEntity) are stable in 10.3; earlier majors used different generic constraints. (6) ObjectExtensionManager.MapEfCoreProperty's signature is stable; the location of MyAppModuleExtensionConfigurator and MyAppEfCoreEntityExtensionMappings inside the EntityFrameworkCore project is a template convention, not a framework requirement. (7) [uncertain] - exact Npgsql major version pinned by Volo.Abp.EntityFrameworkCore.PostgreSql 10.3 is not stated on the page; the EnableLegacyTimestampBehavior switch is required for Npgsql 6+, which is the assumed baseline. (8) [uncertain] - whether DbContext pooling (AddDbContextPool) is officially supported by AddAbpDbContext in 10.3; the EF Core integration page mentions pooling concepts but does not document a first-class API, and ABP's UoW lifecycle traditionally conflicts with EF Core's pooling reset semantics.

## Cross-references

**Phase 1 references:**
- [references/framework/fundamentals.md](../framework/fundamentals.md) — Connection strings (AbpDbConnectionOptions, IConnectionStringResolver, [ConnectionStringName]), module lifecycle (PreConfigureServices/ConfigureServices/OnApplicationInitialization), [DependsOn], and IOptions resolution semantics; load when the user's question is about wiring connection strings, where AddAbpDbContext goes, or option-resolution timing rather than EF Core itself.
- [references/framework/architecture-ddd.md](../framework/architecture-ddd.md) — Aggregates, repositories as a DDD pattern, IUnitOfWork semantics, [UnitOfWork] attribute, when to write a custom repository, IDataFilter / ISoftDelete / IMultiTenant; load when the question is about repository design or unit-of-work boundaries rather than EF Core mechanics.
- [references/framework/architecture-modularity.md](../framework/architecture-modularity.md) — AbpModule dependency graph, [DependsOn], and how AbpEntityFrameworkCore.<Provider>Module fits into a module's [DependsOn] chain; load when the user is composing modules or troubleshooting dependency-graph order.
- [references/framework/multi-tenancy.md](../framework/multi-tenancy.md) — [IgnoreMultiTenancy], MultiTenantConnectionStringResolver, MultiTenancySides used in [ReplaceDbContext], and how IMultiTenant filtering is applied via ConfigureByConvention(); load when the question is about tenant data isolation, per-tenant databases, or host-vs-tenant DbContext scoping.
- [references/templates/single-layer.md / references/templates/layered.md / references/templates/microservice.md](../templates/single-layer.md / references/templates/layered.md / references/templates/microservice.md) — Each template ships an .EntityFrameworkCore project and a .DbMigrator console app whose layout is described here; load when the user asks where a particular EF Core file 'should live' in their solution.

**External docs:**
- [Entity Framework Core - Microsoft Docs](https://learn.microsoft.com/en-us/ef/core/) — Underlying ORM. ABP's AbpDbContext<T> is a plain DbContext subclass; everything in EF Core (DbSet, OnModelCreating, migrations, Include, AsNoTracking, query splitting) applies unchanged.
- [EF Core - Managing Database Schemas / Migrations](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/) — ABP uses standard dotnet ef migrations add / Update-Database commands; the centralized-migration model in ABP sits on top of these tools.
- [EF Core - Single vs Split Queries](https://learn.microsoft.com/en-us/ef/core/querying/single-split-queries) — ABP enables split queries by default in 10.3 and exposes the toggle through AbpDbContextOptions.UseXxx(o => o.UseQuerySplittingBehavior(...)).
- [Npgsql Entity Framework Core Provider](https://www.npgsql.org/efcore/) — Underlying provider used by Volo.Abp.EntityFrameworkCore.PostgreSql. Documents the EnableLegacyTimestampBehavior switch ABP recommends.
- [Pomelo.EntityFrameworkCore.MySql](https://github.com/PomeloFoundation/Pomelo.EntityFrameworkCore.MySql) — Underlying provider used by Volo.Abp.EntityFrameworkCore.MySQL.Pomelo. Documents the ServerVersion.AutoDetect(...) requirement.
- [MySql.EntityFrameworkCore (Oracle official)](https://dev.mysql.com/doc/connector-net/en/connector-net-entityframework-core.html) — Underlying provider used by Volo.Abp.EntityFrameworkCore.MySQL (the official, non-Pomelo path).
- [Microsoft.EntityFrameworkCore.Sqlite](https://learn.microsoft.com/en-us/ef/core/providers/sqlite/) — Underlying provider used by Volo.Abp.EntityFrameworkCore.Sqlite.
- [Oracle EF Core provider](https://www.oracle.com/database/technologies/appdev/dotnet/odpcoremanagedef.html) — Underlying provider used by Volo.Abp.EntityFrameworkCore.Oracle (the official free driver path).
- [Devart dotConnect for Oracle](https://www.devart.com/dotconnect/oracle/) — Commercial provider used by Volo.Abp.EntityFrameworkCore.Oracle.Devart. ABP integrates only and does not provide support for Devart itself.
- [EFCore.BulkExtensions](https://github.com/borisdj/EFCore.BulkExtensions) — Common implementation source plugged in via IEfCoreBulkOperationProvider for true bulk SQL semantics.

## Sources (ABP 10.3)

- [https://abp.io/docs/10.3/framework/data](https://abp.io/docs/10.3/framework/data) — Overview of ABP's data-access stack: IRepository/IUnitOfWork abstractions, EF Core / MongoDB / Dapper providers, and the cross-cutting concerns (filtering, seeding, audit, multi-tenancy, connection strings) that sit on top
- [https://abp.io/docs/10.3/framework/data/entity-framework-core](https://abp.io/docs/10.3/framework/data/entity-framework-core) — Canonical EF Core integration page: AbpDbContext<T>, ConfigureByConvention, AddAbpDbContext, AddDefaultRepositories/AddRepository, EfCoreRepository, IEfCoreDbContextProvider, ReplaceDbContext, IEfCoreBulkOperationProvider, AbpDbContextOptions, AbpEntityOptions, GetDbContextAsync/GetDbSetAsync, ObjectExtensionManager.MapEfCoreProperty
- [https://abp.io/docs/10.3/framework/data/entity-framework-core/migrations](https://abp.io/docs/10.3/framework/data/entity-framework-core/migrations) — Centralized migration model: app-level DbContext consolidates module schemas via [ReplaceDbContext], ConfigureXxx extension methods on ModelBuilder, IDesignTimeDbContextFactory, DbMigrator project, multi-database migrations with -Context/-OutputDir, SetMultiTenancySide for host-vs-tenant tables
- [https://abp.io/docs/10.3/framework/data/entity-framework-core/other-dbms](https://abp.io/docs/10.3/framework/data/entity-framework-core/other-dbms) — Catalog of supported DBMS (SqlServer default, MySQL, MySQL.Pomelo, PostgreSQL, Oracle, Oracle-Devart, Sqlite) and the manual-switch checklist (package replace, [DependsOn] swap, UseXxx call, connection-string format, migration regeneration)
- [https://abp.io/docs/10.3/framework/data/entity-framework-core/postgresql](https://abp.io/docs/10.3/framework/data/entity-framework-core/postgresql) — Volo.Abp.EntityFrameworkCore.PostgreSql + Npgsql, AbpEntityFrameworkCorePostgreSqlModule, UseNpgsql() in module + DbContextFactory, Npgsql.EnableLegacyTimestampBehavior switch
- [https://abp.io/docs/10.3/framework/data/entity-framework-core/mysql](https://abp.io/docs/10.3/framework/data/entity-framework-core/mysql) — Volo.Abp.EntityFrameworkCore.MySQL (official) vs Volo.Abp.EntityFrameworkCore.MySQL.Pomelo, UseMySQL() vs UseMySql(connStr, ServerVersion.AutoDetect(...)), AbpEntityFrameworkCoreMySQLModule / AbpEntityFrameworkCoreMySQLPomeloModule
- [https://abp.io/docs/10.3/framework/data/entity-framework-core/sqlite](https://abp.io/docs/10.3/framework/data/entity-framework-core/sqlite) — Volo.Abp.EntityFrameworkCore.Sqlite, AbpEntityFrameworkCoreSqliteModule, UseSqlite(), Filename=... connection string format
- [https://abp.io/docs/10.3/framework/data/entity-framework-core/oracle](https://abp.io/docs/10.3/framework/data/entity-framework-core/oracle) — Two integration packages: Volo.Abp.EntityFrameworkCore.Oracle (official free driver) and Volo.Abp.EntityFrameworkCore.Oracle.Devart (commercial); ABP integrates only, does not support the underlying driver
- [https://abp.io/docs/10.3/framework/architecture/best-practices/data-access-overview](https://abp.io/docs/10.3/framework/architecture/best-practices/data-access-overview) — High-level pointer page: data-access best practices are split across the EF Core integration and MongoDB integration sub-pages and follow DDD principles
- [https://abp.io/docs/10.3/framework/architecture/best-practices/entity-framework-core-integration](https://abp.io/docs/10.3/framework/architecture/best-practices/entity-framework-core-integration) — Best-practice rules: DbContext interface per module ([ConnectionStringName] + IEfCoreDbContext, only aggregate-root DbSets, no setters), repositories on the DbContext interface (not concrete), per-entity IncludeDetails extension methods, no lazy loading, no model config in OnModelCreating (use ConfigureXxx extension), always call ConfigureByConvention(), TablePrefix/Schema constants, GetCancellationToken() helper

Last verified: 2026-05-10
