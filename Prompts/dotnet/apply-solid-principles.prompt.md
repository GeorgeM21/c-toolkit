---
agent: 'Code-Reviewer'
description: 'Refactor code to follow SOLID principles with best practices'
# Note: Uses Code-Reviewer agent if available in .github/agents/, otherwise falls back to default Copilot agent
---

# Apply SOLID Principles

## Task

Analyze the provided code and refactor it to properly follow SOLID principles. Explain each violation found and demonstrate the correct implementation.

## SOLID Principles Overview

### S - Single Responsibility Principle (SRP)

**"A class should have only one reason to change."**

#### Violation Example

```csharp
// ❌ BAD: Class has multiple responsibilities
public class UserService
{
    public void CreateUser(UserDto dto)
    {
        // Validation logic
        if (string.IsNullOrEmpty(dto.Email))
            throw new ArgumentException("Email required");
        
        // Database logic
        using var connection = new SqlConnection(_connectionString);
        connection.Execute("INSERT INTO Users...", dto);
        
        // Email logic
        var smtp = new SmtpClient("smtp.example.com");
        smtp.Send(new MailMessage("noreply@example.com", dto.Email, "Welcome!", "..."));
        
        // Logging logic
        File.AppendAllText("log.txt", $"User created: {dto.Email}");
    }
}
```

#### Correct Implementation

```csharp
// ✅ GOOD: Each class has single responsibility
public class UserService(
    IUserRepository repository,
    IUserValidator validator,
    IEmailService emailService,
    ILogger<UserService> logger)
{
    public async Task<Result<User>> CreateUserAsync(CreateUserDto dto, CancellationToken ct)
    {
        var validationResult = await validator.ValidateAsync(dto);
        if (!validationResult.IsValid)
            return Result<User>.Failure(validationResult.Errors);
        
        var user = User.Create(dto.Email, dto.Name);
        await repository.AddAsync(user, ct);
        
        logger.LogInformation("User {Email} created successfully", dto.Email);
        
        await emailService.SendWelcomeEmailAsync(user.Email, ct);
        
        return Result<User>.Success(user);
    }
}

public class UserValidator : AbstractValidator<CreateUserDto>
{
    public UserValidator(IUserRepository repository)
    {
        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress()
            .MustAsync(async (email, ct) => !await repository.ExistsAsync(email, ct))
            .WithMessage("Email already exists");
    }
}
```

---

### O - Open/Closed Principle (OCP)

**"Software entities should be open for extension, but closed for modification."**

#### Violation Example

```csharp
// ❌ BAD: Must modify class to add new discount types
public class DiscountCalculator
{
    public decimal CalculateDiscount(Order order, string discountType)
    {
        switch (discountType)
        {
            case "Percentage":
                return order.Total * 0.1m;
            case "FixedAmount":
                return 10m;
            case "BuyOneGetOne":
                return order.Items.Count > 1 ? order.Items.Min(i => i.Price) : 0;
            // Adding new type requires modifying this class
            default:
                return 0;
        }
    }
}
```

#### Correct Implementation

```csharp
// ✅ GOOD: Open for extension via new implementations
public interface IDiscountStrategy
{
    string Type { get; }
    decimal Calculate(Order order);
}

public class PercentageDiscount(decimal percentage) : IDiscountStrategy
{
    public string Type => "Percentage";
    public decimal Calculate(Order order) => order.Total * percentage;
}

public class FixedAmountDiscount(decimal amount) : IDiscountStrategy
{
    public string Type => "FixedAmount";
    public decimal Calculate(Order order) => Math.Min(amount, order.Total);
}

public class BuyOneGetOneFreeDiscount : IDiscountStrategy
{
    public string Type => "BOGO";
    public decimal Calculate(Order order) =>
        order.Items.Count > 1 ? order.Items.Min(i => i.Price) : 0;
}

// New discounts can be added without modifying existing code
public class LoyaltyDiscount(ILoyaltyService loyaltyService) : IDiscountStrategy
{
    public string Type => "Loyalty";
    public decimal Calculate(Order order) =>
        order.Total * loyaltyService.GetDiscountRate(order.CustomerId);
}

public class DiscountCalculator(IEnumerable<IDiscountStrategy> strategies)
{
    private readonly Dictionary<string, IDiscountStrategy> _strategies =
        strategies.ToDictionary(s => s.Type);
    
    public decimal CalculateDiscount(Order order, string discountType) =>
        _strategies.TryGetValue(discountType, out var strategy)
            ? strategy.Calculate(order)
            : 0;
}
```

---

### L - Liskov Substitution Principle (LSP)

**"Objects of a superclass should be replaceable with objects of its subclasses without breaking the application."**

#### Violation Example

```csharp
// ❌ BAD: Square violates Rectangle's behavior expectations
public class Rectangle
{
    public virtual int Width { get; set; }
    public virtual int Height { get; set; }
    
    public int CalculateArea() => Width * Height;
}

public class Square : Rectangle
{
    private int _side;
    
    public override int Width
    {
        get => _side;
        set => _side = value; // Also changes height - unexpected!
    }
    
    public override int Height
    {
        get => _side;
        set => _side = value; // Also changes width - unexpected!
    }
}

// This code fails with Square
void ProcessRectangle(Rectangle rect)
{
    rect.Width = 5;
    rect.Height = 10;
    Debug.Assert(rect.CalculateArea() == 50); // Fails for Square!
}
```

#### Correct Implementation

```csharp
// ✅ GOOD: Use abstraction that works for all shapes
public interface IShape
{
    int CalculateArea();
}

public class Rectangle : IShape
{
    public int Width { get; }
    public int Height { get; }
    
    public Rectangle(int width, int height)
    {
        Width = width;
        Height = height;
    }
    
    public int CalculateArea() => Width * Height;
}

public class Square : IShape
{
    public int Side { get; }
    
    public Square(int side) => Side = side;
    
    public int CalculateArea() => Side * Side;
}

// Both work correctly
void ProcessShape(IShape shape)
{
    var area = shape.CalculateArea(); // Works for all shapes
}
```

---

### I - Interface Segregation Principle (ISP)

**"Clients should not be forced to depend on interfaces they do not use."**

#### Violation Example

```csharp
// ❌ BAD: Fat interface forces unnecessary implementations
public interface IRepository<T>
{
    Task<T?> GetByIdAsync(Guid id);
    Task<IEnumerable<T>> GetAllAsync();
    Task AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(T entity);
    Task<int> CountAsync();
    Task<bool> ExistsAsync(Guid id);
    Task BulkInsertAsync(IEnumerable<T> entities);
    Task BulkDeleteAsync(IEnumerable<T> entities);
    Task<IEnumerable<T>> SearchAsync(string query);
    Task<IEnumerable<T>> GetPagedAsync(int page, int size);
}

// Read-only service forced to implement write methods
public class ProductReadService : IRepository<Product>
{
    public Task AddAsync(Product entity) => throw new NotSupportedException();
    public Task UpdateAsync(Product entity) => throw new NotSupportedException();
    public Task DeleteAsync(Product entity) => throw new NotSupportedException();
    // ... etc
}
```

#### Correct Implementation

```csharp
// ✅ GOOD: Segregated interfaces
public interface IReadRepository<T>
{
    Task<T?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<IEnumerable<T>> GetAllAsync(CancellationToken ct = default);
    Task<bool> ExistsAsync(Guid id, CancellationToken ct = default);
}

public interface IWriteRepository<T>
{
    Task AddAsync(T entity, CancellationToken ct = default);
    Task UpdateAsync(T entity, CancellationToken ct = default);
    Task DeleteAsync(T entity, CancellationToken ct = default);
}

public interface IPagedRepository<T>
{
    Task<PaginatedList<T>> GetPagedAsync(int page, int pageSize, CancellationToken ct = default);
}

public interface IBulkRepository<T>
{
    Task BulkInsertAsync(IEnumerable<T> entities, CancellationToken ct = default);
    Task BulkDeleteAsync(IEnumerable<T> entities, CancellationToken ct = default);
}

public interface ISearchableRepository<T>
{
    Task<IEnumerable<T>> SearchAsync(string query, CancellationToken ct = default);
}

// Compose interfaces as needed
public interface IProductRepository :
    IReadRepository<Product>,
    IWriteRepository<Product>,
    IPagedRepository<Product>
{
}

// Read-only implementation
public class ProductReadOnlyRepository : IReadRepository<Product>
{
    public Task<Product?> GetByIdAsync(Guid id, CancellationToken ct) { /* ... */ }
    public Task<IEnumerable<Product>> GetAllAsync(CancellationToken ct) { /* ... */ }
    public Task<bool> ExistsAsync(Guid id, CancellationToken ct) { /* ... */ }
}
```

---

### D - Dependency Inversion Principle (DIP)

**"High-level modules should not depend on low-level modules. Both should depend on abstractions."**

#### Violation Example

```csharp
// ❌ BAD: High-level OrderService depends on low-level concrete classes
public class OrderService
{
    private readonly SqlServerDatabase _database = new();
    private readonly SmtpEmailSender _emailSender = new();
    private readonly FileLogger _logger = new();
    
    public void PlaceOrder(Order order)
    {
        _database.Save(order);
        _emailSender.Send(order.CustomerEmail, "Order Confirmation", "...");
        _logger.Log($"Order {order.Id} placed");
    }
}
```

#### Correct Implementation

```csharp
// ✅ GOOD: Depend on abstractions, inject dependencies
public interface IOrderRepository
{
    Task<Order> SaveAsync(Order order, CancellationToken ct = default);
}

public interface INotificationService
{
    Task SendAsync(string recipient, string subject, string message, CancellationToken ct = default);
}

public class OrderService(
    IOrderRepository repository,
    INotificationService notificationService,
    ILogger<OrderService> logger)
{
    public async Task<Result<Order>> PlaceOrderAsync(Order order, CancellationToken ct)
    {
        var savedOrder = await repository.SaveAsync(order, ct);
        
        logger.LogInformation("Order {OrderId} placed successfully", order.Id);
        
        await notificationService.SendAsync(
            order.CustomerEmail,
            "Order Confirmation",
            $"Your order {order.Id} has been placed.",
            ct);
        
        return Result<Order>.Success(savedOrder);
    }
}

// Infrastructure implementations
public class SqlServerOrderRepository(AppDbContext context) : IOrderRepository
{
    public async Task<Order> SaveAsync(Order order, CancellationToken ct)
    {
        context.Orders.Add(order);
        await context.SaveChangesAsync(ct);
        return order;
    }
}

public class EmailNotificationService(IEmailClient client) : INotificationService
{
    public async Task SendAsync(string recipient, string subject, string message, CancellationToken ct)
    {
        await client.SendEmailAsync(recipient, subject, message, ct);
    }
}

// Registration in DI container
services.AddScoped<IOrderRepository, SqlServerOrderRepository>();
services.AddScoped<INotificationService, EmailNotificationService>();
services.AddScoped<OrderService>();
```

---

## Execution Steps

1. **Analyze the Code**
   - Identify which SOLID principles are violated
   - Note specific code smells (god classes, tight coupling, etc.)

2. **Prioritize Refactoring**
   - Start with the most impactful violations
   - Consider dependencies between changes

3. **Apply Refactoring**
   - Extract interfaces for dependencies (DIP)
   - Split large classes (SRP)
   - Create strategy/factory patterns (OCP)
   - Segregate interfaces (ISP)
   - Fix inheritance hierarchies (LSP)

4. **Verify Changes**
   - Ensure existing tests still pass
   - Add tests for new abstractions
   - Check that the code is more testable

## Common Refactoring Patterns

| Violation | Pattern to Apply |
|-----------|------------------|
| SRP - God class | Extract Class, Move Method |
| OCP - Switch statements | Strategy Pattern, Factory Pattern |
| LSP - Broken inheritance | Replace Inheritance with Composition |
| ISP - Fat interface | Interface Segregation, Role Interfaces |
| DIP - Concrete dependencies | Dependency Injection, Abstract Factory |
