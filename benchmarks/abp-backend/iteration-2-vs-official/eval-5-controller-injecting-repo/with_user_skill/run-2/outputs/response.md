# Code review — `OrderController`

This controller violates ABP's recommended explicit-controller pattern in three reinforcing ways: it is doing the job of the application service, it is checking authorization imperatively from a role string, and it is reaching across the layer boundary into the domain and persistence. In a Layered ABP 10.3 solution this should be two separate concrete classes that both implement `IOrderAppService`. The fix below is structural — splitting the responsibilities is what eliminates every individual smell at once.

## 1. Split into two classes — `OrderAppService` and `OrderController`

The recommended explicit-controller pattern is two concrete classes that both implement the same `IOrderAppService` interface:

- **`OrderAppService : ApplicationService, IOrderAppService`** (lives in `*.Application`) — owns *all* collaborators (`OrderManager`, `IRepository<Order, Guid>`, mapping, current user, distributed events), performs orchestration, hosts declarative `[Authorize(...)]` attributes, persists, returns DTOs.
- **`OrderController : AbpController, IOrderAppService`** (lives in `*.HttpApi`) — pure HTTP adapter. Injects **exactly one** collaborator: `IOrderAppService`. Action bodies are single delegating expressions.

This is the canonical split because the application service is the real authorization and orchestration boundary — every caller (HTTP, background jobs, integration services, distributed-event handlers, in-process callers from other modules) routes through it. Putting logic in the controller silently bypasses every non-HTTP path.

## 2. Controller must inject only `IOrderAppService`

The current controller injects four collaborators — `IRepository<Order, Guid>`, `OrderManager`, `IObjectMapper`, `ICurrentUser`. Every one of these is forbidden in a controller under the explicit pattern:

- `IRepository<Order, Guid>` — persistence belongs to the AppService.
- `OrderManager` — domain services belong to the AppService.
- `IObjectMapper` — mapping is an AppService concern; the controller hands back DTOs already mapped.
- `ICurrentUser` for business decisions — controllers do **not** make authorization decisions imperatively (declarative `[Authorize]` only, and on the AppService method).

If the urge to inject any of these arises, the logic belongs in the AppService — add a method there and have the controller delegate.

## 3. Authorization is declarative, permission-based, and lives on the AppService

The line

```csharp
if (!_currentUser.IsInRole("sales"))
    throw new AbpAuthorizationException();
```

is wrong on three independent axes:

- **It checks a role, not a permission.** Roles are an *assignment dimension* (which users get which permissions). Permissions are the *authorization unit*. Hardcoding the role string `"sales"` couples the code to a specific tenant/install's role catalog and makes the rule un-grantable to any user who happens not to be in that role. Use `OrderPermissions.Create` (a constant defined in `*.Application.Contracts` alongside `IOrderAppService` and registered through an `OrderPermissionDefinitionProvider`).
- **It's imperative.** ABP's authorization pipeline is declarative — `[Authorize(OrderPermissions.Create)]` — and produces a uniform 401/403 response, integrates with policy providers, and is discoverable for auditing. Throwing `AbpAuthorizationException` by hand is what the attribute does for you.
- **It's on the wrong class.** Putting `[Authorize]` on the controller (let alone an imperative check in the controller body) only covers the HTTP path. Every other entry point — background jobs, integration services, distributed-event handlers, internal callers from another module in the same modular monolith — bypasses it silently. Putting the attribute on the **AppService method** means every caller gets the check. Repeating it on the controller is redundant and rots over time (the two can drift apart). Default to AppService-only; reach for a controller-side `[Authorize]` only when the controller exposes something the AppService genuinely cannot.

## 4. Mint `GuidGenerator` and use `ObjectMapper` from the `ApplicationService` base

`ApplicationService` inherits `GuidGenerator`, `ObjectMapper`, `CurrentUser`, `Clock`, `Logger`, `LazyServiceProvider`, etc. — no need to inject `IObjectMapper` separately, and `GuidGenerator.Create()` is the right Id mint at the AppService layer (the Manager receives the Id as a parameter, per Rule 3 — the Manager doesn't mint Ids and doesn't persist).

## 5. Route attribute and class conventions

`[Route("api/orders")]` is fine. Prefer `sealed` on the concrete controller, and have it implement the `IOrderAppService` interface so the auto-API client generator and proxy tooling both recognize the contract.

## Refactored shape

```csharp
// *.Application.Contracts/Permissions/OrderPermissions.cs
public static class OrderPermissions
{
    public const string GroupName = "Orders";
    public const string Default = GroupName + ".Default";
    public const string Create  = GroupName + ".Create";
}

// *.Application.Contracts/IOrderAppService.cs
public interface IOrderAppService : IApplicationService
{
    Task<OrderDto> CreateAsync(CreateOrderDto input);
}

// *.Application/Orders/OrderAppService.cs — orchestration, authorization, mapping, persistence
public class OrderAppService : ApplicationService, IOrderAppService
{
    private readonly IRepository<Order, Guid> _orderRepository;
    private readonly OrderManager _orderManager;

    public OrderAppService(
        IRepository<Order, Guid> orderRepository,
        OrderManager orderManager)
    {
        _orderRepository = orderRepository;
        _orderManager = orderManager;
    }

    [Authorize(OrderPermissions.Create)]
    public async Task<OrderDto> CreateAsync(CreateOrderDto input)
    {
        var order = await _orderManager.CreateAsync(
            GuidGenerator.Create(), input.CustomerId, input.Number);

        await _orderRepository.InsertAsync(order, autoSave: true);

        return ObjectMapper.Map<Order, OrderDto>(order);
    }
}

// *.HttpApi/Controllers/OrderController.cs — pure HTTP adapter, ONE dependency, NO [Authorize]
[Route("api/orders")]
public sealed class OrderController : AbpController, IOrderAppService
{
    private readonly IOrderAppService _orderAppService;

    public OrderController(IOrderAppService orderAppService)
    {
        _orderAppService = orderAppService;
    }

    [HttpPost]
    public Task<OrderDto> CreateAsync(CreateOrderDto input)
        => _orderAppService.CreateAsync(input);
}
```

## A few smaller notes

- **Don't collapse the two classes with `[ExposeServices(typeof(IOrderAppService), typeof(OrderController))]`.** That forces the controller back to injecting managers and repositories — exactly the dependencies the split is removing. Keep them as two separate registrations.
- **Input validation** (required `CustomerId`, required/format `Number`) belongs on `CreateOrderDto` via data annotations or a FluentValidation `AbstractValidator<CreateOrderDto>` — not as imperative checks in the AppService. ABP's validation pipeline turns those into `AbpValidationException` automatically.
- **Uniqueness / cross-aggregate guards** (e.g., "order number must be unique," "customer must exist and be allowed to receive orders") belong in `OrderManager`, which throws `BusinessException`. Don't move those into the AppService.
- **`autoSave: true`** is fine for a create-and-return endpoint; just be aware it commits inside the UoW boundary rather than at the outer commit. If you later add a second write in the same action, prefer a single outer commit and drop `autoSave`.
- If your project's `.abp-overrides.md` or CLAUDE.md mandates Auto API Controllers throughout, that override wins and you wouldn't be hand-writing a controller for `OrderAppService` at all.
