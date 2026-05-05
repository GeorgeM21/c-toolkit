---
agent: 'Unit-Testing-Generator'
description: 'Generate comprehensive unit tests using xUnit, Moq, and FluentAssertions'
# Note: Uses unit-testing-generator.agent if available in .github/agents/, otherwise falls back to default Copilot agent
---

# Add Unit Tests

## Task

Generate unit tests for the specified code using xUnit, Moq for mocking, and FluentAssertions for readable assertions. Tests should cover happy paths, edge cases, and error scenarios.

## Configuration

| Setting | Options | Default |
|---------|---------|---------|
| Test Framework | xUnit, NUnit, MSTest | xUnit |
| Assertion Library | FluentAssertions, Shouldly, Native | FluentAssertions |
| Mocking Library | Moq, NSubstitute, FakeItEasy | Moq |
| Coverage Target | 70%, 80%, 90% | 80% |

## Test Project Structure

```
tests/
└── ${PROJECT_NAME}.UnitTests/
    ├── ${PROJECT_NAME}.UnitTests.csproj
    ├── GlobalUsings.cs
    ├── Fixtures/
    │   └── TestFixtures.cs
    ├── Helpers/
    │   ├── TestDataBuilder.cs
    │   └── MockExtensions.cs
    ├── Features/
    │   ├── Products/
    │   │   ├── CreateProductCommandHandlerTests.cs
    │   │   ├── ProductServiceTests.cs
    │   │   └── ProductValidatorTests.cs
    │   └── Orders/
    │       └── ...
    └── Common/
        └── ResultTests.cs
```

## Test File Template

```csharp
// GlobalUsings.cs
global using Xunit;
global using Moq;
global using FluentAssertions;
global using AutoFixture;
global using AutoFixture.AutoMoq;
```

## Test Class Examples

### 1. Service Tests

```csharp
public class ProductServiceTests
{
    private readonly Mock<IProductRepository> _repositoryMock;
    private readonly Mock<IUnitOfWork> _unitOfWorkMock;
    private readonly Mock<ILogger<ProductService>> _loggerMock;
    private readonly ProductService _sut;
    private readonly IFixture _fixture;
    
    public ProductServiceTests()
    {
        _fixture = new Fixture().Customize(new AutoMoqCustomization());
        _repositoryMock = new Mock<IProductRepository>();
        _unitOfWorkMock = new Mock<IUnitOfWork>();
        _loggerMock = new Mock<ILogger<ProductService>>();
        
        _sut = new ProductService(
            _repositoryMock.Object,
            _unitOfWorkMock.Object,
            _loggerMock.Object);
    }
    
    #region GetByIdAsync Tests
    
    [Fact]
    public async Task GetByIdAsync_WhenProductExists_ReturnsProduct()
    {
        // Arrange
        var productId = Guid.NewGuid();
        var expectedProduct = _fixture.Build<Product>()
            .With(p => p.Id, productId)
            .Create();
        
        _repositoryMock
            .Setup(r => r.GetByIdAsync(productId, It.IsAny<CancellationToken>()))
            .ReturnsAsync(expectedProduct);
        
        // Act
        var result = await _sut.GetByIdAsync(productId);
        
        // Assert
        result.Should().NotBeNull();
        result.IsSuccess.Should().BeTrue();
        result.Value.Should().BeEquivalentTo(expectedProduct, options =>
            options.ExcludingMissingMembers());
    }
    
    [Fact]
    public async Task GetByIdAsync_WhenProductNotFound_ReturnsFailure()
    {
        // Arrange
        var productId = Guid.NewGuid();
        
        _repositoryMock
            .Setup(r => r.GetByIdAsync(productId, It.IsAny<CancellationToken>()))
            .ReturnsAsync((Product?)null);
        
        // Act
        var result = await _sut.GetByIdAsync(productId);
        
        // Assert
        result.Should().NotBeNull();
        result.IsSuccess.Should().BeFalse();
        result.Error.Code.Should().Be("Product.NotFound");
    }
    
    [Fact]
    public async Task GetByIdAsync_WhenRepositoryThrows_PropagatesException()
    {
        // Arrange
        var productId = Guid.NewGuid();
        
        _repositoryMock
            .Setup(r => r.GetByIdAsync(productId, It.IsAny<CancellationToken>()))
            .ThrowsAsync(new InvalidOperationException("Database error"));
        
        // Act
        var act = () => _sut.GetByIdAsync(productId);
        
        // Assert
        await act.Should().ThrowAsync<InvalidOperationException>()
            .WithMessage("Database error");
    }
    
    #endregion
    
    #region CreateAsync Tests
    
    [Fact]
    public async Task CreateAsync_WithValidData_CreatesProductAndReturnsSuccess()
    {
        // Arrange
        var createDto = _fixture.Create<CreateProductDto>();
        
        _repositoryMock
            .Setup(r => r.AddAsync(It.IsAny<Product>(), It.IsAny<CancellationToken>()))
            .Returns(Task.CompletedTask);
        
        _unitOfWorkMock
            .Setup(u => u.SaveChangesAsync(It.IsAny<CancellationToken>()))
            .ReturnsAsync(1);
        
        // Act
        var result = await _sut.CreateAsync(createDto);
        
        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Should().NotBeNull();
        result.Value.Name.Should().Be(createDto.Name);
        
        _repositoryMock.Verify(
            r => r.AddAsync(It.Is<Product>(p => p.Name == createDto.Name), It.IsAny<CancellationToken>()),
            Times.Once);
        
        _unitOfWorkMock.Verify(
            u => u.SaveChangesAsync(It.IsAny<CancellationToken>()),
            Times.Once);
    }
    
    [Theory]
    [InlineData(null)]
    [InlineData("")]
    [InlineData("   ")]
    public async Task CreateAsync_WithInvalidName_ReturnsFailure(string? invalidName)
    {
        // Arrange
        var createDto = _fixture.Build<CreateProductDto>()
            .With(d => d.Name, invalidName)
            .Create();
        
        // Act
        var result = await _sut.CreateAsync(createDto);
        
        // Assert
        result.IsSuccess.Should().BeFalse();
        result.Error.Code.Should().Be("Validation.InvalidName");
    }
    
    [Fact]
    public async Task CreateAsync_WhenSaveFails_RollsBackAndReturnsFailure()
    {
        // Arrange
        var createDto = _fixture.Create<CreateProductDto>();
        
        _unitOfWorkMock
            .Setup(u => u.SaveChangesAsync(It.IsAny<CancellationToken>()))
            .ThrowsAsync(new DbUpdateException("Save failed", new Exception()));
        
        // Act
        var act = () => _sut.CreateAsync(createDto);
        
        // Assert
        await act.Should().ThrowAsync<DbUpdateException>();
    }
    
    #endregion
}
```

### 2. Command Handler Tests (CQRS)

```csharp
public class CreateProductCommandHandlerTests
{
    private readonly Mock<IProductRepository> _repositoryMock;
    private readonly Mock<IUnitOfWork> _unitOfWorkMock;
    private readonly CreateProductCommandHandler _sut;
    
    public CreateProductCommandHandlerTests()
    {
        _repositoryMock = new Mock<IProductRepository>();
        _unitOfWorkMock = new Mock<IUnitOfWork>();
        
        _sut = new CreateProductCommandHandler(
            _repositoryMock.Object,
            _unitOfWorkMock.Object,
            Mock.Of<ILogger<CreateProductCommandHandler>>());
    }
    
    [Fact]
    public async Task Handle_WithValidCommand_CreatesProductAndReturnsId()
    {
        // Arrange
        var command = new CreateProductCommand("Test Product", "Description", 99.99m, Guid.NewGuid());
        
        _unitOfWorkMock
            .Setup(u => u.SaveChangesAsync(It.IsAny<CancellationToken>()))
            .ReturnsAsync(1);
        
        // Act
        var result = await _sut.Handle(command, CancellationToken.None);
        
        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Should().NotBeEmpty();
        
        _repositoryMock.Verify(
            r => r.AddAsync(
                It.Is<Product>(p =>
                    p.Name == command.Name &&
                    p.Description == command.Description &&
                    p.Price == command.Price),
                It.IsAny<CancellationToken>()),
            Times.Once);
    }
}
```

### 3. Validator Tests

```csharp
public class CreateProductCommandValidatorTests
{
    private readonly CreateProductCommandValidator _sut;
    
    public CreateProductCommandValidatorTests()
    {
        _sut = new CreateProductCommandValidator();
    }
    
    [Fact]
    public async Task Validate_WithValidCommand_ReturnsSuccess()
    {
        // Arrange
        var command = new CreateProductCommand("Valid Name", "Description", 99.99m, Guid.NewGuid());
        
        // Act
        var result = await _sut.ValidateAsync(command);
        
        // Assert
        result.IsValid.Should().BeTrue();
        result.Errors.Should().BeEmpty();
    }
    
    [Theory]
    [InlineData(null, "Name is required")]
    [InlineData("", "Name is required")]
    [InlineData("ab", "Name must be at least 3 characters")]
    public async Task Validate_WithInvalidName_ReturnsValidationError(
        string? name,
        string expectedError)
    {
        // Arrange
        var command = new CreateProductCommand(name!, "Description", 99.99m, Guid.NewGuid());
        
        // Act
        var result = await _sut.ValidateAsync(command);
        
        // Assert
        result.IsValid.Should().BeFalse();
        result.Errors.Should().Contain(e => e.ErrorMessage.Contains(expectedError));
    }
    
    [Theory]
    [InlineData(0)]
    [InlineData(-1)]
    [InlineData(-100)]
    public async Task Validate_WithInvalidPrice_ReturnsValidationError(decimal price)
    {
        // Arrange
        var command = new CreateProductCommand("Valid Name", "Description", price, Guid.NewGuid());
        
        // Act
        var result = await _sut.ValidateAsync(command);
        
        // Assert
        result.IsValid.Should().BeFalse();
        result.Errors.Should().Contain(e =>
            e.PropertyName == nameof(CreateProductCommand.Price));
    }
}
```

### 4. Entity/Domain Tests

```csharp
public class ProductTests
{
    [Fact]
    public void Create_WithValidData_ReturnsProduct()
    {
        // Arrange
        var name = "Test Product";
        var description = "Test Description";
        var price = 99.99m;
        var categoryId = Guid.NewGuid();
        
        // Act
        var product = Product.Create(name, description, price, categoryId);
        
        // Assert
        product.Should().NotBeNull();
        product.Name.Should().Be(name);
        product.Description.Should().Be(description);
        product.Price.Should().Be(price);
        product.CategoryId.Should().Be(categoryId);
        product.Id.Should().NotBeEmpty();
        product.CreatedAt.Should().BeCloseTo(DateTime.UtcNow, TimeSpan.FromSeconds(1));
    }
    
    [Fact]
    public void Create_RaisesProductCreatedEvent()
    {
        // Act
        var product = Product.Create("Name", "Description", 99.99m, Guid.NewGuid());
        
        // Assert
        product.DomainEvents.Should().ContainSingle()
            .Which.Should().BeOfType<ProductCreatedEvent>();
    }
    
    [Fact]
    public void UpdatePrice_WithValidPrice_UpdatesPriceAndRaisesEvent()
    {
        // Arrange
        var product = Product.Create("Name", "Description", 50m, Guid.NewGuid());
        var newPrice = 75m;
        
        // Act
        product.UpdatePrice(newPrice);
        
        // Assert
        product.Price.Should().Be(newPrice);
        product.UpdatedAt.Should().NotBeNull();
        product.DomainEvents.Should().Contain(e => e is ProductPriceChangedEvent);
    }
    
    [Theory]
    [InlineData(0)]
    [InlineData(-1)]
    public void UpdatePrice_WithInvalidPrice_ThrowsException(decimal invalidPrice)
    {
        // Arrange
        var product = Product.Create("Name", "Description", 50m, Guid.NewGuid());
        
        // Act
        var act = () => product.UpdatePrice(invalidPrice);
        
        // Assert
        act.Should().Throw<DomainException>()
            .WithMessage("*price*");
    }
}
```

## Required NuGet Packages

```xml
<ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.*" />
    <PackageReference Include="xunit" Version="2.*" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.*" />
    <PackageReference Include="Moq" Version="4.*" />
    <PackageReference Include="FluentAssertions" Version="6.*" />
    <PackageReference Include="AutoFixture" Version="4.*" />
    <PackageReference Include="AutoFixture.AutoMoq" Version="4.*" />
    <PackageReference Include="coverlet.collector" Version="6.*" />
</ItemGroup>
```

## Best Practices

- ✅ Follow AAA pattern (Arrange, Act, Assert)
- ✅ Use descriptive test names: `MethodName_Scenario_ExpectedBehavior`
- ✅ One assertion per test (logical assertion)
- ✅ Test both success and failure paths
- ✅ Use Theory for parameterized tests
- ✅ Mock only direct dependencies
- ✅ Use AutoFixture for test data generation
- ✅ Verify mock interactions when relevant
- ✅ Keep tests independent and isolated
- ✅ Use meaningful variable names
- ❌ Don't test private methods directly
- ❌ Don't use magic strings/numbers
- ❌ Don't create complex test setups
