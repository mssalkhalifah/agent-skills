Yes — there are several things wrong with this `OrderManager`. They're all variants of the same root issue: **a domain service that persists and owns the read-side fetch isn't a domain service anymore — it's an application service wearing a `Manager` suffix.** ABP's DDD conventions for domain services are explicit about this.

## What's wrong

### 1. The Manager persists. It must not.

`CreateAsync` calls `_orderRepository.InsertAsync(order, autoSave: true)` and `ChangePriceAsync` calls `_orderRepository.UpdateAsync(order)`. Both are forbidden in a `*Manager`. Domain services own *invariants* and *pure logic*; the application service orchestrates persistence. The split exists so every caller (app service, background job, integration service, domain event handler) routes through the same invariant enforcement and the same Unit-of-Work boundary — collapsing them couples the rule to whichever Manager method happens to wrap the `InsertAsync`/`UpdateAsync`.

A practical consequence here: `autoSave: true` inside the Manager forces a flush partway through whatever larger operation called `CreateAsync`. The app service's automatic UoW can no longer batch the order creation with anything else it does in the same call. That's not a domain decision — it's a transactional one, and it doesn't belong in the domain layer.

### 2. The Manager injects `IRepository<T, Guid>` (the tracking repository). It should inject `IReadOnlyRepository<T, Guid>`.

`*Manager` reads should go through `IReadOnlyRepository<TEntity, TKey>` — read-only, no-tracking by default, signals intent. Seeing `IRepository<T, TKey>` injected into a Manager is a smell: either the Manager is persisting (which is what's happening here) or the mutation should flow in as a parameter from the app service, which holds the tracking repository.

This is also why the `_customerRepository` is wrong as `IRepository<Customer, Guid>` — the Manager only needs to *read* the customer to verify it exists / can receive a new order. Tracking is unnecessary and misleading.

### 3. `ChangePriceAsync` loads the entity inside the Manager.

`var order = await _orderRepository.GetAsync(orderId);` is a tracked fetch happening on the Manager's repository. The mutated entity should arrive **as a method parameter** from the application service — the app service holds the tracking `IRepository<Order, Guid>`, fetches, hands the entity to the Manager, the Manager mutates in place (validates the invariant, calls `order.SetPrice(...)`), and the app service persists.

Mutating-an-existing-aggregate is the textbook "entity-by-parameter" case.

### 4. Manager mints the Guid for a new aggregate.

`new Order(GuidGenerator.Create(), customerId, number)` — this works (the `DomainService` base class exposes `GuidGenerator` just like `ApplicationService` does), but the convention is that **the application service mints Ids** via the `GuidGenerator` it inherits from `ApplicationService`, then passes the Id to the Manager. That keeps Id generation a single, predictable step in the orchestration path and matches how the rest of an ABP Layered solution is shaped.

### 5. The `newPrice <= 0` check is misplaced.

`ChangePriceAsync` checks `newPrice <= 0` *after* loading the order. That's a price-shape rule that doesn't depend on any aggregate state — it should live on the entity (inside `Order.SetPrice`) so any caller that goes through `SetPrice` from any code path (Manager, integration service, event handler) gets the rule. Even better, validate the *input* on the DTO with `[Range(0.01, ...)]` for fast-fail UX, and keep the entity's own check as the invariant home.

The uniqueness check (`AnyAsync(o => o.Number == number)`) is correctly in the Manager — that one *does* require a repository read, and `BusinessException` is the right exception type for it.

### 6. Minor: missing `using System;`

`Guid` is in `System` and the file imports `System.Threading.Tasks` but not `System`. Probably fine via implicit usings on a modern target, but worth noting.

## Corrected version

Three files: the Manager, the entity behaviour, and the orchestrating application service.

### `Acme.Sales.Domain/Orders/OrderManager.cs`

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
    private readonly IReadOnlyRepository<Order, Guid>    _orderRepository;

    public OrderManager(
        IReadOnlyRepository<Customer, Guid> customerRepository,
        IReadOnlyRepository<Order, Guid>    orderRepository)
    {
        _customerRepository = customerRepository;
        _orderRepository    = orderRepository;
    }

    // Variant A — creating a new aggregate: Manager validates invariants and returns
    // the new aggregate. The application service mints the Id and persists.
    public async Task<Order> CreateAsync(Guid id, Guid customerId, string number)
    {
        var customer = await _customerRepository.GetAsync(customerId);
        customer.EnsureCanReceiveNewOrder();                          // entity-behavior method throws

        if (await _orderRepository.AnyAsync(o => o.Number == number))
        {
            throw new BusinessException("Sales:OrderNumberAlreadyExists");
        }

        return new Order(id, customerId, number);                      // Manager does NOT persist
    }

    // Variant B — mutating an existing aggregate: the entity arrives as a parameter
    // from the application service, which holds the tracking IRepository<Order, Guid>.
    public Task ChangePriceAsync(Order order, decimal newPrice)
    {
        ArgumentNullException.ThrowIfNull(order);
        order.SetPrice(newPrice);                                      // invariant lives on the entity
        return Task.CompletedTask;
    }
}
```

### `Acme.Sales.Domain/Orders/Order.cs` (the relevant slice)

```csharp
using System;
using Volo.Abp;
using Volo.Abp.Domain.Entities.Auditing;

namespace Acme.Sales.Orders;

public class Order : FullAuditedAggregateRoot<Guid>
{
    public Guid     CustomerId { get; private set; }
    public string   Number     { get; private set; } = default!;
    public decimal  Price      { get; private set; }

    protected Order() { /* EF Core */ }

    // internal — only the Domain assembly can construct (the Manager lives here).
    // App services, controllers, and tests in other projects must go through OrderManager.
    internal Order(Guid id, Guid customerId, string number) : base(id)
    {
        CustomerId = customerId;
        Number     = Check.NotNullOrWhiteSpace(number, nameof(number));
    }

    public void SetPrice(decimal newPrice)
    {
        if (newPrice <= 0)
        {
            throw new BusinessException("Sales:InvalidPrice");         // invariant home
        }
        Price = newPrice;
    }
}
```

The `internal` constructor is deliberate: because `CreateAsync` enforces a business rule (uniqueness on `Number`, customer must be able to receive an order), letting any assembly do `new Order(...)` would let callers bypass the Manager.

### `Acme.Sales.Application/Orders/OrderAppService.cs`

```csharp
using System;
using System.Threading.Tasks;
using Volo.Abp.Application.Services;
using Volo.Abp.Domain.Repositories;

namespace Acme.Sales.Orders;

public class OrderAppService : ApplicationService, IOrderAppService
{
    private readonly OrderManager             _orderManager;
    private readonly IRepository<Order, Guid> _orderRepository;        // tracking lives HERE

    public OrderAppService(
        OrderManager             orderManager,
        IRepository<Order, Guid> orderRepository)
    {
        _orderManager    = orderManager;
        _orderRepository = orderRepository;
    }

    public async Task<OrderDto> CreateAsync(CreateOrderDto input)
    {
        var order = await _orderManager.CreateAsync(
            GuidGenerator.Create(),                                    // app service mints Id
            input.CustomerId,
            input.Number);

        await _orderRepository.InsertAsync(order, autoSave: true);     // app service persists
        return ObjectMapper.Map<Order, OrderDto>(order);
    }

    public async Task ChangePriceAsync(Guid id, decimal newPrice)
    {
        var order = await _orderRepository.GetAsync(id);                // tracked fetch
        await _orderManager.ChangePriceAsync(order, newPrice);          // pass entity in
        await _orderRepository.UpdateAsync(order);                      // app service persists
    }
}
```

## Why these splits matter (one-paragraph summary)

The Manager owns *invariants* — the `Order.Number` uniqueness check requires a repository read, so it correctly lives in `OrderManager` and throws `BusinessException`. The entity owns *state-shape rules* — `newPrice <= 0` is about the entity's own state, so it lives in `Order.SetPrice` where every caller hits it. The application service owns *orchestration* — minting the Id, fetching tracked entities, calling the Manager, and persisting through `IRepository<Order, Guid>`. This split is what lets you call the same domain logic from a controller, a background job, an integration service, or a domain event handler and get identical enforcement, identical UoW behaviour, and no tracking surprises.

Two smaller-but-real notes I'd flag in a code review:

- Consider adding `[Range(0.01, double.MaxValue)]` (or a `decimal`-friendly equivalent like FluentValidation's `.GreaterThan(0)`) to the price field on `ChangePriceDto`. ABP's validation pipeline turns that into a structured 400 before the service body runs — fast-fail UX. Keep `Order.SetPrice`'s own check as the invariant home; the DTO annotation isn't a replacement for it.
- The Manager's `ChangePriceAsync` no longer awaits anything, so it could be `void` or stay `Task` for signature uniformity. I left it as `Task` because future variants (e.g., emitting a domain event via `LocalEventBus`) usually become async, and changing the signature later is a wider blast radius than a stub `Task.CompletedTask` today.
