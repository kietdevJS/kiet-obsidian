# Business Rules

> Enrichment rule engine that automatically validates, enriches, and transforms flight data as it flows through AODB

## Overview

**Business Rules** are the heart of AIRHART's data processing pipeline. They automatically apply business logic to enrich, validate, and transform flight data as messages are processed. Business rules execute in a defined order during **Flight Leg Enrichment** and **Visit Enrichment** phases.

## Architecture

```
Message arrives → AODB Pipeline → Business Rules Engine
                                        ↓
                              ┌─────────────────────┐
                              │ Flight Leg          │
                              │ Enrichment Rules    │
                              │ (individual flight) │
                              └──────────┬──────────┘
                                        ↓
                              ┌─────────────────────┐
                              │ Visit Enrichment    │
                              │ Rules               │
                              │ (linked flights)    │
                              └──────────┬──────────┘
                                        ↓
                              Enriched flight ready
                              for publishing to Kafka
```

### Two-Phase Enrichment

#### Phase 1: Flight Leg Enrichment

Runs on **individual flight legs** when messages update flight data:

```
Input: Raw flight data from adapter
       └─ Flight SK123 with partial information

Business Rules Execute:
├─ Transform flight number (airline IATA/ICAO)
├─ Lookup aircraft details from registration
├─ Assign traffic type and security level
├─ Map delay codes
├─ Lookup ground handlers from service agreement
├─ Assign stand parking rules
├─ Calculate estimated times (EIBT, ETOT, etc.)
├─ Validate gate/stand compatibility
└─ Raise alerts (CDM, RFR, ERR, etc.)

Output: Enriched flight leg
        └─ Complete aircraft details, handlers, times, validation alerts
```

**Examples of Flight Leg Enrichment Rules:**
- Transform flight number using airline transformations
- Assign aircraft details from registration
- Calculate expected flying time
- Assign traffic type and security level
- Validate stand ICAO group compatibility
- Raise alert for missing master data (RFR01)

#### Phase 2: Visit Enrichment

Runs on **linked flight pairs** (arrival + departure visit):

```
Input: Enriched arrival + enriched departure

Business Rules Execute:
├─ Propagate registration between flights
├─ Propagate aircraft type across visit
├─ Validate turnaround time (MTTT vs ETTT)
├─ Check aircraft consistency across visit
├─ Re-trigger dependent rules if propagation occurred
├─ Raise visit-level alerts (VST, CDM07)
└─ Validate security level changes across visit

Output: Fully enriched visit
        └─ Consistent data across arrival/departure
```

**Examples of Visit Enrichment Rules:**
- Propagate registration from arrival to departure (or vice versa)
- Validate aircraft type consistency (FLT03a alert)
- Check minimum turnaround time (MTTT)
- Validate EIBT + MTTT vs EOBT (CDM07 alert)
- Detect security level changes across visit (VST03 alert)

### Re-Triggering Rules After Propagation

When Visit Enrichment propagates registration or aircraft type between flights, **dependent rules are re-executed**:

```
Scenario: Link created between arrival and departure

1. Arrival has registration "OY-ABC"
2. Departure has no registration
3. Visit Enrichment propagates "OY-ABC" to departure

Re-trigger these rules on DEPARTURE:
├─ AssignAircraftDetailsSubRule (sets aircraft type from registration)
├─ AssignExpectedFlyingTimeSubRule (depends on aircraft type)
├─ SetEstimatedInBlockTimeSubRule (depends on expected flying time)
├─ AssignGroundHandlerSubRule (depends on aircraft type)
├─ AssignPublicFlightIdentifierSubRule (depends on ground handler)
└─ MissingPrimaryGroundAlertSubRule (validates ground handler)

Result: Departure now has consistent aircraft-related data
```

This ensures **consistency across the visit** even when data is propagated late.

## Rule Structure

Business rules are implemented as C# classes:

```csharp
public class AssignAircraftDetailsSubRule : SubRule
{
    public override string RuleName => "Assign Aircraft Details";
    
    public override async Task<ProcessResult> Process(
        EnrichmentContext context, 
        FlightLeg flightLeg)
    {
        // 1. Check if registration is present
        if (string.IsNullOrEmpty(flightLeg.Registration))
        {
            return ProcessResult.Success(); // Nothing to do
        }
        
        // 2. Lookup aircraft in master data
        var aircraft = await _masterDataService.GetAircraft(
            flightLeg.Registration, 
            flightLeg.BestOperationTime
        );
        
        if (aircraft == null)
        {
            // 3. Raise alert for missing data
            flightLeg.AddAlert(new Alert
            {
                AlertType = "RFR01",
                AlertMessage = "Missing values from Data & Assets.",
                AlertLevel = AlertLevel.High,
                MasterDataSourceTable = "Aircraft",
                LookupValue = flightLeg.Registration
            });
            
            return ProcessResult.Failure("Aircraft not found");
        }
        
        // 4. Enrich flight with aircraft details
        flightLeg.TypeIata = aircraft.TypeIata;
        flightLeg.TypeIcao = aircraft.TypeIcao;
        flightLeg.IcaoGroup = aircraft.IcaoGroup;
        flightLeg.WingSpan = aircraft.WingSpan;
        flightLeg.MaxTakeoffWeight = aircraft.MaxTakeoffWeight;
        
        return ProcessResult.Success();
    }
}
```

### Key Elements

1. **Rule Name**: Human-readable identifier
2. **Process Method**: Core business logic
3. **Master Data Lookups**: Query reference data
4. **Alert Raising**: Notify users of issues
5. **Field Updates**: Enrich flight with calculated/looked-up values

## Categories of Business Rules

### 1. Transformation Rules

Map external codes to internal standards:

```csharp
// Example: Transform airline IATA to ICAO
public class TransformFlightNumberSubRule : SubRule
{
    public override async Task<ProcessResult> Process(
        EnrichmentContext context, 
        FlightLeg flightLeg)
    {
        // Lookup airline transformation
        var transformation = await _masterData.GetAirlineTransformation(
            flightLeg.OperatorIata,
            flightLeg.BestOperationTime
        );
        
        if (transformation != null)
        {
            flightLeg.OperatorIcao = transformation.IcaoCode;
        }
        
        return ProcessResult.Success();
    }
}
```

**Use cases:**
- Airline IATA ↔ ICAO transformation
- Aircraft type transformations (airline-specific codes)
- Flight number transformations
- Route transformations

### 2. Lookup Rules

Fetch reference data from Master Data Component:

```csharp
// Example: Assign ground handlers from service agreement
public class AssignGroundHandlerSubRule : SubRule
{
    public override async Task<ProcessResult> Process(
        EnrichmentContext context,
        FlightLeg flightLeg)
    {
        // Build lookup criteria
        var criteria = new ServiceAgreementCriteria
        {
            AirlineIata = flightLeg.OperatorIata,
            AircraftTypeIata = flightLeg.TypeIata,
            Direction = flightLeg.Direction,
            ValidAt = flightLeg.BestOperationTime
        };
        
        // Query master data
        var agreements = await _masterData.GetServiceAgreements(criteria);
        
        if (!agreements.Any())
        {
            flightLeg.AddAlert("RFR03", "Primary Ground Handler not assigned.", AlertLevel.High);
            return ProcessResult.Failure();
        }
        
        // Assign handlers
        var primary = agreements.First(a => a.Type == "Primary");
        flightLeg.PrimaryGroundHandler = primary.HandlerCode;
        
        return ProcessResult.Success();
    }
}
```

**Lookup targets:**
- Service agreements (ground handlers)
- Airport details
- Stand parking rules
- Expected flying times
- Security level mappings
- Delay code mappings

### 3. Validation Rules

Check data consistency and raise alerts:

```csharp
// Example: Validate stand ICAO group compatibility
public class ValidateStandIcaoGroupSubRule : SubRule
{
    public override async Task<ProcessResult> Process(
        EnrichmentContext context,
        FlightLeg flightLeg)
    {
        if (string.IsNullOrEmpty(flightLeg.ArrivalStand) || 
            string.IsNullOrEmpty(flightLeg.IcaoGroup))
        {
            return ProcessResult.Success();
        }
        
        var stand = await _masterData.GetStand(
            flightLeg.ArrivalStand,
            flightLeg.BestInBlockTime
        );
        
        // Check if aircraft too large for stand
        if (IsAircraftTooLarge(flightLeg.IcaoGroup, stand.IcaoGroup))
        {
            flightLeg.AddAlert(
                "RES12",
                "Arrival Stand ICAO Group downgrade.",
                AlertLevel.High,
                masterDataTable: "aircraft_type",
                lookupValue: flightLeg.TypeIata
            );
        }
        
        return ProcessResult.Success();
    }
    
    private bool IsAircraftTooLarge(string aircraftGroup, string standGroup)
    {
        // ICAO groups: A < B < C < D < E < F
        var groups = new[] { "A", "B", "C", "D", "E", "F" };
        var aircraftIndex = Array.IndexOf(groups, aircraftGroup);
        var standIndex = Array.IndexOf(groups, standGroup);
        
        return aircraftIndex > standIndex;
    }
}
```

**Validation types:**
- Stand/gate capacity vs aircraft size
- Security level compatibility
- Traffic type matching
- Time window validations (TOBT vs EOBT)
- Route discrepancies

### 4. Calculation Rules

Compute derived values:

```csharp
// Example: Calculate Estimated In-Block Time
public class SetEstimatedInBlockTimeSubRule : SubRule
{
    public override async Task<ProcessResult> Process(
        EnrichmentContext context,
        FlightLeg flightLeg)
    {
        if (flightLeg.Direction != Direction.Arrival)
        {
            return ProcessResult.Success();
        }
        
        // EIBT = Best Take-Off Time + Expected Flying Time
        if (flightLeg.BestTakeOffTime.HasValue && 
            flightLeg.ExpectedFlyingTime.HasValue)
        {
            flightLeg.EstimatedInBlockTime = 
                flightLeg.BestTakeOffTime.Value
                .AddMinutes(flightLeg.ExpectedFlyingTime.Value);
        }
        
        return ProcessResult.Success();
    }
}
```

**Calculated fields:**
- Estimated In-Block Time (EIBT)
- Estimated Take-Off Time (ETOT)
- Estimated Taxi-Out Duration (EXOT)
- Estimated Turnaround Time (ETTT)
- Minimum Turnaround Time (MTTT)

### 5. Propagation Rules

Copy values between linked flights:

```csharp
// Example: Propagate registration across visit
public class PropagateRegistrationSubRule : SubRule
{
    public override async Task<ProcessResult> Process(
        EnrichmentContext context,
        Visit visit)
    {
        var arrival = visit.ArrivalFlight;
        var departure = visit.DepartureFlight;
        
        if (arrival == null || departure == null)
        {
            return ProcessResult.Success();
        }
        
        // Determine which flight has better registration source
        var arrivalSource = GetSourcePriority(arrival.RegistrationSource);
        var departureSource = GetSourcePriority(departure.RegistrationSource);
        
        if (arrivalSource > departureSource && 
            !string.IsNullOrEmpty(arrival.Registration))
        {
            // Arrival has better source → propagate to departure
            departure.Registration = arrival.Registration;
            departure.RegistrationSource = "Propagation";
            
            return ProcessResult.PropagationOccurred; // Trigger re-enrichment
        }
        else if (departureSource > arrivalSource && 
                 !string.IsNullOrEmpty(departure.Registration))
        {
            // Departure has better source → propagate to arrival
            arrival.Registration = departure.Registration;
            arrival.RegistrationSource = "Propagation";
            
            return ProcessResult.PropagationOccurred; // Trigger re-enrichment
        }
        
        return ProcessResult.Success();
    }
    
    private int GetSourcePriority(string source)
    {
        // MVT message = highest priority
        // Flight plan = medium priority
        // Propagation = lowest priority
        return source switch
        {
            "MVT" => 3,
            "FlightPlan" => 2,
            "Propagation" => 1,
            _ => 0
        };
    }
}
```

## Alert System

Business rules raise **alerts** when issues are detected:

### Alert Types

| Prefix | Category | Example |
|--------|----------|---------|
| **CDM** | A-CDM Process | CDM01: No airport slot available |
| **ERR** | Business Errors | ERR01: Mandatory information missing |
| **FLT** | Flight-related | FLT03a: Aircraft type discrepancy across visit |
| **RFR** | Reference Data | RFR01: Missing values from Data & Assets |
| **SEC** | Security | SEC01: Flight traffic type doesn't match gate |
| **RES** | Resource Issues | RES14: No parking rule found for stand |
| **VST** | Visit-related | VST01: Short estimated turnaround time |
| **LNK** | Linking Issues | LNK08: Unlinked departure operation |
| **INA/IND** | INOP on Arrival/Departure | INA01: Arrival stand is inoperable |

### Alert Severity

```csharp
public enum AlertLevel
{
    Info,    // Informational only
    Low,     // Minor issue, no immediate action
    Medium,  // Requires attention
    High     // Critical, immediate action needed
}
```

### Raising Alerts

```csharp
// Simple alert
flightLeg.AddAlert(
    alertType: "RFR01",
    message: "Missing values from Data & Assets.",
    level: AlertLevel.High
);

// Alert with context
flightLeg.AddAlert(new Alert
{
    AlertType = "RFR02",
    AlertMessage = "Multiple values from Data & Assets.",
    AlertLevel = AlertLevel.Medium,
    MasterDataSourceTable = "Service Agreement",
    LookupValue = "SK"
});
```

### Alert Lifecycle

```
Alert raised → Active
         ↓
Issue resolved (data arrives) → Alert cleared
         ↓
No longer active
```

Alerts are **automatically cleared** when the condition no longer applies (e.g., missing data arrives).

## Rule Execution Order

Rules execute in a **defined sequence** to handle dependencies:

```
Flight Leg Enrichment Order:
1. TransformFlightNumberSubRule
2. AssignAircraftDetailsSubRule ← Depends on registration
3. AssignExpectedFlyingTimeSubRule ← Depends on aircraft type
4. SetEstimatedInBlockTimeSubRule ← Depends on expected flying time
5. AssignGroundHandlerSubRule ← Depends on aircraft type
6. AssignPublicFlightIdentifierSubRule ← Depends on ground handler
7. ValidateStandCompatibilitySubRule
8. AssignSecurityLevelSubRule
9. AssignTrafficTypeSubRule
10. ValidateGateSecuritySubRule
...

Visit Enrichment Order:
1. PropagateRegistrationSubRule
2. PropagateAircraftTypeSubRule
3. ValidateAircraftConsistencySubRule ← Check arrival vs departure
4. CalculateTurnaroundTimeSubRule
5. ValidateMinimumTurnaroundSubRule
6. DetectSecurityLevelChangeSubRule
...

If propagation occurred:
└─ Re-trigger dependent Flight Leg rules on affected flight
```

## Configuration

Business rules can be configured via **Master Data**:

### Time Parameters

Used in alerts for tolerance windows:

```sql
-- Example: CDM08 EOBT compliance alert
Time Parameter: TOBT must be within EOBT ± 10 minutes

Configuration in Master Data:
{
    "AlertType": "CDM08",
    "MinMinutes": -10,
    "MaxMinutes": +15
}
```

| Alert | Parameter | Min | Max |
|-------|-----------|-----|-----|
| CDM02 | SOBT vs initial EOBT | -5 min | +5 min |
| CDM08 | EOBT compliance | -10 min | +15 min |
| CDM14 | TOBT not received | -70 min | +5 min |
| FLT72 | Missing aircraft at stand | -30 min | - |

### Rule Toggle

Rules can be enabled/disabled via configuration:

```json
{
    "BusinessRules": {
        "FlightLegEnrichment": {
            "AssignAircraftDetailsSubRule": {
                "Enabled": true
            },
            "ValidateStandCompatibilitySubRule": {
                "Enabled": true
            }
        },
        "VisitEnrichment": {
            "PropagateRegistrationSubRule": {
                "Enabled": true
            }
        }
    }
}
```

## Example: Complete Enrichment Flow

```
1. Adapter receives MVT message:
   MVT
   SK/123/01JAN
   EKCH/ENGM
   AD01/0800
   AA01/0700
   OY-ABC

2. Message mapped to FlightLeg entity:
   {
       Operator: "SK",
       FlightNumber: "123",
       Direction: "Arrival",
       ScheduledInBlockTime: "2025-01-01T08:00Z",
       ActualInBlockTime: "2025-01-01T07:00Z",
       Registration: "OY-ABC",
       ArrivalStation: "EKCH",
       DepartureStation: "ENGM"
   }

3. Flight Leg Enrichment runs:

   TransformFlightNumberSubRule:
   ├─ Lookup airline transformation for "SK"
   ├─ Found: SK → SAS (ICAO)
   └─ Set OperatorIcao = "SAS"

   AssignAircraftDetailsSubRule:
   ├─ Lookup aircraft "OY-ABC" in Master Data
   ├─ Found: Boeing 737-800
   └─ Set:
       TypeIata = "73H"
       TypeIcao = "B738"
       IcaoGroup = "C"
       WingSpan = 34.3m
       MaxTakeoffWeight = 79010kg

   AssignExpectedFlyingTimeSubRule:
   ├─ Lookup expected flying time ENGM → EKCH for 73H
   ├─ Found: 1 hour 20 minutes
   └─ Set ExpectedFlyingTime = 80 minutes

   SetEstimatedInBlockTimeSubRule:
   ├─ Calculate EIBT = ETOT + ExpectedFlyingTime
   ├─ ETOT not available yet
   └─ Skip

   AssignGroundHandlerSubRule:
   ├─ Lookup service agreement: SK + 73H + Arrival
   ├─ Found: Primary handler = "SAS Ground"
   └─ Set PrimaryGroundHandler = "SASGH"

   ValidateStandCompatibilitySubRule:
   ├─ Check if assigned stand "B5" supports ICAO Group C
   ├─ Stand B5 max ICAO Group = D ✓
   └─ No alert

   AssignSecurityLevelSubRule:
   ├─ Lookup security level for route ENGM → EKCH
   ├─ Norway (Schengen) → Denmark (Schengen)
   └─ Set SecurityLevel = 1 (Schengen)

4. Result: Fully enriched flight leg
   {
       Operator: "SK",
       OperatorIcao: "SAS",
       FlightNumber: "123",
       Direction: "Arrival",
       Registration: "OY-ABC",
       TypeIata: "73H",
       TypeIcao: "B738",
       IcaoGroup: "C",
       ExpectedFlyingTime: 80,
       PrimaryGroundHandler: "SASGH",
       SecurityLevel: 1,
       TrafficType: "Schengen",
       ScheduledInBlockTime: "2025-01-01T08:00Z",
       ActualInBlockTime: "2025-01-01T07:00Z",
       Alerts: []
   }

5. Published to Kafka → QueryStream updates document DB
```

## Integration with Master Data

Business rules heavily rely on [[Master Data Component]]:

```
Business Rule needs reference data
         ↓
Query Master Data Component
         ↓
Master Data queries bitemporal storage
         ↓
Returns data valid at specific time (BestOperationTime)
         ↓
Business rule applies data to flight
```

**Example:**

```csharp
// Query master data with temporal context
var aircraft = await _masterDataService.GetAircraft(
    registration: "OY-ABC",
    validAt: flightLeg.BestOperationTime  // ← Bitemporal query
);

// Returns aircraft data that was valid at flight's operation time
```

This ensures **historical accuracy** - flights are enriched with data that was valid when the flight operated, not current data.

## Testing Business Rules

```csharp
[Fact]
public async Task AssignAircraftDetails_WithValidRegistration_EnrichesFlightLeg()
{
    // Arrange
    var flightLeg = new FlightLeg
    {
        Registration = "OY-ABC",
        BestOperationTime = DateTime.Parse("2025-01-01T08:00Z")
    };
    
    var aircraft = new Aircraft
    {
        Registration = "OY-ABC",
        TypeIata = "73H",
        IcaoGroup = "C"
    };
    
    _mockMasterData
        .Setup(m => m.GetAircraft("OY-ABC", It.IsAny<DateTime>()))
        .ReturnsAsync(aircraft);
    
    var rule = new AssignAircraftDetailsSubRule(_mockMasterData.Object);
    
    // Act
    var result = await rule.Process(context, flightLeg);
    
    // Assert
    Assert.True(result.Success);
    Assert.Equal("73H", flightLeg.TypeIata);
    Assert.Equal("C", flightLeg.IcaoGroup);
    Assert.Empty(flightLeg.Alerts);
}

[Fact]
public async Task AssignAircraftDetails_WithMissingAircraft_RaisesAlert()
{
    // Arrange
    var flightLeg = new FlightLeg
    {
        Registration = "UNKNOWN",
        BestOperationTime = DateTime.Parse("2025-01-01T08:00Z")
    };
    
    _mockMasterData
        .Setup(m => m.GetAircraft("UNKNOWN", It.IsAny<DateTime>()))
        .ReturnsAsync((Aircraft)null);
    
    var rule = new AssignAircraftDetailsSubRule(_mockMasterData.Object);
    
    // Act
    var result = await rule.Process(context, flightLeg);
    
    // Assert
    Assert.False(result.Success);
    Assert.Single(flightLeg.Alerts);
    Assert.Equal("RFR01", flightLeg.Alerts[0].AlertType);
    Assert.Equal(AlertLevel.High, flightLeg.Alerts[0].AlertLevel);
}
```

## Performance Considerations

### Rule Execution

- Rules execute **sequentially** within a phase (dependencies)
- **Caching** master data lookups to avoid repeated queries
- **Short-circuit evaluation** - skip rules if prerequisites not met

```csharp
public override async Task<ProcessResult> Process(
    EnrichmentContext context,
    FlightLeg flightLeg)
{
    // Short-circuit if registration not present
    if (string.IsNullOrEmpty(flightLeg.Registration))
    {
        return ProcessResult.Success(); // Skip, nothing to do
    }
    
    // Proceed with expensive lookup
    var aircraft = await _masterDataService.GetAircraft(
        flightLeg.Registration,
        flightLeg.BestOperationTime
    );
    
    // ...
}
```

### Master Data Caching

```csharp
public class MasterDataService
{
    private readonly IMemoryCache _cache;
    
    public async Task<Aircraft> GetAircraft(string registration, DateTime validAt)
    {
        var cacheKey = $"aircraft:{registration}:{validAt:yyyyMMdd}";
        
        if (_cache.TryGetValue(cacheKey, out Aircraft cached))
        {
            return cached;
        }
        
        var aircraft = await _db.Aircraft
            .Where(a => a.Registration == registration)
            .Where(a => a.ValidFrom <= validAt && validAt < a.ValidTo)
            .FirstOrDefaultAsync();
        
        _cache.Set(cacheKey, aircraft, TimeSpan.FromMinutes(10));
        
        return aircraft;
    }
}
```

## Related Patterns

- **[[Master Data Component]]** - Reference data source for business rules
- **[[Entity Model]]** - Flexible schema that business rules enrich
- **[[Adapter System]]** - Provides messages that trigger rule execution
- **[[Message Types]]** - DTOs that carry data into business rules
- **Rule Engine Pattern** - Decoupled business logic execution
- **Pipeline Pattern** - Sequential rule execution with dependencies

---

**Tags:** #airhart #business-rules #enrichment #validation #alerts #master-data
**Created:** 2025-12-15
**Related:** [[Master Data Component]], [[Entity Model]], [[AODB Messaging]], [[Adapter System]]
**References:**
- [Alert Rules DD130](https://dapproduct.atlassian.net/wiki/spaces/PD/pages/74547273)
- [Enrichment Rules Re-triggered](https://dapproduct.atlassian.net/wiki/spaces/PD/pages/395117339)
