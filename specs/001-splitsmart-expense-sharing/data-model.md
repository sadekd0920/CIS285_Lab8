# SplitSmart Data Model

This model consolidates selected Architecture 1: local SQLite for the Swing MVP and proposed PostgreSQL for the full REST product. Canonical relational records and immutable ledger postings remain authoritative; there is no event store or CQRS projection model. Schema and platform details are design choices, not customer requirements. See [design.md](./design.md).

## Entities

| Entity | Core fields | Relationships and purpose |
|---|---|---|
| **Account** | `accountId`, `email`, `credentialHash`, `displayName`, `createdAt`, `status` | Owns group memberships and initiates actions. Email is unique and normalized. Credential material is never stored as plaintext. |
| **Group** | `groupId`, `name`, `createdByAccountId`, `createdAt`, `status`, `currencyCode` | Contains memberships, expenses, approvals, and payments. Baseline assumes one currency per expense/group; currency conversion is not included. |
| **GroupMembership** | `groupId`, `accountId`, `role`, `joinedAt`, `removedAt`, `status` | Join entity between Group and Account; role distinguishes member/admin/creator. Removal revokes access but preserves history for remaining authorized users. |
| **ApprovalPolicy** | `policyId`, `groupId`, `version`, `levels`, `selfApprovalAllowed` | Group policy defines reviewer eligibility and required approval levels. Policy version used at submission is referenced by the expense so later edits do not silently rewrite an in-flight workflow. |
| **Expense** | `expenseId`, `groupId`, `submitterId`, `payerId`, `amount`, `currencyCode`, `expenseDate`, `description`, `splitMethod`, `status`, `policyVersion`, `createdAt`, `approvedAt` | Belongs to one group; payer and submitter are accounts; payer must be a selected participant. Status gates ledger effects. |
| **ExpenseAllocation** | `expenseId`, `participantId`, `inputPercentage` (nullable), `shareAmount`, `remainderRank` (optional) | Join entity between Expense and participant Account. Final share amounts are persisted and sum exactly to the expense total. |
| **Receipt** | `receiptId`, `expenseId`, `objectKey` or local path, `mediaType`, `byteLength`, `createdAt` | Optional attachment. Access follows expense authorization; image bytes are stored outside relational rows in the full product. |
| **ApprovalDecision** | `decisionId`, `expenseId`, `level`, `reviewerId`, `decision`, `comment`, `decidedAt` | Immutable decision history for a required level. Prevents unauthorized, duplicate, or out-of-order decisions according to policy. |
| **LedgerPosting** | `postingId`, `groupId`, `expenseId` or `paymentId`, `debtorId`, `creditorId`, `currencyCode`, `amount`, `effectiveAt`, `postingType` | Append-only source for debt calculations. Expense approval creates debt postings; payment creates offsetting postings. Each posting is positive and has different debtor/creditor accounts. |
| **Payment** | `paymentId`, `groupId`, `payerId` (debtor), `payeeId` (creditor), `amount`, `currencyCode`, `paidAt`, `recordedBy`, `idempotencyKey` | Records a full or partial settlement. It does not represent a payment processor transfer; it records that users report a payment made. |
| **Notification** | `notificationId`, `recipientId`, `groupId`, `expenseId`, `type`, `createdAt`, `deliveryState`, `readAt` | In-app notice for eligible group members. Full product may pair this with an outbox delivery record. |
| **ReportCriteria** | `startDate`, `endDate`, `groupId` and selected filters | Request value, not necessarily a persisted entity. All queries must apply authorization before returning records. |
| **OperationReceipt** | `actorId`, `operation`, `targetId`, `idempotencyKey`, `requestHash`, `resultId`, `createdAt` | Durable accepted-operation retry record. Unique scope is actor/operation/target/key; hash covers normalized semantic input. No credentials or sensitive response copies. |
| **DebtPairGuard** | `groupId`, `currencyCode`, `debtorId`, `creditorId` | Unique stable PostgreSQL locking row, including a zero-debt pair. Contains no authoritative balance. SQLite uses its write transaction instead. |
| **OutboxMessage** | `messageId`, `groupId`, `expenseId`, `type`, `createdAt`, `status`, `attempts`, `nextAttemptAt`, `leaseUntil` | Submission delivery work committed with expense; unique expense/type. Not a canonical event history. Per-recipient delivery records track retries/skips. |
| **NotificationDelivery** | `messageId`, `recipientId`, `channel`, `status`, `providerReference` | Unique message/recipient/channel. Recipient candidates are other active members at submission; current eligibility is rechecked at dispatch. |

## Relationships

- Account many-to-many Group through GroupMembership.
- Group has many Expenses, one or more ApprovalPolicy versions, and many Payments.
- Expense has one submitter, one payer, one or more participant allocations, zero or one Receipt, and zero or more ApprovalDecisions.
- Expense payer must appear among allocations. The sum of `shareAmount` values equals `Expense.amount`.
- LedgerPosting references the originating approved Expense or Payment. It is not created for a pending or rejected expense.
- Notifications target only current authorized recipients; a removed member cannot retrieve old group records through a notification or direct identifier.

## Validation and integrity rules

1. Account email is unique after normalization; password recovery uses a separate verified-email process in the full product.
2. An expense amount is positive and conforms to the currency's supported scale. Use `BigDecimal`; do not use binary floating point.
3. Participants are active group members at submission time and include the explicit payer.
4. Equal or percentage allocations are calculated at the currency scale and sum exactly to the amount. Percentage values are nonnegative and total exactly 100%. Allocate rounding units by largest fractional remainder using a consistent tie-break. Participant ID ascending is a proposed, unconfirmed default; persist final shares so later policy changes do not change historical allocations.
5. An expense begins in `PENDING_REVIEW`; only the last required approval transitions it to `APPROVED`. A rejection transitions it to `REJECTED`. Pending and rejected expenses have no ledger postings.
6. Approval/ledger creation is atomic and idempotent by expense. A retry must not duplicate postings.
7. A payment amount is positive, uses the debt currency, and is at most the current outstanding amount for the specified debtor/creditor pair. Applying it, recording it, and creating its ledger posting are atomic. An idempotency key or equivalent uniqueness guard prevents duplicate retries.
8. A user may access an account, group, expense, receipt, balance, or payment only when the relevant authorization policy grants access. Removal of group membership revokes access while preserving historical rows.
9. Report date range must be valid (`startDate <= endDate`). Every report filter is combined with authorization constraints.
10. Persistence of a successfully approved expense or recorded payment is durable so it remains available in later sessions (NFR-006).

## State transitions

### Expense

```text
DRAFT -> PENDING_REVIEW
PENDING_REVIEW -> APPROVED   (all configured approvals complete; ledger postings created once)
PENDING_REVIEW -> REJECTED   (any authorized rejection; no ledger effect)
```

Editing/reopening, cancellation, and post-approval reversal are not specified; the MVP should avoid silently supporting those behaviors. If needed, they require explicit product rules and an auditable correction workflow.

### Payment

```text
PROPOSED -> RECORDED         (validation and atomic ledger offset succeed)
PROPOSED -> REJECTED         (invalid amount, debt pair, currency, or authorization)
```

A recorded payment is immutable in the initial model. Corrections should be explicit reversal records rather than silent mutation.

## Consolidated constraints and transaction boundaries

- Preserve decimal exactness in adapters: SQLite stores money and percentage values as canonical decimal TEXT, parses them into BigDecimal, and aggregates financial totals in Java rather than SQLite floating-point SUM/coercion. PostgreSQL uses exact NUMERIC columns at the approved precision/scale and exact SQL aggregation. Both enforce supported scale in domain validation; neither converts money through double/float. Overall supported magnitude/precision remains a product decision.
- ApprovalDecision has a stable ID and a unique `(expenseId, level, reviewerId)` decision slot under the captured policy. Do not overwrite a prior decision. Eligibility and level completion follow configured policy; no new quorum policy is invented here. Expense row locking serializes all decisions and the final status change.
- LedgerPosting must reference exactly one approved Expense or recorded Payment. Unique expense/participant posting identity prevents duplicate expense contributions; unique payment offset identity permits exactly one settlement offset. Zero-share allocations produce no debt posting. Pending/rejected expenses have none.
- OperationReceipt is saved in the same transaction as its accepted decision/payment and postings. Look it up after current authorization and before mutable-state validation. Same key/hash returns the prior result; different hash conflicts. A competing key cannot duplicate a final approval because expense status and posting uniqueness still apply. A new payment key represents a new payment request; callers must reuse the original key on retries.
- Payment is positive, at currency precision, between distinct accounts, and no greater than latest directional debt. Lock the pair before calculating debt from committed postings. Final approvals use the same guards for all affected pairs, sorted consistently. The SQLite adapter starts BEGIN IMMEDIATE before these reads; PostgreSQL uses READ COMMITTED after acquiring guards. Do not rely on a UI balance or a stale pre-lock snapshot.
- For PostgreSQL writes, group shared locks precede receipt/expense/pair locks; member/role removal uses an exclusive group lock. Revalidate active membership and permissions inside that boundary. Query authorization joins current membership; never trust historical membership stored on an expense/notification.
- Expense submission, validated allocations/receipt metadata, and OutboxMessage commit atomically. NotificationDelivery rows enumerate eligible other members at submission. Dispatcher lease/retry state is durable. Unique `(messageId, recipientId)` inbox identity prevents duplicate notices; a removed recipient is marked skipped and cannot retrieve prior group notices. A failed external send may be retried with provider deduplication where available; external exactly-once delivery is not guaranteed.
- All financial/history changes roll back together on validation, concurrency, or persistence failure. Retry a complete transaction with the original key after conflicts. Return success only after commit, then read balances from committed primary state. No separate asynchronously updated balance table is required.
- Retain decisions, accepted payments, postings, and required retry uniqueness for the applicable history lifetime. Exact retention/deletion rules remain open; purging retry records must not silently permit duplicate financial effects. Immutable here means no application overwrite, not exemption from an approved retention policy.

## Balance derivation

For each group, currency, debtor, and creditor pair, outstanding debt is the sum of approved-expense postings minus payment offsets. Each non-payer participant owes their allocated share to the named payer; the payer's own allocation produces no self-debt. Preserve directional debts for settlement validation. Any reciprocal netting display/policy requires explicit definition before use; cross-group/cross-currency netting is not assumed. Authorized posting queries with BigDecimal aggregation for SQLite or exact NUMERIC aggregation for PostgreSQL are ordinary balance queries, not CQRS or replay of domain events.
