# Inbound Message Flow

> How messages flow from external systems into AIRHART

## Overview

The inbound message flow handles **integration from external systems** (like ACD, SITA, Amadeus) into the AIRHART core. Messages pass through adapters, are validated and transformed, then enter the AODB pipeline for processing.

## Complete Flow Diagram

```
┌──────────────────────────────────────────────────────────────┐
│              INBOUND MESSAGE FLOW                            │
└──────────────────────────────────────────────────────────────┘

1. EXTERNAL SYSTEM
   ├─ ACD (Flight schedules)
   ├─ SITA (Flight updates, MVT/LDM/PSM/PTM)
   ├─ Amadeus (F-RMS data)
   └─ Other systems
         ↓ (Various protocols: HTTP, FTP, WebSocket)

2. ADAPTER COMPONENT
   ├─ Receive message
   ├─ Protocol handling
   ├─ Initial validation
   ├─ Transform to standard format
   └─ Publish to Kafka
         ↓ (Kafka interface topic)

3. KAFKA INTERFACE TOPIC
   ├─ traffic-operational-flightlegacd
   ├─ traffic-operational-flightlegsita
   ├─ traffic-operational-*
   └─ (Partitioned for parallel processing)
         ↓ (Consumed by AODB Pipeline)

4. AODB ADAPTER CONSUMER
   ├─ Consume from topic
   ├─ Deserialize message
   ├─ Trace ID extraction
   └─ Forward to mapper/matcher
         ↓ (Mapping & Matching stage)

5. MAPPING & MATCHING
   ├─ Map to internal entity format
   ├─ Find existing entity (or create new)
   ├─ Handle unmatched messages
   └─ Prepare for enrichment
         ↓ (AODB internal topic)

6. AODB INTERNAL TOPIC
   ├─ traffic-operational-entityvalue-enrichment-*
   ├─ traffic-operational-entityvalue-recalculation-*
   └─ (Entity-specific routing)
         ↓ (AODB Pipeline processing)

7. AODB PIPELINE
   ├─ Enrichment
   ├─ Business rules
   ├─ Persistence
   └─ Publishing
         ↓ (Output topics)

8. OUTPUT
   ├─ aodb-pipeline-output → QueryStream
   ├─ traffic-operational-adapter-messages → Status tracking
   └─ Event notifications
```

## Stage Details

### 1. External System

**Sources:**
- **ACD (Airport Central Database)** - Flight schedules, planning
- **SITA** - Flight operations (MVT, LDM, PSM, PTM messages)
- **Amadeus AMS** - F-RMS reference data, operational events
- **Eurocontrol** - Network Manager data
- **Ground Handlers** - Service updates
- **Airlines** - Custom integrations

**Protocols:**
- HTTP/REST APIs
- FTP/SFTP file transfers
- WebSocket connections
- Message queues
- Direct database connections

**Message Types:**
- Flight leg updates
- Ground leg movements
- Operational events
- Master data updates
- Reference data

### 2. Adapter Component

**Purpose:** Translate external protocols to AIRHART standard

**Adapter Types:**

#### Inbound Flight Update Adapters
```
SA.Adapter.ACD.Scheduling
├─ Consumes: ACD SCR messages (flight schedules)
├─ Validates: Message format, required fields
├─ Transforms: To FlightLegAcdScr DTO
└─ Publishes: traffic-operational-flightlegacd

SA.Adapter.SITA.SMTP
├─ Consumes: SITA MVT/LDM/PSM/PTM messages
├─ Validates: IATA format
├─ Transforms: To FlightLegSita* DTOs
└─ Publishes: traffic-operational-flightlegsita
```

#### Inbound Reference Data Adapters
```
SA.Adapter.Amadeus.AMS
├─ Consumes: INOP, state changes, health status
├─ Validates: JSON schema
├─ Transforms: To operational event DTOs
└─ Publishes: traffic-operational-operationalevent
```

**Common Adapter Pattern:**
```csharp
1. Receive
   ├─ HTTP endpoint or file watcher
   └─ Authentication/authorization

2. Validate
   ├─ Schema validation
   ├─ Required field check
   └─ Business validation

3. Transform
   ├─ Map to DTO
   ├─ Add metadata (trace ID, timestamp)
   └─ Split multi-messages if needed

4. Publish
   ├─ Serialize to JSON
   ├─ Add Kafka headers
   └─ Publish to interface topic

5. Error Handling
   ├─ Invalid messages → inbound-invalid-{adapter}
   ├─ Retry transient errors
   └─ Dead letter for persistent errors
```

See: [[Adapter Development Guide]]

### 3. Kafka Interface Topics

**Topic Naming:**
- `traffic-operational-{entity}{source}` - Operational data
- `traffic-planning-{entity}{source}` - Planning data
- `inbound-raw-{adapter}` - Raw messages
- `inbound-invalid-{adapter}` - Invalid messages

**Topic Configuration:**

| Topic | Partitions | Retention | Consumer Group |
|-------|------------|-----------|----------------|
| `traffic-operational-flightlegacd` | 50 | 7 days | `aodb-pipeline` |
| `traffic-planning-flightlegacd` | 10 | 30 days | `aodb-pipeline` |
| `traffic-operational-flightlegsita` | 50 | 7 days | `aodb-pipeline` |

**Message Format:**
```json
{
  "meta": {
    "traceId": "abc-123-def",
    "timestamp": "2025-12-15T10:30:00Z",
    "source": "ACD",
    "messageType": "FlightLegUpdate"
  },
  "payload": {
    "flightNumber": "SK1234",
    "operatorIata": "SK",
    "scheduledInBlockTime": "2025-12-15T14:30:00Z",
    "arrivalAirport": "CPH",
    ...
  }
}
```

**Kafka Headers:**
- `traceparent` - W3C trace context
- `content-type` - Message content type
- `message-type` - DTO type name

### 4. AODB Adapter Consumer

**Purpose:** Bridge between Kafka topics and AODB pipeline

**Consumer Implementation:**
```csharp
public class AdapterConsumerMapper<TAdapterMessage> 
    where TAdapterMessage : IAdapterMessage
{
    // 1. Consume from topic (defined by [Channel] attribute)
    public async Task ConsumeAsync(CancellationToken ct)
    {
        await foreach (var message in _consumer.ConsumeAsync(ct))
        {
            // 2. Deserialize
            var adapterMessage = Deserialize<TAdapterMessage>(message);
            
            // 3. Extract trace ID
            var traceId = ExtractTraceId(message.Headers);
            
            // 4. Forward to matcher
            await _matcherInitializer.ProcessAsync(adapterMessage);
        }
    }
}
```

**Consumer Groups:**
- One consumer group per topic
- Multiple consumer instances per group
- Partition assignment managed by Kafka

**Configuration:**
```json
{
  "AdapterMessageTypes-All": [
    "FlightLegAcdScr",
    "FlightLegSitaMvt",
    "FlightLegSitaLdm",
    ...
  ]
}
```

Each type is automatically registered based on `[Channel]` attribute:
```csharp
[Channel("traffic-operational-flightlegacd")]
public class FlightLegAcdScr : IAdapterMessage { }
```

### 5. Mapping & Matching

**Purpose:** Transform adapter messages to internal entity format

#### Mapping Stage

**IMapperRule Interface:**
```csharp
public interface IMapperRule<TAdapterMessage>
{
    EntityValuePipelineInput Map(
        TAdapterMessage message,
        AdapterMessage rawMessage);
}
```

**Mapping Process:**
1. **Extract Identifier**
   ```csharp
   var identifier = new FlightLegIdentifier
   {
       OperatorIata = message.OperatorIata,
       FlightNumber = message.FlightNumber,
       ScheduledDate = message.ScheduledDate,
       ArrivalAirport = message.ArrivalAirport
   };
   ```

2. **Map Attributes**
   ```csharp
   var attributes = new Dictionary<string, AttributeValue>
   {
       ["scheduled_in_block_time"] = new AttributeValue
       {
           Value = message.ScheduledInBlockTime,
           Source = "ACD",
           Received = DateTime.UtcNow
       },
       ...
   };
   ```

3. **Create Pipeline Input**
   ```csharp
   return new EntityValuePipelineInput
   {
       EntityType = "flight_leg",
       Identifier = identifier,
       Attributes = attributes,
       MessageSource = MessageSource.ExternalAdapter,
       SourceMessage = rawMessage
   };
   ```

#### Matching Stage

**IMatcherSubRule Interface:**
```csharp
public interface IMatcherSubRule<TEntityType>
{
    MatchResult Match(
        Identifier identifier,
        AodbContext context);
}
```

**Matching Process:**
1. **Query Database**
   ```csharp
   var existingEntity = context.EntityValues
       .Where(e => e.EntityType == "flight_leg")
       .Where(e => e.Identifier == identifier)
       .FirstOrDefault();
   ```

2. **Evaluate Match**
   - **Single Match** → Return entity ID
   - **No Match** → Create new entity
   - **Multiple Matches** → Return error

3. **Handle Unmatched**
   ```csharp
   if (matchResult.IsUnmatched)
   {
       // Store in raw_messages table
       // Publish unmatched message status
       // Allow manual matching via UI
   }
   ```

**Match Error Types:**
- `NoMatchCreateNew` - Create new entity
- `NoMatchDontCreate` - Skip message
- `MultipleMatches` - Manual resolution needed

See: [[Mapping and Matching Rules]]

#### Unmatched Messages

**Flow:**
```
Unmatched Message
    ↓
Store in raw_messages table
    ↓
Publish to traffic-operational-adapter-messages
    ↓
QueryStream (for UI display)
    ↓
Manual Matching (via UI)
    ↓
Re-evaluate (put back on topic)
```

**Unmatched Reasons:**
- Insufficient identifier information
- Entity not yet created
- Timing issues (out-of-order messages)
- Data quality issues

### 6. AODB Internal Topics

**Purpose:** Internal message routing within AODB

**Topic Structure:**
```
traffic-operational-entityvalue-enrichment-{entity_type}
traffic-operational-entityvalue-recalculation-{entity_type}
```

**Examples:**
- `traffic-operational-entityvalue-enrichment-flight_leg`
- `traffic-operational-entityvalue-recalculation-visit`

**Partitioning:**
- 50 partitions per topic
- Partitioned by entity ID (consistent hashing)
- Ensures ordered processing per entity

**Message Format:**
```json
{
  "entityId": "12345",
  "entityType": "flight_leg",
  "operation": "update",
  "messageSource": "ExternalAdapter",
  "sourceTraceId": "abc-123",
  "requesterId": "matcher",
  "attributes": { ... }
}
```

### 7. AODB Pipeline Processing

See: [[AODB Processing Pipeline]]

**Stages:**
1. Enrichment - Add master data
2. Business Rules - Apply logic
3. Persistence - Save to database
4. Publishing - Output documents

### 8. Output

**Topics:**

| Topic | Consumer | Content |
|-------|----------|---------|
| `aodb-pipeline-output` | QueryStream | Updated entity documents |
| `traffic-operational-adapter-messages` | QueryStream | Adapter message status |
| `aodb-pipeline-dtd` | QueryStream | Schema definitions |

**Adapter Message Status:**
```json
{
  "id": "raw-message-guid",
  "messageType": "FlightLegAcdScr",
  "status": "processed",  // processed, unmatched, failed
  "entityId": "12345",
  "entityType": "flight_leg",
  "receivedTimestamp": "2025-12-15T10:30:00Z",
  "processedTimestamp": "2025-12-15T10:30:02Z",
  "traceId": "abc-123"
}
```

## Message Prioritization

### Priority Topics

**High Priority:**
- `traffic-operational-*` - Operational window messages
- Processed first

**Low Priority:**
- `traffic-planning-*` - Planning/seasonal data
- Processed after operational messages

### ACD Priority Split

**Purpose:** Handle large seasonal batches without blocking operational updates

**Topics:**
1. **Priority Topic** (`traffic-operational-flightlegacd`)
   - Flights in current season
   - Processed immediately

2. **Default Topic** (`traffic-planning-flightlegacd`)
   - Flights in future seasons
   - Batch processed

**Splitting Logic:**
```csharp
if (message.ScheduledDate >= currentSeason.StartDate &&
    message.ScheduledDate <= currentSeason.EndDate)
{
    // Publish to priority topic
}
else
{
    // Publish to default topic
}
```

## Error Handling

### Adapter Level

**Invalid Messages:**
```
Receive → Validate → FAIL
    ↓
Publish to inbound-invalid-{adapter}
    ↓
Log error
    ↓
Raise alarm
```

**Transient Errors:**
- Retry with exponential backoff
- Circuit breaker for external systems
- Fallback to cached data

### AODB Level

**Mapping Errors:**
- Skip message
- Log with trace ID
- Publish failed status

**Matching Errors:**
- Unmatched → Store for manual resolution
- Multiple matches → Store for manual resolution

**Processing Errors:**
- Retry 5 times (default)
- Dead letter topic for persistent errors

See: [[Error Handling Patterns]]

## Performance

### Throughput

**Target:**
- 10,000 messages/second (operational)
- 100,000 messages/second (batch)

**Optimization:**
- Parallel processing (50 partitions)
- Batch database operations
- Async I/O throughout

### Latency

**Target:**
- P50: < 100ms (receive to persist)
- P95: < 500ms
- P99: < 1s

**Measurement:**
- Receive timestamp (adapter)
- Process timestamp (AODB)
- Publish timestamp (output)

## Monitoring

### Metrics

**Adapter Metrics:**
- `adapter_messages_received_total`
- `adapter_messages_invalid_total`
- `adapter_processing_duration_seconds`

**AODB Metrics:**
- `aodb_messages_consumed_total`
- `aodb_messages_unmatched_total`
- `aodb_processing_duration_seconds`

**Kafka Metrics:**
- Consumer lag
- Partition offsets
- Rebalancing events

### Alerts

- High consumer lag (> 1000 messages)
- High error rate (> 5%)
- Slow processing (> 1s P99)
- Dead letter messages

## Trace Example

**End-to-End Message Trace:**

```
1. External System
   Time: 10:30:00.000
   TraceId: abc-123-def

2. Adapter Receive
   Time: 10:30:00.100
   Duration: +100ms

3. Kafka Publish
   Time: 10:30:00.150
   Duration: +50ms

4. AODB Consume
   Time: 10:30:00.200
   Duration: +50ms

5. Mapping & Matching
   Time: 10:30:00.250
   Duration: +50ms

6. Pipeline Processing
   Time: 10:30:00.500
   Duration: +250ms

7. Persistence
   Time: 10:30:00.600
   Duration: +100ms

8. Publish Output
   Time: 10:30:00.650
   Duration: +50ms

Total: 650ms (receive to output)
```

Query trace in QueryStream:
```
GET /api/odata/feeds/adapter_messages?$filter=trace_id eq 'abc-123-def'
  &$expand=mapped_entity,processing_events
```

## Related Notes

- [[Adapter System]] - Adapter development
- [[AODB Processing Pipeline]] - Pipeline details
- [[Mapping and Matching Rules]] - Rule configuration
- [[Kafka Messaging]] - Kafka infrastructure
- [[Error Handling Patterns]] - Error strategies

---

**Tags:** #data-flow #inbound #adapters #kafka #pipeline
**Created:** 2025-12-15
**References:**
- [Confluence: Inbound messaging: Adapters](https://dapproduct.atlassian.net/wiki/spaces/PD/pages/1415898)
- [Confluence: AODB Messaging](https://dapproduct.atlassian.net/wiki/spaces/PD/pages/1542177)
