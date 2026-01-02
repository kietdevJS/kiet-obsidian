# Event Sourcing

> Storing all changes as a sequence of events rather than current state

## Overview

**Event Sourcing** is a pattern where state changes are stored as a sequence of events. Instead of storing the current state of an entity, you store all the events that led to that state.

## Core Concept

### Traditional State Storage

```
Database stores CURRENT state:
┌────────────────────────────┐
│ FlightLeg ID: 123          │
│ Status: "Boarding"         │
│ Stand: "42A"               │
│ Gate: "B12"                │
└────────────────────────────┘
❌ History lost - only current state
```

### Event Sourcing

```
Event Store contains ALL events:
┌────────────────────────────────────────┐
│ 1. FlightLegCreated                    │
│    { id: 123, flightNumber: "SK1234" } │
│                                        │
│ 2. StandAllocated                      │
│    { stand: "42A", time: T1 }          │
│                                        │
│ 3. GateAssigned                        │
│    { gate: "B12", time: T2 }           │
│                                        │
│ 4. StatusChanged                       │
│    { status: "Boarding", time: T3 }    │
└────────────────────────────────────────┘
✅ Complete history preserved
```

**Current state = replay all events**

## Key Concepts

### Events Are Immutable

Once written, events NEVER change:

```csharp
// ❌ NEVER modify existing events
event.Status = "Cancelled"; // Forbidden!

// ✅ Add new compensating event
var cancellationEvent = new FlightCancelled
{
    FlightId = flightId,
    Reason = "Weather",
    Timestamp = DateTime.UtcNow
};

eventStore.Append(cancellationEvent);
```

### Events Are Facts

Events describe **what happened**, not **what to do**:

```csharp
// ❌ Bad - imperative command
public class UpdateStand : Event
{
    public string NewStand { get; set; }
}

// ✅ Good - past-tense fact
public class StandChanged : Event
{
    public string PreviousStand { get; set; }
    public string NewStand { get; set; }
    public DateTime ChangedAt { get; set; }
    public string ChangedBy { get; set; }
    public string Reason { get; set; }
}
```

### Rebuilding State

Current state reconstructed by replaying events:

```csharp
public class FlightLeg
{
    public Guid Id { get; private set; }
    public string Status { get; private set; }
    public string Stand { get; private set; }
    
    // Rebuild state from events
    public static FlightLeg FromEvents(IEnumerable<DomainEvent> events)
    {
        var flight = new FlightLeg();
        
        foreach (var @event in events.OrderBy(e => e.Timestamp))
        {
            flight.Apply(@event);
        }
        
        return flight;
    }
    
    // Apply individual event
    private void Apply(DomainEvent @event)
    {
        switch (@event)
        {
            case FlightLegCreated created:
                Id = created.FlightId;
                Status = "Scheduled";
                break;
                
            case StandAllocated allocated:
                Stand = allocated.StandCode;
                break;
                
            case StatusChanged statusChanged:
                Status = statusChanged.NewStatus;
                break;
                
            case FlightCancelled cancelled:
                Status = "Cancelled";
                break;
        }
    }
}
```

## Event Store

### Structure

```
┌─────────────────────────────────────────────────┐
│ Event Stream: FlightLeg-123                     │
├─────────────────────────────────────────────────┤
│ Seq | Event Type        | Timestamp  | Data     │
├─────┼───────────────────┼────────────┼──────────┤
│  1  │ FlightLegCreated  │ 10:00:00   │ {...}    │
│  2  │ StandAllocated    │ 10:05:00   │ {...}    │
│  3  │ StatusChanged     │ 10:30:00   │ {...}    │
│  4  │ GateAssigned      │ 10:35:00   │ {...}    │
└─────┴───────────────────┴────────────┴──────────┘

Each entity has its own stream
Events ordered by sequence number
Append-only - never update/delete
```

### Implementation

```csharp
public interface IEventStore
{
    // Append new events
    Task AppendEvents(string streamId, IEnumerable<DomainEvent> events);
    
    // Read all events for an entity
    Task<IEnumerable<DomainEvent>> GetEvents(string streamId);
    
    // Read events after specific sequence
    Task<IEnumerable<DomainEvent>> GetEventsSince(string streamId, int sequenceNumber);
}

public class KafkaEventStore : IEventStore
{
    private readonly IProducer<string, DomainEvent> _producer;
    private readonly IConsumer<string, DomainEvent> _consumer;
    
    public async Task AppendEvents(string streamId, IEnumerable<DomainEvent> events)
    {
        foreach (var @event in events)
        {
            await _producer.ProduceAsync(
                topic: "event-store",
                message: new Message<string, DomainEvent>
                {
                    Key = streamId, // Ensures ordering within stream
                    Value = @event
                }
            );
        }
    }
    
    public async Task<IEnumerable<DomainEvent>> GetEvents(string streamId)
    {
        var events = new List<DomainEvent>();
        
        // Read from Kafka topic, filtering by key
        _consumer.Subscribe("event-store");
        
        while (true)
        {
            var result = _consumer.Consume(TimeSpan.FromSeconds(1));
            
            if (result == null) break;
            if (result.Message.Key == streamId)
            {
                events.Add(result.Message.Value);
            }
        }
        
        return events.OrderBy(e => e.Sequence);
    }
}
```

## Event Sourcing in AIRHART

### Kafka as Event Store

AIRHART uses **Kafka** as an event store:

```
AODB Pipeline publishes events:
└─ kafka topic: aodb-output-flightleg
   ├─ Key: FlightLeg ID (ensures ordering)
   ├─ Value: FlightLegUpdated event
   └─ Retention: 30 days (configurable)

Benefits:
✅ Append-only log structure (perfect for events)
✅ Partitioned by key (each flight has ordered events)
✅ Distributed and fault-tolerant
✅ Multiple consumers can replay events
```

### Event Publishing

```csharp
public class AODBPipeline
{
    private readonly IKafkaProducer _producer;
    
    public async Task ProcessMessage(AdapterMessage message)
    {
        // 1. Pipeline processing (validation, enrichment, etc.)
        var flightLeg = await _pipeline.Process(message);
        
        // 2. Save to database (current state)
        await _repository.Save(flightLeg);
        
        // 3. Publish event (event sourcing)
        var @event = new FlightLegUpdated
        {
            EventId = Guid.NewGuid(),
            FlightId = flightLeg.EntityId,
            EventType = "FlightLegUpdated",
            Timestamp = DateTime.UtcNow,
            Sequence = await GetNextSequence(flightLeg.EntityId),
            
            // Payload - all changed fields
            Changes = new
            {
                Status = flightLeg.Status,
                Stand = flightLeg.StandCode,
                ActualTimeOfDeparture = flightLeg.ActualTimeOfDeparture,
                // ... all updated fields
            }
        };
        
        await _producer.PublishEvent("aodb-output-flightleg", @event);
    }
}
```

### Event Replay

QueryStream rebuilds read models from events:

```csharp
public class QueryStreamProjection
{
    private readonly IDocumentStore _documentStore;
    
    // Subscribe to event stream
    public async Task ProjectEvents()
    {
        await foreach (var @event in _eventStream.Subscribe("aodb-output-flightleg"))
        {
            await ProjectEvent(@event);
        }
    }
    
    private async Task ProjectEvent(FlightLegUpdated @event)
    {
        // Get or create document
        var document = await _documentStore.Get(@event.FlightId)
                    ?? new FlightDocument { Id = @event.FlightId };
        
        // Apply event changes
        document.Status = @event.Changes.Status;
        document.Stand = @event.Changes.Stand;
        document.LastUpdated = @event.Timestamp;
        
        // Save updated read model
        await _documentStore.Save(document);
    }
}
```

## Benefits

### 1. Complete Audit Trail

Every change is recorded:

```
Who changed flight status?
When was stand allocated?
Why was gate reassigned?

Query event store:
├─ 10:30:00 | StatusChanged | by: john@airport.com | reason: "Early arrival"
├─ 10:35:00 | StandAllocated | by: system | reason: "Auto-allocation"
└─ 10:40:00 | GateReassigned | by: ops@airport.com | reason: "Stand conflict"
```

**Use Cases:**
- Compliance and regulations
- Debugging production issues
- User activity tracking
- Performance analysis

### 2. Time Travel

Reconstruct state at any point in time:

```csharp
public async Task<FlightLeg> GetFlightLegAt(Guid flightId, DateTime pointInTime)
{
    var events = await _eventStore.GetEvents($"FlightLeg-{flightId}");
    
    // Only replay events up to specific time
    var relevantEvents = events.Where(e => e.Timestamp <= pointInTime);
    
    return FlightLeg.FromEvents(relevantEvents);
}

// What was flight status at 10:15?
var flight = await GetFlightLegAt(flightId, DateTime.Parse("2025-12-15T10:15:00Z"));
Console.WriteLine(flight.Status); // "Scheduled"

// What about at 10:45?
flight = await GetFlightLegAt(flightId, DateTime.Parse("2025-12-15T10:45:00Z"));
Console.WriteLine(flight.Status); // "Boarding"
```

**Use Cases:**
- Historical reporting: "How many delays did we have in Q3?"
- Incident investigation: "What was the system state when the error occurred?"
- Regulatory inquiries: "Show us all flight updates on July 4th"

### 3. Event Replay

Rebuild read models from scratch:

```csharp
// QueryStream crashed and lost data
// No problem - replay all events!

public async Task RebuildReadModel()
{
    // Clear current read model
    await _documentStore.Clear();
    
    // Replay all events from Kafka
    var events = await _eventStore.GetAllEvents("aodb-output-flightleg");
    
    foreach (var @event in events.OrderBy(e => e.Timestamp))
    {
        await ProjectEvent(@event);
    }
    
    Console.WriteLine("Read model rebuilt from events!");
}
```

**Use Cases:**
- Disaster recovery
- New read model creation (add new projection)
- Bug fixes (replay with corrected logic)
- Testing (replay production events in staging)

### 4. Multiple Read Models

Create different projections from same events:

```
              Event Stream
                   ↓
        ┌──────────┼──────────┐
        ↓          ↓          ↓
  Dashboard    Reporting   Analytics
   Projection   Projection  Projection
```

```csharp
// Same events, different projections

// Projection 1: Real-time dashboard (denormalized)
public class DashboardProjection
{
    public async Task Project(FlightLegUpdated @event)
    {
        var doc = new DashboardFlightDTO
        {
            FlightNumber = @event.FlightNumber,
            Status = @event.Status,
            Stand = @event.Stand,
            // Optimized for UI display
        };
        await _cache.Set(@event.FlightId, doc);
    }
}

// Projection 2: Analytics (aggregated)
public class AnalyticsProjection
{
    public async Task Project(FlightLegUpdated @event)
    {
        await _analyticsDb.IncrementCounter($"status.{@event.Status}");
        await _analyticsDb.RecordDelay(@event.FlightId, @event.Delay);
        // Optimized for reporting
    }
}

// Projection 3: Audit log (complete history)
public class AuditProjection
{
    public async Task Project(FlightLegUpdated @event)
    {
        await _auditLog.Append(new AuditEntry
        {
            EventId = @event.EventId,
            Timestamp = @event.Timestamp,
            User = @event.UserId,
            Changes = @event.Changes
        });
    }
}
```

### 5. Debugging & Analysis

```csharp
// Find all events related to incident

var events = await _eventStore.GetEvents("FlightLeg-123");

Console.WriteLine("Event Timeline:");
foreach (var @event in events)
{
    Console.WriteLine($"[{@event.Timestamp:HH:mm:ss}] {@event.Type}");
    Console.WriteLine($"  User: {@event.UserId}");
    Console.WriteLine($"  Changes: {@event.Changes}");
    Console.WriteLine();
}

/*
Output:
[10:00:00] FlightLegCreated
  User: system
  Changes: { flightNumber: "SK1234", ... }

[10:05:00] StandAllocated
  User: auto-allocation-service
  Changes: { stand: "42A" }

[10:30:00] StandChanged
  User: john@airport.com
  Changes: { stand: "45B", reason: "Aircraft too large" }
  ← Found it! Manual override at 10:30
*/
```

## Challenges

### 1. Complexity

More complex than traditional CRUD:

```
Traditional:
├─ UPDATE flight SET status = 'Boarding'
└─ Done!

Event Sourcing:
├─ Create StatusChanged event
├─ Append to event store
├─ Publish event to consumers
├─ Consumers rebuild read models
└─ Eventually consistent
```

**Mitigation:**
- Use frameworks (e.g., Axon, EventStore)
- Clear event naming conventions
- Good documentation
- Team training

### 2. Event Schema Evolution

Events must remain compatible:

```csharp
// Version 1
public class StandAllocated
{
    public string Stand { get; set; }
}

// Version 2 - added fields
public class StandAllocated
{
    public string Stand { get; set; }
    public DateTime AllocatedAt { get; set; } // New field
    public string AllocatedBy { get; set; }   // New field
}

// ❌ Problem: Old events missing new fields!

// ✅ Solution: Handle missing fields gracefully
public class StandAllocated
{
    public string Stand { get; set; }
    public DateTime? AllocatedAt { get; set; } // Nullable
    public string AllocatedBy { get; set; } = "system"; // Default value
}
```

**Best Practices:**
- Only add fields, never remove
- Use nullable types for new fields
- Provide default values
- Version event types if major breaking change

### 3. Storage Requirements

Storing ALL events uses more space:

```
Traditional database:
├─ 1 row per flight (1KB)
└─ 10,000 flights = 10MB

Event store:
├─ 10 events per flight (10KB)
└─ 10,000 flights = 100MB
```

**Mitigation:**
- Kafka topic retention (e.g., 30 days)
- Snapshots (store current state + recent events)
- Archive old events to cheap storage
- Compress event payloads

### 4. Querying Events

Can't query event store like a database:

```sql
-- ❌ Can't do this on event store
SELECT * FROM flights WHERE status = 'Delayed';
```

**Solution:** Build read models (projections) for queries
- AODB stores current state in PostgreSQL
- QueryStream maintains queryable documents
- Event store is source of truth, not query engine

## Snapshots

To avoid replaying millions of events:

```
Snapshot = Current state at point in time

┌─────────────────────────────────┐
│ Snapshot (sequence 1000)        │
│ { status: "Boarding", ... }     │
└─────────────────────────────────┘
         ↓ only replay recent
┌─────────────────────────────────┐
│ Event 1001: StandChanged        │
│ Event 1002: StatusChanged       │
│ Event 1003: GateAssigned        │
└─────────────────────────────────┘

Load = Snapshot + Events since snapshot
```

```csharp
public async Task<FlightLeg> LoadFlightLeg(Guid flightId)
{
    // Try to load latest snapshot
    var snapshot = await _snapshotStore.GetLatest(flightId);
    var flight = snapshot?.ToEntity() ?? new FlightLeg();
    
    // Replay events since snapshot
    var sequenceSince = snapshot?.SequenceNumber ?? 0;
    var recentEvents = await _eventStore.GetEventsSince(flightId, sequenceSince);
    
    foreach (var @event in recentEvents)
    {
        flight.Apply(@event);
    }
    
    // Periodically create new snapshot
    if (recentEvents.Count() > 100)
    {
        await _snapshotStore.Save(new Snapshot
        {
            EntityId = flightId,
            SequenceNumber = recentEvents.Last().Sequence,
            State = flight
        });
    }
    
    return flight;
}
```

## Event Sourcing vs Traditional Storage

| Aspect | Traditional | Event Sourcing |
|--------|-------------|----------------|
| **Storage** | Current state only | All events |
| **History** | Lost (unless audited separately) | Complete |
| **Queries** | Fast (indexed) | Requires projections |
| **Writes** | Update in place | Append only |
| **Debugging** | Difficult | Full audit trail |
| **Complexity** | Simple | More complex |
| **Time Travel** | Impossible | Easy |
| **Recovery** | From backups | Replay events |

## AIRHART's Hybrid Approach

AIRHART uses **both** approaches:

```
┌─────────────────────────────────────┐
│ AODB (Traditional Storage)          │
│ - PostgreSQL with current state     │
│ - Fast queries                      │
│ - Business logic                    │
└─────────────────────────────────────┘
              ↓ publishes events
┌─────────────────────────────────────┐
│ Kafka Event Store                   │
│ - Complete event history            │
│ - 30-day retention                  │
│ - Multiple consumers                │
└─────────────────────────────────────┘
              ↓ consumes events
┌─────────────────────────────────────┐
│ QueryStream (Projection)            │
│ - Document DB with read model       │
│ - Optimized for queries             │
│ - Rebuilt from events if needed     │
└─────────────────────────────────────┘
```

**Benefits:**
- ✅ Current state readily available (AODB PostgreSQL)
- ✅ Complete event history (Kafka)
- ✅ Optimized read models (QueryStream)
- ✅ Audit trail and replay capability
- ✅ [[CQRS Pattern]] implementation

## Best Practices

### 1. Name Events in Past Tense

```csharp
// ✅ Good
FlightCreated
StandAllocated
StatusChanged
FlightCancelled

// ❌ Bad
CreateFlight
AllocateStand
ChangeStatus
CancelFlight
```

### 2. Include Context

```csharp
public class StandAllocated : DomainEvent
{
    public Guid FlightId { get; set; }
    public string StandCode { get; set; }
    
    // Context
    public string AllocatedBy { get; set; } // Who
    public DateTime AllocatedAt { get; set; } // When
    public string Reason { get; set; } // Why
    public string Source { get; set; } // How (manual, automatic, etc.)
}
```

### 3. Keep Events Small

```csharp
// ❌ Bad - huge payload
public class FlightUpdated
{
    public FlightLeg EntireFlightObject { get; set; } // 50KB
}

// ✅ Good - only changed data
public class StatusChanged
{
    public Guid FlightId { get; set; }
    public string OldStatus { get; set; }
    public string NewStatus { get; set; }
} // 200 bytes
```

### 4. One Event Type per Change

```csharp
// ❌ Bad - multiple changes in one event
public class FlightUpdated
{
    public string NewStatus { get; set; }
    public string NewStand { get; set; }
    public string NewGate { get; set; }
}

// ✅ Good - specific events
public class StatusChanged { ... }
public class StandAllocated { ... }
public class GateAssigned { ... }
```

## Related Patterns

- **[[CQRS Pattern]]** - Often used together, CQRS handles reads, Event Sourcing handles writes
- **[[Eventual Consistency]]** - Projections become consistent after events are processed
- **Event-Driven Architecture** - Events drive system behavior
- **Domain-Driven Design** - Aggregates produce domain events

---

**Tags:** #design-pattern #event-sourcing #architecture #audit
**Created:** 2025-12-15
**Related:** [[CQRS Pattern]], [[Eventual Consistency]]
**References:**
- Martin Fowler: [Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)
- Microsoft: [Event Sourcing Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/event-sourcing)
