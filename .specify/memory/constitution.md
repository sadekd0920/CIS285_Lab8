<!--
Sync Impact Report
Version change: unratified template → 1.0.0
Modified principles: none (initial constitution)
Added sections: Core Principles; Implementation Constraints; Development Workflow
Removed sections: none
Follow-up TODOs: Confirm and replace TODO(RATIFICATION_DATE) with the original adoption date.
-->
# SplitSmart Constitution

## Core Principles

### I. Customer Requirements Are Authoritative
Customer requirements in `UserRequirement.md` define required product outcomes and MUST
remain distinct from implementation decisions. Every specification and delivery review MUST
preserve traceability from each customer requirement to its design, implementation, and
verification. An implementation choice MUST NOT be described as a customer requirement or
used to silently narrow, replace, or waive one. Any conflict or proposed change to a customer
requirement MUST be documented and approved by the customer or authorized product owner.

### II. Java 21 and Maven
Application code governed by this constitution MUST use Java 21, and Maven MUST manage
builds, dependencies, and test execution. The Maven build MUST declare its Java release
target as 21 and use reproducible, project-defined dependency versions. A proposed change
to either choice is an implementation decision and requires an approved constitution
amendment before adoption.

### III. Object-Oriented Design
The Java application MUST use object-oriented design to give domain concepts clear
responsibilities and explicit boundaries. Classes and interfaces MUST have cohesive
responsibilities; dependencies MUST be explicit; and avoidable duplication and unnecessary
coupling MUST be reduced. Design complexity MUST be justified by product needs rather than
introduced speculatively.

### IV. GUI and Business Logic Separation
GUI code MUST be separated from business rules and domain behavior. Business logic MUST be
invocable and testable without constructing or displaying GUI components. Swing event
handlers MUST delegate work to appropriate application or domain components and MUST NOT
own balance calculations, expense approval rules, or other core business decisions.

### V. Swing Is an Implementation Choice, Not a Platform Requirement
Where a Java desktop GUI is delivered, it MUST use Java Swing. Swing is an implementation
decision and MUST NOT be represented as fulfilling the customer requirement for a
web-based product and Android and iOS apps. Before claiming customer-platform conformance,
the approved architecture MUST explain how each of those customer-required platforms will
be delivered. Any platform change or limitation requires explicit product-owner approval.

### VI. Exact Monetary Values
All monetary amounts and calculations MUST use `BigDecimal`, with explicit scale and
rounding rules at defined calculation or display boundaries. Binary floating-point types
MUST NOT be used for money. Tests MUST cover relevant rounding and split-allocation
behavior so that persisted and reported totals reconcile.

### VII. Automated Verification with JUnit 5
Automated Java tests MUST use JUnit 5. Business rules, including expense allocation,
approval effects, balance updates, and payment settlement, MUST have focused automated
tests. A change is not complete until its relevant tests pass; tests MUST not depend on
manual GUI interaction to verify core business behavior.

### VIII. Diagrams Communicate Design
Architecture and interaction diagrams maintained as project design artifacts MUST use
PlantUML source. Diagrams MUST be updated when a change materially alters the represented
architecture or interactions, and MUST remain consistent with the implemented design.

### IX. Traceability and Human Review of AI Output
Each customer requirement MUST be traceable through the relevant specification and
implementation to verification evidence. AI-generated requirements, designs, diagrams,
code, and tests MUST be reviewed and accepted by a human before they are treated as
authoritative or merged. Reviewers MUST verify correctness, requirement alignment, and
consistency with this constitution; AI output is assistance, not approval.

## Implementation Constraints
`UserRequirement.md` is the source of customer requirements for user accounts, group
management, expense creation and approval, notifications, balances, payments, reporting,
and web/Android/iOS platform support. These are product outcomes, not mandated
technologies. Java 21, Maven, Swing where a Java desktop GUI is delivered, JUnit 5,
PlantUML, object-oriented design, GUI/business-logic separation, and `BigDecimal` are
implementation decisions governed by this constitution. A desktop Swing application
alone does not satisfy the requested web and mobile platforms. If a technical constraint
prevents a customer outcome, the gap MUST be made explicit and approved rather than
silently treated as compliant.

## Development Workflow
Specifications and implementation plans MUST identify the customer requirements they
address and distinguish those requirements from proposed technical choices. Changes MUST
include JUnit 5 tests for affected business behavior and update applicable PlantUML
diagrams. Reviews MUST check requirement traceability, monetary precision, GUI/business
logic boundaries, and human validation of AI-assisted contributions before approval.

## Governance
This constitution governs project specifications, design, implementation, and review.
Amendments require a documented rationale and approval by the authorized project
stakeholder. Each amendment MUST update the version and last-amended date. Versioning
follows semantic versioning: MAJOR for incompatible governance changes, MINOR for new or
materially expanded principles, and PATCH for clarifications that do not change
requirements. Every change review MUST identify applicable principles and record any
approved exception; exceptions MUST state their scope, rationale, and approving
stakeholder. Compliance is reviewed during design and code review, and customer
requirement changes remain subject to customer or authorized product-owner approval.

**Version**: 1.0.0 | **Ratified**: TODO(RATIFICATION_DATE): confirm original adoption date | **Last Amended**: 2026-10-29
