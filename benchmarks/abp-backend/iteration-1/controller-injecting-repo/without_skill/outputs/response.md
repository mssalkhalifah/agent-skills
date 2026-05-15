There's quite a bit I'd push back on here. The code works, but it's fighting the layered template's design in a few ways. Let me walk through the issues from biggest to smallest.

## 1. Don't write controllers like this in a Layered solution — write an application service

This is the biggest one. In an ABP Layered solution, you almost never hand-write MVC controllers. You write an application service (`OrderAppService : ApplicationService, IOrderAppService`) and let ABP's auto-API-controller feature expose it as `/api/app/order/...` automatically (configured via `ConfigureConventionalControllers` in your `*HttpApiModule`).

The benefits you're throwing away by writing the controller manually:

- Auto-generated dynamic JS/C# proxies on the client side.
- Automatic unit-of-work, audit logging, exception handling, validation, and authorization filters that the application-service convention applies.
- A clean place for your domain orchestration that's reusable from a background job, an integration event handler, or a test — not coupled to HTTP.
- Auto-API discovers `IApplicationService` implementations; controllers don't get the same proxy generation treatment.

If you genuinely need a custom controller (file uploads, non-standard routes, SignalR negotiation, etc.), inherit from `AbpControllerBase` (or `AbpController` for MVC views) and put the controller in the `*.HttpApi` project — but then the controller should still call into an application service, not the repository or the manager directly.

## 2. The controller is reaching past the application layer

`OrderManager` is a domain service. `IRepository<Order, Guid>` is a domain abstraction. Controllers live in the presentation/HTTP layer and should depend on the **application** layer (`IOrderAppService`), not on domain services or repositories. That mixing is what makes Layered solutions degrade into mud over time.

## 3. Authorization should be declarative, not an `if` in the action body

```csharp
if (!_currentUser.IsInRole("sales"))
{
    throw new AbpAuthorizationException();
}
```

This is the wrong layer and the wrong mechanism:

- **Use permissions, not role checks.** Role names are deployment data; permissions are part of your design. Define `OrderPermissions.Create` in your `*.Application.Contracts` project, register it in your `PermissionDefinitionProvider`, grant it to the `sales` role at seed time, then decorate the method with `[Authorize(OrderPermissions.Create)]`. Now an admin can re-grant the permission to a different role without a code change.
- **Use `[Authorize]` attributes on the method**, not imperative checks. The class-level `[Authorize]` you already have only requires authentication; it doesn't enforce a specific permission.
- If you really must check imperatively, inject `IAuthorizationService` and call `AuthorizeAsync(...)`, don't read `ICurrentUser.Roles` directly.

## 4. Domain object construction is leaking into the application/HTTP layer

```csharp
var order = await _orderManager.CreateAsync(GuidGenerator.Create(), input.CustomerId, input.Number);
await _orderRepository.InsertAsync(order, autoSave: true);
```

The `OrderManager.CreateAsync` is presumably your factory, which is good — but then this layer is also picking the Id and inserting. Two issues:

- **The manager should generate the Id.** Inject `IGuidGenerator` into `OrderManager` and let it call `GuidGenerator.Create()` internally. The caller shouldn't know that an `Order` needs a GUID.
- **The manager (or app service) should persist.** It's perfectly fine for the manager to do `return await _orderRepository.InsertAsync(order)`, or for the app service to call `InsertAsync` after the manager validates/builds the aggregate. Splitting "build" and "insert" across two layers is what created this awkward two-line dance.
- **`autoSave: true` is a smell inside a unit of work.** ABP application services already wrap each call in a UoW that commits at the end. Forcing `autoSave: true` defeats that — useful only when you need the generated Id mid-method (e.g., to raise an event referring to the new entity), which isn't the case here.

## 5. `CreateOrderDto` isn't being validated

There are no `[Required]` / `[StringLength]` checks visible here, and the action doesn't have `[ValidateModel]` (it's automatic in app services, not in raw controllers in some configurations). Move to an app service and ABP runs `IObjectValidator` on the input automatically; otherwise, decorate the controller method with `[Authorize]`-style validation or rely on `ApiController` model validation.

## 6. Minor naming / API issues

- `[Route("api/orders")]` — fine if you keep the controller, but combined with auto-API conventions on an app service this would just be `/api/app/order/`. Pick one style across the solution.
- The class is missing `[ApiController]` (it's not inherited through `AbpController`). Without it you don't get automatic 400 on model-binding failures or `[FromBody]` inference.
- `OrderController` should probably be `OrdersController` (REST convention: plural resource).
- `CreateAsync` is a fine method name internally, but the HTTP route ends up as `POST api/orders` — make sure your DTO and return type live in `Application.Contracts`.

## What this should look like

**`*.Application.Contracts/Orders/IOrderAppService.cs`:**

```csharp
public interface IOrderAppService : IApplicationService
{
    Task<OrderDto> CreateAsync(CreateOrderDto input);
}
```

**`*.Application.Contracts/Permissions/OrderPermissions.cs`:**

```csharp
public static class OrderPermissions
{
    public const string GroupName = "Orders";
    public const string Default = GroupName;
    public const string Create  = Default + ".Create";
}
```

…registered in your `PermissionDefinitionProvider`, and granted to the `sales` role in your data-seeder.

**`*.Application/Orders/OrderAppService.cs`:**

```csharp
[Authorize(OrderPermissions.Default)]
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
        var order = await _orderManager.CreateAsync(input.CustomerId, input.Number);
        await _orderRepository.InsertAsync(order);
        return ObjectMapper.Map<Order, OrderDto>(order);
    }
}
```

Note that `ApplicationService` already exposes `ObjectMapper`, `CurrentUser`, `GuidGenerator`, `Logger`, etc. as properties — no need to inject them. Same goes for `AbpController`, by the way; you don't need to inject `IObjectMapper` and `ICurrentUser` in your original code, just use the inherited `ObjectMapper` and `CurrentUser` properties.

**`*.Domain/Orders/OrderManager.cs`:**

```csharp
public class OrderManager : DomainService
{
    private readonly IRepository<Order, Guid> _orderRepository;

    public OrderManager(IRepository<Order, Guid> orderRepository)
    {
        _orderRepository = orderRepository;
    }

    public async Task<Order> CreateAsync(Guid customerId, string number)
    {
        // domain invariants: e.g. number uniqueness check
        if (await _orderRepository.AnyAsync(o => o.Number == number))
        {
            throw new BusinessException(OrderErrorCodes.DuplicateNumber);
        }

        return new Order(GuidGenerator.Create(), customerId, number);
    }
}
```

`DomainService` exposes `GuidGenerator` the same way, so the manager owns Id creation.

**`*.Application/OrderAutoMapperProfile.cs`** (or wherever your profile lives):

```csharp
CreateMap<Order, OrderDto>();
```

**No controller needed** — `ConfigureConventionalControllers` in your `*HttpApiModule` (already there in the layered template) auto-exposes `IOrderAppService` as `POST /api/app/order` with full proxy support.

## TL;DR

- Move the logic to an `OrderAppService`; delete the controller.
- Replace the `IsInRole` check with an ABP **permission** and `[Authorize(...)]`.
- Drop the injected `IObjectMapper` / `ICurrentUser` — `AbpController` and `ApplicationService` already expose them.
- Let `OrderManager` generate the Id via its inherited `GuidGenerator`.
- Drop `autoSave: true` and let the ambient UoW commit.
- Plural the controller name and add `[ApiController]` if you keep any custom controller around.
