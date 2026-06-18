# Novelist Payment Reference

Use this reference for paying Novelist Agentic API endpoints. This file documents public protocol expectations only. It does not include private application code, server code, secrets, API keys, or wallet keys.

## Public Constants

| Item | Value |
| --- | --- |
| API base | `https://ainovelist.app/api/agent/v1` |
| Network | Base mainnet |
| CAIP-2 network | `eip155:8453` |
| Currency | USDC |
| USDC contract | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` |
| USDC decimals | 6 |
| Payment header | `PAYMENT-SIGNATURE` |
| Payment challenge header | `payment-required` |

## Payment Modes

Paid agent operations support two modes:

1. x402 USDC payments on Base.
2. API keys backed by prepaid credits.

For API keys and prepaid credits, the user must sign up first at `https://ainovelist.app`, open the dashboard, create an API key, and add enough prepaid credits. Prepaid credits are account-level funds shared by all API keys on the same account.

## x402 Flow

1. Request a paid resource without `PAYMENT-SIGNATURE`.
2. The API responds with HTTP `402 Payment Required`.
3. Read the `payment-required` header.
4. Select the accepted payment option for `eip155:8453`.
5. Use the agent's wallet-side x402/EIP-3009 tooling to create a valid payment payload.
6. Retry the exact same resource with `PAYMENT-SIGNATURE`.

The private key must remain inside the user's wallet or local signing environment. Do not request or store it in the conversation.

## API Key + Prepaid Credits Flow

1. Ask the user to sign up or sign in at `https://ainovelist.app`.
2. The user opens the dashboard, creates an API key, and purchases prepaid credits.
3. Call paid endpoints with:

```http
Authorization: Bearer <api_key>
Idempotency-Key: <unique-operation-id>
```

4. The API checks the account-level prepaid balance against the full server-side price.
5. If there is enough balance, purchases are charged immediately and generations reserve credits until completion.
6. If there is not enough balance, the API returns HTTP `402` with `code: insufficient_prepaid_credits` and `refill_url: /dashboard`.

Never place API keys in query strings or public logs.

## Payment Requirements Shape

The `payment-required` header describes accepted payment options. The exact amount and recipient can change, so always use the live response.

Expected fields:

```json
{
  "x402Version": 2,
  "accepts": [
    {
      "scheme": "exact",
      "network": "eip155:8453",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "amount": "AMOUNT_IN_USDC_ATOMIC_UNITS",
      "payTo": "PAYMENT_RECIPIENT",
      "extra": {
        "description": "Payment description"
      }
    }
  ]
}
```

## Settlement Models

| Operation | Settlement |
| --- | --- |
| Purchase an existing public novel | Immediate settlement, then download URL. |
| Generate a custom novel | Deferred settlement; authorization is settled only if generation succeeds. |
| API-key prepaid purchase | Immediate prepaid credit spend, then download URL. |
| API-key prepaid generation | Credits are reserved first and captured only if generation succeeds. |

## Paid Endpoints

```text
GET /books/{book_id}/purchase
POST /generate
```

## Read Endpoints

```text
GET /books
GET /books/{book_id}
GET /status/{request_id}?wallet={wallet_address}
GET /wallet/history?wallet={wallet_address}
GET /wallet/purchases?wallet={wallet_address}
```

For prepaid API-key generations, call status with `Authorization: Bearer <api_key>` instead of the `wallet` query parameter.

## Download URLs

Purchased or generated EPUB download URLs are:

- time-limited,
- wallet-bound,
- signed by the server,
- intended to be used as returned by the API.

Do not modify the query parameters.

## Agent Safety Rules

- Never ask the user for a private key, seed phrase, or wallet recovery phrase.
- Never log or reveal payment signatures.
- Never log or reveal API keys.
- Never reuse a nonce or payment header.
- Never invent a price; the API's `payment-required` header is authoritative.
- For prepaid API-key calls, never retry without a stable `Idempotency-Key`.
- Never claim payment succeeded until the API returns success.
- For failed generation, tell the user that x402 deferred settlement or prepaid credit reservation means the failed job should not be charged.

## Common Errors

| Status | Meaning | Agent action |
| --- | --- | --- |
| `400` | Invalid request or invalid payment payload | Fix request fields or regenerate payment. |
| `402` | Payment required or payment failed | Read challenge or report payment failure. |
| `402` with `insufficient_prepaid_credits` | API-key account balance is too low | Ask the user to refill credits in the app dashboard. |
| `403` | Wallet does not have access | Use the wallet that paid/generated. |
| `404` | Resource not found | Check IDs. |
| `429` | Rate limited | Wait and retry later. |
| `500` | Server error | Retry later or report failure. |
