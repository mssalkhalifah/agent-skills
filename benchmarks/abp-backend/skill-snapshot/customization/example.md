---
project: Acme.BookStore
abp-version: "10.3"
template: single-layer
example_only: true
---

# Acme.BookStore — example `.abp-overrides.md`

This file is a **worked example** of the sidecar contract documented in [`customization/README.md`](./README.md). It is shipped inside the public `abp-backend` skill so consumers can see what a real project's overrides look like; in a real project it would live at `<repo-root>/.abp-overrides.md` (or any ancestor folder up to `.git/`). The same file is loaded by the peer **`abp-frontend`** skill — UI-side conventions in the sidecar are interpreted there.

`Acme.BookStore` is a fictional placeholder. Substitute your own project's names when copying this as a starting point.

## Project shape (overrides the skill's defaults)

- **Template:** Single-Layer (`abp new -t app-nolayers`). Read [`references/templates/single-layer.md`](../references/templates/single-layer.md) first; everything below assumes that template.
- **Stack:** .NET 10, EF Core, PostgreSQL, OpenIddict, multi-tenancy enabled.
- **License-tier assumption:** ABP Pro features are available — code may reference SaaS, Forms (Pro), and other Pro modules without a free-tier fallback.

## Naming conventions

The skill's defaults use placeholder names (`MyApp`, `MyAppController`, `MyApp:` exception namespace, `App` table prefix). Acme.BookStore substitutes:

| Placeholder in the skill | Acme.BookStore value |
|---|---|
| `MyApp` (project root) | `Acme.BookStore` |
| `<YourBaseController>` (Rule 4) | `BookStoreController` |
| Localization resource | `BookStoreResource` |
| Exception code namespace | `"BookStore"` (e.g. `BusinessException("BookStore:OrderNumberAlreadyExists")`) |
| Table prefix | `"App"` |
| Connection string name | `Default` (in `Acme.BookStore/appsettings.json` under `ConnectionStrings:Default`) |
| Single AbpModule class | `BookStoreModule` (in `Acme.BookStore/BookStoreModule.cs`) |

## Solution layout

- `Acme.BookStore/` — backend host. Single project, single `AbpModule` (`BookStoreModule.cs`), single-layer template (`app-nolayers`).
- `Acme.BookStore.Tests/` — integration tests via `AbpWebApplicationFactoryIntegratedTest<Program>` against SQLite.
- `modules/acme.bookstore.reviews/` — in-tree ABP module pair (`Reviews` + `Reviews.Contracts`). **Not yet wired** into `BookStoreModule` at the time of writing; consult git history before assuming it's loaded.
- Frontends (interpreted by the **`abp-frontend`** skill):
  - `web/` — Vite SPA, port `5173`.
  - `admin/` — Next.js, port `44371`.
  - `mobile/` — Expo, port `19000`.

## Rule-level interpretation

The skill's six framework guardrails apply unchanged to Acme.BookStore. The notes below capture **how** the rules manifest in this project — they don't override the rule, they pin it to project-specific names.

### Rule 4 — explicit controllers

Acme.BookStore follows the recommended pattern. Controllers extend `BookStoreController`:

```csharp
[Authorize]
[Route("api/bookstore/orders")]
public sealed class OrderController : BookStoreController, IOrderAppService
{
    private readonly IOrderAppService _orderAppService;

    public OrderController(IOrderAppService orderAppService)
    {
        _orderAppService = orderAppService;
    }

    [HttpPost]
    public Task<OrderDto> CreateAsync(CreateOrderDto input)
        => _orderAppService.CreateAsync(input);
}
```

Application services extend `ApplicationService` (the framework default — Acme.BookStore does not introduce a `BookStoreAppService` base).

### Rule 6 — per-module DbContexts and migrations history tables

When the Reviews module is wired in, it ships its own `ReviewsDbContext` and a separate migrations history table named `__EFMigrationsHistory_Reviews`. The public skill's [`references/framework/data-ef-core.md`](../references/framework/data-ef-core.md) shows the canonical factory + runtime registration shape — Acme.BookStore applies it verbatim, with the module project at `modules/acme.bookstore.reviews/Reviews/`. The `IDesignTimeDbContextFactory<ReviewsDbContext>` lives next to `ReviewsDbContext.cs`; its `BuildConfiguration` climbs to `Acme.BookStore/appsettings.json` so design-time and runtime use the same connection string.

The host's `BookStoreDbMigrationService` iterates through every `IXxxDbSchemaMigrator` registered via DI, so adding a new module just means shipping its `IReviewsDbSchemaMigrator` + `EntityFrameworkCoreReviewsDbSchemaMigrator` pair. Migrations are applied via `dotnet run --project Acme.BookStore --migrate-database`, never `dotnet ef database update` alone.

### Rule 7 — exception code namespace

All `BusinessException` codes in Acme.BookStore use the `BookStore:` prefix:

```csharp
throw new BusinessException("BookStore:OrderNumberAlreadyExists")
    .WithData("Number", number);
```

## Pre-existing technical debt the skill should know about

- `AppUserAppService.GetListAsync` calls `_repository.GetQueryableAsync()` directly inside the application service — this **violates Rule 2** (Never expose `IQueryable` to the application service layer). It predates the rule and should be refactored when next touched. Follow the Rule 2 fix pattern: define a custom `IAppUserRepository : IRepository<AppUser, Guid>` in the feature folder with the materialised query method.

## ABP Studio MCP server

ABP Studio's local MCP server (`abp mcp-studio`) is configured for this project. **ABP Studio must be running** for any of its tools to respond — the CLI is a stdio bridge to ABP Studio over HTTP.

> `abp mcp-studio` is **not** the same as `abp mcp`. The latter connects to abp.io's cloud service and requires a commercial licence; for Acme.BookStore we use only the local `mcp-studio` server.

Client configuration files (one per IDE):

```json
// Claude Desktop: ~/Library/Application Support/Claude/claude_desktop_config.json
// Cursor: .cursor/mcp.json
{
  "mcpServers": {
    "abp-studio": {
      "command": "abp",
      "args": ["mcp-studio"]
    }
  }
}
```

```json
// VS Code: .vscode/mcp.json (uses `servers` instead of `mcpServers`)
{
  "servers": {
    "abp-studio": {
      "command": "abp",
      "args": ["mcp-studio"]
    }
  }
}
```

Tool categories the MCP server exposes: monitoring (`get_exceptions`, `get_logs`, `get_requests`, `get_events`), app control (`start_application`, `stop_application`, `restart_application`, `build_application`), containers (`list_containers`, `start_containers`, `stop_containers`), solution structure (`get_solution_info`, `list_modules`, `list_packages`, `get_module_dependencies`). Monitor data is held in memory and is wiped when the solution closes — pull it before shutdown if you need it for an investigation.

## Testing notes specific to Acme.BookStore

- `Acme.BookStore.Tests/` uses `AbpWebApplicationFactoryIntegratedTest<Program>` with SQLite-in-memory. The full pattern is documented in the public skill's [`references/testing.md`](../references/testing.md).
- Integration tests assume PostgreSQL-specific behaviour (e.g., audit columns) is also valid on SQLite — verify any DateTime / text-collation assumptions when writing new tests.

## What this sidecar deliberately does NOT override

The following framework guardrails apply to Acme.BookStore **as-is** without modification:

- Rule 1 (one class per file) — strictly followed.
- Rule 2 (no `IQueryable` in app services) — followed except for the `AppUserAppService.GetListAsync` debt noted above.
- Rule 3 (domain services don't persist; `IReadOnlyRepository`-only injection; mutate-via-pass-in flow) — followed.
- Rule 5 (cross-module reads via Integration Services) — applies as soon as Reviews is wired up; no Acme.BookStore-specific override.
- Rule 7 (no `BusinessException` from app services; domain services own invariants) — followed; only the `BookStore:` exception-code prefix is project-specific.

## UI-side conventions (interpreted by the abp-frontend skill)

The Acme.BookStore frontends (`web/`, `admin/`, `mobile/`) are NOT generated from an ABP UI template — they are bespoke React / Next.js / Expo apps that consume the Acme.BookStore HTTP API. UI-side overrides specific to those apps belong with the **`abp-frontend`** skill's customization story; this backend skill does not interpret them.