# Review — `OrderManager` domain service

Yes, several things are wrong. The domain service is doing the application service's job, and it's using the wrong repository interface for its reads. The fixes touch two layers (Domain and Application), and an `IOrderAppService` interface is missing from the picture.

## What's wrong (mapped to the skill's rules)

1. **Rule 3 — Domain services don't persist; application services orchestrate.**
   - `OrderManager.CreateAsync` calls `_orderRepository.InsertAsync(order, autoSave: true)`. Domain services must **never** call `InsertAsync` / `UpdateAsync` / `DeleteAsync`. That belongs in the `*AppService`.
   - `OrderManager.ChangePriceAsync` does the same thing with `UpdateAsync`, and it also *fetches* the order itself — which means the manager owns the tracking lifecycle of an entity it shouldn't be persisting. The application service should fetch the tracked entity and hand it to the manager by parameter.

2. **Rule 3 — Repository-interface rule for domain services.** A `*Manager` injects `IReadOnlyRepository<T, TKey>` only. `OrderManager` is injecting `IRepository<Customer, Guid>` and `IRepository<Order, Guid>` — the *tracking* repositories. That's a smell: it's the signal that the manager either persists (forbidden) or is about to mutate something it shouldn't own. Switch both to `IReadOnlyRepository<…>` for the uniqueness / customer-existence reads.

3. **Rule 3 — Manager mints its own Id.** `OrderManager.CreateAsync` calls `GuidGenerator.Create()` itself. Per the canonical pattern, the **application service** mints the Id (it inherits `GuidGenerator` from `ApplicationService`) and passes it into the manager. The manager just consumes it.

4. **Rule 7 — Don't throw `BusinessException` from the application service; keep invariants where they're enforced.** The throw `"Sales:OrderNumberAlreadyExists"` is correctly placed (it's a repository-driven uniqueness check inside the Manager — that's exactly what the [Domain Services best-practices doc](https://abp.io/docs/10.3/framework/architecture/best-practices/domain-services) authorises). However, `"Sales:InvalidPrice"` (newPrice <= 0) is reading **only the entity's state / input** — no repository check needed. Per the Rule 3 decision rubric ("entity (or an entity behavior method) when the rule reads only the entity's own state"), that check belongs **inside `Order.SetPrice(newPrice)`**, not in the manager. The manager's `ChangePriceAsync` then becomes a thin pass-through; arguably you don't need a manager method at all for this case and the AppService can call `order.SetPrice(...)` directly — but if you keep the manager method for symmetry, the throw still moves into the entity.

5. **File placement / namespaces.** In a Layered solution:
   - `OrderManager` lives in the **`Acme.Sales.Domain`** project, typically under `Orders/OrderManager.cs`, in namespace `Acme.Sales.Orders` (not the flat `Acme.Sales`).
   - `Order` and `Customer` entities live in the same Domain project alongside the manager.
   - `OrderAppService` lives in **`Acme.Sales.Application`** under `Orders/OrderAppService.cs`.
   - `IOrderAppService` and the DTOs (`CreateOrderDto`, `OrderDto`) live in **`Acme.Sales.Application.Contracts`** under `Orders/` (so other modules and the HttpApi can reference them without pulling in the Application implementation).
   - Rule 1 — one class per file: entity, manager, AppService, interface, and each DTO each get their own `.cs` file.

6. **Minor.** The file is missing `using System;` for `Guid` (the original snippet uses `Guid` but only imports `System.Threading.Tasks`).

## Corrected version

### Domain layer — `src/Acme.Sales.Domain/Orders/OrderManager.cs`

```csharp
using System;
using System.Threading.Tasks;
using Volo.Abp;
using Volo.Abp.Domain.Repositories;
using Volo.Abp.Domain.Services;

namespace Acme.Sales.Orders;

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
    // Id is minted by the AppService and passed in.
    public async Task<Order> CreateAsync(Guid id, Guid customerId, string number)
    {
        // Cross-aggregate guard: customer must exist (and could enforce more invariants
        // via an entity-behavior method like customer.EnsureCanReceiveNewOrder()).
        var customer = await _customerRepository.GetAsync(customerId);

        if (await _orderRepository.AnyAsync(o => o.Number == number))
        {
            throw new BusinessException("Sales:OrderNumberAlreadyExists");
        }

        return new Order(id, customerId, number); // no persistence here
    }

    // Variant B — mutating an existing aggregate: entity arrives by parameter.
    // No repository fetch, no UpdateAsync. The price-range invariant moves into Order.SetPrice.
    public void ChangePrice(Order order, decimal newPrice)
    {
        order.SetPrice(newPrice); // entity enforces newPrice > 0
    }
}
```

Note: `ChangePrice` is now synchronous and doesn't touch any repository — it's effectively a thin orchestration seam. If a future change adds a sibling-aggregate guard you'd switch it back to `async Task`; in this case you could even drop the manager method and have the AppService call `order.SetPrice(newPrice)` directly. I've kept it here to preserve the symmetry of the original API.

### Domain layer — `src/Acme.Sales.Domain/Orders/Order.cs` (excerpt — the invariant moves here)

```csharp
using System;
using Volo.Abp;
using Volo.Abp.Domain.Entities.Auditing;

namespace Acme.Sales.Orders;

public class Order : FullAuditedAggregateRoot<Guid>
{
    public Guid CustomerId { get; private set; }
    public string Number { get; private set; }
    public decimal Price { get; private set; }

    protected Order() { } // for ORM / proxy generators

    internal Order(Guid id, Guid customerId, string number) : base(id)
    {
        CustomerId = customerId;
        Number = Check.NotNullOrWhiteSpace(number, nameof(number));
    }

    public void SetPrice(decimal newPrice)
    {
        if (newPrice <= 0)
        {
            throw new BusinessException("Sales:InvalidPrice");
        }
        Price = newPrice;
    }
}
```

The constructor is `internal` so the only public path to create an `Order` is `OrderManager.CreateAsync` — that keeps the uniqueness check non-bypassable. (See the entity-behavior / `internal` constructor guidance in `references/framework/architecture-ddd.md`.)

### Application.Contracts layer — `src/Acme.Sales.Application.Contracts/Orders/IOrderAppService.cs`

```csharp
using System;
using System.Threading.Tasks;
using Volo.Abp.Application.Services;

namespace Acme.Sales.Orders;

public interface IOrderAppService : IApplicationService
{
    Task<OrderDto> CreateAsync(CreateOrderDto input);
    Task ChangePriceAsync(Guid id, decimal newPrice);
}
```

DTOs (`CreateOrderDto`, `OrderDto`) live next to it, each in their own file (Rule 1). Per Rule 7, use DataAnnotations on `CreateOrderDto` (`[Required]`, `[StringLength]` on `Number`, etc.) so ABP's validation pipeline rejects malformed input as a 400 before the AppService ever runs.

### Application layer — `src/Acme.Sales.Application/Orders/OrderAppService.cs`

```csharp
using System;
using System.Threading.Tasks;
using Volo.Abp.Application.Services;
using Volo.Abp.Domain.Repositories;

namespace Acme.Sales.Orders;

public class OrderAppService : ApplicationService, IOrderAppService
{
    private readonly OrderManager _orderManager;
    private readonly IRepository<Order, Guid> _orderRepository; // tracking lives HERE

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

        await _orderRepository.InsertAsync(order, autoSave: true);
        return ObjectMapper.Map<Order, OrderDto>(order);
    }

    public async Task ChangePriceAsync(Guid id, decimal newPrice)
    {
        var order = await _orderRepository.GetAsync(id);  // tracked fetch
        _orderManager.ChangePrice(order, newPrice);        // mutate in place
        await _orderRepository.UpdateAsync(order);         // app service persists
    }
}
```

## Summary of the fix

| What changed | Why |
|---|---|
| `IRepository<…>` -> `IReadOnlyRepository<…>` in `OrderManager` | Rule 3 — managers inject the read-only interface; tracking belongs to the AppService. |
| Removed `InsertAsync` / `UpdateAsync` from `OrderManager` | Rule 3 — domain services don't persist. |
| `OrderManager.ChangePriceAsync` no longer fetches; takes `Order` by parameter | Rule 3 — mutated entities flow in from the AppService that holds the tracking repository. |
| `GuidGenerator.Create()` moved to AppService; Id passed into manager | Rule 3 — AppService mints the Id (`ApplicationService` inherits `GuidGenerator`). |
| `newPrice <= 0` check moved from manager into `Order.SetPrice` | Rule 7 + Rule 3 decision rubric — rule reads only the entity's own state, so it lives in the entity. Non-bypassable that way. |
| Uniqueness check `"Sales:OrderNumberAlreadyExists"` stays in the manager | Rule 7 — repository-driven invariant, canonical place for a `BusinessException` throw. |
| Added `OrderAppService` + `IOrderAppService` in the Application / Application.Contracts projects | Rule 3 / Rule 4 — orchestration, persistence, mapping, and the public contract live in the application layer, not the domain. |
| Files split, namespaces aligned (`Acme.Sales.Orders` in the Domain project) | Rule 1 — one class per file; conventional ABP layered placement under `<Feature>/` folders. |
