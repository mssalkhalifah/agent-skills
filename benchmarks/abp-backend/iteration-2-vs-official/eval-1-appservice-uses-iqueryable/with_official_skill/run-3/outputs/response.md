# Review: `OrderAppService.GetActiveForCustomerAsync`

## What's wrong with the current approach

Your code mostly works, but it violates several ABP layering and DDD rules. The biggest issues:

1. **`IQueryable` leaks into the Application layer.** `GetQueryableAsync()` plus `.Where(...)` filtering and ordering in `OrderAppService` puts data-access concerns in the application tier. The Application layer must not contain query expressions over the persistence model -- that is the repository's job. (Per `abp-dependency-rules`: "Data access through repositories, not direct DbContext"; per `abp-application-layer`: "Use custom repositories when you need custom queries".)
2. **Method name / shape does not follow ABP convention.** ABP list-style queries use `GetListAsync(GetXListInput input)` (or with a qualifier, `GetActiveListAsync(...)`), not a positional `(customerId, skipCount, maxResultCount)` signature.
3. **Loose input parameters.** ABP convention is a single `PagedAndSortedResultRequestDto`-derived input DTO (`GetOrderListInput`), not raw `int skipCount, int maxResultCount`. The DTO auto-binds correctly for the auto-generated API controller, brings built-in validation, and gives you a place to add filters later (status, date range, etc.).
4. **Mapping with the runtime `ObjectMapper`.** ABP defaults to **Mapperly** (compile-time) now -- prefer an injected `OrderMapper` partial class. AutoMapper-style `ObjectMapper.Map<...>` still works in solutions that explicitly use AutoMapper, but for new code follow the default. Check existing mapper files in the solution to confirm which library is active.
5. **Returning the full `OrderDto` for a list.** List endpoints should return a leaner `OrderListItemDto`, not the full aggregate DTO.
6. **No authorization attribute.** Endpoints should be protected with `[Authorize(...)]`.
7. **Sorting is hardcoded.** Since the input DTO carries a `Sorting` string, honor it (falling back to `CreationTime desc`) rather than burying the order-by in the service.

The fix is to move the custom query behind a **custom repository interface in the Domain project**, implement it in **EntityFrameworkCore**, register it with `options.AddRepository<Order, OrderRepository>()`, and have the application service simply call the repository.

---

## Corrected code

### 1. Custom repository interface -- Domain project

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
        string sorting = null,
        CancellationToken cancellationToken = default);

    Task<long> GetActiveCountByCustomerAsync(
        Guid customerId,
        CancellationToken cancellationToken = default);
}
```

Notes:
- Lives in `MyProject.Domain` (per `abp-dependency-rules`: "Repository interface? -> Domain project").
- Extends `IRepository<Order, Guid>` so you still get the full generic CRUD surface.
- Returns concrete `List<Order>` / `long` -- never `IQueryable`, never DTOs (`abp-ddd`: "Don't return projection classes"; `abp-dependency-rules`: "IQueryable in interface -> Breaks abstraction").
- `CancellationToken` parameter is fine on custom methods; ABP fills it via `GetCancellationToken(...)`.

### 2. EF Core repository implementation -- EntityFrameworkCore project

**File:** `src/MyProject.EntityFrameworkCore/Orders/OrderRepository.cs`

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Linq.Dynamic.Core;
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
        string sorting = null,
        CancellationToken cancellationToken = default)
    {
        var dbSet = await GetDbSetAsync();

        return await dbSet
            .AsNoTracking()
            .Where(o => o.CustomerId == customerId && o.Status == OrderStatus.Active)
            .OrderBy(sorting.IsNullOrWhiteSpace()
                ? nameof(Order.CreationTime) + " desc"
                : sorting)
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
            .Where(o => o.CustomerId == customerId && o.Status == OrderStatus.Active)
            .LongCountAsync(GetCancellationToken(cancellationToken));
    }
}
```

Notes:
- Inherits `EfCoreRepository<TDbContext, TEntity, TKey>` (per `abp-ef-core`).
- Uses `GetDbSetAsync()` -- never inject the `DbContext` directly outside the EF Core project.
- `AsNoTracking()` for read-only list query.
- `OrderBy(string)` uses **`System.Linq.Dynamic.Core`** (already referenced by ABP's EF Core integration) so the input DTO's `Sorting` string is honored. Defaults to `CreationTime desc`.
- Count is a separate query (you do not want `Count()` after `Skip/Take`).

### 3. Application.Contracts -- DTOs and service interface

**File:** `src/MyProject.Application.Contracts/Orders/GetOrderListInput.cs`

```csharp
using System;
using Volo.Abp.Application.Dtos;

namespace MyProject.Orders;

public class GetOrderListInput : PagedAndSortedResultRequestDto
{
    public Guid CustomerId { get; set; }
}
```

**File:** `src/MyProject.Application.Contracts/Orders/OrderListItemDto.cs`

```csharp
using System;
using Volo.Abp.Application.Dtos;

namespace MyProject.Orders;

public class OrderListItemDto : EntityDto<Guid>
{
    public Guid CustomerId { get; set; }
    public OrderStatus Status { get; set; }
    public DateTime CreationTime { get; set; }
    // Add other display fields as needed
}
```

**File:** `src/MyProject.Application.Contracts/Orders/IOrderAppService.cs`

```csharp
using System.Threading.Tasks;
using Volo.Abp.Application.Dtos;
using Volo.Abp.Application.Services;

namespace MyProject.Orders;

public interface IOrderAppService : IApplicationService
{
    Task<PagedResultDto<OrderListItemDto>> GetActiveListAsync(GetOrderListInput input);
}
```

Notes:
- `PagedAndSortedResultRequestDto` gives `SkipCount`, `MaxResultCount`, and `Sorting` with built-in validation (max page size, default page size).
- Method name is `GetActiveListAsync` -- no entity name repeated (per `abp-application-layer`: "Don't repeat entity name in method names").
- A separate `OrderListItemDto` keeps the list payload lean.

### 4. Mapperly mapper -- Application project

**File:** `src/MyProject.Application/Orders/OrderMapper.cs`

```csharp
using System.Collections.Generic;
using Riok.Mapperly.Abstractions;

namespace MyProject.Orders;

[Mapper]
public partial class OrderMapper
{
    public partial OrderListItemDto MapToListItemDto(Order order);
    public partial List<OrderListItemDto> MapToListItemDtoList(List<Order> orders);
}
```

### 5. Refactored application service -- Application project

**File:** `src/MyProject.Application/Orders/OrderAppService.cs`

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

    public async Task<PagedResultDto<OrderListItemDto>> GetActiveListAsync(
        GetOrderListInput input)
    {
        var totalCount = await _orderRepository
            .GetActiveCountByCustomerAsync(input.CustomerId);

        var orders = await _orderRepository.GetActiveListByCustomerAsync(
            input.CustomerId,
            input.SkipCount,
            input.MaxResultCount,
            input.Sorting);

        return new PagedResultDto<OrderListItemDto>(
            totalCount,
            _orderMapper.MapToListItemDtoList(orders));
    }
}
```

Notes:
- No `IQueryable`, no `AsyncExecuter`, no `GetQueryableAsync()` in the app service.
- Injects the **custom interface** `IOrderRepository`, not `IRepository<Order, Guid>`.
- Uses injected `OrderMapper` (Mapperly) instead of `ObjectMapper`.
- `[Authorize(...)]` guards the endpoint.
- Returns `PagedResultDto<OrderListItemDto>` -- list-item DTO, not the full `OrderDto`.

### 6. Module wiring

#### a. EntityFrameworkCore module -- register the custom repository

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
            // Aggregate roots only -- do NOT pass includeAllEntities: true
            options.AddDefaultRepositories();

            // Binds IOrderRepository (and IRepository<Order, Guid>) to OrderRepository
            options.AddRepository<Order, OrderRepository>();
        });

        Configure<AbpDbContextOptions>(options =>
        {
            options.UseSqlServer();
        });
    }
}
```

**Key point:** `options.AddRepository<Order, OrderRepository>()` is what binds `IOrderRepository` to your implementation. Without it, injecting `IOrderRepository` will fail at runtime even though the interface compiles. `AddDefaultRepositories()` alone only registers the generic `IRepository<Order, Guid>`.

#### b. Application module -- register the Mapperly mapper

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
        context.Services.AddSingleton<OrderMapper>();
    }
}
```

(If your solution still uses AutoMapper, define an `OrderApplicationAutoMapperProfile` instead and register it via ABP's AutoMapper integration. Check existing mapper files in the solution to confirm which library is active.)

#### c. DbContext -- ensure the `DbSet<Order>` and index exist

**File:** `src/MyProject.EntityFrameworkCore/EntityFrameworkCore/MyProjectDbContext.cs`

```csharp
public DbSet<Order> Orders { get; set; }
```

And in your entity configuration (`MyProjectDbContextModelCreatingExtensions`):

```csharp
builder.Entity<Order>(b =>
{
    b.ToTable(MyProjectConsts.DbTablePrefix + "Orders", MyProjectConsts.DbSchema);
    b.ConfigureByConvention(); // Required (audit, soft-delete, multi-tenancy, etc.)

    // Composite index supports the active-orders-by-customer query
    b.HasIndex(x => new { x.CustomerId, x.Status, x.CreationTime });
});
```

Then add a migration:

```bash
cd src/MyProject.EntityFrameworkCore
dotnet ef migrations add Added_Order_CustomerId_Status_CreationTime_Index
```

---

## Summary of changes

| Concern | Before | After |
|---|---|---|
| Query location | `OrderAppService` (Application) | `OrderRepository` (EF Core) |
| Repository | `IRepository<Order, Guid>` | `IOrderRepository : IRepository<Order, Guid>` |
| Input | `(Guid, int, int)` positional | `GetOrderListInput : PagedAndSortedResultRequestDto` |
| Method name | `GetActiveForCustomerAsync` | `GetActiveListAsync` |
| Output | `OrderDto` (full aggregate) | `OrderListItemDto` (lean) |
| Mapping | `ObjectMapper.Map<...>` | Mapperly `OrderMapper` |
| Sorting | Hardcoded `OrderByDescending(CreationTime)` | Dynamic from `input.Sorting`, defaults to `CreationTime desc` |
| Authorization | None | `[Authorize(...)]` |
| DI registration | (implicit generic only) | `options.AddRepository<Order, OrderRepository>()` |
| Index | None | Composite `(CustomerId, Status, CreationTime)` |

---

**Skills used:** `abp-application-layer`, `abp-ddd`, `abp-ef-core`, `abp-dependency-rules`.
