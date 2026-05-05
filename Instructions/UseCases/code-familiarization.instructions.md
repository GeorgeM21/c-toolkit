---
name: code-familiarization.instructions
applyTo: '**'
description: "Systematic code comprehension and familiarization workflow"
---

# Code Familiarization Workflow

## Input Expectations
- Key files, modules, or entry points to explore
- High-level description of the system or feature (if available)
- The user’s goal (e.g., "I need to extend feature X" or "I’m onboarding")
- Any existing diagrams, docs, or tests that illustrate typical flows

## Fast Path (Quick Orientation)
- Identify the main entry point(s) for the feature or service
- Provide a 3–5 bullet summary of what the code does
- Suggest a reading order (which files/classes to read first, next, later)

## 1. Gather Initial Context
- Request or infer the business domain and main scenarios
- Identify key business logic, dependencies, and architectural patterns
- Note recent changes, refactors, or onboarding needs
- Review existing tests for examples of typical usage

## 2. Map the Territory
- Examine module/class structure, layers, and dependencies
- Trace primary data and control flows (e.g., request → controller → service → repository → DB)
- Identify integration boundaries (external APIs, queues, shared libraries)
- Sketch a lightweight architecture map or textual diagram

## 3. Build Explanations
- Summarize purpose and responsibilities of key modules and flows
- Clarify important business rules, invariants, and edge cases
- Explain how components collaborate and where extension points exist
- Mark AI-generated explanations for traceability and human review

## 4. Validate Understanding
- Cross-reference explanations with code, tests, and documentation
- Call out any ambiguous or conflicting behavior explicitly
- Ensure explanations are clear for the intended audience and goal
- Highlight security, performance, or reliability hotspots worth extra attention

## 5. Deliver Familiarization
- Provide a structured "tour" with sections (overview, components, flows, hotspots)
- Suggest a reading and experimentation plan (files to open, tests to run)
- Offer guidance on where to change code to achieve the user’s goal
- Mark explanations as suitable for knowledge sharing and future onboarding
- When the goal is long-term knowledge sharing or onboarding, invoke the @code-documentation workflow to turn key explanations into formal, shareable documentation (e.g., READMEs, architecture notes, or runbooks)
- If, during familiarization, you identify suspicious behavior, inconsistencies, or likely defects, invoke the @bug-detective workflow to perform a systematic debugging pass before proposing changes
- When the user explicitly asks for an assessment of code quality or refactoring opportunities after understanding the code, you may follow up with the @code-reviewer workflow to perform a focused review
