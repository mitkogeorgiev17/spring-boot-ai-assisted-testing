# Payments and money

Read this **before planning** any task that touches money. Money code fails
differently from ordinary code: the failure mode is not a stack trace, it is a
customer charged twice, a payment captured with no order behind it, or a balance
that no longer reconciles. Those failures are silent, and they are found by
customers rather than by tests.

## When this document applies

Any task touching: payment, charge, capture, authorization, refund, chargeback,
payout, invoice, billing, subscription, order total, price, discount, tax,
wallet, balance, ledger, or currency.

If a task touches any of these, it routes through `architect` first, regardless
of how small it looks. There is no such thing as a trivial change to money.

---

## 1. Dependency failure matrix (required in the plan)

Every outbound call in a money flow gets a row per failure mode. This table is
part of the plan and is approved before any code is written.

| Outcome | State transition | User sees | Retry safe? |
|---|---|---|---|
| `2xx` success | | | |
| `2xx` but declined | | | |
| `4xx` invalid request | | | |
| `409` duplicate | | | |
| `422` business rejection | | | |
| `5xx` provider error | | | |
| **Timeout** | | | |
| Connection failure | | | |
| Malformed / unparseable body | | | |

### Timeout is the row that matters

A timeout means **the outcome is unknown**. The provider may have charged the
customer. Treating a timeout as a failure is how customers get charged for
orders that were never created.

A timeout must transition to an explicit unresolved state — `PENDING_UNKNOWN`,
or equivalent — and resolve through one of:

- Querying the provider for the operation's status by idempotency key
- A reconciliation job that sweeps unresolved records
- The provider's webhook arriving later

Never map a timeout onto the same state as an explicit decline. Never retry a
money-moving call after a timeout without an idempotency key.

### Distinguish declined from failed

A `2xx` carrying `status: declined` is a **successful call with a business
rejection** — the provider worked, the card did not. A `5xx` is a **failed
call** — the outcome may be anything. These are different states, produce
different user messages, and have different retry semantics. Collapsing them
into one error path is a common and expensive bug.

---

## 2. Representing money

- **`BigDecimal` only.** Never `double`, `float`, or `Double`. Ever.
- Declare scale and rounding explicitly at every operation:
  `amount.setScale(2, RoundingMode.HALF_UP)`. Never rely on a default.
- **Currency travels with the amount.** A bare `BigDecimal amount` field is a
  defect — pair it with a currency, or use a value type carrying both. Never
  compare, add, or total amounts in different currencies.
- Database columns: `NUMERIC(19, 4)` or wider. Never `FLOAT` or `DOUBLE`.
- Integer minor units (cents) are acceptable if the project already uses them —
  match the existing project. Never mix the two representations in one codebase.

---

## 3. Idempotency

Every endpoint that moves money accepts an **idempotency key** and is safe to
call twice.

- The key comes from the client, or is derived deterministically from the
  business operation. Never generated fresh on each attempt.
- Persist the key with the resulting operation, with a unique constraint.
- A repeat call with a known key returns the **original result** — it does not
  execute again and does not error.
- The same key is passed through to the provider on every retry of that
  operation.

Without this, every retry, every double-click, and every client timeout is a
potential double charge.

---

## 4. State machines, not flags

Payment status is an explicit enum with defined transitions. Never a set of
booleans (`paid`, `refunded`, `failed`) — booleans permit impossible
combinations and cannot express "unknown".

```
INITIATED → AUTHORIZED → CAPTURED → REFUNDED
    ↓            ↓           ↓
 FAILED      DECLINED   PARTIALLY_REFUNDED

any → PENDING_UNKNOWN (timeout) → resolves to a terminal state
```

Define the legal transitions and reject illegal ones in the service. Persist
every transition — see §7.

---

## 5. Transaction boundaries

**Never make an external HTTP call inside a transaction that mutates money.**

A transaction held open across a network call holds database locks for the
duration of a remote timeout, and a rollback after a successful provider call
leaves the money moved and no record of it.

The shape:

1. Transaction: persist the intent, in a pending state, with its idempotency
   key. Commit.
2. Outside any transaction: call the provider.
3. Transaction: record the outcome and transition the state. Commit.

Step 1 committing before step 2 is what makes the operation recoverable — a
crash between steps leaves a pending record that reconciliation can resolve.

---

## 6. Webhooks

Provider callbacks are untrusted, unordered, and repeated.

- **Verify the signature** on every webhook before reading the body. An
  unverified webhook is an unauthenticated state change.
- **Handle duplicates** — the same event will arrive more than once. Store
  processed event ids and ignore repeats.
- **Handle out-of-order delivery** — a `captured` event can arrive before the
  `authorized` one. Validate against the state machine and ignore transitions
  that move backwards.
- Respond `2xx` quickly and process asynchronously where the provider expects
  it; a slow handler causes the provider to retry, compounding duplicates.
- Never trust an amount in a webhook without checking it against the stored
  record.

---

## 7. Audit trail

Money history is **append-only**.

- Every state transition writes a new row: what changed, when, triggered by
  what, with the provider's reference id.
- **Never overwrite a balance in place.** A balance is derived from its ledger
  entries, or is a cached projection that can be rebuilt from them.
- Never `DELETE` a payment record. Reversal is a new compensating entry.
- Store the provider's transaction id on every record — without it,
  reconciliation is impossible.

---

## 8. Logging and PII

- **Never log** a full card number (PAN), CVV, expiry, full bank account number,
  or any raw provider credential. Not at any level, not in an exception message,
  not in a request/response dump.
- Log the last four digits, a token, or the provider's reference id.
- Redact request bodies before logging outbound payment calls.
- Per `backend-developer.md`, services log `info`/`warn` only — error logging
  remains in the exception handler.

---

## 9. Testing

Money tests are not optional and they are not "happy path plus one error".

**Every row of the failure matrix gets a WireMock stub and a test**, including:

- Timeout — `withFixedDelay` exceeding the client's read timeout, asserting the
  record lands in the unresolved state, not in `FAILED`
- Connection reset
- `2xx` with a declined body
- `5xx`
- Malformed JSON body

**Also required:**

- **Concurrent double-submit** — two simultaneous calls with the same
  idempotency key produce exactly one charge and two identical responses.
- **Retry safety** — replaying the same request after a timeout does not create
  a second charge.
- **Rounding** — amounts that expose scale and rounding behavior, including
  values that round differently under `HALF_UP` and `HALF_EVEN`.
- **State machine** — every illegal transition is rejected.
- **Webhook** — duplicate delivery, out-of-order delivery, and a bad signature.

Follow the patterns in `INTEGRATION_TESTING.md` for WireMock setup. A money
feature is not done until every failure-matrix row has a passing test behind it,
with the output pasted per `WORKFLOW.md` §5.

---

## 10. Review checklist

- [ ] Failure matrix completed in the plan and approved before coding
- [ ] Timeout maps to an unresolved state with a defined resolution path
- [ ] Declined and failed are distinct states
- [ ] `BigDecimal` throughout, explicit scale and rounding, currency paired
- [ ] Idempotency key accepted, persisted, uniquely constrained, passed through
- [ ] Status is an enum with enforced transitions, not booleans
- [ ] No external call inside a money-mutating transaction
- [ ] Webhooks: signature verified, duplicates ignored, out-of-order handled
- [ ] Append-only audit trail with provider reference ids
- [ ] No PAN/CVV/account numbers in any log
- [ ] A test per failure-matrix row, plus concurrency, rounding, and state tests
- [ ] Test output pasted in the report
