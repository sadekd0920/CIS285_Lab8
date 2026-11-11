# SplitSmart Service Contract (Proposed)

This contract describes the future shared service boundary for web, Android, and iOS. It is not implemented by the Java Swing MVP. Use HTTPS and a versioned JSON REST API; exact authentication provider, token format, host, and transport-level limits remain deployment decisions.

## Contract-wide rules

- Every request is authenticated except registration, sign-in, and password recovery.
- The server derives the acting account from authenticated identity; clients cannot choose an arbitrary submitter or reviewer.
- Every resource read and mutation is checked against current authorization, including retry responses, reports, notices, and receipt bytes. Group identifiers and historical membership do not grant access. Writes coordinate membership removal with the access-lock boundary in `design.md`.
- Monetary amounts are decimal strings in JSON (for example `"12.34"`) plus an ISO currency code; they are never JSON binary floating-point numbers.
- Dates use ISO 8601 date or timestamp formats. IDs are opaque and stable.
- Mutations return a visible success or a structured error. Clients must not treat a timeout as a guaranteed failure. Payment and approval/rejection requests require a stable `Idempotency-Key`; the desktop uses an equivalent application request ID without calling this API.
- API versioning and compatibility policy must be agreed before public release.

## Operations

| Method and path | Purpose | Authorization and outcome |
|---|---|---|
| `POST /api/v1/accounts` | Register an account | Public; validates normalized unique email and password policy; returns account identity without credential material. |
| `POST /api/v1/sessions` | Sign in | Public; returns the configured authenticated session/token response. |
| `POST /api/v1/password-recovery-requests` | Request recovery email | Public; response must not disclose whether an email is registered. |
| `GET /api/v1/me` | Read current account | Authenticated account only. |
| `POST /api/v1/groups` | Create group and initial policy | Authenticated; creator becomes group administrator. |
| `POST /api/v1/groups/{groupId}/members` | Add a member | Group creator/designated administrator only. |
| `DELETE /api/v1/groups/{groupId}/members/{accountId}` | Remove a member | Group creator/designated administrator only; revokes access and preserves history. |
| `POST /api/v1/groups/{groupId}/expenses` | Submit expense | Active group member; returns `PENDING_REVIEW`, validated split shares, and expense ID; no balance effect. |
| `GET /api/v1/groups/{groupId}/expenses/{expenseId}` | Read expense and receipt metadata | Current authorized group member; receipt bytes require an authorized retrieval operation. |
| `POST /api/v1/groups/{groupId}/expenses/{expenseId}/decisions` | Approve or reject at assigned level | Eligible reviewer under the policy version referenced by the expense; final approval creates ledger effect exactly once; rejection has none. |
| `GET /api/v1/groups/{groupId}/balances` | Read pairwise outstanding balances | Current authorized group member; filters by currency if needed. |
| `POST /api/v1/groups/{groupId}/payments` | Record a reported settlement | Authenticated debtor; identifies debtor/payee pair; amount cannot exceed outstanding debt. Requires idempotency key for safe retries; response includes persisted payment and resulting outstanding amount. |
| `GET /api/v1/groups/{groupId}/reports` | Read expense/payment/balance report | Authorized member; accepts date range and supported filters; returns only authorized matching records. |
| `GET /api/v1/notifications` | List in-app notices | Current account; only notices for currently authorized groups. |

Receipt upload may use a separate authenticated upload flow or short-lived signed upload URL; the exact file protocol and allowed formats/limits remain open.

Receipt retrieval is service-mediated (proposed `GET /api/v1/groups/{groupId}/expenses/{expenseId}/receipt`) with current authorization for each request. Upload permission never grants retrieval permission. Validate staged objects before association and remove orphan uploads; no reusable download URL bypasses membership removal.

## Accepted-operation retry and concurrency contract

This refines Architecture 1; it introduces no event-store API or CQRS endpoint.

1. Scope a durable OperationReceipt by authenticated actor, operation, target (expense for decisions; group for payments), and key. Hash normalized semantic request content, including the decision level/outcome or payment pair/amount/currency/date. Missing required key is a `400`; same scope/key with different input is a `409 IDEMPOTENCY_CONFLICT`.
2. Authenticate and recheck current authorization first. An identical accepted retry returns the existing decision/payment ID and recorded outcome before current expense-state/debt validation. A removed member receives no cached protected response. Original financial result is stable; clients can fetch current balances separately because later operations may have changed them.
3. Reserve the unique operation key inside the business transaction. Concurrent identical requests wait/retry and read the committed result; no pending reservation is committed without the financial action. Rollback leaves no accepted receipt. A failed validation may be retried after correction, but a key for an accepted operation cannot change input.
4. Final approval locks the expense, validates the captured policy/current reviewer, and atomically commits decision, status, postings, and receipt. A second key cannot approve the finalized expense again; return a state conflict without another posting. Natural decision/posting uniqueness is a second guard.
5. Settlement acquires the stable directional pair guard and reads the latest committed debt inside the same transaction. All final approvals affecting that pair use the same guard. Validate positive precise amount <= outstanding, then commit payment, offset posting, and receipt together. Concurrent overpayment fails without partial state. Do not validate against a pre-lock UI balance.
6. A new key for an identical-looking payment is a new user action, not a retry. Clients must retain and reuse the original key after uncertainty. Return accepted results only after commit; queries after success use committed primary records. No asynchronous financial projection is used.
7. Keep accepted-operation deduplication durable for the relevant history lifetime; exact retention is unresolved and must not silently allow duplicates. A crash after commit/before reply is resolved by the original key. Exact response status and header conventions remain API-versioning details, but the identity/outcome guarantees are mandatory design invariants.

## Submission notification boundary

Expense creation commits pending expense, allocations, receipt metadata, and outbox work together. Other eligible group members are captured as recipient candidates at submission; a worker rechecks current membership before dispatch and suppresses removed recipients. Inbox queries also apply current membership. Retried delivery creates at most one in-app notice per submission/recipient. Provider failures are retried and never reverse a committed expense. External email/push delivery can be duplicated or already in flight at removal; minimize protected payloads and require authorized retrieval for details.

## Expense request and response shape (illustrative)

```json
{
  "payerId": "account-opaque-id",
  "amount": "18.25",
  "currency": "USD",
  "expenseDate": "2026-11-04",
  "description": "Shared supplies",
  "split": {
    "method": "PERCENTAGE",
    "participants": [
      { "accountId": "account-a", "percentage": "50" },
      { "accountId": "account-b", "percentage": "50" }
    ]
  },
  "receiptUploadId": null
}
```

The authenticated account is the submitter. Payer must be a selected participant. The response includes final `shareAmount` values, `PENDING_REVIEW`, the required approval levels, and no balance adjustment. Equal allocation uses `"method": "EQUAL"` and omits percentages.

## Error behavior

Use a consistent JSON problem response with a stable machine-readable `code`, human-readable `message`, optional `fieldErrors`, and correlation identifier. Suggested status mapping:

- `400` invalid input, percentage total, amount scale, or date range.
- `401` missing/invalid authentication.
- `403` authenticated but not authorized; avoid exposing protected resource details.
- `404` resource absent or intentionally concealed due to access policy.
- `409` stale approval state, conflicting group policy, or payment exceeding current outstanding debt.
- `413` oversized receipt upload.
- `415` unsupported receipt media type.
- `429` rate limit.
- `5xx` unexpected service failure with a correlation ID and no sensitive internals.

Clients must identify the field or action that needs correction for validation failures (NFR-005). A failed operation must not partially persist state. Expense approval and payment mutations must be transactionally atomic and retry-safe.

## Behavioral invariants

1. Pending/rejected expenses do not change balances; an approved expense contributes exactly once.
2. Allocation shares sum exactly to the expense total, with remainder assignment as specified in FR-005/NFR-001.
3. Each accepted payment immediately reduces only its specified outstanding debtor/creditor pair and appears once in history.
4. Report and notification responses are scoped to current authorization.
5. Equivalent operations from web, Android, and iOS yield the same approval, allocation, ledger, and payment outcomes (NFR-004).
