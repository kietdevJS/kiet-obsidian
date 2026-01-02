# Kafka Messaging

> Message broker infrastructure and patterns in AIRHART

## Overview

Kafka is the **central nervous system** of AIRHART, enabling asynchronous, scalable, and fault-tolerant communication between all components.

## Why Kafka?

**Key Benefits:**
- **Decoupling:** Components don't need to know about each other
- **Scalability:** Horizontal scaling via partitions
- **Durability:** Messages persisted to disk
- **Ordering:** Per-partition message ordering
- **Replay:** Reprocess messages from any point
- **High Throughput:** Millions of messages/second

## Architecture

```
┌──────────────────────────────────────────────────────┐
│              Kafka Cluster                           │
│                                                      │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐     │
│  │  Broker 1  │  │  Broker 2  │  │  Broker 3  │     │
│  └────────────┘  └────────────┘  └────────────┘     │
│                                                      │
│  Topics:                                             │
│  ├─ traffic-operational-* (50 partitions)            │
│  ├─ traffic-planning-* (10 partitions)               │
│  ├─ aodb-pipeline-output (50 partitions)             │
│  └─ masterdata-* (10 partitions)                     │
└──────────────────────────────────────────────────────┘
        ↑                ↑                ↓
    Producers       (Kafka itself)    Consumers
    (Adapters,      (Replication)     (AODB, QS)
     AODB)
```

## Topic Structure

### Topic Naming Convention

**Pattern:** `{traffic_type}-{phase}-{entity}{source}`

**Components:**
- `traffic_type`: traffic, system, internal
- `phase`: operational, planning
- `entity`: flightleg, visit, operational event
- `source`: acd, sita, amadeus (optional)

**Examples:**
```
traffic-operational-flightlegacd
traffic-planning-flightlegacd
traffic-operational-visit
traffic-operational-operationalevent
system-internal-recalculation
```

### Topic Categories

#### 1. Interface Topics (External → AIRHART)

| Topic | Producer | Consumer | Partitions |
|-------|----------|----------|------------|
| `traffic-operational-flightlegacd` | ACD Adapter | AODB Pipeline | 50 |
| `traffic-planning-flightlegacd` | ACD Adapter | AODB Pipeline | 10 |
| `traffic-operational-flightlegsita` | SITA Adapter | AODB Pipeline | 50 |
| `traffic-operational-operationalevent` | Amadeus Adapter | AODB Pipeline | 10 |
| `inbound-raw-{adapter}` | Adapter | QueryStream | 5 |

**Purpose:**
- Boundary between adapters and core
- Schema validation point
- Message transformation complete
- Versioned DTOs

#### 2. Internal Topics (Within AODB)

| Topic | Producer | Consumer | Partitions |
|-------|----------|----------|------------|
| `traffic-operational-entityvalue-enrichment-flight_leg` | AODB Matcher | AODB Pipeline | 50 |
| `traffic-operational-entityvalue-enrichment-visit` | AODB Matcher | AODB Pipeline | 50 |
| `traffic-operational-entityvalue-recalculation-*` | AODB API, Services | AODB Pipeline | 50 |
| `traffic-operational-business-rules-updates` | AODB API | AODB Pipeline | 1 |

**Purpose:**
- Internal AODB communication
- Entity-specific routing
- Recalculation requests
- Configuration updates

#### 3. Output Topics (AIRHART → External/QS)

| Topic | Producer | Consumer | Partitions |
|-------|----------|----------|------------|
| `aodb-pipeline-output` | AODB Pipeline | QueryStream DB Manager | 50 |
| `traffic-operational-adapter-messages` | AODB Pipeline | QueryStream DB Manager | 50 |
| `aodb-pipeline-dtd` | AODB Pipeline | QueryStream DB Manager | 1 |
| `masterdata-data-update-topic` | Master Data | QueryStream DB Manager | 10 |
| `qs-internal-updates` | QS DB Manager | QS Subscription Manager | 50 |

**Purpose:**
- Entity update distribution
- Message status tracking
- Schema evolution
- Real-time streaming

#### 4. Invalid/Dead Letter Topics

| Topic | Producer | Purpose |
|-------|----------|---------|
| `inbound-invalid-{adapter}` | Adapter | Validation failures |
| `aodb-deadletter` | AODB Pipeline | Unrecoverable errors |
| `querystream-deadletter` | QueryStream | Processing failures |

## Message Structure

### Standard Message Format

```json
{
  "meta": {
    "messageId": "guid",
    "traceId": "abc-123-def",
    "timestamp": "2025-12-15T10:30:00.000Z",
    "source": "ACD",
    "messageType": "FlightLegAcdScr",
    "version": "1.0"
  },
  "payload": {
    // Message-specific data
  }
}
```

### Kafka Headers

**Standard Headers:**
```
traceparent: 00-abc123def-456789-01  (W3C trace context)
content-type: application/json
message-type: FlightLegAcdScr
message-version: 1.0
correlation-id: request-guid
```

**Usage:**
- `traceparent`: Distributed tracing
- `content-type`: Serialization format
- `message-type`: DTO type for deserialization
- `correlation-id`: Request tracking

### Message Keys

**Purpose:** Determine partition and ordering

**Key Strategies:**

1. **Entity ID** (most common)
   ```csharp
   key = entityId  // e.g., "flight-12345"
   ```
   - Ensures all messages for same entity go to same partition
   - Guarantees ordering per entity

2. **Composite Key**
   ```csharp
   key = $"{operatorIata}_{flightNumber}_{date}"
   ```
   - Used when entity ID not yet known
   - Consistent routing for related messages

3. **Null Key**
   ```csharp
   key = null
   ```
   - Round-robin partition assignment
   - For independent messages

## Partitioning

### Why Partition?

**Benefits:**
- **Parallelism:** Multiple consumers process concurrently
- **Scalability:** Add partitions to increase throughput
- **Ordering:** Guaranteed within partition
- **Isolation:** Failures in one partition don't affect others

### Partition Strategy

**Operational Topics:** 50 partitions
- High throughput requirement
- Parallel processing across many pods
- Balance between ordering and scale

**Planning Topics:** 10 partitions
- Lower throughput
- Still benefits from parallelism
- Fewer consumer instances needed

**DTD/Schema Topics:** 1 partition
- Low volume
- Total ordering required
- Single consumer sufficient

### Partition Assignment

**Kafka Default:**
```
partition = hash(key) % partition_count
```

**Example:**
```csharp
key = "flight-12345"
hash = 0x7A3B2C1D
partition_count = 50
partition = 0x7A3B2C1D % 50 = 29

// All messages for flight-12345 go to partition 29
```

**Ordering Guarantee:**
- Messages with same key → same partition → ordered
- Messages in same partition → ordered by offset
- No ordering across partitions

## Consumer Groups

### What is a Consumer Group?

**Definition:** Set of consumers that coordinate to consume from topic

**Benefits:**
- **Load Balancing:** Partitions distributed across consumers
- **Failover:** Partition reassigned if consumer fails
- **Scalability:** Add consumers to increase processing

### Consumer Group Strategy

**AODB Pipeline:**
```
Consumer Group: "aodb-pipeline"
Topic: "traffic-operational-flightlegacd"
Partitions: 50
Consumer Instances: 10-50 (auto-scaled)

Partition Assignment:
  Instance 1: [0, 1, 2, 3, 4]
  Instance 2: [5, 6, 7, 8, 9]
  ...
  Instance 10: [45, 46, 47, 48, 49]
```

**QueryStream DB Manager:**
```
Consumer Group: "querystream-dbmanager"
Topic: "aodb-pipeline-output"
Partitions: 50
Consumer Instances: 10-50 (auto-scaled)
```

**Rule:** Max consumers = partition count
- More consumers than partitions → idle consumers
- Fewer consumers than partitions → multiple partitions per consumer

### Offset Management

**Offset:** Position in partition

**Commit Strategies:**

1. **Auto-Commit (default)**
   ```csharp
   enable.auto.commit = true
   auto.commit.interval.ms = 5000
   ```
   - Commits offsets every 5 seconds
   - Risk: Message reprocessing on failure
   - Simple but not guaranteed

2. **Manual Commit (recommended)**
   ```csharp
   enable.auto.commit = false
   
   // After successful processing
   consumer.Commit();
   ```
   - Commit only after successful processing
   - At-least-once delivery guarantee
   - Idempotent processing required

3. **Transactional Commit (future)**
   ```csharp
   // Commit offset + publish output in transaction
   // Exactly-once semantics
   ```

## Message Delivery Semantics

### At-Least-Once (Current)

**Guarantee:** Message processed at least once, possibly more

**Implementation:**
```csharp
while (running)
{
    var message = await consumer.ConsumeAsync();
    
    try
    {
        await ProcessMessageAsync(message);
        await consumer.CommitAsync(message);
    }
    catch (Exception)
    {
        // Message not committed
        // Will be reprocessed
    }
}
```

**Requirements:**
- Idempotent message processing
- Duplicate detection
- Consistent state after replay

**Example Idempotent Processing:**
```csharp
// Check if message already processed
if (await IsAlreadyProcessed(message.MessageId))
{
    return; // Skip duplicate
}

// Process message
await ProcessAsync(message);

// Mark as processed
await MarkProcessed(message.MessageId);
```

### Exactly-Once (Future)

**Guarantee:** Message processed exactly once

**Implementation:** Kafka transactions
- Atomic: Consume + Process + Commit + Publish
- Requires Kafka 0.11+
- Performance overhead

## Performance

### Throughput

**Targets:**
- Operational topics: 10,000 messages/second
- Batch topics: 100,000 messages/second

**Optimization:**
- **Compression:** Snappy/LZ4 compression
- **Batching:** Batch multiple messages
- **Async I/O:** Non-blocking operations
- **Partitioning:** Parallel processing

### Latency

**P50:** < 10ms (produce + consume)
**P99:** < 100ms

**Factors:**
- Network latency
- Disk I/O (Kafka persistence)
- Replication lag
- Consumer processing time

## Monitoring

### Producer Metrics

```
kafka_producer_request_latency_ms
kafka_producer_batch_size_avg
kafka_producer_record_send_rate
kafka_producer_record_error_rate
```

### Consumer Metrics

```
kafka_consumer_fetch_latency_ms
kafka_consumer_records_consumed_rate
kafka_consumer_lag
kafka_consumer_commit_latency_ms
```

### Topic Metrics

```
kafka_topic_partition_count
kafka_topic_replication_factor
kafka_topic_log_size_bytes
kafka_topic_messages_in_per_sec
```

### Critical Alerts

- **High Consumer Lag** (> 1000 messages)
  - Consumers falling behind producers
  - Scale up consumers

- **Low Replication** (< 3 replicas)
  - Risk of data loss
  - Check broker health

- **High Error Rate** (> 5%)
  - Processing failures
  - Check application logs

## Configuration

### Producer Config

```json
{
  "bootstrap.servers": "kafka:9092",
  "acks": "all",
  "compression.type": "snappy",
  "batch.size": 16384,
  "linger.ms": 10,
  "max.in.flight.requests.per.connection": 5,
  "enable.idempotence": true
}
```

**Key Settings:**
- `acks=all`: Wait for all replicas (durability)
- `enable.idempotence`: Prevent duplicates on retry
- `compression.type`: Reduce network/disk usage
- `linger.ms`: Batch messages for throughput

### Consumer Config

```json
{
  "bootstrap.servers": "kafka:9092",
  "group.id": "aodb-pipeline",
  "enable.auto.commit": false,
  "max.poll.records": 500,
  "session.timeout.ms": 30000,
  "max.poll.interval.ms": 300000
}
```

**Key Settings:**
- `enable.auto.commit=false`: Manual offset control
- `max.poll.records`: Batch size per poll
- `session.timeout.ms`: Consumer liveness detection
- `max.poll.interval.ms`: Max processing time

### Topic Config

```json
{
  "retention.ms": 604800000,  // 7 days
  "cleanup.policy": "delete",
  "compression.type": "snappy",
  "min.insync.replicas": 2,
  "replication.factor": 3
}
```

**Key Settings:**
- `retention.ms`: How long to keep messages
- `replication.factor`: Copies across brokers
- `min.insync.replicas`: Minimum replicas for ack
- `compression.type`: Storage optimization

## Patterns & Best Practices

### 1. Guaranteed Message Delivery

**Outbox Pattern:**
```csharp
using (var transaction = dbContext.Database.BeginTransaction())
{
    // Save entity
    dbContext.Entities.Add(entity);
    
    // Save outbox message
    dbContext.OutboxMessages.Add(new OutboxMessage
    {
        Topic = "aodb-pipeline-output",
        Key = entity.Id,
        Payload = JsonSerializer.Serialize(entity)
    });
    
    await dbContext.SaveChangesAsync();
    transaction.Commit();
}

// Background service publishes from outbox
while (true)
{
    var messages = await dbContext.OutboxMessages.Take(100).ToListAsync();
    
    foreach (var message in messages)
    {
        await kafkaProducer.ProduceAsync(message.Topic, message.Payload);
        dbContext.OutboxMessages.Remove(message);
    }
    
    await dbContext.SaveChangesAsync();
}
```

### 2. Message Deduplication

**Idempotency Key:**
```csharp
var idempotencyKey = $"{message.MessageId}_{message.Version}";

if (await cache.ContainsKeyAsync(idempotencyKey))
{
    return; // Already processed
}

await ProcessMessageAsync(message);

await cache.SetAsync(idempotencyKey, true, TimeSpan.FromDays(7));
```

### 3. Dead Letter Queue

**Pattern:**
```csharp
try
{
    await ProcessMessageAsync(message);
}
catch (Exception ex)
{
    if (attempt < maxRetries)
    {
        // Retry
        await RetryAsync(message, attempt + 1);
    }
    else
    {
        // Move to dead letter
        await kafkaProducer.ProduceAsync(
            topic: "aodb-deadletter",
            key: message.Key,
            value: new DeadLetterMessage
            {
                OriginalTopic = message.Topic,
                OriginalMessage = message,
                Error = ex.ToString(),
                Attempts = attempt
            }
        );
    }
}
```

### 4. Message Versioning

**Schema Evolution:**
```csharp
// V1 Message
public class FlightLegV1
{
    public string FlightNumber { get; set; }
}

// V2 Message (backward compatible)
public class FlightLegV2
{
    public string FlightNumber { get; set; }
    public string OperatorIata { get; set; } = "SK"; // Default
}

// Deserialize with version check
var version = message.Headers["message-version"];
var flightLeg = version == "2.0" 
    ? Deserialize<FlightLegV2>(message)
    : Deserialize<FlightLegV1>(message);
```

## Related Notes

- [[AODB Messaging]] - AODB-specific topics
- [[Inbound Message Flow]] - Message ingestion
- [[Outbound Message Flow]] - Message distribution
- [[Message Types]] - DTO definitions

---

**Tags:** #kafka #messaging #architecture #infrastructure
**Created:** 2025-12-15
