# Active orders for a customer — what's wrong, and the corrected shape

## What's wrong with the original

Your `OrderAppService.GetActiveForCustomerAsync` is a textbook violation of **Rule 2 — Never expose `IQueryable` to the application service layer**.

The offending line is:

```csharp
var queryable = await _orderRepository.GetQueryableAsync();
var query = queryable.Where(...).OrderByDescending(...);
```

`GetQueryableAsync()` is meant **only for use inside a custom repository implementation**, not from an `*AppService`. By calling it in `OrderAppService` you are:

1. Leaking persistence concerns (LINQ-to-EF translation, `AsyncExecuter`, `Skip`/`Take` pagination, ordering) into the application layer, which is supposed to orchestrate, not query.
2. Coupling the app service to EF Core's `IQueryable` provider — the query is not reusable from any other caller (background job, integration service, domain event handler) and can't be unit-tested without the EF Core provider.
3. Bypassing ABP's recommended seam: when a query is too complex for the generic `IRepository<T, TKey>` surface (here, filter + sort + page), the rule says to put it behind a **custom repository interface in the Domain layer with the EF Core implementation in the EntityFrameworkCore project**.

Secondary issues:

- **Rule 4** — there is no `[Authorize]` on the method. Authorization lives on the AppService (every caller routes through it), not only on a controller.
- **Rule 7** — input validation (`MaxResultCount` capping, `SkipCount` non-negative) should flow through ABP's validation pipeline, not loose `int` parameters. Accept a `PagedAndSortedResultRequestDto` instead.
- **Read-only intent** — this is a pure read path. The custom repository implementation should use `IAsyncQueryableExecuter` (provider-neutral async LINQ), and the app service does not need a tracking `IRepository<Order, Guid>` at all — it just needs the custom repository interface.

The fix below moves the query into a custom repository and reduces the app service to orchestration + mapping.

---

## 1. Domain — custom repository interface

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

Why here: the contract is part of the **Domain** layer alongside the `Order` aggregate. Custom repository interfaces always live in `*.Domain` — the application service depends on the contract, the EF Core project supplies the implementation, and the layer boundary holds.

---

## 2. EntityFrameworkCore — implementation

**File:** `src/Acme.Sales.EntityFrameworkCore/Orders/EfCoreOrderRepository.cs`

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Linq.Dynamic.Core;
using System.Threading;
using System.Threading.Tasks;
using Acme.Sales.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;
using Volo.Abp.Domain.Repositories.EntityFrameworkCore;
using Volo.Abp.EntityFrameworkCore;

namespace Acme.Sales.Orders;

public class EfCoreOrderRepository
    : EfCoreRepository<SalesDbContext, Order, Guid>,
      IOrderRepository
{
    public EfCoreOrderRepository(IDbContextProvider<SalesDbContext> dbContextProvider)
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

        var query = dbSet
            .Where(o => o.CustomerId == customerId && o.Status == OrderStatus.Active)
            .OrderBy(!string.IsNullOrWhiteSpace(sorting)
                ? sorting
                : nameof(Order.CreationTime) + " desc")
            .Skip(skipCount)
            .Take(maxResultCount);

        return await query.ToListAsync(GetCancellationToken(cancellationToken));
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

- Inherits `EfCoreRepository<SalesDbContext, Order, Guid>` so it gets the full `IRepository<Order, Guid>` surface for free, then adds the two custom methods.
- Uses `GetDbSetAsync()` / `GetDbContextAsync()` — the only sanctioned places to touch EF Core directly in an ABP solution.
- `OrderBy(sorting)` uses `System.Linq.Dynamic.Core` (already transitively referenced by ABP) so callers can pass `"CreationTime desc"` or `"TotalAmount asc"` without per-field branches; default falls back to `CreationTime desc` as requested.
- `GetCancellationToken(...)` is the ABP helper inherited from `EfCoreRepository` — falls back to `ICancellationTokenProvider` when the caller didn't pass one.
- Materialised return types only — `Task<List<Order>>` and `Task<long>`, **never** `IQueryable<Order>`.

---

## 3. Application — refactored AppService

**File:** `src/Acme.Sales.Application/Orders/OrderAppService.cs`

```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Authorization;
using Volo.Abp.Application.Dtos;
using Volo.Abp.Application.Services;

namespace Acme.Sales.Orders;

[Authorize(SalesPermissions.Orders.Default)]
public class OrderAppService : ApplicationService, IOrderAppService
{
    private readonly IOrderRepository _orderRepository;

    public OrderAppService(IOrderRepository orderRepository)
    {
        _orderRepository = orderRepository;
    }

    public async Task<PagedResultDto<OrderDto>> GetActiveForCustomerAsync(
        Guid customerId,
        GetActiveOrdersInput input)
    {
        var totalCount = await _orderRepository
            .GetActiveCountForCustomerAsync(customerId);

        var orders = await _orderRepository.GetActiveListForCustomerAsync(
            customerId,
            input.SkipCount,
            input.MaxResultCount,
            input.Sorting ?? nameof(Order.CreationTime) + " desc");

        return new PagedResultDto<OrderDto>(
            totalCount,
            ObjectMapper.Map<List<Order>, List<OrderDto>>(orders));
    }
}
```

**File:** `src/Acme.Sales.Application.Contracts/Orders/GetActiveOrdersInput.cs`

```csharp
using Volo.Abp.Application.Dtos;

namespace Acme.Sales.Orders;

public class GetActiveOrdersInput : PagedAndSortedResultRequestDto
{
    // CustomerId is a route parameter, so it stays as a separate method
    // argument in IOrderAppService. Add additional filters here (e.g., date
    // range, status subset) as they emerge.
}
```

**File:** `src/Acme.Sales.Application.Contracts/Orders/IOrderAppService.cs`

```csharp
using System;
using System.Threading.Tasks;
using Volo.Abp.Application.Dtos;
using Volo.Abp.Application.Services;

namespace Acme.Sales.Orders;

public interface IOrderAppService : IApplicationService
{
    Task<PagedResultDto<OrderDto>> GetActiveForCustomerAsync(
        Guid customerId,
        GetActiveOrdersInput input);
}
```

What changed vs. your version:

- The injected dependency is `IOrderRepository`, **not** `IRepository<Order, Guid>` — Rule 2 wants the narrowest interface that does the job; here the custom repo provides the exact query method we need.
- No `GetQueryableAsync()`, no `AsyncExecuter`, no `Skip`/`Take` in the app service.
- `[Authorize(SalesPermissions.Orders.Default)]` declares the authorization boundary on the application service per Rule 4 — every caller (HTTP, background job, integration handler) gets the check, and the permission constant lives in `*.Application.Contracts` next to `IOrderAppService`.
- `PagedAndSortedResultRequestDto` brings ABP's standard `MaxResultCount` validation (default cap) into the request pipeline (Rule 7 — input validation belongs in the validation pipeline, not in throws inside the service body).
- Returns `PagedResultDto<OrderDto>` — a materialised type, not `IQueryable`.

---

## 4. Module wiring — register the custom repository

**File:** `src/Acme.Sales.EntityFrameworkCore/EntityFrameworkCore/SalesEntityFrameworkCoreModule.cs`

```csharp
using Acme.Sales.Orders;
using Microsoft.Extensions.DependencyInjection;
using Volo.Abp.Data;
using Volo.Abp.EntityFrameworkCore;
using Volo.Abp.Modularity;

namespace Acme.Sales.EntityFrameworkCore;

[DependsOn(
    typeof(AbpEntityFrameworkCoreModule),
    typeof(AbpEntityFrameworkCoreSqlServerModule),   // swap per DBMS (Npgsql/MySQL/etc.)
    typeof(SalesDomainModule)
)]
public class SalesEntityFrameworkCoreModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        context.Services.AddAbpDbContext<SalesDbContext>(options =>
        {
            options.AddDefaultRepositories(includeAllEntities: true);

            // Load-bearing: overrides the default
            // IRepository<Order, Guid> -> EfCoreRepository<...> binding
            // with our custom EfCoreOrderRepository, so injecting either
            // IOrderRepository OR IRepository<Order, Guid> resolves to it.
            options.AddRepository<Order, EfCoreOrderRepository>();
        });

        Configure<AbpDbContextOptions>(options =>
        {
            options.UseSqlServer();
        });
    }
}
```

Why `AddRepository<Order, EfCoreOrderRepository>()` matters (from `framework/data-ef-core.md` Common Mistakes): without it, `AddDefaultRepositories` still registers `IRepository<Order, Guid>` against the default `EfCoreRepository`, and **only** `IOrderRepository` resolves to your custom class. That split means generic code that injects `IRepository<Order, Guid>` bypasses your custom methods. The `AddRepository` override fuses both bindings onto the same instance.

---

## Where each file lives at a glance (Layered template)

| File | Project / Folder |
|---|---|
| `IOrderRepository.cs` | `src/Acme.Sales.Domain/Orders/` |
| `EfCoreOrderRepository.cs` | `src/Acme.Sales.EntityFrameworkCore/Orders/` |
| `OrderAppService.cs` | `src/Acme.Sales.Application/Orders/` |
| `IOrderAppService.cs`, `GetActiveOrdersInput.cs`, `OrderDto.cs` | `src/Acme.Sales.Application.Contracts/Orders/` |
| `SalesEntityFrameworkCoreModule.cs` (the `AddRepository` wiring) | `src/Acme.Sales.EntityFrameworkCore/EntityFrameworkCore/` |

(If this is a **Single-Layer** solution, collapse to `Entities/`, `Data/`, `Services/`, `Services/Dtos/` inside the single `*.csproj` and keep the same module-class wiring. If **Microservice**, the same five files live inside the service's solution folders. The rule and the call-graph are identical across templates.)

---

## Summary of rules applied

- **Rule 2** — `GetQueryableAsync()` removed from the app service; the query is now encapsulated in `IOrderRepository` / `EfCoreOrderRepository`, returning materialised `List<Order>` and `long`.
- **Rule 4** — `[Authorize(SalesPermissions.Orders.Default)]` declared on the application service (the authorization boundary every caller routes through), with the permission constant living in `*.Application.Contracts`.
- **Rule 7** — Input validation handled by `PagedAndSortedResultRequestDto` through ABP's validation pipeline, not by imperative throws in the service body.
- Custom repository registration via `options.AddRepository<Order, EfCoreOrderRepository>()` in the EntityFrameworkCore module fuses the `IRepository<Order, Guid>` and `IOrderRepository` resolutions onto the same instance.
