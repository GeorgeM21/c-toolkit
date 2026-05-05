# GenAI-Dev-Toolkit vs Default Copilot – POC

This document captures a simple A/B comparison between:

- **Baseline**: Default Copilot chat (Ask/Agent, no custom agent selected)
- **Toolkit**: Copilot chat using custom agents and instructions from this repo

Test repo: local clone of `https://github.com/dotnet/eShop` (eShopOnContainers), focusing on the **Ordering** service.

Each scenario uses **the same natural-language prompt** in both VS Code instances. The only difference is which agent handles it.

- **VS Code 1 (Baseline)**: Use default Copilot (no `@agent` prefix, or the default Agent entry).
- **VS Code 2 (Toolkit)**: Select the custom agent from the agent dropdown (e.g., `Code-Reviewer`, `Feature-Scaffolder`).

---

## Scenario 1 – Code Review (GracePeriodManagerService)

**Prompt (identical in both instances)**

> I’m working on the eShopOnContainers repo (local clone of https://github.com/dotnet/eShop).
>
> Please review the following class for correctness, maintainability, performance, security, and test coverage.
>
> Identify the most important issues (bugs, design problems, security/perf risks).
> Classify each finding by severity (critical/high/medium/low).
> Suggest concrete improvements or small refactorings.
> Tell me which tests you would expect to exist or be updated.
>
> Target class: `GracePeriodManagerService.cs` in the Ordering service. Use the existing architecture and coding style as reference.

### Baseline: Default Copilot (GPT-5.1)

- **Behavior**
  - Directly runs a deep analysis of the class and related types.
  - Produces a long, detailed review with findings and concrete code suggestions in prose format.
- **Key Findings** (highlights)
  - [High] Unhandled exceptions in the background loop can permanently stop the service.
  - [High] No cancellation support inside I/O and publishing paths.
  - [Medium] Duplicate event publication risk  query never marks orders as processed.
  - [Medium] Non-sargable SQL predicate (`CURRENT_TIMESTAMP - "OrderDate" >= @GracePeriodTime`).
  - [Medium] `NpgsqlException` catch hides ongoing DB failures (empty list looks like "no work").
  - [Medium] No upper bound or batching on result set.
  - [Low] Logging full `NpgsqlException` may expose infrastructure details (PII/security concern).
  - [Low] Unnecessary `IsEnabled(LogLevel.Debug)` guards and `ValueTask<List<int>>` usage.
- **Improvements Suggested**
  - Wrap loop body in try/catch with logging and optional backoff.
  - Thread `CancellationToken` through `CheckConfirmedGracePeriodOrders` and `GetConfirmedGracePeriodOrders` into `OpenAsync`/`ExecuteReaderAsync`.
  - Rewrite SQL to sargable predicate; ensure `(OrderStatus, OrderDate)` index exists.
  - Enforce idempotency: mark orders as processed or document consumer-side guarantee.
  - Simplify return type to `Task<List<int>>`.
- **Tests Suggested**
  - Unit tests for publishing behavior (non-empty list  one event per ID; empty list  no events).
  - Error handling tests (NpgsqlException logged, loop continues).
  - Cancellation/lifecycle tests (service exits cleanly on token cancellation).
  - Integration test with real/test Postgres for query semantics and grace period correctness.
  - Idempotency/duplication tests (consumer-side or producer-side, depending on design choice).

**Summary:** Very strong single-run review with rich technical detail. Covers all key dimensions (correctness, performance, security, testing). However, findings are presented in flat prose without explicit severity classification or structured sections  the structure and workflow are implicit.

### Toolkit: `Code-Reviewer` Custom Agent (GPT-5.1)

- **Behavior**
  - Produces an explicit **Review Plan** before starting (scope, focus areas, strategy).
  - Follows its documented workflow: structured multi-dimension review with severity table.
  - Output follows a clear template: Review Plan  Review Findings (severity table) -> Improvements  Tests  Clarifications.
- **Key Findings** (highlights)
  - [High] Same loop exception handling risk as baseline.
  - [High] **Config safety** -- `CheckUpdateTime` and `GracePeriodTime` not validated; negative values cause crashes or hammer the DB. Recommends `IValidateOptions<BackgroundTaskOptions>` with min/max enforcement.
  - [Medium] Same cancellation, error semantics, `ValueTask`, and duplicate publication findings.
  - [Medium] **Security/Logging** -- logging full `NpgsqlException` may leak connection details or SQL text.
  - [Low-Medium] Duplicate publication risk with explicit call to document or enforce idempotency contract.
  - [Low] Primary-constructor inconsistency (some deps stored as fields, others not).
  - [Low] No batching/limit on query results.
  - **Positive aspects** section: acknowledges parameterized SQL, `IsEnabled` guards, and alignment with eShop architectural patterns.
- **Improvements Suggested**
  - Same core refactors as baseline (try/catch, cancellation threading, sargable SQL, `Task<List<int>>`).
  - **Additional**: Options validation via `IValidateOptions` with fail-fast on invalid config.
  - Clarify error vs. domain semantics (DB failure  "no orders").
- **Tests Suggested**
  - Similar core coverage (publishing behavior, error handling, cancellation).
  - **Additional**: Configuration validation tests (invalid `CheckUpdateTime`/`GracePeriodTime`  startup failure).
  - Integration test with real/test Postgres for query correctness.
- **Clarifying Questions Asked**
  - "Is the downstream consumer of `GracePeriodConfirmedIntegrationEvent` guaranteed to be idempotent?"
  - "Do you have a global logging policy regarding SQL/connection details in error logs?"

**Summary:** Technical depth comparable to baseline, with these additional advantages:

- Produces an **explicit Review Plan** upfront.
- Uses a **structured severity table** (High/Medium/Low with Area, Finding, Recommendation columns).
- Identifies a finding the baseline missed: **configuration validation** as a High-severity correctness risk.
- Includes a **"Positive aspects"** section acknowledging good patterns (balanced feedback).
- Ends with **clarifying questions** (aligns with the Code-Reviewer agent's mandatory checkpoint workflow).
- Reflects the **Code-Reviewer workflow** defined in this repo.

> **Observed Value:** With both baseline and toolkit on GPT-5.1, the custom `Code-Reviewer` doesn't just match the baseline  it **surfaces an additional High-severity finding** (config validation), provides **more structured output** (severity table, review plan, clarifications), and delivers **balanced feedback** (positive aspects). The structured workflow makes the review more **consistent, auditable, and aligned with the team's review process**.

---


## Scenario 2 – Feature Scaffolding (Paginated Orders Endpoint)

**Prompt (identical in both instances)**

> In the eShopOnContainers repo, within the Ordering service, I want to add a new feature.
>
> Scenario:
> Add an endpoint that returns a paginated list of orders for a given customer, with optional filtering by order status.
>
> Requirements:
> - Read‑only query (no mutation).
> - Input: `customerId` (required), `page`/`pageSize`, optional `status` filter.
> - Output: list of orders (id, date, total, status) plus pagination metadata.
> - Follow the existing clean architecture and patterns already used in the Ordering service.
>
> Please:
> - Propose a minimal design (routes, DTOs, application/service methods, data access changes if needed).
> - Implement the code changes across the relevant projects (API, Application, domain/infrastructure as needed).
> - Add or update unit tests to cover the new behavior.
> - Tell me how to run these tests in this solution.

### Baseline: Default Copilot (Claude Sonnet 4.5, no custom agent)

- **Behavior**
  - Reads Ordering service (API, Application, Queries, tests).
  - Creates todos and then implements the feature end-to-end.
- **Files Touched** (representative)
  - `OrderViewModel.cs`: new `PaginatedOrders`-style DTO.
  - `IOrderQueries.cs`: new method signature.
  - `OrderQueries.cs`: new query using `Skip/Take`, filters by customer and status, sorted by date.
  - `OrdersApi.cs`: new endpoint `GET /api/orders/customer/{customerId}` with query params.
  - New tests: separate query and API test files (`GetOrdersByCustomerQueryTests.cs`, `GetOrdersByCustomerApiTests.cs`).
- **Output**
  - Clear design summary, list of files changed, and test commands.
  - Claims ~14 tests covering pagination, status filtering, edge cases.

**Summary:** Implements the feature correctly with good tests and an understandable summary, but the structure is relatively ad-hoc.

### Toolkit: `Feature-Scaffolder` Custom Agent (Claude Sonnet 4.5)

Two independent runs were executed with the same prompt.

#### Run 1

- **Behavior**
  - Acts as a full agent: reads architecture, creates a feature scaffold plan, edits multiple files, adds tests.
- **Files Touched** (representative)
  - Same core set as baseline: `OrderViewModel.cs`, `IOrderQueries.cs`, `OrderQueries.cs`, `OrdersApi.cs`, plus tests (`OrderQueriesTest`, `OrdersWebApiTest`).
- **Output Style**
  - Uses the agent’s own structure: `🏗️ FEATURE SCAFFOLD`, “Feature: Paginated Orders Query with Status Filtering”, architecture overview, and stepwise implementation.
  - Adds a validation/pagination flow that, subjectively, was a bit more explicit than the baseline.

#### Run 2

- **Behavior**
  - Similar plan and architecture analysis, but only modifies 5 files this time (no separate query test file; focuses on API tests in `OrdersWebApiTest.cs`).
- **Files Touched**
  - `OrderViewModel.cs`: `PaginatedOrders` DTO with metadata.
  - `IOrderQueries.cs`: `GetPaginatedOrdersByCustomerAsync` method.
  - `OrderQueries.cs`: implementation with customer filter, optional status filter, pagination (defaults + max limit), sorted by date.
  - `OrdersApi.cs`: new route `GET /api/orders/customer/{customerId}/paginated` with query params.
  - `OrdersWebApiTest.cs`: 5 focused tests for pagination and filtering.
- **Output Style**
  - Highly structured summary:
    - `🏗️ FEATURE SCAFFOLD SUMMARY`
    - `📁 FILES MODIFIED`
    - `🎯 API ENDPOINT DETAILS`
    - `✅ TEST COVERAGE`
    - `🧪 HOW TO RUN THE TESTS`
    - `🔍 DESIGN DECISIONS`
    - `📊 FEATURE VALIDATION CHECKLIST`
    - `🚀 NEXT STEPS`
  - Deep explanation of design choices (clean architecture, CQRS query side, pagination strategy, status filtering, performance, security).

**Summary:** Quality is comparable to baseline, but the custom agent's output is **much more structured and educative**, mapping directly to the workflow and success criteria defined in `feature-scaffolder.agent.md` and `feature-scaffolder.instructions.md`.

> **Observed Value:** `Feature-Scaffolder` doesn't just "write the code"; it **teaches and documents the feature slice** in your own architectural language (layers, CQRS, pagination rules, validation, tests), making it easier for teams to reuse the pattern.

---

### Critical Observation: Build Failure in Baseline

After running `dotnet test tests/Ordering.UnitTests/Ordering.UnitTests.csproj`, the **baseline implementation failed to build** with 16 errors.

**Toolkit Result:**

The custom `Feature-Scaffolder` agent produced code that **builds and runs successfully**. While the generated code is structurally similar, the custom agent:
- Included proper package references in the test project.
- Used explicit namespace qualifications or type aliases to avoid ambiguity.
- Ensured correct `using` statements for ASP.NET Core result types.

> **Key Insight:** The custom agent's adherence to the team's **coding standards and instructions** (from `copilot-instructions.md`) helps avoid common scaffolding pitfalls. The baseline, lacking this context, made assumptions that led to compilation errors.

---

## Conclusions from Scenarios 1–2

1. **Baseline Copilot is already very strong.**
   - It can perform deep reviews and implement cross-file features with tests using only the prompt and repo context.

2. **Custom agents add consistency and process.**
   - `Code-Reviewer` and `Feature-Scaffolder` always:
     - Create a small plan/todos.
     - Follow a stable, documented workflow.
     - Classify findings/changes and explicitly discuss tests.
   - Outputs are easier to read, audit, and reuse as teaching material.

3. **Custom agents can surface additional findings.**
   - `Code-Reviewer` identified **configuration validation** as a High-severity risk that the baseline missed entirely.
   - The structured review plan and severity table made it easier to spot gaps in coverage.

4. **Custom agents encode team standards.**
   - They speak in terms of your **clean architecture**, **CQRS**, and **testing/security/documentation standards** (from `copilot-instructions.md` and the UseCase instructions).
   - This is hard to guarantee with ad-hoc prompts alone.

5. **For team workflows, the toolkit is clearly preferable.**
   - Using `@code-reviewer` for PRs and `@feature-scaffolder` for new features should give more predictable quality and better alignment with your norms than relying on one-off Ask prompts.

---

## Scenario 3 – Code Familiarization (Ordering Service)

**Prompt (identical in both instances)**

> I’m new to the eShopOnContainers codebase and want to understand the Ordering service.
>
> Please give me a practical, developer‑focused overview:
> 
> - Main responsibilities of the Ordering service.
> - Key projects and layers involved (API, Application, Domain, Infrastructure) and how they interact.
> - A typical request flow from the API down to persistence and back.
> - The top files/classes I should read first if I want to add or modify an endpoint.

### Baseline: Default Copilot (GPT-5.1)

- **Behavior**
  - Uses many workspace references to infer structure.
  - Produces a concise, practical overview focused on concrete types and flows.
- **Summary of Answer**
  - Responsibilities: order management, persistence, integration, idempotency.
  - Layers and examples:
    - API: `Ordering.API`, main entry `OrdersApi`.
    - Application: commands/handlers like `CreateOrderCommandHandler`.
    - Domain: entities like `Order`, `OrderItem`.
    - Infrastructure: `OrderingContext`, repositories.
  - Request flow: HTTP → endpoint (`OrdersApi`) → command → handler → domain → repository/`OrderingContext` → integration events → response.
  - Reading order: start from `OrdersApi`, then commands/handlers, then domain entities and `OrderingContext`/repositories.

**Summary:** Very usable overview with concrete entry points, but the structure is ad-hoc and less tied to explicit architectural terminology.

### Toolkit: `Code-Familiarization` Custom Agent (GPT-4.1)

- **Behavior**
  - Treats the task as a formal onboarding scenario and follows its familiarization workflow.
  - Uses fewer references but leans more on shared instructions (clean architecture).
- **Summary of Answer**
  - Responsibilities: order lifecycle, business rules, integration with other services.
  - Layers and responsibilities described in clean-architecture terms:
    - `Ordering.API`: HTTP entry, validation, mapping to application layer.
    - `Ordering.Application`: use cases (PlaceOrder, CancelOrder), orchestration, transactions, interfaces.
    - `Ordering.Domain`: aggregates/entities (`Order`, `OrderItem`, `Buyer`), invariants, no infra deps.
    - `Ordering.Infrastructure`: EF Core `OrderingDbContext`, repositories, integrations.
  - Request flow: API → Application → Domain, with Infrastructure implementing persistence and integrations.
  - Suggested reading path:
    - `OrderController`/similar controller for endpoints.
    - Application use cases/services directory.
    - `Order` aggregate and repository interfaces/implementations.
    - `OrderingDbContext` for persistence.
  - Includes an explicit mini-analysis and "next steps" checklist for further exploration and tests.

**Summary:** Similar factual content to baseline, but framed in the language of clean architecture and onboarding checklists, making it easier to teach and standardize how new developers learn the service.

> **Observed Value:** For familiarization, the custom agent doesn't reveal new facts, but provides a more **structured learning path** and uses the team's architectural vocabulary, which is useful for onboarding and documentation.

---

## Scenario 4 – Bug Detective (Referential Integrity / Null Pointer Exception)

**Prompt (identical in both instances)**

> I'm investigating a bug in the eShopOnContainers repo (local clone of https://github.com/dotnet/eShop).
>
> **Reported Symptoms:**
> - Sometimes when a customer goes to checkout, the web app crashes with a `KeyNotFoundException`.
> - In other cases, orders get stuck in `AwaitingValidation` status and never proceed.
> - These issues seem to happen after an admin deletes a product from the catalog.
>
> **Suspected Area:**
> - The integration between Catalog, Basket, and Ordering services.
> - Specifically, look at `BasketState.cs` in WebApp, and `OrderStatusChangedToAwaitingValidationIntegrationEventHandler.cs` in Catalog.API.
>
> **Your Task:**
> 1. Reproduce the issue by tracing the code path from basket checkout through order validation.
> 2. Identify the root cause(s) of both symptoms.
> 3. Propose minimal, targeted fixes that prevent the crashes without breaking the existing architecture.
> 4. Suggest regression tests and any follow-up improvements (logging, monitoring, defensive patterns).

### Baseline: Default Copilot (Claude Sonnet 4.5)

- **Behavior**
  - Created 6 todos and worked through them systematically.
  - Read multiple files across services (BasketState.cs, OrderStatusChangedToAwaitingValidationIntegrationEventHandler.cs, CatalogService.cs, CatalogApi.cs, etc.).
  - Correctly identified both root causes.
- **Root Cause Analysis**
  - **Bug #1**: `KeyNotFoundException` in `BasketState.cs` line 136 – dictionary lookup `catalogItems[item.ProductId]` throws when product is deleted.
  - **Bug #2**: Silent skip in `OrderStatusChangedToAwaitingValidationIntegrationEventHandler.cs` – if catalog item is null, it's skipped; if ALL products are deleted, the empty list causes incorrect `OrderStockConfirmedIntegrationEvent` to be published instead of rejection.
- **Fixes Proposed**
  - `BasketState.cs`: Changed to `TryGetValue` with `continue` if product not found.
  - `OrderStatusChangedToAwaitingValidationIntegrationEventHandler.cs`: Added `else` branch with logging and explicit `ConfirmedOrderStockItem(productId, hasStock: false)` for deleted products.
- **Tests Created**
  - 1 test file: `OrderStatusChangedToAwaitingValidationIntegrationEventHandlerTests.cs`
  - 4 tests using NSubstitute for mocking, relying on `CatalogApiFixture`.
  - Tests cover: deleted product rejection, mixed deleted/existing products, all products deleted, valid products confirmation.
- **Style**
  - Ad-hoc structure, findings emerged during exploration.
  - Good technical depth but less structured presentation.

### Toolkit: `Bug-Detective` Custom Agent (Claude Sonnet 4.5)

- **Behavior**
  - Followed structured investigation workflow with clear sections: `🔬 INVESTIGATION`, `Evidence Collected`, `Root Causes`, `Proposed Fixes`.
  - Read similar files but presented findings in a more organized, evidence-based format.
  - Explicitly traced the code path from basket → checkout → order validation → integration event.
- **Root Cause Analysis**
  - **Bug #1**: Same finding – `KeyNotFoundException` in `BasketState.cs` line 136.
  - **Bug #2**: Same finding – silent skip causes stuck orders when all products are deleted.
  - Presented with clear **Location**, **Cause**, **Impact** structure for each bug.
- **Fixes Proposed**
  - `BasketState.cs`: Identical fix using `TryGetValue` with `continue`.
  - `OrderStatusChangedToAwaitingValidationIntegrationEventHandler.cs`: **More concise fix** – simplified the logic to:
    ```csharp
    var hasStock = catalogItem is not null && catalogItem.AvailableStock >= orderStockItem.Units;
    ```
    This eliminates the `if/else` branching entirely, treating null as "no stock" in a single expression.
- **Tests Created**
  - 2 test files:
    - `OrderStatusChangedToAwaitingValidationIntegrationEventHandlerTests.cs` – 3 unit tests using Moq and in-memory database.
    - `DeletedProductTests.cs` – 3 functional/API tests for the Catalog API endpoint behavior.
  - Tests cover: deleted product rejection, all deleted products rejection, valid products confirmation, API-level deleted product handling.
  - Includes XML documentation comments explaining the regression test purpose.
- **Style**
  - Structured presentation with emoji markers (`🔬 INVESTIGATION`).
  - Explicit evidence collection and code path tracing.
  - Clear separation of concerns between unit tests and functional tests.

### Comparison Summary

| Aspect | Baseline | Bug-Detective (Custom) |
|--------|----------|------------------------|
| **Root Cause Identification** | ✅ Both bugs found | ✅ Both bugs found |
| **Investigation Structure** | Ad-hoc, exploratory | Structured (Evidence → Root Cause → Fix → Tests) |
| **Fix Quality** | Good, uses if/else with logging | **Better** – more concise single-expression fix |
| **Test Coverage** | 4 unit tests, 1 file | 6 tests across 2 files (unit + functional) |
| **Test Approach** | NSubstitute + fixture | Moq + in-memory DB + API functional tests |
| **Documentation** | Minimal | XML comments explaining regression purpose |
| **Code Tracing** | Implicit during exploration | Explicit code path documentation |

### Key Observations

1. **Both agents found the same root causes** – The bug detection capability is equivalent.

2. **Custom agent produced a more minimal fix** – The single-expression `var hasStock = catalogItem is not null && ...` is more idiomatic and avoids code duplication compared to the baseline's `if/else` with logging in both branches.

3. **Custom agent created broader test coverage** – Added functional/API tests (`DeletedProductTests.cs`) in addition to unit tests, providing defense-in-depth at multiple layers.

4. **Custom agent followed its documented workflow** – The structured presentation (`🔬 INVESTIGATION`, explicit evidence collection, code path tracing) aligns with the `bug-detective.instructions.md` workflow (Fast Path → Reproduce the Issue → Gather Information → Root Cause Analysis → Propose Fix → Validation & Regression Protection).

5. **Test quality differs in approach**:
   - Baseline: Uses `CatalogApiFixture` (external dependency) with NSubstitute.
   - Custom: Uses self-contained in-memory database (better isolation) with Moq.

> **Observed Value:** For debugging, the custom `Bug-Detective` agent doesn't find more bugs, but produces **cleaner fixes** and **more comprehensive test coverage**. The structured workflow makes the investigation **easier to follow and audit**, which is valuable for incident documentation and knowledge sharing.

---

## Conclusions (Updated with Scenarios 1–4)

1. **Baseline Copilot is already very strong.**
   - It can perform deep reviews, implement cross-file features, and debug complex multi-service issues using only the prompt and repo context.

2. **Custom agents add consistency and process.**
   - `Code-Reviewer`, `Feature-Scaffolder`, `Code-Familiarization`, and `Bug-Detective` always:
     - Follow a stable, documented workflow.
     - Classify findings/changes and explicitly discuss tests.
     - Present information in a structured, auditable format.
   - Outputs are easier to read, audit, and reuse as teaching material.

3. **Custom agents produce higher-quality artifacts.**
   - `Code-Reviewer` surfaced an **additional High-severity finding** (configuration validation) that the baseline missed, plus delivered balanced feedback with a positive-aspects section.
   - `Feature-Scaffolder` produced code that **builds successfully** vs. baseline's 16 compilation errors.
   - `Bug-Detective` produced a **more concise fix** and **broader test coverage** (unit + functional tests).

4. **Custom agents encode team standards.**
   - They speak in terms of your **clean architecture**, **CQRS**, and **testing/security/documentation standards** (from `copilot-instructions.md` and the UseCase instructions).
   - This is hard to guarantee with ad-hoc prompts alone.

5. **For team workflows, the toolkit is clearly preferable.**
   - Using custom agents for PRs, features, onboarding, and debugging gives more predictable quality and better alignment with team norms than relying on one-off Ask prompts.
