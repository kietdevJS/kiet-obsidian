# AODB Component

> Airport Operational Database - The core data processing engine of AIRHART

## Overview

The AODB component is responsible for **gathering, enriching, and managing operational data** for airport operations. It processes messages from adapters, applies business rules, enriches data with master data, and publishes updated entities to downstream consumers.

## Architecture

### High-Level Structure

```
┌─────────────────────────────────────────────────┐
│           AODB Component                    │
│                                             │
│  ┌──────────────┐  ┌────────────────────────┐  │
│  │    API      │  │   Background Services│  │
│  │  (REST)     │  │   (Scheduled Jobs)   │  │
│  └──────────────┘  └────────────────────────┘  │
│                                             │
│  ┌─────────────────────────────────────────┐   │
│  │      AODB Pipeline (Main Processing) │   │
│  │                                      │   │
│  │  Input → Map → Match → Enrich →      │   │
│  │  Rules → Persist → Publish           │   │
│  └─────────────────────────────────────────┘   │
│                                             │
│  ┌──────────────┐  ┌────────────────────────┐  │
│  │   Database   │  │  Plugin Assemblies  │  │
│  │  (PostgreSQL)│  │  (Business Rules)   │  │
│  └──────────────┘  └────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

### Three Main Applications

1. **AODB API** (`API` project)
   - REST endpoints for CRUD operations
   - Schema management
   - System message handling
   - External interface for integrations

2. **AODB Pipeline** (`Pipeline.Host` project)
   - Core message processing engine
   - Consumes from Kafka topics
   - Applies business logic
   - Publishes output documents

3. **Background Services** (within Pipeline.Host)
   - Scheduled recalculations
   - Master data update processing
   - Business rule version updates
   - Maintenance tasks

## AODB Pipeline Architecture

### Pipeline Stages

```
┌─────────────────────────────────────────────────────────────┐
│                    AODB Pipeline Flow                   │
└─────────────────────────────────────────────────────────────┘

1. ADAPTER CONSUMER
   ├─ Consume from interface topic
   ├─ Deserialize AdapterMessage
   └─ Forward to matcher

2. MAPPING & MATCHING
   ├─ Map to internal format (IMapperRule)
   ├─ Match to existing entity (IMatcherSubRule)
   └─ Create/update EntityValue

3. ENRICHMENT
   ├─ Load related entities
   ├─ Fetch master data (airports, airlines, etc.)
   ├─ Apply enrichment rules
   └─ Calculate derived fields

4. BUSINESS RULES
   ├─ Execute pre-persist rules
   ├─ Validate data
   ├─ Apply transformations
   └─ Execute post-persist rules

5. PERSISTENCE
   ├─ Save EntityValue to database
   ├─ Handle concurrency conflicts
   └─ Update version/timestamp

6. PUBLISHING
   ├─ Build document DTOs
   ├─ Publish to output topics
   └─ Publish adapter messages (status)
```

See: [[AODB Processing Pipeline]]

### Key Components

#### Pipeline.Host
**Location:** `Solution/Pipeline/Pipeline.Host/`

**Responsibilities:**
- Host application for pipeline processing
- Kafka consumer management
- Plugin loading and execution
- Metrics and health checks

**Key Files:**
- `Program.cs` - Application entry point
- `Startup.cs` - Dependency injection configuration
- `appsettings.json` - Configuration

#### Plugins System
**Location:** `Solution/Pipeline/Plugins/`

**Plugin Types:**
1. **Plugin.Matching** - Message matching logic
2. **Plugin.Visits** - Visit linking logic
3. **PluginBase** - Common plugin infrastructure

**Plugin Hooks:**
- `IMapperRule<TAdapterMessage>` - Map adapter message to entity
- `IMatcherSubRule<TEntityType>` - Match message to entity
- `IEnrichmentRule<TEntityType>` - Enrich entity data
- `IBusinessRule<TEntityType>` - Apply business logic

See: [[Plugin Architecture]]

#### Pipeline.Publishing
**Location:** `Solution/Pipeline/Pipeline.Publishing/`

**Responsibilities:**
- Build output documents
- Publish to Kafka topics
- Handle DTD (Document Type Definition) changes
- Manage outbox pattern for guaranteed delivery

**Key Topics:**
- `aodb-pipeline-output` - Entity update documents
- `traffic-operational-adapter-messages` - Adapter message status
- `aodb-pipeline-dtd` - Document type definitions

## Data Model

### Entity-Attribute-Value (EAV) Model

The AODB uses a flexible EAV model for storing operational data:

```
EntityType (Schema Definition)
  └─ Attributes[]
      ├─ Name
      ├─ DataType
      └─ Configuration

EntityValue (Actual Data)
  └─ Attributes[]
      ├─ Value
      ├─ Source
      ├─ Received
      └─ Alternatives[]
```

**Example: Flight Leg**
```json
{
  "entity_type": "flight_leg",
  "entity_id": "12345",
  "attributes": {
    "scheduled_in_block_time": {
      "value": "2025-12-15T10:30:00Z",
      "source": "ACD",
      "received": "2025-12-15T08:00:00Z",
      "alternatives": [...]
    },
    "arrival_airport": {
      "value": "CPH",
      "source": "SCR",
      "received": "2025-12-15T08:01:00Z"
    }
  }
}
```

See: [[Entity Model]]

### Core Entity Types

1. **flight_leg** - Individual flight arrival/departure
2. **ground_leg** - Aircraft movement on ground
3. **visit** - Linked inbound + outbound flights
4. **operational_event** - INOP/State changes
5. **raw_message** - Unmatched adapter messages

### Schema Management

**Schema API:**
- `POST /api/schema/entity-types` - Create entity type
- `PUT /api/schema/entity-types/{id}` - Update entity type
- `GET /api/schema/entity-types` - List entity types

**Schema Versioning:**
- Schema changes trigger pipeline restart
- DTD published to `traffic-operational-schema-update`
- Backward compatibility maintained

## Message Processing

### Inbound Topics

| Topic | Producer | Message Type |
|-------|----------|--------------|
| `traffic-operational-flightlegacd` | ACD Adapter | Flight schedules |
| `traffic-planning-flightlegacd` | ACD Adapter | Planning data |
| `traffic-operational-flightlegsita` | SITA Adapter | Flight updates (MVT, etc.) |
| `traffic-operational-*` | Various Adapters | Entity updates |

### Internal Topics

| Topic | Purpose | Partitions |
|-------|---------|------------|
| `traffic-operational-entityvalue-enrichment-*` | Entity enrichment queue | 50 |
| `traffic-operational-entityvalue-recalculation-*` | Recalculation requests | 50 |
| `traffic-operational-business-rules-updates` | Rule version updates | 1 |

### Outbound Topics

| Topic | Consumer | Content |
|-------|----------|---------|
| `aodb-pipeline-output` | QueryStream | Entity documents |
| `traffic-operational-adapter-messages` | QueryStream | Adapter message status |
| `aodb-pipeline-dtd` | QueryStream | DTD definitions |

See: [[AODB Messaging]]

## Services

### Services.Messaging
**Location:** `Solution/Services.Messaging/`

**Responsibilities:**
- Kafka message production/consumption
- Message serialization/deserialization
- Trace ID propagation
- Internal message types

### Services.Plugins
**Location:** `Solution/Services.Plugins/`

**Responsibilities:**
- Plugin discovery and loading
- Assembly management
- Rule execution orchestration
- Plugin context provision

### Services.Schema
**Location:** `Solution/Services.Schema/`

**Responsibilities:**
- Schema definition management
- Entity type CRUD operations
- Attribute configuration
- Schema validation

### Services.Versioning
**Location:** `Solution/Services.Versioning/`

**Responsibilities:**
- Entity version management
- Concurrency control
- Change tracking
- Audit trail

### Services.Recalculations
**Location:** `Solution/Services.Recalculations/`

**Responsibilities:**
- Trigger recalculation workflows
- Handle master data updates
- Manage recalculation scope
- Deduplication logic

### Services.Deduplication
**Location:** `Solution/Services.Deduplication/`

**Responsibilities:**
- Detect duplicate messages
- Filter redundant updates
- Prevent infinite loops
- Message history tracking

## Business Rules

### Rule Types

1. **Mapper Rules** (`IMapperRule<TAdapterMessage>`)
   - Transform adapter message to entity format
   - Extract identifier information
   - Map fields to attributes

2. **Matcher Rules** (`IMatcherSubRule<TEntityType>`)
   - Find existing entity by identifier
   - Create new entity if no match
   - Handle multi-match scenarios

3. **Enrichment Rules** (`IEnrichmentRule<TEntityType>`)
   - Fetch related data (master data, other entities)
   - Calculate derived attributes
   - Apply data quality rules

4. **Business Rules** (`IBusinessRule<TEntityType>`)
   - Validation logic
   - Complex business logic
   - Cross-entity updates
   - Trigger actions

### Rule Configuration

Rules are defined in tenant-specific repositories:
- **CPH.Configuration.AODB** - Copenhagen specific rules
- Compiled as plugin assemblies
- Hot-reloadable without deployment

**Example Rule Registration:**
```csharp
public class AdapterInitializer : IAdapterInitializer
{
    public void RegisterAdapters(IAdapterMappingDefinitions definitions)
    {
        definitions.AddMapping<FlightLegAcdScr, FlightLeg>(
            mapperRule: new FlightLegAcdScrMapper(),
            matcherSubRule: new FlightLegMatcher()
        );
    }
}
```

See: [[Business Rules]]

## Database Structure

### Core Tables

```
aodb_entity_types       → Schema definitions
aodb_entity_values      → Actual entity data
aodb_raw_messages       → Unmatched messages
aodb_versioning         → Version tracking
aodb_recalculations     → Recalculation queue
```

### Database Projects

- **Database** - EF Core models and context
- **Database.Query** - Query optimizations
- **Database.Plugins** - Plugin-specific queries
- **Database.Configuration** - Migrations and seed data

## Resilience

### Retry Policies

1. **Transient Errors** (NpgsqlException, DbUpdateConcurrencyException)
   - Retry forever
   - Exponential backoff

2. **Unsolvable Errors** (InfiniteReprocessingException, LockedFlightLegException)
   - Skip message
   - No dead letter (expected behavior)

3. **Unknown Errors** (Default)
   - Retry 5 times
   - Publish to dead letter topic
   - Log and alert

See: [[Resilience Patterns]]

### Concurrency Handling

- **Optimistic Locking** via ETags
- **Retry on Conflict** - Automatic retry
- **Ordering Preservation** - Same-source messages ordered
- **Partitioning** - Entity ID-based routing

## Performance

### Scaling Strategy
- **Horizontal Scaling** - Multiple pipeline pods
- **Partition Count** - Match pod count (max 50)
- **Consumer Groups** - One consumer per partition
- **Priority Queues** - Operational vs. planning topics

### Optimization Techniques
- **Bulk Operations** - Batch master data fetches
- **Caching** - Master data cache
- **Lazy Loading** - Load related entities on demand
- **Connection Pooling** - Database connection reuse

## Monitoring

### Metrics
- `aodb_messages_consumed_total` - Messages processed
- `aodb_processing_duration_seconds` - Processing time
- `aodb_errors_total` - Errors by type
- `aodb_deadletter_messages_total` - Failed messages

### Health Checks
- Kafka connectivity
- Database connectivity
- Master data availability
- Plugin assembly load status

## Integration Points

### Master Data Component
**Dependency:** Strong coupling for enrichment

**Usage:**
- Fetch airport data
- Fetch airline data
- Fetch resource data (stands, gates, etc.)

**Connection:** Direct database read access

### QueryStream Component
**Interaction:** Producer → Consumer

**Published:**
- Entity update documents
- Adapter message status
- DTD definitions

### Adapter Components
**Interaction:** Consumer ← Producer

**Consumed:**
- Flight leg updates
- Ground leg updates
- Operational events
- Raw messages

## Configuration

### Key Settings (appsettings.json)

```json
{
  "AdapterMessageTypes-All": [
    "FlightLegAcdScr",
    "FlightLegSitaMvt",
    ...
  ],
  "EntityTypes": ["flight_leg", "visit", "ground_leg"],
  "KafkaSettings": {
    "BootstrapServers": "...",
    "ConsumerGroup": "aodb-pipeline"
  },
  "DatabaseSettings": {
    "ConnectionString": "..."
  }
}
```

## Development

### Solution Structure

```
SA.Component.AODB/
├── Solution/
│   ├── API/                      → REST API
│   ├── Pipeline/
│   │   ├── Pipeline.Host/        → Main pipeline app
│   │   ├── Pipeline.Publishing/  → Output publishing
│   │   ├── Plugins/              → Plugin system
│   │   └── ...
│   ├── Services.*/               → Shared services
│   ├── Database/                 → Data access
│   └── ...
└── Test.Configuration.AODB/      → Test configuration
```

### Running Locally

1. **Prerequisites:**
   - Kafka running (local or cluster)
   - PostgreSQL database
   - Master Data database

2. **Configuration:**
   - Update `appsettings.Development.json`
   - Set connection strings
   - Configure Kafka topics

3. **Run:**
   ```bash
   cd Solution/Pipeline/Pipeline.Host
   dotnet run
   ```

## Related Notes

- [[AODB Processing Pipeline]] - Detailed pipeline flow
- [[AODB Messaging]] - Topic and message details
- [[Business Rules]] - Rule development guide
- [[Plugin Architecture]] - Plugin development
- [[Entity Model]] - Data model details

---

**Tags:** #aodb #component #pipeline #processing
**Created:** 2025-12-15
**References:**
- [Confluence: DD130 - AODB Component](https://dapproduct.atlassian.net/wiki/spaces/PD/pages/950525)
- Codebase: `SA.Component.AODB/Solution/`
