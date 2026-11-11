# SplitSmart Planning Research

## Decisions

**Consolidation:** Architecture 1 is selected; see [architecture-review.md](./architecture-review.md) and [design.md](./design.md). Existing source references below are retained planning references, not newly verified research or implementation evidence.

### 1. Keep the Java Swing MVP local and desktop-only

**Decision**: The Lab 7 MVP is a Java 21 Swing application with a local SQLite database. Store user data under the operating system's per-user application-data directory rather than beside the installed executable. Package per target desktop operating system with `jpackage` when a distributable is required.

**Rationale**: This supports durable single-workstation demonstration workflows while avoiding a service, identity-provider, networking, and operations footprint that the requested desktop MVP does not require. It does not imply shared remote access, cross-device collaboration, full notifications, or satisfaction of FR-012.

**Alternatives considered**: A shared remote service would enable collaboration but brings deployment and account lifecycle complexity. A runnable JAR is simpler for development but requires a compatible JDK/runtime on each user's workstation.

**Source references**: [SQLite: Appropriate Uses for SQLite](https://sqlite.org/whentouse.html); [Oracle Java 21 `jpackage`](https://docs.oracle.com/en/java/javase/21/docs/specs/man/jpackage.html).

### 2. Use a single Maven project with explicit package boundaries for the MVP

**Decision**: Start with one Java 21 Maven project and standard Maven source/test layout. Keep domain, application use cases, ports, Swing presentation, and SQLite adapters in distinct packages with one-way dependency direction. Split into Maven modules only if team size or build boundaries demonstrate a need.

**Rationale**: Package-level separation is sufficient to keep business behavior testable without GUI interaction, while avoiding unnecessary multi-module overhead in a small academic MVP. Swing event handlers delegate to application use cases; potentially slow work runs off the Event Dispatch Thread, and UI updates return to that thread.

**Alternatives considered**: Multiple Maven modules provide enforceable compile-time boundaries but add module/build configuration. A single flat package or listeners that contain business and database work are smaller initially but undermine the constitution's separation principle and can make the UI unresponsive.

**Source references**: [Maven standard directory layout](https://maven.apache.org/guides/introduction/introduction-to-the-standard-directory-layout.html); [Oracle Swing model architecture](https://docs.oracle.com/javase/tutorial/uiswing/components/model.html); [Oracle: The Event Dispatch Thread](https://docs.oracle.com/javase/tutorial/uiswing/concurrency/dispatch.html).

### 3. Preserve a clear route to the complete multi-platform product

**Decision**: Treat the future product as web-first, with separate web, Android, and iOS clients using one versioned API. Use a Java 21 modular monolith service as the proposed authoritative policy and authorization boundary, backed by PostgreSQL and private receipt object storage. Notification delivery uses an outbox and worker. The exact client frameworks, identity provider, host, and platform release matrix remain open.

**Rationale**: A shared API avoids independently authoritative implementations of allocation, approval, and payment rules. A modular monolith gives the required clients shared state and centralized authorization without starting with distributed-service operational overhead.

**Alternatives considered**: Duplicating business rules in every client risks behavioral drift. Microservices, event streaming, and dedicated reporting infrastructure are premature without scale or operational evidence. A shared Java library alone does not make browser, Android, and iOS clients share one source of truth.

**Source references**: [OpenAPI Specification](https://spec.openapis.org/oas/latest.html); [Martin Fowler: Monolith First](https://martinfowler.com/bliki/MonolithFirst.html).

### 4. Make money, approval, and settlement outcomes authoritative and testable

**Decision**: Use `BigDecimal`, explicit currency scale, deterministic largest-remainder allocation, immutable approval decisions and ledger postings, and atomic/idempotent payment recording. Exercise the same business outcome examples through automated rule tests and client/service contract tests.

**Rationale**: Exact reconciliation, no balance effect before completed approval, and exactly-once payment history are explicit requirements. An immutable posting history makes pairwise balances auditable and recomputable.

**Alternatives considered**: Binary floating point can introduce monetary drift. Mutable aggregate balances alone are less auditable and can be harder to repair. Client-side balance computation cannot serve as the full product's authorization or consistency boundary.

**Source references**: [requirements.md](./requirements.md), especially FR-005, FR-008–FR-010, NFR-001, and NFR-004; [data-model.md](./data-model.md).

### 5. Adopt transactional safeguards within the modular monolith

**Decision:** Retain local SQLite, one Maven project/package boundaries, and the separately proposed REST/PostgreSQL direction. Add durable OperationReceipt records and unique posting identities, serialized latest-debt checks, current authorization, immutable audit records, and an atomic submission outbox. See `design.md` for lock order and adapter-specific behavior.

**Rationale:** These address duplicate/reordered requests, competing settlements, removed-member access, and notification failures without requiring an event-sourced domain model. Expense/payment records and ledger postings remain canonical; reports use ordinary authorized SQL.

**Alternatives not selected:** Architecture 2's canonical event store, event replay, CQRS projections, group-stream serialization, connected PostgreSQL Swing MVP, and multi-module Maven reactor. The unchanged `alternative-architecture.md` and `architecture-review.md` preserve the comparison and selection rationale.

**Tradeoffs:** Additional uniqueness/receipt/outbox tables and transaction tests cost implementation effort. SQLite serializes local writes; PostgreSQL uses narrowly scoped pair guards plus access/expense locks. Notifications remain eventually delivered; external provider duplicates cannot universally be excluded. Private receipt retrieval is service-mediated to avoid download links surviving group removal. Full-product client toolchains must comply with the Java 21 constitution or obtain an amendment before conflicting application-code choices are adopted.

**Evidence:** Design reasoning and the project-owner architecture choice only; no runtime, concurrency, security, deployment, or performance results are claimed.

## Planning assumptions and decisions still requiring product input

These are not technical blockers to this baseline, but require confirmation before a production implementation or conformance claim:

- Currency list, supported precision, and whether cross-currency sharing or settlement is required. Baseline stores one currency per expense and does not convert currencies.
- The deterministic tie-break is participant ID ascending after comparing fractional remainders. The specification leaves the tie-break rule open; this is a proposed stable default, not an approved customer policy.
- Approval policy administration, reviewer eligibility/order, and whether submitters may approve. Baseline captures a policy version per expense; default self-approval is disabled unless group policy says otherwise.
- Identity assurance beyond email/password, recovery safeguards, account deletion, data retention, and legal/regulatory controls.
- Receipt formats, limits, malware/content validation, retention, and full-product access mechanism.
- Notification channels/preferences and supported web browser, Android, and iOS versions.
- Required report filters, exports, and sharing permissions.
- Directional versus reciprocal-netting presentation, date semantics for historical balances, and settlement handling for outstanding obligations involving removed members. No access restoration or debt cancellation is implied by removal.
- OperationReceipt/outbox/audit retention, deletion rules, and receipt delivery details; retry-record expiry must not reopen duplicate financial effects.
- Target hosting provider, availability, scale, operational ownership, backup/recovery objectives, and performance measurement environment.
- Whether a production-like collaborative pilot requires advancing the shared backend into the desktop MVP. Current MVP assumption is local-only; changing this alters scope.

None of these assumptions narrow the original customer platform requirement. See [architecture.md](./architecture.md) for deployment implications and [contracts/service-contract.md](./contracts/service-contract.md) for the proposed API boundary.
