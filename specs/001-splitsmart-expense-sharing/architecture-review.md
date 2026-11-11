# CIS 285 Lab 7 — Architecture Comparison and Review Report

**Date:** 2026-11-04  
**Project:** SplitSmart  
**Review decision:** Select Architecture 1 (Spec Kit baseline) and selectively incorporate proven safeguards from Architecture 2.  
**Reviewed sources:** `UserRequirement.md`, `spec.md`, `requirements.md`, `plan.md`, `architecture.md`, and `alternative-architecture.md`.

## 1. Purpose and review method

Compare the two documented proposals against the existing customer requirements (FR-001–FR-012), confirmed clarifications, Java 21/Maven/Swing lab constraints, and maintainability, accuracy, security, deployment feasibility, and traceability. This is a documentation/design review, not implementation testing. The PlantUML source exists, but rendered diagrams and runtime behavior were not independently validated.

## 2. Architecture 1 — Spec Kit baseline

**Approach:** A modular monolith with a presentation-independent Java 21 domain/application core, ports and adapters, and strong separation of Swing from business logic. Its *lab MVP* uses Swing and local SQLite; its separately identified *complete-product direction* uses a shared Java 21 REST backend, PostgreSQL, private receipt storage, and a notification outbox for web, Android, and iOS clients.

**Strengths**
- Lower conceptual and operating overhead for a course-level MVP: one Maven project with clear package boundaries.
- BigDecimal monetary arithmetic, largest-remainder allocation, approval-before-ledger posting, atomic settlement, immutable financial postings, and authorization boundaries already address the most important correctness risks.
- Straightforward JUnit 5 unit and integration test seams and explicit traceability to requirements.
- Honestly identifies the desktop demo as *not* satisfying the web/Android/iOS requirement (FR-012).

**Limitations**
- Local SQLite is single-workstation and cannot demonstrate real multi-device collaboration or real cross-device notifications.
- Password recovery, distributed deployment, and full platform delivery require additional work.
- Ledger postings audit money movement, but the baseline does not make all group domain events the canonical replayable history.
- Full-product infrastructure is a proposed direction, not an implemented or validated deployment.

## 3. Architecture 2 — Codex alternative

**Approach:** Event sourcing with group-scoped command handling and CQRS query projections. Ordered domain events become the reconstructable group history. Financial projections update synchronously in the same transaction. The proposed MVP connects Swing to a Java service with PostgreSQL rather than a local SQLite store.

**Strengths**
- Detailed replay and reconstruction of approval, membership, and settlement history.
- Explicit concurrency, retries, command receipts, projection consistency, and idempotency safeguards.
- Shared service allows a collaborative desktop demonstration.

**Limitations**
- More moving parts for a lab: event schema versioning, replays, query projections, stream concurrency, and a running service/database.
- Event retention and private-data deletion become harder.
- Group-level serialization can become a hot spot.
- Greater implementation and operational costs without demonstrated workload requiring these mechanisms.
- PlantUML text was written but rendering was not validated.

## 4. Comparison

| Criterion | Architecture 1 — baseline | Architecture 2 — alternative | Review |
|---|---|---|---|
| Requirements coverage | Retains FR-001–FR-012 and the full-platform distinction | Retains FR-001–FR-012 and the full-platform distinction | Both adequate in documented scope |
| Financial accuracy | BigDecimal, balanced immutable postings, atomic approval/settlement | Same invariants plus event transactions and synchronous projections | Both strong |
| Auditability | Financial postings and persisted decisions | Replayable full domain event history | Alternative is broader, but costs more |
| Complexity | Lower; modular monolith, local MVP | Higher; event store, CQRS, projection/replay rules | Baseline better for this lab |
| MVP practicality | Swing + SQLite on one workstation | Connected Swing + Java service/PostgreSQL | Baseline easier to demonstrate |
| Security | Explicit authorization and protected receipt/identity boundaries | Additional detailed current-access and replay/privacy controls | Adopt useful details from alternative |
| Scalability | Modular server can evolve after measurements | Group concurrency and query projections with group hotspots | No proven need for event sourcing |
| Testability | JUnit 5 and API/adapter tests | JUnit 5 plus replay/concurrency/projection tests | Baseline easier to verify now |
| Deployment | Local MVP, separate web/mobile target | Connected MVP, separate web/mobile target | Neither MVP satisfies FR-012 |

## 5. Decision and rationale

**Selected: Architecture 1 — Spec Kit baseline.** It fulfills the lab's architecture/design-documentation purpose with fewer assumptions and fewer operational dependencies, keeps money and approval rules centralized, and supports future web/mobile delivery. The alternative demonstrates useful advanced patterns but its event store and CQRS complexity are not justified by a defined scale or resilience requirement.

**Retain these safeguards from Architecture 2 without adopting event sourcing/CQRS:**
1. Use durable idempotency keys or equivalent uniqueness controls for payment and final-approval retries; ensure each accepted action affects the ledger once.
2. Serialize or transactionally guard concurrent settlements against the latest outstanding pairwise balance, rejecting overpayment.
3. Recheck *current* group membership/authorization for group reads, receipts, notifications, and write operations.
4. Preserve an immutable audit trail of approval decisions, expense postings, and payments, with transactionally consistent records.
5. Keep the notification outbox and avoid notifying removed members; notification failure must not reverse a committed expense.

These are implementation/design refinements; they do **not** change customer requirements or require the complete event-sourcing model.

## 6. Risks, unresolved decisions, and verification

- Confirm currency set and scale, approval configuration, receipt limits, email recovery provider, supported platforms, and reporting parameters with stakeholders before production commitments.
- Validate PlantUML syntax/rendering and ensure a class diagram plus expense-approval and payment-settlement activity diagrams exist and match the selected architecture.
- Require requirement-ID traceability, JUnit 5 test plans for rounding, approval and partial payments, and explicit deployment differences between desktop demo and complete product.
- Do not claim customer delivery of web, Android, or iOS from a Swing-only MVP.
- No Java implementation, performance run, security audit, diagram-rendering test, or deployment test was performed as part of this review.

## 7. Human review decision

The project owner selected Architecture 1 on 2026-11-04. Consolidation should update the authoritative architecture/design documentation to reflect that choice while keeping both original alternatives and this report as review artifacts.
