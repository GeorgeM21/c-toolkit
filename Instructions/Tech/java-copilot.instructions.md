---
applyTo: '**/*.{java,xml,gradle,properties,yml,yaml}'
description: "Java/Spring Boot-specific coding standards and best practices"
---

# Java/Spring Boot Development Best Practices

These guidelines extend the general standards in `Instructions/copilot-instructions.md`.
If a Java-specific rule conflicts with a general rule, prefer the Java rule.

## Language Features
- Use Java 21+ features (records, sealed classes, pattern matching, text blocks, virtual threads)
- Prefer `var` for local variables where type is obvious
- Use `Optional<T>` to represent nullable values, never return null from methods
- Use records for DTOs and immutable data
- Prefer sealed interfaces/classes for restricted type hierarchies

## Dependency Injection
- Use constructor injection (avoid field injection with @Autowired)
- Register beans via @Configuration classes or component scanning
- Use appropriate scopes (@Scope): singleton (default), prototype, request, session
- Prefer interface-based injection

## Async/Reactive Patterns
- Use `CompletableFuture<T>` for async operations in imperative code
- Use Project Reactor (`Mono<T>`, `Flux<T>`) for reactive code
- Use virtual threads (Java 21+) for blocking I/O operations
- Use `@Async` with proper executor configuration
- Always handle exceptions in async chains

## Spring Data JPA / Hibernate
- Use Flyway or Liquibase for schema migrations
- Use `@Transactional(readOnly = true)` for read-only queries
- Avoid lazy loading in REST controllers (use DTOs with projections)
- Use query projections (interfaces or records) to select only needed columns
- Avoid N+1 queries — use `@EntityGraph` or `JOIN FETCH`

## Spring Boot / Spring MVC Specifics
- Use `@RestController` for REST API endpoints
- Use `@Valid` / `@Validated` with Jakarta Bean Validation annotations (`@NotNull`, `@Size`, etc.)
- Use `ResponseEntity<T>` for typed responses with status codes
- Use `@ControllerAdvice` with `@ExceptionHandler` for global exception handling
- Use `@ConfigurationProperties` for typed configuration binding

## Testing in Java
- Use JUnit 5 (Jupiter) as the test framework
- Use AssertJ for fluent, readable assertions
- Use Mockito for mocking dependencies
- Use `@SpringBootTest` with `MockMvc` or `TestRestTemplate` for integration testing
- Use Testcontainers for database integration tests

## Dependency Management (Maven/Gradle)
- Use Spring Boot BOM for dependency version management
- Use Maven wrapper (`mvnw`) or Gradle wrapper (`gradlew`)
- Keep dependencies up-to-date with `mvn versions:display-dependency-updates` or Dependabot
- Review package vulnerabilities with `mvn dependency-check:check` (OWASP plugin)

## Performance
- Use `StringBuilder` for string concatenation in loops
- Use primitive types where possible, avoid unnecessary boxing
- Profile with JMH (Java Microbenchmark Harness) for micro-benchmarks
- Use connection pooling (HikariCP, default in Spring Boot)
- Enable GraalVM native image for serverless deployments when appropriate

## Related Instructions
- **General practices**: See root `copilot-instructions.md`
