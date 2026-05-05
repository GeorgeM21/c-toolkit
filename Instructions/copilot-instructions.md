---
applyTo: '**'
description: "General development best practices and workflow guidance"
priority: 1
---

# General Development Best Practices

## AI Behavior Summary
- Prefer clear, correct, minimal solutions over clever or complex ones.
- Keep changes small and focused, matching the existing style and patterns.
- When adding or changing non-trivial logic, also suggest or generate appropriate tests.
- Surface obvious security or performance issues you notice in the code you touch.

## Code Quality Standards

### Code Structure
 Keep methods/functions under 30 lines; if logic grows, extract helpers.
- Limit cyclomatic complexity to < 10 per method
- Maximum nesting depth: 3-4 levels
- Classes should be < 200 lines

### Naming Conventions
- Use descriptive, intention-revealing names
- Avoid abbreviations unless industry-standard
- Boolean variables/methods: use `is`, `has`, `can`, `should` prefixes
- Constants: UPPER_SNAKE_CASE
- Functions/methods: camelCase or PascalCase per language conventions

### Error Handling
- Always handle expected exceptions gracefully
- Log errors with sufficient context for debugging
- Never swallow exceptions silently
- Use specific exception types, not generic `Exception`

### Async/Await Best Practices
- Always use async APIs where available for I/O operations
- Avoid blocking async code with `.Wait()` or `.Result`
- Use `ConfigureAwait(false)` in library code
- Handle cancellation tokens properly

## Clean Architecture & Project Structure

### Architecture Principles
- **Dependency Rule**: Dependencies should point inward toward business logic
- **Domain-Centric**: Core business logic should be independent of frameworks, UI, and databases
- **Separation of Concerns**: Each layer has a clear responsibility and purpose
- **Testability**: Business logic should be easily testable without external dependencies
- **Independence**: Core domain should not depend on infrastructure details

### Standard Folder Structure
All projects should follow this generic folder structure:

```
ProjectRoot/
├── src/
│   ├── Api/                    # API/Web presentation layer
│   │   ├── Routes/             # API routes/endpoints/controllers
│   │   ├── Middleware/         # Custom middleware
│   │   ├── Handlers/           # Request handlers
│   │   ├── Models/             # DTOs, view models, request/response models
│   │   └── main.*              # Entry point (main.py, index.js, Program.cs, etc.)
│   ├── Core/                   # Core business logic (domain + application)
│   │   ├── Domain/             # Domain entities, value objects, aggregates
│   │   │   ├── Entities/       # Business entities
│   │   │   ├── ValueObjects/   # Immutable value objects
│   │   │   ├── Enums/          # Domain enumerations
│   │   │   └── Interfaces/     # Domain interfaces/contracts
│   │   └── Application/        # Application services, use cases
│   │       ├── Services/       # Application services
│   │       ├── UseCases/       # Use case implementations
│   │       ├── DTOs/           # Data transfer objects
│   │       ├── Interfaces/     # Application interfaces/ports
│   │       ├── Validators/     # Business validation logic
│   │       └── Mappers/        # Object mapping/transformation logic
│   ├── Infrastructure/         # External concerns implementation
│   │   ├── Database/           # Database access layer
│   │   │   ├── Connection/     # Database connection/session management
│   │   │   ├── Repositories/   # Repository implementations
│   │   │   ├── Models/         # ORM/database models
│   │   │   └── Migrations/     # Database migrations/schema changes
│   │   ├── ExternalServices/   # Third-party service integrations
│   │   ├── Messaging/          # Message queue implementations
│   │   ├── Cache/              # Caching implementations
│   │   └── Auth/               # Authentication/authorization implementations
│   ├── Shared/                 # Shared utilities and cross-cutting concerns
│   │   ├── Common/             # Common utilities
│   │   ├── Utils/              # Utility functions
│   │   ├── Constants/          # Application constants
│   │   ├── Helpers/            # Helper functions/classes
│   │   └── Exceptions/         # Custom exception/error classes
│   └── Jobs/                   # Background jobs, schedulers, workers
│       ├── Workers/            # Background service workers
│       ├── Tasks/              # Scheduled task definitions
│       └── Handlers/           # Event/message handlers
├── tests/
│   ├── Unit/                   # Unit tests
│   │   ├── Core/               # Core business logic tests
│   │   ├── Application/        # Application service tests
│   │   └── Api/                # API handler/route tests
│   ├── Integration/            # Integration tests
│   │   ├── Api/                # API integration tests
│   │   └── Infrastructure/     # Infrastructure tests
│   └── E2E/                    # End-to-end tests
│       └── Scenarios/          # Test scenarios
├── docs/                       # Documentation
│   ├── Architecture/           # Architecture decision records (ADRs)
│   ├── Api/                    # API documentation
│   └── Guides/                 # Development guides
├── config/                     # Configuration files
│   ├── Development/            # Development environment config
│   ├── Staging/                # Staging environment config
│   └── Production/             # Production environment config
└── scripts/                    # Build and deployment scripts
    ├── Build/                  # Build scripts
    ├── Deploy/                 # Deployment scripts
    └── Migrations/             # Migration scripts
```

### Layer Responsibilities

#### API Layer (`src/Api/`)
- Handle HTTP requests/responses
- Input validation and model binding
- Authentication and authorization checks
- Map between DTOs and domain models
- Return appropriate status codes
- No business logic should reside here

#### Core Layer (`src/Core/`)
**Domain (`src/Core/Domain/`):**
- Define business entities and rules
- Domain logic and invariants
- No dependencies on other layers
- Framework and technology-agnostic
- Pure business logic only

**Application (`src/Core/Application/`):**
- Orchestrate domain objects
- Implement use cases and business workflows
- Define interfaces/ports for infrastructure (dependency inversion)
- Transaction and state management
- Business validation and transformation
- Application-level error handling

#### Infrastructure Layer (`src/Infrastructure/`)
- Implement data access (repositories, DAOs)
- External API and service integrations
- File system and storage access
- Email, SMS, and notification services
- Caching mechanisms (Redis, Memcached, etc.)
- Message queue and event bus implementations
- Implements interfaces defined in Application layer

#### Shared Layer (`src/Shared/`)
- Cross-cutting concerns
- Utilities used across all layers
- Extension methods
- Common constants
- No business logic

#### Jobs Layer (`src/Jobs/`)
- Background processing
- Scheduled tasks
- Event handlers
- Message consumers
- Long-running operations

### Dependency Flow
```
API → Application → Domain
  ↓         ↓
Infrastructure
```

### Enforcement Rules
- **Core/Domain** must not reference any other layer or external libraries (except language standard library)
- **Application** can reference Domain only
- **Infrastructure** can reference Application and Domain (implements interfaces from Application)
- **API** can reference all layers but should primarily interact with Application
- **Jobs** follows same rules as API
- All layers can reference **Shared** (cross-cutting utilities)
- Use dependency injection/inversion of control to wire implementations

### Project Organization Guidelines
- One module/package per layer (or logical grouping)
- Keep modules focused and cohesive
- Use proper package/module hierarchy matching folder structure
- Place interfaces/contracts in the layer that defines them, implementations in appropriate layer
- Maintain consistent naming conventions across the structure
- Avoid circular dependencies between layers

### When Creating New Features
1. Start with domain entities and business rules in `src/Core/Domain/`
2. Define use cases and application services in `src/Core/Application/`
3. Define required interfaces/ports in `src/Core/Application/Interfaces/`
4. Create infrastructure implementations in `src/Infrastructure/`
5. Add API endpoints/routes in `src/Api/Routes/`
6. Add background processing in `src/Jobs/` if needed
7. Write tests for each layer in corresponding test directories

## Testing Standards

### Test Coverage
- Minimum 80% code coverage for business logic
- 100% coverage for critical paths (security, payments, data integrity)
- Test edge cases and boundary conditions
- Include negative test cases

### When Generating/Updating Code
- When you introduce or modify business logic, also propose the corresponding unit/integration tests.
- Follow the existing test framework and folder structure in the repository.

### Test Structure (AAA Pattern)
```
// Arrange: Set up test data and dependencies
// Act: Execute the code under test
// Assert: Verify the expected outcome
```

### Test Naming
- Descriptive names: `MethodName_Scenario_ExpectedBehavior`
- Example: `CreateUser_WithInvalidEmail_ThrowsValidationException`

## Security Best Practices

### Input Validation
- Validate all user input at entry points
- Sanitize data before use in queries, commands, or rendering
- Use allowlists over denylists
- Enforce type safety and constraints

### Secrets Management
- Never hardcode credentials, API keys, or secrets
- Use environment variables or secure vaults (Azure Key Vault, AWS Secrets Manager)
- Rotate secrets regularly
- Log secret access for audit trails

### Authentication & Authorization
- Implement authentication for all protected resources
- Use role-based or claims-based authorization
- Validate tokens on every request
- Implement proper session management

## Performance Best Practices

### Database Access
- Use connection pooling
- Implement pagination for large result sets
- Avoid N+1 query problems
- Use indexes for frequently queried columns
- Cache frequently accessed, rarely changed data

### API Design
- Implement retry logic with exponential backoff
- Use circuit breakers for external dependencies
- Set appropriate timeouts
- Implement rate limiting
- Return appropriate HTTP status codes

### Resource Management
- Dispose of resources properly (IDisposable pattern)
- Avoid memory leaks (unsubscribe from events, clear collections)
- Use object pooling for expensive objects
- Profile and optimize hot paths

## Documentation Standards

### Code Comments
- Comment **why**, not **what** (code should be self-explanatory)
- Use XML/JSDoc comments for public APIs
- Keep comments up-to-date with code changes
- Remove commented-out code before committing

### README Requirements
- Project purpose and description
- Prerequisites and dependencies
- Installation/setup instructions
- Usage examples
- Configuration options
- Contributing guidelines
- License information

## Version Control Best Practices

### Commit Messages
- Use conventional commit format: `type(scope): description`

### Change Grouping
- Group related code and test changes into a single suggestion.
- Avoid mixing large refactors with feature changes unless clearly necessary and well justified.
- Types: feat, fix, docs, style, refactor, test, chore
- Keep subject line under 50 characters
- Use imperative mood: "Add feature" not "Added feature"

### Branching Strategy
- Use feature branches for new work
- Keep main/master branch deployable
- Review code before merging
- Delete branches after merging

## Logging & Observability

### Logging Levels
- **ERROR**: Unhandled exceptions, critical failures
- **WARN**: Recoverable issues, degraded functionality
- **INFO**: Important business events, startup/shutdown
- **DEBUG**: Detailed diagnostic information
- **TRACE**: Very detailed diagnostic information

### What to Log
- Application startup/shutdown
- Configuration changes
- Authentication events (success/failure)
- Business-critical operations
- Performance metrics
- Error details with stack traces

### What NOT to Log
- Passwords or secrets
- Personal Identifiable Information (PII) without anonymization
- Credit card numbers
- Health records

## Dependency Management

### Package/Library Selection
- Prefer well-maintained, popular libraries
- Check for security vulnerabilities regularly
- Keep dependencies up-to-date
- Minimize dependency count
- Review licenses for compatibility

### Version Pinning
- Pin major versions, allow minor/patch updates
- Test dependency updates in non-production first
- Document breaking changes

## Code Review Standards

### What to Check
- Code correctness and logic
- Test coverage and quality
- Security vulnerabilities
- Performance implications
- Code readability and maintainability
- Adherence to style guides
- Documentation completeness

### Review Etiquette
- Be constructive and respectful
- Ask questions, don't make demands
- Suggest improvements with examples
- Approve when standards are met

## Workflow Integration

### Before Writing Code
1. Understand requirements fully
2. Design/plan the solution
3. Consider edge cases and failure modes
4. Identify required tests

### During Development
1. Write failing tests first (TDD preferred)
2. Implement minimal code to pass tests
3. Refactor for clarity and performance
4. Update documentation

### Before Committing
1. Run all tests locally
2. Fix linting/formatting issues
3. Review your own changes
4. Write descriptive commit message

### Related Custom Instructions
- **.NET-specific**: See `instructions/dotnet.copilot-instructions.md`
