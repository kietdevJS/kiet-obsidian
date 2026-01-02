# Message Types

> Data Transfer Objects (DTOs) and message contracts that define communication between AIRHART components and external systems

## Overview

**Message Types** are the data contracts used throughout AIRHART for communication between:
- External systems and adapters
- Adapters and AODB
- AODB and Kafka
- Kafka and QueryStream
- QueryStream and clients

Each message type is versioned, validated, and designed for specific purposes in the data flow pipeline.

## Architecture

```
External System → Adapter Message → AODB DTO → Domain Event → QueryStream DTO → Client DTO
                  (incoming)        (internal)    (Kafka)      (projection)    (API)

Example: MVT Message Flow
┌─────────────────────────────────────────────────────────────────────────┐
│ 1. MVT Message (IATA format)                                            │
│    "MVT\nSK/123\nEKCH/ENGM\nAA01/0700"                                  │
└────────────────┬────────────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ 2. MvtMessage (Adapter DTO)                                             │
│    {                                                                     │
│      "airline": "SK",                                                    │
│      "flightNumber": "123",                                              │
│      "actualInBlockTime": "2025-01-01T07:00:00Z"                        │
│    }                                                                     │
└────────────────┬────────────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ 3. UpdateFlightLegCommand (AODB Command)                                │
│    {                                                                     │
│      "flightLegId": "guid",                                              │
│      "actualInBlockTime": "2025-01-01T07:00:00Z",                       │
│      "source": "SITA-MVT"                                                │
│    }                                                                     │
└────────────────┬────────────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ 4. FlightLegUpdated (Domain Event on Kafka)                             │
│    {                                                                     │
│      "eventId": "guid",                                                  │
│      "eventType": "FlightLegUpdated",                                    │
│      "flightLegId": "guid",                                              │
│      "changes": {                                                        │
│        "actualInBlockTime": {                                            │
│          "oldValue": null,                                               │
│          "newValue": "2025-01-01T07:00:00Z"                              │
│        }                                                                 │
│      }                                                                   │
│    }                                                                     │
└────────────────┬────────────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ 5. FlightLegDocument (QueryStream Document)                             │
│    {                                                                     │
│      "id": "guid",                                                       │
│      "flightNumber": "SK123",                                            │
│      "actualInBlockTime": "2025-01-01T07:00:00Z",                       │
│      "status": "Arrived",                                                │
│      "version": 15                                                       │
│    }                                                                     │
└────────────────┬────────────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ 6. FlightLegDTO (Client API Response)                                   │
│    {                                                                     │
│      "id": "guid",                                                       │
│      "flight": "SK123",                                                  │
│      "actualArrival": "2025-01-01T07:00:00Z",                           │
│      "status": "Arrived"                                                 │
│    }                                                                     │
└─────────────────────────────────────────────────────────────────────────┘
```

## Categories of Message Types

### 1. Adapter Messages (Incoming)

Messages received from external systems:

#### MVT Message (Movement Message)

```csharp
public class MvtMessage : IAdapterMessage
{
    public string MessageType => "MVT";
    
    // Identification
    public string Airline { get; set; }
    public string FlightNumber { get; set; }
    public string Suffix { get; set; }
    public DateTime OperationalDate { get; set; }
    
    // Times
    public DateTime? ActualInBlockTime { get; set; }
    public DateTime? ActualOffBlockTime { get; set; }
    public DateTime? ActualTakeOffTime { get; set; }
    public DateTime? ActualLandingTime { get; set; }
    
    // Aircraft
    public string Registration { get; set; }
    public string AircraftType { get; set; }
    
    // Route
    public string DepartureStation { get; set; }
    public string ArrivalStation { get; set; }
    
    // Metadata
    public string Source { get; set; }
    public DateTime ReceivedAt { get; set; }
}
```

#### SCR Message (Schedule Message)

```csharp
public class ScrMessage : IAdapterMessage
{
    public string MessageType => "SCR";
    
    // Flight identification
    public string Airline { get; set; }
    public string FlightNumber { get; set; }
    public DateTime ScheduledInBlockTime { get; set; }
    public DateTime ScheduledOffBlockTime { get; set; }
    
    // Route
    public string DepartureStationIata { get; set; }
    public string ArrivalStationIata { get; set; }
    
    // Aircraft
    public string AircraftTypeIata { get; set; }
    
    // Service
    public string ServiceType { get; set; }
    
    // Linking (for visits)
    public string LinkedFlightNumber { get; set; }
    public DateTime? LinkedScheduledTime { get; set; }
    
    // Slot
    public string SlotCoordinatorMessageId { get; set; }
}
```

#### FPL Message (Flight Plan)

```csharp
public class FplMessage : IAdapterMessage
{
    public string MessageType => "FPL";
    
    // ICAO Flight Plan fields
    public string AircraftIdentification { get; set; }
    public string FlightRules { get; set; }
    public string TypeOfFlight { get; set; }
    
    // Aircraft
    public string AircraftType { get; set; }
    public string AircraftRegistration { get; set; }
    
    // Route
    public string DepartureAerodrome { get; set; }
    public DateTime EstimatedOffBlockTime { get; set; }
    public string DestinationAerodrome { get; set; }
    public TimeSpan EstimatedElapsedTime { get; set; }
    
    // Routing
    public string Route { get; set; }
    public string AlternateAerodromes { get; set; }
    
    // Other info
    public string Remarks { get; set; }
}
```

### 2. AODB Commands (Internal)

Commands that trigger actions in AODB:

```csharp
public abstract class Command
{
    public Guid CommandId { get; set; } = Guid.NewGuid();
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;
    public string User { get; set; }
    public string Source { get; set; }
    public Guid? CorrelationId { get; set; }
}

public class UpdateFlightLegCommand : Command
{
    public Guid FlightLegId { get; set; }
    
    // Fields to update (null = no change)
    public DateTime? ActualInBlockTime { get; set; }
    public DateTime? ActualOffBlockTime { get; set; }
    public DateTime? EstimatedInBlockTime { get; set; }
    public DateTime? EstimatedOffBlockTime { get; set; }
    
    public string Registration { get; set; }
    public string AircraftType { get; set; }
    
    public string ArrivalStand { get; set; }
    public string DepartureStand { get; set; }
    public string ArrivalGate { get; set; }
    public string DepartureGate { get; set; }
    
    public FlightStatus? Status { get; set; }
    
    // Source tracking
    public string MessageSource { get; set; }
    public string MessageType { get; set; }
}

public class CreateFlightLegCommand : Command
{
    public string Airline { get; set; }
    public string FlightNumber { get; set; }
    public string Suffix { get; set; }
    public DateTime OperationalDate { get; set; }
    public Direction Direction { get; set; }
    
    public DateTime ScheduledTime { get; set; }
    public string DepartureStation { get; set; }
    public string ArrivalStation { get; set; }
    
    public string AircraftType { get; set; }
    public string Registration { get; set; }
}

public class LinkFlightsCommand : Command
{
    public Guid ArrivalFlightId { get; set; }
    public Guid DepartureFlightId { get; set; }
    public string LinkType { get; set; } // "Hard", "Soft", "Manual"
    public string Reason { get; set; }
}
```

### 3. Domain Events (Kafka)

Events published when state changes:

```csharp
public abstract class DomainEvent
{
    public Guid EventId { get; set; } = Guid.NewGuid();
    public string EventType { get; set; }
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;
    public int Version { get; set; }
    public Guid? CorrelationId { get; set; }
    public string CausedBy { get; set; } // Command that caused this event
}

public class FlightLegCreated : DomainEvent
{
    public FlightLegCreated()
    {
        EventType = nameof(FlightLegCreated);
    }
    
    public Guid FlightLegId { get; set; }
    public string FlightNumber { get; set; }
    public string Airline { get; set; }
    public Direction Direction { get; set; }
    public DateTime ScheduledTime { get; set; }
    public string Route { get; set; }
}

public class FlightLegUpdated : DomainEvent
{
    public FlightLegUpdated()
    {
        EventType = nameof(FlightLegUpdated);
    }
    
    public Guid FlightLegId { get; set; }
    
    // Changed fields with old/new values
    public Dictionary<string, FieldChange> Changes { get; set; }
}

public class FieldChange
{
    public object OldValue { get; set; }
    public object NewValue { get; set; }
    public DateTime ChangedAt { get; set; }
    public string ChangedBy { get; set; }
}

public class FlightLegDeleted : DomainEvent
{
    public FlightLegDeleted()
    {
        EventType = nameof(FlightLegDeleted);
    }
    
    public Guid FlightLegId { get; set; }
    public string Reason { get; set; }
}

public class VisitLinked : DomainEvent
{
    public VisitLinked()
    {
        EventType = nameof(VisitLinked);
    }
    
    public Guid VisitId { get; set; }
    public Guid ArrivalFlightId { get; set; }
    public Guid DepartureFlightId { get; set; }
    public string LinkType { get; set; }
}
```

### 4. QueryStream DTOs (Read Models)

Optimized for querying:

```csharp
public class FlightLegDocument
{
    // Identity
    public Guid Id { get; set; }
    public string FlightIdentifier { get; set; } // "SK123/01JAN25"
    
    // Basic info (denormalized)
    public string Airline { get; set; }
    public string AirlineName { get; set; } // "Scandinavian Airlines"
    public string FlightNumber { get; set; }
    public string Suffix { get; set; }
    public Direction Direction { get; set; }
    
    // Times (all as ISO strings for easy querying)
    public DateTime? ScheduledTime { get; set; }
    public DateTime? EstimatedTime { get; set; }
    public DateTime? ActualTime { get; set; }
    public DateTime BestTime { get; set; }
    
    // Aircraft (denormalized)
    public string Registration { get; set; }
    public string AircraftType { get; set; }
    public string AircraftTypeName { get; set; } // "Boeing 737-800"
    public string IcaoGroup { get; set; }
    
    // Route (denormalized)
    public string DepartureStation { get; set; }
    public string DepartureStationName { get; set; } // "Copenhagen Airport"
    public string ArrivalStation { get; set; }
    public string ArrivalStationName { get; set; } // "Oslo Airport"
    
    // Resources
    public string ArrivalStand { get; set; }
    public string DepartureStand { get; set; }
    public string ArrivalGate { get; set; }
    public string DepartureGate { get; set; }
    
    // Handlers (denormalized)
    public string PrimaryGroundHandler { get; set; }
    public string PrimaryGroundHandlerName { get; set; }
    
    // Status
    public FlightStatus Status { get; set; }
    public string StatusDisplay { get; set; } // Human-readable
    
    // Security
    public int SecurityLevel { get; set; }
    public string TrafficType { get; set; }
    
    // Visit (if linked)
    public Guid? VisitId { get; set; }
    public string LinkedFlightNumber { get; set; }
    
    // Alerts (denormalized for quick filtering)
    public List<AlertSummary> Alerts { get; set; }
    public bool HasHighPriorityAlerts { get; set; }
    
    // Metadata
    public int Version { get; set; }
    public DateTime LastUpdated { get; set; }
    public string[] Tags { get; set; } // For filtering
}

public class AlertSummary
{
    public string AlertType { get; set; }
    public string Message { get; set; }
    public string Level { get; set; }
    public DateTime RaisedAt { get; set; }
}
```

### 5. Client DTOs (API Responses)

Simplified for client consumption:

```csharp
public class FlightLegDTO
{
    public Guid Id { get; set; }
    
    // Flight info
    public string Flight { get; set; } // "SK123"
    public string Airline { get; set; } // "SK"
    public string AirlineName { get; set; } // "SAS"
    public string Direction { get; set; } // "Arrival" or "Departure"
    
    // Times (formatted)
    public string Scheduled { get; set; } // "2025-01-01T08:00"
    public string Estimated { get; set; }
    public string Actual { get; set; }
    
    // Aircraft
    public string Registration { get; set; }
    public string AircraftType { get; set; }
    
    // Route
    public string From { get; set; } // "OSL"
    public string To { get; set; } // "CPH"
    
    // Resources
    public string Stand { get; set; }
    public string Gate { get; set; }
    
    // Status
    public string Status { get; set; } // "On Time", "Delayed", "Arrived"
    public string StatusColor { get; set; } // "green", "red", "yellow"
    
    // Alerts
    public List<AlertDTO> Alerts { get; set; }
    
    // Metadata
    public string LastUpdated { get; set; } // "2 minutes ago"
}

public class AlertDTO
{
    public string Type { get; set; }
    public string Message { get; set; }
    public string Severity { get; set; } // "info", "warning", "error"
    public string Icon { get; set; } // "warning", "error", "info"
}
```

## Message Validation

All messages are validated before processing:

```csharp
public interface IMessageValidator<T>
{
    ValidationResult Validate(T message);
}

public class MvtMessageValidator : IMessageValidator<MvtMessage>
{
    public ValidationResult Validate(MvtMessage message)
    {
        var errors = new List<ValidationError>();
        
        // Required fields
        if (string.IsNullOrEmpty(message.Airline))
        {
            errors.Add(new ValidationError("Airline", "Airline is required"));
        }
        
        if (string.IsNullOrEmpty(message.FlightNumber))
        {
            errors.Add(new ValidationError("FlightNumber", "Flight number is required"));
        }
        
        // Format validation
        if (!Regex.IsMatch(message.Airline, "^[A-Z0-9]{2,3}$"))
        {
            errors.Add(new ValidationError("Airline", "Airline must be 2-3 uppercase letters/numbers"));
        }
        
        // Business rules
        if (message.ActualInBlockTime.HasValue && 
            message.ActualInBlockTime.Value > DateTime.UtcNow.AddDays(1))
        {
            errors.Add(new ValidationError("ActualInBlockTime", "Actual time cannot be more than 1 day in the future"));
        }
        
        return new ValidationResult(errors);
    }
}
```

## Message Versioning

Messages are versioned to handle schema evolution:

```csharp
public interface IVersionedMessage
{
    int Version { get; }
}

public class FlightLegUpdatedV1 : DomainEvent, IVersionedMessage
{
    public int Version => 1;
    
    public Guid FlightLegId { get; set; }
    public Dictionary<string, object> Changes { get; set; }
}

public class FlightLegUpdatedV2 : DomainEvent, IVersionedMessage
{
    public int Version => 2;
    
    public Guid FlightLegId { get; set; }
    
    // V2 adds detailed change tracking
    public Dictionary<string, FieldChange> Changes { get; set; }
    public string ChangedBy { get; set; }
    public string Reason { get; set; }
}

// Deserializer handles version routing
public class VersionedMessageDeserializer
{
    public DomainEvent Deserialize(string json)
    {
        var envelope = JsonSerializer.Deserialize<MessageEnvelope>(json);
        
        return envelope.Version switch
        {
            1 => JsonSerializer.Deserialize<FlightLegUpdatedV1>(json),
            2 => JsonSerializer.Deserialize<FlightLegUpdatedV2>(json),
            _ => throw new NotSupportedException($"Version {envelope.Version} not supported")
        };
    }
}
```

## Message Mapping

Adapters map external formats to internal DTOs:

```csharp
public class MvtMessageMapper : IMessageMapper<string, MvtMessage>
{
    public MvtMessage Map(string rawMessage)
    {
        var lines = rawMessage.Split('\n');
        var message = new MvtMessage();
        
        foreach (var line in lines)
        {
            // Parse MVT format
            if (line.StartsWith("MVT"))
            {
                message.MessageType = "MVT";
            }
            else if (line.Contains("/"))
            {
                // Flight identifier: SK/123
                var parts = line.Split('/');
                message.Airline = parts[0];
                message.FlightNumber = parts[1];
            }
            else if (line.StartsWith("AA"))
            {
                // Actual arrival: AA01/0700
                var time = ParseTime(line);
                message.ActualInBlockTime = time;
            }
            else if (line.StartsWith("AD"))
            {
                // Actual departure: AD01/0800
                var time = ParseTime(line);
                message.ActualOffBlockTime = time;
            }
            // ... more parsing
        }
        
        return message;
    }
}

// AODB maps adapter message to command
public class FlightLegCommandMapper
{
    public UpdateFlightLegCommand Map(MvtMessage mvtMessage)
    {
        // Find existing flight
        var flight = await _repository.FindFlight(
            mvtMessage.Airline,
            mvtMessage.FlightNumber,
            mvtMessage.OperationalDate
        );
        
        return new UpdateFlightLegCommand
        {
            FlightLegId = flight.Id,
            ActualInBlockTime = mvtMessage.ActualInBlockTime,
            ActualOffBlockTime = mvtMessage.ActualOffBlockTime,
            Registration = mvtMessage.Registration,
            MessageSource = mvtMessage.Source,
            MessageType = "MVT"
        };
    }
}
```

## Serialization

Messages use JSON serialization with specific settings:

```csharp
public class MessageSerializerOptions
{
    public static JsonSerializerOptions Default => new()
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
        DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
        Converters =
        {
            new JsonStringEnumConverter(),
            new DateTimeConverter(),
            new TimeSpanConverter()
        },
        WriteIndented = false // Compact for Kafka
    };
}

// Custom date converter for consistency
public class DateTimeConverter : JsonConverter<DateTime>
{
    public override DateTime Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options)
    {
        return DateTime.Parse(reader.GetString(), CultureInfo.InvariantCulture, DateTimeStyles.RoundtripKind);
    }
    
    public override void Write(Utf8JsonWriter writer, DateTime value, JsonSerializerOptions options)
    {
        // Always UTC, ISO 8601
        writer.WriteStringValue(value.ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ss.fffZ"));
    }
}
```

## Message Contracts in Kafka

Kafka messages include metadata:

```csharp
public class KafkaMessageEnvelope<T>
{
    // Kafka metadata
    public string Topic { get; set; }
    public int Partition { get; set; }
    public long Offset { get; set; }
    public string Key { get; set; } // For partitioning
    
    // Message metadata
    public Guid MessageId { get; set; }
    public string MessageType { get; set; }
    public int Version { get; set; }
    public DateTime Timestamp { get; set; }
    public Guid? CorrelationId { get; set; }
    public Guid? CausationId { get; set; }
    
    // Payload
    public T Payload { get; set; }
    
    // Tracing
    public Dictionary<string, string> Headers { get; set; }
}

// Publishing
await _kafkaProducer.Publish(
    topic: "aodb.flightleg.updated",
    key: flightLeg.Id.ToString(), // Ensures same partition
    value: new KafkaMessageEnvelope<FlightLegUpdated>
    {
        MessageId = Guid.NewGuid(),
        MessageType = "FlightLegUpdated",
        Version = 2,
        Timestamp = DateTime.UtcNow,
        CorrelationId = command.CorrelationId,
        Payload = @event
    }
);
```

## Schema Registry

Message schemas are registered for validation:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "FlightLegUpdated",
  "type": "object",
  "required": ["eventId", "eventType", "flightLegId", "timestamp", "version"],
  "properties": {
    "eventId": {
      "type": "string",
      "format": "uuid"
    },
    "eventType": {
      "type": "string",
      "const": "FlightLegUpdated"
    },
    "flightLegId": {
      "type": "string",
      "format": "uuid"
    },
    "timestamp": {
      "type": "string",
      "format": "date-time"
    },
    "version": {
      "type": "integer",
      "minimum": 1
    },
    "changes": {
      "type": "object",
      "additionalProperties": {
        "type": "object",
        "properties": {
          "oldValue": {},
          "newValue": {},
          "changedAt": {"type": "string", "format": "date-time"},
          "changedBy": {"type": "string"}
        }
      }
    }
  }
}
```

## Error Messages

Special message types for errors:

```csharp
public class ErrorMessage
{
    public Guid ErrorId { get; set; }
    public string ErrorCode { get; set; }
    public string ErrorMessage { get; set; }
    public string Source { get; set; }
    public DateTime Timestamp { get; set; }
    
    // Original message that failed
    public string OriginalMessage { get; set; }
    public string OriginalMessageType { get; set; }
    
    // Error details
    public string StackTrace { get; set; }
    public Dictionary<string, string> Context { get; set; }
    
    // Retry info
    public int RetryCount { get; set; }
    public DateTime? NextRetryAt { get; set; }
}
```

## Testing Message Types

```csharp
[Fact]
public void MvtMessage_Validation_RequiresAirline()
{
    // Arrange
    var message = new MvtMessage
    {
        FlightNumber = "123"
        // Airline missing
    };
    
    var validator = new MvtMessageValidator();
    
    // Act
    var result = validator.Validate(message);
    
    // Assert
    Assert.False(result.IsValid);
    Assert.Contains(result.Errors, e => e.Field == "Airline");
}

[Fact]
public void FlightLegDocument_Serialization_IncludesAllFields()
{
    // Arrange
    var document = new FlightLegDocument
    {
        Id = Guid.NewGuid(),
        FlightNumber = "SK123",
        Airline = "SK",
        Direction = Direction.Arrival
    };
    
    // Act
    var json = JsonSerializer.Serialize(document, MessageSerializerOptions.Default);
    var deserialized = JsonSerializer.Deserialize<FlightLegDocument>(json);
    
    // Assert
    Assert.Equal(document.Id, deserialized.Id);
    Assert.Equal(document.FlightNumber, deserialized.FlightNumber);
}
```

## Best Practices

### 1. Immutability

Messages should be immutable after creation:

```csharp
public class FlightLegCreated : DomainEvent
{
    public FlightLegCreated(Guid flightLegId, string flightNumber)
    {
        FlightLegId = flightLegId;
        FlightNumber = flightNumber;
    }
    
    public Guid FlightLegId { get; }
    public string FlightNumber { get; }
}
```

### 2. Versioning Strategy

Always include version number:

```csharp
public abstract class DomainEvent
{
    public int Version { get; set; } = 1;
}
```

### 3. Backward Compatibility

New versions must handle old data:

```csharp
public class FlightLegUpdatedV2Handler
{
    public async Task Handle(FlightLegUpdatedV2 @event)
    {
        // Handle V2
    }
    
    public async Task Handle(FlightLegUpdatedV1 @event)
    {
        // Convert V1 to V2 and handle
        var v2 = ConvertToV2(@event);
        await Handle(v2);
    }
}
```

### 4. Correlation IDs

Track message flow:

```csharp
command.CorrelationId = Guid.NewGuid();
@event.CorrelationId = command.CorrelationId;
```

## Related Patterns

- **[[Adapter System]]** - Produces adapter messages
- **[[CQRS Pattern]]** - Commands vs Events
- **[[Event Sourcing]]** - Domain events as source of truth
- **Data Transfer Object Pattern** - Message design
- **Command Pattern** - Command messages

---

**Tags:** #airhart #messages #dto #contracts #versioning #serialization
**Created:** 2025-12-15
**Related:** [[Adapter System]], [[CQRS Pattern]], [[Event Sourcing]], [[Business Rules]]
**References:**
- Message versioning strategies
- JSON Schema validation
- Kafka message patterns
