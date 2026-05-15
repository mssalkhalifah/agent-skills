# Review: `FormSubmissionIntegrationService` — delete the hand-written controller

You got the contract and the implementation right. You got the controller wrong — it must be removed entirely.

## What to do

**DELETE `FormSubmissionIntegrationController` from `Acme.Forms.HttpApi`.** Do not replace it with anything. The whole file goes.

## Why — Rule 5: integration services are wired implicitly, not via explicit controllers

The skill's Rule 5 ("Cross-module reads via ABP Integration Services") is explicit on this point:

> **Integration services do not need explicit controllers** — Rule 4 does not apply. ABP wires their HTTP exposure implicitly through the integration-services pipeline. Don't hand-write a `Controller` that implements an `[IntegrationService]` interface.

That is: the explicit-controller pattern from Rule 4 (`XxxController : <BaseController>, IXxxAppService` delegating one-liner per action) is the **recommended pattern for regular application services only**. Integration services are a different surface and ABP wires their HTTP exposure for you:

- An `IApplicationService` decorated with `[IntegrationService]` is picked up by ABP's integration-services pipeline (via `ExposeIntegrationServices` in the host module's `Configure<AbpAspNetCoreMvcOptions>(...)`) and exposed automatically under the dedicated `/integration-api/...` route prefix — separate from the public `/api/...` surface used by Rule 4 controllers.
- In a Microservice topology the gateway routes `/integration-api/...` between services and a generated proxy client consumes it on the other side; in a modular monolith DI resolves it in-process with no HTTP hop. Same `[IntegrationService]` contract, transport differs by deployment.
- They are **hidden from the public API surface, audit-off by default, and have no `[Authorize]` by default**. Hand-writing `[Authorize]` plus `[Route("api/forms/integration/form-submissions")]` puts the endpoint on the *public* `/api` surface with full authorization and auditing — exactly the opposite of what `[IntegrationService]` is for. It also means you now have two different HTTP exposures for the same operation (your `/api/forms/integration/...` controller route *and* ABP's implicit `/integration-api/...` route), which is a real footgun.

Bluntly: writing `FormSubmissionIntegrationController : AbpControllerBase, IFormSubmissionIntegrationService` is doing Rule-4 plumbing for a Rule-5 contract. Rule 5 specifically tells you not to.

## What's correct already

The first two blocks (interface in `*.Application.Contracts`, implementation in `*.Application`) match Rule 5 exactly:

- `IFormSubmissionIntegrationService : IApplicationService` lives in `Acme.Forms.Application.Contracts`, decorated with `[IntegrationService]`. Correct — that's the boundary the consumer module's `[DependsOn(typeof(AcmeFormsApplicationContractsModule))]` will reach for, and contracts-only `[DependsOn]` is the load-bearing line that keeps the seam real (never `[DependsOn]` the producer's full `*.Application`).
- `FormSubmissionIntegrationService : ApplicationService, IFormSubmissionIntegrationService` lives in `Acme.Forms.Application` and uses `IRepository<FormSubmission, Guid>.CountAsync(...)` to return a materialised `Task<int>`. That is materialised (no `IQueryable` leak — Rule 2 holds), goes through the repository, and the body is a single delegating expression. Fine.

So: keep blocks 1 and 2 as-is. Delete block 3.

## After the deletion, what wires it up

Nothing more for you to write in the HttpApi project. In the producing service/module's host `AbpModule`, the integration-services exposure is enabled (typically once, at host level) via something like:

```csharp
Configure<AbpAspNetCoreMvcOptions>(options =>
{
    options.ExposeIntegrationServices = true; // exposes [IntegrationService] endpoints under /integration-api
});
```

(Microservice template ABP-host code does this for you in the integration-services-enabled service host; check the existing host module — you almost certainly don't need to add it.) After that:

- The endpoint is reachable under `/integration-api/...` with a conventional route shape derived from the interface — not the `[Route("api/forms/...")]` you wrote.
- It is **not** on `/api`, so frontend / public-client proxy generation will not pick it up — which is the point.
- No `[Authorize]` by default; if you need authorization on a specific integration endpoint, put `[Authorize(MyPermissions.X)]` on the **app-service method** (the Rule-4-style "authorization lives on the service, not the controller" guidance still applies — it just isn't a controller question here because there is no controller).

## Summary

1. **Delete `FormSubmissionIntegrationController` entirely** — no replacement. Rule 5 says explicit controllers do not apply to integration services; ABP wires the HTTP exposure implicitly through the integration-services pipeline (`/integration-api/...`, `ExposeIntegrationServices`).
2. **Block 1 (contract in `*.Application.Contracts`) and block 2 (implementation in `*.Application`) are correct.** Keep them as written.
3. Integration services are **off the public `/api` surface, audit-off, no `[Authorize]` by default** — your hand-written controller's `[Authorize]` + `/api/...` route inverts all three of those properties and is what tips the code off as wrong.
