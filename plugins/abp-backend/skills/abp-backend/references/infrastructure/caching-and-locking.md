# Infrastructure — Distributed Cache, Entity Cache, Locking, Redis

> Covers ABP's infrastructure for distributed caching (IDistributedCache<T>), high-level entity caching (IEntityCache), Redis integration (Volo.Abp.Caching.StackExchangeRedis), and distributed locking (IAbpDistributedLock) for safely sharing state and coordinating work across application instances.

## When to load this reference

- User asks how to cache data in ABP or inject IDistributedCache<T>.
- User wants to switch ABP's default in-memory cache to Redis.
- User asks about cache key prefixes, multi-tenant cache isolation, or [IgnoreMultiTenancy] on cache items.
- User asks how to invalidate a cached entity automatically when it changes (Entity Cache).
- User asks about distributed locks, mutex across nodes, IAbpDistributedLock, or running scheduled jobs safely on a multi-node deployment.
- User asks how to configure Medallion DistributedLock with Redis in an ABP module.
- User encounters cache serialization issues, sliding vs absolute expiration questions, or considerUow / unit-of-work-aware cache writes.
- User asks 'how do I prevent two app instances from running the same code at once' in ABP.

**Audience:** ABP developers building multi-instance / clustered applications who need shared state, cross-node coordination, or read-through entity caches; module authors integrating Redis as the cache and lock backend; anyone scaling an ABP solution beyond a single process.

## Key concepts

- IDistributedCache<TCacheItem> (Volo.Abp.Caching) - Generic distributed cache with string keys; reach for it whenever you want strongly-typed cached values that auto-namespace per type and per tenant.
- IDistributedCache<TCacheItem, TCacheKey> (Volo.Abp.Caching) - Same as above with a typed key (e.g., Guid or a custom key class with overridden ToString); use to avoid manual key string-formatting.
- AbpDistributedCacheOptions (Volo.Abp.Caching) - Central configuration: KeyPrefix, HideErrors, GlobalCacheEntryOptions; configure once in your module's ConfigureServices.
- IDistributedCacheSerializer / Utf8JsonDistributedCacheSerializer (Volo.Abp.Caching) - Default JSON-to-UTF8 serializer; replace in DI to swap serialization (e.g., MessagePack).
- IDistributedCacheKeyNormalizer (Volo.Abp.Caching) - Builds the final cache key by combining application prefix, tenant id, cache name, and the user-supplied key; replace to customize key shape.
- [CacheName("...")] attribute (Volo.Abp.Caching) - Overrides the auto-derived cache name (which strips the 'CacheItem' suffix from the type name).
- [IgnoreMultiTenancy] (Volo.Abp.MultiTenancy) - Apply to a cache item class so the current tenant id is NOT appended to the cache key; use for host-wide reference data.
- DistributedCacheEntryOptions (Microsoft.Extensions.Caching.Distributed) - Standard expiration options used by ABP's cache APIs: AbsoluteExpiration, AbsoluteExpirationRelativeToNow, SlidingExpiration.
- Volo.Abp.Caching.StackExchangeRedis module - DependsOn this module and install Volo.Abp.Caching.StackExchangeRedis NuGet package to switch from in-memory to Redis; adds optimized SetManyAsync / GetManyAsync.
- RedisCacheOptions (Microsoft.Extensions.Caching.StackExchangeRedis) - Standard options class; configurable in ConfigureServices to override what was read from the 'Redis' configuration section.
- IEntityCache<TEntity, TKey> / IEntityCache<TEntity, TCacheItem, TKey> (Volo.Abp.EntityCache) - High-level read-through cache for entities with automatic invalidation on update/delete; pick this over raw IDistributedCache when caching repository lookups.
- EntityCacheWithObjectMapper<TEntity, TCacheItem, TKey> (Volo.Abp.EntityCache) - Base class for custom entity caches that map entity -> DTO via overridable MapToValue; subclass and register with ReplaceEntityCache.
- AddEntityCache / ReplaceEntityCache extension methods (on IServiceCollection) - Registration entry points for default and custom entity caches.
- IAbpDistributedLock (Volo.Abp.DistributedLocking) - ABP-facing locking abstraction: TryAcquireAsync(name, timeout, cancellationToken) returning a disposable handle (null on failure).
- AbpDistributedLockOptions (Volo.Abp.DistributedLocking) - Holds KeyPrefix to namespace lock names per app, mirroring AbpDistributedCacheOptions.KeyPrefix.
- IDistributedLockProvider (Medallion.Threading) - Underlying provider abstraction; ABP wraps whatever singleton you register (in-process by default, or RedisDistributedSynchronizationProvider for cross-node locks).
- Volo.Abp.DistributedLocking.Abstractions package - Lightweight abstractions + in-process fallback; depend on this from libraries/modules so they don't force a Redis dependency on consumers.
- Volo.Abp.DistributedLocking package - Full module (DependsOn(typeof(AbpDistributedLockingModule))) for applications that wire a real provider.

## Configuration pattern

Distributed cache, Redis, and distributed locking are all wired in the host module's ConfigureServices. Cache defaults are tuned via Configure<AbpDistributedCacheOptions>; Redis is added by depending on AbpCachingStackExchangeRedisModule and reading the 'Redis' section from configuration; locking is enabled by depending on AbpDistributedLockingModule and registering an IDistributedLockProvider singleton (in-process is the implicit fallback from the Abstractions package).

Example wiring:

```csharp
using Medallion.Threading.Redis;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using StackExchange.Redis;
using Volo.Abp;
using Volo.Abp.Caching;
using Volo.Abp.Caching.StackExchangeRedis;
using Volo.Abp.DistributedLocking;
using Volo.Abp.Modularity;

[DependsOn(
    typeof(AbpCachingModule),
    typeof(AbpCachingStackExchangeRedisModule),
    typeof(AbpDistributedLockingModule)
)]
public class MyAppModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        var configuration = context.Services.GetConfiguration();

        // 1) Cache defaults: prefix keys per app + global sliding expiration.
        Configure<AbpDistributedCacheOptions>(options =>
        {
            options.KeyPrefix = "MyApp1";
            options.HideErrors = true;
            options.GlobalCacheEntryOptions = new DistributedCacheEntryOptions
            {
                SlidingExpiration = TimeSpan.FromMinutes(20)
            };
        });

        // 2) Distributed lock prefix (mirrors cache prefix).
        Configure<AbpDistributedLockOptions>(options =>
        {
            options.KeyPrefix = "MyApp1";
        });

        // 3) Register Medallion Redis lock provider as a singleton.
        context.Services.AddSingleton<IDistributedLockProvider>(sp =>
        {
            var connection = ConnectionMultiplexer.Connect(
                configuration["Redis:Configuration"]!);
            return new RedisDistributedSynchronizationProvider(connection.GetDatabase());
        });
    }
}
```

appsettings.json:

```json
{
  "Redis": {
    "IsEnabled": "true",
    "Configuration": "127.0.0.1"
  }
}
```

Key options:
- AbpDistributedCacheOptions.KeyPrefix - prevents key collisions when multiple apps share one Redis.
- AbpDistributedCacheOptions.HideErrors (default true) - swallows cache backend errors and logs them; per-call override via hideErrors: false.
- AbpDistributedCacheOptions.GlobalCacheEntryOptions - default expiration applied when callers don't pass their own DistributedCacheEntryOptions.
- AbpDistributedLockOptions.KeyPrefix - applied to every lock name acquired through IAbpDistributedLock.
- 'Redis' configuration section - the Redis cache module reads it automatically; IsEnabled=false disables the Redis cache and falls back to MemoryDistributedCache.

## Code examples

### Read-through cache for an entity using IDistributedCache<T,TKey>

_Cache a Book lookup by Guid for one hour, hitting the database only on miss._

```csharp
using Microsoft.Extensions.Caching.Distributed;
using Volo.Abp.Caching;
using Volo.Abp.DependencyInjection;

[CacheName("Books")]
public class BookCacheItem
{
    public string Name { get; set; } = default!;
    public float Price { get; set; }
}

public class BookService : ITransientDependency
{
    private readonly IDistributedCache<BookCacheItem, Guid> _cache;

    public BookService(IDistributedCache<BookCacheItem, Guid> cache)
    {
        _cache = cache;
    }

    public Task<BookCacheItem> GetAsync(Guid bookId)
    {
        return _cache.GetOrAddAsync(
            bookId,
            () => LoadFromDbAsync(bookId),
            () => new DistributedCacheEntryOptions
            {
                AbsoluteExpiration = DateTimeOffset.Now.AddHours(1)
            });
    }

    private Task<BookCacheItem> LoadFromDbAsync(Guid bookId)
    {
        // Repository call goes here.
        return Task.FromResult(new BookCacheItem { Name = "Hamlet", Price = 9.99f });
    }
}
```

**Key lines:** [CacheName("Books")] sets the cache namespace; without it ABP would derive 'Book' (stripping 'CacheItem'). GetOrAddAsync is the canonical pattern - factory runs only on miss. The typed-key overload (IDistributedCache<BookCacheItem, Guid>) avoids manual ToString. AbsoluteExpiration overrides the GlobalCacheEntryOptions configured on AbpDistributedCacheOptions.

### Composite cache key with multi-tenancy

_Cache a per-(user, organization) projection. Tenant id is auto-appended to the key by IDistributedCacheKeyNormalizer._

```csharp
public class UserInOrgCacheKey
{
    public Guid UserId { get; set; }
    public Guid OrganizationId { get; set; }

    public override string ToString() => $"{UserId}_{OrganizationId}";
}

[CacheName("UserInOrg")]
public class UserInOrgCacheItem
{
    public string DisplayName { get; set; } = default!;
    public string[] Roles { get; set; } = Array.Empty<string>();
}

public class UserOrgLookupService : ITransientDependency
{
    private readonly IDistributedCache<UserInOrgCacheItem, UserInOrgCacheKey> _cache;

    public UserOrgLookupService(
        IDistributedCache<UserInOrgCacheItem, UserInOrgCacheKey> cache)
    {
        _cache = cache;
    }

    public Task<UserInOrgCacheItem> GetAsync(Guid userId, Guid orgId)
        => _cache.GetOrAddAsync(
            new UserInOrgCacheKey { UserId = userId, OrganizationId = orgId },
            () => BuildAsync(userId, orgId));

    private Task<UserInOrgCacheItem> BuildAsync(Guid userId, Guid orgId) =>
        Task.FromResult(new UserInOrgCacheItem { DisplayName = "..." });
}
```

**Key lines:** Override ToString on the key class - that string becomes part of the final cache key. ABP automatically prepends KeyPrefix, cache name, and current tenant id, so the same code in two tenants yields disjoint cache entries with no extra work. Add [IgnoreMultiTenancy] on UserInOrgCacheItem if the data must be shared across tenants.

### IEntityCache with auto-invalidation

_Frequently read a small reference table (Product) and never want stale data when it's updated._

```csharp
// In your module's ConfigureServices:
context.Services.AddEntityCache<Product, ProductCacheItem, Guid>(
    new DistributedCacheEntryOptions
    {
        SlidingExpiration = TimeSpan.FromMinutes(30)
    });

// Consumer:
public class CatalogService : ITransientDependency
{
    private readonly IEntityCache<ProductCacheItem, Guid> _products;

    public CatalogService(IEntityCache<ProductCacheItem, Guid> products)
    {
        _products = products;
    }

    public async Task<ProductCacheItem?> FindAsync(Guid id)
        => await _products.FindAsync(id);

    public Task<IReadOnlyList<ProductCacheItem>> FindManyAsync(IEnumerable<Guid> ids)
        => _products.FindManyAsync(ids.ToArray());
}
```

**Key lines:** AddEntityCache<TEntity, TCacheItem, TKey> registers the cache + a subscription that invalidates entries when the underlying entity is updated or deleted (via ABP's distributed/local event bus). Use FindAsync for nullable result, GetAsync to throw EntityNotFoundException. Treat the result as read-only - mutate via the regular repository, not via the cache.

### Custom EntityCacheWithObjectMapper for entity -> DTO mapping

_The entity is not directly serializable (lazy proxies, navigation properties), so map to a DTO before caching._

```csharp
public class ProductEntityCache
    : EntityCacheWithObjectMapper<Product, ProductCacheDto, Guid>
{
    public ProductEntityCache(
        IDistributedCache<ProductCacheDto, Guid> cache,
        IReadOnlyRepository<Product, Guid> repository,
        IDistributedEventBus distributedEventBus,
        IObjectMapper objectMapper,
        IEntityCacheOptionsProvider<Product, ProductCacheDto, Guid> optionsProvider)
        : base(cache, repository, distributedEventBus, objectMapper, optionsProvider)
    {
    }

    protected override ProductCacheDto MapToValue(Product entity)
    {
        return new ProductCacheDto
        {
            Id = entity.Id,
            Name = entity.Name.ToUpperInvariant(),
            Price = entity.Price
        };
    }
}

// Registration:
context.Services.ReplaceEntityCache<
    ProductEntityCache, Product, ProductCacheDto, Guid>(
    new DistributedCacheEntryOptions
    {
        AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
    });
```

**Key lines:** Subclass EntityCacheWithObjectMapper and override MapToValue to control the cached projection. Register with ReplaceEntityCache so the framework wires invalidation against your subclass. Cached DTO must be JSON-serializable. [uncertain] - the exact constructor signature of EntityCacheWithObjectMapper / IEntityCacheOptionsProvider parameter list is not fully documented on the 10.3 page; verify against the source if you target that base class directly.

### Distributed lock around a critical section

_A scheduled job runs on every node but must execute its body on only one node at a time._

```csharp
using Volo.Abp.DependencyInjection;
using Volo.Abp.DistributedLocking;

public class NightlyReportJob : ITransientDependency
{
    private readonly IAbpDistributedLock _distributedLock;

    public NightlyReportJob(IAbpDistributedLock distributedLock)
    {
        _distributedLock = distributedLock;
    }

    public async Task RunAsync(CancellationToken cancellationToken)
    {
        await using var handle = await _distributedLock.TryAcquireAsync(
            name: "NightlyReportJob",
            timeout: TimeSpan.FromSeconds(10),
            cancellationToken: cancellationToken);

        if (handle == null)
        {
            // Another node already holds the lock; bail out cleanly.
            return;
        }

        await GenerateReportsAsync(cancellationToken);
    }

    private Task GenerateReportsAsync(CancellationToken ct) => Task.CompletedTask;
}
```

**Key lines:** Always 'await using' the handle so the lock is released even on exceptions. TryAcquireAsync returns null on contention - never skip the null-check. The 'name' is namespaced by AbpDistributedLockOptions.KeyPrefix, so two apps sharing the same Redis cannot collide. Default timeout is TimeSpan.Zero (don't wait) - pass an explicit value when you actually want to block briefly.

## Common mistakes

### Leaving the default MemoryDistributedCache in production behind a load balancer.

**Why wrong:** Each app instance has an isolated in-memory cache, so cache hits/misses become non-deterministic across nodes and IEntityCache invalidation cannot propagate correctly.

**Correct pattern:** Depend on AbpCachingStackExchangeRedisModule and add the 'Redis' section to appsettings.json on every node so all instances share one backend.

```json
[DependsOn(typeof(AbpCachingStackExchangeRedisModule))] on the host module + Volo.Abp.Caching.StackExchangeRedis NuGet.
```

### Forgetting to set AbpDistributedCacheOptions.KeyPrefix when multiple apps share one Redis instance.

**Why wrong:** Two apps can both define a 'Books' cache and clobber each other's entries because the final key is just '<cache name>:<tenant id>:<key>' without an app-level namespace.

**Correct pattern:** Set a unique KeyPrefix per application, and mirror it in AbpDistributedLockOptions.KeyPrefix.

```csharp
Configure<AbpDistributedCacheOptions>(o => o.KeyPrefix = "MyApp1"); Configure<AbpDistributedLockOptions>(o => o.KeyPrefix = "MyApp1");
```

### Caching across tenants by accident, exposing one tenant's data to another.

**Why wrong:** If you set the cache key manually and bypass the typed cache, or if you cache at the host level and lookup at the tenant level, the auto-tenant-prefix is lost.

**Correct pattern:** Use IDistributedCache<TCacheItem, TKey> and let the framework append ICurrentTenant.Id automatically. Add [IgnoreMultiTenancy] only when the data is genuinely tenant-agnostic.

```csharp
Inject IDistributedCache<BookCacheItem, Guid> instead of IDistributedCache<BookCacheItem> with hand-built keys.
```

### Calling _cache.SetAsync in the middle of a unit of work without considerUow.

**Why wrong:** If the surrounding transaction rolls back, the cache still holds the would-be-committed value, producing a phantom record on subsequent reads.

**Correct pattern:** Pass considerUow: true so the cache write is buffered until the UoW commits.

```csharp
await _cache.SetAsync(key, value, options, considerUow: true);
```

### Treating IEntityCache results as mutable.

**Why wrong:** The system is documented as read-only; mutating a cached DTO does not persist to the database and may corrupt later cache reads.

**Correct pattern:** Read from IEntityCache, mutate via the standard IRepository<TEntity, TKey>; the cache will auto-invalidate on update/delete.

```csharp
Inject IRepository<Product, Guid> alongside IEntityCache<ProductCacheItem, Guid> for write paths.
```

### Not null-checking the handle returned by IAbpDistributedLock.TryAcquireAsync.

**Why wrong:** TryAcquireAsync returns null when the lock cannot be acquired within the timeout; dereferencing or proceeding past it executes the critical section without the lock.

**Correct pattern:** Always 'await using var handle = await _lock.TryAcquireAsync(...);' followed by 'if (handle == null) return;'.

```csharp
Add explicit handle == null guard before any state-mutating code.
```

### Shipping a library that takes a hard dependency on Volo.Abp.DistributedLocking (full module) instead of the Abstractions package.

**Why wrong:** Forces every consumer to install Medallion.Threading, even if they only need in-process locking.

**Correct pattern:** Library/module projects depend on Volo.Abp.DistributedLocking.Abstractions (which ships an in-process fallback). The host application installs the full Volo.Abp.DistributedLocking module + provider package.

```csharp
Replace the PackageReference in the library's .csproj with Volo.Abp.DistributedLocking.Abstractions.
```

### Using the same cache name for two unrelated types because both end with 'CacheItem'.

**Why wrong:** ABP derives the cache name by stripping 'CacheItem'. 'OrderCacheItem' in two namespaces both resolve to 'Order'.

**Correct pattern:** Apply [CacheName("...")] explicitly when the type-name-derived name is not unique.

```json
[CacheName("Sales.Order")] public class OrderCacheItem { ... }
```

### Setting HideErrors = false in production without a fallback path.

**Why wrong:** Any Redis hiccup now throws into the caller; ABP's default of HideErrors = true logs the error and falls back to the source-of-truth call.

**Correct pattern:** Keep HideErrors = true globally; only flip it per-call when a code path absolutely requires a fresh value.

```csharp
Remove the override or use the per-call hideErrors: false on the specific GetOrAddAsync invocation only.
```

## Version pins (ABP 10.3)

- ABP 10.3 - IDistributedCache<T> / IDistributedCache<T,TKey>, AbpDistributedCacheOptions (KeyPrefix, HideErrors, GlobalCacheEntryOptions), and the considerUow parameter on SetAsync are stable APIs.
- ABP 10.3 - Volo.Abp.Caching.StackExchangeRedis is the official Redis integration; it adds bulk SetManyAsync/GetManyAsync that the plain Microsoft package lacks.
- ABP 10.3 - 'Redis' configuration section ('IsEnabled', 'Configuration') is read automatically; IsEnabled defaults to true. [uncertain] - the exact precedence between Configure<RedisCacheOptions>(...) overrides and the auto-bound 'Redis' section is not spelled out on the docs page.
- ABP 10.3 - Default IEntityCache expiration is documented as ~2 minutes (AbsoluteExpirationRelativeToNow); pass an explicit DistributedCacheEntryOptions to AddEntityCache to override.
- ABP 10.3 - IAbpDistributedLock.TryAcquireAsync default timeout is TimeSpan.Zero (no wait). [uncertain] - whether passing a negative timeout means 'wait forever' is not documented; treat it as undefined.
- ABP 10.3 - Distributed locking is built on madelson/DistributedLock (Medallion.Threading); upgrading that library across major versions can change provider constructors (RedisDistributedSynchronizationProvider signature). Pin the DistributedLock.Redis version in your .csproj.
- ABP 10.3 - The Abstractions package's in-process fallback is per-process only; it does not coordinate across nodes. Code paths that assume cross-node mutual exclusion must verify a real provider is registered.
- ABP 10.3 - Default IDistributedCacheSerializer is Utf8JsonDistributedCacheSerializer (UTF-8 JSON); [uncertain] whether it now uses System.Text.Json by default in 10.3 - older releases delegated to Newtonsoft via IJsonSerializer. Confirm the registered IJsonSerializer in your module before assuming a wire format.

## Cross-references

**Phase 1 references:**
- references/framework/architecture-modularity.md — cross-link whenever a user asks where to put Configure<AbpDistributedCacheOptions> / DependsOn(typeof(AbpCachingStackExchangeRedisModule)) / IDistributedLockProvider registration; it all happens in AbpModule.ConfigureServices.
- references/framework/multi-tenancy.md — cross-link whenever cache keys, [IgnoreMultiTenancy], or per-tenant cache isolation come up; the cache key normalizer reads ICurrentTenant.
- references/framework/fundamentals.md — cross-link for ITransientDependency on cache consumers, ISingletonDependency for IDistributedLockProvider, and IServiceCollection.Replace patterns used to swap IDistributedCacheSerializer.
- references/infrastructure/eventing.md — cross-link for IEntityCache invalidation; entity cache subscribes to local/distributed events to evict stale entries.
- references/infrastructure/background-processing.md — cross-link whenever distributed locking is invoked from background jobs / hosted services running on multiple nodes; AbpBackgroundWorkerOptions on multi-node deployments typically pairs with IAbpDistributedLock to run a worker on a single node.

**External docs:**
- [Microsoft.Extensions.Caching.Distributed.IDistributedCache](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.caching.distributed.idistributedcache) — ABP's IDistributedCache<T> wraps the standard ASP.NET Core IDistributedCache; understanding the underlying byte[] contract clarifies how ABP layers serialization, key normalization, and tenancy on top.
- [Distributed caching in ASP.NET Core (Microsoft Docs)](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/distributed) — Background on DistributedCacheEntryOptions (SlidingExpiration, AbsoluteExpiration, AbsoluteExpirationRelativeToNow) - the same options ABP accepts.
- [StackExchange.Redis configuration](https://stackexchange.github.io/StackExchange.Redis/Configuration) — The 'Redis:Configuration' string in appsettings.json is parsed by StackExchange.Redis; this doc lists endpoint syntax, password, ssl, abortConnect, and other options.
- [Microsoft.Extensions.Caching.StackExchangeRedis (RedisCacheOptions)](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.caching.stackexchangeredis.rediscacheoptions) — The options surface exposed by Configure<RedisCacheOptions>; ABP's Volo.Abp.Caching.StackExchangeRedis extends but does not replace it.
- [DistributedLock library (madelson/DistributedLock) on GitHub](https://github.com/madelson/DistributedLock) — ABP's IAbpDistributedLock is a thin wrapper over Medallion.Threading's IDistributedLockProvider; the DistributedLock README documents Redis/SQL/ZooKeeper/Postgres/File providers and their semantics.
- [DistributedLock.Redis NuGet](https://www.nuget.org/packages/DistributedLock.Redis) — Provides RedisDistributedSynchronizationProvider used in the AbpModule wiring example.

## Sources (ABP 10.3)

- https://abp.io/docs/10.3/framework/fundamentals/caching — Core distributed caching system: IDistributedCache<T>, IDistributedCache<T,TKey>, AbpDistributedCacheOptions, key prefix, multi-tenancy, serialization, expiration, GetOrAddAsync, considerUow.
- https://abp.io/docs/10.3/framework/fundamentals/redis-cache — Redis integration via Volo.Abp.Caching.StackExchangeRedis, appsettings.json Redis section, SetMany/GetMany batch performance.
- https://abp.io/docs/10.3/framework/infrastructure/entity-cache — IEntityCache<TEntity,TKey> high-level entity caching with automatic invalidation, EntityCacheWithObjectMapper, AddEntityCache/ReplaceEntityCache extension methods.
- https://abp.io/docs/10.3/framework/infrastructure/distributed-locking — IAbpDistributedLock, AbpDistributedLockOptions, Volo.Abp.DistributedLocking + Volo.Abp.DistributedLocking.Abstractions, Medallion.Threading-based Redis provider via RedisDistributedSynchronizationProvider.

Last verified: 2026-05-10
