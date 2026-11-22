# Babelii-Diofis

Connect your Hermes Agent to **Babelii** (Diofis Enterprise, PT / Data Infra Ekselen), an Indonesian **Commerce Platform**: pulsa (phone credit), data vouchers, data packages, PLN tokens, bill payments, e-wallet top-ups (DANA, GoPay, OVO, ShopeePay) and game vouchers, plus physical goods, food, hotspot vouchers and services from other vendors. All of it is in this package.

The package is a **portable Agent Plugins v1 bundle**: one remote MCP connector plus two skills (the buying workflow, and one for signed-in accounts). There is no local agent code and no bundled binary.

## Install

```
hermes plugins install diofis-enterprise/babelii-hermes --enable
```

Portable packages install **disabled**; `--enable` turns it on (or run `hermes plugins enable babelii-diofis` afterwards). To pin an exact version, install a full 40-character commit with `--ref <sha>`.
Plugin key: `babelii-diofis`. A Hermes Plugin Catalog submission is open ([NousResearch/hermes-agent#158469](https://github.com/NousResearch/hermes-agent/pull/158469)); this README will say so when it is listed.

Guide on the Babelii site: [babelii.diofis.net/en/ai/hermes](https://babelii.diofis.net/en/ai/hermes).

## How it works

- `mcp.json` points at Babelii's public commerce MCP, `https://babelii.diofis.net/mcp/commerce/` (Streamable HTTP, no sign-in). The server key is `babelii-diofis`, so Hermes names the tools `mcp__babelii_diofis__<tool>`.
- The bundled `babelii-commerce` skill walks the agent through: search products, ask only for what is missing, `prepare_purchase`, show one summary, the buyer confirms, `execute_purchase`, hand over the payment link. It also covers the Marketplace (`search_marketplace`: how to read each listing's `order_via`, when to use `search_products` and when only the listing's `buy_url`, and that `buy_requests` are what buyers are *looking for*, not items for sale), how to read `needs_inquiry` / `open_denom` / `sold_out`, and the transaction policy (quote validity, renewals, limits, a limit refusal is final) that every `prepare_purchase` result also states in its `policy` block.
- The skill was named `babelii-ppob` up to v0.3.1; it is now `babelii-commerce` because one skill covers everything Babelii sells. Nothing was removed.
- **Payment method.** The default (a payment link, QRIS first) applies unless the buyer asks otherwise. `prepare_purchase` returns the default and the choices; if the buyer already named how to pay, the agent uses it, otherwise the summary tells them they can change it.
- Anonymous orders are paid by the buyer through the returned payment link; fulfilment starts after the payment settles. No card data or passwords ever pass through the agent.
- Buyers with a Babelii account can point the same server at `https://babelii.diofis.net/mcp/account/` (OAuth sign-in once): see **With a Babelii account** below. The package itself does not sign anyone in and starts on the endpoint without sign-in. This is a limit of the portable plugin format, not a choice: a portable `mcp.json` entry may only carry `type`, `url` and `headers`, its URL is taken literally (no placeholders or parameters), and OAuth is switched on only by `auth: oauth` in your own `config.yaml`. So the account address cannot be shipped or selected from inside the package; the one-line command below is the whole step.

## With a Babelii account

The account endpoint is the same tools behind a sign-in, so it is the **same server key** (`babelii-diofis`) with a different address: nothing else changes, and there is only one Babelii server in your Hermes. Signed in, orders are paid from the account balance at the account's own price (no payment link unless the buyer asks for one), within the limits the customer sets on the consent page, and the account tools appear.

```
hermes mcp add babelii-diofis --url https://babelii.diofis.net/mcp/account/ --auth oauth
```

Hermes opens the Babelii sign-in once and keeps the token itself; this package stores no credentials. An entry with the same name in your `config.yaml` replaces the package's own, so add `trust: untrusted` to it if you want to keep Hermes asking before write-capable tools:

```yaml
mcp_servers:
  babelii-diofis:
    url: https://babelii.diofis.net/mcp/account/
    auth: oauth
    trust: untrusted
```

- **Pay from the balance, or another way.** Balance is the default and needs no extra question. If the buyer says they want to pay another way (a QRIS or payment link), the agent passes `pay_with=gateway`; that order waits for payment and is not fulfilled until the link is paid. If the balance does not cover the price, the summary says so before asking for consent and offers the link.
- **Account tools** (read only): `account_overview`, `account_balance_history`, `account_topups`, `account_schedules`, `account_schedule_runs` and `account_notifications`. They are listed only when the sign-in granted account access (`account:read`, shown on the consent page, or the *Account data* choice on a personal token); connections made before that scope existed need to sign in again to get them. `purchase_history` and `search_knowledge` are also available on the account endpoint.
### On a server without a browser (VPS, home lab)

Two ways, pick one:

**A. OAuth, with a longer wait.** `hermes mcp add` waits only 30 seconds for the sign-in by default, which is not enough to open the link, sign in and approve. Put a longer wait in the server entry, then log in separately:

```yaml
mcp_servers:
  babelii-diofis:
    url: https://babelii.diofis.net/mcp/account/
    auth: oauth
    trust: untrusted
    connect_timeout: 300
```

```
hermes mcp login babelii-diofis
```

Open the printed address in any browser (it does not have to be on the server), sign in and approve. The browser then fails to open `http://127.0.0.1:<port>/callback?...` because that address points at your own computer: copy the whole address from the browser's address bar and paste it into the Hermes prompt ("Or paste the redirect URL here"). Then `hermes mcp test babelii-diofis` should list the account tools.

**B. A personal token, no sign-in flow.** In your Babelii account open **AI agent tokens**, create a token, choose what it may do (balance orders, and under *Account data* whether it may read balance and history), and set the limits. The token is shown once. Keep it in the Hermes `.env` and reference it from the server entry:

```
BABELII_TOKEN=<the token>
```

```yaml
mcp_servers:
  babelii-diofis:
    url: https://babelii.diofis.net/mcp/account/
    headers:
      Authorization: Bearer ${BABELII_TOKEN}
    trust: untrusted
```

The tools you see follow what the token may do: a token without *Account data* does not list the `account_*` tools. Revoke it any time from the same page.

- **Top up the balance.** With the account endpoint the agent can also top up the balance: `execute_topup` creates a payment link for the amount the buyer approved (minimum Rp10,000, at most Rp1,000,000 per request, up to 3 unpaid at a time). Nothing is debited, and the balance grows only after the link is paid. It needs the right to order, and `confirmed=true` after the buyer approved the amount.
- Ordering and reading the account are separate grants: the consent page also sets the per-order, per-day and per-hour limits, and the customer can revoke the connection at any time from the AI agent tokens page of their account.

## Trust

`plugin.json` declares the server **`trust: untrusted`**. In Hermes this means every write-capable tool call (any tool without a `readOnlyHint: true` annotation: `prepare_purchase`, `inquire_purchase`, `execute_purchase`, `renew_purchase`) asks for your approval before it runs, and such calls are refused in unattended runs (cron). A server entry with the same name in your own `config.yaml` replaces this declaration, so you can choose a different trust level deliberately.

## Disclosure (what a user should know)

- **Network:** every tool call, on either endpoint, goes to `babelii.diofis.net`, and nowhere else (the OAuth sign-in is also served from there). Destination phone numbers or account identifiers and the buyer name you provide are sent there to place the order. The payment link returned to the buyer opens in the buyer's own browser; the plugin never contacts a payment provider.
- **Money movement:** `prepare_purchase` creates no order; `execute_purchase` places a real order. Without sign-in it becomes payable through a link; with a signed-in account it is paid from the balance by default, or becomes payable through a link when the buyer asks for that. Prices come from the live catalogue. Agent orders are limited (currently Rp25,000 per order and 3 per hour per identity; `discover_babelii` returns the current limits).
- **Local effects:** the plugin runs no shell commands, starts no background processes, reads no files outside its own folder, stores no credentials, and sends no telemetry. (With the optional account sign-in, the OAuth token is stored by Hermes, not by this package.)
- **Remote toolset:** the MCP tools are served by the remote server at runtime and are therefore **not** covered by the pinned commit of this package. The package pins only the configuration and the skill.
- **Data:** see the [privacy policy](https://babelii.diofis.net/en/privacy-policy) and [terms](https://babelii.diofis.net/en/terms-of-service).
- **Support:** post-sale questions go through Babelii's separate support MCP (`https://babelii.diofis.net/mcp/support/`) or the [contact page](https://babelii.diofis.net/contact).

## Legal

Plugin author and maintainer: **Data Infra Ekselen**, brand **Diofis Enterprise**, product unit **Babelii**. MIT licensed (see `LICENSE`).
