# Configuration & Externalized Settings Inventory

PhotoAlbum is an ASP.NET Core 9.0 Razor Pages application with a single configuration source (appsettings.json) and environment-specific overrides. The application uses SQL Server LocalDB for development and supports Azure Blob Storage integration through environment configuration. User Secrets are leveraged for local development credential management.

## Configuration Sources

| Source | Type | Path/Location | Notes |
|--------|------|---------------|-------|
| appsettings.json | Base configuration | `PhotoAlbum/appsettings.json` | Primary configuration file; contains default values for database, file upload, logging, and admin settings |
| appsettings.Development.json | Development overrides | `PhotoAlbum/appsettings.Development.json` | Development-specific settings; overrides logging levels and enables detailed error pages |
| launchSettings.json | Launch profiles | `PhotoAlbum/Properties/launchSettings.json` | Defines HTTP and HTTPS launch profiles with development-specific environment variables |
| User Secrets | Local development secrets | User profile secrets store (ID: `28fdd5b1-4b72-4763-98cc-ac5ebb3f280d`) | Stores sensitive configuration during local development; not committed to version control |
| Environment Variables | Runtime configuration | Process environment | `ASPNETCORE_ENVIRONMENT`, `ConnectionStrings__DefaultConnection`, `FileUpload__*` can override appsettings values |
| Dockerfile | Container configuration | `Dockerfile` | Multi-stage build; exposes port 8080; uses .NET 9.0 runtime and SDK base images |

## Build Profiles

| Profile | Activation | Purpose | Key Dependencies/Plugins |
|---------|-----------|---------|--------------------------|
| Debug | Default in Visual Studio | Development builds with symbols and optimization disabled | Microsoft.AspNetCore.Mvc.Testing (test builds only) |
| Release | Manual `-c Release` or Docker build | Production builds with optimizations enabled; used in Dockerfile multi-stage build | MSBuild optimization, IL trimming disabled |

## Runtime Profiles

| Profile | Activation Method | Config Files | Key Overrides |
|---------|-------------------|--------------|----------------|
| Development | `ASPNETCORE_ENVIRONMENT=Development` (launchSettings.json or environment variable) | appsettings.json + appsettings.Development.json | `Logging.LogLevel.Default` → "Debug" (vs. "Information"), `Logging.LogLevel.Microsoft.EntityFrameworkCore` → "Information", `DetailedErrors` → true, `UseExceptionHandler` middleware skipped |
| Production | Default; no ASPNETCORE_ENVIRONMENT or `ASPNETCORE_ENVIRONMENT=Production` | appsettings.json only | HSTS enabled, exception handler middleware enabled, all logging at default "Information" level |

## Properties Inventory

### Core Application Settings

| Property Key | Default Value | Profiles | Source | Type | Notes |
|--------------|----------------|----------|--------|------|-------|
| `ConnectionStrings.DefaultConnection` | `Server=(localdb)\\mssqllocaldb;Database=PhotoAlbumDb;Trusted_Connection=true;MultipleActiveResultSets=true` | All | appsettings.json or Environment Variable `ConnectionStrings__DefaultConnection` | string | SQL Server LocalDB connection string for development; production deployments override via environment variables |
| `FileUpload.MaxFileSizeBytes` | `10485760` (10 MB) | All | appsettings.json | integer | Maximum file size for photo uploads; enforced in PhotoService and form options |
| `FileUpload.AllowedMimeTypes` | `["image/jpeg", "image/png", "image/gif", "image/webp"]` | All | appsettings.json | array | Whitelist of allowed MIME types for validation |
| `FileUpload.MaxFilesPerUpload` | `10` | All | appsettings.json | integer | Maximum number of files per upload operation |
| `FileUpload.UploadPath` | `wwwroot/uploads` | All | appsettings.json | string | Relative path to uploads directory; created on startup if missing |
| `Logging.LogLevel.Default` | "Information" | All | appsettings.json | string | Default logging level; overridden to "Debug" in Development |
| `Logging.LogLevel.Microsoft.AspNetCore` | "Warning" | All | appsettings.json | string | Reduces verbosity of ASP.NET Core framework logs |
| `Logging.LogLevel.Microsoft.EntityFrameworkCore` | (not set in base) | Development | appsettings.Development.json | string | Development override: set to "Information" to show EF Core SQL |
| `Admin.Username` | `"admin"` | All | appsettings.json | string | Default admin username placeholder |
| `AllowedHosts` | `"*"` | All | appsettings.json | string | Allowed hosts for Host header validation; "*" allows any host |
| `DetailedErrors` | `false` (default) | Development | appsettings.Development.json | boolean | Development override: enables detailed error pages with stack traces |
| `IsTestEnvironment` | `false` (default) | Test | Environment Variable or test configuration | boolean | Feature flag: skips database migrations on startup when true (used in test environment setup) |

## Startup Parameters & Resource Requirements

| Component | Runtime Options | Memory (Container) | CPU (Container) | Instance Count | Notes |
|-----------|-----------------|-------------------|-----------------|-----------------|-------|
| PhotoAlbum ASP.NET Core App | Default CLR settings; no explicit JVM-equivalent options | Not specified in Dockerfile (inherited from base image) | Not specified | Single instance | Application runs in-process; no separate background workers; uses Entity Framework Core lazy-loaded navigation (in-memory for small datasets) |
| SQL Server LocalDB | N/A | N/A | N/A | Single instance (local development only) | Development uses LocalDB; production deployments connect to external SQL Server or Azure SQL Database |
| Uploads Directory | N/A | N/A | N/A | N/A | Created on startup at `{ContentRootPath}/wwwroot/uploads`; uses local filesystem storage (development/testing) or Azure Blob Storage (production) |

**Container Resource Limits (from Dockerfile):**
- No explicit memory or CPU limits defined in Dockerfile or docker-compose
- Port binding: 8080 (HTTP)
- Base image: `mcr.microsoft.com/dotnet/aspnet:9.0` (production runtime)

## Startup Dependency Chain

1. **Program.cs Initialization** (immediate):
   - WebApplicationBuilder created
   - Services registered: Razor Pages, Authentication, DbContext, PhotoService
   - Form options configured for 10 MB file uploads
   
2. **Application startup** (first run only or non-test environments):
   - Uploads directory created if missing (`wwwroot/uploads`)
   - Database migrations applied via `context.Database.MigrateAsync()` (if `IsTestEnvironment` is false)
   - Failure to apply migrations causes immediate application exit
   
3. **Request pipeline initialization** (after services started):
   - Static file serving configured with 1-hour cache headers
   - Authentication and authorization middleware activated
   - Razor Pages route mapping configured

**Readiness Mechanism:** Application is ready to serve requests after successful startup. No explicit health checks defined; HTTP 200 on `/` indicates readiness.

**Startup Timeout:** None explicitly configured; ASP.NET Core default startup timeout is ~30 seconds before process termination.

## Secrets & Sensitive Configuration

| Secret Reference | Type | Storage | Access Pattern |
|------------------|------|---------|-----------------|
| `ConnectionStrings.DefaultConnection` | Database connection string | appsettings.json (local dev) / Environment Variable (production) | Configuration binding via `IConfiguration.GetConnectionString()` in Program.cs |
| Database password (LocalDB) | Implicit (Trusted_Connection=true) | Windows authentication (LocalDB) | No explicit password; uses current Windows identity |
| User Secrets (Development) | General secrets storage | User Secrets manager (local machine, not committed) | Secrets tool integration in Visual Studio; ID `28fdd5b1-4b72-4763-98cc-ac5ebb3f280d` |
| Admin.Username | Placeholder credential | appsettings.json | Plain text (no actual password configured) |

### Secrets Provisioning Workflow

**Development Environment:**
1. Secrets stored in User Secrets manager via `dotnet user-secrets` command or Visual Studio Secrets Manager
2. Visual Studio automatically merges User Secrets at runtime during development
3. Connection string can be overridden via User Secrets or environment variables
4. No external secret store required; development-only

**Production Environment (proposed for Azure deployment):**
1. Connection string provisioned via environment variable `ConnectionStrings__DefaultConnection` (set by deployment pipeline)
2. Deployment (GitHub Actions or manual) injects SQL Server connection string into container at runtime
3. Optional: Azure Key Vault integration can be added by injecting secrets during container orchestration
4. Application code requires no changes; configuration binding handles `__` double-underscore notation (hierarchical key separator)

**Test Environment:**
1. In-memory database via `Microsoft.EntityFrameworkCore.InMemory` (PhotoAlbum.Tests.csproj)
2. No database connection string needed
3. `IsTestEnvironment` flag set to skip migrations on startup
4. Tests use `Microsoft.AspNetCore.Mvc.Testing` WebApplicationFactory for isolated test contexts

## Feature Flags

| Flag Name | Default | Type | Controlled By | Purpose |
|-----------|---------|------|----------------|---------|
| `IsTestEnvironment` | `false` | Boolean | Environment variable or test configuration | Conditional: skip database migrations on startup if true; allows test environments to use in-memory database without migration errors |
| `ASPNETCORE_ENVIRONMENT` | "Production" (if unset) | String | launchSettings.json, environment variable | Selects runtime profile; "Development" enables detailed errors and debug logging; "Production" enables HSTS and exception handler |

## Framework & Runtime Versions

| Component | Version | Source | Notes |
|-----------|---------|--------|-------|
| .NET Framework | 9.0 | PhotoAlbum.csproj `<TargetFramework>` | Target runtime for compilation and execution |
| ASP.NET Core | 9.0 (implicit in .NET 9.0) | NuGet implicit dependency | Web framework version |
| Entity Framework Core | 9.0.9 | PhotoAlbum.csproj NuGet reference | ORM for SQL Server data access |
| Entity Framework Core SqlServer | 9.0.9 | PhotoAlbum.csproj NuGet reference | SQL Server provider for EF Core |
| Entity Framework Core Design | 9.0.9 | PhotoAlbum.csproj NuGet reference | Design-time tools for migrations |
| SixLabors.ImageSharp | 3.1.11 | PhotoAlbum.csproj NuGet reference | Image processing library for extracting dimensions and validation |
| Microsoft.AspNetCore.Mvc.Testing | 9.0.9 | PhotoAlbum.Tests.csproj NuGet reference | Integration testing framework for Razor Pages |
| Microsoft.EntityFrameworkCore.InMemory | 9.0.9 | PhotoAlbum.Tests.csproj NuGet reference | In-memory database provider for unit/integration tests |
| xUnit | 2.9.2 | PhotoAlbum.Tests.csproj NuGet reference | Unit testing framework |
| coverlet.collector | 6.0.2 | PhotoAlbum.Tests.csproj NuGet reference | Code coverage collection tool |
| Docker Runtime Base Image | mcr.microsoft.com/dotnet/aspnet:9.0 | Dockerfile `FROM` | Production runtime container; contains .NET 9.0 runtime only |
| Docker Build Image | mcr.microsoft.com/dotnet/sdk:9.0 | Dockerfile `FROM` | Build container; contains .NET 9.0 SDK for compilation and publish |
| Node.js / npm | Not used | N/A | Frontend build uses none; static assets bundled with publish |
| C# Version | Latest (11.0 default for .NET 9.0) | Implicit via .NET 9.0 | Nullable reference types enabled via `<Nullable>enable</Nullable>` |

## Configuration Validation & Error Handling

- **Connection String Validation:** Applied at startup via EF Core `UseSqlServer()` configuration; migration failure causes immediate application crash (fail-fast)
- **File Upload Validation:** Performed by PhotoService; MIME type and size checked before persistence
- **Missing Uploads Directory:** Automatically created on startup; no error if creation fails silently (logs to console)
- **Logging Configuration:** Invalid log levels silently default to "Information"; no runtime validation errors
- **Form Size Limits:** Enforced at middleware level (FormOptions); oversized requests rejected with HTTP 413 Payload Too Large

## Additional Notes

- **No environment-specific production config file** (e.g., `appsettings.Production.json`): Production uses base `appsettings.json` with environment variable overrides
- **Implicit UsingsEnable:** `<ImplicitUsings>enable</ImplicitUsings>` reduces boilerplate; namespace imports generated at compile time
- **Nullable Reference Types:** `<Nullable>enable</Nullable>` enforces strict null-safety checking at compile time
- **Static File Caching:** 1-hour cache headers applied to all static assets in `wwwroot`; affects browser caching of photos and UI assets
- **Form Options Hardcoded:** File upload limits (10 MB) hardcoded in Program.cs; mirrored with `FileUpload.MaxFileSizeBytes` from config (dual configuration point)
