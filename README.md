# ButterPay Documentation

Non-custodial crypto payment infrastructure for stablecoin settlement. Customers pay any token; merchants receive USDT/USDC. 0.8% fee. Stripe-compatible webhooks.

This documentation covers the public HTTP API and the official `@butterpay/core` SDK.

## What you'll find here

- **[Overview](overview.md)** — what ButterPay is, supported chains, supported tokens, fees
- **[Getting Started](getting-started.md)** — 10-minute end-to-end integration
- **[Authentication](authentication.md)** — Bearer JWT, X-Api-Key, sessionToken
- **[Webhooks](webhooks.md)** — events, signing, retry policy, security
- **API Reference** — [Invoices](api/invoices.md), [Merchants](api/merchants.md), [Plans & Subscriptions](api/plans-subscriptions.md), [Webhooks Management](api/webhooks.md)
- **SDK** — [@butterpay/core](sdk/core.md)

## Status

- Arbitrum mainnet **live**
- Ethereum / BSC / Polygon **coming soon** (audit pending)
- API: `https://api.butterpay.io`
- Dashboard + hosted pages: `https://dashboard.butterpay.io` (e.g. `/pay/<invoiceId>`, `/subscribe/<planId>`)
