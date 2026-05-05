---
agent: 'Code-Generation'
description: 'Scaffold a new .NET Minimal API with best practices and OpenAPI documentation'
# Note: Uses Feature-Scaffolder agent if available in .github/agents/, otherwise falls back to default Copilot agent
---

# Scaffold .NET Minimal API

## Task

Create a .NET Minimal API project following best practices for organization, documentation, and maintainability. The API should be production-ready with proper validation, error handling, and OpenAPI documentation.

## Configuration

| Setting | Options | Default |
|---------|---------|---------|
| Project Name | `${PROJECT_NAME}` | MyMinimalApi |
| .NET Version | 8.0, 9.0 | 9.0 |
| Database | SQL Server, PostgreSQL, SQLite, InMemory | SQLite |
| Authentication | None, JWT, API Keys | None |
| Include Docker | Yes, No | Yes |
| Use Vertical Slices | Yes, No | Yes |

## Solution Structure

### Vertical Slice Architecture (Recommended)

```
${PROJECT_NAME}/
├── src/
│   └── ${PROJECT_NAME}.Api/
│       ├── Features/
│       │   ├── Products/
│       │   │   ├── Endpoints.cs
│       │   │   ├── Models.cs
│       │   │   ├── Handlers.cs
│       │   │   └── Validators.cs
│       │   ├── Categories/
│       │   │   └── ...
│       │   └── Orders/
│       │       └── ...
│       ├── Common/
│       │   ├── Extensions/
│       │   ├── Middleware/
│       │   ├── Filters/
│       │   └── Models/
│       │       └── Result.cs
│       ├── Data/
│       │   ├── AppDbContext.cs
│       │   └── Configurations/
│       ├── Program.cs
│       └── appsettings.json
│
├── tests/
│   ├── ${PROJECT_NAME}.UnitTests/
│   └── ${PROJECT_NAME}.IntegrationTests/
│
├── Dockerfile
└── docker-compose.yml
```

## Core Implementation

### 1. Program.cs Setup

```csharp
using Microsoft.EntityFrameworkCore;
using FluentValidation;
using Serilog;

var builder = WebApplication.CreateBuilder(args);

// Configure Serilog
builder.Host.UseSerilog((context, config) =>
    config.ReadFrom.Configuration(context.Configuration));

// Database
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite(builder.Configuration.GetConnectionString("DefaultConnection")));

// Validation
builder.Services.AddValidatorsFromAssemblyContaining<Program>();

// OpenAPI with .NET 9
builder.Services.AddOpenApi(options =>
{
    options.AddDocumentTransformer((document, context, ct) =>
    {
        document.Info.Title = "${PROJECT_NAME} API";
        document.Info.Version = "v1";
        document.Info.Description = "A modern .NET Minimal API";
        return Task.CompletedTask;
    });
});

// CORS
builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
    {
        policy.WithOrigins(builder.Configuration.GetSection("AllowedOrigins").Get<string[]>() ?? [])
              .AllowAnyMethod()
              .AllowAnyHeader();
    });
});

// Services
builder.Services.AddScoped<IProductRepository, ProductRepository>();

var app = builder.Build();

// Configure pipeline
if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
    app.UseSwaggerUI(options => options.SwaggerEndpoint("/openapi/v1.json", "v1"));
}

app.UseSerilogRequestLogging();
app.UseCors();
app.UseHttpsRedirection();

// Map endpoints
app.MapProductEndpoints();
app.MapCategoryEndpoints();
app.MapOrderEndpoints();

// Health check
app.MapHealthChecks("/health");

app.Run();

// Required for integration testing
public partial class Program { }
```

### 2. Feature-Based Endpoint Organization

```csharp
// Features/Products/Endpoints.cs
public static class ProductEndpoints
{
    public static void MapProductEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/products")
            .WithTags("Products")
            .WithOpenApi();
        
        group.MapGet("/", GetAllProducts)
            .WithName("GetAllProducts")
            .WithSummary("Get all products")
            .WithDescription("Retrieves a paginated list of all products")
            .Produces<PaginatedList<ProductResponse>>()
            .ProducesProblem(StatusCodes.Status500InternalServerError);
        
        group.MapGet("/{id:guid}", GetProductById)
            .WithName("GetProductById")
            .WithSummary("Get product by ID")
            .Produces<ProductResponse>()
            .ProducesProblem(StatusCodes.Status404NotFound);
        
        group.MapPost("/", CreateProduct)
            .WithName("CreateProduct")
            .WithSummary("Create a new product")
            .Produces<ProductResponse>(StatusCodes.Status201Created)
            .ProducesValidationProblem();
        
        group.MapPut("/{id:guid}", UpdateProduct)
            .WithName("UpdateProduct")
            .WithSummary("Update an existing product")
            .Produces(StatusCodes.Status204NoContent)
            .ProducesProblem(StatusCodes.Status404NotFound)
            .ProducesValidationProblem();
        
        group.MapDelete("/{id:guid}", DeleteProduct)
            .WithName("DeleteProduct")
            .WithSummary("Delete a product")
            .Produces(StatusCodes.Status204NoContent)
            .ProducesProblem(StatusCodes.Status404NotFound);
    }
    
    private static async Task<IResult> GetAllProducts(
        [AsParameters] PaginationQuery query,
        IProductRepository repository,
        CancellationToken ct)
    {
        var products = await repository.GetAllAsync(query.Page, query.PageSize, ct);
        return TypedResults.Ok(products);
    }
    
    private static async Task<IResult> GetProductById(
        Guid id,
        IProductRepository repository,
        CancellationToken ct)
    {
        var product = await repository.GetByIdAsync(id, ct);
        
        return product is not null
            ? TypedResults.Ok(product.ToResponse())
            : TypedResults.Problem(
                statusCode: StatusCodes.Status404NotFound,
                title: "Product not found",
                detail: $"Product with ID {id} was not found");
    }
    
    private static async Task<IResult> CreateProduct(
        CreateProductRequest request,
        IValidator<CreateProductRequest> validator,
        IProductRepository repository,
        CancellationToken ct)
    {
        var validationResult = await validator.ValidateAsync(request, ct);
        if (!validationResult.IsValid)
        {
            return TypedResults.ValidationProblem(validationResult.ToDictionary());
        }
        
        var product = request.ToEntity();
        await repository.AddAsync(product, ct);
        
        return TypedResults.CreatedAtRoute(
            product.ToResponse(),
            "GetProductById",
            new { id = product.Id });
    }
    
    private static async Task<IResult> UpdateProduct(
        Guid id,
        UpdateProductRequest request,
        IValidator<UpdateProductRequest> validator,
        IProductRepository repository,
        CancellationToken ct)
    {
        var validationResult = await validator.ValidateAsync(request, ct);
        if (!validationResult.IsValid)
        {
            return TypedResults.ValidationProblem(validationResult.ToDictionary());
        }
        
        var product = await repository.GetByIdAsync(id, ct);
        if (product is null)
        {
            return TypedResults.Problem(
                statusCode: StatusCodes.Status404NotFound,
                title: "Product not found");
        }
        
        product.Update(request.Name, request.Description, request.Price);
        await repository.UpdateAsync(product, ct);
        
        return TypedResults.NoContent();
    }
    
    private static async Task<IResult> DeleteProduct(
        Guid id,
        IProductRepository repository,
        CancellationToken ct)
    {
        var product = await repository.GetByIdAsync(id, ct);
        if (product is null)
        {
            return TypedResults.Problem(
                statusCode: StatusCodes.Status404NotFound,
                title: "Product not found");
        }
        
        await repository.DeleteAsync(product, ct);
        return TypedResults.NoContent();
    }
}
```

### 3. Request/Response Models

```csharp
// Features/Products/Models.cs
public record ProductResponse(
    Guid Id,
    string Name,
    string Description,
    decimal Price,
    string CategoryName,
    DateTime CreatedAt);

public record CreateProductRequest(
    string Name,
    string Description,
    decimal Price,
    Guid CategoryId);

public record UpdateProductRequest(
    string Name,
    string Description,
    decimal Price);

public record PaginationQuery(int Page = 1, int PageSize = 10)
{
    public int Page { get; init; } = Page < 1 ? 1 : Page;
    public int PageSize { get; init; } = PageSize is < 1 or > 100 ? 10 : PageSize;
}

public record PaginatedList<T>(
    IReadOnlyList<T> Items,
    int TotalCount,
    int Page,
    int PageSize)
{
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasPreviousPage => Page > 1;
    public bool HasNextPage => Page < TotalPages;
}

// Mapping extensions
public static class ProductMappings
{
    public static ProductResponse ToResponse(this Product product) =>
        new(
            product.Id,
            product.Name,
            product.Description,
            product.Price,
            product.Category?.Name ?? string.Empty,
            product.CreatedAt);
    
    public static Product ToEntity(this CreateProductRequest request) =>
        new()
        {
            Id = Guid.NewGuid(),
            Name = request.Name,
            Description = request.Description,
            Price = request.Price,
            CategoryId = request.CategoryId,
            CreatedAt = DateTime.UtcNow
        };
}
```

### 4. Validators

```csharp
// Features/Products/Validators.cs
public class CreateProductRequestValidator : AbstractValidator<CreateProductRequest>
{
    public CreateProductRequestValidator(AppDbContext context)
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Name is required")
            .MaximumLength(200).WithMessage("Name must not exceed 200 characters")
            .MustAsync(async (name, ct) => !await context.Products.AnyAsync(p => p.Name == name, ct))
            .WithMessage("A product with this name already exists");
        
        RuleFor(x => x.Description)
            .MaximumLength(2000).WithMessage("Description must not exceed 2000 characters");
        
        RuleFor(x => x.Price)
            .GreaterThan(0).WithMessage("Price must be greater than 0")
            .PrecisionScale(18, 2, true).WithMessage("Price cannot have more than 2 decimal places");
        
        RuleFor(x => x.CategoryId)
            .NotEmpty().WithMessage("Category is required")
            .MustAsync(async (id, ct) => await context.Categories.AnyAsync(c => c.Id == id, ct))
            .WithMessage("Category does not exist");
    }
}

public class UpdateProductRequestValidator : AbstractValidator<UpdateProductRequest>
{
    public UpdateProductRequestValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Name is required")
            .MaximumLength(200).WithMessage("Name must not exceed 200 characters");
        
        RuleFor(x => x.Description)
            .MaximumLength(2000).WithMessage("Description must not exceed 2000 characters");
        
        RuleFor(x => x.Price)
            .GreaterThan(0).WithMessage("Price must be greater than 0");
    }
}
```

### 5. Endpoint Filters

```csharp
// Common/Filters/ValidationFilter.cs
public class ValidationFilter<T>(IValidator<T> validator) : IEndpointFilter where T : class
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var argument = context.Arguments.OfType<T>().FirstOrDefault();
        
        if (argument is null)
        {
            return TypedResults.BadRequest("Invalid request body");
        }
        
        var validationResult = await validator.ValidateAsync(argument);
        
        if (!validationResult.IsValid)
        {
            return TypedResults.ValidationProblem(validationResult.ToDictionary());
        }
        
        return await next(context);
    }
}

// Usage
group.MapPost("/", CreateProduct)
    .AddEndpointFilter<ValidationFilter<CreateProductRequest>>();
```

### 6. Global Exception Handler

```csharp
// Common/Middleware/GlobalExceptionHandler.cs
public class GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        logger.LogError(exception, "An unhandled exception occurred: {Message}", exception.Message);
        
        var problemDetails = exception switch
        {
            ValidationException validationException => new ProblemDetails
            {
                Status = StatusCodes.Status400BadRequest,
                Title = "Validation Error",
                Detail = "One or more validation errors occurred.",
                Extensions = { ["errors"] = validationException.Errors }
            },
            NotFoundException notFoundException => new ProblemDetails
            {
                Status = StatusCodes.Status404NotFound,
                Title = "Not Found",
                Detail = notFoundException.Message
            },
            _ => new ProblemDetails
            {
                Status = StatusCodes.Status500InternalServerError,
                Title = "Server Error",
                Detail = "An unexpected error occurred."
            }
        };
        
        httpContext.Response.StatusCode = problemDetails.Status ?? 500;
        await httpContext.Response.WriteAsJsonAsync(problemDetails, cancellationToken);
        
        return true;
    }
}

// Registration in Program.cs
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();

// In pipeline
app.UseExceptionHandler();
```

## Required NuGet Packages

```xml
<ItemGroup>
    <PackageReference Include="Microsoft.EntityFrameworkCore" Version="9.*" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="9.*" />
    <PackageReference Include="FluentValidation" Version="11.*" />
    <PackageReference Include="FluentValidation.DependencyInjectionExtensions" Version="11.*" />
    <PackageReference Include="Serilog.AspNetCore" Version="8.*" />
    <PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="9.*" />
    <PackageReference Include="Swashbuckle.AspNetCore" Version="6.*" />
    <PackageReference Include="Microsoft.AspNetCore.Diagnostics.EntityFrameworkCore" Version="9.*" />
</ItemGroup>
```

## Best Practices

- ✅ Use `TypedResults` instead of `Results` for strongly-typed responses
- ✅ Use `Results<T1, T2>` to document multiple response types
- ✅ Group related endpoints with `MapGroup()`
- ✅ Use endpoint filters for cross-cutting concerns
- ✅ Apply `[AsParameters]` for complex query parameters
- ✅ Use records for request/response DTOs
- ✅ Leverage native OpenAPI support in .NET 9
- ✅ Implement proper HTTP semantics (201 for creation, 204 for updates)
- ✅ Use ProblemDetails for error responses
- ✅ Add meaningful operation names with `WithName()`
