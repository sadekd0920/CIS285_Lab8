# SplitSmart Consolidated Design — CIS 285 Lab 7

**Date:** 2026-11-04  
**Selection:** Architecture 1, per [architecture-review.md](./architecture-review.md) and [consolidation-instructions.md](./consolidation-instructions.md). Architecture selection is confirmed; implementation details remain a proposal for human review.

## 1. Design boundary and decisions

The selected design is a modular monolith with a presentation-independent object-oriented Java 21 core, one Maven project with package boundaries, BigDecimal monetary arithmetic, JUnit 5, and PlantUML. The Swing desktop MVP uses local SQLite and local receipt files in the user's application-data directory. The complete product separately requires a primary web experience, Android app, and iOS app using a proposed Java 21 REST service, PostgreSQL, private receipt storage, and a notification worker.

The Swing MVP cannot satisfy FR-012. Local account switching and in-app notices demonstrate one-workstation behavior only; email recovery, remote notifications, collaboration, and complete security/platform acceptance remain delivery gaps unless separately implemented and verified. Client frameworks, supported versions, hosting, and identity provider are not selected here; governed application code must comply with the constitution or an approved amendment.

Adopt five safeguards from Architecture 2: durable retry/uniqueness controls, transactionally serialized latest-debt validation, current authorization, immutable financial/approval audit, and submission outbox delivery. These refine existing requirements. Do not adopt a canonical event store, event replay, CQRS projections, group-stream processing, connected PostgreSQL MVP, or multi-module reactor. Ordinary entity records plus immutable ledger postings remain authoritative; balance/report queries use those records.

[alternative-architecture.md](./alternative-architecture.md) and the review report remain unchanged comparison/decision artifacts. [UserRequirement.md](../../UserRequirement.md), [constitution.md](../../.specify/memory/constitution.md), and [spec.md](./spec.md) retain their original authority and wording.

## 2. Classes, interfaces, and dependencies

| Class/component | Responsibility and dependencies |
|---|---|
| Account / Group / GroupMembership | Individual identity, group context, creator/admin/member roles, and active membership; removal preserves historical references. |
| Expense / ExpenseAllocation | Amount/date/description, submitter, one payer among participants, optional receipt, exact shares, captured approval policy, and workflow state. Payer may differ from submitter. |
| Money / AllocationPolicy | BigDecimal amount/currency and supported scale; equal or percentage allocation with largest fractional remainder. Reject nonpositive amounts, excess precision, negative percentages, or totals other than exactly 100%. |
| ApprovalPolicy / ApprovalDecision | Versioned required levels and reviewer eligibility, immutable decisions and audit timestamps. Default self-approval is disabled unless group policy explicitly allows it. Detailed eligibility/quorum options remain open. |
| LedgerPosting / Payment | Append-only approved-expense contributions and settlement offsets; immutable accepted payment identity, debtor, creditor, currency, amount, paidAt, and actor. No payment processor or real fund transfer is introduced. |
| Receipt | Validated attachment metadata and private file/object identifier; access follows current expense/group authorization. |
| ExpenseService | Validate submission, persist PENDING_REVIEW and outbox atomically; coordinate review and final posting through transaction/repository ports. |
| PaymentService | Authorize debtor, resolve retries, serialize debt reads/writes, validate latest debt, persist payment and offset atomically. |
| GroupService / AuthorizationService | Creator/admin-only add/remove; current personal/group access checks; membership mutation ordering against writes. |
| BalanceService / ReportService | Authorized ordinary SQL/repository queries over canonical records. Shares sum exactly; report filters are intersected with authorization, never alternatives to it. |
| OperationReceipt | Durable scope/key/hash/result identity for accepted decisions/payments. Supports lost-response retries; not an event log. |
| DebtPairGuard | Stable PostgreSQL lock identity per directional pair; no stored balance. SQLite substitutes its write transaction. |
| OutboxMessage / NotificationDelivery / Notification | Durable delivery work, recipient/channel retry state, and unique in-app inbox item. Local dispatcher for SQLite; full-product worker for PostgreSQL. |

Application services depend on these ports, implemented by local SQLite adapters or future PostgreSQL/HTTP/storage adapters:

| Interface | Illustrative contract (design only) |
|---|---|
| TransactionRunner | Execute a complete use case atomically; begin SQLite write transaction before financial reads or PostgreSQL transaction with specified guards. |
| CurrentActor / AuthorizationPort | Resolve authenticated account; require current access for target and operation. Never trust caller-supplied submitter/reviewer identity. |
| ExpenseRepository / ApprovalRepository | Load/lock expense, save state, append decisions under uniqueness constraints. |
| LedgerRepository / PaymentRepository | Query directional outstanding debt; append unique expense/payment postings and immutable accepted payments. |
| OperationReceiptRepository | Reserve unique scope/key, compare normalized request hash, and save/read accepted result reference within the transaction. |
| GroupRepository / DebtLockPort | Coordinate current membership/removal and acquire stable pair guards in documented order. |
| ReceiptStore | Stage/validate attachment, associate metadata, and retrieve bytes after current authorization; clean orphan uploads without financial locks. |
| OutboxRepository / NotificationSender | Commit delivery work with submission; claim/retry/skip delivery and deduplicate inbox writes. External sender is never part of financial commit. |

Swing event handlers call application services, not SQL or monetary policies. Slow I/O runs off the Event Dispatch Thread; rendering returns to it. The future REST adapter invokes the same policies and derives actor identity from the session. UI checks are conveniences; application/domain checks remain authoritative. This separation supports GUI-free JUnit 5 tests and avoids duplicating approval/payment rules in clients.

## 3. Transaction and lock protocol

### Common write boundary

For PostgreSQL use READ COMMITTED transactions. Acquire locks in this order: shared group access guard; unique operation-receipt reservation for decision/payment; expense row when applicable; debt-pair rows in stable `(group, currency, debtor, creditor)` order. All membership/role changes acquire an exclusive group guard before authorization/mutation. This creates an order between removal and authorized writes without serializing every group operation as an event stream.

DebtPairGuard rows exist independently of ledger amounts, including zero-debt pairs; create them safely under a unique key before locking. All approvals/payments affecting a pair follow this protocol. Calculate debt from newly committed posting rows after acquiring guards; avoid stale transaction snapshots. Uniqueness races, deadlocks, serialization/busy errors cause whole-transaction rollback and bounded retry with the same request ID. Do not hold a lock while waiting for a reviewer, network, storage upload, or provider.

SQLite uses a short BEGIN IMMEDIATE write transaction before access checks, receipt lookup, expense/debt reads, and writes. Its serialized writers provide the local equivalent; do not pretend it supports remote concurrent clients. Any local process touching the dataset must use the same application rules. PostgreSQL pair guards are adapter infrastructure, not a new money aggregate or a replacement for SQLite.

Keep adapter arithmetic exact: SQLite stores canonical decimal TEXT and the Java repository/application boundary aggregates it with BigDecimal, avoiding SQLite floating-point coercion/SUM. PostgreSQL uses exact NUMERIC columns and aggregation. Precision/scale validation applies before persistence in either adapter; no float/double conversions are permitted.

### Idempotency and immutable audit

Scope OperationReceipt by actor/operation/target/key. Normalize semantic input and persist a hash plus result reference. Check current authorization before exposing prior results, then resolve accepted identical retries before mutable-state validation. A full-payment retry must succeed even when current debt is now zero. Different content under an accepted key is a conflict. A concurrent reservation waits/retries, then retrieves the winner's committed result; do not commit a pending receipt without its business action.

Unique decision slots, final expense state, expense/participant posting identities, and payment-offset identities provide independent once-only ledger guards. New payment keys denote new reported payments, so a client must retain the original key across uncertain responses. A successful payment means payment/history/posting/receipt committed together; return its stable identity and outcome, and fetch fresh current balance separately if later operations occurred.

Decision/payment/posting audit is append-only through application and database permissions. Pending/rejected expenses have no postings. Never silently edit accepted history; corrections/reversals require product rules before delivery. Retention/deletion and deduplication expiry remain policy decisions; do not expire safeguards in a way that permits old retries to produce another financial effect.

## 4. Activity flows

### Submission and expense approval

Submission checks active membership, participants including the explicit payer, positive supported precision, exact nonnegative percentage total, date/description, and optional validated receipt. Persist final shares, policy version, PENDING_REVIEW, and outbox/recipient candidates atomically. Largest-remainder allocation is confirmed; participant-ID ascending is only a proposed tie-break. A later rounding/policy change must not rewrite historical shares.

Each decision is a separate transaction. Recheck current eligible reviewer and current expense state under the expense lock. An identical accepted retry returns its receipt. Rejection records its immutable decision and REJECTED status without postings. A partial approval records only its decision. The final required approval locks affected pairs, sets APPROVED, writes each nonzero non-payer debt contribution once, and saves the operation receipt atomically. The payer's own share creates no self-debt. Concurrent approval/rejection requests cannot both finalize the same expense.

The worker delivers submission notices after commit, independently of approval. Capture other active members at submission as candidates and recheck current membership before dispatch; a newly added member is not retroactively a submission recipient. Unique inbox identity prevents duplicate notices. Skip removed candidates, retry transient failures, and resume after restart. A provider outage never undoes expense creation. An already sent external message cannot be recalled; include minimal detail and require authorized retrieval for expense data. External exactly-once delivery is not promised.

Source: [expense-approval.puml](./diagrams/expense-approval.puml).

### Payment settlement

An authenticated debtor selects group, creditor, amount, currency, date, and stable request ID. Begin transaction, acquire access/receipt guards, and recheck current authorization. Return the accepted result for an identical retry before testing current debt. Otherwise lock the stable pair, read current committed outstanding debt, and validate distinct accounts, positive supported-precision amount, matching currency, and amount <= outstanding.

Commit immutable Payment, one payment offset posting, and OperationReceipt together. A failure leaves all unchanged. Two competing payments validate sequentially against the latest debt; the second cannot overpay. A final expense approval uses the same pair guards, so its ordering against payment is consistent. A crash after commit but before response is recovered through the same key, not a new payment.

Directional debt is approved contributions minus offsets, grouped by group/currency/debtor/creditor. No cross-currency or cross-group settlement is assumed. Reciprocal netting policy is unresolved. Full/partial settlements apply immediately to the specified pair and appear once in history. The system records a payment already made; it does not move funds.

Source: [balance-settlement.puml](./diagrams/balance-settlement.puml).

## 5. Security, deployment, and quality

Read authorization joins current membership in the same query/snapshot as selected records. Every subsequent group, report, receipt, notification, or cached-retry request after removal is denied. Historical records remain for current authorized members. In-flight authorized reads/externally sent notices cannot be retracted; downloaded data cannot be remotely erased. Do not grant a former member access merely because outstanding obligations remain; that settlement policy is still open.

Receipt downloads are mediated by the service rather than reusable object links. Validate bytes/type/size and isolate untrusted uploads; no private filesystem path or object key becomes public permission. Full-product identity uses email/password and actual email recovery. Salted adaptive password hashing, expiring single-use recovery tokens, generic recovery responses, session protection, TLS, secret management, and authorization/rate-limit checks require implementation/security verification. The local demo does not prove production identity or recovery.

| Concern | Swing MVP | Full product direction |
|---|---|---|
| Delivery | Java 21 Swing, one Maven project, per-OS packaging as needed | Primary web plus actual Android and iOS apps; one versioned Java REST backend |
| Storage | Local SQLite, local protected receipt files | PostgreSQL canonical tables/postings, private image objects |
| Delivery worker | Local durable outbox to in-app inbox | Durable outbox worker to in-app and product-selected channels |
| Consistency | Short serialized local write transactions | Shared access/exclusive removal locks, expense locks, sorted pair guards, atomic financial commit |
| Scale | Single-workstation demonstration | Stateless API scaling after measurement; indexed SQL reports; pair contention monitored |
| Acceptance | Local behavior/durability only | Equivalent outcomes on all required platforms and agreed 95%/3-second timing evidence |

Maintain one-way package dependencies, schema migrations, repository/transaction contracts, and PlantUML consistency. Back up and restore canonical records, postings, receipts, and delivery state; no domain-event replay machinery is introduced. Index group/date/pair queries and measure before adding caches or analytics stores. Return payment results from committed primary state; delivery alone may be eventual. Hosting, availability, recovery targets, client toolchains, and supported versions remain unresolved.

## 6. Traceability and planned verification

All FR-001–FR-012, NFR-001–NFR-006, AC-001–AC-010, and the five confirmed clarifications are retained verbatim in the existing specification/requirements inventory. Safeguards are implementation refinements; they do not create customer requirements.

| Requirement / acceptance | Design responsibility | Planned evidence (not executed) |
|---|---|---|
| FR-001–FR-002; AC-001; NFR-002 | Account/current actor, credentials, email recovery, personal authorization | Register/sign-in/recover and account isolation; genuine email recovery for full product |
| FR-003; AC-002; NFR-002 | GroupService, roles, current membership, access guard | Creator/admin-only add/remove; removal/read/write/retry races; history retained |
| FR-004–FR-006; AC-003; NFR-001, NFR-005 | Expense, explicit payer, AllocationPolicy, ReceiptStore | Payer differs from submitter, positive/precision checks, >=100 valid allocation examples reconcile, invalid total/negative share denied, receipt privacy |
| FR-007; AC-004; NFR-002, NFR-006 | Outbox and unique authorized inbox | Commit/provider failure/restart, deduplication, removed-member suppression |
| FR-008; AC-005; NFR-006 | Policy/decisions, locked finalization, posting uniqueness | Single/multiple levels, default self-approval denial, partial/rejected no effect, competing final decisions, identical retry |
| FR-009–FR-010; AC-006; NFR-001, NFR-005–NFR-006 | Ledger, PaymentService, pair guards, receipts | Full/partial payments, overpayment, simultaneous settlement, approval/payment race, lost response, hash conflict, transaction rollback |
| FR-011; AC-007; NFR-002, NFR-005 | ReportService with mandatory access filters | Source-record reconciliation, date/parameter filters, invalid range and unauthorized report denial |
| FR-012; AC-008; NFR-004 | REST adapter plus web/Android/iOS experiences | Equivalent account/approval/balance/payment journeys on all three; Swing cannot supply this evidence |
| NFR-003; AC-009 | Full-product deployment/load measurement | At least 95 of 100 ordinary actions <=3 seconds per platform under agreed conditions |
| NFR-005–NFR-006; AC-010 | Atomic commits, actionable errors, durable audit | Invalid input leaves no partial writes; accepted history survives restart/later sign-in; backup/restore reconciliation |

JUnit 5 covers domain and adapter behavior without constructing Swing. SQLite and PostgreSQL integration tests must exercise their actual locking semantics; mocks cannot prove concurrency safety. Contract/platform checks complement those tests. See [quickstart.md](./quickstart.md) for future execution scenarios and diagram commands.

Report-date semantics need definition: spending normally filters expenseDate, payment history paidAt, and balances must be labeled current unless an approved historical cutoff is implemented. Date ranges and filters cannot broaden authorization. Outstanding debts after removal, reversal/correction workflow, and retention are unresolved rather than silently implemented.

## 7. Diagram inventory and consolidation validation

- [domain-class-diagram.puml](./diagrams/domain-class-diagram.puml): domain relationships plus services/ports and retry/outbox/locking safeguards.
- [expense-approval.puml](./diagrams/expense-approval.puml): atomic submission/outbox, short decision transactions, current authorization, retry handling, and once-only finalization.
- [balance-settlement.puml](./diagrams/balance-settlement.puml): transaction before latest-debt validation, retry before debt revalidation, pair guard, and atomic settlement.
- [component-architecture.puml](./diagrams/component-architecture.puml): selected SQLite local MVP and separate REST/PostgreSQL full product.

Validation for this documentation task: source/flow review, diagram delimiter/branch structure checks, local link checks, requirement-ID preservation, and file-hash verification of protected artifacts. These checks do not constitute PlantUML syntax compilation or rendered-image inspection. Java and PlantUML are unavailable on the current PATH, so rendering was not executed. No Java implementation, Maven/JUnit tests, security audit, deployment, performance run, or mobile acceptance was performed.

## 8. Open questions and review boundary

Retain all specification questions: currencies/scales and cross-currency behavior; additional account safeguards; group/policy administration; reviewer eligibility/levels/order/self-review options; notification channels/preferences; report parameters/export/sharing; receipt formats/limits/retention/access; browser/Android/iOS versions and initial parity; exact lab feature subset/release status; operating load/availability/performance conditions; and deterministic tie-break selection.

Additional design decisions requiring confirmation include reciprocal netting presentation, report effective-date semantics, removed-member debt handling, correction/reversal rules, account/audit/outbox/idempotency retention, identity/email provider, hosting/recovery targets, and constitution-compatible client toolchains. The constitution ratification date remains an unchanged TODO. No material conflict with the selected architecture was identified; these open points prevent production acceptance claims, not completion of this documentation consolidation.
