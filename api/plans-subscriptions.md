# Plans & Subscriptions API

ButterPay V2 subscriptions are **non-custodial** and contract-driven. The flow has three actors:

1. **Merchant** signs `createPlan()` on the `SubscriptionManager` contract to register a plan on chain. The backend builds the calldata; the merchant wallet sends the transaction.
2. **Subscriber** signs `approve()` + `subscribe()` directly against the contract — no backend POST needed.
3. **Backend chain-listener** observes `PlanCreated`, `Subscribed`, `Charged`, `Cancelled`, `ExpiredByFailure`, `Resubscribed`, and `PlanActiveChanged` events and reconciles them into the `plans` and `subscriptions` tables. Webhooks fire from those event handlers.

Because creation/cancellation are on-chain actions, the relevant POST endpoints return an **intent** (`{ to, chainId, encodedTx, ... }`) rather than mutating state directly. The wallet signs and broadcasts; the listener picks up the result.

All authenticated endpoints accept either a session JWT (`Authorization: Bearer ...`) or `X-API-Key`. See [Authentication](../authentication.md).

---

## Contents

- [Endpoint summary](#endpoint-summary)
- Plans
  - [CreatePlan](#createplan)
  - [ListPlans](#listplans)
  - [GetPlan](#getplan)
  - [Deactivating a plan](#deactivating-a-plan)
- Subscriptions
  - [Subscribing on chain](#subscribing-on-chain)
  - [ListSubscriptions](#listsubscriptions)
  - [GetSubscription](#getsubscription)
  - [GetSubscriptionCharges](#getsubscriptioncharges)
  - [CancelSubscription](#cancelsubscription)
- [Webhooks fired](#webhooks-fired)
- [Field reference](#field-reference)

---

## Endpoint summary

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| `POST` | `/v1/plans` | merchant | Build `createPlan()` intent for the merchant wallet to sign |
| `GET` | `/v1/plans` | merchant | List plans owned by the authenticated merchant |
| `GET` | `/v1/plans/:id` | public | Fetch a plan by `pln_*` id or `0x...` `onChainPlanId` |
| `GET` | `/v1/subscriptions` | merchant | List subscriptions for plans owned by the merchant |
| `GET` | `/v1/subscriptions/:id` | merchant | Fetch a single subscription |
| `GET` | `/v1/subscriptions/:id/charges` | merchant | On-chain `Charged` event history for a subscription |
| `POST` | `/v1/subscriptions/:id/cancel` | merchant | Build `merchantCancel()` intent for the merchant wallet to sign |

There is **no** API endpoint for the subscriber to "create a subscription" — that step is on-chain. See [Subscribing on chain](#subscribing-on-chain).

---

## CreatePlan

Builds an unsigned `createPlan()` calldata blob for the merchant's wallet to send to the `SubscriptionManager` contract. Inserts a row in the `plans` table with `active=false`; the chain-listener flips `active=true` after observing the `PlanCreated` event.

- **Method**: `POST`
- **Path**: `/v1/plans`
- **Auth**: Bearer JWT or `X-API-Key`

The merchant must already be registered on chain (`merchants.onChainMerchantId` must be set). If not, the request returns `400`.

### Request body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `externalPlanCode` | string | yes | Stable per-merchant code, 3–64 chars matching `^[a-z0-9_-]{3,64}$`. Used as the salt input. Must be unique per merchant. |
| `tokenAddress` | string | yes | ERC-20 address charged each cycle. Will be EIP-55 checksummed. |
| `amount` | string | yes | Amount **in smallest units** of the token (decimal string, e.g. `"9990000"` for `9.99` USDC with 6 decimals). |
| `monthsPerCycle` | integer | yes | Cycle length in months. Range: `1..12`. |
| `bufferTimeSeconds` | integer | yes | Grace period after `nextDueAt` before the subscription is treated as failed. Must satisfy `0 <= bufferTimeSeconds < monthsPerCycle * 30 * 86400`. |
| `metadata` | object | no | Arbitrary JSON. Hashed (keccak256 of the canonical JSON with sorted keys) into `metadataHash` and stored on chain. Defaults to `{}`. |
| `chainId` | integer | yes | EVM chain id. Currently supported: `42161` (Arbitrum One), `421614` (Arbitrum Sepolia). |

### Response (HTTP 201)

```json
{
  "to": "0xSubscriptionManagerAddress",
  "chainId": 42161,
  "planId": "0xabc...32bytes",
  "salt": "0xdef...32bytes",
  "metadataHash": "0x111...32bytes",
  "encodedTx": "0xa9059cbb..."
}
```

| Field | Description |
|-------|-------------|
| `to` | `SubscriptionManager` address for `chainId`. The wallet sends to this. |
| `planId` | Deterministic on-chain plan id (`bytes32`). Computed as `keccak256(merchantOnChainId, salt)`. Use this to read the plan on chain or to reference it from `subscribe()`. |
| `salt` | Plan salt derived from `externalPlanCode`. |
| `metadataHash` | `keccak256(canonical JSON of metadata)`. |
| `encodedTx` | ABI-encoded calldata for `createPlan(...)`. |

### How the merchant signs

```ts
import { createWalletClient, http } from "viem";
import { arbitrum } from "viem/chains";

const intent = await fetch("https://api.butterpay.io/v1/plans", {
  method: "POST",
  headers: { "X-API-Key": apiKey, "Content-Type": "application/json" },
  body: JSON.stringify({
    externalPlanCode: "pro_monthly",
    tokenAddress: "0xaf88d065e77c8cC2239327C5EDb3A432268e5831", // USDC arb
    amount: "9990000", // 9.99 USDC (6 decimals)
    monthsPerCycle: 1,
    bufferTimeSeconds: 86400,
    metadata: { name: "Pro Monthly" },
    chainId: 42161,
  }),
}).then((r) => r.json());

const wallet = createWalletClient({ chain: arbitrum, transport: http() /* ... */ });
const txHash = await wallet.sendTransaction({
  to: intent.to,
  data: intent.encodedTx,
});
```

After the transaction confirms, the chain-listener observes `PlanCreated`, calls `applyPlanCreated`, and flips `plans.active` to `true`. A `plan.created` webhook fires to the merchant.

### curl

```bash
curl -X POST https://api.butterpay.io/v1/plans \
  -H "X-API-Key: bp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "externalPlanCode": "pro_monthly",
    "tokenAddress": "0xaf88d065e77c8cC2239327C5EDb3A432268e5831",
    "amount": "9990000",
    "monthsPerCycle": 1,
    "bufferTimeSeconds": 86400,
    "metadata": {"name": "Pro Monthly"},
    "chainId": 42161
  }'
```

---

## ListPlans

Returns every plan owned by the authenticated merchant, including pending (`active=false`) plans whose `PlanCreated` event has not yet been observed.

- **Method**: `GET`
- **Path**: `/v1/plans`
- **Auth**: Bearer JWT or `X-API-Key`

### Response (HTTP 200)

```json
{
  "data": [
    {
      "id": "pln_1a2b3c4d5e6f7g8h9i0j1k2l",
      "merchantId": "mer_01hwzq3k5n8ej4v2b7r9abc123",
      "externalPlanCode": "pro_monthly",
      "onChainPlanId": "0xabc...32bytes",
      "tokenAddress": "0xaf88d065e77c8cC2239327C5EDb3A432268e5831",
      "amount": "9990000",
      "monthsPerCycle": 1,
      "bufferTimeSeconds": 86400,
      "metadataHash": "0x111...32bytes",
      "active": true,
      "chainId": 42161,
      "createdAt": "2026-04-27T12:00:00.000Z"
    }
  ],
  "count": 1
}
```

### curl

```bash
curl https://api.butterpay.io/v1/plans \
  -H "X-API-Key: bp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

---

## GetPlan

Public endpoint used by the hosted subscribe page (`pay.butterpay.io/subscribe/:planId`). It accepts **either** the `pln_*` database id **or** the on-chain `0x...` `onChainPlanId` as the path parameter.

Returns `404` when the plan does not exist or is not yet `active=true` (i.e. the `PlanCreated` event has not been observed yet, or the plan has been deactivated on chain).

- **Method**: `GET`
- **Path**: `/v1/plans/:id`
- **Auth**: Public

### Response (HTTP 200)

```json
{
  "id": "pln_1a2b3c4d5e6f7g8h9i0j1k2l",
  "merchantId": "mer_01hwzq3k5n8ej4v2b7r9abc123",
  "externalPlanCode": "pro_monthly",
  "onChainPlanId": "0xabc...32bytes",
  "tokenAddress": "0xaf88d065e77c8cC2239327C5EDb3A432268e5831",
  "amount": "9990000",
  "monthsPerCycle": 1,
  "bufferTimeSeconds": 86400,
  "metadataHash": "0x111...32bytes",
  "active": true,
  "chainId": 42161,
  "createdAt": "2026-04-27T12:00:00.000Z"
}
```

### curl

```bash
# By DB id
curl https://api.butterpay.io/v1/plans/pln_1a2b3c4d5e6f7g8h9i0j1k2l

# By on-chain plan id (bytes32 hex)
curl https://api.butterpay.io/v1/plans/0xabc1234567890abcdef1234567890abcdef1234567890abcdef1234567890abc
```

---

## Deactivating a plan

V2 has **no** REST endpoint for deactivating plans — this is a deliberate non-custodial design. The merchant calls `SubscriptionManager.deactivatePlan(bytes32 planId)` directly from the same wallet that owns the on-chain merchant record.

```ts
import { encodeFunctionData } from "viem";
import { subscriptionManagerAbi } from "./abi";

const data = encodeFunctionData({
  abi: subscriptionManagerAbi,
  functionName: "deactivatePlan",
  args: [onChainPlanId],
});
await wallet.sendTransaction({ to: subscriptionManagerAddress, data });
```

When the `PlanActiveChanged(active=false)` event is observed, `applyPlanActiveChanged` flips `plans.active` to `false` and a `plan.deactivated` webhook fires. Existing subscriptions are then transitioned to `INACTIVATED_BY_PLAN_CLOSED` by the listener as the contract emits its own `Cancelled`-class events for affected subs.

A helper endpoint that returns a `deactivatePlan()` intent will be added in a later release. Until then, build the calldata client-side as shown above.

---

## Subscribing on chain

There is no `POST /v1/subscriptions` endpoint. To subscribe, the subscriber's wallet talks to the `SubscriptionManager` contract directly. The backend learns about the new subscription only when the chain-listener observes the `Subscribed` event.

### End-to-end flow

1. **Read the plan.** Fetch plan details with `GET /v1/plans/:id` (public). Use either the `pln_*` id or the `onChainPlanId`.
2. **Read the contract address.** Look up the `SubscriptionManager` address for the plan's `chainId` via `GET /v1/config` (or hard-code from the [contracts page](../contracts.md)).
3. **Approve the spend.** Have the subscriber sign `ERC20.approve(subscriptionManager, amount * cyclesYouWantToCover)`. The contract will pull `amount` per cycle until allowance is exhausted; then `ExpiredByFailure` fires.
4. **Subscribe.** Have the subscriber sign `SubscriptionManager.subscribe(bytes32 planId)`. This emits `Subscribed(subId, planId, subscriber, anchorTime, anchorDay, serviceFeeBpsSnapshot)` and immediately performs the first charge (emitting `Charged`).
5. **Wait for the listener.** The chain-listener inserts a row in `subscriptions` with `status='ACTIVE'`, `cyclesCharged=1`, and a computed `nextDueAt`. A `subscription.created` webhook fires, followed by `subscription.charged`.

### viem + wagmi pseudocode

```ts
import { useWriteContract, useReadContract } from "wagmi";
import { erc20Abi, parseUnits } from "viem";
import { subscriptionManagerAbi } from "./abi";

// 1. Fetch plan
const plan = await fetch(`https://api.butterpay.io/v1/plans/${planId}`).then(r => r.json());

// 2. SubscriptionManager address (e.g. from /v1/config or a constant)
const subMgr = "0xSubscriptionManagerAddress";

// 3. Approve enough allowance to cover N cycles up front
const cyclesToCover = 12n;
const totalAllowance = BigInt(plan.amount) * cyclesToCover;
const approveHash = await writeContract({
  abi: erc20Abi,
  address: plan.tokenAddress,
  functionName: "approve",
  args: [subMgr, totalAllowance],
});

// 4. Subscribe (this also performs the first charge)
const subHash = await writeContract({
  abi: subscriptionManagerAbi,
  address: subMgr,
  functionName: "subscribe",
  args: [plan.onChainPlanId],
});

// 5. Backend listener picks up `Subscribed` and inserts the subscription row.
//    Poll GET /v1/subscriptions or listen for the subscription.created webhook.
```

### Why no API write?

Subscriber funds and authorisations live in the subscriber's wallet and the ERC-20 contract; ButterPay never custodies them. Forcing the subscribe call through the contract guarantees the backend cannot create or modify subscriptions out of band — every state change is rooted in an on-chain event.

---

## ListSubscriptions

Returns every subscription whose `planId` belongs to a plan owned by the authenticated merchant. Optional `status` query filter.

- **Method**: `GET`
- **Path**: `/v1/subscriptions`
- **Auth**: Bearer JWT or `X-API-Key`

### Query parameters

| Name | Type | Description |
|------|------|-------------|
| `status` | string | One of `ACTIVE`, `CANCELLED`, `EXPIRED_BY_FAILURE`, `INACTIVATED_BY_PLAN_CLOSED`. Omit for all. |

### Response (HTTP 200)

```json
{
  "data": [
    {
      "id": "sub_1a2b3c4d5e6f7g8h9i0j1k2l",
      "planId": "pln_1a2b3c4d5e6f7g8h9i0j1k2l",
      "onChainSubId": "42",
      "subscriberAddress": "0xSubscriberWalletAddress",
      "anchorTime": "2026-04-27T12:00:00.000Z",
      "anchorDay": 27,
      "cyclesCharged": 3,
      "serviceFeeBpsSnapshot": 80,
      "status": "ACTIVE",
      "nextDueAt": "2026-07-27T12:00:00.000Z",
      "cancelReason": null,
      "chainId": 42161,
      "createdAt": "2026-04-27T12:00:30.000Z",
      "updatedAt": "2026-06-27T12:00:30.000Z"
    }
  ],
  "count": 1
}
```

### curl

```bash
curl "https://api.butterpay.io/v1/subscriptions?status=ACTIVE" \
  -H "X-API-Key: bp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

---

## GetSubscription

Fetch a single subscription. Returns `404` if the subscription does not exist or its plan is not owned by the authenticated merchant.

- **Method**: `GET`
- **Path**: `/v1/subscriptions/:id`
- **Auth**: Bearer JWT or `X-API-Key`

### curl

```bash
curl https://api.butterpay.io/v1/subscriptions/sub_1a2b3c4d5e6f7g8h9i0j1k2l \
  -H "X-API-Key: bp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

Response is a single subscription record (same shape as one element of [ListSubscriptions](#listsubscriptions) `data`).

---

## GetSubscriptionCharges

Returns the on-chain charge history for a subscription, sourced from `chain_events` rows where `event_name='Charged'` and the `subId` arg matches.

- **Method**: `GET`
- **Path**: `/v1/subscriptions/:id/charges`
- **Auth**: Bearer JWT or `X-API-Key`

### Response (HTTP 200)

```json
{
  "data": [
    {
      "cycle": 3,
      "txHash": "0xabc...",
      "amount": "9990000",
      "merchantNet": "9910080",
      "serviceFee": "79920",
      "chargedAt": "2026-06-27T12:00:14.000Z",
      "blockNumber": 234567890
    }
  ],
  "count": 1,
  "cyclesTotal": 3
}
```

| Field | Description |
|-------|-------------|
| `cycle` | The cycle number (`cyclesCharged` at the time of this charge). |
| `txHash` | Transaction containing the `Charged` event. |
| `amount` | Total token amount pulled from subscriber, in smallest units. |
| `merchantNet` | Net amount delivered to the merchant after fees, smallest units. |
| `serviceFee` | Service fee taken, smallest units. |
| `chargedAt` | Time the listener processed the event. |
| `cyclesTotal` | Current `subscriptions.cyclesCharged` (top-level, not per row). |

### curl

```bash
curl https://api.butterpay.io/v1/subscriptions/sub_1a2b3c4d5e6f7g8h9i0j1k2l/charges \
  -H "X-API-Key: bp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

---

## CancelSubscription

Builds an unsigned `merchantCancel(uint256 onChainSubId)` calldata blob. The merchant's wallet (which is the on-chain admin for the merchant) must sign and send. The backend does **not** mutate the subscription row; the chain-listener does that when `Cancelled` is observed.

- **Method**: `POST`
- **Path**: `/v1/subscriptions/:id/cancel`
- **Auth**: Bearer JWT or `X-API-Key`

### Response (HTTP 200)

```json
{
  "to": "0xSubscriptionManagerAddress",
  "chainId": 42161,
  "encodedTx": "0x...calldata...",
  "onChainSubId": "42"
}
```

### How the merchant signs

```ts
const intent = await fetch(
  `https://api.butterpay.io/v1/subscriptions/${subId}/cancel`,
  { method: "POST", headers: { "X-API-Key": apiKey } },
).then((r) => r.json());

await wallet.sendTransaction({ to: intent.to, data: intent.encodedTx });
```

When the `Cancelled` event is observed, `applyCancelled` sets `status='CANCELLED'`, clears `nextDueAt`, and a `subscription.cancelled` webhook fires.

### Subscriber-initiated cancel

The subscriber can also cancel by calling `SubscriptionManager.subscriberCancel(uint256 subId)` directly, or by revoking the ERC-20 allowance (which triggers `ExpiredByFailure` on the next cycle). Both paths land in the same listener handlers and emit the corresponding webhook (`subscription.cancelled` or `subscription.expired`).

### curl

```bash
curl -X POST https://api.butterpay.io/v1/subscriptions/sub_1a2b3c4d5e6f7g8h9i0j1k2l/cancel \
  -H "X-API-Key: bp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

---

## Webhooks fired

All webhooks are dispatched by the chain-listener after it processes the corresponding event, never by the API request itself. See [Webhooks](../webhooks.md) for delivery semantics, signing, and full payload shapes.

| Event | Source on chain | When |
|-------|------------------|------|
| `plan.created` | `PlanCreated` | Merchant's `createPlan()` tx confirms; row flips to `active=true`. |
| `plan.deactivated` | `PlanActiveChanged(active=false)` | Merchant's `deactivatePlan()` tx confirms. |
| `subscription.created` | `Subscribed` | Subscriber's `subscribe()` tx confirms; row inserted with `status='ACTIVE'`. |
| `subscription.charged` | `Charged` | Each successful auto-charge cycle (including the first one inside `subscribe()`). |
| `subscription.cancelled` | `Cancelled` | Either `merchantCancel()` or `subscriberCancel()` tx confirms. |
| `subscription.expired` | `ExpiredByFailure` | The contract failed to pull funds (subscriber allowance/balance ran out). |
| `subscription.resubscribed` | `Resubscribed` | A previously cancelled/expired subscriber re-subscribed; the existing row is reset (`cyclesCharged=0`, new anchor). |

---

## Field reference

### Plan (`plans` table)

| Column | Type | Notes |
|--------|------|-------|
| `id` | string | `pln_` + 24-char nanoid (lowercase alphanumeric). |
| `merchantId` | string | FK → `merchants.id` (`mer_*`). |
| `externalPlanCode` | string | 3–64 chars matching `^[a-z0-9_-]{3,64}$`. Unique per merchant. Acts as the salt input. |
| `onChainPlanId` | string | `bytes32` hex. `keccak256(merchantOnChainId, salt)`. |
| `tokenAddress` | string | EIP-55 checksummed ERC-20 address. |
| `amount` | string | Token smallest units, decimal string. |
| `monthsPerCycle` | integer | Cycle length in months, `1..12`. |
| `bufferTimeSeconds` | integer | Grace period after `nextDueAt` before failure escalates. |
| `metadataHash` | string | `keccak256(canonical-JSON(metadata))`. |
| `active` | boolean | `false` until `PlanCreated` is observed; flipped to `false` on `PlanActiveChanged(false)`. |
| `chainId` | integer | EVM chain id. |
| `createdAt` | timestamp | Set when the intent row is inserted, before chain confirmation. |

### Subscription (`subscriptions` table)

| Column | Type | Notes |
|--------|------|-------|
| `id` | string | `sub_` + 24-char nanoid. |
| `planId` | string | FK → `plans.id`. |
| `onChainSubId` | string | `uint256` from `Subscribed`, stored as decimal string. Unique per `(chainId, onChainSubId)`. |
| `subscriberAddress` | string | Wallet that signed `subscribe()`. |
| `anchorTime` | timestamp | Time of the first successful charge. Cycle boundaries are computed from this. |
| `anchorDay` | integer | Day-of-month of the anchor (`1..31`), used for monthly cadence on irregular months. |
| `cyclesCharged` | integer | Increments on each `Charged` event. Reset to `0` on `Resubscribed`. |
| `serviceFeeBpsSnapshot` | integer | Service fee bps captured at subscribe time. |
| `status` | enum | One of `ACTIVE`, `CANCELLED`, `EXPIRED_BY_FAILURE`, `INACTIVATED_BY_PLAN_CLOSED`. |
| `nextDueAt` | timestamp | Read from `SubscriptionManager.nextDueAt(subId)` after each event. `null` when not active. |
| `cancelReason` | string | Free-form reason from the listener (`"merchant_cancel"`, `"subscriber_cancel"`, `"plan_closed"`, etc.). |
| `chainId` | integer | EVM chain id. |
| `createdAt` | timestamp | Set when the listener inserts the row. |
| `updatedAt` | timestamp | Touched on each event. |

---

## See also

- [Authentication](../authentication.md)
- [Webhooks](../webhooks.md)
- [Contracts & addresses](../contracts.md)
- [Invoices API](invoices.md)
