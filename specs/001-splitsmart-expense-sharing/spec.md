# Feature Specification: SplitSmart Expense Sharing

**Feature Branch**: `001-splitsmart-expense-sharing`

**Created**: 2026-10-29

**Status**: Draft

**Input**: User description: "Read UserRequirement.md in this workspace as the authoritative customer requirements. Write a testable SplitSmart feature specification: actors, user scenarios, functional and non-functional requirements, acceptance criteria, requirement IDs (FR-001, etc.), and open questions. Preserve all original customer features in the requirements inventory; label the Java Swing MVP subset separately. Do not write Java code. Do not edit UserRequirement.md."

## Clarifications

### Session 2026-10-29

- Q: How should SplitSmart identify who paid for an expense, separately from the people who share its cost? → A: Record one explicit payer, who must be one of the selected participants; the payer may differ from the submitter.
- Q: Who should control adding or removing people from a group, and what should happen to a removed member's access to earlier group records? → A: Only the group creator or designated administrators may add or remove members; removal ends that person's access to group records while the group's historical records remain available to current authorized members.
- Q: How should SplitSmart validate expense amounts and percentage splits, and allocate any leftover smallest currency units when shares are rounded? → A: Require a positive amount within the supported currency precision and nonnegative percentages totaling exactly 100%; allocate leftover smallest currency units to participants with the largest fractional remainders, using a consistent tie-break rule.
- Q: How should a person prove their identity when signing in to a SplitSmart account? → A: Use an email address and password, with account recovery by email.
- Q: How should a recorded payment change the balance between two group members? → A: Allow full or partial payments up to the outstanding amount; apply each immediately to the specified payer-payee balance and record it once.

## Actors

- **Account holder**: A person who signs in to SplitSmart and manages their own expenses, groups, balances, payments, and reports.
- **Group member**: An account holder who belongs to a group and may participate in its expenses.
- **Expense submitter**: A group member who enters an expense and identifies its participants and split.
- **Expense reviewer**: A group member authorized by the group to review, approve, or reject a submitted expense.
- **Report viewer**: An account holder viewing their own or an authorized group's expense, balance, and payment information.

## Customer Requirements Inventory

The following inventory preserves all product features in [UserRequirement.md](../../UserRequirement.md). FR identifiers in this specification trace each feature into testable behavior. The customer asked for a primary web-based product and accompanying Android and iOS apps; the proposed Swing MVP below does not replace that scope.

| Customer feature | Requirement IDs |
|---|---|
| Individual user accounts, login, and access to own expense and balance information | FR-001–FR-002 |
| Create and manage groups of people | FR-003 |
| Create expenses with amount, date, description, equal or percentage split, optional receipt image, and selected participants | FR-004–FR-006 |
| Notify users when group members create expenses | FR-007 |
| Review and approve expenses before they enter the ledger and affect balances; support group needs for one or multiple approval levels | FR-008 |
| Track how much each user owes every other user based on entered expenses | FR-009 |
| Record payments and reflect settlements in balances | FR-010 |
| Provide expense, balance, and payment reports and summaries, including custom date ranges and parameters | FR-011 |
| Primary web-based access and apps runnable on Android and iOS | FR-012 |

## Proposed Java Swing MVP Subset (Implementation Decision)

This is a proposed limited desktop MVP for early demonstration and is not a customer requirement or a claim of full product delivery. It is restricted to a Java Swing desktop client and may demonstrate account sign-in, group creation and membership, expense entry with equal or percentage allocation and optional receipt, a configurable expense review, balances, payment recording, and basic reports. Notifications may be shown within the desktop application. Any MVP omissions must be made explicit in release notes and must not be reported as satisfying omitted customer requirements.

The Swing MVP does **not** satisfy FR-012's required web, Android, and iOS product scope. Those platforms remain in the full requirements inventory and require separately delivered platform experiences before the customer platform requirement can be accepted. Whether the MVP includes every capability in this subset, or a smaller demo-only slice, remains a planning decision.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Record and Review a Shared Expense (Priority: P1)

An account holder creates or joins a group, submits a shared expense with its amount, date, description, split method, and participants, and sees it included in the group's ledger only after the required review.

**Why this priority**: Expense capture and approval are the source of truth for shared balances.

**Independent Test**: Create a group with multiple members, submit an expense, review it, and verify that approval changes the ledger while rejection does not.

**Acceptance Scenarios**:

1. **Given** a signed-in member of a group, **When** they submit an expense with valid details, an explicit payer selected from its participants, and a participant split, **Then** the expense is recorded as awaiting review and has not yet changed any member's balance.
2. **Given** an expense awaiting review, **When** an authorized reviewer approves it and all required approval levels are complete, **Then** it is added to the ledger once and the resulting shares are reflected in balances.
3. **Given** an expense awaiting review, **When** an authorized reviewer rejects it, **Then** it is excluded from the ledger and does not change balances.
4. **Given** an expense whose percentage shares do not total 100%, **When** the submitter tries to save it, **Then** the system rejects the submission and identifies the invalid split.
5. **Given** an expense submitter who is not the selected payer, **When** the expense is approved, **Then** the recorded payer is credited for the other participants' shares and the submitter is not treated as payer solely because they submitted the expense.

### User Story 2 - Review Group Balances and Settle Up (Priority: P1)

A group member checks amounts owed between members and records a payment so that the outstanding balances reflect the settlement.

**Why this priority**: Members need a reliable view of who owes whom and a way to record completed settlements.

**Independent Test**: Approve an expense that creates an amount owed, record a payment against that debt, and verify the updated balances and payment history.

**Acceptance Scenarios**:

1. **Given** approved expenses in a group, **When** a member views balances, **Then** the system shows the outstanding amount owed by each user to each other user.
2. **Given** an outstanding amount owed, **When** the payer records a full or partial payment up to that amount to the payee, **Then** the payment is applied immediately, appears once in payment history, and the outstanding amount decreases by the recorded amount.
3. **Given** a proposed payment greater than the amount outstanding, **When** the payer attempts to record it, **Then** the system prevents a negative debt balance and explains the validation error.

### User Story 3 - Manage an Account and Group (Priority: P1)

A person creates or accesses an individual account and manages groups with other people so their shared expenses remain associated with the right members.

**Why this priority**: Accounts and groups establish identity, access, and the context for shared activity.

**Independent Test**: Sign in as one account holder, create a group, add another account holder, and verify each can access the group's shared information while personal information remains account-specific.

**Acceptance Scenarios**:

1. **Given** a person without an account, **When** they register with an email address and password, **Then** an individual account is created and can be used to sign in; the person can request password recovery by email.
2. **Given** a signed-in account holder, **When** they create a group and add members, **Then** the group and its membership are visible to authorized members.
3. **Given** two separate account holders, **When** either views personal expenses and balances, **Then** the system displays only information they are authorized to access.
4. **Given** a group creator or designated administrator, **When** they remove a member, **Then** that member loses access to the group's records while the records remain available to current authorized members.

### User Story 4 - Receive Expense Notifications (Priority: P2)

A group member is notified when another member creates an expense in a group they share.

**Why this priority**: Timely notice lets members review activity and respond to expenses requiring approval.

**Independent Test**: Create an expense as one group member and verify that other eligible group members receive a notification identifying the new activity.

**Acceptance Scenarios**:

1. **Given** a group with multiple members, **When** one member creates an expense, **Then** the other members receive a notification that a new expense was submitted.
2. **Given** a person who is not a member of the group, **When** a group expense is created, **Then** that person receives no notification and cannot access the expense.

### User Story 5 - Review Expense and Payment Reports (Priority: P2)

A member reviews a summary of expenses, balances, and payments and can narrow a report by date range and other selected parameters.

**Why this priority**: Reports help members understand shared spending and reconcile settlements.

**Independent Test**: Create approved expenses and payments, request an unfiltered summary and a date-filtered report, and compare the results with the underlying records.

**Acceptance Scenarios**:

1. **Given** expenses and payments in the reporting period, **When** an authorized user requests a report, **Then** the report summarizes the records and balances visible to that user.
2. **Given** a valid custom date range and selected report parameters, **When** the user requests a report, **Then** only matching records are included.
3. **Given** an invalid date range where the start date is after the end date, **When** the user requests a report, **Then** the system rejects the criteria and explains the error.

### User Story 6 - Use SplitSmart on Required Platforms (Priority: P1)

An account holder accesses SplitSmart through the primary web-based product or the Android or iOS app and can use the product capabilities assigned to their account.

**Why this priority**: The customer's platform requirement includes web and both mobile operating systems.

**Independent Test**: Using supported web, Android, and iOS environments, sign in and complete a representative authorized expense-and-balance journey on each.

**Acceptance Scenarios**:

1. **Given** a supported web browser, **When** an account holder signs in and opens their group balances, **Then** they can view the balances for which they are authorized.
2. **Given** a supported Android device, **When** an account holder signs in and opens their group balances, **Then** they can view the balances for which they are authorized.
3. **Given** a supported iOS device, **When** an account holder signs in and opens their group balances, **Then** they can view the balances for which they are authorized.

### Edge Cases

- A split includes a percentage total other than exactly 100%, a negative share, or an invalid participant.
- An expense amount is zero, negative, or exceeds the supported currency precision.
- A currency amount cannot be divided evenly to the smallest supported currency unit, or multiple shares have equal fractional remainders when allocating rounding units.
- An expense submitter is also a required reviewer.
- An expense is rejected after one or more approval levels have approved it.
- A member is removed from a group with outstanding expenses or payments.
- A payment is duplicated, exceeds outstanding debt, or is entered for the wrong payer or payee.
- An optional receipt image is missing, unreadable, or exceeds the supported size or format.
- A notification cannot be delivered while the expense submission succeeds.
- A report has no matching records or uses a date range with no end date.
- The same user accesses the system from more than one supported platform.

## Requirements *(mandatory)*

### Functional Requirements

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

### Non-Functional Requirements

- **NFR-001**: For every approved expense, participant shares MUST sum exactly to the expense total at the supported currency precision. Rounding remainder units MUST be allocated by largest fractional remainder and the resulting shares MUST be visible in the expense detail.
- **NFR-002**: A user MUST NOT be able to view or change account, group, expense, receipt, balance, or payment information unless authorized for that information.
- **NFR-003**: On each supported platform, 95% of ordinary account, group, expense, balance, and report-viewing actions MUST provide a visible result within 3 seconds under the agreed normal operating conditions. The measurement environment and normal load are to be established during planning.
- **NFR-004**: The web, Android, and iOS experiences MUST preserve the same underlying expense approval, allocation, balance, and payment outcomes for equivalent user actions.
- **NFR-005**: User-facing validation failures MUST identify the field or action that needs correction and MUST NOT silently record an invalid expense, approval, payment, or report request.
- **NFR-006**: A successfully approved expense or recorded payment MUST remain available in the applicable history and reports after the user leaves and returns to the product.

### Acceptance Criteria

- **AC-001 (FR-001–FR-002)**: A new person can register with an email address and password, sign in, and request password recovery by email. They can access their own expense and balance information; a different account cannot view that personal information without authorization.
- **AC-002 (FR-003)**: A user can create a group; only its creator or designated administrators can add or remove members. After removal, the former member cannot access group records, while current authorized members retain the group history.
- **AC-003 (FR-004–FR-006, NFR-001)**: A valid positive expense amount within supported currency precision, with each supported split type, an explicit payer among its participants, and an optional receipt, can be submitted. The payer may differ from the submitter. Percentage shares must be nonnegative and total exactly 100%. Equal and percentage shares reconcile to the expense total, with remainder units allocated by largest fractional remainder; invalid amounts or splits are rejected with no ledger change.
- **AC-004 (FR-007)**: Creating an expense causes eligible other group members to receive notice; nonmembers receive neither notice nor access.
- **AC-005 (FR-008)**: A pending expense has no balance effect. Only completion of the configured approval process adds it to the ledger; rejection leaves balances unchanged.
- **AC-006 (FR-009–FR-010)**: Approved expenses change pairwise amounts owed. A full or partial payment up to the outstanding debt is immediately applied to the specified payer-payee balance and recorded exactly once; a payment greater than the outstanding debt is rejected.
- **AC-007 (FR-011)**: Expense, balance, and payment reports match authorized records for the selected date range and parameters; invalid date ranges are rejected.
- **AC-008 (FR-012, NFR-004)**: Representative account and balance journeys work on the web, Android, and iOS product experiences, with equivalent business outcomes.
- **AC-009 (NFR-003)**: In an agreed normal operating environment, at least 95 of 100 measured ordinary actions return visible results in 3 seconds or less on each supported platform.
- **AC-010 (NFR-005–NFR-006)**: Invalid input is reported without being recorded, and successfully approved expenses and payments remain accessible in their histories after a later sign-in.

### Key Entities

- **Account**: A person's individual identity, email sign-in address, password credential, and authorized access to personal information.
- **Group**: A collection of accounts whose members track shared expenses, approvals, balances, and payments.
- **Group Membership**: The association between an account and a group, including membership status and whether the account is the creator or a designated administrator.
- **Expense**: A submitted shared cost including amount, date, description, submitter, one payer selected from its participants, participants, allocation method, optional receipt, and review state.
- **Allocation**: A participant's share of an expense, defined equally or by percentage and reconciled to the expense total.
- **Approval**: A review decision and its reviewer, status, and position in the group's approval process.
- **Notification**: Notice to an eligible group member that another member has created an expense.
- **Balance**: The outstanding amount owed by one account to another, derived from approved expenses (crediting the recorded payer for other participants' shares) and recorded payments.
- **Payment**: A recorded settlement amount between a payer and payee, including its date and associated group or debt context.
- **Report**: A summary or filtered view of authorized expense, balance, and payment records for selected parameters.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: In a representative acceptance test, 100% of accounts can access their own expense and balance information while unauthorized accounts are denied access.
- **SC-002**: In a test set of at least 100 valid equal and percentage-based expenses, every approved expense's participant shares reconcile to its total at supported currency precision.
- **SC-003**: In approval tests covering single-level and multiple-level group policies, 100% of expenses affect balances only after all required approvals; rejected expenses never affect balances.
- **SC-004**: For every payment in a representative settlement test, the post-payment debt equals the pre-payment debt less the accepted payment, and the history includes the payment exactly once.
- **SC-005**: On web, Android, and iOS, at least 95% of ordinary actions in the agreed normal operating environment return visible results within 3 seconds.
- **SC-006**: A representative user can complete account access, group expense submission, review, balance inspection, and payment recording on every required platform without a platform preventing completion.
- **SC-007**: 100% of report results in acceptance tests match the authorized source records selected by the report's date range and parameters.

## Assumptions

- Each expense is recorded in one currency. Cross-currency expenses and currency conversion are not assumed; currency support must be settled as an open question.
- A percentage split must total exactly 100%; the system rejects other totals rather than normalizing them silently.
- A group selects whether expense review uses a single reviewer or multiple approval levels. The authorized group policy determines eligible reviewers and required levels.
- A submitter does not approve their own expense unless the group policy explicitly permits it; the default is that self-approval is not permitted.
- The initial supported receipt image formats, maximum size, and notification delivery channels are established during planning. The product must at minimum notify eligible members through a channel available in each delivered experience.
- Non-functional timing targets are proposed acceptance targets. Normal operating conditions and test environment must be agreed before performance verification.
- The Java Swing MVP is a limited implementation experiment only. It is not a substitute for the primary web product or Android and iOS apps, and its exact included subset is a planning decision.
- Account recovery, account deletion, data-retention periods, and legal or regulatory requirements were not specified in the customer request and remain product decisions.

## Open Questions

1. Which currencies and currency precision must be supported, and must users be able to share or settle expenses in different currencies?
2. What additional account verification or security requirements apply beyond email-and-password sign-in and email-based password recovery?
3. Who may create groups and administer group approval policies?
4. Which approval policy options are needed (including reviewer eligibility, number/order of approval levels, and whether submitters can review)?
5. Which notification channels are required, and should users be able to configure notification preferences?
6. Which report parameters, export formats, and report-sharing permissions are required?
7. What receipt image formats, size limits, retention rules, and access controls are required?
8. What versions of web browsers, Android, and iOS must be supported, and are equivalent features required across all three from the initial release?
9. What is the intended scope and release status of the proposed Java Swing MVP, given the authoritative customer requirement for web, Android, and iOS?
10. What normal load, availability, and response-time conditions should be used to verify the proposed non-functional targets?
11. What deterministic tie-break rule should apply when participants have equal fractional remainders during currency-unit allocation?
