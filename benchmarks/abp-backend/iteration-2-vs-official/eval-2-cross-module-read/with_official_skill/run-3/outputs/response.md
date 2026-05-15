# Cross-module read: `Acme.Reports` reading from `Acme.Forms`

## Short answer

**No — do not inject `IFormSubmissionAppService` into `ReportAppService`.** That is an explicit anti-pattern in ABP. Application services are "frontend" services for the UI: they have their own authorization rules, validation, audit logging, DTO shapes, and (via ABP's Auto API Controllers) are exposed as HTTP endpoints. They are not designed to be called by other modules.

The official ABP pattern for "one module needs to read something from another module synchronously" is an **Integration Service**.

Skills used: `abp-microservice` (Integration Services pattern), `abp-dependency-rules` ("Cross-module app service call -> Tight coupling -> Use events or domain"), `abp-application-layer` ("Calling other app services in the same module" anti-pattern; same rule applies cross-module — use a dedicated contract), `abp-module` (layered module structure and where contracts live), `abp-core` (module `[DependsOn]` wiring), `abp-infrastructure` (when to use distributed events instead).

---

## The pattern: Integration Service

An Integration Service is a small, narrow service intended specifically for inter-module / inter-service consumption. It is decorated with `[IntegrationService]`, lives in the **provider module**, and its **interface + DTOs live in the provider's `Application.Contracts` project** (the only project a consumer is allowed to reference).

In a **modular monolith (your case, ABP 10.3 Layered)** the transport is plain in-process DI — the consumer resolves `IFormSubmissionIntegrationService` from the container and calls the implementation directly. Nothing else changes.

In a **microservice** solution the same interface + DTOs live in the provider's `*.Contracts` package, `ExposeIntegrationServices = true` is set, the consumer adds a NuGet reference to the contracts, runs `abp generate-proxy`, registers `AddStaticHttpClientProxies(...)`, and configures `RemoteServices:<Service>:BaseUrl` in `appsettings.json`. The consumer code is identical — same interface, same call site — only the transport differs.

That is the whole modular-monolith vs. microservice difference: **same contract, swap the wiring**. Designing this way means a future "promote Forms to its own microservice" change is a wiring change in `Reports`, not a code change.

---

## When to choose Integration Service vs. distributed event vs. cache

| Need | Pattern |
|------|---------|
| Reports needs a fresh value right now to render a row ("how many submissions does owner X have?") | **Integration Service** (synchronous read) |
| Forms wants to notify everyone that a submission was created/deleted | **Distributed event** (`SubmissionCreatedEto`) |
| Reports hits the same owner counts repeatedly and tolerates staleness | Integration Service + `IDistributedCache` / Entity Cache, invalidated by the event above |

Your case is a query that must reflect Forms' source of truth, so Integration Service is correct.

---

## Where each file lives (Layered modular monolith)

| Artifact | Project |
|----------|---------|
| `IFormSubmissionIntegrationService` (interface) | `Acme.Forms.Application.Contracts` |
| `OwnerSubmissionCountDto` (DTO) | `Acme.Forms.Application.Contracts` |
| `FormSubmissionIntegrationService` (implementation) | `Acme.Forms.Application` |
| `[DependsOn(typeof(AcmeFormsApplicationContractsModule))]` on the Reports module | `Acme.Reports.Application` (and only the Application project — Reports must *not* depend on `Acme.Forms.Application`, `.Domain`, or `.EntityFrameworkCore`) |

The critical rule from `abp-dependency-rules`: **the consumer references only `*.Application.Contracts`** of the provider, never `Application`, never `Domain`, never `EntityFrameworkCore`. This keeps the modules decoupled and makes the future "extract to microservice" trivial.

Also from `abp-application-layer`: **stop using the cross-module query inside `FormSubmissionAppService` from Reports** (or have it delegate to the integration service). The app service is for the Forms UI; the integration service is for other modules. Don't have two services owning the same query path.

---

## Code

### 1. Contract — `Acme.Forms.Application.Contracts`

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

Notes:
- `[IntegrationService]` is the ABP marker that tells the framework "this is for inter-module calls, not for the UI." In a microservice setup, combined with `ExposeIntegrationServices = true`, it also controls HTTP exposure separately from regular app services.
- Inherits `IApplicationService` so it is auto-registered, gets unit-of-work handling, and (in a microservice) can be proxied via `generate-proxy`.
- Keep the surface **narrow**: one method, one DTO. Do not return `FormSubmissionDto` from the app service — that DTO belongs to Forms' UI and will drift.

### 2. Implementation — `Acme.Forms.Application`

```csharp
// Acme.Forms.Application/IntegrationServices/FormSubmissionIntegrationService.cs
using System;
using System.Threading.Tasks;
using Acme.Forms.FormSubmissions;       // IFormSubmissionRepository (Domain)
using Acme.Forms.IntegrationServices;   // contract namespace
using Volo.Abp.Application.Services;

namespace Acme.Forms.IntegrationServices;

[IntegrationService]
public class FormSubmissionIntegrationService
    : ApplicationService, IFormSubmissionIntegrationService
{
    private readonly IFormSubmissionRepository _formSubmissionRepository;

    public FormSubmissionIntegrationService(
        IFormSubmissionRepository formSubmissionRepository)
    {
        _formSubmissionRepository = formSubmissionRepository;
    }

    public virtual async Task<OwnerSubmissionCountDto> GetSubmissionCountByOwnerAsync(
        Guid ownerId)
    {
        var count = await _formSubmissionRepository.CountByOwnerAsync(ownerId);

        return new OwnerSubmissionCountDto
        {
            OwnerId = ownerId,
            SubmissionCount = count
        };
    }
}
```

Notes:
- Goes through the **repository**, not through `FormSubmissionAppService`. App-service-calls-app-service is the anti-pattern we're avoiding (see `abp-application-layer`).
- `virtual` per `abp-module` extensibility guidance, in case Forms is later packaged as a reusable module.
- Inherits `ApplicationService`, so `CurrentTenant`, `CurrentUser`, `Clock`, etc. flow correctly — useful because the integration service runs inside the caller's unit of work and tenant scope automatically in the monolith.

### 3. Consumer wiring — `Acme.Reports.Application`

```csharp
// Acme.Reports.Application/AcmeReportsApplicationModule.cs
using Acme.Forms;                                  // AcmeFormsApplicationContractsModule
using Volo.Abp.Application;
using Volo.Abp.AutoMapper;
using Volo.Abp.Modularity;

namespace Acme.Reports;

[DependsOn(
    typeof(AcmeReportsDomainModule),
    typeof(AcmeReportsApplicationContractsModule),
    typeof(AbpDddApplicationModule),
    typeof(AbpAutoMapperModule),
    // Cross-module: depend ONLY on Forms' Application.Contracts.
    // Do NOT add AcmeFormsApplicationModule, AcmeFormsDomainModule,
    // or AcmeFormsEntityFrameworkCoreModule here.
    typeof(AcmeFormsApplicationContractsModule)
)]
public class AcmeReportsApplicationModule : AbpModule
{
    // No manual DI registration needed in the monolith: the integration service
    // is auto-registered by ABP because it inherits ApplicationService and the
    // Forms Application module is already loaded by the host. The consumer just
    // resolves IFormSubmissionIntegrationService from DI.
    //
    // In a microservice split, you would instead register an HTTP client proxy
    // here against the same interface:
    //
    //   context.Services.AddStaticHttpClientProxies(
    //       typeof(AcmeFormsApplicationContractsModule).Assembly,
    //       "AcmeForms");
    //
    // ...and configure RemoteServices:AcmeForms:BaseUrl in appsettings.json.
    // The Reports consumer code below does not change.
}
```

Also make sure the host (`Acme.HostWeb` / `Acme.Web` / whichever bootstraps everything) already depends on `AcmeFormsApplicationModule` so the implementation is loaded into the container. The Reports Application module itself must not.

### 4. Consumer usage — `Acme.Reports.Application`

```csharp
// Acme.Reports.Application/Reports/ReportAppService.cs
using System;
using System.Threading.Tasks;
using Acme.Forms.IntegrationServices;
using Acme.Reports.Reports.Dtos;
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
        var counts = await _formSubmissions.GetSubmissionCountByOwnerAsync(ownerId);

        return new OwnerReportDto
        {
            OwnerId = ownerId,
            FormSubmissionCount = counts.SubmissionCount
            // ...other report fields aggregated from Reports' own data
        };
    }
}
```

---

## Monolith vs. microservice — what actually differs

Same interface, same DTOs, same call site in `ReportAppService`. Only the consumer module wiring differs:

| Concern | Modular monolith (your case, ABP 10.3 Layered) | Microservice |
|---|---|---|
| Where contract lives | `Acme.Forms.Application.Contracts` | `AcmeForms.Contracts` (own NuGet/project) |
| Where implementation lives | `Acme.Forms.Application` | `AcmeForms` service project |
| How consumer references it | Project reference to `Acme.Forms.Application.Contracts`; `[DependsOn(typeof(AcmeFormsApplicationContractsModule))]` | NuGet/package reference to `AcmeForms.Contracts` |
| Transport | In-process DI — direct method call, same `IUnitOfWork`, same `CurrentTenant`, same `CurrentUser` flow automatically | HTTP via generated static client proxies |
| Required on provider | Nothing extra — implementation is auto-registered | `Configure<AbpAspNetCoreMvcOptions>(o => o.ExposeIntegrationServices = true);` |
| Required on consumer | Nothing extra | `abp generate-proxy -t csharp -u <provider-url> -m forms --without-contracts` + `context.Services.AddStaticHttpClientProxies(typeof(AcmeFormsContractsModule).Assembly, "AcmeForms");` + `RemoteServices:AcmeForms:BaseUrl` in `appsettings.json` |
| Failure modes to handle | Effectively none beyond normal DI | Network errors, timeouts, retries, auth token propagation |
| Tenant / user / UoW context | Automatic (same process) | Forwarded via HTTP headers by the generated proxy |

Because the contract is identical, "split Forms out into a microservice later" becomes a wiring-only change inside `AcmeReportsApplicationModule` plus a new NuGet reference. `ReportAppService` itself is untouched.

---

## Summary checklist

- [ ] `IFormSubmissionIntegrationService` + `OwnerSubmissionCountDto` in `Acme.Forms.Application.Contracts`, decorated with `[IntegrationService]`.
- [ ] `FormSubmissionIntegrationService` in `Acme.Forms.Application`, going through `IFormSubmissionRepository` (not through `FormSubmissionAppService`).
- [ ] `Acme.Reports.Application` adds `[DependsOn(typeof(AcmeFormsApplicationContractsModule))]` — and only that, never the Forms `Application` / `Domain` / `EntityFrameworkCore` modules.
- [ ] `ReportAppService` injects `IFormSubmissionIntegrationService`, not `IFormSubmissionAppService`.
- [ ] Don't re-expose the same query from `FormSubmissionAppService` for Reports — that surface is for the Forms UI, not for cross-module callers.
- [ ] If/when Forms becomes a microservice: switch to `AddStaticHttpClientProxies` + `ExposeIntegrationServices = true` + `RemoteServices` config. Consumer code does not change.
