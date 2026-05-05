---
name: code-generation.instructions
applyTo: '**'
description: "Systematic code generation workflow"
---

# Code Generation Workflow

## Input Expectations
- User goal and expected output (new feature, endpoint, utility, refactor, tests)
- Target project/module, technology stack, and architectural constraints
- Acceptance criteria, edge cases, and non-functional requirements (security/performance)
- Relevant files, existing patterns, and any required interfaces/contracts

## Fast Path (Minimal Working Output)
- Generate a focused, compilable implementation for the highest-priority scenario
- State assumptions explicitly when requirements are incomplete
- Highlight what remains optional (hardening, tests, broader refactor)

## 1. Gather Context and Requirements
- Clarify scope, expected behavior, and definition of done
- Identify dependencies, shared abstractions, and integration points
- Confirm whether the task is single-file, multi-file, or cross-layer
- Ask concise clarifying questions when critical information is missing

## 2. Analyze Existing Architecture
- Review codebase patterns for naming, structure, layering, and conventions
- Reuse existing modules, utilities, and base types before creating new abstractions
- Verify package/library availability before using APIs
- Respect architecture boundaries (do not bypass established service/repository layers)

## 3. Choose Generation Mode
- **Patch Mode**: minimal targeted changes with low risk
- **Implementation Mode**: coordinated feature development across affected files
- **Scaffold Mode**: structure-first generation with clear TODOs for deferred logic
- Explicitly state selected mode and why it fits the request

## 4. Generate Code
- Produce clean, idiomatic, compilable code aligned with project style
- Implement input validation, null/undefined handling, and safe error paths
- Use async patterns for I/O and avoid blocking calls where applicable
- Keep imports/usings minimal and accurate
- Avoid introducing new dependencies unless clearly justified

## 5. Apply Quality, Security, and Performance Standards
- Enforce secure defaults (no hardcoded secrets, validated inputs, safe data access)
- Ensure consistent naming and maintainable structure
- Handle expected errors with actionable messages and appropriate logging
- Consider common performance pitfalls (N+1 queries, redundant loops, unnecessary allocations)
- Keep changes focused; avoid unrelated refactors

## 6. Validate and Refine
- Run available compile/lint/test checks relevant to generated code
- Verify there are no unresolved symbols, missing imports, or broken contracts
- Refine weak naming, fragile logic, and unclear assumptions
- When generated code introduces or updates behavior with meaningful risk, invoke the @unit-testing-generator workflow to add focused tests for core and edge scenarios

## 7. Review and Handoff
- Before finalizing substantial generated changes, invoke the @code-reviewer workflow to assess code quality, architectural fit, and maintainability
- If unexpected behavior, defects, or ambiguous runtime issues appear, invoke the @bug-detective workflow to isolate and address root causes
- Once implementation is stable, invoke the @code-documentation workflow to capture generated APIs, assumptions, constraints, and usage notes
- Provide a concise handoff summary: what was generated, assumptions made, validations performed, and suggested next steps
