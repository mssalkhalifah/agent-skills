# Review: `OrderManager` Domain Service (ABP 10.3, Layered)

Skills consulted: **abp-ddd**, **abp-application-layer**, **abp-dependency-rules**, **abp-core**.

Yes — there are several issues. Some are clear ABP/DDD anti-patterns, others are layering/idiom problems. Below is a numbered list, followed by the corrected `OrderManager` (Domain project) and the orchestrating `OrderAppService` (Application project) that should own the workflow.

---

## Issues

1. **Persistence inside the domain service (biggest problem).**
   The manager calls `_orderRepository.InsertAsync(order, autoSave: true)` and `_orderRepository.UpdateAsync(order)`. Per `abp-ddd`, a domain service (a `*Manager`) **creates and returns** the aggregate; it does **not** persist it. Persistence (`InsertAsync` / `UpdateAsync`) and the unit of work are the **application service's** responsibility. The skill's canonical example:
   ```csharp
   var book = await _bookManager.CreateAsync(...);
   await _bookRepository.InsertAsync(book);
   ```

2. **`ChangePriceAsync(Guid orderId, ...)` takes an ID instead of the aggregate.**
   `abp-ddd` rule: *"Accept/return domain objects, not DTOs"* — and by extension, not bare IDs that force the manager to do its own load/save. Loading should be done by the app service; the manager mutates the entity that's passed in.

3. **`autoSave: true` is decided by the manager.**
   Save semantics / `UnitOfWork` flushing is an application-layer concern. Removing the explicit `InsertAsync`/`UpdateAsync` calls also removes this leak automatically.

4. **Price validation is in the wrong place.**
   `if (newPrice <= 0) throw new BusinessException(...)` lives in the manager, but `abp-ddd` says invariants belong inside the entity (Rich Domain Model: *"Entity enforces invariants"*). This check belongs inside `Order.SetPrice`, not the manager. The manager has no business calling `SetPrice` here — there's nothing cross-aggregate about a simple price change, so this method shouldn't be in the manager at all. The application service can call `order.SetPrice(...)` directly.

5. **`_customerRepository` is injected and the loaded customer is then discarded.**
   `var customer = await _customerRepository.GetAsync(customerId);` — the result is never used. If the intent is "verify the customer exists before creating an order," use a lighter call (`AnyAsync`). Either delete the dependency or actually use it. The corrected version below keeps it as a real existence guard.

6. **Uniqueness check via `AnyAsync(o => o.Number == ...)` on the generic repository.**
   This works but `abp-ddd` recommends a **custom repository method** (`IOrderRepository.FindByOrderNumberAsync`) when you have a recurring custom query. It also lets you swap in proper indexing/casing rules in the EF Core layer.

7. **No `.WithData(...)` on the `BusinessException`.**
   `abp-core` shows `BusinessException` should be enriched with namespaced data so the message can be localized with the offending value:
   ```csharp
   throw new BusinessException("Sales:OrderNumberAlreadyExists")
       .WithData("OrderNumber", orderNumber);
   ```

8. **Missing `using System;`** for `Guid` (only matters if the project doesn't have implicit/global usings — ABP 10.3 templates usually do; flagged for completeness).

9. **Parameter ordering / convention.**
   The skill's canonical `OrderManager.CreateAsync(string orderNumber, Guid customerId)` orders the natural key first. Minor, but worth aligning.

---

## Corrected Code

### Domain Project (`Acme.Sales.Domain`)

**File:** `Acme.Sales.Domain/Orders/IOrderRepository.cs`
```csharp
using System;
using System.Threading.Tasks;
using Volo.Abp.Domain.Repositories;

namespace Acme.Sales.Orders;

public interface IOrderRepository : IRepository<Order, Guid>
{
    Task<Order?> FindByOrderNumberAsync(string orderNumber);
}
```
(Implementation goes in `Acme.Sales.EntityFrameworkCore/Orders/OrderRepository.cs` — see `abp-ef-core` / `abp-dependency-rules`.)

**File:** `Acme.Sales.Domain/Orders/Order.cs` (the `SetPrice` invariant belongs here)
```csharp
using System;
using Volo.Abp;
using Volo.Abp.Domain.Entities.Auditing;

namespace Acme.Sales.Orders;

public class Order : FullAuditedAggregateRoot<Guid>
{
    public Guid CustomerId { get; private set; }
    public string Number { get; private set; } = default!;
    public decimal Price { get; private set; }

    protected Order() { } // For ORM

    internal Order(Guid id, Guid customerId, string number) : base(id)
    {
        CustomerId = customerId;
        Number = Check.NotNullOrWhiteSpace(number, nameof(number));
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

**File:** `Acme.Sales.Domain/Orders/OrderManager.cs` (the corrected domain service)
```csharp
using System;
using System.Threading.Tasks;
using Volo.Abp;
using Volo.Abp.Domain.Repositories;
using Volo.Abp.Domain.Services;

namespace Acme.Sales.Orders;

public class OrderManager : DomainService
{
    private readonly IOrderRepository _orderRepository;
    private readonly IRepository<Customer, Guid> _customerRepository;

    public OrderManager(
        IOrderRepository orderRepository,
        IRepository<Customer, Guid> customerRepository)
    {
        _orderRepository = orderRepository;
        _customerRepository = customerRepository;
    }

    public async Task<Order> CreateAsync(string number, Guid customerId)
    {
        Check.NotNullOrWhiteSpace(number, nameof(number));

        // Cross-aggregate existence check — actually used now.
        if (!await _customerRepository.AnyAsync(c => c.Id == customerId))
        {
            throw new BusinessException("Sales:CustomerNotFound")
                .WithData("CustomerId", customerId);
        }

        // Uniqueness invariant — uses the custom repository method.
        var existing = await _orderRepository.FindByOrderNumberAsync(number);
        if (existing != null)
        {
            throw new BusinessException("Sales:OrderNumberAlreadyExists")
                .WithData("OrderNumber", number);
        }

        // Build and return the aggregate. No InsertAsync here — that's the app service's job.
        return new Order(GuidGenerator.Create(), customerId, number);
    }
}
```

Notes on what was removed from the manager:
- `InsertAsync` / `UpdateAsync` / `autoSave: true` — moved to the app service.
- `ChangePriceAsync` — removed entirely. It only mutated one aggregate and enforced a simple invariant; both belong on `Order` (which now contains the validation in `SetPrice`). The application service mutates the loaded `Order` directly.
- `GetAsync` for an unused customer entity — replaced with a cheap `AnyAsync` existence check.

---

### Application Project (`Acme.Sales.Application`)

**File:** `Acme.Sales.Application.Contracts/Orders/IOrderAppService.cs`
```csharp
using System;
using System.Threading.Tasks;
using Volo.Abp.Application.Services;

namespace Acme.Sales.Orders;

public interface IOrderAppService : IApplicationService
{
    Task<OrderDto> CreateAsync(CreateOrderDto input);
    Task<OrderDto> UpdateAsync(Guid id, UpdateOrderPriceDto input);
}
```

**File:** `Acme.Sales.Application.Contracts/Orders/CreateOrderDto.cs`
```csharp
using System;
using System.ComponentModel.DataAnnotations;

namespace Acme.Sales.Orders;

public class CreateOrderDto
{
    [Required]
    [StringLength(64, MinimumLength = 1)]
    public string Number { get; set; } = default!;

    [Required]
    public Guid CustomerId { get; set; }
}
```

**File:** `Acme.Sales.Application.Contracts/Orders/UpdateOrderPriceDto.cs`
```csharp
namespace Acme.Sales.Orders;

public class UpdateOrderPriceDto
{
    public decimal Price { get; set; }
}
```

**File:** `Acme.Sales.Application.Contracts/Orders/OrderDto.cs`
```csharp
using System;
using Volo.Abp.Application.Dtos;

namespace Acme.Sales.Orders;

public class OrderDto : EntityDto<Guid>
{
    public Guid CustomerId { get; set; }
    public string Number { get; set; } = default!;
    public decimal Price { get; set; }
}
```

**File:** `Acme.Sales.Application/Orders/OrderAppService.cs` (orchestrator that owns persistence)
```csharp
using System;
using System.Threading.Tasks;
using Volo.Abp.Application.Services;

namespace Acme.Sales.Orders;

public class OrderAppService : ApplicationService, IOrderAppService
{
    private readonly IOrderRepository _orderRepository;
    private readonly OrderManager _orderManager;
    private readonly OrderMapper _orderMapper;

    public OrderAppService(
        IOrderRepository orderRepository,
        OrderManager orderManager,
        OrderMapper orderMapper)
    {
        _orderRepository = orderRepository;
        _orderManager = orderManager;
        _orderMapper = orderMapper;
    }

    public async Task<OrderDto> CreateAsync(CreateOrderDto input)
    {
        // Domain service builds the aggregate and enforces cross-aggregate rules.
        var order = await _orderManager.CreateAsync(input.Number, input.CustomerId);

        // Application service owns persistence + UoW.
        await _orderRepository.InsertAsync(order);

        return _orderMapper.MapToDto(order);
    }

    public async Task<OrderDto> UpdateAsync(Guid id, UpdateOrderPriceDto input)
    {
        var order = await _orderRepository.GetAsync(id);

        // Invariant is enforced inside the entity, not in the manager.
        order.SetPrice(input.Price);

        await _orderRepository.UpdateAsync(order);
        return _orderMapper.MapToDto(order);
    }
}
```

**File:** `Acme.Sales.Application/Orders/OrderMapper.cs` (Mapperly — the ABP 10.3 default)
```csharp
using Riok.Mapperly.Abstractions;

namespace Acme.Sales.Orders;

[Mapper]
public partial class OrderMapper
{
    public partial OrderDto MapToDto(Order order);
}
```
Register in `SalesApplicationModule.ConfigureServices`:
```csharp
context.Services.AddSingleton<OrderMapper>();
```

---

## Summary of Layer Placement (per `abp-dependency-rules`)

| Concern | Project |
|---|---|
| `Order` entity, `Order.SetPrice` invariant | `Acme.Sales.Domain` |
| `IOrderRepository` interface | `Acme.Sales.Domain` |
| `OrderManager` (no persistence, cross-aggregate rules only) | `Acme.Sales.Domain` |
| `OrderRepository` implementation | `Acme.Sales.EntityFrameworkCore` |
| `IOrderAppService`, `CreateOrderDto`, `OrderDto` | `Acme.Sales.Application.Contracts` |
| `OrderAppService`, `OrderMapper` (owns `InsertAsync`/`UpdateAsync`) | `Acme.Sales.Application` |
| Localization keys (`Sales:OrderNumberAlreadyExists`, `Sales:CustomerNotFound`, `Sales:InvalidPrice`) | `Acme.Sales.Domain.Shared/Localization/Sales/en.json` |

---

## Cited Skills
- `abp-ddd` — domain service responsibilities, rich domain model, `*Manager` naming, custom repository for custom queries, no persistence in managers.
- `abp-application-layer` — app service orchestrates manager + repository + mapper; persistence and UoW belong here; Mapperly is the default mapping provider in ABP 10.3.
- `abp-dependency-rules` — Domain vs Application vs EF Core project placement; interfaces in Domain, implementations in EF Core.
- `abp-core` — `BusinessException` with `.WithData(...)`, `GuidGenerator` from base class, `Check.NotNullOrWhiteSpace`.
