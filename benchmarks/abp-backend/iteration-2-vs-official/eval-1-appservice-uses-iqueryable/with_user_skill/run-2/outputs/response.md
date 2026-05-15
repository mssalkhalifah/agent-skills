# Review: `OrderAppService.GetActiveForCustomerAsync`

## What's wrong

Your code violates **Rule 2 - Never expose `IQueryable` to the application service layer** from the ABP backend skill:

> Application services must call repository methods that return materialised results (`Task<TEntity>`, `Task<List<TEntity>>`, `Task<PagedResultDto<...>>`). Encapsulate queries inside a custom repository - do **not** call `_repository.GetQueryableAsync()` from inside `*AppService.cs`. If a query is too complex for the generic repository, define a custom `IXyzRepository : IRepository<X, TKey>` in the feature folder and put its EF Core implementation alongside.

Concrete issues with the snippet:

1. `await _orderRepository.GetQueryableAsync()` plus `.Where(...).OrderByDescending(...)` lives inside `OrderAppService.cs`. The query (LINQ predicate, sort order, paging composition) is a persistence concern - it belongs behind a custom repository in the Domain/EntityFrameworkCore layers, not in Application.
2. `AsyncExecuter.CountAsync` / `AsyncExecuter.ToListAsync` against an `IQueryable` are the EF-Core-aware async drivers - they are intended to be used **inside a custom repository implementation** (or, at the boundary, via `IAsyncQueryableExecuter` to stay provider-neutral). Using them from an `*AppService` is the smell Rule 2 is meant to catch.
3. Read-only query - the AppService is fetching for display, but it injects the tracking `IRepository<Order, Guid>`. A read path should prefer `IReadOnlyRepository<Order, Guid>` (no-tracking by default, signals intent) - see the repository-interface table in the skill.
4. No `CancellationToken` is plumbed through. Long-running list queries should accept and forward one.
5. Paging contract is hand-rolled (`int skipCount, int maxResultCount`). ABP ships `PagedAndSortedResultRequestDto` / `PagedResultRequestDto` for exactly this; use them so the contract is uniform across endpoints.

The fix is the standard ABP pattern: custom repository interface in **Domain**, EF Core implementation in **EntityFrameworkCore**, refactored AppService delegates to it, and the EF Core module registers the binding via `AddRepository<TEntity, TRepository>()`.

Files below assume a Layered solution named `Acme.Sales` (`src/Acme.Sales.Domain`, `src/Acme.Sales.Application`, `src/Acme.Sales.EntityFrameworkCore`). Adjust namespaces to match your project.

---

## 1. Custom repository interface - Domain project

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
        CancellationToken cancellationToken = default);

    Task<long> GetActiveCountForCustomerAsync(
        Guid customerId,
        CancellationToken cancellationToken = default);
}
```

Notes:
- Interface lives in the **Domain** project alongside the `Order` aggregate. It extends `IRepository<Order, Guid>` so all the generic CRUD/query methods stay available.
- Returns **materialised** types (`List<Order>`, `long`) - never `IQueryable<Order>`. The contract honours Rule 2 by construction.
- One file per type (Rule 1) - keep this in its own `.cs`.

---

## 2. EF Core implementation - EntityFrameworkCore project

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
        : base(dbContextProvider)
    {
    }

    public async Task<List<Order>> GetActiveListForCustomerAsync(
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
- Inherits `EfCoreRepository<TDbContext, TEntity, TKey>` and takes `IDbContextProvider<SalesDbContext>` - the canonical custom-repository shape from `references/framework/data-ef-core.md`.
- `GetDbSetAsync()` drops down to EF Core *inside* the repository - that's exactly where `IQueryable` is allowed to live.
- `GetCancellationToken(cancellationToken)` falls back to `ICancellationTokenProvider` when the caller didn't supply one.
- Uses `LongCountAsync` because pageable result counts can exceed `int.MaxValue`; the AppService casts down only at the DTO boundary.

---

## 3. Refactored Application Service

**File:** `src/Acme.Sales.Application/Orders/OrderAppService.cs`

```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
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

    public async Task<PagedResultDto<OrderDto>> GetActiveForCustomerAsync(
        Guid customerId,
        PagedAndSortedResultRequestDto input)
    {
        var totalCount = await _orderRepository.GetActiveCountForCustomerAsync(customerId);

        var orders = await _orderRepository.GetActiveListForCustomerAsync(
            customerId,
            input.SkipCount,
            input.MaxResultCount);

        return new PagedResultDto<OrderDto>(
            totalCount,
            ObjectMapper.Map<List<Order>, List<OrderDto>>(orders));
    }
}
```

Notes:
- The AppService no longer touches `IQueryable`, `GetQueryableAsync`, or `AsyncExecuter`. It calls two materialising repository methods and maps to DTOs - that's the entire body. Rule 2 satisfied.
- Inject `IOrderRepository` directly. ABP still resolves the default `IRepository<Order, Guid>` separately if some other service needs it; the registration step (next) wires both bindings.
- Uses ABP's `PagedAndSortedResultRequestDto` for the input contract - uniform paging shape across the API. (If you don't need server-driven sort, swap to `PagedResultRequestDto`.)
- `ApplicationService` already provides `ObjectMapper`, `CurrentUser`, `GuidGenerator` - no need to inject them.

**Interface (Application.Contracts project), for completeness:**

`src/Acme.Sales.Application.Contracts/Orders/IOrderAppService.cs`

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
        PagedAndSortedResultRequestDto input);
}
```

---

## 4. Module wiring - register the custom repository

**File:** `src/Acme.Sales.EntityFrameworkCore/EntityFrameworkCore/SalesEntityFrameworkCoreModule.cs`

```csharp
using Acme.Sales.Orders;
using Microsoft.Extensions.DependencyInjection;
using Volo.Abp.EntityFrameworkCore;
using Volo.Abp.Modularity;

namespace Acme.Sales.EntityFrameworkCore;

[DependsOn(
    typeof(AbpEntityFrameworkCoreModule),
    typeof(AbpEntityFrameworkCoreSqlServerModule),   // swap per DBMS
    typeof(SalesDomainModule)
)]
public class SalesEntityFrameworkCoreModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        context.Services.AddAbpDbContext<SalesDbContext>(options =>
        {
            options.AddDefaultRepositories(includeAllEntities: true);
            options.AddRepository<Order, OrderRepository>();   // <-- the load-bearing line
        });

        Configure<AbpDbContextOptions>(options =>
        {
            options.UseSqlServer();
        });
    }
}
```

Why `AddRepository<Order, OrderRepository>()` matters: `AddDefaultRepositories` already registers the generic `IRepository<Order, Guid>`. Without the `AddRepository` override, code that injects `IOrderRepository` still resolves (because `OrderRepository : IOrderRepository` is picked up by convention), **but** any caller that injects the generic `IRepository<Order, Guid>` would get the default `EfCoreRepository`, not your subclass - so any custom logic you later add to `OrderRepository` would silently be bypassed for half the call sites. `AddRepository` makes the override the single binding for both `IRepository<Order, Guid>` and `IOrderRepository`.

---

## Summary of what changed and why

| Concern | Before | After | Rule |
|---|---|---|---|
| LINQ predicate / sort / paging | In `OrderAppService` | In `OrderRepository` (EF Core layer) | Rule 2 |
| `GetQueryableAsync` / `AsyncExecuter` | Called from AppService | Confined to repository implementation | Rule 2 |
| Repository contract | Generic `IRepository<Order, Guid>` | Custom `IOrderRepository : IRepository<Order, Guid>` in Domain | Rule 2 |
| `CancellationToken` | Not plumbed | Repository methods accept one; `GetCancellationToken` fallback | EF Core best practice |
| Paging input | Bare `skipCount, maxResultCount` ints | `PagedAndSortedResultRequestDto` | ABP convention |
| Mapper / current user | Manually injected pattern | Inherited via `ApplicationService` | Framework convention |
| DI registration | Generic only | `AddRepository<Order, OrderRepository>` in the EF Core module | EF Core best practice |

If you later need this read to be **read-only** for clarity, change the AppService injection to `IReadOnlyRepository<Order, Guid>` for the simple cases, and keep `IOrderRepository` for the complex list/count above. (Custom repositories can also be designed to inherit read-only semantics, but the simpler split is enough here.)
