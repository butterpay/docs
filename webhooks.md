# Webhooks

ButterPay sends webhooks as HTTP POST requests to your registered endpoint whenever a relevant
on-chain event is observed and confirmed. Each delivery includes a signed payload so you can
verify it originated from ButterPay before acting on it.

---

## Overview

Register a webhook URL in the dashboard (Settings → Webhook URL) or by calling
`PATCH /v1/merchants/me`. When a relevant event fires, the backend serializes a JSON payload,
signs it with your webhook secret, and POSTs it to that URL.

**Headers sent on every delivery**

| Header | Value |
|--------|-------|
| `Content-Type` | `application/json` |
| `X-ButterPay-Signature` | `t=<unix_timestamp>,v1=<hex_hmac>` |
| `X-ButterPay-Event` | Event name, e.g. `payment.confirmed` |

ButterPay considers a delivery successful when your endpoint returns any `2xx` status code within
10 seconds. Non-`2xx` responses and connection errors both trigger the retry schedule described
in [Retry policy](#retry-policy).

**Idempotency**: network conditions may cause the same event to be delivered more than once,
particularly when retries fire close together. Use the webhook log `id` field (available via
`GET /v1/webhooks/logs`) as a deduplication key in your handler. Processing the same `id` twice
should be a no-op.

---

## Event types

All events listed below are emitted only after the corresponding on-chain event has reached the
configured confirmation depth. ButterPay does not deliver speculative or unconfirmed events.

| Event | When it fires |
|-------|---------------|
| `merchant.registered` | A `MerchantRegistered` event was observed for your merchant on-chain |
| `payment.confirmed` | A `PaymentProcessed` event for a one-time invoice was observed and verified |
| `payment.expired` | An invoice's TTL (30 minutes) elapsed before any payment was detected |
| `plan.created` | A `PlanCreated` event was observed for one of your plans |
| `plan.deactivated` | A `PlanActiveChanged(false)` event was observed for one of your plans |
| `subscription.created` | A `Subscribed` event was observed (new subscriber) |
| `subscription.charged` | A `Charged` event was observed for a billing cycle |
| `subscription.cancelled` | A `Cancelled` event was observed (note the double-L spelling) |
| `subscription.expired` | An `ExpiredByFailure` event was observed (subscriber's allowance or balance ran out and the on-chain subscription terminated) |
| `subscription.resubscribed` | A `Resubscribed` event was observed (a previously cancelled or expired subscriber re-subscribed) |

> ButterPay does not currently fire a per-cycle `subscription.charge_failed` webhook. When a
> subscriber's allowance or balance runs out, the on-chain contract eventually emits
> `ExpiredByFailure`, which ButterPay delivers as `subscription.expired`. Use that event to
> detect involuntary churn.

> The test endpoint (`POST /v1/webhooks/test`) lets you send any of the events above to your
> webhook URL with synthetic data. See [Testing webhooks](#testing-webhooks).

---

## Payload shape

Every event uses the same envelope:

```json
{
  "event": "<event_name>",
  "data": { ... }
}
```

The shape of `data` is determined by the `event` field. The full table of `data` fields per
event type is below; an example payload for each event is shown in
[Example payloads](#example-payloads).

| Event | `data` fields |
|-------|---------------|
| `merchant.registered` | `merchantId`, `onChainMerchantId`, `txHash`, `chainId` |
| `payment.confirmed` | `invoiceId`, `txHash`, `amountPaid`, `merchantNet` |
| `payment.expired` | `invoiceId` |
| `plan.created` | `planId`, `externalPlanCode`, `chainId`, `txHash` |
| `plan.deactivated` | `planId`, `chainId` |
| `subscription.created` | `subscriptionId`, `planId`, `subscriber`, `anchorTime`, `anchorDay` |
| `subscription.charged` | `subscriptionId`, `cyclesCharged`, `amount`, `merchantNet`, `txHash` |
| `subscription.cancelled` | `subscriptionId`, `cancelledBy` (`"user"` \| `"merchant"` \| `"plan_closed"`) |
| `subscription.expired` | `subscriptionId`, `lastCyclesCharged`, `reason` |
| `subscription.resubscribed` | `subscriptionId`, `newAnchorTime` |

**Common field semantics**

| Field | Type | Description |
|-------|------|-------------|
| `event` | string | Event name (see table above) |
| `data` | object | Event-specific payload |
| `merchantId` | string | Internal ButterPay merchant ID (`mer_…`) |
| `onChainMerchantId` | string | The `merchantId` value emitted by the contract (decimal string) |
| `chainId` | number | EVM chain ID, e.g. `42161` for Arbitrum One |
| `txHash` | string | On-chain transaction hash that produced the event |
| `invoiceId` | string | One-time invoice ID — internal UUID or on-chain `bytes32` depending on context |
| `amountPaid` | string | Token amount transferred by the payer (smallest unit, e.g. USDC's 6-decimal base units) |
| `merchantNet` | string | Token amount delivered to the merchant after fees (smallest unit) |
| `planId` | string | Plan identifier — for plan and subscription events this is the on-chain `bytes32` plan ID |
| `externalPlanCode` | string | Merchant-supplied plan code (may be empty if the plan was created on-chain only) |
| `subscriptionId` | string | Subscription identifier — internal UUID or on-chain subscription ID |
| `subscriber` | string | Subscriber wallet address (`0x…`) |
| `anchorTime` | string | ISO 8601 timestamp of the subscription anchor point |
| `anchorDay` | number | Day-of-month (1-28) used for monthly anchoring |
| `cyclesCharged` | number | Total number of cycles charged for this subscription so far |
| `amount` | string | Token amount charged this cycle (smallest unit) |
| `cancelledBy` | string | `"user"`, `"merchant"`, or `"plan_closed"` |
| `lastCyclesCharged` | number | Final cycle count at the moment the subscription expired |
| `reason` | string | Raw `bytes32` reason from the contract event (hex-encoded), or a known label |
| `newAnchorTime` | string | ISO 8601 timestamp of the new anchor after a re-subscribe |

> Token amounts are reported in the token's smallest base unit (e.g. for USDC on Arbitrum,
> `"10000000"` represents `10.000000` USDC). Convert using the token's `decimals` value.

---

## Example payloads

### `merchant.registered`

```json
{
  "event": "merchant.registered",
  "data": {
    "merchantId": "mer_01hwz4m8y3g9c5d7f8h0j2kn",
    "onChainMerchantId": "42",
    "txHash": "0xabc123def456...",
    "chainId": 42161
  }
}
```

### `payment.confirmed`

```json
{
  "event": "payment.confirmed",
  "data": {
    "invoiceId": "inv_01hwz4m8y3g9c5d7f8h0j2kn",
    "txHash": "0xabc123def456...",
    "amountPaid": "49990000",
    "merchantNet": "49590200"
  }
}
```

### `payment.expired`

```json
{
  "event": "payment.expired",
  "data": {
    "invoiceId": "inv_01hwz4m8y3g9c5d7f8h0j2kn"
  }
}
```

### `plan.created`

```json
{
  "event": "plan.created",
  "data": {
    "planId": "0x1111111111111111111111111111111111111111111111111111111111111111",
    "externalPlanCode": "pro-monthly",
    "chainId": 42161,
    "txHash": "0xcd34cd34cd34cd34cd34cd34cd34cd34cd34cd34cd34cd34cd34cd34cd34cd34"
  }
}
```

### `plan.deactivated`

```json
{
  "event": "plan.deactivated",
  "data": {
    "planId": "0x1111111111111111111111111111111111111111111111111111111111111111",
    "chainId": 42161
  }
}
```

### `subscription.created`

```json
{
  "event": "subscription.created",
  "data": {
    "subscriptionId": "sub_01hwz9p4q5r6s7t8u9v0w1xy",
    "planId": "0x1111111111111111111111111111111111111111111111111111111111111111",
    "subscriber": "0x2222222222222222222222222222222222222222",
    "anchorTime": "2026-05-01T00:00:00.000Z",
    "anchorDay": 1
  }
}
```

### `subscription.charged`

```json
{
  "event": "subscription.charged",
  "data": {
    "subscriptionId": "sub_01hwz9p4q5r6s7t8u9v0w1xy",
    "cyclesCharged": 3,
    "amount": "19990000",
    "merchantNet": "19830080",
    "txHash": "0xefefefefefefefefefefefefefefefefefefefefefefefefefefefefefefefef"
  }
}
```

### `subscription.cancelled`

```json
{
  "event": "subscription.cancelled",
  "data": {
    "subscriptionId": "sub_01hwz9p4q5r6s7t8u9v0w1xy",
    "cancelledBy": "user"
  }
}
```

### `subscription.expired`

```json
{
  "event": "subscription.expired",
  "data": {
    "subscriptionId": "sub_01hwz9p4q5r6s7t8u9v0w1xy",
    "lastCyclesCharged": 4,
    "reason": "0x0000000000000000000000000000000000000000000000000000000000000000"
  }
}
```

`subscription.expired` is fired when the on-chain contract emits `ExpiredByFailure` — typically
because the subscriber's allowance or balance ran out and a scheduled charge could not settle.
Treat this as terminal involuntary churn for the subscription.

### `subscription.resubscribed`

```json
{
  "event": "subscription.resubscribed",
  "data": {
    "subscriptionId": "sub_01hwz9p4q5r6s7t8u9v0w1xy",
    "newAnchorTime": "2026-06-01T00:00:00.000Z"
  }
}
```

---

## Error codes

Most ButterPay webhook events do not carry an error code. Event delivery on V2 follows the
on-chain state machine: a payment either reaches `payment.confirmed` (the `PaymentProcessed`
event was observed) or `payment.expired` (the invoice TTL elapsed without a transaction).
Reverted on-chain transactions produce no event and therefore no webhook — your integration
should rely on the absence of `payment.confirmed` plus the eventual `payment.expired`, not on a
failure event.

Similarly for subscriptions: an individual cycle that fails to settle does not produce a
webhook. ButterPay only delivers `subscription.expired` once the on-chain contract emits
`ExpiredByFailure`, which terminates the subscription. The `data.reason` field on
`subscription.expired` carries the raw `bytes32` reason from the contract event; treat
unrecognized values as opaque.

If you observe stuck or unexpected behavior, query `GET /v1/webhooks/logs` and `GET /v1/invoices`
or `GET /v1/subscriptions` to inspect the underlying state, and contact support with the
relevant `txHash`.

---

## Signature verification

Every delivery includes an `X-ButterPay-Signature` header in the format:

```
X-ButterPay-Signature: t=1745763121,v1=a3f2c1d9...
```

This follows the same convention as Stripe's webhook signatures. If you already have Stripe
webhook verification logic, the algorithm is identical.

**Algorithm**

1. Split the header value on `,` to extract `t` (Unix timestamp, seconds) and `v1` (hex HMAC).
2. Construct the signed content: `<t>.<raw_request_body>` — concatenate the timestamp string,
   a literal `.`, and the **raw, unmodified request body bytes**.
3. Compute `HMAC-SHA256(webhook_secret, signed_content)` and encode it as lowercase hex.
4. Compare your computed HMAC with the `v1` value using a constant-time comparison function.
5. Reject the request if `Math.abs(Date.now() / 1000 - t) > 300` (5-minute replay window).

Your webhook secret is available at `GET /v1/webhooks/secret` (requires `X-Api-Key`).

> **Critical**: compute the HMAC over the raw request body bytes, not a re-serialized JSON
> object. Any whitespace or key-ordering difference will produce a different HMAC and cause
> verification to fail.

### Node.js / Express

```js
const crypto = require("crypto");

app.post(
  "/webhooks/butterpay",
  express.raw({ type: "application/json" }), // preserve raw bytes
  (req, res) => {
    const webhookSecret = process.env.BUTTERPAY_WEBHOOK_SECRET;
    const sigHeader = req.headers["x-butterpay-signature"];

    if (!sigHeader) {
      return res.status(400).send("Missing signature header");
    }

    // Parse t=... and v1=...
    const parts = Object.fromEntries(
      sigHeader.split(",").map((p) => p.split("="))
    );
    const timestamp = parts["t"];
    const receivedHmac = parts["v1"];

    if (!timestamp || !receivedHmac) {
      return res.status(400).send("Malformed signature header");
    }

    // Replay-window check (5 minutes)
    const nowSecs = Math.floor(Date.now() / 1000);
    if (Math.abs(nowSecs - parseInt(timestamp, 10)) > 300) {
      return res.status(400).send("Timestamp outside replay window");
    }

    // Compute expected HMAC over "<timestamp>.<rawBody>"
    const signedContent = `${timestamp}.${req.body}`;
    const expectedHmac = crypto
      .createHmac("sha256", webhookSecret)
      .update(signedContent)
      .digest("hex");

    // Constant-time comparison
    const expected = Buffer.from(expectedHmac, "hex");
    const received = Buffer.from(receivedHmac, "hex");

    if (
      expected.length !== received.length ||
      !crypto.timingSafeEqual(expected, received)
    ) {
      return res.status(401).send("Signature mismatch");
    }

    const event = JSON.parse(req.body);
    // Handle event ...
    return res.sendStatus(200);
  }
);
```

### Python / Flask

```py
import hashlib
import hmac
import json
import os
import time

from flask import Flask, abort, request

app = Flask(__name__)

@app.route("/webhooks/butterpay", methods=["POST"])
def butterpay_webhook():
    webhook_secret = os.environ["BUTTERPAY_WEBHOOK_SECRET"].encode()
    sig_header = request.headers.get("X-ButterPay-Signature", "")

    if not sig_header:
        abort(400, "Missing signature header")

    # Parse t=... and v1=...
    parts = dict(p.split("=", 1) for p in sig_header.split(","))
    timestamp = parts.get("t")
    received_hmac = parts.get("v1")

    if not timestamp or not received_hmac:
        abort(400, "Malformed signature header")

    # Replay-window check (5 minutes)
    if abs(time.time() - int(timestamp)) > 300:
        abort(400, "Timestamp outside replay window")

    # Compute expected HMAC over "<timestamp>.<rawBody>"
    raw_body = request.get_data()  # raw bytes — do NOT call request.json first
    signed_content = f"{timestamp}.".encode() + raw_body
    expected_hmac = hmac.new(webhook_secret, signed_content, hashlib.sha256).hexdigest()

    # Constant-time comparison
    if not hmac.compare_digest(expected_hmac, received_hmac):
        abort(401, "Signature mismatch")

    event = json.loads(raw_body)
    # Handle event ...
    return "", 200
```

---

## Retry policy

If an initial delivery fails, the backend schedules up to three retries at the following
intervals after each failed attempt:

| Attempt | Delay after previous failure |
|---------|------------------------------|
| 1st retry | 10 seconds |
| 2nd retry | 60 seconds |
| 3rd retry | 300 seconds (5 minutes) |

After the third retry fails, the delivery is marked `failed` and no further attempts are made.
The `nextRetryAt` field in the webhook log is set to `null` once all retries are exhausted.

The URL is re-validated immediately before every retry attempt (including the original send).
If the URL no longer passes validation at retry time — for example because DNS now resolves to
a private address — the attempt is skipped without counting as a failure.

Retrieve delivery history and retry status with:

```bash
curl https://api.butterpay.io/v1/webhooks/logs \
  -H "X-Api-Key: bp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

The response includes `attempts`, `statusCode`, `success`, `nextRetryAt`, and the raw
`response` body returned by your endpoint (truncated to 500 characters).

---

## URL requirements

ButterPay validates your webhook URL both at registration time and immediately before each
send (including retries). The dual-validation approach narrows the DNS rebind window that
would otherwise exist between a passing registration check and the actual HTTP call.

**Protocol**

- In production (`NODE_ENV=production`): HTTPS is required. HTTP URLs are rejected.
- In development: HTTP is permitted.

**Blocked IP ranges**

The backend resolves the hostname with a DNS lookup and rejects the URL if the resolved address
falls in any of the following ranges:

| Range | Classification |
|-------|---------------|
| `10.0.0.0/8` | RFC 1918 private |
| `172.16.0.0/12` | RFC 1918 private |
| `192.168.0.0/16` | RFC 1918 private |
| `127.0.0.0/8` | Loopback |
| `169.254.0.0/16` | Link-local / AWS IMDS |
| `100.64.0.0/10` | CGNAT (RFC 6598) |
| `224.0.0.0/4` | Multicast |
| `240.0.0.0/4` | Reserved |
| `0.0.0.0/8` | "This network" |
| `::1` | IPv6 loopback |
| `fc00::/7` | IPv6 ULA (covers `fc00::` and `fd00::`) |
| `fe80::/10` | IPv6 link-local |
| `ff00::/8` | IPv6 multicast |
| `::ffff:0:0/96` | IPv4-mapped IPv6 (e.g. `::ffff:127.0.0.1`) |

Any URL whose hostname resolves to an address in these ranges is rejected with a `400` error
at registration time. At send time, the delivery is silently skipped with a warning logged.

---

## Idempotency

The retry schedule means the same event payload may arrive at your endpoint more than once.
Design your handler to be idempotent.

The recommended deduplication key is the webhook log `id` returned by `GET /v1/webhooks/logs`.
Each initial delivery attempt creates one log record; retries update that same record rather
than creating new ones. Storing the log `id` and skipping processing when you see a repeated
value is sufficient for most use cases.

For payment events, the combination of `invoiceId` + `event` is also unique per invoice
lifecycle (an invoice can only be `confirmed` or `expired` once), and can serve as a simpler
deduplication strategy if you do not wish to query the logs endpoint.

For subscription events, `subscriptionId` + `event` + `cyclesCharged` (where applicable) is a
stable per-cycle key.

---

## Testing webhooks

`POST /v1/webhooks/test` sends a synthetic payload to your currently configured webhook URL.
The request is authenticated with your API key and the payload is signed with your webhook
secret, so your verification logic runs exactly as it would for a real event.

You may select any V2 event type listed in [Event types](#event-types). The test endpoint
populates the `data` object with placeholder values that match the production schema.

```bash
curl -X POST https://api.butterpay.io/v1/webhooks/test \
  -H "X-Api-Key: bp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" \
  -H "Content-Type: application/json" \
  -d '{"event":"payment.confirmed"}'
```

**Response**

```json
{
  "success": true,
  "statusCode": 200,
  "url": "https://your-server.example.com/webhooks/butterpay",
  "response": "OK"
}
```

If your URL is not configured or fails validation, the endpoint returns a `400` before any
delivery is attempted:

```json
{
  "error": "No webhook URL configured. Set one in Settings first."
}
```

Test deliveries also include an `X-ButterPay-Test: true` header so your handler can short-circuit
business logic when receiving them in production. The synthetic payloads use placeholder IDs
(e.g. `inv_test_…`, `sub_test_…`) and a deterministic `txHash` of repeating bytes, which are
safe to receive as long as your handler checks for the test header (or the placeholder IDs)
before performing real-world side effects.

---

## Troubleshooting

**Signature mismatch**

The most common cause is parsing the JSON body before computing the HMAC. Any JSON library may
produce different whitespace or key ordering than the original payload. Always pass the raw
request body bytes to the HMAC function. In Express, use `express.raw({ type: "application/json" })`
on your route instead of `express.json()`. In Flask, call `request.get_data()` before calling
`request.json`.

**401 Unauthorized on webhook logs or test endpoint**

Your API key may have been rotated or revoked. Retrieve a new key from the dashboard
(Settings → API Key → Rotate) and update your server's environment variables.

**URL rejected at registration**

Registration fails with `400` if:

- The URL uses HTTP in production (HTTPS is required).
- The hostname resolves to a private, loopback, link-local, or CGNAT address.
- The hostname cannot be resolved at all.

Use a public HTTPS endpoint. For local development, tools such as
[ngrok](https://ngrok.com) or [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
expose a local server under a public HTTPS URL.

**Expected `payment.failed` or `subscription.charge_failed` event never arrives**

V2 does not deliver per-attempt failure webhooks. Reverted on-chain transactions emit no event
and produce no webhook; for invoices, the absence of `payment.confirmed` followed by
`payment.expired` indicates the payment did not settle. For subscriptions, repeated cycle
failures eventually trigger `ExpiredByFailure` on-chain, which is delivered as
`subscription.expired`. If you need per-attempt visibility, poll `GET /v1/invoices` or
`GET /v1/subscriptions`.

**Retries exhausted — delivery marked failed**

Check `GET /v1/webhooks/logs` to see the `response` field from your server and the final
`statusCode`. Common causes:

- Your server returned a non-`2xx` status.
- Your server took longer than 10 seconds to respond (the per-request timeout).
- A TLS certificate error prevented the connection.

Fix the underlying issue, then handle any missed events by replaying the payloads from the
logs response or by querying the affected invoices via `GET /v1/invoices` or subscriptions via
`GET /v1/subscriptions`.
