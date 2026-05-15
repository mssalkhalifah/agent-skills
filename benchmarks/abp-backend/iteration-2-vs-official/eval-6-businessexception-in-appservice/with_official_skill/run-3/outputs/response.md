# Code Review: `OrderAppService` (ABP 10.3)

## Summary of Issues

Your `CreateAsync` is doing three jobs that don't belong in an application service:

1. **Input format validation** (`Number` required, `Number.Length > 32`) — these are DTO-level concerns, not domain rules.
2. **Cross-aggregate uniqueness check** (`Number` must be unique across orders) — this is a domain invariant; it belongs in a domain service (`OrderManager`), not the app service.
3. **Entity-state rule** (`customer.IsBlocked` cannot place orders) — this is either an invariant on `Customer` or behavior on `OrderManager`, not on `OrderAppService`.

The application service should orchestrate use cases, not enforce business rules. Per `abp-application-layer`:

> **Business logic in app service**: put it in domain entities or domain services. (`abp-application-layer/SKILL.md`, Anti-Patterns section)

And per `abp-ddd`:

> Use domain services for business logic that **spans multiple aggregates** **or requires repository queries to enforce rules**. (`abp-ddd/SKILL.md`)

The order-number uniqueness check is the textbook example given in the official skill — it even uses your exact code path (`Orders:OrderNumberAlreadyExists`).

We keep the **exception-based error model** (no Result<T>); we simply move throws to where the rule lives. `BusinessException` stays — it just disappears from the app service.

---

## 1. Input validation → Data Annotations on the DTO

`Number` required and `Number.Length > 32` are format rules, not business rules. They must be expressed as DTO validation so ABP rejects the request with `AbpValidationException` (HTTP 400) before the app service is even entered. Per `abp-application-layer/SKILL.md` (Validation section):

> **Application rule**: Use DTO validation (input format, required fields)
> **Domain rule**: Put validation in entity constructor or domain service (enforces business invariants)

**`CreateOrderDto`** (in `*.Application.Contracts`):

```csharp
public class CreateOrderDto
{
    [Required]
    [StringLength(32, MinimumLength = 1)]
    public string Number { get; set; } = default!;

    [Required]
    public Guid CustomerId { get; set; }
}
```

This removes the first two `BusinessException` throws (`Sales:OrderNumberRequired`, `Sales:OrderNumberTooLong`) entirely. ABP validates the DTO via its built-in validation interceptor.

---

## 2. Uniqueness → `OrderManager` (domain service)

Order-number uniqueness requires a repository query to enforce — that's a domain rule per `abp-ddd/SKILL.md`. Move it to a domain service named `OrderManager` (the `*Manager` suffix convention from the same skill).

Add a custom repository method so the manager doesn't leak query details (also per `abp-ddd/SKILL.md`, Repository Pattern):

```csharp
// *.Domain
public interface IOrderRepository : IRepository<Order, Guid>
{
    Task<bool> NumberExistsAsync(string number);
}
```

**`OrderManager`** (in `*.Domain`) — mirrors the example in `abp-ddd/SKILL.md` almost verbatim:

```csharp
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

    public async Task<Order> CreateAsync(Guid customerId, string number)
    {
        var customer = await _customerRepository.FindAsync(customerId);
        if (customer == null)
        {
            throw new EntityNotFoundException(typeof(Customer), customerId);
        }

        customer.EnsureCanPlaceOrders(); // see section 3

        if (await _orderRepository.NumberExistsAsync(number))
        {
            throw new BusinessException("Sales:OrderNumberAlreadyExists")
                .WithData("Number", number);
        }

        return new Order(GuidGenerator.Create(), customerId, number);
    }
}
```

Notes:
- Uses `GuidGenerator` from `DomainService` base class (no injection needed — `abp-core/SKILL.md`).
- Customer not-found becomes `EntityNotFoundException`, which is the idiomatic mapping per `abp-application-layer/SKILL.md` (Error Handling table → HTTP 404).
- `BusinessException` survives — we keep the exception model, we just relocated the throw.

---

## 3. `IsBlocked` → Behavior on the `Customer` entity

`customer.IsBlocked` being a publicly read flag that the app service interprets is the **anemic-entity anti-pattern** called out in `abp-ddd/SKILL.md`:

> **Anemic entities**: public setters with no behavior — use private setters + methods that enforce invariants

Expose the rule as a method on `Customer` so the invariant lives with the data:

```csharp
public class Customer : AggregateRoot<Guid>
{
    public bool IsBlocked { get; private set; }
    // ...

    public void EnsureCanPlaceOrders()
    {
        if (IsBlocked)
        {
            throw new BusinessException("Sales:CustomerIsBlocked")
                .WithData("CustomerId", Id);
        }
    }
}
```

`OrderManager` calls `customer.EnsureCanPlaceOrders()` (shown above). The application service never inspects `IsBlocked`.

---

## 4. Refactored `OrderAppService` — no `BusinessException` throws

The app service is now pure orchestration: validate-by-attribute (automatic), call the manager, persist, map. Note we drop the `_customerRepository` injection — the manager owns that lookup now.

```csharp
public class OrderAppService : ApplicationService, IOrderAppService
{
    private readonly IOrderRepository _orderRepository;
    private readonly OrderManager _orderManager;

    public OrderAppService(
        IOrderRepository orderRepository,
        OrderManager orderManager)
    {
        _orderRepository = orderRepository;
        _orderManager = orderManager;
    }

    public async Task<OrderDto> CreateAsync(CreateOrderDto input)
    {
        var order = await _orderManager.CreateAsync(input.CustomerId, input.Number);
        await _orderRepository.InsertAsync(order, autoSave: true);
        return ObjectMapper.Map<Order, OrderDto>(order);
    }
}
```

What's gone from the original:
- `Sales:OrderNumberRequired` — handled by `[Required]` on DTO
- `Sales:OrderNumberTooLong` — handled by `[StringLength(32)]` on DTO
- `Sales:CustomerNotFound` — `EntityNotFoundException` in `OrderManager`
- `Sales:CustomerIsBlocked` — `Customer.EnsureCanPlaceOrders()` (still a `BusinessException`, just on the entity)
- `Sales:OrderNumberAlreadyExists` — moved into `OrderManager`

Zero `BusinessException` throws remain in the application service. The exception-based error contract is unchanged from the client's perspective.

---

## 5. Loose ends worth fixing while you're in there

- **Localize the codes**: register the namespace once per `abp-core/SKILL.md` so `Sales:*` codes resolve to translated messages:

  ```csharp
  Configure<AbpExceptionLocalizationOptions>(options =>
  {
      options.MapCodeNamespace("Sales", typeof(SalesResource));
  });
  ```

- **`Customer` repository**: if you don't need custom Customer queries elsewhere, leave it as generic `IRepository<Customer, Guid>` inside the manager (`abp-core/SKILL.md` — generic repo is fine for simple CRUD).
- **Object mapping**: check whether the solution uses Mapperly or AutoMapper before adding new mapping code; Mapperly is the ABP 10.x default (`abp-application-layer/SKILL.md`, Object Mapping section). `ObjectMapper.Map<Order, OrderDto>` works with either.
- **`autoSave: true` on Insert**: fine here since this is a single-aggregate write at the end of the use case; ABP's UoW will commit anyway, but explicit is OK.
- **Authorization**: consider `[Authorize(SalesPermissions.Orders.Create)]` on `CreateAsync` — orthogonal to this review but a typical follow-up.

---

## Sources cited

- `abp-core/SKILL.md` — `BusinessException` with namespaced codes; base-class properties (`GuidGenerator`, `L`); localization config.
- `abp-application-layer/SKILL.md` — Anti-Patterns ("Business logic in app service"); Validation ("Domain rule vs Application rule"); Error Handling (`EntityNotFoundException`, `BusinessException`, status-code table).
- `abp-ddd/SKILL.md` — Anemic vs Rich entities; `OrderManager` example for uniqueness check (matches this exact scenario); Domain Service best practices (`*Manager` suffix, no interface by default); custom repository when custom queries needed.
