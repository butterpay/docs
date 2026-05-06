# Invoices API

The Invoices API is the core of ButterPay. It covers the full lifecycle of a one-time
USD-denominated payment: create an invoice from your server, share the hosted payment link with
your customer, and let them settle on-chain via the PayRouter contract on Arbitrum. ButterPay is
**non-custodial** — the customer signs `approve` + `pay()` directly to PayRouter, the merchant's
wallet is never involved in the flow, and ButterPay never holds funds.

There are exactly two endpoints in V2. Payment confirmation is delivered to your server via the
`payment.confirmed` webhook configured in the dashboard — see [Webhooks](../webhooks.md).

| Endpoint | Method | Auth | Purpose |
|----------|--------|------|---------|
| [`/v1/invoices`](#createinvoice) | `POST` | Bearer JWT or `X-Api-Key` | Create a new invoice |
| [`/v1/invoices/:id`](#getinvoice) | `GET` | Public | Look up an invoice by DB id or on-chain id |

---

## CreateInvoice

Create a new USD-denominated invoice. The returned `payUrl` is the hosted payment page; share it
with your customer (redirect, email, QR, etc.). The customer picks any whitelisted token at pay
time, signs `approve` + `pay()` to PayRouter, and the backend marks the invoice `PAID` once the
on-chain `PaymentProcessed` event is observed.

- **Method**: `POST`
- **Path**: `/v1/invoices`
- **Auth**: `Authorization: Bearer <jwt>` **or** `X-Api-Key: <key>`
- **Description**: Creates an invoice scoped to the authenticated merchant and computes the
  on-chain `invoiceId` (a `bytes32` keccak256 hash of `merchantUuid`, the order ID bytes, and a
  fresh 32-byte nonce). The invoice expires 30 minutes after creation if no payment is observed
  on-chain (see [Status lifecycle](#status-lifecycle)).

### Request body

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `amountUsd` | string | Yes | Decimal USD amount, e.g. `"10"` or `"10.00"`. Must parse as a positive number. |
| `merchantOrderId` | string | No | Your own order ID. Stored on the invoice for reconciliation and echoed in webhook payloads. If omitted, an `auto-<timestamp>` placeholder is used internally. |
| `tokenAddress` | string | No | Preferred ERC-20 token address (or the native sentinel `0xeeee…eeee`) shown as the default selection on the payment page. The customer can change it. |
| `referrerAddress` | string | No | EVM address that should receive a referrer cut of this payment. Recorded on-chain and paid out at settlement time. |
| `referrerFeeBps` | number | No | Referrer fee in basis points (1 bps = 0.01%). Required when `referrerAddress` is set. |
| `description` | string | No | Human-readable description shown to the payer on the hosted page, e.g. `"Order #1234"`. |

### Response (HTTP 201)

```json
{
  "id": "inv_dudkhayh6ntw7ou7uz97mhfn",
  "onChainInvoiceId": "0x38e0290dffbdce828672b58cf0ec1f34d06b82581bca186ac73e8f4c65437333",
  "amountUsd": "10",
  "status": "PENDING",
  "merchantOrderId": "order_001",
  "description": "Order #1234",
  "createdAt": "2026-05-05T06:11:09.500Z",
  "payUrl": "https://pay.butterpay.io/pay/inv_dudkhayh6ntw7ou7uz97mhfn"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | ButterPay DB id (`inv_…`). Use this for `GET /v1/invoices/:id` and as the path segment in `payUrl`. |
| `onChainInvoiceId` | string | `bytes32` keccak256 hash used by the PayRouter contract. Emitted in the `PaymentProcessed` event and accepted as a lookup key on `GET /v1/invoices/:id`. |
| `amountUsd` | string | Echoes the requested amount. |
| `status` | string | Always `"PENDING"` for newly created invoices. |
| `merchantOrderId` | string \| null | Echoes the value sent in the request, or `null` if you did not supply one. |
| `description` | string \| null | Echoes the value sent in the request. |
| `createdAt` | string | ISO 8601 timestamp. |
| `payUrl` | string | Hosted payment page URL. Send the customer here. |

### Example

```bash
curl -X POST https://api.butterpay.io/v1/invoices \
  -H "X-Api-Key: bp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx..." \
  -H "Content-Type: application/json" \
  -d '{
    "amountUsd": "10",
    "merchantOrderId": "order_001",
    "description": "Order #1234"
  }'
```

Response:

```json
{
  "id": "inv_dudkhayh6ntw7ou7uz97mhfn",
  "onChainInvoiceId": "0x38e0290dffbdce828672b58cf0ec1f34d06b82581bca186ac73e8f4c65437333",
  "amountUsd": "10",
  "status": "PENDING",
  "merchantOrderId": "order_001",
  "description": "Order #1234",
  "createdAt": "2026-05-05T06:11:09.500Z",
  "payUrl": "https://pay.butterpay.io/pay/inv_dudkhayh6ntw7ou7uz97mhfn"
}
```

After creation, redirect the customer to `payUrl` (or render it as a button / QR code). Listen for
the `payment.confirmed` webhook to fulfil the order on your side.

### Error responses

| Status | Body | Meaning |
|--------|------|---------|
| 400 | `{ "error": "amountUsd must be positive" }` | Body is missing `amountUsd` or it parses to `<= 0`. |
| 400 | `{ "error": "<message>" }` | Validation failure inside `createInvoice` (e.g. malformed referrer fee). |
| 401 | `{ "error": "Unauthorized" }` | Missing or invalid `Authorization` / `X-Api-Key`. |

---

## GetInvoice

Look up a single invoice. This endpoint is **public**: no API key or JWT is required. The hosted
payment page calls this to render invoice details and discover where funds are going.

- **Method**: `GET`
- **Path**: `/v1/invoices/:id`
- **Auth**: Public (no authentication)
- **Description**: Returns the raw `invoices` row, plus `merchantName` and
  `merchantReceivingAddresses` joined from the merchant record. The `:id` parameter accepts
  **either** a ButterPay DB id (`inv_…`) **or** the on-chain `bytes32` `onChainInvoiceId`
  (`0x` + 64 hex chars).

### Path parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | string | Yes | Either the DB id (`inv_dudkhayh6ntw7ou7uz97mhfn`) or the on-chain id (`0x38e0290d…`). |

### Response (HTTP 200)

```json
{
  "id": "inv_dudkhayh6ntw7ou7uz97mhfn",
  "merchantId": "mer_01hwzq3k5n8ej4v2b7r9abc123",
  "merchantOrderId": "order_001",
  "amount": "10",
  "token": "USD",
  "chain": "pending",
  "status": "PENDING",
  "onChainInvoiceId": "0x38e0290dffbdce828672b58cf0ec1f34d06b82581bca186ac73e8f4c65437333",
  "description": "Order #1234",
  "payerAddress": null,
  "txHash": null,
  "paymentMethod": null,
  "merchantNet": null,
  "serviceFee": null,
  "referrerAddress": null,
  "referrerFee": null,
  "referrerFeeBps": null,
  "confirmedAt": null,
  "createdAt": "2026-05-05T06:11:09.500Z",
  "updatedAt": "2026-05-05T06:11:09.500Z",
  "merchantName": "Acme Store",
  "merchantReceivingAddresses": {
    "arbitrum": "0xMerchantReceivingAddress"
  }
}
```

Field notes:

| Field | Description |
|-------|-------------|
| `token` | `"USD"` until the invoice is paid. After settlement this is the ERC-20 contract address (or native sentinel) the customer paid with. |
| `chain` | `"pending"` until paid. (Settlement always happens on Arbitrum in V2.) |
| `paymentMethod` | `null` until paid; afterwards one of `"NATIVE"`, `"APPROVE"`, or `"SWAP"`. |
| `merchantNet`, `serviceFee`, `referrerFee` | Settlement amounts in the **paid token's smallest unit** (e.g. USDC 6-decimals). Populated when status becomes `PAID`. |
| `txHash` | On-chain transaction hash of the `pay()` call, populated on settlement. |
| `merchantName` | Synthesised — joined from the merchant record, not stored on the invoice. |
| `merchantReceivingAddresses` | Synthesised — `{ "arbitrum": "0x…" }` derived from the merchant's V2 `receiverAddress` + `chainId`. The hosted pay page reads this to build the on-chain transaction. |

### Example — lookup by DB id

```bash
curl https://api.butterpay.io/v1/invoices/inv_dudkhayh6ntw7ou7uz97mhfn
```

### Example — lookup by on-chain id

Useful when you only have the `bytes32` from a `PaymentProcessed` event log:

```bash
curl https://api.butterpay.io/v1/invoices/0x38e0290dffbdce828672b58cf0ec1f34d06b82581bca186ac73e8f4c65437333
```

### Error responses

| Status | Body | Meaning |
|--------|------|---------|
| 404 | `{ "error": "invoice not found" }` | No invoice matches the supplied `id` (neither as DB id nor as on-chain id). |

---

## Status lifecycle

```
                  customer pays on-chain
   PENDING ───────────────────────────────────▶ PAID
      │
      │ no payment within 30 minutes
      ▼
   EXPIRED
```

| Status | Meaning |
|--------|---------|
| `PENDING` | Invoice has been created. The hosted page is live and the on-chain `invoiceId` is registered with PayRouter. |
| `PAID` | The backend listener observed a matching `PaymentProcessed` (or `SwapPaymentProcessed`) event on Arbitrum and has populated `txHash`, `payerAddress`, `merchantNet`, `serviceFee`, and (if applicable) `referrerFee`. This triggers the `payment.confirmed` webhook. |
| `EXPIRED` | A background job (`expireStalePending`, runs every 5 minutes) marked the invoice expired because it stayed `PENDING` for more than 30 minutes. |

There is no `FAILED` status on invoices. A failed on-chain attempt simply leaves the invoice in
`PENDING` until either a successful payment lands or the expiry job moves it to `EXPIRED`. (The
`subscription.expired` webhook is emitted from the recurring-billing path, not from invoices —
see [Plans & Subscriptions](plans-subscriptions.md).)

There is also no `REFUNDED` status: refunds are not modelled by the backend in V2. If you need to
return funds to a customer, do so off-platform (a direct on-chain transfer from your receiving
wallet).

## Operations not supported in V2

To save you from looking: the following operations existed in V1 and are **gone** in V2.

- No `PATCH` / `PUT` on invoices — invoices are immutable after creation.
- No `DELETE` — there is no way to cancel an invoice. Stop sharing the `payUrl` and let it expire.
- No refund endpoint — handle refunds off-chain from your receiving wallet.
- No session-token / `submitTransaction` flow — the customer signs `pay()` directly to PayRouter,
  so the backend never needs to be told about the tx hash. The on-chain listener picks it up.
- No `/v1/transactions`, `/v1/transactions/summary`, `/v1/transactions/export`, or `/v1/balances`
  endpoints on the public API. Those reports live in the dashboard.

## Related

- [Webhooks](../webhooks.md) — `payment.confirmed` payload, HMAC signature verification, retry schedule. Configure your receiver URL in the dashboard.
- [Authentication](../authentication.md) — issuing API keys and Bearer JWTs.
- [Plans & Subscriptions API](plans-subscriptions.md) — recurring billing via EIP-2612 permits.
