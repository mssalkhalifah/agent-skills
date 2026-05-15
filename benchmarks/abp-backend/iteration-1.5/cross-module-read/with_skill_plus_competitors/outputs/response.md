# Cross-module reads in an ABP Layered modular monolith — `Reports` reading from `Forms`

Short answer: **no, don't inject `IFormSubmissionAppService` into `ReportAppService`.** That's the right instinct to avoid — it couples two modules through their public, client-facing API surface and makes the Forms module non-extractable later. The right pattern in ABP 10.3 is an **Integration Service**: a separate, narrowly-scoped `[IntegrationService]` interface in the producing module's `*.Application.Contracts` project, implemented in the producing module's `*.Application` project, and consumed via DI from the Reports module exactly like a normal service.

Below: why direct injection is wrong, the integration-service pattern in full, and the in-process-vs-microservice transport question (which is the single biggest reason this pattern exists).

---

## Why not just inject `IFormSubmissionAppService`?

`IFormSubmissionAppService` is the **public, client-facing contract** of the Forms module. It exists to be exposed to the outside world:

- It's auto-exposed at `/api/app/form-submission` by ABP's auto-API controllers (or by an explicit controller).
- It's audit-logged by default — every call shows up in your audit logs as an end-user-style action.
- It's authorized for end-user roles (`[Authorize]`, permission policies for "Forms.Submission.View"), which means a system-to-system call from `ReportAppService` running in a request context where the current user lacks `Forms.Submission.View` will **401**, even though the call is a trusted internal one.
- It returns DTOs shaped for the UI and the public client proxy — they're stable for *external* consumers, so any change you'd want to make for Reports' sake breaks the public contract too.
- It mixes input validation, mapping, audit, and authorization concerns that you don't want on a service-to-service path.
- Most critically: when you inject it cross-module, the consuming module must `[DependsOn]` the producer's `*.Application` module (not just the contracts), which **collapses the module boundary**. You can no longer extract Forms into its own microservice without rewriting Reports.

Rule 5 of the ABP framework guardrails is explicit about this: cross-module reads go through Integration Services, not through public application services and not through a host-level shared type.

(For completeness: cross-*aggregate* reads inside the **same** module are different — those go through a `*Manager` with `IReadOnlyRepository<TOther, TKey>`. Integration Services are specifically for crossing a *module* boundary.)

---

## The right pattern: ABP Integration Services

An ABP Integration Service is an `IApplicationService` marked with `[IntegrationService]`. The attribute changes its disposition in three important ways:

| Default for normal `IApplicationService` | Default for `[IntegrationService]` |
|---|---|
| Exposed at `/api/...` automatically | **Hidden from `/api`**; routes under `/integration-api/...` only when `ExposeIntegrationServices = true` is set on the host |
| Audit logging on by default | **Audit logging off** by default (`AbpAuditingOptions.IsEnabledForIntegrationService = false`) |
| `[Authorize]`-style policies typically applied | **No `[Authorize]` by default** — designed for trusted in-cluster service-to-service calls |
| Listed in Swagger | **Hidden** from Swagger / API Explorer |

That's the whole point: a separate, narrow contract for *internal* consumers, not exposed to external clients, not paying the audit/authz overhead of every call, and explicitly *not* the public API surface.

### Project layout

In a Layered modular monolith with two ABP modules `Acme.Forms` and `Acme.Reports`:

```
src/
  Acme.Forms.Domain.Shared/
  Acme.Forms.Domain/
  Acme.Forms.Application.Contracts/      <-- IFormSubmissionIntegrationService lives here
    IntegrationServices/
      IFormSubmissionIntegrationService.cs
  Acme.Forms.Application/                 <-- FormSubmissionIntegrationService impl lives here
    IntegrationServices/
      FormSubmissionIntegrationService.cs
  Acme.Forms.EntityFrameworkCore/
  Acme.Forms.HttpApi/
  Acme.Forms.HttpApi.Client/

  Acme.Reports.Domain.Shared/
  Acme.Reports.Domain/
  Acme.Reports.Application.Contracts/
  Acme.Reports.Application/               <-- ReportAppService consumes the integration service
    Services/
      Reports/
        ReportAppService.cs
        IReportAppService.cs
```

### 1. Define the contract in the producer's `*.Application.Contracts`

```csharp
// Acme.Forms.Application.Contracts/IntegrationServices/IFormSubmissionIntegrationService.cs
using System;
using System.Threading.Tasks;
using Volo.Abp.Application.Services;

namespace Acme.Forms.IntegrationServices;

[IntegrationService]
public interface IFormSubmissionIntegrationService : IApplicationService
{
    Task<int> GetSubmissionCountForOwnerAsync(Guid ownerId);
}
```

Why `*.Application.Contracts`:

- This is the project ABP designates for service interfaces and DTOs — same convention as `IFormSubmissionAppService`.
- The Reports module's `*.Application` project will `[DependsOn]` `AcmeFormsApplicationContractsModule`, **not** `AcmeFormsApplicationModule`. The consumer never references the implementation assembly. That's what keeps the boundary intact.
- When (if) Forms is later extracted into a microservice, the contracts package is the one that gets reused as the client SDK — `Reports` keeps depending on the same package, only the registration on the consumer side changes (more on that below).

The `[IntegrationService]` attribute goes on the **interface**; the implementation inherits it.

### 2. Implement in the producer's `*.Application` project

```csharp
// Acme.Forms.Application/IntegrationServices/FormSubmissionIntegrationService.cs
using System;
using System.Threading.Tasks;
using Acme.Forms.FormSubmissions;          // domain namespace
using Volo.Abp.Application.Services;
using Volo.Abp.Domain.Repositories;

namespace Acme.Forms.IntegrationServices;

public sealed class FormSubmissionIntegrationService
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

A few things worth pointing out:

- It inherits `ApplicationService` (gets `CurrentUser`, `ObjectMapper`, `GuidGenerator`, `Logger`) — same base class as a regular app service.
- It uses `IReadOnlyRepository<FormSubmission, Guid>` because this is a read. EF Core will run a no-tracking query — faster, signals intent. (This matches Rule 2 / Rule 3 of the framework guardrails — application services don't expose `IQueryable`, they call materialised repository methods like `CountAsync`.)
- I marked the class `sealed` — it's not an extension point. This communicates intent and gives the JIT a chance to devirtualize.
- Don't put a hand-written `Controller` next to it. **Rule 4 (explicit controllers) does not apply to integration services** — ABP wires the HTTP exposure for them implicitly through the integration-services pipeline. In a modular monolith, no HTTP exposure happens at all anyway.

### 3. Consume it from `ReportAppService` — exactly like any other dependency

```csharp
// Acme.Reports.Application/Services/Reports/ReportAppService.cs
using System;
using System.Threading.Tasks;
using Acme.Forms.IntegrationServices;        // from AcmeFormsApplicationContractsModule
using Acme.Reports.Reports;
using Volo.Abp.Application.Services;
using Volo.Abp.Domain.Repositories;

namespace Acme.Reports.Reports;

public sealed class ReportAppService : ApplicationService, IReportAppService
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

        // ... assemble & persist the report ...

        return ObjectMapper.Map<Report, ReportDto>(/* ... */);
    }
}
```

### 4. Wire the module dependency the right way

```csharp
// Acme.Reports.Application/AcmeReportsApplicationModule.cs
using Acme.Forms;                            // contracts module namespace
using Acme.Reports;
using Volo.Abp.Modularity;

namespace Acme.Reports;

[DependsOn(
    typeof(AcmeFormsApplicationContractsModule),     // <-- contracts only
    typeof(AcmeReportsDomainModule),
    typeof(AcmeReportsApplicationContractsModule)
)]
public class AcmeReportsApplicationModule : AbpModule { }
```

That's the load-bearing line: depend on the **`*.ApplicationContracts`** module, never on `AcmeFormsApplicationModule`. The Forms application module itself is registered into the host once (via the host's own `[DependsOn]`), which is what brings the implementation into the DI container; Reports only needs to know about the *contract*.

---

## In-process vs cross-process — same contract, different transport

This is the part that makes Integration Services worth bothering with even when both modules currently live in the same host:

**Today (Layered modular monolith — both modules in the same host):**

- Both `AcmeFormsApplicationModule` and `AcmeReportsApplicationModule` are loaded into the same `WebApplication` via `[DependsOn]` on the host module.
- DI resolves `IFormSubmissionIntegrationService` to the in-process `FormSubmissionIntegrationService` class registered by Forms's application module.
- The call from `ReportAppService` is a **plain method call** through DI. No HTTP. No serialization. No `/integration-api/...` route. `ExposeIntegrationServices` stays at its default `false` because no HTTP exposure is needed.
- It's just as fast as injecting `IFormSubmissionAppService` would have been — but without the audit, authz, and module-coupling problems described above.

**Tomorrow (Forms extracted into its own microservice):**

The same `IFormSubmissionIntegrationService` interface — the same contracts package — gets reused. What changes:

- On the **Forms host** (now a standalone service): set `Configure<AbpAspNetCoreMvcOptions>(o => o.ExposeIntegrationServices = true)`, register a `ConventionalControllers.Create(...)` with `ApplicationServiceTypes.IntegrationServices`, and make sure the API gateway (YARP/Ocelot) blocks external traffic to `/integration-api/*` so only sibling services in the private network can reach it. The interface auto-routes under `/integration-api/form-submission/get-submission-count-for-owner?ownerId=...`.
- On the **Reports host**: swap the in-process registration for `AddHttpClientProxies(typeof(AcmeFormsApplicationContractsModule).Assembly, "Forms")` (dynamic) or `AddStaticHttpClientProxies(...)` (static, after `abp generate-proxy -t csharp`). Add `"RemoteServices": { "Forms": { "BaseUrl": "https://forms-svc/" } }` to `appsettings.json`.
- **`ReportAppService` itself does not change.** It still injects `IFormSubmissionIntegrationService` and calls `GetSubmissionCountForOwnerAsync(ownerId)`. ABP's HTTP client proxy implements the same interface; DI resolves to the proxy instead of the in-process class.

Same contract, same dependency-injection shape, transport differs by deployment. That's the payoff for using `[IntegrationService]` from day one even when both modules currently live in the same host.

---

## A few footnotes that often trip people up

- **Don't add `[Authorize]` to the integration service.** It runs without an end-user identity in service-to-service mode and will 401. If you genuinely need service-to-service auth later (microservice split), use OpenIddict client-credentials with a dedicated client and propagate the bearer token via `AbpHttpClientBuilderOptions.ProxyClientBuildActions` — don't decorate the integration service with `[Authorize]`.
- **Don't hand-write a controller** that implements `IFormSubmissionIntegrationService`. Rule 4 (explicit controllers for app services) does not apply here — ABP's integration-services pipeline handles HTTP exposure when you flip `ExposeIntegrationServices` on. Hand-writing one will collide with the convention.
- **Don't put `IFormSubmissionIntegrationService` in `*.Domain`** — it's an application-layer contract (it's an `IApplicationService`). `*.Application.Contracts` is correct.
- **Multi-tenancy auto-flows.** If your monolith is multi-tenant, the `CurrentTenant` ambient propagates into the integration service as a normal in-process call. After microservice extraction, ABP's HTTP client proxy passes the tenant header automatically — you don't need to do anything extra in `ReportAppService` either way.
- **One method per concern.** Keep the integration-service surface as narrow as Reports actually needs (`GetSubmissionCountForOwnerAsync`). Don't dump every `FormSubmission` query in there "in case Reports needs it later." Add new methods (extend-only) when new consumers actually need them.

So: a separate `[IntegrationService]` interface in `Acme.Forms.Application.Contracts`, an `ApplicationService` implementation in `Acme.Forms.Application`, consume it via constructor DI in `ReportAppService`, and have `AcmeReportsApplicationModule` `[DependsOn]` the Forms **contracts** module. That gives you the right boundary today and a free transport upgrade if/when Forms ever splits out.
