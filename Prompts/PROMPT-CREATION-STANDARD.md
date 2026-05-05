# Prompt Creation Standard

Use this standard when creating new prompt files in this toolkit.

## File Naming Convention

- Format: `action-subject.prompt.md`
- Good examples:
  - `add-unit-tests.prompt.md`
  - `new-blazor-app.prompt.md`
  - `create-readme.prompt.md`
- Avoid:
  - `unitTests.prompt.md`
  - `blazor.prompt.md`

## Required Prompt File Structure

Every prompt file must start with YAML frontmatter, followed by structured Markdown sections.

### Frontmatter (Required)

```yaml
---
agent: 'Feature-Scaffolder'
description: 'One-line summary of what the prompt does'
# Note: Uses <agent> if available in .github/agents/, otherwise falls back to default Copilot agent
---
```

### Body (Required Core Sections)

1. `# <Prompt Title>`
2. `## Task`
3. `## Configuration` (if the task has choices/parameters)

### Body (Common Optional Sections)

Use sections that fit your scenario. Common patterns in this repo:

- `## Solution Structure` / `## Project Structure`
- `## Guidelines`
- `## Execution` / `## Execution Steps`
- `## Best Practices to Follow`
- `## Required Packages` (if needed)
- Scenario-specific deep dives (for example, `## Layer Details`, `## Test Class Examples`)

## Prompt Template (Copy/Paste Starter)

```markdown
---
agent: '<Agent-Name>'
description: '<Brief description>'
# Note: Uses <matching-agent> if available in .github/agents/, otherwise falls back to default Copilot agent
---

# <Prompt Title>

## Task

Describe exactly what should be produced, quality expectations, and constraints.

## Configuration

Before execution, confirm these settings with the user:

| Setting | Options | Default |
|---------|---------|---------|
| Example | A, B, C | B |

## Implementation

Provide concrete instructions, structure, and examples relevant to the use case.

## Execution Steps

1. Step one
2. Step two
3. Step three

## Best Practices to Follow

- ✅ Practice 1
- ✅ Practice 2
- ❌ Anti-pattern to avoid
```

## Writing Guidelines

- Keep language directive and explicit; avoid vague wording.
- Prefer tables for configuration options.
- Include realistic code or structure examples when useful.
- Keep defaults practical and production-oriented.
- Ensure technology/tool versions are current for the prompt’s domain.

## Agent Guidance

- Use the most relevant custom agent in frontmatter (`agent:`).
- Keep the fallback note comment in frontmatter.
- Ensure the selected agent aligns with prompt intent:
  - Scaffolding → `Feature-Scaffolder`
  - Documentation → `Code-Documentation`
  - Testing → `Unit-Testing-Generator`
  - Refactoring/analysis → `Code-Reviewer`

## Quality Checklist

Before adding a new prompt:

- Naming follows `action-subject.prompt.md`
- Frontmatter includes both `agent` and `description`
- Task is unambiguous and outcome-oriented
- Configuration options are explicit (when applicable)
- Execution steps are ordered and actionable
- Examples are valid and consistent with chosen stack
- Prompt is added to [README.md](README.md) in the correct category
- Prompt is tested in GitHub Copilot Chat
