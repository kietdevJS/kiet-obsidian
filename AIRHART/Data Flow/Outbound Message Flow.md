# Outbound Message Flow

> How data flows from AODB to QueryStream and end users

## Overview

After AODB processes and persists entities, the data must be distributed to:
- **QueryStream** for real-time client subscriptions
- **External Systems** via outbound adapters (future)
- **Analytics** for reporting and analysis

This flow ensures that **all clients see consistent, up-to-date data** within milliseconds of changes.

## High-Level Flow

```
┌──────────────┐
│     AODB     │
│   Pipeline   │
└──────┬───────┘
       │ Publish
       ↓
┌──────────────────────────────┐
│   Kafka Output Topics        │
│  • aodb-pipeline-output      │
│  • adapter-messages          │
│  • dtd (schema)              │
└──────┬───────────────────────┘
       │ Consume
       ↓
┌──────────────────────────────┐
│   QueryStream                │
│   Database Manager           │
│  • Compile feeds             │
│  • Update database           │
│  • Publish internal events   │
└──────┬───────────────────────┘
       │ Internal events
       ↓
┌──────────────────────────────┐
│   QueryStream                │
│   Subscription Manager       │
│  • Match subscriptions       │
│  • Stream updates            │
└──────┬───────────────────────┘
       │ WebSocket/SignalR
       ↓
┌──────────────────────────────┐
│      Client Applications     │
│  • CoreUI Dashboard          │
│  • External Systems          │
│  • Analytics Tools           │
└──────────────────────────────┘
```

## Stage 1: AODB Publishing

### When Publishing Occurs

**Trigger Points:**
- After successful entity persistence
- After post-persist business rules complete
- Before pipeline message completion

### What Gets Published

**1. Entity Updates** → `aodb-pipeline-output`

```csharp
// Published by: AODB Pipeline Publisher
var outputMessage = new PipelineOutputMessage
{
    MessageId = Guid.NewGuid(),
    TraceId = context.TraceId,
    OperationType = "Update", // Create, Update, Delete
    EntityType = "FlightLeg",
    EntityId = flightLeg.Id,
    TenantId = context.TenantId,
    Timestamp = DateTime.UtcNow,
    Data = flightLeg, // Full entity snapshot
    ChangedProperties = ["ActualTimeOfArrival", "ArrivalStand"]
};

await kafkaProducer.ProduceAsync(
    topic: "aodb-pipeline-output",
    key: flightLeg.Id,
    value: outputMessage
);
```

**Key Points:**
- **Full Snapshot:** Complete entity state, not just changes
- **Changed Properties:** List of what changed for optimization
- **Partition Key:** Entity ID ensures ordering
- **Trace Context:** End-to-end tracing maintained

**2. Message Status** → `adapter-messages`

```csharp
// Track original inbound message status
var messageStatus = new AdapterMessageStatus
{
    InboundMessageId = context.InboundMessageId,
    Status = "Processed", // Received, Processing, Processed, Failed
    EntityId = flightLeg.Id,
    ProcessedAt = DateTime.UtcNow,
    ProcessingTimeMs = context.ElapsedMs,
    Errors = []
};

await kafkaProducer.ProduceAsync(
    topic: "adapter-messages",
    key: context.InboundMessageId,
    value: messageStatus
);
```

**Purpose:**
- Track message lifecycle
- Monitor processing times
- Audit trail for compliance

**3. Schema Updates** → `aodb-pipeline-dtd`

```csharp
// Published when entity schema changes
var schemaUpdate = new DataTransferDefinition
{
    EntityType = "FlightLeg",
    Version = "1.5",
    Properties = [
        new PropertyDefinition 
        { 
            Name = "ActualTimeOfArrival",
            Type = "DateTime",
            Nullable = true
        },
        // ... all properties
    ],
    PublishedAt = DateTime.UtcNow
};

await kafkaProducer.ProduceAsync(
    topic: "aodb-pipeline-dtd",
    key: "FlightLeg",
    value: schemaUpdate
);
```

**Purpose:**
- Schema evolution
- Client code generation
- Version compatibility

### Publishing Code

**Pipeline Publisher Component:**

```csharp
public class PipelinePublisher : IPipelinePublisher
{
    private readonly IProducer<string, PipelineOutputMessage> _producer;
    private readonly ILogger<PipelinePublisher> _logger;
    private readonly IOutboxRepository _outbox;

    public async Task PublishEntityAsync(
        Entity entity, 
        List<string> changedProperties,
        PipelineContext context)
    {
        var message = new PipelineOutputMessage
        {
            MessageId = Guid.NewGuid(),
            TraceId = context.TraceId,
            OperationType = DetermineOperationType(entity),
            EntityType = entity.GetType().Name,
            EntityId = entity.Id,
            TenantId = context.TenantId,
            Timestamp = DateTime.UtcNow,
            Data = entity,
            ChangedProperties = changedProperties
        };

        try
        {
            // Use outbox pattern for guaranteed delivery
            await _outbox.AddAsync(new OutboxMessage
            {
                Topic = "aodb-pipeline-output",
                Key = entity.Id,
                Payload = JsonSerializer.Serialize(message),
                CreatedAt = DateTime.UtcNow
            });

            _logger.LogInformation(
                "Entity {EntityType}/{EntityId} queued for publishing",
                entity.GetType().Name, entity.Id);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex,
                "Failed to queue entity {EntityType}/{EntityId} for publishing",
                entity.GetType().Name, entity.Id);
            throw;
        }
    }
}
```

**Outbox Processor (Background Service):**

```csharp
public class OutboxProcessor : BackgroundService
{
    private readonly IOutboxRepository _outbox;
    private readonly IProducer<string, string> _producer;
    private readonly ILogger<OutboxProcessor> _logger;

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            try
            {
                // Fetch batch
                var messages = await _outbox.GetPendingAsync(100);

                foreach (var message in messages)
                {
                    // Publish to Kafka
                    var result = await _producer.ProduceAsync(
                        message.Topic,
                        new Message<string, string>
                        {
                            Key = message.Key,
                            Value = message.Payload,
                            Headers = new Headers
                            {
                                { "traceparent", message.TraceContext }
                            }
                        },
                        ct);

                    // Mark as published
                    await _outbox.MarkPublishedAsync(
                        message.Id, 
                        result.Offset);

                    _logger.LogDebug(
                        "Published message {MessageId} to {Topic} at offset {Offset}",
                        message.Id, message.Topic, result.Offset);
                }

                // Wait before next batch
                await Task.Delay(TimeSpan.FromMilliseconds(100), ct);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error processing outbox");
                await Task.Delay(TimeSpan.FromSeconds(5), ct);
            }
        }
    }
}
```

**Why Outbox Pattern?**
- **Transactional:** Entity save + outbox insert in same transaction
- **Guaranteed Delivery:** Messages never lost even if Kafka down
- **Ordered:** Messages published in same order as persisted
- **Resilient:** Automatic retry on Kafka failures

## Stage 2: QueryStream Consumption

### Database Manager

**Role:** Consume output topics and compile feeds

**Component Structure:**

```
QueryStream.DatabaseManager/
├── Consumers/
│   ├── PipelineOutputConsumer.cs     ← Main entity consumer
│   ├── AdapterMessageConsumer.cs     ← Message status tracking
│   └── DtdConsumer.cs                ← Schema updates
├── FeedCompilers/
│   ├── FlightLegFeedCompiler.cs      ← Compile FlightLeg feeds
│   ├── VisitFeedCompiler.cs          ← Compile Visit feeds
│   └── ResourceFeedCompiler.cs       ← Compile Resource feeds
└── Database/
    └── QueryStreamDbContext.cs       ← EF Core context
```

**Consumption Flow:**

```csharp
public class PipelineOutputConsumer : BackgroundService
{
    private readonly IConsumer<string, PipelineOutputMessage> _consumer;
    private readonly IServiceProvider _services;
    private readonly ILogger<PipelineOutputConsumer> _logger;

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        _consumer.Subscribe("aodb-pipeline-output");

        while (!ct.IsCancellationRequested)
        {
            try
            {
                // Consume message
                var message = _consumer.Consume(TimeSpan.FromSeconds(1));
                if (message == null) continue;

                using var scope = _services.CreateScope();
                var compiler = GetCompilerForEntityType(
                    scope, 
                    message.Value.EntityType);

                // Compile feeds
                await compiler.CompileAsync(
                    message.Value, 
                    ct);

                // Commit offset
                _consumer.Commit(message);

                _logger.LogDebug(
                    "Processed {EntityType}/{EntityId} from offset {Offset}",
                    message.Value.EntityType,
                    message.Value.EntityId,
                    message.Offset);
            }
            catch (ConsumeException ex)
            {
                _logger.LogError(ex, "Error consuming message");
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error processing message");
                // Don't commit - will retry
            }
        }
    }

    private IFeedCompiler GetCompilerForEntityType(
        IServiceScope scope, 
        string entityType)
    {
        return entityType switch
        {
            "FlightLeg" => scope.ServiceProvider
                .GetRequiredService<FlightLegFeedCompiler>(),
            "Visit" => scope.ServiceProvider
                .GetRequiredService<VisitFeedCompiler>(),
            "Resource" => scope.ServiceProvider
                .GetRequiredService<ResourceFeedCompiler>(),
            _ => throw new NotSupportedException(
                $"No compiler for {entityType}")
        };
    }
}
```

### Feed Compilation

**What is a Feed?**
A feed is a **denormalized, optimized view** of entities for specific client needs.

**Example: FlightLeg Feed**

```csharp
public class FlightLegFeedCompiler : IFeedCompiler
{
    private readonly QueryStreamDbContext _db;
    private readonly IFeedPublisher _publisher;

    public async Task CompileAsync(
        PipelineOutputMessage message,
        CancellationToken ct)
    {
        var flightLeg = JsonSerializer
            .Deserialize<FlightLeg>(message.Data);

        // Compile feed items
        var feedItems = new List<FeedItem>();

        // 1. Main FlightLeg feed
        feedItems.Add(await CompileFlightLegFeedAsync(flightLeg));

        // 2. Arrival feed (if arrival flight)
        if (flightLeg.IsArrival)
        {
            feedItems.Add(await CompileArrivalFeedAsync(flightLeg));
        }

        // 3. Departure feed (if departure flight)
        if (flightLeg.IsDeparture)
        {
            feedItems.Add(await CompileDepartureFeedAsync(flightLeg));
        }

        // 4. Stand feed (if stand assigned)
        if (flightLeg.ArrivalStandId != null)
        {
            feedItems.Add(await CompileStandFeedAsync(flightLeg));
        }

        // Save all feeds in transaction
        using var transaction = await _db.Database
            .BeginTransactionAsync(ct);

        foreach (var feedItem in feedItems)
        {
            _db.FeedItems.Update(feedItem);
        }

        await _db.SaveChangesAsync(ct);
        await transaction.CommitAsync(ct);

        // Publish internal events for subscription manager
        foreach (var feedItem in feedItems)
        {
            await _publisher.PublishFeedUpdateAsync(feedItem);
        }
    }

    private async Task<FeedItem> CompileFlightLegFeedAsync(
        FlightLeg flightLeg)
    {
        return new FeedItem
        {
            FeedType = "FlightLeg",
            EntityId = flightLeg.Id,
            TenantId = flightLeg.TenantId,
            UpdatedAt = DateTime.UtcNow,
            Data = new
            {
                // Core properties
                FlightNumber = flightLeg.FlightNumber,
                OperatorIata = flightLeg.OperatorIata,
                AircraftTypeIata = flightLeg.AircraftTypeIata,
                
                // Timestamps
                ScheduledTimeOfArrival = flightLeg.ScheduledTimeOfArrival,
                ActualTimeOfArrival = flightLeg.ActualTimeOfArrival,
                ScheduledTimeOfDeparture = flightLeg.ScheduledTimeOfDeparture,
                ActualTimeOfDeparture = flightLeg.ActualTimeOfDeparture,
                
                // Locations
                OriginIata = flightLeg.OriginIata,
                DestinationIata = flightLeg.DestinationIata,
                ArrivalStand = flightLeg.ArrivalStand?.Name,
                DepartureStand = flightLeg.DepartureStand?.Name,
                
                // Status
                Status = flightLeg.Status,
                
                // Denormalized data (from Master Data)
                OperatorName = await GetAirlineName(flightLeg.OperatorIata),
                AircraftTypeName = await GetAircraftTypeName(
                    flightLeg.AircraftTypeIata),
                OriginAirportName = await GetAirportName(
                    flightLeg.OriginIata),
                DestinationAirportName = await GetAirportName(
                    flightLeg.DestinationIata)
            }
        };
    }
}
```

**Feed Types Created:**

1. **Entity Feed** (`FlightLeg`, `Visit`)
   - Complete entity with denormalized references
   - Used for entity detail views

2. **List Feed** (`Arrivals`, `Departures`, `Stands`)
   - Filtered/sorted views
   - Used for list/grid views

3. **Timeline Feed** (`FlightTimeline`)
   - Time-ordered events
   - Used for Gantt charts

4. **Aggregation Feed** (`StandUtilization`, `FlightStatistics`)
   - Computed metrics
   - Used for dashboards

### Feed Storage

**Database Schema:**

```sql
CREATE TABLE feed_items (
    id UUID PRIMARY KEY,
    feed_type VARCHAR(100) NOT NULL,
    entity_id VARCHAR(100) NOT NULL,
    tenant_id VARCHAR(100) NOT NULL,
    data JSONB NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    INDEX idx_feed_tenant (feed_type, tenant_id),
    INDEX idx_entity (entity_id),
    INDEX idx_updated (updated_at)
);

CREATE TABLE feed_subscriptions (
    id UUID PRIMARY KEY,
    connection_id VARCHAR(100) NOT NULL,
    feed_type VARCHAR(100) NOT NULL,
    filters JSONB,
    created_at TIMESTAMP NOT NULL,
    INDEX idx_connection (connection_id),
    INDEX idx_feed (feed_type)
);
```

**Why JSONB?**
- Flexible schema per feed type
- Fast JSON queries with GIN indexes
- No schema migrations needed
- Direct serialization to clients

## Stage 3: Subscription Matching

### Internal Events

**Published by:** Database Manager  
**Topic:** `qs-internal-updates`  
**Consumed by:** Subscription Manager

```csharp
public class FeedPublisher : IFeedPublisher
{
    private readonly IProducer<string, FeedUpdateEvent> _producer;

    public async Task PublishFeedUpdateAsync(FeedItem feedItem)
    {
        var updateEvent = new FeedUpdateEvent
        {
            FeedType = feedItem.FeedType,
            EntityId = feedItem.EntityId,
            TenantId = feedItem.TenantId,
            UpdatedAt = feedItem.UpdatedAt,
            ChangedProperties = DetermineChangedProperties(feedItem)
        };

        await _producer.ProduceAsync(
            topic: "qs-internal-updates",
            key: feedItem.EntityId,
            value: updateEvent
        );
    }
}
```

### Subscription Manager

**Role:** Stream updates to connected clients

**Component Structure:**

```
QueryStream.SubscriptionManager/
├── Hub/
│   └── QueryStreamHub.cs            ← SignalR hub
├── Consumers/
│   └── FeedUpdateConsumer.cs        ← Consume internal events
├── Services/
│   ├── SubscriptionService.cs       ← Manage subscriptions
│   └── FilterService.cs             ← Apply filters
└── Cache/
    └── ConnectionCache.cs           ← Track active connections
```

**Consumption and Distribution:**

```csharp
public class FeedUpdateConsumer : BackgroundService
{
    private readonly IConsumer<string, FeedUpdateEvent> _consumer;
    private readonly IHubContext<QueryStreamHub> _hubContext;
    private readonly ISubscriptionService _subscriptions;
    private readonly IFilterService _filters;
    private readonly QueryStreamDbContext _db;

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        _consumer.Subscribe("qs-internal-updates");

        while (!ct.IsCancellationRequested)
        {
            var message = _consumer.Consume(TimeSpan.FromSeconds(1));
            if (message == null) continue;

            var updateEvent = message.Value;

            // Find matching subscriptions
            var subscriptions = await _subscriptions
                .GetSubscriptionsForFeedAsync(
                    updateEvent.FeedType,
                    updateEvent.TenantId);

            // Load full feed data
            var feedItem = await _db.FeedItems
                .FirstOrDefaultAsync(f => 
                    f.EntityId == updateEvent.EntityId,
                    ct);

            if (feedItem == null) continue;

            // Send to each matching subscription
            foreach (var subscription in subscriptions)
            {
                // Apply filters
                if (!_filters.Matches(feedItem, subscription.Filters))
                    continue;

                // Send via SignalR
                await _hubContext.Clients
                    .Client(subscription.ConnectionId)
                    .SendAsync(
                        "FeedUpdate",
                        new
                        {
                            FeedType = feedItem.FeedType,
                            EntityId = feedItem.EntityId,
                            Data = feedItem.Data,
                            UpdatedAt = feedItem.UpdatedAt
                        },
                        ct);
            }

            _consumer.Commit(message);
        }
    }
}
```

### Filter Application

**Client-Side Filters:**

When client subscribes, it specifies filters:

```typescript
// Client code
const subscription = await queryStreamClient.subscribe({
  feedType: 'FlightLeg',
  filters: {
    operatorIata: ['SK', 'AY'],
    status: ['Scheduled', 'Departed'],
    scheduledTimeOfArrival: {
      from: '2025-12-15T00:00:00Z',
      to: '2025-12-15T23:59:59Z'
    }
  }
});
```

**Server-Side Filter Evaluation:**

```csharp
public class FilterService : IFilterService
{
    public bool Matches(FeedItem feedItem, Dictionary<string, object> filters)
    {
        if (filters == null || !filters.Any())
            return true; // No filters = match all

        var data = JObject.Parse(feedItem.Data.ToString());

        foreach (var filter in filters)
        {
            var value = data[filter.Key];
            if (value == null) return false;

            // Array filter (IN clause)
            if (filter.Value is JArray arrayFilter)
            {
                if (!arrayFilter.Any(f => f.ToString() == value.ToString()))
                    return false;
            }
            // Range filter
            else if (filter.Value is JObject rangeFilter)
            {
                if (rangeFilter["from"] != null &&
                    value.ToObject<DateTime>() < 
                    rangeFilter["from"].ToObject<DateTime>())
                    return false;

                if (rangeFilter["to"] != null &&
                    value.ToObject<DateTime>() > 
                    rangeFilter["to"].ToObject<DateTime>())
                    return false;
            }
            // Exact match
            else
            {
                if (value.ToString() != filter.Value.ToString())
                    return false;
            }
        }

        return true; // All filters matched
    }
}
```

## Stage 4: Client Delivery

### SignalR Hub

**WebSocket Server:**

```csharp
public class QueryStreamHub : Hub
{
    private readonly ISubscriptionService _subscriptions;
    private readonly ILogger<QueryStreamHub> _logger;

    public async Task Subscribe(
        string feedType,
        Dictionary<string, object> filters)
    {
        var subscription = new Subscription
        {
            Id = Guid.NewGuid(),
            ConnectionId = Context.ConnectionId,
            FeedType = feedType,
            Filters = filters,
            CreatedAt = DateTime.UtcNow
        };

        await _subscriptions.AddAsync(subscription);

        _logger.LogInformation(
            "Client {ConnectionId} subscribed to {FeedType}",
            Context.ConnectionId, feedType);

        // Send initial data
        await SendInitialDataAsync(subscription);
    }

    public async Task Unsubscribe(Guid subscriptionId)
    {
        await _subscriptions.RemoveAsync(subscriptionId);

        _logger.LogInformation(
            "Client {ConnectionId} unsubscribed from {SubscriptionId}",
            Context.ConnectionId, subscriptionId);
    }

    public override async Task OnDisconnectedAsync(Exception exception)
    {
        // Clean up subscriptions
        await _subscriptions.RemoveByConnectionAsync(Context.ConnectionId);

        _logger.LogInformation(
            "Client {ConnectionId} disconnected",
            Context.ConnectionId);

        await base.OnDisconnectedAsync(exception);
    }

    private async Task SendInitialDataAsync(Subscription subscription)
    {
        // Load existing data matching filters
        var feedItems = await _subscriptions.GetInitialDataAsync(
            subscription.FeedType,
            subscription.Filters,
            limit: 1000);

        // Send in batches
        await Clients.Client(subscription.ConnectionId)
            .SendAsync("InitialData", feedItems);
    }
}
```

### Client Implementation

**TypeScript Client:**

```typescript
export class QueryStreamClient {
  private connection: signalR.HubConnection;
  private subscriptions = new Map<string, Subscription>();

  async connect(url: string, accessToken: string) {
    this.connection = new signalR.HubConnectionBuilder()
      .withUrl(url, {
        accessTokenFactory: () => accessToken
      })
      .withAutomaticReconnect({
        nextRetryDelayInMilliseconds: (retryContext) => {
          // Exponential backoff
          return Math.min(1000 * Math.pow(2, retryContext.previousRetryCount), 30000);
        }
      })
      .configureLogging(signalR.LogLevel.Information)
      .build();

    // Handle feed updates
    this.connection.on('FeedUpdate', (update) => {
      this.handleFeedUpdate(update);
    });

    // Handle initial data
    this.connection.on('InitialData', (data) => {
      this.handleInitialData(data);
    });

    await this.connection.start();
  }

  async subscribe(options: SubscriptionOptions): Promise<Subscription> {
    const subscriptionId = await this.connection.invoke(
      'Subscribe',
      options.feedType,
      options.filters
    );

    const subscription = new Subscription(
      subscriptionId,
      options.feedType,
      options.onUpdate
    );

    this.subscriptions.set(subscriptionId, subscription);

    return subscription;
  }

  async unsubscribe(subscriptionId: string) {
    await this.connection.invoke('Unsubscribe', subscriptionId);
    this.subscriptions.delete(subscriptionId);
  }

  private handleFeedUpdate(update: FeedUpdate) {
    // Find subscriptions for this feed type
    for (const subscription of this.subscriptions.values()) {
      if (subscription.feedType === update.feedType) {
        subscription.onUpdate(update);
      }
    }
  }

  private handleInitialData(data: any[]) {
    // Initial data sent after subscribe
    console.log(`Received ${data.length} initial items`);
  }
}
```

**React Hook Usage:**

```typescript
export function useFlightLegs(filters: FlightLegFilters) {
  const [flightLegs, setFlightLegs] = useState<FlightLeg[]>([]);
  const queryStreamClient = useQueryStreamClient();

  useEffect(() => {
    const subscription = await queryStreamClient.subscribe({
      feedType: 'FlightLeg',
      filters: {
        operatorIata: filters.airlines,
        status: filters.statuses,
        scheduledTimeOfArrival: {
          from: filters.dateFrom,
          to: filters.dateTo
        }
      },
      onUpdate: (update) => {
        // Update state with new/changed flight leg
        setFlightLegs(prev => {
          const index = prev.findIndex(f => f.id === update.entityId);
          if (index >= 0) {
            // Update existing
            const updated = [...prev];
            updated[index] = update.data;
            return updated;
          } else {
            // Add new
            return [...prev, update.data];
          }
        });
      }
    });

    return () => {
      subscription.unsubscribe();
    };
  }, [filters]);

  return flightLegs;
}
```

## End-to-End Trace

### Example: Flight Arrival Update

**Timeline:**

```
T+0ms     │ ACD adapter receives flight update message
          │
T+5ms     │ Adapter publishes to "traffic-operational-flightlegacd"
          │
T+10ms    │ AODB Pipeline consumes message
          │
T+15ms    │ Deserialization (DTO → Entity)
          │
T+20ms    │ Enrichment (Master Data lookup)
          │
T+25ms    │ Pre-Persist Rules (validation, transformation)
          │
T+30ms    │ Persistence (Database write)
          │
T+35ms    │ Post-Persist Rules (linking, notifications)
          │
T+40ms    │ Publishing (Outbox write)
          │ ─────────────────────────────────────────────
          │ 
T+45ms    │ Outbox Processor publishes to Kafka
          │
T+50ms    │ QueryStream DB Manager consumes
          │
T+55ms    │ Feed Compilation (FlightLeg, Arrival, Stand feeds)
          │
T+60ms    │ Feed Persistence (PostgreSQL write)
          │
T+65ms    │ Internal Event Publishing (qs-internal-updates)
          │
T+70ms    │ QueryStream Subscription Manager consumes
          │
T+75ms    │ Subscription Matching (filter application)
          │
T+80ms    │ SignalR Broadcasting (WebSocket send)
          │
T+85ms    │ Client Receives Update
          │
T+90ms    │ UI Updates (React state change, re-render)
          │
Total: ~90ms from adapter to UI
```

**Tracing Headers:**

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
├─ AODB Pipeline
│  └─ traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-a3ce929d0e0e4736-01
├─ Outbox Processor
│  └─ traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-0e0e47360f067aa0-01
├─ QueryStream DB Manager
│  └─ traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-47360f067aa00ba9-01
└─ QueryStream Subscription Manager
   └─ traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-067aa00ba902b7ce-01
```

All components share same `trace-id` for end-to-end correlation.

## Performance

### Throughput

**Targets:**
- AODB Publishing: 10,000 entities/second
- QueryStream Feed Compilation: 10,000 entities/second
- SignalR Broadcasting: 100,000 messages/second

**Bottlenecks:**
- Feed compilation (JSONB serialization)
- SignalR connection limits (per instance)
- Network bandwidth to clients

### Latency

**P50:** 50-100ms (AODB persist → Client UI)
**P95:** 100-200ms
**P99:** 200-500ms

**Breakdown:**
- AODB Publishing: 5-10ms
- Kafka propagation: 5-10ms
- Feed Compilation: 20-40ms
- Subscription Matching: 5-10ms
- WebSocket Send: 10-20ms
- Client Processing: 5-10ms

### Optimization

**1. Batch Updates**

```csharp
// Instead of sending each update individually
foreach (var feedItem in feedItems)
{
    await SendUpdateAsync(feedItem);
}

// Batch multiple updates
var batch = feedItems.Take(100).ToList();
await SendBatchUpdateAsync(batch);
```

**2. Incremental Updates**

```csharp
// Only send changed properties
var update = new IncrementalUpdate
{
    EntityId = feedItem.EntityId,
    Changes = DiffProperties(oldFeedItem, newFeedItem)
};
```

**3. Subscription Consolidation**

```csharp
// Group similar subscriptions
var groupedSubs = subscriptions
    .GroupBy(s => (s.FeedType, s.FiltersHash))
    .ToList();

// Send once per group
foreach (var group in groupedSubs)
{
    await SendToGroupAsync(group.Key, update);
}
```

## Monitoring

### Key Metrics

**AODB Publishing:**
```
aodb_publishing_rate (entities/sec)
aodb_publishing_latency_ms (P50, P95, P99)
aodb_outbox_size (pending messages)
aodb_outbox_age_seconds (oldest message)
```

**QueryStream DB Manager:**
```
qs_db_consumption_rate (messages/sec)
qs_db_compilation_latency_ms
qs_db_lag (consumer lag)
qs_feed_update_rate (feeds/sec)
```

**QueryStream Subscription Manager:**
```
qs_sub_active_connections (gauge)
qs_sub_active_subscriptions (gauge)
qs_sub_broadcast_rate (messages/sec)
qs_sub_broadcast_latency_ms
```

**End-to-End:**
```
e2e_latency_ms (AODB persist → Client receive)
e2e_message_count (total messages delivered)
```

### Alerts

- **High Outbox Age** (> 10 seconds)
  - Kafka publishing issues
  - Scale up Outbox Processors

- **High Consumer Lag** (> 1000 messages)
  - QueryStream can't keep up
  - Scale up DB Manager instances

- **Low Connection Count** (sudden drop)
  - Subscription Manager crash
  - Network issues

## Error Handling

### Publishing Failures

**Scenario:** Kafka unavailable

**Handling:**
```csharp
try
{
    await kafkaProducer.ProduceAsync(topic, message);
}
catch (ProduceException ex)
{
    // Message stays in outbox
    // Will be retried by Outbox Processor
    _logger.LogWarning(ex, 
        "Failed to publish, will retry");
}
```

**Result:** No data loss, eventual delivery

### Compilation Failures

**Scenario:** Invalid feed data

**Handling:**
```csharp
try
{
    var feedItem = await CompileFeedAsync(message);
    await _db.SaveChangesAsync();
}
catch (Exception ex)
{
    _logger.LogError(ex, 
        "Failed to compile feed for {EntityId}",
        message.EntityId);
    
    // Move to dead letter
    await _deadLetter.AddAsync(message, ex);
    
    // Don't commit offset - will retry
    throw;
}
```

**Result:** Retry, then dead letter

### Broadcasting Failures

**Scenario:** Client disconnected

**Handling:**
```csharp
try
{
    await _hubContext.Clients
        .Client(connectionId)
        .SendAsync("FeedUpdate", update);
}
catch (Exception ex)
{
    _logger.LogWarning(ex,
        "Failed to send to {ConnectionId}",
        connectionId);
    
    // Remove subscription
    await _subscriptions.RemoveByConnectionAsync(connectionId);
}
```

**Result:** Subscription cleaned up, client will reconnect and resubscribe

## Related Notes

- [[Inbound Message Flow]] - How data enters AIRHART
- [[AODB Processing Pipeline]] - How data is processed
- [[Kafka Messaging]] - Kafka infrastructure
- [[QueryStream Component]] - QueryStream details
- [[AODB Component]] - AODB details

---

**Tags:** #dataflow #outbound #querystream #realtime #websocket
**Created:** 2025-12-15
