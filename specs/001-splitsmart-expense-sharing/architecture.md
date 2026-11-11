# SplitSmart Selected Architecture

**Lab 7 decision (2026-11-04):** Architecture 1 is selected by the project owner in [architecture-review.md](./architecture-review.md). This consolidates the baseline with selected correctness safeguards; [alternative-architecture.md](./alternative-architecture.md) remains an unchanged, unselected comparison artifact. See [design.md](./design.md) for responsibilities, interfaces, transaction flows, and traceability.

## Purpose and scope boundary

This is a baseline design for CIS 285 Lab 7, not an implementation commitment. It distinguishes:

1. **Java Swing desktop MVP**: a Java 21/Maven demonstration with local durable storage and a deliberately limited feature set.
2. **Complete customer product**: a web-first product with Android and iOS apps, sharing one backend contract and consistent business outcomes. A Swing application alone does not satisfy FR-012.

The requirements source remains [UserRequirement.md](../../UserRequirement.md), with traceability and testable behavior in [requirements.md](./requirements.md). Product decisions left open in the specification remain open; the proposed technical defaults below do not change customer requirements.

## Component architecture

### Shared domain and application core

- **Domain**: account, group, membership, expense, allocation, approval policy/decision, ledger posting, payment, and reporting value objects and invariants.
- **Application services**: use cases such as submit expense, decide approval, list balances, record payment, and generate report. They coordinate repositories and transaction boundaries and perform authorization checks.
- **Ports**: repository, current-user/authentication, receipt storage, notification, and transaction interfaces. Business rules depend on these interfaces rather than Swing, SQL, or a delivery framework.
- **Monetary rules**: `BigDecimal` calculations use a currency-specific scale. Allocation uses largest fractional remainder and a consistent tie-break. Participant ID ascending is a proposed default only; the exact rule remains open in the specification. Every persisted allocation sums exactly to its expense total.

### Java Swing MVP

- **Swing presentation**: screens/forms for sign-in, groups, expense submission/review, balances, payments, and basic reports. Views collect input and render results only; event handlers call application services and display returned validation errors. Potentially slow work runs off the Event Dispatch Thread, and UI updates are posted back to it.
- **Application/domain**: same separation and business policies as proposed for the complete backend; core logic remains independently testable.
- **Local persistence adapter**: SQLite-backed repositories persist records in the user's operating-system application-data directory. Schema versions are maintained by the persistence package. Sensitive data and receipt paths remain local to the workstation.
- **Local notification adapter**: submission and a durable outbox row commit together in SQLite. A local dispatcher creates unique in-app notices after rechecking membership; it resumes pending work after restart. Delivery to users on other devices is not implied by this MVP.

The MVP is suitable for a lab demonstration and local workflows. SQLite does not provide a shared service for concurrent remote users. A multi-person scenario can be demonstrated in one local dataset, but it does not establish production-grade identity, remote notification, or platform support. Account recovery via email, cross-device synchronization, cloud receipt delivery, and complete operational security are outside this MVP baseline unless separately added to its approved scope.

### Complete-product direction

- **Web, Android, and iOS clients**: platform-specific presentation and navigation; all call the same versioned service API. UI technology choices are intentionally not fixed by this design.
- **API/application server**: proposed Java 21 Spring Boot modular monolith exposing authenticated REST operations. It invokes the same domain/application policies used by the desktop demo, rather than duplicating allocation or settlement rules in clients.
- **Identity and authorization**: email/password account lifecycle and email recovery through a managed identity component or a carefully operated identity module; service authorization is checked on every resource operation. Selection between these is a deployment/security decision.
- **Transactional persistence**: PostgreSQL stores accounts, groups, memberships, approval policy, expenses, allocations, decisions, payments, and durable notification/outbox state. Ledger effects and approval/payment state changes commit atomically.
- **Receipt storage**: private object storage for image bytes; database stores protected object identifiers and validated metadata. Retrieval is service-mediated with current authorization on each request; avoid reusable download links that remain valid after membership removal. Upload protocol and image limits remain open.
- **Notifications**: transactional outbox records events alongside committed business changes. A worker delivers eligible notices through channels selected by product policy; delivery failure is retried and does not roll back a successful expense submission.
- **Reporting**: authorized queries over canonical records, with date and parameter filters. Initial reports can be served from PostgreSQL; a separate analytics store is deferred until measured scale justifies it.

The API is the enforcement boundary, not the user interface. Client-side checks improve usability but never replace server-side authorization or validation. A stable API version and shared contract tests help keep outcomes aligned across platforms (NFR-004).

## Expense and balance behavior

1. Expense submission validates group membership, participants, explicit payer, positive amount, currency precision, and split.
2. The expense is saved as `PENDING_REVIEW`; its allocations may be displayed but create no balance effect.
3. Approval decisions are recorded against configured approval levels. A rejection terminates the workflow without ledger postings. Completing all required levels transitions the expense once to `APPROVED`.
4. The approval transaction writes balanced pairwise ledger postings (each non-payer participant owes the payer their allocated share) and an outbox event. The payer's own share is not a debt to themselves.
5. A payment must identify a debtor/payee pair and amount no greater than the latest outstanding amount. Inside the transaction, recheck current access, check the durable retry record, lock the debt pair, and only then calculate debt and validate the amount. Persist payment, offset posting, and retry result atomically. Require a stable idempotency key for payment/decision requests (an equivalent application request ID in Swing).
6. Balances are derived from immutable approved-expense postings and payment postings, grouped by currency, debtor, creditor, and group. Ordinary SQL queries over these records provide balances and reports; no separate CQRS projections are selected.

## Consolidated transactional safeguards

- **Retries:** store an OperationReceipt keyed by actor, operation, target, and idempotency key, with a request hash and result reference. After current authorization, return an identical successful retry before validating mutable workflow/debt state. Reject a key reused with different input. Unique payment/decision/posting constraints protect financial effects even across competing keys.
- **Final approval:** lock the expense, validate the captured policy version and eligible reviewer, and commit the immutable decision, final status, all participant debt postings, and operation receipt together. Approval and rejection serialize on the same expense; a rejected or already-finalized expense cannot acquire a second ledger effect. Never hold a transaction open while a human decides.
- **Concurrent debt:** use a stable DebtPairGuard row per group/currency/directional debtor/creditor pair in PostgreSQL, including pairs without postings. All final approvals and payments acquire affected pair guards in stable order. Compute debt from committed postings after locking, using READ COMMITTED queries. SQLite uses a short BEGIN IMMEDIATE write transaction for the same business boundary; retry busy/conflict failures as whole operations with the same request ID.
- **Current authorization:** all reads, receipts, reports, notices, and writes check active membership. In PostgreSQL, writes take a shared group-row lock before access checks; membership/role changes take an exclusive group-row lock, establishing an order against removal. SQLite's write transaction provides that ordering for local writes. Reads use current membership in the same statement/snapshot as data selection; subsequent requests after removal fail. Already retrieved data cannot be recalled.
- **Audit and delivery:** decisions, accepted payments, and postings are append-only through application/database permissions. Submission stores its outbox item in the expense transaction; a worker retries delivery and uniquely records each notice. Recheck current membership before dispatch and inbox retrieval. Skip removed recipients; provider failure never reverses an expense. External notices contain minimal detail because an in-flight message cannot be retracted.

Lock order is group access guard, operation-receipt reservation, expense when relevant, then sorted debt-pair guards. A concurrent duplicate receipt reservation waits or retries after rollback and re-reads the committed receipt. No external mail, storage, or user interaction occurs while holding financial locks. Full details and failure handling are in [design.md](./design.md).

These are transactional refinements within one modular monolith. Canonical entity tables, immutable ledger postings, and approval/payment history remain authoritative. There is no canonical event store, domain event replay, CQRS read store, or connected PostgreSQL desktop MVP. Outbox messages are delivery work, not a reconstructable domain history.

## Deployment analysis

| Concern | Lab 7 desktop MVP | Complete product baseline |
|---|---|---|
| Runtime | Java 21 desktop app, packaged per OS with `jpackage`; SQLite stored in the user's application-data directory | Java 21 API service in a managed container/runtime; browser and mobile clients deployed independently |
| Database | SQLite file owned by the local app/user profile | Managed PostgreSQL with backups, migrations, monitoring, and access controls |
| Receipts | Local files with database metadata; strict size/type validation | Private object storage, encryption and retention policy, authenticated retrieval |
| Notifications | In-app only; no cross-device delivery | Outbox + worker and selected email/push/in-app providers |
| Availability and collaboration | Single workstation; no shared concurrent access or synchronization | Shared service with authenticated clients; horizontal API scaling is possible after measurement |
| Operations | Installer/runtime compatibility, local backup and restore, local-data protection | TLS, secret management, observability, database recovery, rate limiting, deployment/rollback, provider operations |
| Customer-platform coverage | Desktop demo only; does not satisfy FR-012 | Web-first, Android, and iOS are required deliverables and require platform acceptance tests |

### Deployment choice and alternatives

- **Chosen baseline for Lab 7**: desktop client + local SQLite. It is low-friction for a Java Swing learning deliverable, supports durable expense/payment history on one workstation, and avoids pretending that a cloud service already exists.
- **Desktop distribution**: use `jpackage` per supported OS when a self-contained installer is needed; target-specific packages must be built and verified on their target operating systems. A runnable JAR remains convenient for development.
- **Chosen direction for the complete product**: a modular monolith REST service + PostgreSQL + private object storage + notification worker. This enables centralized authorization and consistent cross-platform outcomes while keeping the initial operational model simpler than microservices.
- **Rejected for the MVP**: production cloud service setup, because it expands Lab 7 into operations, identity, network, and deployment work. It may be revisited if a real collaborative pilot is required.
- **Rejected as a full-product target**: independent native business logic in each client, because it risks inconsistent approval, rounding, and payment results. Clients should render and validate ergonomically; authoritative rules belong in the service.
- **Deferred**: microservices, event streaming, dedicated analytics infrastructure, cross-currency support, and offline synchronization. No scale or service-level evidence currently justifies their complexity.

## Tradeoffs and assumptions

- A local SQLite MVP is straightforward and durable, but data is machine-local and does not meet full-product access, notification, or collaboration expectations.
- A modular monolith centralizes rules and is easier to operate initially, but it may require later decomposition if independently scaling workloads become necessary.
- Keeping immutable postings and deriving balances improves auditability and recalculation, but reports may require indexes or cached aggregates as data grows.
- An outbox makes notifications resilient to transient provider errors, at the cost of eventual rather than synchronous delivery.
- Currency precision, currency list, account verification/security controls, image formats/limits/retention, supported browser/OS versions, detailed approval-policy administration, report exports, and normal performance-test conditions remain product-owner decisions (see [research.md](./research.md)).
- The API/server framework, exact hosting provider, client frameworks, identity provider, and service sizing are proposed defaults, not customer requirements.
- The default approval policy prohibits self-approval unless group policy explicitly allows it, matching the current specification assumption.
