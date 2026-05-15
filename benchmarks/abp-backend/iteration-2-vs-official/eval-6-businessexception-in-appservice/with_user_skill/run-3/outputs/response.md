# Code review — `OrderAppService.CreateAsync`

The current method does three different jobs in one body: shape-validates the DTO, enforces a cross-aggregate invariant (customer must not be blocked), enforces a uniqueness invariant (order number), and orchestrates persistence. Per ABP 10.3 / DDD guidance (Rule 7 and Rule 3 of the abp-backend skill), an application service should orchestrate only — invariants live on the entity or in a `*Manager`, and shape/required/length checks belong on the DTO via ABP's validation pipeline. The refactor below preserves the existing exception-based error model (still `BusinessException` with the same `Sales:*` codes), just moved to the layer that owns each rule.

---

## Issues, in priority order

1. **`Number` required + length check belong on the DTO, not in the method body.** `string.IsNullOrWhiteSpace` + `Length > 32` are stock data-annotations. ABP's validation pipeline runs them automatically for `ApplicationService` methods (`ApplicationService` implements `IValidationEnabled`) and converts failures into `AbpValidationException` -> HTTP 400 with a structured payload. Imperative throws in the body bypass that structured response and force every caller (controller, background job, integration service) to re-check.
2. **Order-number uniqueness is a domain invariant — move it into `OrderManager`.** It requires a repository check (`AnyAsync`), which is exactly the Rule 3 trigger for a domain service. If a future background job or integration service also creates orders, putting the check in the AppService lets them bypass it; putting it in the Manager guarantees every caller routes through the same enforcement.
3. **`customer.IsBlocked` should be an entity behavior method.** The rule reads only the customer's own state — that's the strongest form per Rule 7 priority 1. Expose `Customer.EnsureCanReceiveNewOrder()` (throws `BusinessException("Sales:CustomerIsBlocked")`) so any caller that touches a `Customer` gets the rule, not just this one path.
4. **"Customer not found" is not a business invariant — it's a missing entity lookup.** Use `GetAsync(id)` from the read-only repository: it throws `EntityNotFoundException` automatically, which ABP serializes to HTTP 404. The current `BusinessException("Sales:CustomerNotFound")` collapses 404 into a generic 4xx with a custom code. Switching to `GetAsync` is both more idiomatic and removes one throw from the AppService.
5. **`_customerRepository` should be `IReadOnlyRepository<Customer, Guid>` here (and inside the Manager).** This AppService never mutates a `Customer`. Per Rule 2, pick the narrowest repository — `IReadOnlyRepository` signals intent and uses no-tracking queries by default.
6. **Method is not `virtual`.** Validation runs through Castle DynamicProxy interception. Make `CreateAsync` virtual (or rely on interface dispatch through `IOrderAppService`) so the pipeline can intercept.
7. **No `[Authorize]`.** Per Rule 4, the declarative authorization attribute belongs on the AppService method, not the controller — every caller (HTTP, jobs, integration services, internal callers) routes through it. Add `[Authorize(OrderPermissions.Create)]`.

---

## Refactored layout

### 1. DTO — shape validation moves here

`Acme.Sales.Application.Contracts/Orders/CreateOrderDto.cs`

```csharp
using System;
using System.ComponentModel.DataAnnotations;

namespace Acme.Sales.Orders;

public class CreateOrderDto
{
    [Required]
    public Guid CustomerId { get; set; }

    [Required]
    [StringLength(32)]
    public string Number { get; set; } = default!;
}
```

`[Required]` covers the whitespace/null case; `[StringLength(32)]` covers the length cap. Both produce structured `AbpValidationException` -> HTTP 400. No service-body throws needed. If the team wants cross-property rules later, implement `IValidatableObject` on the DTO — still no body throws.

### 2. Entity behavior — the "self-only" rule moves here

`Acme.Sales.Domain/Customers/Customer.cs`

```csharp
using Volo.Abp;
using Volo.Abp.Domain.Entities;

namespace Acme.Sales.Customers;

public class Customer : AggregateRoot<Guid>
{
    public bool IsBlocked { get; private set; }
    // ...other state...

    public void EnsureCanReceiveNewOrder()
    {
        if (IsBlocked)
        {
            throw new BusinessException("Sales:CustomerIsBlocked");
        }
    }
}
```

Every code path that needs to assert "this customer can take a new order" now calls one method — the rule cannot drift.

### 3. Domain service — uniqueness + entity-behavior call live here

`Acme.Sales.Domain/Orders/OrderManager.cs`

```csharp
using System;
using System.Threading.Tasks;
using Acme.Sales.Customers;
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
        IReadOnlyRepository<Order, Guid> orderRepository)
    {
        _customerRepository = customerRepository;
        _orderRepository    = orderRepository;
    }

    public async Task<Order> CreateAsync(Guid id, Guid customerId, string number)
    {
        var customer = await _customerRepository.GetAsync(customerId); // 404 if missing
        customer.EnsureCanReceiveNewOrder();                            // entity-behavior throws

        if (await _orderRepository.AnyAsync(o => o.Number == number))
        {
            throw new BusinessException("Sales:OrderNumberAlreadyExists");
        }

        return new Order(id, customerId, number); // Manager doesn't persist
    }
}
```

Note: `IReadOnlyRepository<T, TKey>` only — per Rule 3, the Manager never injects the tracking `IRepository<T, TKey>` and never persists.

### 4. Application service — orchestration only, no `BusinessException`

`Acme.Sales.Application/Orders/OrderAppService.cs`

```csharp
using System;
using System.Threading.Tasks;
using Acme.Sales.Permissions;
using Microsoft.AspNetCore.Authorization;
using Volo.Abp.Application.Services;
using Volo.Abp.Domain.Repositories;

namespace Acme.Sales.Orders;

public class OrderAppService : ApplicationService, IOrderAppService
{
    private readonly OrderManager             _orderManager;
    private readonly IRepository<Order, Guid> _orderRepository;

    public OrderAppService(
        OrderManager orderManager,
        IRepository<Order, Guid> orderRepository)
    {
        _orderManager    = orderManager;
        _orderRepository = orderRepository;
    }

    [Authorize(SalesPermissions.Orders.Create)]
    public virtual async Task<OrderDto> CreateAsync(CreateOrderDto input)
    {
        var order = await _orderManager.CreateAsync(
            GuidGenerator.Create(),     // inherited from ApplicationService — no IGuidGenerator field
            input.CustomerId,
            input.Number);

        await _orderRepository.InsertAsync(order, autoSave: true);
        return ObjectMapper.Map<Order, OrderDto>(order);
    }
}
```

Zero `BusinessException` throws. Zero shape validation. The method is now: mint Id -> hand to Manager -> persist -> map. That is the orchestration contract.

---

## Where each original throw ended up

| Original throw | New location | Why |
|---|---|---|
| `Sales:OrderNumberRequired` | `[Required]` on `CreateOrderDto.Number` | Shape validation — DTO + ABP pipeline -> 400 |
| `Sales:OrderNumberTooLong` | `[StringLength(32)]` on `CreateOrderDto.Number` | Shape validation — DTO + ABP pipeline -> 400 |
| `Sales:CustomerNotFound` | `IReadOnlyRepository<Customer, Guid>.GetAsync(...)` in `OrderManager` | Missing entity is `EntityNotFoundException` -> 404, not a business rule |
| `Sales:CustomerIsBlocked` | `Customer.EnsureCanReceiveNewOrder()` (entity behavior) | Rule reads only the entity's own state — Rule 7 priority 1 |
| `Sales:OrderNumberAlreadyExists` | `OrderManager.CreateAsync` | Needs a repository check — Rule 7 priority 2 (domain service) |

Same exception-based contract, same error codes; just moved to the layer that owns each rule.

---

## Smaller follow-ups (worth doing while you're in there)

- **Localize the codes.** `BusinessException("Sales:OrderNumberAlreadyExists")` should resolve to a localized message via the `Sales` resource (`Localization/Sales/en.json`). Add the keys when you move the throws.
- **`Order` aggregate constructor visibility.** If `Order` is currently `public Order(Guid id, Guid customerId, string number)` and the only valid creation path is through `OrderManager`, mark it `internal` (with `[assembly: InternalsVisibleTo("Acme.Sales.Domain.Tests")]` if you unit-test the constructor). That removes "skip the Manager and `new Order(...)` directly" as a possibility — see the Common Mistakes note on `public` aggregate constructors in the DDD reference.
- **Permission constant.** `SalesPermissions.Orders.Create` lives in `*.Application.Contracts` alongside `IOrderAppService`, registered via a `SalesPermissionDefinitionProvider`. Don't use role strings.
- **Controller stays thin.** If you're on explicit controllers (Rule 4), `OrderController` injects only `IOrderAppService` and delegates; no `[Authorize]` repetition on the controller action — it's already on the AppService method.
- **Make `CreateAsync` `virtual`** (or rely on interface dispatch) so DynamicProxy can intercept for validation, UoW, and authorization. Without `virtual`, the validation pipeline silently no-ops.
