# OrderController review (ABP 10.3 Layered)

Skills consulted: **abp-application-layer**, **abp-authorization**, **abp-ddd**, **abp-core**.

## Top issues

### 1. Orchestration belongs in `OrderAppService`, not the controller
Per **abp-core** ("Business logic in Controllers" anti-pattern -> use Application Services) and **abp-application-layer** ("Application Service Structure"), use-case orchestration (calling `OrderManager`, inserting via the repository, mapping to a DTO) must live in an application service - not a controller. The controller is currently:

- Injecting `IRepository<Order, Guid>` (a domain-side abstraction)
- Injecting `OrderManager` (a domain service)
- Injecting `IObjectMapper` and performing entity->DTO mapping

That is exactly the shape of an `ApplicationService` body, and `AbpController` even exposes the same `GuidGenerator` / `CurrentUser` properties so the controller "looks like" it can do the job - but it shouldn't. Move the body into `OrderAppService.CreateAsync(CreateOrderDto input)`.

### 2. Controller should depend only on `IOrderAppService` (or be deleted)
In a layered ABP solution, application service interfaces inherit `IApplicationService`, which carries `[RemoteService]` - so ABP's **Auto API Controllers** already expose them as REST endpoints (**abp-application-layer** -> "Auto API Controllers"). You typically don't write a manual controller at all; if you do keep one (e.g. for a custom route or non-standard verb mapping), it should be a thin pass-through:

```csharp
[Route("api/orders")]
public class OrderController : AbpController
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

No repository, no domain service, no mapper in the controller.

### 3. Replace the imperative role check with a declarative permission attribute
The current code does:

```csharp
if (!_currentUser.IsInRole("sales"))
{
    throw new AbpAuthorizationException();
}
```

Two problems, both called out by **abp-core** and **abp-authorization**:

- **abp-core** lists "Hardcoded role checks" as a Never-Use anti-pattern; use permission-based authorization instead.
- **abp-authorization** ("Permission Definition" / "Declarative (Attribute)") prescribes declaring a permission constant in `*.Application.Contracts` and registering it in a `PermissionDefinitionProvider`, then applying `[Authorize(Permissions.X)]`.

Define the permission once:

```csharp
// In *.Application.Contracts
public static class OrderPermissions
{
    public const string GroupName = "Orders";
    public const string Default = GroupName;
    public const string Create  = Default + ".Create";
}
```

Register it in your `PermissionDefinitionProvider`, then decorate the app-service method:

```csharp
[Authorize(OrderPermissions.Create)]
public virtual Task<OrderDto> CreateAsync(CreateOrderDto input) { ... }
```

This makes the check declarative, role-agnostic, manageable from the Permission Management UI, and consistent with multi-tenancy / feature gating shown in **abp-authorization**.

### 4. Put `[Authorize]` on the AppService, not on the controller
Because the application service interface is `[RemoteService]`-exposed and auto-wired into the HTTP pipeline, **abp-application-layer** consistently shows `[Authorize(...)]` placed on the application service method (`CreateAsync`, `UpdateAsync`, ...). That single attribute then covers:

- Direct in-process callers (other modules, background jobs, tests)
- The auto-generated API controller
- Any thin manual controller delegating to the service

Putting `[Authorize]` only on the controller would leave the service callable without a permission check from other server-side code. So: drop the class-level `[Authorize]` on `OrderController` and move authorization to `[Authorize(OrderPermissions.Create)]` on `OrderAppService.CreateAsync`.

## Other smaller cleanups

- **Drop `IObjectMapper` from the controller.** Mapping is part of the use case; do it inside `OrderAppService`. **abp-application-layer** -> "Object Mapping" shows the mapper (Mapperly by default in 10.x, or AutoMapper) injected into the app service. Check which is active in this solution before adding a new mapper.
- **Drop the injected `ICurrentUser`.** `AbpController` already exposes `CurrentUser` as a base-class property (**abp-core** -> "Important Base Classes" table; **abp-authorization** -> "Current User"). Same goes for `GuidGenerator` - already a property on `AbpController` and `ApplicationService`, so the explicit `GuidGenerator.Create()` should move into `OrderAppService` (or, better, into `OrderManager.CreateAsync` which is the right place per **abp-ddd** -> "Domain Services").
- **`GuidGenerator.Create()` placement.** **abp-ddd** says don't generate the GUID inside the entity constructor; pass it in from outside. The cleanest spot for `Order` is inside `OrderManager.CreateAsync` (a `DomainService` already has `GuidGenerator`) - the app service then just calls `_orderManager.CreateAsync(input.CustomerId, input.Number)` with no id parameter, matching the `BookManager.CreateAsync` shape in **abp-application-layer**.
- **`autoSave: true` on `InsertAsync`.** In a layered solution the application service method runs inside ABP's ambient Unit of Work, so the `autoSave: true` flag is usually unnecessary (and slightly misleading - it suggests there is no UoW). Remove it unless you specifically need the id materialised before the UoW commits.
- **Repository choice.** `IRepository<Order, Guid>` is fine for plain CRUD (**abp-core** -> "Repository Usage"), but since `Order` is an aggregate root with non-trivial queries likely (by number, by customer), consider a custom `IOrderRepository : IRepository<Order, Guid>` in the Domain project per **abp-ddd** -> "Repository Pattern".
- **`CreateOrderDto` validation.** Add `[Required]` / `[StringLength]` on `Number` and `[Required]` on `CustomerId` (**abp-application-layer** -> "Validation") so ABP's automatic validation rejects bad input before the app service runs.

## Target shape

```csharp
// *.Application
[Authorize(OrderPermissions.Create)]
public virtual async Task<OrderDto> CreateAsync(CreateOrderDto input)
{
    var order = await _orderManager.CreateAsync(input.CustomerId, input.Number);
    await _orderRepository.InsertAsync(order);
    return _orderMapper.MapToDto(order);
}
```

And typically **no manual `OrderController` at all** - ABP's Auto API Controllers expose `/api/app/order/...` automatically from `IOrderAppService`. Only keep a manual controller if you need to override the route or wire a non-standard verb, and make it a one-line delegator as shown above.
