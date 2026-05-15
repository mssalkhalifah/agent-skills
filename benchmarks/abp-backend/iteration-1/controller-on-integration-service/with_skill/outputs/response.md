Short answer: the contract and the implementation look fine, but the **`FormSubmissionIntegrationController` is wrong on three counts and should be deleted entirely**. You don't write controllers for integration services — ABP wires the HTTP exposure for you.

## What's right

The interface and the implementation are textbook. Specifically:

- `[IntegrationService]` is on the **interface** in `*.Application.Contracts` — that's the canonical placement; the implementation inherits the marker.
- The interface inherits `IApplicationService`, so DI / proxy generation / route discovery all work.
- The implementation lives in the producing module's `*.Application` project and depends on `IRepository<FormSubmission, Guid>` directly. That's correct: integration services are read-side adapters owned by the producing module, so going straight to the repository is fine here. (Rule 2's "no `IQueryable` outside the repository" is still satisfied — `CountAsync(predicate)` is a materialised result, not a leaked queryable.)

## What's wrong: the controller

Three issues, each of which alone would be a bug. Together they tell you the whole class shouldn't exist.

### 1. Integration services don't get hand-written controllers (Rule 4 / Rule 5)

This is the headline issue. From the framework guidance:

> Integration services do not need explicit controllers — Rule 4 does not apply. ABP wires their HTTP exposure implicitly through the integration-services pipeline. Don't hand-write a Controller that implements an `[IntegrationService]` interface.

The mechanism: on the host that should expose them, you flip `ExposeIntegrationServices = true` and register `ConventionalControllers.Create(...)` with `ApplicationServiceTypes.IntegrationServices`. ABP then materialises the controllers under the `/integration-api/...` prefix automatically. The route segment is derived from the service name — your method becomes `GET /integration-api/forms/form-submission-integration/submission-count-for-owner?ownerId=...` (or similar; ABP's URL normalizer handles it).

Hand-writing a controller doesn't just duplicate that — it actively defeats it.

### 2. The route is in the wrong place

You routed the controller at `api/forms/integration/form-submissions`. That puts the endpoint under `/api/...`, which is the **public** API surface the gateway is supposed to expose to external clients. Integration services are explicitly designed for the **private** `/integration-api/...` prefix so the gateway can drop external traffic to them at the edge. You've moved an internal-only contract onto the public path.

In a microservice setup that's a real exposure: the gateway sees `/api/forms/...` as a normal public route and lets it through. With a proper `[IntegrationService]` registration the gateway never sees external `/integration-api/*` traffic in the first place — that's the whole point of the prefix split.

### 3. `[Authorize]` on an integration service will 401 every internal call

Integration services bypass authorization by convention — they assume the caller is another trusted microservice on the private network, and they flow without a user token. Slapping `[Authorize]` on the controller re-enables full bearer-token requirements. The calling side is using a generated proxy that doesn't attach a user token, so every call from `Reports` to `Forms` will come back as `401`.

If you genuinely need service-to-service auth (which is reasonable in some architectures), the right move is OpenIddict client-credentials with a dedicated client for the caller, plumbed through the proxy's `AbpHttpClientBuilderOptions.ProxyClientBuildActions` — not `[Authorize]` on the controller.

## Corrected setup

Delete `FormSubmissionIntegrationController` entirely. Keep the interface and the implementation as you have them, and add the host-side wiring on whichever host project is supposed to expose Forms's integration API to other services (typically `Acme.Forms.HttpApi.Host` or a dedicated integration host module):

```csharp
[DependsOn(
    typeof(AcmeFormsApplicationModule),
    typeof(AbpAspNetCoreMvcModule)
)]
public class AcmeFormsHttpApiHostModule : AbpModule
{
    public override void PreConfigureServices(ServiceConfigurationContext context)
    {
        PreConfigure<AbpAspNetCoreMvcOptions>(options =>
        {
            options.ConventionalControllers.Create(
                typeof(AcmeFormsApplicationModule).Assembly,
                opts =>
                {
                    // Filter this Create call to integration-only services.
                    // (You'll typically have a second Create call with
                    //  ApplicationServiceTypes.Default for the public API.)
                    opts.ApplicationServiceTypes =
                        ApplicationServiceTypes.IntegrationServices;
                });
        });
    }

    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        // Mandatory — without this, integration services don't even register
        // as MVC actions and /integration-api/* returns 404.
        Configure<AbpAspNetCoreMvcOptions>(options =>
        {
            options.ExposeIntegrationServices = true;
        });

        // Optional — off by default to keep audit volume sane in chatty
        // service-to-service systems. Flip on if you want them audited.
        Configure<AbpAuditingOptions>(options =>
        {
            options.IsEnabledForIntegrationService = true;
        });
    }
}
```

A couple of important timing details that bite people:

- `ConventionalControllers.Create` **must** go in `PreConfigureServices`, not `ConfigureServices`. By the time `ConfigureServices` runs for downstream modules, MVC has already started building action descriptors and additions silently no-op — your routes just won't exist and you get no error.
- `ExposeIntegrationServices = true` goes in regular `ConfigureServices`. Without it, integration services are filtered out of MVC actions entirely and you'll see 404s.

## Don't forget the gateway

In a microservice deployment, also make sure the public API gateway (YARP / Ocelot / whatever) drops or 404s any external request whose path starts with `/integration-api/`. Integration services skip authorization by design, so they rely on private-network isolation to be safe. Treat `/integration-api/*` as private-network-only and document the boundary in your gateway config.

## Consumer side (just for completeness)

On the `Reports` side that consumes this, you don't change anything based on whether you're in-process or cross-process — that's the whole appeal of this pattern. The `Reports` module:

- Depends on `AcmeFormsApplicationContractsModule` (the `*.Contracts` project, **not** the `*.Application` project).
- Injects `IFormSubmissionIntegrationService` directly into `ReportAppService` (or a `ReportManager`, depending on where the read fits in your DDD layering).
- In a modular monolith, DI resolves it to the in-process implementation. No HTTP hop, `ExposeIntegrationServices` can stay `false`.
- In a real microservice split, register `AddHttpClientProxies(typeof(AcmeFormsApplicationContractsModule).Assembly, "Forms")` on the consumer host and point `RemoteServices:Forms:BaseUrl` at the Forms service. Same consumer code, dynamic proxy generated at runtime.

## TL;DR

- Keep the interface (with `[IntegrationService]`) and the implementation — both correct.
- **Delete the controller.** It violates the "no hand-written controllers for integration services" rule, puts the route on the wrong (public) prefix, and `[Authorize]` will break every caller.
- Replace it with `ExposeIntegrationServices = true` plus a `ConventionalControllers.Create(..., ApplicationServiceTypes.IntegrationServices)` registration in `PreConfigureServices` of the integration host module.
- Make sure the gateway blocks `/integration-api/*` from external traffic.
