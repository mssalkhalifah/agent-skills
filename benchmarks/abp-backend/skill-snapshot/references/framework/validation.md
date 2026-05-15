# Validation Pipeline

> ABP's validation pipeline automatically validates method inputs on application services (and any IValidationEnabled class) using DataAnnotations, IValidatableObject, FluentValidation, and pluggable IObjectValidationContributor extensions, throwing AbpValidationException that is auto-converted to HTTP 400.

## When to load this reference

- User asks how to validate DTOs / inputs in ABP application services.
- User wants to integrate FluentValidation with ABP (Volo.Abp.FluentValidation, AbstractValidator<T>).
- User asks why their [Required]/[StringLength]/[Range] attributes are (or aren't) being enforced automatically.
- User wants to write a custom validation contributor that runs across all DTOs (cross-cutting validation).
- User asks about AbpValidationException, ValidationError, or how validation errors map to HTTP 400 responses.
- User asks how to disable validation on a method, class, or property ([DisableValidation]).
- User asks about IShouldNormalize / normalizing input after validation but before method execution.
- User wants to manually validate an object outside the auto pipeline (IObjectValidator.ValidateAsync / GetErrorsAsync).
- User asks about IgnoredTypes or AbpValidationOptions configuration.
- User questions why validation is not firing — usually because the method isn't virtual, the class isn't resolved through DI, or the type is in IgnoredTypes.

**Audience:** ABP developers writing application services, controllers, and DTOs who need automatic input validation, custom validation rules, FluentValidation integration, or cross-cutting validation contributors.

## Key concepts

- IValidationEnabled (Volo.Abp.Validation): Marker interface; any class implementing it triggers automatic method-input validation via dynamic proxy interception. ApplicationService implements it implicitly. Reach for it on a non-application-service class you also want auto-validated.
- IObjectValidator (Volo.Abp.Validation): Service for manual validation. Methods: ValidateAsync(object, name) — throws AbpValidationException on failure; GetErrorsAsync(object, name) — returns ObjectValidationResult without throwing. Reach for it when validating outside the auto pipeline (background jobs, domain services, custom code paths).
- IMethodInvocationValidator (Volo.Abp.Validation): Internal interceptor service that validates method arguments during invocation. You rarely call this directly; the framework invokes it for IValidationEnabled types.
- IObjectValidationContributor (Volo.Abp.Validation): Extensibility hook. Implement AddErrorsAsync(ObjectValidationContext) to add cross-cutting validation logic. Built-in contributors: DataAnnotationObjectValidationContributor, ValidatableObjectValidationContributor; FluentValidationObjectValidationContributor is added by Volo.Abp.FluentValidation. Reach for it for custom rules that should run for every DTO.
- ObjectValidationContext (Volo.Abp.Validation): Context passed to a contributor. Exposes ValidatingObject (the object being validated) and Errors (List<ValidationError> to append to).
- ValidationError (Volo.Abp.Validation): Represents a single validation failure with a message and member-name list. Equivalent to System.ComponentModel.DataAnnotations.ValidationResult but ABP-native.
- AbpValidationException (Volo.Abp.Validation): Exception thrown when validation fails. Carries IReadOnlyList<ValidationError> ValidationErrors. Logged at Warning level. Auto-converted to HTTP 400 by the ABP exception filter.
- IShouldNormalize (Volo.Abp.Runtime.Validation in older versions; namespace path may differ in 10.3 — see version_pins) [uncertain]: Implement Normalize() on a DTO to massage input (e.g., default sort order, trim strings) AFTER validation passes and BEFORE the target method runs.
- IValidatableObject (System.ComponentModel.DataAnnotations): Standard .NET interface; implement Validate(ValidationContext) on a DTO for custom per-DTO validation logic. ABP resolves services via validationContext.GetRequiredService<T>().
- [DisableValidation] (Volo.Abp.Validation): Attribute applicable to class, method, or property to opt out of the validation pipeline.
- AbpValidationOptions (Volo.Abp.Validation): Options class. Properties: IgnoredTypes (List<Type>) — types to skip; ObjectValidationContributors (ITypeList<IObjectValidationContributor>) — registered contributors. Configure in your module's ConfigureServices.
- AbpValidationModule (Volo.Abp.Validation): The module that registers the validation infrastructure; depended on transitively by AbpDddApplicationModule.
- AbpFluentValidationModule (Volo.Abp.FluentValidation): Adds FluentValidation as a contributor. Bring it in via [DependsOn(typeof(AbpFluentValidationModule))]. Validators inherit from AbstractValidator<T> and are auto-discovered.

## Configuration pattern

Validation is enabled by default for any class implementing IValidationEnabled (which all ApplicationService subclasses do). To customize, configure AbpValidationOptions inside your module's ConfigureServices. To plug in FluentValidation, add the AbpFluentValidationModule dependency. Custom contributors can be registered as DI components implementing IObjectValidationContributor (transient by convention). Example:

```csharp
using Volo.Abp.Modularity;
using Volo.Abp.Validation;
using Volo.Abp.FluentValidation; // optional, only if using FluentValidation

[DependsOn(
    typeof(AbpValidationModule),
    typeof(AbpFluentValidationModule) // only if integrating FluentValidation
)]
public class MyApplicationModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        Configure<AbpValidationOptions>(options =>
        {
            // Skip validation for specific types entirely
            options.IgnoredTypes.Add(typeof(MyExternalLegacyDto));

            // Register a custom contributor (alternative to DI registration)
            options.ObjectValidationContributors.Add<MyCustomValidationContributor>();
        });
    }
}
```

Key options:
- IgnoredTypes: validation pipeline returns early for instances of these types.
- ObjectValidationContributors: ITypeList of contributors run in order; each appends to the shared errors list. After all contributors run, any accumulated errors throw AbpValidationException.

Automatic validation prerequisites: (1) the class must implement IValidationEnabled (ApplicationService does so by default); (2) the method must be virtual or the call must go through an interface, because validation runs via DynamicProxy interception; (3) the class must be resolved through the ABP DI container (not 'new'-ed).

## Code examples

### DataAnnotations on a DTO

_Standard input validation using built-in attributes; runs automatically for application services._

```csharp
using System.ComponentModel.DataAnnotations;

public class CreateBookDto
{
    [Required]
    [StringLength(100)]
    public string Name { get; set; }

    [Required]
    [StringLength(1000)]
    public string Description { get; set; }

    [Range(0, 999.99)]
    public decimal Price { get; set; }
}

public class BookAppService : ApplicationService, IBookAppService
{
    public virtual Task<BookDto> CreateAsync(CreateBookDto input)
    {
        // 'input' is already validated by ABP before this line runs.
        // If validation failed, AbpValidationException was thrown and
        // we never reached here.
        return Task.FromResult(new BookDto());
    }
}
```

**Key lines:** Method is virtual (required for DynamicProxy interception). ApplicationService implements IValidationEnabled. Attributes are read by DataAnnotationObjectValidationContributor.

### IValidatableObject for cross-property rules

_Validation logic that depends on multiple properties or services from DI._

```csharp
using System.Collections.Generic;
using System.ComponentModel.DataAnnotations;

public class CreateBookDto : IValidatableObject
{
    [Required] public string Name { get; set; }
    [Required] public string Description { get; set; }

    public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
    {
        if (Name == Description)
        {
            yield return new ValidationResult(
                "Name and Description cannot be the same!",
                new[] { nameof(Name), nameof(Description) });
        }

        // Resolve services from DI inside validation:
        // var bookRepo = validationContext.GetRequiredService<IRepository<Book, Guid>>();
    }
}
```

**Key lines:** validationContext.GetRequiredService<T>() lets you reach into DI from within Validate(). Errors yielded here are merged with attribute-level errors before AbpValidationException is thrown.

### FluentValidation integration

_Using AbstractValidator<T> for richer/fluent rule definitions._

```csharp
// 1) Module dependency
using Volo.Abp.FluentValidation;
using Volo.Abp.Modularity;

[DependsOn(typeof(AbpFluentValidationModule))]
public class MyApplicationModule : AbpModule { }

// 2) Validator class — auto-discovered, no manual registration required
using FluentValidation;

public class CreateUpdateBookDtoValidator : AbstractValidator<CreateUpdateBookDto>
{
    public CreateUpdateBookDtoValidator()
    {
        RuleFor(x => x.Name).NotEmpty().Length(3, 100);
        RuleFor(x => x.Price).ExclusiveBetween(0.0m, 999.0m);
    }
}
```

**Key lines:** [DependsOn(typeof(AbpFluentValidationModule))] wires the FluentValidationObjectValidationContributor. Validators are discovered automatically by type.

### Custom IObjectValidationContributor

_Cross-cutting validation rule that should apply to many DTOs (e.g., reject objects whose tenant context is wrong)._

```csharp
using System.Threading.Tasks;
using Volo.Abp.DependencyInjection;
using Volo.Abp.Validation;

public class TenantConsistencyValidationContributor
    : IObjectValidationContributor, ITransientDependency
{
    public Task AddErrorsAsync(ObjectValidationContext context)
    {
        if (context.ValidatingObject is IMultiTenantDto dto && dto.TenantId == default)
        {
            context.Errors.Add(new ValidationError(
                "TenantId must be set on multi-tenant DTOs.",
                new[] { nameof(IMultiTenantDto.TenantId) }));
        }
        return Task.CompletedTask;
    }
}
```

**Key lines:** Implementing ITransientDependency auto-registers the contributor in DI; ABP picks it up via the registered IObjectValidationContributor list.

### Manual validation via IObjectValidator

_Validate an object outside an application-service method (e.g., inside a domain service or background job)._

```csharp
using Volo.Abp.DependencyInjection;
using Volo.Abp.Validation;

public class BookImportService : IDomainService, ITransientDependency
{
    private readonly IObjectValidator _objectValidator;

    public BookImportService(IObjectValidator objectValidator)
    {
        _objectValidator = objectValidator;
    }

    public async Task ImportAsync(CreateBookDto input)
    {
        // Throws AbpValidationException on failure:
        await _objectValidator.ValidateAsync(input);

        // Or non-throwing variant:
        // var result = await _objectValidator.GetErrorsAsync(input);
        // if (result.Errors.Any()) { /* handle */ }
    }
}
```

**Key lines:** ValidateAsync throws; GetErrorsAsync returns ObjectValidationResult so you can handle errors without exceptions.

### Disabling validation selectively

_Skip the automatic pipeline for a specific method, class, or property._

```csharp
using Volo.Abp.Validation;

[DisableValidation]
public class LegacyInputDto
{
    public string AnythingGoes { get; set; }
}

public class MyAppService : ApplicationService
{
    [DisableValidation]
    public virtual Task RawIngestAsync(LegacyInputDto input) => Task.CompletedTask;

    public virtual Task PartialAsync(MixedDto input) => Task.CompletedTask;
}

public class MixedDto
{
    [Required] public string ValidatedField { get; set; }

    [DisableValidation]
    public string FreeForm { get; set; }
}
```

**Key lines:** [DisableValidation] is recognized at method, class, and property scope by IMethodInvocationValidator and IObjectValidator.

## Common mistakes

### Forgetting to make application-service methods virtual.

**Why wrong:** Validation runs through Castle DynamicProxy interception. Non-virtual methods on a concrete class cannot be intercepted, so the validator never runs and invalid input slips through.

**Correct pattern:** Either declare methods as `virtual` on the concrete service or always invoke through the interface (e.g., IBookAppService) so the proxy can intercept.

```csharp
public virtual Task<BookDto> CreateAsync(CreateBookDto input) { ... }
```

### Manually `new`-ing a service in tests or code, expecting validation to fire.

**Why wrong:** Validation only runs on DI-resolved instances wrapped by the interception proxy. `new MyAppService(...)` returns the raw class with no interceptor.

**Correct pattern:** Resolve through the IoC container (constructor injection or LazyServiceProvider). In tests use ABP's test base which builds the service through the container.

```csharp
var svc = GetRequiredService<IBookAppService>(); await svc.CreateAsync(input);
```

### Catching AbpValidationException inside the service to 'handle' validation.

**Why wrong:** AbpValidationException is a transport-layer concern that the framework converts to HTTP 400 with a structured error payload. Swallowing it loses error details for clients and breaks the contract.

**Correct pattern:** Let AbpValidationException propagate; if you need pre-flight checks, use IObjectValidator.GetErrorsAsync to inspect errors without throwing.

```csharp
var result = await _objectValidator.GetErrorsAsync(input); if (result.Errors.Any()) { /* decide what to do */ }
```

### Registering a FluentValidation validator manually in DI when AbpFluentValidationModule is referenced.

**Why wrong:** AbpFluentValidationModule auto-discovers AbstractValidator<T> implementations. Double-registration can lead to duplicate error messages or unpredictable ordering.

**Correct pattern:** Define the validator class and rely on automatic discovery; do not call services.AddTransient for it explicitly.

```csharp
Just declare `public class XValidator : AbstractValidator<X> { ... }` and add `[DependsOn(typeof(AbpFluentValidationModule))]` once.
```

### Using IShouldNormalize.Normalize() to perform validation logic.

**Why wrong:** Normalize runs AFTER validation passes; throwing inside Normalize bypasses the validation error pipeline and surfaces as a 500.

**Correct pattern:** Put guard logic in IValidatableObject.Validate or DataAnnotation attributes; reserve Normalize for sanitization (defaults, trimming, lowercasing).

```csharp
Move 'if (x == null) throw' into Validate(), keep Normalize for `Sorting ??= "Name ASC";`.
```

### Adding a type to AbpValidationOptions.IgnoredTypes to silence a stubborn validation error.

**Why wrong:** IgnoredTypes disables ALL validation for that type — DataAnnotations, FluentValidation, custom contributors. The original bug returns later as a corrupt-data issue.

**Correct pattern:** Diagnose and fix the failing rule; use [DisableValidation] only at the property/method scope where appropriate.

```json
[DisableValidation] public string FreeForm { get; set; } instead of options.IgnoredTypes.Add(typeof(MyDto)).
```

### Implementing IObjectValidationContributor without a DI lifetime marker.

**Why wrong:** Contributors are resolved per-validation; without ITransientDependency (or explicit registration in AbpValidationOptions.ObjectValidationContributors / DI), the framework will not invoke your contributor.

**Correct pattern:** Implement ITransientDependency or register via Configure<AbpValidationOptions>(o => o.ObjectValidationContributors.Add<...>()).

```csharp
public class MyContributor : IObjectValidationContributor, ITransientDependency { ... }
```

## Version pins (ABP 10.3)

- ABP 10.3: AbpValidationOptions exposes IgnoredTypes (List<Type>) and ObjectValidationContributors (ITypeList<IObjectValidationContributor>); both are configurable in module ConfigureServices.
- ABP 10.3: AbpValidationException is logged at Warning by default and serialized to HTTP 400 via the standard exception filter — change LogLevel by overriding the exception subscriber if needed.
- ABP 10.3: IObjectValidator.ValidateAsync and GetErrorsAsync are async-only; no synchronous Validate variants are exposed publicly.
- ABP 10.3: Volo.Abp.FluentValidation auto-discovery picks up any AbstractValidator<T> in scanned assemblies — manual AddTransient registrations should be avoided.
- IShouldNormalize: present in ABP, but the exact namespace under 10.3 (legacy ASP.NET Boilerplate placed it in Abp.Runtime.Validation; ABP framework may expose it under Volo.Abp.Validation). [uncertain] — confirm against the running 10.3 NuGet metadata before quoting the namespace.
- DataAnnotation localization keys: error messages are auto-localized using the `DefaultResource` of the validating type's module; behavior may shift if a future ABP version changes the default-resource lookup. [uncertain]
- Built-in contributors registered by AbpValidationModule (DataAnnotationObjectValidationContributor, ValidatableObjectValidationContributor) — ordering relative to FluentValidation contributor is implementation-defined and may change between minor versions. [uncertain]

## Cross-references

**Phase 1 references:**
- [references/framework/application-services.md](../framework/application-services.md) — application services automatically participate in the validation pipeline (IValidationEnabled is implicit). Cross-link when explaining where validation runs.
- [references/framework/exception-handling.md](../framework/exception-handling.md) — AbpValidationException is converted to HTTP 400 by ABP's exception filter; cross-link when covering error responses and the RemoteServiceErrorInfo payload.
- [references/framework/dependency-injection.md](../framework/dependency-injection.md) — custom IObjectValidationContributor and validators are resolved via DI; cross-link when explaining ITransientDependency and module registration.
- [references/framework/modularity.md](../framework/modularity.md) — AbpFluentValidationModule and AbpValidationModule dependency wiring; cross-link when discussing [DependsOn] and ConfigureServices.
- [references/framework/localization.md](../framework/localization.md) — ABP localizes DataAnnotation error messages automatically; cross-link when discussing custom validation messages.
- [references/framework/aspnetcore-mvc.md](../framework/aspnetcore-mvc.md) — model state / controller validation integration; cross-link for MVC vs application service validation paths.

**External docs:**
- FluentValidation — https://docs.fluentvalidation.net/ — referenced for AbstractValidator<T> rule syntax used by Volo.Abp.FluentValidation.
- ASP.NET Core Model Validation — https://learn.microsoft.com/aspnet/core/mvc/models/validation — underlying DataAnnotations and ModelState concepts that ABP builds on.
- System.ComponentModel.DataAnnotations — https://learn.microsoft.com/dotnet/api/system.componentmodel.dataannotations — namespace for [Required], [StringLength], [Range], IValidatableObject, ValidationContext.
- .NET DI (Microsoft.Extensions.DependencyInjection) — https://learn.microsoft.com/dotnet/core/extensions/dependency-injection — required to understand contributor registration and DynamicProxy interception.

## Sources (ABP 10.3)

- [https://abp.io/docs/10.3/framework/fundamentals/validation — Primary reference for ABP's validation pipeline: IValidationEnabled, IObjectValidator, IObjectValidationContributor, AbpValidationException, DataAnnotations integration, IValidatableObject, [DisableValidation] attribute, automatic validation in application services and MVC.](https://abp.io/docs/10.3/framework/fundamentals/validation — Primary reference for ABP's validation pipeline: IValidationEnabled, IObjectValidator, IObjectValidationContributor, AbpValidationException, DataAnnotations integration, IValidatableObject, [DisableValidation] attribute, automatic validation in application services and MVC.)
- [https://abp.io/docs/10.3/framework/fundamentals/fluent-validation — Volo.Abp.FluentValidation package: AbpFluentValidationModule dependency, AbstractValidator<T> usage, automatic discovery of validator classes by ABP, installation via abp add-package or dotnet add package.](https://abp.io/docs/10.3/framework/fundamentals/fluent-validation — Volo.Abp.FluentValidation package: AbpFluentValidationModule dependency, AbstractValidator<T> usage, automatic discovery of validator classes by ABP, installation via abp add-package or dotnet add package.)
- [https://abp.io/docs/latest/framework/fundamentals/validation — Cross-checked latest doc revision; content matches 10.3 (no version-specific deviation noted).](https://abp.io/docs/latest/framework/fundamentals/validation — Cross-checked latest doc revision; content matches 10.3 (no version-specific deviation noted).)
- [https://github.com/abpframework/abp/blob/dev/framework/src/Volo.Abp.Validation/Volo/Abp/Validation/AbpValidationOptions.cs — Source for AbpValidationOptions properties (IgnoredTypes: List<Type>; ObjectValidationContributors: ITypeList<IObjectValidationContributor>) used to configure the pipeline in PreConfigureServices/ConfigureServices.](https://github.com/abpframework/abp/blob/dev/framework/src/Volo.Abp.Validation/Volo/Abp/Validation/AbpValidationOptions.cs — Source for AbpValidationOptions properties (IgnoredTypes: List<Type>; ObjectValidationContributors: ITypeList<IObjectValidationContributor>) used to configure the pipeline in PreConfigureServices/ConfigureServices.)
- [https://github.com/abpframework/abp/blob/dev/framework/src/Volo.Abp.Validation/Volo/Abp/Validation/AbpValidationModule.cs — Confirms AbpValidationModule registration and default contributors wiring.](https://github.com/abpframework/abp/blob/dev/framework/src/Volo.Abp.Validation/Volo/Abp/Validation/AbpValidationModule.cs — Confirms AbpValidationModule registration and default contributors wiring.)
- [https://github.com/abpframework/abp/blob/dev/framework/src/Volo.Abp.Validation/Volo/Abp/Validation/DataAnnotationObjectValidationContributor.cs — Reference implementation of a built-in IObjectValidationContributor (DataAnnotation-based).](https://github.com/abpframework/abp/blob/dev/framework/src/Volo.Abp.Validation/Volo/Abp/Validation/DataAnnotationObjectValidationContributor.cs — Reference implementation of a built-in IObjectValidationContributor (DataAnnotation-based).)

Last verified: 2026-05-10
