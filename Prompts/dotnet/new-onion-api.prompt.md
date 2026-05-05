---
agent: 'Code-Generation'
description: 'Scaffold a new .NET Web API project using Onion/Clean Architecture with all layers'
# Note: Uses Feature-Scaffolder agent if available in .github/agents/, otherwise falls back to default Copilot agent
---

# Scaffold .NET Onion Architecture Web API

## Task

Create a complete .NET Web API solution following **Onion Architecture** (also known as Clean Architecture). The solution must be production-ready with proper separation of concerns, dependency injection, and best practices.

## Configuration

Before scaffolding, confirm these settings with the user:

| Setting | Options | Default |
|---------|---------|---------|
| Solution Name | `${SOLUTION_NAME}` | MyCompany.ProjectName |
| .NET Version | 8.0, 9.0 | 9.0 |
| Database | SQL Server, PostgreSQL, SQLite, InMemory | SQL Server |
| Authentication | None, JWT, Identity | JWT |
| API Style | Controllers, Minimal API | Controllers |
| Include Docker | Yes, No | Yes |
| Include GitHub Actions | Yes, No | Yes |

## Solution Structure

Create the following solution structure:

```
${SOLUTION_NAME}/
├── src/
│   ├── ${SOLUTION_NAME}.Core/                    # Domain Layer (innermost)
│   │   ├── Entities/
│   │   ├── Interfaces/
│   │   │   ├── Repositories/
│   │   │   └── Services/
│   │   ├── Specifications/
│   │   ├── Exceptions/
│   │   └── Events/
│   │
│   ├── ${SOLUTION_NAME}.Application/             # Application Layer
│   │   ├── DTOs/
│   │   ├── Interfaces/
│   │   ├── Services/
│   │   ├── Mappings/
│   │   ├── Validators/
│   │   └── Behaviors/
│   │
│   ├── ${SOLUTION_NAME}.Infrastructure/          # Infrastructure Layer
│   │   ├── Data/
│   │   │   ├── Context/
│   │   │   ├── Configurations/
│   │   │   ├── Repositories/
│   │   │   └── Migrations/
│   │   ├── Services/
│   │   ├── Identity/
│   │   └── Extensions/
│   │
│   └── ${SOLUTION_NAME}.Presentation/            # Presentation Layer (outermost)
│       ├── Controllers/
│       ├── Filters/
│       ├── Middleware/
│       └── Extensions/
│
├── tests/
│   ├── ${SOLUTION_NAME}.UnitTests/
│   ├── ${SOLUTION_NAME}.IntegrationTests/
│   └── ${SOLUTION_NAME}.ArchitectureTests/
│
├── docker-compose.yml
├── Dockerfile
├── .github/
│   └── workflows/
│       └── ci-cd.yml
├── .editorconfig
├── Directory.Build.props
├── Directory.Packages.props
└── README.md
```

## Layer Details

### 1. Core Layer (Domain)

The innermost layer containing:

- **Entities**: Domain models with business logic
- **Value Objects**: Immutable objects without identity
- **Domain Events**: Events raised by domain entities
- **Interfaces**: Repository and service abstractions
- **Specifications**: Query specifications pattern
- **Exceptions**: Domain-specific exceptions

```csharp
// Example Entity
public abstract class BaseEntity
{
    public Guid Id { get; protected set; } = Guid.NewGuid();
    public DateTime CreatedAt { get; protected set; } = DateTime.UtcNow;
    public DateTime? UpdatedAt { get; protected set; }
    
    private readonly List<IDomainEvent> _domainEvents = [];
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();
    
    protected void AddDomainEvent(IDomainEvent domainEvent) => _domainEvents.Add(domainEvent);
    public void ClearDomainEvents() => _domainEvents.Clear();
}
```

### 2. Application Layer

Contains application logic:

- **DTOs**: Data Transfer Objects for API contracts
- **Services**: Application services orchestrating use cases
- **Validators**: FluentValidation validators
- **Mappings**: AutoMapper profiles
- **Behaviors**: MediatR pipeline behaviors (logging, validation, caching)

```csharp
// Example Application Service Interface
public interface IEntityService<TDto, TCreateDto, TUpdateDto>
    where TDto : class
    where TCreateDto : class
    where TUpdateDto : class
{
    Task<Result<TDto>> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<Result<PaginatedList<TDto>>> GetAllAsync(PaginationQuery query, CancellationToken ct = default);
    Task<Result<TDto>> CreateAsync(TCreateDto dto, CancellationToken ct = default);
    Task<Result<TDto>> UpdateAsync(Guid id, TUpdateDto dto, CancellationToken ct = default);
    Task<Result> DeleteAsync(Guid id, CancellationToken ct = default);
}
```

### 3. Infrastructure Layer

Contains external concerns:

- **DbContext**: EF Core database context
- **Repositories**: Generic and specific repository implementations
- **Entity Configurations**: Fluent API configurations
- **External Services**: Email, storage, third-party APIs
- **Identity**: Authentication/authorization implementation

```csharp
// Example Generic Repository
public class GenericRepository<T>(AppDbContext context) : IGenericRepository<T> 
    where T : BaseEntity
{
    protected readonly DbSet<T> _dbSet = context.Set<T>();
    
    public async Task<T?> GetByIdAsync(Guid id, CancellationToken ct = default)
        => await _dbSet.FindAsync([id], ct);
    
    public async Task<IEnumerable<T>> GetAllAsync(CancellationToken ct = default)
        => await _dbSet.AsNoTracking().ToListAsync(ct);
    
    public async Task<T> AddAsync(T entity, CancellationToken ct = default)
    {
        await _dbSet.AddAsync(entity, ct);
        return entity;
    }
    
    public void Update(T entity) => _dbSet.Update(entity);
    
    public void Delete(T entity) => _dbSet.Remove(entity);
}
```

### 4. Presentation Layer

The outermost layer:

- **Controllers**: API endpoints with proper HTTP semantics
- **Middleware**: Exception handling, logging, correlation
- **Filters**: Action and exception filters
- **Extensions**: Service registration extensions

```csharp
// Example Controller
[ApiController]
[Route("api/[controller]")]
[Produces("application/json")]
public class EntitiesController(IEntityService service, ILogger<EntitiesController> logger) : ControllerBase
{
    [HttpGet]
    [ProducesResponseType(typeof(PaginatedList<EntityDto>), StatusCodes.Status200OK)]
    public async Task<IActionResult> GetAll([FromQuery] PaginationQuery query, CancellationToken ct)
    {
        var result = await service.GetAllAsync(query, ct);
        return result.IsSuccess ? Ok(result.Value) : result.ToProblemDetails();
    }
}
```

## Required NuGet Packages

### Core
- No external dependencies (pure domain)

### Application
- AutoMapper
- FluentValidation
- FluentValidation.DependencyInjectionExtensions
- MediatR (optional, for CQRS)

### Infrastructure
- Microsoft.EntityFrameworkCore
- Microsoft.EntityFrameworkCore.SqlServer (or other provider)
- Microsoft.AspNetCore.Identity.EntityFrameworkCore (if using Identity)

### Presentation
- Swashbuckle.AspNetCore
- Serilog.AspNetCore
- Microsoft.AspNetCore.Authentication.JwtBearer (if using JWT)

### Tests
- xunit
- xunit.runner.visualstudio
- Moq
- FluentAssertions
- Microsoft.EntityFrameworkCore.InMemory
- NetArchTest.Rules (for architecture tests)

## Execution Steps

1. Create the solution and project structure
2. Set up `Directory.Build.props` for common properties
3. Set up `Directory.Packages.props` for centralized package management
4. Implement the Core layer entities and interfaces
5. Implement the Application layer services and DTOs
6. Implement the Infrastructure layer with EF Core
7. Implement the Presentation layer with controllers
8. Add global exception handling middleware
9. Configure dependency injection in Program.cs
10. Add Swagger/OpenAPI documentation
11. Create Docker configuration if requested
12. Create GitHub Actions workflow if requested
13. Add README.md with setup instructions
14. Add sample architecture tests

## Best Practices to Follow

- ✅ Dependency Rule: Inner layers don't know about outer layers
- ✅ Use interfaces for all cross-layer dependencies
- ✅ Implement Result pattern for error handling
- ✅ Use async/await throughout
- ✅ Apply nullable reference types
- ✅ Use primary constructors for dependency injection
- ✅ Implement proper logging with structured logs
- ✅ Add XML documentation comments on public APIs
- ✅ Use cancellation tokens for async operations
- ✅ Implement Unit of Work pattern with EF Core
