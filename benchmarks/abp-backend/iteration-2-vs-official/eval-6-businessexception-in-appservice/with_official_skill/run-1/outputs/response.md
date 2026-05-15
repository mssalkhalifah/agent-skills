# Code Review: `OrderAppService` (ABP 10.3)

The app service is doing three jobs that belong elsewhere: DTO-shape validation, uniqueness checking, and customer-state guards. Per the official ABP skills, the application layer should orchestrate use cases only — invariants belong in entities, cross-aggregate rules in a domain service / `*Manager`, and input format in DTO validation.

Skills cited:
- `abp-application-layer` — "Business logic in app service" is listed as an anti-pattern; "Domain rule: put validation in entity constructor or domain service"; data annotations are the canonical place for input-format checks.
- `abp-ddd` — Rich domain model (private setters + behavior methods), uniqueness shown explicitly in the `OrderManager.CreateAsync` example, and `*Manager` suffix for domain services.
- `abp-core` — `BusinessException` with namespaced error codes for domain rule violations; use base-class properties (`GuidGenerator`, `L`) instead of injecting; `EntityNotFoundException` for missing aggregates.

---

## Issues

### 1. Input-format validation lives in the app service
`Number` required + max length are pure DTO-shape rules. The `abp-application-layer` skill puts these directly on the DTO via DataAnnotations (see its `CreateBookDto` example with `[Required]` and `[StringLength]`). Throwing `BusinessException` for "field is empty" or "field is too long" is the wrong exception type — it should surface as `AbpValidationException` (HTTP 400), which DataAnnotations gives you automatically.

### 2. Uniqueness check belongs in a domain service (`OrderManager`)
The `abp-ddd` skill's `OrderManager.CreateAsync` example does exactly this rule: queries `_orderRepository.FindByOrderNumberAsync(orderNumber)` and throws `BusinessException("Orders:OrderNumberAlreadyExists")`. Uniqueness spans the aggregate boundary (it needs the repository to look at *other* `Order`s), so it cannot live on the `Order` entity — it must live in `OrderManager`.

### 3. `customer.IsBlocked` guard is a domain invariant
Whether a blocked customer can place an order is a business rule about `Customer`. Two acceptable homes per `abp-ddd`:
- A method on `Customer` (e.g., `EnsureCanPlaceOrder()`) that throws when blocked — keeps the rule with the entity that owns the state.
- Or, since the rule combines `Customer` + `Order` creation, enforce it inside `OrderManager.CreateAsync` after loading the customer.

### 4. `EntityNotFoundException`, not `BusinessException`, for a missing customer
Per `abp-application-layer` -> Error Handling, "Entity Not Found" maps to `EntityNotFoundException(typeof(Customer), id)` -> HTTP 404. `BusinessException` defaults to 403, which is semantically wrong for "row doesn't exist".

### 5. App service does too much
The whole point of the layered split (per `abp-application-layer` Application Service Structure) is: load aggregates, delegate to manager, persist, map. The current code mixes all four layers' concerns into one method.

### 6. Minor: `_customerRepository.FindAsync` — choose intentionally
If "customer missing" is an error (it is), use `GetAsync` (throws `EntityNotFoundException` automatically) and drop the manual null-check. `FindAsync` is for "maybe exists" flows.

---

## Refactored layout

### `CreateOrderDto` (Application.Contracts) — DataAnnotations replace the format checks

```csharp
public class CreateOrderDto
{
    [Required]
    [StringLength(32, MinimumLength = 1)]
    public string Number { get; set; } = default!;

    public Guid CustomerId { get; set; }
}
```

That single DTO change deletes the first two `BusinessException` throws (`OrderNumberRequired`, `OrderNumberTooLong`); ABP's validation interceptor converts annotation failures into `AbpValidationException` -> HTTP 400 automatically.

### `Customer` entity (Domain) — invariant lives with the data

```csharp
public class Customer : AggregateRoot<Guid>
{
    public bool IsBlocked { get; private set; }
    // ...

    public void EnsureCanPlaceOrder()
    {
        if (IsBlocked)
        {
            throw new BusinessException("Sales:CustomerIsBlocked")
                .WithData("CustomerId", Id);
        }
    }
}
```

### `IOrderRepository` (Domain) — custom query for the uniqueness check

```csharp
public interface IOrderRepository : IRepository<Order, Guid>
{
    Task<Order?> FindByNumberAsync(string number);
}
```

Per `abp-ddd` Repository Best Practices, define a custom repo interface only when you need a custom query — this is one of those times.

### `OrderManager` (Domain) — cross-aggregate rules and aggregate construction

```csharp
public class OrderManager : DomainService
{
    private readonly IOrderRepository _orderRepository;

    public OrderManager(IOrderRepository orderRepository)
    {
        _orderRepository = orderRepository;
    }

    public async Task<Order> CreateAsync(Customer customer, string number)
    {
        customer.EnsureCanPlaceOrder();

        if (await _orderRepository.FindByNumberAsync(number) != null)
        {
            throw new BusinessException("Sales:OrderNumberAlreadyExists")
                .WithData("Number", number);
        }

        return new Order(GuidGenerator.Create(), customer.Id, number);
    }
}
```

Notes matching the skills:
- `*Manager` suffix (`abp-ddd` Domain Service Best Practices).
- Accepts domain objects, not DTOs (`abp-ddd`).
- Uses `GuidGenerator` base-class property — not injected, not generated inside the entity constructor (`abp-ddd` anti-pattern: "Generating GUID in entity constructor").
- Throws `BusinessException` with namespaced code + `WithData(...)` (`abp-core` Business Exceptions).

### `OrderAppService` (Application) — orchestration only, **no `BusinessException` throws**

```csharp
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
        var customer = await _customerRepository.GetAsync(input.CustomerId);
        var order = await _orderManager.CreateAsync(customer, input.Number);
        await _orderRepository.InsertAsync(order, autoSave: true);
        return _orderMapper.MapToDto(order);
    }
}
```

What disappeared from the app service:
- Both shape checks -> DataAnnotations on the DTO.
- Customer null-check -> replaced by `GetAsync`, which throws `EntityNotFoundException` automatically (`abp-application-layer` Error Handling).
- `customer.IsBlocked` check -> `Customer.EnsureCanPlaceOrder()` inside `OrderManager`.
- Uniqueness check -> `OrderManager.CreateAsync`.
- `new Order(...)` construction -> `OrderManager.CreateAsync` (uses its own `GuidGenerator`).
- `ObjectMapper.Map<...>` -> `OrderMapper` (Mapperly is the default in ABP 10.3 per `abp-application-layer` Object Mapping; switch if your solution still uses AutoMapper).

Result: zero `BusinessException` throws in the app service. The exception model is preserved — `BusinessException` for domain rules, `EntityNotFoundException` for missing aggregates, `AbpValidationException` (auto) for malformed input.

---

## Localization wiring (don't forget)

`abp-core` requires namespaced codes to be mapped to a localization resource so the messages render properly:

```csharp
Configure<AbpExceptionLocalizationOptions>(options =>
{
    options.MapCodeNamespace("Sales", typeof(SalesResource));
});
```

And the JSON entries in `*.Domain.Shared/Localization/Sales/en.json`:

```json
{
  "Sales:CustomerIsBlocked": "The customer is blocked and cannot place new orders.",
  "Sales:OrderNumberAlreadyExists": "Order number '{Number}' is already in use."
}
```

---

## Summary checklist

| Concern | Before | After |
|---|---|---|
| Number required / max length | `BusinessException` in app svc | `[Required]`, `[StringLength(32)]` on DTO |
| Customer not found | `FindAsync` + `BusinessException` | `GetAsync` -> `EntityNotFoundException` |
| Customer blocked | `BusinessException` in app svc | `Customer.EnsureCanPlaceOrder()` called by `OrderManager` |
| Order number uniqueness | `AnyAsync` + `BusinessException` in app svc | `OrderManager.CreateAsync` via `IOrderRepository.FindByNumberAsync` |
| Order construction | `new Order(...)` in app svc | `OrderManager.CreateAsync` |
| Mapping | `ObjectMapper` | `OrderMapper` (Mapperly) |
| `BusinessException` in app svc | 5 throws | 0 throws |
