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

## Example: ADR-001 (abbreviated)

```markdown
# ADR-001: Adopt Hexagonal Architecture for Spring Boot Microservices

## Status
Accepted

## Date
2024-06-15

## Context
We need to establish an architectural pattern for our new microservices that promotes maintainability, testability, and separation of concerns.

## Decision Drivers
- Need for clear separation between business logic and infrastructure
- Requirement for high test coverage
- Long-term maintainability

## Decision
We will adopt Hexagonal Architecture with Domain, Application, Infrastructure, and Presentation layers.

## Consequences
### Positive
- Clear boundaries enable independent testing of business logic
- Infrastructure components can be swapped without affecting the domain
### Negative
- Higher initial setup complexity compared to layered architecture
```

Other common ADR topics for Java/Spring projects include database technology selection, API design choices (MVC vs. WebFlux), messaging strategies, and authentication approaches. Follow the same MADR template structure for each.

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

- Keep ADRs immutable -- create new ones to supersede old decisions
- Focus on the "why" not just the "what"
- Include enough context for future readers
- Link related ADRs together
- Date all decisions
- Keep consequences section honest (include negatives)
- Use active voice in the Decision section
- Don't make ADRs too long -- be concise
- Don't document trivial decisions
