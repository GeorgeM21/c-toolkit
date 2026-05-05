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

- Do NOT overuse emojis
- Do NOT include LICENSE, CONTRIBUTING, CHANGELOG sections (use dedicated files)
- Keep the README concise and focused
- Include working code examples
- Add screenshots or diagrams where helpful
- Ensure all links are valid

### Example Badge Section

```markdown
[![Build Status](https://github.com/org/repo/workflows/CI/badge.svg)](https://github.com/org/repo/actions)
[![Maven Central](https://img.shields.io/maven-central/v/com.example/project-name.svg)](https://search.maven.org/artifact/com.example/project-name)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
```

## Technology-Specific Sections

### For Java/Spring Boot Projects

```markdown
## Requirements

- Java 21 SDK or later
- Maven 3.9+ (or use the included Maven Wrapper)
- Docker (optional, for containerized dependencies)

## Quick Start

\`\`\`bash
# Clone the repository
git clone https://github.com/org/repo.git

# Navigate to the project
cd repo

# Run the application using Maven Wrapper
./mvnw spring-boot:run

# Or build and run the JAR
./mvnw clean package
java -jar target/project-name-0.0.1-SNAPSHOT.jar
\`\`\`

## Configuration

Configure the application using `application.yml` or environment variables:

| Variable | Description | Default |
|----------|-------------|---------|
| `SPRING_DATASOURCE_URL` | Database connection URL | `jdbc:h2:mem:testdb` |
| `SPRING_PROFILES_ACTIVE` | Active Spring profile | `default` |
| `SERVER_PORT` | Application port | `8080` |
```

> **Note:** For Frontend projects, include Node.js/npm prerequisites, `npm install`, `npm run dev`, and `npm run build` commands. For API projects, include an endpoint summary table and link to Swagger UI at `/swagger-ui.html`.

## Execution

1. Scan all project files to understand the technology stack
2. Identify the project's purpose and main features
3. Determine prerequisites from project files (pom.xml, build.gradle, package.json, etc.)
4. Extract configuration options from config files
5. Generate appropriate sections based on project type
6. Include practical, working examples
7. Ensure all paths and commands are correct for the project
