# Tools — ABP Studio (incl. MCP server)

> ABP Studio is the cross-platform desktop IDE-companion for ABP solutions that unifies project scaffolding, running, monitoring, Suite-based code generation, Kubernetes development, and an MCP server that exposes solution telemetry and control tools to AI clients such as Claude Desktop, Cursor, and VS Code.

## When to load this reference

- User asks 'how do I install / set up ABP Studio' or troubleshoots a Studio install failure (Docker, mkcert, WireGuard, login).
- User asks how to add / import / update modules or packages, switch ABP version (stable/preview/nightly), or generate client proxies.
- User asks how to run a multi-app ABP solution (microservices, gateways, IdentityServer/OpenIddict host, Angular SPA) under a single profile, or how Tasks differ from Applications.
- User asks how to view logs / HTTP requests / distributed events / exceptions for a running ABP app (telemetry, monitoring, Volo.Abp.Studio.Client.AspNetCore).
- User asks how to expose ABP Studio to Claude Desktop / Cursor / VS Code via MCP, or how the `abp mcp-studio` command differs from `abp mcp` (cloud).
- User asks how Studio integrates with Kubernetes (WireGuard tunnel, service interception, Helm charts, k8ssuffix metadata).
- User asks how to launch ABP Suite from Studio for CRUD generation, or why Suite version mismatch warnings appear.
- User asks how to add a reusable terminal command to Studio's context menus (Custom Commands, Scriban templates).
- User asks for the difference between an ABP Studio 'Solution', 'Module', and 'Package' vs Visual Studio's solution/project terminology.

**Audience:** ABP developers using the Studio desktop tool for solution lifecycle (scaffold, run, monitor, deploy), microservice teams using its Kubernetes/Helm features, and developers who want to drive ABP Studio from AI clients via MCP.

## Key concepts

- ABP Studio Solution: top-level container that may include multiple .NET solutions, modules, and packages (one product per Studio Solution). Reach for it whenever you need a single workspace that orchestrates running and monitoring across many .NET solutions, e.g. a microservice product.
- Module: sub-solution mapped to a .NET solution containing zero+ packages; the unit of import / install / version-pin. Created from Empty, DDD, Standard, or Microservice templates. Use a Module to group related .csproj projects that ship as one logical unit.
- Package: an individual .NET project (a `csproj`) inside a module. Use it as the dependency-management unit (add/remove project references, replace NuGet ref with source).
- Imports: references to other modules — from the same Studio Solution, a local folder, or a NuGet feed. Use Imports when consuming a pre-built ABP module (e.g., Identity Pro) rather than developing it in-tree.
- Profile (Solution Runner): named subset/ordering of applications + their start parameters. The 'Default' profile contains every project; create additional profiles per team/scenario. Profiles persist running apps even when you switch profiles.
- Application (Applications tab): runnable item — C# Application (.NET project, internal or external via .csproj path), CLI Application (PowerShell command), Folder (organizational), or Docker Container (compose service).
- Task (Tasks tab): one-off or initialization work — Manual, Run on Solution Open, or One-time per Computer. Use for DB migrations, seeders, infra bootstrapping.
- Custom Command: PowerShell command bound to a Solution Explorer / Solution Runner / Helm chart context menu, parameterized via Scriban variables (`profile.name`, `application.name`, `metadata.*`, etc.).
- Metadata: hierarchical key-value pairs (Helm Chart > Profile > Solution > Global), used by Kubernetes integration and Custom Commands.
- Secrets: same shape as Metadata but stored locally and excluded from solution files; safe place for connection strings/passwords used by Custom Commands or K8s deployment.
- Monitoring tabs: Overall, Browse, HTTP Requests, Events (filterable by name/app/Direction/Source: Direct/Inbox/Outbox), Exceptions (with full stack incl. inner), Logs (color-coded, filterable, auto-scroll), Tools (Grafana / RabbitMQ / pgAdmin shortcut detection).
- Volo.Abp.Studio.Client.AspNetCore: NuGet package that an ABP application must reference to publish telemetry to Studio's monitoring panes.
- ABP Suite: separate code-generation tool launched from Studio (main menu 'Open' or right-click on solution/module > 'ABP Suite'); generates CRUD pages, supports many-to-many and master-detail, can scaffold from existing DB tables; Studio enforces version match.
- MCP server (`abp mcp-studio`): local CLI bridge that exposes Studio's monitoring + control surface to MCP clients over stdio; distinct from `abp mcp` which talks to the abp.io cloud and requires a license. Studio must be running for `abp mcp-studio` to succeed.
- MCP tools exposed: list_applications, get_exceptions, get_logs, get_requests, get_events, clear_monitor, list_runnable_applications, start_application, stop_application, restart_application, build_application, list_containers, start_containers, stop_containers, get_solution_info, list_modules, list_packages, get_module_dependencies.
- Kubernetes integration: WireGuard VPN auto-installed into the cluster; can append cluster service names to /etc/hosts; service interception redirects traffic for one service to your local debugger; `k8ssuffix` metadata isolates per-developer namespaces; Helm hierarchy is Chart Root > Main Chart > Subchart. Requires Business or Enterprise license.

## Configuration pattern

ABP Studio itself is not configured via AbpModule lifecycle hooks — it is an external desktop tool. The configuration that matters from C# code is on the *target* ABP application: it must reference `Volo.Abp.Studio.Client.AspNetCore` and depend on `[DependsOn(typeof(AbpStudioClientAspNetCoreModule))]` so that telemetry (HTTP requests, exceptions, logs, distributed events) is published to Studio's monitoring panes. Example module wiring:

```csharp
using Volo.Abp.Modularity;
using Volo.Abp.Studio.Client.AspNetCore;

[DependsOn(typeof(AbpStudioClientAspNetCoreModule))]
public class MyAppHttpApiHostModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        // No additional Studio-specific configuration is required for telemetry
        // when the project is started via Solution Runner; Studio injects the
        // connection details through environment variables.
    }
}
```

MCP client wiring is JSON, not C#. For Cursor put `.cursor/mcp.json`, for Claude Desktop put `claude_desktop_config.json` (`~/Library/Application Support/Claude/` on macOS, `%APPDATA%\Claude\` on Windows, `~/.config/Claude/` on Linux), and for VS Code put `.vscode/mcp.json`. All three use the same command shape: `{ "command": "abp", "args": ["mcp-studio"] }` (VS Code uses key `servers` instead of `mcpServers`). Studio can auto-generate the Cursor and VS Code variants when you create the solution.

Custom Commands are configured through Studio's UI (right-click solution root > 'Manage Custom Commands'), not in code. Each command sets Command Name, Display Name, Terminal Command (PowerShell, chain steps with `&&&`), Working Directory (relative to solution), Condition (Scriban expression to gate visibility), and Require Confirmation. Scriban variables available depend on the menu context: Helm chart commands see `profile.name`, `chart.path`, `metadata.*`, `secrets.*`; K8s service commands see `name`, `profile.namespace`, `mainChart.name`; Solution Runner commands see `profile.name`, `application.name`, `application.baseUrl`.

Kubernetes profiles are configured per-cluster/per-namespace and can carry their own metadata/secrets — set `metadata.k8ssuffix` on the profile to suffix the namespace per developer.

## Code examples

### Wire an ASP.NET Core host into Studio's monitoring

_You started the project from `abp new` and want HTTP requests, exceptions, logs, and distributed events to appear in Studio's monitoring tabs._

```csharp
// MyApp.HttpApi.Host/MyAppHttpApiHostModule.cs
using Volo.Abp.Modularity;
using Volo.Abp.AspNetCore.Mvc;
using Volo.Abp.Studio.Client.AspNetCore;

namespace MyApp;

[DependsOn(
    typeof(AbpAspNetCoreMvcModule),
    typeof(AbpStudioClientAspNetCoreModule) // <- enables Studio telemetry
)]
public class MyAppHttpApiHostModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        // No options object required; the module reads connection info
        // from environment variables that Solution Runner injects.
    }
}
```

**Key lines:** The single line that matters is `typeof(AbpStudioClientAspNetCoreModule)` in `[DependsOn]`. Without it the application appears in Solution Runner but its Logs / HTTP Requests / Events / Exceptions tabs stay empty.

### Claude Desktop MCP config for ABP Studio

_Let Claude Desktop drive Studio: list running applications, fetch recent exceptions, restart a failing service._

```json
// ~/Library/Application Support/Claude/claude_desktop_config.json (macOS)
// %APPDATA%\Claude\claude_desktop_config.json (Windows)
// ~/.config/Claude/claude_desktop_config.json (Linux)
{
  "mcpServers": {
    "abp-studio": {
      "command": "abp",
      "args": ["mcp-studio"]
    }
  }
}
```

**Key lines:** `command: abp` requires the ABP CLI on PATH. `args: [mcp-studio]` is the local-Studio bridge — do NOT use `mcp` (that's the cloud-licensed `abp mcp` server). ABP Studio must be running with the solution open; otherwise the bridge errors back to the client.

### Cursor / VS Code MCP config

_Same MCP server but consumed from Cursor or VS Code workspaces; Studio can auto-generate these on solution creation._

```json
// .cursor/mcp.json
{
  "mcpServers": {
    "abp-studio": { "command": "abp", "args": ["mcp-studio"] }
  }
}

// .vscode/mcp.json   (note: 'servers', not 'mcpServers')
{
  "servers": {
    "abp-studio": { "command": "abp", "args": ["mcp-studio"] }
  }
}
```

**Key lines:** VS Code uses the `servers` key while Cursor and Claude Desktop use `mcpServers` — copy/pasting between them silently breaks discovery.

### Custom Command: build & push a Docker image from the chart context

_A team wants a right-click action on a Helm chart that runs their `build-image.ps1` with the chart's metadata._

```text
// 'Manage Custom Commands' dialog values
Command Name:        build-image
Display Name:        Build Image
Terminal Command:    ./build-image.ps1 -Image {{ metadata.image }} -Tag {{ metadata.tag }} -Registry {{ metadata.registry }}
Working Directory:   ./etc/helm/{{ chart.path }}
Condition:           {{ metadata.image != null }}
Require Confirmation: true
```

**Key lines:** `{{ metadata.* }}` is Scriban over the chart's resolved metadata (Helm Chart > Profile > Solution > Global). The `Condition` expression hides the menu item when no image metadata is present, so it doesn't appear on charts that don't build images.

### Solution Runner properties that matter most

_Tuning the run experience for a single API host: skip rebuild, hot-reload, browser auto-refresh, Kubernetes service mapping for monitoring._

```text
// Right-click application > Properties (conceptual; values stored by Studio in profile)
Launch URL:                 https://localhost:44300
Health Check Endpoint:      /health-status
Kubernetes Service Pattern: myapp-api-{{profile.namespace}}
Skip Build:                 true   // when developing UI-only changes against an already-built API
Watch Changes:              true   // dotnet watch behavior
Auto-refresh Browser:       true   // reload the Browse tab when app restarts
```

**Key lines:** Skip Build is the biggest startup-time win when you're iterating on a single tier; Watch Changes turns the application's run into `dotnet watch`; the Kubernetes Service Pattern is what lets the monitoring tab reach a service running in cluster instead of locally.

## Common mistakes

### Configuring the cloud `abp mcp` server in Claude Desktop / Cursor and expecting it to control your local solution.

**Why wrong:** `abp mcp` connects to abp.io's cloud MCP service and requires an active commercial license; it does not see your local Studio. Local control of Solution Runner / monitoring is a different binary path: `abp mcp-studio`.

```text
Use `"args": ["mcp-studio"]` (with the `-studio` suffix). Verify by running `abp help mcp-studio` from a terminal.
```

### Starting an MCP client without ABP Studio running, then debugging why tools time out or return errors.

**Why wrong:** `abp mcp-studio` is a stdio-to-HTTP bridge that proxies into a running Studio instance. With Studio closed there is no HTTP target and the CLI returns errors back to the AI client (which then surfaces as 'tool failed' in Claude/Cursor).

```text
Open ABP Studio and load the solution before invoking any MCP tools. If you closed Studio mid-session, reconnect the MCP client (Claude Desktop > toggle the server, or restart the client).
```

### Adding the application to Solution Runner but forgetting `Volo.Abp.Studio.Client.AspNetCore`.

**Why wrong:** Studio can start the process and show its console logs, but the Monitoring tabs (HTTP Requests, Events, Exceptions, structured Logs) require the in-process telemetry client published by that NuGet. Without it everything in those tabs stays empty and the MCP `get_logs` / `get_requests` / `get_exceptions` tools return nothing for that app.

```xml
Add <PackageReference Include="Volo.Abp.Studio.Client.AspNetCore" /> to the host csproj and [DependsOn(typeof(AbpStudioClientAspNetCoreModule))] on its module.
```

### Pasting the same MCP JSON into Claude Desktop and VS Code and assuming it works in both.

**Why wrong:** VS Code's `mcp.json` uses the top-level key `servers`, while Cursor/Claude Desktop use `mcpServers`. The wrong key is silently ignored and the server never appears.

```text
Use `mcpServers` for Cursor / Claude Desktop, `servers` for VS Code. Let Studio auto-generate the file on solution creation when you can.
```

### Treating an ABP Studio Module the same as an `AbpModule` class in C# code.

**Why wrong:** An ABP Studio Module is a *sub-solution* (a .NET solution containing multiple `csproj` files). An `AbpModule` is a runtime modularity construct (a class deriving from `AbpModule` with `[DependsOn]`). One Studio Module typically contains many `AbpModule` classes.

```text
When the user says 'module', clarify which they mean. In Solution Explorer 'add module' creates a sub-solution; in C# `class FooModule : AbpModule` defines a runtime module.
```

### Trying to use Studio's Kubernetes features without a Business/Enterprise license.

**Why wrong:** Cluster connection, WireGuard tunneling, service interception, Helm chart commands, and the Tools tab's K8s integrations are gated to commercial tiers.

```text
Either upgrade the license or fall back to running services locally via Solution Runner with Docker Compose items.
```

### Two developers sharing a namespace when running the same K8s profile.

**Why wrong:** Without per-developer isolation, both will write into the same in-cluster resources and stomp each other's deploys / interceptions.

```text
Set `metadata.k8ssuffix` to a per-developer value (e.g., the dev's initials). Studio appends it to the namespace, isolating each developer's deploy.
```

### Chaining shell steps in a Custom Command with `&&` or `;`.

**Why wrong:** Studio's Terminal Command field uses `&&&` (triple-ampersand) as the chain separator inside a single command string, not the standard shell `&&`.

```text
Use `step1 &&& step2 &&& step3`. Reserve `&&` for content that should be passed literally to PowerShell (rarely needed).
```

### Skipping the Tasks tab and putting DB migrations / seed steps as regular Applications.

**Why wrong:** Applications are long-running processes; Tasks are one-shot operations with explicit lifecycle (Manual / Run on Solution Open / One-time per Computer). Treating a migration as an Application means 'Stop All' tries to terminate a process that already exited and the run state appears wrong.

```text
Move bootstrap work (DB migrate, seed, infrastructure ensure-created) to the Tasks tab with the appropriate trigger.
```

### Re-running ABP Suite from a browser after Studio updates the framework version.

**Why wrong:** Studio enforces a version match between Suite and the solution's ABP version; an externally launched Suite can be on a different version, generating code that no longer compiles against the solution's runtime.

```text
Always launch Suite from Studio (main menu 'Open' or context menu 'ABP Suite'); accept Studio's update prompt when versions diverge.
```

## Version pins (ABP 10.3)

- Pinned to ABP 10.3 for this reference. Specific 10.3 callouts: (a) Studio targets ABP Framework 10.3.0 and ships with .NET 10 runtime support inherited from the 10.x line; (b) Enhanced Project Wizard with improved optional module selection [uncertain — release-notes phrasing taken from a section that mixes Studio 3.x and ABP 10.3 entries]; (c) `Volo.Abp.Studio.Client.AspNetCore` is the canonical telemetry package name in 10.3, but the exact namespace organization of `AbpStudioClientAspNetCoreModule` may shift in future majors [uncertain]; (d) MCP tool surface listed above (list_applications / get_exceptions / get_logs / get_requests / get_events / clear_monitor / list_runnable_applications / start_application / stop_application / restart_application / build_application / list_containers / start_containers / stop_containers / get_solution_info / list_modules / list_packages / get_module_dependencies) reflects 10.3 docs and is likely to grow in subsequent versions; (e) Kubernetes integration requires a Business or Enterprise license in 10.3; (f) Custom Command chain separator is `&&&`; (g) Linux is not explicitly listed as a first-class supported platform in the 10.3 installation page (Windows via winget and macOS via brew are documented) [uncertain — Linux works in practice for many users per community reports but is not officially documented]; (h) `abp mcp-studio` (local) and `abp mcp` (cloud) are distinct subcommands — do not assume they alias to each other.

## Cross-references

**Phase 1 references:**
- references/tools/cli.md — triggered by any mention of `abp` / `abp mcp-studio` / `abp new`; Studio shells out to the CLI for installs, MCP bridge, and project creation.
- references/tools/suite.md — triggered when the user opens Suite from Studio for CRUD generation; covers Suite-side configuration, templates, and code customization.
- references/templates/microservice.md — triggered when Studio's Kubernetes/Helm/profile features come up; the microservice template is the primary consumer of those features.
- references/templates/layered.md — triggered when discussing Solution Explorer module/package shapes for a typical DDD solution.
- references/framework/architecture-modularity.md — triggered when explaining the ABP Studio Module concept vs. the framework's `AbpModule`; they are related but distinct (Studio Module = sub-solution, AbpModule = runtime module).

**External docs:**
- [Model Context Protocol specification](https://modelcontextprotocol.io/) — Defines the MCP transport (stdio + JSON-RPC) and tool/resource semantics that `abp mcp-studio` implements; useful when debugging client connection issues.
- [Claude Desktop MCP configuration guide](https://support.claude.com/en/articles/10949351-getting-started-with-local-mcp-servers-on-claude-desktop) — Canonical reference for `claude_desktop_config.json` location and schema; Studio's MCP page shows the ABP-specific snippet but Anthropic owns the file format.
- [WireGuard](https://www.wireguard.com/) — Studio uses WireGuard to tunnel into a Kubernetes cluster; the WireGuard docs explain the on-cluster install Studio performs automatically.
- [Helm](https://helm.sh/docs/) — Studio's Kubernetes integration organizes deploys as Helm Chart Root > Main Chart > Subchart; understanding Helm semantics is required to author Custom Commands against charts.
- [mkcert](https://github.com/FiloSottile/mkcert) — Auto-installed by Studio for local HTTPS dev; users hit prompts to add the local CA to their trust store.
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) — Pre-requisite for Solution Runner Docker items and for Kubernetes integration.
- [Scriban template language](https://github.com/scriban/scriban) — Custom Command Conditions and template expansions use Scriban syntax (`{{ ... }}`).
- [.NET SDK](https://dotnet.microsoft.com/download) — Auto-installed by Studio on first run, but users sometimes want a specific channel (LTS vs current) — official .NET docs are the source of truth.

## Sources (ABP 10.3)

- https://abp.io/docs/10.3/studio — ABP Studio overview: cross-platform desktop application for ABP/.NET developers; lists fundamentals (Solution Explorer, Solution Runner, Monitoring, Kubernetes), ABP Suite integration, Custom Commands, MCP support.
- https://abp.io/docs/10.3/studio/installation — Installation prerequisites (Docker, winget on Windows, brew on macOS), auto-installed deps (.NET SDK, Node.js, ABP CLI, mkcert, WireGuard), abp.io login requirement, theme/update flow.
- https://abp.io/docs/10.3/studio/solution-explorer — Solution Explorer panel: Solution > Module > Package hierarchy, module/package operations, IDE integration, version management, build/analyze actions.
- https://abp.io/docs/10.3/studio/running-applications — Solution Runner with Applications/Tasks tabs, profiles, C# / CLI / Folder / Docker item types, properties dialog, start order management, custom commands.
- https://abp.io/docs/10.3/studio/monitoring-applications — Centralized monitoring: Overall, Browse, HTTP Requests, Events, Exceptions, Logs, Tools tabs; requires Volo.Abp.Studio.Client.AspNetCore package on the app.
- https://abp.io/docs/10.3/studio/model-context-protocol — Built-in MCP server bridge (`abp mcp-studio`) over stdio->HTTP; client config for Cursor / Claude Desktop / VS Code; tool list (monitoring, app control, container control, solution structure).
- https://abp.io/docs/10.3/studio/kubernetes — Kubernetes integration: WireGuard VPN tunneling, service interception for local debugging, profiles with metadata/secrets, Helm chart hierarchy, Business/Enterprise license required.
- https://abp.io/docs/10.3/studio/working-with-suite — Suite launch shortcut from Studio main menu and Solution Explorer context menu; module selection prompt; version compatibility check.
- https://abp.io/docs/10.3/studio/custom-commands — Custom command properties (Name, Display Name, Terminal Command, Working Directory, Condition via Scriban, Require Confirmation); managed via 'Manage Custom Commands' dialog; Scriban variables differ by context (Helm Chart, K8s Service, Solution Runner).
- https://abp.io/docs/10.3/studio/concepts — Conceptual model: Solution > Module > Package mapping to .NET (Module=VS Solution, Package=csproj); Metadata and Secrets with hierarchical priority (Helm Chart > Profile > Solution > Global).

Last verified: 2026-05-10
