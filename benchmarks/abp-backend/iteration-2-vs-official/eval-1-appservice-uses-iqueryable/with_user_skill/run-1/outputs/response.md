# Listing active orders for a customer — what's wrong, and the corrected layout

## What's wrong with your approach

Your `OrderAppService.GetActiveForCustomerAsync` violates **Rule 2 — Never expose `IQueryable` to the application service layer**. Specifically:

- `_orderRepository.GetQueryableAsync()` is called **inside** `OrderAppService`. That's exactly the line Rule 2 forbids. `GetQueryableAsync()` is a hatch intended for use *inside a custom repository implementation*, not from the application layer.
- `AsyncExecuter.CountAsync(query)` and `AsyncExecuter.ToListAsync(...)` are downstream symptoms — they only exist in the AppService because an `IQueryable` is in scope to operate on. Once the query lives in a custom repository, the AppService gets back a fully materialised `List<Order>` plus a `totalCount`; no async LINQ executer needed in the application layer.
- The `Where` + `OrderByDescending` + `Skip`/`Take` composition encodes a domain query (the definition of "active orders for a customer, newest first, paged") inside `OrderAppService`. That belongs behind a domain repository interface so the next caller (background job, integration service, another app service) reuses the same query rather than reimplementing it.
- Bonus: `int skipCount, int maxResultCount` should be a `PagedAndSortedResultRequestDto` (or at minimum `PagedResultRequestDto`) so ABP's validation pipeline (Rule 7) enforces `MaxResultCount` automatically. I'll bake that into the corrected version.

The fix is the canonical ABP custom-repository pattern: declare `IOrderRepository : IRepository<Order, Guid>` in the **Domain** project, implement it with `EfCoreRepository<TDbContext, Order, Guid>` in the **EntityFrameworkCore** project, register it with `AddRepository<Order, OrderRepository>()` in that module's `ConfigureServices`, and have `OrderAppService` depend on `IOrderRepository` instead of touching `IQueryable`.

I'm assuming the **Layered template** (the project layout the rule was written against). The same pattern works in Single-Layer (folders instead of projects) and Microservice (per-service solution); just relocate the files accordingly.

---

## 1. Domain project — repository contract

**File:** `src/Acme.Sales.Domain/Orders/IOrderRepository.cs`

```csharp
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using Volo.Abp.Domain.Repositories;

namespace Acme.Sales.Orders;

public interface IOrderRepository : IRepository<Order, Guid>
{
    Task<List<Order>> GetActiveForCustomerAsync(
        Guid customerId,
        int skipCount,
        int maxResultCount,
        CancellationToken cancellationToken = default);

    Task<long> GetActiveForCustomerCountAsync(
        Guid customerId,
        CancellationToken cancellationToken = default);
}
```

**Why here:** Rule 2 says "define a custom `IXyzRepository : IRepository<X, TKey>` in the feature folder." Repository *interfaces* are a domain-layer concept (the domain expresses what data it needs); implementations are infrastructure. Putting the interface in `*.Domain` lets the application layer and domain services depend on it without taking an EF Core reference.

---

## 2. EntityFrameworkCore project — repository implementation

**File:** `src/Acme.Sales.EntityFrameworkCore/Orders/OrderRepository.cs`

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using Acme.Sales.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;
using Volo.Abp.Domain.Repositories.EntityFrameworkCore;
using Volo.Abp.EntityFrameworkCore;

namespace Acme.Sales.Orders;

public class OrderRepository
    : EfCoreRepository<SalesDbContext, Order, Guid>, IOrderRepository
{
    public OrderRepository(IDbContextProvider<SalesDbContext> dbContextProvider)
        : base(dbContextProvider) { }

    public async Task<List<Order>> GetActiveForCustomerAsync(
        Guid customerId,
        int skipCount,
        int maxResultCount,
        CancellationToken cancellationToken = default)
    {
        var dbSet = await GetDbSetAsync();
        return await dbSet
            .Where(o => o.CustomerId == customerId && o.Status == OrderStatus.Active)
            .OrderByDescending(o => o.CreationTime)
            .Skip(skipCount)
            .Take(maxResultCount)
            .ToListAsync(GetCancellationToken(cancellationToken));
    }

    public async Task<long> GetActiveForCustomerCountAsync(
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

**Key points:**

- Inherits `EfCoreRepository<TDbContext, TEntity, TKey>` and implements `IOrderRepository`. The base class wires `IDbContextProvider<SalesDbContext>` through the constructor.
- `GetDbSetAsync()` is the *intra-repository* hatch into EF Core — fine here, forbidden in `OrderAppService`.
- `GetCancellationToken(token)` falls back to ABP's `ICancellationTokenProvider` when the caller didn't pass one (request abort, host shutdown, etc.).
- Count uses `LongCountAsync` because `PagedResultDto<T>` takes `long` for `TotalCount`.
- The query (active-for-customer, newest first, paged) is now encapsulated. Reuse from any caller is one method call.

---

## 3. EntityFrameworkCore module — register the custom repository

**File:** `src/Acme.Sales.EntityFrameworkCore/EntityFrameworkCore/SalesEntityFrameworkCoreModule.cs`

```csharp
using Acme.Sales.Orders;
using Microsoft.Extensions.DependencyInjection;
using Volo.Abp.Domain;
using Volo.Abp.EntityFrameworkCore;
using Volo.Abp.Modularity;

namespace Acme.Sales.EntityFrameworkCore;

[DependsOn(
    typeof(AbpEntityFrameworkCoreModule),
    typeof(AbpEntityFrameworkCoreSqlServerModule), // or PostgreSql / MySQL / Sqlite / Oracle
    typeof(SalesDomainModule)
)]
public class SalesEntityFrameworkCoreModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        context.Services.AddAbpDbContext<SalesDbContext>(options =>
        {
            options.AddDefaultRepositories(includeAllEntities: true);
            options.AddRepository<Order, OrderRepository>();    // override the default IRepository<Order, Guid> binding
        });

        Configure<AbpDbContextOptions>(options =>
        {
            options.UseSqlServer();
        });
    }
}
```

**Why `AddRepository<Order, OrderRepository>()` matters:** `AddDefaultRepositories` registers the generic `IRepository<Order, Guid>` → default `EfCoreRepository<...>` binding. Without the explicit `AddRepository` override, a class that injects `IRepository<Order, Guid>` would still resolve to the default repo and silently bypass your custom methods (and any future cross-cutting behaviour you add to `OrderRepository`). `AddRepository` makes the binding point at `OrderRepository` for *both* `IOrderRepository` and `IRepository<Order, Guid>` injections.

---

## 4. Application.Contracts — paged request DTO

**File:** `src/Acme.Sales.Application.Contracts/Orders/GetActiveOrdersInput.cs`

```csharp
using System;
using Volo.Abp.Application.Dtos;

namespace Acme.Sales.Orders;

public class GetActiveOrdersInput : PagedResultRequestDto
{
    public Guid CustomerId { get; set; }
}
```

`PagedResultRequestDto` carries `SkipCount` and `MaxResultCount` with `[Range]` attributes so ABP's validation pipeline rejects bad paging values before the AppService runs (Rule 7 — input validation through the pipeline, not imperative throws). For sort flexibility later, swap to `PagedAndSortedResultRequestDto`.

---

## 5. Application project — refactored AppService

**File:** `src/Acme.Sales.Application/Orders/OrderAppService.cs`

```csharp
using System.Collections.Generic;
using System.Threading.Tasks;
using Acme.Sales.Permissions;
using Microsoft.AspNetCore.Authorization;
using Volo.Abp.Application.Dtos;
using Volo.Abp.Application.Services;

namespace Acme.Sales.Orders;

public class OrderAppService : ApplicationService, IOrderAppService
{
    private readonly IOrderRepository _orderRepository;

    public OrderAppService(IOrderRepository orderRepository)
    {
        _orderRepository = orderRepository;
    }

    [Authorize(SalesPermissions.Orders.Default)]
    public async Task<PagedResultDto<OrderDto>> GetActiveForCustomerAsync(
        GetActiveOrdersInput input)
    {
        var totalCount = await _orderRepository.GetActiveForCustomerCountAsync(
            input.CustomerId);

        var orders = await _orderRepository.GetActiveForCustomerAsync(
            input.CustomerId,
            input.SkipCount,
            input.MaxResultCount);

        return new PagedResultDto<OrderDto>(
            totalCount,
            ObjectMapper.Map<List<Order>, List<OrderDto>>(orders));
    }
}
```

What changed versus your original:

- Depends on `IOrderRepository` (custom) instead of `IRepository<Order, Guid>` (generic). The query is now a named method call, not a LINQ pipeline owned by the AppService.
- No `GetQueryableAsync()`, no `AsyncExecuter`, no `IQueryable` anywhere in the application layer — Rule 2 satisfied.
- Input is a `PagedResultRequestDto`-derived DTO so the validation pipeline enforces paging bounds (Rule 7).
- `[Authorize(SalesPermissions.Orders.Default)]` lives on the AppService method (Rule 4 — AppService is the canonical authorization boundary; every caller, HTTP or otherwise, routes through it).

---

## 6. (Optional) Controller — Rule 4 explicit-controller pattern

If you're following the recommended explicit-controller pattern, the HTTP adapter is two lines of delegation with no `[Authorize]` (it's already on the AppService method):

**File:** `src/Acme.Sales.HttpApi/Controllers/OrderController.cs`

```csharp
using System.Threading.Tasks;
using Acme.Sales.Orders;
using Microsoft.AspNetCore.Mvc;
using Volo.Abp.Application.Dtos;

namespace Acme.Sales.Controllers;

[Route("api/sales/orders")]
public sealed class OrderController : SalesController, IOrderAppService
{
    private readonly IOrderAppService _orderAppService;

    public OrderController(IOrderAppService orderAppService)
    {
        _orderAppService = orderAppService;
    }

    [HttpGet("active")]
    public Task<PagedResultDto<OrderDto>> GetActiveForCustomerAsync(
        [FromQuery] GetActiveOrdersInput input)
        => _orderAppService.GetActiveForCustomerAsync(input);
}
```

Only `IOrderAppService` is injected (Rule 4 — no repositories, no managers, no mappers in the controller). Skip this entirely if your project sticks with Auto API Controllers — the AppService is sufficient on its own.

---

## Summary of rules applied

- **Rule 2** — Custom `IOrderRepository` in Domain, `EfCoreRepository`-derived implementation in EntityFrameworkCore. No `IQueryable` crosses into the application layer; `GetDbSetAsync()` is only used inside the repository.
- **Rule 4** — `[Authorize]` lives on the AppService method, not the controller; controller (if used) injects only `IOrderAppService`.
- **Rule 7** — Paging bounds enforced declaratively via `PagedResultRequestDto`'s `[Range]` attributes rather than imperative throws in the AppService body.

And the registration line that does the load-bearing work: `options.AddRepository<Order, OrderRepository>();` in `SalesEntityFrameworkCoreModule.ConfigureServices` — without it, the custom repository's CRUD methods quietly route back to the default repo for any `IRepository<Order, Guid>` injection.
