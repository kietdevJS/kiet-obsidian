# AODB Processing Pipeline

> Deep dive into the AODB message processing pipeline

## Overview

The AODB Pipeline is the **core processing engine** that transforms adapter messages into enriched, validated operational data. It applies business rules, enriches with master data, and publishes updated entities.

## Pipeline Architecture

### Complete Pipeline Flow

```
┌──────────────────────────────────────────────────────────────┐
│            AODB PROCESSING PIPELINE                          │
└──────────────────────────────────────────────────────────────┘

INPUT
├─ EntityValuePipelineInput (from adapter mapper)
├─ Recalculation requests (from background services)
└─ Manual updates (from API)
    ↓

┌─────────────────────────────────────────────────────────┐
│ 1. DESERIALIZATION & LOADING                           │
│    ├─ Load entity from database                        │
│    ├─ Load related entities                            │
│    └─ Prepare context                                  │
└─────────────────────────────────────────────────────────┘
    ↓

┌─────────────────────────────────────────────────────────┐
│ 2. ENRICHMENT                                           │
│    ├─ Fetch master data (airports, airlines)           │
│    ├─ Calculate derived fields                         │
│    ├─ Apply enrichment rules                           │
│    └─ Add alternatives (fallback values)               │
└─────────────────────────────────────────────────────────┘
    ↓

┌─────────────────────────────────────────────────────────┐
│ 3. BUSINESS RULES (Pre-Persist)                        │
│    ├─ Validation rules                                 │
│    ├─ Transformation rules                             │
│    ├─ Linking rules (visits, ground legs)              │
│    └─ Custom business logic                            │
└─────────────────────────────────────────────────────────┘
    ↓

┌─────────────────────────────────────────────────────────┐
│ 4. PERSISTENCE                                          │
│    ├─ Optimistic locking check (ETag)                  │
│    ├─ Save entity to database                          │
│    ├─ Update version/timestamp                         │
│    └─ Handle concurrency conflicts                     │
└─────────────────────────────────────────────────────────┘
    ↓

┌─────────────────────────────────────────────────────────┐
│ 5. BUSINESS RULES (Post-Persist)                       │
│    ├─ Trigger related entity updates                   │
│    ├─ Send notifications                               │
│    ├─ Execute actions                                  │
│    └─ Create system messages                           │
└─────────────────────────────────────────────────────────┘
    ↓

┌─────────────────────────────────────────────────────────┐
│ 6. PUBLISHING                                           │
│    ├─ Build document DTOs                              │
│    ├─ Publish to aodb-pipeline-output                  │
│    ├─ Publish adapter message status                   │
│    └─ Update DTD if schema changed                     │
└─────────────────────────────────────────────────────────┘

OUTPUT
├─ Updated entity documents → QueryStream
├─ Adapter message status → UI/monitoring
└─ Recalculation triggers → Related entities
```

## Stage Details

### 1. Deserialization & Loading

**Purpose:** Prepare entity and context for processing

**Process:**
```csharp
// 1. Determine entity ID
string entityId = input.EntityId ?? 
                  await FindExistingEntity(input.Identifier);

// 2. Load entity from database
EntityValue entity = entityId != null 
    ? await context.EntityValues.FindAsync(entityId)
    : CreateNewEntity(input);

// 3. Load related entities
var relatedEntities = await LoadRelatedEntities(entity);

// 4. Prepare pipeline context
var pipelineContext = new PipelineContext
{
    Entity = entity,
    RelatedEntities = relatedEntities,
    MasterDataCache = masterDataCache,
    InputMessage = input
};
```

**Loading Strategy:**
- **Existing Entity:** Load from database with all attributes
- **New Entity:** Create skeleton with identifier
- **Related Entities:** Lazy load as needed
- **Master Data:** Cache frequently accessed data

**Performance Optimization:**
- Batch database queries
- Cache master data
- Async I/O throughout

### 2. Enrichment

**Purpose:** Add calculated fields and external data

#### Master Data Enrichment

**Airport Enrichment:**
```csharp
public class AirportEnrichmentRule : IEnrichmentRule<FlightLeg>
{
    public async Task EnrichAsync(FlightLeg entity, EnrichmentContext context)
    {
        // Get arrival airport code
        var airportCode = entity.GetAttribute("arrival_airport")?.Value;
        
        // Fetch airport master data
        var airport = await context.MasterData.Airports
            .GetByIataCode(airportCode);
        
        // Enrich with additional fields
        entity.SetAttribute("arrival_airport_name", airport.Name);
        entity.SetAttribute("arrival_airport_timezone", airport.Timezone);
        entity.SetAttribute("arrival_airport_city", airport.City);
    }
}
```

**Airline Enrichment:**
```csharp
// Fetch airline data
var airline = await context.MasterData.Airlines
    .GetByIataCode(entity.OperatorIata);

entity.SetAttribute("operator_name", airline.Name);
entity.SetAttribute("operator_country", airline.Country);
```

**Resource Enrichment:**
```csharp
// Fetch stand/gate data
var stand = await context.MasterData.Stands
    .GetByCode(entity.ArrivalStand);

entity.SetAttribute("stand_type", stand.Type);
entity.SetAttribute("stand_terminal", stand.Terminal);
```

#### Calculated Field Enrichment

**Time Calculations:**
```csharp
// Calculate block time
var sibt = entity.GetAttribute("scheduled_in_block_time")?.Value;
var sobt = entity.GetAttribute("scheduled_off_block_time")?.Value;

if (sibt.HasValue && sobt.HasValue)
{
    var blockTime = sobt.Value - sibt.Value;
    entity.SetAttribute("scheduled_block_time", blockTime);
}
```

**Status Calculations:**
```csharp
// Determine flight status
var aibt = entity.GetAttribute("actual_in_block_time")?.Value;
var sibt = entity.GetAttribute("scheduled_in_block_time")?.Value;

string status;
if (!aibt.HasValue)
{
    status = DateTime.Now < sibt ? "Scheduled" : "Delayed";
}
else
{
    status = "Arrived";
}

entity.SetAttribute("flight_status", status);
```

#### Alternative Values

**Purpose:** Store multiple values with precedence

```csharp
entity.SetAttribute("estimated_in_block_time", new AttributeValue
{
    Value = eibt,
    Source = "TowerPrediction",
    Received = DateTime.UtcNow,
    Alternatives = new[]
    {
        new Alternative 
        { 
            Value = eibt2, 
            Source = "AirlinePrediction" 
        },
        new Alternative 
        { 
            Value = sibt, 
            Source = "Schedule" 
        }
    }
});
```

**Precedence Rules:**
- Primary value from most trusted source
- Alternatives from other sources
- Fallback to schedule if no actuals

### 3. Business Rules (Pre-Persist)

**Purpose:** Apply validation and transformation logic

#### Validation Rules

**Required Fields:**
```csharp
public class FlightLegValidationRule : IBusinessRule<FlightLeg>
{
    public Task ExecuteAsync(FlightLeg entity, RuleContext context)
    {
        // Validate required fields
        if (string.IsNullOrEmpty(entity.FlightNumber))
            throw new ValidationException("Flight number is required");
        
        if (string.IsNullOrEmpty(entity.OperatorIata))
            throw new ValidationException("Operator is required");
        
        // Validate logical constraints
        if (entity.SIBT > entity.SOBT)
            throw new ValidationException("Arrival time cannot be after departure");
        
        return Task.CompletedTask;
    }
}
```

**Field Update Restrictions:**
```csharp
public class FieldUpdateRestrictionRule : IBusinessRule<FlightLeg>
{
    public Task ExecuteAsync(FlightLeg entity, RuleContext context)
    {
        // Check if billing locked
        if (entity.IsBillingLocked)
        {
            // Only allow specific sources to update
            if (context.MessageSource != "BillingSystem")
            {
                throw new FieldUpdateRestrictionException(
                    "Flight is locked for billing");
            }
        }
        
        return Task.CompletedTask;
    }
}
```

#### Transformation Rules

**Flight Number Normalization:**
```csharp
public class FlightNumberNormalizationRule : IBusinessRule<FlightLeg>
{
    public Task ExecuteAsync(FlightLeg entity, RuleContext context)
    {
        var flightNumber = entity.FlightNumber;
        
        // Remove leading zeros
        flightNumber = flightNumber.TrimStart('0');
        
        // Ensure minimum length
        flightNumber = flightNumber.PadLeft(3, '0');
        
        entity.FlightNumber = flightNumber;
        
        return Task.CompletedTask;
    }
}
```

#### Linking Rules

**Visit Linking:**
```csharp
public class VisitLinkingRule : IBusinessRule<FlightLeg>
{
    public async Task ExecuteAsync(FlightLeg entity, RuleContext context)
    {
        // Find matching visit
        var visit = await FindMatchingVisit(entity, context);
        
        if (visit == null && ShouldCreateVisit(entity))
        {
            // Create new visit
            visit = new Visit
            {
                AircraftRegistration = entity.AircraftRegistration,
                ArrivalStation = entity.ArrivalStation
            };
            
            // Link flight leg to visit
            if (entity.IsArrival)
                visit.InboundFlightId = entity.Id;
            else
                visit.OutboundFlightId = entity.Id;
            
            await context.SaveAsync(visit);
        }
        else if (visit != null)
        {
            // Update existing visit
            await UpdateVisit(visit, entity, context);
        }
    }
}
```

#### Custom Business Logic

**Delay Calculations:**
```csharp
public class DelayCalculationRule : IBusinessRule<FlightLeg>
{
    public Task ExecuteAsync(FlightLeg entity, RuleContext context)
    {
        var sibt = entity.SIBT;
        var eibt = entity.EIBT ?? entity.AIBT;
        
        if (sibt.HasValue && eibt.HasValue)
        {
            var delay = (eibt.Value - sibt.Value).TotalMinutes;
            
            entity.SetAttribute("delay_minutes", (int)delay);
            
            // Classify delay
            string delayCategory = delay switch
            {
                < 0 => "Early",
                <= 15 => "OnTime",
                <= 60 => "Delayed",
                _ => "SeverelyDelayed"
            };
            
            entity.SetAttribute("delay_category", delayCategory);
        }
        
        return Task.CompletedTask;
    }
}
```

### 4. Persistence

**Purpose:** Save entity to database with concurrency control

**Process:**
```csharp
public async Task<SaveResult> SaveEntityAsync(
    EntityValue entity, 
    PipelineContext context)
{
    try
    {
        // 1. Check optimistic locking
        if (entity.Version != context.OriginalVersion)
        {
            throw new DbUpdateConcurrencyException(
                "Entity was modified by another process");
        }
        
        // 2. Update metadata
        entity.Version++;
        entity.LastModified = DateTime.UtcNow;
        entity.ModifiedBy = context.MessageSource;
        
        // 3. Save to database
        if (entity.IsNew)
            context.DbContext.EntityValues.Add(entity);
        else
            context.DbContext.EntityValues.Update(entity);
        
        await context.DbContext.SaveChangesAsync();
        
        return SaveResult.Success(entity);
    }
    catch (DbUpdateConcurrencyException ex)
    {
        // Retry with fresh entity
        return SaveResult.Retry(ex);
    }
}
```

**Concurrency Handling:**
- **Optimistic Locking:** Version/ETag check
- **Retry on Conflict:** Load fresh entity and retry
- **Max Retries:** 5 attempts (configurable)
- **Exponential Backoff:** Wait between retries

**Transaction Management:**
- Each entity save in separate transaction
- Related entity updates in same transaction
- Outbox pattern for guaranteed publishing

### 5. Business Rules (Post-Persist)

**Purpose:** Trigger side effects after successful save

#### Related Entity Updates

**Trigger Visit Recalculation:**
```csharp
public class TriggerVisitRecalculationRule : IBusinessRule<FlightLeg>
{
    public async Task ExecuteAsync(FlightLeg entity, RuleContext context)
    {
        // If flight leg has a visit, recalculate it
        if (entity.VisitId.HasValue)
        {
            await context.PublishRecalculationRequest(
                entityType: "visit",
                entityId: entity.VisitId.Value,
                reason: $"Flight leg {entity.Id} updated"
            );
        }
    }
}
```

#### Notifications

**Send Delay Notification:**
```csharp
public class DelayNotificationRule : IBusinessRule<FlightLeg>
{
    public async Task ExecuteAsync(FlightLeg entity, RuleContext context)
    {
        // Check if delay increased significantly
        var oldDelay = context.OriginalEntity.DelayMinutes;
        var newDelay = entity.DelayMinutes;
        
        if (newDelay - oldDelay > 30)
        {
            // Send notification
            await context.NotificationService.SendAsync(
                new DelayNotification
                {
                    FlightNumber = entity.FlightNumber,
                    DelayMinutes = newDelay,
                    Reason = entity.DelayReason
                }
            );
        }
    }
}
```

#### Actions

**Execute API Action:**
```csharp
public class TobtUpdateActionRule : IBusinessRule<FlightLeg>
{
    public async Task ExecuteAsync(FlightLeg entity, RuleContext context)
    {
        // If TOBT changed, update external system
        var oldTobt = context.OriginalEntity.TOBT;
        var newTobt = entity.TOBT;
        
        if (oldTobt != newTobt)
        {
            await context.ActionExecutor.ExecuteAsync(
                actionName: "UpdateAmadeusTobt",
                parameters: new
                {
                    FlightNumber = entity.FlightNumber,
                    TOBT = newTobt
                }
            );
        }
    }
}
```

### 6. Publishing

**Purpose:** Distribute updated entity to consumers

#### Build Document DTOs

**Entity Document:**
```csharp
public class EntityDocument
{
    public string EntityId { get; set; }
    public string EntityType { get; set; }
    public long Version { get; set; }
    public string Operation { get; set; }  // create, update, delete
    public DateTime Timestamp { get; set; }
    public Dictionary<string, object> Data { get; set; }
}
```

**Document Building:**
```csharp
var document = new EntityDocument
{
    EntityId = entity.Id,
    EntityType = entity.EntityType,
    Version = entity.Version,
    Operation = entity.IsNew ? "create" : "update",
    Timestamp = DateTime.UtcNow,
    Data = BuildDocumentData(entity)
};
```

#### Publish to Topics

**AODB Pipeline Output:**
```csharp
await kafkaProducer.ProduceAsync(
    topic: "aodb-pipeline-output",
    key: entity.Id,
    value: document,
    headers: new[]
    {
        new Header("traceparent", traceId),
        new Header("content-type", "application/json")
    }
);
```

**Adapter Message Status:**
```csharp
var adapterMessage = new AdapterMessageDocument
{
    Id = context.RawMessage.Id,
    Status = "processed",
    EntityId = entity.Id,
    EntityType = entity.EntityType,
    ProcessedTimestamp = DateTime.UtcNow,
    TraceId = context.TraceId
};

await kafkaProducer.ProduceAsync(
    topic: "traffic-operational-adapter-messages",
    key: adapterMessage.Id,
    value: adapterMessage
);
```

#### DTD Publishing

**When to Publish:**
- Schema definition changed
- New entity type added
- Attribute added/removed/modified

**DTD Message:**
```csharp
var dtd = new DocumentTypeDefinition
{
    TypeName = "flight_leg",
    SourceComponent = "AODB",
    Version = 2,
    Schema = BuildJsonSchema(entity.EntityType)
};

await kafkaProducer.ProduceAsync(
    topic: "aodb-pipeline-dtd",
    key: dtd.TypeName,
    value: dtd
);
```

## Pipeline Execution

### Execution Flow

```csharp
public async Task<ProcessingResult> ExecutePipelineAsync(
    EntityValuePipelineInput input)
{
    var context = await PrepareContext(input);
    
    try
    {
        // 1. Load
        var entity = await LoadEntity(context);
        
        // 2. Enrich
        await ExecuteEnrichmentRules(entity, context);
        
        // 3. Pre-persist rules
        await ExecutePrePersistRules(entity, context);
        
        // 4. Persist
        await SaveEntity(entity, context);
        
        // 5. Post-persist rules
        await ExecutePostPersistRules(entity, context);
        
        // 6. Publish
        await PublishDocuments(entity, context);
        
        return ProcessingResult.Success(entity);
    }
    catch (Exception ex)
    {
        return ProcessingResult.Error(ex);
    }
}
```

### Rule Execution Order

**Pre-Persist Rules (ordered):**
1. Validation rules (fail fast)
2. Transformation rules (normalize data)
3. Enrichment rules (add data)
4. Linking rules (cross-entity updates)
5. Custom business logic

**Post-Persist Rules (parallel):**
- Notifications (fire and forget)
- Actions (best effort)
- Related entity triggers (async)

### Plugin Loading

**Discovery:**
```csharp
// Scan plugin assemblies
var pluginAssemblies = Directory.GetFiles(
    "plugins", 
    "*.BusinessRules.dll"
);

// Load types implementing rule interfaces
var ruleTypes = pluginAssemblies
    .SelectMany(a => Assembly.LoadFrom(a).GetTypes())
    .Where(t => t.IsAssignableTo(typeof(IBusinessRule)));
```

**Registration:**
```csharp
services.AddScoped(ruleType);
```

**Execution:**
```csharp
var rules = serviceProvider.GetServices<IBusinessRule<FlightLeg>>();

foreach (var rule in rules.OrderBy(r => r.Order))
{
    await rule.ExecuteAsync(entity, context);
}
```

## Performance

### Optimization Techniques

**Batch Processing:**
```csharp
// Load related entities in batch
var relatedIds = entities.SelectMany(e => e.RelatedEntityIds);
var relatedEntities = await context.EntityValues
    .Where(e => relatedIds.Contains(e.Id))
    .ToListAsync();
```

**Master Data Caching:**
```csharp
// Cache frequently accessed master data
var airportCache = new MemoryCache();

Airport GetAirport(string iataCode)
{
    return airportCache.GetOrCreate(iataCode, entry =>
    {
        entry.SlidingExpiration = TimeSpan.FromHours(1);
        return masterDataService.GetAirportAsync(iataCode);
    });
}
```

**Async I/O:**
```csharp
// All database and external calls async
await Task.WhenAll(
    LoadEntityAsync(id),
    LoadMasterDataAsync(),
    LoadRelatedEntitiesAsync(ids)
);
```

### Throughput

**Targets:**
- Single entity: 10ms - 100ms
- Batch of 100: 500ms - 1s
- Messages/second: 10,000 (operational) / 100,000 (batch)

**Scaling:**
- 50 pipeline pods × 50 partitions = 2,500 concurrent workers
- Each worker: ~10 messages/second
- Total: 25,000 messages/second capacity

## Error Handling

### Exception Types

See: [[Resilience Patterns]]

**Handled Exceptions:**
1. `DbUpdateConcurrencyException` - Retry forever
2. `NpgsqlException` - Retry forever (transient)
3. `LockedFlightLegException` - Skip (expected)
4. `FieldUpdateRestrictionException` - Skip (expected)
5. `InfiniteReprocessingException` - Skip (loop detection)

**Unhandled Exceptions:**
- Retry 5 times → Dead letter topic

### Infinite Loop Detection

```csharp
// Track message requester chain
if (context.RequesterChain.TakeLast(6).All(r => r == "aodb"))
{
    throw new InfiniteReprocessingException(
        "Detected infinite reprocessing loop");
}
```

**Causes:**
- Circular entity relationships
- Recursive business rules
- Linking cycles

## Monitoring

### Metrics

**Processing Metrics:**
- `aodb_pipeline_processing_duration_seconds{stage="enrichment"}`
- `aodb_pipeline_processing_duration_seconds{stage="rules"}`
- `aodb_pipeline_processing_duration_seconds{stage="persistence"}`

**Entity Metrics:**
- `aodb_entities_processed_total{entity_type="flight_leg"}`
- `aodb_entities_created_total{entity_type="visit"}`
- `aodb_entities_updated_total{entity_type="ground_leg"}`

**Error Metrics:**
- `aodb_processing_errors_total{error_type="concurrency"}`
- `aodb_processing_errors_total{error_type="validation"}`
- `aodb_deadletter_messages_total`

### Tracing

**Span Hierarchy:**
```
Trace: abc-123-def
├─ Span: AdapterConsume (100ms)
├─ Span: Mapping (50ms)
├─ Span: Matching (50ms)
└─ Span: PipelineExecution (500ms)
    ├─ Span: Loading (50ms)
    ├─ Span: Enrichment (100ms)
    │   ├─ Span: MasterDataFetch (80ms)
    │   └─ Span: CalculateFields (20ms)
    ├─ Span: PrePersistRules (100ms)
    ├─ Span: Persistence (150ms)
    ├─ Span: PostPersistRules (50ms)
    └─ Span: Publishing (50ms)
```

## Related Notes

- [[AODB Component]] - Component overview
- [[Business Rules]] - Rule development
- [[Plugin Architecture]] - Plugin system
- [[Resilience Patterns]] - Error handling
- [[Entity Model]] - Data model

---

**Tags:** #pipeline #processing #aodb #business-rules #enrichment
**Created:** 2025-12-15
