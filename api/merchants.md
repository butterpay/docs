# Merchants API

The Merchants API covers account registration, authentication, profile management, and API key
lifecycle for ButterPay merchants. Use these endpoints to sign up, obtain a JWT or API key, update
your account settings, and rotate credentials.

Authentication works in two layers. A **JWT** (obtained via `Login` or `SignUp`) is a short-lived
token (24 hours) used for security-sensitive operations such as key rotation and Dashboard access.
An **API key** (`X-Api-Key`) is a long-lived credential used in server-to-server calls like invoice
creation. See [Authentication](../authentication.md) for full details.

---

- [Login](#login)
- [GetCurrentMerchant](#getcurrentmerchant)
- [SignUp](#signup)
- [GetMe](#getme)
- [UpdateMe](#updateme)
- [V2: On-chain registration](#v2-on-chain-registration)
  - [RegisterIntent](#registerintent)
  - [V2 merchant fields](#v2-merchant-fields)
- [GenerateApiKey](#generateapikey)
- [RotateApiKey](#rotateapikey)

---

## Login

Authenticate with email and password and receive a signed JWT plus the full merchant record,
including the current API key prefix.

- **Method**: `POST`
- **Path**: `/v1/auth/login`
- **Auth**: Public
- **Description**: Validates the supplied credentials against the stored bcrypt hash. On success
  returns a 24-hour JWT and the merchant object (including the stored API key prefix so the
  Dashboard can display it). Returns `401` for any invalid email/password combination — no
  distinction is made to avoid user-enumeration. Rate-limited to **10 requests per minute per IP**.

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `email` | string | Yes | Merchant email address. Normalized to lowercase before lookup. |
| `password` | string | Yes | Account password. |

### Returns

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "merchant": {
    "id": "mer_01hwzq3k5n8ej4v2b7r9abc123",
    "name": "Acme Store",
    "email": "hello@acme.io",
    "apiKey": "bp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx...",
    "serviceFeeBps": 80,
    "webhookUrl": "https://acme.io/webhooks/butter",
    "receivingAddresses": {
      "arbitrum": "0xMerchantWalletAddress",
      "ethereum": "0xMerchantWalletAddress"
    }
  }
}
```

`apiKey` in the response is the stored **prefix** of the key (e.g. `bp_abc1...`), not the
full plaintext key. The full key is only returned once, at generation or rotation time.

### Example

**Request:**

```bash
curl -X POST https://api.butterpay.io/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "hello@acme.io",
    "password": "supersecret"
  }'
```

**Response** (HTTP 200):

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "merchant": {
    "id": "mer_01hwzq3k5n8ej4v2b7r9abc123",
    "name": "Acme Store",
    "email": "hello@acme.io",
    "apiKey": "bp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx...",
    "serviceFeeBps": 80,
    "webhookUrl": "https://acme.io/webhooks/butter",
    "receivingAddresses": {
      "arbitrum": "0xMerchantWalletAddress"
    }
  }
}
```

---

## GetCurrentMerchant

Return the merchant associated with the current Bearer JWT. Primarily used by the Dashboard to
hydrate session state after a page reload.

- **Method**: `GET`
- **Path**: `/v1/auth/me`
- **Auth**: `Bearer JWT`
- **Description**: Verifies the JWT in the `Authorization` header, looks up the merchant, and
  returns the full profile including the API key prefix. Returns `401` if the token is missing,
  expired, or belongs to a deactivated merchant.

### Parameters

None. The merchant identity is derived from the JWT.

### Returns

```json
{
  "id": "mer_01hwzq3k5n8ej4v2b7r9abc123",
  "name": "Acme Store",
  "email": "hello@acme.io",
  "apiKey": "bp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx...",
  "serviceFeeBps": 80,
  "webhookUrl": "https://acme.io/webhooks/butter",
  "receivingAddresses": {
    "arbitrum": "0xMerchantWalletAddress",
    "ethereum": "0xMerchantWalletAddress"
  }
}
```

### Example

**Request:**

```bash
curl https://api.butterpay.io/v1/auth/me \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

**Response** (HTTP 200):

```json
{
  "id": "mer_01hwzq3k5n8ej4v2b7r9abc123",
  "name": "Acme Store",
  "email": "hello@acme.io",
  "apiKey": "bp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx...",
  "serviceFeeBps": 80,
  "webhookUrl": "https://acme.io/webhooks/butter",
  "receivingAddresses": {
    "arbitrum": "0xMerchantWalletAddress"
  }
}
```

---

## SignUp

Register a new merchant account. On success the account is created and a JWT is issued immediately
so the caller lands directly on the Dashboard without a separate login step.

- **Method**: `POST`
- **Path**: `/v1/merchants`
- **Auth**: Public
- **Description**: Creates a merchant record with the supplied name, email, and password (minimum
  8 characters). The password is hashed with bcrypt (12 rounds) before storage — the plaintext is
  never persisted. An API key is **not** generated at this stage; call
  [GenerateApiKey](#generateapikey) after configuring at least one receiving address. Returns `409`
  if the email is already registered. Rate-limited to **5 requests per minute per IP**.

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | Yes | Display name for the merchant account, e.g. `"Acme Store"`. |
| `email` | string | Yes | Email address used for login. Stored in lowercase. |
| `password` | string | Yes | Account password. Minimum 8 characters. |
| `webhookUrl` | string | No | Default webhook URL for payment notifications. Must pass SSRF validation (HTTPS required in production, private IPs rejected). See [Webhooks](../webhooks.md). |
| `receivingAddresses` | object | No | Map of chain name to wallet address, e.g. `{"arbitrum": "0x..."}`. Can also be configured later via [UpdateMe](#updateme). |
| `serviceFeeBps` | number | No | Custom service fee in basis points. Defaults to `80` (0.80 %). Only configurable at account creation. |

### Returns

```json
{
  "id": "mer_01hwzq3k5n8ej4v2b7r9abc123",
  "name": "Acme Store",
  "email": "hello@acme.io",
  "serviceFeeBps": 80,
  "createdAt": "2026-04-27T12:00:00.000Z",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

`token` is a 24-hour JWT. Use it immediately to call [GenerateApiKey](#generateapikey) or browse
the Dashboard. The API key is not included in this response because it has not yet been generated.

### Example

**Request:**

```bash
curl -X POST https://api.butterpay.io/v1/merchants \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Acme Store",
    "email": "hello@acme.io",
    "password": "supersecret",
    "webhookUrl": "https://acme.io/webhooks/butter",
    "receivingAddresses": {
      "arbitrum": "0xMerchantWalletAddress"
    }
  }'
```

**Response** (HTTP 201):

```json
{
  "id": "mer_01hwzq3k5n8ej4v2b7r9abc123",
  "name": "Acme Store",
  "email": "hello@acme.io",
  "serviceFeeBps": 80,
  "createdAt": "2026-04-27T12:00:00.000Z",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

---

## GetMe

Return the authenticated merchant's full profile, including all receiving addresses, the current
webhook URL, and V2 on-chain registration state.

- **Method**: `GET`
- **Path**: `/v1/merchants/me`
- **Auth**: `apiKeyAuth (Bearer JWT or X-Api-Key)`
- **Description**: Looks up the merchant from the hashed API key (or JWT) and returns the stored
  profile. The response includes both V1 fields (`receivingAddresses`, `webhookUrl`) and V2 fields
  (`onChainMerchantId`, `receiverAddress`, `adminAddress`, `metadataHash`, `chainId`,
  `chainActive`, `registeredAt`) — the V2 fields are `null` until the merchant completes
  [on-chain registration](#v2-on-chain-registration). The boolean `apiKeyGenerated` indicates
  whether a real API key has been issued (`false` for newly-signed-up merchants whose `apiKey`
  column still holds a `not_generated_<id>` placeholder). Returns `404` if the merchant record no
  longer exists (e.g. deactivated).

### Parameters

None. The merchant identity is derived from the `X-Api-Key` header (or `Authorization: Bearer`
JWT).

### Returns

```json
{
  "id": "mer_019df68e-4361-7db4-9f63-2d91a9bd6807",
  "name": "Acme Store",
  "email": "hello@acme.io",
  "webhookUrl": "https://acme.io/webhooks/butter",
  "receivingAddresses": {
    "arbitrum": "0xMerchantWalletAddress",
    "ethereum": "0xMerchantWalletAddress"
  },
  "serviceFeeBps": 80,
  "createdAt": "2026-04-01T10:00:00.000Z",
  "onChainMerchantId": "3",
  "chainId": 42161,
  "chainActive": true,
  "receiverAddress": "0xf962939089c02cA84DBc6ea320B8a77d0C2738CE",
  "adminAddress": "0xf962939089c02cA84DBc6ea320B8a77d0C2738CE",
  "metadataHash": "0xe3dae327f164fbea38f35535bfba1ba549d3460c0e72d5e685a0912724544cf3",
  "registeredAt": "2026-05-05T05:38:48.545Z",
  "apiKeyGenerated": true
}
```

For a merchant that has signed up but not yet registered on-chain or generated a key, the V2
fields are `null` and `apiKeyGenerated` is `false`:

```json
{
  "id": "mer_019df68e-4361-7db4-9f63-2d91a9bd6807",
  "name": "Acme Store",
  "email": "hello@acme.io",
  "webhookUrl": null,
  "receivingAddresses": {},
  "serviceFeeBps": 80,
  "createdAt": "2026-04-01T10:00:00.000Z",
  "onChainMerchantId": null,
  "chainId": null,
  "chainActive": null,
  "receiverAddress": null,
  "adminAddress": null,
  "metadataHash": null,
  "registeredAt": null,
  "apiKeyGenerated": false
}
```

### Example

**Request:**

```bash
curl https://api.butterpay.io/v1/merchants/me \
  -H "X-Api-Key: bp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx..."
```

**Response** (HTTP 200):

```json
{
  "id": "mer_019df68e-4361-7db4-9f63-2d91a9bd6807",
  "name": "Acme Store",
  "email": "hello@acme.io",
  "webhookUrl": "https://acme.io/webhooks/butter",
  "receivingAddresses": {
    "arbitrum": "0xMerchantWalletAddress"
  },
  "serviceFeeBps": 80,
  "createdAt": "2026-04-01T10:00:00.000Z",
  "onChainMerchantId": "3",
  "chainId": 42161,
  "chainActive": true,
  "receiverAddress": "0xf962939089c02cA84DBc6ea320B8a77d0C2738CE",
  "adminAddress": "0xf962939089c02cA84DBc6ea320B8a77d0C2738CE",
  "metadataHash": "0xe3dae327f164fbea38f35535bfba1ba549d3460c0e72d5e685a0912724544cf3",
  "registeredAt": "2026-05-05T05:38:48.545Z",
  "apiKeyGenerated": true
}
```

---

## UpdateMe

Update mutable fields on the authenticated merchant's profile. All fields are optional; only
supplied fields are modified.

- **Method**: `PATCH`
- **Path**: `/v1/merchants/me`
- **Auth**: `X-Api-Key`
- **Description**: Applies a partial update to the merchant record. When `webhookUrl` is provided,
  the backend runs SSRF validation before persisting: private-range IPs are rejected and HTTPS is
  required in the production environment. See [Webhooks](../webhooks.md) for full URL rules. Returns
  the updated merchant record.

  Once a merchant has completed on-chain registration (`onChainMerchantId` is set), the `name` and
  `email` fields **cannot be changed** — the business name is committed as part of `metadataHash`
  on the SubscriptionManager contract, so mutating it off-chain would desync the DB row from the
  on-chain commitment. Update `webhookUrl` and other off-chain fields freely.

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | No | New display name. **Locked** once the merchant is on-chain registered (committed in `metadataHash`). |
| `email` | string | No | New email address. **Locked** once the merchant is on-chain registered (committed in `metadataHash`). |
| `webhookUrl` | string | No | New default webhook URL. Must pass SSRF validation. See [Webhooks](../webhooks.md). |
| `receivingAddresses` | object | No | Updated map of chain name to wallet address. Replaces the existing map entirely. Used by the V1 settlement path; on V2 the on-chain `receiverAddress` is the source of truth. |

### Returns

The full updated merchant record (same shape as [GetMe](#getme)).

### Example

**Request:**

```bash
curl -X PATCH https://api.butterpay.io/v1/merchants/me \
  -H "X-Api-Key: bp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx..." \
  -H "Content-Type: application/json" \
  -d '{
    "webhookUrl": "https://acme.io/webhooks/butter-v2",
    "receivingAddresses": {
      "arbitrum": "0xNewWalletAddress",
      "polygon": "0xNewWalletAddress"
    }
  }'
```

**Response** (HTTP 200):

```json
{
  "id": "mer_019df68e-4361-7db4-9f63-2d91a9bd6807",
  "name": "Acme Store",
  "email": "hello@acme.io",
  "webhookUrl": "https://acme.io/webhooks/butter-v2",
  "receivingAddresses": {
    "arbitrum": "0xNewWalletAddress",
    "polygon": "0xNewWalletAddress"
  },
  "serviceFeeBps": 80,
  "createdAt": "2026-04-01T10:00:00.000Z",
  "onChainMerchantId": "3",
  "chainId": 42161,
  "chainActive": true,
  "receiverAddress": "0xf962939089c02cA84DBc6ea320B8a77d0C2738CE",
  "adminAddress": "0xf962939089c02cA84DBc6ea320B8a77d0C2738CE",
  "metadataHash": "0xe3dae327f164fbea38f35535bfba1ba549d3460c0e72d5e685a0912724544cf3",
  "registeredAt": "2026-05-05T05:38:48.545Z"
}
```

---

## V2: On-chain registration

V2 merchants register their identity on the SubscriptionManager smart contract before activating
billing. Registration anchors the merchant's `receiverAddress` (where USDC/USDT settlements are
sent) and a `metadataHash` (a keccak256 commitment to the merchant's canonical metadata, e.g. the
business name and contact email) to a permanent on-chain record. Once registered, the merchant has
an `onChainMerchantId` (a `uint256` assigned by the contract) which becomes the canonical merchant
identity used by all subscription / payment flows.

The flow is:

1. Merchant POSTs to [`/v1/merchants/register-intent`](#registerintent) with their desired
   `receiverAddress`, an arbitrary `metadata` object, and a `chainId`. The backend computes
   `metadataHash = keccak256(canonicalJson(metadata))`, persists `(receiverAddress, metadataHash,
   chainId)` on the merchant row, and returns the encoded `registerMerchant(receiver,
   metadataHash)` calldata along with the SubscriptionManager address.
2. The merchant's wallet (browser-side or external signer) submits a transaction with `to` =
   SubscriptionManager and `data` = `encodedTx`. Gas and signing are entirely client-side — the
   backend never holds keys.
3. The on-chain `registerMerchant` call emits `MerchantRegistered(merchantId, admin, receiver,
   metadataHash)`. ButterPay's chain-listener service observes the event, matches it to the
   merchant row by `(chainId, metadataHash)`, and writes back `onChainMerchantId`, `adminAddress`,
   `chainActive=true`, and `registeredAt = blockTimestamp`.
4. Once the listener has updated the row, [`GET /v1/merchants/me`](#getme) returns a populated
   `onChainMerchantId` and the merchant can call [GenerateApiKey](#generateapikey) to obtain a
   live API key.

A `merchant.registered` webhook is delivered (to `webhookUrl`, if configured) once the listener
finishes processing the event, so dashboards can switch from "pending activation" to "active"
without polling.

### RegisterIntent

Build the `registerMerchant` calldata for the merchant's wallet to sign. Persists `receiverAddress`,
`metadataHash`, and `chainId` on the merchant row so the chain-listener can correlate the resulting
`MerchantRegistered` event back to this merchant.

- **Method**: `POST`
- **Path**: `/v1/merchants/register-intent`
- **Auth**: `apiKeyAuth (Bearer JWT or X-Api-Key)` — typically called from the Dashboard with a
  Bearer JWT, since merchants without a configured `receiverAddress` may not have generated an
  API key yet.
- **Description**: Computes `metadataHash = keccak256(canonicalJson(metadata))` (keys sorted for
  determinism), encodes a call to `SubscriptionManager.registerMerchant(receiver, metadataHash)`,
  and updates the merchant row with the (`receiverAddress`, `metadataHash`, `chainId`) triple. The
  response gives the wallet everything it needs to submit the transaction. Calling this endpoint
  again before the on-chain event is observed simply overwrites the pending intent — only the
  most recent `(chainId, metadataHash)` will match an emitted event.

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `receiverAddress` | string | Yes | EVM address (any case) where USDC / USDT settlements should be paid. Normalised to EIP-55 checksum form before persisting. |
| `metadata` | object | Yes | Arbitrary JSON object committed on-chain via its keccak256 hash. Typical contents: `{ "name": "Acme Store", "email": "hello@acme.io", "version": 1 }`. Keys are sorted before hashing so the same object always produces the same hash. |
| `chainId` | number | Yes | Chain to register on. Currently `42161` (Arbitrum mainnet) is supported in production, with `421614` (Arbitrum Sepolia) for staging. |

### Returns

| Field | Type | Description |
|-------|------|-------------|
| `to` | string | Address of the `SubscriptionManager` contract on `chainId`. The wallet sends the transaction to this address. |
| `chainId` | number | Echo of the requested chain id. |
| `metadataHash` | string | `0x`-prefixed 32-byte keccak256 hash of the canonical metadata JSON. |
| `metadataJson` | string | The canonical (sorted-key) JSON string that was hashed. Stored client-side if you want to re-derive the hash later or display the committed metadata. |
| `encodedTx` | string | ABI-encoded calldata for `registerMerchant(address receiver, bytes32 metadataHash)`. Pass this as the `data` field when calling `eth_sendTransaction` / `wallet.sendTransaction`. |

### Example

**Request:**

```bash
curl -X POST https://api.butterpay.io/v1/merchants/register-intent \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "Content-Type: application/json" \
  -d '{
    "receiverAddress": "0xf962939089c02cA84DBc6ea320B8a77d0C2738CE",
    "metadata": {
      "name": "Acme Store",
      "email": "hello@acme.io",
      "version": 1
    },
    "chainId": 42161
  }'
```

**Response** (HTTP 200):

```json
{
  "to": "0x4f7a2C1B8E2B1c0a9c3D5e6F7a2C1B8E2B1c0a9c",
  "chainId": 42161,
  "metadataHash": "0xe3dae327f164fbea38f35535bfba1ba549d3460c0e72d5e685a0912724544cf3",
  "metadataJson": "{\"email\":\"hello@acme.io\",\"name\":\"Acme Store\",\"version\":1}",
  "encodedTx": "0x1234abcd000000000000000000000000f962939089c02ca84dbc6ea320b8a77d0c2738ce..."
}
```

The merchant's wallet then submits:

```js
await wallet.sendTransaction({
  to: response.to,
  data: response.encodedTx,
  chainId: response.chainId,
});
```

When the transaction is mined, the chain-listener observes `MerchantRegistered`, populates
`onChainMerchantId` / `adminAddress` / `chainActive` / `registeredAt` on the merchant row, and
fires a `merchant.registered` webhook (see [Webhooks](../webhooks.md)).

### Error responses

| Status | Error | Meaning |
|--------|-------|---------|
| 400 | `receiverAddress required` | Body missing `receiverAddress`. |
| 400 | `chainId required` | Body missing `chainId`. |
| 400 | `No V2 contracts configured for chainId=...` | The requested chain is not supported (only Arbitrum mainnet / Sepolia today). |
| 400 | `V2 SubscriptionManager not set for ...` | Backend env is missing the SubscriptionManager address for this chain — contact ButterPay support. |

### V2 merchant fields

The following columns on the `merchants` table back the V2 on-chain registration state. They are
all `null` until the chain-listener processes `MerchantRegistered`:

| Field (camelCase) | Column | Type | Description |
|--------|--------|------|-------------|
| `onChainMerchantId` | `on_chain_merchant_id` | string (uint256) | The id assigned by `SubscriptionManager` when `MerchantRegistered` is emitted. This is the canonical merchant identity on chain — all subscription / payment events reference this id, not the `mer_...` UUID. Returned as a string in JSON to avoid `Number` precision loss. |
| `chainId` | `chain_id` | integer | EVM chain id where the merchant registered. Currently `42161` (Arbitrum One). Combined with `onChainMerchantId`, forms a unique identifier (`merchants_chain_id_on_chain_merchant_id_unq`). |
| `chainActive` | `chain_active` | boolean | Mirrors the contract's active flag. `true` after `MerchantRegistered`; flips to `false` if the merchant (or contract owner) calls a deactivation function and `MerchantActiveChanged` fires. Rare in practice. |
| `receiverAddress` | `receiver_address` | string (EIP-55) | EVM address that receives USDC / USDT settlements. Set by `register-intent` and confirmed by the on-chain event. On V2 this is the source of truth — the V1 `receivingAddresses` map is kept for backwards compatibility only. |
| `adminAddress` | `admin_address` | string (EIP-55) | EVM address that signed the `registerMerchant` transaction. Used for governance ops on the contract: deactivating the merchant, calling `updateReceiver`, etc. |
| `metadataHash` | `metadata_hash` | string (`0x`-prefixed bytes32) | keccak256 of the canonical-JSON merchant metadata committed on chain. Re-deriving from `metadataJson` lets you prove what was committed. |
| `registeredAt` | `registered_at` | timestamptz | Block timestamp of the `MerchantRegistered` event. Set by the chain-listener, not by `register-intent`. |

---

## GenerateApiKey

Generate the live API key for the account. This is a one-time action — a key can only be generated
when no key currently exists. Subsequent rotations use [RotateApiKey](#rotateapikey).

- **Method**: `POST`
- **Path**: `/v1/merchants/me/generate-key`
- **Auth**: `apiKeyAuth (Bearer or X-Api-Key)` — for first-time generation a brand-new merchant has no API key, so use the `Authorization: Bearer <jwt>` returned by [SignUp](#signup) or [Login](#login)
- **Description**: Generates a new API key, stores only its bcrypt hash, and returns the plaintext
  key exactly once. The key is not stored in plaintext anywhere — if it is lost, rotation is the
  only recovery path.

  Before issuing a key, the backend verifies the merchant has at least one configured receiver:

  - **V2** — `receiverAddress` is set (i.e. the merchant has called
    [RegisterIntent](#registerintent) and the chain-listener has confirmed the on-chain
    registration), **or**
  - **V1** — `receivingAddresses` contains at least one chain entry.

  Either path satisfies the prerequisite. Returns `400` with the message *"Activate on-chain (or
  set a receiving address) before generating an API key."* if neither is set, or *"API Key already
  exists. Use the Rotate button to generate a new one."* if a key was previously issued (use
  [RotateApiKey](#rotateapikey) in that case).

### Parameters

None.

### Returns

```json
{
  "apiKey": "bp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx...",
  "message": "Store this key securely. It will not be shown again."
}
```

`apiKey` is the full plaintext key. Copy it immediately — only a short prefix is stored on the
server and shown in subsequent [GetMe](#getme) / [GetCurrentMerchant](#getcurrentmerchant)
responses.

### Example

**Request:**

```bash
curl -X POST https://api.butterpay.io/v1/merchants/me/generate-key \
  -H "X-Api-Key: bp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx..."
```

**Response** (HTTP 200):

```json
{
  "apiKey": "bp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx...",
  "message": "Store this key securely. It will not be shown again."
}
```

---

## RotateApiKey

Replace the current API key with a new one. The old key is invalidated immediately upon rotation.

- **Method**: `POST`
- **Path**: `/v1/merchants/me/rotate-key`
- **Auth**: `Bearer JWT` (Dashboard login — API key is not accepted)
- **Description**: Generates a new API key, invalidates the previous one, and records the rotation
  timestamp. A **180-day cooldown** is enforced between rotations: attempting to rotate before the
  cooldown expires returns `429` with the number of days remaining. Because rotation invalidates the
  current key immediately, this endpoint intentionally requires a JWT (not an API key) to prevent a
  compromised key from being used to rotate itself.

### Parameters

None.

### Returns

```json
{
  "apiKey": "bp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx...",
  "message": "Store this key securely. It cannot be retrieved again."
}
```

`apiKey` is the full plaintext key. The previous key stops working as soon as this response is
returned. Store the new key before closing the session.

### Error responses

| Status | Error | Meaning |
|--------|-------|---------|
| 401 | `JWT required for key rotation (Dashboard login)` | `Authorization: Bearer` header missing. |
| 401 | `Invalid or expired token` | JWT is malformed or has expired. |
| 429 | `API Key can only be rotated once every 180 days. Try again in N days.` | Cooldown not yet elapsed. |

### Example

**Request:**

```bash
curl -X POST https://api.butterpay.io/v1/merchants/me/rotate-key \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

**Response** (HTTP 200):

```json
{
  "apiKey": "bp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx...",
  "message": "Store this key securely. It cannot be retrieved again."
}
```

---

## See also

- [Authentication](../authentication.md)
- [Webhooks](../webhooks.md) — including the `merchant.registered` event fired after the
  chain-listener processes `MerchantRegistered`.
- [Invoices API](invoices.md)
