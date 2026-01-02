# QueryStream Component

> Real-time data query and streaming service for AIRHART

## Overview

QueryStream (QS) is the **data distribution layer** of AIRHART. It consumes entity updates from AODB and other components, stores them in optimized feed databases, and streams real-time updates to clients via WebSocket subscriptions and REST APIs.

## Core Purpose

```
AODB/MasterData → QueryStream → Clients (UI, APIs, External Systems)
     (Events)              (Queries + Streams)
```

**Key Capabilities:**
- Real-time streaming of data changes
- OData query support for flexible filtering
- REST API for point-in-time queries
- Snapshot + incremental updates pattern
- Feed-based data organization

## Architecture

### High-Level Structure

```
┌──────────────────────────────────────────────────┐
│          QueryStream Component                   │
│                                                  │
│  ┌────────────────────┐  ┌───────────────────┐  │
│  │ Database Manager   │  │ Subscription Mgr  │  │
│  │                    │  │                   │  │
│  │ Kafka Consumer     │  │ WebSocket Server  │  │
│  │ Document Compiler  │  │ OData Query       │  │
│  │ Feed Persister     │  │ Client Manager    │  │
│  └────────────────────┘  └───────────────────┘  │
│           ↓                       ↑              │
│  ┌────────────────────────────────────────────┐  │
│  │      Feed Database (PostgreSQL)            │  │
│  │  - flight_legs_feed                        │  │
│  │  - visits_feed                             │  │
│  │  - master_data_feeds                       │  │
│  └────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────┘
```

### Three Main Applications

1. **Database Manager** (Clients/DatabaseManager)
   - Consumes messages from Kafka topics
   - Compiles documents based on Feed definitions
   - Persists to Feed database
   - Publishes to internal topic for Subscription Manager

2. **Subscription Manager** (Clients/SubscriptionManager)
   - Manages client WebSocket connections
   - Handles subscription lifecycle
   - Queries Feed database for snapshots
   - Streams incremental updates to clients

3. **REST Service** (within Subscription Manager)
   - OData endpoint for queries
   - Point-in-time data access
   - Instant views (temporal queries)

4. **Feed Manager** (Clients/FeedManager)
   - Builds and rebuilds Feed databases
   - Schema management
   - Feed initialization

## Feed Concept

### What is a Feed?

A **Feed** is a **curated, optimized view** of source data designed for efficient querying and streaming.

**Feed Creation:**
```
Source Model (AODB flight_leg)
  600 attributes × 10 properties each
              ↓  (Filter & Select)
Flight Leg Feed
  150 attributes × 2 properties each
  (Only: Value, Received)
```

**Benefits:**
- Reduced data size for faster queries
- Optimized for specific use cases
- Combines multiple sources into one view
- Pre-computed joins and aggregations

### Feed Types

1. **Subset Feeds** - Subset of source model
   - Example: `flight_legs_feed` with selected attributes

2. **Composite Feeds** - Multiple sources combined
   - Example: `visits_feed` = visit + inbound flight + outbound flight

3. **Filtered Feeds** - Time-window or condition-based
   - Example: Operational window (now - 2h to now + 24h)

### Feed Definition

**Key Properties:**
- `typename` - Entity type name
- `source_component` - Origin (AODB, MasterData)
- `version` - Schema version
- `is_feed` - Marks as queryable feed
- `feed_state` - Uninitialized, Initializing, Ready

**Storage:**
- DTD table - Feed schema definition
- Feed tables - Actual feed data (one table per feed)

## Document Flow

### Inbound Processing

```
1. SOURCE MESSAGE
   Kafka Topic: aodb-pipeline-output
   {
     "entity_id": "12345",
     "entity_type": "flight_leg",
     "operation": "update",
     "data": { ... }
   }

2. DATABASE MANAGER
   ├─ Consume from topic
   ├─ Identify affected feeds
   ├─ Compile documents for each feed
   └─ Store in feed database

3. FEED DOCUMENT
   ├─ Raw data store (full message)
   └─ Feed data store (compiled view)

4. PUBLISH INTERNALLY
   Internal Topic: qs-internal-updates
   (for Subscription Manager)
```

### Subscription Flow

```
1. CLIENT SUBSCRIBES
   WebSocket: /api/ws/feeds/flight_legs
   Query: $filter=arrival_airport eq 'CPH'

2. SNAPSHOT PHASE
   ├─ Query feed database
   ├─ Apply OData filter
   ├─ Stream matching documents to client
   └─ Mark snapshot complete

3. STREAMING PHASE
   ├─ Listen to internal topic
   ├─ Match documents against subscription filter
   ├─ Stream updates to client in real-time
   └─ Handle client disconnection
```

See: [[QueryStream Subscriptions]]

## Data Sources

### Consumed Topics

| Producer | Topic | Message Type | DTD Topic |
|----------|-------|--------------|-----------|
| AODB Pipeline | `aodb-pipeline-output` | Entity documents | `aodb-pipeline-dtd` |
| AODB Pipeline | `traffic-operational-adapter-messages` | Adapter messages | `aodb-pipeline-dtd` |
| AODB Pipeline | `traffic-operational-mapped-messages` | Mapped messages | `aodb-pipeline-dtd` |
| Master Data | `masterdata-data-update-topic` | Master data updates | `masterdata-schema-update-topic` |
| Adapters | `inbound-raw-*` | Raw adapter messages | - |

### Published Topics

| Topic | Consumer | Purpose |
|-------|----------|---------|
| `qs-internal-updates` | Subscription Manager | Internal document updates |

## Components Deep Dive

### Database Manager

**Location:** `Solution/Clients/DatabaseManager/`

**Responsibilities:**
1. **Message Consumption**
   - Kafka consumer with partition support
   - Concurrent processing across partitions
   - Offset management and checkpointing

2. **Document Compilation**
   - Load Feed definitions
   - Map source message to Feed schema
   - Handle relations and joins
   - Apply Feed filters

3. **Persistence**
   - Upsert to Feed database
   - Transaction management
   - Conflict resolution

4. **Publishing**
   - Publish to internal topic
   - Only for active Feeds
   - Include Feed metadata

**Key Classes:**
- `DatabaseManagerService` - Main service orchestrator
- `FeedCompiler` - Compiles messages into Feed documents
- `DocumentRepository` - Database access
- `MessageConsumer` - Kafka consumer wrapper

### Subscription Manager

**Location:** `Solution/Clients/SubscriptionManager/`

**Responsibilities:**
1. **Connection Management**
   - WebSocket server (SignalR)
   - Client authentication via IDP
   - Connection lifecycle tracking

2. **Subscription Handling**
   - Parse OData queries
   - Validate Feed access
   - Execute snapshot queries
   - Stream incremental updates

3. **Query Processing**
   - OData to SQL translation
   - Filter optimization
   - Pagination support
   - Temporal queries ($at timestamp)

4. **Update Distribution**
   - Match updates to subscriptions
   - Filter by client permissions
   - Handle client buffer overflow
   - Manage disconnections

**Key Classes:**
- `SubscriptionManagerHub` - SignalR hub
- `SubscriptionService` - Subscription lifecycle
- `QueryService` - OData query processing
- `UpdateMatcher` - Match updates to subscriptions

See: [[Subscription Manager Architecture]]

### Feed Manager

**Location:** `Solution/Clients/FeedManager/`

**Responsibilities:**
1. **Feed Initialization**
   - Create Feed tables
   - Build initial dataset from source
   - Update Feed state

2. **Feed Rebuilding**
   - Triggered on schema changes
   - Asynchronous processing
   - Maintains availability during rebuild

3. **Schema Management**
   - DTD tracking
   - Version management
   - Schema evolution

### REST Service

**Location:** Embedded in Subscription Manager

**Endpoints:**
```
GET /api/odata/feeds/{feedName}
  - Query Feed with OData
  - Examples:
    - $filter=arrival_airport eq 'CPH'
    - $select=scheduled_in_block_time,flight_number
    - $top=100&$skip=0
    - $orderby=scheduled_in_block_time desc

GET /api/odata/feeds/{feedName}/{timestamp}
  - Instant view (temporal query)
  - Example: /api/odata/feeds/flight_legs/2025-12-15T10:00:00Z
```

## Core Services

### Core/Feeds
**Location:** `Solution/Core/Feeds/`

**Components:**
- `FeedDefinition` - Feed schema models
- `FeedProcessor` - Feed processing logic
- `FeedValidator` - Feed validation

### Core/QueryService
**Location:** `Solution/Core/QueryService/`

**Components:**
- `ODataQueryParser` - Parse OData syntax
- `QueryExecutor` - Execute queries against database
- `ResultSerializer` - Serialize results to JSON

### Core/LinqVisitor
**Location:** `Solution/Core/LinqVisitor/`

**Purpose:** Translate LINQ expressions to SQL
- Parse OData filter expressions
- Convert to SQL WHERE clauses
- Handle complex predicates

### Core/ODataVisitorForSql
**Location:** `Solution/Core/ODataVisitorForSql/`

**Purpose:** OData-specific SQL generation
- $filter → WHERE
- $select → SELECT columns
- $orderby → ORDER BY
- $expand → JOINs

## Plugins

### Plugin System

**Location:** `Solution/Plugins/`

**Available Plugins:**
1. **AODB Plugin** - Flight legs, visits, ground legs
2. **MasterData Plugin** - Airports, airlines, resources
3. **AdapterMessage Plugin** - Adapter message feed
4. **CustomFeeds Plugin** - Tenant-specific feeds

**Plugin Interface:**
```csharp
public interface IFeedPlugin
{
    string FeedName { get; }
    void ConfigureFeed(FeedDefinitionBuilder builder);
    Document CompileDocument(SourceMessage message);
}
```

See: [[QueryStream Plugins]]

## Subscription Lifecycle

### States

```
1. CONNECTING
   - Client establishes WebSocket
   - Authentication via JWT

2. SUBSCRIBING
   - Client sends subscription request
   - Feed + OData query

3. SNAPSHOTTING
   - Query feed database
   - Stream existing documents
   - Send snapshot complete marker

4. STREAMING
   - Listen for updates
   - Match against query
   - Send incremental updates

5. DISCONNECTED
   - Client closes connection
   - Cleanup subscription state
```

### Subscription Request

```json
{
  "subscriptionId": "client-generated-guid",
  "feedName": "flight_legs",
  "query": "$filter=arrival_airport eq 'CPH' and scheduled_in_block_time gt 2025-12-15T00:00:00Z"
}
```

### Update Message

```json
{
  "subscriptionId": "...",
  "operation": "update",  // create, update, delete
  "document": {
    "entity_id": "12345",
    "entity_type": "flight_leg",
    "data": { ... }
  }
}
```

## Database Structure

### Feed Tables

```sql
-- Flight Legs Feed
CREATE TABLE flight_legs_feed (
    entity_id TEXT PRIMARY KEY,
    entity_type TEXT,
    version BIGINT,
    data JSONB,  -- Full document
    indexed_fields ...  -- For query performance
);

-- Visits Feed
CREATE TABLE visits_feed (
    entity_id TEXT PRIMARY KEY,
    entity_type TEXT,
    version BIGINT,
    data JSONB,
    -- Denormalized fields from related entities
    inbound_flight_id TEXT,
    outbound_flight_id TEXT
);
```

### DTD Tracking

```sql
CREATE TABLE dtd (
    typename TEXT,
    source_component TEXT,
    version INTEGER,
    is_feed BOOLEAN,
    feed_state TEXT,
    schema_definition JSONB,
    PRIMARY KEY (typename, source_component, version)
);
```

## Scaling Strategy

### Horizontal Scaling

**Database Manager:**
- Scale based on Kafka lag
- Partition-based parallelism
- Max instances = partition count

**Subscription Manager:**
- Scale based on active connections
- Load balancer distributes clients
- Stateless connection handling

**Feed Database:**
- Read replicas for queries
- Write-through to primary
- Connection pooling

### Performance Optimization

1. **Indexes**
   - Commonly queried fields indexed
   - Composite indexes for complex queries
   - GIN indexes for JSONB

2. **Caching**
   - Feed metadata cached
   - Recent queries cached
   - Client subscription state in-memory

3. **Partitioning**
   - Kafka topic partitions (50)
   - Database table partitioning (time-based)
   - Connection pooling per instance

## Query Capabilities

### OData Support

**Supported Operators:**
- `$filter` - Filter documents
- `$select` - Select specific fields
- `$orderby` - Sort results
- `$top` / `$skip` - Pagination
- `$count` - Count results
- `$expand` - Expand relations

**Example Queries:**
```
# Filter by airport
$filter=arrival_airport eq 'CPH'

# Time range
$filter=scheduled_in_block_time gt 2025-12-15T00:00:00Z and 
        scheduled_in_block_time lt 2025-12-16T00:00:00Z

# Complex filter
$filter=(arrival_airport eq 'CPH' or departure_airport eq 'CPH') 
        and flight_number ge 100

# With relations
$expand=inbound_flight,outbound_flight

# Temporal query (instant view)
GET /feeds/flight_legs/2025-12-15T10:00:00Z?$filter=...
```

See: [[OData Query Support]]

## Message Tracing

### Trace ID Propagation

```
1. Adapter Message
   ├─ TraceId in message body
   └─ traceparent in Kafka header

2. AODB Processing
   ├─ Preserve TraceId
   └─ Add to output documents

3. QueryStream Storage
   ├─ Store TraceId
   └─ Enable trace queries

4. Query by TraceId
   ├─ $expand for message lineage
   └─ Track end-to-end flow
```

**Trace Query:**
```
GET /feeds/raw_messages?$filter=trace_id eq 'abc123'&$expand=mapped_messages,aodb_entities
```

## Integration Points

### AODB Component
**Relationship:** Primary data source

**Consumed:**
- Entity updates (flight_leg, visit, etc.)
- Adapter message status
- DTD definitions

**Topics:**
- `aodb-pipeline-output`
- `traffic-operational-adapter-messages`
- `aodb-pipeline-dtd`

### Master Data Component
**Relationship:** Reference data source

**Consumed:**
- Airport updates
- Airline updates
- Resource updates

**Topics:**
- `masterdata-data-update-topic`
- `masterdata-schema-update-topic`

### Client Applications
**Relationship:** Data consumers

**Provided:**
- WebSocket subscriptions for real-time updates
- REST API for queries
- OData for flexible filtering

**Protocols:**
- WebSocket (SignalR)
- HTTPS (REST)

## Security

### Authentication
- JWT tokens from IDP
- Token validation on connection
- Token refresh handling

### Authorization
- Scope-based permissions
- Feed-level access control
- Field-level security (future)

### Network
- TLS for all connections
- API Gateway enforcement
- Rate limiting per client

## Monitoring

### Metrics
- `qs_messages_consumed_total` - Messages processed
- `qs_subscription_count` - Active subscriptions
- `qs_query_duration_seconds` - Query performance
- `qs_update_latency_seconds` - Update delivery time

### Health Checks
- Kafka connectivity
- Database connectivity
- Feed availability
- Subscription manager status

## Configuration

### Key Settings

```json
{
  "Kafka": {
    "BootstrapServers": "...",
    "ConsumerGroup": "querystream-dbmanager"
  },
  "Database": {
    "ConnectionString": "...",
    "MaxPoolSize": 100
  },
  "Feeds": [
    {
      "Name": "flight_legs",
      "Source": "AODB",
      "EntityType": "flight_leg",
      "Enabled": true
    }
  ]
}
```

## Development

### Solution Structure

```
SA.Component.QueryStream/
├── Solution/
│   ├── Clients/
│   │   ├── DatabaseManager/       → Kafka consumer
│   │   ├── SubscriptionManager/   → WebSocket server
│   │   └── FeedManager/           → Feed builder
│   ├── Core/
│   │   ├── Feeds/                 → Feed logic
│   │   ├── QueryService/          → Query processing
│   │   ├── ODataVisitorForSql/    → OData to SQL
│   │   └── LinqVisitor/           → LINQ parsing
│   ├── Plugins/                   → Feed plugins
│   └── Infrastructure/
│       ├── Database/              → Data access
│       └── Models/                → DTOs
└── TestApps/                      → Test clients
```

### Running Locally

```bash
# Database Manager
cd Solution/Clients/DatabaseManager
dotnet run

# Subscription Manager
cd Solution/Clients/SubscriptionManager
dotnet run
```

## Related Notes

- [[QueryStream Subscriptions]] - Subscription patterns
- [[QueryStream Feeds]] - Feed configuration
- [[QueryStream Plugins]] - Plugin development
- [[OData Query Support]] - Query capabilities
- [[Subscription Manager Architecture]] - Client handling

---

**Tags:** #querystream #component #feeds #subscriptions #realtime
**Created:** 2025-12-15
**References:**
- [Confluence: DD130 - Query Stream](https://dapproduct.atlassian.net/wiki/spaces/PD/pages/9769080)
- Codebase: `SA.Component.QueryStream/Solution/`
