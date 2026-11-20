# Babelii-Diofis

Connect your Hermes Agent to **Babelii** (Diofis Enterprise, PT / Data Infra Ekselen) — an Indonesian PPOB platform: pulsa (phone credit), data vouchers, data packages, PLN tokens, bill payments, e-wallet top-ups (DANA, GoPay, OVO, ShopeePay) and game vouchers.

The package is a **portable Agent Plugins v1 bundle**: one remote MCP connector plus the buying workflow skill. There is no local agent code and no bundled binary.

## Install

```
hermes plugins install https://github.com/diofis-enterprise/babelii-hermes
```

Plugin key: `babelii-diofis`. Or from the Desktop app's plugin catalog / onboarding card once listed.

## How it works

- `mcp.json` points at Babelii's public commerce MCP: `https://babelii.diofis.net/mcp/commerce/` (Streamable HTTP, no sign-in).
- The bundled skill walks the agent through: search products → ask only for what's missing → `prepare_purchase` → show one summary → buyer confirms → `execute_purchase` → hand over the payment link.
- Anonymous orders are paid by the buyer through the returned payment link (duitku); fulfilment starts after the payment settles. No card data or passwords ever pass through the agent.
- The server is declared **`trust: untrusted`** in `plugin.json`, so Hermes asks before money-moving tools run and refuses them in unattended runs.

## Disclosure (what a user should know)

- **Network:** every tool call goes to `babelii.diofis.net`. Destination phone numbers / account identifiers and buyer names you provide are sent there to place the order.
- **Money movement:** `prepare_purchase` creates no order; `execute_purchase` places a real order that becomes payable. Prices come from the live catalogue.
- **Remote toolset:** the MCP tools are served by the remote server at runtime and are therefore **not** covered by the pinned commit of this package. The package only pins configuration + the skill.
- **No stored credentials:** the public connector needs no API key. Account mode (paid from balance, or support tickets) is out of scope for this first version.
- **Support:** post-sale questions go through Babelii's separate support MCP (`/mcp/support/`).

## Legal

Plugin author/maintainer: **Data Infra Ekselen** — brand **Diofis Enterprise**, product unit **Babelii**. Registered in the Hermes Plugin Catalog as `babelii-diofis`.
