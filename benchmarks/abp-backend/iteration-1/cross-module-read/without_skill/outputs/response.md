# Cross-module reads in an ABP Layered modular monolith

Short answer: **don't inject `IFormSubmissionAppService` into your `ReportAppService`.** It will technically compile and run if `Acme.Reports.Application` references `Acme.Forms.Application.Contracts`, but it's the wrong layer to consume from another module, and it tends to bite you later. The right pattern in ABP is to consume the **Application Contracts** of the source module — but specifically through an *integration* surface, not the public CRUD-style app service that the UI/HTTP API uses.

Below is the reasoning, the four options ABP gives you, and the one I'd actually pick for "how many submissions does owner X have."

---

## Why "just inject the AppService" is a smell

`FormSubmissionAppService` is designed to be the **remote API surface** of the Forms module. That has three consequences you don't want inside another module's server-side code:

1. **It runs through the full app-service pipeline** — authorization (`[Authorize]` policies on Forms endpoints), object-mapping to/from DTOs, audit logging, validation, unit-of-work, etc. When `ReportAppService` calls it server-to-server, you re-pay all of that and you also re-evaluate Forms' permission policies under the *current user* — which may not be what the report needs (e.g. a scheduled report user with no Forms permissions).
2. **It's shaped for the UI**, not for other modules. The DTOs evolve to suit UI screens. A "count submissions by owner" report shouldn't have to drag a `FormSubmissionDto` (with HTML body, attachments, etc.) across a module boundary just to do `.Count()`.
3. **It couples Reports to Forms' UI-facing contract.** When Forms changes a DTO field for the UI, your report breaks for a reason that has nothing to do with reporting.

ABP's modularity guidance treats `*.Application.Contracts` as two things mixed together: the HTTP/UI contract *and* the inter-module contract. The discipline is to separate them.

---

## The four patterns ABP supports, ranked

### 1. Integration Service (recommended for this case)

ABP has a first-class concept for exactly your scenario: an **Integration Service**. It's an application service in the source module whose explicit purpose is "other modules / other microservices call me." It's marked so that:

- Dynamic HTTP API generation **skips** it (it doesn't get auto-exposed as a controller),
- Auto-API-controller conventions ignore it,
- Authorization/audit defaults are tuned for service-to-service rather than end-user calls.

Define it in `Acme.Forms.Application.Contracts`:

```csharp
// Acme.Forms.Application.Contracts/Integration/IFormSubmissionIntegrationService.cs
using Volo.Abp.Application.Services;

namespace Acme.Forms.Integration;

[IntegrationService] // Volo.Abp.Application.Services.IntegrationServiceAttribute
public interface IFormSubmissionIntegrationService : IApplicationService
{
    Task<long> GetSubmissionCountByOwnerAsync(Guid ownerId);
}
```

Implement it in `Acme.Forms.Application`:

```csharp
// Acme.Forms.Application/Integration/FormSubmissionIntegrationService.cs
using Volo.Abp.Application.Services;

namespace Acme.Forms.Integration;

[IntegrationService]
public class FormSubmissionIntegrationService
    : ApplicationService, IFormSubmissionIntegrationService
{
    private readonly IFormSubmissionRepository _submissionRepo;

    public FormSubmissionIntegrationService(IFormSubmissionRepository submissionRepo)
    {
        _submissionRepo = submissionRepo;
    }

    public async Task<long> GetSubmissionCountByOwnerAsync(Guid ownerId)
    {
        return await _submissionRepo.CountAsync(s => s.OwnerId == ownerId);
    }
}
```

Then in Reports (which already references `Acme.Forms.Application.Contracts`):

```csharp
public class ReportAppService : ApplicationService, IReportAppService
{
    private readonly IFormSubmissionIntegrationService _formsIntegration;

    public ReportAppService(IFormSubmissionIntegrationService formsIntegration)
    {
        _formsIntegration = formsIntegration;
    }

    public async Task<OwnerReportDto> GetOwnerReportAsync(Guid ownerId)
    {
        var count = await _formsIntegration.GetSubmissionCountByOwnerAsync(ownerId);
        return new OwnerReportDto { OwnerId = ownerId, SubmissionCount = count };
    }
}
```

Module dependency wiring: `AcmeReportsApplicationModule` declares `[DependsOn(typeof(AcmeFormsApplicationContractsModule))]`. **Do not** depend on `AcmeFormsApplicationModule` from Reports.Application — only on the contracts assembly. The Forms.Application implementation is loaded once at host level (the host module depends on both `AcmeFormsApplicationModule` and `AcmeReportsApplicationModule`), so DI resolves the contract to the implementation at runtime.

Why this is the right one for "count submissions by owner":
- It's a **stable, narrow contract** (just `GetSubmissionCountByOwnerAsync`) — Forms can refactor its DTOs freely.
- It returns a primitive (`long`), no DTO churn.
- It's auto-excluded from the public HTTP API, so you're not accidentally exposing a count endpoint to the world.
- If you ever split Forms out into its own service, the integration service is the seam — you replace the in-process call with an HTTP/`IDistributedEventBus` implementation without touching `ReportAppService`.

### 2. Domain service / repository in Forms — only if Reports has its own EF migrations

If you genuinely need richer querying (joins, projections), expose an `IFormSubmissionRepository` method (e.g. `Task<long> CountByOwnerAsync(Guid ownerId)`) and have a domain-layer integration class call it. Don't try to inject `IFormSubmissionRepository<FormSubmission, Guid>` directly into `ReportAppService` across modules — that pulls Forms' domain types and EF configuration into Reports, and you'll trip over `DbContext` boundaries (each module owns its own `DbContext` in ABP). Keep the cross-module surface at the Application.Contracts layer.

### 3. Distributed event subscription — when the report is precomputed / eventually consistent

If "submission count by owner" is read very often, or you want Reports to keep its own materialized view, have Forms publish `FormSubmissionCreatedEto` / `FormSubmissionDeletedEto` via `IDistributedEventBus`. Reports subscribes and maintains its own `OwnerSubmissionCount` table. Pros: zero coupling to Forms at query time, scales horizontally, survives extracting Forms into a microservice. Cons: eventual consistency, more moving parts. Overkill for a single count, ideal once you have a dashboard hammering this query.

### 4. Direct DB read from Reports across Forms' tables — **don't**

Tempting in a modular monolith because they share a database, but it bypasses Forms' invariants (multi-tenancy filter, soft-delete filter, data filters), and the moment Forms gets split out, every report breaks. Treat Forms' tables as private to Forms.

---

## Concrete recommendation for your scenario

For "Reports needs the submission count by owner from Forms":

1. Add `IFormSubmissionIntegrationService` to `Acme.Forms.Application.Contracts` with a single `GetSubmissionCountByOwnerAsync(Guid ownerId)` method, marked `[IntegrationService]`.
2. Implement it in `Acme.Forms.Application` using the existing `IFormSubmissionRepository` (add a `CountAsync` overload there if needed — that stays inside Forms).
3. In `AcmeReportsApplicationModule`, `[DependsOn(typeof(AcmeFormsApplicationContractsModule))]` (you likely already do).
4. Inject `IFormSubmissionIntegrationService` into `ReportAppService`.
5. Leave `IFormSubmissionAppService` alone — it stays the UI/HTTP-facing surface for Forms.

A few extra notes that catch people out:

- **Multi-tenancy:** the integration service runs in the caller's `ICurrentTenant` scope. If a host-side scheduled report needs to aggregate across tenants, wrap the call in `using (CurrentTenant.Change(tenantId)) { ... }` or `using (DataFilter.Disable<IMultiTenant>()) { ... }` — but do that *inside* the integration service, not in Reports, so Forms keeps control of its own data-access policy.
- **Authorization:** integration services typically don't apply end-user policies (they're internal). If you do need an auth check, use an explicit policy like `FormsPermissions.Submissions.ReadCount` rather than reusing the UI permission.
- **Don't put `FormSubmission` (the entity) on the Reports side.** Only the DTO/primitive crosses the boundary. Reports' module reference graph should be `Reports.Application -> Forms.Application.Contracts` and nothing deeper.
- **Module references stay layered:** `Reports.Domain` should not depend on Forms at all; cross-module reads belong at the application layer (or via distributed events at the domain layer).

So: your instinct that "I should be able to just inject something" is right — ABP gives you DI across modules — but the "something" you inject should be an **integration service interface from Forms.Application.Contracts**, not the public `IFormSubmissionAppService`.
