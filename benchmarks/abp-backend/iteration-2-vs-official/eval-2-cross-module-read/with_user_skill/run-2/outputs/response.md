## No — don't inject `IFormSubmissionAppService` into `ReportAppService`

In an ABP modular monolith, calling another module's `IApplicationService` directly across the module boundary is the anti-pattern Rule 5 exists to prevent. An application service is the producing module's caller-facing surface: it carries `[Authorize(...)]` for the producer's permissions, sits behind the producer's auto-API and UoW conventions, may be tenant-filtered for the producer's tenant context, and pulls in producer-internal collaborators (managers, repositories, mappers) into your consumer's DI graph if you `[DependsOn]` the wrong module to make the type resolvable. It is not a stable inter-module contract.

The right pattern is an ABP **Integration Service**: a separate, narrow contract defined in the producer's `*.Application.Contracts` project, marked `[IntegrationService]`, with audit-off and no `[Authorize]` by default, consumed from the other module via plain DI. Forms' existing `FormSubmissionAppService` keeps its query for HTTP/UI callers; you add a sibling integration service that exposes only the count.

The same `[IntegrationService]` interface is what stays portable when Forms is later extracted into its own microservice — see the transport note at the bottom.

---

## 1. Contract — `Acme.Forms.Application.Contracts`

The interface lives in the producer's **`*.Application.Contracts`** project (`Acme.Forms.Application.Contracts`). That's the project the consumer is allowed to depend on; the interface must be reachable from there.

```csharp
// Acme.Forms.Application.Contracts/IntegrationServices/IFormSubmissionIntegrationService.cs
using System;
using System.Threading.Tasks;
using Volo.Abp.Application.Services;

namespace Acme.Forms;

[IntegrationService]   // hidden from /api, audit-off, no [Authorize] by default
public interface IFormSubmissionIntegrationService : IApplicationService
{
    Task<int> GetSubmissionCountForOwnerAsync(Guid ownerId);
}
```

Keep the surface deliberately narrow — one purpose-built method (`GetSubmissionCountForOwnerAsync`), not a re-export of `FormSubmissionAppService`. Integration contracts are versioning hot spots; the smaller, the better.

---

## 2. Implementation — `Acme.Forms.Application`

The implementation lives in the producer's **`*.Application`** project (`Acme.Forms.Application`), alongside the regular application services. ABP auto-registers it via conventional DI; no `[ExposeServices]` needed.

```csharp
// Acme.Forms.Application/IntegrationServices/FormSubmissionIntegrationService.cs
using System;
using System.Threading.Tasks;
using Acme.Forms.FormSubmissions;          // wherever the FormSubmission entity lives
using Volo.Abp.Application.Services;
using Volo.Abp.Domain.Repositories;

namespace Acme.Forms;

public class FormSubmissionIntegrationService
    : ApplicationService, IFormSubmissionIntegrationService
{
    private readonly IRepository<FormSubmission, Guid> _submissionRepository;

    public FormSubmissionIntegrationService(
        IRepository<FormSubmission, Guid> submissionRepository)
    {
        _submissionRepository = submissionRepository;
    }

    public Task<int> GetSubmissionCountForOwnerAsync(Guid ownerId)
        => _submissionRepository.CountAsync(s => s.OwnerId == ownerId);
}
```

Notes that fall out of Rule 5:
- **No explicit controller.** Integration services don't follow Rule 4's explicit-controller split — ABP wires their HTTP exposure (when enabled) through the integration-services pipeline. Don't hand-write a controller that implements an `[IntegrationService]` interface.
- **No `[Authorize]`** by default. Trust boundary for integration services is the module/service edge, not per-user permission. If a specific call truly needs a permission check, add it deliberately — but the default is correct for an internal cross-module read.
- **Reuse the existing query** by extracting the count logic into a method called by both `FormSubmissionAppService` and `FormSubmissionIntegrationService` (e.g., a custom repository or a domain service) — or have the integration service simply re-issue the `CountAsync` predicate as shown. Don't have the integration service delegate to `FormSubmissionAppService`; the AppService isn't the contract.

---

## 3. Consumer — `Acme.Reports.Application`

Inject the integration-service interface like any other ABP service. No HTTP client, no proxy registration — in a modular monolith both modules live in the same host and DI resolves to the in-process class above.

```csharp
// Acme.Reports.Application/Reports/ReportAppService.cs
using System;
using System.Threading.Tasks;
using Acme.Forms;                            // the integration-service interface namespace
using Volo.Abp.Application.Services;
using Volo.Abp.Domain.Repositories;

namespace Acme.Reports.Reports;

public class ReportAppService : ApplicationService, IReportAppService
{
    private readonly IFormSubmissionIntegrationService _formSubmissions;
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
        var submissionCount =
            await _formSubmissions.GetSubmissionCountForOwnerAsync(ownerId);

        // ...assemble report...
        return new ReportDto { /* ... */ };
    }
}
```

---

## 4. Module wiring — the load-bearing line

The consumer's `AbpModule` `[DependsOn]` the producer's **`*.Application.Contracts`** module only — never the producer's full `*.Application`. This is what keeps the boundary real; a project-level `<ProjectReference>` on `Application.Contracts` is necessary but not sufficient on its own.

```csharp
// Acme.Reports.Application/AcmeReportsApplicationModule.cs
using Acme.Forms;                                                  // namespace of AcmeFormsApplicationContractsModule
using Acme.Reports.Domain;
using Volo.Abp.Application;
using Volo.Abp.Modularity;

namespace Acme.Reports;

[DependsOn(
    typeof(AcmeFormsApplicationContractsModule),   // RIGHT — contracts only
    // typeof(AcmeFormsApplicationModule),         // WRONG — leaks Forms internals into Reports' DI
    typeof(AcmeReportsDomainModule),
    typeof(AbpDddApplicationModule)
)]
public class AcmeReportsApplicationModule : AbpModule { }
```

Why this matters: if you `[DependsOn]` the producer's full `*.Application`, the consumer loads the producer's domain services, custom repositories, mappers, and internal collaborators into the same DI container. Every "private" piece of Forms becomes injectable in Reports, and the boundary collapses in practice even though the source files still live in separate folders. Contracts-only `[DependsOn]` is what makes the seam real.

Who registers the implementation? The host (typically the Web/HttpApi.Host app) `[DependsOn]` `AcmeFormsApplicationModule` — that's what brings `FormSubmissionIntegrationService` into the running container so DI can resolve `IFormSubmissionIntegrationService` for `ReportAppService`. The Reports module itself never sees that type.

`ExposeIntegrationServices` stays at its default (`false`) — there is no HTTP hop and no `/integration-api/...` routes are needed in this monolith.

---

## 5. Modular monolith vs microservice — same contract, different transport

The interface above is deliberately the **same shape** in both deployments. What changes is how DI satisfies it.

| | Modular monolith (your case) | Microservice |
|---|---|---|
| Where both modules run | Same host process | Separate processes behind the API gateway |
| How `IFormSubmissionIntegrationService` resolves in the consumer | Direct in-process resolution to `FormSubmissionIntegrationService` registered by Forms' app module | A generated HTTP proxy client (`IHttpClientProxy<IFormSubmissionIntegrationService>` via `AddStaticHttpClientProxies` / `AddHttpClientProxies`) calling the Forms service through the gateway at `/integration-api/...` |
| HTTP routes for the integration service | None — not exposed | Exposed under `/integration-api/...` on the Forms service host; gateway must block external traffic to that prefix |
| `ExposeIntegrationServices` on the producing host | `false` (default) | `true` on the Forms service host |
| `appsettings.json` `RemoteServices` entry for the consumer | Not needed | Required (`"Forms": { "BaseUrl": "..." }`) |
| Consumer code (`ReportAppService`) | Unchanged | Unchanged — still injects `IFormSubmissionIntegrationService` |
| Latency, failure modes | In-process call: no network, no retry/timeout policy needed | Network call: needs Polly retry/circuit-breaker, timeouts, distributed tracing, and gateway routing |

Because consumer code only depends on the contract, the eventual extraction of Forms into its own microservice is mechanical: swap the producer's in-process registration for an HTTP-client-proxy registration in the consumer, flip `ExposeIntegrationServices = true` on the new Forms service, configure `RemoteServices`, and `ReportAppService` does not change.

---

## TL;DR

- No, don't inject `IFormSubmissionAppService`. AppServices are the producer's caller-facing surface, not its inter-module contract.
- Define `[IntegrationService] IFormSubmissionIntegrationService` in **`Acme.Forms.Application.Contracts`**.
- Implement `FormSubmissionIntegrationService` in **`Acme.Forms.Application`** (no explicit controller, audit-off, no `[Authorize]` by default).
- In **`Acme.Reports.Application`**, inject the interface into `ReportAppService` and `[DependsOn(typeof(AcmeFormsApplicationContractsModule))]` — contracts only, never the full `AcmeFormsApplicationModule`.
- In your modular monolith DI resolves it in-process with no HTTP hop; in a microservice split the same contract becomes an HTTP-client proxy to `/integration-api/...` and the consumer code stays identical.
