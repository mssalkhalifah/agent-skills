---
name: abp-backend
description: ABP Framework backend guidance for ABP 10.3 .NET solutions — Single-Layer, Layered, or Microservice templates; AbpModule wiring, application services, domain services, IRepository / IReadOnlyRepository, EF Core DbContext + migrations, OpenIddict in ABP, multi-tenancy, authorization, validation, API development, modularity, application modules (Identity, Account, OpenIddict, SaaS, Forms, CMS, BlobStoring), infrastructure (background jobs, distributed events, blob storing, caching/locking), and the abp CLI / Studio / Suite. Trigger whenever editing C# in an ABP solution, scaffolding with `abp new`, picking modules, writing or refactoring an *AppService / *Manager / *DbContext, configuring per-module migrations, wiring multi-tenancy, writing ABP integration tests, or answering ABP concept questions — even when the user mentions ABP primitives without naming the framework. Do NOT trigger on plain ASP.NET Core / EF Core / OpenIddict / .NET DI questions that explicitly disclaim ABP.
allowed-tools: Read, Glob, Bash(abp:*), Bash(dotnet --version), Bash(dotnet --info), Bash(dotnet ef:*)
---

# ABP Framework — backend guidance (ABP 10.3)

This skill is template-aware backend guidance for the .NET side of an ABP 10.3 solution. It covers the three first-party solution templates — Single-Layer (`app-nolayers`), Layered (`app`), Microservice (`microservice`) — plus the cross-cutting framework: DDD patterns, fundamentals (DI, modules, options, exceptions, localization, logging), modularity, multi-tenancy, authorization, validation, API development, EF Core data access, ABP application modules, infrastructure subsystems (background processing, blob storing, eventing, caching/locking), the `abp` CLI, ABP Studio, ABP Suite, and integration testing.

UI/frontend concerns (MVC/Razor Pages, Blazor, Angular, React Native, MAUI, themes) live in the peer skill **`abp-frontend`**. If the user's question is purely about UI bindings, navigation menus, theme configuration, or client-side proxies, defer to that skill instead.

Read this file first; load focused references below only when the topic is in scope.

## How this skill is organised

- **SKILL.md** (this file) — template detection, six framework guardrails, customization contract, routing table.
- **references/templates/** — per-template references and a selection guide.
- **references/framework/** — cross-cutting framework concepts (DDD, modularity, multi-tenancy, authz, validation, API, EF Core).
- **references/modules/** — first-party ABP application modules (Identity/Account/OpenIddict, SaaS/Tenant Management, CMS/Forms/Blogging, audit/feature/setting/lepton, messaging/payment, AI/misc).
- **references/infrastructure/** — background processing, blob storing, distributed eventing, distributed caching + locking.
- **references/tools/** — `abp` CLI, ABP Studio, ABP Suite.
- **references/testing.md** — integration testing against an ABP host.
- **customization/** — sidecar contract (`README.md`) and a worked example (`example.md`).

When deciding what to load, prefer the smallest reference that answers the question. The routing table at the end of this file is the source of truth.

## Template detection — figure out which template the project uses

Most guidance differs across the three templates. Before answering a code-shaped question, infer the active template from the project layout:

| Signal in the working directory | Likely template |
|---|---|
| One `*.csproj` at the root with sibling folders `Data/`, `Entities/`, `Services/`, `Pages/`, `Permissions/`, `Localization/` | **Single-Layer** (`-t app-nolayers`) |
| `src/<Name>.Domain/`, `src/<Name>.Application/`, `src/<Name>.EntityFrameworkCore/`, `src/<Name>.HttpApi/`, `src/<Name>.Web/`, plus `test/<Name>.*Tests/` | **Layered** (`-t app`) |
| `apps/`, `gateways/`, `services/`, `etc/{docker,helm,k8s,abp-studio}/` at the top of the repo | **Microservice** (`-t microservice`) |

Do this discovery once per session via `Glob` (don't re-discover every turn — the model retains the result across turns). If signals are mixed (e.g., a Layered solution containing internal ABP modules under `modules/`), the project is most likely a **modular monolith** — that's still the Layered template; surface that interpretation when relevant.

If detection is ambiguous and the answer materially depends on the template, ask the user which one they're on rather than guessing.

## Customization contract — overrides win

This skill provides ABP-framework defaults. **Project-specific overrides take precedence** in this order:

1. **`.abp-overrides.md`** — opt-in sidecar. Walk up from the current working directory toward the filesystem root; stop at any `.git/` or `.hg/` boundary. Read every `.abp-overrides.md` found between cwd and that boundary; **deeper files override shallower** (mirrors hierarchical CLAUDE.md). Do this lookup once per session at the first ABP question; the model retains the loaded content across turns.
2. **`CLAUDE.md`** — already auto-loaded by Claude Code as project context. The skill does not parse it or look for any "ABP overrides" marker. Defer to it where it conflicts with this skill's defaults.
3. **This skill** — applies when the above are silent.

A consumer sidecar may carry an optional frontmatter for human bookkeeping:

```yaml
---
project: AcmeBookStore
abp-version: "10.3"
template: layered
---
```

The skill does not parse this frontmatter; it's only there so the consumer can track when the override file might need refreshing. See `customization/README.md` for the full sidecar template and walk-up rules.

## Documentation version pin and fallback

ABP docs URLs in references are pinned to **`https://abp.io/docs/10.3/...`**. If a `/10.3/` URL returns 404 or redirects to a generic page, retry the same path under `/docs/latest/` and treat the result as best-effort (newer behaviour may not match 10.3 exactly). Note any such fallback when citing docs to the user.

## Tooling — Bash deny-sticky rule

This skill is allowed to invoke a small set of CLI tools (`abp ...`, `dotnet --version`, `dotnet --info`, `dotnet ef ...`) to detect versions, generate proxies, run migrations, or read CLI help. **If the user denies a Bash tool call from this skill, set a session flag and stop attempting Bash for the rest of the session unless the user explicitly re-invokes the skill or asks for a CLI action by name.** Fall back to static guidance from the references for the rest of the session. The denial is a strong signal; respect it without re-asking.

## ABP/DDD layer model

ABP's [DDD docs](https://abp.io/docs/10.3/framework/architecture/domain-driven-design) define four logical layers, and each template represents them differently — but the rules are template-agnostic. The framework is explicit: *the presentation layer is completely isolated from the domain layer* — every cross-layer call routes through the application service.

| Layer | Lives here | Enforced by |
|-------|------------|-------------|
| **Presentation** | Controllers (`XxxController : <BaseController>, IXxxAppService`). HTTP routing, declarative `[Authorize]`, OpenAPI shape. Nothing else. | Rule 4 |
| **Application** | Application services (`XxxAppService : ApplicationService, IXxxAppService`), DTOs, validation, mappers, UoW boundary. Orchestrates the domain and persistence; never owns business invariants directly. | Rules 2, 4, 7 |
| **Domain** | Entities, aggregate roots, value objects, domain services (`*Manager`), repository interfaces, specifications, domain events. Owns invariants; never persists. | Rules 3, 7 |
| **Infrastructure** | `DbContext`, EF entity configuration, repository implementations, design-time factory, schema migrator. | Rule 6 |

In **Single-Layer** these are folders (`Entities/`, `Services/`, `Data/`); in **Layered** they are projects (`*.Domain`, `*.Application`, `*.EntityFrameworkCore`); in **Microservice** they are sub-folders inside each microservice's solution.

## Framework guardrails (recommended patterns)

These rules apply across all three templates. They reflect ABP's documented best practices and the broad consensus from the framework's `references/framework/architecture-ddd.md` and `references/framework/architecture-modularity.md`. Project-specific `.abp-overrides.md` may relax or extend them.

### Rule 1 — One class per file

Never declare multiple top-level classes, records, interfaces, or enums in the same `.cs` file. Split nested-but-public types, DTOs, and helper classes into their own files. Tooling (refactoring, navigation, code-coverage) and code review both work better one-class-per-file.

### Rule 2 — Never expose `IQueryable` to the application service layer

Application services must call repository methods that return materialised results (`Task<TEntity>`, `Task<List<TEntity>>`, `Task<PagedResultDto<...>>`). Encapsulate queries inside a custom repository — do **not** call `_repository.GetQueryableAsync()` from inside `*AppService.cs`. If a query is too complex for the generic repository, define a custom `IXyzRepository : IRepository<X, TKey>` in the feature folder and put its EF Core implementation alongside.

**Pick the narrowest repository interface for the job** — see [framework/data-ef-core.md](./references/framework/data-ef-core.md) and the [Repositories docs](https://abp.io/docs/10.3/framework/architecture/domain-driven-design/repositories) for the full surface of each:

| Interface | Use it for |
|-----------|------------|
| `IRepository<TEntity, TKey>` | Default. Full CRUD plus `GetQueryableAsync()` for use **inside** a custom repository implementation. |
| `IReadOnlyRepository<TEntity, TKey>` | Read-only services and query handlers. EF Core uses no-tracking queries by default — faster, signals intent. |
| `IBasicRepository<TEntity, TKey>` | Provider-agnostic modules where `IQueryable` can't be assumed. Skip unless shipping a portable module. |
| `IReadOnlyBasicRepository<TEntity, TKey>` | Minimal read surface (`GetCountAsync`, `GetListAsync`, `GetPagedListAsync`) under the same provider-agnostic constraint. |
| `IRepository<TEntity>` (no key) | Entities with composite keys or no `Id` property. |

When you need async LINQ inside a custom repository, prefer `IAsyncQueryableExecuter` over a direct dependency on EF Core's async extensions — it keeps the implementation provider-neutral.

### Rule 3 — Domain services don't persist; application services orchestrate

Domain services (`*Manager`) own business invariants and pure logic. They:
- Validate cross-aggregate rules and throw `BusinessException` when a repository check fails (uniqueness, count, sibling-aggregate guard).
- Mutate entities **in place** that the application service hands them.
- **Never** call `InsertAsync` / `UpdateAsync` / `DeleteAsync` themselves.

Application services orchestrate. They mint Ids via the `GuidGenerator` inherited from `ApplicationService` (no `IGuidGenerator` injection — same convention as `CurrentUser` and `ObjectMapper`), fetch tracked entities, pass them into the Manager, and persist after the Manager returns.

**Repository-interface rule for domain services.** A `*Manager` injects `IReadOnlyRepository<T, TKey>` only — for any read it needs, including reads of the entity it's the canonical authority over (uniqueness checks, count guards). Mutated entities arrive **as method parameters** from the application service, which holds the tracking `IRepository<T, TKey>` and persists after. Seeing `IRepository<T, TKey>` injected into a `*Manager` is a smell — either the Manager is persisting (forbidden) or the mutation should flow in as a parameter instead.

```csharp
// Domain service — IReadOnlyRepository-only injection; mutations flow in as parameters
public class OrderManager : DomainService
{
    private readonly IReadOnlyRepository<Customer, Guid> _customerRepository;
    private readonly IReadOnlyRepository<Order, Guid>    _orderRepository;

    public OrderManager(
        IReadOnlyRepository<Customer, Guid> customerRepository,
        IReadOnlyRepository<Order, Guid>    orderRepository)
    {
        _customerRepository = customerRepository;
        _orderRepository = orderRepository;
    }

    // Variant A — creating a new aggregate: Manager builds and returns it
    public async Task<Order> CreateAsync(Guid id, Guid customerId, string number)
    {
        var customer = await _customerRepository.GetAsync(customerId);
        customer.EnsureCanReceiveNewOrder();                              // entity-behavior method throws

        if (await _orderRepository.AnyAsync(o => o.Number == number))
            throw new BusinessException("Sales:OrderNumberAlreadyExists");

        return new Order(id, customerId, number);                          // Manager doesn't persist
    }

    // Variant B — mutating an existing aggregate: entity arrives by parameter
    public Task ChangePriceAsync(Order order, decimal newPrice)
    {
        if (newPrice <= 0)
            throw new BusinessException("Sales:InvalidPrice");
        order.SetPrice(newPrice);                                          // mutates in place; app service persists
        return Task.CompletedTask;
    }
}

// Application service — mints Ids, holds the tracking repository, persists
public class OrderAppService : ApplicationService, IOrderAppService
{
    private readonly OrderManager             _orderManager;
    private readonly IRepository<Order, Guid> _orderRepository;            // tracking lives HERE

    public OrderAppService(OrderManager orderManager, IRepository<Order, Guid> orderRepository)
    {
        _orderManager = orderManager;
        _orderRepository = orderRepository;
    }

    public async Task<OrderDto> CreateAsync(CreateOrderDto input)
    {
        var order = await _orderManager.CreateAsync(
            GuidGenerator.Create(),                                        // ApplicationService inherits GuidGenerator
            input.CustomerId,
            input.Number);
        await _orderRepository.InsertAsync(order, autoSave: true);
        return ObjectMapper.Map<Order, OrderDto>(order);
    }

    public async Task ChangePriceAsync(Guid id, decimal newPrice)
    {
        var order = await _orderRepository.GetAsync(id);                   // tracked fetch
        await _orderManager.ChangePriceAsync(order, newPrice);              // pass in for mutation
        await _orderRepository.UpdateAsync(order);                          // app service persists
    }
}
```

**Decision rubric:** **entity (or an entity behavior method)** when the rule reads only the entity's own state; **`*Manager`** when the rule needs a repository check (uniqueness, count, existence) or a sibling-aggregate guard; **never the application service** — those throws couple the rule to a single caller and let the entity be mutated past the rule through any other path. See [framework/architecture-ddd.md](./references/framework/architecture-ddd.md) for the entity-behavior pattern (`Customer.EnsureCanReceiveNewOrder()`) and the `internal`-vs-`public` constructor guidance.

### Rule 4 — Recommended pattern: explicit controllers (not auto-API)

ABP's default is to expose application services as REST endpoints automatically via Auto API Controllers — see [framework/api-development.md](./references/framework/api-development.md). That works, but the explicit-controller pattern below is the recommended alternative whenever you need fine-grained control over routes, HTTP verbs, file/stream responses, custom headers, or OpenAPI shape.

The explicit pattern is **two separate concrete classes** that both implement the same `IXxxAppService` interface:

- `XxxAppService : ApplicationService, IXxxAppService` — orchestration. Owns *all* collaborators (managers, repositories, mappers, integration services, current user, distributed events). Enforces runtime authorization, performs mapping, persists, returns DTOs.
- `XxxController : <YourBaseController>, IXxxAppService` — HTTP adapter. Constructor injects **exactly one** collaborator: `IXxxAppService`. Action bodies are single delegating expressions: `=> _xxxAppService.MethodAsync(...);`. Route attributes, `[Authorize]`, OpenAPI annotations live here; logic does not.

```csharp
[Authorize]
[Route("api/orders")]
public sealed class OrderController : MyBaseController, IOrderAppService
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

**Don't collapse the two classes** with `[ExposeServices(typeof(IXxxAppService), typeof(XxxController))]`. That forces the controller to inject managers, repositories, mappers, and current-user helpers — the dependencies this rule forbids in a controller. Keep them as two separate classes registered separately.

**Two hard prohibitions that fall out of this split:**

- **Never inject anything other than `IXxxAppService` into a controller.** No repositories, no `*Manager`, no `IObjectMapper`, no integration service, no `ICurrentUser` for business decisions (declarative `[Authorize]` only). If you feel the urge, the logic belongs in the app service — add a method there and have the controller delegate.
- **Never expose `IQueryable` (or `IAsyncEnumerable<T>` / `IOrderedQueryable<T>`) from a controller action.** Return materialised types only — DTOs, `PagedResultDto<T>`, `ListResultDto<T>`, `FileContentResult`, `IActionResult`, primitives. This is the controller-side mirror of Rule 2.

If the project's `.abp-overrides.md` or CLAUDE.md says "use Auto API Controllers throughout," that override wins.

### Rule 5 — Cross-module reads via ABP Integration Services

An ABP **module** is a .NET solution containing projects, designed to be consumed by other solutions. The project count varies by template:

- **Single-Layer module** — a smaller bundle: a main project + `*.Contracts` + tests. DDD principles still apply; you just have fewer project files to enforce them with.
- **Layered module** — the full DDD project set (`*.Domain.Shared`, `*.Domain`, `*.Application.Contracts`, `*.Application`, `*.EntityFrameworkCore`, `*.HttpApi`, `*.HttpApi.Client`, plus tests).
- **Microservice context** — a deployable unit is usually called a *service* (its own .NET solution that gets deployed); a *module* in this context is still a .NET solution but is designed to be imported into multiple services.

Rule 5 applies to cross-**module** reads. **For cross-aggregate reads inside the same module, use a `*Manager` with `IReadOnlyRepository<TOther, TKey>` (see Rule 3)** — that's not what Integration Services are for.

When one module needs to read data owned by another module, **do not** call its `IApplicationService` directly and **do not** add a host-level shared type. Use ABP **Integration Services** — see [Integration Services docs](https://abp.io/docs/10.3/framework/api-development/integration-services) and the in-process worked example in [framework/api-development.md](./references/framework/api-development.md).

Integration services are marked with `[IntegrationService]`, hidden from the public API surface, and audit-off by default. Define the contract in the producing module's `*.Contracts` project; consume it from the other module via DI.

```csharp
// Forms.Contracts/IFormSubmissionIntegrationService.cs — define the contract
using Volo.Abp.Application.Services;

namespace Acme.Forms;

[IntegrationService]                                    // hidden from /api, audit-off, no [Authorize] by default
public interface IFormSubmissionIntegrationService : IApplicationService
{
    Task<int> GetSubmissionCountForOwnerAsync(Guid ownerId);
}

// Forms/IntegrationServices/FormSubmissionIntegrationService.cs — implement in the producing module
public class FormSubmissionIntegrationService
    : ApplicationService, IFormSubmissionIntegrationService
{
    private readonly IRepository<FormSubmission, Guid> _submissionRepository;
    public FormSubmissionIntegrationService(IRepository<FormSubmission, Guid> r) => _submissionRepository = r;

    public Task<int> GetSubmissionCountForOwnerAsync(Guid ownerId)
        => _submissionRepository.CountAsync(s => s.OwnerId == ownerId);
}

// Reports/Services/Reports/ReportAppService.cs — consume from another module via DI
public class ReportAppService : ApplicationService, IReportAppService
{
    private readonly IFormSubmissionIntegrationService _formSubmissions;   // ← cross-module read
    private readonly IRepository<Report, Guid> _reportRepository;

    public ReportAppService(
        IFormSubmissionIntegrationService formSubmissions,
        IRepository<Report, Guid> reportRepository)
    {
        _formSubmissions = formSubmissions;
        _reportRepository = reportRepository;
    }

    public async Task<ReportDto> BuildAsync(Guid ownerId)
    {
        var submissionCount = await _formSubmissions.GetSubmissionCountForOwnerAsync(ownerId);
        // ... assemble report ...
    }
}
```

**In-process (modular monolith) vs cross-process (microservice):** in a Layered modular monolith both modules live in the same host, and DI resolves the integration service to its in-process implementation directly — no HTTP hop. In Microservice the same `[IntegrationService]` interface is exposed over `/integration-api/...` behind the API gateway and consumed via a generated proxy client. Same contract, same dependency-injection shape; transport differs by deployment.

**Integration services do not need explicit controllers** — Rule 4 does not apply. ABP wires their HTTP exposure implicitly through the integration-services pipeline. Don't hand-write a `Controller` that implements an `[IntegrationService]` interface.

**ABP-module disambiguator.** "Module" in ABP-land is overloaded. (a) An **AbpModule class** is the small `XxxModule : AbpModule` C# class that ABP loads at startup to wire DI, options, conventions, and dependencies (see `OnApplicationInitialization`, `ConfigureServices`, `[DependsOn]`). (b) An **ABP application module** is a packaged feature set (Identity, OpenIddict, SaaS, Forms, CMS, BlobStoring, etc.) that you install via `abp add-module` or NuGet — see `references/modules/`. (c) A **module** in the cross-module-reads sense above is a .NET solution authored to be consumed by other solutions. The Rule 5 boundary is (c); the AbpModule class (a) is the wiring artifact each (c) ships; the application modules (b) are the prebuilt instances of (c) ABP itself ships.

### Rule 6 — Each module owns its DbContext, migrations, schema migrator, and a unique migrations history table

When a solution contains multiple ABP modules — common in Layered modular monoliths and required in Microservice — each module must ship its own `DbContext`, its own EF Core migrations folder, its own `IXxxDbSchemaMigrator` implementation, and **its own EF migrations history table**.

Sharing the default `__EFMigrationsHistory` table across modules causes EF to mistake another module's migrations for "missing" and try to re-apply them. Use the naming convention `__EFMigrationsHistory_<Module>`:

```csharp
// Forms/Data/FormsDbContext.cs
public class FormsDbContext : AbpDbContext<FormsDbContext>
{
    public FormsDbContext(DbContextOptions<FormsDbContext> options) : base(options) { }
    // ...DbSet<T> properties...
}

// In the module's ConfigureServicesAsync where the DbContext is registered:
context.Services.AddAbpDbContext<FormsDbContext>(options =>
{
    options.AddDefaultRepositories(includeAllEntities: true);
});

Configure<AbpDbContextOptions>(options =>
{
    options.Configure<FormsDbContext>(c =>
    {
        c.UseNpgsql(b => b.MigrationsHistoryTable("__EFMigrationsHistory_Forms"));
    });
});
```

**Each module also owns an `IDesignTimeDbContextFactory<TDbContext>`** next to its `DbContext` so `dotnet ef migrations add` / `Update-Database` can construct the context at design time. Without it, EF tooling falls back to host-level discovery, which doesn't see module-private DI configuration.

See [framework/data-ef-core.md](./references/framework/data-ef-core.md) for the full pattern, the `EntityFrameworkCore<Module>DbSchemaMigrator` template, and the multi-DbContext "second-schema" scenario.

### Rule 7 — Don't throw business exceptions from application services; use ABP's validation pipeline for input

`BusinessException`, `UserFriendlyException`, and other invariant-violation throws belong in the entity, an entity behaviour method, or a `*Manager` (domain service) — wherever the rule is actually enforced. **Throwing `BusinessException` from a `*Manager` is the right place** — the manager owns the invariant, and any caller (app service, background job, integration service, domain event handler) routes through it and gets the same enforcement.

What this rule forbids is putting `throw new BusinessException(...)` inside an `*AppService.cs`: that couples the rule to a single caller and lets the same entity be mutated past the rule through any other path.

**Where the throw lives, in priority order:**

1. **Entity (or an entity behavior method like `customer.EnsureCanReceiveNewOrder()`)** when the rule reads only the entity's own state. This is the strongest form — every caller (Manager, app service, integration service, event handler) gets the rule by routing through the entity. Pair this with an `internal` constructor when creation requires a business check (see the Common Mistakes entry on `public` aggregate constructors in [framework/architecture-ddd.md](./references/framework/architecture-ddd.md)).
2. **`*Manager` (domain service)** when the rule requires a repository check (uniqueness, count, existence) or a sibling-aggregate guard. This is canonical ABP: domain services are explicitly allowed to throw `BusinessException` for repository-driven checks (see the [Domain Services best-practices doc](https://abp.io/docs/10.3/framework/architecture/best-practices/domain-services) — the doc's own example throws when `openIssueCount >= 3`).
3. **Never the application service.** Application services validate input (next paragraph), call the domain service for invariants, and orchestrate persistence. They do not own business rules.

**For input validation** — required fields, length, range, format, cross-field rules — use ABP's validation pipeline instead of imperative throws in the service body. See [framework/validation.md](./references/framework/validation.md). Data annotations on DTOs, `IValidatableObject`, and FluentValidation `AbstractValidator<T>` all flow through `AbpValidationException` automatically and produce a structured 400 response — without the service ever seeing the invalid input. DTO `[Range]` and friends are **fast-fail UX**, not the invariant's home: keep the entity's own range check inside the entity so any caller bypassing the DTO still gets it.

## Routing table — when to read which reference

Pull the focused reference only when the topic applies. Each file is a self-contained slice of the framework.

### Templates

| Topic | File |
|-------|------|
| "Which template should I pick?" / template comparison / migration paths | [references/templates/overview.md](./references/templates/overview.md) |
| Single-Layer template specifics (`app-nolayers`) | [references/templates/single-layer.md](./references/templates/single-layer.md) |
| Layered template specifics (`app`) — DbMigrator, `--tiered`, `--separate-auth-server` | [references/templates/layered.md](./references/templates/layered.md) |
| Microservice template specifics — apps/gateways/services, YARP, RabbitMQ, Helm, Aspire | [references/templates/microservice.md](./references/templates/microservice.md) |

### Framework concepts

| Topic | File |
|-------|------|
| Module class, DI, base classes, options, exceptions, localization, logging, configuration | [references/framework/fundamentals.md](./references/framework/fundamentals.md) |
| Entities, aggregate roots, value objects, repositories, domain services, specifications, app services, DTOs, UoW | [references/framework/architecture-ddd.md](./references/framework/architecture-ddd.md) |
| Module system, plug-in modules, extending modules (services, entities, UI overrides) | [references/framework/architecture-modularity.md](./references/framework/architecture-modularity.md) |
| `IMultiTenant`, tenant switching, data filters, tenant resolution, SaaS | [references/framework/multi-tenancy.md](./references/framework/multi-tenancy.md) |
| Permissions, `[Authorize]`, `CurrentUser`, dynamic claims, resource-based authorization | [references/framework/authorization.md](./references/framework/authorization.md) |
| Validation pipeline, DataAnnotations, `IValidatableObject`, FluentValidation integration, custom contributors | [references/framework/validation.md](./references/framework/validation.md) |
| Auto API Controllers, Integration Services, dynamic/static C# clients, API versioning, Swagger | [references/framework/api-development.md](./references/framework/api-development.md) |
| `DbContext`, repositories, EF migrations, DBMS providers (Postgres/MySQL/SQLite/Oracle), connection strings | [references/framework/data-ef-core.md](./references/framework/data-ef-core.md) |

### Application modules

| Topic | File |
|-------|------|
| Identity, Account, OpenIddict, LDAP, IdentityServer migration | [references/modules/identity-and-auth.md](./references/modules/identity-and-auth.md) |
| Tenant Management, SaaS (Pro), feature management, edition management | [references/modules/tenancy-and-saas.md](./references/modules/tenancy-and-saas.md) |
| CMS Kit, Blogging, Docs, Forms (Pro), File Management (Pro) | [references/modules/content-and-cms.md](./references/modules/content-and-cms.md) |
| Audit Logging, Setting Management, Background Workers (module-side), Lepton Theme Management | [references/modules/operations.md](./references/modules/operations.md) |
| Chat (Pro), Notifications, Payment (Pro) | [references/modules/messaging-and-payment.md](./references/modules/messaging-and-payment.md) |
| AI module, Identity Server admin (Pro), Time Zone, Comment, Reaction, Rating | [references/modules/ai-and-misc.md](./references/modules/ai-and-misc.md) |

### Infrastructure subsystems

| Topic | File |
|-------|------|
| `IBackgroundJobManager` / `IBackgroundWorker`, Hangfire / Quartz / RabbitMQ providers | [references/infrastructure/background-processing.md](./references/infrastructure/background-processing.md) |
| `IBlobContainer<T>`, blob storage providers (FileSystem, Azure, AWS, MinIO, Database) | [references/infrastructure/blob-storing.md](./references/infrastructure/blob-storing.md) |
| `ILocalEventBus`, `IDistributedEventBus`, RabbitMQ/Kafka/Azure Service Bus, outbox/inbox | [references/infrastructure/eventing.md](./references/infrastructure/eventing.md) |
| `IDistributedCache<T>`, Redis, `IAbpDistributedLock` (lock providers) | [references/infrastructure/caching-and-locking.md](./references/infrastructure/caching-and-locking.md) |

### Tools

| Topic | File |
|-------|------|
| `abp` CLI — `new`, `add-module`, `add-package`, `update`, `generate-proxy`, login, mcp helpers | [references/tools/cli.md](./references/tools/cli.md) |
| ABP Studio (desktop) — solution explorer, run/monitor, MCP server, helm/k8s integrations | [references/tools/studio.md](./references/tools/studio.md) |
| ABP Suite (Pro) — CRUD generator, entity designer, code-generation conventions | [references/tools/suite.md](./references/tools/suite.md) |

### Testing

| Topic | File |
|-------|------|
| Integration test base classes, `AbpWebApplicationFactoryIntegratedTest<Program>`, SQLite/in-memory provider, fixtures, seeding | [references/testing.md](./references/testing.md) |

### Customization

| Topic | File |
|-------|------|
| How to write `.abp-overrides.md` for a project; sidecar contract | [customization/README.md](./customization/README.md) |
| Worked example of a project's overrides | [customization/example.md](./customization/example.md) |

## Peer skill: `abp-frontend`

UI/frontend questions — MVC/Razor Pages, Blazor (Server, WebAssembly, Web App), Angular, React Native, MAUI, LeptonX themes, navigation menus, JS/TS proxy generation, Tag Helpers, ABP UI components — are **not** in this skill. They live in the peer skill `abp-frontend`. The two share the same customization contract (`.abp-overrides.md` walk-up, CLAUDE.md deference) and the same docs version pin (`https://abp.io/docs/10.3/`). If a question spans both backend and UI (e.g., "expose this app service to my Angular client"), this skill covers the C# / Auto API / Integration Services side and `abp-frontend` covers the generated client / proxy / UI binding side.
