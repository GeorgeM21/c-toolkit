---
agent: 'Code-Documentation'
description: 'Create an Architecture Decision Record (ADR) for documenting technical decisions'
# Note: Uses code-documentation.agent if available in .github/agents/, otherwise falls back to default Copilot agent
---

# Create Architecture Decision Record

## Task

Create an Architecture Decision Record (ADR) that documents a specific architectural or technical decision. The ADR should be clear, concise, and provide enough context for future readers to understand why the decision was made.

## Configuration

| Setting | Options | Default |
|---------|---------|---------|
| ADR Number | Sequential number | Auto-detect next number |
| Template Style | MADR, Nygard, Custom | MADR |
| Output Path | Path for ADR files | docs/architecture/decisions/ |

## ADR Template (MADR Format)

```markdown
# ADR-{NUMBER}: {TITLE}

## Status

{Proposed | Accepted | Deprecated | Superseded by ADR-XXX}

## Date

{YYYY-MM-DD}

## Context

{Describe the issue motivating this decision. What is the problem we are trying to solve? What are the forces at play (technical, political, social)? What constraints do we have?}

## Decision Drivers

- {Driver 1: e.g., Performance requirements}
- {Driver 2: e.g., Team expertise}
- {Driver 3: e.g., Time constraints}
- {Driver 4: e.g., Cost considerations}

## Considered Options

### Option 1: {Name}

{Brief description of this option}

**Pros:**
- Pro 1
- Pro 2

**Cons:**
- Con 1
- Con 2

### Option 2: {Name}

{Brief description of this option}

**Pros:**
- Pro 1
- Pro 2

**Cons:**
- Con 1
- Con 2

### Option 3: {Name}

{Brief description of this option}

**Pros:**
- Pro 1
- Pro 2

**Cons:**
- Con 1
- Con 2

## Decision

{State the decision that was made. Use active voice: "We will use..."}

## Rationale

{Explain why this decision was made. Reference the decision drivers and explain how this option best addresses them.}

## Consequences

### Positive

- {Positive consequence 1}
- {Positive consequence 2}

### Negative

- {Negative consequence 1}
- {Negative consequence 2}

### Neutral

- {Neutral consequence/observation}

## Implementation Notes

{Any specific implementation details, migration steps, or technical notes}

## References

- {Link to relevant documentation}
- {Link to related ADRs}
- {Link to external resources}
```

## Common ADR Topics for .NET Projects

### Architecture Pattern Selection

```markdown
# ADR-001: Adopt Clean Architecture for Backend Services

## Context
We need to establish an architectural pattern for our new microservices that promotes maintainability, testability, and separation of concerns.

## Decision Drivers
- Need for clear separation between business logic and infrastructure
- Requirement for high test coverage
- Team familiarity with the pattern
- Long-term maintainability

## Considered Options
1. **Clean/Onion Architecture** - Dependency inversion with layers
2. **Vertical Slice Architecture** - Feature-based organization
3. **Traditional N-Tier** - Simple layered approach

## Decision
We will adopt Clean Architecture with the following layers:
- Core (Domain entities, interfaces)
- Application (Use cases, DTOs)
- Infrastructure (Data access, external services)
- Presentation (API controllers)
```

### Database Technology

```markdown
# ADR-002: Use PostgreSQL as Primary Database

## Context
We need to select a relational database for our application that supports our scalability and feature requirements.

## Decision Drivers
- Need for JSON/JSONB support for flexible data
- Cost (avoiding SQL Server licensing)
- Team expertise
- Cloud provider support (Azure, AWS, GCP)
- Performance requirements

## Decision
We will use PostgreSQL with Entity Framework Core as the ORM.
```

### API Design

```markdown
# ADR-003: Adopt Minimal APIs for New Endpoints

## Context
We are building new API endpoints and need to decide between Controllers and Minimal APIs.

## Decision Drivers
- Developer productivity
- Performance
- Consistency with existing code
- Learning curve

## Decision
New endpoints will use Minimal APIs with vertical slice organization.
Existing controller-based endpoints will be migrated gradually.
```

## Execution Steps

1. **Gather Information**
   - Ask the user what decision needs to be documented
   - Understand the context and constraints
   - Identify the options that were considered

2. **Check Existing ADRs**
   - Look for existing ADR directory structure
   - Determine the next ADR number
   - Check for related decisions

3. **Generate the ADR**
   - Use the MADR template
   - Fill in all sections with provided information
   - Ensure the rationale clearly explains the "why"

4. **Create the File**
   - Save to the appropriate location
   - Use naming convention: `adr-{number}-{kebab-case-title}.md`

## Best Practices

- ✅ Keep ADRs immutable - create new ones to supersede old decisions
- ✅ Focus on the "why" not just the "what"
- ✅ Include enough context for future readers
- ✅ Link related ADRs together
- ✅ Date all decisions
- ✅ Keep consequences section honest (include negatives)
- ✅ Use active voice in the Decision section
- ❌ Don't make ADRs too long - be concise
- ❌ Don't skip the Consequences section
- ❌ Don't document trivial decisions
