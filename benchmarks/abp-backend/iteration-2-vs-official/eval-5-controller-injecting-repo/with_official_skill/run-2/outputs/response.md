# Code Review: `OrderController` (ABP 10.3, Layered Template)

This controller mixes responsibilities that belong in different layers of an ABP layered solution and reinvents work the framework already does. Below are the concrete issues, ranked by impact, with citations from the official ABP skills.

---

## 1. Controller is doing application-layer work — move it to an `OrderAppService`

`OrderController` is currently injecting `IRepository<Order, Guid>`, `OrderManager` (a domain service), `IObjectMapper`, and `ICurrentUser`, and orchestrating: GUID generation, domain-service call, repository insert, DTO mapping. That is the textbook description of an **application service**, not a controller.

Per `abp-dependency-rules` (Layered Template Structure):

> ```
> Application.Contracts → App service interfaces, DTOs
>        ↑
>   Application    → App service implementations
>        ↑
>    HttpApi       → REST controllers (optional)
> ```
> `HttpApi | Contracts only | Host`

And from the same skill's "Always Do" example:

> ```csharp
> public class BookController : AbpController
> {
>     private readonly IBookAppService _bookAppService; // ✅ CORRECT
> }
> ```

The "Never Do" list explicitly calls out controllers that depend on application implementation rather than the contracts interface. Injecting `IRepository<Order, Guid>` and `OrderManager` directly into a controller is worse — it bypasses the application layer entirely and pulls Domain types into HttpApi. `HttpApi` should reference `Application.Contracts` only.

`abp-core` reinforces this with its base-class table — `ApplicationService` is the layer designed for "Use case orchestration", not `AbpController`.

**Fix**: Create `IOrderAppService` in `*.Application.Contracts` and `OrderAppService : ApplicationService, IOrderAppService` in `*.Application`. Move the body of `CreateAsync` there. In most cases you can then **delete the controller entirely** — ABP's auto API controllers (see `abp-application-layer` → "Auto API Controllers") will expose `IOrderAppService` over HTTP for free:

> ABP automatically generates API controllers for application services:
> - Interface must inherit `IApplicationService` (which already has `[RemoteService]` attribute)
> - HTTP methods determined by method name prefix (Get, Create, Update, Delete)

If you keep a hand-written controller for some reason, it should be a thin pass-through that delegates to `IOrderAppService` and lives in `*.HttpApi`.

---

## 2. Replace the imperative role check with a declarative permission

```csharp
if (!_currentUser.IsInRole("sales"))
{
    throw new AbpAuthorizationException();
}
```

This is wrong on three counts:

**(a) Use permissions, not hard-coded role names.** ABP's authorization model is permission-based; roles are just one way permissions are granted. A magic string `"sales"` cannot be managed through the Permission Management UI, cannot be re-granted to another role, and can't be checked from the UI via `abp.auth.isGranted(...)`. `abp-authorization` shows the canonical shape:

> ```csharp
> public static class BookStorePermissions
> {
>     public const string GroupName = "BookStore";
>     public static class Books
>     {
>         public const string Default = GroupName + ".Books";
>         public const string Create = Default + ".Create";
>         ...
>     }
> }
> ```

Define `OrderPermissions.Orders.Create` in `*.Application.Contracts` and register it in an `OrderPermissionDefinitionProvider`.

**(b) Use the declarative `[Authorize(...)]` attribute, not an `if` check.** From `abp-authorization` — "Declarative (Attribute)":

> ```csharp
> [Authorize(BookStorePermissions.Books.Create)]
> public virtual async Task<BookDto> CreateAsync(CreateBookDto input)
> {
>     // Only users with Books.Create permission can execute
> }
> ```

`abp-application-layer` shows the same pattern on `CreateAsync` in `BookAppService`. The current code hand-rolls what ABP's authorization filter does for you, and it does it after method entry (instead of before the model binder/validator). A class-level `[Authorize]` only checks authentication; you still need the per-action permission attribute.

**(c) Apply the attribute at the application-service layer, not the controller.** Once the logic moves to `OrderAppService`, the `[Authorize(OrderPermissions.Orders.Create)]` attribute goes on `OrderAppService.CreateAsync`. That way the check runs whether the method is called via the auto-generated API, from a Razor Page model, a Blazor component, an integration test, or a background job — not only when traffic happens to come through this controller.

---

## 3. Drop the injected `ICurrentUser` and `IObjectMapper`; use base-class properties

`abp-core` ("Important Base Classes") lists what `AbpController` / `ApplicationService` already provide as properties:

> | Property | Available In | Description |
> |----------|--------------|-------------|
> | `GuidGenerator` | All base classes | Generate GUIDs |
> | `CurrentUser` | All base classes | Authenticated user info |
> | `L` (StringLocalizer) | `ApplicationService`, `AbpController` | Localization |
> | `AuthorizationService` | `ApplicationService`, `AbpController` | Permission checks |

And from `abp-application-layer` best practices:

> Use base class properties (`Clock`, `CurrentUser`, `GuidGenerator`, `L`) instead of injecting these services

So `ICurrentUser` should not be in the constructor at all (the code already uses `GuidGenerator` from the base class — be consistent). For mapping, prefer Mapperly (the default in 10.3 per `abp-application-layer` → "Object Mapping"):

> ABP supports **both Mapperly and AutoMapper** integrations. But the default mapping library is Mapperly. ... Mapperly generates mapping code at compile-time, providing better performance than runtime mappers.

If the solution is on AutoMapper, `IObjectMapper` is fine in the app service; otherwise inject an `OrderMapper` partial-class mapper.

---

## 4. Don't pass `autoSave: true` from the app service; let the unit of work commit

`abp-application-layer` ("Application Service Best Practices") notes:

> Call `UpdateAsync` explicitly (don't assume change tracking)

but it doesn't tell you to bypass the UoW. ABP application services run inside an ambient unit of work that commits at the end of the method; `autoSave: true` on the insert forces an immediate SQL round-trip and short-circuits that. Remove the `autoSave: true` argument from `InsertAsync` unless you specifically need the entity to be persisted mid-method (e.g., to get a DB-generated key — which doesn't apply here because the Guid is generated client-side).

---

## 5. Method naming and DTO shape

From `abp-application-layer` ("Anti-Patterns to Avoid" and "DTO Naming Conventions"):

> - **Entity name in method**: use `GetAsync` not `GetBookAsync`
> | Create input | `Create{Entity}Dto` | `CreateBookDto` |

`CreateAsync(CreateOrderDto input)` follows convention, good. Make sure `CreateOrderDto` lives in `*.Application.Contracts` (so Blazor/HttpApi.Client and any UI can share it):

> Define DTOs in `*.Application.Contracts` project — This allows sharing with clients (Blazor, HttpApi.Client)

---

## 6. Domain model concern (call out for the reviewer)

The controller calls `_orderManager.CreateAsync(id, customerId, number)` and then `_orderRepository.InsertAsync`. That two-step is fine, but per `abp-ddd` ("Anti-Patterns to Avoid"):

> - **Generating GUID in entity constructor**: use `IGuidGenerator` from outside and pass `id` parameter
> - **Domain service depending on current user**: accept values from the application layer instead

You're doing both correctly (generating the GUID in the caller, not the entity; not having `OrderManager` reach for `ICurrentUser`). Keep it that way — when the logic moves to `OrderAppService`, continue to pass `GuidGenerator.Create()` and any `CreatorId` from `CurrentUser.GetId()` as parameters to `OrderManager`, rather than having the domain service look them up itself.

---

## Refactored shape

**`*.Application.Contracts/Permissions/OrderPermissions.cs`**
```csharp
public static class OrderPermissions
{
    public const string GroupName = "Ordering";

    public static class Orders
    {
        public const string Default = GroupName + ".Orders";
        public const string Create  = Default + ".Create";
    }
}
```

**`*.Application.Contracts/Orders/IOrderAppService.cs`**
```csharp
public interface IOrderAppService : IApplicationService
{
    Task<OrderDto> CreateAsync(CreateOrderDto input);
}
```

**`*.Application/Orders/OrderAppService.cs`**
```csharp
[Authorize(OrderPermissions.Orders.Default)]
public class OrderAppService : ApplicationService, IOrderAppService
{
    private readonly IRepository<Order, Guid> _orderRepository;
    private readonly OrderManager _orderManager;
    private readonly OrderMapper _orderMapper; // Mapperly; or IObjectMapper if AutoMapper

    public OrderAppService(
        IRepository<Order, Guid> orderRepository,
        OrderManager orderManager,
        OrderMapper orderMapper)
    {
        _orderRepository = orderRepository;
        _orderManager    = orderManager;
        _orderMapper     = orderMapper;
    }

    [Authorize(OrderPermissions.Orders.Create)]
    public async Task<OrderDto> CreateAsync(CreateOrderDto input)
    {
        var order = await _orderManager.CreateAsync(
            GuidGenerator.Create(), input.CustomerId, input.Number);

        await _orderRepository.InsertAsync(order);

        return _orderMapper.MapToDto(order);
    }
}
```

With this in place you generally do **not** need an `OrderController` at all — ABP's auto API controllers expose `IOrderAppService` at `/api/app/order` automatically. If you require the custom route `api/orders`, keep a thin controller in `*.HttpApi`:

```csharp
[Authorize]
[Route("api/orders")]
[RemoteService(Name = "Default")]
public class OrderController : AbpController, IOrderAppService
{
    private readonly IOrderAppService _orderAppService;

    public OrderController(IOrderAppService orderAppService)
        => _orderAppService = orderAppService;

    [HttpPost]
    public Task<OrderDto> CreateAsync(CreateOrderDto input)
        => _orderAppService.CreateAsync(input);
}
```

No business logic, no domain types, no role strings, no `ICurrentUser`, no `GuidGenerator` — just a delegation. The permission attribute stays on the app service so every caller benefits.

---

## Summary of changes

| # | Issue | Fix | Skill citation |
|---|------|-----|----------------|
| 1 | Controller orchestrating domain + repo + mapper | Move to `OrderAppService`; controller (if any) only delegates | `abp-dependency-rules`, `abp-application-layer` (Auto API Controllers) |
| 2a | Hard-coded role `"sales"` | Define `OrderPermissions.Orders.Create` in `Application.Contracts` | `abp-authorization` (Permission Definition) |
| 2b | Imperative `IsInRole` check | Declarative `[Authorize(OrderPermissions.Orders.Create)]` | `abp-authorization` (Declarative) |
| 2c | Authorization in controller | Put the attribute on the **app service** method | `abp-application-layer` (`CreateAsync` example) |
| 3 | Injecting `ICurrentUser`/`IObjectMapper` | Use base-class `CurrentUser`/`GuidGenerator`; use `OrderMapper` (Mapperly) | `abp-core` (base classes), `abp-application-layer` (mapping) |
| 4 | `autoSave: true` | Remove — let the ambient UoW commit | `abp-application-layer` |
| 5 | DTO location | Keep `CreateOrderDto`/`OrderDto` in `Application.Contracts` | `abp-application-layer` (DTO Location) |
| 6 | Domain-service shape (pre-existing, OK) | Continue passing `id` and user data into `OrderManager` from outside | `abp-ddd` (Anti-Patterns) |
