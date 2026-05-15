# Review of `OrderManager` (ABP 10.3 Layered)

**Skills consulted:** `abp-ddd`, `abp-application-layer`, `abp-dependency-rules`, `abp-core`, `abp-development-flow`.

Yes — several things are wrong. The manager mixes concerns that should live in the **entity**, the **application service**, and a **different aggregate's bounded context**. Below is a precise list followed by the corrected code, split into the correct projects.

---

## Issues

### 1. Cross-aggregate repository injection (`IRepository<Customer, Guid>`)
`OrderManager` lives in the `Sales` namespace and is the manager for the `Order` aggregate. Injecting a repository for the `Customer` aggregate root violates the DDD guidance from `abp-ddd`:

> "Navigation properties to other aggregates: reference by Id only, never add full navigation properties across aggregates."

A repository to another aggregate is the moral equivalent of a navigation property — it pulls another aggregate into this aggregate's consistency boundary. Worse, in the submitted code `customer` is fetched and then thrown away (it is never read). This is dead code and a clear smell.

If you need to assert "the customer exists" before creating an order, that check belongs in the **application service** (which is allowed to orchestrate across aggregates) or in an integration service published by the Customer module. Domain services should restrict themselves to aggregates and repositories inside their own bounded context.

### 2. Domain service is persisting (`InsertAsync(... autoSave: true)`)
Per `abp-ddd`, the canonical `OrderManager.CreateAsync` example **constructs and returns** the `Order` — it does not insert it. Persistence is the application service's job:

```csharp
// Domain service (from abp-ddd)
public async Task<Order> CreateAsync(string orderNumber, Guid customerId)
{
    // ...uniqueness check...
    return new Order(GuidGenerator.Create(), orderNumber, customerId);
}
```

```csharp
// Application service (from abp-application-layer)
var book = await _bookManager.CreateAsync(input.Name, input.Price);
await _bookRepository.InsertAsync(book);
```

Forcing `autoSave: true` inside the manager also defeats ABP's automatic Unit of Work — the app service's UoW would have flushed naturally at the end of the request.

### 3. `ChangePriceAsync` does not belong in a domain service at all
`abp-ddd` says: use a domain service for logic that "spans multiple aggregates" or "requires repository queries to enforce rules." `ChangePriceAsync` does neither. The price validation (`newPrice <= 0`) is an **entity invariant** and must live inside `Order.SetPrice` (rich domain model — `abp-ddd` "Entity Best Practices"). The fetch + `UpdateAsync` belongs in the application service.

### 4. `BusinessException` is missing structured data
`abp-core` and `abp-ddd` both show:

```csharp
throw new BusinessException("Sales:OrderNumberAlreadyExists")
    .WithData("OrderNumber", number);
```

Use `.WithData(...)` so the message can be localized with the offending value.

### 5. Custom repository preferred when a custom query exists
The uniqueness check is `AnyAsync(o => o.Number == number)`. That's fine on `IRepository<Order, Guid>`, but per `abp-ddd` "When to Use Custom Repository", once you have a meaningful domain query like "find by order number", encapsulating it on `IOrderRepository.FindByOrderNumberAsync` is the recommended pattern. The corrected version uses the custom interface to match the official example. If you want to keep the generic repository, `AnyAsync` is acceptable.

### 6. Minor: parameter order and convention
- Official example orders `(orderNumber, customerId)`.
- Domain services use `*Manager` suffix (your name is fine).
- `using System;` should be present for `Guid` (file-scoped namespace without global usings).

---

## Corrected code

### `Acme.Sales.Domain/Orders/IOrderRepository.cs`

```csharp
using System;
using System.Threading.Tasks;
using Volo.Abp.Domain.Repositories;

namespace Acme.Sales.Orders;

public interface IOrderRepository : IRepository<Order, Guid>
{
    Task<Order?> FindByNumberAsync(string number);
}
```

> Implementation goes in `Acme.Sales.EntityFrameworkCore` (`EfCoreRepository<SalesDbContext, Order, Guid>, IOrderRepository`) — `abp-dependency-rules` table: repository **interface** in Domain, **implementation** in the data layer.

### `Acme.Sales.Domain/Orders/Order.cs` (relevant excerpts — entity owns its invariants)

```csharp
using System;
using Volo.Abp;
using Volo.Abp.Domain.Entities;

namespace Acme.Sales.Orders;

public class Order : AggregateRoot<Guid>
{
    public string Number { get; private set; } = default!;
    public Guid CustomerId { get; private set; }
    public decimal Price { get; private set; }

    protected Order() { } // For ORM

    internal Order(Guid id, Guid customerId, string number) : base(id)
    {
        CustomerId = customerId;
        Number = Check.NotNullOrWhiteSpace(number, nameof(number));
    }

    public void SetPrice(decimal price)
    {
        if (price <= 0)
        {
            throw new BusinessException("Sales:InvalidPrice")
                .WithData("Price", price);
        }

        Price = price;
    }
}
```

### `Acme.Sales.Domain/Orders/OrderManager.cs` (corrected domain service)

```csharp
using System;
using System.Threading.Tasks;
using Volo.Abp;
using Volo.Abp.Domain.Services;

namespace Acme.Sales.Orders;

public class OrderManager : DomainService
{
    private readonly IOrderRepository _orderRepository;

    public OrderManager(IOrderRepository orderRepository)
    {
        _orderRepository = orderRepository;
    }

    public async Task<Order> CreateAsync(Guid customerId, string number)
    {
        // Domain rule: order numbers are unique within Sales.
        if (await _orderRepository.FindByNumberAsync(number) is not null)
        {
            throw new BusinessException("Sales:OrderNumberAlreadyExists")
                .WithData("OrderNumber", number);
        }

        // Construct only — persistence is the application service's job.
        return new Order(GuidGenerator.Create(), customerId, number);
    }
}
```

Key changes vs. the original:

- Removed `IRepository<Customer, Guid>` — the manager no longer reaches into another aggregate.
- Removed `InsertAsync`/`UpdateAsync` from the manager.
- Removed `ChangePriceAsync` — that's now a method on `Order` (entity), called from the app service.
- Uses `IOrderRepository.FindByNumberAsync` instead of an ad-hoc `AnyAsync` lambda.
- `BusinessException.WithData(...)`.

### `Acme.Sales.Application.Contracts/Orders/IOrderAppService.cs`

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

### `Acme.Sales.Application.Contracts/Orders/CreateOrderDto.cs`

```csharp
using System;
using System.ComponentModel.DataAnnotations;

namespace Acme.Sales.Orders;

public class CreateOrderDto
{
    [Required]
    public Guid CustomerId { get; set; }

    [Required]
    [StringLength(64)]
    public string Number { get; set; } = default!;
}

public class ChangeOrderPriceDto
{
    [Range(0.01, double.MaxValue)]
    public decimal NewPrice { get; set; }
}

public class OrderDto : Volo.Abp.Application.Dtos.EntityDto<Guid>
{
    public Guid CustomerId { get; set; }
    public string Number { get; set; } = default!;
    public decimal Price { get; set; }
}
```

> Note from `abp-application-layer`: "ID inside UpdateDto" is an anti-pattern — `id` is passed as a separate parameter to `ChangePriceAsync`.

### `Acme.Sales.Application/Orders/OrderAppService.cs` (orchestrates across aggregates)

```csharp
using System;
using System.Threading.Tasks;
using Volo.Abp;
using Volo.Abp.Application.Services;
using Volo.Abp.Domain.Entities;
using Volo.Abp.Domain.Repositories;
using Acme.Sales.Customers; // Customer aggregate lives here

namespace Acme.Sales.Orders;

public class OrderAppService : ApplicationService, IOrderAppService
{
    private readonly IOrderRepository _orderRepository;
    private readonly IRepository<Customer, Guid> _customerRepository;
    private readonly OrderManager _orderManager;
    private readonly OrderMapper _orderMapper;

    public OrderAppService(
        IOrderRepository orderRepository,
        IRepository<Customer, Guid> customerRepository,
        OrderManager orderManager,
        OrderMapper orderMapper)
    {
        _orderRepository = orderRepository;
        _customerRepository = customerRepository;
        _orderManager = orderManager;
        _orderMapper = orderMapper;
    }

    public async Task<OrderDto> CreateAsync(CreateOrderDto input)
    {
        // Cross-aggregate existence check belongs HERE, not in OrderManager.
        if (!await _customerRepository.AnyAsync(c => c.Id == input.CustomerId))
        {
            throw new EntityNotFoundException(typeof(Customer), input.CustomerId);
        }

        var order = await _orderManager.CreateAsync(input.CustomerId, input.Number);
        await _orderRepository.InsertAsync(order);

        return _orderMapper.MapToDto(order);
    }

    public async Task<OrderDto> ChangePriceAsync(Guid id, ChangeOrderPriceDto input)
    {
        var order = await _orderRepository.GetAsync(id);
        order.SetPrice(input.NewPrice); // entity enforces the invariant
        await _orderRepository.UpdateAsync(order);

        return _orderMapper.MapToDto(order);
    }
}
```

### `Acme.Sales.Application/Orders/OrderMapper.cs` (Mapperly — current ABP default)

```csharp
using Riok.Mapperly.Abstractions;

namespace Acme.Sales.Orders;

[Mapper]
public partial class OrderMapper
{
    public partial OrderDto MapToDto(Order order);
}
```

Register in `SalesApplicationModule`:

```csharp
context.Services.AddSingleton<OrderMapper>();
```

---

## File-location summary (Layered template — `abp-dependency-rules`)

| File | Project |
|---|---|
| `Order.cs`, `IOrderRepository.cs`, `OrderManager.cs` | `Acme.Sales.Domain` |
| `OrderRepository.cs` (EF Core impl) | `Acme.Sales.EntityFrameworkCore` |
| `IOrderAppService.cs`, `CreateOrderDto.cs`, `ChangeOrderPriceDto.cs`, `OrderDto.cs` | `Acme.Sales.Application.Contracts` |
| `OrderAppService.cs`, `OrderMapper.cs` | `Acme.Sales.Application` |
| `"Sales:OrderNumberAlreadyExists"`, `"Sales:InvalidPrice"` localization keys | `Acme.Sales.Domain.Shared/Localization/...` |

---

## TL;DR of corrections

1. **Drop `IRepository<Customer, Guid>` from `OrderManager`** — cross-aggregate; move the customer check to `OrderAppService`.
2. **Manager constructs, AppService persists** — remove `InsertAsync(autoSave: true)` from the manager; the app service calls `_orderRepository.InsertAsync(order)`.
3. **Move price validation into `Order.SetPrice`** (entity invariant). Delete `OrderManager.ChangePriceAsync`; the app service calls `order.SetPrice(...)` then `UpdateAsync`.
4. **Use a custom `IOrderRepository.FindByNumberAsync`** for the uniqueness query.
5. **`BusinessException` + `.WithData(...)`** with namespaced codes.
6. Pass `id` as a separate parameter from update DTOs; localized error codes belong in `Domain.Shared`.
