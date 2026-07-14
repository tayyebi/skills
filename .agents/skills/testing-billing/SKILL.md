---
name: testing-billing
description: Use when testing financial correctness in a product with a wallet/ledger, invoices, metering, or checkout — verifying that balances, refunds, and invoice totals reconcile exactly across every endpoint that reports them, not just that a payment "went through." Script-driven (curl/jq/bc-style arithmetic, not eyeballing). Distinct from testing-integration, which covers payment-gateway connectivity and signature/webhook plumbing — this skill is business-logic and arithmetic correctness once money has already moved.
---

# Billing & Financial Correctness Testing

Verify the numbers. A checkout that returns 200 isn't proof of correctness — the wallet must be debited by *exactly* the price, the invoice total must match, the ledger must sum to the reported balance, and a refund must never exceed what was actually charged. Every claim in this skill is checked with arithmetic against the actual API responses, not read off a UI.

## When to use

- Testing a wallet/credit balance, invoice generation, metered billing, refunds, or checkout math.
- Any change to pricing, discount, proration, refund, or budget-cap logic — these are exactly the paths where an off-by-a-fee or double-counting bug hides.
- **Not** for "does the Stripe/PayPal charge actually succeed" or "is the webhook signature valid" — that's testing-integration. This skill assumes the payment step works and asks whether the resulting numbers are correct and consistent.

## Core pattern

**1. Snapshot, act, snapshot, diff.** For every mutating financial action (add funds, checkout, refund, cancel), capture the balance/ledger/invoice state immediately before and immediately after, and assert the *delta* equals the expected value exactly — not "roughly," to the cent. A silently-absorbed fee (funded $10, credited $9.95) is a real bug that only shows up as a delta mismatch.

**2. Ledger reconciliation is the anchor check.** Sum every credit and debit across all pages of the ledger and compare it to the balance reported by the summary/costs endpoint. They must match within your currency's smallest unit. This single check catches an entire class of bug — a debit posted to the ledger but not applied to the cached balance, or vice versa — that per-transaction checks miss.

**3. Refunds must never fully undo a charge for consumed service.** Provision something billable, let it run briefly, then cancel/delete it and check the resulting credit. The credit must be strictly less than the original charge (some usage occurred and should be non-refundable), and the final balance must be strictly less than the pre-purchase baseline. "Refund equals full charge" is a free-service bug.

**4. Failure paths must leave money untouched.** Attempt checkout with insufficient balance, an empty cart, or a malformed line item. Assert both the HTTP status is a correct 4xx (not a 500 that happens to also fail safely) and — critically — that the balance is byte-for-byte unchanged afterward. A failed charge that still debits the wallet is worse than the checkout bug itself.

**5. Cross-endpoint consistency.** The same fact reported by two different endpoints must agree: summary balance vs. ledger sum; invoice pagination's declared `total` vs. actual returned items; a list endpoint's item count vs. individually fetching each item by id. Divergence here means the backend has two sources of truth that can drift.

**6. Change-detection snapshots for isolation.** Before an action that should be financially inert (e.g. reading a report, adding an unrelated resource), snapshot unrelated state (another resource list, the wallet balance) and confirm it's byte-identical after. This catches accidental side effects — an unrelated mutation nudging the wallet, or a read endpoint that isn't actually read-only.

**7. Format-validate every financial record, not just its value.** IDs are well-formed (UUID or whatever the system's id scheme is), timestamps are parseable and in the expected format, amounts are numeric and use a consistent sign convention, and totals are non-negative unless a negative is meaningful (e.g. a credit line item).

**8. Budget/spend-cap arithmetic.** If the product supports spend limits, verify the stored limit round-trips exactly, and that an attempted purchase exceeding the cap is blocked with the correct status — and doesn't debit anything.

## A minimal check pattern

```bash
before=$(get_balance)
response=$(add_funds 10.00)
after=$(get_balance)
delta=$(awk -v a="$after" -v b="$before" 'BEGIN{printf "%.2f", a-b}')
[[ "$delta" == "10.00" ]] || fail "funded 10.00 but balance moved by $delta"
```

The pattern generalizes: wrap every check in a `pass`/`fail` helper, use exact decimal comparison (avoid floating-point `==` on raw doubles — round to the currency's minor unit first), and keep a running pass/fail tally so a full pass produces one clear summary line.

## Reporting format

```
FAIL  delta == 10.00  (got 9.95)
  -> Wallet credited $9.95 after adding $10.00 -- $0.05 silently absorbed
  -> Endpoint: POST /add-funds
  -> Likely: gateway fee deducted from credited amount without disclosure
  -> Severity: Medium

FAIL  refund credit < original charge  (credit=5.00, charged=5.00)
  -> Full refund with zero usage deducted -- free service possible
  -> Severity: High
```

## Pitfalls

- Comparing floats with naive equality instead of rounding to the currency's minor unit first — introduces false failures (and can mask real off-by-a-cent bugs the other direction).
- Checking only the happy path's final state and skipping the ledger-sum-vs-balance reconciliation, which is where most real drift hides.
- Trusting that a 4xx status code alone means "handled correctly" without also confirming the balance wasn't touched.
- Testing refund/cancel logic with zero elapsed time between provision and cancel — some usage-based charge logic only misbehaves once at least one billing tick has occurred.
- Treating "the UI shows the right number" as sufficient — the UI can format a wrong number correctly. Always verify at the API/data layer.
- Skipping cross-endpoint consistency checks because "it's the same data" — different endpoints often hit different code paths or caches.

## Checklist

- [ ] Every mutating financial action: snapshot before/after, delta asserted exactly
- [ ] Ledger sum reconciles with reported balance (within the minor currency unit)
- [ ] Refund/cancel leaves a strictly-positive net cost when usage occurred
- [ ] Failed checkout (insufficient funds, empty cart, bad input) returns correct 4xx AND leaves balance untouched
- [ ] Cross-endpoint facts agree (summary vs. ledger, pagination totals vs. actual counts)
- [ ] Unrelated actions produce zero side effects on financial state (change-detection snapshot)
- [ ] Every financial record format-validated (id, timestamp, amount, sign convention)
- [ ] Budget/spend-cap limits round-trip correctly and block over-cap purchases without side effects
- [ ] Findings state expected vs. actual delta, exact endpoint, and severity
