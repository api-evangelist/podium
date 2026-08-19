---
name: Invoice a customer and collect payment
description: Create a Podium invoice, set up the payment method, charge it idempotently, and handle refunds and cancellations — the one Podium flow with a real idempotency contract.
api: openapi/podium-payments-openapi.yml
operations: [invoice.create, invoice.setup, invoice.charge, invoice.get, invoice.index, invoice.cancel, invoice.refund, payment.get, Refund.create, Refund.get, reader.get]
scopes: [read_payments, write_payments]
---

# Invoice a customer and collect payment (Podium)

Base URL `https://api.podium.com/v4/`, OAuth bearer token with `write_payments`,
`podium-version: 2021.04.01`.

## Steps

1. **Create the invoice.** `invoice.create` (`POST /v4/invoices`) for a location and contact with
   the amount and line items.
2. **Set up the payment method.** `invoice.setup` (`POST /v4/invoices/{uid}/setup`) creates the
   setup intent for card-not-present collection. For in-person collection, resolve the terminal
   with `reader.get` (`GET /v4/readers/{uid}`).
3. **Charge — idempotently.** `invoice.charge` (`POST /v4/invoices/{uid}/charge`) accepts an
   optional `idempotencyUid` (UUID) body field described as "Optional idempotency identifier to
   make retries safe." **Always send it.** Generate one UUID per logical charge attempt and reuse
   it verbatim on every retry of that attempt. This is the only idempotency contract Podium
   publishes anywhere in the API.
4. **Confirm.** `invoice.get` (`GET /v4/invoices/{uid}`) and `payment.get`
   (`GET /v4/payments/{uid}`), both `read_payments`.
5. **Cancel or refund.** `invoice.cancel` (`POST /v4/invoices/{uid}/cancel`),
   `invoice.refund` (`POST /v4/invoices/{uid}/refund`), or a standalone
   `Refund.create` (`POST /v4/refunds`) read back with `Refund.get` (`GET /v4/refunds/{uid}`).
6. **Reconcile from events, never from polling.** Subscribe to `invoice.created`,
   `invoice.payment_created`, `invoice.payment_succeeded`, `invoice.payment_failed`,
   `invoice.marked_as_paid`, `invoice.refund_created`, `invoice.refund_failed`,
   `invoice.disabled`.

## Rules

- Only `invoice.charge` is idempotent. `invoice.create`, `invoice.refund`, `Refund.create` and
  `invoice.cancel` are NOT — a blind retry can double-refund. Read the invoice back first.
- Webhook deliveries are retried ~15 times over ~8 hours and then the webhook is DISABLED, and
  subsequent events are paused behind a failing one. A payment-critical consumer must monitor its
  own endpoint rather than rely on Podium's notification email.
- Verify every webhook body with HMAC-SHA256 over `{podium-timestamp}.{raw body}` against
  `podium-signature` before acting on money movement.
