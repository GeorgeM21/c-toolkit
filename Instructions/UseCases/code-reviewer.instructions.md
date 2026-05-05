---
name: code-reviewer.instructions
applyTo: '**'
description: "Systematic code review workflow"
---

# Code Review Workflow

## Input Expectations
- Code diff, PR link, or list of files to review
- Scope of review (full, or focused on performance, security, style, etc.)
- Applicable coding standards, patterns, or style guides (or language defaults)
- Testing expectations (required suites, coverage goals, critical scenarios)

## Fast Path (Quick Review)
- Scan for obvious bugs, security risks, and major anti-patterns
- Report the top 3–5 findings with severity and category
- Suggest the most impactful next steps (tests, refactors, or design changes)

## 1. Gather Context
- Request code/PR and context (branch, files, standards, recent changes)
- Clarify review scope and priorities with the user
- Collect relevant documentation, architectural guidelines, and design decisions
- Consider available static analysis results (linters, SonarQube, etc.)

## 2. Readability & Clarity Assessment
- Evaluate naming conventions (classes, methods, variables)
- Check for clarity and intent revelation in code
- Identify confusing or ambiguous sections
- Review code formatting, indentation, and consistency
- Assess documentation quality (comments, docstrings, README updates)
- Verify adherence to language-specific idioms and conventions

## 3. Complexity & Maintainability Analysis
- Measure cyclomatic complexity and nesting depth
- Identify methods exceeding 50 lines or suspicious patterns
- Detect code duplication (DRY principle violations)
- Assess coupling between modules and dependencies
- Evaluate modularity and single-responsibility compliance
- Check for deprecated patterns or tech debt accumulation

## 4. Security & Performance Review
- Validate input validation and sanitization
- Check for hardcoded secrets or sensitive data exposure
- Assess authentication, authorization, and access control
- Profile algorithm efficiency and time/space complexity
- Identify N+1 queries, unnecessary iterations, or inefficient data structures
- Assess memory usage patterns and potential leaks

## 5. Testing & Architecture Assessment
- Assess unit test coverage and meaningful assertions
- Check for edge case and error path testing
- Identify untestable or difficult-to-test code
- Verify adherence to design patterns in use
- Check consistency with established coding standards
- Assess alignment with module/service boundaries

## 6. Classify Findings
- Tag each finding with severity (critical/high/medium/low)
- Categorize by type (bug, security, performance, maintainability, style)
- Distinguish must-fix items from nice-to-have improvements
- Group related findings for coherent refactoring or redesign work
- Escalate high-risk or cross-cutting findings appropriately

## 7. Bug Detective Step
As part of every code review, reviewers must explicitly apply the @bug-detective workflow to systematically check for potential bugs, even if no defect is reported. This step is required before completing the review.

## 8. Provide Recommendations
- Provide clear, concrete suggestions or small patch-style changes where possible
- Explain rationale and expected impact for each suggestion
- Estimate relative effort and risk (low/medium/high) for larger refactors
- Recommend refactoring strategies or patterns aligned with existing architecture
- Suggest targeted test coverage improvements to guard against regressions

## 9. Validate & Perform Code Documentation
- Confirm recommendations preserve functionality
- Verify alignment with team standards
- Document edge cases or exceptions uncovered during the review
- Flag findings requiring specialist review
- Provide a concise but comprehensive review summary
- When the review results in behavioral changes, new edge cases, or updated contracts, invoke the @code-documentation instruction workflow to update relevant documentation based on review findings and recommendations
