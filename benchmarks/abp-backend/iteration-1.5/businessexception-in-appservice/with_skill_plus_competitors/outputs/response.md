# Code review — `OrderAppService.CreateAsync`

The headline issue is that this app service is doing three jobs that don't belong to it: input-shape validation, entity-state guarding, and cross-aggregate uniqueness checking. ABP's DDD layering says the application service is a use-case orchestrator — it should call into the domain, persist, and map to a DTO. Every `throw new BusinessException(...)` you have here is a signal that a rule has leaked out of the layer that owns it.

The other rule it breaks is the "one rule, one home" principle: if `Number` length, `Customer.IsBlocked`, and `Number` uniqueness are enforced *only* in `OrderAppService.CreateAsync`, then any other caller — a background job that imports orders, an integration service, a future bulk-import endpoint, a domain event handler — can mutate the same entities past those rules. The fix is to push each rule down to the layer where every caller routes through it.

Here is the layout I'd land on.

## 1. Input-format validation moves to the DTO (validation pipeline)

`Number required` and `Number length <= 32` are pure input shape — there is no repository involved, no other aggregate, no entity state. That belongs on the DTO and runs through ABP's automatic validation pipeline before the method body executes:

```csharp
using System.ComponentModel.DataAnnotations;

namespace Acme.Sales;

public class CreateOrderDto
{
    public Guid CustomerId { get; set; }

    [Required]
    [StringLength(32)]
    public string Number { get; set; } = default!;
}
```

Because `OrderAppService` inherits `ApplicationService` (which implements `IValidationEnabled`) and `CreateAsync` is virtual, ABP intercepts the call, runs `DataAnnotationObjectValidationContributor` over `input`, and throws `AbpValidationException` — which the exception filter translates to a structured HTTP 400 — before your method body ever runs. You don't write a single `throw`. Equivalent FluentValidation works too if the team prefers `AbstractValidator<CreateOrderDto>`; just `[DependsOn(typeof(AbpFluentValidationModule))]` on the application module.

One nuance worth calling out: DTO `[Required]` / `[StringLength]` is **fast-fail UX**, not the invariant's permanent home. If `Number` length is a real domain rule (i.e., the database column is `varchar(32)` because the business says so), it should *also* be enforced inside the `Order` constructor via `Check.NotNullOrWhiteSpace(number, nameof(number), maxLength: OrderConsts.MaxNumberLength)`. That way, if anyone constructs an `Order` from an importer that doesn't go through the DTO, the rule still fires.

## 2. Cross-aggregate / repository-driven rules move to a Manager

`Customer not found`, `Customer.IsBlocked`, and `Number already exists` all need a repository lookup, and the uniqueness check spans the whole `Order` aggregate. That is exactly what a domain service (`OrderManager`) is for. ABP's own best-practices doc explicitly says domain services throw `BusinessException` on invariant violations — so the throws don't disappear, they relocate to where every caller has to route through them.

The Manager injects **`IReadOnlyRepository<T, TKey>` only** for its reads (no-tracking, signals "this isn't the persistence path"). The mutated entity flows in as a parameter from the app service, which holds the tracking `IRepository<Order, Guid>` and persists after.

For the `IsBlocked` guard specifically: since the rule reads only `Customer`'s own state, the strongest form is an entity-behavior method on `Customer` itself. That way every caller — Manager, app service, integration handler, event handler — gets the rule by routing through the entity. The Manager just calls it.

```csharp
// Domain/Customers/Customer.cs
public class Customer : FullAuditedAggregateRoot<Guid>
{
    public bool IsBlocked { get; private set; }
    // ...

    public void EnsureCanReceiveNewOrder()
    {
        if (IsBlocked)
        {
            throw new BusinessException("Sales:CustomerIsBlocked")
                .WithData("CustomerId", Id);
        }
    }
}
```

```csharp
// Domain/Orders/OrderManager.cs
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
        _orderRepository    = orderRepository;
    }

    public async Task<Order> CreateAsync(Guid id, Guid customerId, string number)
    {
        var customer = await _customerRepository.FindAsync(customerId)
            ?? throw new BusinessException("Sales:CustomerNotFound")
                   .WithData("CustomerId", customerId);

        customer.EnsureCanReceiveNewOrder();

        if (await _orderRepository.AnyAsync(o => o.Number == number))
        {
            throw new BusinessException("Sales:OrderNumberAlreadyExists")
                .WithData("Number", number);
        }

        return new Order(id, customerId, number); // Manager constructs; doesn't persist
    }
}
```

A few things to notice:

- `OrderManager` does **not** call `InsertAsync`. Domain services own invariants and pure logic; they never persist. The app service does that.
- It depends on `IReadOnlyRepository`, not `IRepository`. Seeing `IRepository<Order, Guid>` injected into a `*Manager` is a smell — either the Manager is persisting (forbidden) or the mutation should flow in as a parameter.
- `BusinessException("Sales:CustomerNotFound")` is the right ABP idiom for "domain rule violated"; it gets serialized to a 403/business-error response. For "this resource doesn't exist as an HTTP concept" you'd typically prefer `EntityNotFoundException` (or just let `GetAsync` throw it for you), but the team may already have decided to model "customer required for this op" as a business rule rather than a not-found — both are defensible.

For the `Order` constructor itself, mark it `internal` because creation now requires a business check (uniqueness + customer state):

```csharp
// Domain/Orders/Order.cs
public class Order : AggregateRoot<Guid>
{
    public Guid CustomerId { get; private set; }
    public string Number { get; private set; } = default!;

    protected Order() { } // EF Core

    internal Order(Guid id, Guid customerId, string number) : base(id)
    {
        CustomerId = customerId;
        Number = Check.NotNullOrWhiteSpace(
            number, nameof(number), maxLength: OrderConsts.MaxNumberLength);
    }
}
```

`internal` means controllers, app services, and integration handlers in *other* assemblies can't bypass `OrderManager` by doing `new Order(...)` directly. The only entry point is the Manager.

## 3. The application service shrinks to orchestration

After the moves above, `OrderAppService.CreateAsync` is what ABP best-practices says it should be — mint Id, delegate to Manager, persist, map:

```csharp
public class OrderAppService : ApplicationService, IOrderAppService
{
    private readonly OrderManager             _orderManager;
    private readonly IRepository<Order, Guid> _orderRepository; // tracking lives HERE

    public OrderAppService(
        OrderManager orderManager,
        IRepository<Order, Guid> orderRepository)
    {
        _orderManager    = orderManager;
        _orderRepository = orderRepository;
    }

    public virtual async Task<OrderDto> CreateAsync(CreateOrderDto input)
    {
        var order = await _orderManager.CreateAsync(
            GuidGenerator.Create(),
            input.CustomerId,
            input.Number);

        await _orderRepository.InsertAsync(order, autoSave: true);

        return ObjectMapper.Map<Order, OrderDto>(order);
    }
}
```

No `throw new BusinessException(...)` anywhere in the file. The method is `virtual` so ABP's UoW + validation interceptors fire. `GuidGenerator.Create()` is inherited from `ApplicationService` (sequential GUIDs, friendlier to clustered indexes than `Guid.NewGuid()`).

## Summary of where each rule landed

| Rule | Before (wrong) | After (correct) |
|---|---|---|
| `Number` required | `throw` in app service | `[Required]` on `CreateOrderDto.Number` (validation pipeline) |
| `Number` length <= 32 | `throw` in app service | `[StringLength(32)]` on DTO + `Check.NotNullOrWhiteSpace(..., maxLength: ...)` in `Order` ctor |
| Customer must exist | `throw` in app service | `OrderManager.CreateAsync` throws `BusinessException` after `FindAsync` |
| Customer not blocked | `throw` in app service | `Customer.EnsureCanReceiveNewOrder()` (entity behavior), called by `OrderManager` |
| `Number` unique | `throw` in app service | `OrderManager.CreateAsync` checks via `IReadOnlyRepository<Order, Guid>` and throws |
| Persist | `_orderRepository.InsertAsync` in app service | unchanged — app service still owns persistence |
| Map to DTO | in app service | unchanged |

## Two smaller things while you're in there

- **Constructor injection ceremony.** You can collapse the constructor with C# 12 primary constructors if your style guide allows: `public class OrderAppService(OrderManager orderManager, IRepository<Order, Guid> orderRepository) : ApplicationService, IOrderAppService`. Cosmetic, not load-bearing.
- **Don't reach for `IQueryable` in the app service.** This file doesn't, which is good — keep it that way. If a future query gets complex, define `IOrderRepository : IRepository<Order, Guid>` in the Domain project and put the query method there. Application services should never call `GetQueryableAsync()`.

## Reference points (ABP 10.3 docs)

- Domain Services best practices — explicitly authorizes `BusinessException` from a `*Manager` for repository-driven invariants: <https://abp.io/docs/10.3/framework/architecture/best-practices/domain-services>
- Application Services best practices — "no LINQ in services, don't call other app services in the same module": <https://abp.io/docs/10.3/framework/architecture/best-practices/application-services>
- Validation pipeline — DataAnnotations / FluentValidation auto-fire on `IValidationEnabled` virtual methods: <https://abp.io/docs/10.3/framework/api-development/validation> (also <https://abp.io/docs/10.3/framework/fundamentals/validation>)
- Aggregate-root entity rules — `internal` ctor + behavior methods + `Check.*`: <https://abp.io/docs/10.3/framework/architecture/best-practices/entities>
