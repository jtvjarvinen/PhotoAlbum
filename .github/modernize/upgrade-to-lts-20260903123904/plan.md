# .NET Upgrade Plan

## Overview

Upgrade PhotoAlbum from .NET 9.0 to .NET 10.0 (latest LTS).

## Project Details

- **Project Name**: PhotoAlbum
- **Current .NET Version**: .NET 9.0 (net9.0)
- **Target .NET Version**: .NET 10.0 (net10.0)
- **Plan Created**: 2026-09-03T15:39:07.522+03:00

## Projects in Solution

1. **PhotoAlbum** (ASP.NET Core Razor Pages application)
   - Current Target Framework: net9.0
   - Target Framework: net10.0
   - SDK Style: ✅ (already modernized)

2. **PhotoAlbum.Tests** (xUnit test project)
   - Current Target Framework: net9.0
   - Target Framework: net10.0
   - SDK Style: ✅ (already modernized)

## Upgrade Scope

The upgrade encompasses:

1. **Target Framework Update**: Update both projects' `<TargetFramework>` from `net9.0` to `net10.0` in their respective .csproj files
2. **NuGet Package Updates**: Update all NuGet packages to their .NET 10.0-compatible versions:
   - Microsoft.EntityFrameworkCore.Design
   - Microsoft.EntityFrameworkCore.SqlServer
   - Microsoft.AspNetCore.Mvc.Testing
   - Microsoft.EntityFrameworkCore.InMemory
   - SixLabors.ImageSharp
   - Microsoft.NET.Test.Sdk
   - xunit and related packages
3. **API Migration**: Review and update any deprecated or changed APIs in .NET 10.0
4. **Testing**: Ensure all unit tests pass with the new framework version

## Technical Details

- **SDK Conversion**: No conversion needed — both projects already use SDK-style project files
- **Breaking Changes**: Minimal expected; review .NET 10.0 release notes for any API changes
- **Database Migration**: Entity Framework Core 9.0 → 10.0 update (no schema changes expected)
- **Build Validation**: Project must compile successfully and all tests must pass

## Success Criteria

✅ Both projects compile successfully with net10.0  
✅ All unit tests pass without modification  
✅ Application runs without errors  
✅ No deprecated API warnings

## Next Steps

Execute the upgrade task: `001-upgrade-dotnet-to-net10`
