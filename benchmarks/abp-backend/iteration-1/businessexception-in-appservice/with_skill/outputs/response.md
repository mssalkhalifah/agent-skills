# Code review: `OrderAppService.CreateAsync`

The class works, but it's quietly violating two of ABP's stronger conventions and one input-validation best practice. There are five issues, and they all push the code in the same direction: move the rules out of the application service.

## What's wrong

### 1. Input validation is being thrown by hand instead of going through ABP's validation pipeline (Rule 7)

These two checks:

```csharp
if (string.IsNullOrWhiteSpace(input.Number))
    throw new BusinessException("Sales:OrderNumberRequired");
if (input.Number.Length > 32)
    throw new BusinessException("Sales:OrderNumberTooLong");
```

…aren't business rules. They're *input shape* — required field, max length. ABP's validation pipeline is built for exactly this: `[Required]` / `[StringLength]` on the DTO are picked up by `DataAnnotationObjectValidationContributor`, the framework throws `AbpValidationException` *before* your method body runs, and the ABP exception filter converts it to a structured HTTP 400 response automatically. You get correct status codes (400 vs. the 403 that `BusinessException` produces), structured per-field error payloads for the client, and the service body never sees the bad input.

`ApplicationService` already implements `IValidationEnabled`, so this works automatically — provided the method is `virtual` (or invoked through `IOrderAppService`, which it always is when ABP resolves it through DI).

### 2. The remaining `BusinessException` throws don't belong in the application service (Rule 7)

```csharp
if (customer.IsBlocked) throw new BusinessException("Sales:CustomerIsBlocked");
if (await _orderRepository.AnyAsync(o => o.Number == input.Number))
    throw new BusinessException("Sales:OrderNumberAlreadyExists");
```

These *are* business rules — they're invariants of the `Order` aggregate ("a blocked customer can't place orders", "order numbers are unique"). But putting them in the app service couples the rule to this single caller. Tomorrow when a background job, an integration service, or a domain event handler also creates orders, none of them will get these checks unless every author remembers to copy them. ABP's DDD guidance is clear: invariants live where they're enforced — in the entity, an entity behavior method, or a domain service (`*Manager`).

The right placement, in priority order:

- **`customer.IsBlocked` check** → an entity behavior method on `Customer`, e.g. `customer.EnsureCanReceiveNewOrder()`. The rule reads only the customer's own state, so the entity is the strongest place — every caller that routes through the entity gets it for free.
- **Order-number uniqueness** → an `OrderManager` (domain service). Uniqueness needs a repository check (`_orderRepository.AnyAsync(...)`), so it can't live on the entity. ABP's own [Domain Services best-practices doc](https://abp.io/docs/10.3/framework/architecture/best-practices/domain-services) shows this exact shape — a manager throwing `BusinessException` after a repository check.

The `CustomerNotFound` case is actually fine to surface as `EntityNotFoundException` — call `_customerRepository.GetAsync(id)` (which throws it for you) instead of `FindAsync(id)` + null check.

### 3. The application service is doing the domain service's job (Rule 3)

There's no `OrderManager` here. The app service is fetching the customer, evaluating cross-aggregate rules, checking uniqueness, and constructing the `Order`. That's all domain-service work. The app service should orchestrate: mint the Id, get the entity from the manager, persist.

### 4. `IRepository<Customer, Guid>` injected into the app service is a smell (Rule 3)

The app service writes orders, not customers. It's reading the customer only to feed the rule. That read belongs to the `OrderManager` via `IReadOnlyRepository<Customer, Guid>` — narrower interface, no-tracking by default, and it signals intent ("I'm only reading"). The app service then keeps just the tracking `IRepository<Order, Guid>` for the side it actually mutates.

### 5. Method should be `virtual`

Without `virtual` (or interface-only invocation, which ABP does provide here), Castle DynamicProxy can't intercept the call and the validation pipeline silently doesn't run. Easy thing to forget; safest to mark it.

## Cleaned-up version

**`CreateOrderDto`** — input validation moves here:

```csharp
using System.ComponentModel.DataAnnotations;

public class CreateOrderDto
{
    [Required]
    public Guid CustomerId { get; set; }

    [Required]
    [StringLength(32)]
    public string Number { get; set; } = default!;
}
```

**`Customer`** — entity behavior owns the self-state rule:

```csharp
public class Customer : FullAuditedAggregateRoot<Guid>
{
    public bool IsBlocked { get; private set; }
    // ... other state ...

    public void EnsureCanReceiveNewOrder()
    {
        if (IsBlocked)
            throw new BusinessException("Sales:CustomerIsBlocked");
    }
}
```

**`OrderManager`** — domain service owns the cross-aggregate / repository-driven rule, and crucially does **not** persist:

```csharp
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

    public async Task<Order> CreateAsync(Guid id, Guid customerId, string number)
    {
        var customer = await _customerRepository.GetAsync(customerId);   // throws EntityNotFoundException
        customer.EnsureCanReceiveNewOrder();                              // entity-behavior throw

        if (await _orderRepository.AnyAsync(o => o.Number == number))
            throw new BusinessException("Sales:OrderNumberAlreadyExists");

        return new Order(id, customerId, number);                         // returned, NOT persisted
    }
}
```

**`OrderAppService`** — pure orchestration:

```csharp
public class OrderAppService : ApplicationService, IOrderAppService
{
    private readonly OrderManager             _orderManager;
    private readonly IRepository<Order, Guid> _orderRepository;   // tracking lives HERE

    public OrderAppService(
        OrderManager             orderManager,
        IRepository<Order, Guid> orderRepository)
    {
        _orderManager = orderManager;
        _orderRepository = orderRepository;
    }

    public virtual async Task<OrderDto> CreateAsync(CreateOrderDto input)
    {
        // No imperative input checks: AbpValidationException already fired
        // before we got here if Number was missing or too long.

        var order = await _orderManager.CreateAsync(
            GuidGenerator.Create(),
            input.CustomerId,
            input.Number);

        await _orderRepository.InsertAsync(order, autoSave: true);
        return ObjectMapper.Map<Order, OrderDto>(order);
    }
}
```

## Why each move pays off

| Change | Rule | Benefit |
|---|---|---|
| `[Required]` / `[StringLength]` on the DTO | Rule 7 (input validation pipeline) | HTTP 400 with structured per-field errors, automatically; service body never sees invalid input. |
| `customer.EnsureCanReceiveNewOrder()` on the entity | Rule 7 (entity owns self-state rules) | Every future caller — jobs, event handlers, integration services — gets the rule for free. |
| Uniqueness check moved into `OrderManager` | Rule 7 (Manager owns repository-driven invariants) | Same enforcement regardless of caller; the rule lives next to the data it guards. |
| `OrderManager` injects `IReadOnlyRepository<…>` only | Rule 3 (Manager doesn't persist) | Compile-time signal that the manager doesn't mutate; no-tracking reads are also faster. |
| `_orderRepository.GetAsync` instead of `FindAsync` + null | DDD convention | `EntityNotFoundException` → 404 automatically; one fewer hand-written `BusinessException`. |
| App service holds the tracking `IRepository<Order, Guid>` and persists after Manager returns | Rule 3 (orchestration boundary) | Clean separation: Manager makes a valid `Order`, app service decides when/how to save it. |
| `virtual` on `CreateAsync` | Validation pipeline prerequisite | Guarantees DynamicProxy can intercept and run validation. |

One final note: if `CustomerNotFound` is genuinely a *business* condition for you (not just "wrong id, 404"), keep the explicit `BusinessException` throw — but still inside the `OrderManager`, not the app service.

After this refactor, `OrderAppService.CreateAsync` is three lines of orchestration, which is exactly the shape ABP is designed around.
