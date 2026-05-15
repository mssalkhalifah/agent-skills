# Architecture — Modularity & Module Extension

> How ABP composes an application from independent modules (AbpModule + [DependsOn]), loads them statically or as runtime plug-ins, and how downstream solutions extend or override module entities, services, and UI without forking source code.

## When to load this reference

- How do I create a new ABP module / what does AbpModule's ConfigureServices vs PreConfigureServices vs OnApplicationInitialization do?
- How do I declare a dependency on another ABP module so its services / migrations / UI are wired in?
- How do I add a custom property (e.g. SocialSecurityNumber, DepartmentId) to IdentityUser or another module's entity and have it show up in the create/edit form, the data table, the API, and the database?
- How do I replace or partially override an application service / domain manager / repository / controller from a pre-built ABP module (e.g. IdentityUserAppService, IdentityUserManager)?
- How do I customize a Razor page / Blazor component / Angular component shipped by an ABP module without editing module source?
- How do I load modules as plug-ins from a folder at runtime (FolderPlugInSource / FilePlugInSource / TypePlugInSource)?
- Should I use abp add-module --with-source-code, package references, or a separated module solution? Which preserves upgradability?
- How do I store an extra property as a real EF Core column instead of in the ExtraProperties JSON blob?

**Audience:** ABP developers depending on pre-built application modules (Identity, Tenant Management, Account, etc.) who need to extend or customize them; module authors composing larger systems out of reusable AbpModule classes; teams evaluating plug-in architectures for ABP-based products.

## Key concepts

- AbpModule (Volo.Abp.Modularity.AbpModule) — abstract base class every module inherits. Override ConfigureServices/PreConfigureServices/PostConfigureServices for DI wiring; OnPreApplicationInitialization/OnApplicationInitialization/OnPostApplicationInitialization for startup work (e.g. middleware); OnApplicationShutdown for cleanup. Reach for it whenever you create a new project that participates in an ABP solution.
- [DependsOn] (Volo.Abp.Modularity.DependsOnAttribute) — declares direct module dependencies on the module class. Transitive dependencies are resolved automatically by the framework's dependency graph. Reach for it on every AbpModule to import another module's services, options, and assemblies.
- [AdditionalAssembly] (Volo.Abp.Modularity.AdditionalAssemblyAttribute) — registers extra assemblies that aren't directly referenced by the module's project. Reach for it when a module spans multiple .csproj files and you need ABP's conventional registration to scan them.
- IPlugInSource (Volo.Abp.Modularity.PlugIns.IPlugInSource) — strategy interface for discovering plug-in module assemblies. Built-in implementations: FolderPlugInSource (scan a directory), FilePlugInSource (explicit assembly paths), TypePlugInSource (already-loaded module types). Reach for it when modules must be loaded dynamically without compile-time references.
- AbpApplicationCreationOptions.PlugInSources — collection populated during AddApplicationAsync<T>(...) to register IPlugInSource instances. Reach for it in Program.cs / startup to enable runtime plug-in loading.
- ObjectExtensionManager (Volo.Abp.ObjectExtending.ObjectExtensionManager) — singleton (ObjectExtensionManager.Instance) used to declare extra properties on existing entities/DTOs of dependent modules. Provides Modules() fluent API and AddOrUpdateProperty<T>. Reach for it whenever you need to add a field to a pre-built module's entity without editing its source.
- OneTimeRunner (Volo.Abp.Threading.OneTimeRunner) — guarantees an Action runs once across the application lifecycle. Reach for it inside *ModuleExtensionConfigurator static methods so configuration isn't repeated when the module type is loaded multiple times (tests, plug-ins).
- IHasExtraProperties (Volo.Abp.Data.IHasExtraProperties) — interface implemented by all pre-built module aggregate roots. Exposes the ExtraProperties dictionary plus SetProperty / GetProperty<T> extension methods. Reach for it when you need ad-hoc per-entity data without schema changes.
- MapEfCoreProperty<TEntity, TProperty> — extension on ObjectExtensionManager that promotes an extra property to a real EF Core column (with HasMaxLength, HasIndex, etc.). Reach for it when you need indexes, foreign keys, or performant LINQ filtering on the extra field.
- [Dependency(ReplaceServices = true)] (Volo.Abp.DependencyInjection.DependencyAttribute) — instructs the DI container to remove existing registrations of the exposed service. Reach for it on a custom class that fully replaces a module-supplied service.
- [ExposeServices] (Volo.Abp.DependencyInjection.ExposeServicesAttribute) — explicitly lists the service types under which a class is registered, bypassing naming conventions. Use IncludeDefaults = true to keep the default conventional registrations alongside the explicit ones. Reach for it whenever your override class name doesn't match the service it's replacing.
- Module Extension Configurator (project-specific *ModuleExtensionConfigurator class in *.Domain.Shared) — convention class containing static ConfigureExtraProperties() / Configure() methods invoked from the Domain.Shared module's PreConfigureServices, wrapped in OneTimeRunner. Reach for it as the canonical place to declare entity extensions.
- Virtual File System (Volo.Abp.VirtualFileSystem) — embeds Razor/Blazor/JS/CSS module assets as virtual files; an application can override any of them by providing a same-named physical file. Reach for it to override module UI without editing the module project.
- Entity / DTO Extensions (AddOrUpdateProperty<TObject, TProperty>) — DTO-side counterpart to entity extensions; ensures extra properties survive entity-to-DTO mapping. Pair with CheckPairDefinitionOnMapping=false when one side has properties the other doesn't.
- Local / Distributed event handlers (ILocalEventHandler<EntityChangedEventData<T>> / IDistributedEventHandler<...Eto>) — used to keep a separate-table extension entity in sync with the source module's aggregate. Reach for these when extra properties grow into a full sub-model owned by your code.

## Configuration pattern

Modules are wired up by overriding the AbpModule lifecycle hooks. PreConfigureServices runs first and is the place to call PreConfigure<TOptions>(...) so options are visible to other modules' ConfigureServices. ConfigureServices is the standard DI configuration phase — register your services, call services.AddAbpDbContext<T>(...), Configure<TOptions>(...), and module-feature setup like Configure<AbpAutoMapperOptions>(...). PostConfigureServices runs after all ConfigureServices and is the place to inspect/finalize configuration. OnApplicationInitialization is the equivalent of Startup.Configure — resolve IApplicationBuilder via context.GetApplicationBuilder() and add middleware (UseAuthentication, UseConfiguredEndpoints, etc.). Sample skeleton:

```csharp
[DependsOn(
    typeof(AbpAspNetCoreMvcModule),
    typeof(AbpAutofacModule),
    typeof(AbpIdentityWebModule),
    typeof(MyDomainModule)
)]
public class MyWebModule : AbpModule
{
    public override void PreConfigureServices(ServiceConfigurationContext context)
    {
        PreConfigure<IMvcBuilder>(mvcBuilder =>
        {
            mvcBuilder.AddApplicationPartIfNotExists(typeof(MyWebModule).Assembly);
        });
    }

    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        var configuration = context.Services.GetConfiguration();
        Configure<AbpNavigationOptions>(options => { /* menus */ });
        context.Services.AddAuthentication(/* ... */);
    }

    public override void OnApplicationInitialization(ApplicationInitializationContext context)
    {
        var app = context.GetApplicationBuilder();
        app.UseAbpRequestLocalization();
        app.UseAuthentication();
        app.UseAuthorization();
        app.UseConfiguredEndpoints();
    }
}
```

For entity extensions, the canonical pattern is a static ModuleExtensionConfigurator invoked from the Domain.Shared module's PreConfigureServices, wrapped in OneTimeRunner so it executes exactly once. For service overrides, decorate a class with [Dependency(ReplaceServices = true)] and [ExposeServices(typeof(IInterfaceBeingReplaced))]; conventional registration picks it up automatically. For plug-ins, configure AbpApplicationCreationOptions.PlugInSources during AddApplicationAsync<T>(...).

## Code examples

### Minimal module class with dependencies

_Define a new web module that depends on the AspNetCore MVC and Autofac integrations._

```csharp
using Volo.Abp.AspNetCore.Mvc;
using Volo.Abp.Autofac;
using Volo.Abp.Modularity;

namespace Acme.BookStore.Web;

[DependsOn(
    typeof(AbpAspNetCoreMvcModule),
    typeof(AbpAutofacModule)
)]
public class BookStoreWebModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        // Register module-specific services or options here.
    }

    public override void OnApplicationInitialization(ApplicationInitializationContext context)
    {
        var app = context.GetApplicationBuilder();
        app.UseRouting();
        app.UseConfiguredEndpoints();
    }
}
```

**Key lines:** [DependsOn(...)] enumerates direct dependencies — ABP resolves transitive ones automatically. AbpAutofacModule swaps the default DI container for Autofac, which is required for property injection support used widely in ABP.

### Loading plug-in modules from a folder at runtime

_Discover and initialize any AbpModule classes whose assemblies are dropped into a known directory after deployment._

```csharp
var builder = WebApplication.CreateBuilder(args);

await builder.AddApplicationAsync<BookStoreWebModule>(options =>
{
    options.PlugInSources.AddFolder(
        Path.Combine(builder.Environment.ContentRootPath, "plug-ins"),
        SearchOption.AllDirectories);

    // Or: options.PlugInSources.AddFiles(new[] { "/abs/MyPlugin.dll" });
    // Or: options.PlugInSources.AddTypes(typeof(MyPluginModule));
});

var app = builder.Build();
await app.InitializeApplicationAsync();
await app.RunAsync();
```

**Key lines:** PlugInSources.AddFolder scans the directory at app startup for assemblies containing AbpModule subclasses; their ConfigureServices and OnApplicationInitialization run as if they had been [DependsOn]-referenced. Remember to delete .deps.json files from plug-in build outputs and copy any external dependencies alongside them.

### Adding a queryable extra property to IdentityUser

_Add a SocialSecurityNumber to the Identity module's user — visible in the UI, validated, and stored as a real EF Core column rather than the ExtraProperties JSON blob._

```csharp
// In <YourProject>.Domain.Shared / <YourProject>ModuleExtensionConfigurator.cs
public static class BookStoreModuleExtensionConfigurator
{
    private static readonly OneTimeRunner OneTimeRunner = new();

    public static void Configure()
    {
        OneTimeRunner.Run(() =>
        {
            ObjectExtensionManager.Instance.Modules()
                .ConfigureIdentity(identity =>
                {
                    identity.ConfigureUser(user =>
                    {
                        user.AddOrUpdateProperty<string>(
                            "SocialSecurityNumber",
                            property =>
                            {
                                property.Attributes.Add(new RequiredAttribute());
                                property.Attributes.Add(new StringLengthAttribute(64) { MinimumLength = 4 });
                                property.UI.OnTable.IsVisible = true;
                                property.UI.OnCreateForm.IsVisible = true;
                                property.UI.OnEditForm.IsVisible = true;
                            });
                    });
                });
        });
    }
}

// In <YourProject>.EntityFrameworkCore / <YourProject>EfCoreEntityExtensionMappings.cs
public static class BookStoreEfCoreEntityExtensionMappings
{
    private static readonly OneTimeRunner OneTimeRunner = new();

    public static void Configure()
    {
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
```

**Key lines:** ObjectExtensionManager.Instance.Modules().ConfigureIdentity(...).ConfigureUser(...) gives intellisense for every extensible entity in the Identity module. AddOrUpdateProperty<T> propagates the property to the entity, the create/edit DTO, the auto-generated UI, and the HTTP API. MapEfCoreProperty<IdentityUser, string>(...) promotes it from JSON storage into a real column — must be configured before DbContext initialization, then applied via standard EF Core migrations.

### Overriding an application service from a module

_Customize Identity's user creation logic by inheriting from IdentityUserAppService and adding extra behavior._

```csharp
using Volo.Abp.DependencyInjection;
using Volo.Abp.Identity;

[Dependency(ReplaceServices = true)]
[ExposeServices(
    typeof(IIdentityUserAppService),
    typeof(IdentityUserAppService),
    typeof(MyIdentityUserAppService)
)]
public class MyIdentityUserAppService : IdentityUserAppService
{
    public MyIdentityUserAppService(
        IdentityUserManager userManager,
        IIdentityUserRepository userRepository,
        IIdentityRoleRepository roleRepository,
        IOptions<IdentityOptions> identityOptions,
        IPermissionChecker permissionChecker)
        : base(userManager, userRepository, roleRepository, identityOptions, permissionChecker)
    {
    }

    public override async Task<IdentityUserDto> CreateAsync(IdentityUserCreateDto input)
    {
        // pre-create custom logic, e.g. validate SocialSecurityNumber
        var user = await base.CreateAsync(input);
        // post-create custom logic, e.g. publish a domain event
        return user;
    }
}
```

**Key lines:** [Dependency(ReplaceServices = true)] removes the default IIdentityUserAppService registration; [ExposeServices] registers the new class under each interface/base type so callers asking for the interface OR the concrete base both resolve to the override. The override calls base.CreateAsync to retain the framework's invariants — only methods marked virtual (which ABP application services almost always are by design) can be overridden this way.

### Reading and writing IHasExtraProperties at runtime

_Persist a per-user 'Title' value without any schema migration by leveraging the ExtraProperties JSON column built into every aggregate root._

```csharp
var user = await _identityUserRepository.GetAsync(userId);
user.SetProperty("Title", "Senior Engineer");
await _identityUserRepository.UpdateAsync(user);

var title = user.GetProperty<string>("Title");
```

**Key lines:** SetProperty/GetProperty are extension methods on IHasExtraProperties; the values are serialized into the ExtraProperties JSON column on the underlying table. No migration is required, but the property is hard to index, foreign-key, or LINQ-filter — promote it via MapEfCoreProperty when those are needed.

## Common mistakes

### Editing the source code of a pre-built ABP module directly (e.g. via abp add-module --with-source-code) just to add a single field.

**Why wrong:** Source-code customization breaks the upgrade path — abp update can no longer pull in upstream bug fixes and security patches without manual three-way merges.

```csharp
Use Module Entity Extensions (ObjectExtensionManager.Instance.Modules().Configure*) for new properties, [Dependency(ReplaceServices=true)] inheritance overrides for behavior, and the Virtual File System for UI tweaks. Reserve --with-source-code for genuinely deep modifications.
```

### Calling ObjectExtensionManager.Instance.Modules().Configure*(...) directly inside ConfigureServices without OneTimeRunner.

**Why wrong:** Configuration accumulates each time the module type is loaded — duplicated validators, duplicated UI columns, and surprising behavior in tests or under plug-in re-loading.

```csharp
Always wrap module-extension configuration in a static *ModuleExtensionConfigurator class with a private static OneTimeRunner, and call its Configure() once from the Domain.Shared module's PreConfigureServices.
```

### Forgetting MapEfCoreProperty when an extra property needs to be queried, indexed, or used in a foreign key.

**Why wrong:** By default the property lives inside the ExtraProperties JSON column. LINQ filters degrade to client-side evaluation, and you cannot define an index or FK on it.

```csharp
Add ObjectExtensionManager.Instance.MapEfCoreProperty<TEntity, TProperty>(name, (eb, pb) => pb.HasMaxLength(...).HasIndex(...)) in *EfCoreEntityExtensionMappings before DbContext initialization, then run dotnet ef migrations add.
```

### Replacing a service with [Dependency(ReplaceServices=true)] but forgetting [ExposeServices] when the class name doesn't follow conventions.

**Why wrong:** Conventional registration only replaces the service if the class is named after the interface (e.g. MyService implementing IMyService). A class named CustomLogicProvider implementing IIdentityUserAppService will not get registered as IIdentityUserAppService automatically, and the original implementation continues to win.

```csharp
Decorate with [ExposeServices(typeof(IIdentityUserAppService), typeof(IdentityUserAppService))] so both the interface and the concrete base type are bound to the override.
```

### Trying to override a non-virtual method on a module service.

**Why wrong:** Although ABP intentionally makes most service methods virtual, a few are not. Inheritance-based overrides will silently shadow rather than override, and the framework continues to call the base implementation through the original reference.

```csharp
If the method isn't virtual, replace the entire interface implementation with [Dependency(ReplaceServices=true)] + [ExposeServices(typeof(IInterface))] on a class that implements the interface from scratch (not inheriting), or open a feature request upstream to make the method virtual.
```

### Shipping a plug-in module with its .deps.json or with NuGet dependencies missing from the plug-in folder.

**Why wrong:** The host's runtime tries to satisfy assembly references using the plug-in's .deps.json, which references packages that are not present in the host's deployment, causing FileNotFoundException at module init.

```csharp
Delete .deps.json from the plug-in build output (or set GenerateDependencyFile=false in the .csproj) and copy any third-party DLLs the plug-in needs into the same plug-in folder so they load alongside the module assembly.
```

### Adding properties to entities but forgetting matching DTO extensions.

**Why wrong:** Mapping between entity and DTO drops the extra properties unless the DTO type is also registered with ObjectExtensionManager, or AutoMapper is configured with MapExtraProperties().

```csharp
Either register the property on both the entity and the DTO via AddOrUpdateProperty<TEntity, T>/<TDto, T>, or set CheckPairDefinitionOnMapping=false and rely on MapExtraProperties() in the AutoMapper profile so unmatched properties are carried over the ExtraProperties bag.
```

### Putting middleware registration (UseAuthentication, UseRouting) in ConfigureServices instead of OnApplicationInitialization.

**Why wrong:** ConfigureServices runs while the IServiceCollection is being built — there is no IApplicationBuilder yet — and middleware ordering can't be expressed there.

```csharp
Override OnApplicationInitialization, call var app = context.GetApplicationBuilder(); and add middleware in the same order you would in ASP.NET Core's Startup.Configure.
```

## Version pins (ABP 10.3)

- ABP 10.3 — confirmed AbpModule lifecycle method names and order (PreConfigureServices, ConfigureServices, PostConfigureServices, OnPreApplicationInitialization, OnApplicationInitialization, OnPostApplicationInitialization, OnApplicationShutdown). Async counterparts are present for each. [uncertain] whether PreConfigure<TOptions> on every options type remains the recommended path in 10.4; the basics page documents it explicitly for 10.3.
- ABP 10.3 — IPlugInSource implementations are FolderPlugInSource, FilePlugInSource, TypePlugInSource. The Razor-pages plug-in pattern (PartManager.ApplicationParts.Add(new AssemblyPart(...)) + new CompiledRazorAssemblyPart(...)) targets .NET 9 in the 10.3 docs; older .NET targets may differ. [uncertain] whether GenerateDependencyFile guidance changes in future SDKs.
- ABP 10.3 — ObjectExtensionManager.Instance.Modules() exposes ConfigureIdentity / ConfigureTenantManagement / etc. The set of modules you can configure depends on which ABP modules your solution depends on; new modules add new Configure* extension methods version over version.
- ABP 10.3 — MapEfCoreProperty<TEntity, TProperty> requires registration before the DbContext is initialized. The recommended location is the *EntityFrameworkCore project's *EfCoreEntityExtensionMappings.Configure() invoked from the EF Core module's PreConfigureServices. The MongoDB provider has an analogous MapEfCoreProperty-equivalent (MapObjectExtension*) — [uncertain] exact name retention across 10.x.
- ABP 10.3 — UI override mechanics differ per UI stack (MVC vs Blazor Server vs Blazor WebAssembly vs Angular). The official docs explicitly defer to per-framework guides; the Virtual File System backs MVC/Blazor overrides while Angular uses Replaceable Components. [uncertain] whether the Replaceable Components API in Angular has been renamed in a recent point release.
- ABP 10.3 — Default behavior of CheckPairDefinitionOnMapping in AbpAutoMapperOptions.AddMaps may flip in future versions; today it's strict by default and you must set false explicitly to skip pair checks for entity<->DTO extensions.
- ABP 10.3 — IncludeDefaults on [ExposeServices] defaults to false (i.e. specifying ExposeServices replaces the conventional registrations). Setting IncludeDefaults = true is only necessary when you want both your explicit list AND the convention-detected ones.

## Cross-references

**Phase 1 references:**
- [references/framework/architecture-ddd.md](../framework/architecture-ddd.md) — DDD layering tells you which project (Domain.Shared / Domain / EntityFrameworkCore / Application) each modularity artifact belongs in (ModuleExtensionConfigurator -> Domain.Shared, EfCoreEntityExtensionMappings -> EntityFrameworkCore, service overrides -> Application).
- [references/framework/dependency-injection.md](../framework/dependency-injection.md) — [Dependency(ReplaceServices=true)], [ExposeServices], conventional registration, and ITransientDependency/IScopedDependency/ISingletonDependency markers used throughout module composition.
- [references/framework/virtual-file-system.md](../framework/virtual-file-system.md) — Mechanism that backs UI overriding: how module-embedded Razor / Blazor / JS / CSS files can be replaced by same-path files in the consuming application.
- [references/framework/options.md](../framework/options.md) — Configure<TOptions> / PreConfigure<TOptions> patterns called from AbpModule.ConfigureServices to extend module-level options.
- [references/framework/data-extra-properties.md](../framework/data-extra-properties.md) — Deeper coverage of IHasExtraProperties, SetProperty/GetProperty, and how the ExtraProperties JSON column is materialized by EF Core and MongoDB providers.
- [references/templates/layered.md](../templates/layered.md) — Layered template ships a *ModuleExtensionConfigurator and *EfCoreEntityExtensionMappings out of the box; this reference explains what those classes are for.
- [references/templates/single-layer.md](../templates/single-layer.md) — Single-layer template collapses the configurator into the same project; cross-link explains the file's role even when the layout differs.

**External docs:**
- [Microsoft.Extensions.DependencyInjection](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection) — AbpModule lifecycle hooks ultimately register services into IServiceCollection; understanding the underlying DI semantics is required for advanced ConfigureServices/PostConfigureServices use.
- [EF Core Fluent API (HasMaxLength / HasIndex / HasOne)](https://learn.microsoft.com/en-us/ef/core/modeling/) — MapEfCoreProperty exposes the standard EF Core PropertyBuilder/EntityTypeBuilder, so column-level customization (length, indexes, conversions) follows the EF Core docs.
- [ASP.NET Core Application Parts / MvcBuilder.PartManager](https://learn.microsoft.com/en-us/aspnet/core/mvc/advanced/app-parts) — Plug-in modules that ship Razor pages must add AssemblyPart and CompiledRazorAssemblyPart to MvcBuilder.PartManager.ApplicationParts — pure ASP.NET Core mechanics.
- [Autofac dynamic proxy / property injection](https://autofac.readthedocs.io/en/latest/) — AbpAutofacModule is a near-mandatory dependency because ABP relies on Autofac's interception (dynamic proxies) for unit-of-work, authorization, audit, and caching attributes on virtual methods.
- [.NET Generic Host lifetime](https://learn.microsoft.com/en-us/dotnet/core/extensions/generic-host) — ApplicationInitializationContext / OnApplicationShutdown are wired into the generic host's IHostApplicationLifetime; understanding StopApplication / ApplicationStopping tokens helps when implementing graceful shutdown in modules.

## Sources (ABP 10.3)

- [https://abp.io/docs/10.3/framework/architecture/modularity/basics — AbpModule base class, lifecycle methods (PreConfigureServices/ConfigureServices/PostConfigureServices/OnApplicationInitialization/OnApplicationShutdown), [DependsOn] and [AdditionalAssembly] attributes, dependency graph resolution](https://abp.io/docs/10.3/framework/architecture/modularity/basics — AbpModule base class, lifecycle methods (PreConfigureServices/ConfigureServices/PostConfigureServices/OnApplicationInitialization/OnApplicationShutdown), [DependsOn] and [AdditionalAssembly] attributes, dependency graph resolution)
- [https://abp.io/docs/10.3/framework/architecture/modularity/plugin-modules — IPlugInSource interface, FolderPlugInSource/FilePlugInSource/TypePlugInSource, AbpApplicationCreationOptions.PlugInSources, runtime/dynamic module loading, Razor pages in plug-ins via PartManager.ApplicationParts](https://abp.io/docs/10.3/framework/architecture/modularity/plugin-modules — IPlugInSource interface, FolderPlugInSource/FilePlugInSource/TypePlugInSource, AbpApplicationCreationOptions.PlugInSources, runtime/dynamic module loading, Razor pages in plug-ins via PartManager.ApplicationParts)
- [https://abp.io/docs/10.3/framework/architecture/modularity/extending/customizing-application-modules-guide — Decision matrix for the four customization approaches (package reference, add-module --with-source-code, separated module solution, private re-publish), upgrade-safety implications, and routing into the four extension sub-pages](https://abp.io/docs/10.3/framework/architecture/modularity/extending/customizing-application-modules-guide — Decision matrix for the four customization approaches (package reference, add-module --with-source-code, separated module solution, private re-publish), upgrade-safety implications, and routing into the four extension sub-pages)
- [https://abp.io/docs/10.3/framework/architecture/modularity/extending/module-entity-extensions — ObjectExtensionManager.Modules() configurator, ConfigureExtraProperties + OneTimeRunner, AddOrUpdateProperty<T> with validation/UI/API/lookup configuration, automatic propagation to entity/DTO/DB/HTTP API/UI](https://abp.io/docs/10.3/framework/architecture/modularity/extending/module-entity-extensions — ObjectExtensionManager.Modules() configurator, ConfigureExtraProperties + OneTimeRunner, AddOrUpdateProperty<T> with validation/UI/API/lookup configuration, automatic propagation to entity/DTO/DB/HTTP API/UI)
- [https://abp.io/docs/10.3/framework/architecture/modularity/extending/customizing-application-modules-extending-entities — IHasExtraProperties + SetProperty/GetProperty, ObjectExtensionManager.MapEfCoreProperty<TEntity, TProperty> for real DB columns, separate-table pattern with ILocalEventHandler / IDistributedEventHandler](https://abp.io/docs/10.3/framework/architecture/modularity/extending/customizing-application-modules-extending-entities — IHasExtraProperties + SetProperty/GetProperty, ObjectExtensionManager.MapEfCoreProperty<TEntity, TProperty> for real DB columns, separate-table pattern with ILocalEventHandler / IDistributedEventHandler)
- [https://abp.io/docs/10.3/framework/architecture/modularity/extending/customizing-application-modules-overriding-services — [Dependency(ReplaceServices=true)] + [ExposeServices] for service replacement, inheritance-based override of application services / domain managers / repositories / controllers, ObjectExtensionManager DTO extension](https://abp.io/docs/10.3/framework/architecture/modularity/extending/customizing-application-modules-overriding-services — [Dependency(ReplaceServices=true)] + [ExposeServices] for service replacement, inheritance-based override of application services / domain managers / repositories / controllers, ObjectExtensionManager DTO extension)
- [https://abp.io/docs/10.3/framework/architecture/modularity/extending/overriding-user-interface — UI override approach varies by framework (MVC/Razor, Blazor, Angular); points to Virtual File System for replacing physical files; lists entity-action / data-table-column / page-toolbar / dynamic-form extension hooks](https://abp.io/docs/10.3/framework/architecture/modularity/extending/overriding-user-interface — UI override approach varies by framework (MVC/Razor, Blazor, Angular); points to Virtual File System for replacing physical files; lists entity-action / data-table-column / page-toolbar / dynamic-form extension hooks)

Last verified: 2026-05-10
