# Cross-module read: `Acme.Reports` reading from `Acme.Forms`

**Short answer: no — don't inject `IFormSubmissionAppService` into `ReportAppService`.** That's a cross-module read, and the right ABP pattern is an **Integration Service** defined in the producing module's `*.Application.Contracts` project, with the consumer module `[DependsOn]`-ing **contracts only**.

This is Rule 5 (Cross-module reads via ABP Integration Services) of the skill. Rule 5 explicitly applies here because Reports and Forms are separate ABP modules in your Layered modular monolith. (For cross-aggregate reads inside the *same* module, you'd use a `*Manager` with `IReadOnlyRepository<TOther, TKey>` instead — that's Rule 3, not what you need here.)

## Why not just inject `IFormSubmissionAppService`?

Two reasons the skill calls out:

1. **`IFormSubmissionAppService` is a public application service.** It lives in `Acme.Forms.Application.Contracts` and is part of the Forms module's public, audited, `[Authorize]`-gated API surface — designed for HTTP/UI clients, not for in-process consumption by sibling modules. Reusing it for internal reads drags permission checks, audit logs, and DTO shapes you didn't design for this consumer into your reporting path.
2. **More importantly — to make it injectable, you'd have to `[DependsOn(typeof(AcmeFormsApplicationModule))]` from Reports.** That's the boundary-breaking move. Per Rule 5: *"If you `[DependsOn]` the producer's full `*.Application`, the consumer loads the producer's domain services, repositories, and internal collaborators into the same DI container — every 'private' piece of the producer becomes injectable in the consumer, and the boundary is gone in practice even though the source files still live in separate folders."*

The `<ProjectReference>` in your `.csproj` is *not* enough on its own. **`[DependsOn]` is the load-bearing line.**

## The right pattern — Integration Service

### 1. Define the contract in `Acme.Forms.Application.Contracts`

```csharp
// Acme.Forms.Application.Contracts/IntegrationServices/IFormSubmissionIntegrationService.cs
using System;
using System.Threading.Tasks;
using Volo.Abp.Application.Services;

namespace Acme.Forms.IntegrationServices;

[IntegrationService]   // hidden from public /api, audit-off, no [Authorize] by default
public interface IFormSubmissionIntegrationService : IApplicationService
{
    Task<int> GetSubmissionCountForOwnerAsync(Guid ownerId);
}
```

`[IntegrationService]` is the marker that makes this a service-to-service contract: ABP keeps it off the public Swagger/`/api` surface and skips audit logging by default.

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
    private readonly IRepository<FormSubmission, Guid> _submissionRepository;

    public FormSubmissionIntegrationService(IRepository<FormSubmission, Guid> submissionRepository)
    {
        _submissionRepository = submissionRepository;
    }

    public Task<int> GetSubmissionCountForOwnerAsync(Guid ownerId)
        => _submissionRepository.CountAsync(s => s.OwnerId == ownerId);
}
```

If `FormSubmissionAppService` already has this exact query, just call the repository the same way here — don't have the integration service call the app service. The integration service is a peer of the app service, not a wrapper around it. (You can factor a shared `*Manager` if both need the same logic, but a one-liner `CountAsync` typically doesn't justify that.)

### 3. Wire the consumer with **contracts-only** `[DependsOn]`

This is the line that actually preserves the boundary:

```csharp
// Acme.Reports.Application/AcmeReportsApplicationModule.cs
using Acme.Forms;
using Volo.Abp.Modularity;

namespace Acme.Reports;

[DependsOn(
    typeof(AcmeFormsApplicationContractsModule),   // contracts only — RIGHT
    // typeof(AcmeFormsApplicationModule),         // full app module — WRONG (leaks producer internals)
    typeof(AcmeReportsDomainModule)
)]
public class AcmeReportsApplicationModule : AbpModule { }
```

Your `Acme.Reports.Application.csproj` carries a `<ProjectReference>` to `Acme.Forms.Application.Contracts.csproj` (and nothing else from Forms). The `[DependsOn]` line is what tells the ABP module system to wire it; ProjectReference alone is necessary but not sufficient.

### 4. Inject the integration service in `ReportAppService`

```csharp
// Acme.Reports.Application/Reports/ReportAppService.cs
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
        var submissionCount = await _formSubmissions.GetSubmissionCountForOwnerAsync(ownerId);
        // ... assemble report ...
    }
}
```

## Modular monolith vs microservice — same contract, different transport

This is worth being explicit about because the skill calls it out:

- **In your Layered modular monolith (your case):** Forms and Reports live in the same host process. ABP's DI container resolves `IFormSubmissionIntegrationService` directly to the in-process `FormSubmissionIntegrationService` instance — **no HTTP hop, no serialization, no gateway**. It's a normal C# method call dispatched through the DI container. Performance is identical to injecting any other service.
- **If you ever split Forms and Reports into separate microservices:** the *same* `[IntegrationService]` interface gets exposed over `/integration-api/...` behind the API gateway, and the consumer side picks it up via a generated proxy client (`abp generate-proxy -t csharp`). The interface, the `[DependsOn]` on `*.Application.Contracts`, and the `ReportAppService` constructor signature **don't change** — only the wiring at module-startup time picks a different implementation (the proxy instead of the local class). That's the payoff of doing this right today: the seam you draw now is also the seam you'd cut across a network later.

## Other gotchas from the skill that apply here

- **No explicit controller** for the integration service. Rule 4 (explicit controllers) does **not** apply to `[IntegrationService]`-marked interfaces — ABP wires their HTTP exposure implicitly through the integration-services pipeline. Don't hand-write a `Controller` that implements `IFormSubmissionIntegrationService`.
- **Don't put `[Authorize]` on the integration service by default.** Integration services are service-to-service and audit-off, `[Authorize]`-off by default. If a specific call genuinely needs a permission check, add it explicitly — but the common case is that the *consuming* app service (`ReportAppService`) already carries the user-facing `[Authorize(ReportPermissions.View)]` and the integration call runs under that authorized context.
- **Don't expose `IQueryable` across the seam.** The integration method returns `Task<int>` (materialised), not `IQueryable<FormSubmission>`. Same Rule 2 / Rule 4 prohibition you'd apply within a single module.
- **The current `FormSubmissionAppService` stays put.** You're not deleting it — it's still the right vehicle for HTTP/UI callers with full audit and authorization. The integration service is a *second* entry point into the same data, sized and shaped for in-process module-to-module reads.

## TL;DR checklist

1. Create `IFormSubmissionIntegrationService` in `Acme.Forms.Application.Contracts`, marked `[IntegrationService]`, inheriting `IApplicationService`.
2. Implement `FormSubmissionIntegrationService : ApplicationService, IFormSubmissionIntegrationService` in `Acme.Forms.Application` — talks to the repository directly.
3. In `Acme.Reports.Application.csproj`, add `<ProjectReference>` to `Acme.Forms.Application.Contracts.csproj` only.
4. In `AcmeReportsApplicationModule`, add `[DependsOn(typeof(AcmeFormsApplicationContractsModule))]` — **never** `AcmeFormsApplicationModule`.
5. Inject `IFormSubmissionIntegrationService` into `ReportAppService`. Done.
