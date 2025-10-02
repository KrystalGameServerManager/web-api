# KGSM API - .NET 9.0 Web API

A modern .NET 9.0 Web API for managing game server instances through the Krystal Game Server Manager (KGSM).

## Project Structure

```
src/
├── Controllers/               # API Controllers
│   ├── BlueprintsController.cs   # Blueprint management endpoints
│   ├── InstancesController.cs    # Instance management endpoints
│   └── SystemController.cs       # System metrics endpoints
├── Services/                  # Business Logic Services
│   ├── ISystemMetricsService.cs  # System metrics interface
│   ├── SystemMetricsService.cs   # System metrics implementation
│   ├── ILogStreamingService.cs   # Log streaming interface
│   └── LogStreamingService.cs    # Log streaming implementation
├── Models/                    # Data Models (One class per file)
│   ├── SystemMetrics.cs          # Main system metrics container
│   ├── CpuMetrics.cs             # CPU usage metrics
│   ├── CpuCoreInfo.cs            # CPU core information
│   ├── CpuHistoryPoint.cs        # CPU history data point
│   ├── MemoryMetrics.cs          # Memory usage metrics
│   ├── DiskMetrics.cs            # Disk usage metrics
│   ├── NetworkMetrics.cs         # Network usage metrics
│   ├── NetworkDataPoint.cs       # Network data point
│   ├── NetworkTotal.cs           # Network totals
│   ├── SystemInfo.cs             # System information
│   ├── LogStreamStatus.cs        # Log stream status
│   └── Dtos/                     # Data Transfer Objects (One class per file)
│       ├── InstanceInstallDto.cs # Instance installation request
│       ├── InstanceCommandDto.cs # Instance command request
│       ├── LogDisconnectDto.cs   # Log disconnect request
│       ├── ApiResponse.cs        # Generic API response
│       └── ApiErrorResponse.cs   # API error response
├── Configuration/             # Configuration Classes
│   └── KgsmApiOptions.cs         # KGSM API configuration options
├── Hubs/                     # SignalR Hubs
│   └── LogStreamingHub.cs        # Real-time log streaming
├── Program.cs                # Application entry point
├── appsettings.json          # Configuration file
└── api.csproj               # Project file
```

## Features

### ✅ Implemented (Scaffolding Complete)

- **Project Structure**: Clean architecture with separation of concerns
- **Dependency Injection**: Proper DI container setup
- **CORS Configuration**: Development-friendly CORS policy
- **KGSM Library Integration**: Uses TheKrystalShip.KGSM.Lib
- **SignalR Support**: Real-time WebSocket communication
- **Health Checks**: Basic health monitoring
- **OpenAPI/Swagger**: API documentation
- **Logging**: Structured logging with different levels

### 🚧 Placeholder Implementations (Ready for Development)

- **System Metrics Collection**: CPU, Memory, Disk, Network monitoring
- **Log Streaming**: Real-time log streaming with buffering
- **KGSM Operations**: All CRUD operations for blueprints and instances

## API Endpoints

### Blueprints
- `GET /api/kgsm/blueprints` - Get all available blueprints

### Instances
- `GET /api/kgsm/instances` - Get all instances
- `POST /api/kgsm/instances` - Install new instance
- `DELETE /api/kgsm/instances/{name}` - Uninstall instance
- `POST /api/kgsm/instances/{name}/start` - Start instance
- `POST /api/kgsm/instances/{name}/stop` - Stop instance
- `POST /api/kgsm/instances/{name}/restart` - Restart instance
- `GET /api/kgsm/instances/{name}/logs` - Get instance logs

### System
- `GET /api/system/metrics` - Get system metrics

### SignalR Hubs
- `/hubs/logs` - Real-time log streaming hub

### Health & Monitoring
- `/health` - Health check endpoint
- `/openapi/v1.json` - OpenAPI specification
- `/swagger` - Swagger UI (Development only)

## Configuration

### appsettings.json
```json
{
  "KgsmApi": {
    "KgsmPath": "kgsm",
    "SocketPath": "/tmp/kgsm.sock",
    "Port": 5167,
    "AllowedOrigins": [
      "http://localhost:3000",
      "http://127.0.0.1:3000"
    ],
    "MaxLogBufferLines": 1000,
    "LogCleanupIntervalSeconds": 30
  }
}
```

## Dependencies

- **.NET 9.0**: Latest .NET framework
- **TheKrystalShip.KGSM.Lib**: KGSM interop library
- **Microsoft.AspNetCore.SignalR**: Real-time communication
- **Microsoft.Extensions.Diagnostics.HealthChecks**: Health monitoring
- **Swashbuckle.AspNetCore**: API documentation

## Design Principles

- **SOLID Principles**: Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion
- **Clean Architecture**: Separation of concerns with clear boundaries
- **Single Class Per File**: Each class, interface, and record gets its own file for better maintainability
- **Dependency Injection**: Constructor injection for all dependencies
- **Async/Await**: Asynchronous programming throughout
- **Structured Logging**: Consistent logging with proper log levels
- **Configuration Pattern**: Strongly-typed configuration options

## Getting Started

1. **Prerequisites**:
   - .NET 9.0 SDK
   - KGSM installed and available in PATH

2. **Build**:
   ```bash
   dotnet build
   ```

3. **Run**:
   ```bash
   dotnet run
   ```

4. **Development**:
   - API available at: `https://localhost:7110` or `http://localhost:5167`
   - Swagger UI: `https://localhost:7110/swagger` (Development only)
   - Health Check: `https://localhost:7110/health`

## Next Steps

This scaffolding provides a solid foundation. To complete the implementation:

1. **System Metrics**: Replace placeholder implementations with actual system monitoring
2. **Log Streaming**: Implement real process spawning for KGSM log tailing
3. **Error Handling**: Add comprehensive error handling and validation
4. **Authentication**: Add authentication/authorization if needed
5. **Testing**: Add unit and integration tests
6. **Deployment**: Configure for production deployment

## Architecture Decisions

- **SignalR over Socket.IO**: Native .NET real-time communication
- **Minimal APIs**: Used traditional controllers for better OpenAPI support
- **Service Lifetime**: Scoped services for request-bound operations
- **Configuration**: Options pattern with strongly-typed configuration
- **Logging**: Built-in .NET logging with structured output
