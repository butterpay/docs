# @butterpay/core

Low-level TypeScript SDK for the ButterPay V2 API: a typed REST client, intent
helpers for non-custodial on-chain writes, wallet adapters, and balance scanning.

The V2 contract model is **non-custodial**. The merchant's wallet — not the
ButterPay backend — signs the on-chain calls that register merchants, create
plans, and cancel subscriptions. Endpoints that trigger a chain write return an
`encodedTx` (ABI-encoded calldata) plus a destination `to` and `chainId` for the
merchant wallet to broadcast.

This page documents `@butterpay/core@0.3.0`.

---

## Install

```bash
npm install @butterpay/core viem
```

`viem ^2.47.10` is a peer-level dependency used by the wallet adapters and is
recommended for executing intents on the merchant side.

---

## Quick start

```ts
import { ApiClient } from "@butterpay/core";

const api = new ApiClient({
  baseUrl: "https://api.butterpay.io",
  apiKey: process.env.BUTTERPAY_API_KEY,
});

const { id, payUrl } = await api.invoices.create({
  amountUsd: "19.99",
  merchantOrderId: "order-123",
});

console.log(`Send your customer to: ${payUrl}`);
```

`ApiClient` is the only entrypoint you need for invoice issuance and read-side
queries. Chain-write operations (registration, plan creation, cancel) return
intents that you sign with a wallet — see [Intent flow](#intent-flow).

---

## API surface

`new ApiClient(config)` exposes four namespaces, mirroring the V2 backend route
files.

| Namespace | Purpose |
|---|---|
| `api.merchants` | Build a registration intent for the merchant wallet to sign. |
| `api.plans` | Create / list / fetch subscription plans. Create returns an intent. |
| `api.subscriptions` | List / fetch subscriptions, build merchant-side cancel intents. |
| `api.invoices` | Create one-time invoices and fetch their status. |

### Constructor

```ts
import { ApiClient, type ApiClientConfig } from "@butterpay/core";

const api = new ApiClient({
  baseUrl: "https://api.butterpay.io", // required
  apiKey: "bp_xxxxxxxxxxxxxxxxxxxxxxxx", // optional; required for write & list endpoints
});
```

The client sends `X-API-Key: <apiKey>` on every request when set. Errors
surface as thrown `Error` instances carrying the server-provided message. The
`api.invoices.get()` and `api.plans.get()` endpoints are public and do not
require an API key.

---

## `api.merchants`

### `createRegistrationIntent(params)`

Build a `registerMerchant()` intent. The merchant's wallet signs and broadcasts
`encodedTx` against `SubscriptionManager`. The chain listener observes
`MerchantRegistered` and fires the `merchant.registered` webhook.

```ts
async createRegistrationIntent(params: {
  receiverAddress: string;
  metadata: Record<string, unknown>;
  chainId: number;
}): Promise<MerchantRegistrationIntent>
```

Returns `MerchantRegistrationIntent`:

```ts
interface MerchantRegistrationIntent {
  metadataHash: `0x${string}`; // keccak256 of canonical metadata JSON
  metadataJson: string;        // canonical JSON string stored off-chain
  encodedTx: `0x${string}`;    // calldata for SubscriptionManager.registerMerchant(...)
}
```

Example:

```ts
const intent = await api.merchants.createRegistrationIntent({
  receiverAddress: "0xMerchantReceiver...",
  metadata: { name: "Acme Inc.", website: "https://acme.example" },
  chainId: 42161,
});

// Hand `intent.encodedTx` to the merchant wallet — see Intent flow below.
```

---

## `api.plans`

### `create(params)`

Build a `createPlan()` intent. `externalPlanCode` is the merchant-chosen
identifier (`^[a-z0-9_-]{3,64}$`) and is unique per merchant; it becomes the
salt (keccak256 of the code). `amount` is in the smallest unit of
`tokenAddress` — for USDC (6 decimals), `"10000000"` is 10 USDC.

```ts
async create(params: {
  externalPlanCode: string;
  tokenAddress: string;
  amount: string;
  monthsPerCycle: number;
  bufferTimeSeconds: number;
  metadata?: Record<string, unknown>;
  chainId: number;
}): Promise<PlanIntent>
```

Returns `PlanIntent`:

```ts
interface PlanIntent {
  planId: `0x${string}`;       // pre-computed bytes32 on-chain plan id
  salt: `0x${string}`;         // keccak256(externalPlanCode)
  metadataHash: `0x${string}`; // keccak256 of canonical metadata JSON
  encodedTx: `0x${string}`;    // calldata for SubscriptionManager.createPlan(...)
}
```

After signing, the `plan.created` webhook fires once the listener observes
`PlanCreated` and flips `active=true`.

Example:

```ts
const intent = await api.plans.create({
  externalPlanCode: "premium-monthly",
  tokenAddress: "0xaf88d065e77c8cC2239327C5EDb3A432268e5831", // USDC on Arbitrum
  amount: "9990000", // 9.99 USDC (6 decimals)
  monthsPerCycle: 1,
  bufferTimeSeconds: 86_400,
  chainId: 42161,
});
```

### `list()` and `get(idOrOnChainId)`

```ts
async list(): Promise<{ data: Plan[]; count: number }>
async get(idOrOnChainId: string): Promise<Plan>
```

`list()` returns plans owned by the authenticated merchant. `get()` accepts
either the DB id (`pln_…`) or the on-chain plan id (bytes32 hex) and is a
public endpoint — no API key required.

```ts
const { data: plans, count } = await api.plans.list();
const plan = await api.plans.get("pln_01HXY…");
```

---

## `api.subscriptions`

Subscription **creation** is non-custodial and happens entirely on-chain — the
subscriber's wallet calls `SubscriptionManager.subscribe(planId)` directly.
The SDK only reads subscriptions and helps merchants cancel.

### `list(params?)` and `get(id)`

```ts
async list(params?: { status?: string }): Promise<{ data: Subscription[]; count: number }>
async get(id: string): Promise<Subscription>
```

```ts
const { data: active } = await api.subscriptions.list({ status: "active" });
const sub = await api.subscriptions.get("sub_01HXY…");
```

### `cancelIntent(id)`

Build a `merchantCancel()` intent. The merchant wallet signs and broadcasts
`encodedTx`; the listener picks up the `Cancelled` event, flips DB status, and
delivers a `subscription.cancelled` webhook with `cancelledBy: "merchant"`.

```ts
async cancelIntent(id: string): Promise<CancelIntent>
```

Returns `CancelIntent`:

```ts
interface CancelIntent {
  encodedTx: `0x${string}`; // calldata for SubscriptionManager.merchantCancel(onChainSubId)
  onChainSubId: string;     // uint256 on-chain subscription id (string for safety)
  chainId: number;
}
```

```ts
const intent = await api.subscriptions.cancelIntent("sub_01HXY…");
await wallet.sendTransaction({
  to: SUBSCRIPTION_MANAGER_ADDRESS,
  data: intent.encodedTx,
  chainId: intent.chainId,
});
```

---

## `api.invoices`

### `create(params)`

Create a one-time invoice. The merchant shares `payUrl` with the user, who
pays via PayRouter on-chain — no further SDK interaction required. Backend
listens for `Paid` and fires `payment.confirmed` (or `payment.expired` after
the deadline).

```ts
async create(params: {
  amountUsd: string;
  merchantOrderId?: string;
  tokenAddress?: string;
  referrerAddress?: string;
  referrerFeeBps?: number;
  description?: string;
}): Promise<InvoiceCreated>
```

`amountUsd` is a USD-denominated decimal string (e.g. `"19.99"`). Setting
`tokenAddress` restricts payment to a single ERC20; omit to allow any
whitelisted token.

Returns `InvoiceCreated`:

```ts
interface InvoiceCreated {
  id: string;                          // DB id, e.g. "inv_01HXY…"
  onChainInvoiceId: `0x${string}`;     // bytes32 used on-chain
  amountUsd: string;
  status: string;                      // PENDING at creation
  merchantOrderId: string | null;
  description: string | null;
  createdAt: string;                   // ISO 8601
  payUrl: string;                      // hosted checkout — share with the customer
  /** @deprecated renamed in 0.2.0; use `onChainInvoiceId` */
  invoiceId?: `0x${string}`;
}
```

Example:

```ts
const invoice = await api.invoices.create({
  amountUsd: "49.99",
  merchantOrderId: "order-001",
  description: "Pro plan — monthly",
});

// Redirect the customer:
res.redirect(invoice.payUrl);
```

### `get(idOrOnChainId)`

Fetch an invoice by DB id (`inv_…`) or on-chain bytes32 id. Public endpoint.

```ts
async get(idOrOnChainId: string): Promise<Invoice>
```

```ts
const invoice = await api.invoices.get("inv_01HXY…");
console.log(invoice.status); // PENDING / PAID / EXPIRED / FAILED
```

---

## Intent flow

Three V2 endpoints return intents instead of performing a chain write
themselves:

- `api.merchants.createRegistrationIntent(...)` → `MerchantRegistrationIntent`
- `api.plans.create(...)` → `PlanIntent`
- `api.subscriptions.cancelIntent(id)` → `CancelIntent`

Each result includes an `encodedTx` (ABI-encoded calldata) for the merchant's
wallet to sign and broadcast against `SubscriptionManager`. The backend never
holds merchant keys; it only observes the resulting on-chain event and
reconciles state.

The expected flow:

1. Backend (server) calls the SDK and receives the intent.
2. Intent is forwarded to the merchant's browser / signing surface.
3. Merchant wallet sends a transaction with `to = SubscriptionManager`,
   `data = encodedTx`, and the matching `chainId`.
4. Chain listener observes the event and fires the corresponding webhook
   (`merchant.registered`, `plan.created`, `subscription.cancelled`).

### Executing an intent with viem

```ts
import { createWalletClient, custom } from "viem";
import { arbitrum } from "viem/chains";

const wallet = createWalletClient({
  chain: arbitrum,
  transport: custom(window.ethereum),
});
const [account] = await wallet.requestAddresses();

const intent = await api.plans.create({
  externalPlanCode: "premium-monthly",
  tokenAddress: USDC_ARBITRUM,
  amount: "9990000",
  monthsPerCycle: 1,
  bufferTimeSeconds: 86_400,
  chainId: arbitrum.id,
});

const txHash = await wallet.sendTransaction({
  account,
  to: SUBSCRIPTION_MANAGER_ADDRESS,
  data: intent.encodedTx,
});
// Wait for the `plan.created` webhook to confirm the chain has settled.
```

The same pattern applies to registration and cancel intents — only the
`encodedTx` payload differs.

---

## Type reference

All types listed below are exported from `@butterpay/core` and originate in
`src/types.ts`.

### `Invoice`

```ts
interface Invoice {
  id: string;
  merchantId: string;
  merchantOrderId?: string | null;
  amount: string;
  /** "USD" until paid, then the actual ERC20 symbol */
  token: string;
  /** "pending" until paid, then the chain name */
  chain: string;
  /** PENDING | PAID | EXPIRED | FAILED */
  status: string;
  onChainInvoiceId?: `0x${string}` | null;
  paymentMethod?: string | null;
  payerAddress?: string | null;
  txHash?: string | null;
  serviceFee?: string | null;
  merchantNet?: string | null;
  referrerAddress?: string | null;
  referrerFeeBps?: number | null;
  createdAt: string;
}
```

### `InvoiceCreated`

```ts
interface InvoiceCreated {
  id: string;
  onChainInvoiceId: `0x${string}`;
  amountUsd: string;
  status: string;
  merchantOrderId: string | null;
  description: string | null;
  createdAt: string;
  payUrl: string;
  /** @deprecated renamed to `onChainInvoiceId` in 0.2.0 */
  invoiceId?: `0x${string}`;
}
```

### `Plan`

```ts
interface Plan {
  id: string;
  merchantId: string;
  externalPlanCode: string;
  onChainPlanId: `0x${string}`;
  tokenAddress: `0x${string}`;
  amount: string;            // smallest unit of tokenAddress
  monthsPerCycle: number;
  bufferTimeSeconds: number;
  metadataHash: `0x${string}`;
  active: boolean;
  chainId: number;
  createdAt?: string;
}
```

### `Subscription`

```ts
interface Subscription {
  id: string;
  planId: string;
  onChainSubId: string;
  subscriber: `0x${string}`;
  status: string;
  anchorTime?: string;
  anchorDay?: number;
  cyclesCharged?: number;
  nextDueAt?: string;
  chainId: number;
  createdAt?: string;
}
```

### `MerchantRegistrationIntent`

```ts
interface MerchantRegistrationIntent {
  metadataHash: `0x${string}`;
  metadataJson: string;
  encodedTx: `0x${string}`;
}
```

### `PlanIntent`

```ts
interface PlanIntent {
  planId: `0x${string}`;
  salt: `0x${string}`;
  metadataHash: `0x${string}`;
  encodedTx: `0x${string}`;
}
```

### `CancelIntent`

```ts
interface CancelIntent {
  encodedTx: `0x${string}`;
  onChainSubId: string;
  chainId: number;
}
```

### Webhook events

`WebhookEvent` is a discriminated union covering every V2 event. The HTTP body
is `{ event, data }` and is signed with
`X-ButterPay-Signature: t=<unix>,v1=<hmac-sha256-hex>` against `${t}.${rawBody}`.

```ts
import type { WebhookEvent } from "@butterpay/core";

function handle(event: WebhookEvent) {
  switch (event.type) {
    case "merchant.registered":   /* { merchantId, onChainMerchantId, txHash, chainId } */ break;
    case "plan.created":          /* { planId, externalPlanCode, chainId, txHash }     */ break;
    case "plan.deactivated":      /* { planId, chainId }                                */ break;
    case "subscription.created":  /* { subscriptionId, planId, subscriber, … }          */ break;
    case "subscription.charged":  /* { subscriptionId, cyclesCharged, amount, … }       */ break;
    case "subscription.charge_failed": /* { subscriptionId, errorCode, errorMessage }   */ break;
    case "subscription.expired":  /* { subscriptionId, lastCyclesCharged, reason }      */ break;
    case "subscription.cancelled":/* { subscriptionId, cancelledBy }                    */ break;
    case "subscription.resubscribed": /* { subscriptionId, newAnchorTime }              */ break;
    case "payment.confirmed":     /* { invoiceId, txHash, amountPaid, merchantNet }     */ break;
    case "payment.expired":       /* { invoiceId }                                      */ break;
  }
}
```

---

## Migration from 0.1.x

The 0.3.0 SDK targets the V2 contract and is not API-compatible with 0.1.x.
The most common breakages:

| 0.1.x | 0.3.0 |
|---|---|
| `api.createInvoice({ amount, token, chain, … })` | `api.invoices.create({ amountUsd, … })` |
| `api.getInvoice(id)` | `api.invoices.get(id)` |
| `sdk.waitForConfirmation(id)` | Removed — listen for the `payment.confirmed` webhook. |
| `sdk.pay(...)` | Removed — share `payUrl` from `api.invoices.create()` instead. |
| `invoiceIdBytes32` (computed client-side) | `onChainInvoiceId` returned on `InvoiceCreated`. |
| `InvoiceCreated.invoiceId` | `@deprecated` — use `id` (DB) or `onChainInvoiceId` (bytes32). |
| `api.createPlan({ name, intervalSeconds, cycles, amountUsd })` | `api.plans.create({ externalPlanCode, tokenAddress, amount, monthsPerCycle, bufferTimeSeconds, chainId })` — returns an intent. |
| `api.cancelSubscription(id)` (server-side cancel) | `api.subscriptions.cancelIntent(id)` — merchant wallet signs. |
| Subscription created via `sdk.subscribe(...)` | Removed — subscriber wallet calls `SubscriptionManager.subscribe(bytes32)` directly. |

The V1 webhook events (`payment.initiated`, `subscription.activated`,
`subscription.paused`, `subscription.completed`, `subscription.canceled`) no
longer exist. Use the V2 list above.

---

## See also

- [API Reference](../api/invoices.md)
- [Authentication](../authentication.md)
- [Webhooks](../webhooks.md)
