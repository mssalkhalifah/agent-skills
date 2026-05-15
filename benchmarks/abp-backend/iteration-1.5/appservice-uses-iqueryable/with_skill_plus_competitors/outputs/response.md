# Yes — there's one core problem, and it's load-bearing

Your snippet works at runtime, but it violates the ABP layering rule that an application service must never compose `IQueryable` / LINQ. The fact that everything else looks fine (paging shape, `AsyncExecuter`, mapping back to a DTO) is what makes this kind of code spread — it reads like idiomatic ABP, but it leaks an EF Core query into a layer that's supposed to be persistence-agnostic.

Here's what's wrong, why it matters, and what the refactor looks like.

## What's wrong

```csharp
var queryable = await _orderRepository.GetQueryableAsync();
var query = queryable
    .Where(o => o.CustomerId == customerId && o.Status == OrderStatus.Active)
    .OrderByDescending(o => o.CreationTime);
```

The application service is now the place where the query for "active orders for a customer, paged, sorted by date desc" lives. That has four downstream effects, in roughly increasing severity:

1. **It puts persistence concerns in the orchestration layer.** `OrderAppService` is supposed to validate input, call domain services for invariants, persist, and map to DTOs. Composing predicates and `OrderByDescending` is data-access work. The application layer is now coupled to the shape of the EF Core `IQueryable` translator (e.g., what's translatable, what causes client evaluation, what triggers split queries).

2. **The query is unreusable.** A second caller that needs "active orders for a customer" — a domain event handler, a background job that emails customers their open orders, an integration service exposing the same data cross-module — has to either re-implement the same `Where` clause or take a dependency on `OrderAppService`. The first creates duplicate filters that drift; the second couples those callers to the application-service contract.

3. **There's no test seam at the data-access boundary.** Want to unit-test `OrderAppService` in isolation? You now need a real EF Core in-memory provider (or a fake `IQueryable`) because the LINQ expression is in the SUT. With the query encapsulated in a custom repository, `OrderAppService` only depends on the repository's interface, which is trivially mockable.

4. **It's the explicit "no" in ABP's recommended patterns.** The ABP framework guidance (Rule 2, recommended pattern across all three solution templates) is: *application services must call repository methods that return materialised results — `Task<TEntity>`, `Task<List<TEntity>>`, `Task<PagedResultDto<...>>` — and never call `_repository.GetQueryableAsync()` from inside `*AppService.cs`*. If a query is too complex for the generic `IRepository<T, TKey>`, you define a custom `IXyzRepository : IRepository<X, TKey>` and put the EF Core implementation alongside.

The other parts of your code are fine — the use of `AsyncExecuter` to await `CountAsync` / `ToListAsync` is correct, `PagedResultDto<T>` is the right return shape, and `ObjectMapper.Map<List<Order>, List<OrderDto>>` is conventional. The only thing that needs to move is the LINQ.

## How it should look

This is a Layered template (`-t app`) layout — projects, not folders. If you're on Single-Layer, the same structure applies, just folders (`Repositories/` next to `Entities/`).

### 1. Define the contract in the Domain project

```csharp
// src/Acme.Sales.Domain/Orders/IOrderRepository.cs
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

A few things worth calling out about this contract:

- It extends `IRepository<Order, Guid>` so consumers still get the full generic surface (`InsertAsync`, `UpdateAsync`, `GetAsync(id)`, etc.) on top of the custom methods. You're adding capability, not replacing it.
- Two methods, not one: count and list separately. This mirrors what `PagedResultDto<T>` needs and lets the EF Core implementation make its own decision about whether to run them in one round-trip (it shouldn't here — paging needs `OrderBy + Skip + Take`, totals don't, and combining them with `GroupBy(_ => 1)` is a known footgun) or two.
- `CancellationToken` parameters with default values, so the application service can pass one through if it has one, but isn't forced to.
- The interface lives in `Acme.Sales.Domain/Orders/`, next to the `Order` aggregate. The domain owns the *contract*; the EF Core project owns the *implementation*.

### 2. EF Core implementation in the EntityFrameworkCore project

```csharp
// src/Acme.Sales.EntityFrameworkCore/Orders/OrderRepository.cs
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
            .LongCountAsync(
                o => o.CustomerId == customerId && o.Status == OrderStatus.Active,
                GetCancellationToken(cancellationToken));
    }
}
```

Notes on this file:

- `EfCoreRepository<TDbContext, TEntity, TKey>` is the canonical base. Constructor takes `IDbContextProvider<TDbContext>`, which resolves the DbContext through ABP's UoW pipeline (so your repository participates in the ambient transaction and inherits connection-string resolution / multi-tenancy switching for free).
- `GetDbSetAsync()` is the right entry point — it returns the typed `DbSet<Order>`, with ABP's global filters (`ISoftDelete`, `IMultiTenant`) already applied. **This** is where it's legitimate to compose `IQueryable`. Rule 2 forbids it in `*AppService`; it's expected here.
- `GetCancellationToken(token)` falls back to ABP's `ICancellationTokenProvider` if the caller passed `default`, so cooperative cancellation works whether the caller threaded a token through or not.
- `LongCountAsync` rather than `CountAsync` for the total — order counts can exceed `int.MaxValue` and `PagedResultDto<T>.TotalCount` is a `long`. This is a small thing that's easy to miss when you're inlining the query in the app service.

### 3. Register the custom repository

```csharp
// src/Acme.Sales.EntityFrameworkCore/EntityFrameworkCore/SalesEntityFrameworkCoreModule.cs
public override void ConfigureServices(ServiceConfigurationContext context)
{
    context.Services.AddAbpDbContext<SalesDbContext>(options =>
    {
        options.AddDefaultRepositories(includeAllEntities: true);
        options.AddRepository<Order, OrderRepository>();   // ← override the default
    });
    // ... other config (UseSqlServer / UseNpgsql, options, etc.) ...
}
```

`AddDefaultRepositories` registers a generic `IRepository<Order, Guid>` for you. `AddRepository<Order, OrderRepository>` overrides that registration so anyone who injects `IRepository<Order, Guid>` *or* `IOrderRepository` resolves to your custom class. Without the second line, your interface still works, but the generic `IRepository<Order, Guid>` keeps resolving to the default `EfCoreRepository<>` — which means a different code path in the same service silently bypasses your custom repository's behavior.

### 4. Refactored application service

```csharp
// src/Acme.Sales.Application/Orders/OrderAppService.cs
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using Acme.Sales.Orders;
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
        int skipCount,
        int maxResultCount)
    {
        var totalCount = await _orderRepository
            .GetActiveCountForCustomerAsync(customerId);

        var orders = await _orderRepository
            .GetActiveListForCustomerAsync(customerId, skipCount, maxResultCount);

        return new PagedResultDto<OrderDto>(
            totalCount,
            ObjectMapper.Map<List<Order>, List<OrderDto>>(orders));
    }
}
```

That's it. No `GetQueryableAsync`, no LINQ, no `AsyncExecuter` — those were only needed because the query was here. The application service is back to what it should be: orchestration. Validate-input, call-collaborator, map-to-DTO.

## A few smaller things while you're in there

These aren't the main issue, but they're worth fixing in the same pass since the code is open:

- **Take a `CancellationToken` on the app-service method.** ABP's controller layer threads `HttpContext.RequestAborted` through automatically. Add `CancellationToken cancellationToken = default` to `GetActiveForCustomerAsync` and pass it down to both repository calls. Long list endpoints under load benefit from this — when the client gives up, your DB query gives up.

- **Consider `PagedAndSortedResultRequestDto`.** ABP ships a request DTO with `SkipCount`, `MaxResultCount`, and `Sorting` that already has `[Range]` validation on `MaxResultCount`. Three loose `int` parameters work, but you'll want input validation (negative `skipCount`, `maxResultCount = 1_000_000`) and the DTO gives it to you for free. If the sort axis is fixed at "creation time desc" you can ignore `Sorting` and keep the repository method signature as-is; if you ever add another sort, that's where it goes.

- **`PagedResultDto<T>` paging math.** `GetCountAsync` runs against the *unpaged* predicate (not the `Skip().Take()` one) — your original code does this correctly because you only paged the second `ToListAsync` call. Worth saying explicitly because I've seen people pass the paged query to both, which makes `TotalCount` always equal `MaxResultCount` whenever the result is full.

- **Don't switch to `IReadOnlyRepository<Order, Guid>` here just because this method only reads.** `IReadOnlyRepository` is a reasonable choice when you can express the whole query against the generic surface — but you can't (you need a custom method), so you'd end up writing the same `IOrderRepository` interface anyway. Within the EF Core implementation, `EfCoreRepository<>` already returns no-tracking queries from `GetQueryableAsync()` for read-only repos; for an `IRepository<>` like above, you can apply `.AsNoTracking()` inside the list method if you've measured tracking overhead. Don't add it speculatively.

## The shape to internalize

When you look at an ABP application service, the rule of thumb is: **every line should be either input validation, calling a domain service / repository method that returns a materialised type, mapping, or throwing — never composing a query.** The moment you reach for `GetQueryableAsync`, `Where`, `OrderBy`, or `Skip` inside `*AppService.cs`, the query has wandered out of the layer that owns it. The fix is always "push it into a custom repository method and inject that interface." It's the same fix every time, and the test for whether you've done it right is whether the application service reads like prose: "get the count, get the page, return a DTO."
