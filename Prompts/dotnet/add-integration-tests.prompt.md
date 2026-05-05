---
agent: 'Unit-Testing-Generator'
description: 'Generate integration tests for ASP.NET Core APIs using WebApplicationFactory'
# Note: Uses unit-testing-generator.agent if available in .github/agents/, otherwise falls back to default Copilot agent
---

# Add Integration Tests

## Task

Generate integration tests for ASP.NET Core APIs using `WebApplicationFactory`, Entity Framework Core with in-memory database or test containers, and proper test isolation.

## Configuration

| Setting | Options | Default |
|---------|---------|---------|
| Test Framework | xUnit, NUnit | xUnit |
| Database Strategy | InMemory, SQLite, TestContainers | SQLite |
| Authentication | Mock, Real | Mock |
| Use Respawn | Yes, No | Yes (for database reset) |

## Test Project Structure

```
tests/
└── ${PROJECT_NAME}.IntegrationTests/
    ├── ${PROJECT_NAME}.IntegrationTests.csproj
    ├── GlobalUsings.cs
    ├── Fixtures/
    │   ├── CustomWebApplicationFactory.cs
    │   ├── DatabaseFixture.cs
    │   └── IntegrationTestBase.cs
    ├── Extensions/
    │   └── HttpClientExtensions.cs
    ├── Endpoints/
    │   ├── ProductsEndpointsTests.cs
    │   └── OrdersEndpointsTests.cs
    └── appsettings.Testing.json
```

## Core Components

### 1. Custom WebApplicationFactory

```csharp
public class CustomWebApplicationFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly string _connectionString = $"Data Source={Guid.NewGuid()}.db";
    private Respawner? _respawner;
    private SqliteConnection? _connection;
    
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("Testing");
        
        builder.ConfigureTestServices(services =>
        {
            // Remove existing DbContext registration
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<AppDbContext>));
            
            if (descriptor != null)
            {
                services.Remove(descriptor);
            }
            
            // Add SQLite for testing (keeps connection open)
            _connection = new SqliteConnection(_connectionString);
            _connection.Open();
            
            services.AddDbContext<AppDbContext>(options =>
            {
                options.UseSqlite(_connection);
            });
            
            // Mock external services
            services.RemoveAll<IEmailService>();
            services.AddScoped<IEmailService, MockEmailService>();
            
            // Mock authentication
            services.AddAuthentication("Test")
                .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>("Test", null);
            
            // Build service provider and ensure database is created
            var sp = services.BuildServiceProvider();
            using var scope = sp.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
            db.Database.EnsureCreated();
        });
    }
    
    public async Task InitializeAsync()
    {
        // Initialize Respawner for database cleanup between tests
        using var conn = new SqliteConnection(_connectionString);
        await conn.OpenAsync();
        
        _respawner = await Respawner.CreateAsync(conn, new RespawnerOptions
        {
            DbAdapter = DbAdapter.Sqlite,
            TablesToIgnore = ["__EFMigrationsHistory"]
        });
    }
    
    public async Task ResetDatabaseAsync()
    {
        if (_respawner is not null && _connection is not null)
        {
            await _respawner.ResetAsync(_connection);
        }
    }
    
    async Task IAsyncLifetime.DisposeAsync()
    {
        if (_connection is not null)
        {
            await _connection.CloseAsync();
            await _connection.DisposeAsync();
        }
    }
}
```

### 2. Test Authentication Handler

```csharp
public class TestAuthHandler : AuthenticationHandler<AuthenticationSchemeOptions>
{
    public const string TestUserId = "test-user-id";
    public const string TestUserName = "testuser@example.com";
    
    public TestAuthHandler(
        IOptionsMonitor<AuthenticationSchemeOptions> options,
        ILoggerFactory logger,
        UrlEncoder encoder)
        : base(options, logger, encoder)
    {
    }
    
    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        // Check for test authentication header
        if (!Request.Headers.TryGetValue("X-Test-Auth", out var authHeader))
        {
            return Task.FromResult(AuthenticateResult.NoResult());
        }
        
        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, TestUserId),
            new Claim(ClaimTypes.Name, TestUserName),
            new Claim(ClaimTypes.Email, TestUserName),
            new Claim(ClaimTypes.Role, "User")
        };
        
        // Add custom claims from header if present
        if (authHeader.ToString() == "Admin")
        {
            claims = [..claims, new Claim(ClaimTypes.Role, "Admin")];
        }
        
        var identity = new ClaimsIdentity(claims, "Test");
        var principal = new ClaimsPrincipal(identity);
        var ticket = new AuthenticationTicket(principal, "Test");
        
        return Task.FromResult(AuthenticateResult.Success(ticket));
    }
}
```

### 3. Integration Test Base Class

```csharp
public abstract class IntegrationTestBase : IClassFixture<CustomWebApplicationFactory>, IAsyncLifetime
{
    protected readonly CustomWebApplicationFactory Factory;
    protected readonly HttpClient Client;
    protected readonly IServiceScope Scope;
    protected readonly AppDbContext DbContext;
    
    protected IntegrationTestBase(CustomWebApplicationFactory factory)
    {
        Factory = factory;
        Client = factory.CreateClient(new WebApplicationFactoryClientOptions
        {
            AllowAutoRedirect = false
        });
        
        // Default to authenticated requests
        Client.DefaultRequestHeaders.Add("X-Test-Auth", "User");
        
        Scope = factory.Services.CreateScope();
        DbContext = Scope.ServiceProvider.GetRequiredService<AppDbContext>();
    }
    
    public Task InitializeAsync() => Task.CompletedTask;
    
    public async Task DisposeAsync()
    {
        await Factory.ResetDatabaseAsync();
        Scope.Dispose();
    }
    
    protected async Task<T?> GetAsync<T>(string url) where T : class
    {
        var response = await Client.GetAsync(url);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<T>();
    }
    
    protected async Task<HttpResponseMessage> PostAsync<T>(string url, T content)
    {
        return await Client.PostAsJsonAsync(url, content);
    }
    
    protected async Task SeedDataAsync(params object[] entities)
    {
        foreach (var entity in entities)
        {
            DbContext.Add(entity);
        }
        await DbContext.SaveChangesAsync();
    }
    
    protected void AuthenticateAs(string role = "User")
    {
        Client.DefaultRequestHeaders.Remove("X-Test-Auth");
        Client.DefaultRequestHeaders.Add("X-Test-Auth", role);
    }
    
    protected void Unauthenticate()
    {
        Client.DefaultRequestHeaders.Remove("X-Test-Auth");
    }
}
```

### 4. Endpoint Tests

```csharp
public class ProductsEndpointsTests : IntegrationTestBase
{
    public ProductsEndpointsTests(CustomWebApplicationFactory factory) : base(factory) { }
    
    #region GET /api/products
    
    [Fact]
    public async Task GetProducts_ReturnsEmptyList_WhenNoProducts()
    {
        // Act
        var response = await Client.GetAsync("/api/products");
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        
        var products = await response.Content.ReadFromJsonAsync<PaginatedList<ProductResponse>>();
        products.Should().NotBeNull();
        products!.Items.Should().BeEmpty();
        products.TotalCount.Should().Be(0);
    }
    
    [Fact]
    public async Task GetProducts_ReturnsProducts_WhenProductsExist()
    {
        // Arrange
        var category = new Category { Id = Guid.NewGuid(), Name = "Electronics" };
        var products = new[]
        {
            new Product { Id = Guid.NewGuid(), Name = "Product 1", Price = 10m, Category = category },
            new Product { Id = Guid.NewGuid(), Name = "Product 2", Price = 20m, Category = category }
        };
        await SeedDataAsync(category);
        await SeedDataAsync(products);
        
        // Act
        var response = await Client.GetAsync("/api/products");
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        
        var result = await response.Content.ReadFromJsonAsync<PaginatedList<ProductResponse>>();
        result.Should().NotBeNull();
        result!.Items.Should().HaveCount(2);
        result.TotalCount.Should().Be(2);
    }
    
    [Theory]
    [InlineData(1, 10)]
    [InlineData(2, 5)]
    public async Task GetProducts_SupportsPagination(int page, int pageSize)
    {
        // Arrange
        var category = new Category { Id = Guid.NewGuid(), Name = "Test" };
        var products = Enumerable.Range(1, 20)
            .Select(i => new Product
            {
                Id = Guid.NewGuid(),
                Name = $"Product {i}",
                Price = i * 10m,
                Category = category
            })
            .ToArray();
        
        await SeedDataAsync(category);
        await SeedDataAsync(products);
        
        // Act
        var response = await Client.GetAsync($"/api/products?page={page}&pageSize={pageSize}");
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        
        var result = await response.Content.ReadFromJsonAsync<PaginatedList<ProductResponse>>();
        result!.Items.Should().HaveCount(pageSize);
        result.Page.Should().Be(page);
        result.TotalCount.Should().Be(20);
    }
    
    #endregion
    
    #region GET /api/products/{id}
    
    [Fact]
    public async Task GetProductById_ReturnsProduct_WhenExists()
    {
        // Arrange
        var category = new Category { Id = Guid.NewGuid(), Name = "Electronics" };
        var product = new Product
        {
            Id = Guid.NewGuid(),
            Name = "Test Product",
            Description = "Test Description",
            Price = 99.99m,
            Category = category
        };
        await SeedDataAsync(category, product);
        
        // Act
        var response = await Client.GetAsync($"/api/products/{product.Id}");
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        
        var result = await response.Content.ReadFromJsonAsync<ProductResponse>();
        result.Should().NotBeNull();
        result!.Id.Should().Be(product.Id);
        result.Name.Should().Be(product.Name);
        result.CategoryName.Should().Be(category.Name);
    }
    
    [Fact]
    public async Task GetProductById_ReturnsNotFound_WhenNotExists()
    {
        // Arrange
        var nonExistentId = Guid.NewGuid();
        
        // Act
        var response = await Client.GetAsync($"/api/products/{nonExistentId}");
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
        
        var problem = await response.Content.ReadFromJsonAsync<ProblemDetails>();
        problem.Should().NotBeNull();
        problem!.Title.Should().Be("Product not found");
    }
    
    #endregion
    
    #region POST /api/products
    
    [Fact]
    public async Task CreateProduct_ReturnsCreated_WithValidData()
    {
        // Arrange
        var category = new Category { Id = Guid.NewGuid(), Name = "Electronics" };
        await SeedDataAsync(category);
        
        var request = new CreateProductRequest("New Product", "Description", 49.99m, category.Id);
        
        // Act
        var response = await Client.PostAsJsonAsync("/api/products", request);
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();
        
        var result = await response.Content.ReadFromJsonAsync<ProductResponse>();
        result.Should().NotBeNull();
        result!.Name.Should().Be(request.Name);
        result.Price.Should().Be(request.Price);
        
        // Verify in database
        var dbProduct = await DbContext.Products.FindAsync(result.Id);
        dbProduct.Should().NotBeNull();
    }
    
    [Fact]
    public async Task CreateProduct_ReturnsBadRequest_WithInvalidData()
    {
        // Arrange
        var request = new CreateProductRequest("", "", -10m, Guid.Empty);
        
        // Act
        var response = await Client.PostAsJsonAsync("/api/products", request);
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
        
        var problem = await response.Content.ReadFromJsonAsync<ValidationProblemDetails>();
        problem.Should().NotBeNull();
        problem!.Errors.Should().ContainKey("Name");
        problem.Errors.Should().ContainKey("Price");
    }
    
    [Fact]
    public async Task CreateProduct_ReturnsUnauthorized_WhenNotAuthenticated()
    {
        // Arrange
        Unauthenticate();
        var request = new CreateProductRequest("Product", "Description", 10m, Guid.NewGuid());
        
        // Act
        var response = await Client.PostAsJsonAsync("/api/products", request);
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    }
    
    #endregion
    
    #region PUT /api/products/{id}
    
    [Fact]
    public async Task UpdateProduct_ReturnsNoContent_WithValidData()
    {
        // Arrange
        var category = new Category { Id = Guid.NewGuid(), Name = "Electronics" };
        var product = new Product
        {
            Id = Guid.NewGuid(),
            Name = "Original Name",
            Price = 50m,
            Category = category
        };
        await SeedDataAsync(category, product);
        
        var request = new UpdateProductRequest("Updated Name", "Updated Description", 75m);
        
        // Act
        var response = await Client.PutAsJsonAsync($"/api/products/{product.Id}", request);
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.NoContent);
        
        // Verify in database
        await DbContext.Entry(product).ReloadAsync();
        product.Name.Should().Be(request.Name);
        product.Price.Should().Be(request.Price);
    }
    
    #endregion
    
    #region DELETE /api/products/{id}
    
    [Fact]
    public async Task DeleteProduct_ReturnsNoContent_WhenExists()
    {
        // Arrange
        var category = new Category { Id = Guid.NewGuid(), Name = "Electronics" };
        var product = new Product { Id = Guid.NewGuid(), Name = "To Delete", Category = category };
        await SeedDataAsync(category, product);
        
        // Act
        var response = await Client.DeleteAsync($"/api/products/{product.Id}");
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.NoContent);
        
        // Verify deleted
        var dbProduct = await DbContext.Products.FindAsync(product.Id);
        dbProduct.Should().BeNull();
    }
    
    [Fact]
    public async Task DeleteProduct_ReturnsForbidden_WhenNotAdmin()
    {
        // Arrange
        AuthenticateAs("User"); // Not admin
        var product = new Product { Id = Guid.NewGuid(), Name = "Protected" };
        await SeedDataAsync(product);
        
        // Act
        var response = await Client.DeleteAsync($"/api/products/{product.Id}");
        
        // Assert (if delete requires admin)
        response.StatusCode.Should().Be(HttpStatusCode.Forbidden);
    }
    
    #endregion
}
```

### 5. HTTP Client Extensions

```csharp
public static class HttpClientExtensions
{
    public static async Task<T?> GetFromJsonAsync<T>(
        this HttpClient client,
        string url,
        HttpStatusCode expectedStatus = HttpStatusCode.OK)
    {
        var response = await client.GetAsync(url);
        response.StatusCode.Should().Be(expectedStatus);
        return await response.Content.ReadFromJsonAsync<T>();
    }
    
    public static async Task<(HttpStatusCode Status, T? Content)> PostAndReadAsync<T>(
        this HttpClient client,
        string url,
        object content)
    {
        var response = await client.PostAsJsonAsync(url, content);
        var result = await response.Content.ReadFromJsonAsync<T>();
        return (response.StatusCode, result);
    }
}
```

## Required NuGet Packages

```xml
<ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="9.*" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="9.*" />
    <PackageReference Include="xunit" Version="2.*" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.*" />
    <PackageReference Include="FluentAssertions" Version="6.*" />
    <PackageReference Include="Respawn" Version="6.*" />
    <PackageReference Include="Testcontainers" Version="3.*" /> <!-- Optional -->
</ItemGroup>
```

## Best Practices

- ✅ Use `WebApplicationFactory<TProgram>` for real HTTP pipeline testing
- ✅ Reset database between tests using Respawn
- ✅ Mock external services (email, payment, etc.)
- ✅ Test complete request/response cycle
- ✅ Test authentication and authorization
- ✅ Use SQLite or TestContainers for database isolation
- ✅ Create helper methods for common operations
- ✅ Test both success and error scenarios
- ✅ Verify side effects (database changes, events)
- ❌ Don't share state between tests
- ❌ Don't rely on test execution order
