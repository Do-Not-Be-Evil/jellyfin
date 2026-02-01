# CLAUDE.md - Jellyfin Server Development Guide

This document provides guidance for AI assistants working with the Jellyfin server codebase.

## Project Overview

Jellyfin is a Free Software Media System - a backend server that manages and streams media to end-user devices. It is the open-source alternative to Emby and Plex, descended from Emby's 3.5.2 release and ported to .NET for cross-platform support.

- **License**: GPL 2.0
- **Current Version**: 10.12.0
- **Target Framework**: .NET 10.0
- **SDK Required**: .NET 10.0 SDK

## Quick Reference - Essential Commands

```bash
# Build the solution
dotnet build

# Run the server
dotnet run --project Jellyfin.Server

# Run with web client path
dotnet run --project Jellyfin.Server --webdir /path/to/jellyfin-web/dist

# Run without web client
dotnet run --project Jellyfin.Server -- --nowebclient

# Run all tests
dotnet test Jellyfin.sln --configuration Release

# Run tests with coverage
dotnet test Jellyfin.sln --configuration Release --collect:"XPlat Code Coverage" --settings tests/coverletArgs.runsettings
```

## Repository Structure

### Core Projects (Root Level)

| Directory | Purpose |
|-----------|---------|
| `Jellyfin.Server/` | Main executable - ASP.NET Core host application |
| `Jellyfin.Api/` | REST API controllers and middleware |
| `Jellyfin.Data/` | Data entities and enums |
| `Jellyfin.Server.Implementations/` | Server-specific implementations |
| `MediaBrowser.Controller/` | Core interfaces and abstractions |
| `MediaBrowser.Model/` | Shared DTOs and models (API contracts) |
| `MediaBrowser.Common/` | Common utilities and base classes |
| `MediaBrowser.Providers/` | Metadata providers (TMDB, TVDB, etc.) |
| `MediaBrowser.MediaEncoding/` | FFmpeg integration and transcoding |
| `MediaBrowser.LocalMetadata/` | Local NFO/XML metadata parsing |
| `MediaBrowser.XbmcMetadata/` | XBMC/Kodi NFO file support |
| `Emby.Server.Implementations/` | Core server implementations (library, sessions, etc.) |
| `Emby.Naming/` | Media file naming and parsing |
| `Emby.Photos/` | Photo library support |

### Source Projects (`src/`)

| Directory | Purpose |
|-----------|---------|
| `src/Jellyfin.Drawing/` | Image processing abstractions |
| `src/Jellyfin.Drawing.Skia/` | SkiaSharp image processing implementation |
| `src/Jellyfin.Networking/` | Network configuration and management |
| `src/Jellyfin.Extensions/` | Extension methods and utilities |
| `src/Jellyfin.LiveTv/` | Live TV and DVR functionality |
| `src/Jellyfin.MediaEncoding.Hls/` | HLS streaming support |
| `src/Jellyfin.MediaEncoding.Keyframes/` | Keyframe extraction |
| `src/Jellyfin.Database/` | Entity Framework database layer |
| `src/Jellyfin.CodeAnalysis/` | Custom Roslyn analyzers |

### Test Projects (`tests/`)

Each main project has a corresponding test project:
- `Jellyfin.Api.Tests/` - API controller tests
- `Jellyfin.Server.Implementations.Tests/` - Server implementation tests
- `Jellyfin.Naming.Tests/` - Media naming tests
- `Jellyfin.Providers.Tests/` - Metadata provider tests
- `Jellyfin.Server.Integration.Tests/` - Integration tests
- And more...

### Configuration Files

| File | Purpose |
|------|---------|
| `Jellyfin.sln` | Main solution file |
| `Directory.Build.props` | Global MSBuild properties (nullable enabled, warnings as errors) |
| `Directory.Packages.props` | Centralized NuGet package versions |
| `global.json` | .NET SDK version pinning (10.0.0) |
| `.editorconfig` | Code style and analyzer rules |
| `stylecop.json` | StyleCop analyzer configuration |
| `BannedSymbols.txt` | Banned API patterns |

## Code Style and Conventions

### Enforced via `.editorconfig`

- **Indentation**: 4 spaces (2 for YAML/XML)
- **Line endings**: LF
- **Charset**: UTF-8
- **Trailing whitespace**: Trimmed
- **Final newline**: Required

### C# Conventions

- **Nullable reference types**: Enabled globally
- **Warnings as errors**: Enabled (`TreatWarningsAsErrors=true`)
- **Using directives**: Sorted alphabetically, System namespaces first
- **Braces**: Required for all control statements
- **var**: Preferred when type is apparent

### Naming Conventions

- **Classes/Methods/Properties**: PascalCase
- **Private fields**: `_camelCase` (underscore prefix)
- **Static fields**: `_camelCase` (underscore prefix)
- **Local variables/parameters**: camelCase
- **Constants**: PascalCase

### Key Analyzer Rules (Errors)

- SA1000: Keyword spacing
- SA1028: No trailing whitespace
- SA1210: Using directives ordered alphabetically
- SA1518: File must end with single newline
- CA1305: Specify IFormatProvider
- CA1307/CA1309/CA1310: Specify StringComparison
- CA1849: Call async methods in async context
- CA2016: Forward CancellationToken to methods

### Banned APIs (`BannedSymbols.txt`)

- `Task<T>.Result` - Use `await` instead
- `Guid.op_Equality`/`Guid.op_Inequality` - Use `Guid.Equals(Guid)` instead
- `Guid.Equals(Object)` - Use type-safe overload

## Architecture Overview

### Layered Architecture

```
┌─────────────────────────────────────┐
│          Jellyfin.Server            │  ← Host/Startup
├─────────────────────────────────────┤
│           Jellyfin.Api              │  ← REST Controllers
├─────────────────────────────────────┤
│   Emby.Server.Implementations       │  ← Core Logic
│   Jellyfin.Server.Implementations   │
├─────────────────────────────────────┤
│       MediaBrowser.Controller       │  ← Interfaces/Abstractions
├─────────────────────────────────────┤
│         MediaBrowser.Model          │  ← DTOs/Contracts
│          Jellyfin.Data             │  ← Entities
├─────────────────────────────────────┤
│        MediaBrowser.Common          │  ← Base utilities
└─────────────────────────────────────┘
```

### Key Interfaces (in MediaBrowser.Controller)

- `ILibraryManager` - Media library management
- `IUserManager` - User account management
- `ISessionManager` - Client session tracking
- `IProviderManager` - Metadata provider coordination
- `IMediaEncoder` - FFmpeg transcoding interface
- `IServerApplicationHost` - Application host services

### Dependency Injection

The project uses ASP.NET Core's built-in DI. Services are registered in `Emby.Server.Implementations/ApplicationHost.cs`.

## API Development

### Controller Pattern

Controllers inherit from `BaseJellyfinApiController`:

```csharp
[Route("")]
public class LibraryController : BaseJellyfinApiController
{
    // Constructor injection for dependencies
    public LibraryController(
        ILibraryManager libraryManager,
        IUserManager userManager,
        ILogger<LibraryController> logger)
    {
        _libraryManager = libraryManager;
        _userManager = userManager;
        _logger = logger;
    }

    [HttpGet("Items/{itemId}/File")]
    [Authorize]
    public async Task<ActionResult> GetFile([FromRoute] Guid itemId)
    {
        // Implementation
    }
}
```

### API Documentation

- Swagger UI: `http://localhost:8096/api-docs/swagger/index.html`
- Uses Swashbuckle for OpenAPI generation
- XML documentation comments are required

### Authentication

- Custom authentication via `Jellyfin.Api/Auth/`
- `[Authorize]` attribute for protected endpoints
- Policy-based authorization (Admin, FirstTimeSetup, etc.)

## Database Layer

### Entity Framework Core

- **Context**: `JellyfinDbContext` in `src/Jellyfin.Database/Jellyfin.Database.Implementations/`
- **Provider**: SQLite (default), PostgreSQL (experimental)
- **Migrations**: Provider-specific in `Jellyfin.Database.Providers.Sqlite/`

### Running Migrations

```bash
# Create a new migration
dotnet ef migrations add MIGRATION_NAME \
  --project "src/Jellyfin.Database/Jellyfin.Database.Providers.Sqlite" \
  -- --migration-provider Jellyfin-SQLite
```

### Key Entities

- `User` - User accounts
- `Device` - Registered client devices
- `BaseItemEntity` - Media items
- `MediaStreamInfo` - Audio/video stream metadata
- `UserData` - User-specific item data (played, position, etc.)

## Testing

### Test Framework

- **xUnit** for unit tests
- **Moq** for mocking
- **AutoFixture** for test data generation
- **FsCheck** for property-based testing

### Test Pattern Example

```csharp
public class UserControllerTests
{
    private readonly UserController _subject;
    private readonly Mock<IUserManager> _mockUserManager;

    public UserControllerTests()
    {
        _mockUserManager = new Mock<IUserManager>();
        _subject = new UserController(_mockUserManager.Object, ...);
    }

    [Theory]
    [AutoData]
    public async Task UpdateUserPolicy_WhenUserNotFound_ReturnsNotFound(
        Guid userId,
        UserPolicy userPolicy)
    {
        _mockUserManager
            .Setup(m => m.GetUserById(userId))
            .Returns((User?)null);

        Assert.IsType<NotFoundResult>(
            await _subject.UpdateUserPolicy(userId, userPolicy));
    }
}
```

### Running Tests

```bash
# All tests
dotnet test

# Specific project
dotnet test tests/Jellyfin.Api.Tests

# With verbosity
dotnet test --verbosity normal

# Filter by test name
dotnet test --filter "FullyQualifiedName~UserController"
```

## Common Development Tasks

### Adding a New API Endpoint

1. Create or modify controller in `Jellyfin.Api/Controllers/`
2. Add DTOs in `Jellyfin.Api/Models/` if needed
3. Implement business logic in appropriate service layer
4. Add XML documentation for Swagger
5. Write tests in `tests/Jellyfin.Api.Tests/Controllers/`

### Adding a New Service

1. Define interface in `MediaBrowser.Controller/`
2. Implement in `Emby.Server.Implementations/` or `Jellyfin.Server.Implementations/`
3. Register in DI container (`ApplicationHost.cs`)
4. Write unit tests

### Adding a Metadata Provider

1. Implement `IRemoteMetadataProvider<T>` in `MediaBrowser.Providers/`
2. Register provider in plugin system
3. Add configuration options if needed
4. Write tests in `tests/Jellyfin.Providers.Tests/`

## Development Environment

### VS Code

Launch configurations in `.vscode/launch.json`:
- `.NET Launch (console)` - Full server with web client
- `.NET Launch (nowebclient)` - Server only
- `ghcs .NET Launch (nowebclient, ffmpeg)` - Codespaces with FFmpeg

### Visual Studio

Open `Jellyfin.sln` and press F5 to run with debugging.

### GitHub Codespaces

Pre-configured devcontainers available:
- Default: Basic server environment
- FFmpeg: Includes jellyfin-ffmpeg

## CI/CD

### GitHub Actions Workflows

- `ci-tests.yml` - Runs tests on Ubuntu, macOS, Windows
- `ci-codeql-analysis.yml` - Security scanning
- `ci-openapi.yml` - OpenAPI spec generation
- `ci-compat.yml` - API compatibility checks

### Pre-commit Checks

Ensure before committing:
1. `dotnet build` succeeds
2. `dotnet test` passes
3. No analyzer warnings/errors

## External Dependencies

### FFmpeg

Required for media transcoding. Install `jellyfin-ffmpeg` or system FFmpeg.

### Web Client

The web UI is in a separate repository: `jellyfin/jellyfin-web`. Either:
- Download pre-built from Azure DevOps
- Build from source
- Run with `--nowebclient` flag

## Useful Links

- Documentation: https://jellyfin.org/docs/
- Contributing Guide: https://jellyfin.org/docs/general/contributing/
- Feature Requests: https://features.jellyfin.org/
- Translations: https://translate.jellyfin.org/

## Troubleshooting

### Build Errors

- Ensure .NET 10.0 SDK is installed (`dotnet --version`)
- Run `dotnet restore` to restore packages
- Check `global.json` for SDK version requirements

### Test Failures

- Run with `--verbosity detailed` for more info
- Check for platform-specific tests (Linux/Windows/macOS)
- Ensure no leftover test databases

### Analyzer Errors

- Many rules are set to error severity
- Check `.editorconfig` for rule configuration
- Some rules disabled for test projects
