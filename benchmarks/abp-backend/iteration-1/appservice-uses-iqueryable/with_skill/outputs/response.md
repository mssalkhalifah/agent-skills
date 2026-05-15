Yes — there's one structural problem with this approach, and once you fix it the rest of the code falls into place.

## What's wrong

Your `OrderAppService` calls `_orderRepository.GetQueryableAsync()` and then composes a LINQ query (`Where` / `OrderByDescending` / `Skip` / `Take`) directly inside the application service. That violates the framework guardrail that **application services must not consume `IQueryable`**:

> Application services must call repository methods that return materialised results (`Task<TEntity>`, `Task<List<TEntity>>`, `Task<PagedResultDto<...>>`). Encapsulate queries inside a custom repository — do **not** call `_repository.GetQueryableAsync()` from inside `*AppService.cs`.

Concretely, the problems this causes:

1. **The query shape leaks across layers.** The predicate (`Status == OrderStatus.Active`), the sort (`CreationTime desc`), and the paging are all *data-access concerns*, but they now live in the application layer. If you later add a second caller (a background job, an integration service, a domain event handler) that needs the same "active orders for a customer" list, it has to re-implement the same LINQ — and the moment one of those copies drifts, you have a subtle bug.
2. **The application service can no longer be reasoned about without EF Core knowledge.** Anyone reviewing `OrderAppService` has to know which LINQ operators translate to SQL, whether `OrderStatus` is mapped as int/string, etc. That's exactly the coupling the repository abstraction is supposed to hide.
3. **You can't ship the module on a non-`IQueryable` provider** (e.g., Mongo) without rewriting the service. The custom-repository pattern keeps the service unchanged and only swaps the implementation.
4. **It's a one-way ratchet.** Once one method in a service does this, the next one tends to as well, and your application layer slowly turns into a query-building layer.

A few smaller things while we're here:

- `OrderByDescending` is being applied **before** `CountAsync`. EF will (usually) strip the ordering for the count, but it's still wasted server thinking and signals that the query is being assembled in the wrong place. The custom-repository version below puts the ordering only on the page-fetch path.
- The DTO method probably wants to return `PagedResultDto<OrderDto>` driven by an input DTO that derives from `PagedAndSortedResultRequestDto` — it gives you `MaxResultCount`/`SkipCount` validation for free and lines up with ABP's standard CRUD conventions. Not a hard rule, just nicer.

## How it should look

Define a custom repository on the **Domain** side (the contract) and implement it on the **EntityFrameworkCore** side. The application service just calls two methods that return materialised data.

### 1. Domain layer — the repository contract

```csharp
// Orders/IOrderRepository.cs (in the Domain project / Domain folder)
public interface IOrderRepository : IRepository<Order, Guid>
{
    Task<long> CountActiveForCustomerAsync(
        Guid customerId,
        CancellationToken cancellationToken = default);

    Task<List<Order>> GetActiveForCustomerAsync(
        Guid customerId,
        int skipCount,
        int maxResultCount,
        CancellationToken cancellationToken = default);
}
```

### 2. EntityFrameworkCore layer — the implementation

```csharp
// Orders/OrderRepository.cs (in the EntityFrameworkCore project / Data folder)
public class OrderRepository
    : EfCoreRepository<MyAppDbContext, Order, Guid>, IOrderRepository
{
    public OrderRepository(IDbContextProvider<MyAppDbContext> dbContextProvider)
        : base(dbContextProvider) { }

    public async Task<long> CountActiveForCustomerAsync(
        Guid customerId,
        CancellationToken cancellationToken = default)
    {
        var dbSet = await GetDbSetAsync();
        return await dbSet
            .Where(o => o.CustomerId == customerId && o.Status == OrderStatus.Active)
            .LongCountAsync(GetCancellationToken(cancellationToken));
    }

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
}
```

`IQueryable` lives **inside** the repository implementation — that's where ABP sanctions it. `GetDbSetAsync()` and `GetCancellationToken()` are inherited from `EfCoreRepository<,,>`.

### 3. Register the custom repository

```csharp
// MyAppEntityFrameworkCoreModule.cs
context.Services.AddAbpDbContext<MyAppDbContext>(options =>
{
    options.AddDefaultRepositories(includeAllEntities: true);
    options.AddRepository<Order, OrderRepository>();   // override the default IRepository<Order, Guid>
});
```

The `AddRepository<Order, OrderRepository>()` line is important — without it, anything that injects `IRepository<Order, Guid>` keeps resolving to the default repository and silently bypasses your custom logic. With it, **both** `IRepository<Order, Guid>` *and* `IOrderRepository` resolve to the same `OrderRepository` instance.

### 4. Application service — back to plain orchestration

```csharp
public class OrderAppService : ApplicationService, IOrderAppService
{
    private readonly IOrderRepository _orderRepository;   // depend on the custom interface

    public OrderAppService(IOrderRepository orderRepository)
    {
        _orderRepository = orderRepository;
    }

    public async Task<PagedResultDto<OrderDto>> GetActiveForCustomerAsync(
        Guid customerId, int skipCount, int maxResultCount)
    {
        var totalCount = await _orderRepository
            .CountActiveForCustomerAsync(customerId);

        var orders = await _orderRepository
            .GetActiveForCustomerAsync(customerId, skipCount, maxResultCount);

        return new PagedResultDto<OrderDto>(
            totalCount,
            ObjectMapper.Map<List<Order>, List<OrderDto>>(orders));
    }
}
```

No `GetQueryableAsync`, no `AsyncExecuter`, no LINQ in the service. The service does what it should: ask the repository for two things, map, and return. Any future caller that needs "active orders for a customer" reuses `_orderRepository.GetActiveForCustomerAsync(...)` instead of duplicating the predicate.

## Optional polish

If you want to lean into ABP's CRUD conventions, take an input DTO instead of three loose parameters:

```csharp
public class GetActiveOrdersInput : PagedAndSortedResultRequestDto
{
    public Guid CustomerId { get; set; }
}

public Task<PagedResultDto<OrderDto>> GetActiveForCustomerAsync(GetActiveOrdersInput input);
```

`PagedAndSortedResultRequestDto` already validates `MaxResultCount` (max 1000 by default, configurable) and gives you a `Sorting` string you can pass through to the repository if you want client-driven sorting later. Since your method has a fixed sort order, that's optional — but it's the standard shape for "list paged X" endpoints in ABP.

## TL;DR

- Rule being applied: **never expose `IQueryable` to the application service layer** — encapsulate queries inside a custom repository.
- Move the predicate, sort, and paging into an `IOrderRepository : IRepository<Order, Guid>` with two methods (count + page).
- Register it with `options.AddRepository<Order, OrderRepository>()` so the custom implementation also satisfies the default `IRepository<Order, Guid>` injection.
- `OrderAppService` then just calls those two methods, maps, and returns `PagedResultDto<OrderDto>`.
