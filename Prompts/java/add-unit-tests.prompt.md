---
agent: 'Unit-Testing-Generator'
description: 'Generate comprehensive unit tests using JUnit 5, Mockito, and AssertJ'
# Note: Uses unit-testing-generator.agent if available in .github/agents/, otherwise falls back to default Copilot agent
---

# Generate Unit Tests (Java / Spring Boot)

## Task

Generate comprehensive unit tests for Java/Spring Boot code using **JUnit 5** as the test framework, **Mockito** for mocking dependencies, and **AssertJ** for fluent, readable assertions. Tests must follow the **Arrange-Act-Assert (AAA)** pattern, achieve a configurable coverage target, and be structured for maintainability.

## Configuration

| Setting            | Options                                         | Default   |
| ------------------ | ----------------------------------------------- | --------- |
| Test Framework     | JUnit 5                                         | JUnit 5   |
| Assertion Library  | AssertJ, Hamcrest, Native (JUnit Assertions)    | AssertJ   |
| Mocking Library    | Mockito, EasyMock                               | Mockito   |
| Coverage Target    | 70%, 80%, 90%                                   | 80%       |

## Test Project Structure

Follow the Maven/Gradle standard layout. Mirror the main source package hierarchy under `src/test/java`:

```
src/
  test/
    java/
      com/example/sample/
        sample/                       # Tests for the feature
          SampleServiceTest.java
        model/                       # Domain / entity tests
          SampleTest.java
        helpers/                     # Shared test utilities
          TestDataFactory.java
    resources/
      application-test.yml    # Test-specific configuration
```

## Required Maven Dependencies

All of these are included transitively through `spring-boot-starter-test`. For non-Spring projects, add JUnit 5, Mockito, and AssertJ individually.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

## Test Class Examples

### 1. Service Tests

Service tests isolate business logic by mocking all collaborators (repositories, external clients).

```java
@ExtendWith(MockitoExtension.class)
@DisplayName("ProductService")
class ProductServiceTest {

    @Mock
    private ProductRepository productRepository;

    @InjectMocks
    private ProductService productService;

    private Product sampleProduct;

    @BeforeEach
    void setUp() {
        sampleProduct = new Product();
        sampleProduct.setId(1L);
        sampleProduct.setName("Flea Collar");
        sampleProduct.setPrice(new BigDecimal("19.99"));
    }

    @Nested
    @DisplayName("getById")
    class GetById {

        @Test
        @DisplayName("should return product when found")
        void getById_existingId_returnsProduct() {
            // Arrange
            given(productRepository.findById(1L)).willReturn(Optional.of(sampleProduct));

            // Act
            Optional<Product> result = productService.getById(1L);

            // Assert
            assertThat(result)
                .isPresent()
                .hasValueSatisfying(product -> {
                    assertThat(product.getId()).isEqualTo(1L);
                    assertThat(product.getName()).isEqualTo("Flea Collar");
                });
            then(productRepository).should().findById(1L);
        }

        @Test
        @DisplayName("should return empty when product not found")
        void getById_nonExistingId_returnsEmpty() {
            given(productRepository.findById(999L)).willReturn(Optional.empty());

            Optional<Product> result = productService.getById(999L);

            assertThat(result).isEmpty();
        }

        @Test
        @DisplayName("should propagate repository exception")
        void getById_repositoryThrows_propagatesException() {
            given(productRepository.findById(anyLong()))
                .willThrow(new RuntimeException("Database connection lost"));

            assertThatThrownBy(() -> productService.getById(1L))
                .isInstanceOf(RuntimeException.class)
                .hasMessageContaining("Database connection lost");
        }
    }

    @Nested
    @DisplayName("createProduct")
    class CreateProduct {

        @Test
        @DisplayName("should create and return product when input is valid")
        void createProduct_validInput_createsAndReturnsProduct() {
            // Arrange
            CreateProductRequest request = new CreateProductRequest("Flea Collar", new BigDecimal("19.99"), "Accessories");
            given(productRepository.save(any(Product.class))).willReturn(sampleProduct);

            // Act
            Product result = productService.createProduct(request);

            // Assert
            assertThat(result).isNotNull();
            assertThat(result.getName()).isEqualTo("Flea Collar");
            then(productRepository).should().save(any(Product.class));
        }

        @ParameterizedTest
        @NullAndEmptySource
        @ValueSource(strings = {"   ", "\t"})
        @DisplayName("should throw when product name is blank or null")
        void createProduct_blankOrNullName_throwsException(String invalidName) {
            CreateProductRequest request = new CreateProductRequest(invalidName, new BigDecimal("19.99"), "Accessories");

            assertThatThrownBy(() -> productService.createProduct(request))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("name");

            then(productRepository).should(never()).save(any(Product.class));
        }
    }

    // Apply the same @Nested pattern for findAll, deleteProduct, updateProduct, etc.
}
```

### 2. Controller Tests (Unit-Level with `@WebMvcTest`)

Controller tests focus on HTTP request mapping, serialization, and response status codes without loading the full application context.

```java
@WebMvcTest(ProductController.class)
@DisplayName("ProductController")
class ProductControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @MockBean
    private ProductService productService;

    @Nested
    @DisplayName("GET /api/products/{id}")
    class GetProductById {

        @Test
        @DisplayName("should return 200 with product when found")
        void getProductById_existingId_returnsOk() throws Exception {
            Product product = new Product(1L, "Flea Collar", new BigDecimal("19.99"), "Accessories");
            given(productService.getById(1L)).willReturn(Optional.of(product));

            mockMvc.perform(get("/api/products/{id}", 1L)
                    .accept(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.id", is(1)))
                .andExpect(jsonPath("$.name", is("Flea Collar")));
        }

        @Test
        @DisplayName("should return 404 when product not found")
        void getProductById_nonExistingId_returnsNotFound() throws Exception {
            given(productService.getById(999L)).willReturn(Optional.empty());

            mockMvc.perform(get("/api/products/{id}", 999L)
                    .accept(MediaType.APPLICATION_JSON))
                .andExpect(status().isNotFound());
        }
    }

    @Nested
    @DisplayName("POST /api/products")
    class CreateProduct {

        @Test
        @DisplayName("should return 201 when product is created successfully")
        void createProduct_validRequest_returnsCreated() throws Exception {
            CreateProductRequest request = new CreateProductRequest("Flea Collar", new BigDecimal("19.99"), "Accessories");
            Product created = new Product(1L, "Flea Collar", new BigDecimal("19.99"), "Accessories");
            given(productService.createProduct(any(CreateProductRequest.class))).willReturn(created);

            mockMvc.perform(post("/api/products")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.id", is(1)));
        }

        @Test
        @DisplayName("should return 400 when request body is invalid")
        void createProduct_invalidRequest_returnsBadRequest() throws Exception {
            CreateProductRequest request = new CreateProductRequest("", new BigDecimal("19.99"), "Accessories");

            mockMvc.perform(post("/api/products")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isBadRequest());
        }
    }

    // Apply the same @Nested pattern for GET /api/products, PUT, DELETE endpoints.
}
```

> **Additional test types:** Apply the same AAA pattern for **Validator tests** (instantiate `jakarta.validation.Validator`, call `validate()`, assert constraint violations by property path) and **Entity/Domain tests** (test business methods, factory methods, and invariants directly with no mocking).

## Best Practices

### Naming & Structure
- **Naming pattern**: `methodName_scenario_expectedBehavior` (e.g., `getById_existingId_returnsProduct`)
- **`@Nested` classes**: Group tests by the method they exercise for clear tree output in test runners
- **`@DisplayName`**: Add human-readable labels to all test classes and methods

### Mockito Style
- Prefer **BDD-style** Mockito (`given`/`then`) over classic (`when`/`verify`) for AAA alignment
- Only verify interactions when the call itself is the expected behavior (e.g., `then(repo).should(never()).save(...)` on validation failure)

### Parameterized & Data-Driven Tests

```java
@ParameterizedTest
@CsvSource({"Flea Collar, 19.99", "Dog Food, 34.50"})
@DisplayName("should create product for various valid inputs")
void createProduct_variousValidInputs_succeeds(String name, BigDecimal price) { ... }

@ParameterizedTest
@NullAndEmptySource
@ValueSource(strings = {"   ", "\t"})
@DisplayName("should reject blank names")
void createProduct_blankNames_throws(String invalidName) { ... }
```

### Assertions
- Use **AssertJ fluent assertions** (`assertThat(...).isEqualTo(...)`) over JUnit's `assertEquals`
- Use **`SoftAssertions.assertSoftly`** when verifying multiple fields to collect all failures instead of stopping at the first
- Use **`assertThatThrownBy`** for exception assertions with message and cause checks
- Use **`assertThatCode(...).doesNotThrowAnyException()`** to explicitly verify no exception is thrown

### Test Utilities
- Create a `TestDataFactory` with static factory methods for common test objects to reduce setup duplication across test classes

## Coverage Checklist

For each class under test, ensure the following scenarios are covered:

- [ ] Happy path (valid inputs, expected outputs)
- [ ] Null or missing inputs
- [ ] Blank/empty string inputs
- [ ] Boundary values (max length, zero, negative numbers)
- [ ] Exception paths (repository failures, external service errors)
- [ ] Edge cases (duplicate entries, concurrent modifications)
- [ ] Interaction verification (correct collaborators called with correct arguments)
