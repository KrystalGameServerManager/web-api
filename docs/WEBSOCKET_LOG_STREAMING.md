# KGSM WebSocket Log Streaming API

This document describes the real-time log streaming functionality in the KGSM API, which allows clients to receive live log feeds from KGSM instances via WebSocket connections using SignalR.

## Overview

The WebSocket log streaming feature provides:

- **Real-time log streaming** from KGSM instances using the `--follow` flag
- **Event-driven architecture** with SignalR for reliable WebSocket communication
- **Log level filtering** to reduce bandwidth and processing overhead
- **Multiple client support** for the same instance
- **Automatic reconnection** and connection health monitoring
- **Buffered log history** for new connections
- **Structured log parsing** with timestamp, level, and source extraction

## Architecture

```
┌─────────────────┐    WebSocket/SignalR    ┌─────────────────┐    KGSM Process    ┌─────────────────┐
│   Web Client    │◄─────────────────────►│   KGSM API      │◄─────────────────►│   KGSM Instance │
│                 │                        │                 │                    │                 │
│ - JavaScript    │    Log Events:         │ - LogStreaming  │   --logs --follow  │ - Game Server   │
│ - SignalR       │    • LogMessage        │   Service       │                    │ - Log Output    │
│ - HTML UI       │    • LogError          │ - SignalR Hub   │                    │                 │
└─────────────────┘    • LogStatusChange   └─────────────────┘                    └─────────────────┘
```

## SignalR Hub Endpoint

**Hub URL:** `/hubs/logs`

**Full URL Example:** `http://localhost:5000/hubs/logs`

## Client Methods (Invoke)

These methods can be called by clients to interact with the log streaming service:

### `SubscribeLogs(instanceName)`

Subscribe to all log levels for a specific instance.

**Parameters:**
- `instanceName` (string): Name of the KGSM instance

**Example:**
```javascript
await connection.invoke("SubscribeLogs", "myserver");
```

### `SubscribeLogsFiltered(instanceName, minimumLogLevel?, includeRawLines?)`

Subscribe to logs with filtering options.

**Parameters:**
- `instanceName` (string): Name of the KGSM instance
- `minimumLogLevel` (string, optional): Minimum log level ("Trace", "Debug", "Info", "Warning", "Error", "Fatal")
- `includeRawLines` (bool, optional): Whether to include raw log lines (default: true)

**Example:**
```javascript
await connection.invoke("SubscribeLogsFiltered", "myserver", "Warning", true);
```

### `UnsubscribeLogs(instanceName)`

Unsubscribe from logs for a specific instance.

**Parameters:**
- `instanceName` (string): Name of the KGSM instance

**Example:**
```javascript
await connection.invoke("UnsubscribeLogs", "myserver");
```

### `GetStreamStatus()`

Get the current status of all log streams.

**Example:**
```javascript
await connection.invoke("GetStreamStatus");
```

### `GetBufferedLogs(instanceName)`

Retrieve buffered logs for an instance.

**Parameters:**
- `instanceName` (string): Name of the KGSM instance

**Example:**
```javascript
await connection.invoke("GetBufferedLogs", "myserver");
```

### `GetConnectionInfo()`

Get information about the current connection.

**Example:**
```javascript
await connection.invoke("GetConnectionInfo");
```

### `Ping()`

Send a ping to test connection health.

**Example:**
```javascript
await connection.invoke("Ping");
```

## Server Events (Listen)

These events are sent from the server to clients:

### `Connected`

Sent when a client successfully connects to the hub.

**Data:**
```json
{
  "connectionId": "string",
  "availableMethods": ["SubscribeLogs", "UnsubscribeLogs", ...],
  "events": ["LogMessage", "LogError", ...]
}
```

### `SubscriptionConfirmed`

Sent when a log subscription is successfully created.

**Data:**
```json
{
  "instanceName": "string",
  "bufferSize": 0,
  "isActive": true,
  "subscriptionId": "guid"
}
```

### `SubscriptionError`

Sent when a subscription fails.

**Data:**
```json
{
  "instanceName": "string",
  "error": "Error message"
}
```

### `UnsubscriptionConfirmed`

Sent when a client successfully unsubscribes.

**Data:**
```json
{
  "instanceName": "string"
}
```

### `LogMessage`

Sent for each log entry received from the KGSM instance.

**Data:**
```json
{
  "instanceName": "string",
  "timestamp": "2024-01-01T12:00:00Z",
  "level": "Info",
  "message": "Server started successfully",
  "source": "KGSM",
  "threadId": "1",
  "rawLine": "[12:00:00] [INFO] Server started successfully"
}
```

### `LogHistory`

Sent when buffered logs are available for a new subscription.

**Data:** String containing historical log content

### `LogError`

Sent when an error occurs in the log streaming process.

**Data:**
```json
{
  "instanceName": "string",
  "error": "Error message",
  "timestamp": "2024-01-01T12:00:00Z"
}
```

### `LogStatusChange`

Sent when the log stream status changes.

**Data:**
```json
{
  "instanceName": "string",
  "status": "Connected",
  "message": "Status description",
  "timestamp": "2024-01-01T12:00:00Z"
}
```

### `StreamStatus`

Response to `GetStreamStatus()` request.

**Data:**
```json
{
  "instanceName": {
    "isActive": true,
    "bufferSize": 150,
    "clientCount": 2,
    "lastActivity": "2024-01-01T12:00:00Z",
    "clientIds": ["connection1", "connection2"]
  }
}
```

### `ConnectionInfo`

Response to `GetConnectionInfo()` request.

**Data:**
```json
{
  "connectionId": "string",
  "connectedAt": "2024-01-01T12:00:00Z",
  "activeStreams": ["server1", "server2"],
  "totalActiveStreams": 5
}
```

### `BufferedLogs`

Response to `GetBufferedLogs()` request.

**Data:**
```json
{
  "instanceName": "string",
  "logs": "Multi-line log content..."
}
```

### `Pong`

Response to `Ping()` request.

**Data:**
```json
{
  "timestamp": "2024-01-01T12:00:00Z"
}
```

## Usage Examples

### JavaScript/HTML Client

```html
<!DOCTYPE html>
<html>
<head>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/microsoft-signalr/8.0.0/signalr.min.js"></script>
</head>
<body>
    <div id="logs"></div>

    <script>
        const connection = new signalR.HubConnectionBuilder()
            .withUrl("http://localhost:5000/hubs/logs")
            .withAutomaticReconnect()
            .build();

        // Handle log messages
        connection.on("LogMessage", (data) => {
            const logDiv = document.getElementById("logs");
            const entry = document.createElement("div");
            entry.innerHTML = `
                <span>${new Date(data.timestamp).toLocaleTimeString()}</span>
                <span>[${data.level}]</span>
                <span>${data.message}</span>
            `;
            logDiv.appendChild(entry);
        });

        // Handle subscription confirmation
        connection.on("SubscriptionConfirmed", (data) => {
            console.log("Subscribed to", data.instanceName);
        });

        // Start connection and subscribe
        connection.start().then(() => {
            connection.invoke("SubscribeLogs", "myserver");
        });
    </script>
</body>
</html>
```

### C# Client

```csharp
using Microsoft.AspNetCore.SignalR.Client;

var connection = new HubConnectionBuilder()
    .WithUrl("http://localhost:5000/hubs/logs")
    .WithAutomaticReconnect()
    .Build();

// Handle log messages
connection.On<object>("LogMessage", (data) => {
    var json = JsonSerializer.Serialize(data, new JsonSerializerOptions { WriteIndented = true });
    Console.WriteLine($"Log: {json}");
});

// Start connection
await connection.StartAsync();

// Subscribe to logs
await connection.InvokeAsync("SubscribeLogs", "myserver");

// Keep running
Console.ReadKey();

// Clean up
await connection.InvokeAsync("UnsubscribeLogs", "myserver");
await connection.StopAsync();
```

### Python Client

```python
import asyncio
import json
from signalrcore.hub_connection_builder import HubConnectionBuilder

async def main():
    connection = HubConnectionBuilder()\
        .with_url("http://localhost:5000/hubs/logs")\
        .with_automatic_reconnect({
            "type": "raw",
            "keep_alive_interval": 10,
            "reconnect_interval": 5,
        })\
        .build()

    # Handle log messages
    def on_log_message(data):
        print(f"Log: {json.dumps(data, indent=2)}")

    connection.on("LogMessage", on_log_message)

    # Start connection
    await connection.start()

    # Subscribe to logs
    await connection.send("SubscribeLogs", ["myserver"])

    # Keep running
    await asyncio.sleep(60)

    # Clean up
    await connection.send("UnsubscribeLogs", ["myserver"])
    await connection.stop()

if __name__ == "__main__":
    asyncio.run(main())
```

## Log Levels

The system supports the following log levels (in order of priority):

1. **Trace** - Detailed diagnostic information
2. **Debug** - Debug information useful during development
3. **Info** - General informational messages
4. **Warning** - Warning messages for potential issues
5. **Error** - Error messages for failures
6. **Fatal** - Critical errors that may cause application termination

When using `SubscribeLogsFiltered` with a minimum log level, only logs at that level and higher will be received.

## Performance Considerations

### Client-Side

- **Buffer Management**: The client should implement log buffering to handle high-frequency log messages
- **UI Updates**: Use efficient DOM manipulation techniques for real-time log display
- **Memory Management**: Limit the number of displayed log entries to prevent memory issues
- **Connection Health**: Implement ping/pong monitoring for connection health

### Server-Side

- **Automatic Cleanup**: Inactive streams are automatically cleaned up after 5 minutes
- **Buffer Limits**: Server maintains a buffer of the last 1000 log lines per instance
- **Resource Management**: KGSM processes are properly disposed when no clients are connected
- **Concurrent Connections**: Multiple clients can subscribe to the same instance efficiently

## Error Handling

### Common Error Scenarios

1. **Instance Not Found**: When subscribing to a non-existent instance
2. **KGSM Process Failure**: When the underlying KGSM process fails or exits
3. **Connection Loss**: When the WebSocket connection is interrupted
4. **Permission Issues**: When KGSM cannot access log files

### Error Recovery

- **Automatic Reconnection**: SignalR automatically attempts to reconnect on connection loss
- **Subscription Recovery**: Clients should re-subscribe after reconnection
- **Error Events**: Listen for `LogError` and `SubscriptionError` events
- **Status Monitoring**: Use `LogStatusChange` events to monitor stream health

## Security Considerations

### Authentication & Authorization

Currently, the log streaming API does not implement authentication. In production environments, consider:

- **JWT Token Authentication**: Implement token-based authentication
- **Role-Based Access**: Restrict access based on user roles
- **Instance-Level Permissions**: Control which instances a user can access
- **Rate Limiting**: Implement rate limiting to prevent abuse

### Network Security

- **HTTPS/WSS**: Use secure connections in production
- **CORS Configuration**: Properly configure CORS for web clients
- **Firewall Rules**: Restrict access to the WebSocket endpoint
- **Monitoring**: Log and monitor connection attempts

## Configuration

### API Configuration

```json
{
  "KgsmApi": {
    "KgsmPath": "/path/to/kgsm",
    "SocketPath": "/path/to/kgsm.sock"
  }
}
```

### SignalR Configuration

The SignalR hub is automatically configured in `Program.cs`:

```csharp
builder.Services.AddSignalR();
app.MapHub<LogStreamingHub>("/hubs/logs");
```

## Troubleshooting

### Common Issues

1. **Connection Refused**
   - Verify the API is running and accessible
   - Check firewall settings
   - Confirm the correct hub URL

2. **Subscription Fails**
   - Verify the instance name exists
   - Check KGSM is properly configured
   - Ensure sufficient permissions

3. **No Log Messages**
   - Verify the instance is running and generating logs
   - Check if logs are being written to the expected location
   - Confirm KGSM can access the log files

4. **High Memory Usage**
   - Implement client-side log rotation
   - Limit the number of concurrent subscriptions
   - Monitor server-side buffer usage

### Debugging

Enable detailed logging in the API:

```json
{
  "Logging": {
    "LogLevel": {
      "TheKrystalShip.KGSM.Api.Services.LogStreamingService": "Debug",
      "TheKrystalShip.KGSM.Api.Hubs.LogStreamingHub": "Debug"
    }
  }
}
```

## Best Practices

### Client Implementation

1. **Connection Management**
   - Always handle connection state changes
   - Implement proper cleanup on page unload
   - Use automatic reconnection with backoff

2. **UI Performance**
   - Use virtual scrolling for large log displays
   - Implement log level filtering on the client
   - Batch DOM updates for better performance

3. **Error Handling**
   - Always listen for error events
   - Implement retry logic for failed operations
   - Provide user feedback for connection issues

### Server Implementation

1. **Resource Management**
   - Monitor memory usage and implement limits
   - Clean up inactive connections promptly
   - Use efficient data structures for log buffering

2. **Scalability**
   - Consider using Redis for distributed scenarios
   - Implement connection pooling for KGSM processes
   - Monitor performance metrics

3. **Reliability**
   - Implement health checks for KGSM processes
   - Use structured logging for debugging
   - Monitor and alert on error rates

## API Versioning

The current implementation is version 1.0. Future versions may include:

- Enhanced filtering capabilities
- Real-time log analysis features
- Integration with external logging systems
- Advanced security features
- Performance optimizations

## Support

For issues and questions:

1. Check the server logs for error messages
2. Verify KGSM configuration and permissions
3. Test with the provided example clients
4. Review the troubleshooting section above

## Examples

Complete working examples are provided in the `/Examples` directory:

- `WebSocketLogStreamingExample.html` - Comprehensive HTML/JavaScript client
- `LogStreamingClientExample.cs` - C# console application example

These examples demonstrate all the features and best practices described in this documentation.
