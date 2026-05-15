# Review: `IFormSubmissionIntegrationService` cross-module read

## Verdict

**Delete the `FormSubmissionIntegrationController`.** Hand-written controllers do not belong on `[IntegrationService]` interfaces. The interface and the application-service implementation are correct as written — keep them, drop the controller.

## What's wrong — the controller

Per **Rule 5 — Cross-module reads via ABP Integration Services**, the skill is explicit:

> Integration services do not need explicit controllers — Rule 4 does not apply. ABP wires their HTTP exposure implicitly through the integration-services pipeline. Don't hand-write a `Controller` that implements an `[IntegrationService]` interface.

Several specific problems flow from that, all of which disappear once you delete the controller:

1. **You've duplicated the HTTP surface.** ABP already exposes `[IntegrationService]`-marked services over `/integration-api/...` automatically in a Microservice deployment (behind the API gateway, consumed via a generated proxy client). Hand-writing `api/forms/integration/form-submissions` creates a second, parallel, public-API-shaped endpoint that defeats the whole point of integration services.

2. **You've put it on the *public* `/api` surface.** Integration services are *hidden from `/api`* by default — that's one of their defining properties. Your controller drags `GetSubmissionCountForOwnerAsync` back onto the public API surface where any caller (the SPA, mobile clients, external consumers) can hit it. That's the opposite of what `[IntegrationService]` is for.

3. **You've turned auditing back on.** Integration services are **audit-off by default**. Routing the call through a normal `AbpControllerBase` action re-enables the standard audit pipeline for those calls, which is almost certainly not what you want for high-volume internal cross-module reads.

4. **You've added `[Authorize]` where the default is no `[Authorize]`.** Integration services have **no `[Authorize]` by default** — they're trusted internal callers (other modules / services within the boundary), authenticated at the transport/gateway layer in Microservice and resolved directly via DI in a modular monolith. Slapping `[Authorize]` on the controller imposes an end-user authorization check on an internal contract that wasn't designed to carry one, and the producing `IntegrationService` itself has no permission declared to match. This is also a Rule 4 problem in the explicit-controller pattern (authorization lives on the AppService, not the controller, because every caller routes through the AppService) — but for integration services the cleaner answer is just *don't write the controller*.

5. **It's redundant boilerplate that will rot.** Even setting all four points above aside, the controller is a pure pass-through (`=> _service.GetSubmissionCountForOwnerAsync(ownerId)`). The integration-services pipeline gives you the same thing for free, and a hand-written copy is one more place to forget to update when the contract changes.

## What's right — keep these as-is

**The interface** — `IFormSubmissionIntegrationService` in `Acme.Forms.Application.Contracts`, marked `[IntegrationService]`, extending `IApplicationService`. That is exactly the Rule 5 shape: the contract lives in the producing module's `*.Application.Contracts` project so consumers can `[DependsOn]` contracts-only.

**The implementation** — `FormSubmissionIntegrationService : ApplicationService, IFormSubmissionIntegrationService` in `Acme.Forms.Application`, injecting `IRepository<FormSubmission, Guid>` and delegating to `CountAsync`. Matches the worked example in Rule 5 essentially verbatim.

A couple of small style nits, not bugs:

- `_submissionRepository.CountAsync(...)` returns `Task<int>` on a generic repository, which is fine. If submission counts can plausibly exceed `int.MaxValue` in your domain (unlikely for forms, but worth a thought), switch to `LongCountAsync` and `Task<long>`.
- Consider `IReadOnlyRepository<FormSubmission, Guid>` instead of `IRepository<...>` since this is a pure read — signals intent and lets EF Core use no-tracking queries. The skill's Rule 2 table calls this out: "Read-only services and query handlers."

## Consumer side — the load-bearing piece you didn't show

When the consumer module (e.g. `Acme.Reports`) takes a dependency on this contract, its `AbpModule` must `[DependsOn]` **`AcmeFormsApplicationContractsModule` only** — never the full `AcmeFormsApplicationModule`. Per Rule 5:

> If you `[DependsOn]` the producer's full `*.Application`, the consumer loads the producer's domain services, repositories, and internal collaborators into the same DI container — every "private" piece of the producer becomes injectable in the consumer, and the boundary is gone in practice even though the source files still live in separate folders. Contracts-only `[DependsOn]` is what makes the seam real.

The HTTP transport (`/integration-api/...`) is generated automatically in the Microservice deployment shape; in a modular monolith both modules resolve the same in-process implementation through DI. Same contract, same DI shape — transport differs by deployment.

## Concrete action items

1. **Delete `Acme.Forms.HttpApi/FormSubmissionIntegrationController.cs`.** Do not replace it with anything.
2. **Keep** the interface and the `FormSubmissionIntegrationService` implementation as written.
3. (Optional) Swap `IRepository<FormSubmission, Guid>` for `IReadOnlyRepository<FormSubmission, Guid>` in the implementation — Rule 2 narrowest-interface guidance.
4. In each consumer module, confirm the `AbpModule` `[DependsOn]` references `AcmeFormsApplicationContractsModule` only, not `AcmeFormsApplicationModule`.

## Rules cited

- **Rule 5 — Cross-module reads via ABP Integration Services** (the controlling rule here): "Integration services do not need explicit controllers — Rule 4 does not apply. ABP wires their HTTP exposure implicitly through the integration-services pipeline. Don't hand-write a `Controller` that implements an `[IntegrationService]` interface." Integration services are also documented as hidden from the public API surface, audit-off, and without `[Authorize]` by default.
- **Rule 4 — Explicit-controller pattern**: applies to *application services*, not integration services. Even where it does apply, `[Authorize]` belongs on the AppService method/class — not on the controller — because every caller (HTTP, background job, integration service, event handler, in-process) routes through the AppService.
- **Rule 2 — Narrowest repository interface**: minor suggestion to prefer `IReadOnlyRepository<TEntity, TKey>` for read-only services.
