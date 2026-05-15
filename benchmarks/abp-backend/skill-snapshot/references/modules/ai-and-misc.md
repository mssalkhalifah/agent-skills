# Modules — AI / GDPR / Elsa / Language Management / Virtual File Explorer

> Five auxiliary ABP modules covering AI workspace management (AI Management), GDPR compliance (data export/deletion/cookie consent), workflow automation (Elsa Pro), runtime translation management (Language Management), and virtual file system browsing (Virtual File Explorer).

## When to load this reference

- How do I integrate ChatGPT/OpenAI/Ollama into an ABP application?
- How do I let users configure AI workspaces dynamically (model, API key, system prompt) without redeploying?
- How do I add Retrieval-Augmented Generation (RAG) over uploaded PDFs/Markdown using ABP?
- How do I expose an OpenAI-compatible /v1 chat completions API from my ABP app?
- How do I let users download or delete their personal data to satisfy GDPR right-to-access / right-to-erasure?
- How do I add a cookie consent banner in ABP?
- How do I collect personal data from multiple modules into one GDPR export?
- How do I add visual workflow design (Elsa Studio) to an ABP solution and authenticate it with ABP Identity?
- How do I add or edit translations at runtime through an admin UI instead of editing JSON localization files?
- How do I create a new system language (e.g., add Japanese) and seed translations during DbMigrator?
- How do I browse the contents of the embedded virtual file system (e.g., to debug missing themes/layouts)?
- What is the difference between system workspaces (PreConfigure) and dynamic workspaces (database-stored) in AI Management?
- How do I subscribe to GdprUserDataRequestedEto to contribute data from my custom module to a GDPR export?

**Audience:** ABP developers (Team license or higher) building AI-enabled features, satisfying GDPR/privacy requirements, automating processes with workflows, supporting multilingual UIs through admin-managed translations, or debugging the embedded virtual file system.

## Key concepts

- ApplicationWorkspaceManager (Volo.AIManagement.Workspaces) - Domain manager for creating and updating AI workspaces; reach for it in data seeders or admin services to create workspaces programmatically.
- IWorkspaceRepository (Volo.AIManagement.Workspaces) - Persistence abstraction for Workspace aggregate root; use to insert/update/delete dynamic workspaces.
- IChatClient<TWorkspace> (Volo.AIManagement.ChatClients) - Strongly-typed chat-completion client bound to a specific workspace; inject it into application services to call the configured LLM.
- IAIChatClientProviderFactory (Volo.AIManagement.Providers) - Extension point for adding custom AI providers beyond built-in OpenAI/Ollama; implement and register to support additional LLM backends.
- IDocumentSearchService (Volo.AIManagement.DataSources) - Retrieves the top-K relevant document chunks for a query; called automatically when a workspace has an embedder configured.
- DocumentProcessingManager (Volo.AIManagement.DataSources) - Orchestrates document chunking and embedding generation via a background job (IndexDocumentJob).
- IVectorStoreProvider (Volo.AIManagement.VectorStores) - Abstraction over MongoDB, PostgreSQL pgvector, and Qdrant vector backends for RAG.
- IChatCompletionClientAppService (Volo.AIManagement) - HTTP application service backing the OpenAI-compatible /v1/chat/completions endpoint.
- IEmbeddingClientAppService (Volo.AIManagement) - HTTP application service for the /v1/embeddings endpoint, used by external SDKs.
- AbpAIWorkspaceOptions (Volo.AIManagement) - PreConfigure-time options for declaring system (code-defined) workspaces that are auto-created on startup and read-only by default.
- WorkspaceDataSourceOptions (Volo.AIManagement.DataSources) - Configures allowed file extensions, max file size, chunking parameters for RAG document uploads.
- GdprRequest (Volo.Gdpr) - Aggregate root representing a user's personal-data export request; carries UserId, ReadyTime, and a collection of GdprInfo entries.
- GdprInfo (Volo.Gdpr) - Entity holding serialized personal data contributed by one module (Provider field identifies the source).
- IGdprRequestRepository (Volo.Gdpr) - Repository for GdprRequest aggregate.
- GdprRequestAppService (Volo.Gdpr) - Application service backing the personal-data UI page (create request, poll status, download JSON, request deletion).
- GdprUserDataRequestedEto (Volo.Gdpr) - Distributed event published when a user requests data export; modules subscribe to contribute their data.
- GdprUserDataPreparedEto (Volo.Gdpr) - Distributed event signaling all collectors have responded and the JSON aggregate is ready.
- GdprUserDataDeletionRequestedEto (Volo.Gdpr) - Distributed event triggering permanent deletion of the user's data across all subscribing modules.
- AbpGdprOptions (Volo.Gdpr) - Configures RequestTimeInterval (rate-limit between requests) and MinutesForDataPreparation (timeout for collectors).
- AbpCookieConsentOptions (Volo.Abp.AspNetCore.Mvc.UI.Cookies) - Configures the cookie-consent banner: IsEnabled, CookiePolicyUrl, PrivacyPolicyUrl, Expiration.
- UseAbpCookieConsent middleware (Volo.Abp.AspNetCore.Mvc.UI.Cookies) - ASP.NET Core middleware that injects the consent banner into responses.
- AbpElsaAspNetCoreModule (Volo.Abp.Elsa.AspNetCore) - Bridges Elsa's authentication to ASP.NET Core pipeline.
- AbpElsaIdentityModule (Volo.Abp.Elsa.Identity) - Adapts Elsa's user/role checks to ABP Identity (so Elsa Studio logs in with admin/1q2w3E*).
- AbpElsaApplicationModule (Volo.Abp.Elsa.Application) - Declares Elsa permission definitions inside ABP's permission system.
- Language (Volo.Abp.LanguageManagement) - Aggregate root representing an enabled/disabled language with culture name, display name, flag icon.
- LanguageText (Volo.Abp.LanguageManagement) - Entity holding a single translated text for a (resource, culture, key) tuple; overrides static JSON resources at runtime.
- LocalizationResourceRecord / LocalizationTextRecord (Volo.Abp.LanguageManagement) - Database-backed equivalents of code-declared LocalizationResource and texts, enabling runtime resource creation.
- ILanguageAppService / ILanguageTextAppService (Volo.Abp.LanguageManagement) - CRUD APIs for languages and text entries, consumed by the admin UI.
- ILanguageRepository, ILanguageTextRepository, ILocalizationResourceRecordRepository, ILocalizationTextRecordRepository (Volo.Abp.LanguageManagement) - Repository interfaces for the four entities above.
- LanguageEto / LanguageTextEto (Volo.Abp.LanguageManagement) - Distributed ETOs for cache invalidation across nodes/microservices when languages or texts change.
- AbpVirtualFileExplorerOptions (Volo.Abp.VirtualFileExplorer) - PreConfigure option exposing IsEnabled to disable the explorer UI in production.
- AbpVirtualFileExplorerWebModule (Volo.Abp.VirtualFileExplorer.Web) - Module to depend on from your Web layer to mount the /VirtualFileExplorer page.

## Configuration pattern

AI Management - register provider package(s) and configure system workspaces and RAG options:

```csharp
[DependsOn(
    typeof(AIManagementApplicationModule),
    typeof(AIManagementHttpApiModule),
    typeof(AIManagementWebModule),
    typeof(AIManagementOpenAIModule),
    typeof(AIManagementVectorStoresMongoDbModule)
)]
public class MyAppWebModule : AbpModule
{
    public override void PreConfigureServices(ServiceConfigurationContext context)
    {
        PreConfigure<AbpAIWorkspaceOptions>(options =>
        {
            options.AddSystem<MyChatWorkspace>("MyChat", w =>
            {
                w.Provider = "OpenAI";
                w.ModelName = "gpt-4o-mini";
                w.ApiKey = context.Services.GetConfiguration()["OpenAI:ApiKey"];
                w.SystemPrompt = "You are a helpful support agent.";
            });
        });
    }

    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        Configure<WorkspaceDataSourceOptions>(options =>
        {
            options.AllowedFileExtensions = new[] { ".txt", ".md", ".pdf" };
            options.MaxFileSize = 10 * 1024 * 1024;
        });
    }
}
```

GDPR - configure request rate-limit and cookie consent:

```csharp
public override void ConfigureServices(ServiceConfigurationContext context)
{
    Configure<AbpGdprOptions>(options =>
    {
        options.RequestTimeInterval = TimeSpan.FromDays(1);
        options.MinutesForDataPreparation = 60;
    });

    context.Services.AddAbpCookieConsent(options =>
    {
        options.IsEnabled = true;
        options.CookiePolicyUrl = "/CookiePolicy";
        options.PrivacyPolicyUrl = "/PrivacyPolicy";
        options.Expiration = TimeSpan.FromDays(180);
    });
}

public override void OnApplicationInitialization(ApplicationInitializationContext context)
{
    var app = context.GetApplicationBuilder();
    // ... other middleware
    app.UseAbpCookieConsent();
}
```

Elsa Pro - register Elsa with ABP Identity bridge in ConfigureServices:

```csharp
private void ConfigureElsa(ServiceConfigurationContext context, IConfiguration configuration)
{
    var connectionString = configuration.GetConnectionString("Default")!;
    context.Services.AddElsa(elsa => elsa
        .UseAbpIdentity(identity =>
        {
            identity.TokenOptions = options =>
                options.SigningKey = configuration["Elsa:SigningKey"];
        })
        .UseWorkflowManagement(management =>
            management.UseEntityFrameworkCore(ef => ef.UseSqlServer(connectionString)))
        .UseWorkflowRuntime(runtime =>
            runtime.UseEntityFrameworkCore(ef => ef.UseSqlServer(connectionString)))
        .UseHttp()
        .UseScheduling()
        .UseJavaScript()
        .UseLiquid());
}
```

Language Management - typically pre-installed; ensure desired languages are seeded via AbpLocalizationOptions and that DbMigrator runs:

```csharp
Configure<AbpLocalizationOptions>(options =>
{
    options.Languages.Add(new LanguageInfo("en", "en", "English"));
    options.Languages.Add(new LanguageInfo("ar", "ar", "العربية"));
    options.Languages.Add(new LanguageInfo("tr", "tr", "Türkçe"));
});
```

Virtual File Explorer - depend on the Web module and optionally disable in production:

```csharp
[DependsOn(typeof(AbpVirtualFileExplorerWebModule))]
public class MyAppWebModule : AbpModule
{
    public override void PreConfigureServices(ServiceConfigurationContext context)
    {
        var env = context.Services.GetHostingEnvironment();
        PreConfigure<AbpVirtualFileExplorerOptions>(options =>
        {
            options.IsEnabled = env.IsDevelopment();
        });
    }
}
```

## Code examples

### AI Management - Seed a dynamic workspace and call its chat client

_Create a 'SupportAssistant' OpenAI workspace at startup, then inject IChatClient<MyWorkspace> in an application service to get completions._

```csharp
public class MyWorkspaceSeeder : IDataSeedContributor, ITransientDependency
{
    private readonly ApplicationWorkspaceManager _manager;
    private readonly IWorkspaceRepository _repository;

    public MyWorkspaceSeeder(ApplicationWorkspaceManager manager, IWorkspaceRepository repository)
    { _manager = manager; _repository = repository; }

    public async Task SeedAsync(DataSeedContext context)
    {
        var workspace = await _manager.CreateAsync(
            name: "SupportAssistant",
            provider: "OpenAI",
            modelName: "gpt-4o-mini");
        workspace.ApiKey = Environment.GetEnvironmentVariable("OPENAI_API_KEY");
        workspace.SystemPrompt = "You are a helpful support agent.";
        workspace.Temperature = 0.7m;
        workspace.EmbedderProvider = "OpenAI";
        workspace.EmbedderModelName = "text-embedding-3-small";
        workspace.VectorStoreProvider = "MongoDb";
        await _repository.InsertAsync(workspace);
    }
}

public class TicketAppService : ApplicationService
{
    private readonly IChatClient<SupportAssistantWorkspace> _chat;
    public TicketAppService(IChatClient<SupportAssistantWorkspace> chat) { _chat = chat; }

    public async Task<string> AskAsync(string question)
    {
        var response = await _chat.CompleteAsync(new[]
        {
            new ChatMessage(ChatRole.User, question)
        });
        return response.Message.Content;
    }
}
```

**Key lines:** ApplicationWorkspaceManager.CreateAsync constructs a Workspace aggregate; IChatClient<TWorkspace> resolves the provider/model/embedder from that workspace's persisted config so swapping the model is a database change, not a code change.

### GDPR - Subscribe to data-collection ETO from a custom module

_When a user requests their personal data, contribute records from a 'Books' module into the export._

```csharp
public class BooksGdprDataCollector :
    IDistributedEventHandler<GdprUserDataRequestedEto>,
    IDistributedEventHandler<GdprUserDataDeletionRequestedEto>,
    ITransientDependency
{
    private readonly IRepository<Book, Guid> _books;
    private readonly IDistributedEventBus _eventBus;

    public BooksGdprDataCollector(IRepository<Book, Guid> books, IDistributedEventBus eventBus)
    { _books = books; _eventBus = eventBus; }

    public async Task HandleEventAsync(GdprUserDataRequestedEto e)
    {
        var myBooks = await _books.GetListAsync(b => b.AuthorId == e.UserId);
        await _eventBus.PublishAsync(new GdprUserDataPreparedEto
        {
            RequestId = e.RequestId,
            Provider = "Books",
            Data = JsonSerializer.Serialize(myBooks)
        });
    }

    public async Task HandleEventAsync(GdprUserDataDeletionRequestedEto e)
    {
        await _books.DeleteAsync(b => b.AuthorId == e.UserId);
    }
}
```

**Key lines:** Each contributing module handles GdprUserDataRequestedEto independently and publishes a GdprUserDataPreparedEto with its Provider tag; the GDPR module aggregates results into the downloadable JSON. The deletion handler ensures right-to-erasure across the whole solution.

### Elsa Pro - Register Elsa with ABP Identity and SQL Server persistence

_Wire Elsa Workflows into an ABP Web project so admins log into Elsa Studio with their ABP Identity credentials._

```csharp
public override void ConfigureServices(ServiceConfigurationContext context)
{
    var configuration = context.Services.GetConfiguration();
    var connectionString = configuration.GetConnectionString("Default")!;

    context.Services.AddElsa(elsa => elsa
        .UseAbpIdentity(identity =>
        {
            identity.TokenOptions = options =>
                options.SigningKey = configuration["Elsa:SigningKey"]
                    ?? throw new InvalidOperationException("Set Elsa:SigningKey");
        })
        .UseWorkflowManagement(m => m.UseEntityFrameworkCore(ef => ef.UseSqlServer(connectionString)))
        .UseWorkflowRuntime(r => r.UseEntityFrameworkCore(ef => ef.UseSqlServer(connectionString)))
        .UseHttp()
        .UseScheduling()
        .UseJavaScript()
        .UseLiquid());
}
```

**Key lines:** UseAbpIdentity makes Elsa Studio's password flow validate against AbpUsers; UseWorkflowManagement/UseWorkflowRuntime persist workflow definitions and instances to the same SQL database (Elsa creates its own tables on first run, so no ABP migration is required).

### Language Management - Add a language and override a text at runtime

_Add Arabic via the admin UI equivalent in code, then override the 'Welcome' translation for the AbpUi resource._

```csharp
public class ArabicSeeder : IDataSeedContributor, ITransientDependency
{
    private readonly ILanguageAppService _languages;
    private readonly ILanguageTextAppService _texts;

    public ArabicSeeder(ILanguageAppService languages, ILanguageTextAppService texts)
    { _languages = languages; _texts = texts; }

    public async Task SeedAsync(DataSeedContext context)
    {
        await _languages.CreateAsync(new CreateLanguageDto
        {
            CultureName = "ar",
            UiCultureName = "ar",
            DisplayName = "العربية",
            FlagIcon = "sa",
            IsEnabled = true
        });

        await _texts.UpdateAsync(
            resourceName: "AbpUi",
            cultureName: "ar",
            name: "Welcome",
            input: new LanguageTextUpdateDto { Value = "أهلاً وسهلاً" });
    }
}
```

**Key lines:** Languages added through ILanguageAppService persist as Language rows and immediately participate in ABP's localization pipeline; LanguageText rows take precedence over JSON resource files, so admin overrides do not require redeployment.

### Virtual File Explorer - Mount the explorer in development only

_Enable /VirtualFileExplorer in dev environments so theme/layout files inside embedded resources can be inspected, but disable it in production._

```csharp
[DependsOn(typeof(AbpVirtualFileExplorerWebModule))]
public class MyAppWebModule : AbpModule
{
    public override void PreConfigureServices(ServiceConfigurationContext context)
    {
        var env = context.Services.GetHostingEnvironment();
        PreConfigure<AbpVirtualFileExplorerOptions>(options =>
        {
            options.IsEnabled = env.IsDevelopment();
        });
    }

    public override void OnApplicationInitialization(ApplicationInitializationContext context)
    {
        var app = context.GetApplicationBuilder();
        app.UseConfiguredEndpoints();
        // Page is reachable at /VirtualFileExplorer when IsEnabled=true.
    }
}
```

**Key lines:** AbpVirtualFileExplorerOptions.IsEnabled gates the route at startup; dependency on AbpVirtualFileExplorerWebModule registers the Razor Pages and JavaScript assets (after install-libs has copied @abp/virtual-file-explorer).

## Common mistakes

### Configuring an AI workspace with an embedder model but no vector store (or vice versa).

**Why wrong:** RAG requires both halves: the embedder turns chunks into vectors, and the vector store persists/queries them. Setting only one leaves IDocumentSearchService unable to operate, and uploaded documents stay flagged IsProcessed=false.

```csharp
Always set EmbedderProvider, EmbedderModelName, and VectorStoreProvider together (e.g., OpenAI + text-embedding-3-small + MongoDb). Verify by uploading a document and confirming WorkspaceDataSource.IsProcessed becomes true.
```

### Treating system workspaces (declared via PreConfigure<AbpAIWorkspaceOptions>) as editable from the UI.

**Why wrong:** System workspaces are read-only by design; UI edits are silently discarded unless OverrideSystemConfiguration is enabled, which leads admins to think the change took effect.

```csharp
Either enable OverrideSystemConfiguration explicitly when allowing UI overrides, or model user-editable assistants as dynamic workspaces created through ApplicationWorkspaceManager.CreateAsync and persisted via IWorkspaceRepository.
```

### Storing AI provider API keys in appsettings.json committed to source control.

**Why wrong:** OpenAI/Azure OpenAI keys are bearer credentials with billing impact; leaking them into git history means rotating keys and possibly absorbing fraudulent usage.

```bash
Use environment variables, dotnet user-secrets in development, and a secrets manager (Azure Key Vault, AWS Secrets Manager) in production. Inject via IConfiguration in the seeder/manager that creates the workspace.
```

### Forgetting to register a GDPR data collector for a module that holds personal data.

**Why wrong:** GdprUserDataRequestedEto is a fan-out event - only modules that explicitly subscribe contribute data. A non-subscribing module silently omits its records, leaving the export incomplete and the deletion incomplete (which is itself a GDPR violation).

```csharp
For every module that stores user-linked data, implement IDistributedEventHandler<GdprUserDataRequestedEto> AND IDistributedEventHandler<GdprUserDataDeletionRequestedEto>. Document the contract in the module README.
```

### Calling app.UseAbpCookieConsent() without registering AddAbpCookieConsent or with IsEnabled=false.

**Why wrong:** The middleware silently no-ops; the banner never renders, and visitors continue tracking without consent. Quick smoke tests in dev pass, but compliance audits fail.

```csharp
Always pair AddAbpCookieConsent (with IsEnabled=true and policy URLs set) with UseAbpCookieConsent in the request pipeline. Verify by clearing cookies and reloading the home page.
```

### Running ABP Identity migrations and assuming Elsa tables will appear too.

**Why wrong:** Elsa maintains its own DbContext separate from AbpDbContext; Elsa creates its tables on first run via its internal migrator, not via ABP's DbMigrator project, so EF migration files for Elsa do not exist in the ABP solution.

```csharp
Let Elsa auto-create its tables on first startup, or pre-provision them with `elsa migrate`. If using a separate database, set its connection string explicitly inside UseElsa(... UseEntityFrameworkCore(...)) instead of reusing AbpDbContext options.
```

### Editing Language Management JSON resource files at runtime instead of using the LanguageText admin UI.

**Why wrong:** JSON files in Localization/* are embedded resources baked into assemblies; runtime edits do nothing. Worse, developers may add a Language row through the UI but expect translations to appear from JSON files that don't exist.

```csharp
Use the Language Texts admin page (or ILanguageTextAppService.UpdateAsync) to override per-(resource, culture, key). LanguageText rows always win over JSON resources.
```

### Disabling a language by setting IsEnabled=false but leaving it as the user's preferred culture.

**Why wrong:** ABP falls back to the default culture, which surprises users who set their preference. Cached UI strings may also show stale translations until cache refresh.

```csharp
When disabling a language, run a one-off update on AbpUserSettings to clear DefaultLanguage for affected users, and clear localization cache (IDistributedCache for AbpLocalizationOptions).
```

### Leaving Virtual File Explorer enabled in production.

**Why wrong:** The explorer exposes the entire embedded virtual file system - including theme files, internal scripts, and any sensitive content stored as embedded resources - over an HTTP endpoint. While it does not expose the host filesystem, it leaks intellectual property and aids attacker reconnaissance.

```csharp
PreConfigure AbpVirtualFileExplorerOptions.IsEnabled = env.IsDevelopment(), or guard the route with an admin permission. Consider not depending on AbpVirtualFileExplorerWebModule from production builds at all.
```

### Not setting Elsa:SigningKey to a strong, stable value.

**Why wrong:** Elsa signs JWTs that authorize Elsa Studio and API calls; a weak key allows token forgery, and a key that changes per deployment invalidates all live sessions.

```csharp
Generate a 256-bit random key, store it as a secret (Key Vault, env var), and inject it into UseAbpIdentity(identity => identity.TokenOptions = o => o.SigningKey = ...). Rotate via standard JWT key-rotation procedures.
```

### Uploading large files to AI Management RAG without raising hosting limits.

**Why wrong:** Even if WorkspaceDataSourceOptions.MaxFileSize allows 100 MB, Kestrel/IIS/nginx default request-body limits (often 30 MB) reject the upload before ABP sees it, producing confusing 413 errors.

```csharp
Align Kestrel's MaxRequestBodySize, IIS's maxAllowedContentLength, and nginx's client_max_body_size with WorkspaceDataSourceOptions.MaxFileSize. Document the matrix in your deployment guide.
```

### Assuming GdprRequest deletion is synchronous.

**Why wrong:** GdprUserDataDeletionRequestedEto is a distributed event; subscribers run asynchronously and can fail silently. Telling a user 'your data has been deleted' immediately may be inaccurate.

```csharp
Communicate that deletion is being processed; rely on AbpGdprOptions.MinutesForDataPreparation as the realistic SLA window; instrument failed handlers via the outbox/inbox pattern and surface failures in audit logs.
```

## Version pins (ABP 10.3)

- ABP 10.3 is the latest stable release at time of writing; 10.4 is in preview. All Volo.AIManagement.* package versions must match the ABP version exactly - mixing 10.3.x and 10.4.x packages causes runtime DI errors.
- AI Management's IChatClient<TWorkspace> generic shape is based on Microsoft.Extensions.AI abstractions, which are still evolving in .NET; the ChatMessage / ChatRole types may shift namespace in ABP 10.4+ as Microsoft.Extensions.AI stabilizes. [uncertain]
- AI Management vector-store providers in 10.3 are MongoDB, PostgreSQL pgvector, and Qdrant; additional providers (e.g., Azure AI Search, Pinecone) are not built-in and require custom IVectorStoreProvider implementations. [uncertain whether 10.4 adds more]
- GDPR module's tables AbpGdprRequests / AbpGdprInfos use the AbpGdpr connection-string key (falling back to Default). Renaming the connection-string key after deployment requires explicit data migration.
- AbpCookieConsentOptions.Expiration default is 180 days; this may not satisfy stricter regional requirements (some EU DPAs recommend 6-13 months max for analytics consent). Configure explicitly per legal review.
- Elsa Pro currently surfaces AbpElsaAspNetCoreModule, AbpElsaIdentityModule, and AbpElsaApplicationModule as concrete modules; Domain, EntityFrameworkCore, HttpApi, and Blazor variants exist as empty shells reserved for future implementation. [uncertain - planned timeline not documented]
- Default Elsa Studio admin credentials are admin / 1q2w3E* in development templates - same as ABP's default admin. Change in production.
- Language Management's LanguageText cache uses AbpDistributedCache; in microservice deployments with separate caches per service, distributed events (LanguageTextEto) are required for invalidation. Without a distributed event bus, translation edits won't propagate.
- Virtual File Explorer's NPM dependency is `@abp/virtual-file-explorer` and must be resynced via `abp install-libs` after add-module; the version constraint in the docs (`^2.9.0`) lags behind the NuGet 10.3 version because the JS package versions independently. [uncertain whether this is reconciled in 10.4]
- Cookie consent middleware order matters: register UseAbpCookieConsent after UseRouting and authentication but before endpoint execution; otherwise the banner injection misses the response.
- Pro modules (AI Management, GDPR, Elsa Pro, Language Management) all require an ABP Team license or higher; package downloads from abp.io require account credentials configured via `abp login`. License checks are at NuGet acquisition time, not at runtime.
- Permission names use module prefixes (AIManagement.Workspaces, LanguageManagement.Languages, GDPR.GdprRequest); these names are stable in 10.3 but treat them as identifiers - renaming would invalidate granted permissions in AbpPermissionGrants.

## Cross-references

**Phase 1 references:**
- [references/modules/identity-and-auth.md](../modules/identity-and-auth.md) — When discussing GDPR account deletion or Elsa's UseAbpIdentity bridge, link to Identity reference because the user/account lifecycle and authentication flows live there.
- [references/modules/operations.md](../modules/operations.md) — AI Management workspace permissions, GDPR data-request permissions, and Language Management permissions are all defined as ABP permissions; cross-link when explaining permission registration.
- [references/framework/fundamentals.md](../framework/fundamentals.md) — Language Management overrides the static localization layer; link any time a question mentions runtime translation editing vs. JSON resource files.
- [references/framework/architecture-modularity.md](../framework/architecture-modularity.md) — Virtual File Explorer is a UI on top of the Virtual File System; link whenever debugging missing or overridden virtual files.
- [references/infrastructure/eventing.md](../infrastructure/eventing.md) — GDPR's GdprUserData*Eto and Language Management's LanguageEto/LanguageTextEto rely on the distributed event bus; cross-link for microservice scenarios.
- [references/infrastructure/background-processing.md](../infrastructure/background-processing.md) — AI Management RAG indexing uses IndexDocumentJob; reference background-jobs when explaining document processing latency.
- [references/infrastructure/blob-storing.md](../infrastructure/blob-storing.md) — AI Management stores uploaded RAG documents through BLOB storing; link when configuring file-size limits or storage providers.

**External docs:**
- [Elsa Workflows Documentation](https://docs.elsaworkflows.io/) — The Elsa Pro module is a thin ABP integration over Elsa 3.x; activity authoring, expression languages (JavaScript/Liquid/C#), and workflow runtime details live in Elsa's own docs.
- [OpenAI API Reference](https://platform.openai.com/docs/api-reference) — AI Management exposes an OpenAI-compatible /v1 surface and consumes OpenAI's chat-completions and embeddings APIs; client behavior matches the upstream contract.
- [Ollama Documentation](https://ollama.com/docs) — AI Management's Ollama provider requires models to be pre-pulled with the Ollama runtime; configuration of OLLAMA_HOST and model tags is covered upstream.
- [Model Context Protocol (MCP) Specification](https://modelcontextprotocol.io/) — AI Management workspaces can register MCP servers as tool sources; tool/resource semantics are defined by the MCP spec.
- [pgvector](https://github.com/pgvector/pgvector) — One of the supported vector store backends for AI Management RAG; index types and distance operators come from the pgvector extension.
- [Qdrant Documentation](https://qdrant.tech/documentation/) — Alternative vector store provider for AI Management; collection configuration and HNSW parameters are upstream concerns.
- [EU GDPR Regulation (Regulation 2016/679)](https://eur-lex.europa.eu/eli/reg/2016/679/oj) — The legal basis for the GDPR module's data-export (Art. 15) and right-to-erasure (Art. 17) features.
- [ASP.NET Core Cookie Policy](https://learn.microsoft.com/aspnet/core/security/gdpr) — The AbpCookieConsent middleware layers on top of ASP.NET Core's cookie-policy infrastructure; understanding TrackingConsent/CanTrack is helpful.
- [Microsoft.Extensions.Localization](https://learn.microsoft.com/aspnet/core/fundamentals/localization) — ABP's localization (which Language Management extends at runtime) is built on the Microsoft.Extensions.Localization abstractions.

## Sources (ABP 10.3)

- [https://abp.io/docs/10.3/modules/ai-management](https://abp.io/docs/10.3/modules/ai-management)
- [https://abp.io/docs/10.3/modules/gdpr](https://abp.io/docs/10.3/modules/gdpr)
- [https://abp.io/docs/10.3/modules/elsa-pro](https://abp.io/docs/10.3/modules/elsa-pro)
- [https://abp.io/docs/10.3/modules/language-management](https://abp.io/docs/10.3/modules/language-management)
- [https://abp.io/docs/10.3/modules/virtual-file-explorer](https://abp.io/docs/10.3/modules/virtual-file-explorer)

Last verified: 2026-05-10
