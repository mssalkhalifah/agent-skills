Short answer: no, don't inject `IFormSubmissionAppService` into `ReportAppService`. The right pattern for a cross-module read in ABP is an **Integration Service**.

This is exactly what Rule 5 (cross-module reads via ABP Integration Services) is about. Let me walk through why direct injection is wrong, then show the correct shape end-to-end.

## Why not just inject `IFormSubmissionAppService`?

`IFormSubmissionAppService` is the producer module's *public* application-service interface. It lives in `Acme.Forms.Application.Contracts`, and to make it injectable from a different module you'd have to do one of two things — both bad:

1. **Add `[DependsOn(typeof(AcmeFormsApplicationModule))]`** on `AcmeReportsApplicationModule`. That pulls in the entire Forms application module — its domain services, repositories, mappers, internal collaborators — into the same DI container as Reports. Every "private" Forms type becomes injectable from Reports, and the module boundary is gone in practice even though the folders still look separate.
2. **Call it as a regular HTTP `IApplicationService`.** That breaks isolation in a different way: the public app service is the surface for *external* clients. It carries `[Authorize(...)]` permission checks designed for end users, returns full DTOs shaped for UI, is included in the public Swagger / OpenAPI doc, and is audit-logged. None of that is what you want for an in-process inter-module read.

There's also no host-level "shared types" layer in ABP that you can drop the call into — the framework's answer to this scenario is integration services.

## The right pattern — Integration Services

### 1. Define the contract in the producer's `*.Application.Contracts`

```csharp
// Acme.Forms.Application.Contracts/IFormSubmissionIntegrationService.cs
using System;
using System.Threading.Tasks;
using Volo.Abp.Application.Services;

namespace Acme.Forms;

[IntegrationService]
public interface IFormSubmissionIntegrationService : IApplicationService
{
    Task<int> GetSubmissionCountForOwnerAsync(Guid ownerId);
}
```

The `[IntegrationService]` attribute is load-bearing. It tells ABP that:

- The endpoint is **hidden from the public API surface** (excluded from the regular `/api/...` Auto API Controllers and from the public Swagger doc).
- **Auditing is off** by default — these are inter-module calls, not user actions.
- **`[Authorize]` is not applied** by default — the caller is another trusted module, not an end user. If you genuinely need authorization, opt in explicitly.

Put this interface in `Acme.Forms.Application.Contracts`, alongside your normal `IFormSubmissionAppService`. That's the project Reports will be allowed to depend on.

### 2. Implement it in the producer's `*.Application` project

```csharp
// Acme.Forms.Application/IntegrationServices/FormSubmissionIntegrationService.cs
using System;
using System.Threading.Tasks;
using Volo.Abp.Application.Services;
using Volo.Abp.Domain.Repositories;

namespace Acme.Forms.IntegrationServices;

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

Note: integration services do **not** need a hand-written explicit controller. Rule 4 (the explicit-controller pattern) doesn't apply here — ABP wires the HTTP exposure for `[IntegrationService]` interfaces implicitly through the integration-services pipeline. Don't try to author a `Controller` that implements an `[IntegrationService]` interface.

### 3. Consume it from `ReportAppService`

```csharp
// Acme.Reports.Application/Services/Reports/ReportAppService.cs
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

        // ... assemble report ...
    }
}
```

### 4. The load-bearing piece — module dependency on Contracts only

This is the detail that actually preserves the module boundary:

```csharp
// Acme.Reports.Application/AcmeReportsApplicationModule.cs
[DependsOn(
    typeof(AcmeFormsApplicationContractsModule),   // contracts only — RIGHT
    // typeof(AcmeFormsApplicationModule),         // full app module — WRONG
    typeof(AcmeReportsDomainModule)
)]
public class AcmeReportsApplicationModule : AbpModule { }
```

The Reports application module `[DependsOn]`s **`AcmeFormsApplicationContractsModule` only** — never `AcmeFormsApplicationModule`. If you depend on the full `*.Application`, you load all of Forms's domain services, repositories, and internal collaborators into the same DI container and the boundary collapses regardless of which folders the source files live in. Contracts-only `[DependsOn]` is what makes the seam real. The corresponding `<ProjectReference>` in your `.csproj` should likewise point at `Acme.Forms.Application.Contracts` only.

## In-process now vs cross-process later — same contract, different transport

You said you're on a **Layered modular monolith** today, but ABP keeps the door open to splitting into microservices later, and the integration-services pattern is what makes that migration cheap:

- **Modular monolith (your case):** both modules live in the same host process. ABP's DI resolves `IFormSubmissionIntegrationService` directly to the in-process `FormSubmissionIntegrationService` — no HTTP hop, no serialization, just a method call.
- **Microservice deployment (later, if you split):** the same `[IntegrationService]` interface is exposed over `/integration-api/...` behind the API gateway, and the consumer side gets a generated proxy client (via `abp generate-proxy`) registered against the same `IFormSubmissionIntegrationService` interface.

Your `ReportAppService` constructor signature does not change. The contract is the same; only the transport differs based on deployment topology. That's the payoff for using `[IntegrationService]` instead of either ad-hoc cross-module method calls or a full HTTP-from-the-start design.

## A note on Rule 5's scope

Rule 5 is for **cross-module** reads. If you ever need a *cross-aggregate* read **inside the same module** (say, two aggregates inside Reports), the right tool is a `*Manager` (domain service) with `IReadOnlyRepository<TOther, TKey>` — not an integration service. Integration services are specifically the seam between modules.

## TL;DR

- Don't inject `IFormSubmissionAppService` from Reports — it leaks the producer's public API and forces an over-broad module dependency.
- Define `IFormSubmissionIntegrationService` with `[IntegrationService]` in `Acme.Forms.Application.Contracts`; implement it in `Acme.Forms.Application`.
- `[DependsOn(typeof(AcmeFormsApplicationContractsModule))]` on `AcmeReportsApplicationModule` — contracts only, never the full Application module.
- In-process today, HTTP/generated proxy if you split later — same interface, different transport.
