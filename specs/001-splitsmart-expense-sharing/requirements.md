# SplitSmart Requirements

**Source of customer requirements**: [`UserRequirement.md`](../../UserRequirement.md)  
**Specification**: [`spec.md`](./spec.md)

This document consolidates the original customer feature inventory, the functional and
non-functional requirement IDs currently in the feature specification, and their
traceability. Customer needs are kept distinct from implementation decisions and from
clarifications that add testable detail.

## Original Customer Requirements

The following inventory preserves the complete set of features stated in
[`UserRequirement.md`](../../UserRequirement.md):

1. **User accounts**: Each person has an individual account to log in and access their own
   expense and balance information.
2. **Group management**: Users can create and manage groups of people, such as roommates,
   trip participants, or party participants, to track shared expenses and balances.
3. **Expense creation**: Users can create expenses containing an amount, date, description,
   sharing method (equal or percentage-based), and optional receipt image. Users can identify
   which other users were involved so appropriate balance adjustments can be made.
4. **Notifications**: Users can be notified when other group members create an expense.
5. **Approval workflow**: Expenses can be reviewed and approved before being added to the
   system and used to adjust balances. Approval may use a single review or multiple levels,
   depending on group needs.
6. **Balance tracking**: The system tracks the balance owed by each user to every other
   user, taking entered expenses into account.
7. **Payment tracking**: Users can record payments made to settle amounts owed, and
   balances reflect those payments.
8. **Reporting**: Users can see reports and summaries of expenses, balances, and payments,
   with possible custom reports using date ranges and other parameters.
9. **Supported platforms**: The product is primarily web-based and also has an app that can
   run on Android and an app that can run on iOS.

## Functional Requirements

These are the current specification requirements. Requirements FR-001 through FR-012
express the customer features and associated behavior; any details beyond the original
request are specification-level decisions or clarifications, not additional original
customer requests.

- **FR-001**: The system MUST allow each person to create an individual account and sign in using an email address and password. The system MUST provide an account password recovery flow using email.
- **FR-002**: The system MUST allow a signed-in account holder to access their own expense and balance information and MUST restrict other users' personal information to authorized access.
- **FR-003**: The system MUST allow users to create groups. Only the group creator or a designated administrator MAY add or remove members. Removing a member MUST end that person's access to group records without deleting the group's history for current authorized members.
- **FR-004**: The system MUST allow a group member to submit an expense with an amount, date, description, selected participating users, and one explicit payer selected from those participants. The payer MAY differ from the submitter.
- **FR-005**: The system MUST support equally divided and percentage-based expense allocations. Expense amounts MUST be positive and conform to the supported currency precision. Percentage shares MUST be nonnegative and total exactly 100%; invalid amounts or splits MUST be rejected. When share calculations create leftover smallest currency units, the system MUST allocate them to participants with the largest fractional remainders using a consistent tie-break rule.
- **FR-006**: The system MUST allow an expense submitter to attach an optional receipt image and associate it with the expense.
- **FR-007**: When a group member creates an expense, the system MUST notify other members of that group of the new expense, subject to their authorization to view that group.
- **FR-008**: The system MUST provide an expense review and approval process. Expenses MUST NOT enter the approved ledger or change balances until all required approvals are complete. The process MUST support a single review or multiple approval levels according to the group's configured needs, and MUST support rejection without balance impact.
- **FR-009**: The system MUST calculate and show amounts owed by each user to every other user based on approved expenses and recorded payments.
- **FR-010**: The system MUST allow a user to record a full or partial payment up to the outstanding amount between a specified payer and payee. Each accepted payment MUST immediately reduce the corresponding outstanding balance and appear exactly once in payment history. The system MUST reject a payment greater than the outstanding amount.
- **FR-011**: The system MUST provide authorized users with expense, balance, and payment summaries and reports. Users MUST be able to specify a date range and other report parameters; the resulting report MUST include only matching authorized records.
- **FR-012**: The product MUST be primarily web-based and MUST also be available as an app runnable on Android and an app runnable on iOS.

## Non-Functional Requirements

- **NFR-001**: For every approved expense, participant shares MUST sum exactly to the expense total at the supported currency precision. Rounding remainder units MUST be allocated by largest fractional remainder and the resulting shares MUST be visible in the expense detail.
- **NFR-002**: A user MUST NOT be able to view or change account, group, expense, receipt, balance, or payment information unless authorized for that information.
- **NFR-003**: On each supported platform, 95% of ordinary account, group, expense, balance, and report-viewing actions MUST provide a visible result within 3 seconds under the agreed normal operating conditions. The measurement environment and normal load are to be established during planning.
- **NFR-004**: The web, Android, and iOS experiences MUST preserve the same underlying expense approval, allocation, balance, and payment outcomes for equivalent user actions.
- **NFR-005**: User-facing validation failures MUST identify the field or action that needs correction and MUST NOT silently record an invalid expense, approval, payment, or report request.
- **NFR-006**: A successfully approved expense or recorded payment MUST remain available in the applicable history and reports after the user leaves and returns to the product.

## Traceability

| Original customer feature | Source need | Functional requirement IDs | Related non-functional requirements | Acceptance criteria in `spec.md` |
|---|---|---|---|---|
| User accounts | Individual accounts, login, and access to own expense and balance information | FR-001–FR-002 | NFR-002, NFR-005–NFR-006 | AC-001 |
| Group management | Create and manage groups to track shared expenses and balances | FR-003 | NFR-002, NFR-005–NFR-006 | AC-002 |
| Expense creation | Amount, date, description, equal or percentage split, optional receipt, and involved users | FR-004–FR-006 | NFR-001–NFR-002, NFR-005–NFR-006 | AC-003 |
| Notification system | Notify users when other group members create expenses | FR-007 | NFR-002, NFR-005 | AC-004 |
| Approval workflow | Review and approve expenses before ledger inclusion and balance adjustment; single or multiple review levels | FR-008 | NFR-002, NFR-005–NFR-006 | AC-005 |
| Balance tracking | Track amounts owed by each user to every other user based on expenses | FR-009 | NFR-001–NFR-002, NFR-004, NFR-006 | AC-006 |
| Payment tracking | Record settlement payments and reflect them in balances | FR-010 | NFR-002, NFR-005–NFR-006 | AC-006 |
| Reporting | Expense, balance, and payment reports/summaries, with date ranges and other parameters | FR-011 | NFR-002–NFR-003, NFR-005–NFR-006 | AC-007 |
| Web, Android, and iOS platforms | Primary web-based product plus apps runnable on Android and iOS | FR-012 | NFR-003–NFR-004 | AC-008–AC-009 |

## Confirmed Clarifications

The following decisions were confirmed during clarification and are preserved from
[`spec.md`](./spec.md). They resolve behavior left open by the original request; they do not
add customer feature categories.

- **Payer and participants**: Each expense has one explicit payer selected from its
  participants. The payer may differ from the person submitting the expense.
- **Group membership**: Only the group creator or a designated administrator may add or
  remove members. A removed member loses access to group records; current authorized
  members retain the group's historical records.
- **Amounts, percentages, and rounding**: Expense amounts must be positive and within
  supported currency precision. Percentage shares must be nonnegative and total exactly
  100%. Rounding remainder units are allocated to participants with the largest fractional
  remainders, using a consistent tie-break rule.
- **Authentication**: Account sign-in uses an email address and password, with password
  recovery by email.
- **Balance settlement**: Full or partial payments up to the outstanding amount are
  applied immediately to the specified payer-payee balance and recorded exactly once.
  Payments greater than the outstanding amount are rejected.

## Implementation Scope Note

Architecture 1 is selected per [architecture-review.md](./architecture-review.md). The following safeguards refine the implementation of existing requirements; they are not new customer feature categories or changes to FR/NFR/AC wording. [design.md](./design.md) maps them to verification scenarios.

| Safeguard | Existing requirement / acceptance trace | Selected design |
|---|---|---|
| Retry-safe payment and final approval | FR-008–FR-010, NFR-006; AC-005–AC-006, AC-010 | Durable operation receipts, request hashes, natural uniqueness, atomic decisions/payments/postings; identical authorized retries return prior results before debt/state validation |
| Concurrent debt validation | FR-009–FR-010, NFR-001, NFR-005; AC-006 | Serialize affected debt pairs, calculate latest debt inside the transaction, reject overpayment without partial writes |
| Current authorization | FR-002–FR-003, FR-006–FR-007, FR-011, NFR-002; AC-001–AC-004, AC-007 | Recheck current membership for reads, writes, receipts, reports, notices, and retry responses; preserve historical records after removal |
| Immutable financial and approval audit | FR-008–FR-011, NFR-006; AC-005–AC-007, AC-010 | Persist approval decisions, payments, and postings together where applicable; no silent mutation of accepted history |
| Transactional submission notifications | FR-007, NFR-002, NFR-006; AC-004, AC-010 | Submission/outbox commit together; retry delivery, deduplicate inbox items, suppress removed recipients; delivery failure never undoes expense creation |

All five confirmed clarifications above remain unchanged. Participant-ID rounding tie-break, detailed approval eligibility, and other unresolved policy choices remain proposals, not confirmed decisions. Architecture 2's Event Sourcing/CQRS and connected PostgreSQL MVP are unselected.

The proposed Java Swing MVP is an implementation decision documented separately in
[`spec.md`](./spec.md). It is not an original customer requirement and does not replace
FR-012: a Swing desktop client alone does not satisfy the requested web, Android, and iOS
platforms. The specification remains authoritative for unresolved product questions.
