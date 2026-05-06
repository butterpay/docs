# Config

Public endpoint exposing the per-chain V2 contract addresses ButterPay is
running against. Useful for client-side flows (the public subscribe page,
the SDK, custom integrations) that need to know which `SubscriptionManager` /
`PayRouter` to interact with — without baking addresses into your build.

> Authenticated intent endpoints (`POST /v1/merchants/register-intent`,
> `POST /v1/plans`, `POST /v1/subscriptions/:id/cancel`) already include
> the `to` contract address in their response, so most server-side
> integrators won't need to call this endpoint directly.

---

## GetConfig

Return the chains and contract addresses currently active in the deployment.

- **Path**: `/v1/config`
- **Method**: `GET`
- **Auth**: none (public)
- **Use cases**:
  - Public subscribe pages (`/subscribe/{planId}`) — need `subscriptionManager`
    to construct ERC-20 `approve` + `subscribe(bytes32)` calls
  - SDKs / dApps embedding ButterPay — replaces shipping
    `NEXT_PUBLIC_V2_PAY_ROUTER_*` / `NEXT_PUBLIC_V2_SUB_MGR_*` env vars
  - Healthchecks confirming a deployment is wired to the right contracts

### Request

```bash
curl https://api.butterpay.io/v1/config
```

### Response

```json
{
  "chains": {
    "42161": {
      "chainId": 42161,
      "name": "Arbitrum",
      "payRouter": "0x5FbAc17B617b34c6D27723B3A0A588A11099D347",
      "subscriptionManager": "0x96039F9bBDe549882ff2ABCBCd95FF666FFf2030",
      "authorityManager": null
    }
  }
}
```

The top-level `chains` object is keyed by **decimal `chainId` as a
string**. Iterate `Object.values(response.chains)` if you don't know which
chains are active in advance.

### Field reference

| Field | Type | Notes |
|---|---|---|
| `chainId` | number | EIP-155 chain id (e.g. `42161` = Arbitrum One) |
| `name` | string | Human-readable display label (`"Arbitrum"`, `"Arbitrum Sepolia"`) |
| `payRouter` | `0x` string | Address of the `PayRouter` contract on this chain. Used for one-time invoice payments. May be `null` if not deployed in this env |
| `subscriptionManager` | `0x` string | Address of the `SubscriptionManager` contract. Used for `registerMerchant` / `createPlan` / `subscribe` / `merchantCancel` / `charge`. Always set when the chain entry is present |
| `authorityManager` | `0x` string \| null | Address of an optional separate `AuthorityManager` contract. `null` when authority is co-located on `SubscriptionManager` (default) |

### Behavior

- **Chains with no contracts deployed are omitted.** If you only see
  `42161` in the response, mainnet is the only active chain in this
  deployment. Don't assume Sepolia is also there.
- **`null` payRouter / authorityManager are explicit.** They mean "no
  contract deployed on this chain in this env", not "unknown".
- **Cache freely.** The response only changes when ButterPay deploys new
  contract addresses (rare). A 5-minute client cache is reasonable; a
  reload on app start is fine for most apps.
- **No auth header is needed.** Sending `Authorization` or `X-API-Key` is
  not an error, but is ignored.

---

## Example: building a subscribe transaction client-side

```ts
import { encodeFunctionData, createPublicClient, http } from "viem";
import { arbitrum } from "viem/chains";

// 1. Look up SubscriptionManager address for the plan's chain
const cfg = await fetch("https://api.butterpay.io/v1/config").then(r => r.json());
const chain = cfg.chains[String(plan.chainId)];
if (!chain) throw new Error(`Chain ${plan.chainId} not configured`);
const subMgr = chain.subscriptionManager;

// 2. ERC-20 approve(subMgr, amount * cycles)
const approveData = encodeFunctionData({
  abi: ERC20_ABI,
  functionName: "approve",
  args: [subMgr, BigInt(plan.amount) * 12n], // 12 cycles of headroom
});
await wallet.sendTransaction({ to: plan.tokenAddress, data: approveData, chainId: plan.chainId });

// 3. SubscriptionManager.subscribe(bytes32 planId)
const subscribeData = encodeFunctionData({
  abi: SUBSCRIBE_ABI,
  functionName: "subscribe",
  args: [plan.onChainPlanId],
});
await wallet.sendTransaction({ to: subMgr, data: subscribeData, chainId: plan.chainId });
```

The SDK (`@butterpay/core`) wraps this pattern via `api.config.get()`.

---

## Errors

This endpoint does not error under normal operation. It returns 200 with
an empty `chains` object only if the backend has been started without any
`V2_*` contract env vars configured (a misconfiguration; the listener
will also have logged `[v2] No V2 contracts configured`).
