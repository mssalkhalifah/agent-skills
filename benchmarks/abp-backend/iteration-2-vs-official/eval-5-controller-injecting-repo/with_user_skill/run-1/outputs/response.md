# Code review — `OrderController` (ABP 10.3 Layered)

This controller violates the explicit-controller pattern (Rule 4) in several load-bearing ways. The fix is a structural split, not a tweak — the controller should become a thin HTTP adapter and all orchestration/authorization should move into an `OrderAppService`.

## What's wrong

1. **One class is doing two jobs.** `OrderController` inherits `AbpController` *and* contains application logic (id minting, manager call, repository insert, mapping). In the explicit two-class pattern the controller is the HTTP adapter; orchestration belongs in `OrderAppService : ApplicationService, IOrderAppService`. Both classes implement the same `IOrderAppService`.

2. **The controller injects collaborators it must not see.** Rule 4 forbids injecting anything other than `IXxxAppService` into a controller. Today it pulls in:
   - `IRepository<Order, Guid>` — repository access belongs in the AppService (Rule 2: presentation isolated from domain).
   - `OrderManager` — the domain service is an Application-layer collaborator, not a controller one.
   - `IObjectMapper` — mapping lives in the AppService (and `ApplicationService` already exposes `ObjectMapper` as a property — no injection needed).
   - `ICurrentUser` for business decisions — declarative `[Authorize]` only; `ICurrentUser` should not drive authorization from a controller (and `ApplicationService` exposes it as `CurrentUser` if you do need claim reads inside orchestration).

   After the split, the controller's constructor injects exactly **one** dependency: `IOrderAppService`.

3. **Imperative role check instead of declarative permission.** `if (!_currentUser.IsInRole("sales")) throw new AbpAuthorizationException();` is wrong on two axes:
   - **Roles aren't the authorization unit — permissions are.** Roles are an *assignment dimension*: which roles get which permissions is an admin/runtime concern. Hard-coding a role name into code couples the deployment's role taxonomy into your binary, and you lose the ability to grant the capability to a different role, a specific user, or a client without a code change. Even `[Authorize(Roles="sales")]` would be wrong for the same reason — replace with a permission constant.
   - **Imperative check vs declarative attribute.** Use `[Authorize(OrderPermissions.Create)]` and let ABP's authorization pipeline enforce it. The permission constant lives in `Acme.Sales.Application.Contracts` (next to `IOrderAppService`), registered through a `SalesPermissionDefinitionProvider`.

4. **`[Authorize]` is on the controller, not the AppService.** Even if you swap the role check for a permission check, putting the attribute *only* on the controller covers just the HTTP path. The AppService is the canonical authorization boundary because every caller routes through it — HTTP via the controller, background jobs, integration services, distributed-event handlers, in-process callers from other modules. Put `[Authorize(OrderPermissions.Create)]` on the **AppService method** (or class). Don't duplicate it on the controller — it's redundant and rots over time as the two attributes drift.

5. **Minor:** `GuidGenerator.Create()` belongs in the AppService (inherited from `ApplicationService`); the controller has no business minting ids. Same for `ObjectMapper`. Once the split is done, both fall away naturally.

## Refactored shape

```csharp
// Acme.Sales.Application.Contracts/Permissions/SalesPermissions.cs
public static class SalesPermissions
{
    public const string GroupName = "Sales";

    public static class Orders
    {
        public const string Default = GroupName + ".Orders";
        public const string Create  = Default + ".Create";
        public const string Edit    = Default + ".Edit";
        public const string Delete  = Default + ".Delete";
    }
}

// Acme.Sales.Application.Contracts/Permissions/SalesPermissionDefinitionProvider.cs
public class SalesPermissionDefinitionProvider : PermissionDefinitionProvider
{
    public override void Define(IPermissionDefinitionContext context)
    {
        var group = context.AddGroup(SalesPermissions.GroupName,
            LocalizableString.Create<SalesResource>("Permission:Sales"));

        var orders = group.AddPermission(SalesPermissions.Orders.Default,
            LocalizableString.Create<SalesResource>("Permission:Orders"));
        orders.AddChild(SalesPermissions.Orders.Create,
            LocalizableString.Create<SalesResource>("Permission:Orders.Create"));
    }
}

// Acme.Sales.Application.Contracts/Orders/IOrderAppService.cs
public interface IOrderAppService : IApplicationService
{
    Task<OrderDto> CreateAsync(CreateOrderDto input);
}

// Acme.Sales.Application/Orders/OrderAppService.cs — orchestration + authorization live HERE
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
        // No IObjectMapper / ICurrentUser / IGuidGenerator injection —
        // ApplicationService exposes ObjectMapper, CurrentUser, GuidGenerator already.
    }

    [Authorize(SalesPermissions.Orders.Create)]
    public async Task<OrderDto> CreateAsync(CreateOrderDto input)
    {
        var order = await _orderManager.CreateAsync(
            GuidGenerator.Create(), input.CustomerId, input.Number);
        await _orderRepository.InsertAsync(order, autoSave: true);
        return ObjectMapper.Map<Order, OrderDto>(order);
    }
}

// Acme.Sales.HttpApi/Controllers/OrderController.cs — pure HTTP adapter
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

## Why each change matters (short form)

| Issue | Fix | Why |
|---|---|---|
| Controller owns logic | Move to `OrderAppService : IOrderAppService`; controller delegates | Rule 4: presentation is isolated; only HTTP shape lives in the controller. |
| `IRepository<Order,Guid>` injected into controller | Inject into AppService instead | Repositories are Application/Domain concerns; controllers must not touch persistence. |
| `OrderManager` injected into controller | Inject into AppService instead | Domain services are orchestrated by the AppService, never by the controller. |
| `IObjectMapper` injected | Use `ApplicationService.ObjectMapper` in the AppService | Mapping is an Application-layer concern; base class already provides the property. |
| `ICurrentUser` + `IsInRole("sales")` | Remove; rely on `[Authorize(SalesPermissions.Orders.Create)]` | Permissions are the authorization unit; roles are assignment. Declarative > imperative. |
| `[Authorize]` only on controller (currently bare `[Authorize]`) | Put `[Authorize(SalesPermissions.Orders.Create)]` on the **AppService** method | AppService is the canonical authorization boundary — every caller (HTTP, jobs, integration, events, in-process) routes through it. Controller-only checks miss non-HTTP callers. |
| `GuidGenerator.Create()` in controller | Use inherited `GuidGenerator` in AppService | Id minting is orchestration; ApplicationService provides it without injection. |

## Things to double-check while you're in there

- **`OrderManager` shouldn't be persisting.** Confirm `_orderManager.CreateAsync(...)` returns the `Order` and lets the AppService call `InsertAsync` — that's the shape shown above (Rule 3). If the manager is calling `InsertAsync` internally, fix that too.
- **`autoSave: true` on `InsertAsync`.** Fine inside an AppService method (UoW boundary), but redundant if the unit of work commits at the end of the request anyway. Leave it if you rely on the inserted entity having its DB-generated values for the mapping step; otherwise it's noise.
- **Don't collapse the two classes with `[ExposeServices(typeof(IOrderAppService), typeof(OrderController))]`.** That trick forces the controller to inject the AppService's collaborators — exactly the dependencies Rule 4 forbids. Keep them as two registered classes.
- **`OrderController` should be `sealed`** (and live in `Acme.Sales.HttpApi`, not the Application project). Permission constants and `IOrderAppService` live in `Acme.Sales.Application.Contracts`.
