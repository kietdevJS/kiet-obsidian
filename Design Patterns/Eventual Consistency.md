# Eventual Consistency

> Data becomes consistent across the system over time, not immediately

## Overview

**Eventual Consistency** is a consistency model where updates to data will **eventually** propagate throughout the system, but there's no guarantee of immediate consistency. The system tolerates temporary inconsistencies for better availability and performance.

## Strong vs Eventual Consistency

### Strong Consistency (Traditional)

```
Client writes data
      ↓
  ┌──────────┐
  │ Database │ ← Single source of truth
  └──────────┘
      ↓
All readers see update IMMEDIATELY
```

**Characteristics:**
- ✅ Always see latest data
- ✅ No stale reads
- ❌ Slower writes (wait for all replicas)
- ❌ Lower availability (can't write if replicas down)
- ❌ Doesn't scale well geographically

### Eventual Consistency (Distributed)

```
Client writes to System A
      ↓
  ┌──────────┐
  │ System A │ ← Accepts write immediately
  └──────────┘
      ↓ (asynchronous propagation)
  ┌──────────┐
  │ System B │ ← Receives update later
  └──────────┘
      ↓ (asynchronous propagation)
  ┌──────────┐
  │ System C │ ← Receives update even later
  └──────────┘

Eventually all systems have the same data
```

**Characteristics:**
- ✅ Fast writes (don't wait for propagation)
- ✅ High availability (systems independent)
- ✅ Scales globally
- ❌ Temporary inconsistency
- ❌ Must handle stale data

## CAP Theorem

Systems can have at most 2 of 3 properties:

```
┌─────────────────────────────────┐
│ C = Consistency                 │
│   All nodes see same data       │
│                                 │
│ A = Availability                │
│   System always responds        │
│                                 │
│ P = Partition Tolerance         │
│   Works despite network issues  │
└─────────────────────────────────┘

Traditional DB: C + A (single node, no partitions)
AIRHART: A + P (chose eventual consistency)
```

AIRHART prioritizes **Availability** and **Partition Tolerance** → accepts **Eventual Consistency**.

## Eventual Consistency in AIRHART

### The Pipeline

```
Adapter → AODB → Kafka → QueryStream → Client

T0: Adapter sends "Flight SK123 Status = Boarding"
T1: AODB receives and processes (+100ms)
T2: AODB writes to database (+50ms)
T3: AODB publishes to Kafka (+10ms)
T4: QueryStream receives event (+50ms)
T5: QueryStream updates document (+100ms)
T6: Client receives SignalR push (+50ms)

Total delay (T0 → T6): ~360ms

During T0-T6: INCONSISTENT
├─ AODB has new data
└─ QueryStream has old data

After T6: CONSISTENT
├─ AODB and QueryStream agree
└─ Client sees latest data
```

### Multi-Component Inconsistency

```
Time: T0 (Initial State)
┌────────────────────────────────────────┐
│ AODB: Flight SK123 Status = "Landed"   │
│ Kafka: (empty)                         │
│ QueryStream: Status = "Landed"         │
│ Client UI: Status = "Landed"           │
└────────────────────────────────────────┘

Time: T1 (After write to AODB)
┌────────────────────────────────────────┐
│ AODB: Status = "At Gate" ✅            │
│ Kafka: (propagating...)                │
│ QueryStream: Status = "Landed" ❌      │
│ Client UI: Status = "Landed" ❌        │
└────────────────────────────────────────┘
  ↑ INCONSISTENT

Time: T2 (Event published to Kafka)
┌────────────────────────────────────────┐
│ AODB: Status = "At Gate" ✅            │
│ Kafka: StatusChanged event ✅          │
│ QueryStream: Status = "Landed" ❌      │
│ Client UI: Status = "Landed" ❌        │
└────────────────────────────────────────┘
  ↑ STILL INCONSISTENT

Time: T3 (QueryStream updated)
┌────────────────────────────────────────┐
│ AODB: Status = "At Gate" ✅            │
│ Kafka: StatusChanged event ✅          │
│ QueryStream: Status = "At Gate" ✅    │
│ Client UI: Status = "Landed" ❌        │
└────────────────────────────────────────┘
  ↑ STILL INCONSISTENT

Time: T4 (Client receives update)
┌────────────────────────────────────────┐
│ AODB: Status = "At Gate" ✅            │
│ Kafka: StatusChanged event ✅          │
│ QueryStream: Status = "At Gate" ✅    │
│ Client UI: Status = "At Gate" ✅      │
└────────────────────────────────────────┘
  ↑ EVENTUALLY CONSISTENT! 🎉
```

## Handling Eventual Consistency

### 1. Accept Temporary Inconsistency

Users understand data isn't always real-time:

```
UI Message:
┌───────────────────────────────────────┐
│ Flight SK123                          │
│ Status: Boarding                      │
│ Last updated: 2 seconds ago           │
│                                       │
│ ℹ️  Data updates every few seconds    │
└───────────────────────────────────────┘
```

### 2. Use Timestamps

Include version/timestamp to detect staleness:

```csharp
public class FlightDTO
{
    public Guid Id { get; set; }
    public string Status { get; set; }
    public DateTime LastUpdated { get; set; } // ← Important!
    public long Version { get; set; } // ← Sequence number
}

// Client can detect out-of-order updates
public void UpdateFlight(FlightDTO update)
{
    var current = _cache.Get(update.Id);
    
    if (current != null && current.Version > update.Version)
    {
        // Discard older update that arrived late
        _logger.LogWarning("Discarding stale update for flight {FlightId}", update.Id);
        return;
    }
    
    _cache.Set(update.Id, update);
    NotifyUI(update);
}
```

### 3. Optimistic UI Updates

Update UI immediately, handle conflicts later:

```javascript
// User clicks "Allocate Stand"
async function allocateStand(flightId, standCode) {
    // 1. Optimistic update (immediate)
    updateUI({
        flightId: flightId,
        stand: standCode,
        status: 'pending' // Show as pending
    });
    
    try {
        // 2. Send command to AODB
        await api.allocateStand(flightId, standCode);
        
        // 3. Mark as confirmed when response received
        updateUI({
            flightId: flightId,
            status: 'confirmed'
        });
    } catch (error) {
        // 4. Revert on failure
        updateUI({
            flightId: flightId,
            stand: null,
            status: 'failed',
            error: error.message
        });
    }
}
```

### 4. Real-Time Push Notifications

Use SignalR to push updates immediately:

```csharp
public class QueryStreamHub : Hub
{
    public async Task SubscribeToFlight(Guid flightId)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, $"flight-{flightId}");
    }
}

// When event processed
public async Task OnFlightUpdated(FlightUpdated @event)
{
    await _hubContext.Clients
        .Group($"flight-{@event.FlightId}")
        .SendAsync("FlightUpdated", @event.ToDTO());
}
```

**Benefit:** Clients receive updates ~500ms after write, instead of polling every 5 seconds.

### 5. Read-Your-Writes Consistency

User should see their own changes immediately:

```csharp
public async Task<FlightDTO> AllocateStand(Guid flightId, string standCode)
{
    // Write to AODB
    await _aodb.AllocateStand(flightId, standCode);
    
    // Don't return stale data from QueryStream!
    // Read directly from AODB for immediate consistency
    var flight = await _aodb.GetFlightLeg(flightId);
    
    return flight.ToDTO();
}
```

Alternative: Include write timestamp in response, client waits for matching update.

### 6. Conflict Resolution

Multiple concurrent updates need resolution:

```csharp
// Scenario: Two operators allocate stands simultaneously

// Operator A: Allocate stand "42A" at T1
// Operator B: Allocate stand "45B" at T2

// Strategy 1: Last-Write-Wins
var winner = updates.OrderByDescending(u => u.Timestamp).First();

// Strategy 2: Prioritize specific users
var winner = updates.OrderBy(u => u.Priority).First();

// Strategy 3: Merge if possible
if (!IsConflicting(updateA, updateB))
{
    return Merge(updateA, updateB);
}

// Strategy 4: Manual resolution
await NotifyOperators("Conflict detected, please resolve manually");
```

## Consistency Boundaries

AIRHART defines consistency boundaries:

### Strong Consistency Within AODB

```
┌─────────────────────────────────┐
│ AODB Component                  │
│                                 │
│ Within a single transaction:    │
│ ✅ Flight update                │
│ ✅ Stand allocation             │
│ ✅ Alert generation             │
│ ✅ Visit linking                │
│                                 │
│ All or nothing (ACID)           │
└─────────────────────────────────┘
```

**Within AODB:** Strong consistency via PostgreSQL transactions

### Eventual Consistency Across Components

```
┌──────────┐    ┌──────────────┐    ┌──────────┐
│   AODB   │ →  │    Kafka     │ →  │QueryStream│
│ (Write)  │    │  (Events)    │    │  (Read)  │
└──────────┘    └──────────────┘    └──────────┘
     ↑                                     ↓
     └─────────────────────────────────────┘
              Eventually Consistent
```

**Across components:** Eventual consistency via events

### External Systems

```
AIRHART → Amadeus: Eventually consistent
AIRHART → Display Systems: Eventually consistent
AIRHART → EDW: Eventually consistent (batch)
```

## Consistency Latency

Typical delays in AIRHART:

| Path | Typical Latency | SLA |
|------|-----------------|-----|
| AODB → Kafka | 10-50ms | <100ms |
| Kafka → QueryStream | 50-200ms | <500ms |
| QueryStream → Client (SignalR) | 50-100ms | <200ms |
| **End-to-End (Write → UI)** | **110-350ms** | **<1s** |

99th percentile: <1 second from write to client seeing update.

## Scenarios

### Scenario 1: Stand Allocation Race

```
Time: T0
Operator A: "Allocate Flight SK123 to Stand 42"
Operator B: "Allocate Flight SK456 to Stand 42"

Time: T1 (Both reach AODB simultaneously)
AODB processes Operator A first (wins)
├─ SK123 → Stand 42 ✅
└─ Publishes StandAllocated event

AODB processes Operator B next
├─ Checks: Stand 42 already allocated?
├─ YES → Reject
└─ Returns error to Operator B

Time: T2
Operator A sees success ✅
Operator B sees error: "Stand 42 already allocated" ❌

Result: Consistent (only one flight per stand)
```

AODB ensures consistency through database constraints.

### Scenario 2: Delayed Update

```
Time: T0
Flight SK123: Stand = "42A"

Time: T1
User updates: Stand = "45B"
├─ AODB: Stand = "45B" (immediately)
└─ QueryStream: Stand = "42A" (not updated yet)

Time: T1 + 200ms
Another user queries QueryStream
└─ Sees Stand = "42A" (STALE DATA ❌)

Time: T1 + 500ms
QueryStream receives event
└─ Stand = "45B" (NOW CONSISTENT ✅)
```

**Handling:**
- Include "Last Updated" timestamp
- Show loading indicator during propagation
- Use SignalR to push update immediately

### Scenario 3: Out-of-Order Events

```
Time: T0
Event 1: Stand = "42A" (sent at T0)
Event 2: Stand = "45B" (sent at T1)

Due to network delays:
Event 2 arrives first
Event 1 arrives second

If applied in arrival order:
└─ Final state: Stand = "42A" ❌ WRONG!

Solution: Use sequence numbers
Event 1 (seq=100): Stand = "42A"
Event 2 (seq=101): Stand = "45B"

Apply in sequence order:
└─ Final state: Stand = "45B" ✅ CORRECT!
```

Kafka preserves order within partition (key = FlightId).

## Monitoring Eventual Consistency

### Lag Metrics

```
consistency.lag.aodb_to_kafka.ms - Time from AODB write to Kafka publish
consistency.lag.kafka_to_querystream.ms - Time from Kafka to QueryStream update
consistency.lag.end_to_end.ms - Total time from write to client

Alert if lag > 5 seconds (indicates system issues)
```

### Consistency Checks

Periodic validation:

```csharp
public async Task ValidateConsistency()
{
    // Sample 100 random flights
    var flights = await _aodb.GetRandomFlights(100);
    
    foreach (var flight in flights)
    {
        var aodbState = await _aodb.Get(flight.Id);
        var queryStreamState = await _queryStream.Get(flight.Id);
        
        if (!AreConsistent(aodbState, queryStreamState))
        {
            _logger.LogWarning(
                "Inconsistency detected for flight {FlightId}. " +
                "AODB: {AODBState}, QueryStream: {QSState}",
                flight.Id, aodbState, queryStreamState
            );
            
            // Trigger manual reconciliation
            await _reconciliation.Fix(flight.Id);
        }
    }
}
```

## Trade-offs

### Benefits ✅

1. **High Availability**
   - System remains available even if some components are down
   - Writes succeed even if QueryStream is temporarily unavailable

2. **Better Performance**
   - Don't wait for synchronous propagation
   - Each component optimized independently
   - Lower latency for writes

3. **Scalability**
   - Components scale independently
   - Geographic distribution possible
   - Handle high throughput

4. **Resilience**
   - Network partitions tolerated
   - Components can work offline and sync later
   - Graceful degradation

### Challenges ❌

1. **Complexity**
   - Harder to reason about system state
   - Must handle conflicts and races
   - More complex testing

2. **User Experience**
   - Stale data shown temporarily
   - Loading states and spinners
   - Potential confusion ("Why isn't my update showing?")

3. **Debugging**
   - Harder to trace issues across async boundaries
   - Timing-dependent bugs
   - Need good logging and tracing

4. **Data Integrity**
   - Must design for eventual consistency
   - Handle duplicates, out-of-order events
   - Conflict resolution strategies needed

## Best Practices

### 1. Design for Idempotency

Operations should be safe to retry:

```csharp
// ❌ Bad - not idempotent
public async Task AllocateStand(Guid flightId, string standCode)
{
    flight.Stand = standCode;
    flight.AllocationCount++; // Side effect!
}

// ✅ Good - idempotent
public async Task AllocateStand(Guid flightId, string standCode, Guid requestId)
{
    if (await _processedRequests.Contains(requestId))
    {
        return; // Already processed
    }
    
    flight.Stand = standCode;
    await _processedRequests.Add(requestId);
}
```

### 2. Use Correlation IDs

Track requests across components:

```csharp
var correlationId = Guid.NewGuid();

// AODB
_logger.LogInformation("Processing request {CorrelationId}", correlationId);
await _aodb.Process(message, correlationId);

// Kafka
await _kafka.Publish(new Event
{
    CorrelationId = correlationId,
    ...
});

// QueryStream
_logger.LogInformation("Received event {CorrelationId}", correlationId);
await _queryStream.Process(@event);

// End-to-end tracing possible!
```

### 3. Version All Data

Use optimistic locking:

```csharp
public class FlightLeg
{
    public long Version { get; set; }
}

public async Task Update(FlightLeg flight)
{
    var current = await _db.Get(flight.Id);
    
    if (current.Version != flight.Version)
    {
        throw new ConcurrencyException(
            "Flight was modified by another user. " +
            "Please refresh and try again."
        );
    }
    
    flight.Version++;
    await _db.Save(flight);
}
```

### 4. Provide Feedback

Keep users informed:

```
┌─────────────────────────────────┐
│ ⏳ Updating flight...           │
│ ✅ Update saved to AODB          │
│ 🔄 Refreshing display...        │
│ ✅ Done!                         │
└─────────────────────────────────┘
```

## Related Patterns

- **[[CQRS Pattern]]** - Separates reads and writes, accepts eventual consistency between models
- **[[Event Sourcing]]** - Events propagate asynchronously, causing eventual consistency
- **Saga Pattern** - Coordinates distributed transactions with eventual consistency
- **Event-Driven Architecture** - Async events lead to eventual consistency

---

**Tags:** #design-pattern #consistency #distributed-systems #architecture
**Created:** 2025-12-15
**Related:** [[CQRS Pattern]], [[Event Sourcing]]
**References:**
- Werner Vogels: [Eventually Consistent](https://www.allthingsdistributed.com/2008/12/eventually_consistent.html)
- Consistency Models: [CAP Theorem](https://en.wikipedia.org/wiki/CAP_theorem)
