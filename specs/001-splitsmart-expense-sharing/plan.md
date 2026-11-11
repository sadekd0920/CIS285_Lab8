# Implementation Plan: SplitSmart Expense Sharing

**Branch**: `001-splitsmart-expense-sharing` | **Date**: 2026-11-04 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `specs/001-splitsmart-expense-sharing/spec.md`

## Summary

**Selected architecture:** Architecture 1, chosen on 2026-11-04 in [architecture-review.md](./architecture-review.md), consolidated per [consolidation-instructions.md](./consolidation-instructions.md). [design.md](./design.md) defines the selected safeguards. Architecture 2 remains an unchanged comparison artifact; Event Sourcing, CQRS, and its connected PostgreSQL MVP are not selected.

Design SplitSmart as a modular Java 21 product whose domain and application rules are independent of presentation. The Lab 7 MVP is a Maven-built Java Swing desktop demonstration with local durable storage; it is deliberately not the complete customer product. The complete product remains web-first with Android and iOS clients using a shared cloud API and business rules. The architecture, data model, deployment alternatives, API contract, and PlantUML diagrams are documented alongside this plan.

## Technical Context

**Language/Version**: Java 21 for the MVP and proposed backend. Web and mobile client implementation technologies are not specified.

**Primary Dependencies**: Maven; Swing for the Java desktop MVP; JUnit 5 for Java tests. Proposed full-product server: Spring Boot REST API. Proposed persistence: SQLite for the local MVP and PostgreSQL for the shared full product.

**Storage**: MVP local SQLite database; full product PostgreSQL for transactional records and private object storage for receipts. No cross-currency ledger is assumed.

**Testing**: JUnit 5 unit tests for domain/application rules; adapter and API integration tests; contract tests for client-server behavior. Core business tests must not require GUI interaction.

**Target Platform**: MVP: Java 21 desktop runtime on supported Lab 7 workstations (exact OS matrix is not specified). Full product: supported web browsers, Android, and iOS; versions remain product decisions.

**Project Type**: Desktop MVP plus a proposed shared backend and web/mobile clients for the complete product.

**Performance Goals**: Meet NFR-003/AC-009 for the complete product: at least 95 of 100 ordinary actions provide a visible result within 3 seconds under agreed normal conditions. The environment and load profile must be established before verification.

**Constraints**: `BigDecimal` for monetary values with explicit currency scale and rounding; allocation shares reconcile exactly; approval must precede ledger effects; payment application is atomic and recorded once; authorization applies to every data access; Swing event handlers delegate to application services. A local desktop MVP is not a multi-device collaboration solution.

**Scale/Scope**: No workload or account-count target is specified. The design is a baseline rather than a capacity commitment. The MVP supports demonstration-scale local data; full-product sizing is a deployment decision.

## Constitution Check

**Pre-research gate — PASS**

- **Customer scope**: The design preserves FR-001–FR-012, including web, Android, and iOS. The Swing MVP is clearly identified as a limited desktop implementation decision and does not claim FR-012 conformance.
- **Java and build**: Java 21 and Maven are the proposed implementation baseline for Java components.
- **Object-oriented boundaries**: Domain entities and application use cases are separated from UI, persistence, delivery, and notification adapters.
- **GUI/business separation**: Swing views delegate to application services; domain tests run without constructing UI.
- **Monetary precision**: Monetary amounts use `BigDecimal`; scale and remainder behavior are explicit.
- **Verification**: JUnit 5 is the Java test baseline for allocation, approval, balance, and payment behavior.
- **Diagrams and traceability**: PlantUML source diagrams are provided; requirements remain traceable to the requirements inventory and contract.
- **Human review**: This AI-generated baseline requires stakeholder review before becoming authoritative.
- **Known limitation**: Local SQLite in the desktop MVP provides durability only on that workstation. It does not provide production identity recovery, shared real-time collaboration, cross-platform access, or full delivery-channel notifications.

**Post-design documentation gate:** The consolidated design preserves the stated boundaries and scope distinctions; no constitution exception is proposed. This is a documentation assessment, not evidence of passing runtime tests or human acceptance of every generated detail. Product questions listed in `research.md` remain explicit decision points. Diagram-validation evidence and limits are recorded in `design.md`.

## Architecture and Structure Decision

See [research.md](./research.md) for decisions, rationale, alternatives, source references, and remaining product questions. See [architecture.md](./architecture.md) for the component architecture, MVP/full-product boundary, deployment analysis, tradeoffs, and assumptions; see `diagrams/` for PlantUML sources.

The proposed Lab 7 Java code organization is one Maven project using the standard source/test layout and explicit package boundaries:

```text
splitsmart/
├── pom.xml
├── src/main/java/.../domain/         # entities, value objects, domain policies
├── src/main/java/.../application/    # use cases and ports; no Swing/database code
├── src/main/java/.../persistence/    # MVP SQLite adapters
├── src/main/java/.../desktop/        # Swing views and event handlers
└── src/test/java/.../                # JUnit 5 unit and integration tests
```

The MVP uses one build with clear package boundaries; a future API server may become a separate module or project if its delivery requires it. Web, Android, and iOS clients represent the complete-product direction and are not delivered by the desktop MVP. Future clients must use the shared service contract and preserve equivalent business outcomes. No Java source is created by this planning task.

## Complexity Tracking

No constitution violations are proposed. Package boundaries separate Swing from business behavior and preserve a path to the separately required web and mobile experiences without imposing unnecessary MVP build modules.

## Consolidated implementation and verification sequence

This sequence describes future implementation, not work performed by this documentation task.

1. Implement pure domain classes/policies and JUnit 5 tests for the explicit payer, positive precision-valid amounts, exact nonnegative percentage total, largest-remainder allocation, and configurable approvals. Keep proposed tie-break and approval details distinguishable from confirmed clarifications.
2. Implement SQLite repositories and one transaction interface. Add immutable decisions/payments/postings, durable OperationReceipt uniqueness, and transactional outbox/delivery records. Use BEGIN IMMEDIATE for local financial writes and retry whole transactions on conflicts; never hold locks across UI/network interactions.
3. Implement application services that check current authorization, resolve identical retries before debt/state validation, serialize approval finalization, and reject concurrent overpayments against current postings. Add rollback, competing approval/payment, lost-response retry, and removal-race adapter tests.
4. Connect Swing views and local in-app dispatcher; keep work off the Event Dispatch Thread. Validate local restart durability and skipped notifications for removed members. Record email recovery, real remote notification, and FR-012 gaps explicitly.
5. Separately deliver the proposed REST modular monolith/PostgreSQL adapter and private receipt storage for web/Android/iOS. Implement group access locking, stable debt-pair guards, current resource authorization, and worker recovery without adopting event sourcing or CQRS. Validate the same business scenarios through each client.
6. Run future Maven/JUnit 5 checks, platform acceptance, agreed-load timing, recovery exercises, and PlantUML rendering. Human reviewers check FR/NFR/AC traceability and limitations before implementation acceptance. See `quickstart.md` and `design.md` for verification cases and current evidence.
