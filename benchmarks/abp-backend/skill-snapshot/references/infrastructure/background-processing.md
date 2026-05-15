# Infrastructure — Background Jobs & Workers

> Comprehensive guide to ABP's background-processing infrastructure: one-shot Background Jobs and recurring Background Workers, the default storage-based provider, and the swappable Hangfire, Quartz, RabbitMQ, and TickerQ providers wired through AbpModule dependencies.

## When to load this reference

Load when the user asks about: enqueueing fire-and-forget work (sending email, generating reports, processing uploads); choosing between Hangfire vs Quartz vs RabbitMQ vs TickerQ for ABP; running periodic/cron tasks (cleanup workers, passive-user checks, expiration sweeps); IBackgroundJobManager, IBackgroundWorkerManager, AsyncBackgroundJob<TArgs>, AsyncPeriodicBackgroundWorkerBase, BackgroundJobPriority; configuring the Hangfire dashboard or securing it; persisting Quartz schedules; splitting a separate worker process from web apps (IsJobExecutionEnabled = false); RabbitMQ queue isolation across apps; TickerQ source-generator-based scheduling; retry/timeout tuning for failed jobs.

**Audience:** ABP developers building any solution template (Single-Layer, Layered, Microservice) who need to offload work from request threads or run scheduled jobs, and DevOps engineers deciding which job-engine package to plug in for production scale, persistence, and observability.

## Key concepts

- IBackgroundJob<TArgs> / IAsyncBackgroundJob<TArgs> (Volo.Abp.BackgroundJobs.Abstractions): contract for a background job class typed by an args/payload object. Implement when you want a strongly-typed handler for a serialized argument.
- AsyncBackgroundJob<TArgs> / BackgroundJob<TArgs> (Volo.Abp.BackgroundJobs): convenience base classes; override ExecuteAsync(TArgs) (or Execute(TArgs)). Reach for these for 90% of jobs - they bring Logger, ILazyServiceProvider, and CancellationToken integration.
- IBackgroundJobManager (Volo.Abp.BackgroundJobs): the queue-side API. Call EnqueueAsync(args, priority, delay) from application services to schedule work.
- BackgroundJobPriority (Volo.Abp.BackgroundJobs): enum (Low, BelowNormal, Normal, AboveNormal, High). Used as the priority parameter to EnqueueAsync; honored only by providers that support priorities (default + TickerQ + Hangfire queues).
- IDynamicBackgroundJobManager (Volo.Abp.BackgroundJobs): register handlers by string name at runtime; useful for plugin scenarios or when payloads come from JSON without compile-time types.
- [BackgroundJobName("name")] attribute: overrides the default fully-qualified-type-name used as the job key on storage and across providers. Required when renaming/refactoring args classes that already have queued items in flight.
- IBackgroundJobStore (Volo.Abp.BackgroundJobs): persistence contract for the default provider; the bundled Volo.Abp.BackgroundJobs.EntityFrameworkCore / .MongoDB packages implement it.
- IBackgroundWorker (Volo.Abp.BackgroundWorkers): contract for a long-running singleton worker that starts with the host. Implement directly only for unusual lifecycle needs.
- BackgroundWorkerBase (Volo.Abp.BackgroundWorkers): abstract base providing Logger, ServiceProvider, and StartAsync/StopAsync hooks.
- AsyncPeriodicBackgroundWorkerBase / PeriodicBackgroundWorkerBase (Volo.Abp.BackgroundWorkers): base for periodic tasks driven by AbpAsyncTimer. Override DoWorkAsync(PeriodicBackgroundWorkerContext) and set Timer.Period (ms). The framework opens a fresh service-scope per tick.
- IBackgroundWorkerManager (Volo.Abp.BackgroundWorkers): the runtime registry; reach for it via context.AddBackgroundWorkerAsync<T>() inside OnApplicationInitializationAsync.
- IDynamicBackgroundWorkerManager (Volo.Abp.BackgroundWorkers): add/remove/update workers at runtime by name and DynamicBackgroundWorkerSchedule.
- AbpBackgroundJobOptions: master switch (IsJobExecutionEnabled), GetBackgroundJobName delegate, retry-defaults; configured in PreConfigureServices/ConfigureServices.
- AbpBackgroundJobWorkerOptions: tunables for the default polling provider (JobPollPeriod, MaxJobFetchCount, DefaultFirstWaitDuration, DefaultWaitFactor, DefaultTimeout, ApplicationName).
- AbpBackgroundWorkerOptions: master IsEnabled switch for the entire worker subsystem.
- AbpBackgroundJobsHangfireModule (Volo.Abp.BackgroundJobs.Hangfire): swap default IBackgroundJobManager for Hangfire; combine with AddHangfire() and UseAbpHangfireDashboard.
- AbpBackgroundWorkersHangfireModule + HangfireBackgroundWorkerBase: cron-based recurring workers (set RecurringJobId and CronExpression).
- AbpBackgroundJobsQuartzModule + AbpQuartzOptions/AbpBackgroundJobQuartzOptions: Quartz-backed job manager with persistent JobStore and configurable RetryCount/RetryIntervalMillisecond/RetryStrategy.
- AbpBackgroundWorkersQuartzModule + QuartzBackgroundWorkerBase + QuartzPeriodicBackgroundWorkerAdapter: full Quartz scheduling for workers (cron triggers, misfire policies). AutoRegister via AbpBackgroundWorkerQuartzOptions.IsAutoRegisterEnabled.
- AbpBackgroundJobsRabbitMqModule + AbpRabbitMqOptions + AbpRabbitMqBackgroundJobOptions: RabbitMQ-backed distributed queue with DefaultQueueNamePrefix and PrefetchCount; ideal for separating producer and consumer processes.
- AbpBackgroundJobsTickerQModule + AbpBackgroundJobsTickerQOptions: TickerQ provider using source generators; AddJobConfiguration<T>(AbpBackgroundJobsTimeTickerConfiguration{Retries, RetryIntervals, Priority}).
- AbpBackgroundWorkersTickerQModule + AbpBackgroundWorkersTickerQOptions + ICronTickerManager<CronTickerEntity>: TickerQ-backed cron scheduling with EF Core persistence and a real-time dashboard.

## Configuration pattern

Background-processing wiring lives in your application's ABP module. The default provider works out of the box; switching providers means (1) installing the package, (2) adding the [DependsOn] entry, (3) registering the underlying engine in ConfigureServices and starting/dashboarding it in OnApplicationInitialization.

Default provider (no extra package):

```csharp
[DependsOn(typeof(AbpBackgroundJobsModule), typeof(AbpBackgroundWorkersModule))]
public class MyAppModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        Configure<AbpBackgroundJobOptions>(o =>
        {
            // Set false on web nodes when a separate worker process owns execution.
            o.IsJobExecutionEnabled = true;
        });
        Configure<AbpBackgroundJobWorkerOptions>(o =>
        {
            o.JobPollPeriod = 5000;            // ms between polls
            o.MaxJobFetchCount = 1000;         // batch size per poll
            o.DefaultFirstWaitDuration = 60;   // seconds before first retry
            o.DefaultWaitFactor = 2.0;         // exponential backoff multiplier
            o.DefaultTimeout = 172800;         // total seconds before giving up (2 days)
            o.ApplicationName = "MyApp";       // isolate jobs per app on shared store
        });
        Configure<AbpBackgroundWorkerOptions>(o => o.IsEnabled = true);
    }

    public override async Task OnApplicationInitializationAsync(
        ApplicationInitializationContext context)
    {
        await context.AddBackgroundWorkerAsync<PassiveUserCheckerWorker>();
    }
}
```

Hangfire provider (jobs + workers):

```csharp
[DependsOn(
    typeof(AbpBackgroundJobsHangfireModule),
    typeof(AbpBackgroundWorkersHangfireModule))]
public class MyAppModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        var configuration = context.Services.GetConfiguration();
        context.Services.AddHangfire(cfg =>
            cfg.UseSqlServerStorage(configuration.GetConnectionString("Default")));
    }

    public override void OnApplicationInitialization(
        ApplicationInitializationContext context)
    {
        var app = context.GetApplicationBuilder();
        app.UseAbpHangfireDashboard("/hangfire", options =>
        {
            options.AsyncAuthorization = new[]
            {
                new AbpHangfireAuthorizationFilter(
                    requiredPermissionName: "MyApp.HangfireDashboard")
            };
        });
    }
}
```

Quartz provider (persistent JobStore):

```csharp
[DependsOn(
    typeof(AbpBackgroundJobsQuartzModule),
    typeof(AbpBackgroundWorkersQuartzModule))]
public class MyAppModule : AbpModule
{
    public override void PreConfigureServices(ServiceConfigurationContext context)
    {
        var configuration = context.Services.GetConfiguration();
        PreConfigure<AbpQuartzOptions>(options =>
        {
            options.Configurator = configure =>
            {
                configure.UsePersistentStore(store =>
                    store.UseSqlServer(configuration.GetConnectionString("Quartz")));
            };
        });
    }

    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        Configure<AbpBackgroundJobQuartzOptions>(o =>
        {
            o.RetryCount = 3;
            o.RetryIntervalMillisecond = 3000;
        });
        Configure<AbpBackgroundWorkerQuartzOptions>(o =>
        {
            o.IsAutoRegisterEnabled = true; // auto-discover IQuartzBackgroundWorker
        });
    }
}
```

RabbitMQ provider:

```csharp
[DependsOn(typeof(AbpBackgroundJobsRabbitMqModule))]
public class MyAppModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        Configure<AbpRabbitMqOptions>(options =>
        {
            // Reads RabbitMQ:Connections:Default from appsettings.json by default.
        });
        Configure<AbpRabbitMqBackgroundJobOptions>(options =>
        {
            options.DefaultQueueNamePrefix = "my_app_jobs.";
            options.PrefetchCount = 1; // 1 in-flight message per consumer
        });
    }
}
```

TickerQ provider:

```csharp
[DependsOn(
    typeof(AbpBackgroundJobsTickerQModule),
    typeof(AbpBackgroundWorkersTickerQModule))]
public class MyAppModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        context.Services.AddTickerQ(x => { /* engine options */ });
        Configure<AbpBackgroundJobsTickerQOptions>(o =>
        {
            o.AddJobConfiguration<EmailSendingJob>(
                new AbpBackgroundJobsTimeTickerConfiguration
                {
                    Retries = 3,
                    RetryIntervals = new[] { 30, 60, 120 },
                    Priority = TickerTaskPriority.High
                });
        });
    }

    public override async Task OnApplicationInitializationAsync(
        ApplicationInitializationContext context)
    {
        context.GetHost().UseAbpTickerQ(qStartMode: TickerQStartMode.Immediate);
    }
}
```

## Code examples

1. Title: Define and enqueue a typed background job (default provider).
Scenario: Send a confirmation email asynchronously after an order is placed.
```csharp
using Volo.Abp.BackgroundJobs;
using Volo.Abp.DependencyInjection;
public class EmailSendingArgs
{
public string EmailAddress { get; set; }
public string Subject { get; set; }
public string Body { get; set; }
}
public class EmailSendingJob : AsyncBackgroundJob<EmailSendingArgs>, ITransientDependency
{
private readonly IEmailSender _emailSender;
public EmailSendingJob(IEmailSender emailSender) => _emailSender = emailSender;
public override async Task ExecuteAsync(EmailSendingArgs args)
{
await _emailSender.SendAsync(args.EmailAddress, args.Subject, args.Body);
}
}
public class OrderAppService : ApplicationService, IOrderAppService
{
private readonly IBackgroundJobManager _jobManager;
public OrderAppService(IBackgroundJobManager jobManager) => _jobManager = jobManager;
public async Task PlaceOrderAsync(PlaceOrderInput input)
{
// ...persist order...
await _jobManager.EnqueueAsync(
new EmailSendingArgs
{
EmailAddress = input.Email,
Subject = "Order received",
Body = $"Order {input.OrderNo} confirmed."
},
priority: BackgroundJobPriority.High,
delay: TimeSpan.FromSeconds(10));
}
}
```
Key lines: AsyncBackgroundJob<TArgs> gives Logger and avoids hand-rolled try/catch. The args class is the only thing serialized to storage; keep it primitive-typed. EnqueueAsync's delay is the minimum wait before the first dispatch attempt; priority is only honored by providers that implement queues.
2. Title: Recurring periodic worker.
Scenario: Every 30 minutes, mark users with no logins for 30+ days as 'passive'.
```csharp
using Volo.Abp.BackgroundWorkers;
using Volo.Abp.Threading;
public class PassiveUserCheckerWorker : AsyncPeriodicBackgroundWorkerBase
{
public PassiveUserCheckerWorker(
AbpAsyncTimer timer,
IServiceScopeFactory serviceScopeFactory)
: base(timer, serviceScopeFactory)
{
Timer.Period = 30 * 60 * 1000; // 30 minutes
}
protected override async Task DoWorkAsync(PeriodicBackgroundWorkerContext workerContext)
{
Logger.LogInformation("Starting passive user check...");
var userRepo = workerContext.ServiceProvider
.GetRequiredService<IRepository<AppUser, Guid>>();
var threshold = DateTime.UtcNow.AddDays(-30);
var passive = await userRepo.GetListAsync(u => u.LastLoginTime < threshold);
foreach (var u in passive) u.IsPassive = true;
await userRepo.UpdateManyAsync(passive);
Logger.LogInformation($"Marked {passive.Count} users as passive.");
}
}
// In your module:
public override async Task OnApplicationInitializationAsync(
ApplicationInitializationContext context)
{
await context.AddBackgroundWorkerAsync<PassiveUserCheckerWorker>();
}
```
Key lines: Always resolve scoped services (repositories, DbContext) from PeriodicBackgroundWorkerContext.ServiceProvider, never via constructor. Workers are singletons and would otherwise capture a disposed scope. Timer.Period must be set in the constructor before the framework calls StartAsync.
3. Title: Hangfire-backed worker on a cron schedule.
Scenario: Daily log archival driven by Hangfire's recurring-job machinery and visible in the dashboard.
```csharp
using Hangfire;
using Volo.Abp.BackgroundWorkers.Hangfire;
public class DailyLogArchiveWorker : HangfireBackgroundWorkerBase
{
public DailyLogArchiveWorker()
{
RecurringJobId = nameof(DailyLogArchiveWorker);
CronExpression = Cron.Daily(); // 00:00 every day
Queue = "maintenance";          // optional Hangfire queue
}
public override async Task DoWorkAsync(CancellationToken cancellationToken = default)
{
Logger.LogInformation("Archiving yesterday's logs...");
// ...do the work...
await Task.CompletedTask;
}
}
public override async Task OnApplicationInitializationAsync(
ApplicationInitializationContext context)
{
await context.AddBackgroundWorkerAsync<DailyLogArchiveWorker>();
}
```
Key lines: Setting CronExpression turns the worker into a Hangfire RecurringJob (visible at /hangfire). Without it, HangfireBackgroundWorkerBase falls back to AbpAsyncTimer-style fixed-period execution.
4. Title: Quartz worker with explicit cron trigger and misfire handling.
Scenario: Roll up nightly metrics at 02:30 with strict cron semantics and persistent triggers.
```csharp
using Quartz;
using Volo.Abp.BackgroundWorkers.Quartz;
public class NightlyMetricsRollupWorker : QuartzBackgroundWorkerBase
{
public NightlyMetricsRollupWorker()
{
JobDetail = JobBuilder.Create<NightlyMetricsRollupWorker>()
.WithIdentity(nameof(NightlyMetricsRollupWorker))
.Build();
Trigger = TriggerBuilder.Create()
.WithIdentity(nameof(NightlyMetricsRollupWorker))
.WithCronSchedule("0 30 2 * * ?",          // 02:30 every day
x => x.WithMisfireHandlingInstructionFireAndProceed())
.Build();
}
public override async Task Execute(IJobExecutionContext context)
{
Logger.LogInformation("Rolling up metrics for {Date}", DateTime.UtcNow.Date);
// ...do the work...
await Task.CompletedTask;
}
}
```
Key lines: QuartzBackgroundWorkerBase is auto-registered when AbpBackgroundWorkerQuartzOptions.IsAutoRegisterEnabled is true (default). Misfire policies survive process restarts when persistent JobStore is configured.
5. Title: Run jobs in a separate process while web nodes only enqueue.
Scenario: Two deployments share one storage: API pods enqueue, a dedicated worker pod processes.
```csharp
// Web/API module:
Configure<AbpBackgroundJobOptions>(o => o.IsJobExecutionEnabled = false);
Configure<AbpBackgroundWorkerOptions>(o => o.IsEnabled = false);
// Worker host module:
Configure<AbpBackgroundJobOptions>(o => o.IsJobExecutionEnabled = true);
Configure<AbpBackgroundWorkerOptions>(o => o.IsEnabled = true);
Configure<AbpBackgroundJobWorkerOptions>(o =>
{
o.ApplicationName = "MyApp.Worker"; // isolate from other apps on the store
o.JobPollPeriod = 2000;             // tighter polling for the dedicated worker
});
```
Key lines: IsJobExecutionEnabled = false stops dequeue/execute on the web tier without touching producer code. ApplicationName partitions jobs in the shared IBackgroundJobStore so two deployments don't steal each other's work.
6. Title: Dynamic background job by name (no compile-time args type).
Scenario: A plugin sends a job whose payload is JSON; the host registers a handler at startup.
```csharp
using Volo.Abp.BackgroundJobs;
public class DynamicJobRegistrar : ITransientDependency
{
private readonly IDynamicBackgroundJobManager _dynamic;
public DynamicJobRegistrar(IDynamicBackgroundJobManager dynamic) => _dynamic = dynamic;
public async Task RegisterAsync()
{
_dynamic.RegisterHandler("ProcessOrder", async (ctx, ct) =>
{
var json = ctx.JsonData; // raw JSON payload
// parse + handle...
await Task.CompletedTask;
});
await _dynamic.EnqueueAsync("ProcessOrder",
new { OrderId = "ORD-001", Total = 99.95m });
}
}
```
Key lines: IDynamicBackgroundJobManager is the escape hatch when args types live outside the host assembly. The handler receives JsonData; deserialize manually.
## Common mistakes

- 1. Mistake: Constructor-injecting a scoped service (DbContext, IRepository) into a periodic worker. Why wrong: Workers are singletons; a captured scoped dependency is disposed after first tick or leaks across requests, producing 'Cannot access a disposed object' or stale-tracking bugs. Correct pattern: Resolve from PeriodicBackgroundWorkerContext.ServiceProvider inside DoWorkAsync.
- ```csharp
- protected override async Task DoWorkAsync(PeriodicBackgroundWorkerContext ctx)
- {
- var repo = ctx.ServiceProvider.GetRequiredService<IRepository<AppUser, Guid>>();
- // use repo here
- }
- ```
- 2. Mistake: Catching exceptions inside ExecuteAsync to 'be safe'. Why wrong: ABP relies on the exception to trigger its retry-with-exponential-backoff loop; swallowing it marks the job as completed and you silently lose the work. Correct pattern: Let exceptions propagate; only catch if you have a transient error you can correct in-band, then rethrow.
- 3. Mistake: Putting entity references or large blobs in the args class. Why wrong: Args are JSON-serialized into the store; tracked entities, lazy collections, or megabyte payloads cause serializer crashes or bloat the queue table. Correct pattern: Pass IDs and re-load inside the job.
- 4. Mistake: Renaming the args class without [BackgroundJobName("OldName")]. Why wrong: The default job name is the full type name; in-flight queued items become orphaned and never execute. Correct pattern: Add [BackgroundJobName("OriginalNamespace.OriginalClassName")] before deploying the rename, or drain the queue first.
- 5. Mistake: Running both web nodes and a worker pod with IsJobExecutionEnabled = true on shared storage without setting AbpBackgroundJobWorkerOptions.ApplicationName per app. Why wrong: Two app names compete for the same rows, leading to duplicate execution and visibility-lock thrash. Correct pattern: Either set IsJobExecutionEnabled = false on the web tier, or assign a distinct ApplicationName per logical worker pool.
- 6. Mistake: Exposing /hangfire without authorization. Why wrong: The default Hangfire dashboard is open to localhost only; once you put the app behind a reverse proxy it becomes world-readable, leaking job args (including emails, IDs). Correct pattern: Always pass AbpHangfireAuthorizationFilter (or a permission-checking filter) to UseAbpHangfireDashboard.
- 7. Mistake: Switching to RabbitMQ-backed jobs and assuming Hangfire-style guaranteed-once semantics. Why wrong: AMQP gives at-least-once; if PrefetchCount is high and the consumer crashes mid-work, the same message redelivers. Correct pattern: Make handlers idempotent (deduplicate by an ID column or distributed lock), and keep PrefetchCount = 1 for non-idempotent jobs.
- 8. Mistake: Using Hangfire cron syntax inside Quartz (or vice versa). Why wrong: Hangfire uses NCrontab (5-field) while Quartz uses a 7-field cron with seconds and a '?' day-of-week placeholder; the same string means different schedules or fails to parse. Correct pattern: Always check which provider you're configuring; NightlyMetricsRollupWorker uses '0 30 2 * * ?' for Quartz, but a Hangfire RecurringJob would use '30 2 * * *'.
- 9. Mistake: Forgetting ICurrentTenant.Change(tenantId) inside a worker that operates per-tenant. Why wrong: Workers run with no tenant context; multi-tenant queries with IDataFilter on TenantId silently return zero rows or the host tenant's data. Correct pattern: Wrap per-tenant work in using (_currentTenant.Change(tenantId)) { ... }.
- 10. Mistake: Calling EnqueueAsync inside an EF Core transaction expecting transactional consistency. Why wrong: The default storage-based provider writes to the same DbContext, but Hangfire/RabbitMQ/Quartz/TickerQ use their own connections, so the message goes out before your tx commits and a downstream rollback leaves a phantom job. Correct pattern: Use the outbox pattern (Volo.Abp.EventBus.Outbox) or enqueue inside an OnUowCompleted hook.
- 11. Mistake: Setting CronExpression on HangfireBackgroundWorkerBase but never installing AbpBackgroundWorkersHangfireModule on the running process. Why wrong: The worker silently falls back to default polling and your cron is ignored. Correct pattern: Verify the worker module is referenced in the host that actually runs the worker.
- 12. Mistake: Treating BackgroundJobPriority as a cross-provider contract. Why wrong: Only the default provider and a few others honor it; RabbitMQ requires per-queue priority arguments, Quartz uses Trigger priority, Hangfire uses queue ordering. Correct pattern: Document priority semantics per provider and don't rely on Low vs High when migrating.
## Version pins (ABP 10.3)

- ABP 10.3 specifics: (1) AbpBackgroundJobWorkerOptions defaults are JobPollPeriod=5000ms, MaxJobFetchCount=1000, DefaultFirstWaitDuration=60s, DefaultWaitFactor=2.0, DefaultTimeout=172800s (2 days) - subject to change in 11.x. (2) The Hangfire package id is Volo.Abp.BackgroundJobs.HangFire (note the capital F in 'HangFire'); some examples in older docs use lowercase. (3) AbpBackgroundJobQuartzOptions.RetryStrategy is async (Func<int, IJobExecutionContext, Exception, Task>) in 10.3; pre-9.x docs show a synchronous signature. (4) TickerQ integration is documented as released in 10.3 with the 10.4 docs noting it as 'preview'; APIs (AddJobConfiguration<T>, AbpBackgroundJobsTimeTickerConfiguration property names) [uncertain] - may rename when TickerQ leaves preview. (5) IDynamicBackgroundJobManager and IDynamicBackgroundWorkerManager are present in 10.3 but their exact public surface for cancellation/rescheduling is [uncertain] from the public docs alone. (6) AbpRabbitMqBackgroundJobOptions.PrefetchCount default is 1 in 10.3; older versions defaulted to a higher value. (7) UseAbpHangfireDashboard signature accepts an Action<DashboardOptions> overload; pre-9.x used a different option type. (8) QuartzPeriodicBackgroundWorkerAdapter is the 10.x mechanism that lets a PeriodicBackgroundWorkerBase run on Quartz unchanged; the adapter class name [uncertain] for future versions. (9) The Volo.Abp.BackgroundJobs.EntityFrameworkCore module is the storage default for templates that include EF Core; switching to MongoDB swaps in Volo.Abp.BackgroundJobs.MongoDB. (10) Setting Configure<AbpBackgroundJobOptions>(o => o.IsJobExecutionEnabled = false) only affects the default provider's worker loop; Hangfire/Quartz/RabbitMQ require their own server-side disablement (e.g., not calling AddHangfireServer).

## Cross-references

**Phase 1 references:**
- - references/framework/architecture-modularity.md - triggered when explaining where [DependsOn(AbpBackgroundJobsHangfireModule)] and PreConfigureServices/ConfigureServices/OnApplicationInitialization fit; the whole wiring lives inside an AbpModule.
- - references/infrastructure/eventing.md - cross-link when readers ask 'should I use a background job or publish a distributed event?' and when discussing RabbitMQ as the underlying transport for both subsystems.
- - references/framework/data-ef-core.md - triggered when the user asks how AbpBackgroundJobsEntityFrameworkCoreModule provides IBackgroundJobStore, or when configuring the Quartz ADO JobStore against EF Core's connection string.
- - references/framework/fundamentals.md - cross-link for the 'don't constructor-inject scoped services into singleton workers' rule and PeriodicBackgroundWorkerContext.ServiceProvider pattern.
- - references/framework/multi-tenancy.md - load when discussing how jobs serialize the current tenant in args and how ICurrentTenant.Change(...) is required inside DoWorkAsync to scope multi-tenant queries.
- - references/infrastructure/caching-and-locking.md - cross-link from the clustering section: a single-leader worker requires IAbpDistributedLock to avoid double execution across instances.

**External docs:**
- - Hangfire documentation, https://docs.hangfire.io/, for storage providers (SqlServer, PostgreSql, Redis), dashboard authorization, queues, and recurring-job APIs that ABP wraps.
- - Quartz.NET documentation, https://www.quartzscheduler.net/documentation/, for cron expressions, trigger builders, ADO JobStore schema scripts, and clustering.
- - Quartz.NET cron tutorial, https://www.quartzscheduler.net/documentation/quartz-3.x/tutorial/crontrigger.html, for the specific cron-expression dialect Quartz uses (different from Hangfire's NCrontab).
- - TickerQ documentation, https://tickerq.arcenox.com/, for source-generator-driven scheduling, dashboard, and EF Core persistence configuration that AbpBackgroundJobsTickerQModule layers on.
- - RabbitMQ .NET client, https://www.rabbitmq.com/dotnet.html, for connection-recovery semantics, exchanges/queues that AbpRabbitMqOptions configures.
- - ASP.NET Core hosted services, https://learn.microsoft.com/aspnet/core/fundamentals/host/hosted-services, because ABP's IBackgroundWorkerManager runs as a hosted service under the host.
- - .NET CancellationToken docs, https://learn.microsoft.com/dotnet/api/system.threading.cancellationtoken, for graceful-shutdown patterns referenced by ICancellationTokenProvider.
- - EF Core connection-resiliency docs, https://learn.microsoft.com/ef/core/miscellaneous/connection-resiliency, when configuring Quartz JobStore retries against transient SQL failures.

## Sources (ABP 10.3)

- [https://abp.io/docs/10.3/framework/infrastructure/background-jobs](https://abp.io/docs/10.3/framework/infrastructure/background-jobs)
- [https://abp.io/docs/10.3/framework/infrastructure/background-jobs/hangfire](https://abp.io/docs/10.3/framework/infrastructure/background-jobs/hangfire)
- [https://abp.io/docs/10.3/framework/infrastructure/background-jobs/rabbitmq](https://abp.io/docs/10.3/framework/infrastructure/background-jobs/rabbitmq)
- [https://abp.io/docs/10.3/framework/infrastructure/background-jobs/quartz](https://abp.io/docs/10.3/framework/infrastructure/background-jobs/quartz)
- [https://abp.io/docs/10.3/framework/infrastructure/background-jobs/tickerq](https://abp.io/docs/10.3/framework/infrastructure/background-jobs/tickerq)
- [https://abp.io/docs/10.3/framework/infrastructure/background-workers](https://abp.io/docs/10.3/framework/infrastructure/background-workers)
- [https://abp.io/docs/10.3/framework/infrastructure/background-workers/quartz](https://abp.io/docs/10.3/framework/infrastructure/background-workers/quartz)
- [https://abp.io/docs/10.3/framework/infrastructure/background-workers/hangfire](https://abp.io/docs/10.3/framework/infrastructure/background-workers/hangfire)
- [https://abp.io/docs/10.3/framework/infrastructure/background-workers/tickerq](https://abp.io/docs/10.3/framework/infrastructure/background-workers/tickerq)

Last verified: 2026-05-10
