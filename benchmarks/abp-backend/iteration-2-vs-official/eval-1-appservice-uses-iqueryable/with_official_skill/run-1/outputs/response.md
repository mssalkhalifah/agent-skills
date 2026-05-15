## Review: `GetActiveForCustomerAsync` — what's wrong, and how it should look

**Official skills used:** `abp-application-layer`, `abp-ddd`, `abp-ef-core`, `abp-dependency-rules`.

### What's wrong with the current approach

Your `OrderAppService` is composing a custom `IQueryable` inside the application layer. According to ABP's dependency and DDD rules, that pushes data-access concerns up into a layer that should only orchestrate use cases.

Specific issues, mapped to the skills:

1. **Custom query in the application layer (abp-dependency-rules, abp-ddd).** Filtering by `CustomerId` + `Status` and ordering by `CreationTime` is a *custom query*. The DDD skill is explicit: "Define custom repository only when custom queries are needed" — and this is exactly that case. The query belongs behind an `IOrderRepository` method whose interface lives in `Domain` and whose implementation lives in `EntityFrameworkCore`. Calling `GetQueryableAsync()` from the app service leaks EF Core composition (LINQ provider semantics, tracking behavior) into a layer that, per the dependency table, must not know about EF Core.
2. **Application-layer method name repeats the entity name (abp-application-layer).** The skill says "Don't repeat entity name in method names" and uses `GetListAsync(GetXxxListInput input)` with a typed input DTO that carries paging + filters. That also lets ABP's auto API controllers infer the route/verb correctly.
3. **Raw `skipCount` / `maxResultCount` instead of `PagedAndSortedResultRequestDto` (abp-application-layer).** ABP has a standard paged-input DTO; using it gives you `MaxResultCount` clamping, a `Sorting` field, and consistent client proxies.
4. **`ObjectMapper.Map<List<Order>, List<OrderDto>>` (abp-application-layer).** The current guidance is to use **Mapperly** (default) via an injected partial mapper class. `ObjectMapper` (AutoMapper-style) still works but isn't the default — check what the solution uses; the example below shows Mapperly.
5. **No `[Authorize]` (abp-application-layer).** A "list orders for a customer" endpoint should be permission-checked.
6. **`o.Status == OrderStatus.Active` as a literal in the app service.** "Is this order active?" is a domain concept. If `Active` is ever more than a single enum compare, it should be a method/property on the entity or a `Specification<Order>`, not a `.Where(...)` literal in the application layer.

The good parts: you correctly used `GetQueryableAsync()` (not a sync `GetQueryable`), `AsyncExecuter` (not `await query.ToListAsync()` directly — important so the code stays provider-agnostic), and a `PagedResultDto<OrderDto>` return type. Those stay.

---

### Corrected layout

Below is the full refactor across the four layers. File paths assume the standard ABP **layered application template**.

#### 1. Custom repository interface — Domain layer

**File:** `src/Acme.MyProject.Domain/Orders/IOrderRepository.cs`

```csharp
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using Volo.Abp.Domain.Repositories;

namespace Acme.MyProject.Orders;

public interface IOrderRepository : IRepository<Order, Guid>
{
    Task<List<Order>> GetActiveListForCustomerAsync(
        Guid customerId,
        int skipCount,
        int maxResultCount,
        string sorting = null,
        CancellationToken cancellationToken = default);

    Task<long> GetActiveCountForCustomerAsync(
        Guid customerId,
        CancellationToken cancellationToken = default);
}
```

Notes:
- Interface lives in `Domain` per the dependency table (`Domain` owns repository interfaces) — never in `Application` or `EntityFrameworkCore`.
- No `IQueryable` in the signature. The skill's "Common Violations" table explicitly lists `IQueryable in interface` as a violation that "breaks abstraction".
- Returns the domain entity `Order`, not a DTO. Per abp-ddd: "Don't return projection classes" from repositories.
- `sorting` is a string in ABP's idiom (Dynamic LINQ), with a sensible default applied inside the implementation.

#### 2. EF Core implementation — EntityFrameworkCore layer

**File:** `src/Acme.MyProject.EntityFrameworkCore/Orders/EfCoreOrderRepository.cs`

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Linq.Dynamic.Core;
using System.Threading;
using System.Threading.Tasks;
using Acme.MyProject.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;
using Volo.Abp.Domain.Repositories.EntityFrameworkCore;
using Volo.Abp.EntityFrameworkCore;

namespace Acme.MyProject.Orders;

public class EfCoreOrderRepository
    : EfCoreRepository<MyProjectDbContext, Order, Guid>, IOrderRepository
{
    public EfCoreOrderRepository(IDbContextProvider<MyProjectDbContext> dbContextProvider)
        : base(dbContextProvider)
    {
    }

    public async Task<List<Order>> GetActiveListForCustomerAsync(
        Guid customerId,
        int skipCount,
        int maxResultCount,
        string sorting = null,
        CancellationToken cancellationToken = default)
    {
        var dbSet = await GetDbSetAsync();

        return await dbSet
            .Where(o => o.CustomerId == customerId && o.Status == OrderStatus.Active)
            .OrderBy(sorting.IsNullOrWhiteSpace()
                ? nameof(Order.CreationTime) + " desc"
                : sorting)
            .Skip(skipCount)
            .Take(maxResultCount)
            .ToListAsync(GetCancellationToken(cancellationToken));
    }

    public async Task<long> GetActiveCountForCustomerAsync(
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
- Inherits `EfCoreRepository<TDbContext, TEntity, TKey>` per the abp-ef-core skill.
- Uses `GetDbSetAsync()` and `GetCancellationToken(cancellationToken)` — both straight from the skill's repository example.
- Uses Dynamic LINQ `.OrderBy(sorting)` (`System.Linq.Dynamic.Core`, already referenced by ABP's EF Core layer) so callers can pass `"CreationTime desc"`, `"OrderNumber asc"`, etc. Default falls back to `CreationTime desc`, matching the original intent.
- Returns the domain entity. No mapping here.

#### 3. Refactored Application Service

**Input DTO file:** `src/Acme.MyProject.Application.Contracts/Orders/GetActiveOrderListInput.cs`

```csharp
using System;
using Volo.Abp.Application.Dtos;

namespace Acme.MyProject.Orders;

public class GetActiveOrderListInput : PagedAndSortedResultRequestDto
{
    public Guid CustomerId { get; set; }
}
```

**Interface file:** `src/Acme.MyProject.Application.Contracts/Orders/IOrderAppService.cs`

```csharp
using System.Threading.Tasks;
using Volo.Abp.Application.Dtos;
using Volo.Abp.Application.Services;

namespace Acme.MyProject.Orders;

public interface IOrderAppService : IApplicationService
{
    Task<PagedResultDto<OrderDto>> GetActiveListAsync(GetActiveOrderListInput input);
}
```

**Implementation file:** `src/Acme.MyProject.Application/Orders/OrderAppService.cs`

```csharp
using System.Threading.Tasks;
using Microsoft.AspNetCore.Authorization;
using Acme.MyProject.Permissions;
using Volo.Abp.Application.Dtos;
using Volo.Abp.Application.Services;

namespace Acme.MyProject.Orders;

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

    public async Task<PagedResultDto<OrderDto>> GetActiveListAsync(GetActiveOrderListInput input)
    {
        var totalCount = await _orderRepository.GetActiveCountForCustomerAsync(input.CustomerId);

        var orders = await _orderRepository.GetActiveListForCustomerAsync(
            input.CustomerId,
            input.SkipCount,
            input.MaxResultCount,
            input.Sorting);

        return new PagedResultDto<OrderDto>(
            totalCount,
            _orderMapper.MapToDtoList(orders));
    }
}
```

**Mapperly mapper file:** `src/Acme.MyProject.Application/Orders/OrderMapper.cs`

```csharp
using System.Collections.Generic;
using Riok.Mapperly.Abstractions;

namespace Acme.MyProject.Orders;

[Mapper]
public partial class OrderMapper
{
    public partial OrderDto MapToDto(Order order);
    public partial List<OrderDto> MapToDtoList(List<Order> orders);
}
```

Notes:
- Method renamed `GetActiveListAsync` — drops the redundant `Order` token and follows the `GetList...` convention.
- Input is a single DTO inheriting `PagedAndSortedResultRequestDto`, which gives you `SkipCount`, `MaxResultCount` (auto-clamped to `MaxMaxResultCount`), and `Sorting` for free.
- No `IQueryable`, no `AsyncExecuter`, no `GetQueryableAsync` in the app service — all of that moved into the repository.
- `[Authorize]` added (define the permission in `MyProjectPermissionDefinitionProvider` per the abp-authorization guidance).
- Mapping is done by an injected Mapperly partial class, per current abp-application-layer guidance ("the default mapping library is Mapperly"). If your solution is still on AutoMapper, swap to `ObjectMapper.Map<List<Order>, List<OrderDto>>(orders)` and an AutoMapper `Profile` instead — but the rest of the structure stays identical.

#### 4. Module wiring — EntityFrameworkCore project

ABP auto-registers any class that inherits `EfCoreRepository<,,>` and implements a custom interface derived from `IRepository<,>`, so the custom repo is picked up by convention once `AddDefaultRepositories()` is called. **You do not need to register `EfCoreOrderRepository` manually.** Per the abp-ef-core skill's "Repositories for Aggregate Roots Only" guidance, keep `AddDefaultRepositories()` without `includeAllEntities: true`.

**File:** `src/Acme.MyProject.EntityFrameworkCore/EntityFrameworkCore/MyProjectEntityFrameworkCoreModule.cs`

```csharp
using Microsoft.Extensions.DependencyInjection;
using Volo.Abp.EntityFrameworkCore;
using Volo.Abp.Modularity;

namespace Acme.MyProject.EntityFrameworkCore;

[DependsOn(typeof(AbpEntityFrameworkCoreModule))]
public class MyProjectEntityFrameworkCoreModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        context.Services.AddAbpDbContext<MyProjectDbContext>(options =>
        {
            // Default repos for aggregate roots (Order is an aggregate root, so
            // IRepository<Order, Guid> is registered automatically). Our custom
            // EfCoreOrderRepository : IOrderRepository is auto-discovered and
            // registered as the implementation of IOrderRepository as well.
            options.AddDefaultRepositories();

            // Avoid options.AddDefaultRepositories(includeAllEntities: true) —
            // it would create repositories for child entities (e.g. OrderLine),
            // breaking aggregate-root encapsulation per abp-ef-core.
        });

        Configure<AbpDbContextOptions>(options =>
        {
            options.UseSqlServer();
        });
    }
}
```

If you ever need to register the custom repository **explicitly** (e.g. to override the default for `IRepository<Order, Guid>` so both interfaces resolve to the same instance), the form is:

```csharp
options.AddDefaultRepositories();
options.AddRepository<Order, EfCoreOrderRepository>();
```

#### 5. Mapperly mapper wiring — Application module

Per the abp-application-layer skill: "Register in module: `context.Services.AddSingleton<BookMapper>();`".

**File:** `src/Acme.MyProject.Application/MyProjectApplicationModule.cs`

```csharp
public override void ConfigureServices(ServiceConfigurationContext context)
{
    context.Services.AddSingleton<OrderMapper>();
    // ... other registrations
}
```

---

### Summary of the changes

| Concern | Before | After |
|---|---|---|
| Where the query lives | `OrderAppService` (Application) | `EfCoreOrderRepository` (EntityFrameworkCore), behind `IOrderRepository` (Domain) |
| `IQueryable` exposed | Yes, via `GetQueryableAsync()` in the app service | No — repository returns `List<Order>` / `long` |
| Method signature | `(Guid customerId, int skipCount, int maxResultCount)` | `(GetActiveOrderListInput input)` inheriting `PagedAndSortedResultRequestDto` |
| Method name | `GetActiveForCustomerAsync` | `GetActiveListAsync` (no entity name repetition, `List` convention) |
| Sorting | Hard-coded `CreationTime desc` | Configurable via `input.Sorting`, default `CreationTime desc` |
| Mapping | `ObjectMapper.Map<...>` | Injected `OrderMapper` (Mapperly partial class) |
| Authorization | Missing | `[Authorize(MyProjectPermissions.Orders.Default)]` |
| Repository DI | N/A | Auto-registered by `AddDefaultRepositories()` (custom impl discovered by convention) |
