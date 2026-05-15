Yes — this `OrderManager` violates two of ABP's domain-service guardrails. They're related, and they both come from the same root cause: the manager has been handed responsibilities that belong to the application service.

## What's wrong

### 1. Domain services must not persist (Rule 3)

`*Manager` is a domain service. Its job is to enforce business invariants — uniqueness checks, count guards, sibling-aggregate rules — and to mutate entities in place. **It must never call `InsertAsync` / `UpdateAsync` / `DeleteAsync`.** Persistence is the application service's job; the manager runs inside the same Unit of Work and the app service flushes when its method returns.

Your `CreateAsync` calls `_orderRepository.InsertAsync(order, autoSave: true)` and `ChangePriceAsync` calls `_orderRepository.UpdateAsync(order)`. Both are forbidden in a domain service.

`autoSave: true` makes it worse — it forces an immediate `SaveChanges` mid-Unit-of-Work, which can split work that should commit atomically with the rest of the operation into two transactions.

### 2. Wrong repository interface — should be `IReadOnlyRepository<T, TKey>` (Rule 3)

A `*Manager` should inject `IReadOnlyRepository<T, TKey>` for *every* repository it needs, including for the entity it's the canonical authority over. Reads use `IReadOnlyRepository` (no-tracking by default — faster, signals intent); mutations arrive **as method parameters** from the application service, which holds the tracking `IRepository<T, TKey>`.

Seeing `IRepository<Order, Guid>` injected into `OrderManager` is the classic smell: it means either (a) the manager is persisting (forbidden — your case) or (b) a mutation that should flow in as a parameter is being re-fetched inside the manager instead. Same goes for `IRepository<Customer, Guid>` — the manager only reads the customer, so `IReadOnlyRepository` is the right choice.

### 3. `GuidGenerator` lives on the application service (Rule 3)

Minor but conventional: minting Ids is an orchestration concern. Both `ApplicationService` and `DomainService` expose `GuidGenerator`, so it technically compiles either way, but the ABP convention is for the app service to mint the Id and pass it into the manager — that keeps the manager focused on invariants.

### 4. `BusinessException` placement is correct — but `InvalidPrice` is in the wrong place

Throwing `BusinessException("Sales:OrderNumberAlreadyExists")` from the manager is exactly right — it's a uniqueness check that requires a repository read, which is canonically what domain services are for (Rule 7).

But `if (newPrice <= 0) throw new BusinessException("Sales:InvalidPrice")` reads only the input — no repository check, no sibling aggregate. That's an entity-level invariant, so it belongs inside `Order.SetPrice(decimal)`. That way *every* path that ever sets a price (manager, future event handler, integration service, seeder) gets the rule enforced. Rule 7's priority order: entity first, manager only when a repository check is needed, never the application service.

## Corrected version

Two files — the manager (read-only, no persistence, no Id minting) and the application service that orchestrates around it.

```csharp
// Domain/Orders/OrderManager.cs
using System;
using System.Threading.Tasks;
using Volo.Abp;
using Volo.Abp.Domain.Repositories;
using Volo.Abp.Domain.Services;

namespace Acme.Sales;

public class OrderManager : DomainService
{
    private readonly IReadOnlyRepository<Customer, Guid> _customerRepository;
    private readonly IReadOnlyRepository<Order, Guid> _orderRepository;

    public OrderManager(
        IReadOnlyRepository<Customer, Guid> customerRepository,
        IReadOnlyRepository<Order, Guid> orderRepository)
    {
        _customerRepository = customerRepository;
        _orderRepository = orderRepository;
    }

    // Variant A — creating a new aggregate: Manager builds and returns it.
    // Id is minted by the app service and passed in.
    public async Task<Order> CreateAsync(Guid id, Guid customerId, string number)
    {
        var customer = await _customerRepository.GetAsync(customerId);
        customer.EnsureCanReceiveNewOrder(); // entity-behavior method throws if not allowed

        if (await _orderRepository.AnyAsync(o => o.Number == number))
        {
            throw new BusinessException("Sales:OrderNumberAlreadyExists");
        }

        return new Order(id, customerId, number); // Manager does NOT persist
    }

    // Variant B — mutating an existing aggregate: entity arrives by parameter.
    // The price-range check has been pushed into Order.SetPrice itself
    // (see Rule 7 — invariants that read only the entity belong in the entity).
    public Task ChangePriceAsync(Order order, decimal newPrice)
    {
        order.SetPrice(newPrice); // throws inside the entity if newPrice <= 0
        return Task.CompletedTask;
    }
}
```

```csharp
// Application/Orders/OrderAppService.cs
using System;
using System.Threading.Tasks;
using Volo.Abp.Application.Services;
using Volo.Abp.Domain.Repositories;

namespace Acme.Sales;

public class OrderAppService : ApplicationService, IOrderAppService
{
    private readonly OrderManager _orderManager;
    private readonly IRepository<Order, Guid> _orderRepository; // tracking repo lives HERE

    public OrderAppService(
        OrderManager orderManager,
        IRepository<Order, Guid> orderRepository)
    {
        _orderManager = orderManager;
        _orderRepository = orderRepository;
    }

    public async Task<OrderDto> CreateAsync(CreateOrderDto input)
    {
        var order = await _orderManager.CreateAsync(
            GuidGenerator.Create(),     // ApplicationService inherits GuidGenerator
            input.CustomerId,
            input.Number);

        await _orderRepository.InsertAsync(order); // no autoSave — UoW will flush
        return ObjectMapper.Map<Order, OrderDto>(order);
    }

    public async Task ChangePriceAsync(Guid id, decimal newPrice)
    {
        var order = await _orderRepository.GetAsync(id);     // tracked fetch
        await _orderManager.ChangePriceAsync(order, newPrice); // pass entity in
        await _orderRepository.UpdateAsync(order);            // app service persists
    }
}
```

And the entity-side change that goes with this:

```csharp
// Domain/Orders/Order.cs (excerpt)
public void SetPrice(decimal newPrice)
{
    if (newPrice <= 0)
    {
        throw new BusinessException("Sales:InvalidPrice");
    }
    Price = newPrice;
}
```

## Quick rule recap

- **Rule 3** — domain services don't persist; they own invariants and mutate in place. `*Manager` injects `IReadOnlyRepository<T, TKey>` only; mutated entities flow in as method parameters from the app service, which holds the tracking `IRepository<T, TKey>`.
- **Rule 7** — `BusinessException` lives where the rule is enforced. Entity first (state-only checks like `newPrice <= 0`), manager when a repository check is needed (uniqueness, count, sibling guard), never the app service. Throwing from the manager for `OrderNumberAlreadyExists` is correct; throwing from the manager for `InvalidPrice` is misplaced — it should be inside `Order.SetPrice`.

One more incidental thing worth flagging: the original file uses `Guid` without `using System;`. Make sure the corrected version's `using` directives match your file's existing imports — ABP's templates usually have `System` implicitly via `<ImplicitUsings>enable</ImplicitUsings>` in the `*.Domain.csproj`, but it's worth confirming.
