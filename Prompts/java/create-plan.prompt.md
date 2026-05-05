---
agent: 'Plan-Orchestrator'
description: 'Create a detailed implementation plan through codebase analysis, optional research, and iterative clarification'
# Note: Uses Plan-Orchestrator agent if available in .github/agents/, otherwise falls back to default Copilot agent
---

# Create Implementation Plan

## Task

Create a detailed, actionable implementation plan for a feature, bug fix, refactor, or architectural change.

Start by analyzing the current codebase and the user request. Use external research only when the task depends on current best practices, external APIs, framework-version behavior, security guidance, or other time-sensitive information.

Do not create, edit, or delete files.
Do not implement code.
Do not claim completion if only a plan was produced.

Present the plan directly in the chat instead of saving it to a file.

## Configuration

Before execution, confirm or infer these settings:

| Setting | Options | Default |
|---------|---------|---------|
| Scope | Feature, Bug fix, Refactor, Architecture change | Feature |
| Research Mode | Auto, Local only, Web validated | Auto |
| Planning Depth | Standard, Deep | Deep |
| Include File Impact | Yes, No | Yes |
| Include Risks | Yes, No | Yes |
| Include Testing Plan | Yes, No | Yes |
| Include Rollout Notes | Yes, No | Yes |

Configuration rules:

- Start with local code analysis before using web research.
- Use web research only when it materially improves the plan.
- Ask questions only after inspecting the code and only for gaps that cannot be resolved through investigation.
- If the request conflicts with existing architecture or conventions, explicitly call that out and propose alternatives.
- The final output must be implementation-ready, not a brainstorming dump.

## Execution Steps

1. Read the request and identify the real outcome being asked for.
2. Inspect the relevant code, configuration, tests, and surrounding architecture.
3. Summarize the current state and verify assumptions against the codebase.
4. Decide whether deeper code familiarization or web research is needed.
5. If needed, use specialist agents to gather additional context.
6. Present an informed summary of findings and any focused clarification questions.
7. Propose a phased implementation approach.
8. Refine the approach based on user feedback.
9. Present the final implementation plan in chat.

## Guidelines

- Be skeptical of vague requirements and verify assumptions against real code.
- Prefer evidence from the repository over guesses.
- Prefer current official documentation when researching external guidance.
- Do not ask the user questions that can be answered by reading the code.
- Do not over-engineer the plan if the requested change is simple.
- Include concrete file paths, modules, or components likely to be touched.
- Distinguish clearly between facts, assumptions, risks, and recommendations.
- If important unknowns remain, stop and resolve them before finalizing the plan.

## Response Structure

Return the plan using this structure:

### 1. Goal
- What is being implemented and why

### 2. Current State Analysis
- What exists today
- Relevant patterns, constraints, and dependencies
- Specific file or module references when available

### 3. Proposed Approach
- Recommended implementation strategy
- Why this approach fits the current codebase
- Rejected alternatives when relevant

### 4. Implementation Phases
- Phase-by-phase breakdown
- What each phase changes
- What each phase should achieve

### 5. Impacted Areas
- Files, directories, services, endpoints, schemas, jobs, or UI surfaces likely to change

### 6. Risks and Edge Cases
- Technical risks
- Regression concerns
- Data, performance, security, or migration considerations

### 7. Testing and Validation Plan
- Automated verification
- Manual verification
- Rollout or migration checks when applicable

### 8. Out of Scope
- Explicitly state what this plan does not include

### 9. Open Questions
- Only include questions that could not be resolved through code investigation or research

## Best Practices to Follow

- Ground recommendations in the existing codebase.
- Use external research only when it is justified.
- Keep phases incremental and testable.
- Include measurable validation steps.
- Push back on approaches that add unnecessary complexity.
- Prefer implementation-ready plans over abstract architecture discussion.
