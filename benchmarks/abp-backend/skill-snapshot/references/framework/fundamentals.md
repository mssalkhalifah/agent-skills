# Framework fundamentals (DI, modules, config, options, exceptions, localization, logging)

> Canonical reference for ABP 10.3 cross-cutting framework primitives - module lifecycle, dependency injection, configuration, connection strings, options, exception handling, logging, localization, and object extensions - that every ABP service relies on regardless of template.

## When to load this reference

- User asks how to register or resolve a service in an ABP module (ITransientDependency, ISingletonDependency, IScopedDependency, [Dependency], [ExposeServices])
- User asks where to put startup code, what runs in PreConfigureServices vs ConfigureServices vs OnApplicationInitialization, or about [DependsOn]
- User asks about reading appsettings.json, IConfiguration, environment variables, or user secrets in an ABP app
- User asks how to wire connection strings for a module, multi-tenant database, or DbContext-to-connection-string mapping (AbpDbConnectionOptions, [ConnectionStringName])
- User asks how to configure framework options - 'how do I configure AbpAuditingOptions / AbpLocalizationOptions / AbpExceptionHandlingOptions' - or hits the IOptions resolution-timing pitfall
- User asks how to throw a domain error that returns 4xx instead of 500, error-code localization, BusinessException vs UserFriendlyException, or how to map an error code to a custom HTTP status
- User asks about logging - Serilog default sinks, UseAbpSerilogEnrichers, correlation/tenant/user enrichers, ILogger<T> injection, logs.txt location
- User asks how to localize a string in an application service, controller, or Razor page (IStringLocalizer<T>, the L property, JSON resource files, LocalizationResource attributes)
- User asks how to add custom fields to IdentityUser or any aggregate root without forking the module (ObjectExtensionManager, ExtraProperties, IHasExtraProperties)
- User asks 'why does Configure<MyOptions> not work in PreConfigureServices' or similar lifecycle/timing confusion
- User needs Autofac-only features (interceptors, dynamic proxying, property injection) and asks why their interceptor isn't firing

**Audience:** Any developer building on ABP 10.3 - applies equally to single-layer, layered, and microservice templates. Especially relevant to module authors, anyone wiring up new services, and developers debugging startup or DI behavior.

## Key concepts

- AbpApplicationFactory (Volo.Abp) - Static factory that boots a non-web ABP application from a startup module. Reach for it in console hosts, tests, or background workers. Methods: CreateAsync<TModule>(), CreateAsync(Type, IServiceCollection), Create<TModule>() and overloads accepting AbpApplicationCreationOptions (ApplicationName, Environment, PlugInSources, Services, Configuration).
- IAbpApplication (Volo.Abp) - Root container interface exposing StartupModuleType, Services (read-only after init), ServiceProvider, Modules, ApplicationName, InstanceId. Extends IApplicationInfoAccessor. Use it when you need the root provider; for non-singleton resolution always create a scope.
- AbpModule (Volo.Abp.Modularity) - Base class every ABP module derives from. Override PreConfigureServices, ConfigureServices, PostConfigureServices, OnPreApplicationInitialization, OnApplicationInitialization, OnPostApplicationInitialization, OnApplicationShutdown. Decorate with [DependsOn(typeof(OtherModule))] to declare module dependencies and force ordering.
- ServiceConfigurationContext (Volo.Abp.Modularity) - Argument passed to (Pre/Post)ConfigureServices. Exposes Services (IServiceCollection) and Configuration (IConfiguration). The canonical way to read appsettings during startup.
- IApplicationInitializationContext (Volo.Abp) - Argument passed to On(Pre/Post)ApplicationInitialization. Provides ServiceProvider and a GetEnvironment() helper. Use ApplicationInitializationContext.GetApplicationBuilder() inside ASP.NET Core hosts to register middleware (e.g., app.UseAbpSerilogEnrichers()).
- ITransientDependency / ISingletonDependency / IScopedDependency (Volo.Abp.DependencyInjection) - Marker interfaces. Implementing one auto-registers the class with the corresponding lifetime. Reach for ITransientDependency on stateless services (default for app/domain services), ISingletonDependency for caches/singletons, IScopedDependency for per-request state.
- [Dependency] attribute (Volo.Abp.DependencyInjection) - Lifetime/TryRegister/ReplaceServices overrides on a class. Use when the marker interfaces are insufficient (e.g., to replace an existing registration without changing the implementation type).
- [ExposeServices] / [ExposeKeyedService] (Volo.Abp.DependencyInjection) - Controls which interfaces a class exposes. Default conventional exposure: the class itself plus interfaces matching the I{Name} pattern. Use [ExposeServices(typeof(IFoo))] when convention does not match, or [ExposeKeyedService] for keyed registrations resolved via GetRequiredKeyedService<T>(key).
- IConfiguration (Microsoft.Extensions.Configuration) - Standard ASP.NET Core configuration root. ABP exposes it via ServiceConfigurationContext.Configuration during startup and via DI at runtime. ABP adds nothing on top - all sources (appsettings.json, env vars, user secrets, args) work as in vanilla ASP.NET Core.
- IConnectionStringResolver (Volo.Abp.Data) - Resolves the connection string for a given DbContext at runtime. Default = DefaultConnectionStringResolver (uses AbpDbConnectionOptions). MultiTenantConnectionStringResolver (Volo.Abp.MultiTenancy.ConnectionStrings) is layered on when AbpMultiTenancyModule is loaded; it picks tenant-specific strings before falling back.
- AbpDbConnectionOptions (Volo.Abp.Data) - Holds ConnectionStrings dictionary and Databases map. Configure inside ConfigureServices to add/override module-specific strings or to group multiple module names under one logical database.
- [ConnectionStringName] attribute (Volo.Abp.Data) - Decorate a DbContext (or MongoDB context) class to declare which logical name it expects from AbpDbConnectionOptions. Resolution order: module-specific name -> mapped database -> Default.
- PreConfigure<T> / Configure<T> (Volo.Abp.Modularity, AbpModule extension methods) - Configure<T> is the standard Microsoft.Extensions.Options call. PreConfigure<T> stores the configuration into a pre-options bag readable during ConfigureServices of subsequent modules - use it when an option must influence other DI registrations.
- IOptions<T> / IOptionsSnapshot<T> / IOptionsMonitor<T> (Microsoft.Extensions.Options) - Standard injection points. IOptions = singleton snapshot at first resolution; IOptionsSnapshot = per-scope, picks up reload; IOptionsMonitor = singleton with change-token notifications. ABP options classes follow this contract verbatim.
- BusinessException (Volo.Abp) - Base for expected business-rule failures. Implements IBusinessException, IHasErrorCode, IHasErrorDetails, IHasLogLevel (default Warning). Throw with a Code like "YourNamespace:010001" so the error message can be localized.
- UserFriendlyException (Volo.Abp) - Subclass of BusinessException that implements IUserFriendlyException. Its Message is sent to clients verbatim - use when you have already-formatted, end-user-safe text and do not want the localization round-trip.
- AbpExceptionHandlingOptions (Volo.Abp.AspNetCore.ExceptionHandling) - Toggles SendExceptionsDetailsToClients (default false) and SendStackTraceToClients (default true; both must be true for stack traces to ship).
- AbpExceptionLocalizationOptions (Volo.Abp.AspNetCore.ExceptionHandling) - Map error-code namespaces to LocalizationResource types via options.MapCodeNamespace. Lets BusinessException codes auto-resolve to localized resource keys.
- AbpExceptionHttpStatusCodeOptions (Volo.Abp.AspNetCore.ExceptionHandling) - options.Map("Code", HttpStatusCode.Conflict) overrides the framework's default code-to-status mapping (401/403/400/404/501/500).
- ExceptionSubscriber (Volo.Abp.ExceptionHandling) - Base class to plug into the framework's exception notification pipeline. Override HandleAsync to capture every caught exception (e.g., for telemetry or alerting). Subscriber failures are logged but do not propagate.
- ILogger<T> (Microsoft.Extensions.Logging) - Standard injection point for logging. ABP does not wrap or replace this. In templates, Serilog supplies the underlying provider via builder.Host.UseSerilog(...) in Program.cs.
- AbpSerilogEnrichers (Volo.Abp.AspNetCore.Serilog) - Middleware registered with app.UseAbpSerilogEnrichers() in OnApplicationInitialization to push CurrentTenant.Id, CurrentUser.Id, ClientId, and CorrelationId into Serilog's LogContext.
- IStringLocalizer<TResource> (Microsoft.Extensions.Localization) - Inject to localize strings keyed against a resource class. ABP base classes (ApplicationService, AbpController, AbpPageModel) expose the same lookup via the L property.
- IHtmlLocalizer<TResource> (Microsoft.Extensions.Localization) - HTML-aware variant for Razor views/pages.
- AbpLocalizationOptions (Volo.Abp.Localization) - Register resource classes (Resources.Add<TResource>("en")) and bind them to JSON files via AddVirtualJson("/Localization/Resources/..."). Set DefaultResourceType for fallback. Languages list supplies AbpLanguageInfo entries used by the language-switch UI.
- [LocalizationResourceName] / [InheritResource] (Volo.Abp.Localization) - Attribute the resource class with a short name (used by the JS/MVC client) and declare inheritance chains so child resources fall back to parent keys.
- ICurrentCulture (Volo.Abp.Localization) [uncertain] - Documented hook for setting the current culture programmatically; exact API surface in 10.3 not confirmed from the fetched fundamentals page.
- IHasExtraProperties (Volo.Abp.Data) - Single-property interface (ExtraPropertyDictionary ExtraProperties) that makes any object extensible. Implemented by AggregateRoot, ExtensibleEntityDto, ExtensibleAuditedEntityDto, ExtensibleObject.
- ObjectExtensionManager (Volo.Abp.ObjectExtending) - Singleton (ObjectExtensionManager.Instance). Call AddOrUpdate<T> / AddOrUpdateProperty<TProperty> at startup to declare extra properties so that EF Core mapping, validation, and DTO mapping pick them up.
- Volo.Abp.Autofac integration - Optional but recommended. Register via builder.Host.UseAutofac() (web) or options.UseAutofac() (console). Required for ABP's interception/dynamic proxy features (e.g., audit, UoW, validation interceptors that wrap virtual methods).

## Configuration pattern

Module lifecycle (deterministic across all modules in dependency-graph order):
1. PreConfigureServices(context) - run BEFORE every module's ConfigureServices. Use it to call PreConfigure<TOptions>(...) when later modules need to read those values during their own ConfigureServices.
2. ConfigureServices(context) - main DI registration phase. Call context.Services.Add..., Configure<T>(...), context.Services.AddAbpDbContext<T>(...), etc. Read context.Configuration here for appsettings values.
3. PostConfigureServices(context) - run AFTER every module's ConfigureServices. Use it for cross-module reconciliation (e.g., decorating registrations once everyone has registered).
4. OnPreApplicationInitialization(context) - first runtime phase. context.ServiceProvider is now resolvable.
5. OnApplicationInitialization(context) - main runtime wiring. In ASP.NET Core hosts use context.GetApplicationBuilder() to register middleware in order: routing, auth, AbpSerilogEnrichers, exception handler, etc.
6. OnPostApplicationInitialization(context) - final runtime hook (e.g., warmup tasks).
7. OnApplicationShutdown(context) - cleanup hook on graceful shutdown.

Canonical example wiring all the fundamentals at once:

[DependsOn(typeof(AbpAutofacModule), typeof(AbpAspNetCoreSerilogModule))]
public class MyAppModule : AbpModule
{
    public override void PreConfigureServices(ServiceConfigurationContext context)
    {
        // Set values consumed by other modules' ConfigureServices.
        PreConfigure<AbpAspNetCoreMvcOptions>(o => o.ConventionalControllers.Create(typeof(MyAppModule).Assembly));
    }

    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        var configuration = context.Configuration;

        // Connection strings
        Configure<AbpDbConnectionOptions>(options =>
        {
            options.ConnectionStrings.Default = configuration.GetConnectionString("Default");
            options.Databases.Configure("MainDb", db =>
            {
                db.MappedConnections.Add("AbpIdentity");
                db.MappedConnections.Add("AbpPermissionManagement");
            });
        });

        // Localization
        Configure<AbpLocalizationOptions>(options =>
        {
            options.Resources
                .Add<MyAppResource>("en")
                .AddVirtualJson("/Localization/Resources");
            options.Languages.Add(new LanguageInfo("en", "en", "English"));
            options.Languages.Add(new LanguageInfo("tr", "tr", "Turkce"));
        });

        // Exception handling
        Configure<AbpExceptionLocalizationOptions>(options =>
        {
            options.MapCodeNamespace("MyApp", typeof(MyAppResource));
        });
        Configure<AbpExceptionHttpStatusCodeOptions>(options =>
        {
            options.Map("MyApp:010002", HttpStatusCode.Conflict);
        });

        // Plain DI
        context.Services.AddTransient<ITaxCalculator, TaxCalculator>();
    }

    public override void OnApplicationInitialization(ApplicationInitializationContext context)
    {
        var app = context.GetApplicationBuilder();
        app.UseAbpSerilogEnrichers();
        app.UseAbpRequestLocalization();
        app.UseExceptionHandler("/Error");
        app.UseRouting();
        app.UseAuthentication();
        app.UseAuthorization();
        app.UseConfiguredEndpoints();
    }
}

Key rules:
- Configure<T> may be called from any module; values are merged by Microsoft.Extensions.Options.
- Resolving IOptions<T> inside (Pre)ConfigureServices is unsafe - the DI container is not yet built. Use PreConfigure<T>/the static accessors documented per options type, or defer the read to OnApplicationInitialization.
- ServiceConfigurationContext.Services is the same IServiceCollection across all modules; mutations are visible to subsequent modules.

## Code examples

### Console host bootstrap with Autofac

_Run an ABP module outside ASP.NET Core (console worker, integration test, CLI)._

```csharp
using Volo.Abp;
using Volo.Abp.Autofac;
using Volo.Abp.Modularity;

[DependsOn(typeof(AbpAutofacModule))]
public class MyConsoleModule : AbpModule { }

public static class Program
{
    public static async Task Main()
    {
        using var application = await AbpApplicationFactory.CreateAsync<MyConsoleModule>(options =>
        {
            options.UseAutofac();
            options.ApplicationName = "MyConsoleApp";
            options.Environment = Environments.Development;
        });
        await application.InitializeAsync();
        // ... resolve services, do work ...
        await application.ShutdownAsync();
    }
}
```

**Key lines:** AbpApplicationFactory.CreateAsync<TModule> bootstraps the container; options.UseAutofac() swaps the provider before InitializeAsync builds it; ShutdownAsync triggers OnApplicationShutdown across all modules.

### Throw a localized BusinessException with a stable error code

_Domain rule violation that should surface to clients as HTTP 409 with a translated message._

```csharp
// In your localization JSON (en.json):
// { "culture": "en", "texts": { "Shop:010002": "Order {orderId} is already paid." } }

throw new BusinessException("Shop:010002")
    .WithData("orderId", order.Id);

// In MyAppModule.ConfigureServices:
Configure<AbpExceptionLocalizationOptions>(o => o.MapCodeNamespace("Shop", typeof(ShopResource)));
Configure<AbpExceptionHttpStatusCodeOptions>(o => o.Map("Shop:010002", HttpStatusCode.Conflict));
```

**Key lines:** Code namespace prefix Shop: maps to ShopResource so the framework looks up the message there. WithData(...) supplies named parameters for the {orderId} placeholder. The status-code map overrides the default 403 BusinessException -> 409.

### Inject IStringLocalizer and use the L property

_Localize a message inside an application service - use either explicit injection or the inherited L property._

```csharp
public class OrderAppService : ApplicationService, IOrderAppService
{
    public OrderAppService()
    {
        LocalizationResource = typeof(ShopResource);
    }

    public string Greet(string name)
        => L["Welcome", name];
}

public class StandaloneGreeter : ITransientDependency
{
    private readonly IStringLocalizer<ShopResource> _localizer;
    public StandaloneGreeter(IStringLocalizer<ShopResource> localizer)
        => _localizer = localizer;

    public string Greet(string name) => _localizer["Welcome", name];
}
```

**Key lines:** Setting LocalizationResource in the ctor lets ApplicationService.L resolve against ShopResource. For non-ABP-base classes, inject IStringLocalizer<TResource> directly. Both forms accept format arguments matching {0}/{1} placeholders in the JSON.

### Add an extra property to IdentityUser without forking the module

_Persist a SocialSecurityNumber column on IdentityUser using ObjectExtensionManager._

```csharp
// In MyAppModule.PreConfigureServices (so EF Core picks it up at model-build time):
ObjectExtensionManager.Instance.Modules()
    .ConfigureIdentity(identity =>
    {
        identity.ConfigureUser(user =>
        {
            user.AddOrUpdateProperty<string>("SocialSecurityNumber", p =>
            {
                p.Attributes.Add(new StringLengthAttribute(32));
                p.MapEfCore(b => b.HasMaxLength(32));
            });
        });
    });

// Read/write at runtime:
user.SetProperty("SocialSecurityNumber", "123-45-6789");
var ssn = user.GetProperty<string>("SocialSecurityNumber");
```

**Key lines:** ObjectExtensionManager.Instance is a singleton mutated before app start. AddOrUpdateProperty<T> registers the property; MapEfCore configures the column shape. SetProperty/GetProperty go through the ExtraProperties dictionary on IHasExtraProperties.

### Multi-tenant connection strings and per-DbContext binding

_Map an isolated module to its own database while letting tenants override per-tenant._

```csharp
// appsettings.json
// "ConnectionStrings": {
//     "Default": "Server=...;Database=MainDb;...",
//     "AbpAuditLogging": "Server=...;Database=AuditDb;..."
// }

[ConnectionStringName("AbpAuditLogging")]
public class AuditLoggingDbContext : AbpDbContext<AuditLoggingDbContext>
{
    public AuditLoggingDbContext(DbContextOptions<AuditLoggingDbContext> options) : base(options) { }
}

Configure<AbpDbConnectionOptions>(options =>
{
    options.Databases.Configure("AnalyticsDb", db =>
    {
        db.MappedConnections.Add("AbpAuditLogging");
        db.MappedConnections.Add("AbpReporting");
    });
});

// Tenant-specific override (typed model varies by version, see version_pins):
// SaaS module persists tenant connection strings; MultiTenantConnectionStringResolver consults them automatically when AbpMultiTenancyModule is referenced.
```

**Key lines:** [ConnectionStringName] tells the resolver which logical name to look up. options.Databases.Configure groups module names so they share a single connection string. MultiTenantConnectionStringResolver layers tenant overrides on top of the default resolution chain.

## Common mistakes

### Trying to read IOptions<T> from inside ConfigureServices.

**Why wrong:** The DI container is not built until all modules finish configuring. Calling context.Services.BuildServiceProvider() works around this but creates a second container, leading to duplicate singletons and silently wrong values.

**Correct pattern:** Either move the read into OnApplicationInitialization (where ServiceProvider is real), or use PreConfigure<T>/the static option accessors that are designed to expose values during ConfigureServices.

```csharp
// WRONG
var opts = context.Services.BuildServiceProvider().GetRequiredService<IOptions<MyOptions>>().Value;
// RIGHT
public override void OnApplicationInitialization(ApplicationInitializationContext context)
{
    var opts = context.ServiceProvider.GetRequiredService<IOptions<MyOptions>>().Value;
}
```

### Marker interface plus manual context.Services.AddTransient<IFoo, Foo>().

**Why wrong:** Conventional registration already added Foo for every interface matching the I{Name} convention. The manual call adds a second registration; resolving IEnumerable<IFoo> now yields two instances.

**Correct pattern:** Pick one. If conventions cover you, drop the manual Add. If you need a non-conventional interface, use [ExposeServices(typeof(IBar))] or skip the marker interface entirely.

```csharp
// Either remove ITransientDependency, or remove the AddTransient line. Do not keep both.
```

### Throwing System.Exception (or InvalidOperationException) from an application service to signal a domain failure.

**Why wrong:** Anything that does not implement IBusinessException is treated as an infrastructure error and mapped to HTTP 500. Stack traces leak (because SendStackTraceToClients defaults to true). Logs noise up to Error level.

**Correct pattern:** Throw BusinessException with a stable code (and WithData for placeholders), or UserFriendlyException when the message is already user-safe and localized.

```csharp
throw new BusinessException("Shop:010002").WithData("orderId", id);
```

### Forgetting to register an error-code namespace -> resource mapping.

**Why wrong:** BusinessException("Foo:010001") will surface the literal code Foo:010001 to clients instead of the localized message.

**Correct pattern:** Configure<AbpExceptionLocalizationOptions>(o => o.MapCodeNamespace("Foo", typeof(FooResource))) once per namespace.

```csharp
Configure<AbpExceptionLocalizationOptions>(o => o.MapCodeNamespace("Foo", typeof(FooResource)));
```

### Defining extra properties via ObjectExtensionManager AFTER the application has initialized.

**Why wrong:** EF Core has already built its model and validation has already discovered attributes. Late-added properties never reach the database and are not validated.

**Correct pattern:** Call ObjectExtensionManager.Instance.AddOrUpdate(...) inside PreConfigureServices (or ConfigureServices at the latest) of your module so it runs before the EF Core integration module's OnApplicationInitialization.

```csharp
Move the AddOrUpdateProperty calls to PreConfigureServices.
```

### Skipping app.UseAbpSerilogEnrichers() and then wondering why TenantId / UserId are missing from logs.

**Why wrong:** The enricher middleware is what pushes ICurrentTenant.Id and ICurrentUser.Id into Serilog's LogContext. Without it, those fields are never on the log record - even if Serilog is otherwise configured.

**Correct pattern:** Add app.UseAbpSerilogEnrichers() inside OnApplicationInitialization, before any middleware that performs business logic so that early logs are also enriched.

```csharp
context.GetApplicationBuilder().UseAbpSerilogEnrichers();
```

### Relying on conventional DI registration while inheriting from a non-ABP base class.

**Why wrong:** ABP's auto-registration walks types implementing the marker interfaces or matching MVC/AppService/Repository conventions. Custom inheritance hierarchies that bypass these signals are not picked up.

**Correct pattern:** Either implement ITransientDependency on the leaf type, decorate with [Dependency], or call context.Services.AddTransient<...> explicitly.

```json
[Dependency(ServiceLifetime.Transient, ReplaceServices = true)] on the class, or implement ITransientDependency.
```

### Configuring connection strings only via raw Configuration without setting AbpDbConnectionOptions.

**Why wrong:** The default IConnectionStringResolver reads AbpDbConnectionOptions, not IConfiguration directly. ABP's default startup wires Configuration -> AbpDbConnectionOptions, but custom hosts that skip the default templates miss this binding.

**Correct pattern:** If you bootstrap the host yourself, explicitly Configure<AbpDbConnectionOptions>(o => o.ConnectionStrings.Default = configuration.GetConnectionString("Default"));

```csharp
Configure<AbpDbConnectionOptions>(o => o.ConnectionStrings.Default = context.Configuration.GetConnectionString("Default"));
```

### Forgetting [DependsOn(typeof(AbpAutofacModule))] but calling builder.Host.UseAutofac() in Program.cs.

**Why wrong:** UseAutofac swaps the provider, but ABP's interceptor-aware integration (and module-system Autofac registrations) only register if the AbpAutofacModule is in the dependency graph. Interceptors silently no-op.

**Correct pattern:** Add [DependsOn(typeof(AbpAutofacModule))] to the startup module AND keep UseAutofac in Program.cs.

```json
[DependsOn(typeof(AbpAutofacModule))] on the startup module.
```

## Version pins (ABP 10.3)

- ABP 10.3 docs base path is /docs/10.3/. Several deeper sections (e.g., logging detail) defer to Microsoft docs - any future ABP release that ships its own logging abstraction would change this delegation.
- AbpExceptionHandlingOptions defaults: SendExceptionsDetailsToClients=false, SendStackTraceToClients=true. The fact that stack traces require BOTH flags to be true is current 10.3 behavior; any single-flag simplification in a future major would break security expectations.
- AbpApplicationFactory.CreateAsync overloads accepting an AbpApplicationCreationOptions delegate (with ApplicationName, Environment, PlugInSources, Configuration, Services) are confirmed for 10.3. The historical sync Create<T>() entry point is still present but async is the documented path.
- Conventional DI registration covers AbpModule (singleton), MVC controllers (transient), application services (transient), repositories (transient), and domain services (transient). The list of conventions is stable since 7.x but new conventions occasionally land - re-verify against future release notes.
- Configure<AbpDbConnectionOptions> Databases.Configure(...) grouping API was introduced before 10.x and is current; the older single-string-per-module style still works as a fallback.
- [uncertain] ICurrentCulture API surface in 10.3 - the dedicated fundamentals/localization page references resource registration and language list management, but whether ICurrentCulture remains the preferred programmatic culture-switch API or whether Microsoft RequestLocalization is the only documented path is not unambiguously stated on the fetched page.
- [uncertain] Whether MultiTenantConnectionStringResolver is registered automatically by AbpMultiTenancyModule or only when an additional MultiTenancy.ConnectionStrings module is referenced - the connection-strings page describes the resolver but the exact module producing the registration in 10.3 was not pinned down from the fetched documentation.

## Cross-references

**Phase 1 references:**
- [references/framework/architecture-ddd.md](../framework/architecture-ddd.md) — triggered when the user asks how AbpModule lifecycle interacts with domain/application service registration; both files reference [DependsOn] and ApplicationService base class behavior.
- [references/framework/architecture-modularity.md](../framework/architecture-modularity.md) — triggered for module-extension/plugin scenarios; ObjectExtensionManager and module dependency basics flow into that reference.
- [references/framework/multi-tenancy.md](../framework/multi-tenancy.md) — triggered when discussing MultiTenantConnectionStringResolver or tenant-aware Serilog enrichers.
- [references/framework/authorization.md](../framework/authorization.md) — triggered when BusinessException + 401/403 mapping or AbpExceptionHttpStatusCodeOptions intersects with authorization failures.
- [references/framework/validation.md](../framework/validation.md) — triggered when discussing the validationErrors response shape or AbpExceptionFilter pipeline.
- [references/framework/api-development.md](../framework/api-development.md) — triggered when discussing AbpExceptionFilter, controller exception handling, and Swagger error responses.
- [references/framework/data-ef-core.md](../framework/data-ef-core.md) — triggered when [ConnectionStringName] / AbpDbConnectionOptions / MapEfCore (object extensions) come up in EF Core context.
- [references/templates/single-layer.md, references/templates/layered.md, references/templates/microservice.md](../templates/single-layer.md, references/templates/layered.md, references/templates/microservice.md) — triggered when a user asks 'where is Program.cs/UseSerilog wired in my template?' - those references show the per-template host wiring that consumes these fundamentals.

**External docs:**
- [ASP.NET Core Configuration](https://learn.microsoft.com/aspnet/core/fundamentals/configuration/) — ABP's configuration page explicitly defers to ASP.NET Core for sources, ordering, env vars, and user secrets.
- [ASP.NET Core Logging](https://learn.microsoft.com/aspnet/core/fundamentals/logging/) — ABP fundamentals/logging page states ABP implements no logging itself and uses ASP.NET Core's logging system - all ILogger<T> semantics come from there.
- [Microsoft.Extensions.Options](https://learn.microsoft.com/dotnet/core/extensions/options) — PreConfigure<T>/Configure<T> sit on top of the standard options pattern; IOptions/IOptionsSnapshot/IOptionsMonitor semantics are inherited unchanged.
- [Microsoft.Extensions.Localization](https://learn.microsoft.com/aspnet/core/fundamentals/localization) — IStringLocalizer<T>/IHtmlLocalizer<T> are the Microsoft contracts ABP implements; culture-fallback rules and request-localization middleware come from there.
- [Autofac Documentation](https://autofac.readthedocs.io/) — Volo.Abp.Autofac wraps Autofac; advanced container scenarios (modules, registration sources, interceptors) reference Autofac's own docs.
- [Serilog](https://serilog.net/) — Templates ship Serilog as the concrete logging provider behind Microsoft.Extensions.Logging. Sink and enricher concepts are Serilog terminology.
- [Volo.Abp.AspNetCore.Serilog package](https://abp.io/package-detail/Volo.Abp.AspNetCore.Serilog) — Source of UseAbpSerilogEnrichers and the tenant/user/correlation enricher set.

## Sources (ABP 10.3)

- [https://abp.io/docs/10.3/framework/fundamentals/application-startup](https://abp.io/docs/10.3/framework/fundamentals/application-startup) — AbpApplicationFactory, IAbpApplication, IAbpHostEnvironment, lifecycle initialization/shutdown sequence
- [https://abp.io/docs/10.3/framework/fundamentals/configuration](https://abp.io/docs/10.3/framework/fundamentals/configuration) — IConfiguration access via ServiceConfigurationContext.Configuration and constructor injection; ABP defers to ASP.NET Core configuration system
- [https://abp.io/docs/10.3/framework/fundamentals/connection-strings](https://abp.io/docs/10.3/framework/fundamentals/connection-strings) — IConnectionStringResolver, DefaultConnectionStringResolver, MultiTenantConnectionStringResolver, AbpDbConnectionOptions, [ConnectionStringName]
- [https://abp.io/docs/10.3/framework/fundamentals/dependency-injection](https://abp.io/docs/10.3/framework/fundamentals/dependency-injection) — ITransient/ISingleton/IScopedDependency, [Dependency], [ExposeServices], [ExposeKeyedService], conventional registration
- [https://abp.io/docs/10.3/framework/fundamentals/autofac-integration](https://abp.io/docs/10.3/framework/fundamentals/autofac-integration) — Volo.Abp.Autofac package, UseAutofac, GetContainerBuilder(), property injection / interceptor support
- [https://abp.io/docs/10.3/framework/fundamentals/options](https://abp.io/docs/10.3/framework/fundamentals/options) — PreConfigure<T>/Configure<T>, IOptions<T> access, when to use each phase
- [https://abp.io/docs/10.3/framework/fundamentals/exception-handling](https://abp.io/docs/10.3/framework/fundamentals/exception-handling) — BusinessException, UserFriendlyException, IBusinessException/IHasErrorCode/IHasErrorDetails/IHasLogLevel, AbpExceptionHandlingOptions, AbpExceptionLocalizationOptions, AbpExceptionHttpStatusCodeOptions, ExceptionSubscriber
- [https://abp.io/docs/10.3/framework/fundamentals/logging](https://abp.io/docs/10.3/framework/fundamentals/logging) — ABP defers entirely to ASP.NET Core logging; templates layer Serilog on top
- [https://abp.io/docs/10.3/framework/fundamentals/localization](https://abp.io/docs/10.3/framework/fundamentals/localization) — IStringLocalizer<TResource>, IHtmlLocalizer<T>, AbpLocalizationOptions, virtual JSON resources, [LocalizationResourceName], [InheritResource]
- [https://abp.io/docs/10.3/framework/fundamentals/object-extensions](https://abp.io/docs/10.3/framework/fundamentals/object-extensions) — IHasExtraProperties, ObjectExtensionManager, SetProperty/GetProperty/HasProperty/RemoveProperty, EF Core MapEfCore, MapExtraPropertiesTo
- [https://abp.io/docs/10.3/solution-templates/layered-web-application/logging](https://abp.io/docs/10.3/solution-templates/layered-web-application/logging) — Concrete Serilog wiring used by templates (Console + File + ABP Studio sinks, UseAbpSerilogEnrichers in OnApplicationInitialization)

Last verified: 2026-05-10
