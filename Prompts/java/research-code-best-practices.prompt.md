---
agent: 'Web-Search-Researcher'
description: 'Analyze selected Java code or configuration and research current best practices from authoritative sources'
# Note: Uses Web-Search-Researcher agent if available in .github/agents/, otherwise falls back to default Copilot agent
---

# Research Code Best Practices (Java)

## Task

Analyze the selected Java functionality, source code, configuration, build file, or architectural slice, then use live web research to identify current best practices that apply to it.

Start by reading the provided code or files and summarizing what the current implementation is doing before searching the web. After that, research best practices using authoritative and up-to-date sources such as official Java, Spring, Jakarta, Maven, Gradle, testing framework, security, cloud, or vendor documentation. Use reputable technical sources only when official documentation does not fully answer the question.

The goal is not to produce generic advice. The goal is to compare the actual implementation against current best practices and provide a practical, evidence-based improvement report.

## Configuration

Before execution, confirm or infer these settings from the selected files and user request:

| Setting | Options | Default |
|---------|---------|---------|
| Review Scope | Single class, package, feature flow, configuration file, build setup | Single class |
| Focus Area | General best practices, security, performance, maintainability, testing, architecture, configuration | General best practices |
| Source Priority | Official docs first, official + expert sources, broad web search | Official docs first |
| Recommendation Depth | Quick review, standard review, deep review | Standard review |
| Output Style | Gap analysis only, recommendations with examples, recommendations with refactoring sketch | Recommendations with examples |

## Guidelines

- Read the selected code, configuration, or files before performing any search.
- Infer the framework and runtime context from the codebase when possible, for example Spring Boot, Jakarta EE, Maven, Gradle, JUnit, Testcontainers, Hibernate, or Kafka.
- Prefer official documentation, release notes, vendor guidance, and standards bodies for best-practice claims.
- Include publication dates, version context, and source links when they affect the recommendation.
- Separate universally accepted best practices from version-specific or opinionated recommendations.
- Do not recommend patterns or dependencies that conflict with the existing stack unless there is a clear benefit and the tradeoff is explained.
- Call out when the current implementation is already aligned with best practices instead of inventing changes.
- Push back on weak or trendy advice if the sources do not justify it.

## Execution Steps

1. Read the selected code, configuration, or files and summarize the current behavior, purpose, and technology context.
2. Identify the key review dimensions based on the material, such as error handling, dependency injection, validation, concurrency, transaction management, testing, observability, security, configuration hygiene, or package structure.
3. Research current best practices using live web search, starting with official documentation and version-specific guidance.
4. Cross-check findings across multiple high-quality sources when the topic is sensitive, evolving, or potentially opinionated.
5. Compare the current implementation against the researched guidance and identify strengths, gaps, risks, and outdated patterns.
6. Produce prioritized recommendations tailored to the selected code, not generic framework summaries.
7. When useful, include small example snippets or refactoring sketches that show how a recommendation could be applied in this codebase.

## Best Practices to Follow

- Use authoritative sources first and provide direct links.
- Anchor every recommendation to the selected code or configuration.
- Prefer current Java and framework conventions over legacy patterns when the project version supports them.
- Highlight security, maintainability, and operational risks before stylistic suggestions.
- Distinguish between required fixes, recommended improvements, and optional refinements.
- Mention uncertainty explicitly when the selected code does not provide enough context.
- Avoid speculative rewrites and avoid suggesting architecture changes without evidence.

## Response Structure

Return the answer using this structure:

### 1. Current Implementation Summary
- What the selected code or configuration appears to do
- Technologies and versions detected or inferred
- Assumptions made due to missing context

### 2. Research Findings
- Best practice finding
- Why it matters here
- Source link and version/date context

### 3. Gap Analysis
- What is already good
- What is missing, risky, outdated, or inconsistent with best practices
- Severity or priority for each gap

### 4. Recommendations
- Concrete improvement
- Expected benefit
- Suggested implementation direction

### 5. Optional Example Changes
- Small code, config, or structure examples tailored to the selected code when they would help

### 6. Sources
- List all referenced links in a clean, scannable format
