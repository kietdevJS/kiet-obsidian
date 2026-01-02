# AODB Pipeline

> Core data processing pipeline that orchestrates message ingestion, enrichment, validation, and event publishing

## Overview

The **AODB Pipeline** is AIRHART's central nervous system - it receives messages from adapters, enriches flight data through business rules, validates the results, and publishes events to Kafka. Every data change flows through this pipeline.

## Pipeline Architecture

```
┌──────────────────────────────────────────────────────────────┐
│ AODB Pipeline                                                 │
│                                                               │
│  Adapter → Message Queue → Processor → Enrichment →          │
│              ↓               ↓            ↓                   │
│           Validate      Transform    Business Rules           │
│              ↓               ↓            ↓                   │
│           Dedup         Map to Entity  Validate →             │
│              ↓               ↓            ↓                   │
│           Filter        Persist      Publish Events           │
└──────────────────────────────────────────────────────────────┘

Detailed Flow:
┌─────────────┐
│ 1. Receive  │ ← Adapter publishes message
└──────┬──────┘
       ↓
┌─────────────┐
│ 2. Validate │ ← Check message format
└──────┬──────┘
       ↓
┌─────────────┐
│ 3. Dedup    │ ← Check if already processed
└──────┬──────┘
       ↓
┌─────────────┐
│ 4. Match    │ ← Find existing flight or create new
└──────┬──────┘
       ↓
┌─────────────┐
│ 5. Transform│ ← Map message to entity
└──────┬──────┘
       ↓
┌─────────────┐
│ 6. Enrich   │ ← Flight Leg Enrichment
└──────┬──────┘
       ↓
┌─────────────┐
│ 7. Link     │ ← Find/create visit
└──────┬──────┘
       ↓
┌─────────────┐
│ 8. Visit    │ ← Visit Enrichment
│   Enrich    │
└──────┬──────┘
       ↓
┌─────────────┐
│ 9. Validate │ ← Final validation
└──────┬──────┘
       ↓
┌─────────────┐
│10. Persist  │ ← Save to PostgreSQL
└──────┬──────┘
       ↓
┌─────────────┐
│11. Publish  │ ← Publish events to Kafka
└──────┬──────┘
       ↓
┌─────────────┐
│12. Response │ ← Return result to adapter
└─────────────┘
```

## Pipeline Stages

### Stage 1: Message Reception

```csharp
public class MessageReceiver
{
    private readonly IMessageQueue _queue;
    
    public async Task ReceiveMessage(AdapterMessage message)
    {
        // Add to processing queue
        await _queue.Enqueue(new MessageEnvelope
        {
            MessageId = Guid.NewGuid(),
            ReceivedAt = DateTime.UtcNow,
            Source = message.Source,
            MessageType = message.MessageType,
            Payload = message,
            CorrelationId = message.CorrelationId
        });
    }
}
```

### Stage 2: Message Validation

```csharp
public class MessageValidator
{
    public ValidationResult Validate(AdapterMessage message)
    {
        var errors = new List<ValidationError>();
        
        // Required fields
        if (string.IsNullOrEmpty(message.Airline))
            errors.Add(new ValidationError("Airline", "Required"));
        
        if (string.IsNullOrEmpty(message.FlightNumber))
            errors.Add(new ValidationError("FlightNumber", "Required"));
        
        // Format validation
        if (!Regex.IsMatch(message.Airline, @"^[A-Z0-9]{2,3}$"))
            errors.Add(new ValidationError("Airline", "Invalid format"));
        
        // Business validation
        if (message.ScheduledTime < DateTime.UtcNow.AddYears(-1))
            errors.Add(new ValidationError("ScheduledTime", "Too old"));
        
        return new ValidationResult
        {
            IsValid = errors.Count == 0,
            Errors = errors
        };
    }
}
```

### Stage 3: Deduplication

```csharp
public class MessageDeduplicator
{
    private readonly IDistributedCache _cache;
    
    public async Task<bool> IsDuplicate(MessageEnvelope envelope)
    {
        // Generate message fingerprint
        var fingerprint = GenerateFingerprint(envelope);
        var cacheKey = $"msg:{fingerprint}";
        
        // Check if seen recently (last 5 minutes)
        var existing = await _cache.GetStringAsync(cacheKey);
        
        if (existing != null)
        {
            _logger.LogWarning(
                "Duplicate message detected: {MessageId}",
                envelope.MessageId
            );
            return true;
        }
        
        // Mark as seen
        await _cache.SetStringAsync(
            cacheKey,
            envelope.MessageId.ToString(),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
            }
        );
        
        return false;
    }
    
    private string GenerateFingerprint(MessageEnvelope envelope)
    {
        var message = envelope.Payload as MvtMessage;
        
        // Fingerprint based on key fields + timestamp
        var data = $"{message.Airline}|{message.FlightNumber}|" +
                   $"{message.OperationalDate:yyyyMMdd}|" +
                   $"{message.ActualInBlockTime:yyyyMMddHHmm}";
        
        return ComputeHash(data);
    }
}
```

### Stage 4: Flight Matching

```csharp
public class FlightMatcher
{
    private readonly IFlightRepository _repository;
    
    public async Task<FlightLeg> MatchOrCreate(AdapterMessage message)
    {
        // Try to find existing flight
        var existingFlight = await _repository.FindFlight(
            airline: message.Airline,
            flightNumber: message.FlightNumber,
            operationalDate: message.OperationalDate,
            direction: DetermineDirection(message)
        );
        
        if (existingFlight != null)
        {
            _logger.LogInformation(
                "Matched message to existing flight {FlightId}",
                existingFlight.Id
            );
            return existingFlight;
        }
        
        // Create new flight
        var newFlight = new FlightLeg
        {
            Id = Guid.NewGuid(),
            Airline = message.Airline,
            FlightNumber = message.FlightNumber,
            OperationalDate = message.OperationalDate,
            Direction = DetermineDirection(message),
            Source = message.Source,
            CreatedAt = DateTime.UtcNow
        };
        
        _logger.LogInformation(
            "Created new flight {FlightId} for {Airline}{FlightNumber}",
            newFlight.Id,
            message.Airline,
            message.FlightNumber
        );
        
        return newFlight;
    }
}
```

### Stage 5: Message Transformation

```csharp
public class MessageTransformer
{
    public void ApplyMessageToFlight(AdapterMessage message, FlightLeg flight)
    {
        // Update only fields present in message
        if (message is MvtMessage mvt)
        {
            ApplyMvtMessage(mvt, flight);
        }
        else if (message is ScrMessage scr)
        {
            ApplyScrMessage(scr, flight);
        }
        else if (message is FplMessage fpl)
        {
            ApplyFplMessage(fpl, flight);
        }
        
        // Track message source
        flight.LastMessageType = message.MessageType;
        flight.LastMessageReceivedAt = DateTime.UtcNow;
        flight.MessageCount++;
    }
    
    private void ApplyMvtMessage(MvtMessage mvt, FlightLeg flight)
    {
        // Times
        if (mvt.ActualInBlockTime.HasValue)
        {
            flight.ActualInBlockTime = mvt.ActualInBlockTime.Value;
            flight.ActualInBlockTimeSource = mvt.Source;
        }
        
        if (mvt.ActualOffBlockTime.HasValue)
        {
            flight.ActualOffBlockTime = mvt.ActualOffBlockTime.Value;
            flight.ActualOffBlockTimeSource = mvt.Source;
        }
        
        // Aircraft
        if (!string.IsNullOrEmpty(mvt.Registration))
        {
            flight.Registration = mvt.Registration;
            flight.RegistrationSource = mvt.Source;
        }
        
        if (!string.IsNullOrEmpty(mvt.AircraftType))
        {
            flight.TypeIata = mvt.AircraftType;
        }
        
        // Route
        if (!string.IsNullOrEmpty(mvt.DepartureStation))
            flight.DepartureStation = mvt.DepartureStation;
        
        if (!string.IsNullOrEmpty(mvt.ArrivalStation))
            flight.ArrivalStation = mvt.ArrivalStation;
    }
}
```

### Stage 6: Flight Leg Enrichment

```csharp
public class FlightLegEnrichmentStage
{
    private readonly IBusinessRuleEngine _ruleEngine;
    private readonly IPluginManager _pluginManager;
    
    public async Task<EnrichmentResult> Enrich(FlightLeg flight)
    {
        var context = new EnrichmentContext
        {
            FlightLeg = flight,
            StartTime = DateTime.UtcNow
        };
        
        try
        {
            // Pre-enrichment plugins
            await _pluginManager.ExecuteBusinessRulePlugins(
                context,
                flight,
                RuleExecutionPhase.PreEnrichment
            );
            
            // Core enrichment rules (defined order)
            await ExecuteCoreRules(context, flight);
            
            // Flight leg enrichment plugins
            await _pluginManager.ExecuteBusinessRulePlugins(
                context,
                flight,
                RuleExecutionPhase.FlightLegEnrichment
            );
            
            // Validation
            var validationResult = await ValidateFlight(flight);
            
            return new EnrichmentResult
            {
                Success = true,
                Flight = flight,
                ValidationResult = validationResult,
                Duration = DateTime.UtcNow - context.StartTime
            };
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Enrichment failed for flight {FlightId}", flight.Id);
            
            return new EnrichmentResult
            {
                Success = false,
                Error = ex.Message
            };
        }
    }
    
    private async Task ExecuteCoreRules(EnrichmentContext context, FlightLeg flight)
    {
        // Ordered execution
        var rules = new ISubRule[]
        {
            new TransformFlightNumberSubRule(),
            new AssignAircraftDetailsSubRule(),
            new AssignExpectedFlyingTimeSubRule(),
            new SetEstimatedInBlockTimeSubRule(),
            new AssignGroundHandlerSubRule(),
            new AssignPublicFlightIdentifierSubRule(),
            new AssignSecurityLevelSubRule(),
            new AssignTrafficTypeSubRule(),
            new ValidateStandCompatibilitySubRule(),
            // ... more rules
        };
        
        foreach (var rule in rules)
        {
            var result = await rule.Process(context, flight);
            
            if (!result.Success)
            {
                _logger.LogWarning(
                    "Rule {RuleName} failed: {Error}",
                    rule.RuleName,
                    result.ErrorMessage
                );
            }
        }
    }
}
```

### Stage 7: Visit Linking

```csharp
public class VisitLinkingStage
{
    private readonly IVisitRepository _visitRepository;
    private readonly ILinkingRuleEngine _linkingEngine;
    
    public async Task<Visit> LinkFlight(FlightLeg flight)
    {
        // Check if already linked
        if (flight.VisitId.HasValue)
        {
            return await _visitRepository.GetById(flight.VisitId.Value);
        }
        
        // Try to find linkable flight
        var linkableFlights = await FindLinkablFlights(flight);
        
        if (linkableFlights.Any())
        {
            var bestMatch = await _linkingEngine.FindBestMatch(flight, linkableFlights);
            
            if (bestMatch != null)
            {
                return await LinkFlights(flight, bestMatch);
            }
        }
        
        // Create standalone visit
        var visit = new Visit
        {
            Id = Guid.NewGuid(),
            Registration = flight.Registration,
            AircraftType = flight.TypeIata
        };
        
        if (flight.Direction == Direction.Arrival)
            visit.ArrivalFlightId = flight.Id;
        else
            visit.DepartureFlightId = flight.Id;
        
        flight.VisitId = visit.Id;
        
        return visit;
    }
    
    private async Task<Visit> LinkFlights(FlightLeg flight1, FlightLeg flight2)
    {
        var visit = new Visit
        {
            Id = Guid.NewGuid(),
            Registration = flight1.Registration ?? flight2.Registration,
            AircraftType = flight1.TypeIata ?? flight2.TypeIata
        };
        
        if (flight1.Direction == Direction.Arrival)
        {
            visit.ArrivalFlightId = flight1.Id;
            visit.DepartureFlightId = flight2.Id;
        }
        else
        {
            visit.ArrivalFlightId = flight2.Id;
            visit.DepartureFlightId = flight1.Id;
        }
        
        flight1.VisitId = visit.Id;
        flight2.VisitId = visit.Id;
        
        _logger.LogInformation(
            "Linked flights {Flight1} and {Flight2} into visit {VisitId}",
            flight1.FlightNumber,
            flight2.FlightNumber,
            visit.Id
        );
        
        return visit;
    }
}
```

### Stage 8: Visit Enrichment

```csharp
public class VisitEnrichmentStage
{
    public async Task<EnrichmentResult> Enrich(Visit visit)
    {
        var context = new EnrichmentContext { Visit = visit };
        
        // Propagate registration across visit
        await PropagateRegistration(visit);
        
        // Propagate aircraft type
        await PropagateAircraftType(visit);
        
        // Check if re-enrichment needed
        if (visit.PropagationOccurred)
        {
            // Re-run dependent flight leg rules
            await ReEnrichFlights(visit);
        }
        
        // Visit-specific rules
        await ExecuteVisitRules(context, visit);
        
        // Visit enrichment plugins
        await _pluginManager.ExecuteBusinessRulePlugins(
            context,
            visit,
            RuleExecutionPhase.VisitEnrichment
        );
        
        return new EnrichmentResult { Success = true };
    }
    
    private async Task PropagateRegistration(Visit visit)
    {
        var arrival = await GetArrivalFlight(visit);
        var departure = await GetDepartureFlight(visit);
        
        if (arrival == null || departure == null)
            return;
        
        // Determine which flight has better source
        var arrivalPriority = GetSourcePriority(arrival.RegistrationSource);
        var departurePriority = GetSourcePriority(departure.RegistrationSource);
        
        if (arrivalPriority > departurePriority &&
            !string.IsNullOrEmpty(arrival.Registration))
        {
            // Propagate from arrival to departure
            departure.Registration = arrival.Registration;
            departure.RegistrationSource = "Propagation";
            visit.PropagationOccurred = true;
        }
        else if (departurePriority > arrivalPriority &&
                 !string.IsNullOrEmpty(departure.Registration))
        {
            // Propagate from departure to arrival
            arrival.Registration = departure.Registration;
            arrival.RegistrationSource = "Propagation";
            visit.PropagationOccurred = true;
        }
    }
}
```

### Stage 9: Persistence

```csharp
public class PersistenceStage
{
    private readonly IAodbDbContext _dbContext;
    
    public async Task<bool> Persist(FlightLeg flight, Visit visit)
    {
        using var transaction = await _dbContext.Database.BeginTransactionAsync();
        
        try
        {
            // Save flight
            if (flight.Id == Guid.Empty)
                await _dbContext.FlightLegs.AddAsync(flight);
            else
                _dbContext.FlightLegs.Update(flight);
            
            // Save visit
            if (visit != null)
            {
                if (visit.Id == Guid.Empty)
                    await _dbContext.Visits.AddAsync(visit);
                else
                    _dbContext.Visits.Update(visit);
            }
            
            // Save changes
            await _dbContext.SaveChangesAsync();
            
            // Commit transaction
            await transaction.CommitAsync();
            
            _logger.LogInformation(
                "Persisted flight {FlightId} version {Version}",
                flight.Id,
                flight.Version
            );
            
            return true;
        }
        catch (DbUpdateConcurrencyException ex)
        {
            _logger.LogWarning(
                "Concurrency conflict for flight {FlightId}",
                flight.Id
            );
            
            await transaction.RollbackAsync();
            throw new ConcurrencyException("Flight was modified by another process", ex);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to persist flight {FlightId}", flight.Id);
            await transaction.RollbackAsync();
            throw;
        }
    }
}
```

### Stage 10: Event Publishing

```csharp
public class EventPublishingStage
{
    private readonly IKafkaProducer _kafkaProducer;
    
    public async Task PublishEvents(FlightLeg flight, ChangeSet changes)
    {
        var events = GenerateEvents(flight, changes);
        
        foreach (var @event in events)
        {
            await _kafkaProducer.Publish(
                topic: "aodb.flightleg.updated",
                key: flight.Id.ToString(),
                value: @event
            );
            
            _logger.LogInformation(
                "Published event {EventType} for flight {FlightId}",
                @event.EventType,
                flight.Id
            );
        }
    }
    
    private List<DomainEvent> GenerateEvents(FlightLeg flight, ChangeSet changes)
    {
        var events = new List<DomainEvent>();
        
        if (changes.IsNew)
        {
            events.Add(new FlightLegCreated
            {
                FlightLegId = flight.Id,
                FlightNumber = flight.FlightNumber,
                Airline = flight.OperatorIata,
                Direction = flight.Direction
            });
        }
        else
        {
            events.Add(new FlightLegUpdated
            {
                FlightLegId = flight.Id,
                Changes = changes.GetChanges()
            });
        }
        
        // Alert events
        foreach (var alert in flight.Alerts.Where(a => a.IsNew))
        {
            events.Add(new AlertRaised
            {
                FlightLegId = flight.Id,
                AlertType = alert.AlertType,
                AlertMessage = alert.AlertMessage,
                AlertLevel = alert.AlertLevel
            });
        }
        
        return events;
    }
}
```

## Error Handling

```csharp
public class PipelineErrorHandler
{
    public async Task<ProcessResult> HandleError(
        Exception exception,
        MessageEnvelope envelope,
        PipelineStage failedStage)
    {
        _logger.LogError(
            exception,
            "Pipeline failed at stage {Stage} for message {MessageId}",
            failedStage,
            envelope.MessageId
        );
        
        // Determine if retryable
        if (IsRetryable(exception))
        {
            return await RetryMessage(envelope, failedStage);
        }
        
        // Move to dead letter queue
        await SendToDeadLetterQueue(envelope, exception);
        
        return ProcessResult.Failure($"Pipeline failed: {exception.Message}");
    }
    
    private bool IsRetryable(Exception exception)
    {
        return exception is DbUpdateException ||
               exception is TimeoutException ||
               exception is HttpRequestException;
    }
    
    private async Task<ProcessResult> RetryMessage(
        MessageEnvelope envelope,
        PipelineStage failedStage)
    {
        envelope.RetryCount++;
        envelope.LastRetryAt = DateTime.UtcNow;
        
        if (envelope.RetryCount > 3)
        {
            await SendToDeadLetterQueue(envelope, new Exception("Max retries exceeded"));
            return ProcessResult.Failure("Max retries exceeded");
        }
        
        // Exponential backoff
        var delay = TimeSpan.FromSeconds(Math.Pow(2, envelope.RetryCount));
        await Task.Delay(delay);
        
        // Re-queue from failed stage
        await RequeueMessage(envelope, failedStage);
        
        return ProcessResult.Retry();
    }
}
```

## Performance Monitoring

```csharp
public class PipelineMetrics
{
    private readonly IMetrics _metrics;
    
    public void RecordProcessingTime(TimeSpan duration, PipelineStage stage)
    {
        _metrics.Histogram(
            "pipeline.stage.duration",
            duration.TotalMilliseconds,
            tags: new[] { $"stage:{stage}" }
        );
    }
    
    public void RecordThroughput(int messagesProcessed)
    {
        _metrics.Counter(
            "pipeline.messages.processed",
            messagesProcessed
        );
    }
    
    public void RecordError(PipelineStage stage, string errorType)
    {
        _metrics.Counter(
            "pipeline.errors",
            tags: new[] { $"stage:{stage}", $"error:{errorType}" }
        );
    }
}
```

## Related Patterns

- **[[Business Rules]]** - Execute enrichment logic
- **[[Message Types]]** - Process different message formats
- **[[Event Sourcing]]** - Publish domain events
- **Pipeline Pattern** - Sequential processing stages
- **Saga Pattern** - Coordinate distributed transactions

---

**Tags:** #airhart #pipeline #aodb #processing #enrichment
**Created:** 2025-12-15
**Related:** [[Business Rules]], [[Message Types]], [[Event Sourcing]]
