---
agent: 'Code-Generation'
description: 'Scaffold a new Spring Boot REST API using classic layered architecture with controller, service, repository, and Docker support'
# Note: Uses Code-Generation agent if available in .github/agents/, otherwise falls back to default Copilot agent
---

# Scaffold Spring Boot Layered REST API

## Task

Create a new **Spring Boot REST API** using a simple, maintainable **layered architecture**:

- `controller` for HTTP concerns
- `service` for business logic and transactions
- `repository` for persistence

The generated project should be simple enough for small and medium applications, but still follow current Spring Boot best practices for validation, error handling, observability, API documentation, persistence, and containerization.

This prompt is specifically for a classic Spring MVC application, not CQRS, not WebFlux, and not Clean/Hexagonal/Onion architecture unless the user explicitly asks for that complexity.

The output is only considered complete after the project has been scaffolded, built, and validated in the target environment.

## Configuration

Before execution, confirm or infer these settings:

| Setting | Options | Default |
|---------|---------|---------|
| Project Name | `${PROJECT_NAME}` | layered-api |
| Group ID | `${GROUP_ID}` | com.example |
| Artifact ID | `${ARTIFACT_ID}` | layered-api |
| Base Package | `${BASE_PACKAGE}` | com.example.layeredapi |
| Java Version | 17, 21 | 21 |
| Spring Boot Version | 3.5.x, 4.0.x | 3.5.x |
| Build Tool | Maven, Gradle | Maven |
| Database | PostgreSQL, MySQL, H2 | PostgreSQL |
| Include Flyway | Yes, No | Yes |
| Include Docker | Yes, No | Yes |
| Include Docker Compose | Yes, No | Yes |
| Authentication | None, JWT | None |
| Use Lombok | Yes, No | No |

Configuration rules:

- Prefer **Spring Boot 3.5.x** by default for broad ecosystem compatibility unless the user explicitly requests Spring Boot 4.
- Default to **Java 21** when available, while still supporting Java 17 as a baseline option.
- Derive `${BASE_PACKAGE}` from the selected group and artifact, but normalize it into a valid Java package name.
- If Docker Compose is enabled and a relational database is selected, wire the application to that real database rather than generating an in-memory-only setup.
- If H2 is selected, treat it as a lightweight development option, not the production default.
- Use the Maven or Gradle wrapper and validate with the selected Java version.

## Project Structure

Generate a straightforward package layout with clear separation of responsibilities:

```text
${ARTIFACT_ID}/
├── .mvn/ or gradle/
├── mvnw / gradlew
├── mvnw.cmd / gradlew.bat
├── pom.xml or build.gradle
├── .dockerignore
├── Dockerfile
├── docker-compose.yml
├── README.md
├── src/
│   ├── main/
│   │   ├── java/${BASE_PACKAGE_AS_PATH}/
│   │   │   ├── Application.java
│   │   │   ├── config/
│   │   │   │   ├── OpenApiConfig.java
│   │   │   │   └── SecurityConfig.java              # if JWT selected
│   │   │   ├── controller/
│   │   │   │   └── ProductController.java
│   │   │   ├── service/
│   │   │   │   ├── ProductService.java
│   │   │   │   └── ProductServiceImpl.java
│   │   │   ├── repository/
│   │   │   │   └── ProductRepository.java
│   │   │   ├── model/
│   │   │   │   └── Product.java
│   │   │   ├── dto/
│   │   │   │   ├── CreateProductRequest.java
│   │   │   │   ├── UpdateProductRequest.java
│   │   │   │   ├── ProductResponse.java
│   │   │   │   └── PagedResponse.java
│   │   │   ├── mapper/
│   │   │   │   └── ProductMapper.java
│   │   │   ├── exception/
│   │   │   │   ├── GlobalExceptionHandler.java
│   │   │   │   ├── ResourceNotFoundException.java
│   │   │   │   └── ApiExceptionMessage.java        # optional
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-dev.yml
│   │       └── db/migration/
│   │           └── V1__create_products_table.sql   # if Flyway enabled
│   └── test/
│       └── java/${BASE_PACKAGE_AS_PATH}/
│           ├── controller/
│           │   └── ProductControllerTest.java
│           ├── service/
│           │   └── ProductServiceTest.java
│           ├── repository/
│           │   └── ProductRepositoryTest.java
│           └── integration/
│               └── ProductApiIntegrationTest.java
```

## Required Dependencies

Use current, compatible versions for the selected Spring Boot line.

- `spring-boot-starter-web`
- `spring-boot-starter-validation`
- `spring-boot-starter-data-jpa`
- `spring-boot-starter-actuator`
- `springdoc-openapi-starter-webmvc-ui` 2.8.6
- `postgresql` or `mysql-connector-j` or `h2`
- `flyway-core` when migrations are enabled
- `spring-boot-starter-security`, `spring-security-oauth2-resource-server`, `spring-security-oauth2-jose` when JWT is selected
- `spring-boot-starter-test`
- `testcontainers`, `junit-jupiter`, and the matching database module for full integration tests when container-backed testing is requested

Dependency rules:

- Do not add MapStruct, Lombok, or extra mapping frameworks unless the user asks for them.
- Do not expose JPA entities directly from controllers; use DTOs.
- Prefer Spring Boot starters over hand-assembled dependency sets.

## Core Implementation

### 1. Application Configuration

Use profile-aware configuration and keep secrets out of source control.

```yaml
# application.yml
spring:
  application:
    name: ${ARTIFACT_ID}
  datasource:
    url: jdbc:postgresql://localhost:5432/${ARTIFACT_ID}
    username: ${DB_USER:app}
    password: ${DB_PASSWORD:app}
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        format_sql: true
  jackson:
    default-property-inclusion: non_null

management:
  endpoints:
    web:
      exposure:
        include: health,info
  endpoint:
    health:
      probes:
        enabled: true

springdoc:
  api-docs:
    path: /api-docs
  swagger-ui:
    path: /swagger-ui.html
```

```yaml
# application-dev.yml
spring:
  datasource:
    url: jdbc:h2:mem:${ARTIFACT_ID};MODE=PostgreSQL;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
    driver-class-name: org.h2.Driver
    username: sa
    password:
  jpa:
    hibernate:
      ddl-auto: create-drop
```

Configuration guidelines:

- Set `spring.jpa.open-in-view=false` to avoid leaking persistence concerns into the web layer.
- Use `ddl-auto=validate` for the main profile when Flyway is enabled.
- Expose only minimal actuator endpoints by default.
- Keep datasource credentials externalized through environment variables.

### 2. Entity and Repository

Keep entities focused on persistence and repositories focused on data access.

```java
package com.example.layeredapi.model;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.OffsetDateTime;

@Entity
@Table(name = "products")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 200)
    private String name;

    @Column(length = 1000)
    private String description;

    @Column(nullable = false, precision = 12, scale = 2)
    private BigDecimal price;

    @Column(nullable = false, updatable = false)
    private OffsetDateTime createdAt;

    @Column(nullable = false)
    private OffsetDateTime updatedAt;

    @PrePersist
    void onCreate() {
        var now = OffsetDateTime.now();
        this.createdAt = now;
        this.updatedAt = now;
    }

    @PreUpdate
    void onUpdate() {
        this.updatedAt = OffsetDateTime.now();
    }

    // getters and setters
}
```

```java
package com.example.layeredapi.repository;

import com.example.layeredapi.model.Product;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ProductRepository extends JpaRepository<Product, Long> {
    boolean existsByNameIgnoreCase(String name);
}
```

### 3. DTOs and Validation

Use request and response DTOs so the API contract stays independent from persistence.

```java
package com.example.layeredapi.dto;

import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;

import java.math.BigDecimal;

public record CreateProductRequest(
        @NotBlank @Size(max = 200) String name,
        @Size(max = 1000) String description,
        @NotNull @DecimalMin(value = "0.0", inclusive = false) BigDecimal price
) {}
```

```java
package com.example.layeredapi.dto;

import java.math.BigDecimal;
import java.time.OffsetDateTime;

public record ProductResponse(
        Long id,
        String name,
        String description,
        BigDecimal price,
        OffsetDateTime createdAt,
        OffsetDateTime updatedAt
) {}
```

### 4. Service Layer

Business rules, transaction boundaries, and orchestration belong in the service layer, not the controller.

```java
package com.example.layeredapi.service;

import com.example.layeredapi.dto.CreateProductRequest;
import com.example.layeredapi.dto.ProductResponse;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

public interface ProductService {
    ProductResponse create(CreateProductRequest request);
    ProductResponse getById(Long id);
    Page<ProductResponse> getAll(Pageable pageable);
    void delete(Long id);
}
```

```java
package com.example.layeredapi.service;

import com.example.layeredapi.dto.CreateProductRequest;
import com.example.layeredapi.dto.ProductResponse;
import com.example.layeredapi.exception.ResourceNotFoundException;
import com.example.layeredapi.mapper.ProductMapper;
import com.example.layeredapi.model.Product;
import com.example.layeredapi.repository.ProductRepository;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
class ProductServiceImpl implements ProductService {

    private final ProductRepository repository;
    private final ProductMapper mapper;

    ProductServiceImpl(ProductRepository repository, ProductMapper mapper) {
        this.repository = repository;
        this.mapper = mapper;
    }

    @Override
    @Transactional
    public ProductResponse create(CreateProductRequest request) {
        Product saved = repository.save(mapper.toEntity(request));
        return mapper.toResponse(saved);
    }

    @Override
    @Transactional(readOnly = true)
    public ProductResponse getById(Long id) {
        return repository.findById(id)
                .map(mapper::toResponse)
                .orElseThrow(() -> new ResourceNotFoundException("Product", id));
    }

    @Override
    @Transactional(readOnly = true)
    public Page<ProductResponse> getAll(Pageable pageable) {
        return repository.findAll(pageable).map(mapper::toResponse);
    }

    @Override
    @Transactional
    public void delete(Long id) {
        if (!repository.existsById(id)) {
            throw new ResourceNotFoundException("Product", id);
        }
        repository.deleteById(id);
    }
}
```

### 5. REST Controller

Controllers should stay thin and delegate business logic to services.

```java
package com.example.layeredapi.controller;

import com.example.layeredapi.dto.CreateProductRequest;
import com.example.layeredapi.dto.ProductResponse;
import com.example.layeredapi.service.ProductService;
import jakarta.validation.Valid;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.support.ServletUriComponentsBuilder;

import java.net.URI;

@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductService service;

    public ProductController(ProductService service) {
        this.service = service;
    }

    @PostMapping
    public ResponseEntity<ProductResponse> create(@Valid @RequestBody CreateProductRequest request) {
        ProductResponse created = service.create(request);
        URI location = ServletUriComponentsBuilder.fromCurrentRequest()
                .path("/{id}")
                .buildAndExpand(created.id())
                .toUri();
        return ResponseEntity.created(location).body(created);
    }

    @GetMapping("/{id}")
    public ProductResponse getById(@PathVariable Long id) {
        return service.getById(id);
    }

    @GetMapping
    public Page<ProductResponse> getAll(Pageable pageable) {
        return service.getAll(pageable);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        service.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

### 6. Error Handling with ProblemDetail

Use Spring’s `ProblemDetail` support for structured API errors instead of ad hoc maps or string responses.

```java
package com.example.layeredapi.exception;

import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setTitle("Resource not found");
        return problem;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        problem.setTitle("Validation failed");
        problem.setDetail("One or more request fields are invalid.");

        Map<String, String> errors = ex.getBindingResult().getFieldErrors().stream()
                .collect(java.util.stream.Collectors.toMap(
                        error -> error.getField(),
                        error -> error.getDefaultMessage() == null ? "Invalid value" : error.getDefaultMessage(),
                        (first, second) -> first
                ));

        problem.setProperty("errors", errors);
        return problem;
    }
}
```

### 7. OpenAPI and Actuator

- Configure OpenAPI metadata so Swagger UI is immediately usable.
- Include Actuator and expose `health` and `info` only by default.
- If security is enabled, explicitly permit health checks and document actuator access rules.

## Docker Assets

Generate production-friendly container assets derived from the selected stack.

### Dockerfile

Use a multi-stage build, non-root runtime user, and layered jar extraction.

```dockerfile
# syntax=docker/dockerfile:1.7

FROM eclipse-temurin:21-jdk-noble AS build
WORKDIR /workspace

COPY .mvn/ .mvn/
COPY mvnw pom.xml ./

RUN --mount=type=cache,target=/root/.m2 \
    chmod +x mvnw && ./mvnw -B -q -DskipTests dependency:go-offline

COPY src/ src/

RUN --mount=type=cache,target=/root/.m2 \
    ./mvnw -B -q -DskipTests package && \
    java -Djarmode=tools -jar target/*.jar extract --layers --launcher --destination target/extracted

FROM eclipse-temurin:21-jre-noble AS runtime
WORKDIR /app

RUN groupadd --system spring && useradd --system --gid spring spring && \
    apt-get update && apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*

COPY --link --from=build --chown=spring:spring /workspace/target/extracted/dependencies/ ./
COPY --link --from=build --chown=spring:spring /workspace/target/extracted/spring-boot-loader/ ./
COPY --link --from=build --chown=spring:spring /workspace/target/extracted/snapshot-dependencies/ ./
COPY --link --from=build --chown=spring:spring /workspace/target/extracted/application/ ./

USER spring:spring

EXPOSE 8080

ENV JAVA_TOOL_OPTIONS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 -XX:InitialRAMPercentage=50.0"

HEALTHCHECK --interval=30s --timeout=3s --start-period=30s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

### docker-compose.yml

Use Docker Compose for local development when a real database is selected.

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: docker
      DB_USER: app
      DB_PASSWORD: app
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/${ARTIFACT_ID}
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:17-alpine
    environment:
      POSTGRES_DB: ${ARTIFACT_ID}
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d ${ARTIFACT_ID}"]
      interval: 5s
      timeout: 5s
      retries: 10
```

Docker rules:

- Always generate `.dockerignore`.
- Use a non-root runtime user.
- Prefer layered jar extraction over copying a single fat jar when possible.
- Do not hardcode secrets in Docker assets.
- Align the container configuration with the actual Spring configuration contract.

## Testing Expectations

Generate at least these tests unless the user explicitly asks for scaffold-only output:

- `@WebMvcTest` for controller behavior and validation
- service unit tests with Mockito
- `@DataJpaTest` for repository behavior when custom queries exist
- one integration test covering the happy path through the running application

Use Testcontainers for integration tests when the selected database is PostgreSQL or MySQL.

## Execution Steps

1. Generate the project using Spring Boot with the selected Java version, build tool, and dependencies.
2. Create the layered package structure and a complete example feature such as `Product`.
3. Implement DTOs, validation, mapping, repository, service, controller, and exception handling.
4. Add OpenAPI configuration, Actuator health support, and profile-aware application configuration.
5. Add Flyway migrations when enabled.
6. Generate `Dockerfile`, `.dockerignore`, and `docker-compose.yml` when requested.
7. Build the project using the wrapper and run the relevant tests.
8. Start the application locally or in Docker and validate:
   - `GET /actuator/health`
   - `GET /swagger-ui.html`
   - CRUD endpoints for the sample feature
9. Summarize any environment blockers instead of claiming success without verification.

## Best Practices to Follow

- Keep controllers thin and free of business logic.
- Put transaction boundaries in the service layer and use `readOnly = true` for read operations.
- Return DTOs, not JPA entities, from API endpoints.
- Use Bean Validation on request DTOs with `@Valid`.
- Use `ProblemDetail` and centralized exception handling for consistent errors.
- Disable Open Session in View for APIs with `spring.jpa.open-in-view=false`.
- Prefer explicit database migrations over `ddl-auto=create` outside local development.
- Expose only minimal Actuator endpoints by default.
- Use pagination for collection endpoints instead of returning unbounded lists.
- Generate wrapper files and verify the build in the actual target runtime.
- Prefer simplicity over architecture theater; do not add mediators, event buses, or extra layers unless the user asks for them.

## Anti-Patterns to Avoid

- Do not inject repositories directly into controllers.
- Do not put validation only in the frontend or only in the database.
- Do not return `Map<String, Object>` for errors when `ProblemDetail` is available.
- Do not keep all classes in one package.
- Do not use H2-only SQL if PostgreSQL or MySQL is the selected target database.
- Do not silently skip tests, Docker validation, or startup verification and still claim the scaffold is complete.
