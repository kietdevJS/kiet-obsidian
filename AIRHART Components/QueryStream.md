# QueryStream

> Real-time queryable read model providing flight and visit data with subscriptions, feeds, and OData queries

## Overview

**QueryStream** is AIRHART's read side in the CQRS pattern. It maintains denormalized, queryable projections of flight data optimized for fast reads. Clients subscribe to QueryStream feeds to receive real-time updates via SignalR, and can query using OData.

## Architecture

```
┌───────────────────────────────────────────────────────────┐
│ QueryStream Architecture                                   │
│                                                           │
│  Kafka Events → Projection Engine → Document DB          │
│                       ↓                    ↓              │
│                  Projections          Feeds & Queries     │
│                       ↓                    ↓              │
│                  Update Docs          SignalR Push        │
│                                            ↓              │
│                                       Client Apps         │
└───────────────────────────────────────────────────────────┘

Data Flow:
┌─────────────┐
│ AODB        │ Publishes events
│ (Write)     │───────────────┐
└─────────────┘               │
                              ↓
                    ┌──────────────────┐
                    │ Kafka            │
                    │ Event Topics     │
                    └────────┬─────────┘
                             │ Consume events
                             ↓
                    ┌──────────────────┐
                    │ Projection       │
                    │ Engine           │
                    └────────┬─────────┘
                             │ Update documents
                             ↓
                    ┌──────────────────┐
                    │ Document DB      │
                    │ (Denormalized)   │
                    └────────┬─────────┘
                             │ Query & Subscribe
                             ↓
                    ┌──────────────────┐
                    │ QueryStream API  │
                    │ - OData Queries  │
                    │ - Subscriptions  │
                    │ - SignalR Feeds  │
                    └────────┬─────────┘
                             │ HTTP / WebSocket
                             ↓
                    ┌──────────────────┐
                    │ Client Apps      │
                    └──────────────────┘
```

## Core Concepts

### 1. Projections

Projections consume Kafka events and build denormalized read models:

```csharp
public class FlightLegProjection : IProjection
{
    private readonly IDocumentRepository _repository;
    
    public async Task Handle(FlightLegCreated @event)
    {
        var document = new FlightLegDocument
        {
            Id = @event.FlightLegId,
            FlightNumber = @event.FlightNumber,
            Airline = @event.Airline,
            Direction = @event.Direction,
            CreatedAt = @event.Timestamp,
            Version = 1
        };
        
        await _repository.Insert(document);
    }
    
    public async Task Handle(FlightLegUpdated @event)
    {
        var document = await _repository.GetById(@event.FlightLegId);
        
        // Apply changes
        foreach (var change in @event.Changes)
        {
            ApplyChange(document, change);
        }
        
        document.Version++;
        document.LastModifiedAt = @event.Timestamp;
        
        await _repository.Update(document);
        
        // Notify subscribers
        await NotifySubscribers(document);
    }
    
    private void ApplyChange(FlightLegDocument doc, FieldChange change)
    {
        // Use reflection or property mapper
        var property = doc.GetType().GetProperty(change.FieldName);
        property?.SetValue(doc, change.NewValue);
    }
}
```

### 2. Document Model

Documents are denormalized for efficient querying:

```csharp
public class FlightLegDocument
{
    public Guid Id { get; set; }
    
    // Flight identity
    public string Airline { get; set; }
    public string FlightNumber { get; set; }
    public DateTime OperationalDate { get; set; }
    public Direction Direction { get; set; }
    
    // Times (denormalized)
    public DateTime? ScheduledTime { get; set; }
    public DateTime? EstimatedTime { get; set; }
    public DateTime? ActualTime { get; set; }
    public DateTime? InBlockTime { get; set; }
    public DateTime? OffBlockTime { get; set; }
    
    // Route
    public string DepartureStation { get; set; }
    public string ArrivalStation { get; set; }
    
    // Aircraft (denormalized from Master Data)
    public string Registration { get; set; }
    public string AircraftType { get; set; }
    public string AircraftTypeName { get; set; }
    public string AircraftConfiguration { get; set; }
    
    // Stand/Gate (denormalized)
    public string Stand { get; set; }
    public string StandType { get; set; }
    public string Gate { get; set; }
    public string Terminal { get; set; }
    
    // Status
    public FlightStatus Status { get; set; }
    public string StatusReason { get; set; }
    
    // Visit (denormalized)
    public Guid? VisitId { get; set; }
    public string VisitRegistration { get; set; }
    public Guid? LinkedFlightId { get; set; }
    public string LinkedFlightNumber { get; set; }
    
    // Alerts (embedded)
    public List<AlertDocument> Alerts { get; set; }
    
    // Metadata
    public int Version { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime LastModifiedAt { get; set; }
    
    // Full-text search
    public string SearchText { get; set; }  // Combined searchable fields
}

public class AlertDocument
{
    public Guid AlertId { get; set; }
    public string AlertType { get; set; }
    public string AlertMessage { get; set; }
    public AlertLevel AlertLevel { get; set; }
    public DateTime RaisedAt { get; set; }
    public bool IsAcknowledged { get; set; }
}
```

### 3. Subscription Manager

Clients subscribe to specific flights or filters:

```csharp
public class SubscriptionManager
{
    private readonly ConcurrentDictionary<string, Subscription> _subscriptions;
    private readonly IHubContext<QueryStreamHub> _hubContext;
    
    public async Task<Guid> Subscribe(SubscriptionRequest request)
    {
        var subscription = new Subscription
        {
            Id = Guid.NewGuid(),
            ConnectionId = request.ConnectionId,
            Filter = request.Filter,
            CreatedAt = DateTime.UtcNow
        };
        
        _subscriptions[subscription.Id.ToString()] = subscription;
        
        _logger.LogInformation(
            "Client {ConnectionId} subscribed with filter: {Filter}",
            request.ConnectionId,
            request.Filter
        );
        
        // Send initial data
        var initialData = await GetMatchingDocuments(request.Filter);
        await SendToClient(request.ConnectionId, initialData);
        
        return subscription.Id;
    }
    
    public async Task Unsubscribe(Guid subscriptionId)
    {
        if (_subscriptions.TryRemove(subscriptionId.ToString(), out var subscription))
        {
            _logger.LogInformation(
                "Client {ConnectionId} unsubscribed",
                subscription.ConnectionId
            );
        }
    }
    
    public async Task NotifySubscribers(FlightLegDocument document)
    {
        var matchingSubscriptions = _subscriptions.Values
            .Where(sub => MatchesFilter(document, sub.Filter))
            .ToList();
        
        foreach (var subscription in matchingSubscriptions)
        {
            await SendToClient(subscription.ConnectionId, document);
        }
        
        _logger.LogDebug(
            "Notified {Count} subscribers about flight {FlightId}",
            matchingSubscriptions.Count,
            document.Id
        );
    }
    
    private bool MatchesFilter(FlightLegDocument document, SubscriptionFilter filter)
    {
        if (filter.FlightIds != null && !filter.FlightIds.Contains(document.Id))
            return false;
        
        if (filter.Airlines != null && !filter.Airlines.Contains(document.Airline))
            return false;
        
        if (filter.Direction.HasValue && document.Direction != filter.Direction)
            return false;
        
        if (filter.DateRange != null)
        {
            if (document.OperationalDate < filter.DateRange.From ||
                document.OperationalDate > filter.DateRange.To)
                return false;
        }
        
        return true;
    }
}
```

### 4. SignalR Hub

Real-time push to connected clients:

```csharp
public class QueryStreamHub : Hub
{
    private readonly ISubscriptionManager _subscriptionManager;
    private readonly IQueryStreamRepository _repository;
    
    public async Task<Guid> Subscribe(SubscriptionFilter filter)
    {
        var subscriptionId = await _subscriptionManager.Subscribe(new SubscriptionRequest
        {
            ConnectionId = Context.ConnectionId,
            Filter = filter
        });
        
        _logger.LogInformation(
            "Client {ConnectionId} created subscription {SubscriptionId}",
            Context.ConnectionId,
            subscriptionId
        );
        
        return subscriptionId;
    }
    
    public async Task Unsubscribe(Guid subscriptionId)
    {
        await _subscriptionManager.Unsubscribe(subscriptionId);
    }
    
    public override async Task OnDisconnectedAsync(Exception exception)
    {
        await _subscriptionManager.RemoveConnectionSubscriptions(Context.ConnectionId);
        
        await base.OnDisconnectedAsync(exception);
    }
    
    // Server calls this to push updates
    public async Task SendFlightUpdate(FlightLegDocument flight)
    {
        await Clients.Client(Context.ConnectionId).SendAsync("FlightUpdated", flight);
    }
    
    public async Task SendFlightDeleted(Guid flightId)
    {
        await Clients.Client(Context.ConnectionId).SendAsync("FlightDeleted", flightId);
    }
}
```

### 5. OData Query Support

Flexible querying with OData protocol:

```csharp
[ApiController]
[Route("api/querystream/flights")]
public class FlightQueryController : ControllerBase
{
    private readonly IQueryStreamRepository _repository;
    
    [HttpGet]
    [EnableQuery(MaxExpansionDepth = 3, PageSize = 100)]
    public IQueryable<FlightLegDocument> GetFlights()
    {
        return _repository.GetQueryable();
    }
    
    [HttpGet("{id}")]
    public async Task<ActionResult<FlightLegDocument>> GetFlight(Guid id)
    {
        var flight = await _repository.GetById(id);
        
        if (flight == null)
            return NotFound();
        
        return flight;
    }
}

// Client usage:
// GET /api/querystream/flights?$filter=airline eq 'SK' and direction eq 'Arrival'
// GET /api/querystream/flights?$filter=operationalDate ge 2025-01-01
// GET /api/querystream/flights?$orderby=scheduledTime desc&$top=20
// GET /api/querystream/flights?$search="LN-ABC"
// GET /api/querystream/flights?$expand=alerts
```

## Client Integration

### JavaScript Client

```javascript
class QueryStreamClient {
    constructor(baseUrl) {
        this.baseUrl = baseUrl;
        this.connection = null;
        this.subscriptions = new Map();
    }
    
    async connect() {
        this.connection = new signalR.HubConnectionBuilder()
            .withUrl(`${this.baseUrl}/querystream`)
            .withAutomaticReconnect()
            .build();
        
        this.connection.on("FlightUpdated", (flight) => {
            this.handleFlightUpdate(flight);
        });
        
        this.connection.on("FlightDeleted", (flightId) => {
            this.handleFlightDeleted(flightId);
        });
        
        await this.connection.start();
        console.log("Connected to QueryStream");
    }
    
    async subscribe(filter, callback) {
        const subscriptionId = await this.connection.invoke("Subscribe", filter);
        
        this.subscriptions.set(subscriptionId, callback);
        
        return subscriptionId;
    }
    
    async unsubscribe(subscriptionId) {
        await this.connection.invoke("Unsubscribe", subscriptionId);
        this.subscriptions.delete(subscriptionId);
    }
    
    handleFlightUpdate(flight) {
        // Notify all matching subscriptions
        for (const [subscriptionId, callback] of this.subscriptions) {
            callback({ type: 'update', flight });
        }
    }
    
    handleFlightDeleted(flightId) {
        for (const [subscriptionId, callback] of this.subscriptions) {
            callback({ type: 'delete', flightId });
        }
    }
    
    async query(odataQuery) {
        const response = await fetch(
            `${this.baseUrl}/api/querystream/flights?${odataQuery}`
        );
        return await response.json();
    }
}

// Usage:
const client = new QueryStreamClient('https://airhart.api');
await client.connect();

// Subscribe to arrivals
const subscriptionId = await client.subscribe(
    { direction: 'Arrival', airlines: ['SK', 'DY'] },
    (event) => {
        if (event.type === 'update') {
            updateFlightBoard(event.flight);
        } else if (event.type === 'delete') {
            removeFlightFromBoard(event.flightId);
        }
    }
);

// Query for specific flights
const flights = await client.query(
    "$filter=operationalDate eq 2025-01-15&$orderby=scheduledTime"
);
```

### C# Client

```csharp
public class QueryStreamClient
{
    private readonly HubConnection _connection;
    private readonly HttpClient _httpClient;
    
    public async Task ConnectAsync()
    {
        _connection = new HubConnectionBuilder()
            .WithUrl("https://airhart.api/querystream")
            .WithAutomaticReconnect()
            .Build();
        
        _connection.On<FlightLegDocument>("FlightUpdated", flight =>
        {
            OnFlightUpdated?.Invoke(this, flight);
        });
        
        _connection.On<Guid>("FlightDeleted", flightId =>
        {
            OnFlightDeleted?.Invoke(this, flightId);
        });
        
        await _connection.StartAsync();
    }
    
    public async Task<Guid> SubscribeAsync(SubscriptionFilter filter)
    {
        return await _connection.InvokeAsync<Guid>("Subscribe", filter);
    }
    
    public async Task<List<FlightLegDocument>> QueryAsync(string odataQuery)
    {
        var response = await _httpClient.GetAsync(
            $"api/querystream/flights?{odataQuery}"
        );
        
        response.EnsureSuccessStatusCode();
        
        return await response.Content.ReadAsAsync<List<FlightLegDocument>>();
    }
    
    public event EventHandler<FlightLegDocument> OnFlightUpdated;
    public event EventHandler<Guid> OnFlightDeleted;
}

// Usage:
var client = new QueryStreamClient();
await client.ConnectAsync();

client.OnFlightUpdated += (sender, flight) =>
{
    Console.WriteLine($"Flight updated: {flight.FlightNumber}");
    UpdateUI(flight);
};

var subscriptionId = await client.SubscribeAsync(new SubscriptionFilter
{
    Direction = Direction.Arrival,
    Airlines = new[] { "SK", "DY" }
});

var todaysFlights = await client.QueryAsync(
    "$filter=operationalDate eq 2025-01-15&$orderby=scheduledTime"
);
```

## Projection Engine

```csharp
public class ProjectionEngine
{
    private readonly IKafkaConsumer _consumer;
    private readonly Dictionary<string, IProjection> _projections;
    
    public async Task Start()
    {
        _consumer.Subscribe(new[] {
            "aodb.flightleg.created",
            "aodb.flightleg.updated",
            "aodb.flightleg.deleted",
            "aodb.visit.created",
            "aodb.visit.updated"
        });
        
        while (true)
        {
            var message = await _consumer.Consume();
            
            try
            {
                await ProcessEvent(message);
                await _consumer.Commit(message);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to process event {EventId}", message.Key);
                // Don't commit, will retry
            }
        }
    }
    
    private async Task ProcessEvent(ConsumeResult<string, string> message)
    {
        var eventType = message.Headers
            .FirstOrDefault(h => h.Key == "EventType")
            ?.GetValueBytes()
            ?.ToString();
        
        if (!_projections.TryGetValue(eventType, out var projection))
        {
            _logger.LogWarning("No projection for event type {EventType}", eventType);
            return;
        }
        
        // Deserialize event
        var @event = JsonSerializer.Deserialize(message.Value, GetEventType(eventType));
        
        // Apply projection
        await projection.Handle(@event);
        
        _logger.LogDebug("Processed event {EventType} with key {Key}", eventType, message.Key);
    }
}
```

## Performance & Scaling

```csharp
public class QueryStreamPerformance
{
    // Caching frequently accessed documents
    private readonly IDistributedCache _cache;
    
    public async Task<FlightLegDocument> GetFlight(Guid flightId)
    {
        var cacheKey = $"flight:{flightId}";
        
        var cached = await _cache.GetStringAsync(cacheKey);
        if (cached != null)
        {
            return JsonSerializer.Deserialize<FlightLegDocument>(cached);
        }
        
        var document = await _repository.GetById(flightId);
        
        if (document != null)
        {
            await _cache.SetStringAsync(
                cacheKey,
                JsonSerializer.Serialize(document),
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
                }
            );
        }
        
        return document;
    }
    
    // Horizontal scaling with partitioned subscriptions
    public async Task NotifySubscribers(FlightLegDocument document)
    {
        // Partition subscriptions across servers
        var partitionKey = document.Airline; // or hash of FlightId
        
        await _messageBus.Publish(
            topic: $"querystream.updates.{partitionKey}",
            message: document
        );
    }
}
```

## Related Patterns

- **[[CQRS Pattern]]** - QueryStream is the read side
- **[[Event Sourcing]]** - Consumes events from Kafka
- **[[Eventual Consistency]]** - Eventually consistent with AODB
- **Observer Pattern** - Subscription mechanism
- **Materialized View Pattern** - Denormalized documents

---

**Tags:** #airhart #querystream #cqrs #read-model #subscriptions #signalr
**Created:** 2025-12-15
**Related:** [[CQRS Pattern]], [[Event Sourcing]], [[Eventual Consistency]]
