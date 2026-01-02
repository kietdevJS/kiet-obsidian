# Recalculation and Dependency Tracking

> How AODB detects when entities need to be recalculated

## Overview

When data changes in AIRHART (Master Data updates, operational events, visit linking), **dependent entities may need to be recalculated**. AODB has sophisticated mechanisms to detect these dependencies and trigger recalculation of affected entities.

**Key Question:** *How does AODB know that a FlightLeg needs to be recalculated when a Master Data entry (like an airline, stand, or aircraft type) changes?*

**Answer:** Through **dependency tracking**, **enrichment rules**, and **matching/mapping logic** that declare what Master Data they depend on.

## Recalculation Triggers

```
┌──────────────────────────────────────────────────────────┐
│         What Triggers Recalculation?                     │
├──────────────────────────────────────────────────────────┤
│  1. Master Data Updates                                  │
│     • Airline changes                                    │
│     • Aircraft type changes                              │
│     • Stand/gate changes                                 │
│     • Airport configuration changes                      │
│                                                          │
│  2. Business Rule Updates                                │
│     • New enrichment rules deployed                      │
│     • Rule logic changed                                 │
│     • Alert rules modified                               │
│                                                          │
│  3. Operational Event Changes                            │
│     • INOP (stand closure)                               │
│     • Runway status changes                              │
│     • De-icing platform updates                          │
│                                                          │
│  4. Visit Linking                                        │
│     • Arrival + Departure linked                         │
│     • Registration propagated                            │
│     • Aircraft type propagated                           │
│                                                          │
│  5. Rule-Triggered Dependencies                          │
│     • Rule explicitly triggers recalculation             │
│     • Dependency chain detected                          │
└──────────────────────────────────────────────────────────┘
```

## Master Data Recalculation

### How It Works

When Master Data changes, **two topics** are published:

1. **`masterdata-data-update-topic`** - Contains full before/after data
2. **`masterdata-update-delta`** - Contains only affected time ranges and attributes

**AODB subscribes to `masterdata-update-delta`** to determine which entities need recalculation.

### Message Format

**Delta Message Example:**

```json
{
  "EntityName": "hard_linking_rule",
  "Changes": [
    {
      "From": "2021-08-24T00:00:00Z",
      "To": "2021-10-24T00:00:00Z",
      "Attributes": ["days_of_operation", "frequency"]
    }
  ]
}
```

**What This Means:**
- The Master Data entity `hard_linking_rule` changed
- Between August 24 and October 24, 2021
- The attributes `days_of_operation` and `frequency` were modified
- **All FlightLegs in this time range that depend on this entity must be recalculated**

### Determining Affected Time Ranges

**Master Data uses bitemporal storage:**

- **Valid Time:** When the data is valid in the real world
- **Registration Time:** When the data was entered into the system

**Example Change:**

```
OLD: Stand "A5" valid from 2021-08-01 to 2021-10-01
NEW: Stand "A5" valid from 2021-08-01 to 2021-12-31
     
Δ (Delta): Recalculate flights from 2021-10-01 to 2021-12-31
```

**Visualization:**

```
Timeline:        Aug 1           Oct 1           Dec 31
                 |               |               |
OLD:             [===============]
NEW:             [================================]
                                 ^^^^^^^^^^^^^^^^^
                                 This range changed!
```

The delta message identifies this **changed range** (Oct 1 to Dec 31) so AODB can query for all flights in that period.

### Recalculation Process

**Flow:**

```
Master Data Update
       ↓
1. Calculate delta (before/after comparison)
       ↓
2. Publish to masterdata-update-delta topic
       ↓
3. AODB consumes delta message
       ↓
4. Query for affected entities
   (FlightLegs with BestInBlockTime or BestOffBlockTime in delta range)
       ↓
5. Mark entities for recalculation
   (Set ProcessingStatus = Requested, ProcessingReasons = MasterdataUpdate)
       ↓
6. Publish recalculation messages to Kafka
   (traffic-operational-entityvalue-recalculation-*)
       ↓
7. AODB Pipeline consumes and reprocesses entities
```

**Code Example:**

```csharp
public class DefaultMasterdataUpdateRecalculationRequester : 
    BaseRecalculationRequester<MasterDataDataUpdateDelta>
{
    private readonly ILookupIntervalProvider manager;

    protected override IEnumerable<(DateTime lower, DateTime upper)> 
        GetBoundaries(MasterDataDataUpdateDelta request) =>
        manager.GetBoundaries(request);

    protected override ProcessingReasons GetProcessingReasons() => 
        ProcessingReasons.MasterdataUpdate;
}
```

**Query Example:**

```csharp
// Find all FlightLegs in affected time range
var affectedFlights = context.Aodb.FlightLeg
    .Where(f => 
        (f.BestInBlockTime >= change.From && f.BestInBlockTime <= change.To) ||
        (f.BestOffBlockTime >= change.From && f.BestOffBlockTime <= change.To))
    .Select(f => f.Id)
    .ToList();

// Mark for recalculation
foreach (var flightId in affectedFlights)
{
    await MarkForRecalculation(
        flightId, 
        reason: "MasterData update: hard_linking_rule",
        priority: false);
}
```

## Enrichment Rule Dependencies

### Dependency Declaration

Enrichment rules **implicitly declare dependencies** by querying Master Data during enrichment.

**Example: Aircraft Details Enrichment**

```csharp
public class AssignAircraftDetailsSubRule : IEnrichmentSubrule<IFlightLeg>
{
    public void Enrich(IFlightLeg flightLeg, RuleContext context)
    {
        var registration = flightLeg.Registration.Best?.Value;
        if (registration == null) return;

        // Query Master Data for aircraft details
        var aircraft = context.Masterdata.Aircraft
            .FirstOrDefault(a => a.Registration == registration);

        if (aircraft != null)
        {
            // Enrich from Master Data
            flightLeg.AircraftTypeIata.Best = aircraft.AircraftTypeIata;
            flightLeg.MaximumTakeoffWeight.Best = aircraft.MTOW;
            flightLeg.ServiceType.Best = aircraft.ServiceType;
        }
    }
}
```

**Dependencies:**
- This rule **depends on Master Data entity `Aircraft`**
- When an `Aircraft` entry changes (e.g., registration reassigned to different aircraft type)
- All FlightLegs with that registration **must be recalculated**

### Visit Enrichment Re-triggering

**Scenario:** When two flights are linked in a visit, **data propagates** between them.

**Example:**
- Arrival flight has `Registration = "LN-ABC"`
- Departure flight has `Registration = null`
- After linking: Departure inherits `Registration = "LN-ABC"`

**Problem:** Departure flight's `Aircraft Type` should now be enriched based on "LN-ABC"!

**Solution:** Re-run enrichment rules that depend on `Registration` or `AircraftType`

**Code Implementation:**

```csharp
public class VisitEnrichmentRule : IEnrichmentRule<IVisit>
{
    public void Enrich(IVisit visit, RuleContext context)
    {
        // Link arrival and departure
        LinkFlights(visit);

        // Check if registration or aircraft type was propagated
        bool arrivalRegistrationChanged = 
            visit.ArrivalFlight.Registration.Previous != 
            visit.ArrivalFlight.Registration.Best;
            
        bool departureRegistrationChanged = 
            visit.DepartureFlight.Registration.Previous != 
            visit.DepartureFlight.Registration.Best;

        // Re-trigger dependent enrichment rules
        if (arrivalRegistrationChanged)
        {
            RetriggerEnrichment(visit.ArrivalFlight);
        }
        
        if (departureRegistrationChanged)
        {
            RetriggerEnrichment(visit.DepartureFlight);
        }
    }

    private void RetriggerEnrichment(IFlightLeg flight)
    {
        // Rules that depend on Registration or AircraftType
        new AssignAircraftDetailsSubRule().Enrich(flight, context);
        new AssignExpectedFlyingTimeSubRule().Enrich(flight, context);
        new SetEstimatedInBlockTimeSubRule().Enrich(flight, context);
        new AssignGroundHandlerSubRule().Enrich(flight, context);
        new AssignPublicFlightIdentifierSubRule().Enrich(flight, context);
    }
}
```

**List of Re-triggered Rules:**

1. **AssignAircraftDetailsSubRule** - Aircraft details from registration
2. **AssignExpectedFlyingTimeSubRule** - Flying time depends on aircraft type
3. **SetEstimatedInBlockTimeSubRule** - Estimated time depends on flying time
4. **AssignGroundHandlerSubRule** - Handler depends on aircraft type
5. **AssignPublicFlightIdentifierSubRule** - Identifier depends on handler

## Operational Event Recalculation

### Trigger: INOP or State Changes

**Operational Events** (like stand closures, runway closures) affect flights in a time window.

**Rule: TriggerRecalculationForEventsSubRule**

```csharp
public class TriggerRecalculationForEventsSubRule : 
    IEnrichmentSubrule<IOperationalEvent>
{
    public void Enrich(IOperationalEvent inputMessage, RuleContext context)
    {
        bool isRecurring = inputMessage.IsRecurringEvent.Best?.Value == true;
        bool isOpenEnded = inputMessage.TimelineGroupId.Best != null;
        bool isDeleted = inputMessage.Deleted.Best?.Value == true;
        var start = inputMessage.Start.Best?.Value;
        var end = inputMessage.End.Best?.Value;

        if (!isRecurring && !isOpenEnded && start != null && end != null)
        {
            // Ignore events too far in future (>72h) or too far in past (>2h)
            if (start - DateTime.UtcNow > TimeSpan.FromHours(72) || 
                end.Value < DateTime.UtcNow.AddHours(-2))
            {
                return;
            }

            // Find affected flights in time range
            var flights = context.Aodb.FlightLeg
                .Where(f => 
                    (f.BestInBlockTime >= start && f.BestInBlockTime <= end) ||
                    (f.BestOffBlockTime >= start && f.BestOffBlockTime <= end));

            // If event is stand-specific, filter by stand
            var stand = inputMessage.RelatedStandName.Best?.Value?.ToLower();
            if (stand != null)
            {
                flights = flights.Where(f => 
                    (f.ArrivalStand != null && f.ArrivalStand.ToLower() == stand) ||
                    (f.DepartureStand != null && f.DepartureStand.ToLower() == stand));
            }

            var flightIds = flights.Select(f => f.Id).ToList();

            // Trigger recalculation
            inputMessage.AddRecalculation(flightIds, "Check INOP");
        }
    }
}
```

**Example:**

```
Event: Stand A5 closed from 10:00 to 14:00
       
Query: Find all flights assigned to Stand A5 with:
       • BestInBlockTime between 10:00-14:00, OR
       • BestOffBlockTime between 10:00-14:00
       
Result: 15 flights found → Mark for recalculation
        
Why: These flights may need:
     • Alert raised (stand unavailable)
     • Alternative stand assigned
     • Turnaround time recalculated
```

### Delta Time Ranges

When an event **changes** (start/end time updated), calculate delta range:

```csharp
private IQueryable<IFlightLeg> FindAffectedFlights(
    RuleContext context, 
    DateTime start, DateTime end, 
    DateTime previousStart, DateTime previousEnd)
{
    // If both start and end changed
    if (start > previousStart && end > previousEnd)
    {
        // Two delta ranges:
        // 1. previousStart → start (new flights entering)
        // 2. previousEnd → end (event extended)
        return context.Aodb.FlightLeg.Where(
            Within(previousStart, start) || Within(previousEnd, end));
    }
    // ... other scenarios
}
```

**Visual:**

```
OLD Event:    [========]
              10:00  12:00

NEW Event:    [================]
              10:00         14:00
                            ^^^^
                            Delta: 12:00-14:00
                            
Flights in delta range need recalculation!
```

## De-icing Sequence Recalculation

**Trigger:** Flights added/removed from de-icing sequence

```csharp
public class TriggerRecalculationSubRule : IEnrichmentSubrule<IDeicingSequence>
{
    public void Enrich(IDeicingSequence inputMessage, RuleContext context)
    {
        var sequence = JsonSerializer.Deserialize<List<DeicingSequenceEntry>>(
            inputMessage.Sequence.Best?.Value);

        // Find flights currently flagged for de-icing
        var flightsWithDeicingFlag = context.Aodb.FlightLeg
            .Where(flight => flight.HasRequestedDeicing == true)
            .Select(flight => new { flight.Id, flight.DeicingPlatform })
            .ToList();

        // Flights removed from sequence (were flagged, now not in sequence)
        var flightsRemovedFromSequence = flightsWithDeicingFlag
            .Where(f => !sequence.Any(entry => 
                f.Id == entry.FlightId && 
                f.DeicingPlatform == entry.DeicePlatform))
            .Select(f => f.Id);

        inputMessage.AddRecalculation(
            flightsRemovedFromSequence, 
            "[Enrichment] NITOS DeiceSequence");

        // Flights added to sequence (not flagged before, now in sequence)
        var flightsAddedToSequence = sequence
            .Where(entry => entry.FlightId.HasValue && 
                !flightsWithDeicingFlag.Any(f => f.Id == entry.FlightId))
            .Select(entry => entry.FlightId!.Value);

        inputMessage.AddRecalculation(
            flightsAddedToSequence, 
            "[Enrichment] NITOS DeiceSequence");
    }
}
```

**Why Recalculate:**
- **Added:** Set `HasRequestedDeicing = true`, assign `DeicingPlatform`
- **Removed:** Set `HasRequestedDeicing = false`, clear platform

## Business Rule Update Recalculation

### Trigger: New Rule Version Deployed

When business rules are updated (new version deployed), **all entities may need recalculation**.

**Reason:** Rule logic changed, so existing data may be inconsistent with new rules.

**Message:**

```json
{
  "EntityType": "FlightLeg",
  "RuleVersion": "1.5.0",
  "PreviousVersion": "1.4.3",
  "TenantId": "CPH"
}
```

**Process:**

```csharp
public class BusinessRuleUpdateRecalculationRequester : 
    BaseRecalculationRequester<BusinessRulesVersion>
{
    public async Task<int> Perform(
        BusinessRulesVersion request, 
        CancellationToken ct)
    {
        // Query all entities (usually with filters for operational time)
        var entities = context.Aodb.FlightLeg
            .Where(f => f.BestInBlockTime >= DateTime.UtcNow.AddHours(-24) &&
                        f.BestInBlockTime <= DateTime.UtcNow.AddHours(72))
            .Select(f => f.Id)
            .ToList();

        // Mark all for recalculation
        foreach (var entityId in entities)
        {
            await MarkForRecalculation(
                entityId, 
                reason: $"Business Rule Update: v{request.RuleVersion}",
                priority: false);
        }

        return entities.Count;
    }
}
```

**Optimization:** Only recalculate entities in **operational window** (-24h to +72h) to avoid overwhelming the system.

## Matching and Mapping

### Message-to-Entity Matching

**Problem:** How does an inbound message find the correct FlightLeg entity?

**Matching Logic:**

```csharp
public class FlightLegMatcher : IEntityMatcher<FlightLegAcdScr>
{
    public IFlightLeg? Match(
        FlightLegAcdScr message, 
        RuleContext context)
    {
        // Try matching by unique identifiers
        var flightLeg = context.Aodb.FlightLeg
            .FirstOrDefault(f => 
                f.OperatorIata == message.OperatorIata &&
                f.FlightNumber == message.FlightNumber &&
                f.ScheduledDate == message.ScheduledDate &&
                f.OriginIata == message.OriginIata &&
                f.DestinationIata == message.DestinationIata);

        if (flightLeg != null)
        {
            // Match found!
            return flightLeg;
        }

        // No match - create new entity
        return null;
    }
}
```

**Matching Criteria:**
- Operator IATA + Flight Number + Date
- Origin + Destination
- May also check: Aircraft Registration, STD/STA

**Why Matching Matters for Recalculation:**
- If matching logic changes, messages may match **different entities**
- Old entities may become **orphaned** (no more updates)
- Need to recalculate both old and new matches

### Mapping Attributes

**Mapping Rules** transform DTO attributes to entity attributes:

```csharp
public class FlightLegAcdMappingRule : IMappingRule<FlightLegAcdScr, IFlightLeg>
{
    public void Map(FlightLegAcdScr message, IFlightLeg entity, RuleContext context)
    {
        // Direct mapping
        entity.FlightNumber.Best = message.FlightNumber;
        entity.OperatorIata.Best = message.OperatorIata;

        // Lookup Master Data
        var airline = context.Masterdata.Airline
            .FirstOrDefault(a => a.IataCode == message.OperatorIata);

        if (airline != null)
        {
            entity.OperatorName.Best = airline.Name;
            entity.OperatorIcao.Best = airline.IcaoCode;
        }
        else
        {
            // Missing Master Data - alert and mark for recalculation when MD available
            entity.AddAlert(AlertLevel.Medium, "Airline not found in Master Data");
        }
    }
}
```

**Dependency:**
- This mapping **depends on Master Data `Airline`**
- If `Airline` Master Data changes → Recalculate all FlightLegs with that operator

## Rule-Triggered Recalculation

### Explicit Recalculation in Rules

Rules can **explicitly trigger recalculation** of other entities:

```csharp
public class LinkFlightsRule : IEnrichmentSubrule<IVisit>
{
    public void Enrich(IVisit visit, RuleContext context)
    {
        // Link arrival and departure
        visit.ArrivalFlightId.Best = visit.ArrivalFlight.Id;
        visit.DepartureFlightId.Best = visit.DepartureFlight.Id;

        // Propagate registration
        if (visit.ArrivalFlight.Registration.Best != null &&
            visit.DepartureFlight.Registration.Best == null)
        {
            visit.DepartureFlight.Registration.Best = 
                visit.ArrivalFlight.Registration.Best;

            // Trigger recalculation of departure flight
            visit.AddRecalculation(
                new[] { visit.DepartureFlight.Id }, 
                "Registration propagated from arrival");
        }
    }
}
```

**API:**

```csharp
// Trigger recalculation from within a rule
entity.AddRecalculation(
    entityIds: new[] { flightId1, flightId2 },
    reason: "Linked visit created");

// Or on input message
inputMessage.AddRecalculation(
    entityIds: affectedFlightIds,
    reason: "Check INOP");
```

**Implementation:**

```csharp
internal class RuleTriggeredRecalculationRequester : 
    IRecalculationRequester<RuleTriggeredRecalculationEvent>
{
    public async Task<int> Perform(
        RuleTriggeredRecalculationEvent request, 
        CancellationToken ct)
    {
        // Exclude entities already being processed in current pipeline
        var triggers = request.RecalculationTriggers
            .Where(t => !request.EntityValuesToBePersisted.Any(e => e.Id == t.Id));

        int count = 0;

        // Group by reason for efficient batch queries
        var queries = triggers
            .GroupBy(t => t.Reason)
            .Select(g => BuildQuery(g.Key, g.Select(t => t.Id)));

        foreach (var (statement, parameters) in queries)
        {
            count += await queryPerformer.Execute(statement, parameters, ct);
        }

        // For entities in current pipeline, update status directly
        count += UpdateStatusOnEntityValuesToBePersisted(request);

        return count;
    }

    private int UpdateStatusOnEntityValuesToBePersisted(
        RuleTriggeredRecalculationEvent request)
    {
        var recalculations = request.RecalculationTriggers
            .Join(request.EntityValuesToBePersisted, 
                trigger => trigger.Id, 
                ev => ev.Id,
                (trigger, ev) => (ev, trigger.Reason))
            .ToList();

        foreach (var (entity, reason) in recalculations)
        {
            var processingState = entity.ProcessingState;
            processingState.ProcessingStatus = ProcessingStatus.Requested;
            processingState.ProcessingReasons |= 
                ProcessingReasons.RuleTrigger | ProcessingReasons.Priority;
            processingState.ProcessingRuleTriggerReasons = 
                AppendReason(processingState.ProcessingRuleTriggerReasons, reason);
        }

        return recalculations.Count;
    }
}
```

## Recalculation Queue

### Processing Status

Each entity has a **processing status**:

```csharp
public enum ProcessingStatus
{
    None = 0,
    Pending = 1,      // New message, not yet processed
    Processing = 2,   // Currently in pipeline
    Processed = 3,    // Successfully processed
    Requested = 4,    // Recalculation requested
    Failed = 5        // Processing failed
}
```

**Reasons Flags:**

```csharp
[Flags]
public enum ProcessingReasons
{
    None = 0,
    InboundMessage = 1,        // New message from adapter
    MasterdataUpdate = 2,      // Master Data changed
    BusinessRuleUpdate = 4,    // Rules updated
    RuleTrigger = 8,           // Triggered by another rule
    Priority = 16,             // High priority (operational)
    Scheduled = 32             // Scheduled recalculation
}
```

### Kafka Topics for Recalculation

**Recalculation Messages Published To:**

| Topic | Priority | Use Case |
|-------|----------|----------|
| `traffic-operational-entityvalue-recalculation-flight_leg` | High | Operational flights (±72h) |
| `traffic-operational-entityvalue-recalculation-visit` | High | Active visits |
| `traffic-planning-entityvalue-recalculation-flight_leg` | Low | Planning flights (>72h future) |

**Message Format:**

```json
{
  "EntityType": "FlightLeg",
  "EntityId": "abc-123-def",
  "ProcessingReasons": "MasterdataUpdate|RuleTrigger",
  "TriggerReason": "Master Data update: Airline SK",
  "Timestamp": "2025-12-15T10:30:00.000Z",
  "Priority": true
}
```

### Consuming Recalculation Messages

**AODB Pipeline Consumer:**

```csharp
public class RecalculationConsumer : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        consumer.Subscribe("traffic-operational-entityvalue-recalculation-flight_leg");

        while (!ct.IsCancellationRequested)
        {
            var message = consumer.Consume(TimeSpan.FromSeconds(1));
            if (message == null) continue;

            var recalcRequest = message.Value;

            // Load entity from database
            var entity = await LoadEntity(recalcRequest.EntityId);

            // Re-run entire pipeline
            await pipeline.Process(entity, isRecalculation: true);

            // Commit offset
            consumer.Commit(message);
        }
    }
}
```

**Recalculation vs. New Message:**

| Aspect | New Message | Recalculation |
|--------|-------------|---------------|
| **Source** | External adapter | Internal trigger |
| **DTO** | Fresh DTO from adapter | No DTO (uses existing data) |
| **Matching** | Required | Skip (entity known) |
| **Mapping** | Required | Skip (data unchanged) |
| **Enrichment** | Full enrichment | Full enrichment |
| **Business Rules** | Pre/Post Persist | Pre/Post Persist |
| **Persistence** | Update with new data | Update with recalculated fields |

## Optimization Strategies

### 1. Time Windows

Don't recalculate **everything** - limit to operational window:

```csharp
// Only recalculate flights in ±72h window
var affectedFlights = context.Aodb.FlightLeg
    .Where(f => 
        f.BestInBlockTime >= DateTime.UtcNow.AddHours(-72) &&
        f.BestInBlockTime <= DateTime.UtcNow.AddHours(72))
    .ToList();
```

**Why:**
- Old flights (>72h past): Already operated, data is historical
- Far future flights (>72h future): Will get updated messages before operation
- Focus on **operational flights** that users are actively monitoring

### 2. Batch Processing

Group recalculations into batches:

```csharp
// Instead of 1000 individual messages
foreach (var flightId in affectedFlights)
{
    await PublishRecalculation(flightId); // ❌ Inefficient
}

// Batch into groups of 100
var batches = affectedFlights.Chunk(100);
foreach (var batch in batches)
{
    await PublishRecalculationBatch(batch); // ✅ Efficient
}
```

### 3. Priority Queues

Separate **operational** from **planning** recalculations:

```csharp
if (flight.BestInBlockTime <= DateTime.UtcNow.AddHours(72))
{
    // High priority - operational
    await PublishTo("traffic-operational-entityvalue-recalculation-*");
}
else
{
    // Low priority - planning
    await PublishTo("traffic-planning-entityvalue-recalculation-*");
}
```

### 4. Deduplication

Avoid recalculating same entity multiple times:

```csharp
// Track entities already marked for recalculation
var recalculationSet = new HashSet<Guid>();

foreach (var trigger in triggers)
{
    if (recalculationSet.Add(trigger.EntityId))
    {
        // First time seeing this entity
        await MarkForRecalculation(trigger.EntityId, trigger.Reason);
    }
    else
    {
        // Already marked - append reason
        await AppendRecalculationReason(trigger.EntityId, trigger.Reason);
    }
}
```

### 5. Incremental Enrichment

Only re-run enrichment rules that **depend on changed data**:

```csharp
// Instead of full re-enrichment
await RunAllEnrichmentRules(entity); // ❌ Slow

// Determine what changed
var changedAttributes = DetermineChanges(entity);

// Only run dependent rules
if (changedAttributes.Contains("Registration"))
{
    await RunRegistrationDependentRules(entity); // ✅ Fast
}

if (changedAttributes.Contains("AircraftType"))
{
    await RunAircraftTypeDependentRules(entity);
}
```

## Monitoring

### Metrics

```
aodb_recalculation_requested_total{reason="masterdata"}
aodb_recalculation_requested_total{reason="business_rule"}
aodb_recalculation_requested_total{reason="operational_event"}
aodb_recalculation_requested_total{reason="rule_trigger"}

aodb_recalculation_processed_total
aodb_recalculation_failed_total
aodb_recalculation_latency_seconds
```

### Alerts

**High Recalculation Rate:**
```
aodb_recalculation_requested_total > 10000 in 1m
```
- Possible: Large Master Data import
- Possible: Business rule update affecting many entities
- Action: Check recalculation queues, scale consumers

**Recalculation Lag:**
```
aodb_recalculation_lag_seconds > 300
```
- Entities waiting >5 minutes for recalculation
- Action: Scale up pipeline instances

## Example Scenarios

### Scenario 1: Airline Name Change

**Event:** Master Data - Airline "SK" name changed from "SAS" to "SAS Scandinavian Airlines"

**Flow:**

1. Master Data publishes delta:
   ```json
   {
     "EntityName": "Airline",
     "Changes": [{
       "From": "2025-01-01T00:00:00Z",
       "To": "9999-12-31T23:59:59Z",
       "Attributes": ["Name"]
     }]
   }
   ```

2. AODB queries affected flights:
   ```sql
   SELECT id FROM flight_leg
   WHERE operator_iata = 'SK'
     AND best_in_block_time >= '2025-01-01'
     AND best_in_block_time <= '2025-12-31'
   ```

3. Result: 15,000 flights found

4. Publish recalculation messages (batched):
   - 15,000 messages to `traffic-operational-entityvalue-recalculation-flight_leg`

5. Pipeline reprocesses each flight:
   - Enrichment rule looks up airline name from Master Data
   - `OperatorName.Best = "SAS Scandinavian Airlines"`
   - Persist updated entity

6. QueryStream receives updates:
   - Feeds updated with new airline name
   - Clients see updated name in UI

**Timeline:** ~10 minutes for 15,000 flights (25 flights/second throughput)

### Scenario 2: Stand Closure (INOP)

**Event:** Stand A5 closed 10:00-14:00

**Flow:**

1. Operational Event message received

2. `TriggerRecalculationForEventsSubRule` runs:
   ```csharp
   var flights = context.Aodb.FlightLeg
       .Where(f => 
           (f.BestInBlockTime >= "10:00" && f.BestInBlockTime <= "14:00") ||
           (f.BestOffBlockTime >= "10:00" && f.BestOffBlockTime <= "14:00"))
       .Where(f => 
           f.ArrivalStand == "A5" || f.DepartureStand == "A5")
       .Select(f => f.Id)
       .ToList();
   ```

3. Result: 12 flights affected

4. Mark for recalculation with reason "Check INOP"

5. Pipeline reprocesses:
   - Business rules check stand availability
   - Alert raised: "Stand A5 unavailable 10:00-14:00"
   - Alternative stand may be assigned (if rule configured)

6. Users see alerts in UI for affected flights

**Timeline:** ~30 seconds for 12 flights

### Scenario 3: Visit Linking

**Event:** Flights linked in visit, registration propagated

**Flow:**

1. Arrival flight (SK1234) has `Registration = "LN-ABC"`
2. Departure flight (SK1235) has `Registration = null`

3. Visit enrichment rule links them:
   ```csharp
   visit.DepartureFlight.Registration.Best = 
       visit.ArrivalFlight.Registration.Best; // "LN-ABC"
   ```

4. Registration propagated → Trigger recalculation:
   ```csharp
   visit.AddRecalculation(
       new[] { visit.DepartureFlight.Id },
       "Registration propagated from arrival");
   ```

5. Departure flight reprocessed:
   - `AssignAircraftDetailsSubRule` runs
   - Looks up "LN-ABC" in Master Data
   - Sets `AircraftTypeIata = "73H"`
   - Sets `MTOW = 79000`
   - Sets `ServiceType = "J"` (Passenger)

6. Downstream rules re-run:
   - Expected flying time recalculated based on aircraft type
   - Ground handler assigned based on aircraft type
   - Stand compatibility checked

**Timeline:** ~5 seconds for 1 flight (high priority, same pipeline execution)

## Related Notes

- [[AODB Processing Pipeline]] - Full pipeline flow
- [[Kafka Messaging]] - Recalculation topics
- [[AODB Component]] - Recalculation services

---

**Tags:** #recalculation #dependencies #masterdata #enrichment #matching
**Created:** 2025-12-15
