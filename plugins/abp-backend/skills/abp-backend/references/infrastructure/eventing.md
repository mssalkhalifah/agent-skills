# Infrastructure — Event Bus (local + distributed)

> Covers ABP's Event Bus infrastructure - the local in-process event bus and the distributed event bus with pluggable providers (RabbitMQ, Kafka, Azure Service Bus, Rebus) including the transactional outbox/inbox patterns.

## When to load this reference

- User asks how to publish or subscribe to events in ABP (ILocalEventBus / IDistributedEventBus).
- User wants cross-service communication between ABP microservices or modules in a modular monolith.
- User asks about EntityCreatedEventData / EntityUpdatedEto / EntityDeletedEto and pre-built entity events.
- User is configuring RabbitMQ, Kafka, Azure Service Bus, or Rebus with ABP.
- User asks about transactional outbox / inbox for guaranteed event delivery in ABP.
- User asks why event handlers run inside or outside the Unit of Work, or why a handler exception rolls back their transaction.
- User is implementing AddLocalEvent / AddDistributedEvent on an AggregateRoot.
- User wants to control event handler execution order via [LocalEventHandlerOrder].
- User is choosing between local and distributed buses for an in-process scenario.
- User is using dynamic (string-named) events without CLR types.

**Audience:** ABP application and module developers wiring event-driven communication - either in-process between services in a single host, or across microservices and modular-monolith boundaries via a real message broker.

## Key concepts

- ILocalEventBus (Volo.Abp.EventBus.Local): In-process event bus that publishes events to handlers running in the same application process. Use when sender and handler live in the same host and you want transactional, non-serialized event passing.
- ILocalEventHandler<TEvent> (Volo.Abp.EventBus): Marker interface implemented by classes that handle local events. Implement HandleEventAsync(TEvent eventData); ABP auto-discovers handlers when the class is registered as a dependency (e.g., ITransientDependency).
- IDistributedEventBus (Volo.Abp.EventBus.Distributed): Bus that publishes events across service boundaries. Falls back to LocalDistributedEventBus when no provider is configured, allowing the same code to evolve from in-process to cross-process delivery.
- IDistributedEventHandler<TEvent> (Volo.Abp.EventBus.Distributed): Handler interface for distributed events. With real providers (RabbitMQ/Kafka/Azure SB), ABP subscribes automatically and ACKs the message after a successful HandleEventAsync.
- AggregateRoot.AddLocalEvent / AddDistributedEvent (Volo.Abp.Domain.Entities): Methods on aggregate roots used to enqueue events on the entity itself; published by ABP's UoW after the entity persists (EF Core SaveChanges or MongoDB Insert/Update/DeleteAsync).
- ETO (Event Transfer Object) (Volo.Abp.EventBus.Distributed): Convention for distributed event payload classes - serializable POCO with default constructor, no circular references. Optionally annotated with [EventName("...")] (Volo.Abp.EventBus); otherwise the FQCN is used.
- Pre-defined entity events: EntityCreatedEventData<T> / EntityUpdatedEventData<T> / EntityDeletedEventData<T> / EntityChangedEventData<T> (Volo.Abp.Domain.Entities.Events) for local; EntityCreatedEto<T> / EntityUpdatedEto<T> / EntityDeletedEto<T> (Volo.Abp.EventBus.Distributed) for distributed.
- AbpDistributedEntityEventOptions (Volo.Abp.EventBus.Distributed): Selects which entities auto-publish entity events (AutoEventSelectors) and registers ETO mappings (EtoMappings.Add<TEntity, TEto>()).
- AbpEventBusBoxesOptions (Volo.Abp.EventBus.Distributed): Tunes outbox/inbox processing: BatchPublishOutboxEvents, PeriodTimeSpan, InboxProcessorFailurePolicy.
- IHasEventOutbox / IHasEventInbox (Volo.Abp.EventBus.Distributed): Interfaces a DbContext implements to participate in transactional outbox/inbox; companion DbSets OutgoingEventRecord / IncomingEventRecord and ConfigureEventOutbox()/ConfigureEventInbox() in OnModelCreating.
- LocalEventHandlerOrderAttribute (Volo.Abp.EventBus.Local): Controls relative execution order between local handlers for the same event (default 0; lower runs first).
- DistributedEventSent / DistributedEventReceived (Volo.Abp.EventBus.Distributed): Local events ABP raises when distributed events are sent/received - useful for monitoring (Source values: Direct, Inbox, Outbox).
- AbpRabbitMqOptions / AbpRabbitMqEventBusOptions (Volo.Abp.RabbitMQ / Volo.Abp.EventBus.RabbitMq): Connection (Connections.Default.HostName/Port/UserName/Password) and bus settings (ClientName as queue, ExchangeName, PrefetchCount, QueueArguments, ExchangeArguments).
- AbpKafkaOptions / AbpKafkaEventBusOptions (Volo.Abp.Kafka / Volo.Abp.EventBus.Kafka): Connections.Default.BootstrapServers, ConfigureConsumer/ConfigureProducer/ConfigureTopic delegates, GroupId, TopicName, ConnectionName.
- AbpAzureServiceBusOptions / AbpAzureEventBusOptions (Volo.Abp.Azure.ServiceBus / Volo.Abp.EventBus.Azure): Connections.Default.ConnectionString or FullyQualifiedNamespace + TokenCredential, EventBus.SubscriberName, EventBus.TopicName, ServiceBusClient/Admin/Processor sub-options.
- AbpRebusEventBusOptions (Volo.Abp.EventBus.Rebus): Configured in PreConfigureServices. Properties: InputQueueName, Configurer (raw Rebus configurer for Transport/Subscriptions/etc.), Publish delegate to override publish behavior.

## Configuration pattern

Distributed event bus providers are wired in your AbpModule by depending on the integration module and calling Configure<TOptions> (or PreConfigure for Rebus). RabbitMQ example:

[DependsOn(typeof(AbpEventBusRabbitMqModule))]
public class MyModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        Configure<AbpRabbitMqOptions>(o =>
        {
            o.Connections.Default.HostName = "rabbit-host";
            o.Connections.Default.Port = 5672;
            o.Connections.Default.UserName = "user";
            o.Connections.Default.Password = "pass";
        });
        Configure<AbpRabbitMqEventBusOptions>(o =>
        {
            o.ClientName = "MyApp";       // queue name
            o.ExchangeName = "MyAppExchange";
            o.PrefetchCount = 1;
        });

        // Auto-publish entity events for selected aggregates
        Configure<AbpDistributedEntityEventOptions>(o =>
        {
            o.AutoEventSelectors.Add<Product>();
            o.EtoMappings.Add<Product, ProductEto>();
        });

        // Tune outbox/inbox processors (only relevant when outbox/inbox are configured on a DbContext)
        Configure<AbpEventBusBoxesOptions>(o =>
        {
            o.PeriodTimeSpan = TimeSpan.FromSeconds(2);
            o.BatchPublishOutboxEvents = true;
        });
    }
}

For outbox/inbox you also implement IHasEventOutbox/IHasEventInbox on your DbContext, expose DbSet<OutgoingEventRecord> / DbSet<IncomingEventRecord>, call builder.ConfigureEventOutbox()/ConfigureEventInbox() in OnModelCreating, then bind them via Configure<AbpDistributedEventBusOptions>(o => { o.Outboxes.Configure(c => c.UseDbContext<AppDbContext>()); o.Inboxes.Configure(c => c.UseDbContext<AppDbContext>()); }). Outbox/Inbox require ABP distributed locking and storing the inbox/outbox tables in the same database as the business data so that the application transaction includes them.

Kafka uses Configure<AbpKafkaOptions> + Configure<AbpKafkaEventBusOptions> with [DependsOn(typeof(AbpEventBusKafkaModule))]. Azure Service Bus uses Configure<AbpAzureServiceBusOptions> + Configure<AbpAzureEventBusOptions> with [DependsOn(typeof(AbpEventBusAzureModule))]; supports DefaultAzureCredential via FullyQualifiedNamespace + TokenCredential. Rebus uses PreConfigure<AbpRebusEventBusOptions>(o => { o.InputQueueName = "eventbus"; o.Configurer = c => c.Transport(t => t.UseRabbitMq(...)); }) with [DependsOn(typeof(AbpEventBusRebusModule))]. For all providers, the same IDistributedEventBus / IDistributedEventHandler<TEto> abstractions are used in application code regardless of transport.

## Code examples

Title: Publishing a local event from an application service
Scenario: Same-process notification when a product's stock changes; subscribers run in the same UoW/transaction.
```csharp
public class StockChangedEvent
{
    public Guid ProductId { get; set; }
    public int NewCount { get; set; }
}

public class ProductAppService : ApplicationService
{
    private readonly ILocalEventBus _localEventBus;
    public ProductAppService(ILocalEventBus localEventBus) => _localEventBus = localEventBus;

    public async Task ChangeStockAsync(Guid productId, int newCount)
    {
        // ... persist change ...
        await _localEventBus.PublishAsync(new StockChangedEvent
        {
            ProductId = productId,
            NewCount  = newCount
        });
    }
}

public class StockChangedHandler
    : ILocalEventHandler<StockChangedEvent>, ITransientDependency
{
    [LocalEventHandlerOrder(-1)] // runs before default-order handlers
    public Task HandleEventAsync(StockChangedEvent eventData) { /* ... */ return Task.CompletedTask; }
}
```
Key lines: ITransientDependency is what makes ABP register and discover the handler; [LocalEventHandlerOrder(-1)] gives this handler higher priority than default-0 handlers; PublishAsync defers execution until the UoW completes - pass onUnitOfWorkComplete: false to publish immediately.
Title: Raising a distributed event from an aggregate root
Scenario: Domain method on an aggregate root produces an ETO that ABP publishes when EF Core SaveChanges runs.
```csharp
[EventName("catalog.product.stock-changed")]
public class StockChangedEto
{
    public Guid ProductId { get; set; }
    public int NewCount { get; set; }
}

public class Product : AggregateRoot<Guid>
{
    public int StockCount { get; private set; }
    public void ChangeStock(int newCount)
    {
        StockCount = newCount;
        AddDistributedEvent(new StockChangedEto { ProductId = Id, NewCount = newCount });
    }
}
```
Key lines: [EventName] gives the event a stable broker-level name independent of CLR type/namespace renames; AddDistributedEvent enqueues on the entity, ABP publishes only after the persistence call succeeds.
Title: Subscribing to a distributed event
Scenario: Another microservice listens for stock changes and updates its read model.
```csharp
public class StockChangedEtoHandler
    : IDistributedEventHandler<StockChangedEto>, ITransientDependency
{
    private readonly IRepository<ProductCacheItem, Guid> _cache;
    public StockChangedEtoHandler(IRepository<ProductCacheItem, Guid> cache) => _cache = cache;

    public async Task HandleEventAsync(StockChangedEto eventData)
    {
        var item = await _cache.GetAsync(eventData.ProductId);
        item.SetStock(eventData.NewCount);
        await _cache.UpdateAsync(item);
    }
}
```
Key lines: ABP wires the broker subscription automatically once a handler implementing IDistributedEventHandler<TEto> is registered through DI; ACKs are sent after HandleEventAsync completes - throwing causes broker-level redelivery (or retry-via-inbox if configured).
Title: Enabling the transactional outbox on an EF Core DbContext
Scenario: Distributed events are saved into the same DB transaction as application data and shipped to the broker by a background worker.
```csharp
public class AppDbContext : AbpDbContext<AppDbContext>, IHasEventOutbox, IHasEventInbox
{
    public DbSet<OutgoingEventRecord> OutgoingEvents { get; set; }
    public DbSet<IncomingEventRecord> IncomingEvents { get; set; }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);
        builder.ConfigureEventOutbox();
        builder.ConfigureEventInbox();
    }
}

// In your module:
Configure<AbpDistributedEventBusOptions>(options =>
{
    options.Outboxes.Configure(config => { config.UseDbContext<AppDbContext>(); });
    options.Inboxes .Configure(config => { config.UseDbContext<AppDbContext>(); });
});
Configure<AbpEventBusBoxesOptions>(o =>
{
    o.PeriodTimeSpan = TimeSpan.FromSeconds(2);
    o.BatchPublishOutboxEvents = true;
});
```
Key lines: ConfigureEventOutbox/Inbox creates the OutgoingEvents/IncomingEvents tables; the same DbContext is reused so the outbox row commits atomically with the business data; AbpEventBusBoxesOptions controls the worker tick. Distributed locking must be configured (a separate ABP feature) when running multiple instances.
Title: Wiring RabbitMQ as the distributed event bus provider
Scenario: Replace the default in-process distributed bus with RabbitMQ across services.
```csharp
[DependsOn(typeof(AbpEventBusRabbitMqModule))]
public class CatalogServiceModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        var config = context.Services.GetConfiguration();
        Configure<AbpRabbitMqOptions>(o =>
        {
            o.Connections.Default.HostName = config["RabbitMQ:Host"];
            o.Connections.Default.UserName = config["RabbitMQ:User"];
            o.Connections.Default.Password = config["RabbitMQ:Pass"];
        });
        Configure<AbpRabbitMqEventBusOptions>(o =>
        {
            o.ClientName    = "CatalogService";
            o.ExchangeName  = "AbpEvents";
            o.PrefetchCount = 10;
        });
    }
}
```
appsettings.json equivalent:
```json
{
  "RabbitMQ": {
    "Connections": { "Default": { "HostName": "rabbit-host;rabbit-host-2", "Port": "5672" } },
    "EventBus":   { "ClientName": "CatalogService", "ExchangeName": "AbpEvents" }
  }
}
```
Key lines: ClientName becomes the per-application queue (so each microservice gets its own queue bound to the shared exchange); semicolon-separated HostName enables cluster failover; code-level Configure<...> overrides JSON.
Title: Kafka provider with consumer / producer / topic spec
Scenario: Use Kafka with explicit producer acks and a topic with 3 partitions.
```csharp
[DependsOn(typeof(AbpEventBusKafkaModule))]
public class OrdersServiceModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        Configure<AbpKafkaOptions>(o =>
        {
            o.Connections.Default.BootstrapServers = "kafka-1:9092,kafka-2:9092";
            o.ConfigureProducer = p => { p.Acks = Acks.All; p.MessageTimeoutMs = 6000; };
            o.ConfigureConsumer = c => { c.EnableAutoCommit = false; };
            o.ConfigureTopic    = t => { t.NumPartitions = 3; t.ReplicationFactor = 3; };
        });
        Configure<AbpKafkaEventBusOptions>(o =>
        {
            o.GroupId   = "orders-service";
            o.TopicName = "abp-events";
        });
    }
}
```
Key lines: GroupId acts as the Kafka consumer group identifier (one app == one group, scaling out by partition); ConfigureTopic governs how ABP creates the topic if it doesn't exist; EnableAutoCommit=false hands commit responsibility to ABP after a successful handler invocation.
## Common mistakes

- - Mistake: Forgetting to mark a handler class with ITransientDependency (or another DI lifetime). Why wrong: ABP discovers handlers via DI registration; an unregistered class is silently never invoked. Fix: implement ITransientDependency or register manually in ConfigureServices.
- - Mistake: Putting EF Core navigation properties or cyclic graphs into an ETO. Why wrong: ETOs are serialized to the broker; cycles and lazy-loaded navigations break serialization or pull in the entire object graph. Fix: design ETOs as flat DTOs with primitive/Guid properties and a parameterless constructor.
- - Mistake: Expecting distributed events to be transactional with the database without enabling the outbox. Why wrong: Without outbox, ABP publishes after the UoW completes - if the broker call fails, you lose the event; if the broker succeeds and the DB rolls back, you have a phantom event. Fix: implement IHasEventOutbox on your DbContext, call ConfigureEventOutbox(), and bind it through AbpDistributedEventBusOptions.Outboxes.Configure.
- - Mistake: Storing outbox/inbox tables in a different database than business data. Why wrong: The whole guarantee relies on a single local DB transaction containing both your data write and the outbox row. Fix: keep OutgoingEvents/IncomingEvents in the same DbContext/database as the application's aggregates.
- - Mistake: Running multiple application instances with the outbox/inbox enabled but no ABP distributed lock provider. Why wrong: Each instance will race to read and dispatch the same outbox rows, producing duplicate publishes. Fix: configure ABP's distributed locking (see distributed-locking reference) before scaling out.
- - Mistake: Calling AddDistributedEvent on a non-aggregate entity. Why wrong: AddLocalEvent/AddDistributedEvent are members of AggregateRoot<>; child entities don't get their events flushed by ABP's UoW. Fix: raise events on the aggregate root that owns the change.
- - Mistake: Throwing in a local event handler under the assumption it is fire-and-forget. Why wrong: Local handlers run inside the publisher's UoW by default - an unhandled exception rolls back the entire transaction. Fix: catch and log inside the handler if you intentionally want to swallow, or publish with onUnitOfWorkComplete: false to escape the UoW boundary.
- - Mistake: Using two different ETO classes (or renaming an ETO) on either side of the broker without an [EventName] attribute. Why wrong: Without [EventName], ABP uses the FQCN; renaming or moving the type breaks the contract. Fix: pin a stable [EventName("bounded.context.event")] on shared ETOs.
- - Mistake: Configuring AbpRebusEventBusOptions with Configure<...> in ConfigureServices instead of PreConfigure<...> in PreConfigureServices. Why wrong: Rebus is wired during PreConfigure; settings applied later don't take effect. Fix: PreConfigure<AbpRebusEventBusOptions>(...) inside PreConfigureServices.
- - Mistake: Sharing the same RabbitMQ ClientName between multiple applications. Why wrong: ClientName is the queue name; sharing it makes the apps compete for messages instead of each receiving its own copy. Fix: give each application a unique ClientName.
- - Mistake: Sharing the same Kafka GroupId across application copies that should each receive every event. Why wrong: Same GroupId puts instances in one consumer group, so each event is delivered once per group, not per instance. Fix: use distinct GroupIds when fan-out semantics are required.
## Version pins (ABP 10.3)

- ABP 10.3 ships these provider integrations under Volo.Abp.EventBus.{RabbitMQ,Kafka,Azure,Rebus} - same module names as 10.x have used; future majors may rename or split the packages [uncertain].
- Outbox/Inbox APIs (IHasEventOutbox, IHasEventInbox, ConfigureEventOutbox, ConfigureEventInbox, OutgoingEventRecord, IncomingEventRecord) are stable in 10.3 and live under Volo.Abp.EventBus.Distributed.
- AbpEventBusBoxesOptions defaults in 10.3: BatchPublishOutboxEvents = true (per docs); PeriodTimeSpan default not explicitly stated in the 10.3 page [uncertain].
- Distributed lock timeout for outbox/inbox is documented as 15 seconds (TimeSpan.FromSeconds(15)) by default per the ABP docs answer thread; configurable via AbpEventBusBoxesOptions [uncertain - exact option name may differ in 10.3].
- Azure Service Bus integration in 10.3 supports both ConnectionString and FullyQualifiedNamespace + TokenCredential (DefaultAzureCredential) authentication paths.
- Rebus options must be applied via PreConfigure<AbpRebusEventBusOptions> in PreConfigureServices; default subscriptions are stored in memory unless you set options.Configurer to a persistent Subscriptions store.
- RabbitMQ HostName supports semicolon-separated cluster nodes; PrefetchCount and ExchangeArguments/QueueArguments are exposed for tuning in 10.3.
- Kafka integration exposes ConfigureProducer / ConfigureConsumer / ConfigureTopic delegates over Confluent.Kafka ClientConfig types; the topic is auto-created using the spec when missing - in production environments you may want to pre-create topics and not rely on auto-creation [uncertain - production guidance].
- Default IDistributedEventBus implementation when no provider module is registered is LocalDistributedEventBus, which behaves identically to ILocalEventBus in-process; production microservices must explicitly add a provider module.

## Cross-references

**Phase 1 references:**
- references/framework/architecture-ddd.md — Triggered when AddLocalEvent / AddDistributedEvent are discussed on AggregateRoot<T>; events are a domain-level pattern.
- references/framework/data-ef-core.md — Triggered when configuring outbox/inbox via IHasEventOutbox / IHasEventInbox, ConfigureEventOutbox/Inbox in OnModelCreating, and DbSet<OutgoingEventRecord/IncomingEventRecord>.
- references/infrastructure/caching-and-locking.md — Triggered when discussing outbox/inbox: ABP requires distributed locking to coordinate multiple instances reading the outbox/inbox tables.
- references/infrastructure/background-processing.md — Triggered when explaining the outbox/inbox processing loop, AbpEventBusBoxesOptions.PeriodTimeSpan, and how events are shipped to brokers asynchronously.
- references/templates/microservice.md — Triggered when discussing cross-service eventing patterns and choosing distributed event bus over sync HTTP between microservices.

**External docs:**
- RabbitMQ .NET Client - https://www.rabbitmq.com/dotnet.html - underlying client used by Volo.Abp.EventBus.RabbitMQ; ConnectionFactory properties are exposed.
- Confluent Kafka .NET (Confluent.Kafka) - https://docs.confluent.io/kafka-clients/dotnet/current/overview.html - underlying ClientConfig, ProducerConfig, ConsumerConfig types used by AbpKafkaOptions.
- Azure.Messaging.ServiceBus - https://learn.microsoft.com/dotnet/api/overview/azure/messaging.servicebus-readme - ServiceBusClientOptions, ServiceBusProcessorOptions, ServiceBusAdministrationClientOptions referenced by AbpAzureServiceBusOptions.
- Azure Identity (DefaultAzureCredential) - https://learn.microsoft.com/dotnet/api/azure.identity.defaultazurecredential - TokenCredential alternative to connection strings for Azure Service Bus.
- Rebus documentation - https://github.com/rebus-org/Rebus/wiki - underlying message bus library; transports (RabbitMQ, MSMQ, Azure SB, etc.) and subscription stores are configured through the Configurer property.
- Microsoft Azure Service Bus topics & subscriptions - https://learn.microsoft.com/azure/service-bus-messaging/service-bus-queues-topics-subscriptions - background concepts behind SubscriberName / TopicName.
- Transactional Outbox pattern (microservices.io) - https://microservices.io/patterns/data/transactional-outbox.html - external explanation of the outbox pattern ABP implements.

## Sources (ABP 10.3)

- https://abp.io/docs/10.3/framework/infrastructure/event-bus — Top-level Event Bus overview comparing local vs distributed buses and routing guidance for monolith / modular monolith / microservice systems.
- https://abp.io/docs/10.3/framework/infrastructure/event-bus/local — Local Event Bus: ILocalEventBus, ILocalEventHandler<TEvent>, AddLocalEvent on aggregate roots, LocalEventHandlerOrder, transactional behavior, dynamic events.
- https://abp.io/docs/10.3/framework/infrastructure/event-bus/distributed — Distributed Event Bus: IDistributedEventBus, IDistributedEventHandler<TEvent>, ETOs, EventName attribute, AbpDistributedEntityEventOptions, Outbox/Inbox patterns, AbpEventBusBoxesOptions.
- https://abp.io/docs/10.3/framework/infrastructure/event-bus/distributed/azure — Azure Service Bus integration: Volo.Abp.EventBus.Azure, AbpEventBusAzureModule, AbpAzureServiceBusOptions, AbpAzureEventBusOptions, TokenCredential support.
- https://abp.io/docs/10.3/framework/infrastructure/event-bus/distributed/rabbitmq — RabbitMQ integration: Volo.Abp.EventBus.RabbitMQ, AbpEventBusRabbitMqModule, AbpRabbitMqOptions, AbpRabbitMqEventBusOptions, exchange/queue arguments, cluster hostnames.
- https://abp.io/docs/10.3/framework/infrastructure/event-bus/distributed/kafka — Kafka integration: Volo.Abp.EventBus.Kafka, AbpEventBusKafkaModule, AbpKafkaOptions, AbpKafkaEventBusOptions, ConfigureConsumer / ConfigureProducer / ConfigureTopic.
- https://abp.io/docs/10.3/framework/infrastructure/event-bus/distributed/rebus — Rebus integration: Volo.Abp.EventBus.Rebus, AbpEventBusRebusModule, AbpRebusEventBusOptions (PreConfigure), Configurer for transport/subscriptions, Publish delegate.

Last verified: 2026-05-10
