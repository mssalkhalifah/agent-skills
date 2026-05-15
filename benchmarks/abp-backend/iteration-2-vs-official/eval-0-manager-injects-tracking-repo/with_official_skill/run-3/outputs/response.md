# Review: `OrderManager` Domain Service

**Skills used:** `abp-ddd`, `abp-application-layer`, `abp-dependency-rules`, `abp-core`.

## Summary

Yes — there are several problems with this domain service. The most important one is an architectural / DDD violation: **the `OrderManager` is acting like an application service**. A domain service should focus on enforcing invariants across aggregates and return domain objects to its caller; it should not orchestrate the use case (fetching unrelated aggregates just to validate IDs, persisting via `InsertAsync` / `UpdateAsync`, owning the transaction with `autoSave: true`). That orchestration belongs in an `OrderAppService`.

Below is the full list of issues and the corrected code for **both** the domain manager and the orchestrating application service.

---

## Issues Found

### 1. Domain service persists aggregates (`InsertAsync` / `UpdateAsync` / `autoSave: true`)
Per `abp-ddd`, a domain service's `CreateAsync` should construct and **return** the aggregate. Persistence belongs to the application service (which owns the unit-of-work boundary). The reference example in the skill ends with `return new Order(...)` — it does not call `InsertAsync`. Using `autoSave: true` inside a domain service is doubly wrong: it forces an early `SaveChanges` and breaks the UoW that the app service is supposed to control.

### 2. Domain service injects `IRepository<Customer, Guid>` (a foreign aggregate's repository)
`Customer` is a different aggregate root and (almost certainly) lives in a different bounded context. `OrderManager` only needs the `customerId` — it should not load the `Customer` aggregate. The skill explicitly warns against navigation across aggregates: *"Reference by Id only, never add full navigation properties across aggregates."* If you must verify the customer exists, that is an application-layer concern (or a separate, explicit cross-context check via a dedicated service / integration). The "tracking repo" the manager grabs here is dead weight — the loaded `customer` is never used.

### 3. `ChangePriceAsync` validates *after* loading and mutates the entity from outside
- `if (newPrice <= 0)` is an **entity invariant**. It must live inside `Order.SetPrice(...)` (rich domain model), not in the manager. The `abp-ddd` skill: *"Entity enforces invariants"* — see the `OrderLine.SetCount` example.
- Even if you keep a manager method (e.g. because cross-aggregate rules apply), it should accept the loaded `Order` (or do the lookup but return it) and let the **app service** call `UpdateAsync`. The skill's `BookAppService.UpdateAsync` example shows the app service doing the `GetAsync` + `UpdateAsync`, not the manager.

### 4. `ChangePriceAsync` is not really a domain service operation
There is no cross-aggregate rule here. If `SetPrice` already enforces the invariant, this method should not exist at all on `OrderManager`. The app service can call `order.SetPrice(newPrice)` directly. Domain services are for *"business logic that spans multiple aggregates or requires repository queries to enforce rules"*.

### 5. `CreateAsync` signature returns an entity but also persists it — pick one role
Returning the `Order` is correct for a domain service. Persisting it inside is not. After fixing #1, the manager returns the new `Order` and the app service inserts it.

### 6. Parameter order convention
The skill's reference `OrderManager.CreateAsync` is `(string orderNumber, Guid customerId)`. Minor, but worth aligning with the documented convention.

### 7. `BusinessException` could carry contextual data
The skill recommends `.WithData("OrderNumber", orderNumber)` so the error payload is useful for logs/UI.

### 8. (Style) No `IOrderRepository` is needed *yet*
Generic `IRepository<Order, Guid>` is fine because no custom queries are used. If you later need `FindByOrderNumberAsync`, introduce `IOrderRepository : IRepository<Order, Guid>` in the **Domain** project (interface) with the implementation in the EF Core project — see `abp-dependency-rules`.

---

## Corrected Code

### `Order` aggregate (Domain project) — invariant lives here

File: `src/Acme.Sales.Domain/Orders/Order.cs`

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

    protected Order() { /* For ORM */ }

    internal Order(Guid id, Guid customerId, string number)
        : base(id)
    {
        CustomerId = customerId;
        Number = Check.NotNullOrWhiteSpace(number, nameof(number));
        Price = 0m;
    }

    public void SetPrice(decimal newPrice)
    {
        if (newPrice <= 0)
        {
            throw new BusinessException("Sales:InvalidPrice")
                .WithData("Price", newPrice);
        }

        Price = newPrice;
    }
}
```

### `OrderManager` (Domain project) — corrected

File: `src/Acme.Sales.Domain/Orders/OrderManager.cs`

```csharp
using System;
using System.Threading.Tasks;
using Volo.Abp;
using Volo.Abp.Domain.Repositories;
using Volo.Abp.Domain.Services;

namespace Acme.Sales.Orders;

public class OrderManager : DomainService
{
    private readonly IRepository<Order, Guid> _orderRepository;

    public OrderManager(IRepository<Order, Guid> orderRepository)
    {
        _orderRepository = orderRepository;
    }

    public async Task<Order> CreateAsync(Guid customerId, string number)
    {
        Check.NotNullOrWhiteSpace(number, nameof(number));

        // Cross-aggregate rule that needs a repository query: number uniqueness.
        if (await _orderRepository.AnyAsync(o => o.Number == number))
        {
            throw new BusinessException("Sales:OrderNumberAlreadyExists")
                .WithData("Number", number);
        }

        // Construct and RETURN. Do not persist here.
        return new Order(GuidGenerator.Create(), customerId, number);
    }
}
```

Notes:
- `GuidGenerator` comes from the `DomainService` base class — do not inject `IGuidGenerator`.
- No `Customer` repository — the manager only needs the `customerId`.
- No `InsertAsync`/`UpdateAsync` and no `autoSave`. The app service owns persistence and the UoW.
- `ChangePriceAsync` is removed; the rule is already enforced in `Order.SetPrice`.

### `OrderAppService` (Application project) — orchestrator

Interface in `src/Acme.Sales.Application.Contracts/Orders/IOrderAppService.cs`:

```csharp
using System;
using System.Threading.Tasks;
using Volo.Abp.Application.Services;

namespace Acme.Sales.Orders;

public interface IOrderAppService : IApplicationService
{
    Task<OrderDto> CreateAsync(CreateOrderDto input);
    Task<OrderDto> ChangePriceAsync(Guid id, ChangeOrderPriceDto input);
}
```

DTOs in `src/Acme.Sales.Application.Contracts/Orders/`:

```csharp
using System;
using System.ComponentModel.DataAnnotations;

namespace Acme.Sales.Orders;

public class CreateOrderDto
{
    [Required]
    public Guid CustomerId { get; set; }

    [Required]
    [StringLength(64, MinimumLength = 1)]
    public string Number { get; set; } = default!;
}

public class ChangeOrderPriceDto
{
    [Range(0.01, double.MaxValue)]
    public decimal NewPrice { get; set; }
}

public class OrderDto
{
    public Guid Id { get; set; }
    public Guid CustomerId { get; set; }
    public string Number { get; set; } = default!;
    public decimal Price { get; set; }
}
```

Implementation in `src/Acme.Sales.Application/Orders/OrderAppService.cs`:

```csharp
using System;
using System.Threading.Tasks;
using Volo.Abp.Application.Services;
using Volo.Abp.Domain.Repositories;

namespace Acme.Sales.Orders;

public class OrderAppService : ApplicationService, IOrderAppService
{
    private readonly IRepository<Order, Guid> _orderRepository;
    private readonly OrderManager _orderManager;
    private readonly OrderMapper _orderMapper;

    public OrderAppService(
        IRepository<Order, Guid> orderRepository,
        OrderManager orderManager,
        OrderMapper orderMapper)
    {
        _orderRepository = orderRepository;
        _orderManager = orderManager;
        _orderMapper = orderMapper;
    }

    public async Task<OrderDto> CreateAsync(CreateOrderDto input)
    {
        var order = await _orderManager.CreateAsync(input.CustomerId, input.Number);
        await _orderRepository.InsertAsync(order, autoSave: true);
        return _orderMapper.MapToDto(order);
    }

    public async Task<OrderDto> ChangePriceAsync(Guid id, ChangeOrderPriceDto input)
    {
        var order = await _orderRepository.GetAsync(id);
        order.SetPrice(input.NewPrice);          // Invariant enforced inside the entity
        await _orderRepository.UpdateAsync(order);
        return _orderMapper.MapToDto(order);
    }
}
```

(Optional) Mapperly mapper in `src/Acme.Sales.Application/Orders/OrderMapper.cs`:

```csharp
using Riok.Mapperly.Abstractions;

namespace Acme.Sales.Orders;

[Mapper]
public partial class OrderMapper
{
    public partial OrderDto MapToDto(Order order);
}
```

Register it in the Application module:

```csharp
public override void ConfigureServices(ServiceConfigurationContext context)
{
    context.Services.AddSingleton<OrderMapper>();
}
```

---

## What Goes Where (Layered template, ABP 10.3)

| Artifact | Project |
|---|---|
| `Order` entity, `OrderManager` | `Acme.Sales.Domain` |
| `IOrderAppService`, `CreateOrderDto`, `ChangeOrderPriceDto`, `OrderDto` | `Acme.Sales.Application.Contracts` |
| `OrderAppService`, `OrderMapper` | `Acme.Sales.Application` |
| Localization for `Sales:OrderNumberAlreadyExists`, `Sales:InvalidPrice` | `Acme.Sales.Domain.Shared/Localization/Sales/*.json` |
| EF Core `OrderRepository` (only if custom queries are added later) | `Acme.Sales.EntityFrameworkCore` |

Dependency direction is preserved: `Application -> Application.Contracts -> Domain -> Domain.Shared`. The domain layer references no application abstractions, no `DbContext`, and no foreign-aggregate repositories.

---

## Key Takeaways

- Domain services **construct/return** aggregates and enforce **cross-aggregate** rules; they do not call `InsertAsync` / `UpdateAsync` and do not use `autoSave`.
- Entity invariants (`newPrice <= 0`) live inside the entity (`Order.SetPrice`), not in the manager or app service.
- Don't inject another aggregate's repository to "validate" an Id you already have — reference by Id and let the FK / app-layer check do the rest.
- App services orchestrate: load aggregates, call the manager or entity methods, persist, map to DTOs.
