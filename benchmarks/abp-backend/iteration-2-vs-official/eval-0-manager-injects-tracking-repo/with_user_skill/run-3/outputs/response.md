# Review: `OrderManager` (ABP 10.3, Layered template)

Yes — this `OrderManager` violates several of the framework's recommended patterns. The single biggest problem is that the domain service is persisting, which the skill calls out explicitly.

## What's wrong

### 1. Wrong repository interface — Rule 3 (Domain services don't persist)

The Manager injects `IRepository<Customer, Guid>` and `IRepository<Order, Guid>` (the full tracking repositories). The rule is:

> A `*Manager` injects `IReadOnlyRepository<T, TKey>` only — for any read it needs, including reads of the entity it's the canonical authority over. Mutated entities arrive **as method parameters** from the application service, which holds the tracking `IRepository<T, TKey>` and persists after.

> Seeing `IRepository<T, TKey>` injected into a `*Manager` is a smell — either the Manager is persisting (forbidden) or the mutation should flow in as a parameter instead.

Both apply here.

### 2. Domain service is persisting — Rule 3

`CreateAsync` calls `_orderRepository.InsertAsync(order, autoSave: true)`, and `ChangePriceAsync` calls `_orderRepository.UpdateAsync(order)`. Per Rule 3:

> Domain services... **Never** call `InsertAsync` / `UpdateAsync` / `DeleteAsync` themselves.

> Application services orchestrate. They mint Ids via the `GuidGenerator` inherited from `ApplicationService`... fetch tracked entities, pass them into the Manager, and persist after the Manager returns.

### 3. The Manager fetches the entity it's supposed to mutate — Rule 3

`ChangePriceAsync(Guid orderId, ...)` does `await _orderRepository.GetAsync(orderId)` itself. For mutation flows (Variant B in the skill), the entity must arrive **as a method parameter** from the AppService — the AppService holds the tracking repository, fetches the tracked entity, hands it to the Manager, and persists after.

### 4. The Manager mints the Id — Rule 3

`new Order(GuidGenerator.Create(), customerId, number)` should not happen in the Manager. The AppService mints the Id via its inherited `GuidGenerator` and passes it into the Manager's `CreateAsync`. The Manager's job is to build (or mutate) the aggregate, not to choose its identity.

### 5. Input validation in the wrong place — Rule 7

`if (newPrice <= 0) throw new BusinessException("Sales:InvalidPrice")` is a pure input-shape check that should travel through ABP's validation pipeline on the DTO (e.g., `[Range(0.01, ...)]` on `ChangeOrderPriceDto.NewPrice`). The invariant's true home is still inside the entity (`Order.SetPrice` should also guard its own state so any caller bypassing the DTO still gets the rule), but the throw doesn't belong in the Manager body for a single-field range check — that's exactly the "fast-fail UX" case Rule 7 reserves for the validation pipeline.

The uniqueness throw in `CreateAsync` (`BusinessException("Sales:OrderNumberAlreadyExists")`) **is** correctly placed in the Manager — that's a repository-driven check and Rule 7 explicitly endorses it there. Keep it.

### 6. Dead read

`var customer = await _customerRepository.GetAsync(customerId);` is fetched and then never used. The skill's reference pattern in Rule 3 fetches the customer to call an entity-behavior method like `customer.EnsureCanReceiveNewOrder()`. Either invoke a real entity-behavior check or remove the fetch. The corrected version below keeps the call and assumes a `Customer.EnsureCanReceiveNewOrder()` method exists (add one if not — that's the canonical pattern).

### 7. Minor: missing `using System;`

The file uses `Guid` but only imports `System.Threading.Tasks;`. Likely covered by `<ImplicitUsings>enable</ImplicitUsings>` in the .csproj, but worth flagging.

---

## Corrected version

Two files, in two different projects.

### `src/Acme.Sales.Domain/Orders/OrderManager.cs` (Domain project)

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

    // Variant A — create: Manager builds and returns; AppService persists.
    // Id is minted by the AppService and passed in.
    public async Task<Order> CreateAsync(Guid id, Guid customerId, string number)
    {
        var customer = await _customerRepository.GetAsync(customerId);
        customer.EnsureCanReceiveNewOrder(); // entity-behavior method throws if invariant fails

        if (await _orderRepository.AnyAsync(o => o.Number == number))
        {
            throw new BusinessException("Sales:OrderNumberAlreadyExists");
        }

        return new Order(id, customerId, number);
    }

    // Variant B — mutate: entity arrives by parameter (tracked by the AppService's repo);
    // Manager mutates in place; AppService persists.
    public Task ChangePriceAsync(Order order, decimal newPrice)
    {
        // The primary range invariant lives inside Order.SetPrice (the entity owns its state).
        // Input-shape validation belongs on the DTO via [Range] — see Rule 7.
        order.SetPrice(newPrice);
        return Task.CompletedTask;
    }
}
```

Notes on namespaces / file location:

- Lives in the `Acme.Sales.Domain` project, typically under `Orders/` (alongside `Order.cs`; keep `Customer.cs` under `Customers/` if it's its own aggregate).
- Namespace `Acme.Sales.Orders` — the original `Acme.Sales` would only be right if the file sat at the project root, which isn't conventional for an aggregate.
- One class per file (Rule 1) — `Order`, `Customer`, `OrderManager` each get their own `.cs`.

### `src/Acme.Sales.Application/Orders/OrderAppService.cs` (Application project)

```csharp
using System;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Authorization;
using Volo.Abp.Application.Services;
using Volo.Abp.Domain.Repositories;
using Acme.Sales.Permissions;

namespace Acme.Sales.Orders;

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

    [Authorize(SalesPermissions.Orders.Create)]
    public async Task<OrderDto> CreateAsync(CreateOrderDto input)
    {
        var order = await _orderManager.CreateAsync(
            GuidGenerator.Create(),          // inherited from ApplicationService — no IGuidGenerator injection
            input.CustomerId,
            input.Number);

        await _orderRepository.InsertAsync(order, autoSave: true);

        return ObjectMapper.Map<Order, OrderDto>(order);
    }

    [Authorize(SalesPermissions.Orders.Edit)]
    public async Task ChangePriceAsync(Guid id, ChangeOrderPriceDto input)
    {
        var order = await _orderRepository.GetAsync(id);            // tracked fetch
        await _orderManager.ChangePriceAsync(order, input.NewPrice); // pass in for mutation
        await _orderRepository.UpdateAsync(order);                   // AppService persists
    }
}
```

### Supporting DTOs (Application.Contracts project)

`src/Acme.Sales.Application.Contracts/Orders/ChangeOrderPriceDto.cs`:

```csharp
using System.ComponentModel.DataAnnotations;

namespace Acme.Sales.Orders;

public class ChangeOrderPriceDto
{
    [Range(0.01, double.MaxValue, ErrorMessage = "Sales:InvalidPrice")]
    public decimal NewPrice { get; set; }
}
```

This is the Rule 7 home for "newPrice must be positive" — the validation pipeline produces a structured 400 via `AbpValidationException` before the service ever runs. The `Order` entity should *also* guard its own state inside `SetPrice` (`if (value <= 0) throw new BusinessException("Sales:InvalidPrice");`) so any non-DTO caller still hits the rule — that's the canonical entity-behavior pattern.

---

## Summary of changes mapped to the skill's rules

| Issue in original | Rule | Fix |
|---|---|---|
| Manager injects `IRepository<Customer, Guid>` and `IRepository<Order, Guid>` | Rule 3 | Inject `IReadOnlyRepository<,>` only |
| Manager calls `InsertAsync` / `UpdateAsync` | Rule 3 | AppService persists; Manager just builds/mutates |
| Manager fetches the order to mutate it | Rule 3 (Variant B) | AppService fetches via tracking repo and passes the entity in |
| Manager mints the Id via `GuidGenerator` | Rule 3 | AppService mints `GuidGenerator.Create()` and passes the Id |
| Imperative `newPrice <= 0` throw in Manager body | Rule 7 | Move to DTO `[Range]`; keep invariant inside `Order.SetPrice` |
| `customer` fetched but unused | Rule 3 (entity-behavior pattern) | Call `customer.EnsureCanReceiveNewOrder()` or drop the fetch |
| Single-file namespace `Acme.Sales` for a domain aggregate | Convention | Use `Acme.Sales.Orders` and place under `Orders/` |

The corrected pair follows the exact split shown in SKILL.md Rule 3: the Manager owns the invariants and stays pure; the AppService orchestrates persistence, mints Ids, and is the boundary where `[Authorize]` lives (Rule 4).
