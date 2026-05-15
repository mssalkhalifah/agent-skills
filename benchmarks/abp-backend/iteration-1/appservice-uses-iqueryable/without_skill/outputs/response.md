There are a few things worth fixing here. The code "works" — it will return the right rows — but it leaks query concerns into the application layer, mixes responsibilities that DDD wants separated, and has a couple of small ABP-specific paper cuts. Let me walk through each issue, then show what it should look like.

## What's wrong

### 1. The application service is doing query construction

The biggest smell is this:

```csharp
var queryable = await _orderRepository.GetQueryableAsync();
var query = queryable
    .Where(o => o.CustomerId == customerId && o.Status == OrderStatus.Active)
    .OrderByDescending(o => o.CreationTime);
```

In an ABP DDD-style solution, the application layer is supposed to be a thin orchestrator: validate input, authorize, call domain/repository methods, map to DTOs, return. Hand-rolling LINQ over `IQueryable<Order>` here means:

- The query lives in the wrong layer. If another caller (a domain service, a background job, a different app service) needs "active orders for a customer," they either re-write the same `Where` or take a dependency on this app service. Both are bad.
- `OrderStatus.Active` is a domain concept (what does "active" actually mean — `Pending | Processing | Shipped`? a single enum value?). The application service shouldn't be the place that decides.
- It's hard to test. You'd have to spin up an EF Core in-memory provider or the full integration test host just to unit-test the predicate.
- `IQueryable<Order>` exposes the entity to LINQ leakage. Anyone editing this method can accidentally call `.Include(...)`, `.AsNoTracking()`, or some EF-specific method, coupling the app service to EF Core.

The right move is to push the query into a **custom repository** (or a domain service if there's logic beyond a query). The app service then just calls `_orderRepository.GetActiveForCustomerAsync(...)`.

### 2. `Order` is being mapped directly to `OrderDto`

```csharp
ObjectMapper.Map<List<Order>, List<OrderDto>>(orders)
```

Mapping aggregate roots straight to DTOs works, but it tightly couples your contract (`OrderDto`) to your aggregate shape. Two related issues:

- You're materializing full `Order` aggregates (with all owned types, navigations EF chooses to load, etc.) just to project a few fields out. For a list endpoint, that's wasteful.
- If the aggregate gains a new property (an internal one, a child collection), AutoMapper will happily try to map it onto the DTO and you can leak data.

For list endpoints, projecting straight to the DTO inside the repository (`Select` to `OrderDto`) is faster and safer. For a single-aggregate "get details" endpoint, mapping the aggregate is fine.

### 3. There's no authorization or input validation

The method takes a `customerId` and returns that customer's orders. As written, **any authenticated user can pass any `customerId` and read someone else's orders**. You need either:

- `[Authorize(OrderPermissions.Orders.Default)]` plus a check that `customerId == CurrentUser.GetId()` (or similar) for the "current customer" case, or
- An admin-only permission for the "look up any customer" case, or
- Both, branching on the caller's permissions.

Also: `skipCount` and `maxResultCount` are unvalidated. ABP gives you `PagedAndSortedResultRequestDto` (or at minimum `PagedResultRequestDto`) which has `MaxMaxResultCount` enforcement built in — use it instead of two loose `int`s. Otherwise a caller can pass `maxResultCount = 1_000_000` and DOS your DB.

### 4. `OrderByDescending(o => o.CreationTime)` assumes the entity has it

Minor, but: `Order` in your snippet probably extends `FullAuditedAggregateRoot<Guid>` or similar, which provides `CreationTime`. That's fine. Just be aware the prompt says "sorted by date desc" — if "date" means an `OrderDate` business field (when the customer placed it), not `CreationTime` (when the row was inserted), those can drift apart and you want the business field. Check with your domain.

### 5. Small ABP nits

- `GetQueryableAsync()` is correct (the older sync `GetQueryable()` is deprecated/avoided), so good.
- Using `AsyncExecuter` is correct — you'd otherwise need to reference `Microsoft.EntityFrameworkCore` in the application layer just to call `ToListAsync`/`CountAsync`, which breaks the layering rule. Keep `AsyncExecuter` and push the LINQ to the repo (where referencing EF is fine).
- `_orderRepository` typed as `IRepository<Order, Guid>` is fine for the **default** repository, but once you have any custom query you should declare an `IOrderRepository` interface in the Domain layer and inject that instead.

## How it should look

Below is the layered version. It assumes a standard ABP layered template (Domain, Application, EntityFrameworkCore projects).

### Domain layer — the custom repository interface

`Acme.OrdersService.Domain/Orders/IOrderRepository.cs`

```csharp
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using Volo.Abp.Domain.Repositories;

namespace Acme.OrdersService.Orders;

public interface IOrderRepository : IRepository<Order, Guid>
{
    Task<List<OrderListItem>> GetActiveForCustomerAsync(
        Guid customerId,
        int skipCount,
        int maxResultCount,
        string? sorting = null,
        CancellationToken cancellationToken = default);

    Task<long> CountActiveForCustomerAsync(
        Guid customerId,
        CancellationToken cancellationToken = default);
}
```

`OrderListItem` is a small **read model** (POCO) that lives next to the repository interface in Domain. It carries only what the list endpoint needs:

```csharp
public class OrderListItem
{
    public Guid Id { get; set; }
    public string OrderNumber { get; set; } = default!;
    public DateTime OrderDate { get; set; }
    public OrderStatus Status { get; set; }
    public decimal TotalAmount { get; set; }
}
```

(If you'd rather not introduce a read model, return `List<Order>` and project to DTO in the app service — but you give up the projection speed-up.)

### EntityFrameworkCore layer — the repository implementation

`Acme.OrdersService.EntityFrameworkCore/Orders/EfCoreOrderRepository.cs`

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Linq.Dynamic.Core;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.EntityFrameworkCore;
using Volo.Abp.Domain.Repositories.EntityFrameworkCore;
using Volo.Abp.EntityFrameworkCore;

namespace Acme.OrdersService.Orders;

public class EfCoreOrderRepository
    : EfCoreRepository<OrdersServiceDbContext, Order, Guid>, IOrderRepository
{
    public EfCoreOrderRepository(IDbContextProvider<OrdersServiceDbContext> dbContextProvider)
        : base(dbContextProvider)
    {
    }

    public async Task<List<OrderListItem>> GetActiveForCustomerAsync(
        Guid customerId,
        int skipCount,
        int maxResultCount,
        string? sorting = null,
        CancellationToken cancellationToken = default)
    {
        var dbSet = await GetDbSetAsync();

        return await dbSet
            .Where(o => o.CustomerId == customerId && o.Status == OrderStatus.Active)
            .OrderBy(sorting ?? nameof(Order.CreationTime) + " desc")
            .Skip(skipCount)
            .Take(maxResultCount)
            .Select(o => new OrderListItem
            {
                Id = o.Id,
                OrderNumber = o.OrderNumber,
                OrderDate = o.OrderDate,
                Status = o.Status,
                TotalAmount = o.TotalAmount
            })
            .ToListAsync(GetCancellationToken(cancellationToken));
    }

    public async Task<long> CountActiveForCustomerAsync(
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

A few things to notice:

- EF Core APIs (`ToListAsync`, `LongCountAsync`) are referenced directly here — that's fine, this project depends on EF Core.
- `System.Linq.Dynamic.Core` lets you accept a `sorting` string like `"OrderDate desc"` from the caller (matches `PagedAndSortedResultRequestDto.Sorting`).
- The projection (`Select`) means the DB only ships back the columns you need.
- ABP's `AbpAutofacModule` will discover this implementation and wire `IOrderRepository` to it; if it doesn't (older templates), register it in your `*EntityFrameworkCoreModule.ConfigureServices` with `context.Services.AddAbpDbContext<OrdersServiceDbContext>(opt => opt.AddRepository<Order, EfCoreOrderRepository>())` or similar. ABP 10.x should auto-register.

### Application layer — the slimmed-down app service

`Acme.OrdersService.Application/Orders/OrderAppService.cs`

```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Authorization;
using Volo.Abp;
using Volo.Abp.Application.Dtos;
using Volo.Abp.Application.Services;

namespace Acme.OrdersService.Orders;

[Authorize(OrderPermissions.Orders.Default)]
public class OrderAppService : ApplicationService, IOrderAppService
{
    private readonly IOrderRepository _orderRepository;

    public OrderAppService(IOrderRepository orderRepository)
    {
        _orderRepository = orderRepository;
    }

    public async Task<PagedResultDto<OrderListItemDto>> GetActiveForCustomerAsync(
        GetActiveOrdersInput input)
    {
        // Authorization: non-admins can only read their own orders.
        if (!await AuthorizationService.IsGrantedAsync(OrderPermissions.Orders.ReadAny))
        {
            var currentUserId = CurrentUser.GetId();
            if (input.CustomerId != currentUserId)
            {
                throw new AbusinessException(OrderErrorCodes.CannotReadOtherCustomersOrders);
            }
        }

        var totalCount = await _orderRepository.CountActiveForCustomerAsync(input.CustomerId);

        var items = await _orderRepository.GetActiveForCustomerAsync(
            input.CustomerId,
            input.SkipCount,
            input.MaxResultCount,
            input.Sorting);

        return new PagedResultDto<OrderListItemDto>(
            totalCount,
            ObjectMapper.Map<List<OrderListItem>, List<OrderListItemDto>>(items));
    }
}
```

And the input DTO:

```csharp
using System;
using Volo.Abp.Application.Dtos;

public class GetActiveOrdersInput : PagedAndSortedResultRequestDto
{
    public Guid CustomerId { get; set; }
}
```

`PagedAndSortedResultRequestDto` brings `SkipCount`, `MaxResultCount`, and `Sorting` along with their validation attributes (and respects `AbpValidationOptions.MaxMaxResultCount` if you've configured it).

`OrderListItemDto` lives in `*.Application.Contracts` and is mapped from `OrderListItem` in your `*ApplicationAutoMapperProfile`:

```csharp
CreateMap<OrderListItem, OrderListItemDto>();
```

### What we've gained

- **Layering is honest.** App service has no LINQ, no `IQueryable`, no `AsyncExecuter`. The query is in the EF Core project where EF dependencies belong.
- **Reusable.** Any other caller — a domain service, a background worker, a `CustomerAppService` — can inject `IOrderRepository` and call `GetActiveForCustomerAsync` without re-deriving the predicate.
- **Authorized.** Non-admin callers can only see their own orders.
- **Validated.** `MaxResultCount` is bounded by ABP's framework config.
- **Faster.** The DB returns five columns, not the whole `Order` row plus its owned types.
- **Testable.** You can unit-test `OrderAppService` with a mocked `IOrderRepository`. The repo itself is integration-tested against a real EF provider.

## When the original style is acceptable

To be fair, there are cases where pulling `IQueryable` into the app service is fine:

- One-off admin tools / list pages where you accept the coupling.
- Very early prototyping before the data shape is stable.
- A genuinely simple list (`GetListAsync(predicate, sorting, skipCount, maxResultCount)` on the default `IRepository`) that you can express without a custom repo at all. ABP's default `IRepository<TEntity, TKey>` already has `GetPagedListAsync` overloads — for a trivial filter, those are enough.

But once the query has any business meaning (and "active orders" qualifies — the definition of "active" is domain knowledge), it belongs behind a named repository method.
