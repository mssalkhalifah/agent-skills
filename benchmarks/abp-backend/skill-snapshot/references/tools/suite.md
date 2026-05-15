# Tools — ABP Suite

> ABP Suite is a commercial .NET global tool (also bundled with ABP Studio) that drives a browser UI for adding existing ABP solutions and generating CRUD pages, entities, relationships, and database-first scaffolds across MVC, Blazor, and Angular. UI-side customization details (Razor / Blazor `<suite-custom-code-block-N>` / Angular `*.abstract.*` regeneration boundary) are summarised here for context but are owned by the abp-frontend skill.

## When to load this reference

Load this reference whenever the user asks: 'how do I install/start ABP Suite', 'how do I scaffold a CRUD page', 'how do I generate entities from an existing database', 'how do I model many-to-many or master-detail in Suite', 'how do I keep my hand-written code when Suite regenerates', 'where are Suite templates / how do I edit them', 'why isn't there a create-solution option in Suite anymore', or anything mentioning `abp suite install`, `abp suite update`, `abp suite`, `Customizable code`, `*.Extended.cs`, or `suite-custom-code-block`.

**Audience:** ABP Team-or-higher license holders building Layered, Single-Layer, or Microservice ABP solutions who want to scaffold entities, services, DTOs, EF Core configuration, migrations, API proxies, and UI pages from a metadata UI instead of hand-writing the stack.

## Key concepts

- ABP Suite (tool): .NET global tool packaged as `Volo.Abp.Suite`; install via `abp suite install`, launch with `abp suite` (default http://localhost:3000). Reach for it as the primary code generator for any ABP commercial solution.
- Solution registration (Suite UI): Pointer to a `YourProject.sln` (or its containing folder if unique) of an ABP solution previously created by ABP Studio / ABP CLI. Suite does not create solutions in 8.3+.
- CRUD Page Generator: The main Suite workflow that emits domain entity, repository, application service + DTOs, AutoMapper/Mapperly profile, EF Core/Mongo configuration, migration, API proxy, UI pages, menu entry, and unit + integration tests from an entity definition.
- Entity Type — Master vs Child: Master entities get full UI; Child entities are owned by a master, surface as expandable grids in tabs on the master page, and cannot participate in many-to-many relationships.
- Property metadata: Per-property type, default value, min/max length, regex, email validation, required, nullable, advanced filter, textarea rendering, sortable column, and UI-list visibility.
- Base class options: `Entity<TKey>` or `AggregateRoot<TKey>`, each with optional `Audited` / `FullAudited` variants; `IMultiTenant`; concurrency stamp.
- Primary key: Choice of `Guid` (recommended), `int`, `long`, or `string`.
- Navigation property: 1-to-many lookup to another entity. Suite renders a modal lookup or dropdown depending on cardinality and target row count.
- Navigation collection: Many-to-many relationship modeled as a navigation tab on the master entity; Suite generates the third (junction) table, e.g. `AppBookCategory`, and a typeahead-driven UI tab.
- `Load Entity From Database`: Database-first import that reads an existing table via a connection string, maps columns to properties, and treats the existing primary key (e.g. uncheck the auto-generated `Id`) before generating CRUD.
- Customizable code: A per-entity flag (default on) that wraps generated code in hook points so regeneration preserves manual edits.
- `*.Extended.cs` partial classes: Companion partial files (e.g. `BookAppService.Extended.cs`, `EfCoreBookRepository.Extended.cs`, `IBookRepository.Extended.cs`) where developers add behavior that survives regeneration.
- HTML/Razor/Blazor hook blocks: `<suite-custom-code-block-N>...</suite-custom-code-block-N>` placeholders (up to 20 per file from v8.0+) inside `.cshtml` and `.razor` files, with paired `OnGetAsync` / `OnPostAsync` extension overrides in code-behind partials. (UI-side customization detail belongs to abp-frontend.)
- Angular regeneration boundary: `*.abstract.service.ts` and `*.abstract.component.ts` are regenerated every run; the concrete `*.service.ts` / `*.component.ts` are emitted once and then preserved for hand edits. (See abp-frontend for the full Angular customization story.)
- Editing Templates: Suite UI page that exposes the embedded `Volo.Abp.Commercial.SuiteTemplates` templates filtered by UI framework (MVC / Blazor / Angular) and DB provider (EF Core / MongoDB). Customized templates are flagged and revertible.
- Template syntax: A custom placeholder language (not Razor/Scriban): `%%variable-name%%` for substitutions and `%%<if:IMultiTenant>%%...%%</if:IMultiTenant>%%` for conditional blocks.
- Module source code download: Suite UI action (license-gated) to pull the source of an ABP commercial module (Identity, Payment, etc.) into the current solution.
- ABP Studio integration: ABP Studio bundles Suite, launches it without a separate terminal, and is the recommended host for new users.
- `appsettings.json`: Suite's own settings file under `~/.dotnet/tools/.store/volo.abp.suite/<version>/volo.abp.suite/<version>/tools/netcoreapp3.1/any/appsettings.json` (Linux/macOS) or `%USERPROFILE%\.dotnet\tools\.store\volo.abp.suite\<version>\...` (Windows).

## Configuration pattern

ABP Suite is a developer-time global tool, not an ABP module — it does not register services in `AbpModule.PreConfigureServices` / `ConfigureServices` / `OnApplicationInitialization` of the target solution. Instead, it is configured at three layers:

1) Install / version pinning (shell):
```bash
# Prereq: ABP CLI installed and `abp login` performed.
dotnet tool install -g Volo.Abp.Studio.Cli   # or the older Volo.Abp.Cli
abp login <username>
abp suite install                              # pinned latest stable
abp suite install --preview                    # or -p
abp suite install --version 10.3.0             # or -v <ver>
abp suite update                               # in-place upgrade
abp suite update -v 10.3.1
```
The Suite version MUST match the `Volo.Abp.*` package version of the target solution; otherwise generated templates do not align with the project layout.

2) Launch / port (shell + appsettings):
```bash
abp suite                                                       # opens http://localhost:3000
abp-suite --AbpSuite:ApplicationUrl="http://localhost:4000"     # one-shot override
```
Persistent override is by editing `ApplicationUrl` in Suite's own `appsettings.json` at the path noted under `concepts_list`. The terminal must stay open for the duration of the Suite session; CTRL+C cleanly stops it.

3) Per-entity generation options (Suite UI, persisted into solution metadata):
Key toggles set when generating an entity — `Customizable code` (default on), `Multi-tenancy` (`IMultiTenant`), `Concurrency stamp`, `Excel export`, `Bulk delete`, `Generate unit tests`, `Generate integration tests`, `Master`/`Child` entity type, primary-key type, base class (`Entity<TKey>` vs `AggregateRoot<TKey>`, plus `Audited`/`FullAudited`).

These choices are translated to (a) `partial class` C# files (when `Customizable code` is on), (b) `<suite-custom-code-block-N>` placeholders in Razor/Blazor templates, and (c) the `*.abstract.*` boundary in Angular. The actual ABP module wiring done by Suite is the conventional ABP module config inside the generated code itself — e.g. AutoMapper profile registration, EF Core `DbSet`/entity-type-configuration calls in the project's `*DbContext` and `*DbContextModelCreatingExtensions`, and menu contributor registration in the UI module — all of which flow through the project's existing `AbpModule.ConfigureServices`. Suite does not expose `AddAbpSuite()` or similar; there is nothing to call from the application's runtime composition root.

## Code examples

### Install and start ABP Suite

_Bringing Suite onto a workstation that already has the ABP CLI and a logged-in abp.io session._

```bash
# Install the global tool (requires ABP Team license + abp login)
abp suite install

# Start it; opens http://localhost:3000 in the default browser
abp suite

# Use a different port without editing appsettings.json
abp-suite --AbpSuite:ApplicationUrl="http://localhost:4000"

# Pin to a specific version that matches the solution's ABP packages
abp suite update -v 10.3.0

# Stop Suite cleanly
# (CTRL+C in the terminal that ran `abp suite`)
```

**Key lines:** `abp suite install` deploys the global tool; the Suite version MUST match `Volo.Abp.*` in the target solution. `abp suite` blocks the terminal — closing it kills Suite. `--AbpSuite:ApplicationUrl` is the standard way to free port 3000 (e.g. when running multiple Suite sessions or behind a reverse proxy).

### Generate a customizable CRUD entity (Book) with audit + multi-tenancy

_Adding a `Book` aggregate root with full audit, multi-tenant scoping, and customizable code so future regenerations preserve hand edits._

```csharp
// Suite-generated base file: src/Acme.BookStore.Domain/Books/Book.cs
using System;
using Volo.Abp.Domain.Entities.Auditing;
using Volo.Abp.MultiTenancy;

namespace Acme.BookStore.Books;

public partial class Book : FullAuditedAggregateRoot<Guid>, IMultiTenant
{
    public virtual Guid? TenantId { get; set; }
    public virtual string Title { get; set; }
    public virtual int PageCount { get; set; }
}

// Companion file you write — preserved across regenerations:
// src/Acme.BookStore.Domain/Books/Book.Extended.cs
using System;
using Volo.Abp.Domain.Entities.Auditing;

namespace Acme.BookStore.Books;

public partial class Book
{
    public virtual bool IsArchived { get; set; }

    public void Archive()
    {
        IsArchived = true;
    }
}
```

**Key lines:** `partial class Book` in two files is the contract: Suite owns the first file, you own `*.Extended.cs`. `FullAuditedAggregateRoot<Guid>` is selected via the entity-type + base-class pickers in the Suite UI. `IMultiTenant` is added by ticking `Multi-tenancy`; the `TenantId` column is then auto-included in EF Core configuration.

### Many-to-many via navigation collection (Book ↔ Category)

_Modeling a many-to-many between `Book` and `Category` so Suite generates the junction table and a typeahead UI tab._

```csharp
// 1) In Suite, generate the dependent entity first:
//    Category { Name : string }  -> CRUD page generated, sample rows seeded.
//
// 2) Then generate Book and switch to the "Navigations" tab:
//      Add navigation collection -> pick Category.cs
//      Display Property: Name
//
// 3) Suite emits three tables:
//      AppBooks, AppCategories, AppBookCategory  (junction, PK = (BookId, CategoryId))
//
// 4) Navigation collection on the C# entity (simplified):
public partial class Book : FullAuditedAggregateRoot<Guid>
{
    public virtual string Title { get; set; }
    public virtual ICollection<BookCategory> Categories { get; set; }
}

public class BookCategory : Entity
{
    public virtual Guid BookId { get; set; }
    public virtual Guid CategoryId { get; set; }
    public override object[] GetKeys() => new object[] { BookId, CategoryId };
}
```

**Key lines:** Choosing `Add navigation collection` (not `Add navigation property`) is what triggers the junction-table strategy. The `Display Property` drives both the typeahead label and the UI tab's grid column. `BookCategory.GetKeys()` returns the composite key — Suite generates this for every junction entity.

### Master-Detail (Order → OrderLine) via Child entity

_Creating an `Order` master with `OrderLine` children that surface as an expandable grid on the order page._

```csharp
// In Suite:
//   1) Generate Order with Entity type = Master (default). Properties: CustomerName, OrderDate.
//   2) Generate OrderLine with Entity type = Child, Master entity = Order.
//      Properties: ProductName, Quantity, UnitPrice.
//
// Resulting domain (Suite output, abridged):
public partial class Order : FullAuditedAggregateRoot<Guid>
{
    public virtual string CustomerName { get; set; }
    public virtual DateTime OrderDate { get; set; }
    public virtual ICollection<OrderLine> OrderLines { get; set; }
}

public partial class OrderLine : FullAuditedEntity<Guid>
{
    public virtual Guid OrderId { get; set; }   // FK back to Order
    public virtual string ProductName { get; set; }
    public virtual int Quantity { get; set; }
    public virtual decimal UnitPrice { get; set; }
}

// UI: Suite does NOT generate a top-level page for OrderLine.
// Instead, the Order detail page renders a child tab with an expandable grid
// for OrderLines (and any additional children). Many-to-many is disabled for
// child entities by Suite's UI on purpose.
```

**Key lines:** `Entity type = Child` is the lever; it (a) suppresses the top-level CRUD page and menu entry and (b) wires the child grid into the master page. Children inherit multi-tenancy from the master. Suite intentionally disables many-to-many on children — model that case as two masters with a navigation collection instead.

### Preserving page-model edits with an `Index.cshtml.Extended.cs` partial

_The page-model half of Suite's customization story — a developer-owned partial that survives regeneration. The Razor markup half (`<suite-custom-code-block-N>` markers, 20-block cap) lives in the abp-frontend skill._

```csharp
// src/Acme.BookStore.Web/Pages/Books/Index.cshtml.Extended.cs — preserved across re-generation
using Microsoft.AspNetCore.Mvc.RazorPages;

namespace Acme.BookStore.Web.Pages.Books;

public partial class IndexModel : PageModel
{
    public override async Task<IActionResult> OnGetAsync()
    {
        // your pre-load logic
        await base.OnGetAsync();
        // your post-load logic
        return Page();
    }
}
```

**Key lines:** The Suite-generated page model is a `partial class`; the developer-owned `*.Extended.cs` companion holds your `OnGetAsync` / `OnPostAsync` overrides and any custom server-side hooks. Same composition pattern as the `BookAppService.Extended.cs` / `EfCoreBookRepository.Extended.cs` / `IBookRepository.Extended.cs` files covered earlier. The `*.Extended.cs` file is only emitted when `Customizable code` is checked on the entity in the Suite UI — without that toggle, the partial-class file pair does not exist and overrides have nowhere to live.

### Editing a Suite template with `%%variable%%` placeholders

_Renaming a generated repository class or relocating a hook point by editing the template that produces it._

```csharp
// In Suite UI: "Edit Templates"
//   Filter: Server, EF Core
//   Open: EfCore_Repository  (example name)
//
// Template body fragment (the syntax is Suite's own, NOT Razor/Scriban):
using Volo.Abp.Domain.Repositories.EntityFrameworkCore;
using Volo.Abp.EntityFrameworkCore;

namespace %%namespace%%.%%entity-plural%%;

public class EfCore%%entity%%Repository
    : EfCoreRepository<%%dbcontext%%, %%entity%%, %%primary-key-type%%>,
      I%%entity%%Repository
{
    public EfCore%%entity%%Repository(IDbContextProvider<%%dbcontext%%> provider)
        : base(provider)
    {
    }

    %%<if:IMultiTenant>%%
    // Tenant-scoped overrides only when IMultiTenant was selected
    %%</if:IMultiTenant>%%
}

// After saving, the template gets a "Customized" badge in Suite.
// You can revert to the shipped version at any time from the same UI.
```

**Key lines:** `%%name%%` is a substitution; `%%<if:Flag>%%...%%</if:Flag>%%` is a conditional that depends on options chosen in the entity wizard (here `IMultiTenant`). The shipped templates live in the embedded `Volo.Abp.Commercial.SuiteTemplates` package — the version of that package follows the Suite version, which is why mismatched Suite/solution versions corrupt generation.

## Common mistakes

- - Mistake: Installing or upgrading Suite to a version that does not match the solution's `Volo.Abp.*` package version. Why wrong: templates assume the project layout and APIs of a specific ABP minor version, so mismatches produce compile errors or silently corrupt files. Fix: pin Suite explicitly with `abp suite install --version 10.3.0` (or `update -v 10.3.0`) and re-pin whenever the solution upgrades ABP packages.
- - Mistake: Closing the terminal that started `abp suite` while the browser tab is still open. Why wrong: Suite is the local process that backs the UI; closing the terminal kills the backend, after which the browser appears responsive but every action 500s or hangs. Fix: leave the terminal running, and stop with CTRL+C to exit cleanly. Use ABP Studio's built-in Suite launcher to avoid the bare terminal.
- - Mistake: Hand-editing a Suite-owned file (e.g. `Book.cs`, `BookAppService.cs`, the `Index.cshtml` body outside hook blocks, or any `*.abstract.service.ts`) and re-running generation. Why wrong: Suite overwrites those files on every generation. Fix: put backend changes in `*.Extended.cs` partial files **and ensure `Customizable code` is checked on the entity in the Suite UI** — without that toggle, the partial-class file pair is not emitted and `*.Extended.cs` cannot exist. UI-side custom-code-block / abstract-service guidance lives in the abp-frontend skill.
- - Mistake: Using `Add navigation property` when the relationship is many-to-many. Why wrong: that path emits a single FK column and no junction table; subsequent reverse-side updates fail. Fix: use `Add navigation collection` on the master, pick the dependent entity, and let Suite emit the junction table (e.g. `AppBookCategory`).
- - Mistake: Trying to put a many-to-many on a Child entity. Why wrong: Suite intentionally disables many-to-many for children because they have no top-level UI to host the picker, and the relationship would be ambiguous against the master. Fix: model the related side as a master and link via navigation collection on that master.
- - Mistake: Re-generating an entity from an existing database table without unchecking the `Id` column. Why wrong: ABP base classes (`Entity<TKey>`, `AggregateRoot<TKey>`) already declare `Id`, so leaving it checked produces a duplicate property and a generation error. Fix: in `Load Entity From Database`, uncheck `Id` and pick the matching primary-key type so the generated entity reuses the base-class `Id`.
- - Mistake: Calling Suite an ABP runtime module and trying to register it via `AddAbpSuite()` or in `AbpModule.ConfigureServices`. Why wrong: Suite is a developer-time tool; it has no runtime registration surface. Fix: configure it through its own CLI (`abp suite ...`) and its `appsettings.json` only.
- - Mistake: Running Suite without an ABP Team-or-higher license, or against an open-source-only ABP solution. Why wrong: Suite is commercial; both `abp login` and the embedded `Volo.Abp.Commercial.SuiteTemplates` package are license-gated. Fix: ensure the abp.io account has a Team-or-higher subscription before installing.
- - Mistake: Expecting Suite to create new solutions in 8.3+. Why wrong: the `create solution` workflow was removed; Suite only adds existing solutions now. Fix: create the solution with ABP Studio or `abp new` (CLI), then point Suite at the resulting `.sln` (or its directory).
- - Mistake: Adding more than 20 hook blocks to a single Razor/Blazor file. Why wrong: Suite caps customization placeholders at 20 per file (v8.0+); excess blocks are not preserved. Fix: split the page into components, or move logic into the `*.Extended.cs` partial.
## Version pins (ABP 10.3)

- ABP 10.3 (May 2026): Suite is documented under `/docs/10.3/suite/*`; all URLs in this reference resolved on `abp.io` without needing the `/latest/` fallback rule.
- 10.3 retains the 8.3+ behavior that Suite no longer creates solutions — the `create-solution` page is now an explanatory redirect to ABP Studio / ABP CLI.
- 10.3 retains the `<suite-custom-code-block-N>` mechanism with up to 20 placeholders per file (introduced in v8.0); any future bump to that cap is a documented breaking change to watch.
- `appsettings.json` path still references `tools/netcoreapp3.1/any/...` even on .NET 8/9 hosts because that is the `RuntimeIdentifier` folder used by the `dotnet tool` packaging — do not 'fix' the path. [uncertain] whether ABP plans to repackage Suite under a newer TFM in 11.x.
- The `Volo.Abp.Commercial.SuiteTemplates` NuGet package version is locked to the Suite version; upgrading one without the other is the most common cause of 'templates not suitable for your project' errors.
- The Mapperly / AutoMapper toggle: Suite 10.3 still emits AutoMapper profiles by default for new entities. [uncertain] whether the per-entity option to switch to Mapperly is wired into 10.3 across MVC, Blazor, and Angular paths uniformly — verify in the Suite UI's `Mapper` setting before assuming.
- Default Suite port remains `3000`; the override key `AbpSuite:ApplicationUrl` is unchanged in 10.3.
- Source-code download for ABP commercial modules continues to require an appropriate license tier; the page does not document Suite's own source availability, and the `source-code` doc is module-scoped only — do not assume Suite itself is source-distributed.

## Cross-references

**Phase 1 references:**
- - references/tools/cli.md — triggered when the user installs Suite (`abp suite install` requires ABP CLI + `abp login`) or pins versions.
- - references/tools/studio.md — triggered when the user is on ABP Studio: Suite ships inside Studio, and solution creation now lives in Studio/CLI rather than Suite (`create-solution` page).
- - references/templates/layered.md, references/templates/single-layer.md, references/templates/microservice.md — triggered when Suite asks to add an existing solution; generated artifacts target the project layout of one of these templates.
- - references/framework/architecture-ddd.md — triggered when discussing entity-type choices (Entity vs AggregateRoot, Master vs Child) and the Domain / Application / EF Core split that Suite emits.
- - references/framework/multi-tenancy.md — triggered when ticking the `Multi-tenancy` flag on an entity (`IMultiTenant`, `TenantId`).
- - references/framework/data-ef-core.md — triggered for the EF Core branch of generation: `DbSet`, `*DbContextModelCreatingExtensions`, migrations, and `Load Entity From Database`.
- - references/modules/operations.md — triggered by Suite's audit-logging integration on `Audited` / `FullAudited` base classes.
- - (UI-framework switch — MVC / Blazor / Angular customization) — see the abp-frontend skill.

**External docs:**
- - .NET Global Tools — https://learn.microsoft.com/dotnet/core/tools/global-tools — Suite is distributed as a `dotnet tool` (`Volo.Abp.Suite`) and lives under `~/.dotnet/tools/.store/volo.abp.suite/<version>/...`; understanding global-tool layout is required to find Suite's `appsettings.json`.
- - ASP.NET Core appsettings.json / configuration providers — https://learn.microsoft.com/aspnet/core/fundamentals/configuration/ — Suite reads its `ApplicationUrl` from `appsettings.json` and accepts the `AbpSuite:ApplicationUrl` override on the command line via the standard ASP.NET Core configuration binder.
- - Entity Framework Core — https://learn.microsoft.com/ef/core/ — generated repositories inherit `EfCoreRepository<TDbContext, TEntity, TKey>`; `Load Entity From Database` reads SQL Server / configured providers; migrations created by Suite are applied with `dotnet ef database update` or the project's DbMigrator.
- - ASP.NET Core Razor Pages — https://learn.microsoft.com/aspnet/core/razor-pages/ — `OnGetAsync` / `OnPostAsync` hooks customized in `*.Extended.cs` partials are standard Razor Pages members.

## Sources (ABP 10.3)

- https://abp.io/docs/10.3/suite — Suite overview: purpose, capabilities, licensing, and roadmap.
- https://abp.io/docs/10.3/suite/how-to-install — Installation prerequisites, `abp suite install`, update, preview/version flags.
- https://abp.io/docs/10.3/suite/how-to-start — Launch via `abp suite`, default port 3000, `AbpSuite:ApplicationUrl` override, ABP Studio integration.
- https://abp.io/docs/10.3/suite/add-solution — Adding an existing ABP `.sln` (or its directory) to Suite.
- https://abp.io/docs/10.3/suite/create-solution — Note that solution creation moved to ABP Studio / ABP CLI in 8.3+; Suite no longer creates solutions.
- https://abp.io/docs/10.3/suite/generating-crud-page — End-to-end CRUD generation, entity types, properties, navigation properties, generated artifacts.
- https://abp.io/docs/10.3/suite/creating-many-to-many-relationship — Navigation collections, junction table generation (e.g. `AppBookCategory`), UI tab.
- https://abp.io/docs/10.3/suite/generating-entities-from-an-existing-database-table — `Load Entity From Database`, connection strings, primary key handling.
- https://abp.io/docs/10.3/suite/creating-master-detail-relationship — Master/Child entity types, child UI suppression, expandable child grids.
- https://abp.io/docs/10.3/suite/customizing-the-generated-code — `*.Extended.cs` partials, `<suite-custom-code-block-N>` hooks, `*.abstract.*` regeneration boundary in Angular.
- https://abp.io/docs/10.3/suite/source-code — Suite's source-code download feature for ABP modules (not Suite itself); license-gated.
- https://abp.io/docs/10.3/suite/configuration — Location of Suite's `appsettings.json` under `~/.dotnet/tools/.store/volo.abp.suite/<version>/...`.
- https://abp.io/docs/10.3/suite/editing-templates — `Edit Templates` UI, `%%variable%%` and `%%<if:Flag>%%...%%</if:Flag>%%` syntax, embedded in `Volo.Abp.Commercial.SuiteTemplates`.

Last verified: 2026-05-10
