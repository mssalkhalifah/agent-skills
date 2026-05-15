# Review: integration service for cross-module reads (ABP 10.3 microservices)

Short version: the contract and the application-service implementation are correct. The third block — the hand-written `FormSubmissionIntegrationController` — should be **deleted**. ABP wires HTTP exposure for `[IntegrationService]` services implicitly through its own integration-services pipeline; writing your own controller actively breaks the conventions an integration service is supposed to inherit.

## What's good (keep these)

### Block 1 — `IFormSubmissionIntegrationService` (Application.Contracts)

Correct as written:

- `[IntegrationService]` lives on the interface in the producing module's `*.Application.Contracts` project — exactly where it belongs. Implementations inherit the attribute, and consumers depend only on `Acme.Forms.Application.Contracts`, not on `Acme.Forms.Application`.
- Inheriting `IApplicationService` is required (it gives the interface its proxy/registration semantics).
- Returning a primitive `Task<int>` for a count is fine for an integration contract — it's a server-to-server read, no DTO ceremony needed.

### Block 2 — `FormSubmissionIntegrationService` (Application)

Also correct:

- It extends `ApplicationService` and implements the `[IntegrationService]` interface.
- It uses `IRepository<FormSubmission, Guid>` (the producing module owns the aggregate, so the full repository is appropriate; `IReadOnlyRepository<FormSubmission, Guid>` would also be reasonable since the method is read-only and would more clearly signal intent — minor style nit, not wrong).
- The implementation is a thin pass-through to `CountAsync` with a predicate — that's the right shape for an integration read.

## What needs to change — delete the controller (Block 3)

`FormSubmissionIntegrationController` should not exist. Several things go wrong by hand-writing it:

### 1. ABP exposes integration services automatically — you don't write a controller for them

This is the core point. In a microservice host, you turn on integration-service exposure once at the host module:

```csharp
// In your <Forms>.HttpApi.Host module (or wherever the producing service is hosted)
public override void PreConfigureServices(ServiceConfigurationContext context)
{
    PreConfigure<AbpAspNetCoreMvcOptions>(options =>
    {
        options.ConventionalControllers.Create(
            typeof(AcmeFormsApplicationModule).Assembly,
            opts =>
            {
                opts.ApplicationServiceTypes = ApplicationServiceTypes.IntegrationServices;
            });
    });
}

public override void ConfigureServices(ServiceConfigurationContext context)
{
    Configure<AbpAspNetCoreMvcOptions>(options =>
    {
        options.ExposeIntegrationServices = true; // off by default
    });

    // Optional — audit logs for integration services (off by default)
    Configure<AbpAuditingOptions>(options =>
    {
        options.IsEnabledForIntegrationService = true;
    });
}
```

That two-flag setup is what makes ABP discover every `[IntegrationService]` in the registered assembly and automatically generate MVC controllers for them under `/integration-api/...`. The controller for `IFormSubmissionIntegrationService` is generated for you — endpoint, route, HTTP verb, model binding, and OpenAPI metadata included. (Verb is inferred from the method-name prefix: `Get*` → GET, so `GetSubmissionCountForOwnerAsync` becomes a GET automatically.)

If `ExposeIntegrationServices` is false (the default) and/or you don't register a `ConventionalControllers.Create` with `ApplicationServiceTypes.IntegrationServices`, the integration service simply isn't reachable over HTTP — that's the entire mechanism the framework provides.

Reference: ABP 10.3 docs — [Integration Services](https://abp.io/docs/10.3/framework/api-development/integration-services) and [Auto API Controllers](https://abp.io/docs/10.3/framework/api-development/auto-controllers).

### 2. Your hand-written controller drops the service onto `/api`, which is the wrong surface

Integration services are deliberately not on the public `/api/...` surface. By auto-wiring them:

- ABP routes them under `/integration-api/...` (different prefix from public APIs).
- They are hidden from `/api/abp/api-definition` and from Swagger by default (no client-app proxy will ever discover them).
- Your API gateway is supposed to *block* external traffic to `/integration-api/*` — only sister microservices on the private network reach it.

Your controller's `[Route("api/forms/integration/form-submissions")]` puts the endpoint on the public `/api` tree. Even with the word "integration" in the path, the gateway has no rule that distinguishes this from a normal public endpoint, so external clients will be able to hit it. That's the security boundary an integration service is supposed to live behind, and the hand-written controller silently dismantles it.

### 3. Your `[Authorize]` attribute contradicts the integration-service contract

`[IntegrationService]` services run **without `[Authorize]` by default**. The model is: the caller is another trusted service inside the private network, identified (if at all) by a service-to-service token (e.g. an OpenIddict `client_credentials` application). Putting `[Authorize]` on the controller turns every internal call into a 401 unless the caller starts attaching user-context bearer tokens, which is not how sister microservices typically call integration APIs in ABP. Either you'll change every caller to forward a token (expensive and wrong for service-to-service) or — more likely — you'll quietly disable `[Authorize]` later and end up worse off than the default.

The right answer is to **not** add `[Authorize]` and to rely on:
- Private-network isolation of `/integration-api/*` at the gateway, and/or
- A dedicated `client_credentials` OpenIddict client for the calling service if you do want token-based service auth (configured at the proxy level, not on the producer's controller).

### 4. Audit-off behavior is also lost

`AbpAuditingOptions.IsEnabledForIntegrationService` defaults to `false` because chatty service-to-service calls would flood the audit log. ABP's auto-generated integration controllers honor that flag. A hand-written `AbpControllerBase` controller doesn't — it's audited like any normal app-service controller. That's another implicit convention the auto pipeline gives you that the hand-written controller silently breaks.

### 5. The "controller as HTTP adapter" pattern that's recommended for *regular* application services does **not** apply here

In ABP 10.3 it is reasonable, for normal application services, to write an explicit thin controller that implements the same `IXxxAppService` interface and just delegates to the app service — that gives you control over routes, verbs, OpenAPI shape, file responses, etc. Integration services are explicitly exempted from that pattern: they have their own pipeline (the `/integration-api` one above), and the framework wires it for you. Don't hand-write a controller that implements an `[IntegrationService]` interface.

## What to do — concrete steps

1. **Delete** `FormSubmissionIntegrationController.cs` from `Acme.Forms.HttpApi`. The whole file goes.
2. In your `Acme.Forms.HttpApi.Host` (or whichever host project boots the Forms microservice), add the two-step wiring shown above:
   - `PreConfigure<AbpAspNetCoreMvcOptions>` with a `ConventionalControllers.Create(...)` call scoped to `ApplicationServiceTypes.IntegrationServices` for the Forms application assembly.
   - `Configure<AbpAspNetCoreMvcOptions>(o => o.ExposeIntegrationServices = true)`.
3. At the API gateway, confirm there's a rule that drops or 404s any external request whose path starts with `/integration-api/`. Only `/api/*` should be reachable from outside the cluster.
4. On the consumer side (the sister microservice that needs the count), depend on `Acme.Forms.Application.Contracts` and register a proxy:
   ```csharp
   context.Services.AddHttpClientProxies(
       typeof(AcmeFormsApplicationContractsModule).Assembly,
       remoteServiceConfigurationName: "Forms");
   ```
   with a corresponding `RemoteServices.Forms.BaseUrl` in `appsettings.json` pointing at the Forms service's internal endpoint. The consumer then injects `IFormSubmissionIntegrationService` directly — same shape as in-process DI, the proxy translates the call to a `GET /integration-api/forms/form-submission-integration/submission-count-for-owner?ownerId=...` (or similar; ABP's URL normalizer derives it from the type and method name).

That's it. The contract and implementation you already have stay; the controller goes; two lines in the host module replace it.

## One small style note (not a bug)

Inside an integration service a read-only method is a clean fit for `IReadOnlyRepository<FormSubmission, Guid>`:

```csharp
public class FormSubmissionIntegrationService
    : ApplicationService, IFormSubmissionIntegrationService
{
    private readonly IReadOnlyRepository<FormSubmission, Guid> _submissionRepository;

    public FormSubmissionIntegrationService(IReadOnlyRepository<FormSubmission, Guid> submissionRepository)
        => _submissionRepository = submissionRepository;

    public Task<int> GetSubmissionCountForOwnerAsync(Guid ownerId)
        => _submissionRepository.CountAsync(s => s.OwnerId == ownerId);
}
```

EF Core uses no-tracking by default on `IReadOnlyRepository`, which is a tiny perf win and — more importantly — signals that this method does not mutate. Optional, but worth doing on a method whose entire job is a count.
