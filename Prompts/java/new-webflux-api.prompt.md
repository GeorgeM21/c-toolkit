---
agent: 'Feature-Scaffolder'
description: 'Scaffold a new Spring WebFlux reactive API with functional endpoints and OpenAPI documentation'
# Note: Uses Feature-Scaffolder agent if available in .github/agents/, otherwise falls back to default Copilot agent
---

# Scaffold a New Spring WebFlux Reactive API

## Task

Create a new Spring WebFlux reactive API project using **functional endpoints** (the `RouterFunction` / `HandlerFunction` pattern). This is the reactive, non-blocking equivalent of traditional Spring MVC REST controllers and the closest Java analogue to .NET Minimal APIs.

The generated project must follow reactive best practices end-to-end: R2DBC for database access, `Mono`/`Flux` return types everywhere, and zero blocking calls on the event loop.

The generated output is only considered complete after it has been built, started, and validated in the target environment.

---

## Configuration

| Parameter              | Options                                          | Default            |
|------------------------|--------------------------------------------------|--------------------|
| Project Name           | *user-provided*                                  | `webflux-api`      |
| Java Version           | 17, 21                                           | 21                 |
| Database               | R2DBC PostgreSQL, R2DBC MySQL, R2DBC H2          | R2DBC H2           |
| Authentication         | None, JWT                                        | None               |
| Include Docker         | Yes / No                                         | Yes                |
| Use Functional Endpoints | Yes / No                                       | Yes                |

Verification rules:

- If Maven is used, generate the Maven wrapper (`mvnw`, `mvnw.cmd`, `.mvn/wrapper/**`) and use it for all build and run validation.
- If Java 21 is selected, validate with a JDK 21+ runtime. Do not silently verify with Java 17 and claim success.
- If the required JDK is unavailable, report that validation is blocked by the environment instead of claiming the scaffold works.

---

## Project Structure

Generate a **feature-based** package layout. Each feature gets its own package containing router, handler, model, and validator classes.

```
{{projectName}}/
├── .mvn/
│   └── wrapper/
├── pom.xml (or build.gradle)
├── mvnw
├── mvnw.cmd
├── Dockerfile
├── docker-compose.yml
├── src/
│   ├── main/
│   │   ├── java/com/example/{{projectName}}/
│   │   │   ├── Application.java
│   │   │   ├── config/
│   │   │   │   ├── OpenApiConfig.java
│   │   │   │   ├── R2dbcConfig.java
│   │   │   │   └── SecurityConfig.java          # if JWT selected
│   │   │   ├── common/
│   │   │   │   ├── model/
│   │   │   │   │   ├── PaginatedResponse.java
│   │   │   │   │   ├── PaginationQuery.java
│   │   │   │   │   └── ErrorResponse.java
│   │   │   │   ├── handler/
│   │   │   │   │   └── GlobalErrorHandler.java
│   │   │   │   └── validation/
│   │   │   │       └── RequestValidator.java
│   │   │   └── product/
│   │   │       ├── ProductRouter.java
│   │   │       ├── ProductHandler.java
│   │   │       ├── ProductRepository.java
│   │   │       ├── ProductValidator.java
│   │   │       └── model/
│   │   │           ├── Product.java
│   │   │           ├── ProductResponse.java
│   │   │           └── CreateProductRequest.java
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-dev.yml
│   │       └── schema.sql
│   └── test/
│       └── java/com/example/{{projectName}}/
│           ├── product/
│           │   └── ProductRouterIntegrationTest.java
│           └── ApplicationTests.java
```

---

## Required Dependencies

- `spring-boot-starter-parent` 3.3.0
- `spring-boot-starter-webflux`
- `spring-boot-starter-data-r2dbc`
- `r2dbc-h2` (runtime; swap for `r2dbc-postgresql` if needed)
- `springdoc-openapi-starter-webflux-ui` 2.5.0
- `spring-boot-starter-validation`
- `spring-boot-starter-security` + `spring-security-oauth2-resource-server` + `spring-security-oauth2-jose` (optional, if JWT)
- `spring-boot-starter-test` (test)
- `reactor-test` (test)

---

## Core Implementation

### 1. Application Configuration

```yaml
# application.yml
spring:
  application:
    name: {{projectName}}
  r2dbc:
    url: r2dbc:h2:mem:///testdb
    username: sa
    password:
  sql:
    init:
      mode: always
      schema-locations: classpath:schema.sql

springdoc:
  api-docs:
    path: /api-docs
  swagger-ui:
    path: /swagger-ui.html

server:
  port: 8080

logging:
  level:
    org.springframework.r2dbc: DEBUG
```

```yaml
# application-dev.yml -- PostgreSQL example
spring:
  r2dbc:
    url: r2dbc:postgresql://localhost:5432/{{projectName}}
    username: postgres
    password: postgres
```

### 2. Database Schema

```sql
-- schema.sql
CREATE TABLE IF NOT EXISTS product (
                                       id          BIGINT AUTO_INCREMENT PRIMARY KEY,
                                       name        VARCHAR(255)   NOT NULL,
    description VARCHAR(1000),
    price       DECIMAL(10,2)  NOT NULL,
    category    VARCHAR(100),
    in_stock    BOOLEAN        NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMP      NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP      NOT NULL DEFAULT CURRENT_TIMESTAMP
    );
```

---

### 3. Domain Entity

```java
package com.example.webfluxapi.product.model;

import org.springframework.data.annotation.Id;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.relational.core.mapping.Table;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Table("product")
public class Product {

  @Id private Long id;
  private String name;
  private String description;
  private BigDecimal price;
  private String category;
  private boolean inStock;
  @CreatedDate private LocalDateTime createdAt;
  @LastModifiedDate private LocalDateTime updatedAt;

  public Product() {}

  public Product(String name, String description, BigDecimal price,
      String category, boolean inStock) {
    this.name = name; this.description = description;
    this.price = price; this.category = category; this.inStock = inStock;
  }

  // Getters and setters
  public Long getId() { return id; }
  public void setId(Long id) { this.id = id; }
  public String getName() { return name; }
  public void setName(String name) { this.name = name; }
  public String getDescription() { return description; }
  public void setDescription(String description) { this.description = description; }
  public BigDecimal getPrice() { return price; }
  public void setPrice(BigDecimal price) { this.price = price; }
  public String getCategory() { return category; }
  public void setCategory(String category) { this.category = category; }
  public boolean isInStock() { return inStock; }
  public void setInStock(boolean inStock) { this.inStock = inStock; }
  public LocalDateTime getCreatedAt() { return createdAt; }
  public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }
  public LocalDateTime getUpdatedAt() { return updatedAt; }
  public void setUpdatedAt(LocalDateTime updatedAt) { this.updatedAt = updatedAt; }
}
```

### 4. Request and Response Records

```java
package com.example.webfluxapi.product.model;

import jakarta.validation.constraints.*;
import java.math.BigDecimal;

public record CreateProductRequest(
    @NotBlank(message = "Name is required") String name,
    String description,
    @NotNull @DecimalMin(value = "0.01", message = "Price must be greater than zero") BigDecimal price,
    String category,
    Boolean inStock
) {
  public Product toEntity() {
    return new Product(name, description, price, category, inStock != null ? inStock : true);
  }
}
```

```java
package com.example.webfluxapi.product.model;

import java.math.BigDecimal;
import java.time.LocalDateTime;

public record ProductResponse(Long id, String name, String description, BigDecimal price,
                              String category, boolean inStock, LocalDateTime createdAt, LocalDateTime updatedAt) {
  public static ProductResponse from(Product p) {
    return new ProductResponse(p.getId(), p.getName(), p.getDescription(), p.getPrice(),
        p.getCategory(), p.isInStock(), p.getCreatedAt(), p.getUpdatedAt());
  }
}
```

### 5. Common Models

```java
package com.example.webfluxapi.common.model;

public record PaginationQuery(int page, int size) {
  public PaginationQuery {
    if (page < 0) page = 0;
    if (size <= 0) size = 20;
    if (size > 100) size = 100;
  }
  public static PaginationQuery of(int page, int size) { return new PaginationQuery(page, size); }
  public long offset() { return (long) page * size; }
}
```

```java
package com.example.webfluxapi.common.model;

import java.util.List;

public record PaginatedResponse<T>(List<T> items, int page, int size, long totalElements, int totalPages) {
  public static <T> PaginatedResponse<T> of(List<T> items, int page, int size, long total) {
    return new PaginatedResponse<>(items, page, size, total, (int) Math.ceil((double) total / size));
  }
}
```

```java
package com.example.webfluxapi.common.model;

import java.time.LocalDateTime;
import java.util.Map;

public record ErrorResponse(int status, String message, Map<String, String> errors, LocalDateTime timestamp) {
  public static ErrorResponse of(int status, String message) {
    return new ErrorResponse(status, message, Map.of(), LocalDateTime.now());
  }
  public static ErrorResponse of(int status, String message, Map<String, String> errors) {
    return new ErrorResponse(status, message, errors, LocalDateTime.now());
  }
}
```

---

### 6. Reactive Repository

```java
package com.example.webfluxapi.product;

import com.example.webfluxapi.product.model.Product;
import org.springframework.data.r2dbc.repository.Query;
import org.springframework.data.repository.reactive.ReactiveCrudRepository;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

public interface ProductRepository extends ReactiveCrudRepository<Product, Long> {
  Flux<Product> findByCategory(String category);
  Flux<Product> findByInStockTrue();
  @Query("SELECT * FROM product ORDER BY created_at DESC LIMIT :size OFFSET :offset")
  Flux<Product> findAllPaginated(int size, long offset);
  @Query("SELECT COUNT(*) FROM product")
  Mono<Long> countAll();
  Flux<Product> findByNameContainingIgnoreCase(String name);
}
```

---

### 7. Validator

```java
package com.example.webfluxapi.product;

import com.example.webfluxapi.product.model.CreateProductRequest;
import org.springframework.stereotype.Component;
import org.springframework.validation.*;

@Component
public class ProductValidator implements Validator {

  @Override
  public boolean supports(Class<?> clazz) { return CreateProductRequest.class.isAssignableFrom(clazz); }

  @Override
  public void validate(Object target, Errors errors) {
    CreateProductRequest request = (CreateProductRequest) target;
    if (request.name() != null && request.name().length() > 255)
      errors.rejectValue("name", "name.tooLong", "Name must not exceed 255 characters");
    // Add additional business-rule validations as needed
  }

  public Errors validateRequest(CreateProductRequest request) {
    Errors errors = new BeanPropertyBindingResult(request, "createProductRequest");
    validate(request, errors);
    return errors;
  }
}
```

### 8. Generic Request Validator Utility

```java
package com.example.webfluxapi.common.validation;

import org.springframework.stereotype.Component;
import org.springframework.validation.*;
import org.springframework.web.server.ServerWebInputException;
import reactor.core.publisher.Mono;
import java.util.*;
import java.util.stream.Collectors;

@Component
public class RequestValidator {

  private final jakarta.validation.Validator beanValidator;
  public RequestValidator(jakarta.validation.Validator beanValidator) { this.beanValidator = beanValidator; }

  public <T> Mono<T> validate(T body) {
    var violations = beanValidator.validate(body);
    if (violations.isEmpty()) return Mono.just(body);
    Map<String, String> errors = violations.stream()
        .collect(Collectors.toMap(v -> v.getPropertyPath().toString(), v -> v.getMessage(), (a, b) -> a + "; " + b));
    return Mono.error(new ServerWebInputException("Validation failed: " + errors));
  }

  public <T> Mono<T> validate(T body, Validator customValidator) {
    Errors errors = new BeanPropertyBindingResult(body, body.getClass().getSimpleName());
    customValidator.validate(body, errors);
    if (!errors.hasErrors()) return Mono.just(body);
    Map<String, String> fieldErrors = new HashMap<>();
    errors.getFieldErrors().forEach(fe -> fieldErrors.put(fe.getField(), fe.getDefaultMessage()));
    return Mono.error(new ServerWebInputException("Validation failed: " + fieldErrors));
  }
}
```

---

### 9. Product Handler (Functional Endpoint Logic)

```java
package com.example.webfluxapi.product;

import com.example.webfluxapi.common.model.*;
import com.example.webfluxapi.common.validation.RequestValidator;
import com.example.webfluxapi.product.model.*;
import org.springframework.http.*;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.server.*;
import reactor.core.publisher.Mono;
import java.net.URI;

@Component
public class ProductHandler {

  private final ProductRepository repository;
  private final RequestValidator requestValidator;
  private final ProductValidator productValidator;

  public ProductHandler(ProductRepository repository,
      RequestValidator requestValidator,
      ProductValidator productValidator) {
    this.repository = repository;
    this.requestValidator = requestValidator;
    this.productValidator = productValidator;
  }

  public Mono<ServerResponse> getAll(ServerRequest request) {
    int page = request.queryParam("page").map(Integer::parseInt).orElse(0);
    int size = request.queryParam("size").map(Integer::parseInt).orElse(20);
    PaginationQuery pagination = PaginationQuery.of(page, size);

    return repository.countAll()
        .flatMap(total ->
            repository.findAllPaginated(pagination.size(), pagination.offset())
                .map(ProductResponse::from).collectList()
                .map(items -> PaginatedResponse.of(items, pagination.page(), pagination.size(), total)))
        .flatMap(body -> ServerResponse.ok().contentType(MediaType.APPLICATION_JSON).bodyValue(body));
  }

  public Mono<ServerResponse> getById(ServerRequest request) {
    Long id = Long.parseLong(request.pathVariable("id"));
    return repository.findById(id).map(ProductResponse::from)
        .flatMap(p -> ServerResponse.ok().contentType(MediaType.APPLICATION_JSON).bodyValue(p))
        .switchIfEmpty(ServerResponse.status(HttpStatus.NOT_FOUND)
            .bodyValue(ErrorResponse.of(404, "Product not found with id: " + id)));
  }

  public Mono<ServerResponse> create(ServerRequest request) {
    return request.bodyToMono(CreateProductRequest.class)
        .flatMap(requestValidator::validate)
        .flatMap(body -> requestValidator.validate(body, productValidator))
        .map(CreateProductRequest::toEntity)
        .flatMap(repository::save)
        .map(ProductResponse::from)
        .flatMap(product -> ServerResponse.created(URI.create("/api/products/" + product.id()))
            .contentType(MediaType.APPLICATION_JSON).bodyValue(product))
        .onErrorResume(e -> ServerResponse.badRequest()
            .bodyValue(ErrorResponse.of(400, e.getMessage())));
  }

  public Mono<ServerResponse> update(ServerRequest request) {
    Long id = Long.parseLong(request.pathVariable("id"));
    return repository.findById(id)
        .switchIfEmpty(Mono.error(new ProductNotFoundException(id)))
        .flatMap(existing -> request.bodyToMono(CreateProductRequest.class)
            .flatMap(requestValidator::validate)
            .flatMap(body -> requestValidator.validate(body, productValidator))
            .map(body -> {
              existing.setName(body.name()); existing.setDescription(body.description());
              existing.setPrice(body.price()); existing.setCategory(body.category());
              existing.setInStock(body.inStock() != null ? body.inStock() : existing.isInStock());
              return existing;
            }))
        .flatMap(repository::save).map(ProductResponse::from)
        .flatMap(p -> ServerResponse.ok().contentType(MediaType.APPLICATION_JSON).bodyValue(p))
        .onErrorResume(ProductNotFoundException.class, e ->
            ServerResponse.status(HttpStatus.NOT_FOUND).bodyValue(ErrorResponse.of(404, e.getMessage())))
        .onErrorResume(e -> ServerResponse.badRequest().bodyValue(ErrorResponse.of(400, e.getMessage())));
  }

  public Mono<ServerResponse> delete(ServerRequest request) {
    Long id = Long.parseLong(request.pathVariable("id"));
    return repository.findById(id)
        .switchIfEmpty(Mono.error(new ProductNotFoundException(id)))
        .flatMap(product -> repository.deleteById(id).thenReturn(product))
        .flatMap(deleted -> ServerResponse.noContent().build())
        .onErrorResume(ProductNotFoundException.class, e ->
            ServerResponse.status(HttpStatus.NOT_FOUND).bodyValue(ErrorResponse.of(404, e.getMessage())));
  }

  public Mono<ServerResponse> search(ServerRequest request) {
    String name = request.queryParam("name").orElse("");
    return repository.findByNameContainingIgnoreCase(name)
        .map(ProductResponse::from).collectList()
        .flatMap(products -> ServerResponse.ok().contentType(MediaType.APPLICATION_JSON).bodyValue(products));
  }

  static class ProductNotFoundException extends RuntimeException {
    ProductNotFoundException(Long id) { super("Product not found with id: " + id); }
  }
}
```

---

### 10. Router Configuration (Functional Endpoints)

```java
package com.example.webfluxapi.product;

import org.springdoc.core.annotations.RouterOperation;
import org.springdoc.core.annotations.RouterOperations;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.MediaType;
import org.springframework.web.reactive.function.server.RouterFunction;
import org.springframework.web.reactive.function.server.ServerResponse;

import static org.springframework.web.reactive.function.server.RequestPredicates.*;
import static org.springframework.web.reactive.function.server.RouterFunctions.route;

@Configuration
public class ProductRouter {

  @Bean
  @RouterOperations({
      @RouterOperation(path = "/api/products", method = org.springframework.web.bind.annotation.RequestMethod.GET, beanClass = ProductHandler.class, beanMethod = "getAll"),
      @RouterOperation(path = "/api/products/search", method = org.springframework.web.bind.annotation.RequestMethod.GET, beanClass = ProductHandler.class, beanMethod = "search"),
      @RouterOperation(path = "/api/products/{id}", method = org.springframework.web.bind.annotation.RequestMethod.GET, beanClass = ProductHandler.class, beanMethod = "getById"),
      @RouterOperation(path = "/api/products", method = org.springframework.web.bind.annotation.RequestMethod.POST, beanClass = ProductHandler.class, beanMethod = "create"),
      @RouterOperation(path = "/api/products/{id}", method = org.springframework.web.bind.annotation.RequestMethod.PUT, beanClass = ProductHandler.class, beanMethod = "update"),
      @RouterOperation(path = "/api/products/{id}", method = org.springframework.web.bind.annotation.RequestMethod.DELETE, beanClass = ProductHandler.class, beanMethod = "delete")
  })
  public RouterFunction<ServerResponse> productRoutes(ProductHandler handler) {
    return route()
        .GET("/api/products", accept(MediaType.APPLICATION_JSON), handler::getAll)
        .GET("/api/products/search", accept(MediaType.APPLICATION_JSON), handler::search)
        .GET("/api/products/{id}", accept(MediaType.APPLICATION_JSON), handler::getById)
        .POST("/api/products", contentType(MediaType.APPLICATION_JSON), handler::create)
        .PUT("/api/products/{id}", contentType(MediaType.APPLICATION_JSON), handler::update)
        .DELETE("/api/products/{id}", handler::delete)
        .build();
  }
}
```

Important: when generating SpringDoc metadata for functional endpoints, always set the HTTP `method` explicitly on every `@RouterOperation`, including `GET` routes. Leaving `GET` operations without an explicit method can cause `/api-docs` generation to fail with HTTP 500 on some SpringDoc versions.

---

### 11. Global Error Handler

Generate a `@Component @Order(-2)` class implementing `ErrorWebExceptionHandler` that maps:
- `ServerWebInputException` -> 400
- `ResponseStatusException` -> its status code
- `NumberFormatException` -> 400
- All others -> 500

Returns JSON `ErrorResponse` bodies.

### 12. Configuration Classes

Generate `OpenApiConfig` (Swagger metadata), `R2dbcConfig` (`@EnableR2dbcAuditing` + schema initializer), and optionally `SecurityConfig` (JWT resource server) as `@Configuration` classes.

### 13. Docker Support

For Dockerfile and docker-compose.yml, use the `/add-dockerfile` prompt or generate:
- Dockerfile: use a multi-stage build. In the build stage, use `eclipse-temurin:21-jdk-alpine`, copy `.mvn`, `mvnw`, `pom.xml`, and `src`, then run `./mvnw -DskipTests package`. In the runtime stage, use `eclipse-temurin:21-jre-alpine`, copy the built JAR, and expose 8080. Do not require a prebuilt local `target/*.jar` before `docker build`.
- docker-compose: app service + PostgreSQL 16 with R2DBC env vars, healthcheck, and volume

Important: `docker compose up --build` should work from a clean checkout without any manual Maven packaging step.

---

## Execution and Completion Gates

Treat the following as required validation steps, not optional follow-up:

1. Generate all project files, including wrapper files if Maven is used.
2. Build and test the generated project using the wrapper:
    - Maven: `./mvnw test`
    - Gradle: `./gradlew test`
3. Start the application locally using the generated wrapper and the selected Java version.
4. Run smoke tests against the live app with `curl` or equivalent:
    - `GET /api-docs` must return `200 OK`
    - Swagger UI path must respond successfully (`/swagger-ui.html` or its redirect target)
    - `GET /api/products` must return `200 OK`
    - `POST /api/products` with a valid payload must return `201 Created`
    - `POST /api/products` with an invalid payload must return `400 Bad Request` with structured field errors
    - `GET /api/products/{missingId}` must return `404 Not Found`
5. If Docker support is included and Docker is available, validate `docker compose up --build` or an equivalent clean-checkout Docker path.
6. If any validation step fails, fix the scaffold and rerun the failed checks before considering the task complete.

The final handoff must explicitly report:

- the commands that were executed
- the JDK/version used for validation
- the endpoints that were smoke-tested
- whether Docker was validated or skipped, and why
- any remaining environment-specific blockers

---

## Tests

### Integration Test (WebTestClient)

```java
package com.example.webfluxapi.product;

import com.example.webfluxapi.product.model.CreateProductRequest;
import com.example.webfluxapi.product.model.ProductResponse;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.reactive.AutoConfigureWebTestClient;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.web.reactive.server.WebTestClient;
import java.math.BigDecimal;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWebTestClient
class ProductRouterIntegrationTest {

  @Autowired private WebTestClient webTestClient;

  @Test
  void shouldCreateAndRetrieveProduct() {
    var request = new CreateProductRequest("Integration Test Product",
        "Created during integration test", new BigDecimal("29.99"), "Testing", true);

    ProductResponse created = webTestClient.post().uri("/api/products")
        .contentType(MediaType.APPLICATION_JSON).bodyValue(request)
        .exchange().expectStatus().isCreated()
        .expectBody(ProductResponse.class).returnResult().getResponseBody();

    assertThat(created).isNotNull();
    assertThat(created.name()).isEqualTo("Integration Test Product");

    webTestClient.get().uri("/api/products/{id}", created.id())
        .accept(MediaType.APPLICATION_JSON).exchange()
        .expectStatus().isOk()
        .expectBody(ProductResponse.class)
        .value(product -> {
          assertThat(product.name()).isEqualTo("Integration Test Product");
          assertThat(product.price()).isEqualByComparingTo(new BigDecimal("29.99"));
        });
  }

  @Test
  void shouldReturn404ForMissingProduct() {
    webTestClient.get().uri("/api/products/{id}", 99999)
        .accept(MediaType.APPLICATION_JSON).exchange()
        .expectStatus().isNotFound();
  }

  @Test
  void shouldRejectInvalidProduct() {
    var invalid = new CreateProductRequest("", null, null, null, null);
    webTestClient.post().uri("/api/products")
        .contentType(MediaType.APPLICATION_JSON).bodyValue(invalid)
        .exchange().expectStatus().isBadRequest();
  }

  @Test
  void shouldServeOpenApiDocs() {
    webTestClient.get().uri("/api-docs")
        .accept(MediaType.APPLICATION_JSON).exchange()
        .expectStatus().isOk()
        .expectBody()
        .jsonPath("$.paths['/api/products']").exists()
        .jsonPath("$.paths['/api/products/{id}']").exists();
  }
}
```

---

## Best Practices

- **Use `Mono` and `Flux` everywhere** -- never call `.block()` on the event loop.
- **Use `switchIfEmpty`** to handle "not found" scenarios reactively instead of null checks.
- **Return `ServerResponse`** using the builder pattern for consistent HTTP responses.
- **Use records for DTOs** -- immutable, concise, and ideal for request/response models.
- **Validate reactively** -- pipe validation through `Mono` operators so errors propagate naturally.
- **Use R2DBC** -- never mix a blocking JDBC driver into a WebFlux application.
- **Test with `WebTestClient`** -- purpose-built for testing reactive endpoints.
- **Document with SpringDoc** -- `springdoc-openapi-starter-webflux-ui` generates Swagger UI at `/swagger-ui.html`.
- **Protect the docs route with a regression test** -- generated projects should assert that `/api-docs` returns `200 OK` and includes the expected product paths.
- **Use wrapper-based validation** -- build, test, and run the generated project with `./mvnw` or `./gradlew`, not only with globally installed tooling.
- **Do not claim success without execution** -- code generation alone is insufficient; the scaffold must pass live smoke tests before handoff.
