# Code review — `OrderController` in an ABP 10.3 Layered solution

There are quite a few things I'd change here. The controller is doing the application service's job: it's holding the business collaborators (`OrderManager`, `IRepository<Order, Guid>`, `IObjectMapper`, `ICurrentUser`), running an orchestration sequence (manager → insert → map), and even doing an imperative role check. In ABP's DDD layering, all of that belongs one layer down. The controller's only job is to be a thin HTTP adapter.

Concretely:

## 1. Wrong dependencies — controllers inject one thing only

A controller in this pattern should constructor-inject **exactly one** collaborator: the application-service interface (`IOrderAppService`). Everything you've injected is forbidden in a controller:

- `IRepository<Order, Guid>` — data access belongs in the application/infrastructure layer, not in the presentation layer. The presentation layer is "completely isolated from the domain layer" in ABP's DDD model; every cross-layer call routes through the app service.
- `OrderManager` — domain services orchestrate invariants for the *application service* to call, not for a controller. Putting them in a controller leaks the domain into HTTP.
- `IObjectMapper` — entity → DTO mapping is part of the orchestration the app service owns. The controller should never see the entity.
- `ICurrentUser` — fine to *inject* in principle, but not for business decisions like role checks (see point 3). For declarative `[Authorize]` you don't even need it.

If you feel the urge to inject any of these into a controller, that's the signal that the logic belongs in `OrderAppService`. Add a method there and have the controller delegate.

## 2. The orchestration belongs in `OrderAppService`

Move the entire body of `CreateAsync` — the manager call, the `InsertAsync`, the `ObjectMapper.Map` — into `OrderAppService.CreateAsync`. The app service inherits `GuidGenerator` and `ObjectMapper` from `ApplicationService`, so you don't need to manually inject `IObjectMapper`, and `GuidGenerator.Create()` works there directly.

```csharp
public class OrderAppService : ApplicationService, IOrderAppService
{
    private readonly OrderManager _orderManager;
    private readonly IRepository<Order, Guid> _orderRepository;

    public OrderAppService(OrderManager orderManager, IRepository<Order, Guid> orderRepository)
    {
        _orderManager = orderManager;
        _orderRepository = orderRepository;
    }

    [Authorize(OrdersPermissions.Create)]   // see point 3
    public async Task<OrderDto> CreateAsync(CreateOrderDto input)
    {
        var order = await _orderManager.CreateAsync(
            GuidGenerator.Create(), input.CustomerId, input.Number);
        await _orderRepository.InsertAsync(order, autoSave: true);
        return ObjectMapper.Map<Order, OrderDto>(order);
    }
}
```

The split is two concrete classes that both implement `IOrderAppService`: the `OrderAppService` (orchestration, the *real* implementation) and the `OrderController` (HTTP adapter). Don't collapse them with `[ExposeServices(typeof(IOrderAppService), typeof(OrderController))]` — that brings all the forbidden dependencies right back into the controller.

## 3. Replace the imperative role check with declarative authorization

```csharp
if (!_currentUser.IsInRole("sales"))
{
    throw new AbpAuthorizationException();
}
```

This is the wrong tool in three ways:

- **Hard-coded role string `"sales"`** — coupling business logic to a role name. ABP's authorization model is permission-based: define a permission (e.g. `OrdersPermissions.Create`) in a `PermissionDefinitionProvider` in `*.Application.Contracts`, then grant it to whichever role(s) the operation requires through the Permission Management UI or seeders. That way "who can create an order" is admin-configurable rather than a `git diff`.
- **Imperative check inside the action body** — should be a declarative `[Authorize(OrdersPermissions.Create)]` attribute on the app-service method (or, if you really want it on the controller side, on the controller action). The framework runs the check before your code does, returns a structured 401/403, and the policy is visible at the signature.
- **`ICurrentUser` for business decisions in a controller** — even if you wanted programmatic checks, those belong in the app service via the inherited `AuthorizationService` (`await AuthorizationService.CheckAsync(OrdersPermissions.Create)`), not in the controller. Reserve controller-level `ICurrentUser` for "I literally cannot express this declaratively" — and that is rarely true for a role/permission gate.

If for some reason you genuinely cannot map this to a permission and must keep a role gate, the attribute form is `[Authorize(Roles = "sales")]` — but a permission is the right model.

## 4. The corrected controller — single delegating expression

After the changes above, the controller becomes a one-liner per action with route metadata only:

```csharp
[Authorize]
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

Notes on what changed:

- One injected dependency: `IOrderAppService`.
- Action body is a single delegating expression — no business logic, no `await` ceremony (just return the `Task`).
- No imperative role check; the permission check is declared on `OrderAppService.CreateAsync` (or, if you prefer it co-located with the route, also on the action — but don't duplicate; pick one layer and put it there).
- `sealed` because the controller isn't an extension point.
- `IOrderAppService` is implemented by both this controller and `OrderAppService` — that's the explicit-controller pattern's contract; it keeps the HTTP shape and the service shape in sync at compile time.

## 5. A couple of smaller things while we're here

- **`autoSave: true`** on `InsertAsync` is fine but worth a note: ABP's UoW would commit at the end of the request anyway, and `autoSave: true` forces an immediate flush. Use it when you need the generated identity *before* control returns (e.g. to publish a domain event with the persisted id), otherwise drop it and let the UoW handle it.
- **`OrderManager` injecting tracking `IRepository<Order, Guid>`** — confirm in your `OrderManager` that it isn't itself calling `InsertAsync`/`UpdateAsync`/`DeleteAsync` and isn't injecting a tracking `IRepository<T, TKey>` for entities it merely needs to *read*. The Manager should hold `IReadOnlyRepository<T, TKey>` only; the application service holds the tracking repository and persists. (Can't tell from the snippet whether your `OrderManager` does this right, but it's the matching half of the rule above.)
- **`AbpController` is fine** as the base class here. If you have a project-specific controller base (e.g. `MyAppController : AbpController`) for shared cross-cutting concerns — global filters, common produces-attributes, error envelope — prefer that.

## TL;DR — checklist of changes

1. Remove `IRepository<Order, Guid>`, `OrderManager`, `IObjectMapper`, `ICurrentUser` from the controller's constructor.
2. Inject only `IOrderAppService`.
3. Move the three lines of orchestration (manager call, `InsertAsync`, `Map`) into `OrderAppService.CreateAsync`.
4. Delete the imperative `IsInRole("sales")` check and replace it with a declarative `[Authorize(OrdersPermissions.Create)]` on the app-service method, where `OrdersPermissions.Create` is a permission you define in `Acme.Orders.Application.Contracts`.
5. Reduce the action body to `=> _orderAppService.CreateAsync(input);`.
6. Mark the controller `sealed` and have it implement `IOrderAppService`.

That gives you a controller that is purely an HTTP adapter — routing, verb, OpenAPI shape, and one declarative `[Authorize]` — with all logic and authorization decisions living one layer down where they belong.
