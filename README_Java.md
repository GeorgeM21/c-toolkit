# Java DevToolkit for GitHub Copilot

This package adds reusable GitHub Copilot agents, prompts, and instructions to a local project.
It is designed to be copied into another repository under the `.github` folder.

---

## 1. What is included

Copy these folders into your target project:

- `agents/`        -> specialized Copilot agents
- `instructions/`  -> reusable coding and workflow instructions
- `prompts/`       -> user-facing slash prompts

Main examples:

- `prompts/create-plan.prompt.md` -> creates implementation plans with `Plan-Orchestrator`
- `prompts/add-unit-tests.prompt.md` -> generates tests with `Unit-Testing-Generator`
- `prompts/create-readme.prompt.md` -> creates README content with `Code-Documentation`
- `instructions/Tech/java-copilot.instructions.md` -> Java/Spring conventions
- `instructions/UseCases/feature-scaffolder.instructions.md` -> feature workflow guidance

## 2. Prerequisites

Before installing this pack, make sure you have:

1. A local Git repository or project folder.
2. GitHub Copilot access enabled on your GitHub account.
3. An IDE with GitHub Copilot support, such as:
	 - Visual Studio Code
	 - JetBrains IDEs with the GitHub Copilot plugin
4. GitHub Copilot Chat installed and signed in.

Recommended:

- Open the target project as the active workspace before copying these files.
- Make sure hidden folders such as `.github` are visible in your editor.

## 3. Installation step by step

Use these steps in the local project where you want Copilot to use this toolkit.

Step 1 - Open your target project

- Open your own repository in VS Code or IntelliJ.
- Confirm that the project root is the folder where you want `.github` to live.

Step 2 - Create the `.github` folder

- At the root of your target project, create this folder if it does not already exist:

	`.github`

Step 3 - Copy the toolkit folders

- Copy the following folders from this package into your target project's `.github` folder:

	- `agents`
	- `instructions`
	- `prompts`

Your target project should look similar to this:

	.github/
		agents/
		instructions/
			Tech/
			UseCases/
		prompts/

Step 4 - Generate baseline workspace instructions

- Open GitHub Copilot Chat in your IDE.
- Run:

	`/init`

- Copilot should generate a workspace instruction file, typically:

	`.github/copilot-instructions.md`

- This file gives Copilot a baseline understanding of your own codebase, style, and conventions.

Step 5 - Review the generated instructions

- Open `.github/copilot-instructions.md`.
- Keep the generated project-specific guidance.
- If your team already has rules, merge them carefully instead of deleting useful content.

Step 6 - Refresh prompt discovery if needed

If the new prompts do not appear immediately:

- close and reopen the workspace, or
- reload the IDE window, or
- reopen Copilot Chat and type `/` again

## 4. Basic configuration guidance

After installation, review these areas in the target project:

1. `.github/copilot-instructions.md`
	 - store project-specific rules here
	 - keep it short, factual, and tailored to the repo

2. `.github/instructions/Tech/`
	 - use this for language or platform-specific guidance
	 - example: `java-copilot.instructions.md` applies to Java-related files

3. `.github/instructions/UseCases/`
	 - use this for workflows such as code generation, review, testing, and bug investigation

4. `.github/agents/`
	 - use these when you want specialized Copilot behavior behind a prompt

5. `.github/prompts/`
	 - this is the main user entry point
	 - each `.prompt.md` file becomes a slash prompt in Copilot Chat

## 5. How to use the prompts

Prompts are the easiest way to use this toolkit.

How to open them:

- Open Copilot Chat.
- Type `/` to open the prompt list.
- On Windows, `Ctrl+Space` can help surface suggestions depending on IDE settings.
- Select the prompt you want, then provide the task details.

Common prompts included in this pack:

- `/create-plan`
	- analyzes the codebase and creates an implementation plan

- `/add-unit-tests`
	- adds or proposes unit tests for existing code

- `/add-integration-tests`
	- helps create integration tests

- `/add-testcontainers-support`
	- adds Testcontainers-related setup

- `/add-dockerfile`
	- creates a Dockerfile aligned to the project

- `/add-github-actions`
	- scaffolds GitHub Actions workflows

- `/create-readme`
	- generates a structured README for the target project

- `/new-layered-api`
	- scaffolds a layered API

- `/new-webflux-api`
	- scaffolds a WebFlux API

- `/new-thymeleaf-app`
	- scaffolds a Thymeleaf application

- `/research-code-best-practices`
	- gathers best-practice guidance for the requested topic

Tip:

- Start with `/create-plan` for larger changes.
- Use generation prompts only after the plan and scope are clear.

## 6. How the agents work

Agents are specialized Copilot personas used by prompts.
In most cases, users do not call agent files directly. Instead, a prompt points to an agent through the `agent:` field in the prompt front matter.

Examples:

- `create-plan.prompt.md` -> `Plan-Orchestrator`
- `add-unit-tests.prompt.md` -> `Unit-Testing-Generator`
- `create-readme.prompt.md` -> `Code-Documentation`
- `research-code-best-practices.prompt.md` -> `Web-Search-Researcher`

Included agents:

- `Plan-Orchestrator` -> planning and task breakdown
- `Code-Generation` -> implementation and patch creation
- `Code-Documentation` -> README and documentation generation
- `Code-Familiarization` -> explain existing code paths
- `Code-Reviewer` -> review quality, risks, and maintainability
- `Bug-Detective` -> investigate defects systematically
- `Feature-Scaffolder` -> scaffold larger features or new components
- `Unit-Testing-Generator` -> add tests and testing strategy
- `Web-Search-Researcher` -> gather external best practices and official guidance

## 7. How to use the use cases and instructions

The `instructions/` folder gives Copilot reusable rules.

There are two types:

1. Tech instructions
	 - example: `instructions/Tech/java-copilot.instructions.md`
	 - use these for stack-specific rules such as Java, Spring Boot, YAML, Gradle, and configuration style

2. Use case instructions
	 - examples:
		 - `bug-detective.instructions.md`
		 - `code-generation.instructions.md`
		 - `code-reviewer.instructions.md`
		 - `feature-scaffolder.instructions.md`
		 - `unit-testing-generator.instructions.md`
	 - use these for repeatable workflows

How they are used:

- Keep them inside `.github/instructions/`.
- Let prompts and agents reference them during execution.
- Update them when you want to change the workflow for your whole team.

Simple rule:

- prompts are what the user selects
- agents are the specialized worker behind the prompt
- instructions are the reusable rules that shape the result

## 8. Recommended first-time workflow

For a new project, use this order:

1. Install GitHub Copilot and Copilot Chat.
2. Copy `agents`, `instructions`, and `prompts` into `.github/`.
3. Run `/init`.
4. Review `.github/copilot-instructions.md`.
5. Try `/create-plan` on a small feature or refactor.
6. Use `/add-unit-tests` or `/create-readme` to validate that prompts are working.

## 9. Troubleshooting

Prompts do not appear:

- verify the files are inside `.github/prompts/`
- verify the files still end with `.prompt.md`
- reload the workspace or IDE
- type `/` again in Copilot Chat

Agents are not behaving as expected:

- verify the matching `.agent.md` file exists in `.github/agents/`
- verify the prompt `agent:` value matches the agent `name`
- check that the YAML front matter was not broken during editing

Instructions do not seem to apply:

- verify the files are under `.github/instructions/`
- check the `applyTo` pattern in the instruction file
- keep the YAML keys and indentation unchanged

`/init` does not generate instructions:

- confirm Copilot Chat is installed and signed in
- confirm you are running the command inside the target project workspace
- reopen the project and try again

## 10. Important editing notes

- Keep YAML front matter valid in all `.agent.md`, `.instructions.md`, and `.prompt.md` files.
- Do not rename metadata keys such as `name`, `description`, `agent`, `tools`, `applyTo`, or `handoffs` unless you intend to change behavior.
- Make small, careful edits and test prompt discovery after changes.
- Treat this toolkit as shared team configuration for Copilot behavior.

## 11. Summary

If you only remember the essentials:

1. Copy `agents`, `instructions`, and `prompts` into `.github/` in your project.
2. Run `/init`.
3. Open Copilot Chat and type `/` to use the prompts.
4. Customize `.github/copilot-instructions.md` and the instruction files to match your project.

That is all you need to install, configure, and start using this toolkit locally.