# Master Data Component

> Reference data management with bitemporal storage and flexible schema

## Overview

The **Master Data Component** is a flexible data management system that maintains reference data (airports, airlines, aircraft, etc.) used throughout AIRHART. It provides both current and historical views of data through bitemporal storage, supporting time-based queries and future-dated changes.

## Core Concepts

### Bitemporal Storage

Master Data uses **two time dimensions** for tracking changes:

```
┌─────────────────────────────────────────┐
│ ValidFrom / ValidTo                      │  ← Business Time
│  (When the data is effective)            │
├──────────────────────────────────────────┤
│ RegistrationFrom / RegistrationTo        │  ← System Time
│  (When we knew about the change)         │
└──────────────────────────────────────────┘
```

**Business Time (Valid Time)**
- `ValidFrom`: When the data becomes operationally effective
- `ValidTo`: When the data ceases to be operationally effective
- Used for operational queries: "What stands are available on Dec 25?"

**System Time (Registration Time)**
- `RegistrationFrom`: When the record was inserted/updated in the system
- `RegistrationTo`: When the record was superseded by a new version
- Used for auditing: "What did we know on Nov 15?"

### Key Features

1. **Future-Dated Changes**
   - Schedule changes in advance (e.g., "Stand 42 will be unavailable starting next month")
   - System automatically applies changes at the effective time
   - No manual intervention required

2. **Historical Tracking**
   - Complete audit trail of all changes
   - Query historical state: "What was the aircraft type on July 4?"
   - Regulatory compliance and analysis

3. **Flexible Schema**
   - Dynamic entity types (can add custom Master Data entities)
   - Configurable attributes
   - JSON-based schema definition

4. **Delta Publishing**
   - Publishes only changed data to `masterdata-update-delta` topic
   - Includes time ranges and affected attributes
   - Triggers AODB recalculation for affected periods

## Entity Types

### Standard Master Data Entities

**Airport Entities:**
- **Airport** - Airport codes and details
- **Terminal** - Terminal facilities
- **Stand** - Aircraft parking positions
- **Gate** - Passenger boarding gates
- **Counter** - Check-in counters
- **Belt** - Baggage claim carousels
- **Runway** - Runway definitions

**Airline & Aircraft:**
- **Airline** - Airline codes and details
- **Aircraft** - Individual aircraft registrations
- **AircraftType** - Aircraft models (A320, B737, etc.)
- **Country** - Country codes

**Operational:**
- **DelayCode** - IATA delay classification codes
- **Season** - IATA season definitions

### Schema Structure

Each entity type has:
- **Standard attributes** (code, name, validFrom, validTo)
- **Custom attributes** (tenant-specific fields)
- **Relationships** (e.g., Stand → Terminal)

```csharp
// Example: Stand entity
{
  "Code": "42A",
  "Name": "Stand 42A",
  "Terminal": "T2",
  "StandType": "Contact",
  "MaxAircraftSize": "CodeE",
  "ValidFrom": "2025-01-01T00:00:00Z",
  "ValidTo": "9999-12-31T23:59:59Z",
  "RegistrationFrom": "2024-12-15T10:30:00Z",
  "RegistrationTo": "9999-12-31T23:59:59Z"
}
```

## Data Flow

### 1. Inbound Data Sources

```
External Systems (Amadeus, Manual Entry)
         ↓
   Master Data API
         ↓
   Validation & Versioning
         ↓
   PostgreSQL (Bitemporal Tables)
```

**Sources:**
- **Amadeus AMS** - Automated feed of airport resources
- **Manual Entry** - UI for authorized users to maintain data
- **Bulk Import** - CSV/Excel import for initial setup

### 2. Delta Calculation & Publishing

When Master Data changes:

```csharp
// Pseudo-code for delta calculation
foreach (var changedEntity in updatedEntities)
{
    // Calculate affected time ranges
    var oldValidRange = GetValidRange(oldVersion);
    var newValidRange = GetValidRange(newVersion);
    
    // Determine delta range
    var deltaStart = Min(oldValidRange.Start, newValidRange.Start);
    var deltaEnd = Max(oldValidRange.End, newValidRange.End);
    
    // Identify changed attributes
    var changedAttributes = GetDifferences(oldVersion, newVersion);
    
    // Publish delta message
    PublishToTopic("masterdata-update-delta", new {
        EntityType = "Stand",
        EntityId = entity.Id,
        TimeRangeStart = deltaStart,
        TimeRangeEnd = deltaEnd,
        ChangedAttributes = changedAttributes
    });
}
```

**Delta Message Structure:**

```json
{
  "entityType": "Airline",
  "entityId": "SK",
  "validFrom": "2025-01-15T00:00:00Z",
  "validTo": "9999-12-31T23:59:59Z",
  "changedAttributes": ["Name", "IataCode"],
  "changeType": "Update"
}
```

### 3. Consumption by AODB

AODB subscribes to delta topic and:

1. **Identifies Affected Entities**
   - Queries flights/entities within the time range
   - Example: "Find all flights using Airline SK between Jan 15 and Dec 31"

2. **Triggers Recalculation**
   - Marks affected entities with `ProcessingStatus.Requested`
   - Republishes to recalculation topic
   - [[Recalculation and Dependency Tracking|Full recalculation flow]]

3. **Re-runs Enrichment Rules**
   - Rules that query the changed Master Data entity
   - Rules with `DependsOn` declarations
   - Produces updated output

## API Endpoints

### Read API (OData)

```http
GET /api/odata/masterdata/{EntityType}
  ?$filter=ValidFrom le 2025-12-25 and ValidTo ge 2025-12-25
  &$orderby=Code
```

**Common Queries:**

```http
# Get current airlines
GET /api/odata/masterdata/Airline
  ?$filter=ValidTo gt now()

# Get stands available on specific date
GET /api/odata/masterdata/Stand
  ?$filter=ValidFrom le 2025-12-25T10:00:00Z 
    and ValidTo ge 2025-12-25T10:00:00Z

# Get historical aircraft type
GET /api/odata/masterdata/Aircraft
  ?$filter=Registration eq 'OY-KBH'
    and ValidFrom le 2024-07-04T00:00:00Z
    and ValidTo ge 2024-07-04T00:00:00Z
    and RegistrationFrom le 2024-07-04T00:00:00Z
```

### Write API (REST)

```http
# Create new entity
POST /api/masterdata/{EntityType}
Content-Type: application/json

{
  "code": "42B",
  "name": "Stand 42B",
  "validFrom": "2025-02-01T00:00:00Z",
  "terminal": "T2"
}

# Update entity (creates new version)
PUT /api/masterdata/{EntityType}/{id}
Content-Type: application/json

{
  "name": "Stand 42B - Renovated",
  "validFrom": "2025-03-01T00:00:00Z"
}

# Schedule future deactivation
PUT /api/masterdata/Stand/42B
{
  "validTo": "2025-12-31T23:59:59Z"
}
```

## Integration with AODB

### Enrichment Rules Query Master Data

Business rules in AODB enrichment stage query Master Data:

```csharp
// Example: Enrich flight with airline name
public class EnrichAirlineDetails : IEnrichmentRule
{
    public void Execute(RuleContext context)
    {
        var flight = context.FlightLeg;
        var operationTime = flight.ScheduledTimeOfDeparture;
        
        // Query Master Data with bitemporal constraints
        var airline = context.MasterData.Airlines
            .Where(a => a.IataCode == flight.AirlineIataCode)
            .Where(a => a.ValidFrom <= operationTime 
                     && a.ValidTo >= operationTime)
            .FirstOrDefault();
        
        if (airline != null)
        {
            flight.AirlineName = airline.Name;
            flight.AirlineIcaoCode = airline.IcaoCode;
        }
    }
}
```

**Implicit Dependency Tracking:**
- Rule queries `Airlines` → Declares dependency on Airline Master Data
- When Airline changes → Rule automatically re-triggered
- [[Business Rules|More on rule development]]

### Recalculation Trigger Example

**Scenario:** Airline SK changes IATA code from "SK" to "SAS"

1. Master Data updated with `ValidFrom = 2025-06-01`
2. Delta message published with time range `2025-06-01` to `9999-12-31`
3. AODB queries: "Find flights with airline SK after June 1"
4. 1,247 flights identified
5. All flights marked for recalculation
6. Enrichment rules re-run with updated airline data
7. Output published to QueryStream with corrected airline name

## UI Management

The Master Data UI provides:

**Entity Browser:**
- List view with filtering and search
- Timeline view showing validity periods
- Audit log showing all changes

**Entity Editor:**
- Form-based editing with validation
- Future-date scheduling
- Bulk import/export

**Schema Manager:**
- Add/modify entity types
- Define custom attributes
- Configure relationships

**Permissions:**
- Read: View Master Data
- Write: Create/update entities
- Admin: Modify schema

## Storage Architecture

### Database Schema

```sql
-- Example: Airline table with bitemporal columns
CREATE TABLE masterdata.airline (
    entity_id UUID PRIMARY KEY,
    iata_code VARCHAR(3) NOT NULL,
    icao_code VARCHAR(4),
    name VARCHAR(255) NOT NULL,
    country_code VARCHAR(3),
    
    -- Business time
    valid_from TIMESTAMPTZ NOT NULL,
    valid_to TIMESTAMPTZ NOT NULL DEFAULT '9999-12-31 23:59:59+00',
    
    -- System time
    registration_from TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    registration_to TIMESTAMPTZ NOT NULL DEFAULT '9999-12-31 23:59:59+00',
    
    -- Custom attributes (JSONB for flexibility)
    custom_attributes JSONB,
    
    -- Indexes for bitemporal queries
    CONSTRAINT valid_time_range CHECK (valid_from < valid_to),
    CONSTRAINT reg_time_range CHECK (registration_from < registration_to)
);

CREATE INDEX idx_airline_valid_time ON masterdata.airline 
    USING GIST (tstzrange(valid_from, valid_to));
CREATE INDEX idx_airline_reg_time ON masterdata.airline 
    USING GIST (tstzrange(registration_from, registration_to));
```

### Query Optimization

PostgreSQL's GIST indexes enable efficient temporal queries:

```sql
-- Find airlines valid on specific date (millisecond performance)
SELECT * FROM masterdata.airline
WHERE tstzrange(valid_from, valid_to) @> '2025-12-25 10:00:00+00'::timestamptz
  AND tstzrange(registration_from, registration_to) @> NOW();
```

## Kafka Topics

### Published Topics

**masterdata-update-delta**
- Published when Master Data changes
- Contains entity type, ID, time range, changed attributes
- Consumed by AODB for recalculation
- Partitioned by entity type
- Compacted for efficiency

**masterdata-snapshot**
- Full snapshot published on schedule (e.g., daily)
- Used for disaster recovery and new consumer initialization
- Contains all currently valid Master Data

## Monitoring & Metrics

### Key Metrics

```
masterdata.entities.count{entity_type=Airline} - Total entities per type
masterdata.updates.rate - Updates per second
masterdata.delta.publish.latency - Time from update to delta publish
masterdata.query.duration{query_type=bitemporal} - Query performance
```

### Health Checks

- Database connectivity
- Delta publish lag
- Schema validation
- API response times

### Alerts

- **High delta lag** - > 5 seconds from update to publish
- **Schema validation failures** - Invalid data attempted
- **Query performance degradation** - Slow bitemporal queries
- **Recalculation backlog** - AODB not consuming delta messages

## Best Practices

### Data Management

1. **Plan Future Changes**
   - Use ValidFrom for scheduled updates
   - Test changes in staging environment first
   - Coordinate with operations team

2. **Avoid Frequent Changes**
   - Batch updates when possible
   - Each change triggers recalculation
   - Consider impact on downstream systems

3. **Maintain Data Quality**
   - Validate codes against standards (IATA, ICAO)
   - Use consistent naming conventions
   - Document custom attributes

### Query Optimization

1. **Always Use Time Constraints**
   ```csharp
   // Good
   var airline = masterData.Airlines
       .Where(a => a.ValidFrom <= operationTime 
                && a.ValidTo >= operationTime)
       .FirstOrDefault();
   
   // Bad - returns all versions
   var airline = masterData.Airlines.FirstOrDefault();
   ```

2. **Use Proper Indexes**
   - GIST indexes for time ranges
   - B-tree for codes and IDs
   - Avoid SELECT * in enrichment rules

## Common Scenarios

### Scenario 1: Scheduled Stand Closure

**Requirement:** Stand 42 closed for maintenance March 1-15, 2025

**Implementation:**
```json
PUT /api/masterdata/Stand/42

{
  "code": "42",
  "validTo": "2025-03-01T00:00:00Z"
}

// Create new version with future reactivation
POST /api/masterdata/Stand

{
  "code": "42",
  "validFrom": "2025-03-16T00:00:00Z",
  "validTo": "9999-12-31T23:59:59Z"
}
```

**Result:**
- Flights scheduled for March 1-15 trigger recalculation
- Stand allocation rules see Stand 42 unavailable during closure
- Automatic reactivation on March 16

### Scenario 2: Airline Merger

**Requirement:** Airline "XY" merges into "AB" on July 1, 2025

**Implementation:**
```http
# End XY airline
PUT /api/masterdata/Airline/XY
{
  "validTo": "2025-06-30T23:59:59Z"
}

# Update AB airline (if needed)
PUT /api/masterdata/Airline/AB
{
  "name": "AB Airways (merged with XY)",
  "validFrom": "2025-07-01T00:00:00Z"
}
```

**Result:**
- Historical flights show correct airline at time of operation
- Future flights automatically reference new airline
- Reporting preserves pre-merger data

### Scenario 3: New Terminal Opening

**Requirement:** Terminal T3 opens January 15 with 20 new stands

**Implementation:**
```http
# Create terminal
POST /api/masterdata/Terminal
{
  "code": "T3",
  "name": "Terminal 3",
  "validFrom": "2025-01-15T00:00:00Z"
}

# Bulk create stands
POST /api/masterdata/Stand/bulk
{
  "stands": [
    { "code": "301", "terminal": "T3", "validFrom": "2025-01-15T00:00:00Z" },
    { "code": "302", "terminal": "T3", "validFrom": "2025-01-15T00:00:00Z" },
    // ... 18 more stands
  ]
}
```

## Related Notes

- [[AODB Component]] - How AODB consumes Master Data
- [[Recalculation and Dependency Tracking]] - Recalculation triggered by Master Data changes
- [[Business Rules]] - Enrichment rules that query Master Data
- [[Entity Model]] - Flexible schema system

---

**Tags:** #masterdata #bitemporal #reference-data #component
**Created:** 2025-12-15
**References:**
- [Confluence: DD130 - Master Data Component](https://dapproduct.atlassian.net/wiki/spaces/PD/pages/426913)
- [Confluence: DD120 - Master Data](https://dapproduct.atlassian.net/wiki/spaces/PD/pages/93194709)
- Codebase: `SA.Component.MasterData/Solution/`
