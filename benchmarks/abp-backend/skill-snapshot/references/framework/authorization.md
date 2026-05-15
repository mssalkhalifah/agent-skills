# Authorization, Permissions & Current User

> Reference for ABP's authorization stack: defining and checking permissions, resource-based authorization, dynamic claims for mid-session updates, and accessing the current user/principal.

## When to load this reference

- User asks how to define a new permission or permission group in an ABP solution.
- User asks how to protect an application service / controller / Razor Page with [Authorize] or programmatic checks.
- User asks how to grant or revoke permissions to roles, users, or clients.
- User asks how to do per-instance access control (e.g., 'only the owner can edit this document').
- User asks why a role/permission change doesn't take effect until logout, or how to enable dynamic claims.
- User asks how to read the logged-in user's id, tenant id, claims, or roles from server-side code.
- User asks how to impersonate a user or run code as a different principal in background jobs / tests.
- User asks how to integrate a custom claim source (e.g., department, license tier) into authorization decisions.
- User asks why AlwaysAllowAuthorizationService is wired up in test projects.

**Audience:** Developers configuring authorization on ABP services, controllers, or pages; developers implementing custom permission providers, resource-based access control, or dynamic claim contributors; anyone reading ICurrentUser values or extending the claims principal.

## Key concepts

- PermissionDefinitionProvider (Volo.Abp.Authorization.Permissions) — abstract base class auto-discovered by ABP; override Define(IPermissionDefinitionContext) to add permission groups and permissions. Reach for it whenever a module needs to publish new permission names. Conventionally lives in the Application.Contracts project.
- IPermissionDefinitionContext (Volo.Abp.Authorization.Permissions) — passed into Define(); exposes AddGroup(name, displayName), GetGroup, GetPermissionOrNull. Reach for it to add or modify groups and permissions at startup.
- PermissionDefinition (Volo.Abp.Authorization.Permissions) — value type returned by AddPermission/AddChild; supports .RequireFeatures(), .RequireGlobalFeatures(), .WithProviders(), MultiTenancySide, IsEnabled. Reach for it to constrain when a permission is visible or grantable.
- IAuthorizationService (Microsoft.AspNetCore.Authorization) — ASP.NET Core service ABP extends with permission-aware policy evaluation; injected automatically into ApplicationService via the AuthorizationService property. Use AuthorizeAsync, CheckAsync (throws AbpAuthorizationException), or IsGrantedAsync.
- [Authorize] / [AllowAnonymous] (Microsoft.AspNetCore.Authorization) — declarative attribute applied to controllers, application services, or methods; ABP treats the policy name as a permission name. Reach for it whenever the check is static and method-scoped.
- IPermissionChecker (Volo.Abp.Authorization.Permissions) — lower-level service that resolves a permission name against the current principal across all PermissionValueProviders; AuthorizationService delegates here. Reach for it when you need to bypass policy evaluation and just ask 'is this permission granted?'.
- PermissionValueProvider (Volo.Abp.Authorization.Permissions) — abstract base class for contributing grant decisions. Built-ins: UserPermissionValueProvider, RolePermissionValueProvider, ClientPermissionValueProvider. Subclass and register via AbpPermissionOptions.ValueProviders to introduce custom signals (claim flags, license tier, etc.). Returns PermissionGrantResult.Granted/Prohibited/Undefined.
- IPermissionManager (Volo.Abp.PermissionManagement) — provided by the Permission Management module; SetForUserAsync/SetForRoleAsync/SetForClientAsync to grant or revoke permissions at runtime. Reach for it from seeders or admin UIs.
- IPermissionStore (Volo.Abp.Authorization.Permissions) — abstraction the checker calls to read persisted grants; default implementation comes from the Permission Management module and is cached.
- AbpAuthorizationOptions (Volo.Abp.Authorization) — options object configured via Configure<AbpAuthorizationOptions>(); manage policy-to-permission mapping and the resource-based handler list.
- AbpPermissionOptions (Volo.Abp.Authorization.Permissions) — registers PermissionValueProviders and tweaks DefinitionProviders order.
- AddResourcePermission(name, resourceName, managementPermissionName, displayName) — fluent extension on IPermissionDefinitionContext for declaring instance-level permissions. The resourceName is typically the entity full-type name (e.g. 'Acme.BookStore.Books.Book').
- IResourcePermissionChecker (Volo.Abp.Authorization.Permissions) — checks one or more resource permissions against a (resourceName, resourceKey) pair, returning per-permission PermissionGrantResult. Reach for it for batch checks or when you don't have the entity instance.
- IKeyedObject (Volo.Abp.ObjectExtending) — entities targeted by resource-based authorization expose GetObjectKey(); base entity types implement this automatically.
- ICurrentUser (Volo.Abp.Users) — strongly-typed read accessor for the current authenticated principal: Id, UserName, Email, EmailVerified, PhoneNumber, PhoneNumberVerified, TenantId, Roles, IsAuthenticated, plus FindClaim/FindClaims/GetAllClaims/IsInRole. Inject anywhere; pre-injected in ApplicationService as CurrentUser.
- ICurrentPrincipalAccessor (Volo.Abp.Security.Claims) — lower-level abstraction over the underlying ClaimsPrincipal. Use .Principal to read raw claims, and .Change(newPrincipal) inside a using block to temporarily run code as another user (background jobs, integration tests, audit-log replay).
- AbpClaimTypes (Volo.Abp.Security.Claims) — static class with constants for UserId, UserName, Email, EmailVerified, PhoneNumber, PhoneNumberVerified, Role, TenantId, EditionId, ClientId, etc. Use these instead of magic strings when constructing or reading claims.
- IAbpClaimsPrincipalFactory (Volo.Abp.Security.Claims) — produces the ClaimsPrincipal used by HttpContext.User; supports dynamic claims so claim values are recomputed per request when enabled.
- IAbpDynamicClaimsPrincipalContributor (Volo.Abp.Security.Claims) — implement ContributeAsync to inject claims at request time. Built-in implementations: IdentityDynamicClaimsPrincipalContributor (auth server, writes to distributed cache), RemoteDynamicClaimsPrincipalContributor (tiered UI, refreshes via HTTP from cache), WebRemoteDynamicClaimsPrincipalContributor (microservice gateways).
- AbpClaimsPrincipalFactoryOptions (Volo.Abp.Security.Claims) — IsDynamicClaimsEnabled, RemoteRefreshUrl (default '/api/account/dynamic-claims/refresh'), DynamicClaims list of claim types eligible for override, ClaimsMap mapping auth-server claim types to client claim types.
- WebRemoteDynamicClaimsPrincipalContributorOptions — IsEnabled (off by default) and AuthenticationScheme used when calling out to the auth server from a microservice.
- UseDynamicClaims() middleware — must run before UseAuthorization() so the principal is refreshed before policy evaluation.
- AlwaysAllowAuthorizationService (Volo.Abp.Authorization) — registered via context.Services.AddAlwaysAllowAuthorization() in test modules; short-circuits all permission checks. Pre-wired in the *.TestBase project of standard templates.

## Configuration pattern

Authorization is wired up at module load time. (1) Permission definitions live in PermissionDefinitionProvider subclasses inside *.Application.Contracts; ABP discovers them automatically — no manual registration required. (2) Custom value providers register through AbpPermissionOptions in ConfigureServices: `Configure<AbpPermissionOptions>(options => { options.ValueProviders.Add<SystemAdminPermissionValueProvider>(); });`. (3) Resource-based handlers register through AbpAuthorizationOptions when you go beyond AddResourcePermission and need a custom IAuthorizationHandler. (4) Dynamic claims are toggled and tuned via AbpClaimsPrincipalFactoryOptions in ConfigureServices, and the middleware is added in OnApplicationInitialization before UseAuthorization. (5) Tests opt out via AddAlwaysAllowAuthorization() in the *.TestBase module. Example wiring inside ConfigureServices:

```csharp
public override void ConfigureServices(ServiceConfigurationContext context)
{
    Configure<AbpPermissionOptions>(options =>
    {
        options.ValueProviders.Add<SystemAdminPermissionValueProvider>();
    });

    Configure<AbpClaimsPrincipalFactoryOptions>(options =>
    {
        options.IsDynamicClaimsEnabled = true;
        // For tiered/microservice UI tier:
        // options.RemoteRefreshUrl = configuration["AuthServer:Authority"] + options.RemoteRefreshUrl;
    });
}

public override void OnApplicationInitialization(ApplicationInitializationContext context)
{
    var app = context.GetApplicationBuilder();
    app.UseAuthentication();
    app.UseDynamicClaims();   // must come before UseAuthorization
    app.UseAuthorization();
}
```

Notes: `ICurrentUser` and `ICurrentPrincipalAccessor` need no registration — they are wired by Volo.Abp.Authorization / Volo.Abp.Security packages. Permission Management persistence (IPermissionStore, IPermissionManager) is provided by the Volo.Abp.PermissionManagement.* packages and is opt-in via module dependency, not by configuration here.

## Code examples

### Define a permission group with localized display names

_Publishing a new permission for an Author entity in the Application.Contracts project._

```csharp
using Volo.Abp.Authorization.Permissions;
using Volo.Abp.Localization;

namespace Acme.BookStore.Permissions;

public class BookStorePermissionDefinitionProvider : PermissionDefinitionProvider
{
    public override void Define(IPermissionDefinitionContext context)
    {
        var bookStoreGroup = context.AddGroup(
            BookStorePermissions.GroupName,
            LocalizableString.Create<BookStoreResource>("Permission:BookStore"));

        var authors = bookStoreGroup.AddPermission(
            BookStorePermissions.Authors.Default,
            LocalizableString.Create<BookStoreResource>("Permission:Authors"));

        authors.AddChild(
            BookStorePermissions.Authors.Create,
            LocalizableString.Create<BookStoreResource>("Permission:Authors.Create"));
        authors.AddChild(
            BookStorePermissions.Authors.Edit,
            LocalizableString.Create<BookStoreResource>("Permission:Authors.Edit"));
        authors.AddChild(
            BookStorePermissions.Authors.Delete,
            LocalizableString.Create<BookStoreResource>("Permission:Authors.Delete"));
    }
}
```

**Key lines:** AddGroup creates a logical grouping shown in the Permission Management UI. AddPermission returns a PermissionDefinition you can chain AddChild on for parent/child grants. Constants in BookStorePermissions keep the permission names DRY and refactor-safe.

### Declarative + programmatic permission checks in an ApplicationService

_An author CRUD service that uses [Authorize] for static cases and IAuthorizationService for conditional checks._

```csharp
using Microsoft.AspNetCore.Authorization;
using Volo.Abp.Application.Services;

public class AuthorAppService : ApplicationService, IAuthorAppService
{
    [Authorize(BookStorePermissions.Authors.Create)]
    public async Task<AuthorDto> CreateAsync(CreateAuthorDto input)
    {
        // ...
        return new AuthorDto();
    }

    [AllowAnonymous]
    public async Task<AuthorDto> GetAsync(Guid id)
    {
        // public read
        return new AuthorDto();
    }

    public async Task UpdateAsync(Guid id, UpdateAuthorDto input)
    {
        // Conditional: editors can edit anyone, otherwise must own the record.
        if (!await AuthorizationService.IsGrantedAsync(BookStorePermissions.Authors.Edit))
        {
            await AuthorizationService.CheckAsync(BookStorePermissions.Authors.EditOwn);
            // additional ownership validation...
        }
    }
}
```

**Key lines:** [Authorize("...")] resolves the policy name as an ABP permission. AuthorizationService is pre-injected on ApplicationService. CheckAsync throws AbpAuthorizationException (mapped to 403); IsGrantedAsync returns a bool for branching logic.

### Custom PermissionValueProvider driven by a claim

_Granting all permissions when the user has a User_Type=SystemAdmin claim, regardless of role assignment._

```csharp
using System.Threading.Tasks;
using Volo.Abp.Authorization.Permissions;

public class SystemAdminPermissionValueProvider : PermissionValueProvider
{
    public override string Name => "SystemAdmin";

    public SystemAdminPermissionValueProvider(IPermissionStore permissionStore)
        : base(permissionStore) { }

    public override Task<PermissionGrantResult> CheckAsync(PermissionValueCheckContext context)
    {
        if (context.Principal?.FindFirst("User_Type")?.Value == "SystemAdmin")
        {
            return Task.FromResult(PermissionGrantResult.Granted);
        }
        return Task.FromResult(PermissionGrantResult.Undefined);
    }
}

// In your AbpModule.ConfigureServices:
Configure<AbpPermissionOptions>(options =>
{
    options.ValueProviders.Add<SystemAdminPermissionValueProvider>();
});
```

**Key lines:** Return Undefined (not Prohibited) when the rule does not apply, so other providers can still grant. Prohibited is sticky and overrides any later Granted from another provider.

### Resource-based authorization on a Book entity

_Allow only the owner (or a manager with the management permission) to view a specific Book instance._

```csharp
// 1. Permission definition (Application.Contracts):
context.AddResourcePermission(
    name: BookStorePermissions.Books.Resources.View,
    resourceName: "Acme.BookStore.Books.Book",
    managementPermissionName: BookStorePermissions.Books.Manage,
    displayName: LocalizableString.Create<BookStoreResource>("Permission:Books.View"));

// 2. Use in an application service:
public class BookAppService : ApplicationService
{
    private readonly IRepository<Book, Guid> _books;
    public BookAppService(IRepository<Book, Guid> books) => _books = books;

    public async Task<BookDto> GetAsync(Guid id)
    {
        var book = await _books.GetAsync(id);
        await AuthorizationService.CheckAsync(book, BookStorePermissions.Books.Resources.View);
        return ObjectMapper.Map<Book, BookDto>(book);
    }
}

// 3. Batch check via IResourcePermissionChecker:
public class BookSharingService : ITransientDependency
{
    private readonly IResourcePermissionChecker _checker;
    public BookSharingService(IResourcePermissionChecker checker) => _checker = checker;

    public async Task<IDictionary<string, PermissionGrantResult>> GetMyAccessAsync(Book book)
        => await _checker.IsGrantedAsync(
            new[] { "View", "Edit", "Delete" },
            "Acme.BookStore.Books.Book",
            book.GetObjectKey()!);
}
```

**Key lines:** AddResourcePermission ties a permission to a resourceName/managementPermissionName pair. Passing the entity to AuthorizationService lets the framework call GetObjectKey() automatically. IResourcePermissionChecker is for batch or instance-key-only paths.

### Enable dynamic claims and impersonate a user in a background job

_Refresh role/permission claims mid-session and run a job as another user using ICurrentPrincipalAccessor.Change()._

```csharp
// Module wiring:
public override void ConfigureServices(ServiceConfigurationContext context)
{
    Configure<AbpClaimsPrincipalFactoryOptions>(options =>
    {
        options.IsDynamicClaimsEnabled = true;
    });
}

public override void OnApplicationInitialization(ApplicationInitializationContext context)
{
    var app = context.GetApplicationBuilder();
    app.UseAuthentication();
    app.UseDynamicClaims();   // before UseAuthorization
    app.UseAuthorization();
}

// Background job impersonating a user:
public class GenerateMonthlyReportJob : IAsyncBackgroundJob<GenerateMonthlyReportArgs>, ITransientDependency
{
    private readonly ICurrentPrincipalAccessor _principalAccessor;
    private readonly ICurrentUser _currentUser;
    private readonly IReportWriter _writer;

    public GenerateMonthlyReportJob(
        ICurrentPrincipalAccessor principalAccessor,
        ICurrentUser currentUser,
        IReportWriter writer)
    {
        _principalAccessor = principalAccessor;
        _currentUser = currentUser;
        _writer = writer;
    }

    public async Task ExecuteAsync(GenerateMonthlyReportArgs args)
    {
        var principal = new ClaimsPrincipal(new ClaimsIdentity(new[]
        {
            new Claim(AbpClaimTypes.UserId, args.RunAsUserId.ToString()),
            new Claim(AbpClaimTypes.UserName, args.RunAsUserName),
            new Claim(AbpClaimTypes.TenantId, args.TenantId?.ToString() ?? "")
        }, "BackgroundJob"));

        using (_principalAccessor.Change(principal))
        {
            // CurrentUser.Id == args.RunAsUserId here; permission checks honor the impersonated principal.
            await _writer.WriteAsync(_currentUser.Id!.Value);
        }
    }
}
```

**Key lines:** UseDynamicClaims() must precede UseAuthorization() so refreshed claims are visible to policy evaluation. ICurrentPrincipalAccessor.Change() returns IDisposable; the original principal is restored when the using block exits, including across awaits.

## Common mistakes

- Mistake: Hard-coding permission name strings at every call site. Why it's wrong: typos compile fine and silently fail to grant access; renames cascade poorly. Correct pattern: declare a static class (e.g. BookStorePermissions) with `public const string` constants and use them in both the Define() method and [Authorize(...)] / IsGrantedAsync() calls.
- Mistake: Returning PermissionGrantResult.Prohibited from a custom PermissionValueProvider as the 'default' branch. Why it's wrong: Prohibited is sticky and overrides Granted from any other provider, so adding the provider effectively revokes permissions you intended to leave alone. Correct pattern: return Undefined whenever the provider has nothing to say; only return Prohibited when you mean to actively deny.
- Mistake: Adding UseDynamicClaims() after UseAuthorization() (or omitting it entirely while setting IsDynamicClaimsEnabled=true). Why it's wrong: the principal isn't refreshed before policy evaluation, so role/permission revocations don't take effect until the next login — silently defeating the point of enabling dynamic claims. Correct pattern: `app.UseAuthentication(); app.UseDynamicClaims(); app.UseAuthorization();` in OnApplicationInitialization.
- Mistake: Using `new ClaimsPrincipal(...)` directly to 'change the user' instead of ICurrentPrincipalAccessor.Change(). Why it's wrong: ICurrentUser, IDataFilter (multi-tenancy), audit logging, and unit-of-work tenant scoping all read from ICurrentPrincipalAccessor; constructing a principal locally won't route through them. Correct pattern: build the ClaimsPrincipal once and wrap the work in `using (_currentPrincipalAccessor.Change(newPrincipal)) { ... }`.
- Mistake: Putting PermissionDefinitionProvider in the Domain or EntityFrameworkCore project. Why it's wrong: contracts modules consume the permission names, so domain-side placement creates wrong dependencies and breaks separation. Correct pattern: place providers in *.Application.Contracts and reference the constants from both Application and HttpApi layers.
- Mistake: Calling AuthorizationService.CheckAsync(entity, permission) without first ensuring the entity implements IKeyedObject (custom POCOs/DTOs). Why it's wrong: GetObjectKey() returns null and the resource-based check has nothing to bind to. Correct pattern: check resource permissions on the actual entity (which inherits IKeyedObject via Entity<TKey>), not on a DTO; or use IResourcePermissionChecker with an explicit (resourceName, key) pair.
- Mistake: Reading `_currentUser.Id.Value` without first checking IsAuthenticated. Why it's wrong: anonymous requests yield Id=null; .Value throws InvalidOperationException. Correct pattern: guard with `if (!_currentUser.IsAuthenticated) ...` or use `_currentUser.GetId()` (throws AbpException with a clear message) when authentication is required.
- Mistake: Forgetting to add AddAlwaysAllowAuthorization() in the test module and then debugging mysterious 403s in integration tests. Why it's wrong: the standard test base already does this for you, but only if you depend on it. Correct pattern: ensure your *.TestBase module derives from the ABP test module (or call `context.Services.AddAlwaysAllowAuthorization();` explicitly in ConfigureServices).
- Mistake: Setting permissions via SetForUserAsync inside a unit of work that doesn't get saved (e.g., a read-only AppService method). Why it's wrong: IPermissionManager calls participate in the ambient UoW; without commit they vanish and clients see stale grants. Correct pattern: perform grant changes inside a [UnitOfWork] method or wrap with `using var uow = _unitOfWorkManager.Begin(); ... await uow.CompleteAsync();`.
## Version pins (ABP 10.3)

- ABP 10.3 keeps the ASP.NET Core integration surface (IAuthorizationService, [Authorize], IAuthorizationHandler) identical to .NET 9; no migration if you upgrade from 9.x.
- Dynamic claims feature is enabled by default in v8.0+ startup templates — confirmed in 10.3 docs. If you started on a pre-8.0 template you may need to opt in with IsDynamicClaimsEnabled=true and add the UseDynamicClaims() middleware.
- AbpClaimsPrincipalFactoryOptions.RemoteRefreshUrl default is `/api/account/dynamic-claims/refresh` in 10.3 [uncertain — exact path is documented as default but not version-pinned].
- WebRemoteDynamicClaimsPrincipalContributor is disabled by default (IsEnabled=false); microservice gateways must opt in via WebRemoteDynamicClaimsPrincipalContributorOptions.
- AddResourcePermission and IResourcePermissionChecker shape are present in 10.3; older minor versions may differ in parameter names [uncertain — older-version drift not enumerated in current docs].
- AlwaysAllowAuthorizationService and AddAlwaysAllowAuthorization() helper remain in Volo.Abp.Authorization in 10.3 and are still pre-wired in the standard *.TestBase modules.
- ICurrentUser surface (Id/UserName/Email/EmailVerified/PhoneNumber/PhoneNumberVerified/TenantId/Roles/IsAuthenticated + FindClaim/FindClaims/GetAllClaims/IsInRole) is stable in 10.3.
- PermissionGrantResult.Prohibited semantics (sticky deny) are unchanged; this has been stable across major versions but remains a frequent footgun and is worth re-checking on each major upgrade [uncertain — no announced change in 10.x changelog].

## Cross-references

**Phase 1 references:**
- [references/framework/modules-and-startup.md](../framework/modules-and-startup.md) — Triggered by 'where do I register PermissionDefinitionProvider / AbpPermissionOptions / dynamic claims middleware?'. Module lifecycle (PreConfigureServices/ConfigureServices/OnApplicationInitialization) is the host for every wiring step on this page.
- [references/framework/multi-tenancy.md](../framework/multi-tenancy.md) — Triggered by 'host vs tenant permissions' and 'TenantId on ICurrentUser'. Permissions take MultiTenancySides; ICurrentUser surfaces TenantId; ICurrentPrincipalAccessor is the official way to switch tenant context alongside user context.
- [references/framework/application-services.md](../framework/application-services.md) — Triggered by 'how do I [Authorize] my AppService / use AuthorizationService'. ApplicationService is the canonical host for [Authorize] attributes and pre-injects AuthorizationService and CurrentUser.
- [references/framework/audit-logging.md](../framework/audit-logging.md) — Triggered by 'who did what?'. The audit log records UserId/TenantId taken from ICurrentUser; impersonation via ICurrentPrincipalAccessor.Change() therefore changes the audited actor.
- [references/framework/background-jobs.md](../framework/background-jobs.md) — Triggered by 'run a job as a user'. Background jobs typically have no HttpContext.User, so ICurrentPrincipalAccessor.Change() is the standard pattern to make permission-aware code work inside them.
- [references/framework/features-system.md](../framework/features-system.md) — Triggered by 'permission requires a feature'. PermissionDefinition.RequireFeatures()/RequireGlobalFeatures() couple permissions to the features system.

**External docs:**
- [ASP.NET Core Authorization](https://learn.microsoft.com/aspnet/core/security/authorization/introduction) — ABP layers on top of Microsoft.AspNetCore.Authorization — IAuthorizationService, [Authorize], policies, and IAuthorizationHandler all come from there.
- [ASP.NET Core Resource-Based Authorization](https://learn.microsoft.com/aspnet/core/security/authorization/resourcebased) — ABP's resource-based authorization uses the ASP.NET Core IAuthorizationService overloads that accept a resource argument; the underlying handler/requirement contract is from Microsoft.
- [System.Security.Claims (ClaimsPrincipal / ClaimsIdentity)](https://learn.microsoft.com/dotnet/api/system.security.claims) — ICurrentPrincipalAccessor wraps ClaimsPrincipal; AbpClaimTypes constants align with System.Security.Claims.ClaimTypes where appropriate.
- [OpenIddict](https://documentation.openiddict.com/) — ABP's OpenIddict module is the typical issuer of the access tokens that produce the claims read by ICurrentUser; dynamic claims interact with the OpenIddict-issued principal.
- [Microsoft.AspNetCore.Identity](https://learn.microsoft.com/aspnet/core/security/authentication/identity) — ABP Identity wraps ASP.NET Core Identity; user/role records that back the User/Role permission value providers come from Identity stores.

## Sources (ABP 10.3)

- [https://abp.io/docs/10.3/framework/fundamentals/authorization — Core authorization concepts: defining permissions via PermissionDefinitionProvider, checking permissions via IAuthorizationService / [Authorize], custom PermissionValueProvider, modifying existing permissions, feature dependencies, AlwaysAllowAuthorizationService for tests.](https://abp.io/docs/10.3/framework/fundamentals/authorization — Core authorization concepts: defining permissions via PermissionDefinitionProvider, checking permissions via IAuthorizationService / [Authorize], custom PermissionValueProvider, modifying existing permissions, feature dependencies, AlwaysAllowAuthorizationService for tests.)
- [https://abp.io/docs/10.3/framework/fundamentals/dynamic-claims — Dynamic claims system: IAbpClaimsPrincipalFactory, IAbpDynamicClaimsPrincipalContributor (Identity/Remote/WebRemote variants), AbpClaimsPrincipalFactoryOptions, UseDynamicClaims() middleware, how role/permission revocations apply mid-session.](https://abp.io/docs/10.3/framework/fundamentals/dynamic-claims — Dynamic claims system: IAbpClaimsPrincipalFactory, IAbpDynamicClaimsPrincipalContributor (Identity/Remote/WebRemote variants), AbpClaimsPrincipalFactoryOptions, UseDynamicClaims() middleware, how role/permission revocations apply mid-session.)
- [https://abp.io/docs/10.3/framework/fundamentals/authorization/resource-based-authorization — Instance-level authorization: AddResourcePermission(), IAuthorizationService.IsGrantedAsync(resource, permission), IResourcePermissionChecker, IKeyedObject contract, management permissions for resource permission UI.](https://abp.io/docs/10.3/framework/fundamentals/authorization/resource-based-authorization — Instance-level authorization: AddResourcePermission(), IAuthorizationService.IsGrantedAsync(resource, permission), IResourcePermissionChecker, IKeyedObject contract, management permissions for resource permission UI.)
- [https://abp.io/docs/10.3/framework/infrastructure/current-user — ICurrentUser (Volo.Abp.Users): Id/UserName/Email/TenantId/Roles/IsAuthenticated, FindClaim/FindClaims/GetAllClaims, ICurrentPrincipalAccessor.Change() impersonation pattern, AbpClaimTypes constants.](https://abp.io/docs/10.3/framework/infrastructure/current-user — ICurrentUser (Volo.Abp.Users): Id/UserName/Email/TenantId/Roles/IsAuthenticated, FindClaim/FindClaims/GetAllClaims, ICurrentPrincipalAccessor.Change() impersonation pattern, AbpClaimTypes constants.)

Last verified: 2026-05-10
