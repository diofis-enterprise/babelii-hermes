---
name: babelii-commerce
description: "Use when buying on Babelii through its commerce MCP from Hermes Agent: pulsa, data vouchers, data packages, PLN tokens, bills, e-wallet top-ups and game vouchers, plus items from other vendors (physical goods, food, hotspot vouchers, services) and what buyers are looking for."
---

# Buy on Babelii (Hermes Agent)

Babelii (Diofis Enterprise) is an Indonesian Commerce Platform: pulsa, data, PLN tokens, bills, e-wallets and game vouchers, plus physical goods, food,
hotspot vouchers and services from other vendors. Its public commerce MCP follows one flow for what can be ordered through
it: search → prepare → buyer confirms → execute → hand over the payment link.
You act for the buyer; never present yourself as Babelii or as its adviser Abel.

## Connection

- Remote MCP: `https://babelii.diofis.net/mcp/commerce/` (Streamable HTTP, no sign-in, stateless: there is no session id to keep).
- Installed from this package, Hermes lists the tools as `mcp__<server>__<tool>` (here `mcp__babelii_diofis__search_products` and so on);
  added by hand with `hermes mcp add`, the prefix is the server name you chose. Below, tools are named without the prefix.
- Buyers with a Babelii account can point the same server at `https://babelii.diofis.net/mcp/account/` (OAuth sign-in once) to pay from their
  balance at their account price: `hermes mcp add babelii-diofis --url https://babelii.diofis.net/mcp/account/ --auth oauth`. This package starts on the
  endpoint without sign-in and cannot sign anyone in: if the buyer mentions an account or a balance, tell them about that address
  (the `babelii-account` skill covers the signed-in case).
- Plain HTTP fallback, only if the MCP connector is unavailable: JSON-RPC `POST` to the URL with `Content-Type: application/json`,
  `Accept: application/json, text/event-stream` and an app `User-Agent` such as `hermes-agent/1.0`; send `initialize`
  (protocolVersion `2025-06-18`), then `notifications/initialized`, then `tools/call`. If blocked with `retryable=false`,
  `owner_action_required`, or Cloudflare 1010: stop and report (include `CF-Ray`). Never rotate user agents to push through.

## Tools

Tools marked **account only** exist only on the signed-in endpoint and are not returned on the default one; never call a tool that is not in your `tools/list`.

<!-- tools-table:begin (generated from the Hub toolspec: python -m hub.mcp.skilltable; do not edit by hand) -->
| Tool | Purpose |
|---|---|
| `discover_babelii` | identifies Babelii and the access your connection has (guest or account); call it first if you are unsure which tools are available |
| `search_products` | search products; every orderable row states the price and `requires_destination`, `needs_inquiry`, `open_denom` as true/false (nothing is left to guess). Pages of 10 rows (`limit` up to 20); a `more` note means there are more |
| `get_product` | details of one product (optional) |
| `search_marketplace` | browse Babelii's Marketplace: other vendors sell hotspot vouchers, food, services and goods. Read-only; hotspot vouchers in stock are ordered like ordinary products, other listings are bought through the `buy_url` on their product page. Each listing has an `order_via` that says how to buy it (see the Marketplace section). A result with `sold_out=true` means that vendor's stock is currently empty. `buy_requests` are things buyers are LOOKING FOR, not things for sale |
| `prepare_purchase` | prepares an order without creating it; for `needs_inquiry` products it runs the name/bill check itself. The result carries a `policy` block: validity, renewals left, per-order and hourly limits, and that a limit refusal is final |
| `execute_purchase` | creates the order from the result of `prepare_purchase`. The default payment method (`payment.default` in the summary) is used when `pay_with` is omitted; if the buyer already named how to pay, pass `pay_with` (and `channel` for gateway) from `payment.options`. A signed-in account pays from its balance by default; if the buyer asks for a payment link, `pay_with=gateway` creates an order that waits for payment (hand over `payment_url`; not done until paid) |
| `purchase_status` | order status; pass EXACTLY ONE of `intent_id`, `order_number`, `order_id` or `track_url` (two or none is refused) |
| `inquire_purchase` | separate customer name check (rarely needed; it has a cost, do not repeat it) |
| `renew_purchase` | refreshes the price of an already prepared quote (at most three times, no new order) |
| `purchase_history` | **account only** (login required): the account's order history. Not available on the connection without login; do not try it there |
| `account_overview` | **account only** (needs login and the account permission): own account summary: balance, credit limit, spendable amount, pending top-ups, active schedules, unread notifications. Read only; absent on the connection without login |
| `account_balance_history` | **account only** (needs login and the account permission): own balance ledger, newest first. Read only |
| `account_topups` | **account only** (needs login and the account permission): own top-up requests with status. Read only |
| `execute_topup` | **account only** (login required): requests a top-up of the own balance: creates a payment link (nothing is debited; the balance grows after the link is paid). Show the amount and get the account owner's approval first, then `confirmed=true`; hand over `payment_url`, never call it done |
| `account_schedules` | **account only** (needs login and the account permission): own scheduled purchases; the destination is masked. Read only |
| `account_schedule_runs` | **account only** (needs login and the account permission): run history of one scheduled purchase (schedule_id from `account_schedules`). Read only |
| `account_notifications` | **account only** (needs login and the account permission): own notifications. Read only |
| `ask_abel` | ask Abel, the separate product adviser, if this tool is available to you |
| `search_knowledge` | **account only** (login required): official explanation of how Babelii works; not a source of prices. Without login, use the guide at https://babelii.diofis.net/en/ai or the customer service address below |
<!-- tools-table:end -->

`product_ref` = `catalogue:<group>:<code>`.

## Which tool: digital product or Marketplace

- The buyer wants pulsa, data, a PLN token, a bill, an e-wallet top-up or a game voucher: `search_products`.
- The buyer asks what the Marketplace has, names a vendor, or wants food, a service, a hotspot voucher or goods: `search_marketplace` (read only; empty `query` = a sample).
  Then follow each listing's `order_via`, see **Marketplace** below. Not sure which side it is on? Try `search_products` first, then `search_marketplace`.

## Flow

1. `search_products` once. Filters: `query`, `catalogue`, `operator`, `requires_destination`, `max_price`,
   `min/max_volume_gb`, `min/max_validity_days`, `sort` (`relevance|price_asc|value_per_gb_day`), `limit<=20`.
2. Ask only what is missing: `destination` (if `requires_destination`), `amount` (if `open_denom`),
   `buyer_name` (required for anonymous buyers; never invent it). Phone and email are optional: never demand them.
   Every orderable row states `requires_destination`, `needs_inquiry` and `open_denom` as true/false (see **Reading a search row**).
3. Match the nominal EXACTLY to a catalogue row ("5000" = Rp5,000, not Rp50,000); confirm briefly if ambiguous.
4. `prepare_purchase(product_ref, destination, buyer_name[, amount])` returns `intent_id`, `buyer_name`, `maximum_amount`,
   `expires_at` (about 5 minutes), a `payment` block (`payment.default` = the method that will be used, `payment.options` = what the buyer may choose
   instead, `payment.changeable`) and a `policy` block (see **Policy**).
   Expiring soon? `renew_purchase(intent_id, quote_version)` (at most 3 per 30 minutes, +5 minutes each), then show the new summary and ask again.
5. Show ONE summary: product, destination, customer name, max price, and the payment method. Wait for explicit consent.
6. On consent: `execute_purchase(intent_id, confirmed=true, terms_accepted=true[, quote_version][, pay_with, channel])`
   returns `order_number`, `payment_url`, `track_url`. Hand the buyer the payment link.
7. `purchase_status` takes EXACTLY ONE of `intent_id`, `order_number`, `order_id` or `track_url` (two or none is refused). Use `intent_id` right after
   `execute_purchase`; a guest can use `order_number` (summary-level only) or the `track_url` (full details). Never invent or fetch a tracking URL.

## Reading a search row

- `needs_inquiry=true`: a bill or destination-name check is needed first (BPJS, PDAM, PLN postpaid, e-wallet top-up). Such rows often have `price_type: open` because the
  price comes from the check. `prepare_purchase` runs the check itself; do not call `inquire_purchase` separately unless you need the name before preparing.
- `open_denom=true`: the BUYER states the amount. Send `amount` (minimum Rp1,000) to `prepare_purchase`.
- These two are independent: a product can have either, both or neither.
- `available=false` with `sold_out=true`: a real product whose stock is empty right now. Say "out of stock" (not "does not exist") and never order it; it carries no `ref`.

## Marketplace

The Marketplace is sold by other vendors. `search_marketplace` returns `listings`, optional `buy_requests`, and a `marketplace` block with links.

- Each listing has `order_via`, which decides the flow:
  - starts with `search_products ...`: a stocked voucher. Find its title with `search_products`, then `prepare_purchase` like any product; the buyer may also open `buy_url` themselves.
  - starts with `buy_url ...`: bought only on that page (a form on the product page). Give the buyer the link; do NOT try `prepare_purchase`.
- `sold_out=true`: a real vendor product whose stock is empty right now; say so.
- `buy_requests` ("Barang yang sedang dicari") are other buyers' wishes: title, kind, quantity, budget, link, never names or contacts. They are NOT items for sale, so
  never offer one as something to buy. Vendors answer them at `buy_requests_url`; send a buyer who wants to join the search to that link.
- `search_marketplace` creates no order and takes no payment. State prices exactly as returned.

## Policy (shown with every `prepare_purchase`)

The result's `policy` block says what holds for this identity; tell the buyer the parts that matter rather than complying silently:

- The quote is valid for about 5 minutes (`valid_for_minutes`, `expires_at`). Renewals: at most 3 within 30 minutes (`renewals_remaining`), each needs a fresh confirmation.
- Limits: `max_order_amount` and `max_orders_per_hour` (for an agent without an account currently Rp25,000 per order and 3 per hour; always read the exact values from `policy`).
- A refusal because of a limit is FINAL: do not split the order, retry, or look for a way around it. Tell the buyer, and point them to the product page if useful.

## Payment method

- The default stays in force when you pass nothing (a payment link, QRIS first).
- If the buyer already said how they want to pay (QRIS, a named channel, and so on), use it: pass `pay_with` (and `channel` for
  `gateway`) to `execute_purchase`, and name that method in your summary.
- If they did not, show the default in the summary and add ONE short line that they can change the method when `payment.changeable`
  is true, naming the choices from `payment.options` in plain words.
- Offer only what `payment.options` lists. Never invent a method and never pick cash or coin on the buyer's behalf. A method that is not listed is refused by the server.

## Rules

- Never execute without the buyer's confirmation of an unchanged summary; advice or a recommendation is not permission.
- Retrying the same intent is safe; on a timeout check `purchase_status` on the SAME intent. Do not prepare a new one.
- Agent limits are returned by `discover_babelii` (`agent_limits`) and by every `prepare_purchase` (`policy`).
- Server errors are written for buyers: relay them as they are.
- Do not claim delivery before the order says so; `payment_status:paid` is not fulfilment.
- Support (after-sales, separate endpoint, no transaction authority): `https://babelii.diofis.net/mcp/support/`.
- Privacy policy: `https://babelii.diofis.net/en/privacy-policy`; terms: `https://babelii.diofis.net/en/terms-of-service`.
- Do not use the web form `/guest/orders` (Turnstile-protected, a human path).
