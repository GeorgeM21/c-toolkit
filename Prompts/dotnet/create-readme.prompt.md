---
agent: 'Code-Documentation'
description: 'Generate a comprehensive README.md file for any project'
# Note: Uses code-documentation.agent if available in .github/agents/, otherwise falls back to default Copilot agent
---

# Create README

## Task

1. Analyze the entire project and workspace structure
2. Create a comprehensive and well-structured README.md file
3. Ensure the README is appealing, informative, and easy to read

## Guidelines

### Structure

Include these sections as appropriate:

```markdown
# Project Name

Brief, compelling description (1-2 sentences)

## Features

- Key feature 1
- Key feature 2
- Key feature 3

## Prerequisites

List required tools and versions

## Getting Started

### Installation

Step-by-step installation instructions

### Configuration

Configuration options and environment variables

### Running the Application

Commands to run the app locally

## Usage

Basic usage examples with code snippets

## API Reference (if applicable)

Brief API overview or link to API docs

## Project Structure

High-level folder structure explanation

## Development

### Building

Build instructions

### Testing

How to run tests

### Contributing

Brief contribution guidelines or link to CONTRIBUTING.md

## Deployment

Deployment instructions or link to deployment docs

## Troubleshooting

Common issues and solutions

## License

License information
```

### Formatting Guidelines

- Use GitHub Flavored Markdown (GFM)
- Use admonitions for important notes:
  ```markdown
  > [!NOTE]
  > Useful information
  
  > [!TIP]
  > Helpful advice
  
  > [!IMPORTANT]
  > Key information
  
  > [!WARNING]
  > Potential issues
  
  > [!CAUTION]
  > Dangerous actions
  ```
- Include badges for build status, version, license
- Use code blocks with language identifiers
- Keep content concise and scannable
- Use relative links for internal documentation
- Include a logo or icon if available

### Content Rules

- ❌ Do NOT overuse emojis
- ❌ Do NOT include LICENSE, CONTRIBUTING, CHANGELOG sections (use dedicated files)
- ✅ Keep the README concise and focused
- ✅ Include working code examples
- ✅ Add screenshots or diagrams where helpful
- ✅ Ensure all links are valid

### Example Badge Section

```markdown
[![Build Status](https://github.com/org/repo/workflows/CI/badge.svg)](https://github.com/org/repo/actions)
[![NuGet](https://img.shields.io/nuget/v/PackageName.svg)](https://www.nuget.org/packages/PackageName)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
```

## Technology-Specific Sections

### For .NET Projects

```markdown
## Requirements

- .NET 9.0 SDK or later
- Visual Studio 2022 / VS Code / JetBrains Rider

## Quick Start

\`\`\`bash
# Clone the repository
git clone https://github.com/org/repo.git

# Navigate to the project
cd repo

# Restore dependencies
dotnet restore

# Run the application
dotnet run --project src/ProjectName.Api
\`\`\`

## Configuration

Configure the application using `appsettings.json` or environment variables:

| Variable | Description | Default |
|----------|-------------|---------|
| `ConnectionStrings__Default` | Database connection string | SQLite |
| `Jwt__Secret` | JWT signing key | - |
\`\`\`

### For Frontend Projects

```markdown
## Requirements

- Node.js 20+ and npm/pnpm/yarn
- Modern browser (Chrome, Firefox, Safari, Edge)

## Quick Start

\`\`\`bash
# Install dependencies
npm install

# Start development server
npm run dev

# Build for production
npm run build
\`\`\`
```

### For API Projects

```markdown
## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/products` | Get all products |
| GET | `/api/products/{id}` | Get product by ID |
| POST | `/api/products` | Create a product |
| PUT | `/api/products/{id}` | Update a product |
| DELETE | `/api/products/{id}` | Delete a product |

For detailed API documentation, run the application and navigate to `/swagger`.
```

## Execution

1. Scan all project files to understand the technology stack
2. Identify the project's purpose and main features
3. Determine prerequisites from project files (csproj, package.json, etc.)
4. Extract configuration options from config files
5. Generate appropriate sections based on project type
6. Include practical, working examples
7. Ensure all paths and commands are correct for the project
