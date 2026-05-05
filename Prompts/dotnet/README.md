# Cognizant GenAI Dev Toolkit - Prompts

This folder contains reusable prompt templates for common development scenarios. These prompts are designed to be used with GitHub Copilot Chat and can be fetched locally to your `.github/prompts` folder using our MCP server.

## 🚀 Quick Start

1. Copy the desired prompt file to your project's `.github/prompts/` folder
2. In VS Code with GitHub Copilot, type `/` followed by the prompt name
3. The prompt will appear and guide you through the task

```
/new-onion-api
/add-unit-tests
/create-readme
```

### Using with MCP Server

If you have the Cognizant MCP Server configured, you can use #get_cognizant_prompts or #get_cognizant_prompt_content to query or see the prompt content. 
Fetch prompt remains in the todo list.

## 📚 Available Prompts

### 🏗️ Project Scaffolding

| Prompt | Usage | Description |
|--------|-------|-------------|
| [new-onion-api.prompt.md](new-onion-api.prompt.md) | `/new-onion-api` | Scaffold a .NET Web API using Onion/Clean Architecture with Core, Application, Infrastructure, and Presentation layers |
| [new-cqrs-api.prompt.md](new-cqrs-api.prompt.md) | `/new-cqrs-api` | Scaffold a .NET Web API using CQRS pattern with MediatR, including Commands, Queries, and Pipeline Behaviors |
| [new-blazor-app.prompt.md](new-blazor-app.prompt.md) | `/new-blazor-app` | Scaffold a Blazor application (Server, WebAssembly, or Hybrid) with component architecture and state management |
| [new-minimal-api.prompt.md](new-minimal-api.prompt.md) | `/new-minimal-api` | Scaffold a .NET Minimal API with vertical slices, OpenAPI documentation, and validation |

### 📝 Documentation

| Prompt | Usage | Description |
|--------|-------|-------------|
| [create-readme.prompt.md](create-readme.prompt.md) | `/create-readme` | Generate a comprehensive README.md for any project (works for .NET, JavaScript, Python, etc.) |
| [create-adr.prompt.md](create-adr.prompt.md) | `/create-adr` | Create an Architecture Decision Record (ADR) using MADR format |

### ✅ Testing

| Prompt | Usage | Description |
|--------|-------|-------------|
| [add-unit-tests.prompt.md](add-unit-tests.prompt.md) | `/add-unit-tests` | Generate unit tests with xUnit, Moq, FluentAssertions, and AutoFixture |
| [add-integration-tests.prompt.md](add-integration-tests.prompt.md) | `/add-integration-tests` | Generate integration tests using WebApplicationFactory, Respawn, and test containers |

### 🔧 Code Quality

| Prompt | Usage | Description |
|--------|-------|-------------|
| [apply-solid-principles.prompt.md](apply-solid-principles.prompt.md) | `/apply-solid-principles` | Analyze and refactor code to follow SOLID principles with examples |
| [apply-design-patterns.prompt.md](apply-design-patterns.prompt.md) | `/apply-design-patterns` | Apply appropriate GoF design patterns (Factory, Strategy, Decorator, etc.) |

### 🚀 DevOps

| Prompt | Usage | Description |
|--------|-------|-------------|
| [add-dockerfile.prompt.md](add-dockerfile.prompt.md) | `/add-dockerfile` | Add optimized multi-stage Dockerfile and docker-compose for .NET applications |
| [add-github-actions.prompt.md](add-github-actions.prompt.md) | `/add-github-actions` | Add CI/CD pipeline with build, test, security scanning, and deployment |

## 🎯 Use Cases by Role

### Junior Developers
- `/new-minimal-api` - Start with simpler API structure
- `/add-unit-tests` - Learn testing patterns
- `/apply-solid-principles` - Understand SOLID with examples

### Mid-Level Developers
- `/new-onion-api` - Build properly architected applications
- `/add-integration-tests` - Test complete workflows
- `/apply-design-patterns` - Apply appropriate patterns

### Senior Developers / Architects
- `/new-cqrs-api` - Implement advanced patterns
- `/create-adr` - Document architectural decisions
- `/add-github-actions` - Set up complete CI/CD

## 🤖 Custom Agents Integration

These prompts are designed to work with the **custom agents** from this toolkit for enhanced capabilities. Each prompt references a specific agent in its frontmatter:

| Prompt Category | Custom Agent | Purpose |
|-----------------|--------------|---------|
| Project Scaffolding | `Feature-Scaffolder` | Professional scaffolding with buildable, testable code |
| Documentation | `code-documentation.agent` | Clear, maintainable documentation |
| Testing | `unit-testing-generator.agent` | High-quality tests with AAA structure |
| Code Quality | `Code-Reviewer` | Refactoring and code analysis |

### How Agent Fallback Works

```
┌─────────────────────────────────────────────────────────┐
│ Developer runs /new-onion-api prompt                    │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│ Prompt specifies: agent: 'Feature-Scaffolder'           │
└─────────────────────────────────────────────────────────┘
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
┌─────────────────────────┐ ┌─────────────────────────────┐
│ Agent EXISTS in         │ │ Agent NOT in                │
│ .github/agents/         │ │ .github/agents/             │
└─────────────────────────┘ └─────────────────────────────┘
              │                       │
              ▼                       ▼
┌─────────────────────────┐ ┌─────────────────────────────┐
│ ✅ Uses custom agent     │ │ ⚠️ Falls back to default    │
│ with specialized        │ │ Copilot 'agent'             │
│ capabilities            │ │ (prompt still works!)       │
└─────────────────────────┘ └─────────────────────────────┘
```

### Setting Up Custom Agents

For the best experience, copy the toolkit agents to your project:

```bash
# Using MCP Server (recommended)
# In Copilot Chat: use #fetch_cognizant_agent to download agents

# Manual setup
cp -r GenAI-Dev-Toolkit/Agents/*.agent.md your-project/.github/agents/
```

> [!TIP]
> You'll see a VS Code warning "Unknown agent 'XYZ'" if the agent isn't installed locally. This is expected - the prompt will still work with the default Copilot agent.

## 📋 Prompt Format

The prompt structure guideline has been extracted to a dedicated file for easier reuse:

- [PROMPT-CREATION-STANDARD.md](../PROMPT-CREATION-STANDARD.md)

## 🤝 Contributing

When adding new prompts:

1. **Follow naming convention**: `action-subject.prompt.md`
   - Good: `add-unit-tests.prompt.md`, `new-blazor-app.prompt.md`
   - Bad: `unitTests.prompt.md`, `blazor.prompt.md`

   See [PROMPT-CREATION-STANDARD.md](../PROMPT-CREATION-STANDARD.md) for the complete format and checklist.

2. **Include these sections**:
   - Role definition
   - Task description
   - Configuration options
   - Implementation examples
   - Best practices

3. **Add to this README** in the appropriate category

4. **Test the prompt** with GitHub Copilot before committing

## 📄 License

These prompts are part of the Cognizant GenAI Dev Toolkit.
