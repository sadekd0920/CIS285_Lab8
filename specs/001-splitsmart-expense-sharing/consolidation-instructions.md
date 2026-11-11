# CIS 285 Lab 7 — Consolidation Instructions

**Decision:** Use the Spec Kit modular-monolith baseline (`architecture.md`) as the selected SplitSmart architecture. The alternative event-sourced/CQRS proposal (`alternative-architecture.md`) is a comparison artifact, **not** the target architecture.

## Authoritative inputs

Read `UserRequirement.md`, `.specify/memory/constitution.md`, `specs/001-splitsmart-expense-sharing/spec.md`, `requirements.md`, `plan.md`, `architecture.md`, `data-model.md`, `research.md`, `contracts/service-contract.md`, `quickstart.md`, `diagrams/`, `alternative-architecture.md`, and `architecture-review.md`.

## Required work

1. **Preserve customer scope and decisions.** Keep all original features and FR-001–FR-012, NFRs, acceptance criteria, and the five confirmed clarifications. Do not edit `UserRequirement.md`. Do not silently treat implementation choices as customer demands. Keep the full web, Android, and iOS product distinct from the limited Swing MVP.
2. **Consolidate Architecture 1.** Use Java 21, Maven, Swing UI separated from domain/application services, BigDecimal money, largest-remainder allocation, JUnit 5, and PlantUML. Preserve the local SQLite MVP and the separately proposed Java REST/PostgreSQL full-product design.
3. **Selectively incorporate safeguards, not architectural replacement.** Explicitly document idempotent payment/approval handling, concurrent debt validation, current authorization checks, immutable financial and approval audit records, and a transactional notification outbox. Do **not** introduce a canonical event store, event replay, CQRS projections, or a connected PostgreSQL MVP unless identified as an unselected alternative.
4. **Update consolidated docs** in this feature directory: `requirements.md`, `architecture.md`, `data-model.md`, `plan.md`, `research.md`, `contracts/service-contract.md`, and `quickstart.md` as needed for consistency. Create `design.md` summarizing class responsibilities, interfaces, activity flows, decisions, and traceability. Update relevant `diagrams/*.puml` for a class diagram plus expense-approval and payment-settlement activity diagrams. Do not overwrite `alternative-architecture.md` or `architecture-review.md`.
5. **Verify consistency.** Cross-check FR/AC/NFR traceability; explicit payer vs submitter; admin-only membership changes; positive amounts and exact percentage total; fair rounding; email/password and recovery; approval-before-balance changes; full/partial payments up to debt applied once; receipt/privacy constraints; and full-platform scope. Flag unresolved policy choices as open questions rather than inventing answers.
6. **Report changes.** Provide a concise summary identifying updated documents, retained alternatives, unresolved issues, and any PlantUML rendering or tests that could not be executed.

## Constraints

- Documentation and PlantUML source only; **do not create Java code**.
- Do not modify `UserRequirement.md` or `.specify/memory/constitution.md`.
- Do not delete existing baseline or alternative design artifacts.
- Do not claim tests, deployed services, rendered diagrams, or supported mobile apps that were not actually verified.
- Before committing, present changed-file list and any material conflicts for human review.

## Suggested Codex/Copilot instruction

Read `specs/001-splitsmart-expense-sharing/consolidation-instructions.md` and `architecture-review.md`. Consolidate the SplitSmart documentation according to those instructions. Architecture 1 is selected. Update documentation and PlantUML source only, preserve the original requirements and alternative architecture, and report changed files and unresolved questions before committing.
