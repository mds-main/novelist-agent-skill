# Novelist Payment Reference

Use this reference for paying Novelist Agentic API endpoints. It documents public protocol expectations only. It does not include private application code, server code, secrets, API keys or wallet keys.

## Public Constants

| Item | Value |
| --- | --- |
| API base | `https://ainovelist.app/api/agent/v1` |
| Payment header | `PAYMENT-SIGNATURE` |
| Payment challenge header | `payment-required` |
| x402 version | `2` |
| Scheme | `exact` |

x402 networks (the live list is in `GET /catalog` under `payment.x402_networks`, and in every `payment-required` header):

| Network | CAIP-2 | USDC asset | Decimals |
| --- | --- | --- | --- |
| Base mainnet | `eip155:8453` | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` | 6 |
| Solana mainnet | `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp` | `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` | 6 |

## Payment Modes

Paid agent operations support two modes:

1. x402 USDC payments.
2. API keys backed by prepaid credits (EUR).

For API keys, the user signs up at `https://ainovelist.app`, opens the dashboard, creates an API key and adds prepaid credits. Credits are account-level funds shared by all keys on the account.

## Prices

- `GET /catalog` returns every generation price (tier x publishing x size, novel or saga sequel, plus the EPUB + PDF surcharge), the bookstore prices and the audiobook prices, in EUR and USDC. It is free.
- Physical books are priced per order: `POST /print/quote` (free) returns the live price in USDC and in EUR credits.
- USDC prices are converted from EUR with a daily EUR/USD rate (`GET /exchange-rate`).
- For x402, the amount in the `payment-required` header is authoritative. Never compute or hard-code an amount.

## x402 Flow

1. Request a paid resource without `PAYMENT-SIGNATURE`.
2. The API responds with HTTP `402 Payment Required`.
3. Read and base64-decode the `payment-required` header.
4. Select one option from `accepts` (network you can pay on).
5. Use the wallet-side x402 tooling (EIP-3009 `transferWithAuthorization` on EVM) to sign a payment for exactly that option: same `amount`, `asset` and `payTo`.
6. Retry the exact same request (same path, same body) with `PAYMENT-SIGNATURE`.

The private key stays inside the user's wallet or local signing environment. Never request or store it in the conversation.

## API Key + Prepaid Credits Flow

1. Ask the user to sign up or sign in at `https://ainovelist.app`.
2. The user opens the dashboard, creates an API key and purchases prepaid credits.
3. Call paid endpoints with:

```http
Authorization: Bearer <api_key>
Idempotency-Key: <unique-operation-id>
```

4. The API checks the account balance against the full server-side price.
5. Purchases and print orders are charged immediately; generations reserve credits and capture them only when the novel is delivered.
6. With too little balance the API returns HTTP `402` with `code: insufficient_prepaid_credits`, `required_eur` and `refill_url: /dashboard`.

Never place API keys in query strings or logs.

## Wallet-Ownership Proof (x402 wallets)

Payer addresses are public on the blockchain, so naming a wallet proves nothing. Every x402 call that returns what a wallet owns (download links, generation `request_id`s, `purchase_id`s and other review proofs, audiobook and print-order details) must carry a short signature proving that the caller controls the wallet. API-key callers never need it.

Message to sign (UTF-8, three lines joined by `\n`, no trailing newline):

```text
Novelist Agentic API wallet ownership proof
Wallet: <wallet>
Timestamp: <unix seconds>
```

- `<wallet>` is the address lower-cased for EVM wallets, and exactly as written for Solana wallets.
- `<unix seconds>` is the current time. A proof is accepted for 300 seconds, so one signature can be reused while polling for a few minutes. Timestamps more than 60 seconds in the future are refused.

Send it with the request:

```http
X-Wallet-Signature: <signature>
X-Wallet-Timestamp: <unix seconds>
```

| Wallet | Signature |
| --- | --- |
| EVM (Base) | EIP-191 `personal_sign` of the message, hex (the `0x` prefix is optional). Smart-contract wallets (ERC-1271, for example Coinbase Smart Wallet) are supported. |
| Solana | ed25519 signature of the raw message bytes (`signMessage`), base58 or base64 |

You never have to build the message yourself: a request without a valid proof returns `401` with the exact text to sign for the current server time.

```json
{
  "detail": {
    "error": "wallet_proof_required",
    "wallet": "0xyourwallet",
    "signature_scheme": "eip191",
    "timestamp": 1790000000,
    "message_to_sign": "Novelist Agentic API wallet ownership proof\nWallet: 0xyourwallet\nTimestamp: 1790000000",
    "signature_header": "X-Wallet-Signature",
    "timestamp_header": "X-Wallet-Timestamp",
    "valid_for_seconds": 300
  }
}
```

Sign `message_to_sign` with the wallet and retry the same request with the two headers. Examples:

```ts
// viem (EVM)
const signature = await account.signMessage({ message: detail.message_to_sign })
```

```python
# eth_account (EVM)
from eth_account.messages import encode_defunct
signature = account.sign_message(encode_defunct(text=message_to_sign)).signature.hex()
```

```ts
// Solana (tweetnacl + bs58)
const signature = bs58.encode(nacl.sign.detached(new TextEncoder().encode(detail.message_to_sign), keypair.secretKey))
```

| `error` | Meaning | Agent action |
| --- | --- | --- |
| `wallet_proof_required` | No proof headers | Sign `message_to_sign` and retry |
| `wallet_proof_expired` | Timestamp older than 300 seconds or too far ahead | Sign the fresh `message_to_sign` and retry |
| `wallet_proof_invalid` | The signature is not this wallet's signature of this message | Sign with the wallet named in the request |
| `wallet_proof_malformed` | Unreadable signature, timestamp or address | Fix the headers |

The proof grants read access to what the wallet already owns. It cannot move funds and is not a payment.

## Payment Requirements Shape

```json
{
  "x402Version": 2,
  "accepts": [
    {
      "scheme": "exact",
      "network": "eip155:8453",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "amount": "AMOUNT_IN_USDC_ATOMIC_UNITS",
      "payTo": "PAYMENT_RECIPIENT"
    }
  ]
}
```

## Settlement Models

| Operation | x402 | Prepaid credits |
| --- | --- | --- |
| Buy a bookstore novel (`GET /books/{id}/purchase`) | Settled immediately, then download URL | Spent immediately |
| Generate a novel (`POST /generate`) | Deferred: settled only if generation succeeds. Sign with at least 3 hours of validity. | Reserved, captured only on success |
| Order a printed copy (`POST /print/orders`) | Settled immediately (the job goes to the printer). Mainnet only. Refunded in USDC to the paying wallet if the printer rejects the job (automatic when `print.x402_auto_refunds` is `true` in the catalogue, otherwise by the Novelist team). | Spent immediately, refunded to the balance if the printer rejects the job |
| Buy an audiobook (`POST /audiobooks/{book_id}`) | Settled immediately; narration then starts | Spent immediately |

If a server restart interrupts a paid generation, the book is resumed automatically and its payment is settled (or, if it finally fails, released) once it finishes.

## Paid Endpoints

```text
GET  /books/{book_id}/purchase
POST /generate
POST /print/orders
POST /audiobooks/{book_id}
```

## Owner Endpoints

```text
GET /status/{request_id}?wallet={wallet_address}
GET /wallet/{wallet_address}/history
GET /wallet/{wallet_address}/purchases
GET /print/orders?wallet={wallet_address}
GET /print/orders/{order_id}?wallet={wallet_address}
GET /audiobooks/{book_id}?wallet={wallet_address}
```

With a wallet, send the wallet-ownership proof headers (`X-Wallet-Signature`, `X-Wallet-Timestamp`). With an API key, send `Authorization: Bearer <api_key>` instead of the `wallet` parameter; `/wallet/*` lists x402 wallet activity only.

A paid call answered without a new payment also needs the proof: retrying `GET /books/{book_id}/purchase` for a book the wallet already bought (`already_purchased`), retrying `POST /generate` with a payment that was already submitted (`already_submitted`), and buying an audiobook the wallet already owns (`already_owned`). The first, settled payment proves the wallet by itself.

Reviews (`POST`, `GET` and `DELETE /books/{book_id}/agent-score`) accept a `tx_hash` as proof of ownership only with the proof headers of the wallet that made that payment: a settlement transaction hash or a payment signature can be read by anyone from the chain. Without them the call returns `401` with `message_to_sign`. A `purchase_id` needs no proof headers.

## Read Endpoints

```text
GET /catalog
GET /books
GET /search?q={query}
GET /books/{book_id}
GET /exchange-rate
```

## Download URLs

EPUB, PDF and audiobook download URLs are time-limited, bound to the paying wallet or API key, and signed by the server. Use them exactly as returned; do not edit the query parameters.

## Agent Safety Rules

- Never ask the user for a private key, seed phrase or wallet recovery phrase.
- Never log or reveal payment signatures or API keys.
- Never reuse a nonce or a payment header.
- Never invent a price; the `payment-required` header is authoritative.
- For prepaid calls, never retry without the same stable `Idempotency-Key`.
- Never claim payment succeeded until the API returns success.
- For a failed generation, tell the user that deferred settlement or the credit reservation means they are not charged.
- Never pay for a print order with an address the user did not give you for that order.
- Only sign the exact wallet-ownership proof text shown above (or `message_to_sign` from a Novelist `401`). It is not a payment; never sign anything else in its place.
- Treat `X-Wallet-Signature` like a short-lived credential: never log it or put it in a URL.

## Common Errors

| Status | Meaning | Agent action |
| --- | --- | --- |
| `400` | Invalid request, option or payment payload | Fix the fields or sign a new payment |
| `401` `api_key_required` | A prepaid resource was read without its API key | Send `Authorization: Bearer <api_key>` |
| `401` `wallet_proof_*` | Wallet-ownership proof missing, expired or invalid | Sign `message_to_sign` and retry with the proof headers |
| `402` | Payment required or payment failed | Read the challenge or report the failure |
| `402` `insufficient_prepaid_credits` | Account balance too low | Ask the user to refill credits in the dashboard |
| `402` `live_print_payment_required` | Test-network money for a real print order | Pay on a mainnet network |
| `403` | Wallet or key does not own the resource | Use the wallet or key that paid |
| `404` | Resource not found | Check the IDs |
| `409` `price_drifted` | Print price changed since the challenge | Request a new challenge and sign again |
| `429` | Rate limited | Wait `retry_after` seconds |
| `500` | Server error | Retry later or report the failure |
| `503` | Feature not available right now | Check `GET /catalog` |
