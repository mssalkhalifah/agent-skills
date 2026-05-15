# Modules — Operations (Audit Logging, Permissions, Settings, Features, Rate Limiting)

> Reference for ABP 10.3 cross-cutting Operations modules — Audit Logging (open-source + Pro), Permission Management, Setting Management, Feature Management, and Operation Rate Limiting (Pro) — which persist and manage runtime values (audits, granted permissions, settings, feature toggles, rate-limit counters) on top of the framework-level definition systems.

## When to load this reference

- How do I save audit logs to a database / customize what gets audited?
- How do I exclude a property or action from audit logging?
- How do I export audit logs to Excel or schedule audit-log cleanup?
- How do I grant/revoke permissions for a user or role from code?
- How do I implement permissions on a per-document or per-resource basis (resource permissions)?
- How do I read or change a setting for the current user / tenant / globally?
- How do I add a custom setting management provider?
- How do I check if a feature is enabled for the current tenant or set a feature value for an edition?
- How do I rate-limit a method (e.g., SMS send, login attempts) at the application-service level rather than HTTP middleware?
- What HTTP status / error code does ABP return when a rate limit is exceeded?
- What's the difference between IPermissionManager vs IPermissionChecker, ISettingManager vs ISettingProvider, IFeatureManager vs IFeatureChecker?

**Audience:** ABP backend developers wiring up authorization, configuration, multi-tenant feature flags, audit trails, and abuse-protection rate limits in any ABP 10.3 solution (Single-Layer, Layered, or Microservice).

## Key concepts

- AuditLog (Volo.Abp.AuditLogging.AuditLog) — aggregate root persisted by the Audit Logging module; contains AuditLogAction and EntityChange (with nested EntityPropertyChange) collections. Reach for it when querying historical audit data via IAuditLogRepository.
- IAuditLogRepository (Volo.Abp.AuditLogging.IAuditLogRepository) — domain repository for audit log persistence (EF Core: AbpAuditLogs/AbpAuditLogActions/AbpEntityChanges/AbpEntityPropertyChanges; Mongo: AbpAuditLogs collection). Use to read or query stored audit records.
- IAuditingStore (Volo.Abp.Auditing.IAuditingStore) — framework abstraction for persisting AuditLogInfo. Default: SimpleLogAuditingStore (logs only); the Audit Logging module replaces it with a DB-backed implementation. Implement to ship audit logs elsewhere (e.g., Elasticsearch).
- IAuditingManager (Volo.Abp.Auditing.IAuditingManager) — framework service exposing the current audit scope and BeginScope() for manual audit sessions in non-web flows (background jobs, console hosts).
- AbpAuditingOptions (Volo.Abp.Auditing.AbpAuditingOptions) — framework options: IsEnabled (default true), IsEnabledForAnonymousUsers (default true), AlwaysLogOnException (default true), ApplicationName (multi-app discriminator), EntityHistorySelectors (which entities track property changes), Contributors, IgnoredTypes.
- [Audited] / [DisableAuditing] attributes (Volo.Abp.Auditing) — class/method/property toggles for inclusion/exclusion in audit scope. [DisableAuditing] also accepts UpdateModificationProps and PublishEntityEvent flags.
- ExpiredAuditLogDeleterOptions / AuditLogExcelFileOptions (Pro) — schedule (CronExpression) for periodic cleanup and configuration for Excel-export retention/download base URL. Requires Hangfire or Quartz background workers and a BLOB Storage Provider.
- PermissionDefinitionProvider (Volo.Abp.Authorization.Permissions.PermissionDefinitionProvider) — base class auto-discovered by ABP; override Define(IPermissionDefinitionContext) to register groups and permissions (this is the *definition* side that the Permission Management module reads).
- PermissionGroupDefinition / PermissionDefinition (Volo.Abp.Authorization.Permissions) — built via context.AddGroup(...) and group.AddPermission(...) / parent.AddChild(...). Carry MultiTenancySides (Host/Tenant/Both), localized DisplayName, IsEnabled, StateCheckers, and Providers list.
- IPermissionManager (Volo.Abp.PermissionManagement.IPermissionManager) — read/modify granted permissions for users/roles. Methods include SetForUserAsync, SetForRoleAsync, GetAllForUserAsync, GetAllForRoleAsync. Used by the standard Permission Management dialog.
- IPermissionStore (Volo.Abp.Authorization.Permissions.IPermissionStore) — abstraction for checking whether a permission is granted to a provider (U=User, R=Role). The Permission Management module supplies the DB-backed implementation; framework default is AlwaysAllowPermissionStore.
- IResourcePermissionManager / IResourcePermissionStore (Volo.Abp.PermissionManagement) — manage permissions on specific resource instances (e.g., document id) with provider U/R; read via UserResourcePermissionValueProvider and RoleResourcePermissionValueProvider.
- IPermissionManagementProvider / PermissionManagementProvider (Volo.Abp.PermissionManagement) — pluggable storage strategy per provider name. Built-ins: UserPermissionManagementProvider, RolePermissionManagementProvider. Override IsAvailableAsync for host-only or tenant-only providers.
- PermissionManagementOptions (Volo.Abp.PermissionManagement.PermissionManagementOptions) — register custom providers via ManagementProviders.Add<T>() and configure ProviderPolicies for permission UI.
- SettingDefinitionProvider (Volo.Abp.Settings.SettingDefinitionProvider) — auto-discovered base class; override Define(ISettingDefinitionContext) to add SettingDefinition entries. Definition-only; values are managed by the Setting Management module.
- SettingDefinition (Volo.Abp.Settings.SettingDefinition) — Name, DefaultValue, DisplayName, Description, IsVisibleToClients (default false), IsInherited (default true), IsEncrypted, Providers (whitelist of provider names).
- ISettingProvider (Volo.Abp.Settings.ISettingProvider) — read-only, cached. GetAsync, GetOrNullAsync, GetAsync<T>, IsTrueAsync. Use this for normal reads in services.
- ISettingManager (Volo.Abp.SettingManagement.ISettingManager) — read/write across scopes: GetOrNullForUserAsync, SetForUserAsync, SetForTenantAsync, SetGlobalAsync, GetOrNullForCurrentTenantAsync. Always mutate through ISettingManager so cache invalidation fires.
- ISettingManagementProvider / SettingManagementProvider (Volo.Abp.SettingManagement) — pluggable management storage; built-ins (executed in reverse-registration order): UserSettingManagementProvider, TenantSettingManagementProvider, GlobalSettingManagementProvider, ConfigurationSettingManagementProvider, DefaultValueSettingManagementProvider.
- ISettingEncryptionService (Volo.Abp.Settings.ISettingEncryptionService) — used when SettingDefinition.IsEncrypted == true to encrypt/decrypt values transparently.
- FeatureDefinitionProvider (Volo.Abp.Features.FeatureDefinitionProvider) — auto-discovered; Define(IFeatureDefinitionContext) registers FeatureGroupDefinition and FeatureDefinition (with ValueType: ToggleStringValueType, FreeTextStringValueType, or SelectionStringValueType).
- IFeatureChecker (Volo.Abp.Features.IFeatureChecker) — read-only API: IsEnabledAsync, GetOrNullAsync, GetAsync<T>. Use to gate logic; cached.
- IFeatureManager (Volo.Abp.FeatureManagement.IFeatureManager) — read/modify per-tenant or per-edition feature values: SetForTenantAsync, SetForEditionAsync, GetOrNullForTenantAsync, GetOrNullForEditionAsync. Triggers cache invalidation.
- [RequiresFeature] attribute (Volo.Abp.Features) — declarative gate on application-service methods/classes; throws AbpAuthorizationException with a proper response when a required feature is disabled.
- Feature value providers (Volo.Abp.Features) — execution order Tenant -> Edition -> Configuration -> DefaultValue. The Feature Management module supplies Tenant/Edition providers; framework supplies Configuration/Default.
- IOperationRateLimitingChecker (Volo.Abp.OperationRateLimiting) — Pro module entry point: CheckAsync (increments and throws on excess), IsAllowedAsync (peek), GetStatusAsync (status without increment), ResetAsync (admin reset).
- [OperationRateLimiting("PolicyName")] / [RateLimitingParameter] attributes — class/method-level enforcement and partition-key parameter marker. Auto-applied via interceptors on application services and an MVC action filter (AbpOperationRateLimitingActionFilter).
- AbpOperationRateLimitingOptions — IsEnabled, LockTimeout, AddPolicy/ConfigurePolicy/RemovePolicy, AddPartitionKeyResolver/ReplacePartitionKeyResolver. Policies define one or more rules joined by AND.
- Rate-limit rules and partitions — WithFixedWindow(TimeSpan, maxCount), WithName(...), WithMultiTenancy(); partitions: PartitionByParameter, PartitionByCurrentUser, PartitionByCurrentTenant, PartitionByClientIp, PartitionByEmail, PartitionByPhoneNumber, PartitionBy("name"). maxCount: 0 acts as a permanent ban.
- AbpOperationRateLimitingException (extends BusinessException) — returns HTTP 429 via ABP exception pipeline; carries PolicyName, MaxCount, CurrentCount, RemainingCount, RetryAfterSeconds/Minutes, WindowDescription, RuleDetails. Error codes Volo.Abp.OperationRateLimiting:010001 (limit) and 010002 (ban).
- IOperationRateLimitingStore — abstraction over counter persistence; default uses IDistributedCache. Replace for Redis-Lua, custom DB, or sliding-window stores.

## Configuration pattern

Each Operations module is wired in an AbpModule via ConfigureServices using its options class. The Audit Logging module is depended on by the AbpAuditLoggingDomainModule (EF Core or Mongo variant) and tunes both AbpAuditingOptions (framework) and module-specific options (e.g., ExpiredAuditLogDeleterOptions in Pro). Permission/Setting/Feature management modules each expose an *Options class to register custom providers and use the connection-string convention (e.g., AbpPermissionManagement / AbpSettingManagement / AbpFeatureManagement, falling back to Default). Operation Rate Limiting registers policies against AbpOperationRateLimitingOptions; the AspNetCore companion package auto-registers AbpOperationRateLimitingActionFilter. Definitions for permissions, settings, and features live in framework-level *DefinitionProvider classes (no registration needed — auto-discovered).

Example composite ConfigureServices:

public override void ConfigureServices(ServiceConfigurationContext context)
{
    var hostEnvironment = context.Services.GetHostingEnvironment();

    // Audit Logging (framework-level switches)
    Configure<AbpAuditingOptions>(options =>
    {
        options.IsEnabled = true;
        options.IsEnabledForAnonymousUsers = false;
        options.ApplicationName = "BookStore";
        options.EntityHistorySelectors.AddAllEntities();
        options.IgnoredTypes.Add(typeof(SensitiveDto));
    });

    // Audit Logging Pro: cleanup + Excel export
    Configure<ExpiredAuditLogDeleterOptions>(options =>
    {
        options.IsEnabled = true;
        options.ExpiredAuditLogDeletePeriod = 30; // days
        options.CronExpression = "0 2 * * *";    // 02:00 daily
    });
    Configure<AuditLogExcelFileOptions>(options =>
    {
        options.FileRetentionHours = 24;
        options.DownloadBaseUrl = "https://my-app.com";
    });

    // Permission Management: register a custom provider, e.g. for an API-key principal
    Configure<PermissionManagementOptions>(options =>
    {
        options.ManagementProviders.Add<ApiKeyPermissionManagementProvider>();
    });

    // Setting Management: register a custom management provider
    Configure<SettingManagementOptions>(options =>
    {
        options.Providers.Add<OrganizationSettingManagementProvider>();
    });

    // Feature Management: register a custom provider
    Configure<FeatureManagementOptions>(options =>
    {
        options.Providers.Add<OrganizationFeatureManagementProvider>();
    });

    // Operation Rate Limiting (Pro)
    Configure<AbpOperationRateLimitingOptions>(options =>
    {
        options.IsEnabled = !hostEnvironment.IsDevelopment();
        options.LockTimeout = TimeSpan.FromSeconds(5);

        options.AddPolicy("SmsPerPhone", policy =>
        {
            policy.WithFixedWindow(TimeSpan.FromMinutes(1), maxCount: 1)
                  .PartitionByPhoneNumber();
        });

        options.AddPolicy("LoginAttempts", policy =>
        {
            policy.AddRule(rule => rule
                    .WithName("PerMinute")
                    .WithFixedWindow(TimeSpan.FromMinutes(1), maxCount: 5)
                    .PartitionByEmail()
                    .WithMultiTenancy());
            policy.AddRule(rule => rule
                    .WithName("PerHour")
                    .WithFixedWindow(TimeSpan.FromHours(1), maxCount: 50)
                    .PartitionByEmail());
        });
    });
}

Notes on options:
- AbpAuditingOptions.EntityHistorySelectors controls which entities track property-level changes; without selectors only metadata is recorded.
- SettingManagementOptions / FeatureManagementOptions / PermissionManagementOptions providers run in *reverse* of registration order: the most-specific (User > Tenant > Global > Configuration > Default) overrides earlier providers in the chain.
- AbpOperationRateLimitingOptions.IsEnabled = false is the recommended pattern for local development to avoid blocking yourself during testing.
- All five modules use ABP's connection-string fallback (Abp* dedicated connection -> Default) when their EF Core variant is registered.

## Code examples

### Disable auditing on a sensitive method

_Stop ABP from recording calls to a method that handles secret material so the audit log doesn't leak the inputs._

```csharp
using Volo.Abp.Auditing;
using Volo.Abp.Application.Services;

namespace BookStore.Vault;

public class VaultAppService : ApplicationService, IVaultAppService
{
    [DisableAuditing]
    public virtual Task<string> RotateSecretAsync(string newSecret)
    {
        // Method call, parameters and return value will not be saved to AuditLog
        return Task.FromResult("rotated");
    }
}
```

**Key lines:** [DisableAuditing] on the method excludes it from the current audit scope; combine with [DisableAuditing(UpdateModificationProps = false)] on entities you don't want CreatorId/LastModifierId touched on.

### Define a permission tree with multi-tenancy scope

_Add Author management permissions to BookStore, restricted to tenant side, with Create/Edit/Delete children of a parent permission._

```csharp
using Volo.Abp.Authorization.Permissions;
using Volo.Abp.MultiTenancy;

namespace BookStore.Permissions;

public class BookStorePermissionDefinitionProvider : PermissionDefinitionProvider
{
    public override void Define(IPermissionDefinitionContext context)
    {
        var group = context.AddGroup("BookStore");

        var authors = group.AddPermission(
            "BookStore.Authors",
            multiTenancySide: MultiTenancySides.Tenant);

        authors.AddChild("BookStore.Authors.Create");
        authors.AddChild("BookStore.Authors.Edit");
        authors.AddChild("BookStore.Authors.Delete");
    }
}
```

**Key lines:** PermissionDefinitionProvider is auto-registered by ABP. multiTenancySide ensures these permissions never appear on the host side. Child permissions are only grantable when the parent is granted.

### Grant a permission to a role programmatically

_During a data-seed contributor, give the 'admin' role the BookStore.Authors permission without going through the UI._

```csharp
using Volo.Abp.PermissionManagement;
using Volo.Abp.Authorization.Permissions;

public class BookStoreDataSeedContributor : IDataSeedContributor, ITransientDependency
{
    private readonly IPermissionManager _permissionManager;

    public BookStoreDataSeedContributor(IPermissionManager permissionManager)
    {
        _permissionManager = permissionManager;
    }

    public async Task SeedAsync(DataSeedContext context)
    {
        await _permissionManager.SetForRoleAsync(
            roleName: "admin",
            permissionName: "BookStore.Authors",
            isGranted: true);
    }
}
```

**Key lines:** SetForRoleAsync writes through the RolePermissionManagementProvider; the value is read back via the RolePermissionValueProvider, which is consulted by IPermissionChecker on each authorization check.

### Resource-level permission for a specific document

_Grant a single user view rights on a single Document instance._

```csharp
await _resourcePermissionManager.SetAsync(
    permissionName: "BookStore.Documents.View",
    resourceName: "BookStore.Document",
    resourceKey: documentId.ToString(),
    providerName: "U", // U = User, R = Role
    providerKey: userId.ToString(),
    isGranted: true);
```

**Key lines:** Resource permissions store one row per (resource instance, principal) — remember to delete them when the resource is deleted to avoid orphaned grants.

### Define a setting and read it via ISettingProvider

_Define an SMTP host setting with a default and read it from an application service._

```csharp
using Volo.Abp.Settings;

public class BookStoreSettingDefinitionProvider : SettingDefinitionProvider
{
    public override void Define(ISettingDefinitionContext context)
    {
        context.Add(
            new SettingDefinition(
                name: "BookStore.Smtp.Host",
                defaultValue: "localhost",
                displayName: L("DisplayName:Smtp.Host"),
                description: L("Description:Smtp.Host"),
                isVisibleToClients: false,
                isInherited: true,
                isEncrypted: false));
    }

    private static LocalizableString L(string n) => LocalizableString.Create<BookStoreResource>(n);
}

// Reading:
public class MailerService : ITransientDependency
{
    private readonly ISettingProvider _settingProvider;
    public MailerService(ISettingProvider settingProvider) => _settingProvider = settingProvider;

    public async Task<string> GetHostAsync() =>
        await _settingProvider.GetOrNullAsync("BookStore.Smtp.Host");
}
```

**Key lines:** ISettingProvider is the cached read path. To mutate, inject ISettingManager and call SetForUserAsync / SetForTenantAsync / SetGlobalAsync (the Setting Management module).

### Define and gate on a feature with [RequiresFeature]

_PDF reporting is only allowed for tenants whose edition includes BookStore.PdfReporting._

```csharp
using Volo.Abp.Features;

public class BookStoreFeatureDefinitionProvider : FeatureDefinitionProvider
{
    public override void Define(IFeatureDefinitionContext context)
    {
        var group = context.AddGroup("BookStore");
        group.AddFeature(
            name: "BookStore.PdfReporting",
            defaultValue: "false",
            displayName: LocalizableString.Create<BookStoreResource>("PdfReporting"));
    }
}

[RequiresFeature("BookStore.PdfReporting")]
public class PdfReportAppService : ApplicationService, IPdfReportAppService
{
    public Task<byte[]> GenerateAsync(Guid bookId) => /* ... */;
}
```

**Key lines:** Definition is auto-discovered. To set the feature on a tenant: await _featureManager.SetForTenantAsync(tenantId, "BookStore.PdfReporting", "true"). To set it for an edition (SaaS module): SetForEditionAsync.

### Rate-limit SMS sending per phone number

_Throw HTTP 429 if the same phone number is sent more than one SMS per minute._

```csharp
// Module ConfigureServices
Configure<AbpOperationRateLimitingOptions>(options =>
{
    options.AddPolicy("SmsPerPhone", policy =>
    {
        policy.WithFixedWindow(TimeSpan.FromMinutes(1), maxCount: 1)
              .PartitionByPhoneNumber();
    });
});

// Application service
public class SmsAppService : ApplicationService, ISmsAppService
{
    [OperationRateLimiting("SmsPerPhone")]
    public virtual Task SendAsync([RateLimitingParameter] string phoneNumber, string text)
    {
        // ... actually send
        return Task.CompletedTask;
    }
}
```

**Key lines:** PartitionByPhoneNumber normalizes the phone number (strips formatting); [RateLimitingParameter] tells the interceptor which method argument supplies the partition key. On the second call within 60s, IOperationRateLimitingChecker.CheckAsync throws AbpOperationRateLimitingException -> 429 with RetryAfter headers.

### Programmatic rate-limit check inside a domain service

_Manually call CheckAsync from a domain service that is not an application service / controller._

```csharp
public class LoginAttemptManager : DomainService
{
    private readonly IOperationRateLimitingChecker _rateLimit;
    public LoginAttemptManager(IOperationRateLimitingChecker rateLimit) => _rateLimit = rateLimit;

    public async Task RegisterAttemptAsync(string email)
    {
        await _rateLimit.CheckAsync("LoginAttempts", new OperationRateLimitingContext
        {
            Parameter = email
        });
    }
}
```

**Key lines:** Use CheckAsync (not IsAllowedAsync) when you want the counter incremented and the exception thrown on violation; use GetStatusAsync to query without consuming a slot.

## Common mistakes

### Modifying setting/feature values directly through ISettingProvider or by writing rows into the management tables

**Why wrong:** ISettingProvider is read-only. Direct DB writes bypass cache invalidation, so other nodes keep returning stale values. The same applies to per-tenant feature values written outside IFeatureManager.

```csharp
Always inject ISettingManager / IFeatureManager and call SetForUserAsync / SetForTenantAsync / SetForEditionAsync / SetGlobalAsync. They invalidate the distributed cache for you.
```

### Forgetting that management providers execute in REVERSE order of registration

**Why wrong:** If you Add<MyProvider>() expecting it to run first, you'll find the built-in User/Tenant providers shadow it because they were registered earlier and therefore execute later (last writer wins).

```csharp
Treat the providers list as a stack. To make MyProvider override User/Tenant, register it AFTER them (the framework registers built-ins during PreConfigureServices), or override the existing provider name (e.g., subclass UserSettingManagementProvider with the same Name).
```

### Calling [DisableAuditing] on the entity but expecting modification props (CreationTime, LastModifierId) to still be filled

**Why wrong:** [DisableAuditing] also sets UpdateModificationProps = true by default — it'll skip touching IHasModificationTime fields and won't publish entity events.

```csharp
Use [DisableAuditing(UpdateModificationProps = false, PublishEntityEvent = false)] only for fields you really don't want recorded; otherwise rely on AbpAuditingOptions.IgnoredTypes for narrower control.
```

### Using PartitionByParameter() with raw user input like emails / phone numbers

**Why wrong:** Two visually-identical inputs ("User@Example.COM" vs "user@example.com") hash to different partition keys, so attackers bypass the limit by case-flipping.

```csharp
Use the dedicated PartitionByEmail() / PartitionByPhoneNumber() resolvers, which normalize before hashing. For other domain values, register a custom resolver via AddPartitionKeyResolver and normalize there.
```

### Leaving AbpOperationRateLimitingOptions.IsEnabled = true in development

**Why wrong:** You repeatedly hit your own limits during local testing and confuse legitimate failures with rate-limit responses.

```csharp
Wrap registration in if (!hostEnvironment.IsDevelopment()) options.IsEnabled = true; or set IsEnabled = false in the Development module.
```

### Forgetting that the Operation Rate Limiting counter store defaults to IDistributedCache

**Why wrong:** On a horizontally-scaled deployment with the default in-memory cache (no Redis), each node has its own counter — effective limit becomes N x maxCount.

```csharp
Register a real distributed cache (typically Volo.Abp.Caching.StackExchangeRedis) before relying on rate limits in production. For lower latency, implement IOperationRateLimitingStore on Redis with Lua-based atomic INCR/EXPIRE.
```

### Defining PermissionDefinition without MultiTenancySides and granting it on the host side

**Why wrong:** Default MultiTenancySides is Both. Permissions intended only for tenants leak to the host, and vice versa, producing confusing UIs.

```csharp
Always pass multiTenancySide explicitly when the permission is scope-specific: MultiTenancySides.Tenant or MultiTenancySides.Host.
```

### Deleting a resource (e.g., Document) without cleaning up its resource permissions

**Why wrong:** AbpResourcePermissionGrants rows linger forever, slowing queries and potentially re-granting access if a new resource is created with the reused id.

```csharp
On resource deletion, call IResourcePermissionManager.DeleteAsync(resourceName, resourceKey) (or wire it to a domain event handler) so permission rows are purged transactionally with the resource.
```

### Treating Audit Logging Pro features as available without a license

**Why wrong:** Excel export, the Audit Log UI, periodic cleanup, and dashboard widgets ship in Volo.AuditLogging.Ui (Pro). Open-source Volo.AuditLogging only persists logs.

```csharp
Confirm the solution is on an ABP Team or higher license before promising Excel-export, scheduled cleanup, or in-app log browsing; otherwise build a custom UI on IAuditLogRepository.
```

### Reading settings/features inside a constructor

**Why wrong:** Setting/feature reads are async (cache or DB) and require the current scope (user/tenant) — calling them synchronously in the ctor risks blocked threads or wrong-tenant values.

```csharp
Inject ISettingProvider / IFeatureChecker and resolve values inside the method, lazily; or inject IOptions snapshots only for static configuration values.
```

## Version pins (ABP 10.3)

- ABP 10.3 keeps AbpAuditingOptions defaults: IsEnabled=true, IsEnabledForAnonymousUsers=true, AlwaysLogOnException=true. Confirm in your version before relying on them; flip via Configure<AbpAuditingOptions>.
- Audit Logging Pro UI is Volo.AuditLogging.Ui in 10.3 (Volo.* namespace). The package id and Excel-cleanup options layout (ExpiredAuditLogDeleterOptions, AuditLogExcelFileOptions) may shift in 10.4 [uncertain — only a 10.4 preview is referenced from the docs index].
- Operation Rate Limiting is part of the Pro license set in 10.3 and uses IDistributedCache by default; the API surface (IOperationRateLimitingChecker, AbpOperationRateLimitingOptions, [OperationRateLimiting]) is stable in 10.3 but the module is relatively new — minor naming changes in 10.4 are possible [uncertain].
- PermissionManagement / SettingManagement / FeatureManagement provider chains in 10.3 run in reverse-registration order; this ordering convention has been stable across recent ABP majors but is undocumented as a guarantee.
- AbpAuditLogging / AbpPermissionManagement / AbpSettingManagement / AbpFeatureManagement connection-string names are conventional in 10.3 and fall back to Default. Renames are unlikely but not contractually fixed.
- Error codes Volo.Abp.OperationRateLimiting:010001 and :010002 are the codes reported in 10.3 docs; treat them as version-pinned strings in client error-handling logic.
- [DisableAuditing] options UpdateModificationProps and PublishEntityEvent default to true in 10.3; older ABP versions had different defaults — check before upgrading from <8.x.
- The Permission Management resource-permission API (IResourcePermissionManager, providerName "U"/"R") is current in 10.3; check release notes if upgrading, as resource permissions are a relatively recent addition [uncertain — exact version of introduction not confirmed in the consulted pages].

## Cross-references

**Phase 1 references:**
- [references/framework/authorization.md](../framework/authorization.md) — triggered when a question is about authoring permissions (PermissionDefinitionProvider, [Authorize], IPermissionChecker) rather than managing granted values.
- [references/framework/multi-tenancy.md](../framework/multi-tenancy.md) — triggered whenever permissions/settings/features are used across host/tenant boundaries (MultiTenancySides, TenantSettingValueProvider, EditionFeatureValueProvider).
- [references/modules/identity-and-auth.md](../modules/identity-and-auth.md) — triggered when permissions/settings interact with users, roles, organization units, OpenIddict.
- [references/modules/tenancy-and-saas.md](../modules/tenancy-and-saas.md) — triggered for editions and feature management against editions (IFeatureManager.SetForEditionAsync).
- [references/infrastructure/background-processing.md](../infrastructure/background-processing.md) — triggered for Pro audit-log cleanup (Hangfire/Quartz dependency) and rate-limit store TTL eviction.
- [references/infrastructure/blob-storing.md](../infrastructure/blob-storing.md) — triggered for the Audit Logging Pro Excel-export download URLs / file retention.
- [references/infrastructure/caching-and-locking.md](../infrastructure/caching-and-locking.md) — triggered because Setting/Feature reads and Operation Rate Limiting counters all sit on IDistributedCache (Redis recommended for prod).

**External docs:**
- [ASP.NET Core Authorization](https://learn.microsoft.com/aspnet/core/security/authorization/introduction) — ABP's permission system plugs into ASP.NET Core policy-based authorization; understanding [Authorize], requirements, and handlers helps reason about IPermissionChecker.
- [ASP.NET Core Rate Limiting middleware](https://learn.microsoft.com/aspnet/core/performance/rate-limit) — The framework HTTP rate limiter complements ABP's Operation Rate Limiting (which is application-layer). Knowing the difference avoids double-throttling.
- [Microsoft.Extensions.Caching.Distributed (IDistributedCache)](https://learn.microsoft.com/aspnet/core/performance/caching/distributed) — Default ISettingProvider/IFeatureChecker caches and the default IOperationRateLimitingStore all use IDistributedCache; configure Redis for multi-instance deployments.
- [EF Core Query Filters](https://learn.microsoft.com/ef/core/querying/filters) — AbpAuditLogs / permission / setting tables are tenant-aware via ABP's data-filter system layered on EF Core query filters.
- [Hangfire (recommended worker for Pro cleanup)](https://docs.hangfire.io/) — ExpiredAuditLogDeleterOptions.CronExpression runs as a background worker; Hangfire is the typical implementation in ABP solutions.
- [Quartz.NET (alternative worker)](https://www.quartz-scheduler.net/documentation/) — Same role as Hangfire — schedules the periodic audit-log delete and Excel-file cleanup jobs.
- [RFC 6585 — HTTP 429 Too Many Requests](https://www.rfc-editor.org/rfc/rfc6585#section-4) — The status code returned by AbpOperationRateLimitingException; the Retry-After header semantics drive the exception's RetryAfterSeconds/Minutes properties.

## Sources (ABP 10.3)

- [https://abp.io/docs/10.3/modules/audit-logging](https://abp.io/docs/10.3/modules/audit-logging)
- [https://abp.io/docs/10.3/modules/audit-logging-pro](https://abp.io/docs/10.3/modules/audit-logging-pro)
- [https://abp.io/docs/10.3/modules/permission-management](https://abp.io/docs/10.3/modules/permission-management)
- [https://abp.io/docs/10.3/modules/setting-management](https://abp.io/docs/10.3/modules/setting-management)
- [https://abp.io/docs/10.3/modules/feature-management](https://abp.io/docs/10.3/modules/feature-management)
- [https://abp.io/docs/10.3/modules/operation-rate-limiting](https://abp.io/docs/10.3/modules/operation-rate-limiting)
- [https://abp.io/docs/10.3/framework/infrastructure/audit-logging](https://abp.io/docs/10.3/framework/infrastructure/audit-logging)
- [https://abp.io/docs/10.3/framework/fundamentals/authorization](https://abp.io/docs/10.3/framework/fundamentals/authorization)
- [https://abp.io/docs/10.3/framework/infrastructure/settings](https://abp.io/docs/10.3/framework/infrastructure/settings)
- [https://abp.io/docs/10.3/framework/infrastructure/features](https://abp.io/docs/10.3/framework/infrastructure/features)

Last verified: 2026-05-10
