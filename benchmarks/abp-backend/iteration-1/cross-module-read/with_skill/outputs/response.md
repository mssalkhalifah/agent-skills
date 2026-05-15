**Short answer: no — don't inject `IFormSubmissionAppService` into `ReportAppService`.** That couples one module's internal API to another and turns a normal client-facing app service into an unintended cross-module contract. The right pattern in ABP is an **Integration Service** (Rule 5 in the ABP backend guidance, and ABP's own [Integration Services docs](https://abp.io/docs/10.3/framework/api-development/integration-services)).

Here's why, and exactly how to wire it.

## Why not the application service directly

`IFormSubmissionAppService` exists to serve Forms' own clients (the UI, external API consumers). It tends to:

- be `[Authorize]`'d for end-user permissions (a `Reports` background context likely doesn't have those),
- be audit-logged on every call (you don't want chatty cross-module reads polluting Forms' audit log),
- be exposed at `/api/app/form-submission/...` for external consumption,
- evolve at the pace of the Forms UI's needs — its DTOs and method signatures change to suit the UI, and now your Reports module silently rides those changes.

It also blurs the module boundary: you've effectively made `Acme.Reports` a downstream client of `Acme.Forms`'s end-user API surface, which is the wrong contract. ABP gives you a separate, purpose-built mechanism for "module B needs to read data owned by module A."

## The right pattern — Integration Service

Define the cross-module contract in **Forms' `*.Contracts` project**, mark it `[IntegrationService]`, implement it in Forms, consume it from Reports via DI. Same shape you'd use if Forms and Reports later split into microservices — the contract is portable.

```csharp
// Acme.Forms.Application.Contracts/IntegrationServices/IFormSubmissionIntegrationService.cs
using Volo.Abp.Application.Services;

namespace Acme.Forms;

[IntegrationService]   // hidden from /api, audit-off by default, no [Authorize] by default
public interface IFormSubmissionIntegrationService : IApplicationService
{
    Task<int> GetSubmissionCountForOwnerAsync(Guid ownerId);
}
```

```csharp
// Acme.Forms.Application/IntegrationServices/FormSubmissionIntegrationService.cs
public class FormSubmissionIntegrationService
    : ApplicationService, IFormSubmissionIntegrationService
{
    private readonly IRepository<FormSubmission, Guid> _submissionRepository;

    public FormSubmissionIntegrationService(IRepository<FormSubmission, Guid> submissionRepository)
        => _submissionRepository = submissionRepository;

    public Task<int> GetSubmissionCountForOwnerAsync(Guid ownerId)
        => _submissionRepository.CountAsync(s => s.OwnerId == ownerId);
}
```

```csharp
// Acme.Reports.Application/Reports/ReportAppService.cs
public class ReportAppService : ApplicationService, IReportAppService
{
    private readonly IFormSubmissionIntegrationService _formSubmissions;   // <- cross-module read
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

To make `Acme.Reports` see the type, the consuming side just needs a project reference (or NuGet dep) on **`Acme.Forms.Application.Contracts`** — that's the whole point of the `Contracts` project. Don't reference `Acme.Forms.Application` from Reports; the contract is enough.

## What's actually different about an Integration Service

Compared to a vanilla `IApplicationService`, the `[IntegrationService]` attribute (`Volo.Abp.Application.Services.IntegrationServiceAttribute`) gives you:

| Behavior | Regular app service | Integration service |
|---|---|---|
| HTTP route | `/api/app/...` | `/integration-api/...` (only when explicitly exposed) |
| Visible in default `/api` host | Yes | **No** — hidden unless `AbpAspNetCoreMvcOptions.ExposeIntegrationServices = true` |
| Audit logging | On | **Off** by default (`AbpAuditingOptions.IsEnabledForIntegrationService`) |
| `[Authorize]` expected | Usually | Typically not — designed for trusted intra-system callers |
| Swagger/UI surface | Public | Hidden by default |

That set of defaults is exactly what you want for "another module reads my data": quiet, internal, not part of the customer-facing API.

## In-process now, microservice later — same contract

Since you're on a Layered modular monolith, both modules live in the same host. ABP's DI just resolves `IFormSubmissionIntegrationService` to the in-process `FormSubmissionIntegrationService` — **no HTTP hop, no serialization**. You get plain method calls within the same `IUnitOfWork`.

If you later split Forms and Reports into separate microservices, the `[IntegrationService]` is exposed over `/integration-api/...` behind the API gateway, and Reports consumes it via a generated proxy client. Same interface, same DI shape; only the transport changes. That's the payoff for using this pattern from day one.

## A few related rules to keep in mind

- **No explicit controller for the integration service.** Rule 4's "explicit controller per app service" pattern doesn't apply here — ABP wires integration-service HTTP exposure implicitly through the integration-services pipeline. Don't hand-write a `FormSubmissionIntegrationController`.
- **This is a *cross-module* read.** Rule 5 only applies across module boundaries (`Acme.Forms` ↔ `Acme.Reports`). For cross-aggregate reads *inside* the same module, use a `*Manager` (domain service) injecting `IReadOnlyRepository<TOther, TKey>` per Rule 3. Don't reach for an integration service for in-module reads.
- **Don't expose `IQueryable`.** Whatever the integration method returns, return materialised data — `int`, a DTO, `List<T>`, `PagedResultDto<T>`. Same Rule 2 that applies to your normal app services.
- **Where it lives.** Contract goes in `Acme.Forms.Application.Contracts` (so consumers can reference just the contract assembly). Implementation goes in `Acme.Forms.Application` next to the regular app services — convention is an `IntegrationServices/` folder, but ABP doesn't require it.

## TL;DR

Don't inject `IFormSubmissionAppService` into `ReportAppService`. Define `IFormSubmissionIntegrationService` in `Acme.Forms.Application.Contracts` with `[IntegrationService]`, implement it in `Acme.Forms.Application`, and inject it into `ReportAppService`. In-process today, microservice-ready tomorrow, and you keep Forms' end-user API surface decoupled from your reporting needs.
