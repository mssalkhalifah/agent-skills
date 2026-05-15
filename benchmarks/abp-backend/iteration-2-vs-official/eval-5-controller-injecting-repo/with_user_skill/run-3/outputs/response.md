# Code review — `OrderController` (ABP 10.3 Layered)

This controller is doing the application service's job. In a Layered solution it should be a thin HTTP adapter — split it into two classes, keep authorization declarative, and stop injecting domain/persistence/mapping collaborators at the presentation layer.

## What's wrong, in order of severity

### 1. Controller injects domain/persistence/mapping collaborators it shouldn't see

`IRepository<Order, Guid>`, `OrderManager`, `IObjectMapper`, and `ICurrentUser` all belong on the **application service**, not the controller. The controller is presentation — its only collaborator should be `IOrderAppService`. Today this controller orchestrates the domain (calls `OrderManager`), persists (`InsertAsync(..., autoSave: true)`), and maps (`_objectMapper.Map<...>`); that orchestration is the application service's contract.

### 2. Imperative role check instead of declarative permission

```csharp
if (!_currentUser.IsInRole("sales"))
    throw new AbpAuthorizationException();
```

Two problems:

- **Roles are not the authorization unit in ABP — permissions are.** Roles are an *assignment* dimension; you grant *permissions* to roles. A hard-coded `"sales"` role string can't be granted/revoked per-tenant, can't be queried by the permission management UI, and breaks the moment someone renames the role or introduces a second role that should also be allowed.
- **The check lives on the controller**, so it only fires for HTTP callers. Any background job, distributed-event handler, integration service, or in-process call from another module that resolves `IOrderAppService` (or its concrete) skips the check entirely. The authorization boundary in ABP is the application service because *every* caller routes through it.

Replace with a declarative `[Authorize(OrderPermissions.Create)]` attribute on the AppService method, with `OrderPermissions.Create` registered through an `OrderPermissionDefinitionProvider` in `*.Application.Contracts`.

### 3. `[Authorize]` at the class level is the wrong layer

Same reason as #2. Class-level `[Authorize]` on the controller only covers the HTTP entry point. Move authorization to the AppService — controller-side `[Authorize]` is redundant when the AppService has it, and the two will rot out of sync over time.

### 4. The controller is `public` and not `sealed`

Minor, but explicit-pattern controllers should be `sealed` — there's no inheritance story for an HTTP adapter, and `sealed` removes a class of accidental customization.

### 5. No `[HttpPost]` route shape verification / `CreateAsync` suffix

The action name `CreateAsync` will produce a route ending in `/CreateAsync` if any convention picks up the method name. Pin it explicitly. (ABP strips the `Async` suffix for Auto API but in an explicit controller the route comes from your attributes — keep them clean.)

## Recommended rewrite — two classes, one interface

**Contract — `*.Application.Contracts/Orders/IOrderAppService.cs`**

```csharp
public interface IOrderAppService : IApplicationService
{
    Task<OrderDto> CreateAsync(CreateOrderDto input);
}
```

**Permission constant — `*.Application.Contracts/Permissions/OrderPermissions.cs`** (registered via an `OrderPermissionDefinitionProvider`)

```csharp
public static class OrderPermissions
{
    public const string GroupName = "Orders";
    public const string Create = GroupName + ".Create";
}
```

**Application service — `*.Application/Orders/OrderAppService.cs`** (orchestration + declarative authorization lives here)

```csharp
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
            GuidGenerator.Create(),          // inherited from ApplicationService — don't inject IGuidGenerator
            input.CustomerId,
            input.Number);

        await _orderRepository.InsertAsync(order, autoSave: true);

        return ObjectMapper.Map<Order, OrderDto>(order);   // inherited too — don't inject IObjectMapper
    }
}
```

Notes on the AppService body:
- `GuidGenerator` and `ObjectMapper` are inherited from `ApplicationService` — drop the explicit `IObjectMapper` field (the original controller's injection was a smell carried over from the wrong layer).
- The `OrderManager.CreateAsync` shape (uniqueness check on order number, returns the entity without persisting) matches the Rule 3 pattern — Manager doesn't call `InsertAsync`; the AppService does.
- Throwing `BusinessException` for "order number already exists" lives in `OrderManager`, not here.

**Controller — `*.HttpApi/Orders/OrderController.cs`** (HTTP adapter, one collaborator, no logic)

```csharp
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

Notes on the controller:
- One ctor parameter, by design. No `IRepository`, no `OrderManager`, no `IObjectMapper`, no `ICurrentUser`. If you ever feel the urge to add one, the logic belongs in the AppService — add a method there and have the controller delegate.
- No `[Authorize]` — it's already on `OrderAppService.CreateAsync`, so HTTP callers, background jobs, integration services, and event handlers all get the same check.
- Implementing `IOrderAppService` on the controller keeps the two classes contractually aligned: if you add `GetAsync(Guid)` to the interface, the controller fails to compile until you add the delegating action.
- `sealed` because the controller has no inheritance story.

**Do not** collapse the two classes with `[ExposeServices(typeof(IOrderAppService), typeof(OrderController))]` — that forces the controller back into injecting the AppService's collaborators, which is exactly the smell we're removing.

## Quick checklist for the diff

- [ ] Move `IRepository<Order, Guid>`, `OrderManager`, and any business logic out of `OrderController` into `OrderAppService`.
- [ ] Drop `IObjectMapper` and `ICurrentUser` injections; use inherited `ObjectMapper` / declarative `[Authorize]`.
- [ ] Define `OrderPermissions.Create` in `*.Application.Contracts` and register it via an `OrderPermissionDefinitionProvider`.
- [ ] Replace `if (!_currentUser.IsInRole("sales")) throw new AbpAuthorizationException();` with `[Authorize(OrderPermissions.Create)]` on the AppService method.
- [ ] Make `OrderController` `sealed`, inject only `IOrderAppService`, and have actions be single-expression delegates.
- [ ] Confirm `OrderController : IOrderAppService` so interface drift fails the build.
