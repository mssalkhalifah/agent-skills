# Modules — Content (CMS Kit, Docs, File Management, Forms)

> Reference for the four ABP 10.3 content/document modules — CMS Kit (and CMS Kit Pro), Docs, File Management, and Forms — covering installation, AbpModule wiring, BLOB storage hookup, GlobalFeature toggles, and the canonical app-service surface for each.

## When to load this reference

- User asks how to add a blog/page/comments/ratings system to an ABP app (CMS Kit)
- User asks how to add newsletters, FAQs, polls, contact forms, page feedback, or URL forwarding (CMS Kit Pro)
- User asks how to host versioned product/feature documentation inside their ABP app (Docs module)
- User asks how to integrate GitHub or filesystem-backed Markdown docs with multi-language switching
- User wants a folder/file explorer with upload/download/sharing inside an ABP app (File Management)
- User asks how to wire BLOB storing (FileSystem/Azure/AWS) for the File Management module
- User asks how to build, embed, or share questionnaires and capture responses (Forms module)
- User asks about FormApplicationService, FormResponse export to CSV, or per-question types
- User asks which CMS-Kit features are open-source vs Pro-licensed
- User asks where module tables live (Cms, Frm, Fm prefixes) or how to set per-module connection strings

**Audience:** ABP 10.3 developers integrating off-the-shelf content, documentation, file-storage, or form-collection modules into MVC/Razor Pages or Blazor solutions, including teams evaluating which features are free (CMS Kit) vs Pro-licensed (CMS Kit Pro, File Management, Forms).

## Key concepts

- CmsKitWebModule / CmsKitApplicationModule / CmsKitDomainModule / CmsKitEntityFrameworkCoreModule (Volo.CmsKit.*) — module classes added via [DependsOn] in each layer to bring in pages, blogs, comments, reactions, ratings, tags, menus, marked items, global resources, and dynamic widgets. Reach for these whenever you want a packaged, public-website CMS feature set without re-building it.
- CmsKitProWebModule / CmsKitProApplicationModule (Volo.CmsKit.Pro.*) — Pro extension layered on top of the base CMS Kit. Reach for it when the open-source feature list is insufficient and the solution holds a Team-or-higher license.
- GlobalFeatureManager (Volo.Abp.GlobalFeatures) — feature gate that turns CMS-Kit and CMS-Kit-Pro sub-systems on/off at the application level. Reach for it in GlobalFeatureConfigurator to cherry-pick only the sub-systems you ship.
- DocsWebModule / DocsApplicationModule / DocsDomainModule / DocsEntityFrameworkCoreModule (Volo.Docs.*) — module classes for the open-source Docs module. Reach for them when adding versioned, multi-language, multi-source documentation to an ABP host.
- IDocumentStore (Volo.Docs.Documents) — abstraction for document sources; concrete implementations include FileSystemDocumentStore, GitHubDocumentStore, and a database-backed store. Reach for it when registering or implementing a new document source provider.
- DocsProjects table & DocumentStoreType column — runtime project registry where each project's store type and ExtraProperties JSON pick the storage backend per project. Reach for it when seeding documentation projects or switching backends.
- FileManagementApplicationModule / FileManagementDomainModule / FileManagementEntityFrameworkCoreModule / FileManagementHttpApiModule / FileManagementBlazorModule / FileManagementBlazorWebAssemblyModule / FileManagementBlazorServerModule (Volo.FileManagement.*) — Pro-licensed modules for hierarchical file/folder management. Reach for them when you need a Dropbox-style explorer per tenant.
- FileManagementContainer (Volo.FileManagement) — typed BLOB-storing container key; configure its provider (UseDatabase/UseFileSystem/UseAzure/UseAws) inside AbpBlobStoringOptions. Reach for it when choosing where uploaded bytes live.
- IFileDescriptorAppService / IDirectoryDescriptorAppService — primary app-service contracts for file and folder CRUD/move/share. Reach for them from custom UIs or REST clients.
- FileManagementPermissions / FileManagementFeatures — permission and feature definition classes; FileManagementFeatures includes per-tenant max-storage size and module-enable toggles.
- FormsWebModule / FormsApplicationModule / FormsDomainModule / FormsEntityFrameworkCoreModule (Volo.Forms.*) — Pro-licensed modules for questionnaires.
- FormApplicationService / QuestionAppService / ResponseAppService — service triad covering form CRUD, question authoring, and submission/export.
- Form / QuestionBase / FormResponse aggregates (Volo.Forms.Domain) — DDD aggregates: Form is the root, QuestionBase is dependent on Form via FormId, FormResponse stores submitted answers.
- IFormRepository / IQuestionRepository / IChoiceRepository / IResponseRepository — repository contracts for direct domain-layer access.
- FormsPermissions — permission constants for form management UI/API access.
- Per-module connection string keys — CmsKit, Docs, FileManagement, and Forms each look up their own ConnectionStrings entry first, falling back to Default. Reach for them when isolating module schemas in separate databases.
- Per-module DbContext extension methods — modelBuilder.ConfigureCmsKit(), .ConfigureCmsKitPro(), .ConfigureDocs(), .ConfigureFileManagement(), .ConfigureForms() inside OnModelCreating; called once each in the host DbContext to register module entities.

## Configuration pattern

All four modules follow the standard ABP module integration pattern: (1) install via 'abp add-module Volo.<Module>' (skip-db-migrations recommended), (2) add the per-layer [DependsOn] attribute in the matching host AbpModule (e.g., CmsKitWebModule in your *WebModule, CmsKitApplicationModule in your *ApplicationModule, CmsKitEntityFrameworkCoreModule in your *EntityFrameworkCoreModule, plus *DomainModule/*DomainSharedModule/*HttpApiModule/*HttpApiClientModule equivalents), (3) call modelBuilder.ConfigureXxx() in your DbContext.OnModelCreating, (4) add a migration and update the database, (5) for CMS Kit and CMS Kit Pro additionally call GlobalFeatureManager.Instance.Modules.CmsKit(...) (and .CmsKitPro(...)) in a GlobalFeatureConfigurator to enable sub-systems, (6) for File Management configure AbpBlobStoringOptions to bind FileManagementContainer to a concrete provider (Database/FileSystem/Azure/AWS/Google). Example minimal Web-layer wiring:

[DependsOn(
    typeof(CmsKitWebModule),
    typeof(CmsKitProWebModule),
    typeof(DocsWebModule),
    typeof(FileManagementBlazorModule),
    typeof(FormsWebModule)
)]
public class MyAppWebModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        Configure<AbpBlobStoringOptions>(options =>
        {
            options.Containers.Configure<FileManagementContainer>(c =>
            {
                c.UseDatabase();
            });
        });
    }
}

GlobalFeature toggles are typically registered in a static OneTimeRunner-style configurator:

public static class GlobalFeatureConfigurator
{
    public static void Configure()
    {
        OneTimeRunner.Run(() =>
        {
            GlobalFeatureManager.Instance.Modules.CmsKit(cmsKit => cmsKit.EnableAll());
            GlobalFeatureManager.Instance.Modules.CmsKitPro(cmsKitPro => cmsKitPro.EnableAll());
        });
    }
}

DbContext wiring (host EF Core context):

protected override void OnModelCreating(ModelBuilder builder)
{
    base.OnModelCreating(builder);
    builder.ConfigureCmsKit();
    builder.ConfigureCmsKitPro();
    builder.ConfigureDocs();
    builder.ConfigureFileManagement();
    builder.ConfigureForms();
}

Connection strings can be overridden per module in appsettings.json under ConnectionStrings: { 'CmsKit': '...', 'Docs': '...', 'FileManagement': '...', 'Forms': '...' }; otherwise all four resolve to 'Default'.

## Code examples

### Install CMS Kit (and CMS Kit Pro) into an existing solution

_Adding the open-source CMS Kit, optionally extended with CMS Kit Pro, to an existing ABP 10.3 layered solution._

```bash
# At the solution root
abp add-module Volo.CmsKit --skip-db-migrations

# Pro license required for the next line
abp add-module Volo.CmsKit.Pro --skip-db-migrations

# After running, add an EF Core migration that includes the new Cms* tables
dotnet ef migrations add Added_CmsKit -p src/MyApp.EntityFrameworkCore -s src/MyApp.DbMigrator
dotnet run --project src/MyApp.DbMigrator
```

**Key lines:** 'abp add-module' patches every layer's csproj and *Module.cs DependsOn list automatically; '--skip-db-migrations' lets you batch all migrations in a single follow-up 'ef migrations add'. CMS Kit Pro depends on (and does not replace) the base Volo.CmsKit.* packages — both must be installed.

### Enable CMS Kit sub-systems via GlobalFeatureManager

_Selectively turning on only the page, blog, and comment sub-systems of CMS Kit while leaving reactions/ratings/tags off._

```csharp
using Volo.Abp.GlobalFeatures;
using Volo.Abp.Threading;
using Volo.CmsKit.GlobalFeatures;

public static class MyAppGlobalFeatureConfigurator
{
    private static readonly OneTimeRunner OneTimeRunner = new OneTimeRunner();

    public static void Configure()
    {
        OneTimeRunner.Run(() =>
        {
            GlobalFeatureManager.Instance.Modules.CmsKit(cmsKit =>
            {
                cmsKit.Pages.Enable();
                cmsKit.Blogs.Enable();
                cmsKit.Comments.Enable();
                // Reactions, Ratings, Tags, Menus, MarkedItems, GlobalResources stay disabled
            });
        });
    }
}

// Call MyAppGlobalFeatureConfigurator.Configure() before context.Services.AddAbpDbContext...
// Typically invoked from your *DomainSharedModule.PreConfigureServices.
```

**Key lines:** OneTimeRunner ensures GlobalFeatureManager state is configured exactly once per process. Sub-systems are independent — features you do not Enable() stay off and their tables/endpoints remain inactive even though the migration creates them.

### Wire the Docs module DbContext and Web layer

_Adding versioned documentation backed by GitHub to a layered ABP solution._

```csharp
using Volo.Docs;
using Volo.Docs.EntityFrameworkCore;

[DependsOn(
    typeof(DocsWebModule),
    typeof(DocsApplicationModule),
    typeof(DocsHttpApiModule),
    typeof(DocsEntityFrameworkCoreModule)
)]
public class MyAppWebModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        Configure<AbpRouterOptions>(options =>
        {
            // Docs UI is mounted at /Documents by default
        });
    }
}

// In MyAppDbContext.cs
protected override void OnModelCreating(ModelBuilder builder)
{
    base.OnModelCreating(builder);
    builder.ConfigureDocs();
}

// Seed a project row to point at GitHub
await _projectRepository.InsertAsync(new Project(
    id: GuidGenerator.Create(),
    name: "MyApp Docs",
    shortName: "myapp",
    documentStoreType: "GitHub",   // FileSystemDocumentStore.Type or GithubDocumentSource.Type
    format: "md",
    defaultDocumentName: "index.md",
    navigationDocumentName: "docs-nav.json")
{
    ExtraProperties = { ["GitHubRootUrl"] = "https://github.com/myorg/myapp/tree/{version}/docs/{languageCode}" }
});
```

**Key lines:** DocumentStoreType is a free-form string — 'GitHub', 'FileSystem', or a custom IDocumentStore.Type — and ExtraProperties carries per-store config (e.g., GitHubRootUrl with {version}/{languageCode} placeholders). builder.ConfigureDocs() is a one-liner that registers all DocsProjects/DocsDocuments tables.

### Configure File Management BLOB container

_Storing File Management uploads in Azure Blob Storage instead of the database._

```csharp
using Volo.Abp.BlobStoring;
using Volo.Abp.BlobStoring.Azure;
using Volo.FileManagement;

[DependsOn(
    typeof(FileManagementApplicationModule),
    typeof(FileManagementHttpApiModule),
    typeof(FileManagementEntityFrameworkCoreModule),
    typeof(FileManagementBlazorServerModule),
    typeof(AbpBlobStoringAzureModule)
)]
public class MyAppWebModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        Configure<AbpBlobStoringOptions>(options =>
        {
            options.Containers.Configure<FileManagementContainer>(container =>
            {
                container.UseAzure(azure =>
                {
                    azure.ConnectionString = context.Services
                        .GetConfiguration()["Azure:Storage:ConnectionString"];
                    azure.ContainerName = "file-management";
                    azure.CreateContainerIfNotExists = true;
                });
            });
        });
    }
}

// In MyAppDbContext.cs
builder.ConfigureFileManagement();
```

**Key lines:** FileManagementContainer is the typed key — switching providers (UseDatabase/UseFileSystem/UseAzure/UseAws/UseAliyun) is a one-line change with no domain-code impact. CreateContainerIfNotExists avoids a manual provisioning step in dev.

### Add the Forms module and consume FormApplicationService

_Building a custom Razor Page that lists forms and submits responses programmatically._

```csharp
using Volo.Forms;

[DependsOn(
    typeof(FormsWebModule),
    typeof(FormsApplicationModule),
    typeof(FormsHttpApiModule),
    typeof(FormsEntityFrameworkCoreModule)
)]
public class MyAppWebModule : AbpModule { }

// In MyAppDbContext.cs
builder.ConfigureForms();

// Consuming the form services from a custom page model
public class SurveyPageModel : AbpPageModel
{
    private readonly IFormAppService _formAppService;
    private readonly IResponseAppService _responseAppService;

    public SurveyPageModel(
        IFormAppService formAppService,
        IResponseAppService responseAppService)
    {
        _formAppService = formAppService;
        _responseAppService = responseAppService;
    }

    public async Task<IActionResult> OnPostAsync(Guid formId, List<AnswerCreateDto> answers)
    {
        await _responseAppService.CreateAsync(formId, new ResponseCreateDto
        {
            Answers = answers
        });
        return RedirectToPage("./Thanks");
    }
}
```

**Key lines:** IFormAppService/IResponseAppService are the public surface — the underlying Form, QuestionBase, and FormResponse aggregates are kept behind repositories. CSV export is exposed on ResponseAppService and surfaced in the built-in /Forms admin UI.

## Common mistakes

### Calling 'abp add-module Volo.CmsKit.Pro' without first installing the open-source Volo.CmsKit package, then puzzling over missing Cms* tables.

**Why wrong:** CMS Kit Pro layers on top of CMS Kit — it does not replace it. Without the base packages and their migrations, Pro entities reference missing FK targets and DI registration is incomplete.

```bash
Install both: 'abp add-module Volo.CmsKit' followed by 'abp add-module Volo.CmsKit.Pro', then run a single 'ef migrations add' that picks up both schemas.
```

### Forgetting to call GlobalFeatureManager.Instance.Modules.CmsKit(...).Enable() and assuming sub-systems are on by default.

**Why wrong:** CMS Kit ships with all sub-systems disabled by default; controllers/pages/menu items are silently absent until the matching GlobalFeature is enabled before module configuration runs.

```csharp
Call GlobalFeatureConfigurator.Configure() (wrapping a OneTimeRunner) from PreConfigureServices in your *DomainSharedModule, and either Enable() each sub-system or call EnableAll().
```

### Configuring AbpBlobStoringOptions for the default container only and assuming File Management uploads will land there.

**Why wrong:** File Management binds to its own typed container (FileManagementContainer). Default-container settings are ignored, and uploads silently use the in-memory or DB fallback depending on what was last registered.

```csharp
Always configure the FileManagementContainer explicitly: options.Containers.Configure<FileManagementContainer>(c => c.UseAzure(...)).
```

### Adding builder.ConfigureDocs() (or ConfigureCmsKit, ConfigureFileManagement, ConfigureForms) to multiple DbContexts in the same solution.

**Why wrong:** ABP routes module entities to the host context that registered them; calling the extension from two contexts produces duplicate-mapping errors at startup or, worse, two parallel schemas.

```csharp
Call each builder.ConfigureXxx() exactly once, in the consolidated host DbContext (typically *DbContext in the EntityFrameworkCore project).
```

### Trying to use the File Management or Forms module on a Free / Community license.

**Why wrong:** Both modules ship as Pro; their NuGet feed requires an ABP Team-or-higher license token and 'abp add-module' will fail authentication.

```csharp
Either upgrade the license, or pick the open-source CMS Kit (which covers blogs, comments, ratings, reactions) and a custom file/form solution.
```

### Pointing the Docs module's GitHub document source at a private repo without an access token.

**Why wrong:** GitHubDocumentStore uses the GitHub REST API; requests to a private repo return 404 anonymously, and the Docs UI shows an empty navigation.

```csharp
Set the project's ExtraProperties JSON with 'GitHubAccessToken' (or organization-level secret) and ensure the token has 'repo' read scope; rotate it via the DocsProjects table if it expires.
```

### Letting Forms responses accumulate without cleanup and exporting via the UI's CSV button each time.

**Why wrong:** The export endpoint streams every FormResponse for the form; with anonymous public forms, response counts can grow into the millions and time out the export.

```csharp
Use IResponseRepository directly with paging, or schedule a background job that exports in chunks; gate anonymous submissions behind a CAPTCHA and, where possible, a feature flag per tenant.
```

## Version pins (ABP 10.3)

- ABP 10.3 — CMS Kit, CMS Kit Pro, Docs, File Management, and Forms modules all ship as part of the 10.3 train; package versions track the framework version (Volo.CmsKit 10.3.x, Volo.Docs 10.3.x, etc.).
- ABP 10.3 — File Management and Forms modules support MVC/Razor Pages and Blazor (Server, WebAssembly, and Blazor Web App via dedicated *.Blazor.Server / *.Blazor.WebAssembly packages); Angular and React Native UIs are not officially shipped for these two modules. [uncertain]
- ABP 10.3 — CMS Kit Pro UI is shipped for MVC/Razor Pages and Blazor only; Angular UI for CMS Kit Pro is not provided. [uncertain]
- ABP 10.3 — Docs module dependencies include Volo.Docs.Domain, Volo.Docs.EntityFrameworkCore, Volo.Docs.Application, Volo.Docs.Web; legacy MongoDB packaging may still be available but the canonical 10.3 path is EF Core.
- ABP 10.3 — GlobalFeatureManager.Instance.Modules.CmsKit(...) and CmsKitPro(...) are the recommended entry points; older Configure<CmsKitOptions> patterns from earlier versions have been superseded. [uncertain]
- ABP 10.3 — FileManagementContainer name and FileManagementFeatures.MaxStoreSizeAsMegabytes feature key are stable in 10.3 but were renamed in earlier minor versions; do not assume identical names below 10.0.
- ABP 10.3 — Per-module ConnectionStringName attributes are 'CmsKit', 'Docs', 'FileManagement', and 'Forms'; missing keys fall back to 'Default' (this fallback behavior is framework-wide and unlikely to change).
- Breaking-change watch — the Docs module's IDocumentStore signature and GitHub-store ExtraProperties keys have shifted across major versions; pin DocumentStoreType strings to the 10.3 constants exposed by each store class. [uncertain]

## Cross-references

**Phase 1 references:**
- [references/infrastructure/blob-storing.md](../infrastructure/blob-storing.md) — File Management depends on the BLOB Storing system; cross-link whenever FileManagementContainer is configured.
- [references/modules/identity-and-auth.md](../modules/identity-and-auth.md) — All four modules use ABP authorization; CMS Kit Pro newsletter and Forms anonymous-submission decisions interact with account/identity policy.
- [references/framework/architecture-modularity.md](../framework/architecture-modularity.md) — CMS Kit and CMS Kit Pro sub-systems are gated by GlobalFeatureManager; trigger when discussing how to enable/disable individual features.
- [references/framework/multi-tenancy.md](../framework/multi-tenancy.md) — File Management ties storage quotas to tenants via FileManagementFeatures; CMS Kit content can be tenant-scoped.
- [references/framework/data-ef-core.md](../framework/data-ef-core.md) — Each module exposes a builder.ConfigureXxx() extension and a per-module ConnectionStringName; cross-link from any DbContext wiring discussion.

**External docs:**
- [Markdig (Markdown processor)](https://github.com/xoofx/markdig) — Docs module renders Markdown via Markdig; relevant when customizing Markdown rendering or extensions.
- [Scriban templating engine](https://github.com/scriban/scriban) — Docs module uses Scriban for conditional sections inside Markdown; needed when authoring version-conditional documentation.
- [Elasticsearch .NET client](https://www.elastic.co/guide/en/elasticsearch/client/net-api/current/index.html) — Docs module's full-text search backs onto Elasticsearch; referenced when configuring the search provider.
- [Azure Blob Storage](https://learn.microsoft.com/azure/storage/blobs/) — Common backend for FileManagementContainer via AbpBlobStoringAzureModule.
- [AWS S3](https://docs.aws.amazon.com/s3/) — Alternative backend for FileManagementContainer via AbpBlobStoringAwsModule.
- [GitHub REST API](https://docs.github.com/en/rest) — Used by the Docs module's GitHub document store provider to fetch versioned Markdown.
- [EF Core OnModelCreating](https://learn.microsoft.com/ef/core/modeling/) — All four modules require a single builder.ConfigureXxx() call inside the host DbContext's OnModelCreating override.

## Sources (ABP 10.3)

- [https://abp.io/docs/10.3/modules/cms-kit](https://abp.io/docs/10.3/modules/cms-kit)
- [https://abp.io/docs/10.3/modules/cms-kit-pro](https://abp.io/docs/10.3/modules/cms-kit-pro)
- [https://abp.io/docs/10.3/modules/docs](https://abp.io/docs/10.3/modules/docs)
- [https://abp.io/docs/10.3/modules/file-management](https://abp.io/docs/10.3/modules/file-management)
- [https://abp.io/docs/10.3/modules/forms](https://abp.io/docs/10.3/modules/forms)
- [https://github.com/abpframework/cms-kit-demo](https://github.com/abpframework/cms-kit-demo)

Last verified: 2026-05-10
