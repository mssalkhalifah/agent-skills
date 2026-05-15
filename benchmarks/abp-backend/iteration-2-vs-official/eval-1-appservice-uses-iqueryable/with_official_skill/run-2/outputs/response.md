# Review: `OrderAppService.GetActiveForCustomerAsync`

## What's wrong with the current approach

Your code "works", but it violates several ABP conventions that matter for layering, testability, and provider portability. Cited from the official ABP skills:

1. **Querying logic leaks into the application layer.** The application service is composing `IQueryable<Order>` with `Where`/`OrderByDescending`/`Skip`/`Take`. Per `abp-application-layer`, you should "use custom repositories when you need custom queries; the generic repository is fine for simple CRUD." Anything beyond trivial filtering belongs behind a repository method.
2. **Hidden coupling to EF Core.** `IRepository<T, Guid>.GetQueryableAsync()` returns an EF-shaped `IQueryable` and `AsyncExecuter` is essentially a portability shim around `IQueryable`. Per `abp-dependency-rules` ("Common Violations to Watch"), `IQueryable` in interface breaks abstraction. The application layer should not know whether persistence is EF Core or MongoDB. Encapsulate it in the repository implementation.
3. **The query is not reusable.** Other callers (domain services, other app services, jobs) will repeat the same `Where/OrderBy/Skip/Take` snippet.
4. **Method naming.** `GetActiveForCustomerAsync` repeats the entity context awkwardly. Per `abp-application-layer` ("Don't repeat entity name in method names"), prefer something like `GetActiveListAsync(input)` where `input` carries `CustomerId`. Using a `Get{Entity}ListInput`-style DTO also matches the DTO naming convention table in `abp-application-layer`.
5. **Paging input shape.** ABP ships `PagedAndSortedResultRequestDto` (and `PagedResultRequestDto`) which already define `SkipCount`/`MaxResultCount` with validation. Use it instead of two raw `int` parameters.
6. **No authorization / no `CancellationToken` discipline in the repository.** Custom repository methods should accept and forward `CancellationToken` via `GetCancellationToken(...)` per `abp-ef-core`. The app service itself does not need to pass `CancellationToken` — ABP supplies it via `ICancellationTokenProvider` automatically.
7. **Object mapping.** `ObjectMapper.Map<List<Order>, List<OrderDto>>(orders)` works, but the default mapper in current ABP is Mapperly (compile-time). Per `abp-application-layer`, "prefer the mapping provider already used in the solution". The example below uses a Mapperly partial class; swap to AutoMapper profiles if your solution still uses that.

The fix: define `IOrderRepository` in **Domain**, implement it in **EntityFrameworkCore**, and refactor the app service to call it. The EF Core module registers default repositories for aggregate roots; the example below also explicitly overrides the default `IRepository<Order, Guid>` registration with our custom class so both interfaces resolve to the same instance.

---

## 1. Custom repository interface — Domain project

**File:** `src/MyProject.Domain/Orders/IOrderRepository.cs`

```csharp
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using Volo.Abp.Domain.Repositories;

namespace MyProject.Orders;

public interface IOrderRepository : IRepository<Order, Guid>
{
    Task<List<Order>> GetActiveListByCustomerAsync(
        Guid customerId,
        int skipCount,
        int maxResultCount,
        CancellationToken cancellationToken = default);

    Task<long> GetActiveCountByCustomerAsync(
        Guid customerId,
        CancellationToken cancellationToken = default);
}
```

Notes:
- Interface lives in `*.Domain` (`abp-dependency-rules`: "Repository interface? Domain project").
- It extends `IRepository<Order, Guid>` so all generic CRUD remains available.
- One repository per aggregate root only (`abp-ddd`). `Order` is the aggregate root, so this is correct.
- Returns concrete types (`List<Order>`, `long`), never `IQueryable` (`abp-dependency-rules` "Common Violations").

---

## 2. EF Core implementation — EntityFrameworkCore project

**File:** `src/MyProject.EntityFrameworkCore/Orders/OrderRepository.cs`

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.EntityFrameworkCore;
using MyProject.EntityFrameworkCore;
using Volo.Abp.Domain.Repositories.EntityFrameworkCore;
using Volo.Abp.EntityFrameworkCore;

namespace MyProject.Orders;

public class OrderRepository
    : EfCoreRepository<MyProjectDbContext, Order, Guid>, IOrderRepository
{
    public OrderRepository(IDbContextProvider<MyProjectDbContext> dbContextProvider)
        : base(dbContextProvider)
    {
    }

    public async Task<List<Order>> GetActiveListByCustomerAsync(
        Guid customerId,
        int skipCount,
        int maxResultCount,
        CancellationToken cancellationToken = default)
    {
        var dbSet = await GetDbSetAsync();

        return await dbSet
            .Where(o => o.CustomerId == customerId
                        && o.Status == OrderStatus.Active)
            .OrderByDescending(o => o.CreationTime)
            .Skip(skipCount)
            .Take(maxResultCount)
            .ToListAsync(GetCancellationToken(cancellationToken));
    }

    public async Task<long> GetActiveCountByCustomerAsync(
        Guid customerId,
        CancellationToken cancellationToken = default)
    {
        var dbSet = await GetDbSetAsync();

        return await dbSet
            .Where(o => o.CustomerId == customerId
                        && o.Status == OrderStatus.Active)
            .LongCountAsync(GetCancellationToken(cancellationToken));
    }
}
```

Notes (from `abp-ef-core`):
- Inherits `EfCoreRepository<TDbContext, TEntity, TKey>`.
- Uses `GetDbSetAsync()` inside the repository — never expose `DbContext` outside the EF Core project.
- Forwards `CancellationToken` via `GetCancellationToken(cancellationToken)`.
- Returns `List<Order>`, not `IQueryable`.

Make sure your aggregate root configuration includes an index that backs this query — for example in `MyProjectDbContextModelCreatingExtensions`:

```csharp
builder.Entity<Order>(b =>
{
    b.ToTable(MyProjectConsts.DbTablePrefix + "Orders", MyProjectConsts.DbSchema);
    b.ConfigureByConvention();

    b.Property(x => x.Status).IsRequired();

    // Backs GetActiveListByCustomerAsync filter + ordering
    b.HasIndex(x => new { x.CustomerId, x.Status, x.CreationTime });
});
```

---

## 3. Refactored application service — Application project

**Input DTO (Application.Contracts):** `src/MyProject.Application.Contracts/Orders/GetActiveOrderListInput.cs`

```csharp
using System;
using Volo.Abp.Application.Dtos;

namespace MyProject.Orders;

public class GetActiveOrderListInput : PagedResultRequestDto
{
    public Guid CustomerId { get; set; }
}
```

**Service interface (Application.Contracts):** `src/MyProject.Application.Contracts/Orders/IOrderAppService.cs`

```csharp
using System.Threading.Tasks;
using Volo.Abp.Application.Dtos;
using Volo.Abp.Application.Services;

namespace MyProject.Orders;

public interface IOrderAppService : IApplicationService
{
    Task<PagedResultDto<OrderDto>> GetActiveListAsync(GetActiveOrderListInput input);
}
```

**Mapperly mapper (Application):** `src/MyProject.Application/Orders/OrderMapper.cs`

```csharp
using System.Collections.Generic;
using Riok.Mapperly.Abstractions;

namespace MyProject.Orders;

[Mapper]
public partial class OrderMapper
{
    public partial OrderDto MapToDto(Order order);
    public partial List<OrderDto> MapToDtoList(List<Order> orders);
}
```

**Service implementation (Application):** `src/MyProject.Application/Orders/OrderAppService.cs`

```csharp
using System.Threading.Tasks;
using Microsoft.AspNetCore.Authorization;
using Volo.Abp.Application.Dtos;
using Volo.Abp.Application.Services;

namespace MyProject.Orders;

[Authorize(MyProjectPermissions.Orders.Default)]
public class OrderAppService : ApplicationService, IOrderAppService
{
    private readonly IOrderRepository _orderRepository;
    private readonly OrderMapper _orderMapper;

    public OrderAppService(
        IOrderRepository orderRepository,
        OrderMapper orderMapper)
    {
        _orderRepository = orderRepository;
        _orderMapper = orderMapper;
    }

    public async Task<PagedResultDto<OrderDto>> GetActiveListAsync(
        GetActiveOrderListInput input)
    {
        var totalCount = await _orderRepository.GetActiveCountByCustomerAsync(
            input.CustomerId);

        var orders = await _orderRepository.GetActiveListByCustomerAsync(
            input.CustomerId,
            input.SkipCount,
            input.MaxResultCount);

        return new PagedResultDto<OrderDto>(
            totalCount,
            _orderMapper.MapToDtoList(orders));
    }
}
```

Notes:
- Depends on `IOrderRepository`, not `IRepository<Order, Guid>` — query logic moved out of the app service (`abp-application-layer`).
- No `IQueryable`, no `AsyncExecuter` in the app service — provider portability preserved (`abp-dependency-rules`).
- Method renamed `GetActiveListAsync` (no entity name; matches the auto API controller HTTP-GET prefix rule in `abp-application-layer`).
- Uses `PagedResultRequestDto` for input — built-in `SkipCount`/`MaxResultCount` validation.
- `[Authorize]` placeholder — wire to your real permission constant.
- Uses base-class properties only (no injected `IGuidGenerator`/`IClock`).

---

## 4. Module wiring

### Domain module — nothing to add

`IOrderRepository` is just an interface; ABP's conventional registrar picks up the implementation in the EF Core project automatically.

### EntityFrameworkCore module

**File:** `src/MyProject.EntityFrameworkCore/EntityFrameworkCore/MyProjectEntityFrameworkCoreModule.cs`

```csharp
using Microsoft.Extensions.DependencyInjection;
using MyProject.Orders;
using Volo.Abp.EntityFrameworkCore;
using Volo.Abp.Modularity;

namespace MyProject.EntityFrameworkCore;

[DependsOn(typeof(AbpEntityFrameworkCoreModule))]
public class MyProjectEntityFrameworkCoreModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        context.Services.AddAbpDbContext<MyProjectDbContext>(options =>
        {
            // Aggregate roots only — do NOT pass includeAllEntities: true
            options.AddDefaultRepositories();

            // Replace the default IRepository<Order, Guid> with our custom impl,
            // so callers that inject the generic repo also get our overrides.
            options.AddRepository<Order, OrderRepository>();
        });

        Configure<AbpDbContextOptions>(options =>
        {
            options.UseSqlServer();
        });
    }
}
```

Why `AddRepository<Order, OrderRepository>()`? Per `abp-ef-core`, `AddDefaultRepositories()` already generates a generic repository for every aggregate root. Calling `AddRepository<Order, OrderRepository>()` overrides that registration with `OrderRepository`, so both `IRepository<Order, Guid>` and `IOrderRepository` resolve to the same instance. Without it, the conventional registrar still binds `IOrderRepository` to `OrderRepository`, but consumers that inject the generic interface get the default generic repository, not yours.

### Application module

**File:** `src/MyProject.Application/MyProjectApplicationModule.cs`

```csharp
using Microsoft.Extensions.DependencyInjection;
using MyProject.Orders;
using Volo.Abp.Application;
using Volo.Abp.Modularity;

namespace MyProject;

[DependsOn(typeof(AbpDddApplicationModule))]
public class MyProjectApplicationModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        // Register Mapperly mappers as singletons (compile-time generated, stateless)
        context.Services.AddSingleton<OrderMapper>();
    }
}
```

---

## Summary of fixes vs. original

| Original | Problem | Fix |
|---|---|---|
| `IRepository<Order, Guid>` injected, query composed in app service | Query logic in wrong layer; provider-coupled | `IOrderRepository` in Domain with named methods |
| `GetQueryableAsync()` + `AsyncExecuter` in app service | Leaks `IQueryable` shape | Encapsulated in `EfCoreRepository` |
| `int skipCount, int maxResultCount` raw params | No validation, not a DTO | `GetActiveOrderListInput : PagedResultRequestDto` |
| `GetActiveForCustomerAsync` | Repeats entity name | `GetActiveListAsync(input)` |
| `ObjectMapper.Map<List<Order>, List<OrderDto>>` | Works but not Mapperly-aligned | `OrderMapper` (Mapperly partial) |
| Two queries hand-written in app service | Not reusable | `GetActiveCountByCustomerAsync` + `GetActiveListByCustomerAsync` on repo |
| No `[Authorize]` | Open endpoint | Add permission attribute |
| No supporting index | Slow ordering on `CreationTime DESC` | Composite index `(CustomerId, Status, CreationTime)` |

---

**Skills cited:** `abp-application-layer`, `abp-ddd`, `abp-ef-core`, `abp-dependency-rules` (with supporting context from `abp-core`).
