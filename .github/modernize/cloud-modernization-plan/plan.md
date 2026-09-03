# Modernization Plan: Cloud Modernization

**Project**: PhotoAlbum

---

## Technical Framework

- **Language**: C# / .NET 10.0
- **Framework**: ASP.NET Core 10.0 Razor Pages
- **Build Tool**: MSBuild / .NET CLI
- **Database**: SQL Server LocalDB (development)
- **Key Dependencies**: Entity Framework Core 10.0, SixLabors.ImageSharp

---

## Overview

This migration prepares the PhotoAlbum application for reliable operation on Azure.
The application currently stores photos on the local filesystem and uses a local
SQL Server connection. The new architecture will:

- Store photo content in durable Azure storage so files are not tied to an
  individual application instance.
- Use a managed Azure SQL database and workload identity authentication for
  cloud-hosted persistence.
- Externalize runtime configuration and provide repeatable Azure infrastructure
  and deployment.

The application is already on .NET 10.0, so framework upgrade recommendations
from the previous assessment are intentionally out of scope.

---

## Migration Impact Summary

| Application | Original Service | New Azure Service | Authentication | Comments |
|-------------|------------------|-------------------|----------------|----------|
| PhotoAlbum | Local photo files | Azure Blob Storage | Managed Identity | Durable photo storage |
| PhotoAlbum | SQL Server LocalDB | Azure SQL Database | Managed Identity | Cloud database |
| PhotoAlbum | Local appsettings | Azure App Configuration | Managed Identity | Runtime settings |
| PhotoAlbum | Local deployment | Azure Container Apps | Managed Identity | Container deployment |

---

## Open Questions & Questionnaire

- [x] Q: Which Azure deployment target should be used? → A: Azure Container Apps,
  based on the existing Dockerfile and Azure deployment configuration.
- [x] Q: Should an OracleDB migration to PostgreSQL be included? → A: No.
  The current application uses SQL Server LocalDB and no OracleDB exists.
- [x] Q: Should integration tests be added? → A: Not requested; no integration
  test task is included.

---

## Security Compliance

Scan all project dependencies for known CVEs and remediate any identified
vulnerabilities before deployment. Upgrade vulnerable dependencies to the
minimum patched version, document any unavoidable major-version changes, and
verify that the project builds and its tests pass.

