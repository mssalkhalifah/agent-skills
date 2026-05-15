# Code review — `OrderAppService.CreateAsync`

The headline problem: this application service is doing three different jobs that ABP wants you to split across three different layers. Every rule it currently violates is `throw new BusinessException(...)` inside the service body — that's the Rule 7 smell. The fix is mechanical once you see where each check actually belongs.

## What's wrong, by category

There are three kinds of "validation" tangled together in `CreateAsync`. Treat them separately.

| # | Current check | Real nature | Belongs in |
|---|---|---|---|
| 1 | `string.IsNullOrWhiteSpace(input.Number)` | Input-format (DTO shape) | DTO via data annotations / FluentValidation |
| 2 | `input.Number.Length > 32` | Input-format (DTO shape) | DTO via data annotations / FluentValidation |
| 3 | `customer == null` after `FindAsync` | Existence of a tracked aggregate the manager needs | `OrderManager` (use `GetAsync`, which throws `EntityNotFoundException`) |
| 4 | `customer.IsBlocked` | Invariant that reads only the customer's own state | `Customer.EnsureCanReceiveNewOrder()` entity-behavior method |
| 5 | Uniqueness of `Number` (cross-aggregate repo check) | Domain invariant requiring a repository check | `OrderManager` (domain service) |

Once you move each one, the application service has **zero** `throw new BusinessException(...)` calls — which is exactly what Rule 7 prescribes ("Don't throw business exceptions from application services; use ABP's validation pipeline for input"). The skill is explicit: app services validate input *via the pipeline*, call the domain service for invariants, and orchestrate persistence — they do not own business rules.

A few other observations while we're in here:

- `FindAsync` + null-check is verbose. If the customer is required for the operation, use `GetAsync` and let ABP's `EntityNotFoundException` produce the structured 404. This isn't a `BusinessException` in disguise — it's "this Id doesn't exist," which has its own canonical exception.
- The `OrderManager` (domain service) is missing entirely. Per Rule 3, the manager owns cross-aggregate validation; the app service mints the Id (`GuidGenerator` is inherited from `ApplicationService` — no need to inject `IGuidGenerator`), passes inputs in, and persists what comes back.
- Per Rule 3, the manager should inject `IReadOnlyRepository<T, TKey>` only — both for `Customer` (it only reads) and for `Order` (the uniqueness check is a read). The tracking `IRepository<Order, Guid>` stays in the app service, which is the layer that persists.
- Per Rule 7 priority order, `Customer.IsBlocked` is best enforced **on the entity itself** (priority 1 — "rule reads only the entity's own state"), not inside the manager. That way every caller — manager, integration service, event handler, future code paths — gets the same guard by routing through the entity.
- Keep ABP's exception-based error model. Don't refactor to `Result<T,E>` or similar — ABP wires `BusinessException` / `AbpValidationException` / `EntityNotFoundException` straight into the HTTP error pipeline and the localization stack. Switching to a Result type loses all of that for no win.

## Refactor

### 1. Put input-format rules on the DTO

Either data annotations (built-in, lowest friction) or FluentValidation if the project already uses it. Both flow through `AbpValidationException` and produce a structured 400 — the service never sees the invalid input.

```csharp
// CreateOrderDto.cs — Application.Contracts
using System;
using System.ComponentModel.DataAnnotations;

public class CreateOrderDto
{
    [Required]
    public Guid CustomerId { get; set; }

    [Required]
    [StringLength(32)]                  // covers both "required" trim-empty and max-length
    public string Number { get; set; } = default!;
}
```

If you want trim-empty semantics specifically (annotations treat `"   "` as valid), reach for FluentValidation:

```csharp
// CreateOrderDtoValidator.cs — Application
using FluentValidation;

public class CreateOrderDtoValidator : AbstractValidator<CreateOrderDto>
{
    public CreateOrderDtoValidator()
    {
        RuleFor(x => x.CustomerId).NotEmpty();
        RuleFor(x => x.Number).NotEmpty().MaximumLength(32);
    }
}
```

ABP's FluentValidation integration is auto-discovered — registering the validator is enough.

### 2. Put the "can the customer receive an order?" rule on the entity

This is Rule 7 priority 1 — the rule reads only the customer's own state, so the entity owns it. Every caller routes through this method.

```csharp
// Customer.cs — Domain
public class Customer : FullAuditedAggregateRoot<Guid>
{
    public bool IsBlocked { get; private set; }
    // ... rest of the entity ...

    public void EnsureCanReceiveNewOrder()
    {
        if (IsBlocked)
        {
            throw new BusinessException("Sales:CustomerIsBlocked");
        }
    }
}
```

### 3. Put the cross-aggregate uniqueness check on `OrderManager`

This requires a repository check — Rule 7 priority 2, the canonical place. Manager builds and returns the aggregate; it does **not** persist (Rule 3).

```csharp
// OrderManager.cs — Domain
using System;
using System.Threading.Tasks;
using Volo.Abp;
using Volo.Abp.Domain.Repositories;
using Volo.Abp.Domain.Services;

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
        var customer = await _customerRepository.GetAsync(customerId);   // throws EntityNotFoundException if missing
        customer.EnsureCanReceiveNewOrder();                              // entity-behavior throws if blocked

        if (await _orderRepository.AnyAsync(o => o.Number == number))
        {
            throw new BusinessException("Sales:OrderNumberAlreadyExists");
        }

        return new Order(id, customerId, number);                         // Manager doesn't persist
    }
}
```

A small extra step worth taking: make `Order`'s constructor `internal` so the only legitimate construction path runs through `OrderManager`. That's the structural enforcement that prevents callers from sneaking around the uniqueness/blocked guards.

### 4. The new `OrderAppService` — orchestration only, zero `BusinessException`

```csharp
// OrderAppService.cs — Application
using System;
using System.Threading.Tasks;
using Volo.Abp.Application.Services;
using Volo.Abp.Domain.Repositories;

public class OrderAppService : ApplicationService, IOrderAppService
{
    private readonly OrderManager             _orderManager;
    private readonly IRepository<Order, Guid> _orderRepository;   // tracking lives HERE

    public OrderAppService(
        OrderManager             orderManager,
        IRepository<Order, Guid> orderRepository)
    {
        _orderManager    = orderManager;
        _orderRepository = orderRepository;
    }

    [Authorize(OrderPermissions.Create)]                          // declarative auth on the app service (Rule 4)
    public async Task<OrderDto> CreateAsync(CreateOrderDto input)
    {
        var order = await _orderManager.CreateAsync(
            GuidGenerator.Create(),                                // inherited from ApplicationService
            input.CustomerId,
            input.Number);

        await _orderRepository.InsertAsync(order, autoSave: true);
        return ObjectMapper.Map<Order, OrderDto>(order);
    }
}
```

Notice what's *not* in there: no `if (string.IsNullOrWhiteSpace(...))`, no `Length > 32`, no `customer == null`, no `IsBlocked` check, no uniqueness query, and zero `throw new BusinessException(...)`. The service does what an application service is supposed to do — mint an Id, call the domain, persist, map, return.

## Why each move matters (one line each)

- **DTO validation** — runs before the service body, returns a structured 400 with field-level errors, and is automatically localized. Imperative throws can't do any of that without extra code.
- **Entity-behavior method** for `IsBlocked` — every future caller (background job, integration service, event handler, another app service) gets the guard by construction. Today's callsite is just the first.
- **`OrderManager` for uniqueness** — couples the rule to the invariant, not the HTTP entry point. The same rule fires whether `CreateAsync` was triggered by a controller, a Hangfire job, or an `IIntegrationService` consumer.
- **`EntityNotFoundException` via `GetAsync`** — the right exception type, mapped to 404 by ABP, no `BusinessException` impedance.
- **Keep exceptions, don't switch to `Result<T,E>`** — ABP's whole error pipeline (HTTP status mapping, localization, audit logs, client-proxy error reconstruction) is built around exceptions; a Result type would re-implement that infrastructure poorly.

## Migration checklist

1. Add `[Required]` / `[StringLength(32)]` to `CreateOrderDto.Number` (or a `CreateOrderDtoValidator`).
2. Add `Customer.EnsureCanReceiveNewOrder()` on the `Customer` entity.
3. Add `OrderManager : DomainService` in the Domain layer with the two `IReadOnlyRepository` dependencies and the `CreateAsync` shown above.
4. Optional but recommended: switch `Order`'s public constructor to `internal` so callers can't bypass the manager.
5. Replace the body of `OrderAppService.CreateAsync` with the orchestration-only version above; drop the `IRepository<Customer, Guid>` dependency from the app service entirely.
6. Move localization keys (`Sales:CustomerIsBlocked`, `Sales:OrderNumberAlreadyExists`) — they don't move, just verify they exist in the localization resource.
7. Update tests: unit-test `OrderManager.CreateAsync` for the two invariant paths, integration-test `OrderAppService` for the happy path + a validation-failure case (asserts `AbpValidationException` / 400).
