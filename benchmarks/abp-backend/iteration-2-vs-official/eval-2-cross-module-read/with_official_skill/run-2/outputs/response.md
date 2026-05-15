# Cross-Module Read: `Acme.Reports` reading from `Acme.Forms`

## Short answer

**No — do not inject `IFormSubmissionAppService` directly into `ReportAppService`.**

`FormSubmissionAppService` is a UI-facing application service. It is owned by the Forms module's UI use cases (its authorization policies, validation, DTO shape, and auto-API exposure are all tuned for the end user calling the Forms UI). Reusing it from another module:

- couples Reports to Forms' UI permission policies (e.g. `Forms.Submissions.Default`) that may not match what a Reports caller should be allowed to do,
- forces Reports to take a transitive dependency on Forms' full `Application` project (implementation), which violates the layered dependency direction rules,
- breaks the rule "**Don't call other app services across modules**" (the cross-module variant of the same rule that says don't call other app services in the same module — `abp-application-layer` lists it as an explicit anti-pattern, and `abp-dependency-rules` lists "Cross-module app service call" as a tight-coupling violation to fix with events or a domain/integration boundary).

The correct pattern in an ABP **layered modular monolith** is an **Integration Service** (the same concept used in the microservice template, but kept as an in-process interface here). The Forms module publishes a stable, narrow contract for "things other modules are allowed to ask Forms about", and Reports depends only on that contract.

## The pattern, project-by-project

For a layered module the relevant projects are:

```
Acme.Forms.Domain.Shared
Acme.Forms.Domain
Acme.Forms.Application.Contracts   <-- integration-service INTERFACE + DTO lives here
Acme.Forms.Application             <-- integration-service IMPLEMENTATION lives here
Acme.Forms.EntityFrameworkCore
Acme.Forms.HttpApi
Acme.Forms.HttpApi.Client          <-- (used in the microservice variant; see bottom)
...

Acme.Reports.Application.Contracts
Acme.Reports.Application           <-- consumes IFormSubmissionIntegrationService
```

### 1. Interface + DTO — in `Acme.Forms.Application.Contracts`

This is the only Forms project Reports is allowed to reference. It is the public, stable surface of the Forms module.

```csharp
// Acme.Forms.Application.Contracts/IntegrationServices/IFormSubmissionIntegrationService.cs
using System;
using System.Threading.Tasks;
using Volo.Abp.Application.Services;

namespace Acme.Forms.IntegrationServices;

[IntegrationService]
public interface IFormSubmissionIntegrationService : IApplicationService
{
    Task<OwnerSubmissionCountDto> GetSubmissionCountByOwnerAsync(Guid ownerId);
}
```

```csharp
// Acme.Forms.Application.Contracts/IntegrationServices/OwnerSubmissionCountDto.cs
using System;

namespace Acme.Forms.IntegrationServices;

public class OwnerSubmissionCountDto
{
    public Guid OwnerId { get; set; }
    public long SubmissionCount { get; set; }
}
```

Key points:

- The interface is decorated with `[IntegrationService]`. That marker tells ABP this is an inter-module/inter-service contract, not a UI use case. By default, integration services are **not** exposed as auto-API endpoints (you opt in with `ExposeIntegrationServices = true` only if you actually want them on HTTP — in a single-process monolith you usually don't).
- It still derives from `IApplicationService` so it benefits from ABP conventions (DI registration, interception, UoW, validation).
- The DTO is a brand-new, narrow type — **don't** reuse `FormSubmissionDto` from the UI service. The integration contract should be the minimal shape Reports needs and should be allowed to evolve independently of the UI DTO.
- Authorization on integration services is typically not the UI permission. If access control is needed, put a dedicated permission (e.g. `Forms.Integration.ReadSubmissionStats`) on the implementation, or omit `[Authorize]` and rely on the fact that it isn't exposed over HTTP.

### 2. Implementation — in `Acme.Forms.Application`

```csharp
// Acme.Forms.Application/IntegrationServices/FormSubmissionIntegrationService.cs
using System;
using System.Threading.Tasks;
using Acme.Forms.FormSubmissions;          // domain
using Volo.Abp.Application.Services;
using Volo.Abp.Domain.Repositories;

namespace Acme.Forms.IntegrationServices;

[IntegrationService]
public class FormSubmissionIntegrationService
    : ApplicationService, IFormSubmissionIntegrationService
{
    private readonly IRepository<FormSubmission, Guid> _submissionRepository;

    public FormSubmissionIntegrationService(
        IRepository<FormSubmission, Guid> submissionRepository)
    {
        _submissionRepository = submissionRepository;
    }

    public virtual async Task<OwnerSubmissionCountDto> GetSubmissionCountByOwnerAsync(
        Guid ownerId)
    {
        var count = await _submissionRepository.CountAsync(s => s.OwnerId == ownerId);

        return new OwnerSubmissionCountDto
        {
            OwnerId = ownerId,
            SubmissionCount = count
        };
    }
}
```

Key points:

- It goes against the **repository / domain service**, not against `FormSubmissionAppService`. The query logic that already exists in `FormSubmissionAppService` should be factored down into the domain (repository extension method or a `FormSubmissionManager` domain service) so both the UI app service and the integration service call the same domain code. The integration service must not call the UI app service.
- `virtual` is kept on the public method so that consumers of the Forms module can override it (per `abp-module` extensibility rules — applies because the module will be consumed by other modules in the same solution).
- Lives in `Acme.Forms.Application`, which is **not** referenced by Reports.

### 3. Consumer wiring — `Acme.Reports.Application`

Reports adds a `[DependsOn]` on the Forms **Application.Contracts module**, and only on that. It must never depend on `Acme.Forms.Application` or `Acme.Forms.EntityFrameworkCore`.

```csharp
// Acme.Reports.Application/AcmeReportsApplicationModule.cs
using Acme.Forms;
using Acme.Reports;
using Volo.Abp.Application;
using Volo.Abp.Modularity;

namespace Acme.Reports;

[DependsOn(
    typeof(AcmeReportsApplicationContractsModule),
    typeof(AcmeReportsDomainModule),
    typeof(AbpDddApplicationModule),
    // Cross-module dependency: ONLY the Contracts module of Forms.
    typeof(AcmeFormsApplicationContractsModule)
)]
public class AcmeReportsApplicationModule : AbpModule
{
}
```

`csproj` of `Acme.Reports.Application` references **only**:

```xml
<ProjectReference Include="..\Acme.Forms.Application.Contracts\Acme.Forms.Application.Contracts.csproj" />
```

Never reference `Acme.Forms.Application.csproj` from Reports — that would re-introduce the tight coupling we are trying to avoid. The final host (the modular-monolith Web/HttpApi.Host) is what loads both `AcmeFormsApplicationModule` and `AcmeReportsApplicationModule`, and ABP's DI resolves the interface to the in-process implementation there.

### 4. Consume from `ReportAppService`

```csharp
// Acme.Reports.Application/Reports/ReportAppService.cs
using System;
using System.Threading.Tasks;
using Acme.Forms.IntegrationServices;
using Acme.Reports.Reports;
using Volo.Abp.Application.Services;

namespace Acme.Reports.Reports;

public class ReportAppService : ApplicationService, IReportAppService
{
    private readonly IFormSubmissionIntegrationService _formSubmissions;

    public ReportAppService(IFormSubmissionIntegrationService formSubmissions)
    {
        _formSubmissions = formSubmissions;
    }

    public virtual async Task<OwnerReportDto> GetOwnerReportAsync(Guid ownerId)
    {
        var stats = await _formSubmissions.GetSubmissionCountByOwnerAsync(ownerId);

        return new OwnerReportDto
        {
            OwnerId = ownerId,
            FormSubmissions = stats.SubmissionCount
            // ...combine with other modules' integration data here
        };
    }
}
```

Because the host loads both modules in the same process, DI resolves `IFormSubmissionIntegrationService` to the in-process implementation from `Acme.Forms.Application`. No HTTP, no proxies, no serialization — it's an ordinary method call.

## Why this respects the ABP dependency rules

Per `abp-dependency-rules`:

| Project | May reference |
|---|---|
| `Acme.Reports.Application.Contracts` | `Acme.Forms.Application.Contracts` (interface + DTO only) |
| `Acme.Reports.Application` | `Acme.Reports.Application.Contracts`, `Acme.Reports.Domain`, `Acme.Forms.Application.Contracts` |
| `Acme.Reports.Application` | **NOT** `Acme.Forms.Application`, **NOT** `Acme.Forms.Domain`, **NOT** `Acme.Forms.EntityFrameworkCore` |

Reports never sees Forms entities, Forms' DbContext, or Forms' UI service implementation. It sees only a small, intentional contract — exactly the kind of seam that lets you later split Forms out into its own microservice without changing a line in Reports.

## Modular-monolith vs microservice: the transport difference

This is the whole point of using the integration-service pattern even inside a monolith — **the contract stays identical; only the transport changes.**

### Layered modular monolith (your current setup, ABP 10.3)

- All modules run in the **same process / same host**.
- `Acme.Reports.Application` `[DependsOn]` `AcmeFormsApplicationContractsModule`; the host references both Application modules so both are loaded in the same DI container.
- DI resolves `IFormSubmissionIntegrationService` to the **concrete `FormSubmissionIntegrationService`** in `Acme.Forms.Application`.
- The call is an **in-process method call**. No HTTP, no serialization, shares the same `IUnitOfWork`, the same `ICurrentUser`, the same `ICurrentTenant`, the same DB transaction if you want one.
- You do **not** need `AddStaticHttpClientProxies`, do **not** need `generate-proxy`, do **not** need `RemoteServices` config, and do **not** set `ExposeIntegrationServices = true`. The integration-service interface is purely a logical seam.

### Microservice template (future-state, if you split Forms out)

Per `abp-microservice`, the exact same `IFormSubmissionIntegrationService` interface and DTO in `Acme.Forms.Application.Contracts` (or in microservice naming, `Acme.Forms.Contracts`) become the **HTTP contract** between services. The transport now changes:

1. Forms (provider) sets `options.ExposeIntegrationServices = true` so the integration service is exposed as an HTTP endpoint, segregated from regular auto-API endpoints.
2. Reports (consumer) generates a typed HTTP client proxy:
   ```bash
   abp generate-proxy -t csharp -u http://forms-service -m forms --without-contracts
   ```
3. Reports registers the proxy in its module:
   ```csharp
   context.Services.AddStaticHttpClientProxies(
       typeof(AcmeFormsApplicationContractsModule).Assembly,
       remoteServiceConfigurationName: "FormsService");
   ```
4. Reports configures the remote base URL:
   ```json
   "RemoteServices": {
     "FormsService": { "BaseUrl": "http://forms-service" }
   }
   ```
5. The injection site in `ReportAppService` (`private readonly IFormSubmissionIntegrationService _formSubmissions;`) **does not change**. DI now resolves the interface to the generated **HTTP client proxy** instead of the in-process implementation. Every call becomes an HTTP request, with its own transaction boundary, its own DB, its own auth token forwarded by ABP.

That symmetry — same interface, swappable transport — is exactly why ABP recommends Integration Services rather than reaching across into another module's `AppService`, even when "everything is in one process today".

### When *not* to use a synchronous integration service

Per `abp-microservice` / `abp-infrastructure`: if Reports doesn't need a live read but instead is maintaining a denormalized view ("how many submissions does each owner have"), the better pattern is **distributed events** (`IDistributedEventBus` + Outbox/Inbox) — Forms publishes `FormSubmissionCreatedEto` / `FormSubmissionDeletedEto`, Reports subscribes and increments/decrements its own counter. That decouples Reports entirely and survives Forms being offline. Use synchronous integration services when Reports genuinely needs the current count at the moment the report is generated.

---

**Skills cited:** `abp-dependency-rules` (cross-module app-service-call violation, project reference matrix, contracts-only sharing), `abp-microservice` (`[IntegrationService]` attribute, contracts-only sharing, `ExposeIntegrationServices`, `AddStaticHttpClientProxies`, `generate-proxy`, sync-vs-async guidance), `abp-application-layer` ("don't call other app services" anti-pattern, DTOs live in Application.Contracts), `abp-module` (virtual methods for module extensibility), `abp-core` (`[DependsOn]` module wiring, auto-DI via `IApplicationService`).
