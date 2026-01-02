# Plugin Architecture

> Extensible plugin system that allows dynamic loading of custom business logic without modifying core code

## Overview

AIRHART's **Plugin Architecture** enables extending functionality through **plugins** - self-contained modules that can be dynamically loaded and executed at runtime. Plugins allow airports to customize behavior without modifying the core system.

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│ AIRHART Core                                            │
│ ├─ Plugin Manager                                       │
│ ├─ Plugin Registry                                      │
│ ├─ Plugin Loader                                        │
│ └─ Plugin Host                                          │
└────────────────┬────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────┐
│ Plugin Interfaces                                        │
│ ├─ IBusinessRulePlugin                                  │
│ ├─ IValidationPlugin                                    │
│ ├─ IEnrichmentPlugin                                    │
│ ├─ IQueryStreamPlugin                                   │
│ └─ IAdapterPlugin                                       │
└────────────────┬────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────┐
│ Custom Plugins (DLLs)                                   │
│ ├─ CustomValidation.dll                                 │
│ ├─ AirportSpecificRules.dll                            │
│ ├─ CustomQueryStreamProjection.dll                     │
│ └─ SpecializedAdapter.dll                              │
└─────────────────────────────────────────────────────────┘

Flow:
1. System starts → Plugin Manager scans plugin directory
2. Plugins loaded dynamically via Assembly.LoadFrom()
3. Plugin instances registered in DI container
4. Plugins executed at defined extension points
```

## Core Concepts

### 1. Plugin Interface

Base interface all plugins implement:

```csharp
public interface IPlugin
{
    // Identity
    string Name { get; }
    string Version { get; }
    string Author { get; }
    string Description { get; }
    
    // Lifecycle
    Task Initialize(IPluginContext context);
    Task Shutdown();
    
    // Metadata
    PluginMetadata GetMetadata();
}

public class PluginMetadata
{
    public string Name { get; set; }
    public string Version { get; set; }
    public string Author { get; set; }
    public string Description { get; set; }
    public string[] Dependencies { get; set; }
    public Dictionary<string, string> Configuration { get; set; }
}
```

### 2. Plugin Types

#### Business Rule Plugin

Extends enrichment logic:

```csharp
public interface IBusinessRulePlugin : IPlugin
{
    // Execution
    Task<ProcessResult> Execute(
        EnrichmentContext context,
        FlightLeg flightLeg
    );
    
    // When to run
    RuleExecutionPhase Phase { get; }
    int Priority { get; }
}

public enum RuleExecutionPhase
{
    PreEnrichment,
    FlightLegEnrichment,
    VisitEnrichment,
    PostEnrichment
}

// Example: Custom priority calculation
public class CustomPriorityPlugin : IBusinessRulePlugin
{
    public string Name => "Custom Priority Calculator";
    public string Version => "1.0.0";
    public RuleExecutionPhase Phase => RuleExecutionPhase.FlightLegEnrichment;
    public int Priority => 100;
    
    public async Task<ProcessResult> Execute(
        EnrichmentContext context,
        FlightLeg flightLeg)
    {
        // Custom business logic
        int priority = CalculatePriority(flightLeg);
        
        // Set extended property
        flightLeg["CustomPriority"] = priority;
        
        _logger.LogInformation(
            "Assigned priority {Priority} to flight {FlightNumber}",
            priority,
            flightLeg.FlightNumber
        );
        
        return ProcessResult.Success();
    }
    
    private int CalculatePriority(FlightLeg flight)
    {
        int priority = 5; // Default
        
        // VIP airlines get higher priority
        if (flight.OperatorIata == "SK")
            priority += 2;
        
        // Peak hours get lower priority
        var hour = flight.BestOperationTime.Hour;
        if (hour >= 6 && hour <= 9)
            priority -= 1;
        
        // Delays increase priority
        if (flight.IsDelayed)
            priority += 3;
        
        return Math.Clamp(priority, 1, 10);
    }
    
    public Task Initialize(IPluginContext context)
    {
        _logger = context.GetLogger<CustomPriorityPlugin>();
        return Task.CompletedTask;
    }
}
```

#### Validation Plugin

Custom validation rules:

```csharp
public interface IValidationPlugin : IPlugin
{
    Task<ValidationResult> Validate(FlightLeg flightLeg);
}

// Example: Airport-specific validation
public class CopenhagenValidationPlugin : IValidationPlugin
{
    public string Name => "Copenhagen Validation Rules";
    public string Version => "1.0.0";
    
    public async Task<ValidationResult> Validate(FlightLeg flightLeg)
    {
        var errors = new List<ValidationError>();
        
        // CPH-specific: Check for dangerous goods notification
        if (HasDangerousGoods(flightLeg) && 
            !HasDangerousGoodsNotification(flightLeg))
        {
            errors.Add(new ValidationError
            {
                Field = "DangerousGoods",
                Message = "Dangerous goods require advance notification",
                Severity = ErrorSeverity.High
            });
        }
        
        // CPH-specific: Terminal 3 only for certain airlines
        if (flightLeg.ArrivalGate?.StartsWith("T3") == true &&
            !IsAllowedInTerminal3(flightLeg.OperatorIata))
        {
            errors.Add(new ValidationError
            {
                Field = "ArrivalGate",
                Message = "Airline not permitted in Terminal 3",
                Severity = ErrorSeverity.High
            });
        }
        
        return new ValidationResult(errors);
    }
}
```

#### QueryStream Plugin

Custom read model projections:

```csharp
public interface IQueryStreamPlugin : IPlugin
{
    // Event handling
    Task HandleEvent(DomainEvent @event);
    
    // Projection
    Task<object> Project(Guid entityId);
    
    // Subscription
    string[] SubscribedEventTypes { get; }
}

// Example: Custom statistics projection
public class FlightStatisticsPlugin : IQueryStreamPlugin
{
    public string Name => "Flight Statistics Projector";
    public string[] SubscribedEventTypes => new[]
    {
        "FlightLegCreated",
        "FlightLegUpdated",
        "FlightLegDeleted"
    };
    
    private readonly IDocumentStore _documentStore;
    
    public async Task HandleEvent(DomainEvent @event)
    {
        switch (@event)
        {
            case FlightLegCreated created:
                await IncrementFlightCount(created.Airline);
                break;
                
            case FlightLegUpdated updated:
                await UpdateStatistics(updated);
                break;
                
            case FlightLegDeleted deleted:
                await DecrementFlightCount(deleted.FlightLegId);
                break;
        }
    }
    
    public async Task<object> Project(Guid entityId)
    {
        // Return custom projection
        return await _documentStore.GetById<FlightStatistics>(entityId);
    }
    
    private async Task IncrementFlightCount(string airline)
    {
        var stats = await GetOrCreateStatistics(airline);
        stats.TotalFlights++;
        stats.LastUpdated = DateTime.UtcNow;
        await _documentStore.Save(stats);
    }
}
```

#### Adapter Plugin

Custom adapter implementations:

```csharp
public interface IAdapterPlugin : IPlugin
{
    // Message processing
    Task<ProcessResult> ProcessMessage(string rawMessage);
    
    // Capabilities
    string[] SupportedMessageTypes { get; }
    bool CanHandle(string messageType);
}

// Example: Custom weather adapter
public class WeatherAdapterPlugin : IAdapterPlugin
{
    public string Name => "Weather Data Adapter";
    public string[] SupportedMessageTypes => new[] { "METAR", "TAF" };
    
    public async Task<ProcessResult> ProcessMessage(string rawMessage)
    {
        if (rawMessage.StartsWith("METAR"))
        {
            var weather = ParseMetar(rawMessage);
            await PublishWeatherUpdate(weather);
        }
        else if (rawMessage.StartsWith("TAF"))
        {
            var forecast = ParseTaf(rawMessage);
            await PublishWeatherForecast(forecast);
        }
        
        return ProcessResult.Success();
    }
    
    public bool CanHandle(string messageType)
    {
        return SupportedMessageTypes.Contains(messageType);
    }
}
```

## Plugin Lifecycle

```
1. Discovery Phase
   ├─ Scan plugin directory
   ├─ Load plugin assemblies
   └─ Discover plugin types

2. Validation Phase
   ├─ Verify plugin interface implementation
   ├─ Check dependencies
   └─ Validate metadata

3. Registration Phase
   ├─ Register in DI container
   ├─ Subscribe to events
   └─ Configure plugin

4. Initialization Phase
   ├─ Call Initialize()
   ├─ Load configuration
   └─ Setup resources

5. Execution Phase
   ├─ Plugin executes at extension points
   └─ Responds to events

6. Shutdown Phase
   ├─ Call Shutdown()
   ├─ Cleanup resources
   └─ Unload assembly
```

## Plugin Manager

Central orchestrator:

```csharp
public class PluginManager : IPluginManager
{
    private readonly Dictionary<string, IPlugin> _plugins = new();
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<PluginManager> _logger;
    
    public async Task LoadPlugins(string pluginDirectory)
    {
        _logger.LogInformation("Loading plugins from {Directory}", pluginDirectory);
        
        // 1. Discover plugin DLLs
        var pluginFiles = Directory.GetFiles(pluginDirectory, "*.dll");
        
        foreach (var pluginFile in pluginFiles)
        {
            try
            {
                // 2. Load assembly
                var assembly = Assembly.LoadFrom(pluginFile);
                
                // 3. Find plugin types
                var pluginTypes = assembly.GetTypes()
                    .Where(t => typeof(IPlugin).IsAssignableFrom(t) && !t.IsAbstract);
                
                foreach (var pluginType in pluginTypes)
                {
                    // 4. Create instance
                    var plugin = (IPlugin)Activator.CreateInstance(pluginType);
                    
                    // 5. Validate
                    if (!ValidatePlugin(plugin))
                    {
                        _logger.LogWarning("Plugin {Name} failed validation", plugin.Name);
                        continue;
                    }
                    
                    // 6. Initialize
                    var context = new PluginContext(_serviceProvider);
                    await plugin.Initialize(context);
                    
                    // 7. Register
                    _plugins[plugin.Name] = plugin;
                    
                    _logger.LogInformation(
                        "Loaded plugin: {Name} v{Version} by {Author}",
                        plugin.Name,
                        plugin.Version,
                        plugin.Author
                    );
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to load plugin from {File}", pluginFile);
            }
        }
        
        _logger.LogInformation("Loaded {Count} plugins", _plugins.Count);
    }
    
    public async Task ExecuteBusinessRulePlugins(
        EnrichmentContext context,
        FlightLeg flightLeg,
        RuleExecutionPhase phase)
    {
        var plugins = _plugins.Values
            .OfType<IBusinessRulePlugin>()
            .Where(p => p.Phase == phase)
            .OrderBy(p => p.Priority);
        
        foreach (var plugin in plugins)
        {
            try
            {
                var result = await plugin.Execute(context, flightLeg);
                
                if (!result.Success)
                {
                    _logger.LogWarning(
                        "Plugin {Name} failed: {Error}",
                        plugin.Name,
                        result.ErrorMessage
                    );
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(
                    ex,
                    "Plugin {Name} threw exception",
                    plugin.Name
                );
            }
        }
    }
    
    public IPlugin GetPlugin(string name)
    {
        return _plugins.TryGetValue(name, out var plugin) ? plugin : null;
    }
    
    public IEnumerable<IPlugin> GetPlugins<T>() where T : IPlugin
    {
        return _plugins.Values.OfType<T>();
    }
}
```

## Plugin Context

Provides services to plugins:

```csharp
public interface IPluginContext
{
    // Logging
    ILogger<T> GetLogger<T>();
    
    // Configuration
    IConfiguration GetConfiguration();
    string GetSetting(string key);
    
    // Services
    T GetService<T>();
    
    // Data access
    IRepository<TEntity> GetRepository<TEntity>();
    
    // Events
    Task PublishEvent(DomainEvent @event);
    void SubscribeToEvent<TEvent>(Func<TEvent, Task> handler) where TEvent : DomainEvent;
}

public class PluginContext : IPluginContext
{
    private readonly IServiceProvider _serviceProvider;
    
    public PluginContext(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }
    
    public ILogger<T> GetLogger<T>()
    {
        return _serviceProvider.GetRequiredService<ILogger<T>>();
    }
    
    public T GetService<T>()
    {
        return _serviceProvider.GetRequiredService<T>();
    }
    
    public IRepository<TEntity> GetRepository<TEntity>()
    {
        return _serviceProvider.GetRequiredService<IRepository<TEntity>>();
    }
    
    public async Task PublishEvent(DomainEvent @event)
    {
        var eventBus = _serviceProvider.GetRequiredService<IEventBus>();
        await eventBus.Publish(@event);
    }
}
```

## Plugin Configuration

Plugins configured via JSON:

```json
{
  "Plugins": {
    "Directory": "./plugins",
    "AutoLoad": true,
    "Enabled": [
      "CustomPriorityPlugin",
      "CopenhagenValidationPlugin",
      "FlightStatisticsPlugin"
    ],
    "Configurations": {
      "CustomPriorityPlugin": {
        "VipAirlines": ["SK", "DY", "AY"],
        "PeakHours": [6, 7, 8, 9]
      },
      "FlightStatisticsPlugin": {
        "AggregationInterval": "5m",
        "RetentionDays": 30
      }
    }
  }
}
```

## Extension Points

Plugins hook into specific points:

```csharp
public class FlightLegEnrichmentPipeline
{
    private readonly IPluginManager _pluginManager;
    
    public async Task<FlightLeg> Enrich(FlightLeg flightLeg)
    {
        var context = new EnrichmentContext();
        
        // Extension point 1: Pre-enrichment
        await _pluginManager.ExecuteBusinessRulePlugins(
            context,
            flightLeg,
            RuleExecutionPhase.PreEnrichment
        );
        
        // Core enrichment rules
        await ExecuteCoreRules(flightLeg);
        
        // Extension point 2: Flight leg enrichment
        await _pluginManager.ExecuteBusinessRulePlugins(
            context,
            flightLeg,
            RuleExecutionPhase.FlightLegEnrichment
        );
        
        // Extension point 3: Validation
        await ExecuteValidationPlugins(flightLeg);
        
        return flightLeg;
    }
    
    private async Task ExecuteValidationPlugins(FlightLeg flightLeg)
    {
        var validationPlugins = _pluginManager.GetPlugins<IValidationPlugin>();
        
        foreach (var plugin in validationPlugins)
        {
            var result = await plugin.Validate(flightLeg);
            
            if (!result.IsValid)
            {
                // Add validation errors as alerts
                foreach (var error in result.Errors)
                {
                    flightLeg.AddAlert(
                        "PLUGIN_" + plugin.Name.ToUpper(),
                        error.Message,
                        MapSeverity(error.Severity)
                    );
                }
            }
        }
    }
}
```

## Plugin Isolation

Plugins run in isolated contexts:

```csharp
public class IsolatedPluginExecutor
{
    public async Task<ProcessResult> Execute(
        IBusinessRulePlugin plugin,
        EnrichmentContext context,
        FlightLeg flightLeg)
    {
        // Clone flight leg to prevent side effects
        var clonedFlight = flightLeg.Clone();
        
        try
        {
            // Execute with timeout
            var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
            var result = await plugin.Execute(context, clonedFlight)
                .WithCancellation(cts.Token);
            
            if (result.Success)
            {
                // Merge changes back
                MergeChanges(flightLeg, clonedFlight);
            }
            
            return result;
        }
        catch (OperationCanceledException)
        {
            return ProcessResult.Failure("Plugin execution timed out");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Plugin {Name} failed", plugin.Name);
            return ProcessResult.Failure($"Plugin failed: {ex.Message}");
        }
    }
}
```

## Hot Reload

Plugins can be reloaded without restart:

```csharp
public class PluginHotReloader
{
    private readonly IPluginManager _pluginManager;
    private readonly FileSystemWatcher _watcher;
    
    public void EnableHotReload(string pluginDirectory)
    {
        _watcher = new FileSystemWatcher(pluginDirectory, "*.dll");
        _watcher.Changed += OnPluginChanged;
        _watcher.EnableRaisingEvents = true;
    }
    
    private async void OnPluginChanged(object sender, FileSystemEventArgs e)
    {
        _logger.LogInformation("Plugin file changed: {File}", e.Name);
        
        // Wait for file to be fully written
        await Task.Delay(1000);
        
        // Reload plugin
        await _pluginManager.ReloadPlugin(e.FullPath);
    }
}
```

## Benefits

### 1. Extensibility Without Code Changes

```
Old approach:
Custom logic → Modify AODB code → Rebuild → Deploy

New approach:
Custom logic → Create plugin DLL → Drop in folder → Done
```

### 2. Airport-Specific Customization

Each airport can have unique plugins:

```
Copenhagen:
├─ CPH.CustomPriority.dll
├─ CPH.SpecialValidation.dll
└─ CPH.WeatherIntegration.dll

Stockholm:
├─ ARN.CustomHandling.dll
└─ ARN.NordicRules.dll
```

### 3. A/B Testing

Deploy multiple versions:

```
plugins/
├─ PriorityCalculator.v1.dll (50% traffic)
└─ PriorityCalculator.v2.dll (50% traffic)
```

### 4. Third-Party Integration

External vendors can build plugins:

```
Vendor plugins:
├─ Assaia.ApronAI.dll
├─ Amadeus.Integration.dll
└─ AODB.CustomAdapter.dll
```

## Best Practices

### 1. Versioning

```csharp
[assembly: AssemblyVersion("1.2.3")]

public class MyPlugin : IPlugin
{
    public string Version => "1.2.3";
}
```

### 2. Error Handling

Plugins should never crash the host:

```csharp
public async Task<ProcessResult> Execute(...)
{
    try
    {
        // Plugin logic
        return ProcessResult.Success();
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Plugin error");
        return ProcessResult.Failure(ex.Message);
    }
}
```

### 3. Performance

```csharp
// Cache expensive operations
private static readonly ConcurrentDictionary<string, int> _cache = new();

public int CalculatePriority(FlightLeg flight)
{
    return _cache.GetOrAdd(flight.Id.ToString(), key =>
    {
        // Expensive calculation
        return DoCalculation(flight);
    });
}
```

### 4. Testing

```csharp
[Fact]
public async Task CustomPriorityPlugin_AssignsPriority()
{
    // Arrange
    var plugin = new CustomPriorityPlugin();
    var context = new MockPluginContext();
    await plugin.Initialize(context);
    
    var flight = new FlightLeg
    {
        OperatorIata = "SK",
        IsDelayed = true
    };
    
    // Act
    var result = await plugin.Execute(new EnrichmentContext(), flight);
    
    // Assert
    Assert.True(result.Success);
    Assert.Equal(10, flight["CustomPriority"]);
}
```

## Related Patterns

- **[[Business Rules]]** - Plugins extend business rules
- **[[Entity Model]]** - Plugins work with flexible entities
- **[[Message Types]]** - Plugins process messages
- **Plugin Pattern** - Extensible architecture
- **Strategy Pattern** - Interchangeable algorithms

---

**Tags:** #airhart #plugins #extensibility #architecture #dynamic-loading
**Created:** 2025-12-15
**Related:** [[Business Rules]], [[Entity Model]], [[Message Types]]
