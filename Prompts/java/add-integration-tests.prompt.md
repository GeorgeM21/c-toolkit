---
agent: 'Unit-Testing-Generator'
description: 'Generate Spring Boot integration tests that validate the real application flow and reuse existing Testcontainers support when available'
# Note: Uses Unit-Testing-Generator if available in .github/agents/, otherwise falls back to default Copilot agent
---

# Generate Integration Tests (Java / Spring Boot)

## Task

Generate integration tests for an existing **Spring Boot** application that validate the real application flow across HTTP, serialization, validation, security, persistence, and error handling.

Before generating tests, inspect the repository and derive the test strategy from the existing project instead of applying a generic template blindly.

Required discovery steps:

- Detect the Spring Boot version, build tool, and test stack.
- Detect whether the project already has integration test support classes or conventions.
- Detect whether the project already uses Testcontainers, and if so, reuse its support layer instead of creating a second one.
- Detect the actual production database or backing service and whether H2 would be an inaccurate substitute.
- Detect whether Flyway or Liquibase is part of the application startup path.
- Detect the project’s security model and existing test authentication patterns.

Output requirements:

- Prefer repository-aligned tests over generic examples.
- Reuse existing Testcontainers support when available.
- Prefer realistic database-backed integration tests for dialect-sensitive or migration-sensitive applications.
- Keep unit-test concerns out of this prompt.
- Do not generate a parallel container-support architecture if the repository already has one.

## Configuration

Before execution, confirm or infer these settings:

| Setting | Options | Default |
|---------|---------|---------|
| Test Framework | JUnit 5 | JUnit 5 |
| HTTP Style | MockMvc, TestRestTemplate, Existing project style | Existing project style |
| Database Strategy | Existing Testcontainers support, `@ServiceConnection`, `@DynamicPropertySource`, H2 fallback | Prefer existing support |
| Authentication | Mock, Real security flow, Existing project style | Existing project style |
| Test Scope | Full application flow, Repository-backed API flow, Selected endpoints only | Full application flow |

Configuration rules:

- For Spring Boot **3.1+**, prefer existing Testcontainers support or `@ServiceConnection` when the project supports it.
- Use `@DynamicPropertySource` as a fallback for older projects or unsupported connection-detail cases.
- Use H2 only when it is a reasonable substitute for the project’s real data behavior; do not default to it blindly.
- If Flyway or Liquibase is part of the application, keep migrations active in integration tests unless the repository has a deliberate alternative.
- Choose `MockMvc` for in-process MVC integration tests and `TestRestTemplate` only when the full HTTP server path is intentionally required.

## Recommended Test Structure

Follow the repository’s existing test structure. A typical shape is:

```text
src/
  test/
    java/
      com/example/app/
        integration/
          support/
            AbstractIntegrationTest.java
          product/
            ProductApiIntegrationTest.java
    resources/
      application-test.yml
```

If the project already has a shared Testcontainers support layer, reuse it instead of generating new container wiring here.

## Required Dependencies

Use the project’s dependency management when available.

Typical Maven dependencies:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-test</artifactId>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers-junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
```

Dependency rules:

- Add Testcontainers dependencies only when the repository does not already have the required support.
- Add only the modules needed by the actual project.
- Do not use outdated Testcontainers coordinates.
- Do not pin versions when the Spring Boot BOM or existing dependency management already handles them.

## Implementation

Choose the test design based on the detected project context.

### 1. Base Integration Test

For MockMvc-based integration tests, prefer a full Spring context with in-process MVC:

```java
@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
abstract class AbstractIntegrationTest {

    @Autowired
    protected MockMvc mockMvc;

    @Autowired
    protected ObjectMapper objectMapper;
}
```

Use `@Transactional` only when it fits the repository’s persistence model and test style. Do not apply it blindly to every integration test base class.

### 2. Reuse Existing Testcontainers Support

If the project already has shared container support, extend or import it instead of recreating it:

```java
class ProductApiIntegrationTest extends AbstractIntegrationTest {
    // Reuse existing container-backed integration support
}
```

If container support does not exist yet and the project is Spring Boot 3.1+, a typical modern setup uses `@ServiceConnection`:

```java
@TestConfiguration(proxyBeanMethods = false)
class ContainersConfig {

    @Bean
    @ServiceConnection
    PostgreSQLContainer<?> postgresContainer() {
        return new PostgreSQLContainer<>("postgres:17-alpine");
    }
}
```

Fallback example when `@DynamicPropertySource` is required:

```java
@Testcontainers
abstract class AbstractContainerSupport {

    @Container
    static PostgreSQLContainer<?> postgres =
            new PostgreSQLContainer<>("postgres:17-alpine");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}
```

### 3. MockMvc-Based Integration Test Example

```java
@DisplayName("Product API Integration Tests")
class ProductApiIntegrationTest extends AbstractIntegrationTest {

    @Autowired
    private ProductRepository productRepository;

    @BeforeEach
    void clearData() {
        productRepository.deleteAll();
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    void createProduct_validRequest_returnsCreated() throws Exception {
        CreateProductRequest request = new CreateProductRequest(
                "Flea Collar", new BigDecimal("19.99"), "Accessories");

        mockMvc.perform(post("/api/products")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.id", is(notNullValue())))
                .andExpect(jsonPath("$.name", is("Flea Collar")));
    }

    @Test
    void createProduct_unauthenticated_returnsUnauthorized() throws Exception {
        CreateProductRequest request = new CreateProductRequest(
                "Flea Collar", new BigDecimal("19.99"), "Accessories");

        mockMvc.perform(post("/api/products")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isUnauthorized());
    }
}
```

### 4. TestRestTemplate Example

Use `TestRestTemplate` only when the goal is to validate the embedded server over real HTTP:

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
class ProductApiHttpIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void getProduct_existingId_returnsOk() {
        ResponseEntity<ProductResponse> response =
                restTemplate.getForEntity("/api/products/{id}", ProductResponse.class, 1L);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
    }
}
```

Do not combine `MockMvc` and `RANDOM_PORT` as the default example unless the repository already does so deliberately.

### 5. Test Data Setup

Choose the repository’s existing style:

- repository-based seeding in `@BeforeEach`
- SQL seeding with `@Sql`
- fixture builders or test factories

If the project uses Flyway or Liquibase, do not disable it by default in the integration-test profile just to simplify seeding.

### 6. What the Tests Should Validate

For each selected endpoint or flow, integration tests should validate:

- successful requests through the real application stack
- persistence effects against the configured database
- validation failures and error responses
- authentication and authorization behavior
- not-found and conflict scenarios when relevant
- serialization and deserialization behavior
- migration-backed startup when the application depends on migrations

## Execution Steps

1. Inspect the repository’s current test, security, database, and migration setup.
2. Detect whether existing Testcontainers support should be reused.
3. Choose the appropriate integration-test style:
   - MockMvc for in-process MVC flow
   - TestRestTemplate for real HTTP server flow
4. Add only the dependencies and support pieces that are actually missing.
5. Generate integration tests for the selected endpoints or flows using the project’s real conventions.
6. Keep migrations, datasource configuration, and security aligned with the actual application.
7. Run the relevant tests and verify they pass with the intended infrastructure.

## Best Practices to Follow

- Inspect the project before choosing H2, Testcontainers, MockMvc, or TestRestTemplate.
- Reuse existing Testcontainers support whenever it already exists.
- Prefer `@ServiceConnection` for Spring Boot 3.1+ when new Testcontainers wiring is needed.
- Use `@DynamicPropertySource` as a fallback, not the default for all modern projects.
- Keep migration tools active in integration tests when they are part of application startup.
- Use realistic backing services when SQL dialect or migration behavior matters.
- Separate integration-test concerns from unit-test concerns.
- Keep tests independent and deterministic.
- Validate the behavior that the real application stack is responsible for, not just controller method invocation.

## Anti-Patterns to Avoid

- Do not default every project to H2 when the production database is materially different.
- Do not disable Flyway or Liquibase by default just to make tests easier.
- Do not generate new container-support infrastructure if the repository already has it.
- Do not mix MockMvc and real-server HTTP examples as the default baseline.
- Do not treat integration tests as oversized unit tests with excessive mocking.
- Do not assume parallel JUnit execution is safe for the chosen Testcontainers lifecycle.
