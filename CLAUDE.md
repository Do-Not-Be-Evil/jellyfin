# CLAUDE.md - AI Assistant Guide for Jellyfin

This document provides guidance for AI assistants working on the Jellyfin codebase.

## Project Overview

Jellyfin is a free software media server written in .NET that allows users to manage and stream media to various client devices. It's descended from Emby 3.5.2 and ported to .NET for full cross-platform support.

- **License**: GPL-2.0
- **Framework**: .NET 10.0 (`net10.0`)
- **Language**: C# with nullable reference types enabled
- **Web Framework**: ASP.NET Core with Kestrel
- **ORM**: Entity Framework Core 10.x (SQLite default, PostgreSQL experimental)

## Quick Commands

```bash
# Build the solution
dotnet build

# Run the server (requires jellyfin-web in separate directory)
dotnet run --project Jellyfin.Server --webdir /path/to/jellyfin-web/dist

# Run all tests
dotnet test

# Run tests with coverage
dotnet test --collect:"XPlat Code Coverage" --settings tests/coverletArgs.runsettings

# Run specific test project
dotnet test tests/Jellyfin.Server.Tests/

# Add EF Core migration (SQLite)
dotnet ef migrations add MIGRATION_NAME --project src/Jellyfin.Database/Jellyfin.Database.Providers.Sqlite -- --migration-provider Jellyfin-SQLite
```

## Directory Structure

```
jellyfin/
├── Jellyfin.Server/              # Entry point, ASP.NET Core startup, DI configuration
├── Jellyfin.Server.Implementations/  # Core service implementations (UserManager, etc.)
├── Jellyfin.Api/                 # REST API controllers (60+ controllers)
│   ├── Controllers/              # API endpoints
│   ├── Auth/                     # Authorization handlers
│   └── Constants/                # Claim types, user roles
├── MediaBrowser.Controller/      # Interface contracts and abstractions
├── MediaBrowser.Model/           # DTOs and value objects
├── MediaBrowser.Common/          # Base utilities, shared abstractions
├── Jellyfin.Data/                # Database DTOs and enums
├── src/
│   ├── Jellyfin.Database/        # EF Core DbContext and migrations
│   ├── Jellyfin.Drawing.Skia/    # Image processing
│   ├── Jellyfin.Extensions/      # Extension methods
│   ├── Jellyfin.LiveTv/          # Live TV functionality
│   ├── Jellyfin.MediaEncoding.*/ # HLS, keyframes
│   └── Jellyfin.Networking/      # Network configuration
├── Emby.*/                       # Legacy code (being refactored)
├── MediaBrowser.Providers/       # Metadata providers
├── MediaBrowser.LocalMetadata/   # Local NFO parsing
├── MediaBrowser.XbmcMetadata/    # XBMC/Kodi metadata
└── tests/                        # Test projects (xUnit)
```

## Code Style & Conventions

### Formatting (enforced by `.editorconfig`)
- **Indentation**: 4 spaces (2 for YAML/XML)
- **Line endings**: LF
- **Charset**: UTF-8
- **Trailing whitespace**: Not allowed
- **Final newline**: Required

### Naming Conventions
```csharp
// PascalCase: Classes, methods, properties, public fields, constants, local functions
public class MediaLibrary { }
public void ProcessMedia() { }
public const string DefaultPath = "/media";

// camelCase: Local variables, parameters
var mediaItem = GetItem();
void ProcessFile(string filePath) { }

// _camelCase: Private instance fields (underscore prefix)
private readonly ILogger _logger;
private string _currentPath;

// Static fields also use underscore prefix
private static readonly JsonSerializerOptions _jsonOptions;
```

### C# Style Preferences
- Use `var` when type is apparent
- Expression-bodied members for simple properties/indexers
- Pattern matching over `is` with cast or `as` with null check
- Braces required for all control flow statements
- `System` usings sorted first, then alphabetical
- No `this.` qualification (disabled SA1101)
- Prefer BCL type keywords (`int` not `Int32`)

### Critical Analyzer Rules (Errors)
These will fail the build:
- **SA1518**: File must end with single newline
- **CA1305**: Specify `IFormatProvider`
- **CA1307/CA1309/CA1310**: Specify `StringComparison`
- **CA1849**: Call async methods when in async context
- **CA2016**: Forward `CancellationToken` to methods that accept it
- **RS0030**: No banned APIs (see `BannedSymbols.txt`)

### Banned APIs (`BannedSymbols.txt`)
Do NOT use these:
```csharp
// BANNED - use async/await instead
Task<T>.Result

// BANNED - use Guid.Equals(Guid) instead
Guid == Guid
Guid != Guid
Guid.Equals(object)
```

## Architecture Patterns

### Dependency Injection
Everything is injected via ASP.NET Core's DI container:
```csharp
public class MyController : BaseJellyfinApiController
{
    private readonly ILibraryManager _libraryManager;
    private readonly ILogger<MyController> _logger;

    public MyController(ILibraryManager libraryManager, ILogger<MyController> logger)
    {
        _libraryManager = libraryManager;
        _logger = logger;
    }
}
```

### Manager Pattern
Business logic lives in `*Manager` classes implementing interfaces:
- `ILibraryManager` - Media library operations
- `IUserManager` - User accounts and authentication
- `ISessionManager` - Client sessions
- `IDeviceManager` - Device management
- `IMediaEncoder` - Transcoding operations
- `IProviderManager` - Metadata provider coordination

### API Controllers
All controllers inherit from `BaseJellyfinApiController`:
```csharp
[Route("Items")]
public class ItemsController : BaseJellyfinApiController
{
    [HttpGet("{itemId}")]
    [Authorize]
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public ActionResult<BaseItemDto> GetItem([FromRoute] Guid itemId)
    {
        // Implementation
    }
}
```

### Authorization Policies
Use policy-based authorization (defined in `MediaBrowser.Common.Api.Policies`):
```csharp
[Authorize(Policy = Policies.RequiresElevation)]      // Admin only
[Authorize(Policy = Policies.Download)]               // Download permission
[Authorize(Policy = Policies.LiveTvAccess)]           // Live TV access
[Authorize(Policy = Policies.FirstTimeSetupOrElevated)] // Setup wizard or admin
[Authorize(Policy = Policies.LocalAccessOnly)]        // Local network only
```

### User Roles (in `Jellyfin.Api.Constants.UserRoles`)
```csharp
UserRoles.Administrator  // Admin user
UserRoles.User          // Regular user
UserRoles.Guest         // Guest user
```

## Testing

### Test Framework
- **xUnit** with `[Fact]` and `[Theory]` attributes
- **Moq** for mocking interfaces
- **AutoFixture** for test data generation

### Test Structure
```csharp
public class MyServiceTests
{
    private readonly Mock<IDependency> _mockDependency;
    private readonly MyService _sut; // System Under Test

    public MyServiceTests()
    {
        _mockDependency = new Mock<IDependency>();
        _sut = new MyService(_mockDependency.Object);
    }

    [Fact]
    public void MethodName_StateUnderTest_ExpectedBehavior()
    {
        // Arrange
        // Act
        // Assert
    }

    [Theory]
    [InlineData("input1", "expected1")]
    [InlineData("input2", "expected2")]
    public void MethodName_WithVariousInputs_ReturnsExpected(string input, string expected)
    {
        // Test with multiple inputs
    }
}
```

### Test File Naming
- Test projects: `Jellyfin.{ProjectName}.Tests`
- Test classes: `{ClassName}Tests.cs`
- Test methods: `MethodName_StateUnderTest_ExpectedBehavior`

## Common Tasks

### Adding a New API Endpoint
1. Create or modify controller in `Jellyfin.Api/Controllers/`
2. Inherit from `BaseJellyfinApiController`
3. Add appropriate `[Authorize]` policy
4. Add `[ProducesResponseType]` attributes for OpenAPI docs
5. Write tests in corresponding test project

### Adding a New Service
1. Define interface in `MediaBrowser.Controller/` (appropriate subfolder)
2. Implement in `Jellyfin.Server.Implementations/`
3. Register in DI (see `Jellyfin.Server/Extensions/` for examples)
4. Write unit tests

### Adding Database Changes
1. Modify entities in database provider project
2. Update `JellyfinDbContext` if adding new `DbSet<>`
3. Create migration: `dotnet ef migrations add MigrationName --project src/Jellyfin.Database/Jellyfin.Database.Providers.Sqlite -- --migration-provider Jellyfin-SQLite`

### Adding a Scheduled Task
1. Implement `IScheduledTask` interface
2. Register in DI container
3. Task is automatically discovered and scheduled

## Key Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| ASP.NET Core | 10.0.x | Web framework |
| Entity Framework Core | 10.0.2 | ORM |
| Serilog | 10.0.0 | Logging |
| SkiaSharp | 3.116.1 | Image processing |
| System.Text.Json | 10.0.2 | JSON serialization |
| xUnit | 2.9.3 | Testing |
| Moq | 4.18.4 | Mocking |

## Important Notes

1. **Nullable Reference Types**: Enabled project-wide. Handle nulls explicitly.

2. **Async/Await**: Used throughout. Always forward `CancellationToken`.

3. **Warnings as Errors**: Build fails on warnings in CI.

4. **Cross-Platform**: Code must work on Windows, macOS, and Linux.

5. **No External FFmpeg Dependency**: FFmpeg is called as external process (`jellyfin-ffmpeg` package).

6. **Plugin System**: Plugins loaded via reflection. Implement `IPluginServiceRegistrator` for DI.

7. **JSON Serialization**: Prefer `System.Text.Json` over `Newtonsoft.Json`.

8. **String Comparisons**: Always specify `StringComparison` (usually `Ordinal` or `OrdinalIgnoreCase`).

9. **Format Providers**: Always specify `CultureInfo` for formatting operations.

10. **Logging**: Use Serilog with structured logging. Avoid string interpolation in log messages:
    ```csharp
    // Good
    _logger.LogInformation("Processing item {ItemId}", itemId);

    // Bad - creates new string even if logging level disabled
    _logger.LogInformation($"Processing item {itemId}");
    ```

## File Locations Reference

| What | Where |
|------|-------|
| Solution file | `Jellyfin.sln` |
| Global build props | `Directory.Build.props` |
| Package versions | `Directory.Packages.props` |
| Code style | `.editorconfig` |
| StyleCop config | `stylecop.json` |
| Banned APIs | `BannedSymbols.txt` |
| CI workflows | `.github/workflows/` |
| Test settings | `tests/coverletArgs.runsettings` |
| API docs | `http://localhost:8096/api-docs/swagger/index.html` (when running) |
