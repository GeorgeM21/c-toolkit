---
applyTo: '**/*.{cs,csproj,sln}'
description: ".NET-specific coding standards and best practices"
priority: 8
---

# .NET Development Best Practices

These guidelines extend the general standards in `Instructions/copilot-instructions.md`.
If a .NET-specific rule conflicts with a general rule, prefer the .NET rule.

## Language Features
- Use C# latest language version features appropriately
- Prefer pattern matching over type checking
- Use `nullable reference types` (enabled by default in .NET 6+)
- Use records for immutable data transfer objects
- Use `init` accessors for immutable properties

## Dependency Injection
- Register services in `Program.cs` or `Startup.cs`
- Use constructor injection (avoid service locator pattern)
- Prefer `AddScoped` for request-scoped services
- Use `AddSingleton` for stateless services
- Use `AddTransient` for lightweight, stateless services

## Async/Await in .NET
- Use `async`/`await` for all I/O operations
- Avoid `async void` except for event handlers
- Use `Task.WhenAll` for parallel operations
- Use `ValueTask<T>` for hot paths (when optimization needed)
- Use `ConfigureAwait(false)` in library code, in addition to the general async guidance

## Entity Framework Core
- Use migrations for schema changes
- Enable query tracking only when needed
- Use `AsNoTracking()` for read-only queries
- Avoid lazy loading in APIs (prefer eager loading)
- Use projections to select only needed columns

## ASP.NET Core Specifics
- Use minimal APIs for simple endpoints (.NET 6+)
- Implement proper model validation with `[Required]`, `[Range]`, etc.
- Use `ActionResult<T>` for typed responses
- Implement proper exception handling middleware
- Use `IOptions<T>` for configuration

## Testing in .NET
- Use xUnit, NUnit, or MSTest
- Use FluentAssertions for readable assertions
- Mock dependencies with Moq or NSubstitute
- Use `WebApplicationFactory<T>` for integration testing

## NuGet Package Management
- Use Central Package Management (.NET SDK 7.0+)
- Keep packages up-to-date
- Review package vulnerabilities with `dotnet list package --vulnerable`

## Performance
- Use `Span<T>` and `Memory<T>` for low-allocation code
- Use `ArrayPool<T>` for array rentals
- Profile with BenchmarkDotNet
- Use `StringBuilder` for string concatenation in loops

## Related Instructions
- **General practices**: See root `copilot-instructions.md`
