---
name: code-documentation.instructions
applyTo: '**'
description: "Systematic code documentation workflow"
---

# Code Documentation Workflow

## Input Expectations
- Target code files/modules (or PR/diff) to document
- Intended audience (new developers, SREs, architects, external integrators, etc.)
- Preferred documentation format (Markdown, docstrings, Javadoc/XML docs, ADRs, etc.)
- Any team style guide, existing docs, or examples to align with

## Fast Path (Quick Docs)
- Summarize the module’s purpose in 2–3 sentences
- Document public APIs (methods, endpoints, events) with inputs, outputs, and side effects
- Add a short "How to use" example or recipe

## 1. Analyze Code for Documentation
- Identify public surfaces: classes, interfaces, functions, endpoints, configuration points
- Focus on behaviors that affect consumers: inputs, outputs, side effects, failure modes
- Note extension points and integration boundaries (DB, queues, external services)
- Locate existing comments, README sections, or design docs to reuse or align with

## 2. Design Documentation Artifacts
- Choose appropriate documentation types based on audience and scope:
	- API reference (signatures, parameters, return values, exceptions)
	- High-level overviews (module responsibility, dependencies, lifecycle)
	- How-to guides and integration recipes (step-by-step usage)
	- Design notes or ADR-style entries (context, decision, consequences)
- Decide placement: inline comments/docstrings, separate Markdown file, or PR description

## 3. Generate Documentation
- Describe purpose, responsibilities, and collaboration with other components
- Document inputs/outputs, error handling, and important invariants
- Clarify complex or non-obvious logic, including business rules and constraints
- Mark AI-generated sections for traceability and human review
- When documentation is created as part of a bug fix or incident, incorporate the root cause, fix summary, and key lessons learned from any prior @bug-detective workflow

## 4. Validate Documentation
- Cross-check each documented behavior against code, tests, and existing standards
- Explicitly call out any uncertainties instead of guessing
- Ensure clarity, conciseness, and accessibility for the intended audience
- Validate that security, performance, and compliance-relevant behavior is accurately described
- When documentation is produced after a code review, align terminology, scope, and emphasized risks with the findings and recommendations from the @code-reviewer workflow

## 5. Deliver Documentation
- Provide a structured, scannable output (headings, bullets, examples)
- Include integration notes: where this module is used and how changes propagate
- Suggest where to store or link the docs (repo path, wiki, PR description)
- Highlight follow-up actions or questions for maintainers to confirm
- When the primary goal is onboarding or knowledge transfer, consider following up with the @code-familiarization workflow to generate guided walkthroughs or learning materials