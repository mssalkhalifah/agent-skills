# Review: `FormSubmissionIntegrationService` cross-module read

**Verdict: delete the hand-written `FormSubmissionIntegrationController`.** The interface and the application-service implementation are correct; the controller is the one piece that's wrong, and it actively breaks the contract you're trying to honor.

## What's right

Per Rule 5 — Cross-module reads via ABP Integration Services:

- `IFormSubmissionIntegrationService` is defined in the producing module's `*.Application.Contracts` project, marked `[IntegrationService]`, and extends `IApplicationService`. Correct.
- `FormSubmissionIntegrationService` lives in the producing module's `*.Application` project, derives from `ApplicationService`, and implements the contract. Correct.
- The implementation uses `IRepository<FormSubmission, Guid>` and returns a materialised `Task<int>` from `CountAsync(...)` — no `IQueryable` leakage. Correct (Rule 2 holds).

Don't forget the load-bearing wiring on the **consumer** side: the consuming module's `AbpModule` must `[DependsOn(typeof(AcmeFormsApplicationContractsModule))]` — contracts only, never the producer's full `*.Application` module. That's the line that actually preserves the boundary.

## What's wrong — delete the controller

`FormSubmissionIntegrationController` in `Acme.Forms.HttpApi` should not exist. From Rule 5:

> **Integration services do not need explicit controllers** — Rule 4 does not apply. ABP wires their HTTP exposure implicitly through the integration-services pipeline. Don't hand-write a `Controller` that implements an `[IntegrationService]` interface.

That is, the explicit-controller pattern from Rule 4 is for *regular* application services. Integration services are deliberately different: ABP's integration-services pipeline handles their HTTP exposure for you (in Microservice deployments, under `/integration-api/...` behind the gateway; in a modular monolith, DI resolves the consumer's `IFormSubmissionIntegrationService` directly to the in-process implementation with no HTTP hop at all).

Three concrete problems your controller introduces beyond just being redundant:

1. **`[Authorize]` on the controller contradicts the `[IntegrationService]` contract.** Integration services are **audit-off and have no `[Authorize]` by default** — they're an internal seam between trusted modules, not a public API. Slapping `[Authorize]` on the controller turns this into a normal user-authenticated endpoint and defeats the design. If you want a check, put it on the AppService method as a deliberate authorization decision; do not blanket-`[Authorize]` the HTTP adapter.
2. **`[Route("api/forms/integration/form-submissions")]` re-exposes it on the public `/api` surface.** Integration services are **hidden from `/api` by default** — that's one of the three things `[IntegrationService]` buys you (hidden from public API surface, audit-off, no `[Authorize]` by default). Your hand-written controller punches them back onto `/api` and undoes the hiding.
3. **It duplicates work ABP already does.** The integration-services pipeline routes the call for you. The controller adds nothing and creates a parallel route that will drift from the auto-wired one.

## What to do

1. Delete `Acme.Forms.HttpApi/FormSubmissionIntegrationController.cs`.
2. Keep the interface (`Acme.Forms.Application.Contracts`) and the implementation (`Acme.Forms.Application`) exactly as written.
3. On the consumer side, confirm `[DependsOn(typeof(AcmeFormsApplicationContractsModule))]` is set on the consuming module's `AbpModule` and inject `IFormSubmissionIntegrationService` directly into the consumer's `*AppService` / `*Manager`.
4. For Microservice deployments, generate the proxy client in the consumer service via `abp generate-proxy` (it will pick up the integration-service endpoint automatically); for a modular monolith, DI handles it with no extra step.

One small note unrelated to the rule: `CountAsync` in ABP/EF Core returns `int`, so `Task<int>` is fine here. If the table could ever exceed `int.MaxValue` rows for a single owner you'd want `LongCountAsync` and `Task<long>`, but that's a future-proofing comment, not a correctness one.
