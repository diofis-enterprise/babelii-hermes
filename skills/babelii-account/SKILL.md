---
name: babelii-account
description: "Use when the Babelii server is connected to a signed-in customer account (the /mcp/account/ endpoint): paying from the account balance, paying by link on request, and reading the account's balance, history, top-ups, schedules and notifications."
---

# Babelii with a signed-in account (Hermes Agent)

Use this together with `babelii-commerce`: the buying flow is the same (search, prepare, buyer confirms, execute). What changes is who pays and what you can read.
You act for the customer who signed in; never present yourself as Babelii or as its adviser Abel.

## How to tell which endpoint you are on

Look at the tools you were given, not at what you expect:

- `account_overview` (and the other `account_*` tools) are listed: the customer signed in AND allowed reading the account.
- `purchase_history` is listed but no `account_*` tool: they signed in but did not allow reading the account (or signed in before that permission existed).
  Say so and tell them they can sign in again to allow it (or, for a personal token, set *Account data* to read on the AI agent tokens page); never guess balances or history.
- Neither is listed: this is the anonymous endpoint. Orders there are paid through a link. If the customer says they have an account or a balance, tell them to run
  `hermes mcp add babelii-diofis --url https://babelii.diofis.net/mcp/account/ --auth oauth`.

## Paying

- The default is the account's own source of funds (balance, or the partner wallet): no payment link and no extra question. `prepare_purchase` returns it as `payment.default`.
- Show it in the summary. If `payment.default.sufficient` is `false`, the balance does not cover the price: say so BEFORE asking for consent and offer the payment link.
- If the customer says they want to pay another way (a QRIS or payment link), pass `pay_with=gateway` and a `channel` from `payment.options` to `execute_purchase`.
  Offer only what `payment.options` lists (it can include cash and coin when the account may use them); never pick one for the customer.
- A link order is NOT paid yet: it comes back with `payment_url` and the status `pending_payment`. Hand over the link, say the order is waiting for payment, and do not call it
  done or fulfilled until `purchase_status` says so.
- A balance order is paid the moment it is created; fulfilment can still take a while or fail, so report what `purchase_status` returns.
- The customer set per-order, per-day and per-hour limits when they signed in. A refusal for a limit is final: relay it, do not split an order to get around it.

## Reading the account (read only; none of these change anything)

| Tool | Use it for |
|---|---|
| `account_overview` | balance, credit limit, spendable amount, counts of pending top-ups, active schedules, unread notifications |
| `account_balance_history` | the balance ledger, newest first (`limit` up to 50; pass `next_before_id` as `before_id` for the next page) |
| `account_topups` | top-up requests and their status |
| `account_schedules` | scheduled purchases: product, masked destination, recurrence, status, next run |
| `account_schedule_runs` | what happened on each run of one schedule (`schedule_id` from `account_schedules`) |
| `account_notifications` | notifications, `unread_only` to filter |
| `purchase_history` | the account's orders |

- State amounts exactly as returned. Destinations are masked by the server; do not try to reconstruct them.
- Only the signed-in customer's own data is reachable. If asked about someone else's account, say you cannot.
- Never ask for passwords, PINs or OTPs. Signing in happens in the customer's browser on babelii.diofis.net, not through you.
