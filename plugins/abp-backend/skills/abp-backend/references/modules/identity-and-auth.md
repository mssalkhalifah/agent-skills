# Modules — Identity, Account, OpenIddict

> Reference for ABP 10.3's three identity-and-auth modules — Identity (users/roles/permissions/organization-units), Account (login/register/forgot-password/profile UI), and OpenIddict (OAuth2/OIDC server) — covering the OSS packages, the Pro extensions, and the legacy IdentityServer module that OpenIddict has replaced since ABP v6.0.

## When to load this reference

- User asks how to add or customize users, roles, organization units, claim types, or security logs in an ABP solution.
- User asks how the login / register / forgot-password / change-password / external-login pages work and how to override them.
- User asks how OAuth2/OpenID Connect, access tokens, refresh tokens, /connect/token, /connect/authorize, or client credentials flow are configured in ABP.
- User asks how to register or seed an OpenIddict client application or scope (e.g., adding a new SPA/mobile client, defining an API resource).
- User asks 'IdentityServer vs OpenIddict in ABP', how to migrate from IdentityServer to OpenIddict, or whether to use the IdentityServer module on a new 10.3 project.
- User asks about LDAP, social/external logins (Google/Microsoft/Twitter/Facebook/GitHub), two-factor authentication, password aging/history, account lockout, or session management — i.e., features that span Identity Pro + Account Pro.
- User asks how Identity module permissions plug into ABP's authorization system (PermissionDefinitionProvider, RolePermissionValueProvider, IPermissionManager).
- User encounters AddDevelopmentEncryptionAndSigningCertificate, missing certs in production, OpenIddict client_id/redirect_uri errors, or HTTPS-only token-endpoint failures from RN/MAUI emulators.
- User asks which packages (Volo.Abp.Identity.*, Volo.Abp.Account.*, Volo.Abp.OpenIddict.*) to depend on for a new microservice or external module.

**Audience:** ABP developers configuring authentication, authorization, user management, or an OAuth2/OIDC server in an ABP 10.3 solution — application-service/API authors who need to know which manager/repository to inject, module authors wiring DependsOn graphs, and architects choosing between the OSS and Pro variants or planning an IdentityServer-to-OpenIddict migration.

## Key concepts

- IdentityUser — Volo.Abp.Identity.IdentityUser — Aggregate root for system users; full-trust replacement for ASP.NET Core Identity's user entity, augmented with multi-tenancy, soft delete, claims, logins, tokens, and organization-unit links. Reach for it whenever you need to read or extend user state and prefer it over hand-rolling a User entity.
- IdentityRole — Volo.Abp.Identity.IdentityRole — Aggregate root for roles; carries IsDefault (auto-assigned to new users), IsStatic (system-protected), IsPublic (visible in tenant), plus claims. Reach for it for any role CRUD and for assigning permissions to roles.
- IdentityUserManager — Volo.Abp.Identity.IdentityUserManager — ABP-derived UserManager<IdentityUser>; the canonical domain service for create/update/delete users, password set/change, role/claim assignment, lockout. Reach for it from app services or domain services rather than constructing an IdentityUser directly.
- IdentityRoleManager — Volo.Abp.Identity.IdentityRoleManager — ABP-derived RoleManager<IdentityRole>; create/update roles and manage their claims. Reach for it whenever role state needs business validation.
- IIdentityUserRepository / IIdentityRoleRepository / IIdentityClaimTypeRepository / IIdentitySecurityLogRepository / IOrganizationUnitRepository — Volo.Abp.Identity — Data-access surfaces for the corresponding aggregates; reach for them when you need queries the manager doesn't expose (paged user lists, by-claim filters, OU traversal).
- OrganizationUnit / OrganizationUnitManager — Volo.Abp.Identity — Hierarchical user grouping with materialized-path Code; manager handles parent/child moves and uniqueness. Reach for it when modelling departments/teams that should cascade permissions to users.
- IdentityClaimType / IdentityClaimTypeManager — Volo.Abp.Identity — Tenant-scoped definitions of custom user claims (name, value type, validation, regex). Reach for them to add structured profile data without subclassing IdentityUser.
- IdentitySecurityLog / IdentitySecurityLogManager — Volo.Abp.Identity — Persistent audit trail of auth events; written via IdentitySecurityLogManager.SaveAsync(IdentitySecurityLogContext). Reach for it from custom auth code paths to keep the security-log UI complete.
- IIdentityUserAppService / IIdentityRoleAppService / IIdentityClaimTypeAppService / IIdentitySettingsAppService / IProfileAppService / IIdentityUserIntegrationService — Volo.Abp.Identity (Application.Contracts) — REST-exposed app services backing the Identity admin UI and profile page; IIdentityUserIntegrationService is the service-to-service lookup contract used across microservices.
- AbpIdentityModule — Volo.Abp.Identity.AbpIdentityModule (and AbpIdentityDomainModule, AbpIdentityApplicationModule, AbpIdentityHttpApiModule, AbpIdentityEntityFrameworkCoreModule, AbpIdentityMongoDbModule) — DependsOn target for consuming the Identity module; pick the layer-specific submodule that matches the project layer.
- AbpIdentityDbProperties — Volo.Abp.Identity.AbpIdentityDbProperties — Static class exposing DbTablePrefix='Abp', DbSchema=null, ConnectionStringName='AbpIdentity'. Reach for it when wiring a separate physical database for Identity (microservice split).
- IdentityOptions / AbpIdentityOptions — Microsoft.AspNetCore.Identity.IdentityOptions / Volo.Abp.Identity.AbpIdentityOptions — Standard ASP.NET Core Identity password/lockout/sign-in options, plus ABP additions; configured in AbpModule.ConfigureServices and overridable per-tenant via ISettingManager + IdentitySettingNames.
- Distributed events — Volo.Abp.Users.UserEto, Volo.Abp.Identity.IdentityRoleEto, IdentityClaimTypeEto, OrganizationUnitEto — Published as EntityCreatedEto/EntityUpdatedEto/EntityDeletedEto on user/role/claim-type/OU mutations. Subscribe with IDistributedEventHandler<EntityCreatedEto<UserEto>> in microservices that mirror identity state.
- AccountController + Account Razor Pages — Volo.Abp.Account.Web — Pages under /Account/Login, /Account/Register, /Account/ForgotPassword, /Account/Manage; provided by Volo.Abp.Account.Web (OSS) plus Volo.Abp.Account.Web.OpenIddict integration package.
- IAccountAppService / IAccountSettingsAppService / IProfileAppService — Volo.Abp.Account — Application services for register, password reset, account settings, and profile self-management; reach for these instead of the IIdentityUserAppService when implementing self-service flows.
- AbpAccountOptions — Volo.Abp.Account.AbpAccountOptions — Configures Windows-auth scheme, tenant admin username, impersonation permissions, external provider icon mapping. Reach for it when whitelabelling sign-in or wiring impersonation.
- AbpProfilePictureOptions — Volo.Abp.Account.AbpProfilePictureOptions — Profile picture image-compression knobs (Pro-only). [uncertain] — exact namespace/Pro-only gating.
- IPostConfigureAccountExternalProviderOptions<T> — Volo.Abp.Account — Hook for late-binding external-provider options per tenant; reach for it on multi-tenant SaaS where each tenant brings its own Google/Azure-AD app credentials.
- OpenIddictApplication — Volo.Abp.OpenIddict.Applications.OpenIddictApplication — Aggregate for OAuth client registrations (ClientId, ClientSecret, RedirectUris, PostLogoutRedirectUris, Permissions, DisplayName); seeded by OpenIddictDataSeedContributor.
- OpenIddictScope — Volo.Abp.OpenIddict.Scopes.OpenIddictScope — Aggregate for API/identity scopes (Name, DisplayName, Description, Resources). Reach for it whenever a new API resource needs to be advertised on /.well-known/openid-configuration.
- OpenIddictAuthorization / OpenIddictToken — Volo.Abp.OpenIddict.Authorizations / .Tokens — Persisted authorizations and tokens; participate in TokenCleanupOptions sweeps.
- IOpenIddictApplicationManager / IOpenIddictScopeManager — OpenIddict.Abstractions — Standard OpenIddict managers, available via DI; ABP adds AbpApplicationManager : OpenIddictApplicationManager that copies ClientUri / LogoUri to ABP-side properties.
- AbpOpenIddictAspNetCoreOptions — Volo.Abp.OpenIddict.AspNetCore — UpdateAbpClaimTypes (default true) keeps OpenIddict's claim names in sync with ABP, AddDevelopmentEncryptionAndSigningCertificate (default true) auto-registers user-scoped dev certs — must be set false in production.
- TokenCleanupOptions — Volo.Abp.OpenIddict — Background sweep for expired tokens/authorizations: IsCleanupEnabled (true), CleanupPeriod (3,600,000 ms), MinimumAuthorizationLifespan / MinimumTokenLifespan (14 days).
- OpenIddictDataSeedContributor — Volo.Abp.OpenIddict — Default seeder that creates sample applications/scopes from configuration; replace or extend it (subclass + DependsOn order) to seed your real clients during deployment.
- /connect/authorize, /connect/token, /connect/logout, /connect/userinfo — Volo.Abp.OpenIddict.Controllers — Standard OIDC endpoints exposed by AuthorizeController/TokenController/LogoutController/UserInfoController; clients consume these as the OIDC issuer.
- AbpAccountWebOpenIddictModule — Volo.Abp.Account.Web.OpenIddict — Glue module that wires the Account UI's login/consent flow into OpenIddict; DependsOn this in the AuthServer/HttpApi.Host project.
- AbpIdentityServerModule (legacy) — Volo.Abp.IdentityServer — Wraps IdentityServer4; explicitly replaced by OpenIddict in startup templates after ABP v6.0; cannot be used alongside the OpenIddict module in the same host.

## Configuration pattern

Identity, Account, and OpenIddict are independent ABP modules; you wire each by adding the appropriate submodule to your AbpModule's [DependsOn] and configuring its options in PreConfigureServices/ConfigureServices.

Identity (domain layer):
```csharp
[DependsOn(
    typeof(AbpIdentityDomainModule),
    typeof(AbpIdentityEntityFrameworkCoreModule)
)]
public class MyDomainModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        Configure<IdentityOptions>(options =>
        {
            options.Password.RequiredLength = 8;
            options.Password.RequireDigit = true;
            options.Lockout.MaxFailedAccessAttempts = 5;
        });

        Configure<AbpIdentityOptions>(options =>
        {
            options.SaveStaticRoles = true;
        });
    }
}
```
IdentityOptions values can also be overridden per-tenant at runtime via ISettingManager + IdentitySettingNames (e.g., IdentitySettingNames.Password.RequiredLength).

Account + OpenIddict (auth-server / web layer):
```csharp
[DependsOn(
    typeof(AbpAccountWebOpenIddictModule),
    typeof(AbpOpenIddictAspNetCoreModule)
)]
public class MyAuthServerModule : AbpModule
{
    public override void PreConfigureServices(ServiceConfigurationContext context)
    {
        PreConfigure<OpenIddictBuilder>(builder =>
        {
            builder.AddValidation(options =>
            {
                options.AddAudiences("MyApp");
                options.UseLocalServer();
                options.UseAspNetCore();
            });
        });

        PreConfigure<OpenIddictServerBuilder>(builder =>
        {
            builder.SetAccessTokenLifetime(TimeSpan.FromMinutes(30));
            builder.SetRefreshTokenLifetime(TimeSpan.FromDays(14));
        });
    }

    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        var hostingEnv = context.Services.GetHostingEnvironment();
        if (!hostingEnv.IsDevelopment())
        {
            Configure<AbpOpenIddictAspNetCoreOptions>(options =>
            {
                options.AddDevelopmentEncryptionAndSigningCertificate = false;
            });
            // load production X.509 certs via PreConfigure<OpenIddictServerBuilder>
        }

        Configure<TokenCleanupOptions>(options =>
        {
            options.IsCleanupEnabled = true;
        });
    }
}
```
Account-side knobs (AbpAccountOptions, AbpProfilePictureOptions, AbpIdentityAspNetCoreOptions.ConfigureAuthentication) are configured the same way inside ConfigureServices. The Account UI's external/social logins are added via standard ASP.NET Core builder.AddGoogle()/AddMicrosoftAccount()/AddFacebook() calls in PreConfigureServices, with credentials from appsettings.json or User Secrets.

## Code examples

### Inject IdentityUserManager to create a user from a domain service

_Programmatically creating a tenant admin during data seeding or a custom signup flow._

```csharp
public class TenantAdminCreator : IDomainService, ITransientDependency
{
    private readonly IdentityUserManager _userManager;
    private readonly IIdentityRoleRepository _roleRepository;
    private readonly IGuidGenerator _guidGenerator;

    public TenantAdminCreator(
        IdentityUserManager userManager,
        IIdentityRoleRepository roleRepository,
        IGuidGenerator guidGenerator)
    {
        _userManager = userManager;
        _roleRepository = roleRepository;
        _guidGenerator = guidGenerator;
    }

    public async Task<IdentityUser> CreateAsync(string userName, string email, string password)
    {
        var user = new IdentityUser(_guidGenerator.Create(), userName, email);
        (await _userManager.CreateAsync(user, password)).CheckErrors();

        var adminRole = await _roleRepository.FindByNormalizedNameAsync("ADMIN");
        if (adminRole != null)
        {
            (await _userManager.AddToRoleAsync(user, adminRole.Name)).CheckErrors();
        }
        return user;
    }
}
```

**Key lines:** IdentityUserManager.CreateAsync wraps Microsoft Identity's UserManager and returns IdentityResult; CheckErrors() throws on failure with ABP's UserFriendlyException. Always look up roles via the repository's FindByNormalizedNameAsync (uppercase) — ABP normalizes names automatically on save.

### Subscribe to UserEto to mirror users into a microservice

_A separate ProductCatalog microservice keeps a denormalized copy of users for activity feeds._

```csharp
public class UserCreatedHandler :
    IDistributedEventHandler<EntityCreatedEto<UserEto>>,
    ITransientDependency
{
    private readonly IRepository<LocalUser, Guid> _localUsers;

    public UserCreatedHandler(IRepository<LocalUser, Guid> localUsers)
        => _localUsers = localUsers;

    public async Task HandleEventAsync(EntityCreatedEto<UserEto> eventData)
    {
        var u = eventData.Entity;
        await _localUsers.InsertAsync(new LocalUser(u.Id, u.UserName, u.Email), autoSave: true);
    }
}
```

**Key lines:** ABP publishes EntityCreatedEto/EntityUpdatedEto/EntityDeletedEto wrapping UserEto whenever IdentityUser changes; subscribing keeps satellite services consistent without coupling them to the Identity DB. Make sure the consuming module DependsOn AbpEventBusModule and a transport (RabbitMQ/Kafka/etc.).

### Seed a new OpenIddict client from a custom data-seed contributor

_Adding a SPA client (Angular) and a server-to-server client (background worker) at first run._

```csharp
public class MyOpenIddictDataSeedContributor : IDataSeedContributor, ITransientDependency
{
    private readonly IOpenIddictApplicationManager _applicationManager;
    private readonly IOpenIddictScopeManager _scopeManager;

    public MyOpenIddictDataSeedContributor(
        IOpenIddictApplicationManager applicationManager,
        IOpenIddictScopeManager scopeManager)
    {
        _applicationManager = applicationManager;
        _scopeManager = scopeManager;
    }

    public async Task SeedAsync(DataSeedContext context)
    {
        if (await _scopeManager.FindByNameAsync("MyApp") == null)
        {
            await _scopeManager.CreateAsync(new OpenIddictScopeDescriptor
            {
                Name = "MyApp",
                DisplayName = "MyApp API",
                Resources = { "MyApp" }
            });
        }

        if (await _applicationManager.FindByClientIdAsync("MyApp_Angular") == null)
        {
            await _applicationManager.CreateAsync(new OpenIddictApplicationDescriptor
            {
                ClientId = "MyApp_Angular",
                ConsentType = OpenIddictConstants.ConsentTypes.Implicit,
                DisplayName = "MyApp Angular Client",
                ClientType = OpenIddictConstants.ClientTypes.Public,
                RedirectUris = { new Uri("http://localhost:4200") },
                PostLogoutRedirectUris = { new Uri("http://localhost:4200") },
                Permissions =
                {
                    OpenIddictConstants.Permissions.Endpoints.Authorization,
                    OpenIddictConstants.Permissions.Endpoints.Token,
                    OpenIddictConstants.Permissions.GrantTypes.AuthorizationCode,
                    OpenIddictConstants.Permissions.GrantTypes.RefreshToken,
                    OpenIddictConstants.Permissions.ResponseTypes.Code,
                    OpenIddictConstants.Permissions.Scopes.Email,
                    OpenIddictConstants.Permissions.Scopes.Profile,
                    OpenIddictConstants.Permissions.Scopes.Roles,
                    OpenIddictConstants.Permissions.Prefixes.Scope + "MyApp"
                }
            });
        }
    }
}
```

**Key lines:** IDataSeedContributor runs from the DbMigrator project; using IOpenIddictApplicationManager + OpenIddictApplicationDescriptor is the supported way to register clients in code (do not insert into OpenIddictApplications directly because ClientSecret must be hashed via the manager). For confidential server-to-server clients, set ClientType = Confidential and supply ClientSecret on the descriptor — the manager hashes it before persistence.

### Disable dev certs and force production OpenIddict signing/encryption certs

_Deploying the AuthServer to production where dev user-store certs aren't available._

```csharp
public override void ConfigureServices(ServiceConfigurationContext context)
{
    var env = context.Services.GetHostingEnvironment();
    if (env.IsProduction())
    {
        Configure<AbpOpenIddictAspNetCoreOptions>(options =>
        {
            options.AddDevelopmentEncryptionAndSigningCertificate = false;
        });

        PreConfigure<OpenIddictServerBuilder>(builder =>
        {
            var cert = new X509Certificate2("openiddict.pfx",
                context.Services.GetConfiguration()["AuthServer:CertificatePassPhrase"],
                X509KeyStorageFlags.MachineKeySet);
            builder.AddSigningCertificate(cert);
            builder.AddEncryptionCertificate(cert);
        });
    }
}
```

**Key lines:** Leaving AddDevelopmentEncryptionAndSigningCertificate=true in production silently fails on Linux/containers because the dev cert lives in the per-user store; the auth server then issues unverifiable tokens. Setting it false and registering an explicit X.509 cert via OpenIddictServerBuilder is the canonical fix.

### Add Google as an external login from the Account module

_Wiring a social login button on /Account/Login that round-trips through Google._

```csharp
public override void PreConfigureServices(ServiceConfigurationContext context)
{
    context.Services.AddAuthentication()
        .AddGoogle(GoogleDefaults.AuthenticationScheme, options =>
        {
            options.ClientId = context.Services.GetConfiguration()["Authentication:Google:ClientId"];
            options.ClientSecret = context.Services.GetConfiguration()["Authentication:Google:ClientSecret"];
            options.Scope.Add("email");
            options.SaveTokens = true;
        });
}
```

**Key lines:** ABP's Account module renders an external-login button for every authentication scheme registered here; the icon/label come from AbpAccountOptions.WindowsAuthenticationSchemeName and the AccountExternalProviderIconNames map. ClientId/ClientSecret should sit in User Secrets locally and Key Vault / environment variables in production. For Account Pro, IPostConfigureAccountExternalProviderOptions<GoogleOptions> lets you override these per tenant.

### Configure password policy and lockout via IdentityOptions

_Tightening defaults for a regulated environment._

```csharp
Configure<IdentityOptions>(options =>
{
    options.Password.RequiredLength = 12;
    options.Password.RequireDigit = true;
    options.Password.RequireNonAlphanumeric = true;
    options.Password.RequireUppercase = true;
    options.Password.RequiredUniqueChars = 4;

    options.Lockout.MaxFailedAccessAttempts = 5;
    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);

    options.SignIn.RequireConfirmedEmail = true;
});
```

**Key lines:** Static configuration applies to the host. Per-tenant overrides at runtime go through ISettingManager.SetForCurrentTenantAsync(IdentitySettingNames.Password.RequiredLength, "12"); the Identity module reads these settings into IdentityOptions on each request via its options-builder.

## Common mistakes

### Constructing IdentityUser directly with `new IdentityUser(...)` and saving via IRepository<IdentityUser, Guid>.InsertAsync, bypassing IdentityUserManager.

**Why wrong:** Skips password hashing, normalized-name population, security-stamp generation, lockout init, and tenant-id assignment that IdentityUserManager.CreateAsync handles. The user is then unable to log in and several admin operations throw NRE on missing security stamp.

**Correct pattern:** Always go through `IdentityUserManager.CreateAsync(user, password)` and `_userManager.AddToRoleAsync(user, roleName)`; check the returned IdentityResult with `.CheckErrors()`.

### Inserting OpenIddict applications directly into the OpenIddictApplications table (or via the EF DbContext) instead of using IOpenIddictApplicationManager.CreateAsync(OpenIddictApplicationDescriptor).

**Why wrong:** ClientSecret must be hashed with OpenIddict's password hasher; raw inserts store the secret in plaintext or with the wrong hash, and authentication silently fails with 'invalid client'. Permissions/RedirectUris/Properties are JSON-serialized on save — direct inserts skip that.

**Correct pattern:** Always seed/register clients via `IOpenIddictApplicationManager.CreateAsync` with an `OpenIddictApplicationDescriptor`. Update via `_applicationManager.UpdateAsync(application, descriptor)`.

### Leaving `AddDevelopmentEncryptionAndSigningCertificate = true` in production (e.g., when deploying to Linux containers or IIS).

**Why wrong:** The dev certificate is generated in the *current user's* X.509 store. In a container the app pool/service-account user has no such store, so OpenIddict either crashes at startup or signs tokens with a key that is regenerated each cold-start, invalidating every issued token after a restart.

**Correct pattern:** In production, set `options.AddDevelopmentEncryptionAndSigningCertificate = false` on AbpOpenIddictAspNetCoreOptions and register explicit X.509 certs via PreConfigure<OpenIddictServerBuilder>(b => b.AddSigningCertificate(cert).AddEncryptionCertificate(cert)). Persist the cert in Key Vault / a mounted secret, not the repo.

### Adding both AbpIdentityServerModule (legacy) and AbpOpenIddictModule to the same host.

**Why wrong:** ABP docs explicitly state 'You can not use IdentityServer and OpenIddict modules together. They are separate OpenID provider libraries for the same job.' Discovery endpoints conflict, JWKS endpoints conflict, and dependency-injection registers two competing IPasswordHasher / token-validation pipelines.

**Correct pattern:** On new 10.3 solutions use only the OpenIddict module. To migrate an existing IdentityServer-based solution, follow the official 'Migrating from IdentityServer to OpenIddict' guide and remove every Volo.Abp.IdentityServer.* package and DependsOn entry.

### Pointing a React Native or .NET MAUI mobile client at the AuthServer's HTTPS dev URL and getting silent 'invalid_client' / 'unauthorized_client' responses.

**Why wrong:** Mobile emulators do not trust the dev HTTPS cert; OpenIddict's RequireHttpsMetadata is enforced by default. The browser-based auth flow inside the emulator cannot complete and the token endpoint refuses the redirect.

**Correct pattern:** In Development, bind Kestrel to `http://0.0.0.0:<port>`, call `builder.UseAspNetCore().DisableTransportSecurityRequirement()` on the OpenIddictServerBuilder, and update the client's apiUrl/issuer in Environment.js (RN) or appsettings.json (MAUI) to the same HTTP URL. Never disable TLS in Production.

### Calling `_userManager.GetUserAsync(currentUser.Id)` from a microservice that does not host the Identity tables.

**Why wrong:** IdentityUserManager runs against the local DbContext; in a microservice that has no Identity DbContext registered, this throws 'No DbContext for IdentityUser was registered'. Hand-rolling a HTTP call to the Identity service duplicates logic.

**Correct pattern:** Inject IIdentityUserIntegrationService (or IExternalUserLookupServiceProvider) and call its lookup methods. The HttpClient-backed implementation `HttpClientExternalUserLookupServiceProvider` ships with Volo.Abp.Identity.HttpApi.Client and goes through the standard ABP dynamic-client-proxy pipeline.

### Forking the Account module's Razor Pages to add a logo or change the layout.

**Why wrong:** Forking divorces you from upstream bug fixes and Pro feature updates; ABP exposes virtual-file-system overrides and theme branding for these cases.

**Correct pattern:** Implement IBrandingProvider for app name/logo. To customize a single page, use ABP's virtual-file override (drop a Razor file at the same path under your Web project). To swap the entire login UI, replace it via the page-replacement attribute documented under the Account module's customization section. [uncertain] — exact attribute name.

### Mixing role names with different casing assuming ABP normalizes on read.

**Why wrong:** ABP normalizes only when persisting; queries that filter on raw `.Name` against an unnormalized stored value fail. Conversely, look-ups via `.NormalizedName` require uppercase input.

**Correct pattern:** Use IIdentityRoleRepository.FindByNormalizedNameAsync(role.ToUpperInvariant()) or _roleManager.FindByNameAsync(role); never compare Name directly with `==` in a query.

## Version pins (ABP 10.3)

- ABP 10.3 ships OpenIddict as the default OAuth2/OIDC server; IdentityServer module is documented but explicitly superseded since v6.0 — new projects must use OpenIddict.
- Identity Pro, Account Pro, OpenIddict Pro, and IdentityServer Pro all require an ABP Team license or higher (Community OSS license excludes them).
- AbpOpenIddictAspNetCoreOptions.AddDevelopmentEncryptionAndSigningCertificate defaults to true and is named exactly that in 10.3; older docs may use a different setting name. [uncertain] — name stability beyond 10.3.
- TokenCleanupOptions defaults: IsCleanupEnabled=true, CleanupPeriod=3,600,000 ms (1 hour), MinimumAuthorizationLifespan and MinimumTokenLifespan=14 days. These come from the ABP module on top of OpenIddict; raw OpenIddict has no cleanup background service.
- Connection-string conventions in 10.3: Identity uses 'AbpIdentity' (AbpIdentityDbProperties.ConnectionStringName) and OpenIddict uses 'AbpOpenIddict'; both fall back to 'Default' if the named string is absent.
- Default DB table prefix is 'Abp' for Identity (AbpIdentityDbProperties.DbTablePrefix) and 'OpenIddict' for the OpenIddict module entities; changing them requires a migration. [uncertain] — exact OpenIddict-side prefix constant name in 10.3.
- Distributed events for Identity (UserEto, IdentityRoleEto, IdentityClaimTypeEto, OrganizationUnitEto) wrap into EntityCreatedEto<T>/EntityUpdatedEto<T>/EntityDeletedEto<T>; subscribing modules need AbpEventBusModule + a transport.
- Account Pro's external providers (Twitter/Google/Microsoft) are pre-configured via AbpAccountOptions; Facebook is shipped as a documented add-on example. Provider list and pre-configuration set may evolve. [uncertain] — current pre-configured set in 10.3 vs 10.4+.
- AbpIdentityAspNetCoreOptions.ConfigureAuthentication defaults to true — setting it false stops the Identity module from auto-wiring AddAuthentication/AddIdentity, which is required when integrating a non-default auth pipeline.
- ABP's IdentityUserManager.UpdateSecurityStampAsync is invoked automatically by ABP after permission/role changes in 10.3; do not call it manually after RBAC mutations or you risk forcing two re-logins.
- MongoDB and EF Core support are at parity for both Identity and OpenIddict in 10.3; the same managers/repositories are used regardless of provider — the DbContext / collection mappings live in Volo.Abp.Identity.MongoDB / Volo.Abp.OpenIddict.MongoDB.

## Cross-references

**Phase 1 references:**
- [references/framework/authorization.md](../framework/authorization.md) — Identity supplies the user/role identities that ABP authorization grants permissions to; cross-link any time the question touches PermissionDefinitionProvider, RolePermissionValueProvider, IPermissionManager.SetForRoleAsync/SetForUserAsync, or [Authorize]/IAuthorizationService usage. [uncertain] — exact filename.
- [references/framework/multi-tenancy.md](../framework/multi-tenancy.md) — IdentityUser, IdentityRole, OpenIddictApplication and OpenIddictScope are tenant-aware; cross-link for questions about per-tenant user isolation, tenant-resolved login, or per-tenant external provider configuration. [uncertain] — exact filename.
- [references/framework/modularity.md](../framework/modularity.md) — Identity, Account, and OpenIddict are themselves ABP modules wired via [DependsOn]; cross-link for questions about which DependsOn to add when consuming them or when extending them in a custom module. [uncertain] — exact filename.
- [references/framework/data-seeding.md](../framework/data-seeding.md) — OpenIddict client/scope registration and the default Admin user are set up through IDataSeedContributor implementations; cross-link for any 'how do I create a client/role/user at first run' question. [uncertain] — exact filename.
- [references/framework/distributed-events.md](../framework/distributed-events.md) — UserEto / IdentityRoleEto / OrganizationUnitEto subscriptions; cross-link for any microservice replication question. [uncertain] — exact filename.
- [references/modules/permission-management.md](../modules/permission-management.md) — Permission Management module persists role/user grants used by Identity; bundled with Identity in startup templates. Cross-link when the user asks where 'permission grants' live. [uncertain] — whether this Phase-1 reference exists.

**External docs:**
- ASP.NET Core Identity — https://learn.microsoft.com/aspnet/core/security/authentication/identity — Underlying Microsoft library that IdentityUser/IdentityRole/IdentityUserManager/IdentityRoleManager derive from; relevant for password hashing, IdentityResult error codes, IUserStore<TUser> contract.
- ASP.NET Core Authentication — https://learn.microsoft.com/aspnet/core/security/authentication/ — AddAuthentication(), AddGoogle/AddMicrosoftAccount/AddFacebook, cookie vs JWT bearer schemes; relevant for wiring external/social logins on the Account module.
- OpenIddict documentation — https://documentation.openiddict.com/ — Reference for OpenIddictBuilder/OpenIddictServerBuilder, descriptors, permission constants, certificate guidance; ABP's OpenIddict module is a thin persistence + UI wrapper over this library.
- OpenID Connect Core 1.0 — https://openid.net/specs/openid-connect-core-1_0.html — Specification for /connect/authorize, /connect/token, ID-token claims; useful when debugging client-misconfiguration errors.
- OAuth 2.0 RFC 6749 — https://datatracker.ietf.org/doc/html/rfc6749 — Underlying OAuth flows (authorization_code, client_credentials, password, refresh_token) that the OpenIddict module exposes.
- Microsoft Identity claim types — https://learn.microsoft.com/dotnet/api/system.security.claims.claimtypes — Standard claim URIs that AbpOpenIddictAspNetCoreOptions.UpdateAbpClaimTypes synchronizes.
- X.509 certificates on .NET — https://learn.microsoft.com/dotnet/standard/security/cryptography-model — Required reading when replacing OpenIddict's dev cert with a production cert.
- Migrating from IdentityServer to OpenIddict (ABP guide) — https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/migration-guides — Walkthrough of the v6.0+ replacement; relevant for any team upgrading a pre-6.0 ABP solution. [uncertain] — exact URL slug under /framework or /tutorials.

## Sources (ABP 10.3)

- [https://abp.io/docs/10.3/modules/identity — Open-source Identity module](https://abp.io/docs/10.3/modules/identity)
- [https://abp.io/docs/10.3/modules/identity-pro — Identity Pro (Team+ license)](https://abp.io/docs/10.3/modules/identity-pro)
- [https://abp.io/docs/10.3/modules/account — Open-source Account module](https://abp.io/docs/10.3/modules/account)
- [https://abp.io/docs/10.3/modules/account-pro — Account Pro (Team+ license)](https://abp.io/docs/10.3/modules/account-pro)
- [https://abp.io/docs/10.3/modules/openiddict — Open-source OpenIddict module](https://abp.io/docs/10.3/modules/openiddict)
- [https://abp.io/docs/10.3/modules/openiddict-pro — OpenIddict Pro (Team+ license)](https://abp.io/docs/10.3/modules/openiddict-pro)
- [https://abp.io/docs/10.3/modules/identity-server — Legacy IdentityServer module (superseded by OpenIddict after v6.0)](https://abp.io/docs/10.3/modules/identity-server)
- [https://abp.io/docs/10.3/framework/fundamentals/authorization — Identity ↔ permission system bridge](https://abp.io/docs/10.3/framework/fundamentals/authorization)

Last verified: 2026-05-10
