## Code review: `OrderAppService.CreateAsync`

There's nothing catastrophically wrong here, but the service has drifted into a few anti-patterns that ABP gives you cleaner tools for. Two main themes:

1. **Validation vs. business rules are mixed together** and both expressed as raw `BusinessException`. Input validation belongs on the DTO; business rules belong in the domain.
2. **The application service is doing domain work** (uniqueness check, customer state check, order construction). In ABP/DDD, an application service should orchestrate — not enforce invariants.

Below is the cleanup I'd recommend, in order of impact.

---

### 1. Move primitive input validation to the DTO

`OrderNumberRequired` and `OrderNumberTooLong` aren't business rules — they're shape-of-the-input rules. ABP runs `IValidatableObject` / data annotations / FluentValidation automatically before the method body executes, so you get a proper `AbpValidationException` (which maps to HTTP 400, not 500/403) for free.

```csharp
public class CreateOrderDto
{
    [Required]
    [StringLength(OrderConsts.MaxNumberLength)]
    public string Number { get; set; } = default!;

    [Required]
    public Guid CustomerId { get; set; }
}
```

Define the constant once on the domain side (`OrderConsts.MaxNumberLength = 32`) and reuse it from the entity, the DTO, and EF Core configuration so the three never drift.

After this, the first two `if`/`throw` blocks in `CreateAsync` disappear entirely.

---

### 2. Use `GetAsync` instead of `FindAsync` + null check

`IRepository<T,TKey>.GetAsync(id)` already throws `EntityNotFoundException` (which ABP maps to HTTP 404) if the row isn't there. You only need `FindAsync` when null is a legitimate, expected outcome — here it isn't.

```csharp
var customer = await _customerRepository.GetAsync(input.CustomerId);
```

That's one less branch and a more accurate HTTP status code than the generic 403 you'd get from `BusinessException`.

---

### 3. Localize your `BusinessException` codes

`throw new BusinessException("Sales:CustomerIsBlocked")` only sets the error code — the user sees a generic message. ABP resolves `Sales:CustomerIsBlocked` against the localization resource if you tell it to, and you can attach extra data:

```csharp
throw new BusinessException(SalesDomainErrorCodes.CustomerIsBlocked)
    .WithData("CustomerId", customer.Id);
```

Then add `Sales:CustomerIsBlocked` (or whatever your code constants are) into your `Localization/Sales/en.json` and `zh-Hans.json` resource files. Centralize the codes in a `SalesDomainErrorCodes` static class so you're not stringly-typing them at every throw site.

---

### 4. Push the business rules into a domain service / manager

The two real business rules here are:

- "An order number must be unique."
- "An order cannot be placed for a blocked customer."

Both depend on entity state and invariants — they belong in the domain layer, not the application layer. The standard ABP pattern is an `OrderManager` (a `DomainService`) that owns order creation:

```csharp
// Sales.Domain/Orders/OrderManager.cs
public class OrderManager : DomainService
{
    private readonly IOrderRepository _orderRepository;

    public OrderManager(IOrderRepository orderRepository)
    {
        _orderRepository = orderRepository;
    }

    public async Task<Order> CreateAsync(Customer customer, string number)
    {
        Check.NotNull(customer, nameof(customer));
        Check.NotNullOrWhiteSpace(number, nameof(number));

        if (customer.IsBlocked)
        {
            throw new BusinessException(SalesDomainErrorCodes.CustomerIsBlocked)
                .WithData("CustomerId", customer.Id);
        }

        if (await _orderRepository.NumberExistsAsync(number))
        {
            throw new BusinessException(SalesDomainErrorCodes.OrderNumberAlreadyExists)
                .WithData("Number", number);
        }

        return new Order(GuidGenerator.Create(), customer.Id, number);
    }
}
```

Couple things to notice:

- I introduced `IOrderRepository` (a custom repository interface in the Domain layer) with a meaningful method `NumberExistsAsync(number)` instead of leaking a LINQ predicate to the app service. That's the ABP convention when a query is a domain concept.
- The manager doesn't persist — it returns the new aggregate. Persistence stays the application service's job. This keeps the manager unit-testable and gives the app service a single, obvious place for `InsertAsync`.

---

### 5. Slim app service

After the changes above, the application service becomes orchestration only:

```csharp
public class OrderAppService : ApplicationService, IOrderAppService
{
    private readonly IOrderRepository _orderRepository;
    private readonly IRepository<Customer, Guid> _customerRepository;
    private readonly OrderManager _orderManager;

    public OrderAppService(
        IOrderRepository orderRepository,
        IRepository<Customer, Guid> customerRepository,
        OrderManager orderManager)
    {
        _orderRepository = orderRepository;
        _customerRepository = customerRepository;
        _orderManager = orderManager;
    }

    [Authorize(SalesPermissions.Orders.Create)]
    public async Task<OrderDto> CreateAsync(CreateOrderDto input)
    {
        var customer = await _customerRepository.GetAsync(input.CustomerId);

        var order = await _orderManager.CreateAsync(customer, input.Number);

        await _orderRepository.InsertAsync(order, autoSave: true);

        return ObjectMapper.Map<Order, OrderDto>(order);
    }
}
```

---

### 6. Don't forget authorization

I added `[Authorize(SalesPermissions.Orders.Create)]` above. The original method has no permission check at all — anyone authenticated can create orders. In an ABP solution you almost always want explicit permissions defined in your `SalesPermissionDefinitionProvider` and checked declaratively. If the *whole service* should require auth, you can also set `protected override string[]? GetDefaultEntityActionPolicies` or just put `[Authorize]` on the class.

---

### 7. Minor things worth fixing

- **`autoSave: true`** is fine here because there's only one write, but if `CreateAsync` ever grows (e.g., raise a domain event, append a line item), prefer relying on the unit-of-work to flush at the end of the request and drop `autoSave`.
- **Domain events**: when an order is created, raise `OrderCreatedEto` (or a local domain event) inside the `Order` constructor or `OrderManager`. ABP's `IDistributedEventBus` / `ILocalEventBus` plus the unit-of-work integration will handle the dispatch. This keeps integrations (email, inventory reservation, audit) decoupled from the create path.
- **Constructor on `Order`**: make sure the entity constructor itself runs `Check.NotNullOrWhiteSpace(number)` and `Check.Length(number, MaxLength)`. The DTO check protects the API; the entity check protects the domain from any other call site (tests, seeders, importers).
- **Consider a typed `OrderNumber` value object** if the rules around the number grow (format, prefix, checksum). Right now a string is fine.

---

### TL;DR refactor checklist

- [ ] Move `Number` required/length checks to `CreateOrderDto` data annotations.
- [ ] Replace `_customerRepository.FindAsync` + null check with `GetAsync`.
- [ ] Introduce `IOrderRepository` with `NumberExistsAsync(string)`.
- [ ] Introduce `OrderManager : DomainService` and move both business-rule throws there.
- [ ] Centralize error codes in `SalesDomainErrorCodes` and add localized messages.
- [ ] Add `[Authorize(SalesPermissions.Orders.Create)]`.
- [ ] Optionally raise an `OrderCreated` event from the domain.

The end state is an app service that reads like a story (load customer, ask the domain to create an order, persist, map) and a domain layer that actually owns its invariants — which is what the ABP+DDD layout is designed to give you.
