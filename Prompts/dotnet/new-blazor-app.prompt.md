---
agent: 'Feature-Scaffolder'
description: 'Scaffold a new Blazor application (Server, WebAssembly, or Hybrid)'
# Note: Uses Feature-Scaffolder agent if available in .github/agents/, otherwise falls back to default Copilot agent
---

# Scaffold New Blazor Application

## Task

Create a complete Blazor application with proper project structure, component organization, state management, and best practices for the chosen hosting model.

## Configuration

| Setting | Options | Default |
|---------|---------|---------|
| Project Name | `${PROJECT_NAME}` | MyBlazorApp |
| .NET Version | 8.0, 9.0 | 9.0 |
| Hosting Model | Server, WebAssembly, Auto (Hybrid) | Auto |
| CSS Framework | Bootstrap, Tailwind, None | Bootstrap |
| State Management | Simple Services, Fluxor, None | Simple Services |
| Authentication | None, Individual Accounts, Azure AD, Azure AD B2C | None |
| Include PWA Support | Yes, No | No |
| Include Docker | Yes, No | Yes |

## Solution Structure

### Blazor Web App (Auto/Hybrid - .NET 8+)

```
${PROJECT_NAME}/
├── ${PROJECT_NAME}/                          # Server project (startup)
│   ├── Components/
│   │   ├── Layout/
│   │   │   ├── MainLayout.razor
│   │   │   ├── NavMenu.razor
│   │   │   └── NavMenu.razor.css
│   │   ├── Pages/
│   │   │   ├── Home.razor
│   │   │   ├── Counter.razor
│   │   │   └── Error.razor
│   │   ├── _Imports.razor
│   │   ├── App.razor
│   │   └── Routes.razor
│   ├── wwwroot/
│   │   ├── css/
│   │   └── favicon.ico
│   ├── Program.cs
│   └── appsettings.json
│
├── ${PROJECT_NAME}.Client/                   # WebAssembly project
│   ├── Pages/
│   │   └── Counter.razor                     # Interactive WebAssembly component
│   ├── Services/
│   ├── _Imports.razor
│   └── Program.cs
│
├── ${PROJECT_NAME}.Shared/                   # Shared library
│   ├── Models/
│   ├── DTOs/
│   └── Services/
│
└── tests/
    └── ${PROJECT_NAME}.Tests/
```

### Blazor Server Only

```
${PROJECT_NAME}/
├── Components/
│   ├── Layout/
│   ├── Pages/
│   └── Shared/
├── Data/
├── Services/
├── wwwroot/
├── Program.cs
└── appsettings.json
```

### Blazor WebAssembly Standalone

```
${PROJECT_NAME}/
├── ${PROJECT_NAME}/                          # WASM Client
│   ├── Layout/
│   ├── Pages/
│   ├── Services/
│   ├── wwwroot/
│   └── Program.cs
│
└── ${PROJECT_NAME}.Api/                      # Optional backend API
    ├── Controllers/
    └── Program.cs
```

## Core Components

### 1. App Component (App.razor)

```razor
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <base href="/" />
    <link rel="stylesheet" href="bootstrap/bootstrap.min.css" />
    <link rel="stylesheet" href="app.css" />
    <link rel="stylesheet" href="${PROJECT_NAME}.styles.css" />
    <link rel="icon" type="image/png" href="favicon.png" />
    <HeadOutlet @rendermode="InteractiveAuto" />
</head>
<body>
    <Routes @rendermode="InteractiveAuto" />
    <script src="_framework/blazor.web.js"></script>
</body>
</html>
```

### 2. Main Layout

```razor
@inherits LayoutComponentBase

<div class="page">
    <div class="sidebar">
        <NavMenu />
    </div>

    <main>
        <div class="top-row px-4">
            <AuthorizeView>
                <Authorized>
                    <span class="me-3">Hello, @context.User.Identity?.Name!</span>
                    <a href="authentication/logout">Log out</a>
                </Authorized>
                <NotAuthorized>
                    <a href="authentication/login">Log in</a>
                </NotAuthorized>
            </AuthorizeView>
        </div>

        <article class="content px-4">
            @Body
        </article>
    </main>
</div>

<div id="blazor-error-ui" data-nosnippet>
    An unhandled error has occurred.
    <a href="." class="reload">Reload</a>
    <span class="dismiss">🗙</span>
</div>
```

### 3. Base Component with Common Patterns

```csharp
// BaseComponent.cs - Inherit for common functionality
public abstract class BaseComponent : ComponentBase, IDisposable
{
    private CancellationTokenSource? _cts;
    
    protected CancellationToken CancellationToken => (_cts ??= new()).Token;
    
    [Inject] protected ILogger<BaseComponent> Logger { get; set; } = default!;
    [Inject] protected NavigationManager Navigation { get; set; } = default!;
    
    protected bool IsLoading { get; set; }
    protected string? ErrorMessage { get; set; }
    
    protected async Task ExecuteAsync(Func<Task> action, string? loadingMessage = null)
    {
        try
        {
            IsLoading = true;
            ErrorMessage = null;
            StateHasChanged();
            
            await action();
        }
        catch (Exception ex)
        {
            Logger.LogError(ex, "Error in component");
            ErrorMessage = "An error occurred. Please try again.";
        }
        finally
        {
            IsLoading = false;
            StateHasChanged();
        }
    }
    
    public virtual void Dispose()
    {
        _cts?.Cancel();
        _cts?.Dispose();
        GC.SuppressFinalize(this);
    }
}
```

### 4. Sample Interactive Component

```razor
@page "/products"
@attribute [StreamRendering]
@inject IProductService ProductService
@inject ILogger<Products> Logger

<PageTitle>Products</PageTitle>

<h1>Products</h1>

@if (_isLoading)
{
    <div class="d-flex justify-content-center">
        <div class="spinner-border" role="status">
            <span class="visually-hidden">Loading...</span>
        </div>
    </div>
}
else if (_error is not null)
{
    <div class="alert alert-danger" role="alert">
        @_error
    </div>
}
else if (_products is null || !_products.Any())
{
    <p>No products found.</p>
}
else
{
    <div class="row row-cols-1 row-cols-md-3 g-4">
        @foreach (var product in _products)
        {
            <div class="col">
                <ProductCard Product="product" OnDelete="HandleDelete" />
            </div>
        }
    </div>
}

@code {
    private List<ProductDto>? _products;
    private bool _isLoading = true;
    private string? _error;

    protected override async Task OnInitializedAsync()
    {
        await LoadProductsAsync();
    }

    private async Task LoadProductsAsync()
    {
        try
        {
            _isLoading = true;
            _error = null;
            _products = await ProductService.GetAllAsync();
        }
        catch (Exception ex)
        {
            Logger.LogError(ex, "Failed to load products");
            _error = "Failed to load products. Please try again.";
        }
        finally
        {
            _isLoading = false;
        }
    }

    private async Task HandleDelete(Guid id)
    {
        var result = await ProductService.DeleteAsync(id);
        if (result.IsSuccess)
        {
            _products?.RemoveAll(p => p.Id == id);
        }
    }
}
```

### 5. Reusable Component with Parameters

```razor
@* ProductCard.razor *@
<div class="card h-100">
    @if (!string.IsNullOrEmpty(Product.ImageUrl))
    {
        <img src="@Product.ImageUrl" class="card-img-top" alt="@Product.Name">
    }
    <div class="card-body">
        <h5 class="card-title">@Product.Name</h5>
        <p class="card-text">@Product.Description</p>
        <p class="card-text">
            <strong>@Product.Price.ToString("C")</strong>
        </p>
    </div>
    <div class="card-footer">
        <div class="btn-group w-100">
            <a href="/products/@Product.Id" class="btn btn-primary">View</a>
            <a href="/products/@Product.Id/edit" class="btn btn-outline-secondary">Edit</a>
            <button class="btn btn-outline-danger" @onclick="HandleDeleteClick">
                Delete
            </button>
        </div>
    </div>
</div>

@code {
    [Parameter, EditorRequired]
    public ProductDto Product { get; set; } = default!;
    
    [Parameter]
    public EventCallback<Guid> OnDelete { get; set; }
    
    private async Task HandleDeleteClick()
    {
        if (await OnDelete.HasDelegate)
        {
            await OnDelete.InvokeAsync(Product.Id);
        }
    }
}
```

### 6. State Management Service

```csharp
// Simple state management with events
public class AppState
{
    private readonly List<ProductDto> _cart = [];
    
    public IReadOnlyList<ProductDto> CartItems => _cart.AsReadOnly();
    public int CartCount => _cart.Count;
    public decimal CartTotal => _cart.Sum(p => p.Price);
    
    public event Action? OnChange;
    
    public void AddToCart(ProductDto product)
    {
        _cart.Add(product);
        NotifyStateChanged();
    }
    
    public void RemoveFromCart(Guid productId)
    {
        var item = _cart.FirstOrDefault(p => p.Id == productId);
        if (item is not null)
        {
            _cart.Remove(item);
            NotifyStateChanged();
        }
    }
    
    public void ClearCart()
    {
        _cart.Clear();
        NotifyStateChanged();
    }
    
    private void NotifyStateChanged() => OnChange?.Invoke();
}

// Usage in component
@inject AppState AppState
@implements IDisposable

<span class="badge bg-primary">@AppState.CartCount</span>

@code {
    protected override void OnInitialized()
    {
        AppState.OnChange += StateHasChanged;
    }
    
    public void Dispose()
    {
        AppState.OnChange -= StateHasChanged;
    }
}
```

### 7. HTTP Client Service

```csharp
public interface IProductService
{
    Task<List<ProductDto>> GetAllAsync(CancellationToken ct = default);
    Task<ProductDto?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<Result<ProductDto>> CreateAsync(CreateProductRequest request, CancellationToken ct = default);
    Task<Result> UpdateAsync(Guid id, UpdateProductRequest request, CancellationToken ct = default);
    Task<Result> DeleteAsync(Guid id, CancellationToken ct = default);
}

public class ProductService(HttpClient http, ILogger<ProductService> logger) : IProductService
{
    public async Task<List<ProductDto>> GetAllAsync(CancellationToken ct = default)
    {
        try
        {
            var products = await http.GetFromJsonAsync<List<ProductDto>>("api/products", ct);
            return products ?? [];
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Failed to fetch products");
            throw;
        }
    }
    
    public async Task<ProductDto?> GetByIdAsync(Guid id, CancellationToken ct = default)
    {
        return await http.GetFromJsonAsync<ProductDto>($"api/products/{id}", ct);
    }
    
    public async Task<Result<ProductDto>> CreateAsync(CreateProductRequest request, CancellationToken ct = default)
    {
        var response = await http.PostAsJsonAsync("api/products", request, ct);
        
        if (response.IsSuccessStatusCode)
        {
            var product = await response.Content.ReadFromJsonAsync<ProductDto>(ct);
            return Result<ProductDto>.Success(product!);
        }
        
        var error = await response.Content.ReadFromJsonAsync<ProblemDetails>(ct);
        return Result<ProductDto>.Failure(error?.Detail ?? "Failed to create product");
    }
}
```

### 8. Program.cs Setup

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add services
builder.Services.AddRazorComponents()
    .AddInteractiveServerComponents()
    .AddInteractiveWebAssemblyComponents();

// Authentication (if configured)
builder.Services.AddAuthentication(options =>
{
    options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
});

// HTTP Client for API calls
builder.Services.AddHttpClient<IProductService, ProductService>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration["ApiBaseUrl"] ?? "https://localhost:5001");
});

// State management
builder.Services.AddScoped<AppState>();

// Logging
builder.Services.AddSerilog();

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseAntiforgery();

app.MapRazorComponents<App>()
    .AddInteractiveServerRenderMode()
    .AddInteractiveWebAssemblyRenderMode()
    .AddAdditionalAssemblies(typeof(Client._Imports).Assembly);

app.Run();
```

## Best Practices

### Component Design
- ✅ Use `@attribute [StreamRendering]` for async data loading
- ✅ Use `[EditorRequired]` for required parameters
- ✅ Implement `IDisposable` when subscribing to events
- ✅ Use `EventCallback<T>` for parent-child communication
- ✅ Keep components small and focused
- ✅ Extract common UI into reusable components

### Performance
- ✅ Use `@key` directive for lists
- ✅ Implement virtualization for large lists with `<Virtualize>`
- ✅ Use `StateHasChanged()` sparingly
- ✅ Consider component isolation for CSS
- ✅ Lazy load assemblies when appropriate

### Render Modes (.NET 8+)
- ✅ Use `InteractiveServer` for real-time updates with SignalR
- ✅ Use `InteractiveWebAssembly` for offline scenarios
- ✅ Use `InteractiveAuto` for best of both worlds
- ✅ Use static SSR for SEO-critical pages

### Error Handling
- ✅ Use `<ErrorBoundary>` components
- ✅ Implement global error handling
- ✅ Log errors with structured logging
- ✅ Show user-friendly error messages
