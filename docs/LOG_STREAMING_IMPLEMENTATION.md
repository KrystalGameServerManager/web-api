# KGSM Log Streaming Implementation

This document describes the continuous log streaming functionality implemented for the KGSM library. The implementation provides a clean, event-driven API for subscribing to real-time log streams from KGSM instances using the `--follow` flag.

## Overview

The log streaming functionality allows applications to subscribe to continuous log output from KGSM instances. It uses .NET's event system to provide real-time log updates with proper error handling, filtering, and resource management.

## Key Features

- **Real-time Log Streaming**: Continuous log output using KGSM's `--follow` flag
- **Event-Driven Architecture**: Clean event system for handling log entries, errors, and status changes
- **Log Level Filtering**: Filter logs by minimum severity level
- **Structured Log Parsing**: Automatic parsing of log entries with timestamp and level detection
- **Proper Resource Management**: Automatic cleanup of processes and resources
- **Thread-Safe Operations**: Safe for concurrent access and multiple subscriptions
- **Cancellation Support**: Full support for cancellation tokens and graceful shutdown

## Architecture

### Core Components

1. **LogEntry** - Represents a single log entry with metadata
2. **LogSubscription** - Manages a single log stream subscription
3. **LogStreamEventArgs** - Event arguments for log-related events
4. **LogParser** - Utility for parsing raw log lines into structured data
5. **IInstanceService Extensions** - New methods for log streaming

### Class Hierarchy

```
LogEntry (record)
├── Timestamp: DateTime
├── Level: LogLevel
├── Message: string
├── InstanceName: string
├── RawLine: string
├── Source: string?
└── ThreadId: string?

LogSubscription (IDisposable)
├── Events: LogReceived, ErrorOccurred, StatusChanged
├── Properties: SubscriptionId, InstanceName, IsActive, etc.
└── Methods: StopAsync(), Dispose()

LogStreamEventArgs
├── LogStreamEventArgs(LogEntry)
├── LogStreamErrorEventArgs(string, Exception)
└── LogStreamStatusEventArgs(string, bool, string?)
```

## Implementation Details

### 1. LogEntry Model

```csharp
public record LogEntry
{
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;
    public LogLevel Level { get; set; } = LogLevel.Info;
    public string Message { get; set; } = string.Empty;
    public string InstanceName { get; set; } = string.Empty;
    public string RawLine { get; set; } = string.Empty;
    public string? Source { get; set; }
    public string? ThreadId { get; set; }
}
```

### 2. LogSubscription Class

The `LogSubscription` class manages individual log stream subscriptions:

- **Process Management**: Handles the underlying KGSM process
- **Event System**: Provides events for log entries, errors, and status changes
- **Resource Cleanup**: Implements `IDisposable` for proper cleanup
- **Status Monitoring**: Tracks connection status and process health

### 3. Log Parser

The `LogParser` utility class handles parsing of raw log lines:

- **Multiple Format Support**: Recognizes various log formats
- **Timestamp Parsing**: Extracts timestamps using multiple formats
- **Log Level Detection**: Identifies log levels from various patterns
- **Thread Information**: Extracts thread IDs when available

### 4. Instance Service Extensions

New methods added to `IInstanceService`:

```csharp
Task<LogSubscription> SubscribeToLogsAsync(string instanceName, CancellationToken cancellationToken = default);

Task<LogSubscription> SubscribeToLogsAsync(string instanceName, LogLevel minimumLogLevel, bool includeRawLines = true, CancellationToken cancellationToken = default);
```

## Usage Examples

### Basic Usage

```csharp
// Get the instance service
var instanceService = serviceProvider.GetRequiredService<IInstanceService>();

// Subscribe to logs
var subscription = await instanceService.SubscribeToLogsAsync("my-server");

// Set up event handlers
subscription.LogReceived += (sender, args) =>
{
    var entry = args.LogEntry;
    Console.WriteLine($"[{entry.Timestamp:HH:mm:ss}] [{entry.Level}] {entry.Message}");
};

subscription.ErrorOccurred += (sender, args) =>
{
    Console.WriteLine($"Error: {args.Exception.Message}");
};

subscription.StatusChanged += (sender, args) =>
{
    Console.WriteLine($"Status: {args.InstanceName} is {(args.IsConnected ? "connected" : "disconnected")}");
};

// Stop when done
await subscription.StopAsync();
subscription.Dispose();
```

### Filtered Logging

```csharp
// Only receive warnings and errors
var subscription = await instanceService.SubscribeToLogsAsync(
    "my-server",
    LogLevel.Warning,
    includeRawLines: false);

subscription.LogReceived += (sender, args) =>
{
    // Only warnings, errors, and fatal messages will be received
    Console.WriteLine($"Important: {args.LogEntry.Message}");
};
```

### Multiple Subscriptions

```csharp
var subscriptions = new List<LogSubscription>();

foreach (var instanceName in instanceNames)
{
    var subscription = await instanceService.SubscribeToLogsAsync(instanceName);
    subscription.LogReceived += (sender, args) =>
    {
        Console.WriteLine($"[{args.LogEntry.InstanceName}] {args.LogEntry.Message}");
    };
    subscriptions.Add(subscription);
}

// Clean up all subscriptions
foreach (var subscription in subscriptions)
{
    await subscription.StopAsync();
    subscription.Dispose();
}
```

## Error Handling

The implementation provides comprehensive error handling:

### Process Errors
- Failed process startup
- Unexpected process termination
- Process communication errors

### Parsing Errors
- Invalid log formats
- Timestamp parsing failures
- Encoding issues

### Network/IO Errors
- Stream reading failures
- Buffer overflow protection
- Timeout handling

## Performance Considerations

### Memory Management
- Efficient character-by-character processing
- Configurable raw line inclusion
- Automatic buffer cleanup
- Proper disposal patterns

### CPU Usage
- Minimal overhead per log line
- Efficient regex compilation
- Background processing
- Non-blocking operations

### Scalability
- Support for multiple concurrent subscriptions
- Thread-safe operations
- Resource pooling
- Graceful degradation

## Configuration Options

### Log Level Filtering
```csharp
LogLevel.Trace    // Most verbose
LogLevel.Debug    // Debug information
LogLevel.Info     // General information (default)
LogLevel.Warning  // Potentially harmful situations
LogLevel.Error    // Error events
LogLevel.Fatal    // Very severe errors
```

### Parsing Options
- `includeRawLines`: Whether to include original log lines
- Custom timestamp formats
- Log level mapping
- Thread ID extraction

## Testing

### Unit Tests
- Log parsing accuracy
- Event firing verification
- Resource cleanup validation
- Error condition handling

### Integration Tests
- Real KGSM process interaction
- Multiple subscription scenarios
- Long-running stability tests
- Resource leak detection

## File Structure

```
kgsm-lib/
├── Core/
│   ├── Models/
│   │   ├── LogEntry.cs
│   │   ├── LogSubscription.cs
│   │   └── LogStreamEventArgs.cs
│   └── Interfaces/
│       └── IInstanceService.cs (extended)
├── Services/
│   └── InstanceService.cs (extended)
├── Utilities/
│   └── LogParser.cs
└── Examples/
    └── LogStreamingExample.cs
```

## Dependencies

- **.NET 9.0**: Core framework
- **Microsoft.Extensions.Logging**: Structured logging
- **System.Diagnostics.Process**: Process management
- **System.Text.RegularExpressions**: Log parsing

## Best Practices

### Resource Management
```csharp
// Always use using statements or explicit disposal
using var subscription = await instanceService.SubscribeToLogsAsync("instance");

// Or explicit cleanup
try
{
    var subscription = await instanceService.SubscribeToLogsAsync("instance");
    // Use subscription...
}
finally
{
    await subscription.StopAsync();
    subscription.Dispose();
}
```

### Error Handling
```csharp
subscription.ErrorOccurred += async (sender, args) =>
{
    _logger.LogError(args.Exception, "Log streaming error for {Instance}", args.InstanceName);

    // Optionally restart the subscription
    if (args.Exception is not OperationCanceledException)
    {
        await RestartSubscription(args.InstanceName);
    }
};
```

### Performance Optimization
```csharp
// For high-volume logs, consider filtering
var subscription = await instanceService.SubscribeToLogsAsync(
    instanceName,
    LogLevel.Warning,  // Only warnings and above
    includeRawLines: false  // Save memory
);
```

## Limitations

1. **Platform Dependency**: Requires KGSM to support the `--follow` flag
2. **Process Overhead**: Each subscription creates a separate KGSM process
3. **Memory Usage**: Log entries are kept in memory until processed
4. **Network Latency**: Local process communication only

## Future Enhancements

1. **Batch Processing**: Group multiple log entries for efficiency
2. **Compression**: Compress log data for large volumes
3. **Persistence**: Optional log storage and replay
4. **Metrics**: Built-in performance and health metrics
5. **Filtering**: Advanced filtering with custom predicates

## Troubleshooting

### Common Issues

1. **Process Not Starting**
   - Verify KGSM path is correct
   - Check instance name exists
   - Ensure proper permissions

2. **No Log Output**
   - Verify instance is active and generating logs
   - Check log level filtering
   - Confirm KGSM supports `--follow` flag

3. **Memory Issues**
   - Set `includeRawLines: false` for high-volume logs
   - Implement log level filtering
   - Monitor subscription count

4. **Performance Problems**
   - Reduce number of concurrent subscriptions
   - Implement batching in event handlers
   - Use background processing for heavy operations

## Conclusion

The KGSM log streaming implementation provides a robust, enterprise-grade solution for real-time log monitoring. It follows .NET best practices, provides comprehensive error handling, and offers flexible configuration options for various use cases.

The event-driven architecture ensures clean separation of concerns while maintaining high performance and reliability. The implementation is designed to be extensible and can be easily adapted for specific requirements or integrated into larger monitoring systems.
