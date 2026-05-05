---
name: bug-detective.instructions
applyTo: '**'
description: "Systematic debugging workflow"
---

# Bug Detective Workflow

## Input Expectations
- Code snippet or link to relevant file(s)
- Exact error message / stack trace (if any)
- Clear reproduction steps, environment (OS, runtime, versions), branch/commit
- Expected vs actual behavior as described by the user

## Fast Path (Quick Triage)
- Restate the problem in 1–2 sentences
- Propose 1–3 plausible root-cause hypotheses
- Suggest 1–2 high-value diagnostic steps to confirm/falsify each hypothesis

## 1. Reproduce the Issue
- Request or refine reproduction steps from user
- Verify the bug is reproducible (or highlight if not reproducible)
- Identify preconditions, environment, and data dependencies

## 2. Gather Information
- Review error messages, stack traces, and logs
- Examine recent code or configuration changes
- Review related test failures and flaky tests
- Identify boundaries involved (I/O, DB, external APIs, queues, serialization)

## 3. Root Cause Analysis
- Use collected data to narrow down likely sources
- Trace execution path leading to failure
- Identify violated assumptions and missing guards
- Check for common bug patterns (off-by-one, nullability, race conditions, async issues)
- Explicitly list 1–3 candidate root causes and how to verify each

## 4. Propose Fix
- Suggest a minimal, targeted patch that addresses the identified root cause
- Explain how the change affects behavior and what risks it introduces
- Offer alternative fix options when confidence is low, with trade-offs
- Call out any unclear requirements or business rules needing confirmation

## 5. Validation & Regression Protection
- Propose regression tests to prevent recurrence
- Suggest related existing tests to update or extend
- Recommend integration or end-to-end testing when cross-boundary behavior is affected
- Highlight follow-up monitoring/logging improvements where useful
- When the bug fix affects externally visible behavior, contracts, or critical flows, explicitly invoke the @unit-testing-generator workflow to create or refine regression tests that cover the root cause and key scenarios

## 6. Quality & Standards Review
- Before finalizing the fix, invoke the @code-reviewer workflow to validate code quality, maintainability, and adherence to team standards
- Ensure naming, structure, and patterns are consistent with the surrounding codebase
- Confirm that logging, error handling, and observability meet expectations
- Ask for clarification where standards or expectations are unclear

## 7. Document the Incident and Fix
- Summarize the incident: symptoms, impact, and affected components
- Describe the identified root cause and why it occurred
- Outline the implemented fix and its implications
- Note any remaining risks, assumptions, or follow-up work
- Invoke the @code-documentation instruction workflow to update relevant documentation (runbooks, READMEs, API docs, or design notes) with the incident summary, root cause, and fix
