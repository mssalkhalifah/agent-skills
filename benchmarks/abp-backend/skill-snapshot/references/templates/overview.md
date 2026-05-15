# Solution templates — selection guide and comparison

> Pick the right starting point before scaffolding. Three first-party ABP 10.3 templates: Single-Layer, Layered, Microservice.

## When to load this reference

- "Which ABP template should I use?" / "single-layer vs layered vs microservice"
- About to run `abp new` or open ABP Studio's New Solution wizard, unsure of `-t` value
- Describing a new project (team size, lifetime, scaling, deployment) and asking for an architectural recommendation
- Asking about migrating between templates ("can I move my single-layer app to layered later?")
- Wanting a side-by-side comparison of project counts, built-in features, or deployment options
- Mentioning "modular monolith" in ABP context (it is the Layered template configured with internal modules)
- Asking about license tiers — Microservice template requires ABP Business+; some Layered features are premium

## Quick decision matrix

| Signal | Single-Layer (`-t app-nolayers`) | Layered (`-t app`) — default | Microservice (`-t microservice`, Business+) |
|---|---|---|---|
| Team size | 1-3 devs | 2+ devs | Multiple teams |
| Lifetime | Weeks-months, POC | Years | Years, multi-product |
| Complexity | 1-2 bounded contexts | Multiple bounded contexts | Independent scale/deploy/SLA per service |
| Tests | Not shipped | 6 test projects shipped | Per-service test projects |
| Mobile | Not supported | MAUI / React Native (Team+) | MAUI / React Native with own gateway |
| Kubernetes/Helm | Not documented | Helm charts (Business+) | First-class (helm + k8s manifests in `/etc`) |
| Migration in/out | Rewrite required to move | Cleanest path to microservice | Hard from Single-Layer; feasible from Layered |
| License | Free | Free (Business+ for some features) | **Business+ required** |

**When in doubt, recommend Layered.** Single-Layer is a deliberate downgrade for simplicity; Microservice is a deliberate upgrade for scale.

## SaaS / multi-tenancy as a deciding factor

Multi-tenancy is one of the most common reasons developers reach for ABP. It is **not** a reason to jump to Microservice. All three templates support multi-tenancy; the question is which one fits the rest of your situation:

| If your project... | Pick |
|---|---|
| Is a B2B SaaS, 2+ developers, expected to live for years, needs tenants from day one | **Layered** with the SaaS module (Pro+) wired in. The Tenant Management OSS module ships free; the SaaS Pro module adds editions, billing tiers, and the management UI. Resolve tenants by subdomain, header, or route segment via `AbpTenantResolveOptions`. |
| Is a small internal tool that happens to need 2-3 host-managed tenants without a per-tenant UI | **Single-Layer** with the OSS Tenant Management module. Skip the SaaS module — you don't need editions/billing. |
| Already needs per-tenant SLAs, isolated databases per tenant *and* per-service deploys | **Microservice** — but only if the other Microservice signals also hold (Business+ license, multi-team org, mature DevOps). The dedicated SaaS service in the Microservice template is convenient, not a justification on its own. |

**OSS vs Pro module split for tenancy:**

- **Tenant Management** (OSS / free) — basic tenant CRUD, `IMultiTenant` data filter, `ICurrentTenant`, tenant resolution. This is enough for many B2B SaaS apps.
- **SaaS** (Pro+) — adds editions, feature-tier mapping, billing-aware tenant lifecycle, and the polished tenant-management UI. Needed when commercial SaaS UX matters.

If you mention "multi-tenancy" in the same breath as "Layered template," you are on the right path. See [framework/multi-tenancy.md](../framework/multi-tenancy.md) for the wiring details.

## CLI template ids vs marketing names

The marketing names in docs differ from CLI ids. Always reach for CLI ids in scripts:

| Marketing name | CLI id |
|---|---|
| Single-Layer | `app-nolayers` |
| Layered | `app` |
| Modular Monolith | `app` (Layered + internal ABP modules — there is no separate id) |
| Microservice | `microservice` |
| Application Module | `module` |
| Console | `console` |

## `abp new` command shape

```
abp new <CompanyName>.<ProjectName> \
  -t <template>                       # app-nolayers | app | microservice | module | console
  -u <ui>                             # mvc | angular | blazor | blazor-server | blazor-webapp | none
  -d <db-provider>                    # ef | mongodb
  --database-management-system <dbms> # SqlServer | MySQL | SQLite | Oracle | Oracle-Devart | PostgreSQL
  -m <mobile>                         # none | maui | react-native
  [--tiered]                          # split UI host from HttpApi.Host
  [--separate-auth-server]            # extract OpenIddict server (default for Angular)
  [-csf]                              # create solution folder
  [-o <output>]
  [--connection-string "..."]
```

Post-scaffold, template choices are baked in — switching UI framework or going Single-Layer → Layered is a manual rewrite, not a flag flip. The only configuration that survives template selection is the per-module wiring in `*Module.cs` (covered in [framework/fundamentals.md](../framework/fundamentals.md)).

## Code examples

### Scaffold a Single-Layer MVC + PostgreSQL app

POC / short-lived internal tool — minimum projects, fastest startup.

```bash
# Prerequisites: dotnet --version >= 10, abp --version >= 10.3
abp new Acme.QuickApp \
    -t app-nolayers \
    -u mvc \
    -d ef \
    --database-management-system PostgreSQL \
    -csf

# Result: one project Acme.QuickApp.csproj with folders
#   Data/  Entities/  Services/  Pages/  Permissions/
#   Localization/  Menus/  ObjectMapping/  Migrations/

cd Acme.QuickApp
dotnet run --migrate-database   # creates DB and seeds initial data
dotnet run                       # runs the app on https://localhost:44…
```

`-t app-nolayers` selects Single-Layer (template id is `app-nolayers`, **not** `single-layer`). `dotnet run --migrate-database` is the Single-Layer equivalent of the DbMigrator project — there is no separate DbMigrator in this template.

### Scaffold a Layered Blazor WASM + EF Core SQL Server app (default)

Standard greenfield enterprise app, 3+ devs, long-lived, will need tests and clean DDD boundaries.

```bash
abp new Acme.BookStore \
    -t app \
    -u blazor \
    -d ef \
    --database-management-system SqlServer \
    -m none \
    -csf

# Resulting tree (abridged):
# Acme.BookStore/
# ├── src/
# │   ├── Acme.BookStore.Domain.Shared/
# │   ├── Acme.BookStore.Domain/
# │   ├── Acme.BookStore.Application.Contracts/
# │   ├── Acme.BookStore.Application/
# │   ├── Acme.BookStore.EntityFrameworkCore/
# │   ├── Acme.BookStore.HttpApi/
# │   ├── Acme.BookStore.HttpApi.Client/
# │   ├── Acme.BookStore.Blazor/
# │   └── Acme.BookStore.DbMigrator/
# └── test/
#     ├── Acme.BookStore.TestBase/
#     ├── Acme.BookStore.Domain.Tests/
#     ├── Acme.BookStore.Application.Tests/
#     ├── Acme.BookStore.EntityFrameworkCore.Tests/
#     └── Acme.BookStore.HttpApi.Client.ConsoleTestApp/

cd Acme.BookStore/src/Acme.BookStore.DbMigrator
dotnet run                       # initializes DB + seeds data
cd ../Acme.BookStore.Blazor
dotnet run
```

`-t app` is the default Layered template. The DbMigrator is a real console project here, not a CLI flag — always run it before the host. Test projects ship out-of-the-box (one per layer) — a key advantage over Single-Layer.

### Scaffold a Tiered Layered solution with separate auth server

UI and HTTP API need to scale or deploy independently; multiple client apps share one OpenIddict server.

```bash
abp new Acme.BookStore \
    -t app \
    -u angular \
    -d ef \
    --database-management-system PostgreSQL \
    --tiered \
    --separate-auth-server

# Adds two extra hostable projects:
#   Acme.BookStore.HttpApi.Host         (was bundled into Web in untiered)
#   Acme.BookStore.AuthServer           (OpenIddict host, only with --separate-auth-server)
# Angular UI lives in /angular/ alongside the .NET solution.
```

`--tiered` splits Web and HttpApi.Host into separately deployable units. `--separate-auth-server` extracts AuthServer; for Angular this is enabled by default because the SPA cannot host the auth server. **Use these flags upfront** — converting a non-tiered solution is non-trivial.

### Scaffold a Microservice solution (Business+ license required)

Multi-team org, true independent deployment per service, needs Kubernetes from day one.

```bash
# Requires ABP Business or higher license activated via `abp login`
abp new Acme.CloudCrm \
    -t microservice \
    -u angular \
    -d ef \
    --database-management-system PostgreSQL \
    -m none

# Top-level layout:
# Acme.CloudCrm/
# ├── apps/        # angular UI, public website
# ├── gateways/    # YARP: web-gateway, public-web-gateway (BFF per UI)
# ├── services/    # administration/, identity/  (always)
# │                # + saas/, audit-logging/, file-management/, gdpr/, chat/ (optional)
# ├── etc/
# │   ├── docker/         # docker-compose for RabbitMQ, Redis, ElasticSearch, etc.
# │   ├── helm/           # Helm charts for k8s deploy
# │   ├── k8s/            # K8s Dashboard setup
# │   └── abp-studio/     # shared ABP Studio settings (in source control)
# └── .abpstudio/         # personal dev settings (gitignored)

cd Acme.CloudCrm/etc/docker && docker compose up -d
# Then open the .sln in Visual Studio or use ABP Studio to start each module.
```

Each service/app/gateway is its own ABP Studio module with its own .NET solution. `etc/docker/docker-compose.yml` brings up RabbitMQ (event bus), Redis (cache + distributed lock), ElasticSearch (logs), Prometheus + Grafana — all wired by default. Without a Business license `abp new -t microservice` will fail.

## Solution structure (cross-template summary)

Per-template references contain the full trees. Quick comparison:

```
# ===== Single-Layer (-t app-nolayers) =====
# One .csproj. Concerns are folders, not projects.
Acme.QuickApp/
  Acme.QuickApp.csproj         # the only project; host + everything
  Data/                        # EF Core / MongoDB mappings, repositories
  Entities/                    # domain entities
  Services/                    # application services
  Pages/                       # Razor Pages UI (or wwwroot+Components for Blazor)
  Permissions/                 # AbpPermissionDefinitionProvider impls
  Localization/                # *.json resource files
  Menus/                       # IMenuContributor impls
  ObjectMapping/               # AutoMapper profiles
  Migrations/                  # EF Core migrations live here
  appsettings.json
  Program.cs                   # WebApplication + AbpApplication bootstrap
  *Module.cs                   # the single AbpModule for the solution

# ===== Layered (-t app) =====
# Nine src/ projects + six test/ projects. Full DDD layering.
Acme.BookStore/
  src/
    Acme.BookStore.Domain.Shared/         # constants, enums, error codes — leaf, no deps
    Acme.BookStore.Domain/                # entities, repos (interfaces), domain services, value objects
    Acme.BookStore.Application.Contracts/ # service interfaces, DTOs, permission constants
    Acme.BookStore.Application/           # IAppService implementations, AutoMapper profiles
    Acme.BookStore.EntityFrameworkCore/   # DbContext, repository implementations, EF migrations
    Acme.BookStore.HttpApi/               # MVC controllers (often auto-generated)
    Acme.BookStore.HttpApi.Client/        # static C# client proxies for inter-service calls
    Acme.BookStore.Web/                   # MVC/Blazor UI host
    Acme.BookStore.DbMigrator/            # console app — runs at deploy to migrate + seed
  test/                                   # 6 test projects, one per layer
# With --tiered: add Acme.BookStore.HttpApi.Host
# With --separate-auth-server: add Acme.BookStore.AuthServer
# With -u angular: add /angular/ folder

# ===== Microservice (-t microservice) =====
# Top-level *containers*; each child is itself an ABP Studio module with its own .sln.
Acme.CloudCrm/
  apps/         # web/, public-web/
  gateways/     # web-gateway/, public-web-gateway/, [mobile-gateway/]
  services/     # administration/ identity/  (always)
                # saas/ audit-logging/ file-management/ gdpr/ chat/  (optional)
  etc/          # docker/  helm/  k8s/  abp-studio/
  .abpstudio/   # personal dev preferences — GIT-IGNORED
```

## When to use each template

**Pick Single-Layer** when ALL of these hold: 1-3 developers, project lifetime in weeks/months, the domain fits in one bounded context, no shared mobile clients or per-layer test isolation needed, and Kubernetes is overkill. Typical fits: internal admin tools, demo apps, hackathon projects, prototypes.

**Pick Layered (the default)** when ANY of these hold: lifetime in years, two or more developers in parallel, multiple bounded contexts that benefit from DDD layering, you want test projects out-of-the-box, you need a clean migration path to microservices later, or you ship both web and mobile UIs sharing the same backend. This is also the right template for a 'modular monolith' — Layered + internal ABP modules.

**Pick Microservice** ONLY when ALL of these hold: ABP Business+ license active; multiple teams will own services independently; independent deploy/scale/SLA per service is a real (not aspirational) requirement; mature DevOps (Kubernetes, observability, distributed tracing, message-broker ops); polyglot or per-service tech-stack churn expected; the cost of running RabbitMQ + Redis + ElasticSearch + Prometheus + Grafana from day one is acceptable.

## When to avoid

**Avoid Single-Layer** if you'll need real test coverage (zero test projects shipped), if you'll deliver a mobile app sharing the backend (not supported), if you anticipate the project living past ~12 months with growing scope, or if compliance/audit requires hard layer boundaries — use **Layered** instead.

**Avoid Layered** if you genuinely need independent deployability per bounded context with separate teams, or if you've already hit operational ceilings on a layered monolith that can only be unblocked by per-service scaling — promote to **Microservice**. Conversely, avoid Layered for a one-week throwaway tool.

**Avoid Microservice** unless every condition under "when to use" is true. The most common foot-gun is choosing it for a 3-person team that "might scale someday" — start with **Layered** as a modular monolith and split out services only when concrete pain forces it. Also avoid without a Business+ license, without DevOps capacity, and when the domain is one bounded context (you'll get all the operational cost and none of the architectural benefit).

## Built-in features (cross-template summary)

Always-shipped (all three templates): Authentication (OpenIddict), Permission Management, Setting Management, Feature Management, Localization, Background Jobs, Background Workers, Distributed Locking, BLOB Storing, CORS, Swagger, Logging (Serilog), Database Configurations.

| Feature | Single-Layer | Layered | Microservice |
|---|---|---|---|
| Auto API Controllers | hand-written | auto-generated | auto-generated |
| Multi-Tenancy (SaaS, Pro+) | wireable in-process | wireable in-process | dedicated SaaS service |
| Distributed Event Bus | — | — | RabbitMQ + Inbox/Outbox |
| Distributed Cache | in-memory | in-memory | Redis |
| API Gateways | — | — | YARP BFF per UI |
| Observability | Serilog file/console | Serilog file/console | Prometheus + Grafana + ElasticSearch + Kibana |
| Auto-DB-Migration on Deploy | `--migrate-database` flag | `--migrate-database` (Business+) | per-service migrators (Business+) |
| LeptonX Themes | LeptonX Lite (free) / LeptonX (Pro) | same | same |

## Deployment options

**Single-Layer** — Designed for the simplest deploy targets: `dotnet publish` to IIS, an Azure App Service, a single Linux VM, or a single-container Docker image. Migrations run via `dotnet run --migrate-database` at deploy time. **Kubernetes / Helm are not documented for this template**; if you need them, choose Layered.

**Layered** — Documented deployment targets are Docker Compose, Azure Web App Service, IIS, and Kubernetes via Helm charts (Business+). The DbMigrator console project is meant to be invoked as a Kubernetes Job / Azure pre-deploy step. Separate guides ship for IdentityServer-based and OpenIddict-based auth-server deploys. See [layered.md](./layered.md) for full deployment trees.

**Microservice** — Kubernetes is the primary target: ships Helm charts under `etc/helm/` and K8s Dashboard manifests under `etc/k8s/`. Local dev uses `docker compose -f etc/docker/docker-compose.yml up` for RabbitMQ + Redis + ElasticSearch + Prometheus + Grafana. .NET Aspire integration is documented for AppHost-orchestrated local runs. Each microservice + gateway + app is independently deployable; the umbrella Helm chart composes them. See [microservice.md](./microservice.md).

## Differences between templates (migration notes)

**Single-Layer vs Layered.** Project count: 1 vs 15 (9 src + 6 tests). DDD layering: folders vs projects (Single-Layer can't enforce layer boundaries at compile time). Test projects: zero vs six. Mobile: not supported vs MAUI/React Native. Kubernetes/Helm: not documented vs first-class. DbMigrator: integrated `--migrate-database` flag vs separate console project. **Migration path:** rewriting Single-Layer → Layered means manually splitting folders into the nine .csproj layers, re-wiring AbpModule dependencies, and extracting EF migrations — feasible but non-trivial; do it BEFORE the codebase grows past a few thousand lines.

**Layered vs Microservice.** Project layout: nine src in one solution vs apps/+gateways/+services/ each as its own ABP Studio module with its own .sln. Inter-module calls: in-process method calls vs HTTP/gRPC + RabbitMQ distributed events. Database: shared by default vs database-per-service. Caching: in-memory default vs Redis. Auth: in-process or separate auth-server vs dedicated auth service behind gateways. Observability: Serilog file/console vs Serilog→ElasticSearch + Prometheus + Grafana. Deploy unit: monolith (or 2-3 hosts if `--tiered`) vs N independent K8s deployments. **Migration path:** a well-modularized Layered solution (one ABP module per bounded context) is the *intended* pre-stage to Microservice — extract a module's Domain+Application+EF projects into a new microservice solution and replace in-process calls with distributed events.

**Single-Layer vs Microservice.** These are opposite ends. There is **no** direct migration path; in practice you go Single-Layer → Layered → Microservice if and when each step is justified by real pain. Do not try to skip Layered.

## Common mistakes

### Choosing Microservice "because we'll need to scale eventually" on a 3-person team

**Why wrong:** You inherit RabbitMQ + Redis + ElasticSearch + Prometheus + Grafana operational cost on day one with zero scale benefit. Distributed event consistency (Inbox/Outbox) and per-service migrations are real engineering tax. You'll spend more time on plumbing than features.

**Correct pattern:** Start with Layered (`-t app`) as a modular monolith — one ABP module per bounded context. Promote to Microservice only when concrete pain (a module needing independent scale, a team needing independent deploy cadence) forces the split. The Layered → Microservice migration path is well-trodden; Layered → Microservice for the right reason is far cheaper than Microservice → "distributed monolith" rescue.

```bash
abp new Acme.App -t app -u angular -d ef --database-management-system PostgreSQL    # not: -t microservice
```

### Choosing Single-Layer for a project that will live more than ~12 months

**Why wrong:** Single-Layer cannot enforce layer boundaries at compile time (folders are not assemblies), ships zero test projects, and has no documented Kubernetes/Helm story. When the codebase grows, refactoring folders→projects is mechanical but disruptive (touches every namespace and every Module reference).

**Correct pattern:** Use Single-Layer ONLY for short-lived tools, POCs, and learning projects. For anything you expect to maintain, default to Layered. The marginal effort of working in 9 projects vs 1 is paid back many times over by enforced boundaries and out-of-box tests.

```bash
abp new Acme.App -t app   # not: -t app-nolayers
```

### Confusing CLI template ids: writing `-t single-layer`, `-t layered`, or `-t modular-monolith`

**Why wrong:** Those ids do not exist. The CLI accepts `-t app-nolayers` (Single-Layer), `-t app` (Layered — and how you build a modular monolith), `-t microservice`, `-t module`, `-t console`. The marketing names in docs differ from CLI ids.

**Correct pattern:** Always reach for CLI ids in scripts. The "Modular Monolith" option in the selection guide is not a separate template — it is the Layered template populated with internal ABP modules.

```bash
# wrong:  abp new Acme.X -t layered
# right:  abp new Acme.X -t app
```

### Skipping `--tiered` / `--separate-auth-server` at scaffold time, planning to add them later

**Why wrong:** Both flags fundamentally change project topology (extra hosts, different OpenIddict client config, different gateway/CORS posture). Adding them after the fact is manual surgery: extracting AuthServer, splitting Web from HttpApi.Host, re-issuing client secrets. Cost upfront ≈ 0; cost retrofitting ≈ days.

**Correct pattern:** Decide tiered-ness and separate-auth-server-ness at scaffold time based on a 5-minute deployment-topology conversation. For Angular, `--separate-auth-server` is the default and right answer. For MVC monolith deploys, leave both off.

```bash
abp new Acme.App -t app -u angular --tiered --separate-auth-server   # decide once, upfront
```

### Running the host before running DbMigrator (Layered) or `dotnet run --migrate-database` (Single-Layer)

**Why wrong:** OpenIddict, Identity, Permissions, Settings, and (if SaaS) tenants/editions are seeded by the migrator. Without seeding, login fails, no admin user exists, and dynamic permissions don't show in the UI. Errors at runtime are cryptic ("OpenIddict application not found").

**Correct pattern:** First run after scaffold (and after any new migration) MUST run the migrator. CI/CD should make this a separate step (Kubernetes Job, Azure pre-deploy task) running before the host pods.

```bash
# Layered
cd src/Acme.App.DbMigrator && dotnet run
# Single-Layer
dotnet run --migrate-database
```

### Trying to scaffold the Microservice template without a Business+ license

**Why wrong:** `abp new -t microservice` is gated server-side; without an active Business or Enterprise license token (`abp login`), the CLI errors out or downloads a crippled template.

**Correct pattern:** Confirm license tier first (`abp login` and check the dashboard at abp.io). For free-tier users, the right answer is Layered with internal modules — modular-monolith benefits without the licensing requirement.

### Using ABP's optional Pro modules (Chat, File Management, GDPR, Language Management, SaaS) on a free-tier license

**Why wrong:** These are commercial modules. The CLI may scaffold references to them, but at runtime the NuGet packages won't restore for non-Pro accounts. Builds fail with 401 from the ABP private NuGet feed.

**Correct pattern:** Cross-check the optional-modules list against your license tier BEFORE running `abp new`. Free tier ships only OSS modules (Identity, OpenIddict, Account, Audit Logging, Permissions, Settings, Features, basic Tenant Management).

```bash
# Free-tier safe:
abp new Acme.App -t app -u mvc -d ef --database-management-system PostgreSQL
# Then add only OSS modules via: abp add-module Volo.Abp.AuditLogging
```

### Treating "Modular Monolith" as a separate template to scaffold

**Why wrong:** The selection guide lists Modular Monolith as a fourth architecture, which makes users hunt for `-t modular-monolith`. There is no such template id.

**Correct pattern:** Build a modular monolith with `-t app` (Layered) and add internal ABP modules under a `/modules/` folder, each as a self-contained ABP module project. See [framework/architecture-modularity.md](../framework/architecture-modularity.md).

```bash
abp new Acme.App -t app
# then per bounded context:
abp new-module ProductCatalog -t module --no-ui
```

## Version pins (ABP 10.3)

- ABP 10.3 ships these CLI template ids: `app-nolayers`, `app`, `microservice`, `module`, `console`. Earlier names like `app-pro` / `app-nolayers-pro` were unified.
- Layered template's get-started page references `.NET 10.0+`. Confirm against your local `dotnet --version`.
- OpenIddict is the default; IdentityServer is on a deprecation arc upstream — treat as legacy for new solutions.
- On-the-fly auto-migration (`--migrate-database` host flag) is documented as Business+ in 10.3.
- Microservice observability stack in 10.3 is Serilog → ElasticSearch + Prometheus + Grafana. Earlier 10.x minors may have used Seq.
- Helm chart structure changed across 10.x. Treat any specific chart-name guidance as 10.3-only.
- The selection-guide page lists FOUR architectures (Single-Layer, Layered, Modular Monolith, Microservice) while the CLI exposes THREE host-app templates. This is doc-vs-CLI vocabulary, not a missing template.

## Cross-references

**Phase 1 references:**
- [templates/single-layer.md](./single-layer.md) — Single-Layer specifics, structure, `--migrate-database` workflow
- [templates/layered.md](./layered.md) — Layered DDD project tree, `--tiered`/`--separate-auth-server` flags, DbMigrator project
- [templates/microservice.md](./microservice.md) — apps/gateways/services layout, YARP, RabbitMQ event bus, Helm/Aspire deploys
- [framework/architecture-ddd.md](../framework/architecture-ddd.md) — The Layered template's four-layer split is a direct mapping of ABP's DDD guidance
- [framework/architecture-modularity.md](../framework/architecture-modularity.md) — Modular monolith pattern, ABP module structure
- [framework/multi-tenancy.md](../framework/multi-tenancy.md) — Multi-Tenancy / SaaS (in-process vs SaaS microservice)
- [framework/authorization.md](../framework/authorization.md) — Permission Management and OpenIddict pre-wiring
- [framework/api-development.md](../framework/api-development.md) — Auto API Controllers, static C# client proxies, Swagger
- [framework/data-ef-core.md](../framework/data-ef-core.md) — `--database-management-system` choices, DbMigrator workflow

**External docs:**
- [OpenIddict](https://documentation.openiddict.com/) — Default auth server in all three templates
- [YARP](https://microsoft.github.io/reverse-proxy/) — Powers Microservice template gateways
- [Entity Framework Core](https://learn.microsoft.com/ef/core/) — Default ORM behind ABP repositories
- [.NET Aspire](https://learn.microsoft.com/dotnet/aspire/) — Microservice template's local-orchestration integration
- [ASP.NET Core](https://learn.microsoft.com/aspnet/core/) — Underlies all template hosts
- [RabbitMQ](https://www.rabbitmq.com/documentation.html) — Default event bus in Microservice template
- [Redis](https://redis.io/documentation) — Distributed cache + lock in Microservice template
- [Helm](https://helm.sh/docs/) — Microservice template's chart engine
- [Prometheus](https://prometheus.io/docs/) + [Grafana](https://grafana.com/docs/grafana/latest/) — Microservice metrics
- [ABP CLI new-command-samples](https://abp.io/docs/10.3/cli/new-command-samples) — Source of truth for `abp new` flags

## Sources (ABP 10.3)

- [Solution Templates index](https://abp.io/docs/10.3/solution-templates)
- [Selection Guide](https://abp.io/docs/10.3/solution-templates/guide)
- [Get Started](https://abp.io/docs/10.3/get-started)
- [Single-Layer overview](https://abp.io/docs/10.3/solution-templates/single-layer-web-application/overview) and [solution-structure](https://abp.io/docs/10.3/solution-templates/single-layer-web-application/solution-structure)
- [Layered overview](https://abp.io/docs/10.3/solution-templates/layered-web-application/overview) and [solution-structure](https://abp.io/docs/10.3/solution-templates/layered-web-application/solution-structure)
- [Microservice overview](https://abp.io/docs/10.3/solution-templates/microservice/overview), [solution-structure](https://abp.io/docs/10.3/solution-templates/microservice/solution-structure), [microservices](https://abp.io/docs/10.3/solution-templates/microservice/microservices)
- [Get Started: Single-Layer](https://abp.io/docs/10.3/get-started/single-layer-web-application), [Layered](https://abp.io/docs/10.3/get-started/layered-web-application)
- Last verified: 2026-05-10
