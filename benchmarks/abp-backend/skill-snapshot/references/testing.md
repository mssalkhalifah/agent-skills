# Testing (unit, integration, per-template)

> Defines ABP's automated testing strategy (unit, integration) and the framework-provided base classes, modules, and helpers for testing ABP modules and applications across Single-Layer, Layered, and Microservice templates. UI-test concerns (Razor Pages HTML inspection, Angular, Blazor) are covered in the abp-frontend skill.

## When to load this reference

- User asks how to write unit tests for an entity, domain service, or application service in an ABP solution.
- User asks how to write integration tests against an ABP module (in-memory SQLite, AbpIntegratedTest, GetRequiredService).
- User asks how to seed test data (IDataSeedContributor in the .TestBase project).
- User asks how to mock dependencies, the current user, permissions, or features in tests.
- User asks how tests are structured in the microservice solution template (per-service test projects, *TestsModule, *TestBase).
- User asks why tests use NSubstitute / Shouldly / xUnit and how to swap them.
- User encounters errors like 'no IDataSeedContributor', missing AbpIntegratedTest base class, or DbContext provider issues inside tests.

**Audience:** Developers writing automated tests against any ABP 10.3 backend solution (Single-Layer, Layered, or Microservice) and module/library authors who ship test bases for downstream consumers.

## Key concepts

- AbpIntegratedTest<TStartupModule> (Volo.Abp.Testing) - Base class for ABP-integrated tests; bootstraps a full IAbpApplication using TStartupModule as the root module and exposes the DI container. Reach for it whenever you need real DI, options, unit-of-work, and seeded data instead of mocks.
- AbpAsyncTestBase (Volo.Abp.Testing) - Underlying async-aware test base used by AbpIntegratedTest; provides hooks for asynchronous setup/teardown. Reach for it directly only when building a custom test bootstrapper.
- AbpWebApplicationFactoryIntegratedTest<TStartupModule, TStartup> (Volo.Abp.AspNetCore.TestBase) - Combines AbpIntegratedTest with ASP.NET Core's WebApplicationFactory to host the web app in-process and expose an HttpClient. Reach for it in .Web.Tests projects to drive Razor Pages / MVC / API endpoints over HTTP.
- IDataSeedContributor (Volo.Abp.Data) - Interface implemented in the .TestBase project (e.g. MyProjectTestDataSeedContributor : IDataSeedContributor, ITransientDependency) to populate fixed test data on startup; runs once per integrated test through the IDataSeeder pipeline.
- ITransientDependency (Volo.Abp.DependencyInjection) - Marker interface required on test data seed contributors so ABP auto-registers them in DI without an explicit module entry.
- IUnitOfWorkManager (Volo.Abp.Uow) - Resolved via GetRequiredService<IUnitOfWorkManager>() to wrap repository calls inside an explicit unit of work scope (uow.Begin(...) / WithUnitOfWorkAsync) when integration tests need transactional context.
- IDbContextProvider<TDbContext> (Volo.Abp.EntityFrameworkCore) - Resolves the active DbContext inside a unit of work; useful in EntityFrameworkCore.Tests when you must execute raw EF Core queries that the repository abstraction doesn't expose.
- ICurrentUser / ICurrentTenant - Resolved or substituted in tests to assert authorization-sensitive logic; integration tests typically log a user in via the test module configuration, while unit tests substitute the interface with NSubstitute.
- AbpEntityFrameworkCoreSqliteTestModule pattern - Convention where the EntityFrameworkCore.Tests test module replaces the production EF Core DbContext options with an in-memory SQLite connection so each test class boots a fresh schema.
- {ServiceName}TestsModule (microservice template) - Per-microservice test module that DependsOn the service's main module via [AdditionalAssembly] and swaps distributed-cache (Redis), event-bus (RabbitMQ), and outbox/inbox infrastructure for in-memory implementations.
- {ServiceName}TestBase (microservice template) - Concrete test base inheriting AbpIntegratedTest<{ServiceName}TestsModule> exposing GetRequiredService and HTTP helpers; concrete test classes derive from it.
- GetResponseAsStringAsync(url) / GetResponseAsObjectAsync<T>(url) (Volo.Abp.AspNetCore.TestBase) - Helper methods on the web test base that issue HTTP GET, assert 200, and either return the raw HTML or deserialize JSON for API assertions.
- xUnit / NSubstitute / Shouldly - Default test stack: xUnit drives discovery and the [Fact] / [Theory] attributes; NSubstitute provides Substitute.For<T>(), .Returns(), .Received() for unit-test mocks; Shouldly supplies fluent assertions like ShouldBe(), ShouldNotBeNull().

## Configuration pattern

Each test project is itself an ABP module. The .TestBase project (e.g. MyProjectTestBaseModule) DependsOn the production domain/EF Core/application modules, registers test data via IDataSeedContributor implementations, and overrides infrastructure services in ConfigureServices/PreConfigureServices (e.g. swap distributed cache for in-memory). Layer-specific test modules (MyProjectDomainTestModule, MyProjectApplicationTestModule, MyProjectEntityFrameworkCoreTestModule) DependsOn MyProjectTestBaseModule plus the layer being tested. Test classes inherit a project-specific base (e.g. MyProjectApplicationTestBase : AbpIntegratedTest<MyProjectApplicationTestModule>) and call GetRequiredService<T>() in the constructor.

Example test module wiring:

[DependsOn(
    typeof(MyProjectApplicationModule),
    typeof(MyProjectEntityFrameworkCoreTestModule)
)]
public class MyProjectApplicationTestModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        Configure<AbpBackgroundJobOptions>(options => options.IsJobExecutionEnabled = false);
    }

    public override void OnApplicationInitialization(ApplicationInitializationContext context)
    {
        SeedTestData(context);
    }

    private static void SeedTestData(ApplicationInitializationContext context)
    {
        AsyncHelper.RunSync(async () =>
        {
            using var scope = context.ServiceProvider.CreateScope();
            await scope.ServiceProvider.GetRequiredService<IDataSeeder>().SeedAsync();
        });
    }
}

In the EF Core test module, replace AddAbpDbContext options with an in-memory SQLite connection that is opened for the lifetime of the test application. For microservices, the {ServiceName}TestsModule registers the service module via AdditionalAssembly attribute (instead of [DependsOn]) so the test module fully owns infrastructure substitution.

## Code examples

### Unit test for an entity (no dependencies)

_Domain.Tests project verifying that calling Close() on an Issue sets IsClosed to true._

```csharp
using Shouldly;
using Xunit;

namespace MyProject.Issues;

public class Issue_Tests
{
    [Fact]
    public void Should_Close_An_Open_Issue()
    {
        // Arrange
        var issue = new Issue();

        // Act
        issue.Close();

        // Assert
        issue.IsClosed.ShouldBeTrue();
    }
}
```

**Key lines:** [Fact] marks an xUnit test; ShouldBeTrue() comes from Shouldly. No ABP infrastructure is needed because Issue is a plain entity.

### Unit test for a domain service with NSubstitute

_Mock IIssueRepository to verify IssueManager enforces the user's open-issue quota._

```csharp
using NSubstitute;
using Shouldly;
using Volo.Abp;
using Xunit;

namespace MyProject.Issues;

public class IssueManager_Tests
{
    [Fact]
    public async Task Should_Not_Assign_Issue_When_User_Has_Three_Open_Issues()
    {
        // Arrange
        var userId = Guid.NewGuid();
        var fakeRepo = Substitute.For<IIssueRepository>();
        fakeRepo.GetIssueCountOfUserAsync(userId).Returns(3);

        var issueManager = new IssueManager(fakeRepo);
        var issue = new Issue();

        // Act + Assert
        await Assert.ThrowsAsync<BusinessException>(
            async () => await issueManager.AssignToAsync(issue, userId)
        );
        await fakeRepo.Received(1).GetIssueCountOfUserAsync(userId);
    }
}
```

**Key lines:** Substitute.For<T>() builds the mock; .Returns() seeds it; .Received(n) verifies a call count. Assert.ThrowsAsync<BusinessException> covers ABP's domain exception type.

### Test data seed contributor

_Seed a known Issue row inside the .TestBase project so integration tests can query it._

```csharp
using System;
using System.Threading.Tasks;
using Volo.Abp.Data;
using Volo.Abp.DependencyInjection;
using Volo.Abp.Domain.Repositories;

namespace MyProject;

public class MyProjectTestDataSeedContributor : IDataSeedContributor, ITransientDependency
{
    public static readonly Guid TestIssueId = Guid.NewGuid();

    private readonly IRepository<Issue, Guid> _issueRepository;

    public MyProjectTestDataSeedContributor(IRepository<Issue, Guid> issueRepository)
    {
        _issueRepository = issueRepository;
    }

    public async Task SeedAsync(DataSeedContext context)
    {
        await _issueRepository.InsertAsync(
            new Issue(TestIssueId, "Seeded test issue"),
            autoSave: true
        );
    }
}
```

**Key lines:** Implementing IDataSeedContributor + ITransientDependency in the .TestBase project causes ABP's IDataSeeder (invoked from the test module's OnApplicationInitialization) to run this on every test bootstrap.

### Integration test for an application service

_Resolve IIssueAppService through the configured DI container, exercise it end-to-end against in-memory SQLite, and assert via the repository._

```csharp
using System.Threading.Tasks;
using Microsoft.Extensions.DependencyInjection;
using Shouldly;
using Volo.Abp.Domain.Repositories;
using Xunit;

namespace MyProject.Issues;

public class IssueAppService_Tests : MyProjectApplicationTestBase
{
    private readonly IIssueAppService _issueAppService;
    private readonly IRepository<Issue, Guid> _issueRepository;

    public IssueAppService_Tests()
    {
        _issueAppService = GetRequiredService<IIssueAppService>();
        _issueRepository = GetRequiredService<IRepository<Issue, Guid>>();
    }

    [Fact]
    public async Task Should_Create_A_New_Issue()
    {
        var dto = await _issueAppService.CreateAsync(new IssueCreationDto { Title = "Bug" });

        dto.Id.ShouldNotBe(Guid.Empty);
        (await _issueRepository.FindAsync(dto.Id)).ShouldNotBeNull();
    }
}
```

**Key lines:** MyProjectApplicationTestBase derives from AbpIntegratedTest<MyProjectApplicationTestModule>, so GetRequiredService<T>() uses the same DI container that production code uses; the in-memory SQLite database is seeded via IDataSeedContributor before the constructor runs.

### Web test against a Razor Page (HTTP-level)

_Use AbpWebApplicationFactoryIntegratedTest to perform an HTTP GET against /Issues and parse the response with HtmlAgilityPack._

```csharp
using System.Threading.Tasks;
using HtmlAgilityPack;
using Shouldly;
using Xunit;

namespace MyProject.Pages.Issues;

public class Index_Tests : MyProjectWebTestBase
{
    [Fact]
    public async Task Should_Get_Table_Of_Issues()
    {
        var response = await GetResponseAsStringAsync("/Issues");

        var html = new HtmlDocument();
        html.LoadHtml(response);

        var table = html.GetElementbyId("IssueTable");
        table.ShouldNotBeNull();
    }
}
```

**Key lines:** MyProjectWebTestBase inherits AbpWebApplicationFactoryIntegratedTest; GetResponseAsStringAsync asserts a 200 response and returns the body. Use GetResponseAsObjectAsync<T>("/api/issues") for JSON endpoints. For client-side UI testing (Angular/Blazor), refer to the abp-frontend skill.

## Common mistakes

### Putting IDataSeedContributor implementations in the production project instead of the .TestBase project.

**Why wrong:** Production seed runs against real environments; mixing test fixtures there pollutes real databases and may fail uniqueness checks. The test module's OnApplicationInitialization is the correct trigger for IDataSeeder.SeedAsync().

**Correct pattern:** Place MyProjectTestDataSeedContributor : IDataSeedContributor, ITransientDependency in the .TestBase project; let production seeders live in the .Domain or .DbMigrator project.

### Calling repositories outside an active unit of work in integration tests.

**Why wrong:** Some EF Core operations (lazy navigation, transactional behavior) require a unit-of-work scope; without one, you get ObjectDisposedException or incorrect transaction semantics.

**Correct pattern:** Wrap reads/writes in IUnitOfWorkManager.Begin() or use WithUnitOfWorkAsync(async () => { ... }) inside the test.

### Inheriting AbpIntegratedTest<T> directly in every test class.

**Why wrong:** Each constructor of AbpIntegratedTest spins up a full ABP application, which is expensive and breaks DRY when many tests share configuration.

**Correct pattern:** Use the project-specific test base (MyProjectApplicationTestBase, MyProjectDomainTestBase, MyProjectWebTestBase) the template generates and add helpers there.

### Trying to assert UI behavior using ABP-supplied JavaScript test infrastructure for MVC/Razor Pages.

**Why wrong:** ABP does not ship JS testing infrastructure for Razor Pages; assertions must run server-side against rendered HTML.

**Correct pattern:** Use AbpWebApplicationFactoryIntegratedTest with HtmlAgilityPack for HTML inspection, or stand up Selenium / Playwright separately for visual tests. Defer Angular / Blazor / browser-side test stacks to the abp-frontend skill.

### Registering microservice modules with [DependsOn] in {ServiceName}TestsModule.

**Why wrong:** [DependsOn] would force the production infrastructure (Redis, RabbitMQ) to load, defeating the goal of an isolated test bootstrap.

**Correct pattern:** Use the AdditionalAssembly attribute pattern shown in the microservice template: register the service module's assembly without taking a hard module dependency, then override the cache, event bus, and outbox/inbox with in-memory equivalents.

### Sharing static state across xUnit test classes expecting database isolation.

**Why wrong:** ABP's in-memory SQLite gives each test class its own connection-scoped database; static caches (e.g. cached query results) survive across classes and create false positives.

**Correct pattern:** Treat each constructor as a fresh boot; never store cross-test state in statics, and clear distributed cache via IDistributedCache test overrides if you must.

## Version pins (ABP 10.3)

- ABP 10.3 keeps NSubstitute, Shouldly, and xUnit as the default test stack across all solution templates; replacing them is documented as supported but on the developer.
- ABP 10.3 uses in-memory SQLite (Microsoft.Data.Sqlite shared-connection mode) for EF Core integration tests and EphemeralMongo for MongoDB integration tests; the choice of EphemeralMongo is a 10.x convention that may shift in later versions.
- Volo.Abp.TestBase exposes AbpIntegratedTest<TStartupModule>; Volo.Abp.AspNetCore.TestBase exposes AbpWebApplicationFactoryIntegratedTest. The exact generic-parameter shape and helper method names (GetResponseAsStringAsync / GetResponseAsObjectAsync) are stable in 10.3 but treat any extension method beyond these as version-pinned. [uncertain]
- Microservice template uses {ServiceName}TestsModule plus [AdditionalAssembly] to swap Redis with in-memory cache and RabbitMQ with the in-memory event bus; the exact attribute name and substitution APIs are 10.3-specific. [uncertain]
- ABP 10.3 explicitly states 'Visual UI Testing is out of the scope for the ABP'; this stance has held across recent versions but is a policy statement that could change.

## Cross-references

**Phase 1 references:**
- references/framework/architecture-ddd.md — triggered when a test exercises domain services, aggregates, or repositories so the reader understands the layering being tested.
- references/framework/fundamentals.md — triggered when explaining IServiceProvider, ITransientDependency on seed contributors, and GetRequiredService<T>() in test bases.
- references/framework/architecture-modularity.md — triggered when describing test modules, [DependsOn], and the AdditionalAssembly attribute used by the microservice template tests.
- references/framework/data-ef-core.md — triggered when EntityFrameworkCore.Tests use IDbContextProvider<T> or substitute SQLite for the production provider.
- references/framework/authorization.md — triggered when tests substitute ICurrentUser / ICurrentTenant or assert permission-protected services.
- references/templates/microservice.md — triggered when the user follows the per-service {ServiceName}TestsModule / {ServiceName}TestBase pattern.
- (UI test guidance for Angular / Blazor / MVC client-side) — see the abp-frontend skill.

**External docs:**
- [xUnit.net](https://xunit.net/) — Default test runner / assertion attributes ([Fact], [Theory]) used by every ABP .NET test project.
- [NSubstitute](https://nsubstitute.github.io/) — Mocking library used in unit tests for Substitute.For<T>(), .Returns(), and .Received() verifications.
- [Shouldly](https://docs.shouldly.org/) — Fluent assertion library shipped in every ABP test template (ShouldBe, ShouldNotBeNull, ShouldThrowAsync).
- [ASP.NET Core WebApplicationFactory](https://learn.microsoft.com/aspnet/core/test/integration-tests) — Underlying test host that AbpWebApplicationFactoryIntegratedTest extends; explains the in-process server lifecycle.
- [EF Core SQLite in-memory](https://learn.microsoft.com/ef/core/testing/sqlite) — ABP integration tests use SQLite in-memory mode (Microsoft.Data.Sqlite with shared connection) for isolated, fast EF Core test runs.
- [EphemeralMongo](https://github.com/asimmon/ephemeral-mongo) — Used by ABP MongoDB integration tests as the in-memory MongoDB replacement equivalent to in-memory SQLite.
- [HtmlAgilityPack](https://html-agility-pack.net/) — Recommended HTML parser for asserting elements in Razor Pages / MVC web tests via GetResponseAsStringAsync.
- [Selenium](https://www.selenium.dev/) — Suggested external tool for visual UI tests, since ABP explicitly states visual UI testing is out of scope.

## Sources (ABP 10.3)

- https://abp.io/docs/10.3/testing/overall — Overview of unit/integration/UI testing layers, test project layout (Domain.Tests, Application.Tests, EntityFrameworkCore.Tests, Web.Tests, TestBase), default libraries (xUnit, NSubstitute, Shouldly).
- https://abp.io/docs/10.3/testing/unit-tests — AAA pattern, testing entities directly, testing services with NSubstitute mocks, [Fact], Substitute.For<T>(), Received().
- https://abp.io/docs/10.3/testing/integration-tests — AbpIntegratedTest<T>, in-memory SQLite (EphemeralMongo for MongoDB), IDataSeedContributor pattern, IUnitOfWorkManager/WithUnitOfWorkAsync, IDbContextProvider<T>, GetRequiredService<T>().
- https://abp.io/docs/10.3/testing/ui-tests — Distinction between Non-Visual and Visual UI tests, statement that Visual UI Testing is out of scope for ABP and developers should use tools like Selenium.
- https://abp.io/docs/10.3/framework/ui/mvc-razor-pages/testing — Volo.Abp.AspNetCore.TestBase package, AbpWebApplicationFactoryIntegratedTest base, GetResponseAsStringAsync/GetResponseAsObjectAsync helpers, HtmlAgilityPack for HTML assertions.
- https://abp.io/docs/10.3/solution-templates/microservice/authoring-unit-and-integration-tests — Per-microservice {ServiceName}TestsModule and {ServiceName}TestBase pattern, AdditionalAssembly registration to swap Redis/RabbitMQ for in-memory equivalents, dotnet test execution.

Last verified: 2026-05-10
