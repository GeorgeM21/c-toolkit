---
name: unit-testing-generator.instructions
applyTo: '**'
description: "Systematic unit test generation workflow"
---

# Unit Testing Generator Workflow

## Input Expectations
- Class/module or functions under test, plus any existing tests
- Test framework and runner (e.g., xUnit, JUnit, Jest, NUnit)
- Coverage goals and critical scenarios to prioritize
- Recent changes, bug fixes, or refactors that require regression tests

## Fast Path (High-Value Tests)
- Identify the most critical public behaviors and edge cases
- Generate 3–5 focused tests that cover them end-to-end
- Suggest how to extend coverage if needed

## 1. Gather Context
- Request class/module under test and relevant usage examples
- Review unit testing guidelines, framework, and project conventions
- Identify public methods, observable behaviors, and coverage requirements
- Note recent bugs, regressions, or risky changes to target with tests

## 2. Scaffold Tests
- Generate test class/suite using the appropriate framework and conventions
- Apply AAA (Arrange, Act, Assert) structure consistently
- Use clear, descriptive test names (e.g., `MethodName_Should_DoX_WhenY`)
- Cover core public behaviors, edge cases, and boundary conditions

## 3. Apply Standards
- Enforce framework- and language-specific best practices
- Use mocks/stubs/test doubles appropriately for external dependencies
- Avoid over-mocking or asserting implementation details rather than behavior
- Add traceability markers for AI-generated tests where appropriate
- Where applicable, align test code style and structure with the general code-generation guidelines (naming, formatting, and error handling conventions) used in the project.
## 4. Verify & Refine
- Run tests to ensure they compile, execute, and are stable
- Check that coverage meaningfully exercises behavior, not just lines
- Strengthen weak assertions and remove redundant or flaky tests
- Add or refine regression tests for known bugs and edge cases
- When tests are generated in response to a recent bug or incident, ensure they directly cover the root cause and scenarios identified by any prior @bug-detective workflow

## 5. Review Test Quality
- When test suites are substantial or critical for safety, invoke the @code-reviewer workflow to assess test quality, realism, and adherence to team standards
- Confirm that tests are behavior-focused rather than implementation-detail-focused
- Check that naming, structure, and setup/teardown patterns are consistent with the rest of the codebase
- Adjust tests based on review feedback before relying on them for sign-off

## 6. React to Failing or Flaky Tests
- If new or existing tests reveal failing paths, flaky behavior, or suspected defects, invoke the @bug-detective workflow to systematically investigate and identify root causes
- Update or extend regression tests once the root cause is understood, ensuring they guard against recurrence
- Re-run the relevant test suites to validate that fixes are effective and stable

## 7. Document & Communicate
- Provide a brief summary of what the tests cover and what they do not
- Describe how to run the new or updated tests
- Suggest next steps to improve coverage or test structure
- Highlight any gaps that need product or domain clarification
- When tests represent a significant change in test strategy or coverage, invoke the @code-documentation workflow to describe key scenarios, how to run tests, and how they relate to the system’s critical behaviors
