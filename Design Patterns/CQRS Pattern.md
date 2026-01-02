# CQRS Pattern (Command Query Responsibility Segregation)

> Separating read and write operations for scalability and performance

## Overview

**CQRS (Command Query Responsibility Segregation)** is an architectural pattern that separates read operations (queries) from write operations (commands) by using different models for updating and reading data.

## Core Principle

Traditional CRUD systems use the same model for both reads and writes:

```
       ┌─────────────┐
Read ─→│  Unified    │←─ Write
       │  Data Model │
       └─────────────┘
```

CQRS separates these concerns:

```
       ┌─────────────┐
Write→ │  Command    │ ─→ Write DB
       │   Model     │
       └─────────────┘

       ┌─────────────┐
Read ← │   Query     │ ←─ Read DB
       │   Model     │
       └─────────────┘
```

## Key Concepts

### Commands (Write Side)

**Commands** represent intentions to change state:

```csharp
// Command represents an action
public class UpdateFlightStatus : ICommand
{
    public Guid FlightId { get; set; }
    public string NewStatus { get; set; }
    public DateTime Timestamp { get; set; }
}

// Command Handler processes the action
public class UpdateFlightStatusHandler : ICommandHandler<UpdateFlightStatus>
{
    private readonly IAODBRepository _repository;
    private readonly IEventBus _eventBus;

    public async Task Handle(UpdateFlightStatus command)
    {
        // Load aggregate
        var flight = await _repository.GetFlightLeg(command.FlightId);
        
        // Business logic
        flight.UpdateStatus(command.NewStatus, command.Timestamp);
        
        // Persist changes
        await _repository.Save(flight);
        
        // Publish events
        await _eventBus.Publish(new FlightStatusChanged
        {
            FlightId = flight.Id,
            NewStatus = command.NewStatus,
            Timestamp = command.Timestamp
        });
    }
}
```

**Characteristics:**
- **Task-based**: `UpdateFlightStatus`, not generic "Update"
- **Validates business rules**: Ensures data consistency
- **Returns void or status**: Does not return domain data
- **Transactional**: All-or-nothing operations
- **Asynchronous**: Can be queued for later processing

### Queries (Read Side)

**Queries** retrieve data without modifying state:

```csharp
// Query represents a request for information
public class GetFlightsByDate : IQuery<List<FlightDTO>>
{
    public DateTime Date { get; set; }
    public string AirportCode { get; set; }
}

// Query Handler returns data
public class GetFlightsByDateHandler : IQueryHandler<GetFlightsByDate, List<FlightDTO>>
{
    private readonly IQueryStreamClient _queryStream;

    public async Task<List<FlightDTO>> Handle(GetFlightsByDate query)
    {
        // Direct database query - no business logic
        return await _queryStream.GetFlights(
            date: query.Date,
            airport: query.AirportCode
        );
    }
}
```

**Characteristics:**
- **Read-only**: Never modifies state
- **Optimized for display**: Denormalized, pre-computed data
- **Fast**: Uses indexes, caching, materialized views
- **Eventually consistent**: May not reflect latest writes immediately
- **Idempotent**: Same query always returns same result (at that moment)

## CQRS in AIRHART

### Write Side: AODB Component

**AODB** handles all write operations:

```
Adapter Messages (Commands)
         ↓
    AODB Pipeline
         ↓
  ┌────────────────┐
  │ 1. Validation  │ ← Business rules
  │ 2. Matching    │ ← Identity resolution
  │ 3. Enrichment  │ ← Data enhancement
  │ 4. Linking     │ ← Relationship management
  │ 5. Storage     │ ← PostgreSQL (Write DB)
  │ 6. Publishing  │ ← Event publication
  └────────────────┘
         ↓
   Output Topics (Events)
```

**Command Model:**
- Complex business logic (enrichment rules)
- Normalized storage (relational DB)
- Strong consistency within transactions
- Entity Framework for ORM

**Write Operations:**
- Create/update/delete FlightLeg
- Link flights into visits
- Apply business rules
- Validate data integrity

### Read Side: QueryStream Component

**QueryStream** handles all read operations:

```
AODB Output Topics (Events)
         ↓
   QueryStream Feed Compilers
         ↓
  ┌────────────────────┐
  │ 1. Receive Events  │
  │ 2. Transform to DTO│ ← Flatten structure
  │ 3. Compute Fields  │ ← Calculate display fields
  │ 4. Store Document  │ ← Document DB (Read DB)
  │ 5. Index for Query │ ← Full-text search
  │ 6. Cache in Memory │ ← Hot data
  └────────────────────┘
         ↓
    SignalR / OData API
         ↓
      Clients
```

**Query Model:**
- Denormalized documents (DTOs)
- Optimized for specific queries
- Eventually consistent
- Caching at multiple levels

**Read Operations:**
- Get flights by date
- Subscribe to real-time updates
- Filter/sort/search
- No business logic - just data retrieval

## Benefits

### 1. Independent Scaling

Scale read and write sides independently:

```
Write Side (AODB):
├─ 3 instances
├─ CPU-intensive (business rules)
└─ High memory (caching Master Data)

Read Side (QueryStream):
├─ 10 instances
├─ Network-intensive (many clients)
└─ High throughput (read queries)
```

In AIRHART, read traffic far exceeds write traffic (100:1 ratio), so QueryStream can scale horizontally while AODB remains smaller.

### 2. Optimized Data Models

**Write Model (AODB):**
```sql
-- Normalized for consistency
FlightLeg (entity_id, airline_id, aircraft_id, ...)
Airline (id, iata_code, name, ...)
Aircraft (id, registration, type_id, ...)
AircraftType (id, icao_code, manufacturer, ...)
```

**Read Model (QueryStream):**
```json
// Denormalized for performance
{
  "flightId": "abc-123",
  "airlineIataCode": "SK",
  "airlineName": "SAS Scandinavian Airlines",
  "aircraftRegistration": "OY-KBH",
  "aircraftTypeIcaoCode": "A320",
  "aircraftManufacturer": "Airbus",
  // ... all data flattened for display
}
```

No joins needed in read model → faster queries!

### 3. Simpler Queries

**Without CQRS (complex joins):**
```sql
SELECT f.*, a.name AS airline_name, ac.registration, at.icao_code
FROM flight_leg f
JOIN airline a ON f.airline_id = a.id
JOIN aircraft ac ON f.aircraft_id = ac.id
JOIN aircraft_type at ON ac.type_id = at.id
WHERE f.scheduled_departure BETWEEN @start AND @end
ORDER BY f.scheduled_departure;
```

**With CQRS (simple query):**
```javascript
// Query pre-built document
db.flights.find({
  scheduledDeparture: {
    $gte: startDate,
    $lte: endDate
  }
}).sort({ scheduledDeparture: 1 })
```

### 4. Performance

- **Reads:** Direct access to optimized documents, no complex joins
- **Writes:** No read query optimization concerns, focus on business logic
- **Caching:** Easy to cache read model without affecting write consistency
- **Indexing:** Create indexes specifically for query patterns

### 5. Technology Choice

Different technologies for different needs:

**Write Side:**
- PostgreSQL (ACID transactions, strong consistency)
- Entity Framework (ORM)
- Business rule engine

**Read Side:**
- Document DB (flexible schema)
- Redis (caching)
- Elasticsearch (full-text search)

## Challenges

### 1. Eventual Consistency

Read model may lag behind write model:

```
T0: Write "Flight SK123 status = Boarding"
T1: AODB processes and publishes event
T2: QueryStream receives event
T3: Document updated in read database
T4: Client queries and sees update

Delay (T0 → T4): typically < 1 second in AIRHART
```

**Mitigation in AIRHART:**
- Accept eventual consistency for most use cases
- Use SignalR for real-time push to avoid polling
- Include timestamp in responses for client-side ordering
- Provide "last updated" indicator in UI

### 2. Complexity

More components to maintain:

- Write side: AODB with business logic
- Read side: QueryStream with denormalization
- Synchronization: Event bus between sides
- Consistency: Eventually consistent, not immediately

**Trade-off:** Increased complexity for better scalability and performance

### 3. Data Duplication

Same data exists in multiple places:

```
FlightLeg in AODB (PostgreSQL)
     ↓
FlightLeg event on Kafka
     ↓
Flight document in QueryStream (Document DB)
     ↓
Flight in UI cache (Redis)
```

**Cost:** Storage and synchronization overhead
**Benefit:** Each optimized for its purpose

## Event-Driven Synchronization

CQRS in AIRHART uses events to synchronize:

```csharp
// Write side publishes events
public class FlightLegUpdated : IDomainEvent
{
    public Guid FlightId { get; set; }
    public string Status { get; set; }
    public string Stand { get; set; }
    public DateTime UpdatedAt { get; set; }
}

// Read side subscribes to events
public class FlightLegUpdatedHandler : IEventHandler<FlightLegUpdated>
{
    private readonly IDocumentStore _documentStore;

    public async Task Handle(FlightLegUpdated @event)
    {
        // Update read model
        var flightDoc = await _documentStore.Get(@event.FlightId);
        flightDoc.Status = @event.Status;
        flightDoc.Stand = @event.Stand;
        flightDoc.LastUpdated = @event.UpdatedAt;
        
        await _documentStore.Save(flightDoc);
        
        // Notify clients via SignalR
        await _hubContext.Clients.All.SendAsync(
            "FlightUpdated", 
            flightDoc
        );
    }
}
```

See [[Event Sourcing]] for more on event-driven architecture.

## When to Use CQRS

### Good Fit ✅

- **High read-to-write ratio** (like AIRHART: 100:1)
- **Complex business logic on writes**
- **Simple, fast reads required**
- **Different scaling needs** for reads vs writes
- **Multiple read representations** of same data

### Poor Fit ❌

- **Simple CRUD applications**
- **Low traffic** where scaling isn't needed
- **Strong consistency required** everywhere
- **Team unfamiliar** with pattern (learning curve)
- **Small applications** where complexity > benefit

## CQRS Variants

### 1. Simple CQRS (AIRHART uses this)

Separate models, shared database:

```
Commands → Write Model → Database ← Read Model ← Queries
```

- Same storage technology
- Easier to maintain consistency
- Still benefits from separated concerns

### 2. CQRS with Event Sourcing

Write side stores events, read side projects from events:

```
Commands → Events → Event Store
                         ↓
                    Projections → Read Models
```

- Complete audit trail
- Time-travel queries
- More complex implementation

### 3. CQRS with Separate Databases

Completely independent storage:

```
Commands → Write DB (PostgreSQL)
                ↓ events
           Read DB (MongoDB)
```

- Maximum independence
- Technology choice per side
- More complex synchronization

## Implementation Guidelines

### 1. Define Clear Boundaries

```csharp
// Commands - clear, intention-revealing names
public interface ICommand { }

public class AllocateStandToFlight : ICommand
{
    public Guid FlightId { get; set; }
    public string StandCode { get; set; }
}

// Queries - describe what data is requested
public interface IQuery<TResult> { }

public class GetFlightsForDashboard : IQuery<DashboardFlightsDTO>
{
    public DateTime Date { get; set; }
}
```

### 2. Keep Queries Simple

Queries should have NO business logic:

```csharp
// ❌ Bad - business logic in query
public async Task<List<FlightDTO>> GetFlights(DateTime date)
{
    var flights = await _db.Flights.Where(f => f.Date == date).ToList();
    
    // Business logic doesn't belong here!
    foreach (var flight in flights)
    {
        if (flight.Stand == null && flight.Status == "Boarding")
        {
            flight.Alert = "No stand allocated";
        }
    }
    
    return flights;
}

// ✅ Good - pure data retrieval
public async Task<List<FlightDTO>> GetFlights(DateTime date)
{
    return await _db.Flights
        .Where(f => f.Date == date)
        .ToList();
}
```

Business logic belongs in write side (AODB enrichment rules).

### 3. Use DTOs for Queries

Never expose domain entities in queries:

```csharp
// Domain entity (write model)
public class FlightLeg
{
    public Guid EntityId { get; private set; }
    private List<Alert> _alerts;
    
    public void AllocateStand(string standCode)
    {
        // Complex business logic
    }
}

// DTO (read model)
public class FlightDTO
{
    public Guid Id { get; set; }
    public string FlightNumber { get; set; }
    public string Status { get; set; }
    // Flattened, display-ready fields
}
```

### 4. Version Your Read Models

Support multiple client versions:

```csharp
public class FlightDTOV1
{
    public string Status { get; set; }
}

public class FlightDTOV2
{
    public string DetailedStatus { get; set; }
    public string StatusReason { get; set; }
}

// API versioning
[Route("api/v1/flights")]
public async Task<List<FlightDTOV1>> GetFlightsV1() { ... }

[Route("api/v2/flights")]
public async Task<List<FlightDTOV2>> GetFlightsV2() { ... }
```

## Related Patterns

- **[[Event Sourcing]]** - Often combined with CQRS, stores all changes as events
- **[[Eventual Consistency]]** - Read model becomes consistent over time
- **Event-Driven Architecture** - Uses events to synchronize read and write sides
- **Domain-Driven Design** - CQRS fits well with DDD aggregates and bounded contexts

## Real-World Example in AIRHART

### Write Operation: Update Flight Stand

```
1. Client → AODB API: POST /api/flights/{id}/allocate-stand
   {
     "standCode": "42A",
     "timestamp": "2025-12-15T10:30:00Z"
   }

2. AODB → Command Handler:
   - Validate stand availability
   - Apply business rules (aircraft size vs stand capacity)
   - Update FlightLeg aggregate
   - Save to PostgreSQL (commit transaction)
   - Publish "FlightLegUpdated" event to Kafka

3. Kafka → QueryStream:
   - Receive event
   - Transform to FlightDTO
   - Update document in Document DB
   - Notify subscribed clients via SignalR
   
4. Client receives update via SignalR push
   Total latency: ~500ms
```

### Read Operation: Query Today's Flights

```
1. Client → QueryStream API: GET /api/flights?date=2025-12-15

2. QueryStream → Query Handler:
   - Parse OData query
   - Fetch from optimized read database (Document DB)
   - Apply filters, sorting (all indexes available)
   - Return FlightDTO[]
   
3. Client receives data
   Latency: ~50ms (no joins, cached)
```

**10x faster reads** through CQRS!

---

**Tags:** #design-pattern #architecture #cqrs #scalability
**Created:** 2025-12-15
**Related:** [[Event Sourcing]], [[Eventual Consistency]]
**References:**
- Martin Fowler: [CQRS](https://martinfowler.com/bliki/CQRS.html)
- Microsoft: [CQRS Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/cqrs)
