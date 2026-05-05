---
name: feature-scaffolder.instructions
applyTo: '**'
description: "Systematic feature implementation workflow"
---

# Feature Scaffolder Workflow

## Input Expectations
- User story or feature description with acceptance criteria
- Target system (repo/project), technology stack, and relevant environments
- Non-functional constraints (performance, security, compliance, observability)
- Links to existing patterns, modules, or features to align with

## Fast Path (Minimal Feature Spike)
- Propose a high-level design: key components, data flow, and boundaries
- Scaffold minimal interfaces/endpoints and data models
- Provide a small, vertical slice that implements one core scenario end-to-end

## 1. Gather Requirements
- Clarify functional requirements, edge cases, and failure scenarios
- Identify entities, operations, and business rules
- Understand technology constraints and deployment targets
- Note recent architectural or domain decisions impacting the feature

## 2. Analyze Architecture
- Review existing code patterns, layering, and module boundaries
- Identify relevant database schema, data models, and integration points
- Examine API design patterns (REST, GraphQL, messaging, etc.)
- Ensure new feature aligns with current architectural style and conventions

## 3. Choose Path: Skeleton vs Full Implementation
- **Skeleton mode**: focus on interfaces, routes, DTOs, and test skeletons
- **Full implementation mode**: implement business logic, persistence, and tests
- Explicitly state which mode is being used and why

## 4. Scaffold Feature Structure
- Define entities, DTOs, and domain models following existing naming conventions
- Create controllers/handlers, services, and data access components as needed
- Establish validation, error handling, and authorization patterns
- Wire up configuration and dependency injection consistent with the existing project

## 5. Implement Feature Logic
- Implement endpoints or functions with the agreed behavior
- Integrate with existing data access layers and external services
- Handle filtering, paging, sorting, and edge cases where applicable
- Make assumptions explicit when requirements are ambiguous and propose clarifying questions

## 6. Add Quality Attributes
- Add or extend unit and integration tests for key scenarios
- Set up or update API documentation (Swagger/OpenAPI or equivalent)
- Add input validation, security controls, and authorization checks
- Implement structured logging and, where relevant, metrics or tracing
- When test coverage for the new or changed behavior is unclear or insufficient, invoke the @unit-testing-generator workflow to design and generate focused tests for critical paths and edge cases
- When generating new code as part of this workflow, follow the global code-generation guidelines from code-generation.instructions.md (compilable, secure, consistent with existing architecture and patterns)

## 7. Verification & Handoff
- Verify the feature works in isolation and alongside existing flows
- Suggest how to test it in relevant environments (dev, staging, production)
- Before considering the feature ready, invoke the @code-reviewer workflow to validate quality, maintainability, and architectural fit for the new or modified code
- When feature implementation surfaces ambiguous behavior, unexpected interactions, or suspected defects, invoke the @bug-detective workflow to systematically investigate and resolve them before sign-off
- Once the feature behavior and interfaces are stable, invoke the @code-documentation workflow to document APIs, complex logic, integration behavior, and any important operational considerations
