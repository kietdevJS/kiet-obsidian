# Entity Model

> Flexible, extensible data model that allows dynamic schema configuration without code changes

## Overview

AIRHART's **Entity Model** is a flexible data schema system that allows fields, relationships, and validations to be configured dynamically through master data rather than being hardcoded. This enables the system to adapt to different airports' requirements without rebuilding the application.

## Core Concept

```
Traditional Approach (Rigid):
┌────────────────────────────┐
│ C# Class FlightLeg         │
│ {                          │
│   public string FlightNo;  │
│   public DateTime SOBT;    │
│   public string Stand;     │
│   // All fields hardcoded  │
│ }                          │
└────────────────────────────┘
      ↓
To add field → Modify code → Rebuild → Redeploy


AIRHART Approach (Flexible):
┌────────────────────────────┐
│ Entity Configuration       │
│ (Master Data)              │
│                            │
│ FlightLeg:                 │
│   - FlightNumber (string)  │
│   - SOBT (datetime)        │
│   - Stand (string)         │
│   - CustomField1 (string)  │← Added via config
│   - AirportSpecific (int)  │← Added via config
└────────────────────────────┘
      ↓
To add field → Update configuration → Apply → Done
```

## Architecture

```
┌─────────────────────────────────────────────┐
│ Entity Definition (Master Data)             │
│ ├─ FlightLeg entity                         │
│ ├─ Visit entity                             │
│ ├─ Stand entity                             │
│ ├─ Gate entity                              │
│ └─ Custom entities                          │
└─────────────────┬───────────────────────────┘
                  ↓
┌─────────────────────────────────────────────┐
│ Field Configuration                         │
│ ├─ Field name                               │
│ ├─ Data type (string, int, datetime, etc.)  │
│ ├─ Validation rules                         │
│ ├─ Default values                           │
│ └─ Display properties                       │
└─────────────────┬───────────────────────────┘
                  ↓
┌─────────────────────────────────────────────┐
│ Runtime Entity Instance                     │
│ ├─ Core fields (always present)             │
│ ├─ Extended fields (configured)             │
│ └─ Dynamic properties bag                   │
└─────────────────────────────────────────────┘
```

## Entity Definition

Entities are defined in **Data & Assets** (Master Data):

```json
{
  "EntityType": "FlightLeg",
  "DisplayName": "Flight Leg",
  "Description": "Represents a single flight operation (arrival or departure)",
  "CoreFields": [
    {
      "FieldName": "Id",
      "DataType": "Guid",
      "Required": true,
      "Immutable": true
    },
    {
      "FieldName": "FlightNumber",
      "DataType": "String",
      "MaxLength": 10,
      "Required": true,
      "ValidationPattern": "^[A-Z0-9]{2}\\d{1,4}[A-Z]?$"
    },
    {
      "FieldName": "OperatorIata",
      "DataType": "String",
      "MaxLength": 3,
      "Required": true
    },
    {
      "FieldName": "Direction",
      "DataType": "Enum",
      "EnumType": "Direction",
      "Required": true,
      "AllowedValues": ["Arrival", "Departure"]
    }
  ],
  "ExtendedFields": [
    {
      "FieldName": "CustomHandlingNote",
      "DataType": "String",
      "MaxLength": 500,
      "Required": false,
      "DisplayName": "Custom Handling Instructions",
      "Category": "Airport Specific"
    }
  ],
  "Relationships": [
    {
      "Name": "Visit",
      "TargetEntity": "Visit",
      "Cardinality": "ManyToOne",
      "ForeignKey": "VisitId"
    }
  ]
}
```

## Data Types

Supported field data types:

| Type | Description | Example |
|------|-------------|---------|
| **String** | Text | "SK123" |
| **Int** | Integer | 42 |
| **Long** | Long integer | 9876543210 |
| **Decimal** | Decimal number | 123.45 |
| **DateTime** | Date and time | "2025-01-01T08:00:00Z" |
| **Boolean** | True/false | true |
| **Guid** | Unique identifier | "3fa85f64-5717-4562-b3fc-2c963f66afa6" |
| **Enum** | Enumeration | Direction.Arrival |
| **Json** | JSON object | { "key": "value" } |
| **Array** | List | ["item1", "item2"] |

## Field Configuration

Each field can be configured with:

```csharp
public class FieldConfiguration
{
    // Identity
    public string FieldName { get; set; }
    public string DisplayName { get; set; }
    public string Description { get; set; }
    
    // Data type
    public DataType Type { get; set; }
    public string EnumType { get; set; } // If Type == Enum
    
    // Validation
    public bool Required { get; set; }
    public int? MinLength { get; set; }
    public int? MaxLength { get; set; }
    public string ValidationPattern { get; set; } // Regex
    public object MinValue { get; set; }
    public object MaxValue { get; set; }
    public string[] AllowedValues { get; set; }
    
    // Behavior
    public bool Immutable { get; set; } // Cannot change after creation
    public bool Audited { get; set; } // Track changes
    public object DefaultValue { get; set; }
    
    // Display
    public string Category { get; set; } // UI grouping
    public int DisplayOrder { get; set; }
    public bool Visible { get; set; }
    public string Icon { get; set; }
}
```

## Core Entities

### FlightLeg

Represents a single flight operation:

```
Core Fields (Always Present):
├─ Id (Guid)
├─ FlightNumber (string)
├─ OperatorIata (string)
├─ OperatorIcao (string)
├─ Direction (enum: Arrival/Departure)
├─ Registration (string)
├─ TypeIata (string) - Aircraft type
├─ IcaoGroup (string)
├─ ScheduledInBlockTime / ScheduledOffBlockTime (datetime)
├─ EstimatedInBlockTime / EstimatedOffBlockTime (datetime)
├─ ActualInBlockTime / ActualOffBlockTime (datetime)
├─ ArrivalStand / DepartureStand (string)
├─ ArrivalGate / DepartureGate (string)
├─ ArrivalStation / DepartureStation (string)
├─ PrimaryGroundHandler (string)
├─ SecurityLevel (int)
├─ TrafficType (enum)
├─ VisitId (Guid?) - Link to paired flight
├─ Source (string) - Message source
├─ Status (enum: Scheduled, Active, Departed, Arrived, Cancelled)
└─ Alerts (collection)

Extended Fields (Configurable):
├─ CustomField1 (string)
├─ AirportSpecificNote (string)
└─ ...more as configured
```

### Visit

Represents linked arrival and departure:

```
Core Fields:
├─ Id (Guid)
├─ ArrivalFlightId (Guid?)
├─ DepartureFlightId (Guid?)
├─ Registration (string)
├─ AircraftType (string)
├─ Status (enum)
├─ MinimumTurnaroundTime (int) - Minutes
├─ EstimatedTurnaroundTime (int) - Minutes
└─ ActualTurnaroundTime (int?) - Minutes
```

### Stand

Airport parking position:

```
Core Fields:
├─ Id (Guid)
├─ StandName (string)
├─ Terminal (string)
├─ IcaoGroup (string) - Max aircraft size
├─ Latitude (decimal)
├─ Longitude (decimal)
├─ HasJetBridge (bool)
├─ ContactStand (bool)
├─ Status (enum: Open, Closed, Downgraded)
└─ OperationalEvents (collection)

Extended Fields:
├─ CustomEquipment1 (bool)
├─ AirportSpecificAttribute (string)
└─ ...
```

### Gate

Passenger boarding gate:

```
Core Fields:
├─ Id (Guid)
├─ GateName (string)
├─ Terminal (string)
├─ Stand (string) - Associated stand
├─ AllowedSecurityLevels (int[])
├─ AllowedTrafficTypes (enum[])
├─ Status (enum)
└─ OperationalEvents (collection)
```

## Dynamic Properties

For fields not defined in schema:

```csharp
public class FlightLeg : DynamicEntity
{
    // Core fields (strongly typed)
    public Guid Id { get; set; }
    public string FlightNumber { get; set; }
    // ...
    
    // Dynamic properties bag
    private Dictionary<string, object> _extendedProperties = new();
    
    public object this[string fieldName]
    {
        get => _extendedProperties.TryGetValue(fieldName, out var value) ? value : null;
        set => _extendedProperties[fieldName] = value;
    }
    
    public T GetExtendedProperty<T>(string fieldName)
    {
        if (_extendedProperties.TryGetValue(fieldName, out var value))
        {
            return (T)Convert.ChangeType(value, typeof(T));
        }
        return default(T);
    }
    
    public void SetExtendedProperty(string fieldName, object value)
    {
        _extendedProperties[fieldName] = value;
    }
}

// Usage
var flight = new FlightLeg();

// Core property (strongly typed)
flight.FlightNumber = "SK123";

// Extended property (dynamic)
flight["CustomHandlingNote"] = "Requires special catering";
flight.SetExtendedProperty("PriorityLevel", 5);

var note = flight.GetExtendedProperty<string>("CustomHandlingNote");
var priority = flight.GetExtendedProperty<int>("PriorityLevel");
```

## Validation

Fields are validated based on configuration:

```csharp
public class EntityValidator
{
    private readonly IEntityConfiguration _config;
    
    public ValidationResult Validate(FlightLeg flight)
    {
        var errors = new List<ValidationError>();
        
        foreach (var field in _config.GetFields("FlightLeg"))
        {
            var value = GetFieldValue(flight, field.FieldName);
            
            // Required check
            if (field.Required && IsNullOrEmpty(value))
            {
                errors.Add(new ValidationError
                {
                    FieldName = field.FieldName,
                    Message = $"{field.DisplayName} is required"
                });
            }
            
            // Type check
            if (value != null && !IsCorrectType(value, field.Type))
            {
                errors.Add(new ValidationError
                {
                    FieldName = field.FieldName,
                    Message = $"{field.DisplayName} must be of type {field.Type}"
                });
            }
            
            // Length check
            if (value is string str)
            {
                if (field.MinLength.HasValue && str.Length < field.MinLength.Value)
                {
                    errors.Add(new ValidationError
                    {
                        FieldName = field.FieldName,
                        Message = $"{field.DisplayName} must be at least {field.MinLength} characters"
                    });
                }
                
                if (field.MaxLength.HasValue && str.Length > field.MaxLength.Value)
                {
                    errors.Add(new ValidationError
                    {
                        FieldName = field.FieldName,
                        Message = $"{field.DisplayName} cannot exceed {field.MaxLength} characters"
                    });
                }
            }
            
            // Pattern check
            if (value is string strValue && !string.IsNullOrEmpty(field.ValidationPattern))
            {
                if (!Regex.IsMatch(strValue, field.ValidationPattern))
                {
                    errors.Add(new ValidationError
                    {
                        FieldName = field.FieldName,
                        Message = $"{field.DisplayName} format is invalid"
                    });
                }
            }
            
            // Range check
            if (field.MinValue != null || field.MaxValue != null)
            {
                var comparable = value as IComparable;
                if (comparable != null)
                {
                    if (field.MinValue != null && comparable.CompareTo(field.MinValue) < 0)
                    {
                        errors.Add(new ValidationError
                        {
                            FieldName = field.FieldName,
                            Message = $"{field.DisplayName} must be at least {field.MinValue}"
                        });
                    }
                    
                    if (field.MaxValue != null && comparable.CompareTo(field.MaxValue) > 0)
                    {
                        errors.Add(new ValidationError
                        {
                            FieldName = field.FieldName,
                            Message = $"{field.DisplayName} cannot exceed {field.MaxValue}"
                        });
                    }
                }
            }
            
            // Allowed values check
            if (field.AllowedValues != null && field.AllowedValues.Length > 0)
            {
                if (!field.AllowedValues.Contains(value?.ToString()))
                {
                    errors.Add(new ValidationError
                    {
                        FieldName = field.FieldName,
                        Message = $"{field.DisplayName} must be one of: {string.Join(", ", field.AllowedValues)}"
                    });
                }
            }
        }
        
        return new ValidationResult(errors);
    }
}
```

## Relationships

Entities can be related:

```csharp
public class EntityRelationship
{
    public string Name { get; set; }
    public string SourceEntity { get; set; }
    public string TargetEntity { get; set; }
    public Cardinality Cardinality { get; set; }
    public string ForeignKey { get; set; }
    public string InverseName { get; set; } // Navigation from target to source
}

public enum Cardinality
{
    OneToOne,
    OneToMany,
    ManyToOne,
    ManyToMany
}

// Example: FlightLeg → Visit (ManyToOne)
var visitRelationship = new EntityRelationship
{
    Name = "Visit",
    SourceEntity = "FlightLeg",
    TargetEntity = "Visit",
    Cardinality = Cardinality.ManyToOne,
    ForeignKey = "VisitId",
    InverseName = "FlightLegs"
};

// Usage
var flight = await _repository.GetById<FlightLeg>(flightId, 
    include: f => f.Visit);

Console.WriteLine($"Flight {flight.FlightNumber} is part of visit {flight.Visit.Id}");
```

## Schema Evolution

Adding new fields without code changes:

### Step 1: Define Field in Master Data

```sql
INSERT INTO EntityFieldConfiguration
(EntityType, FieldName, DataType, Required, DisplayName, Category)
VALUES
('FlightLeg', 'CustomPriority', 'Int', 0, 'Custom Priority Level', 'Airport Specific');
```

### Step 2: Use Field Immediately

```csharp
// Business rule can now use the new field
public class AssignCustomPrioritySubRule : SubRule
{
    public override async Task<ProcessResult> Process(
        EnrichmentContext context,
        FlightLeg flightLeg)
    {
        // Determine priority based on airline, time, etc.
        int priority = CalculatePriority(flightLeg);
        
        // Set extended property
        flightLeg["CustomPriority"] = priority;
        
        return ProcessResult.Success();
    }
}
```

### Step 3: UI Automatically Shows Field

```typescript
// UI automatically queries entity configuration
const fields = await entityService.getFields('FlightLeg');

// Dynamically render form
fields.forEach(field => {
    if (field.category === 'Airport Specific') {
        renderField(field);
    }
});

// Field appears in UI without code changes!
```

## Serialization

Entities serialize to JSON with extended properties:

```csharp
public class EntityJsonConverter : JsonConverter<DynamicEntity>
{
    public override void Write(Utf8JsonWriter writer, DynamicEntity entity, JsonSerializerOptions options)
    {
        writer.WriteStartObject();
        
        // Write core properties
        foreach (var prop in entity.GetType().GetProperties())
        {
            writer.WritePropertyName(ToCamelCase(prop.Name));
            JsonSerializer.Serialize(writer, prop.GetValue(entity), options);
        }
        
        // Write extended properties
        foreach (var (key, value) in entity.GetExtendedProperties())
        {
            writer.WritePropertyName(ToCamelCase(key));
            JsonSerializer.Serialize(writer, value, options);
        }
        
        writer.WriteEndObject();
    }
}

// Result JSON
{
    "id": "123e4567-e89b-12d3-a456-426614174000",
    "flightNumber": "SK123",
    "operatorIata": "SK",
    "direction": "Arrival",
    // ... core fields ...
    "customPriority": 5,  // ← Extended property
    "customHandlingNote": "Special catering"  // ← Extended property
}
```

## Database Storage

### Option 1: Relational (PostgreSQL)

```sql
-- Core table with fixed columns
CREATE TABLE flight_legs (
    id UUID PRIMARY KEY,
    flight_number VARCHAR(10) NOT NULL,
    operator_iata VARCHAR(3) NOT NULL,
    direction VARCHAR(20) NOT NULL,
    registration VARCHAR(10),
    -- ... more core fields ...
    
    -- Extended properties as JSON
    extended_properties JSONB
);

-- Query extended properties
SELECT *
FROM flight_legs
WHERE extended_properties->>'CustomPriority' = '5';

-- Index on extended property
CREATE INDEX idx_custom_priority 
ON flight_legs ((extended_properties->>'CustomPriority'));
```

### Option 2: Document DB (QueryStream)

```json
{
    "_id": "123e4567-e89b-12d3-a456-426614174000",
    "flightNumber": "SK123",
    "operatorIata": "SK",
    "direction": "Arrival",
    "registration": "OY-ABC",
    "customPriority": 5,
    "customHandlingNote": "Special catering",
    "_metadata": {
        "version": 15,
        "lastUpdated": "2025-01-01T08:00:00Z",
        "updatedBy": "AODB"
    }
}
```

## Benefits

### 1. Airport-Specific Customization

Different airports can add their own fields:

```
Copenhagen Airport:
├─ CustomPriority (int)
├─ HandlingNote (string)
└─ SpecialCatering (bool)

Stockholm Airport:
├─ WeatherDelay (bool)
├─ IceRemoval (bool)
└─ SASPriority (int)

Oslo Airport:
├─ NorwegianSpecial (string)
└─ ArcticOperation (bool)
```

### 2. No Code Changes for New Fields

```
Old approach:
Add field → Modify C# class → Recompile → Test → Deploy → Restart
(Days/weeks of work)

New approach:
Add field configuration → Done
(Minutes of work)
```

### 3. UI Auto-Adapts

```
Field configuration includes:
├─ Display name
├─ Category
├─ Display order
├─ Validation rules
└─ Help text

UI reads configuration and automatically:
├─ Renders form fields
├─ Groups fields by category
├─ Applies validation
└─ Shows help text
```

### 4. API Auto-Documents

```csharp
// Swagger automatically includes extended fields
[HttpGet("api/flightlegs/{id}")]
public async Task<FlightLegDTO> GetFlightLeg(Guid id)
{
    var flight = await _repository.GetById<FlightLeg>(id);
    var fields = await _entityConfig.GetFields("FlightLeg");
    
    // Swagger generates schema including extended fields
    return MapToDTO(flight, fields);
}
```

## Challenges

### Type Safety

Extended properties lose compile-time type checking:

```csharp
// ❌ No compile-time check
flight["CustomPrority"] = 5; // Typo! Should be "CustomPriority"

// ✅ Mitigate with constants
public static class FlightLegFields
{
    public const string CustomPriority = "CustomPriority";
    public const string HandlingNote = "CustomHandlingNote";
}

flight[FlightLegFields.CustomPriority] = 5; // Typo caught by compiler
```

### Performance

Dynamic property access is slower:

```csharp
// Fast (core property)
var flightNo = flight.FlightNumber; // Direct field access

// Slower (extended property)
var priority = flight["CustomPriority"]; // Dictionary lookup
```

**Mitigation:** Cache frequently accessed extended properties in core fields.

### Querying

Extended properties harder to query efficiently:

```sql
-- Slow (JSONB query)
SELECT * FROM flight_legs
WHERE extended_properties->>'CustomPriority' = '5';

-- Fast (indexed column)
SELECT * FROM flight_legs
WHERE custom_priority = 5;
```

**Mitigation:** Promote frequently queried extended properties to core fields.

## Best Practices

### 1. Core vs Extended

Use **core fields** for:
- Frequently accessed
- Performance-critical
- Complex relationships
- Heavy querying

Use **extended fields** for:
- Airport-specific
- Rarely queried
- Simple types
- Temporary experiments

### 2. Naming Conventions

```
Core fields: PascalCase (FlightNumber, OperatorIata)
Extended fields: PascalCase with prefix (CustomPriority, AirportSpecificNote)
```

### 3. Versioning

Track entity schema version:

```csharp
public class DynamicEntity
{
    public int SchemaVersion { get; set; }
    
    // Migration logic when schema version changes
    public void MigrateToVersion(int targetVersion)
    {
        while (SchemaVersion < targetVersion)
        {
            SchemaVersion++;
            ApplyMigration(SchemaVersion);
        }
    }
}
```

### 4. Documentation

Document extended fields in master data:

```json
{
    "FieldName": "CustomPriority",
    "Description": "Priority level for custom handling (1-10)",
    "Usage": "Set by AssignCustomPrioritySubRule based on airline and time of day",
    "DisplayName": "Priority Level",
    "HelpText": "Higher priority flights receive expedited handling"
}
```

## Related Patterns

- **[[Master Data Component]]** - Stores entity configuration
- **[[Business Rules]]** - Enriches entity fields
- **[[Message Types]]** - DTOs map to entities
- **Property Bag Pattern** - Dynamic properties storage
- **Schema-on-Read** - Flexible schema interpretation

---

**Tags:** #airhart #entity-model #flexible-schema #dynamic-data #configuration
**Created:** 2025-12-15
**Related:** [[Master Data Component]], [[Business Rules]], [[Message Types]]
**References:**
- Entity Framework Core Dynamic Properties
- JSON Schema Validation
