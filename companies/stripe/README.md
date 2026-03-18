# Stripe

> Payment infrastructure — idempotency, ledger design, fraud, correctness

## What to Study

- Idempotency keys — client-generated keys, server-side deduplication, retry safety
- Double-entry ledger — how every financial movement is recorded as paired debit/credit entries
- Payment rails integration — card networks (Visa/MC), ACH, SEPA, and abstraction layers over them
- Fraud ML — feature engineering on transaction signals, model serving latency constraints

## Why This Matters for Platform Architecture

Stripe's idempotency key pattern is the most copied API design in financial infrastructure — it solves the distributed systems problem of "did my payment go through?" without requiring client-side coordination. Its ledger design demonstrates how correctness-first thinking shapes every layer of a financial system.

## Study Checklist

Use [TEMPLATE.md](../../TEMPLATE.md) as your study guide.

- [ ] Business context & scale documented
- [ ] Requirements defined
- [ ] Architecture diagram created
- [ ] Data model analyzed
- [ ] Key trade-offs articulated
- [ ] At least 2 primary sources read
- [ ] Failure modes analyzed
- [ ] Can compare to a previously studied system

## Status

Not started
