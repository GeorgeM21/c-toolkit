---
agent: 'Code-Generation'
description: 'Scaffold a new .NET Web API using CQRS pattern with MediatR'
# Note: Uses Feature-Scaffolder agent if available in .github/agents/, otherwise falls back to default Copilot agent
---

# Scaffold .NET CQRS Web API with MediatR

## Task

Create a complete .NET Web API solution implementing the **CQRS pattern with MediatR**. The solution separates commands (writes) from queries (reads) and uses pipeline behaviors for cross-cutting concerns.

## Configuration

| Setting | Options | Default |
|---------|---------|---------|
| Solution Name | `${SOLUTION_NAME}` | MyCompany.ProjectName |
| .NET Version | 8.0, 9.0 | 9.0 |
| Database | SQL Server, PostgreSQL, SQLite | SQL Server |
| Separate Read DB | Yes, No | No |
| Include Event Sourcing | Yes, No | No |
| Include Docker | Yes, No | Yes |

## Solution Structure

```
${SOLUTION_NAME}/
├── src/
│   ├── ${SOLUTION_NAME}.Domain/
│   │   ├── Entities/
│   │   ├── Events/
│   │   ├── Exceptions/
│   │   └── ValueObjects/
│   │
│   ├── ${SOLUTION_NAME}.Application/
│   │   ├── Common/
│   │   │   ├── Behaviors/
│   │   │   │   ├── LoggingBehavior.cs
│   │   │   │   ├── ValidationBehavior.cs
│   │   │   │   ├── PerformanceBehavior.cs
│   │   │   │   └── TransactionBehavior.cs
│   │   │   ├── Interfaces/
│   │   │   ├── Mappings/
│   │   │   └── Models/
│   │   │       └── Result.cs
│   │   │
│   │   ├── Features/
│   │   │   └── ${FeatureName}/
│   │   │       ├── Commands/
│   │   │       │   ├── Create${Entity}/
│   │   │       │   │   ├── Create${Entity}Command.cs
│   │   │       │   │   ├── Create${Entity}CommandHandler.cs
│   │   │       │   │   └── Create${Entity}CommandValidator.cs
│   │   │       │   ├── Update${Entity}/
│   │   │       │   └── Delete${Entity}/
│   │   │       └── Queries/
│   │   │           ├── Get${Entity}ById/
│   │   │           │   ├── Get${Entity}ByIdQuery.cs
│   │   │           │   ├── Get${Entity}ByIdQueryHandler.cs
│   │   │           │   └── ${Entity}Dto.cs
│   │   │           └── Get${Entity}List/
│   │   │
│   │   └── DependencyInjection.cs
│   │
│   ├── ${SOLUTION_NAME}.Infrastructure/
│   │   ├── Persistence/
│   │   │   ├── WriteDbContext.cs
│   │   │   ├── ReadDbContext.cs (if separate read DB)
│   │   │   └── Configurations/
│   │   ├── Repositories/
│   │   └── DependencyInjection.cs
│   │
│   └── ${SOLUTION_NAME}.API/
│       ├── Controllers/
│       ├── Middleware/
│       └── Program.cs
│
└── tests/
    ├── ${SOLUTION_NAME}.UnitTests/
    └── ${SOLUTION_NAME}.IntegrationTests/
```

## Core Components

### 1. Command Pattern

```csharp
// ICommand marker interface
public interface ICommand<TResponse> : IRequest<Result<TResponse>> { }

// Command record
public sealed record CreateProductCommand(
    string Name,
    string Description,
    decimal Price,
    int CategoryId
) : ICommand<Guid>;

// Command Handler
public sealed class CreateProductCommandHandler(
    IProductRepository repository,
    IUnitOfWork unitOfWork,
    ILogger<CreateProductCommandHandler> logger)
    : IRequestHandler<CreateProductCommand, Result<Guid>>
{
    public async Task<Result<Guid>> Handle(
        CreateProductCommand request,
        CancellationToken cancellationToken)
    {
        logger.LogInformation("Creating product: {Name}", request.Name);
        
        var product = Product.Create(
            request.Name,
            request.Description,
            request.Price,
            request.CategoryId);
        
        await repository.AddAsync(product, cancellationToken);
        await unitOfWork.SaveChangesAsync(cancellationToken);
        
        return Result<Guid>.Success(product.Id);
    }
}

// Command Validator
public sealed class CreateProductCommandValidator : AbstractValidator<CreateProductCommand>
{
    public CreateProductCommandValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Name is required")
            .MaximumLength(200).WithMessage("Name must not exceed 200 characters");
        
        RuleFor(x => x.Price)
            .GreaterThan(0).WithMessage("Price must be greater than 0");
        
        RuleFor(x => x.CategoryId)
            .GreaterThan(0).WithMessage("Valid category is required");
    }
}
```

### 2. Query Pattern

```csharp
// IQuery marker interface
public interface IQuery<TResponse> : IRequest<Result<TResponse>> { }

// Query record
public sealed record GetProductByIdQuery(Guid Id) : IQuery<ProductDto>;

// Query Handler (uses read-optimized approach)
public sealed class GetProductByIdQueryHandler(
    IReadDbContext context,
    ILogger<GetProductByIdQueryHandler> logger)
    : IRequestHandler<GetProductByIdQuery, Result<ProductDto>>
{
    public async Task<Result<ProductDto>> Handle(
        GetProductByIdQuery request,
        CancellationToken cancellationToken)
    {
        var product = await context.Products
            .AsNoTracking()
            .Where(p => p.Id == request.Id)
            .Select(p => new ProductDto
            {
                Id = p.Id,
                Name = p.Name,
                Description = p.Description,
                Price = p.Price,
                CategoryName = p.Category.Name,
                CreatedAt = p.CreatedAt
            })
            .FirstOrDefaultAsync(cancellationToken);
        
        return product is null
            ? Result<ProductDto>.Failure(Error.NotFound("Product.NotFound", $"Product {request.Id} not found"))
            : Result<ProductDto>.Success(product);
    }
}
```

### 3. Pipeline Behaviors

```csharp
// Validation Behavior
public sealed class ValidationBehavior<TRequest, TResponse>(
    IEnumerable<IValidator<TRequest>> validators,
    ILogger<ValidationBehavior<TRequest, TResponse>> logger)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        if (!validators.Any())
            return await next();
        
        var context = new ValidationContext<TRequest>(request);
        var validationResults = await Task.WhenAll(
            validators.Select(v => v.ValidateAsync(context, cancellationToken)));
        
        var failures = validationResults
            .SelectMany(r => r.Errors)
            .Where(f => f is not null)
            .ToList();
        
        if (failures.Count > 0)
        {
            logger.LogWarning("Validation failed for {Request}: {Errors}",
                typeof(TRequest).Name, string.Join(", ", failures.Select(f => f.ErrorMessage)));
            throw new ValidationException(failures);
        }
        
        return await next();
    }
}

// Logging Behavior
public sealed class LoggingBehavior<TRequest, TResponse>(
    ILogger<LoggingBehavior<TRequest, TResponse>> logger)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        var requestName = typeof(TRequest).Name;
        logger.LogInformation("Handling {RequestName}: {@Request}", requestName, request);
        
        var stopwatch = Stopwatch.StartNew();
        var response = await next();
        stopwatch.Stop();
        
        logger.LogInformation("Handled {RequestName} in {ElapsedMs}ms",
            requestName, stopwatch.ElapsedMilliseconds);
        
        return response;
    }
}

// Transaction Behavior (for commands only)
public sealed class TransactionBehavior<TRequest, TResponse>(
    IUnitOfWork unitOfWork,
    ILogger<TransactionBehavior<TRequest, TResponse>> logger)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : ICommand<TResponse>
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        await using var transaction = await unitOfWork.BeginTransactionAsync(cancellationToken);
        try
        {
            var response = await next();
            await transaction.CommitAsync(cancellationToken);
            return response;
        }
        catch
        {
            await transaction.RollbackAsync(cancellationToken);
            throw;
        }
    }
}
```

### 4. API Controllers

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController(ISender sender) : ControllerBase
{
    [HttpGet("{id:guid}")]
    [ProducesResponseType(typeof(ProductDto), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetById(Guid id, CancellationToken ct)
    {
        var result = await sender.Send(new GetProductByIdQuery(id), ct);
        return result.IsSuccess ? Ok(result.Value) : NotFound(result.Error.ToProblemDetails());
    }
    
    [HttpGet]
    [ProducesResponseType(typeof(PaginatedList<ProductDto>), StatusCodes.Status200OK)]
    public async Task<IActionResult> GetAll([FromQuery] GetProductListQuery query, CancellationToken ct)
    {
        var result = await sender.Send(query, ct);
        return Ok(result.Value);
    }
    
    [HttpPost]
    [ProducesResponseType(typeof(Guid), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> Create([FromBody] CreateProductCommand command, CancellationToken ct)
    {
        var result = await sender.Send(command, ct);
        return result.IsSuccess
            ? CreatedAtAction(nameof(GetById), new { id = result.Value }, result.Value)
            : BadRequest(result.Error.ToProblemDetails());
    }
    
    [HttpPut("{id:guid}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Update(Guid id, [FromBody] UpdateProductCommand command, CancellationToken ct)
    {
        if (id != command.Id)
            return BadRequest("Route ID and command ID mismatch");
        
        var result = await sender.Send(command, ct);
        return result.IsSuccess ? NoContent() : NotFound(result.Error.ToProblemDetails());
    }
    
    [HttpDelete("{id:guid}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Delete(Guid id, CancellationToken ct)
    {
        var result = await sender.Send(new DeleteProductCommand(id), ct);
        return result.IsSuccess ? NoContent() : NotFound(result.Error.ToProblemDetails());
    }
}
```

### 5. Dependency Injection Setup

```csharp
// Application Layer DI
public static class DependencyInjection
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        var assembly = Assembly.GetExecutingAssembly();
        
        services.AddMediatR(cfg =>
        {
            cfg.RegisterServicesFromAssembly(assembly);
            cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
            cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
            cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(PerformanceBehavior<,>));
        });
        
        services.AddValidatorsFromAssembly(assembly);
        services.AddAutoMapper(assembly);
        
        return services;
    }
}
```

## Required NuGet Packages

```xml
<!-- Application Layer -->
<PackageReference Include="MediatR" Version="12.*" />
<PackageReference Include="FluentValidation" Version="11.*" />
<PackageReference Include="FluentValidation.DependencyInjectionExtensions" Version="11.*" />
<PackageReference Include="AutoMapper" Version="13.*" />

<!-- Infrastructure Layer -->
<PackageReference Include="Microsoft.EntityFrameworkCore" Version="9.*" />
<PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="9.*" />

<!-- API Layer -->
<PackageReference Include="Swashbuckle.AspNetCore" Version="6.*" />
<PackageReference Include="Serilog.AspNetCore" Version="8.*" />
```

## Best Practices

- ✅ One handler per command/query (Single Responsibility)
- ✅ Use records for immutable commands and queries
- ✅ Validate commands before execution with FluentValidation
- ✅ Use Result pattern instead of exceptions for expected failures
- ✅ Separate read and write models (DTOs vs Entities)
- ✅ Use pipeline behaviors for cross-cutting concerns
- ✅ Keep handlers small and focused
- ✅ Use cancellation tokens throughout
- ✅ Log at the behavior level, not in every handler
- ✅ Consider separate read database for complex read scenarios
