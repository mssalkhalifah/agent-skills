# Infrastructure — BLOB Storing (all providers)

> Reference for ABP's BLOB Storing system — a provider-agnostic abstraction (IBlobContainer / IBlobContainer<T>) with pluggable backends (Memory, File System, Database, Azure, AWS S3, Google Cloud Storage, Aliyun OSS, MinIO, Bunny.net, custom) and built-in multi-tenant isolation.

## When to load this reference

Load when the user asks any of: 'how do I store/save/upload files in ABP', 'how to read/download a file or stream from ABP storage', 'how do I configure Azure Blob / S3 / MinIO / Google Cloud / Aliyun OSS / Bunny / file system / in-memory storage in ABP', 'how do I switch BLOB providers per container', 'what is IBlobContainer / IBlobContainer<T> / BlobContainerName', 'how does ABP isolate tenant files', 'how to write a custom BLOB provider', 'how to migrate the BlobStoring database schema', 'AbpBlobStoring connection string', 'what NuGet package for ABP file storage'.

**Audience:** ABP developers persisting binary content (images, documents, attachments) who need to choose, configure, or swap a storage backend, isolate BLOBs per tenant, define typed containers, or build a provider for an unsupported backend.

## Key concepts

- IBlobContainer (Volo.Abp.BlobStoring): default container interface for the application — methods SaveAsync(name, Stream/byte[], overrideExisting), GetAsync, GetOrNullAsync, GetAllBytesAsync, GetAllBytesOrNullAsync, DeleteAsync, ExistsAsync. Reach for it when a single, app-wide BLOB store is enough.
- IBlobContainer<TContainer> (Volo.Abp.BlobStoring): typed/generic container that gets its own configuration and provider. Inject this when a module or feature needs its own physically separate store (e.g., ProfilePictureContainer vs ReportContainer).
- BlobContainerName attribute (Volo.Abp.BlobStoring): decorates the marker class used as the generic argument; provides a stable string container identity that survives class renames. Apply on every typed-container marker class.
- IBlobContainerFactory (Volo.Abp.BlobStoring): runtime factory — Create(string name) and Create<T>() — for code that needs to pick a container dynamically rather than via DI generic type.
- IBlobProvider (Volo.Abp.BlobStoring): the per-backend implementation contract; concrete providers (FileSystemBlobProvider, AzureBlobProvider, AwsBlobProvider, GoogleBlobProvider, MinioBlobProvider, AliyunBlobProvider, BunnyBlobProvider, DatabaseBlobProvider, MemoryBlobProvider, custom) all implement this.
- BlobProviderBase (Volo.Abp.BlobStoring): abstract base used when authoring a custom provider; override SaveAsync, DeleteAsync, ExistsAsync, GetOrNullAsync taking BlobProviderSaveArgs / DeleteArgs / ExistsArgs / GetArgs.
- IBlobProviderSelector (Volo.Abp.BlobStoring): resolves which IBlobProvider serves a container at runtime by inspecting BlobContainerConfiguration.ProviderType.
- IBlobContainerConfigurationProvider (Volo.Abp.BlobStoring): looks up a container's BlobContainerConfiguration (provider type, multi-tenancy flag, provider-specific options).
- AbpBlobStoringOptions / BlobContainerConfigurations (Volo.Abp.BlobStoring): the central options class; expose Configure<T>(...), Configure(name,...), ConfigureDefault(...), ConfigureAll((name,cfg)=>...).
- BlobContainerConfiguration.IsMultiTenant (bool, default true): toggles per-tenant isolation; when true, providers prefix paths/keys with 'host' or 'tenants/<tenantId>'.
- BlobContainerConfiguration.ProviderType (Type): the IBlobProvider implementation; UseMemory/UseFileSystem/UseDatabase/UseAzure/UseAws/UseGoogle/UseMinio/UseAliyun/UseBunny extension methods set this and store provider-specific values via SetConfiguration<T>.
- Provider-specific name calculators (IAzureBlobNameCalculator, IAwsBlobNameCalculator, IBunnyBlobNameCalculator, IBlobFilePathCalculator): compute the final key/path including tenant scoping; replace via DI to override layout.

## Configuration pattern

All wiring happens in your AbpModule's ConfigureServices(ServiceConfigurationContext context) via Configure<AbpBlobStoringOptions>(...). The options object exposes a Containers dictionary with four shaping methods:

```csharp
[DependsOn(
    typeof(AbpBlobStoringModule),
    typeof(AbpBlobStoringFileSystemModule) // or any provider module
)]
public class MyAppModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        Configure<AbpBlobStoringOptions>(options =>
        {
            // 1. Default container fallback (for IBlobContainer and any unconfigured typed container)
            options.Containers.ConfigureDefault(container =>
            {
                container.UseFileSystem(fs =>
                {
                    fs.BasePath = "/var/app/blobs";
                });
                container.IsMultiTenant = true; // default
            });

            // 2. Configure a specific typed container by marker class
            options.Containers.Configure<ProfilePictureContainer>(container =>
            {
                container.UseAzure(az =>
                {
                    az.ConnectionString = context.Services
                        .GetConfiguration()["Azure:Storage:ConnectionString"];
                    az.ContainerName = "profile-pictures";
                    az.CreateContainerIfNotExists = true;
                });
            });

            // 3. Configure by string container name (e.g., for IBlobContainerFactory.Create("reports"))
            options.Containers.Configure("reports", container =>
            {
                container.UseDatabase();
            });

            // 4. Apply settings to ALL containers (e.g., bulk multi-tenant toggle)
            options.Containers.ConfigureAll((name, cfg) =>
            {
                cfg.IsMultiTenant = true;
            });
        });
    }
}
```

Key rules: (a) Provider modules must be added to [DependsOn] alongside AbpBlobStoringModule. (b) Configure<T>(...) takes precedence over ConfigureDefault for that typed container. (c) ConfigureAll runs after others and can override them — order matters. (d) The Database provider additionally needs ConfigureBlobStoring() inside the migration DbContext's OnModelCreating, plus the AbpBlobStoring connection string (or it falls back to Default).

## Code examples

Example 1 — Typed container with BlobContainerName and consumer service:
Scenario: A module persists user profile pictures in its own container so storage and tenant policy are independent of the rest of the app.
```csharp
using Volo.Abp.BlobStoring;
[BlobContainerName("profile-pictures")]
public class ProfilePictureContainer { }
public class ProfilePictureService : ITransientDependency
{
private readonly IBlobContainer<ProfilePictureContainer> _blobContainer;
public ProfilePictureService(IBlobContainer<ProfilePictureContainer> blobContainer)
=> _blobContainer = blobContainer;
public Task SaveAsync(Guid userId, Stream stream)
=> _blobContainer.SaveAsync(userId.ToString(), stream, overrideExisting: true);
public Task<byte[]?> GetOrNullAsync(Guid userId)
=> _blobContainer.GetAllBytesOrNullAsync(userId.ToString());
}
```
Key lines: `[BlobContainerName("profile-pictures")]` decouples the storage key from the C# class name (rename-safe). `IBlobContainer<ProfilePictureContainer>` tells DI which configuration to bind. `overrideExisting: true` is required to replace an existing BLOB (the default is false and would throw).
Example 2 — File System provider, default container:
Scenario: Single-server deployment writes uploads to local disk, with each tenant getting an isolated subfolder.
```csharp
[DependsOn(
typeof(AbpBlobStoringModule),
typeof(AbpBlobStoringFileSystemModule)
)]
public class MyAppModule : AbpModule
{
public override void ConfigureServices(ServiceConfigurationContext context)
{
Configure<AbpBlobStoringOptions>(options =>
{
options.Containers.ConfigureDefault(container =>
{
container.UseFileSystem(fs =>
{
fs.BasePath = "/var/app/blobs";
fs.AppendContainerNameToBasePath = true; // adds container subfolder
});
});
});
}
}
```
Key lines: `BasePath` is required and must be writable by the host process. With multi-tenancy on (default), files land under `/var/app/blobs/{containerName}/host/...` for the host or `/var/app/blobs/{containerName}/tenants/{tenantId}/...` for a tenant. Forward slashes inside the BLOB name (e.g., "images/avatars/x.png") create nested directories.
Example 3 — Database provider (EF Core) with migrations wired up:
Scenario: Centralized deployment without external object storage; BLOBs live in the same RDBMS as domain data so they participate in backups and transactions.
```csharp
[DependsOn(
typeof(AbpBlobStoringModule),
typeof(BlobStoringDatabaseDomainSharedModule),
typeof(BlobStoringDatabaseDomainModule),
typeof(BlobStoringDatabaseEntityFrameworkCoreModule)
)]
public class MyAppEntityFrameworkCoreModule : AbpModule
{
public override void ConfigureServices(ServiceConfigurationContext context)
{
Configure<AbpBlobStoringOptions>(options =>
{
options.Containers.ConfigureDefault(container =>
{
container.UseDatabase();
});
});
}
}
// In your migration DbContext:
public class MyAppMigrationsDbContext : AbpDbContext<MyAppMigrationsDbContext>
{
protected override void OnModelCreating(ModelBuilder builder)
{
base.OnModelCreating(builder);
builder.ConfigureBlobStoring(); // creates AbpBlobContainers + AbpBlobs tables
}
}
```
Key lines: `BlobStoringDatabaseEntityFrameworkCoreModule` plus `builder.ConfigureBlobStoring()` together register `DatabaseBlobContainer` and `DatabaseBlob` aggregates. After this, `dotnet ef migrations add ...` produces the schema. To use a separate database, add a connection string named `AbpBlobStoring` to appsettings.json; otherwise the `Default` connection string is reused.
Example 4 — Mixing providers per container (Azure for one, S3 for another):
Scenario: Marketing assets live on Azure (CDN-friendly), customer reports live on S3 (regulatory locality), profile pictures stay on local disk.
```csharp
[BlobContainerName("marketing-assets")] public class MarketingAssetsContainer { }
[BlobContainerName("customer-reports")] public class CustomerReportsContainer { }
[BlobContainerName("profile-pictures")] public class ProfilePictureContainer { }
Configure<AbpBlobStoringOptions>(options =>
{
options.Containers.Configure<MarketingAssetsContainer>(c =>
c.UseAzure(az =>
{
az.ConnectionString = configuration["Azure:Storage:ConnectionString"];
az.ContainerName = "marketing-assets";
az.CreateContainerIfNotExists = true;
}));
options.Containers.Configure<CustomerReportsContainer>(c =>
c.UseAws(aws =>
{
aws.AccessKeyId = configuration["Aws:AccessKeyId"];
aws.SecretAccessKey = configuration["Aws:SecretAccessKey"];
aws.Region = "eu-west-1";
aws.ContainerName = "acme-customer-reports";
aws.UseCredentials = true;
aws.CreateContainerIfNotExists = true;
}));
options.Containers.Configure<ProfilePictureContainer>(c =>
c.UseFileSystem(fs => fs.BasePath = "/var/app/profile-pictures"));
});
```
Key lines: each typed container can have a different provider; the provider-module DependsOn list must include every backend you reference (AbpBlobStoringAzureModule + AbpBlobStoringAwsModule + AbpBlobStoringFileSystemModule). Note ContainerName is the cloud-side bucket/container, while BlobContainerName is the ABP-side identity.
Example 5 — Custom provider using BlobProviderBase:
Scenario: Persist BLOBs in an in-house key-value store (e.g., Redis, etcd, proprietary CDN) for which no first-party provider exists.
```csharp
using Volo.Abp.BlobStoring;
using Volo.Abp.DependencyInjection;
public class MyKvBlobProvider : BlobProviderBase, ITransientDependency
{
private readonly IMyKvClient _kv;
public MyKvBlobProvider(IMyKvClient kv) => _kv = kv;
public override async Task SaveAsync(BlobProviderSaveArgs args)
{
if (!args.OverrideExisting && await ExistsAsync(args))
throw new BlobAlreadyExistsException(
$"Blob '{args.BlobName}' already exists in container '{args.ContainerName}'.");
var bytes = await args.BlobStream.GetAllBytesAsync();
await _kv.PutAsync(Key(args), bytes);
}
public override async Task<bool> DeleteAsync(BlobProviderDeleteArgs args)
=> await _kv.DeleteAsync(Key(args));
public override async Task<bool> ExistsAsync(BlobProviderExistsArgs args)
=> await _kv.ExistsAsync(Key(args));
public override async Task<Stream?> GetOrNullAsync(BlobProviderGetArgs args)
{
var bytes = await _kv.GetOrNullAsync(Key(args));
return bytes is null ? null : new MemoryStream(bytes);
}
private static string Key(BlobProviderArgs args)
=> $"{args.ContainerName}/{args.BlobName}";
}
// Optional fluent extension:
public static class MyKvBlobContainerConfigurationExtensions
{
public static BlobContainerConfiguration UseMyKv(
this BlobContainerConfiguration cfg)
{
cfg.ProviderType = typeof(MyKvBlobProvider);
return cfg;
}
}
// Wire-up:
Configure<AbpBlobStoringOptions>(o =>
o.Containers.ConfigureDefault(c => c.UseMyKv()));
```
Key lines: `BlobProviderBase` is mandatory only if you want default plumbing — implementing `IBlobProvider` directly is also valid. Set `ProviderType = typeof(YourProvider)` (directly or via an extension method) to register; the provider must be DI-registered (here via `ITransientDependency`). Always honour `args.OverrideExisting` and the multi-tenancy contract by deriving keys from `args.ContainerName` plus tenant scope as needed.
## Common mistakes

- - Forgetting overrideExisting=true: SaveAsync defaults overrideExisting to false and throws if the BLOB already exists. Why it's wrong: callers expect upload-or-replace semantics; the default protects accidental overwrites. Correct pattern: pass `overrideExisting: true` for replace flows, or call `ExistsAsync` first and branch.
- - Calling GetAsync when a BLOB might be missing: GetAsync throws BlobNotFoundException for a missing key. Use GetOrNullAsync / GetAllBytesOrNullAsync for optional reads; reserve GetAsync for required-by-contract reads.
- - Renaming the typed-container marker class without the BlobContainerName attribute: without [BlobContainerName("...")], ABP derives the container key from the class's full name; renaming or moving the namespace silently changes the container path and orphans existing BLOBs. Correct: always decorate marker classes with a stable [BlobContainerName].
- - Forgetting [DependsOn(typeof(AbpBlobStoringXyzModule))] for the chosen backend. ConfigureDefault(c => c.UseAzure(...)) compiles fine, but the AzureBlobProvider isn't registered, so requests fail at runtime with 'No provider registered'. Add the provider module to your AbpModule's [DependsOn].
- - Database provider: skipping `builder.ConfigureBlobStoring()` in the migration DbContext. Without this, EF migrations don't include AbpBlobContainers/AbpBlobs tables and runtime calls fail. Correct pattern: call ConfigureBlobStoring() inside OnModelCreating right after `base.OnModelCreating(builder)`.
- - Database provider: assuming a separate DB without configuring the AbpBlobStoring connection string. The provider falls back to `Default`, so BLOBs land in the main database unless you explicitly add `"AbpBlobStoring": "..."` under ConnectionStrings.
- - Azure / S3 / GCS / MinIO / Bunny: violating cloud naming rules. Each backend enforces its own bucket/container constraints (Azure: lowercase, 3-63 chars, no consecutive hyphens; S3/MinIO: 3-63 chars, no IP-format; Bunny: 4-64 chars, globally unique). Correct: explicitly set ContainerName/BucketName to a compliant string instead of relying on the [BlobContainerName] default, especially for class names with uppercase or underscores.
- - Setting CreateContainerIfNotExists=false in production then deploying to an empty account. The first SaveAsync throws because the cloud-side bucket/container doesn't exist. Either pre-create infrastructure (recommended) or set the flag to true with proper IAM permissions.
- - Disabling IsMultiTenant on a multi-tenant app to 'simplify paths'. This causes tenants to share a flat namespace and overwrite each other's files using the same BLOB name. Keep IsMultiTenant=true (the default) unless the container is genuinely shared (e.g., a public marketing-assets container).
- - Using the Memory provider in production: Volo.Abp.BlobStoring.Memory is for tests/demos only; data is lost on restart and not synchronized across pods/instances.
- - Storing very large files in the Database provider: BLOBs become rows whose payloads bloat backups, replication, and EF Core memory usage. For >~10 MB blobs, prefer object storage (S3/Azure/GCS/MinIO) or the file-system provider on a shared volume.
- - AWS provider: setting AccessKeyId/SecretAccessKey but leaving UseCredentials=false. The provider then ignores explicit keys and falls through to default credential resolution (env vars, instance profile). Set `UseCredentials = true` (or use UseTemporaryCredentials / UseTemporaryFederatedCredentials / ProfileName depending on auth mode).
- - Forgetting that paths use forward slashes: BLOB names like `images/avatars/x.png` create nested folders on the file-system provider and key prefixes on object stores. Mixing backslashes is non-portable.
## Version pins (ABP 10.3)

- Pinned to ABP 10.3 NuGet package names: Volo.Abp.BlobStoring, Volo.Abp.BlobStoring.Memory, Volo.Abp.BlobStoring.FileSystem, Volo.Abp.BlobStoring.Database.EntityFrameworkCore, Volo.Abp.BlobStoring.Database.MongoDB, Volo.Abp.BlobStoring.Azure, Volo.Abp.BlobStoring.Aliyun, Volo.Abp.BlobStoring.Minio, Volo.Abp.BlobStoring.Aws, Volo.Abp.BlobStoring.Google, Volo.Abp.BlobStoring.Bunny.
- IsMultiTenant default is true in 10.3; do not assume this default is stable across major versions.
- BlobContainerConfiguration uses ProviderType plus a string-keyed configuration bag (SetConfiguration<T>/GetConfiguration<T>); custom-provider configuration wrappers depend on this contract — treat key strings (e.g., "MyCustomBlobProvider.MyOption1") as private-to-the-provider.
- Database provider in 10.3 ships both EF Core and MongoDB packages and uses the connection-string name `AbpBlobStoring` (falls back to `Default`); both are 10.3 conventions.
- Bunny.net region codes documented in 10.3: DE, NY, LA, SG (default DE). [uncertain] whether additional regions (e.g., new POPs) have been added since the 10.3 docs were authored.
- AWS provider exposes UseCredentials, UseTemporaryCredentials, and UseTemporaryFederatedCredentials as separate booleans in 10.3; precedence among them when more than one is true is [uncertain] in the public docs — set exactly one.
- Aliyun STS options (UseSecurityTokenService, RegionId, RoleArn, RoleSessionName, DurationSeconds) are documented for 10.3; DurationSeconds bounds (900-3600) are provider-enforced and may shift if Aliyun changes its STS limits.
- MinIO provider's PresignedGetExpirySeconds bounds are 1-604800 with a 7-day default in 10.3; whether this is exposed for non-presigned reads is [uncertain].
- The custom-provider contract (BlobProviderBase + BlobProviderSaveArgs/DeleteArgs/ExistsArgs/GetArgs) is stable in 10.3 but is the surface most likely to gain optional methods (e.g., streaming/range reads) in future majors — code defensively against breaking signature changes.
- File-system provider's IBlobFilePathCalculator and Azure/AWS/Bunny IXxxBlobNameCalculator services are public extension points in 10.3; no guarantee these interfaces stay byte-compatible across majors.

## Cross-references

**Phase 1 references:**
- references/framework/multi-tenancy.md — triggered whenever IsMultiTenant or tenant-prefixed BLOB paths come up; the BlobStoring system inherits its tenant-scoping behaviour from the multi-tenancy layer.
- references/framework/architecture-modularity.md — triggered when explaining DependsOn for AbpBlobStoringModule and the provider-specific modules (Memory/FileSystem/Database/Azure/Aws/Google/Aliyun/Minio/Bunny).
- references/framework/fundamentals.md — triggered when discussing IBlobContainer<T> resolution, ITransientDependency, and registering custom providers.
- references/framework/data-ef-core.md — triggered by the Database provider: ConfigureBlobStoring() in the migration DbContext, AbpBlobStoring connection string, EF migrations.
- references/modules/content-and-cms.md — triggered when users ask 'do I use the File Management module or BLOB Storing directly?'; the File Management module is built on top of IBlobContainer.

**External docs:**
- Microsoft Azure Storage Blobs — https://learn.microsoft.com/azure/storage/blobs/ — referenced because the Azure provider wraps Azure.Storage.Blobs and inherits Azure's container naming rules and Shared Key auth.
- Amazon S3 documentation — https://docs.aws.amazon.com/s3/ — referenced for AWS S3 bucket naming rules, regions, and IAM credential models that the AWS provider exposes (UseCredentials/UseTemporaryCredentials/UseTemporaryFederatedCredentials, ProfileName, Policy).
- Google Cloud Storage docs — https://cloud.google.com/storage/docs — referenced for service-account credentials, ADC (UseApplicationDefaultCredentials), bucket naming, and OAuth scopes consumed by the Google provider.
- MinIO documentation — https://min.io/docs/minio/linux/index.html — referenced for endpoint setup, access keys, S3-compatible bucket naming, and presigned URL semantics (PresignedGetExpirySeconds).
- Aliyun OSS documentation — https://www.alibabacloud.com/help/en/oss — referenced for endpoints, AccessKeyId/Secret, STS RoleArn flow, and bucket naming used by the Aliyun provider.
- Bunny.net Storage docs — https://docs.bunny.net/reference/storage-api — referenced for storage-zone access keys, region codes (DE, NY, LA, SG), and zone naming rules used by the Bunny provider.
- Entity Framework Core migrations — https://learn.microsoft.com/ef/core/managing-schemas/migrations/ — referenced because the Database provider's schema is created via standard EF Core migrations after ConfigureBlobStoring().
- .NET Stream API — https://learn.microsoft.com/dotnet/api/system.io.stream — referenced because IBlobContainer.SaveAsync/GetAsync exchange System.IO.Stream and consumers must dispose returned streams.

## Sources (ABP 10.3)

- https://abp.io/docs/10.3/framework/infrastructure/blob-storing — BLOB Storing system overview, IBlobContainer, IBlobContainer<T>, IBlobContainerFactory, AbpBlobStoringOptions, BlobContainerName attribute, multi-tenancy, IBlobProviderSelector
- https://abp.io/docs/10.3/framework/infrastructure/blob-storing/memory — Memory provider (Volo.Abp.BlobStoring.Memory), AbpBlobStoringMemoryModule, UseMemory(), tests/demos only
- https://abp.io/docs/10.3/framework/infrastructure/blob-storing/file-system — File System provider (Volo.Abp.BlobStoring.FileSystem), BasePath, AppendContainerNameToBasePath, IBlobFilePathCalculator
- https://abp.io/docs/10.3/framework/infrastructure/blob-storing/database — Database provider, EF Core (Volo.Abp.BlobStoring.Database.EntityFrameworkCore) and MongoDB (Volo.Abp.BlobStoring.Database.MongoDB), ConfigureBlobStoring(), AbpBlobStoring connection string
- https://abp.io/docs/10.3/framework/infrastructure/blob-storing/azure — Azure Blob Storage (Volo.Abp.BlobStoring.Azure), ConnectionString, ContainerName, CreateContainerIfNotExists
- https://abp.io/docs/10.3/framework/infrastructure/blob-storing/aliyun — Aliyun OSS (Volo.Abp.BlobStoring.Aliyun), AccessKeyId/Secret, Endpoint, STS options
- https://abp.io/docs/10.3/framework/infrastructure/blob-storing/minio — MinIO (Volo.Abp.BlobStoring.Minio), EndPoint, AccessKey, SecretKey, BucketName, WithSSL, PresignedGetExpirySeconds
- https://abp.io/docs/10.3/framework/infrastructure/blob-storing/aws — AWS S3 (Volo.Abp.BlobStoring.Aws), AccessKeyId, SecretAccessKey, Region, UseCredentials, UseTemporaryCredentials, UseTemporaryFederatedCredentials, ProfileName, Policy
- https://abp.io/docs/10.3/framework/infrastructure/blob-storing/google — Google Cloud Storage (Volo.Abp.BlobStoring.Google), ClientEmail, ProjectId, PrivateKey, UseApplicationDefaultCredentials
- https://abp.io/docs/10.3/framework/infrastructure/blob-storing/bunny — Bunny.net (Volo.Abp.BlobStoring.Bunny), AccessKey, Region (DE/NY/LA/SG), ContainerName
- https://abp.io/docs/10.3/framework/infrastructure/blob-storing/custom-provider — Building a custom provider via BlobProviderBase, BlobProviderSaveArgs/DeleteArgs/ExistsArgs/GetArgs, ProviderType registration, configuration wrapper pattern

Last verified: 2026-05-10
