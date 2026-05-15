# Cross-Module Reads: `Acme.Reports` reading from `Acme.Forms`

## Short answer

**No — do not inject `IFormSubmissionAppService` into `ReportAppService`.** That cross-module app-service call is the exact pattern the ABP guidance flags as a violation:

- `abp-dependency-rules` lists **"Cross-module app service call"** as a common violation, fix: *"Use events or domain"* (i.e. don't bind module B's UI-facing app service into module A's call graph).
- `abp-application-layer` lists **"Calling other app services in the same module"** as an anti-pattern — and crossing a module boundary is even stronger: app services are shaped for UI (their auth, validation, paging, auto-API exposure, and DTOs are tailored to that). They are not a stable inter-module contract.

The right pattern is an **ABP Integration Service**: a dedicated, `[IntegrationService]`-marked contract defined in the producing module's `*.Application.Contracts` project, implemented in `*.Application`, and consumed via DI from the other module. Integration services are hidden from the public `/api` surface, audit-off by default, and don't get `[Authorize]` attributes by default — they're explicitly designed for module-to-module reads.

For cross-aggregate reads *inside* the same module you wouldn't reach for this at all — that's a `*Manager` with `IReadOnlyRepository<TOther, TKey>`. Integration services are for cross-**module** reads, which is your case.

---

### 1. Contract — `Acme.Forms.Application.Contracts`

The interface lives in the producing module's **`*.Application.Contracts`** project (alongside `IFormSubmissionAppService`, permission constants, and DTOs). This is the project that ships the public-facing definitions other modules are allowed to depend on.

```csharp
// Acme.Forms.Application.Contracts/IntegrationServices/IFormSubmissionIntegrationService.cs
using System;
using System.Threading.Tasks;
using Volo.Abp.Application.Services;

namespace Acme.Forms.IntegrationServices;

[IntegrationService]   // hides from /api, audit-off, no [Authorize] by default
public interface IFormSubmissionIntegrationService : IApplicationService
{
    Task<int> GetSubmissionCountForOwnerAsync(Guid ownerId);
}
```

### 2. Implementation — `Acme.Forms.Application`

The implementation lives in the producing module's **`*.Application`** project (next to `FormSubmissionAppService`). It injects the Forms-owned repository directly — that repository is internal to Forms and stays internal; only the integration-service interface crosses the boundary.

```csharp
// Acme.Forms.Application/IntegrationServices/FormSubmissionIntegrationService.cs
using System;
using System.Threading.Tasks;
using Acme.Forms.FormSubmissions;
using Volo.Abp.Application.Services;
using Volo.Abp.Domain.Repositories;

namespace Acme.Forms.IntegrationServices;

public class FormSubmissionIntegrationService
    : ApplicationService, IFormSubmissionIntegrationService
{
    private readonly IReadOnlyRepository<FormSubmission, Guid> _submissionRepository;

    public FormSubmissionIntegrationService(
        IReadOnlyRepository<FormSubmission, Guid> submissionRepository)
    {
        _submissionRepository = submissionRepository;
    }

    public Task<int> GetSubmissionCountForOwnerAsync(Guid ownerId)
        => _submissionRepository.CountAsync(s => s.OwnerId == ownerId);
}
```

`IReadOnlyRepository<,>` signals intent (no-tracking, read-only). Nothing else from Forms' internals leaks. Note: you can keep your existing query inside `FormSubmissionAppService` for the public API path, and have the integration service either call into Forms' own domain layer or duplicate the small `CountAsync(...)` — both are fine; what matters is the boundary contract.

### 3. Consume from `Acme.Reports.Application`

`ReportAppService` injects the integration-service interface like any other service — same DI shape as if it were in the same module.

```csharp
// Acme.Reports.Application/Reports/ReportAppService.cs
using System;
using System.Threading.Tasks;
using Acme.Forms.IntegrationServices;
using Acme.Reports.Reports;
using Volo.Abp.Application.Services;
using Volo.Abp.Domain.Repositories;

namespace Acme.Reports.Reports;

public class ReportAppService : ApplicationService, IReportAppService
{
    private readonly IFormSubmissionIntegrationService _formSubmissions;
    private readonly IRepository<Report, Guid>        _reportRepository;

    public ReportAppService(
        IFormSubmissionIntegrationService formSubmissions,
        IRepository<Report, Guid>         reportRepository)
    {
        _formSubmissions  = formSubmissions;
        _reportRepository = reportRepository;
    }

    public async Task<ReportDto> BuildForOwnerAsync(Guid ownerId)
    {
        var submissionCount =
            await _formSubmissions.GetSubmissionCountForOwnerAsync(ownerId);

        // ...assemble and persist the report...
        var report = new Report(GuidGenerator.Create(), ownerId, submissionCount);
        await _reportRepository.InsertAsync(report, autoSave: true);
        return ObjectMapper.Map<Report, ReportDto>(report);
    }
}
```

### 4. The load-bearing wiring — `[DependsOn]` contracts only

This is the line that actually preserves the boundary. The consumer module's `AbpModule` `[DependsOn]`s the producer's **`*.Application.Contracts`** module — *never* the producer's full `*.Application` module.

```csharp
// Acme.Reports.Application/AcmeReportsApplicationModule.cs
using Acme.Forms;                  // AcmeFormsApplicationContractsModule
using Acme.Reports;                // AcmeReportsDomainModule
using Volo.Abp.Application;
using Volo.Abp.Modularity;

namespace Acme.Reports;

[DependsOn(
    typeof(AcmeFormsApplicationContractsModule),   // RIGHT — contracts only
    // typeof(AcmeFormsApplicationModule),         // WRONG — would leak Forms' internals
    typeof(AcmeReportsDomainModule),
    typeof(AbpDddApplicationModule)
)]
public class AcmeReportsApplicationModule : AbpModule
{
}
```

If you `[DependsOn]` `AcmeFormsApplicationModule` (the full app module), `Acme.Reports` ends up with every Forms domain service, repository, and internal collaborator registered in the same DI container — every "private" piece of Forms becomes injectable inside Reports, and the boundary you drew on paper is gone in practice. Contracts-only `[DependsOn]` is what makes the seam real. The project-level `<ProjectReference>` in the `.csproj` is not enough on its own.

You will also need a matching `<ProjectReference Include="..\..\..\modules\Acme.Forms\src\Acme.Forms.Application.Contracts\Acme.Forms.Application.Contracts.csproj" />` (or `PackageReference` if Forms ships as NuGet) in `Acme.Reports.Application.csproj`.

---

### Modular monolith vs microservice — same contract, different transport

You said you're on a **Layered modular monolith** in ABP 10.3, so:

- **In-process (your case — modular monolith).** Both `Acme.Forms` and `Acme.Reports` are loaded into the same host. ABP's DI container resolves `IFormSubmissionIntegrationService` to the concrete `FormSubmissionIntegrationService` directly — it's a normal in-process method call, no HTTP hop, no serialization, no gateway. The `[IntegrationService]` attribute still buys you the API-surface hiding and audit-off defaults if/when the host happens to also expose integration endpoints, but for your Reports-to-Forms call it's pure DI.

- **Cross-process (microservice).** If you later split Forms and Reports into separate services, the *same* `IFormSubmissionIntegrationService` interface is what survives the split. ABP exposes it over `/integration-api/...` in the Forms service (behind the API gateway, not the public `/api`), and the Reports service consumes it through a generated proxy client that implements the same interface. The DI shape on both sides is identical to the in-process case — only the transport changes. That's the whole point of doing this via Integration Services rather than inventing a host-level shared type or sharing a DbContext: you get a clean seam today and a free upgrade path to out-of-process tomorrow.

### Two related rules worth flagging

- **No hand-written controller for the integration service.** Don't write a `Controller` class that implements `IFormSubmissionIntegrationService`. In the modular-monolith case the call is in-process DI — no HTTP at all. In the microservice case ABP wires HTTP exposure implicitly through the integration-services pipeline once the producer sets `Configure<AbpAspNetCoreMvcOptions>(o => o.ExposeIntegrationServices = true)` and the consumer registers static client proxies (`AddStaticHttpClientProxies(typeof(AcmeFormsApplicationContractsModule).Assembly, "Forms")` plus a `RemoteServices:Forms:BaseUrl` setting) — see `abp-microservice`.
- **`[Authorize]`.** Integration services are audit-off and `[Authorize]`-off by default because they're a server-to-server seam. If a specific operation needs an explicit permission check (e.g., when consumed across a tenant boundary), put `[Authorize(...)]` on the integration-service method — same rule as application services: authorization lives on the service, not on any controller.
- **Methods should be `virtual` if Forms is a reusable module.** `abp-module` requires `public`/`protected` methods on a reusable module's services to be `virtual` so consumers can override; if `Acme.Forms` is a distributable module, mark `GetSubmissionCountForOwnerAsync` as `public virtual`.

---

### Sources

Official ABP skills consulted:
- **`abp-dependency-rules`** — "Cross-module app service call → Tight coupling → Use events or domain"; Contracts-only consumption rule (consumers depend on `*.Application.Contracts`, never on `*.Application`).
- **`abp-application-layer`** — anti-pattern: "Calling other app services in the same module: use domain services or repositories directly" (the rule is even stronger across modules).
- **`abp-microservice`** — `[IntegrationService]` pattern, producer-side `ExposeIntegrationServices`, consumer-side `AddStaticHttpClientProxies` / `generate-proxy`, sync-vs-async decision (integration services for reads, distributed events for state changes).
- **`abp-module`** — virtual methods on integration-service implementations; Contracts project as the public surface for reusable modules.
- **`abp-core`** — `[DependsOn]` module wiring; automatic DI registration of `ApplicationService` subclasses.
- **`abp-ddd`** — distributed events / ETOs in `Domain.Shared` for the asynchronous counterpart to integration services.
