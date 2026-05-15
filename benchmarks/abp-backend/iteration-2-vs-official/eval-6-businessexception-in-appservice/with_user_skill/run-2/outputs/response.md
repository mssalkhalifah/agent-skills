# Code review: OrderAppService cleanup

The current `OrderAppService.CreateAsync` throws `BusinessException` directly from the application service. ABP's guidance (Rule 7) is that application services don't own business invariants — they validate input, orchestrate the domain, and persist. Every throw in your method belongs somewhere else, and each one belongs in a different "somewhere." Below is the breakdown plus a refactored version.

## What's wrong, line by line

| Throw | Category | Where it belongs |
|---|---|---|
| `Number` is null/whitespace | Input validation (shape of the request) | DTO via `[Required]` |
| `Number.Length > 32` | Input validation | DTO via `[StringLength(32)]` |
| `Customer` not found | Domain lookup failure | `IRepository<Customer>.GetAsync(id)` — ABP's `GetAsync` already throws `EntityNotFoundException` (auto-maps to HTTP 404). Don't re-throw a `BusinessException` for "not found." |
| `customer.IsBlocked` | Entity-only state check | Entity behaviour method on `Customer` — every caller routes through it |
| `Number` uniqueness | Cross-aggregate repository check | `OrderManager` (domain service), since it needs an `IReadOnlyRepository<Order>` query |
| Constructing `Order` | Aggregate creation that has an associated invariant | `OrderManager.CreateAsync(...)` — Manager builds it, app service persists |

Net effect: the refactored `OrderAppService` has **zero** `throw new BusinessException` calls. It still uses the exception-based error model — the throws just live in the layer that actually owns each rule.

## 1. Input validation moves to the DTO

`CreateOrderDto` carries the shape rules. ABP's validation pipeline runs DataAnnotations automatically on any `ApplicationService` method, throws `AbpValidationException`, and the exception filter converts that to a structured HTTP 400 — the service never sees the invalid input.

```csharp
using System.ComponentModel.DataAnnotations;

public class CreateOrderDto
{
    public Guid CustomerId { get; set; }

    [Required]
    [StringLength(32)]
    public string Number { get; set; } = default!;
}
```

Notes:
- `[Required]` covers null *and* whitespace (treat-empty-string-as-null is on by default for `[Required]` on strings; if you want to be explicit about whitespace, add a small `IValidatableObject` rule or use FluentValidation `NotEmpty()`).
- For automatic interception to fire, the app-service method must be virtual (or called through the interface). `ApplicationService` already implements `IValidationEnabled`.

## 2. `IsBlocked` becomes entity behaviour on `Customer`

The rule reads only the customer's own state, so per Rule 7 priority #1 it lives on the entity. That way the rule is enforced even if someone fetches `Customer` through a different code path.

```csharp
// Customer.cs (Domain)
public class Customer : FullAuditedAggregateRoot<Guid>
{
    public bool IsBlocked { get; private set; }
    // ...

    public void EnsureCanReceiveNewOrder()
    {
        if (IsBlocked)
            throw new BusinessException("Sales:CustomerIsBlocked");
    }
}
```

## 3. Uniqueness + aggregate construction move to `OrderManager`

Uniqueness requires a repository check, so per Rule 3 / Rule 7 priority #2 it belongs in a domain service. The Manager:

- Injects `IReadOnlyRepository<Customer>` and `IReadOnlyRepository<Order>` (read-only, per Rule 3 — the Manager doesn't persist).
- Calls `GetAsync` on the customer (gets `EntityNotFoundException` for free — no `Sales:CustomerNotFound` re-throw needed).
- Invokes the entity-behaviour method for `IsBlocked`.
- Throws `BusinessException("Sales:OrderNumberAlreadyExists")` for the uniqueness check — this is the canonical place for a repository-driven invariant throw.
- Returns the new `Order`; persistence is the application service's job.

```csharp
// OrderManager.cs (Domain)
public class OrderManager : DomainService
{
    private readonly IReadOnlyRepository<Customer, Guid> _customerRepository;
    private readonly IReadOnlyRepository<Order, Guid> _orderRepository;

    public OrderManager(
        IReadOnlyRepository<Customer, Guid> customerRepository,
        IReadOnlyRepository<Order, Guid> orderRepository)
    {
        _customerRepository = customerRepository;
        _orderRepository = orderRepository;
    }

    public async Task<Order> CreateAsync(Guid id, Guid customerId, string number)
    {
        var customer = await _customerRepository.GetAsync(customerId); // 404 if missing
        customer.EnsureCanReceiveNewOrder();                            // entity-owned rule

        if (await _orderRepository.AnyAsync(o => o.Number == number))
            throw new BusinessException("Sales:OrderNumberAlreadyExists");

        return new Order(id, customerId, number);
    }
}
```

Consider making `Order`'s constructor `internal` so it can't be `new`-ed from anywhere except the Manager — that hardens the boundary further (see Rule 7 / aggregate-constructor guidance).

## 4. Refactored `OrderAppService`

The app service is now purely orchestration: mint the Id, ask the Manager to build the aggregate, persist it, map the DTO. Zero `BusinessException` throws.

```csharp
public class OrderAppService : ApplicationService, IOrderAppService
{
    private readonly OrderManager _orderManager;
    private readonly IRepository<Order, Guid> _orderRepository;

    public OrderAppService(
        OrderManager orderManager,
        IRepository<Order, Guid> orderRepository)
    {
        _orderManager = orderManager;
        _orderRepository = orderRepository;
    }

    public virtual async Task<OrderDto> CreateAsync(CreateOrderDto input)
    {
        var order = await _orderManager.CreateAsync(
            GuidGenerator.Create(),    // inherited from ApplicationService
            input.CustomerId,
            input.Number);

        await _orderRepository.InsertAsync(order, autoSave: true);
        return ObjectMapper.Map<Order, OrderDto>(order);
    }
}
```

## Why this layout

- **Every caller is covered.** Background jobs, integration services, distributed-event handlers — anything that goes through `OrderManager` gets uniqueness + blocked-customer checks. Anything that touches `Customer` directly still gets `EnsureCanReceiveNewOrder()`.
- **Input vs invariant separation.** `Number` length is a shape rule on the wire and lives on the DTO. The aggregate's own range/length invariants (if any) should *also* live on `Order` itself so they hold for any non-DTO caller — DTO attributes are fast-fail UX, not the home of the invariant.
- **Exception model unchanged.** Still using `BusinessException` / `AbpValidationException` / `EntityNotFoundException`. No `Result<T, E>` introduced. Each exception is just thrown from the layer that owns the rule.
- **Repository interfaces tightened.** `OrderManager` uses `IReadOnlyRepository<>` only (no-tracking reads, signals intent, can't accidentally persist). `OrderAppService` keeps the tracking `IRepository<Order, Guid>` because *it* persists.

## Optional follow-ups (not strictly required)

- Add `[Authorize(OrderPermissions.Create)]` on the AppService method — per Rule 4, declarative authorization belongs on the AppService so non-HTTP callers also get the check.
- If you have an `IOrderRepository` custom repository, the `AnyAsync(o => o.Number == ...)` check inside the Manager is a candidate for a named method (`NumberExistsAsync(string)`) so the query intent is explicit and the LINQ doesn't sprawl across the codebase.
- If `Customer` has more "can this customer do X" rules, group them on the entity (`EnsureCanPlaceOrder`, `EnsureCanReceiveRefund`, etc.) rather than scattering `if (customer.IsBlocked)` checks across services.
