No — do not inject `IFormSubmissionAppService` from `Acme.Forms` directly into `ReportAppService` in `Acme.Reports`. `IFormSubmissionAppService` is a public, audited, `[Authorize]`-gated, end-user-facing application service. Reusing it as the cross-module read channel couples your report to the consumer-facing permission set, audit log, and DTO shape of the Forms module, and forces `Acme.Reports.Application` to `[DependsOn]` the Forms `Application` module (not just its contracts) — which dumps every Forms domain service, repository, mapper, and internal collaborator into the same DI container and silently dissolves the module boundary.

The right pattern is an **ABP Integration Service**.

## The pattern — Rule 5 (cross-module reads)

Integration services are dedicated cross-module read APIs marked with `[IntegrationService]`. They are hidden from the public `/api` surface, audit-off by default, and not wrapped in `[Authorize]` automatically — they are the producer module's *internal* contract for other modules to read its data.

### 1. Define the contract in `Acme.Forms.Application.Contracts`

Placement matters: the interface must live in `*.Application.Contracts`, **not** `*.Application`. The contracts project is the only thing the consumer is allowed to depend on.

```csharp
// Acme.Forms.Application.Contracts/IntegrationServices/IFormSubmissionIntegrationService.cs
using System;
using System.Threading.Tasks;
using Volo.Abp.Application.Services;

namespace Acme.Forms.IntegrationServices;

[IntegrationService]   // hidden from /api, audit-off, no [Authorize] by default
public interface IFormSubmissionIntegrationService : IApplicationService
{
    Task<int> GetSubmissionCountForOwnerAsync(Guid ownerId);
}
```

### 2. Implement it in `Acme.Forms.Application`

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
    private readonly IReadOnlyRepository<FormSubmission, Guid> _submissions;

    public FormSubmissionIntegrationService(
        IReadOnlyRepository<FormSubmission, Guid> submissions)
    {
        _submissions = submissions;
    }

    public Task<int> GetSubmissionCountForOwnerAsync(Guid ownerId)
        => _submissions.CountAsync(s => s.OwnerId == ownerId);
}
```

A few things to notice:

- `IReadOnlyRepository<,>` — Rule 2 + the narrowest-repository guidance. This is a count query, no tracking needed.
- The existing `FormSubmissionAppService` query is **not** reused. Even though it would compile, the integration service is a separate contract with a separate audit/authorization posture; have it run its own count against the repository.
- The implementation lives in `Acme.Forms.Application`, but only the interface (in `*.Application.Contracts`) is what `Acme.Reports` is allowed to see.

### 3. Wire the consumer — contracts-only `[DependsOn]`

This is the load-bearing line. The Reports `Application` module must declare a dependency on Forms' **`Application.Contracts`** module, never the full `Application` module.

```csharp
// Acme.Reports.Application/AcmeReportsApplicationModule.cs
using Acme.Forms;                              // AcmeFormsApplicationContractsModule
using Acme.Reports;                            // AcmeReportsDomainModule
using Volo.Abp.Modularity;

namespace Acme.Reports;

[DependsOn(
    typeof(AcmeFormsApplicationContractsModule),   // contracts only — RIGHT
    // typeof(AcmeFormsApplicationModule),         // full app module — WRONG (leaks producer internals)
    typeof(AcmeReportsDomainModule)
)]
public class AcmeReportsApplicationModule : AbpModule { }
```

If you `[DependsOn]` the producer's full `*.Application`, every Forms domain service, repository, integration handler, mapper, and current-user helper becomes injectable inside `Acme.Reports` — your module boundary still exists in the folder tree but is gone in the DI container. **Contracts-only `[DependsOn]` is what makes the seam real.** A bare `<ProjectReference>` in the csproj is not enough on its own; ABP's module system only loads the wiring that `[DependsOn]` declares.

### 4. Consume in `ReportAppService`

```csharp
// Acme.Reports.Application/Reports/ReportAppService.cs
public class ReportAppService : ApplicationService, IReportAppService
{
    private readonly IFormSubmissionIntegrationService _formSubmissions;
    private readonly IRepository<Report, Guid>        _reportRepository;

    public ReportAppService(
        IFormSubmissionIntegrationService formSubmissions,
        IRepository<Report, Guid> reportRepository)
    {
        _formSubmissions  = formSubmissions;
        _reportRepository = reportRepository;
    }

    public async Task<ReportDto> BuildAsync(Guid ownerId)
    {
        var submissionCount = await _formSubmissions
            .GetSubmissionCountForOwnerAsync(ownerId);
        // ... assemble the report DTO ...
    }
}
```

## Modular monolith vs microservice — same contract, different transport

You are on a Layered **modular monolith**: both modules live in the same host process, share the same DI container, and ABP resolves `IFormSubmissionIntegrationService` to its in-process concrete `FormSubmissionIntegrationService`. The call is a direct C# method invocation — no HTTP, no serialization, no gateway hop.

If the same solution were promoted to the **Microservice** template later, the contract does not change. ABP exposes `[IntegrationService]` interfaces on the producing service under `/integration-api/...` (separate from the public `/api` surface, behind the internal gateway), and the consuming service injects a generated proxy that implements the same `IFormSubmissionIntegrationService`. Same interface, same `[DependsOn(AcmeFormsApplicationContractsModule)]` shape on the consumer, same call site — only the transport flips from in-process DI to HTTP-over-gateway. That portability is exactly why Rule 5 forbids the direct-`IApplicationService` shortcut: it would only "work" in-process and would have to be torn out the moment Forms moved into its own service.

## Why not just inject `IFormSubmissionAppService`?

- **Wrong authorization posture.** `IFormSubmissionAppService` carries `[Authorize(FormPermissions.*)]` aimed at the end user who originally submitted the form. A background report builder running under a service identity (or no `CurrentUser` at all) will trip those permissions or rely on a `CurrentTenant.Change` / `AuthorizationOptions` workaround. Integration services are intentionally permission-free at the framework layer; if you need a check, add an explicit policy that reflects *module-to-module* trust, not end-user permissions.
- **Wrong audit posture.** The public app service writes audit-log entries shaped for end-user activity. A cross-module read should not pollute that log.
- **DTO coupling.** `FormSubmissionDto` is shaped for the Forms UI; tying `Acme.Reports` to it means every UI-driven change to the DTO ripples into the reports module.
- **Forces `[DependsOn]` on the full Forms `Application` module.** The whole point of the contracts/application split is to ship a tiny contract surface; injecting the public app service negates the split.

Keep `IFormSubmissionAppService` for the Forms UI and HTTP clients. Use `IFormSubmissionIntegrationService` — defined in `Acme.Forms.Application.Contracts`, implemented in `Acme.Forms.Application`, wired into the consumer via `[DependsOn(typeof(AcmeFormsApplicationContractsModule))]` — for `Acme.Reports`.
