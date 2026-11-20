---
name: babelii-ppob
description: Use when buying Indonesian digital products (pulsa, data vouchers, packages, PLN tokens, bills, e-wallet top-ups, game vouchers) through Babelii's commerce MCP from Hermes Agent.
---

# Buy PPOB on Babelii (Hermes Agent)

Babelii (Diofis Enterprise) is an Indonesian PPOB platform with a public commerce MCP:
search → prepare → buyer confirms → execute → hand over the payment link.
You act for the buyer; never present yourself as Babelii or its adviser Abel.

## Connection

- Remote MCP: `https://babelii.diofis.net/mcp/commerce/` (Streamable HTTP, no sign-in).
- In Hermes: `hermes mcp add` (HTTP transport, url above) — tools load as `mcp_babelii_<tool>`.
- Or install the portable package (`plugin.json` + `mcp.json` + this skill) from the repo and set
  `servers.babelii.trust: "untrusted"` so money-moving calls ask first and are refused unattended.
- Plain HTTP fallback: JSON-RPC `POST` to the URL, headers `Content-Type: application/json`,
  `Accept: application/json, text/event-stream`, and an app `User-Agent` like `hermes-agent/1.0`.
  If blocked with `retryable=false`, `owner_action_required`, or Cloudflare 1010: stop and report (include `CF-Ray`). Never rotate UAs to push through.
- Sequence: `initialize` (protocolVersion `2025-03-26`) → `notifications/initialized` → `tools/call`; return `Mcp-Session-Id`.

## Tools

`discover_babelii`, `search_products`, `get_product`, `search_marketplace`, `ask_abel`,
`prepare_purchase`, `inquire_purchase`, `execute_purchase`, `purchase_status`, `renew_purchase`.
(`purchase_history` / `search_knowledge` appear in some docs but are NOT returned for anonymous identities.)
`product_ref` = `catalogue:<group>:<code>`.

## Flow

1. `search_products` once. Filters: `query`, `catalogue`, `operator`, `requires_destination`, `max_price`,
   `min/max_volume_gb`, `min/max_validity_days`, `sort` (`relevance|price_asc|value_per_gb_day`), `limit<=20`.
2. Ask only what's missing: `destination` (if `requires_destination`), `amount` (if `open_denom`),
   `buyer_name` (required for anonymous buyers; never invent it).
3. Match the nominal EXACTLY to a catalogue row ("5000" = Rp5,000, not Rp50,000); confirm briefly if ambiguous.
4. `prepare_purchase(product_ref, destination, buyer_name[, amount])` → `intent_id`, `buyer_name`,
   `maximum_amount`, `expires_at` (~5 min). Expiring soon? `renew_purchase(intent_id, quote_version)`
   (max 3 per 30 min, +5 min each) → show the new summary and get confirmation again.
5. Show ONE summary: product, destination, customer name, max price. Wait for explicit consent.
6. On consent: `execute_purchase(intent_id, confirmed=true, terms_accepted=true[, quote_version])`
   → `order_number`, `payment_url`, `track_url`. Hand the buyer the payment link.
7. `purchase_status` accepts `intent_id`, `order_number`, `order_id`, or `track_url`.
   Guest status by `order_number` is summary-level only; full details need the `track_url`.

## Rules

- Never execute without the buyer's confirmation of an unchanged summary; advice/recommendation is not permission.
- Retrying the same intent is safe; on timeout check `purchase_status` on the SAME intent — do not prepare a new one.
- Agent channel limits (from `discover_babelii` → `agent_limits`): Rp25,000/order, 3 orders/hour per identity.
- `available:false` / `sold_out:true` means out of stock right now — say so, never order it.
- Don't claim delivery before the order says so; `payment_status:paid` is not fulfilment.
- Support (post-sale, separate endpoint, no transaction authority): `https://babelii.diofis.net/mcp/support/`.
- Do not use the web form `/guest/orders` (Turnstile-protected, human path).
