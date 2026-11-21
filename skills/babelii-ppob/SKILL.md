---
name: babelii-ppob
description: Use when buying Indonesian digital products (pulsa, data vouchers, data packages, PLN tokens, bills, e-wallet top-ups, game vouchers) through Babelii's commerce MCP from Hermes Agent.
---

# Buy PPOB on Babelii (Hermes Agent)

Babelii (Diofis Enterprise) is an Indonesian PPOB platform with a public commerce MCP:
search → prepare → buyer confirms → execute → hand over the payment link.
You act for the buyer; never present yourself as Babelii or as its adviser Abel.

## Connection

- Remote MCP: `https://babelii.diofis.net/mcp/commerce/` (Streamable HTTP, no sign-in, stateless: there is no session id to keep).
- Installed from this package, Hermes lists the tools as `mcp__<server>__<tool>` (here `mcp__babelii_diofis__search_products` and so on);
  added by hand with `hermes mcp add`, the prefix is the server name you chose. Below, tools are named without the prefix.
- Buyers with a Babelii account can connect `https://babelii.diofis.net/mcp/account/` instead (OAuth sign-in once) to pay from their balance at
  their account price. This package uses the endpoint without sign-in and cannot sign anyone in: if the buyer mentions an account or a balance,
  tell them about that address.
- Plain HTTP fallback, only if the MCP connector is unavailable: JSON-RPC `POST` to the URL with `Content-Type: application/json`,
  `Accept: application/json, text/event-stream` and an app `User-Agent` such as `hermes-agent/1.0`; send `initialize`
  (protocolVersion `2025-06-18`), then `notifications/initialized`, then `tools/call`. If blocked with `retryable=false`,
  `owner_action_required`, or Cloudflare 1010: stop and report (include `CF-Ray`). Never rotate user agents to push through.

## Tools

`discover_babelii`, `search_products`, `get_product`, `search_marketplace`, `ask_abel`,
`prepare_purchase`, `inquire_purchase`, `execute_purchase`, `purchase_status`, `renew_purchase`.
(`purchase_history` and `search_knowledge` exist only on the signed-in endpoint and are not returned here.)
`product_ref` = `catalogue:<group>:<code>`.

## Flow

1. `search_products` once. Filters: `query`, `catalogue`, `operator`, `requires_destination`, `max_price`,
   `min/max_volume_gb`, `min/max_validity_days`, `sort` (`relevance|price_asc|value_per_gb_day`), `limit<=20`.
2. Ask only what is missing: `destination` (if `requires_destination`), `amount` (if `open_denom`),
   `buyer_name` (required for anonymous buyers; never invent it). Phone and email are optional: never demand them.
3. Match the nominal EXACTLY to a catalogue row ("5000" = Rp5,000, not Rp50,000); confirm briefly if ambiguous.
4. `prepare_purchase(product_ref, destination, buyer_name[, amount])` returns `intent_id`, `buyer_name`, `maximum_amount`,
   `expires_at` (about 5 minutes) and a `payment` block: `payment.default` (the method that will be used),
   `payment.options` (what the buyer may choose instead) and `payment.changeable`.
   Expiring soon? `renew_purchase(intent_id, quote_version)` (at most 3 per 30 minutes, +5 minutes each), then show the new summary and ask again.
5. Show ONE summary: product, destination, customer name, max price, and the payment method. Wait for explicit consent.
6. On consent: `execute_purchase(intent_id, confirmed=true, terms_accepted=true[, quote_version][, pay_with, channel])`
   returns `order_number`, `payment_url`, `track_url`. Hand the buyer the payment link.
7. `purchase_status` accepts `intent_id`, `order_number`, `order_id`, or `track_url`.
   Status by `order_number` is summary-level only; full details need the `track_url`.

## Payment method

- The default stays in force when you pass nothing (a payment link, QRIS first).
- If the buyer already said how they want to pay (QRIS, a named channel, and so on), use it: pass `pay_with` (and `channel` for
  `gateway`) to `execute_purchase`, and name that method in your summary.
- If they did not, show the default in the summary and add ONE short line that they can change the method when `payment.changeable`
  is true, naming the choices from `payment.options` in plain words.
- Offer only what `payment.options` lists. Never invent a method. On this endpoint the options are the payment link channels;
  cash and coin are not offered here. A method that is not listed is refused by the server.

## Rules

- Never execute without the buyer's confirmation of an unchanged summary; advice or a recommendation is not permission.
- Retrying the same intent is safe; on a timeout check `purchase_status` on the SAME intent. Do not prepare a new one.
- Agent limits are returned by `discover_babelii` (`agent_limits`); at the time of writing: Rp25,000 per order and 3 orders per hour per identity.
- `available:false` / `sold_out:true` means out of stock right now: say so and never order it.
- Server errors are written for buyers: relay them as they are.
- Do not claim delivery before the order says so; `payment_status:paid` is not fulfilment.
- Support (after-sales, separate endpoint, no transaction authority): `https://babelii.diofis.net/mcp/support/`.
- Privacy policy: `https://babelii.diofis.net/en/privacy-policy`; terms: `https://babelii.diofis.net/en/terms-of-service`.
- Do not use the web form `/guest/orders` (Turnstile-protected, a human path).
