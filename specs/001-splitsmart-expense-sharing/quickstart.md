# SplitSmart Design Validation Quickstart

This guide defines validation runs for the planned implementation; no Java application is created by this planning task. The commands below become runnable after the Maven project and JUnit 5 tests are implemented.

## Prerequisites

- Java 21 JDK (`java -version` reports 21).
- Maven (`mvn -version`) and network access to resolve dependencies on first build.
- For the desktop demonstration, a supported workstation with a graphical desktop.
- Complete-product contract checks additionally require a deployed API and supported test environments for web, Android, and iOS.

## Build and automated checks

From the future Maven project root:

```powershell
mvn -B clean verify
```

Expected: the Java project compiles for release 21; JUnit 5 domain tests and adapter/integration checks pass; no test requires interactive Swing use to verify domain rules.

Select focused domain tests by their JUnit class/name after those tests exist rather than assuming a multi-module layout. Expected coverage includes positive amount/currency precision, equal and percentage allocations, largest-remainder tie-breaking, pending/approved/rejected transitions, self-approval policy, pairwise debt calculation, partial/full settlement, overpayment rejection, and idempotent retries.

## Desktop MVP scenario

After packaging the planned desktop app:

```powershell
mvn -B package
```

For a self-contained desktop package, build a target-OS-specific installer with Java 21 `jpackage` after configuring the eventual Maven packaging profile. Launch the generated application using the packaging instructions. Verify:

1. Create/sign in to local demo accounts and create a group with members.
2. Submit an expense with payer different from submitter; verify it is pending and balances are unchanged.
3. Approve all configured levels; verify allocation shares sum to the total and balances change once.
4. Reject a second expense; verify it has no ledger effect.
5. Record a partial then a full payment within outstanding debt; verify the balance and history update exactly once.
6. Attempt an overpayment and invalid percentage total; verify actionable errors and no persisted changes.
7. Restart the app and confirm approved expenses and recorded payments remain in history.

This scenario validates local demonstration behavior only. It does not validate multi-device collaboration, remote notifications, email recovery, or FR-012.

## Complete-product acceptance scenarios

Once the API and clients are available, execute equivalent account, expense approval, balance, and payment journeys against the service contract in [contracts/service-contract.md](./contracts/service-contract.md) from:

- A supported web browser.
- A supported Android device/emulator.
- A supported iOS device/simulator.

For each platform, compare persisted allocation and balance outcomes for identical inputs. Verify denied access for a removed member, no access for a nonmember, report filtering by date range, and notification privacy. Measure 100 ordinary actions per platform under the agreed operating environment and confirm at least 95 return visible results within 3 seconds (AC-009). Do not declare this criterion measured until the environment, load, and measurement method are approved.

## Traceability

See [requirements.md](./requirements.md), [data-model.md](./data-model.md), and [architecture.md](./architecture.md). Acceptance criteria are in [spec.md](./spec.md); API expectations are in [contracts/service-contract.md](./contracts/service-contract.md).

## Selected-safeguard validation scenarios

Architecture 1 remains selected. Exercise these future JUnit 5 integration/contract cases against SQLite locally and, separately, PostgreSQL for the REST product. They do not require Event Sourcing/CQRS or a connected desktop MVP.

| Scenario | Expected evidence |
|---|---|
| Final approval repeated under same request ID | Same accepted result; one immutable decision per valid decision slot and one set of debt postings |
| Competing approvals/rejection with different IDs | Expense serialization permits one valid final outcome; no approval postings for rejected expenses and no duplicate posting set |
| Full settlement commits but response is lost | Retry same key returns original payment even though outstanding is now zero; one payment/offset/receipt |
| Same accepted key reused with different content | Actionable conflict and no new history or ledger effects |
| Two payments race against the same last debt | Post-lock validation accepts only amounts still within latest debt; never a negative outstanding amount |
| Final approval races with settlement | Both use compatible guard protocol; result matches a valid serial ordering and persisted postings reconcile |
| Failure between financial writes and commit | Decision/payment/status/postings/receipt roll back together; retry cannot expose partial success |
| Member removal races with a write | Access locking establishes ordering; writes after removal are denied, including cached retry results |
| Removed member uses old receipt/report/notice IDs | Current authorization denies retrieval while current members retain history |
| Submission commits, worker crashes or provider fails | Expense remains pending/durable; outbox resumes after restart; one in-app item per eligible recipient, removed recipients skipped |
| Audit review and report reconciliation | Accepted decisions/payments/postings cannot be silently edited; authorized filtered reports match canonical records |

## Diagram and documentation validation

The selected sources are `diagrams/domain-class-diagram.puml`, `expense-approval.puml`, `balance-settlement.puml`, and `component-architecture.puml`. With a locally installed Java/PlantUML toolchain, run from this feature directory:

```powershell
plantuml -checkonly diagrams/*.puml
plantuml -tsvg diagrams/*.puml
```

Inspect rendered relationships, transaction/retry ordering, and the SQLite/REST platform boundary. These are future commands, not evidence that rendering occurred. Current consolidation evidence and tool limitations are recorded in [design.md](./design.md). No Maven build, JUnit run, deployment, or platform acceptance is claimed by this documentation task.
