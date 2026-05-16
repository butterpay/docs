# Telegram Mini App Integration

Accept crypto payments inside Telegram with one shareable link. No bot to build, no Mini App to host. You create an invoice the same way you do for web, and the API hands back a `t.me/...` URL that opens ButterPay's payment page inside Telegram on tap.

## How it works

```
1.  POST /v1/invoices                       (your server → ButterPay API)
    → { id, payUrl, tgPayUrl, … }

2.  Send tgPayUrl to your customer          (your bot, your channel, a DM)
    e.g. "💰 Pay $10  →  https://t.me/ButterPayBot/pay?startapp=inv_abc"

3.  Customer taps the link in Telegram
    → Telegram opens the ButterPay Mini App (no app install, no leaving TG)

4.  Customer connects their wallet, signs   (MetaMask, OKX, Bitget, …
                                              via WalletConnect deep-link)
5.  Payment confirms on Arbitrum

6.  Your webhookUrl gets POSTed payment.confirmed
    (same payload, same signature, same handler you already wrote)
```

The TG path produces the **same on-chain payment** as the web path. There is no new event type, no separate webhook, no second integration. `tgPayUrl` is just an alternative way to drive the same invoice to the same outcome.

## Prerequisites

- A ButterPay merchant account with an API key.
- A `webhookUrl` configured. Your handler already processes `payment.confirmed`; nothing changes for the TG path.

## Create an invoice

The only difference vs. a regular invoice: read `tgPayUrl` from the response.

```bash
curl -X POST https://api.butterpay.io/v1/invoices \
  -H "X-Api-Key: bp_…" \
  -H "Content-Type: application/json" \
  -d '{
    "amountUsd": "10",
    "merchantOrderId": "order_user123_abc",
    "description": "Pro subscription · monthly"
  }'
```

Response:

```json
{
  "id": "inv_z23b38whprf87buud2b0n0",
  "onChainInvoiceId": "0x38e0290d…",
  "amountUsd": "10",
  "status": "PENDING",
  "merchantOrderId": "order_user123_abc",
  "description": "Pro subscription · monthly",
  "createdAt": "2026-05-14T06:11:09.500Z",
  "payUrl": "https://dashboard.butterpay.io/pay/inv_z23b38whprf87buud2b0n0",
  "tgPayUrl": "https://t.me/ButterPayBot/pay?startapp=inv_z23b38whprf87buud2b0n0"
}
```

> `tgPayUrl` is present whenever ButterPay's backend has a Telegram bot configured (production: `@ButterPayBot`). On self-hosted deployments without a bot configured, the field is omitted — your code should treat it as optional.

## Send the link

### Option 1 — Inside your own Telegram bot

If you already run a bot, drop the URL into an inline keyboard's `url` field. Telegram detects the `t.me/<bot>` link and opens the Mini App in-place.

```ts
// Node.js, using grammY or node-telegram-bot-api
await myBot.sendMessage(chatId, "Tap to pay $10:", {
  reply_markup: {
    inline_keyboard: [[
      { text: "💰 Pay $10", url: invoice.tgPayUrl },
    ]],
  },
});
```

### Option 2 — In a Telegram channel, group, or private message

Just paste it as a normal link. Telegram renders the standard preview card.

```
New pre-order open — pay 10 USDC: https://t.me/ButterPayBot/pay?startapp=inv_…
```

### Option 3 — Outside Telegram (email, SMS, your website)

`tgPayUrl` works anywhere a URL works. When opened in Telegram's mobile or desktop client, it launches the Mini App. When opened elsewhere, Telegram's web fallback handles it.

## Handle the result

You already wrote a `payment.confirmed` webhook handler for the web path. No changes needed — the TG payment fires the same event:

```ts
app.post("/webhooks/butterpay", express.raw({ type: "*/*" }), (req, res) => {
  // … signature verification …
  const event = JSON.parse(req.body.toString());
  if (event.event === "payment.confirmed") {
    fulfillOrder(event.data.merchantOrderId);
  }
  res.sendStatus(200);
});
```

## Using the SDK

`@butterpay/core` ≥ 0.4.0 exposes the same field on the typed `Invoice`, plus a helper for callers that need to compose the URL from an id without an API round-trip:

```ts
import { ButterPay, tgPayUrl } from "@butterpay/core";

const bp = new ButterPay({ apiKey: process.env.BUTTERPAY_API_KEY! });

// 1. Read from the create response (server already composed it).
const invoice = await bp.api.invoices.create({ amountUsd: "10" });
console.log(invoice.tgPayUrl);

// 2. Or compose locally from a cached id.
console.log(tgPayUrl("inv_abc123"));
// → https://t.me/ButterPayBot/pay?startapp=inv_abc123

// 3. Override the bot for self-hosted deployments.
console.log(tgPayUrl(invoice, { botUsername: "MyShopBot" }));
```

## What the payer sees

1. Tap the link in Telegram → Telegram opens the Mini App (no install, no leaving TG).
2. Mini App displays amount, token (USDT / USDC), and a TG-native primary button at the bottom.
3. Tap **Connect wallet** → pick MetaMask / OKX / Bitget / others. Telegram deep-links to the wallet app.
4. Approve the spend (one-time per token) in the wallet → return to Telegram.
5. Tap **Pay $X** → confirm the transfer in the wallet → return.
6. Mini App polls for on-chain confirmation, shows ✓, optionally closes itself.

Total time from tap to payment confirmation: typically 30–45 seconds on a known wallet.

## Limitations & notes

- **EVM only for now.** Payment is settled on Arbitrum in USDC or USDT. TON Connect / TON-native payments are not in Phase 1; planned for Phase 3 based on demand.
- **Wallet must be installed.** The user must already have a wallet app (MetaMask, OKX, Bitget, Trust, Rainbow, Coinbase, Rabby). If none, Telegram's `web_app` button still opens the Mini App and the wallet picker shows "Get a wallet" links.
- **`tgUserId` pass-through.** If you want to bind a payment to a specific Telegram user on your side, write their `id` to `metadata.tgUserId` when creating the invoice — it'll appear unchanged in the `payment.confirmed` webhook payload. ButterPay does not currently verify Telegram's `initData` signature; treat the value as a hint your own account system cross-references, not as a proof.
- **iOS vs Android parity.** Both platforms work identically. The wallet roundtrip (Mini App → wallet app → Mini App) relies on universal links; Telegram and the major wallets all support them correctly.

## Self-hosting

If you run your own ButterPay deployment, set two backend env vars:

```bash
TG_BOT_USERNAME=YourBotUsername    # without the leading @
TG_MINI_APP_SHORT_NAME=pay         # the BotFather short name you chose
```

Unset → `tgPayUrl` is omitted from invoice responses, the rest of the API behaves as before.

You'll also need to run the `butterpay-tg-bot` Cloudflare Worker (the bot daemon that handles `/start <inv_id>` and surfaces the Mini App button). See [butterpay/butterpay-tg-bot](https://github.com/butterpay/butterpay-tg-bot) for setup.
