---
agent: 'Code-Generation'
description: 'Add Testcontainers support and wiring to an existing Java or Spring Boot project'
# Note: Uses Code-Generation if available in .github/agents/, otherwise falls back to default Copilot agent
---

# Add Testcontainers Support to Java / Spring Boot Project

## Task

Add **Testcontainers support** to the current Java or Spring Boot project so the repository is ready for reliable integration testing against real services such as PostgreSQL, MySQL, MariaDB, MongoDB, Redis, Kafka, RabbitMQ, or other supported containers.

This prompt is for **project setup and wiring**, not for writing a full suite of business-level integration tests. It should prepare the project so future tests can use containerized services cleanly and consistently.

Before generating or modifying anything, inspect the repository and derive the implementation from the current project instead of applying a generic template blindly.

Required discovery steps:

- Detect the build tool from the repository (`pom.xml`, `build.gradle`, wrapper files).
- Detect the Spring Boot version and current test stack.
- Detect which services the application and tests actually depend on.
- Inspect existing integration tests, test profiles, and test utilities.
- Detect whether the project already uses Testcontainers, Docker Compose, embedded databases, or `schema.sql` / `data.sql`.
- Detect whether the project can use Spring Boot **service connections** or needs a fallback strategy.

Output requirements:

- Prefer extending existing test patterns over creating a parallel test architecture.
- Prepare shared container infrastructure and wiring rather than generating a large set of feature tests.
- Do not add Testcontainers to pure unit-test scenarios.
- Do not hardcode `localhost` or fixed mapped ports in tests.
- Do not emit version pins when the project’s dependency management already controls them.

## Configuration

Before execution, confirm or infer these settings:

| Setting | Options | Default |
|---------|---------|---------|
| Scope | Whole project, Specific module, Specific test support layer | Whole project |
| Service Type | PostgreSQL, MySQL, MariaDB, MongoDB, Redis, Kafka, RabbitMQ, Custom container | Infer from project |
| Wiring Strategy | `@ServiceConnection`, `@DynamicPropertySource`, Manual bean/test config | Infer from project |
| Test Scope | `@SpringBootTest`, `@DataJpaTest`, `@JdbcTest`, Mixed | Mixed |
| Lifecycle Style | Spring-managed bean, Static `@Container`, Shared singleton pattern | Infer from project |
| Docker Availability Policy | Required, Skip when unavailable | Required |

Configuration rules:

- If the project uses Spring Boot with service-connection support, prefer `@ServiceConnection` and `spring-boot-testcontainers`.
- If the project cannot use service connections, fall back to `@DynamicPropertySource`.
- Keep the selected service aligned with the real production dependency where possible, for example PostgreSQL support for a PostgreSQL-backed application.
- If the repository already has a stable in-memory-only testing strategy for fast tests, add Testcontainers as an additional integration capability rather than replacing everything.
- Do not configure tests to silently skip because Docker is unavailable unless the user explicitly asks for that behavior.

## Required Dependencies

Use current, compatible dependencies for the selected project line.

Typical Maven dependencies:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
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

- Add only the modules the project actually needs.
- Prefer the project’s dependency management to supply versions.
- Do not keep obsolete Testcontainers artifact names or mixed old/new module naming.

## Implementation

Choose the implementation based on the detected project context.

### 1. Preferred Spring Boot Strategy: `@ServiceConnection`

For Spring Boot projects that support service connections, prefer this approach:

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

Use this when:

- the project already uses Spring Boot test slices or full-context integration tests
- the service is supported by Spring Boot connection details
- the goal is clean wiring with minimal custom property plumbing

If a `GenericContainer<?>` is used, provide the connection hint explicitly:

```java
@Bean
@ServiceConnection(name = "redis")
GenericContainer<?> redisContainer() {
    return new GenericContainer<>("redis:7").withExposedPorts(6379);
}
```

### 2. Fallback Strategy: `@DynamicPropertySource`

Use this when service connections are not supported or the repository already uses explicit property registration:

```java
@Testcontainers
abstract class AbstractContainerSupport {

    @Container
    static PostgreSQLContainer<?> postgres =
            new PostgreSQLContainer<>("postgres:17-alpine");

    @DynamicPropertySource
    static void registerProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}
```

### 3. Spring-Managed Container Beans

When application beans rely on the service during context shutdown or startup ordering matters, prefer Spring-managed container beans:

- reuse a dedicated `ContainersConfig`
- import it from integration tests or a shared abstract base class
- keep startup and shutdown under Spring’s lifecycle when the repository benefits from it

### 4. Recommended Test Support Structure

Keep the structure aligned with the repository:

```text
src/
  test/
    java/com/example/app/
      integration/
        support/
          ContainersConfig.java
          AbstractContainerSupport.java
    resources/
      application-test.yml
```

This setup layer should provide reusable infrastructure for later tests rather than generating a complete feature-specific test suite.

### 5. JPA / SQL Projects

For relational database projects:

- use the matching Testcontainers database module
- keep migrations active in integration environments when Flyway is part of the application
- prefer the real database engine instead of H2 for persistence-sensitive integration support
- keep repository slice tests and full API tests separate when the repository already distinguishes them

### 6. Minimal Validation

The setup should include only enough validation to prove the infrastructure wiring works, for example:

- application context starts with the containerized dependency
- the datasource or connection properties are resolved correctly
- migrations or schema initialization still run as expected

Do not generate a broad suite of business-level endpoint tests unless the user explicitly asks for that next.

## Execution Steps

1. Inspect the repository’s build, test, database, and Spring Boot setup.
2. Detect whether the project already has integration test support and where shared container support belongs.
3. Choose the narrowest Testcontainers setup that improves reliability and matches the repository’s architecture.
4. Add the required dependencies and service module(s).
5. Implement the preferred wiring strategy:
   - `@ServiceConnection` when supported
   - `@DynamicPropertySource` when necessary
   - Spring-managed container beans when lifecycle ordering matters
6. Add or adapt shared test support classes and test configuration.
7. Reconcile the setup with the project’s existing profiles, migrations, and security configuration.
8. Add only minimal validation necessary to prove the containerized setup works.
9. Run the relevant verification path and confirm the infrastructure is ready for future integration tests.

## Best Practices to Follow

- Inspect the existing project before choosing a strategy.
- Prefer `@ServiceConnection` for modern Spring Boot projects that support it.
- Use `@DynamicPropertySource` as a compatibility fallback, not the default for every project.
- Use `getHost()` and mapped ports when manual wiring is required; never assume `localhost`.
- Keep containers shared only where it improves speed without causing state leakage.
- Reuse shared container configuration when the repository already has a test support layer.
- Keep unit tests fast and isolated; do not convert them into integration tests.
- Keep Docker required by default; do not hide failures by silently skipping container setup.
- Validate the setup with the smallest useful proof, not a large generated test suite.

## Anti-Patterns to Avoid

- Do not add Testcontainers to every test class indiscriminately.
- Do not hardcode container hostnames, fixed ports, or local machine assumptions.
- Do not use outdated dependency coordinates when the current project line expects newer modules.
- Do not mix incompatible wiring strategies unless the repository genuinely needs both.
- Do not assume parallel JUnit execution is safe with the chosen container lifecycle.
- Do not turn a setup prompt into a feature-test-generation prompt unless the user explicitly asks for it.
