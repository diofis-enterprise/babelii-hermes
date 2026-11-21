# Babelii-Diofis

Connect your Hermes Agent to **Babelii** (Diofis Enterprise, PT / Data Infra Ekselen), an Indonesian PPOB platform: pulsa (phone credit), data vouchers, data packages, PLN tokens, bill payments, e-wallet top-ups (DANA, GoPay, OVO, ShopeePay) and game vouchers.

The package is a **portable Agent Plugins v1 bundle**: one remote MCP connector plus the buying workflow skill. There is no local agent code and no bundled binary.

## Install

```
hermes plugins install diofis-enterprise/babelii-hermes --enable
```

Portable packages install **disabled**; `--enable` turns it on (or run `hermes plugins enable babelii-diofis` afterwards). To pin an exact version, install a full 40-character commit with `--ref <sha>`.
Plugin key: `babelii-diofis`. It is not yet in the Hermes Plugin Catalog; a submission is planned, and this README will say so when it is listed.

## How it works

- `mcp.json` points at Babelii's public commerce MCP, `https://babelii.diofis.net/mcp/commerce/` (Streamable HTTP, no sign-in). The server key is `babelii-diofis`, so Hermes names the tools `mcp__babelii_diofis__<tool>`.
- The bundled skill walks the agent through: search products, ask only for what is missing, `prepare_purchase`, show one summary, the buyer confirms, `execute_purchase`, hand over the payment link.
- **Payment method.** The default (a payment link, QRIS first) applies unless the buyer asks otherwise. `prepare_purchase` returns the default and the choices; if the buyer already named how to pay, the agent uses it, otherwise the summary tells them they can change it.
- Anonymous orders are paid by the buyer through the returned payment link; fulfilment starts after the payment settles. No card data or passwords ever pass through the agent.
- Buyers with a Babelii account can use `https://babelii.diofis.net/mcp/account/` (OAuth sign-in once) to pay from their balance. This package does not sign anyone in and uses the endpoint without sign-in.

## Trust

`plugin.json` declares the server **`trust: untrusted`**. In Hermes this means every write-capable tool call (any tool without a `readOnlyHint: true` annotation: `prepare_purchase`, `inquire_purchase`, `execute_purchase`, `renew_purchase`) asks for your approval before it runs, and such calls are refused in unattended runs (cron). A server entry with the same name in your own `config.yaml` replaces this declaration, so you can choose a different trust level deliberately.

## Disclosure (what a user should know)

- **Network:** every tool call goes to `babelii.diofis.net`, and nowhere else. Destination phone numbers or account identifiers and the buyer name you provide are sent there to place the order. The payment link returned to the buyer opens in the buyer's own browser; the plugin never contacts a payment provider.
- **Money movement:** `prepare_purchase` creates no order; `execute_purchase` places a real order that becomes payable. Prices come from the live catalogue. Agent orders are limited (currently Rp25,000 per order and 3 per hour per identity; `discover_babelii` returns the current limits).
- **Local effects:** the plugin runs no shell commands, starts no background processes, reads no files outside its own folder, stores no credentials, and sends no telemetry.
- **Remote toolset:** the MCP tools are served by the remote server at runtime and are therefore **not** covered by the pinned commit of this package. The package pins only the configuration and the skill.
- **Data:** see the [privacy policy](https://babelii.diofis.net/en/privacy-policy) and [terms](https://babelii.diofis.net/en/terms-of-service).
- **Support:** post-sale questions go through Babelii's separate support MCP (`https://babelii.diofis.net/mcp/support/`) or the [contact page](https://babelii.diofis.net/contact).

## Legal

Plugin author and maintainer: **Data Infra Ekselen**, brand **Diofis Enterprise**, product unit **Babelii**. MIT licensed (see `LICENSE`).
