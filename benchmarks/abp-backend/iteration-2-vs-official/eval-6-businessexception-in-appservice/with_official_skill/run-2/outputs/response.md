# Code Review: `OrderAppService.CreateAsync`

The current implementation mixes three concerns inside the application service: DTO input-format validation, domain invariants (uniqueness, customer-blocked rule), and orchestration. Per ABP guidance, the application service should be a thin orchestrator; format validation belongs on the DTO and business rules belong in the entity / domain service ("Manager"). This also removes every `BusinessException` throw from the app service — they stay in the system but now live where the rule belongs.

## 1. Input validation -> DataAnnotations on the DTO

`Number` is a length/required check, which is pure input-format validation. `abp-application-layer` explicitly puts this kind of check on the DTO with DataAnnotations, not as `if (...) throw new BusinessException(...)` inside the service:

> "**Domain rule**: Put validation in entity constructor or domain service ... **Application rule**: Use DTO validation (input format, required fields)" — `abp-application-layer/SKILL.md`

Also from `abp-application-layer`, the canonical pattern:

```csharp
public class CreateBookDto
{
    [Required]
    [StringLength(100, MinimumLength = 3)]
    public string Name { get; set; }
    ...
}
```

Apply the same to `CreateOrderDto` — `Sales:OrderNumberRequired` and `Sales:OrderNumberTooLong` become attributes, and ABP's automatic validation interceptor will throw `AbpValidationException` (HTTP 400) before the service body even runs.

```csharp
// *.Application.Contracts
public class CreateOrderDto
{
    [Required]
    [StringLength(32)]
    public string Number { get; set; } = default!;

    [Required]
    public Guid CustomerId { get; set; }
}
```

DTO location is per `abp-application-layer`: "Define DTOs in `*.Application.Contracts` project".

## 2. Uniqueness rule -> `OrderManager` (domain service)

"Order number must be unique" is a domain invariant that requires a repository query, so it belongs in a domain service. `abp-ddd/SKILL.md` shows exactly this pattern, almost verbatim to your case:

```csharp
// from abp-ddd/SKILL.md
public class OrderManager : DomainService
{
    private readonly IOrderRepository _orderRepository;
    ...
    public async Task<Order> CreateAsync(string orderNumber, Guid customerId)
    {
        // Business rule: Order number must be unique
        var existing = await _orderRepository.FindByOrderNumberAsync(orderNumber);
        if (existing != null)
        {
            throw new BusinessException("Orders:OrderNumberAlreadyExists")
                .WithData("OrderNumber", orderNumber);
        }
        return new Order(GuidGenerator.Create(), orderNumber, customerId);
    }
}
```

Notes that match the skills:
- `*Manager` suffix, no interface by default (`abp-ddd` — "Use `*Manager` suffix naming", "No interface by default").
- `GuidGenerator` comes from the `DomainService` base class — do not inject `IGuidGenerator` (`abp-core` base-class properties table).
- A custom `IOrderRepository : IRepository<Order, Guid>` with `FindByOrderNumberAsync` is preferred over `AnyAsync(...)` inlined in the app service. `abp-ddd`: "Define custom repository only when custom queries are needed" — uniqueness check is one.

## 3. `IsBlocked` rule -> on `Customer` (entity) and checked by the Manager

"Customer is blocked, cannot create order" is a domain rule too. Two valid placements:

- **Preferred**: a behavior method on the `Customer` aggregate (`EnsureNotBlocked()` or `PlaceOrder(...)` that throws when `IsBlocked`), which keeps the invariant with the entity that owns the state. `abp-ddd` Rich Domain Model section: "Logic in entity methods", "Entity enforces invariants", expose behavior through methods, not anemic getters.
- The `OrderManager` then loads the customer and invokes the entity method (or checks the state and throws). `EntityNotFoundException` covers the "customer not found" case — `abp-application-layer` Error Handling section shows `throw new EntityNotFoundException(typeof(Book), id);` is the right shape rather than a `BusinessException` for "not found".

```csharp
// Domain
public class Customer : AggregateRoot<Guid>
{
    public bool IsBlocked { get; private set; }
    // ...
    public void EnsureCanPlaceOrder()
    {
        if (IsBlocked)
            throw new BusinessException("Sales:CustomerIsBlocked")
                .WithData("CustomerId", Id);
    }
}
```

```csharp
// Domain - OrderManager
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
        var customer = await _customerRepository.FindAsync(customerId);
        if (customer == null)
            throw new EntityNotFoundException(typeof(Customer), customerId);

        customer.EnsureCanPlaceOrder();

        if (await _orderRepository.FindByOrderNumberAsync(number) != null)
            throw new BusinessException("Sales:OrderNumberAlreadyExists")
                .WithData("Number", number);

        return new Order(GuidGenerator.Create(), number, customerId);
    }
}
```

This keeps the **exception-based** error model — `BusinessException` / `EntityNotFoundException` are still thrown, just from the right layer. `abp-application-layer` HTTP mapping table confirms these are first-class citizens (`BusinessException` -> 403 configurable, `EntityNotFoundException` -> 404, `AbpValidationException` -> 400).

## 4. Refactored `OrderAppService` — no `BusinessException` throws

```csharp
public class OrderAppService : ApplicationService, IOrderAppService
{
    private readonly IOrderRepository _orderRepository;
    private readonly OrderManager _orderManager;
    private readonly OrderMapper _orderMapper; // Mapperly, per abp-application-layer

    public OrderAppService(
        IOrderRepository orderRepository,
        OrderManager orderManager,
        OrderMapper orderMapper)
    {
        _orderRepository = orderRepository;
        _orderManager    = orderManager;
        _orderMapper     = orderMapper;
    }

    [Authorize(SalesPermissions.Orders.Create)]
    public async Task<OrderDto> CreateAsync(CreateOrderDto input)
    {
        var order = await _orderManager.CreateAsync(input.Number, input.CustomerId);
        await _orderRepository.InsertAsync(order, autoSave: true);
        return _orderMapper.MapToDto(order);
    }
}
```

What disappeared from the app service and why (each tied to a skill):

| Removed line | Where it went | Skill reference |
|---|---|---|
| `if (string.IsNullOrWhiteSpace(input.Number)) throw ...` | `[Required]` on `CreateOrderDto.Number` | `abp-application-layer` — Validation / DataAnnotations |
| `if (input.Number.Length > 32) throw ...` | `[StringLength(32)]` on DTO | `abp-application-layer` — Validation |
| `_customerRepository.FindAsync` + null throw `BusinessException` | `OrderManager` with `EntityNotFoundException` | `abp-application-layer` — Error Handling ("Entity Not Found") |
| `if (customer.IsBlocked) throw ...` | `Customer.EnsureCanPlaceOrder()` invoked by `OrderManager` | `abp-ddd` — Rich Domain Model, "Entity enforces invariants" |
| `if (await _orderRepository.AnyAsync(...))` uniqueness throw | `OrderManager.CreateAsync` (matches the skill's example almost verbatim) | `abp-ddd` — Domain Services section |
| `new Order(GuidGenerator.Create(), ...)` | Constructed inside `OrderManager` | `abp-ddd` — "Don't generate GUID in constructor … Use `IGuidGenerator` externally" |
| `ObjectMapper.Map<Order, OrderDto>` | `OrderMapper` (Mapperly partial) | `abp-application-layer` — Object Mapping (Mapperly default) |

## 5. Other cleanups worth applying

- **Custom repository**: introduce `IOrderRepository : IRepository<Order, Guid>` in the Domain project with `FindByOrderNumberAsync(string number)`. `abp-ddd`: "Interface in Domain, implementation in data layer", "Define custom repository only when custom queries are needed".
- **Localization keys**: keep the `Sales:` namespace (`Sales:OrderNumberAlreadyExists`, `Sales:CustomerIsBlocked`) and wire them via `AbpExceptionLocalizationOptions.MapCodeNamespace("Sales", typeof(SalesResource))` per `abp-core` — Business Exceptions section. Add the strings under `*.Domain.Shared/Localization/Sales/en.json`.
- **Authorization**: the original service has no `[Authorize]`. Add `[Authorize(SalesPermissions.Orders.Create)]` as shown in `abp-application-layer`'s `BookAppService.CreateAsync` example.
- **No app-service-to-app-service calls** — already fine here, but if Customer logic grows, do not call `ICustomerAppService` from `OrderAppService` (`abp-application-layer` Anti-Patterns: "Calling other app services in the same module").
- **`Clock` / `CurrentUser`**: if the entity needs `CreatedAt` or `CreatorId`, use the audited base entity rather than passing `DateTime.Now` (`abp-core` — never use `DateTime.Now`).

## 6. Net effect

- App service: pure orchestration, zero `BusinessException`, zero hand-written input validation, zero entity instantiation.
- Domain: `Customer.EnsureCanPlaceOrder()` + `OrderManager.CreateAsync(...)` hold the rules and continue to throw `BusinessException` / `EntityNotFoundException` (exception-based model preserved, HTTP mapping unchanged).
- Contracts: `CreateOrderDto` carries DataAnnotations; `AbpValidationException` handles bad input before the service runs.
