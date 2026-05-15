# Review: `OrderManager` domain service

Short answer: yes, several things are wrong. The class violates the two load-bearing rules for ABP domain services in a Layered solution.

## What's wrong

### 1. Rule 3 violation — injects the **tracking** repository

```csharp
private readonly IRepository<Customer, Guid> _customerRepository;
private readonly IRepository<Order, Guid>   _orderRepository;
```

A `*Manager` (domain service) must inject `IReadOnlyRepository<T, TKey>` only — never `IRepository<T, TKey>`. The tracking repository belongs in the orchestrating `*AppService`, which is the one that mints Ids, fetches tracked entities, and persists. Seeing `IRepository<T, TKey>` in a `DomainService` is a smell that the manager is either (a) persisting itself (forbidden) or (b) should be receiving the mutated entity as a method parameter instead — see the "Repository-interface rule for domain services" paragraph under Rule 3.

### 2. Rule 3 violation — domain service is **persisting**

```csharp
await _orderRepository.InsertAsync(order, autoSave: true);
...
await _orderRepository.UpdateAsync(order);
```

Domain services don't persist. Per Rule 3 they "Never call `InsertAsync` / `UpdateAsync` / `DeleteAsync` themselves." They validate invariants, mutate entities in place that the application service hands them, and return. The application service performs the `InsertAsync` / `UpdateAsync`. Removing the persistence calls also drops the need for `autoSave: true` — the Unit of Work commit at the end of the app service call covers it.

### 3. Rule 3 — `CreateAsync` should take the `id` from the caller, not mint it

The current `CreateAsync` calls `GuidGenerator.Create()` itself. The skill's Rule 3 worked example shows the **app service** mints the Id and passes it in, so the manager signature is `CreateAsync(Guid id, Guid customerId, string number)` and returns an unpersisted `Order`. `GuidGenerator` is inherited by `ApplicationService` — the convention is to keep Id minting on the app-service side so the app service can echo the Id back in the response DTO without re-reading the aggregate.

(The base `DomainService` does expose `GuidGenerator` too, so this point is a convention/consistency one rather than a compile error — but the convention in Rule 3 is unambiguous.)

### 4. `ChangePriceAsync` should take the **entity**, not the **id**

```csharp
public async Task ChangePriceAsync(Guid orderId, decimal newPrice)
{
    var order = await _orderRepository.GetAsync(orderId);
    ...
    await _orderRepository.UpdateAsync(order);
}
```

For mutation of an existing aggregate, Rule 3 Variant B applies: the entity arrives **as a method parameter** from the app service (which holds the tracking `IRepository<Order, Guid>` and did the tracked `GetAsync`). The manager mutates in place and returns — no fetch, no update call.

### 5. Rule 7 nuance — `newPrice <= 0` is own-state, belongs in the entity

```csharp
if (newPrice <= 0)
    throw new BusinessException("Sales:InvalidPrice");
```

Throwing `BusinessException` from a `*Manager` is allowed in principle (Rule 7 explicitly lists it as the second-priority correct home for invariant throws). But "price must be positive" is a rule about the entity's own state, not something needing a repository check. Per Rule 7's priority order, this throw belongs in **the entity** (inside `Order.SetPrice`) so every caller — manager, integration service, future event handler — gets the rule for free. Keep `BusinessException` throws inside the manager for repository-driven checks like the existing `Number` uniqueness check, where the rule genuinely needs a repo read.

For DTO-level fast-fail UX, also add `[Range(0.01, double.MaxValue)]` to a `ChangeOrderPriceDto` so `AbpValidationException` rejects bad input before it reaches the service — Rule 7 is explicit that this DTO annotation is fast-fail UX, not the invariant's home.

### 6. Other smaller issues

- `await _customerRepository.GetAsync(customerId)` reads a `Customer` but the result is never used. Either it should drive an entity-behavior call (the skill's example uses `customer.EnsureCanReceiveNewOrder()`) or be replaced with `if (!await _customerRepository.AnyAsync(c => c.Id == customerId)) throw new BusinessException("Sales:CustomerNotFound");` if you only need an existence check. Loading the full aggregate just to confirm existence is wasteful.
- `using System;` is missing (the file uses `Guid`).
- Namespace `Acme.Sales` — in a Layered solution, the **domain service file lives in the Domain project** (`Acme.Sales.Domain` csproj), typically at `src/Acme.Sales.Domain/Orders/OrderManager.cs`, with namespace `Acme.Sales.Orders`. The application service lives in `src/Acme.Sales.Application/Orders/OrderAppService.cs`.

## Corrected code

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
    private readonly IReadOnlyRepository<Order, Guid>    _orderRepository;

    public OrderManager(
        IReadOnlyRepository<Customer, Guid> customerRepository,
        IReadOnlyRepository<Order, Guid>    orderRepository)
    {
        _customerRepository = customerRepository;
        _orderRepository = orderRepository;
    }

    // Variant A — creating a new aggregate: manager validates + builds, app service persists
    public async Task<Order> CreateAsync(Guid id, Guid customerId, string number)
    {
        var customer = await _customerRepository.GetAsync(customerId);
        customer.EnsureCanReceiveNewOrder();   // entity-behavior method; throws BusinessException if not

        if (await _orderRepository.AnyAsync(o => o.Number == number))
        {
            throw new BusinessException("Sales:OrderNumberAlreadyExists");
        }

        return new Order(id, customerId, number);   // manager doesn't persist
    }

    // Variant B — mutating an existing aggregate: entity arrives by parameter
    public Task ChangePriceAsync(Order order, decimal newPrice)
    {
        // The "newPrice > 0" check now lives inside Order.SetPrice (Rule 7 — entity owns its own-state invariants).
        // The manager would still be the right place for any repo-driven price invariant (e.g., "below current floor price").
        order.SetPrice(newPrice);
        return Task.CompletedTask;
    }
}
```

Notes on this version:
- `IReadOnlyRepository` uses EF Core no-tracking queries by default — both faster and a signal of intent (Rule 2 table).
- If `Customer.EnsureCanReceiveNewOrder()` does not yet exist, add it on the `Customer` aggregate (Rule 7 priority 1). If there is no customer-side guard at all, replace the `GetAsync` with an `AnyAsync` existence check as noted above.
- The constructor of `Order` should be `internal` so it can only be created through `OrderManager` — see the Common Mistakes entry on `public` aggregate constructors in `architecture-ddd.md`. The `Order.SetPrice` method holds the `if (newPrice <= 0) throw new BusinessException("Sales:InvalidPrice");` guard.

### Application layer — `src/Acme.Sales.Application/Orders/OrderAppService.cs`

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
    private readonly OrderManager             _orderManager;
    private readonly IRepository<Order, Guid> _orderRepository;   // tracking repository lives HERE

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
            GuidGenerator.Create(),       // ApplicationService inherits GuidGenerator (Rule 3 convention)
            input.CustomerId,
            input.Number);

        await _orderRepository.InsertAsync(order, autoSave: true);

        return ObjectMapper.Map<Order, OrderDto>(order);
    }

    [Authorize(SalesPermissions.Orders.Edit)]
    public async Task ChangePriceAsync(Guid id, ChangeOrderPriceDto input)
    {
        var order = await _orderRepository.GetAsync(id);                  // tracked fetch
        await _orderManager.ChangePriceAsync(order, input.NewPrice);      // pass in for mutation
        await _orderRepository.UpdateAsync(order);                        // app service persists
        // `autoSave` not needed — the ambient UoW commits on app-service return.
    }
}
```

Authorization sits on the **AppService**, not the controller — Rule 4: the AppService is the canonical authorization boundary, since every caller (HTTP, background jobs, integration services, event handlers, internal callers) routes through it.

### Supporting pieces (sketch)

`src/Acme.Sales.Application.Contracts/Orders/ChangeOrderPriceDto.cs`:

```csharp
using System.ComponentModel.DataAnnotations;

namespace Acme.Sales.Orders;

public class ChangeOrderPriceDto
{
    [Range(0.01, double.MaxValue, ErrorMessage = "Price must be positive.")]
    public decimal NewPrice { get; set; }
}
```

`src/Acme.Sales.Domain/Orders/Order.cs` (only the price method shown — pattern, not the full entity):

```csharp
public void SetPrice(decimal newPrice)
{
    if (newPrice <= 0)
    {
        throw new BusinessException("Sales:InvalidPrice");
    }
    Price = newPrice;
}
```

## Summary of rule citations

- **Rule 2 — Never expose `IQueryable` to the application service layer.** Picking the narrowest repository interface for the job is the same rule applied to domain services: `IReadOnlyRepository<T, TKey>` here.
- **Rule 3 — Domain services don't persist; application services orchestrate.** A `*Manager` injects `IReadOnlyRepository<T, TKey>` only; mutated entities flow in as method parameters; the tracking `IRepository<T, TKey>` lives in the app service, which mints Ids, fetches tracked entities, and persists.
- **Rule 7 — Don't throw business exceptions from application services; use ABP's validation pipeline for input.** Entity owns own-state invariants (`Order.SetPrice`); `*Manager` owns repo-driven invariants (uniqueness); the app service throws neither. DTO `[Range]` is fast-fail UX, not the invariant's home.

File locations recap (Layered template, project naming `Acme.Sales`):

- Domain service: `src/Acme.Sales.Domain/Orders/OrderManager.cs`, namespace `Acme.Sales.Orders`.
- Application service: `src/Acme.Sales.Application/Orders/OrderAppService.cs`, namespace `Acme.Sales.Orders`.
- `IOrderAppService`, DTOs, `SalesPermissions`: `src/Acme.Sales.Application.Contracts/Orders/` and `src/Acme.Sales.Application.Contracts/Permissions/`.
- Entities `Order` and `Customer`: `src/Acme.Sales.Domain/Orders/Order.cs` and `src/Acme.Sales.Domain/Customers/Customer.cs`.
