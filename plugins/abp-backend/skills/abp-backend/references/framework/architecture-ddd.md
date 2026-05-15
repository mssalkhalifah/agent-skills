# Architecture — Domain-Driven Design

> Reference for ABP's Domain-Driven Design implementation: the four-layer architecture, every domain and application building block (entities, aggregate roots, value objects, repositories, domain services, specifications, application services, DTOs, unit of work), and the layered package/dependency rules that hold it together.

## When to load this reference

- User asks how to model an entity, aggregate root, or value object in ABP
- User is choosing between a domain service and an application service, or between IRepository and IBasicRepository
- User asks where business logic should live (entity vs. domain service vs. application service)
- User asks about the Unit of Work, [UnitOfWork] attribute, transaction boundaries, or RequiresNew
- User asks where DTOs go, what base class to use, or how DTOs relate to entities
- User asks about specifications / reusable filters
- User wants to understand the Domain.Shared / Domain / Application.Contracts / Application / EF Core / HttpApi / Web package layout and dependency direction
- User asks 'why does ABP say not to use IQueryable in application code?' or similar best-practice clarification
- User asks how to structure a new ABP module per DDD conventions

**Audience:** Backend developers building or maintaining ABP-based applications and modules who need to apply ABP's DDD conventions correctly — across entity design, persistence, application orchestration, transactions, and module/package layout.

## Key concepts

- Entity<TKey> (Volo.Abp.Domain.Entities) — base class for domain objects with a typed Id; reach for it when modeling any domain object that has identity but is not an aggregate root.
- AggregateRoot<TKey> (Volo.Abp.Domain.Entities) — entity that is the consistency boundary and the only type ABP creates default repositories for; use for the root of every aggregate. Recommended key type: Guid.
- Audited base classes: CreationAuditedEntity<TKey>, AuditedEntity<TKey>, FullAuditedEntity<TKey> and their AggregateRoot variants (Volo.Abp.Domain.Entities.Auditing) — pick when you want CreationTime / Creator / LastModifier / IsDeleted populated automatically.
- ISoftDelete, IMultiTenant, IHasCreationTime, IHasModificationTime, ICreationAuditedObject, IAuditedObject, IFullAuditedObject (Volo.Abp.Domain.Entities.Auditing / Volo.Abp.MultiTenancy) — opt-in marker interfaces; ABP filters or stamps fields when present.
- IHasExtraProperties + ExtensibleObject (Volo.Abp.Data) — dynamic property bag for entities and DTOs (SetProperty / GetProperty / HasProperty / RemoveProperty); reach for it for tenant-customizable fields without schema changes.
- ValueObject (Volo.Abp.Domain.Values) — abstract base for immutable, identity-less domain concepts; override GetAtomicValues() to define equality (ValueEquals).
- IRepository<TEntity, TKey> (Volo.Abp.Domain.Repositories) — full generic repository: CRUD + GetQueryableAsync() + LINQ-friendly extension methods. Use from infrastructure or framework code, not directly from application code per best practices.
- IBasicRepository<TEntity, TKey> (Volo.Abp.Domain.Repositories) — minimal CRUD-only contract; best-practice base for custom repository interfaces so application code never depends on IQueryable.
- IReadOnlyRepository<TEntity, TKey> (Volo.Abp.Domain.Repositories) — query-only repository that uses EF Core no-tracking; use for read-side or report scenarios.
- IAsyncQueryableExecuter (Volo.Abp.Linq) — DB-provider-agnostic executor for IQueryable (ToListAsync, FirstOrDefaultAsync, etc.); use inside reusable modules where you can't take an EF Core dependency.
- IDomainService / DomainService (Volo.Abp.Domain.Services) — transient-registered base for stateless domain logic that spans aggregates or needs other services; convention is the 'Manager' suffix (e.g., IssueManager).
- ISpecification<T> / Specification<T> / AndSpecification<T>, OrSpecification<T>, NotSpecification<T>, AndNotSpecification<T> (Volo.Abp.Specifications) — reusable, composable, named filters; override ToExpression() and combine via And/Or/Not/AndNot. Package: Volo.Abp.Specifications.
- IApplicationService / ApplicationService (Volo.Abp.Application.Services) — transient base for use-case orchestrators called by the presentation layer; provides IGuidGenerator, ICurrentUser, ILogger, ObjectMapper, AsyncExecuter, etc.
- ICrudAppService<TEntityDto, TKey, TGetListInput, TCreateInput, TUpdateInput> / CrudAppService — generic interface and base implementation for the standard GetAsync/GetListAsync/CreateAsync/UpdateAsync/DeleteAsync surface; subclass and override only what you need.
- AbstractKeyCrudAppService — variant for entities with composite or non-standard keys; you implement DeleteByIdAsync and GetEntityByIdAsync.
- IEntityDto<TKey> / EntityDto<TKey> + audited variants (CreationAuditedEntityDto, AuditedEntityDto, FullAuditedEntityDto) and user-info variants (Volo.Abp.Application.Dtos) — DTO base classes mirroring entity auditing so client receives the same metadata.
- ExtensibleObject / ExtensibleEntityDto (Volo.Abp.ObjectExtending) — DTO counterparts of IHasExtraProperties for round-tripping extra properties.
- IListResult<T> / ListResultDto<T> / PagedResultDto<T> / PagedAndSortedResultRequestDto (Volo.Abp.Application.Dtos) — standardized list and paging contracts: { Items, TotalCount } in, { SkipCount, MaxResultCount, Sorting } out.
- IUnitOfWork / IUnitOfWorkManager / ICurrentUnitOfWork (Volo.Abp.Uow) — ambient transaction scope abstraction; access the active UoW via IUnitOfWorkManager.Current.
- [UnitOfWork] attribute (Volo.Abp.Uow) — opt-in / opt-out / configure UoW per method or class (IsTransactional, RequiresNew, IsolationLevel, Timeout, IsDisabled).
- IUnitOfWorkEnabled (Volo.Abp.Uow) — marker interface to enable automatic UoW on arbitrary services that ABP wouldn't otherwise wrap.

## Configuration pattern

DDD building blocks are wired up almost entirely by convention — there is no central 'AddDdd()' call. The relevant module-level configuration lives in three places:

1) Module dependencies (declared on your AbpModule classes via [DependsOn]) determine which DDD packages are pulled in: AbpDddDomainModule for the domain layer, AbpDddApplicationModule + AbpDddApplicationContractsModule for the application layer, an EF Core / MongoDB module for the repository implementations, and AbpUnitOfWorkModule for UoW. ABP's solution templates set this up for you — you generally only need to add [DependsOn] when introducing new modules.

2) Default UoW behavior is configured via AbpUnitOfWorkDefaultOptions in your *.Application or *.Web module's ConfigureServices:

   public override void ConfigureServices(ServiceConfigurationContext context)
   {
       Configure<AbpUnitOfWorkDefaultOptions>(options =>
       {
           options.TransactionBehavior = UnitOfWorkTransactionBehavior.Auto; // Auto | Enabled | Disabled
           options.IsolationLevel = IsolationLevel.ReadCommitted;
           options.Timeout = TimeSpan.FromSeconds(30);
       });
   }

   - TransactionBehavior=Auto means HTTP GET handlers run non-transactional, other verbs run transactional. Override per-method with [UnitOfWork(IsTransactional = true|false, RequiresNew = true)].

3) Object mapping is wired by adding an AutoMapper or Mapperly profile and registering it in ConfigureServices:

   Configure<AbpAutoMapperOptions>(options =>
   {
       options.AddMaps<MyApplicationModule>(validate: true); // validate mappings at startup
   });

Repositories, domain services, and application services are all auto-registered by convention (transient lifetime). Custom repositories are registered by adding them to the EF Core module in ConfigureAbpDbContext / AddDefaultRepositories(includeAllEntities: true) calls — the default repository for an aggregate root is created automatically when the entity is mapped via AddAbpDbContext<T>().

Key options to know:
- AbpUnitOfWorkDefaultOptions — default transaction/isolation/timeout for ambient UoWs.
- AbpAutoMapperOptions / AbpMapperlyOptions — register mapping profiles, optionally validate at startup.
- AbpDbContextOptions — per-DB-context configuration; controls EF Core options used by IRepository.

## Code examples

### Aggregate root with encapsulated invariants

_Model an Issue as an aggregate root with a validating constructor, internal setters, and behavior methods that keep state consistent._

```csharp
using System;
using Volo.Abp;
using Volo.Abp.Domain.Entities.Auditing;

namespace MyProject.Issues;

public class Issue : FullAuditedAggregateRoot<Guid>
{
    public Guid RepositoryId { get; private set; }
    public string Title { get; private set; }
    public string Text { get; private set; }
    public Guid? AssignedUserId { get; private set; }
    public bool IsClosed { get; private set; }
    public IssueCloseReason? CloseReason { get; private set; }

    // Required by ORM / proxy generators.
    protected Issue() { }

    internal Issue(
        Guid id,
        Guid repositoryId,
        string title,
        string text = null) : base(id)
    {
        RepositoryId = repositoryId;
        Title = Check.NotNullOrWhiteSpace(title, nameof(title), maxLength: IssueConsts.MaxTitleLength);
        Text  = Check.Length(text, nameof(text), maxLength: IssueConsts.MaxTextLength);
    }

    public void Close(IssueCloseReason reason)
    {
        IsClosed    = true;
        CloseReason = reason;
    }

    public void ReOpen()
    {
        IsClosed    = false;
        CloseReason = null;
    }
}
```

**Key lines:** FullAuditedAggregateRoot<Guid> opts in to creation/modification/soft-delete auditing. The protected parameterless ctor is required so EF Core / proxy frameworks can rehydrate the entity. The internal ctor validates inputs via Volo.Abp.Check, and Close()/ReOpen() encapsulate the rule that IsClosed and CloseReason must change together.

### Value object with GetAtomicValues

_An immutable Address value object whose equality is value-based, not identity-based._

```csharp
using System;
using System.Collections.Generic;
using Volo.Abp.Domain.Values;

namespace MyProject.Customers;

public class Address : ValueObject
{
    public string Street { get; }
    public Guid CityId { get; }
    public string Number { get; }

    public Address(string street, Guid cityId, string number)
    {
        Street = street;
        CityId = cityId;
        Number = number;
    }

    protected override IEnumerable<object> GetAtomicValues()
    {
        yield return Street;
        yield return CityId;
        yield return Number;
    }
}
```

**Key lines:** Inheriting from ValueObject gives you ValueEquals() and structural GetHashCode() for free. GetAtomicValues() yields the components that define equality. All properties are get-only — value objects must be immutable.

### Domain service that spans aggregates

_An IssueManager that enforces an invariant requiring a repository lookup (max open issues per user) before mutating an aggregate._

```csharp
using System;
using System.Threading.Tasks;
using Volo.Abp;
using Volo.Abp.Domain.Services;

namespace MyProject.Issues;

public class IssueManager : DomainService
{
    private readonly IIssueRepository _issueRepository;

    public IssueManager(IIssueRepository issueRepository)
    {
        _issueRepository = issueRepository;
    }

    public async Task AssignToAsync(Issue issue, Guid userId)
    {
        var openCount = await _issueRepository.GetOpenIssueCountForUserAsync(userId);
        if (openCount >= 3)
        {
            throw new BusinessException("Issues:TooManyOpenIssuesForUser")
                .WithData("UserId", userId);
        }

        issue.AssignTo(userId);
    }
}
```

**Key lines:** Inheriting DomainService auto-registers as transient and exposes Logger / GuidGenerator. The method takes the Issue aggregate (not a DTO), throws BusinessException on rule violation, and delegates the actual state mutation to a behavior method on the aggregate (issue.AssignTo).

### Specification used with a repository

_Define a reusable Age18PlusCustomerSpecification and combine it with a premium-customer specification to count adult premium customers._

```csharp
using System.Linq.Expressions;
using Volo.Abp.Specifications;

public class Age18PlusCustomerSpecification : Specification<Customer>
{
    public override Expression<System.Func<Customer, bool>> ToExpression()
        => c => c.Age >= 18;
}

public class PremiumCustomerSpecification : Specification<Customer>
{
    public override Expression<System.Func<Customer, bool>> ToExpression()
        => c => c.IsPremium;
}

// Inside an application service:
var count = await _customerRepository.CountAsync(
    new Age18PlusCustomerSpecification()
        .And(new PremiumCustomerSpecification())
        .ToExpression());
```

**Key lines:** Override only ToExpression(); IsSatisfiedBy() is provided. .And()/.Or()/.Not()/.AndNot() return composed specifications, and .ToExpression() converts the composite into an expression the repository can push down to SQL.

### Application service that orchestrates the domain

_Create-issue use case: load aggregate dependencies, delegate to a domain service for cross-aggregate rules, persist via repository, return DTO._

```csharp
using System;
using System.Threading.Tasks;
using Volo.Abp.Application.Services;

namespace MyProject.Issues;

public class IssueAppService : ApplicationService, IIssueAppService
{
    private readonly IIssueRepository _issueRepository;
    private readonly IssueManager _issueManager;

    public IssueAppService(
        IIssueRepository issueRepository,
        IssueManager issueManager)
    {
        _issueRepository = issueRepository;
        _issueManager    = issueManager;
    }

    public async Task<IssueDto> CreateAsync(CreateIssueDto input)
    {
        var issue = new Issue(
            GuidGenerator.Create(),
            input.RepositoryId,
            input.Title,
            input.Text);

        if (input.AssignedUserId.HasValue)
        {
            await _issueManager.AssignToAsync(issue, input.AssignedUserId.Value);
        }

        await _issueRepository.InsertAsync(issue, autoSave: true);
        return ObjectMapper.Map<Issue, IssueDto>(issue);
    }
}
```

**Key lines:** The service inherits ApplicationService for IGuidGenerator / ObjectMapper / ICurrentUser. Inputs and outputs are DTOs only — entities never leak. Cross-aggregate rules are pushed into IssueManager. The whole method runs in an automatic UoW because it is a public virtual application-service method.

### Per-method UoW configuration

_Force a new transactional scope with serializable isolation for a money-transfer method, regardless of the ambient UoW._

```csharp
using System.Data;
using System.Threading.Tasks;
using Volo.Abp.Application.Services;
using Volo.Abp.Uow;

public class TransferAppService : ApplicationService, ITransferAppService
{
    [UnitOfWork(
        IsTransactional = true,
        RequiresNew = true,
        IsolationLevel = IsolationLevel.Serializable,
        Timeout = 30000)]
    public virtual async Task TransferAsync(TransferInput input)
    {
        // ... debit + credit ...
    }
}
```

**Key lines:** [UnitOfWork] requires the method to be virtual (interception-based). RequiresNew = true forces a fresh UoW even inside an ambient one; IsolationLevel.Serializable hardens the read/write semantics for the transfer; Timeout caps the transaction at 30 s.

## Common mistakes

### Adding navigation properties from one aggregate root to another (e.g., Order.Customer of type Customer).

**Why wrong:** Crosses aggregate boundaries, lets ORMs cascade-load or cascade-save across roots, and makes consistency rules non-local.

**Correct pattern:** Reference other aggregate roots only by Id (CustomerId : Guid). Load the other aggregate via its repository when actually needed.

```csharp
// before
public Customer Customer { get; set; }
// after
public Guid CustomerId { get; private set; }
```

### Using IRepository<TEntity, TKey> directly from application services and calling .Where(...).ToList() on its IQueryable.

**Why wrong:** Best practices forbid IQueryable in application code: it leaks persistence concerns, prevents swapping ORMs, and makes async behavior provider-specific.

**Correct pattern:** Define a custom repository interface in the Domain project that inherits from IBasicRepository<TEntity, TKey>; expose intent-revealing methods (GetByEmailAsync, GetOpenIssuesAsync). Implement them in the EF Core / MongoDB project.

```csharp
public interface IIssueRepository : IBasicRepository<Issue, Guid>
{
    Task<List<Issue>> GetOpenIssuesAsync(Guid userId);
}
```

### Creating repositories for non-aggregate-root entities.

**Why wrong:** Aggregates are the consistency unit; loading or saving a child entity outside of its root bypasses invariants and produces inconsistent state.

**Correct pattern:** Only define repositories for aggregate roots. Mutate child entities through methods on their aggregate root and persist via the root's repository.

```csharp
// remove IOrderLineRepository; expose order.AddLine(...) and order.RemoveLine(...) on Order instead.
```

### Generating Ids inside an entity constructor with Guid.NewGuid().

**Why wrong:** Bypasses ABP's IGuidGenerator, which produces sequential Guids better suited for clustered indexes; also makes the caller unable to control Id selection.

**Correct pattern:** Take the Id as a constructor parameter. The caller (typically an application service) calls GuidGenerator.Create().

```csharp
internal Order(Guid id, ...) : base(id) { ... } // and in the app service: new Order(GuidGenerator.Create(), ...);
```

### `public` constructor on an aggregate root that has a business-level creation rule.

**Why wrong:** A `public` constructor is callable from any assembly — controllers, app services, tests, integration handlers can all do `new Order(...)` and bypass the `*Manager` that owns the creation rule (uniqueness check, customer must be active, computed initial state). The rule lives in the Manager but stops being enforced once any caller can construct the entity directly.

**Correct pattern:** When creation requires a business check, mark the constructor `internal` — visible to other code in the same Domain assembly (where the Manager lives) but not to controllers or app services in other assemblies. The Manager is then the only entry point that constructs the entity. The parameterless `protected` constructor that EF Core requires stays.

```csharp
// Aggregate WITH a creation rule: internal ctor, Manager is the only entry point
public class Order : AggregateRoot<Guid>
{
    protected Order() { }                                  // EF Core
    internal Order(Guid id, Guid customerId, string number) : base(id)
    {
        CustomerId = customerId;
        Number = number;
    }
}
```

**When `public` is the right answer:** if the entity has only structural / data-shape validation (a non-empty name, a sensible date range that's already covered by DTO validation) and no rule requires a repository or another aggregate at creation time, leave the constructor `public`. Don't add a Manager just to gate creation that doesn't need gating — that's ceremony, not encapsulation.

```csharp
// Aggregate with NO creation rule: public ctor is fine, no Manager needed
public class Bookmark : AggregateRoot<Guid>
{
    protected Bookmark() { }
    public Bookmark(Guid id, string url, string label) : base(id)
    {
        Url = url;
        Label = label;
    }
}
```

The decision is "does creating this entity require a business check that must be enforced uniformly across callers?" — if yes, `internal` + Manager; if no, `public` and skip the Manager.

### Forgetting the protected parameterless constructor on an entity that also has a parameterized one.

**Why wrong:** EF Core proxies, deserializers, and some interception scenarios need a parameterless ctor; without it you get runtime construction failures.

**Correct pattern:** Always pair a public/internal validating ctor with a protected/private parameterless one.

```csharp
protected Order() { /* for EF Core */ }
```

### Putting business rules into application services instead of entities or domain services.

**Why wrong:** Application services are use-case orchestrators; rules placed there can't be reused across use cases and bypass aggregate invariants.

**Correct pattern:** Put state-changing rules on the entity (Close(reason), AssignTo(userId)). Put cross-aggregate rules in a Manager domain service. Application services just orchestrate.

```csharp
// in app service: issue.Close(reason); await _issueRepository.UpdateAsync(issue);
```

### Returning entities from application service methods (or accepting entities as inputs).

**Why wrong:** Couples the presentation/HTTP layer to the persistence model, leaks lazy-loaded navigation properties to the wire, and exposes private state.

**Correct pattern:** Always accept input DTOs and return output DTOs; map via ObjectMapper.

```csharp
public Task<IssueDto> GetAsync(Guid id) => ObjectMapper.Map<Issue, IssueDto>(await _repo.GetAsync(id));
```

### Calling another module's application service from inside your application service.

**Why wrong:** Best practices forbid this within the same module: it hides the dependency, may create cyclic references, and can yield inconsistent transaction boundaries. Across modules it bypasses module contracts.

**Correct pattern:** Call a domain service or shared helper class instead. Across modules, integrate via distributed events or explicit module-level integration interfaces.

```csharp
// don't: await _otherAppService.DoSomething(); // do: await _someManager.DoSomethingAsync(domainObject);
```

### Decorating a non-virtual method with [UnitOfWork] (or making the calling method non-async).

**Why wrong:** ABP intercepts UoW via a runtime proxy; non-virtual methods aren't intercepted, so the attribute silently has no effect.

**Correct pattern:** Methods with [UnitOfWork] (or that should participate in a UoW via interception) must be public virtual async.

```json
[UnitOfWork(RequiresNew = true)] public virtual async Task TransferAsync(...) { ... }
```

### Marking an HTTP GET as non-transactional but performing writes inside it.

**Why wrong:** Default behavior is non-transactional UoW for GETs; writes inside a non-transactional UoW won't be rolled back atomically on exception.

**Correct pattern:** Either change the verb or annotate with [UnitOfWork(IsTransactional = true)].

```json
[UnitOfWork(IsTransactional = true)] public virtual async Task<IActionResult> SomeWriteyGet(...) { ... }
```

### Adding business logic to DTOs (computed properties, validation that calls services, mapping back to entities).

**Why wrong:** DTOs must remain serializable, transport-only types; logic in them couples the wire format to behavior and breaks JSON round-tripping.

**Correct pattern:** DTOs only carry data and may implement IValidatableObject for cross-field validation. Put logic in entities, domain services, or application services.

```csharp
// Move IsExpired logic from CertificateDto to the Certificate entity; the DTO just carries ExpirationTime.
```

### Reusing a single input DTO across Create and Update operations.

**Why wrong:** The two use cases almost always diverge (Update may not allow changing tenant Id; Create may require fields Update doesn't). Reuse causes nullable-everywhere DTOs and weakens validation.

**Correct pattern:** Define CreateIssueDto and UpdateIssueDto separately; share a base only when fields are truly identical.

```csharp
public class CreateIssueDto { ... } public class UpdateIssueDto { ... }
```

### Composite primary keys on aggregate roots.

**Why wrong:** Best practices state aggregate roots must use Guid as primary key — composite keys complicate routing, caching, distributed events, and replication.

**Correct pattern:** Use Guid Ids on aggregate roots; if you have a natural composite identity, model it as a uniqueness constraint plus a Guid surrogate key.

```csharp
public class Membership : AggregateRoot<Guid> { public Guid UserId { get; } public Guid OrganizationId { get; } /* + unique index */ }
```

## Version pins (ABP 10.3)

- ABP 10.3 ships Mapperly as the default object-mapping engine in solution templates; AutoMapper remains supported through Volo.Abp.AutoMapper but is no longer the default.
- AbpUnitOfWorkDefaultOptions on 10.3 still defaults HTTP GET requests to non-transactional and other verbs to transactional via UnitOfWorkTransactionBehavior.Auto. [uncertain] whether the default isolation level is explicitly fixed at ReadCommitted in 10.3 or inherited from the underlying provider.
- Specifications package: Volo.Abp.Specifications is referenced in startup templates as of 10.3; older code may still rely on bare ToExpression() Funcs without the package — verify the package reference if AndSpecification/OrSpecification types are missing.
- Repository tracking helpers DisableTracking()/EnableTracking() are EF Core extension methods provided by ABP and may not be available on MongoDB or other providers — rely on IReadOnlyRepository<TEntity, TKey> for cross-provider read-side queries.
- [uncertain] the exact set of HTTP verbs that map to non-transactional UoW under TransactionBehavior.Auto beyond GET (e.g., HEAD, OPTIONS) — the docs only call out GET explicitly.
- Soft delete and multi-tenancy data filters are applied at the UoW boundary; toggling them mid-method requires IDataFilter.Disable<ISoftDelete>() / Enable<ISoftDelete>() patterns. In 10.3 this still flows through the per-UoW IDataFilter scope.
- ABP 10.3 keeps [UnitOfWork] interception requirement that target methods are public virtual; source-generator-based interception alternatives are [uncertain] in 10.3 and should be verified against the migration guide before relying on them.
- FullAuditedAggregateRoot still implements ISoftDelete in 10.3, meaning DeleteAsync sets IsDeleted rather than physically removing the row; use HardDeleteAsync for permanent deletion when needed.

## Cross-references

**Phase 1 references:**
- [references/framework/modules.md](../framework/modules.md) — triggered when discussing the [DependsOn] graph and which AbpDdd* modules to include for domain/application/UoW support
- [references/framework/dependency-injection.md](../framework/dependency-injection.md) — triggered for the auto-registration / transient lifetime rules used by repositories, domain services, and application services
- [references/framework/data-access-ef-core.md](../framework/data-access-ef-core.md) — triggered when implementing custom repositories, AddDefaultRepositories, or includeDetails projections
- [references/framework/data-access-mongodb.md](../framework/data-access-mongodb.md) — triggered for the MongoDB-backed alternative to EF Core repositories
- [references/framework/object-mapping.md](../framework/object-mapping.md) — triggered whenever DTO<->entity mapping (Mapperly default, AutoMapper option) comes up in app-service code
- [references/framework/validation.md](../framework/validation.md) — triggered when discussing data-annotation validation on input DTOs and IValidatableObject
- [references/framework/authorization.md](../framework/authorization.md) — triggered for declarative [Authorize] / IAuthorizationService usage on application services
- [references/framework/exception-handling.md](../framework/exception-handling.md) — triggered for BusinessException / UserFriendlyException raised from domain services
- [references/framework/multi-tenancy.md](../framework/multi-tenancy.md) — triggered when entities implement IMultiTenant or when UoW data filters discuss tenant isolation
- [references/framework/data-filtering.md](../framework/data-filtering.md) — triggered when discussing soft-delete (ISoftDelete) and multi-tenant data filters at the UoW level
- [references/framework/distributed-events.md](../framework/distributed-events.md) — triggered when domain events / outbox / inbox patterns are mentioned alongside aggregate roots
- [references/templates/layered.md](../templates/layered.md) — triggered for the concrete project layout that realizes the DDD package layering
- [references/templates/single-layer.md](../templates/single-layer.md) — triggered when comparing a flatter template against the layered DDD layout
- [references/templates/microservice.md](../templates/microservice.md) — triggered when discussing how the DDD package layout is split across services

**External docs:**
- [Eric Evans — Domain-Driven Design (Reference book)](https://www.domainlanguage.com/ddd/reference/) — Canonical source for the DDD vocabulary (entity, value object, aggregate, repository, domain service, specification) ABP's documentation builds on.
- [Microsoft — Designing a DDD-oriented microservice](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/ddd-oriented-microservice) — Conceptual background for the layering and aggregate boundary ideas mirrored in ABP's four-layer model.
- [EF Core — Asynchronous queries](https://learn.microsoft.com/en-us/ef/core/miscellaneous/async) — Underlies ABP's repository async APIs and the warning against synchronous LINQ on database queries.
- [EF Core — No-tracking queries](https://learn.microsoft.com/en-us/ef/core/querying/tracking#no-tracking-queries) — Backing concept for IReadOnlyRepository<TEntity, TKey>'s read-side optimization.
- [EF Core — Transactions](https://learn.microsoft.com/en-us/ef/core/saving/transactions) — Underpins Unit of Work IsTransactional / RequiresNew / IsolationLevel options.
- [.NET — System.Data.IsolationLevel enum](https://learn.microsoft.com/en-us/dotnet/api/system.data.isolationlevel) — Type used by [UnitOfWork(IsolationLevel = ...)] when overriding the default.
- [Mapperly](https://mapperly.riok.app/) — Default object-mapping engine wired up by ABP's application module template, used to project entities to DTOs.
- [AutoMapper](https://docs.automapper.org/) — Alternative object-mapping engine still supported via AbpAutoMapperOptions / AddMaps<TModule>().
- [ASP.NET Core — Model validation](https://learn.microsoft.com/en-us/aspnet/core/mvc/models/validation) — Mechanism behind ABP's automatic input-DTO validation in application services.
- [.NET — IValidatableObject](https://learn.microsoft.com/en-us/dotnet/api/system.componentmodel.dataannotations.ivalidatableobject) — The single piece of logic best-practices allow inside a DTO.

## Sources (ABP 10.3)

- [https://abp.io/docs/10.3/framework/architecture/domain-driven-design — Top-level DDD overview: defines the four-layer model (Presentation/Application/Domain/Infrastructure) and lists the building blocks ABP ships](https://abp.io/docs/10.3/framework/architecture/domain-driven-design — Top-level DDD overview: defines the four-layer model (Presentation/Application/Domain/Infrastructure) and lists the building blocks ABP ships)
- [https://abp.io/docs/10.3/framework/architecture/domain-driven-design/domain-layer — Domain layer overview: enumerates Entities & Aggregate Roots, Value Objects, Repositories, Domain Services, Specifications](https://abp.io/docs/10.3/framework/architecture/domain-driven-design/domain-layer — Domain layer overview: enumerates Entities & Aggregate Roots, Value Objects, Repositories, Domain Services, Specifications)
- [https://abp.io/docs/10.3/framework/architecture/domain-driven-design/entities — Entity<TKey>, AggregateRoot<TKey>, IEntity, audited base classes, IGuidGenerator guidance, composite keys, IHasExtraProperties](https://abp.io/docs/10.3/framework/architecture/domain-driven-design/entities — Entity<TKey>, AggregateRoot<TKey>, IEntity, audited base classes, IGuidGenerator guidance, composite keys, IHasExtraProperties)
- [https://abp.io/docs/10.3/framework/architecture/domain-driven-design/value-objects — abstract ValueObject base class, GetAtomicValues(), ValueEquals, immutability rules](https://abp.io/docs/10.3/framework/architecture/domain-driven-design/value-objects — abstract ValueObject base class, GetAtomicValues(), ValueEquals, immutability rules)
- [https://abp.io/docs/10.3/framework/architecture/domain-driven-design/repositories — IRepository / IBasicRepository / IReadOnlyRepository, GetQueryableAsync, IAsyncQueryableExecuter, soft-delete vs HardDeleteAsync](https://abp.io/docs/10.3/framework/architecture/domain-driven-design/repositories — IRepository / IBasicRepository / IReadOnlyRepository, GetQueryableAsync, IAsyncQueryableExecuter, soft-delete vs HardDeleteAsync)
- [https://abp.io/docs/10.3/framework/architecture/domain-driven-design/domain-services — IDomainService, DomainService base class, transient registration, when to introduce a domain service](https://abp.io/docs/10.3/framework/architecture/domain-driven-design/domain-services — IDomainService, DomainService base class, transient registration, when to introduce a domain service)
- [https://abp.io/docs/10.3/framework/architecture/domain-driven-design/specifications — ISpecification<T>, Specification<T>, ToExpression(), IsSatisfiedBy(), And/Or/Not/AndNot composition](https://abp.io/docs/10.3/framework/architecture/domain-driven-design/specifications — ISpecification<T>, Specification<T>, ToExpression(), IsSatisfiedBy(), And/Or/Not/AndNot composition)
- [https://abp.io/docs/10.3/framework/architecture/domain-driven-design/application-layer — Application layer purpose, building blocks (Application Services, DTOs, Unit of Work)](https://abp.io/docs/10.3/framework/architecture/domain-driven-design/application-layer — Application layer purpose, building blocks (Application Services, DTOs, Unit of Work))
- [https://abp.io/docs/10.3/framework/architecture/domain-driven-design/application-services — IApplicationService, ApplicationService, ICrudAppService, CrudAppService, AbstractKeyCrudAppService, automatic validation, Mapperly mapping](https://abp.io/docs/10.3/framework/architecture/domain-driven-design/application-services — IApplicationService, ApplicationService, ICrudAppService, CrudAppService, AbstractKeyCrudAppService, automatic validation, Mapperly mapping)
- [https://abp.io/docs/10.3/framework/architecture/domain-driven-design/data-transfer-objects — IEntityDto<TKey>, EntityDto, audited DTO bases, ExtensibleObject, ListResultDto, PagedResultDto, PagedAndSortedResultRequestDto](https://abp.io/docs/10.3/framework/architecture/domain-driven-design/data-transfer-objects — IEntityDto<TKey>, EntityDto, audited DTO bases, ExtensibleObject, ListResultDto, PagedResultDto, PagedAndSortedResultRequestDto)
- [https://abp.io/docs/10.3/framework/architecture/domain-driven-design/unit-of-work — IUnitOfWork, IUnitOfWorkManager, [UnitOfWork] attribute, IsTransactional / RequiresNew / IsolationLevel, automatic UoW for app services & controllers](https://abp.io/docs/10.3/framework/architecture/domain-driven-design/unit-of-work — IUnitOfWork, IUnitOfWorkManager, [UnitOfWork] attribute, IsTransactional / RequiresNew / IsolationLevel, automatic UoW for app services & controllers)
- [https://abp.io/docs/10.3/framework/architecture/best-practices — Module best-practices entry page: DDD compliance, DBMS/ORM independence, monolith-or-microservice fit](https://abp.io/docs/10.3/framework/architecture/best-practices — Module best-practices entry page: DDD compliance, DBMS/ORM independence, monolith-or-microservice fit)
- [https://abp.io/docs/10.3/framework/architecture/best-practices/module-architecture — Layered package layout (Domain.Shared, Domain, Application.Contracts, Application, EF Core/MongoDB, HttpApi, HttpApi.Client, Web) and dependency rules](https://abp.io/docs/10.3/framework/architecture/best-practices/module-architecture — Layered package layout (Domain.Shared, Domain, Application.Contracts, Application, EF Core/MongoDB, HttpApi, HttpApi.Client, Web) and dependency rules)
- [https://abp.io/docs/10.3/framework/architecture/best-practices/domain-layer-overview — Domain layer best-practices entry page: Entities/Repositories/Domain Services + Domain.Shared vs Domain split](https://abp.io/docs/10.3/framework/architecture/best-practices/domain-layer-overview — Domain layer best-practices entry page: Entities/Repositories/Domain Services + Domain.Shared vs Domain split)
- [https://abp.io/docs/10.3/framework/architecture/best-practices/entities — Aggregate-root rules: Guid keys, no navigation properties to other aggregate roots, encapsulated setters, validating constructors, parameterless ctor for ORMs](https://abp.io/docs/10.3/framework/architecture/best-practices/entities — Aggregate-root rules: Guid keys, no navigation properties to other aggregate roots, encapsulated setters, validating constructors, parameterless ctor for ORMs)
- [https://abp.io/docs/10.3/framework/architecture/best-practices/repositories — Inherit from IBasicRepository, do not expose IQueryable to application code, only define repositories for aggregate roots, includeDetails parameter conventions](https://abp.io/docs/10.3/framework/architecture/best-practices/repositories — Inherit from IBasicRepository, do not expose IQueryable to application code, only define repositories for aggregate roots, includeDetails parameter conventions)
- [https://abp.io/docs/10.3/framework/architecture/best-practices/domain-services — Manager suffix, accept domain objects (never DTOs), don't return DTOs, throw BusinessException for invariant violations](https://abp.io/docs/10.3/framework/architecture/best-practices/domain-services — Manager suffix, accept domain objects (never DTOs), don't return DTOs, throw BusinessException for invariant violations)
- [https://abp.io/docs/10.3/framework/architecture/best-practices/application-layer-overview — Application layer best-practices entry page (Application Services, DTOs)](https://abp.io/docs/10.3/framework/architecture/best-practices/application-layer-overview — Application layer best-practices entry page (Application Services, DTOs))
- [https://abp.io/docs/10.3/framework/architecture/best-practices/application-services — One service per aggregate root, AppService suffix, separate input/output DTOs, no LINQ in services, don't call other app services in the same module](https://abp.io/docs/10.3/framework/architecture/best-practices/application-services — One service per aggregate root, AppService suffix, separate input/output DTOs, no LINQ in services, don't call other app services in the same module)
- [https://abp.io/docs/10.3/framework/architecture/best-practices/data-transfer-objects — DTOs in Application.Contracts, public getters/setters, [Serializable], no logic except IValidatableObject, no entity references](https://abp.io/docs/10.3/framework/architecture/best-practices/data-transfer-objects — DTOs in Application.Contracts, public getters/setters, [Serializable], no logic except IValidatableObject, no entity references)

Last verified: 2026-05-10
