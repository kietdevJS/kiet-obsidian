# Adapter System

> Integration framework for external system connectivity using message-based patterns

## Overview

The **Adapter System** provides a standardized framework for integrating external systems with AIRHART. Adapters act as translation layers, converting between external formats (AIDX, SITA, custom APIs) and AIRHART's internal message formats.

## Architecture

```
External Systems
      ↓
┌─────────────────┐
│  Adapter Layer  │  ← Translation & Validation
├─────────────────┤
│ • Endpoint      │  ← Receive/Send messages
│ • Processor     │  ← Transform & Validate
│ • Error Handler │  ← Resilience & Retry
└─────────────────┘
      ↓
   Kafka Topics
      ↓
   AODB Pipeline
```

## Adapter Types

### 1. Inbound Adapters

**Purpose:** Receive messages from external systems and publish to AIRHART

**Pattern: EVENT (Push)**
- External system pushes messages to adapter endpoint
- Adapter validates and transforms immediately
- Examples: Amadeus AMS Flight Updates, SITA MVT/MVA

**Pattern: SCHEDULED (Pull)**
- Adapter polls external system on schedule
- Fetches new/updated data periodically
- Examples: Amadeus AMS Reference Data batch

**Pattern: REQUEST (On-Demand)**
- AIRHART requests specific data from external system
- Synchronous or asynchronous response
- Examples: Manual data imports, API queries

### 2. Outbound Adapters

**Purpose:** Publish AIRHART data to external systems

**Pattern: EVENT (Reactive)**
- Subscribe to AODB output topics
- Transform and send on every update
- Examples: Amadeus AIDX Flight Updates, Display Systems

**Pattern: SCHEDULED (Batch)**
- Collect changes over time window
- Send batch updates on schedule
- Examples: EDW Data Warehouse, Reporting Systems

**Pattern: PULL (On-Demand)**
- External system requests data from adapter
- Adapter queries QueryStream and responds
- Examples: REST APIs, Web Services

## Generic Adapter Library

AIRHART provides a **Generic Adapter Library** (`SA.Library.GenericAdapter`) that implements common adapter patterns.

### Core Components

**1. Endpoint Component**
- HTTP/HTTPS listener for inbound messages
- HTTP client for outbound messages
- Authentication and authorization
- Rate limiting and throttling

**2. Processor Component**
- Message parsing (JSON, XML, CSV, etc.)
- Schema validation
- Data transformation/mapping
- Business logic execution

**3. Message Router**
- Content-based routing
- Topic selection based on message type
- Filtering and enrichment

**4. Error Handler**
- Dead letter queue management
- Retry with exponential backoff
- Error notification and logging

### Adapter Pipeline

```csharp
// Simplified Generic Adapter pipeline
Message → Receive → Validate → Transform → Route → Publish → Commit

┌──────────┐   ┌──────────┐   ┌───────────┐   ┌──────┐   ┌─────────┐
│ External │ → │ Endpoint │ → │ Processor │ → │ Kafka│ → │  AODB   │
│  System  │   │  (HTTP)  │   │  (Logic)  │   │Topics│   │Pipeline │
└──────────┘   └──────────┘   └───────────┘   └──────┘   └─────────┘
                     ↓              ↓
                 ┌─────────┐   ┌──────────┐
                 │ Auth &  │   │Validation│
                 │ Logging │   │& Mapping │
                 └─────────┘   └──────────┘
```

## Kafka Topics Pattern

Each adapter uses a standardized set of topics:

### Inbound Adapter Topics

1. **inbound-raw-{adapter}**
   - Raw messages as received from external system
   - No transformation applied
   - Used for auditing and debugging

2. **inbound-invalid-{adapter}**
   - Messages that failed validation
   - Includes validation errors
   - Monitored for data quality issues

3. **traffic-operational-{adapter}**
   - Valid operational messages (flights in ±72h window)
   - Consumed by AODB Pipeline immediately
   - High priority processing

4. **traffic-planning-{adapter}**
   - Valid planning messages (flights beyond operational window)
   - Consumed by AODB with lower priority
   - Batch processing acceptable

5. **{adapter}-health**
   - Adapter health status and metrics
   - Connection status to external system
   - Processing statistics

### Outbound Adapter Topics

1. **aodb-output-flightleg** / **aodb-output-groundleg**
   - AODB publishes processed entities
   - Adapter subscribes and filters

2. **outbound-{adapter}-pending**
   - Messages queued for delivery
   - Awaiting external system acknowledgment

3. **outbound-{adapter}-sent**
   - Successfully delivered messages
   - Includes delivery timestamp and correlation ID

4. **outbound-{adapter}-failed**
   - Failed delivery attempts
   - Includes error details and retry count

## Message Transformation

### Mapping Configuration

Adapters use declarative mapping configurations:

```json
{
  "mappingName": "AIDX-to-FlightLeg",
  "sourceFormat": "XML",
  "targetFormat": "JSON",
  "mappings": [
    {
      "source": "//IATA_AIDX_FlightLegRQ/FlightLeg/LegIdentifier/AirlineDesignator/@CodeContext",
      "target": "airlineIataCode",
      "transform": "toUpperCase",
      "required": true
    },
    {
      "source": "//FlightLeg/LegData/RevisedLegTimes/ScheduledTime",
      "target": "scheduledTimeOfDeparture",
      "transform": "parseDateTime",
      "required": true
    }
  ],
  "conditionalMappings": [
    {
      "condition": "messageType == 'Arrival'",
      "mappings": [
        { "source": "//ArrivalAirport", "target": "arrivalAirportIataCode" }
      ]
    }
  ]
}
```

### Transformation Functions

Common transformations available in Generic Adapter:

- **String:** `toUpperCase`, `toLowerCase`, `trim`, `substring`
- **Date/Time:** `parseDateTime`, `formatDateTime`, `addHours`
- **Numeric:** `parseInt`, `parseDouble`, `round`
- **Lookup:** `lookupMasterData`, `lookupMapping`
- **Conditional:** `ifEmpty`, `coalesce`, `switch`

### Example Transformation Code

```csharp
public class AidxToFlightLegMapper : IMessageMapper<AidxMessage, FlightLeg>
{
    private readonly IMasterDataClient _masterData;

    public FlightLeg Map(AidxMessage aidx)
    {
        return new FlightLeg
        {
            // Direct mapping
            AirlineIataCode = aidx.AirlineDesignator?.ToUpper(),
            FlightNumber = aidx.FlightNumber,
            
            // DateTime parsing
            ScheduledTimeOfDeparture = ParseAidxDateTime(
                aidx.ScheduledTime, 
                aidx.DepartureAirport
            ),
            
            // Master Data lookup
            AircraftType = _masterData.GetAircraftType(
                aidx.AircraftRegistration, 
                aidx.ScheduledTime
            )?.IcaoCode,
            
            // Conditional mapping
            ActualTimeOfDeparture = aidx.EstimatedTime != null 
                ? ParseAidxDateTime(aidx.EstimatedTime, aidx.DepartureAirport)
                : null,
            
            // Complex transformation
            StandCode = DetermineStand(aidx.GateIdentifier, aidx.Parking Position)
        };
    }
}
```

## Validation

### Schema Validation

Messages validated against schemas:

```csharp
// JSON Schema validation
public class MessageValidator
{
    private readonly JSchema _schema;

    public ValidationResult Validate(string message)
    {
        try
        {
            var json = JObject.Parse(message);
            var isValid = json.IsValid(_schema, out IList<string> errors);
            
            return new ValidationResult
            {
                IsValid = isValid,
                Errors = errors.ToList()
            };
        }
        catch (JsonException ex)
        {
            return ValidationResult.Failure($"Invalid JSON: {ex.Message}");
        }
    }
}
```

### Business Rules Validation

Beyond schema, adapters validate business rules:

```csharp
public class BusinessRuleValidator
{
    public ValidationResult Validate(FlightLeg flight)
    {
        var errors = new List<string>();

        // Validate time sequence
        if (flight.ActualTimeOfDeparture < flight.ScheduledTimeOfDeparture.AddHours(-24))
        {
            errors.Add("Actual departure more than 24 hours before scheduled");
        }

        // Validate airport codes
        if (!IsValidIataCode(flight.DepartureAirportIataCode))
        {
            errors.Add($"Invalid airport code: {flight.DepartureAirportIataCode}");
        }

        // Validate flight number format
        if (!Regex.IsMatch(flight.FlightNumber, @"^\d{1,4}[A-Z]?$"))
        {
            errors.Add($"Invalid flight number format: {flight.FlightNumber}");
        }

        return new ValidationResult
        {
            IsValid = !errors.Any(),
            Errors = errors
        };
    }
}
```

## Error Handling & Resilience

### Retry Strategy

```csharp
public class RetryPolicy
{
    public async Task<T> ExecuteWithRetry<T>(
        Func<Task<T>> operation,
        int maxRetries = 3,
        TimeSpan baseDelay = default)
    {
        baseDelay = baseDelay == default ? TimeSpan.FromSeconds(1) : baseDelay;

        for (int attempt = 0; attempt <= maxRetries; attempt++)
        {
            try
            {
                return await operation();
            }
            catch (Exception ex) when (attempt < maxRetries && IsTransient(ex))
            {
                var delay = TimeSpan.FromSeconds(
                    baseDelay.TotalSeconds * Math.Pow(2, attempt) // Exponential backoff
                );
                
                _logger.LogWarning(
                    "Attempt {Attempt} failed, retrying in {Delay}ms: {Error}",
                    attempt + 1, delay.TotalMilliseconds, ex.Message
                );
                
                await Task.Delay(delay);
            }
        }
        
        throw new MaxRetriesExceededException($"Failed after {maxRetries} attempts");
    }

    private bool IsTransient(Exception ex)
    {
        return ex is HttpRequestException ||
               ex is TimeoutException ||
               ex is TaskCanceledException;
    }
}
```

### Dead Letter Queue

Failed messages sent to DLQ for manual intervention:

```json
{
  "originalMessage": "...",
  "originalTopic": "traffic-operational-amadeus",
  "errorType": "ValidationException",
  "errorMessage": "Missing required field: airlineIataCode",
  "attemptCount": 3,
  "firstAttempt": "2025-12-15T10:30:00Z",
  "lastAttempt": "2025-12-15T10:32:15Z",
  "stackTrace": "..."
}
```

## Common Adapter Implementations

### 1. Amadeus AMS Adapters

**Inbound Flight Updates** (`cph-adapter-amadeus-aidx`)
- Receives AIDX XML messages via HTTPS
- Validates against AIDX 21.3 schema
- Maps to FlightLeg/GroundLeg entities
- Publishes to operational/planning topics

**Inbound Reference Data** (`cph-adapter-amadeus-referencedata`)
- Polls Amadeus API for Master Data updates
- Scheduled execution (every 5 minutes)
- Maps Gate, Stand, Counter, Belt entities
- Publishes INOP and SwingGate events

**Outbound Flight Updates** (`cph-adapter-amadeus-aidx-outbound`)
- Subscribes to AODB output topics
- Filters by airline (configurable)
- Transforms to AIDX format
- POSTs to Amadeus API

### 2. SITA Adapter

**SMTP-based messaging**
- Receives MVT, MVA, LDM messages via email
- Parses SITA message format
- Maps to standardized FlightLeg
- Supports historical message replay

### 3. EDW Adapter

**Data Warehouse export**
- Subscribes to all AODB output topics
- Writes JSON files to Azure Blob Storage
- Batches messages (100 per file)
- File naming: `{datetime}-{guid}.json`

### 4. FOS (Flight Operations System) Adapter

**Bidirectional integration**
- Inbound: Crew assignments, maintenance status
- Outbound: Flight schedule and status
- Custom JSON format
- API-based communication

## Configuration

### appsettings.json

```json
{
  "Adapter": {
    "Name": "amadeus-aidx-inbound",
    "Type": "Inbound",
    "Pattern": "Event",
    "Enabled": true
  },
  "Endpoint": {
    "Protocol": "HTTPS",
    "Port": 8443,
    "Path": "/api/aidx",
    "Authentication": {
      "Type": "Certificate",
      "ValidateClientCertificate": true
    }
  },
  "Kafka": {
    "BootstrapServers": "kafka:9092",
    "Topics": {
      "Raw": "inbound-raw-amadeus",
      "Invalid": "inbound-invalid-amadeus",
      "Operational": "traffic-operational-amadeus",
      "Planning": "traffic-planning-amadeus"
    },
    "ProducerConfig": {
      "Acks": "all",
      "MaxInFlight": 5,
      "CompressionType": "gzip"
    }
  },
  "Validation": {
    "SchemaPath": "/schemas/aidx-21.3.xsd",
    "ValidateOnReceive": true
  },
  "Transformation": {
    "MappingConfigPath": "/config/aidx-mapping.json",
    "DefaultTimeZone": "UTC"
  },
  "Resilience": {
    "MaxRetries": 3,
    "RetryDelaySeconds": 1,
    "DeadLetterTopic": "dlq-amadeus"
  }
}
```

## Monitoring & Metrics

### Key Metrics

```
adapter.messages.received{adapter=amadeus} - Messages received
adapter.messages.valid{adapter=amadeus} - Successfully validated
adapter.messages.invalid{adapter=amadeus} - Validation failures
adapter.messages.published{adapter=amadeus,topic=operational} - Published to topics
adapter.processing.duration.ms{adapter=amadeus} - Processing time
adapter.external.api.duration.ms{adapter=amadeus,endpoint=/aidx} - External API latency
```

### Health Checks

```http
GET /health

{
  "status": "Healthy",
  "components": {
    "endpoint": {
      "status": "Healthy",
      "description": "Listening on port 8443"
    },
    "kafka": {
      "status": "Healthy",
      "description": "Connected to 3 brokers"
    },
    "externalSystem": {
      "status": "Degraded",
      "description": "Last successful call 2 minutes ago",
      "lastError": "Timeout after 30s"
    }
  },
  "timestamp": "2025-12-15T10:45:00Z"
}
```

### Alerts

- **Message validation failure rate > 5%** - Data quality issue
- **Processing lag > 30 seconds** - Performance degradation
- **External system unavailable > 5 minutes** - Integration failure
- **Dead letter queue size > 100** - Requires manual intervention

## Development Guide

### Creating a New Adapter

1. **Define Requirements**
   - External system protocol (REST, SOAP, FTP, etc.)
   - Message format (JSON, XML, CSV)
   - Frequency (real-time, scheduled, on-demand)
   - Error handling requirements

2. **Choose Base Pattern**
   ```bash
   # Use Generic Adapter template
   dotnet new genericadapter -n MyNewAdapter
   ```

3. **Implement Mapping**
   ```csharp
   public class ExternalToFlightLegMapper : IMessageMapper
   {
       public FlightLeg Map(ExternalMessage external)
       {
           // Implement transformation logic
       }
   }
   ```

4. **Configure Topics**
   - Add topic definitions to configuration
   - Set up permissions in Kafka
   - Configure consumers in AODB

5. **Test**
   - Unit tests for mapping logic
   - Integration tests with mock external system
   - End-to-end tests with actual messages

6. **Deploy**
   - Package as Docker container
   - Deploy to Kubernetes
   - Configure monitoring and alerts

### Testing Adapters

```csharp
[Fact]
public async Task Should_Map_AIDX_To_FlightLeg()
{
    // Arrange
    var aidxMessage = LoadTestMessage("samples/aidx-arrival.xml");
    var mapper = new AidxToFlightLegMapper();

    // Act
    var flightLeg = mapper.Map(aidxMessage);

    // Assert
    Assert.Equal("SK", flightLeg.AirlineIataCode);
    Assert.Equal("1234", flightLeg.FlightNumber);
    Assert.Equal("CPH", flightLeg.ArrivalAirportIataCode);
}

[Fact]
public async Task Should_Reject_Invalid_Message()
{
    // Arrange
    var invalidMessage = "<Invalid>XML</Invalid>";
    var validator = new AidxValidator();

    // Act
    var result = validator.Validate(invalidMessage);

    // Assert
    Assert.False(result.IsValid);
    Assert.Contains("Missing required element", result.Errors);
}
```

## Best Practices

### Performance

1. **Batch Processing**
   - Group small messages for efficiency
   - Use Kafka batching configuration
   - Balance latency vs throughput

2. **Connection Pooling**
   - Reuse HTTP connections
   - Configure connection limits
   - Monitor connection exhaustion

3. **Async Processing**
   - Use async/await throughout
   - Avoid blocking calls
   - Configure thread pools appropriately

### Reliability

1. **Idempotency**
   - Include correlation ID in messages
   - Deduplicate on receiver side
   - Use Kafka's exactly-once semantics

2. **Error Handling**
   - Distinguish transient from permanent errors
   - Implement circuit breaker for external calls
   - Log all errors with context

3. **Monitoring**
   - Track end-to-end latency
   - Monitor external system health
   - Alert on anomalies

### Security

1. **Authentication**
   - Use mutual TLS for inbound adapters
   - Store credentials in Key Vault
   - Rotate certificates regularly

2. **Authorization**
   - Validate sender identity
   - Implement rate limiting
   - Log all access attempts

3. **Data Protection**
   - Encrypt sensitive data in topics
   - Mask PII in logs
   - Comply with GDPR requirements

## Related Notes

- [[AODB Processing Pipeline]] - How adapter messages are processed
- [[Inbound Message Flow]] - Detailed message flow from adapters
- [[Kafka Messaging]] - Topic structure and configuration
- [[Mapping and Matching Rules]] - Business rules for message transformation
- [[Error Handling Patterns]] - Error handling strategies

---

**Tags:** #adapters #integration #messaging #kafka #transformation
**Created:** 2025-12-15
**References:**
- [Confluence: General Inbound Adapter Pattern](https://dapproduct.atlassian.net/wiki/spaces/PD/pages/96174267)
- [Confluence: General Outbound Adapter Pattern](https://dapproduct.atlassian.net/wiki/spaces/PD/pages/122495436)
- Codebase: `SA.Library.GenericAdapter/`, `CPH.Component.GenericAdapter/`
