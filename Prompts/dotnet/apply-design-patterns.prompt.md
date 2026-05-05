---
agent: 'Code-Reviewer'
description: 'Apply appropriate design patterns to solve common software design problems'
# Note: Uses Code-Reviewer agent if available in .github/agents/, otherwise falls back to default Copilot agent
---

# Apply Design Patterns

## Task

Analyze the code or problem description and recommend appropriate design patterns. Implement the patterns using modern C# features and best practices.

## Pattern Categories

### Creational Patterns

#### Factory Method

**Use when:** You need to create objects without specifying exact classes.

```csharp
// Abstract factory for notification channels
public interface INotificationChannel
{
    Task SendAsync(string recipient, string message, CancellationToken ct = default);
}

public interface INotificationChannelFactory
{
    INotificationChannel Create(NotificationType type);
}

public class NotificationChannelFactory(IServiceProvider serviceProvider) : INotificationChannelFactory
{
    public INotificationChannel Create(NotificationType type) => type switch
    {
        NotificationType.Email => serviceProvider.GetRequiredService<EmailChannel>(),
        NotificationType.Sms => serviceProvider.GetRequiredService<SmsChannel>(),
        NotificationType.Push => serviceProvider.GetRequiredService<PushNotificationChannel>(),
        NotificationType.Slack => serviceProvider.GetRequiredService<SlackChannel>(),
        _ => throw new ArgumentOutOfRangeException(nameof(type))
    };
}

// Usage
public class NotificationService(INotificationChannelFactory factory)
{
    public async Task NotifyAsync(User user, string message, NotificationType type, CancellationToken ct)
    {
        var channel = factory.Create(type);
        await channel.SendAsync(user.GetContactFor(type), message, ct);
    }
}
```

#### Builder Pattern

**Use when:** You need to construct complex objects step by step.

```csharp
public class EmailBuilder
{
    private readonly Email _email = new();
    
    public EmailBuilder From(string address)
    {
        _email.From = address;
        return this;
    }
    
    public EmailBuilder To(string address)
    {
        _email.To.Add(address);
        return this;
    }
    
    public EmailBuilder To(IEnumerable<string> addresses)
    {
        _email.To.AddRange(addresses);
        return this;
    }
    
    public EmailBuilder Cc(string address)
    {
        _email.Cc.Add(address);
        return this;
    }
    
    public EmailBuilder Subject(string subject)
    {
        _email.Subject = subject;
        return this;
    }
    
    public EmailBuilder Body(string body, bool isHtml = false)
    {
        _email.Body = body;
        _email.IsHtml = isHtml;
        return this;
    }
    
    public EmailBuilder WithAttachment(string path, string? name = null)
    {
        _email.Attachments.Add(new Attachment(path, name ?? Path.GetFileName(path)));
        return this;
    }
    
    public EmailBuilder WithPriority(EmailPriority priority)
    {
        _email.Priority = priority;
        return this;
    }
    
    public Email Build()
    {
        Validate();
        return _email;
    }
    
    private void Validate()
    {
        if (string.IsNullOrEmpty(_email.From))
            throw new InvalidOperationException("From address is required");
        if (!_email.To.Any())
            throw new InvalidOperationException("At least one recipient is required");
    }
}

// Usage
var email = new EmailBuilder()
    .From("noreply@company.com")
    .To("customer@example.com")
    .Cc("support@company.com")
    .Subject("Your Order Confirmation")
    .Body("<h1>Thank you!</h1>", isHtml: true)
    .WithAttachment("invoice.pdf")
    .WithPriority(EmailPriority.High)
    .Build();
```

---

### Structural Patterns

#### Decorator Pattern

**Use when:** You need to add behavior to objects dynamically.

```csharp
// Base interface
public interface IOrderProcessor
{
    Task<OrderResult> ProcessAsync(Order order, CancellationToken ct = default);
}

// Core implementation
public class OrderProcessor(IOrderRepository repository) : IOrderProcessor
{
    public async Task<OrderResult> ProcessAsync(Order order, CancellationToken ct)
    {
        await repository.SaveAsync(order, ct);
        return new OrderResult(order.Id, OrderStatus.Processed);
    }
}

// Decorators
public class LoggingOrderProcessor(IOrderProcessor inner, ILogger<LoggingOrderProcessor> logger) : IOrderProcessor
{
    public async Task<OrderResult> ProcessAsync(Order order, CancellationToken ct)
    {
        logger.LogInformation("Processing order {OrderId}", order.Id);
        var stopwatch = Stopwatch.StartNew();
        
        var result = await inner.ProcessAsync(order, ct);
        
        logger.LogInformation("Order {OrderId} processed in {ElapsedMs}ms with status {Status}",
            order.Id, stopwatch.ElapsedMilliseconds, result.Status);
        
        return result;
    }
}

public class ValidationOrderProcessor(IOrderProcessor inner, IValidator<Order> validator) : IOrderProcessor
{
    public async Task<OrderResult> ProcessAsync(Order order, CancellationToken ct)
    {
        var validationResult = await validator.ValidateAsync(order, ct);
        if (!validationResult.IsValid)
        {
            return new OrderResult(order.Id, OrderStatus.Failed, validationResult.Errors);
        }
        
        return await inner.ProcessAsync(order, ct);
    }
}

public class RetryOrderProcessor(IOrderProcessor inner, ILogger<RetryOrderProcessor> logger) : IOrderProcessor
{
    private const int MaxRetries = 3;
    
    public async Task<OrderResult> ProcessAsync(Order order, CancellationToken ct)
    {
        for (int attempt = 1; attempt <= MaxRetries; attempt++)
        {
            try
            {
                return await inner.ProcessAsync(order, ct);
            }
            catch (Exception ex) when (attempt < MaxRetries)
            {
                logger.LogWarning(ex, "Order processing attempt {Attempt} failed, retrying...", attempt);
                await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, attempt)), ct);
            }
        }
        
        return await inner.ProcessAsync(order, ct); // Final attempt, let it throw
    }
}

// DI Registration (order matters - outermost first)
services.AddScoped<OrderProcessor>();
services.AddScoped<IOrderProcessor>(sp =>
    new LoggingOrderProcessor(
        new RetryOrderProcessor(
            new ValidationOrderProcessor(
                sp.GetRequiredService<OrderProcessor>(),
                sp.GetRequiredService<IValidator<Order>>()),
            sp.GetRequiredService<ILogger<RetryOrderProcessor>>()),
        sp.GetRequiredService<ILogger<LoggingOrderProcessor>>()));
```

#### Adapter Pattern

**Use when:** You need to make incompatible interfaces work together.

```csharp
// External payment gateway SDK (can't modify)
public class LegacyPaymentGateway
{
    public PaymentResponse MakePayment(string cardNumber, string expiry, decimal amount, string currency)
    {
        // Legacy implementation
        return new PaymentResponse { Success = true, TransactionId = Guid.NewGuid().ToString() };
    }
}

// Your application's interface
public interface IPaymentService
{
    Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request, CancellationToken ct = default);
}

public record PaymentRequest(
    string CardNumber,
    string ExpiryMonth,
    string ExpiryYear,
    string Cvv,
    decimal Amount,
    string Currency);

public record PaymentResult(bool IsSuccessful, string TransactionId, string? ErrorMessage = null);

// Adapter
public class LegacyPaymentAdapter(LegacyPaymentGateway gateway, ILogger<LegacyPaymentAdapter> logger) : IPaymentService
{
    public Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request, CancellationToken ct)
    {
        try
        {
            // Adapt the interface
            var expiry = $"{request.ExpiryMonth}/{request.ExpiryYear}";
            var response = gateway.MakePayment(request.CardNumber, expiry, request.Amount, request.Currency);
            
            return Task.FromResult(new PaymentResult(
                response.Success,
                response.TransactionId,
                response.ErrorMessage));
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Payment processing failed");
            return Task.FromResult(new PaymentResult(false, string.Empty, ex.Message));
        }
    }
}
```

---

### Behavioral Patterns

#### Strategy Pattern

**Use when:** You need to define a family of algorithms and make them interchangeable.

```csharp
public interface IPricingStrategy
{
    decimal CalculatePrice(Order order);
}

public class RegularPricing : IPricingStrategy
{
    public decimal CalculatePrice(Order order) => order.Items.Sum(i => i.Price * i.Quantity);
}

public class PremiumMemberPricing : IPricingStrategy
{
    private const decimal DiscountRate = 0.15m;
    
    public decimal CalculatePrice(Order order)
    {
        var total = order.Items.Sum(i => i.Price * i.Quantity);
        return total * (1 - DiscountRate);
    }
}

public class VolumeDiscountPricing : IPricingStrategy
{
    public decimal CalculatePrice(Order order)
    {
        var total = order.Items.Sum(i => i.Price * i.Quantity);
        var discount = order.Items.Sum(i => i.Quantity) switch
        {
            >= 100 => 0.20m,
            >= 50 => 0.15m,
            >= 20 => 0.10m,
            >= 10 => 0.05m,
            _ => 0m
        };
        return total * (1 - discount);
    }
}

// Context
public class OrderPriceCalculator(IEnumerable<IPricingStrategy> strategies)
{
    public decimal Calculate(Order order, CustomerType customerType)
    {
        var strategy = customerType switch
        {
            CustomerType.Regular => strategies.OfType<RegularPricing>().First(),
            CustomerType.Premium => strategies.OfType<PremiumMemberPricing>().First(),
            CustomerType.Wholesale => strategies.OfType<VolumeDiscountPricing>().First(),
            _ => strategies.OfType<RegularPricing>().First()
        };
        
        return strategy.CalculatePrice(order);
    }
}
```

#### Chain of Responsibility

**Use when:** You want to pass requests along a chain of handlers.

```csharp
public interface IRequestHandler<TRequest>
{
    Task<Result> HandleAsync(TRequest request, CancellationToken ct = default);
    IRequestHandler<TRequest> SetNext(IRequestHandler<TRequest> handler);
}

public abstract class BaseHandler<TRequest> : IRequestHandler<TRequest>
{
    private IRequestHandler<TRequest>? _next;
    
    public IRequestHandler<TRequest> SetNext(IRequestHandler<TRequest> handler)
    {
        _next = handler;
        return handler;
    }
    
    public async Task<Result> HandleAsync(TRequest request, CancellationToken ct)
    {
        var result = await ProcessAsync(request, ct);
        
        if (!result.IsSuccess)
            return result;
        
        if (_next is not null)
            return await _next.HandleAsync(request, ct);
        
        return Result.Success();
    }
    
    protected abstract Task<Result> ProcessAsync(TRequest request, CancellationToken ct);
}

// Concrete handlers
public class AuthenticationHandler(IAuthService authService) : BaseHandler<ApiRequest>
{
    protected override async Task<Result> ProcessAsync(ApiRequest request, CancellationToken ct)
    {
        if (!await authService.ValidateTokenAsync(request.Token, ct))
            return Result.Failure("Unauthorized");
        return Result.Success();
    }
}

public class RateLimitHandler(IRateLimiter rateLimiter) : BaseHandler<ApiRequest>
{
    protected override async Task<Result> ProcessAsync(ApiRequest request, CancellationToken ct)
    {
        if (!await rateLimiter.AllowRequestAsync(request.ClientId, ct))
            return Result.Failure("Rate limit exceeded");
        return Result.Success();
    }
}

public class ValidationHandler(IValidator<ApiRequest> validator) : BaseHandler<ApiRequest>
{
    protected override async Task<Result> ProcessAsync(ApiRequest request, CancellationToken ct)
    {
        var result = await validator.ValidateAsync(request, ct);
        if (!result.IsValid)
            return Result.Failure(result.Errors);
        return Result.Success();
    }
}

// Setup and usage
var authHandler = new AuthenticationHandler(authService);
var rateLimitHandler = new RateLimitHandler(rateLimiter);
var validationHandler = new ValidationHandler(validator);

authHandler.SetNext(rateLimitHandler).SetNext(validationHandler);

var result = await authHandler.HandleAsync(request, ct);
```

#### Observer Pattern (with Events)

**Use when:** You need to notify multiple objects about state changes.

```csharp
// Using C# events and delegates
public class Order
{
    public event EventHandler<OrderStatusChangedEventArgs>? StatusChanged;
    
    private OrderStatus _status;
    public OrderStatus Status
    {
        get => _status;
        set
        {
            if (_status != value)
            {
                var oldStatus = _status;
                _status = value;
                OnStatusChanged(new OrderStatusChangedEventArgs(Id, oldStatus, value));
            }
        }
    }
    
    public Guid Id { get; init; }
    
    protected virtual void OnStatusChanged(OrderStatusChangedEventArgs e)
    {
        StatusChanged?.Invoke(this, e);
    }
}

public record OrderStatusChangedEventArgs(Guid OrderId, OrderStatus OldStatus, OrderStatus NewStatus);

// Observers
public class OrderNotificationObserver(IEmailService emailService) : IObserver<OrderStatusChangedEventArgs>
{
    public async void OnNext(OrderStatusChangedEventArgs e)
    {
        await emailService.SendOrderStatusUpdateAsync(e.OrderId, e.NewStatus);
    }
}

// Or using MediatR notifications
public record OrderStatusChangedNotification(Guid OrderId, OrderStatus OldStatus, OrderStatus NewStatus) 
    : INotification;

public class SendEmailHandler(IEmailService emailService) : INotificationHandler<OrderStatusChangedNotification>
{
    public async Task Handle(OrderStatusChangedNotification notification, CancellationToken ct)
    {
        await emailService.SendOrderStatusUpdateAsync(notification.OrderId, notification.NewStatus, ct);
    }
}

public class UpdateDashboardHandler(IDashboardService dashboardService) 
    : INotificationHandler<OrderStatusChangedNotification>
{
    public async Task Handle(OrderStatusChangedNotification notification, CancellationToken ct)
    {
        await dashboardService.RefreshOrderStatsAsync(ct);
    }
}
```

---

## Pattern Selection Guide

| Problem | Recommended Pattern |
|---------|-------------------|
| Object creation with complex setup | Builder, Factory Method |
| Adding features to existing objects | Decorator |
| Making incompatible interfaces work | Adapter |
| Switching algorithms at runtime | Strategy |
| Processing through multiple handlers | Chain of Responsibility |
| Notifying multiple subscribers | Observer, MediatR Notifications |
| Simplifying complex subsystems | Facade |
| Managing object state transitions | State |
| Undoing operations | Command, Memento |
| Traversing collections | Iterator (use LINQ) |

## Execution Steps

1. **Identify the Problem**
   - What design challenge are you facing?
   - What flexibility do you need?

2. **Select Appropriate Pattern**
   - Review pattern selection guide
   - Consider trade-offs (complexity vs. flexibility)

3. **Implement Pattern**
   - Use modern C# features (records, primary constructors)
   - Follow DI-friendly design
   - Keep implementations testable

4. **Register with DI**
   - Configure appropriate lifetimes
   - Consider decoration libraries for decorators
