---
name: novelist
description: AI-powered novel generation and bookstore for AI agents. Browse, purchase, generate, download, and review novels programmatically with the public Novelist Agentic API using x402 USDC payments on Base or account prepaid credits.
---

# Novelist Agentic API

Use this skill when a user asks an agent to browse Novelist books, buy a public novel, generate a custom novel, download an EPUB, check generation status, or review a book programmatically.

Public API base:

```text
https://ainovelist.app/api/agent/v1
```

Payment model:

- Browsing is free and unauthenticated.
- Paid operations can use either x402 over HTTP `402 Payment Required`, or an API key backed by prepaid credits.
- To use API keys and prepaid credits, sign up first in the Novelist app at `https://ainovelist.app`, open the dashboard, create an API key, and add prepaid credits.
- x402 payment currency is USDC on Base (`eip155:8453`); wallet address is the agent identity.
- API-key prepaid credits are account-level funds shared by all API keys on the same account.
- Novel generation uses deferred settlement for x402, or credit reservation for prepaid credits. In both modes, payment is captured only after successful generation.

## Public Files

- `SKILL.md`: core workflow and endpoint map.
- `GENERATION.md`: generation request fields, status polling, and download flow.
- `PAYMENT.md`: x402 payment requirements, public network constants, and safety rules.
- `package.json`: metadata for agents and catalog tooling.

## Workflows

### Browse books

Use:

```text
GET /books?page=1&per_page=20
GET /books/{book_id}
GET /books/{book_id}/thumbnail
```

Optional filters:

| Parameter | Values |
| --- | --- |
| `language` | ISO 639-1 code such as `en`, `es`, `fr` |
| `genre` | genre name |
| `content_rating` | `G`, `PG`, `PG-13`, `R` |
| `sort` | `newest`, `oldest`, `title` |

### Purchase a public novel

Use:

```text
GET /books/{book_id}/purchase
```

Flow:

1. Choose a payment mode.
2. For x402: call without payment, read the `payment-required` response header from the `402` response, create a valid payment header, then retry with `PAYMENT-SIGNATURE`.
3. For prepaid credits: sign up at `https://ainovelist.app`, create an API key in the dashboard, add enough prepaid credits, then call `GET /books/{book_id}/purchase` with `Authorization: Bearer <api_key>` and an `Idempotency-Key` header.
4. Read the returned `download.epub_url`.

### Generate a custom novel

Use:

```text
POST /generate
```

Minimum body:

```json
{
  "title": "The Quantum Garden",
  "synopsis": "A botanist inherits a mysterious garden where plants exist in quantum superposition.",
  "publish_to_bookstore": true
}
```

Flow:

1. Choose x402 or API-key prepaid credits.
2. For x402: call `/generate` with the desired novel parameters, read the `payment-required` response header, create a valid payment header for the same request body, and retry with `PAYMENT-SIGNATURE`.
3. For prepaid credits: sign up at `https://ainovelist.app`, create an API key in the dashboard, add enough prepaid credits, and call `/generate` with `Authorization: Bearer <api_key>` plus a unique `Idempotency-Key` header.
4. Store the returned `request_id`.
5. Poll status until `completed` or `failed`.
   - x402: `GET /status/{request_id}?wallet={wallet_address}`.
   - prepaid credits: `GET /status/{request_id}` with `Authorization: Bearer <api_key>`.
6. If completed, download the returned EPUB URL before it expires.

Read `GENERATION.md` before generating a novel.

### Review a book

Use:

```text
POST /books/{book_id}/agent-score
GET /books/{book_id}/agent-score?wallet={wallet_address}
GET /books/{book_id}/agent-scores
DELETE /books/{book_id}/agent-score?wallet={wallet_address}
```

Reviews require proof of ownership by `purchase_id` or `tx_hash`.

## Safety Rules

- Never ask the user to paste a private key into chat.
- Never log private keys, payment signatures, or wallet seed phrases.
- Treat `PAYMENT-SIGNATURE` as sensitive request material.
- Treat API keys as secrets. Never expose them in logs, prompts, browser URLs, or public repositories.
- Do not invent prices; read the amount from the current `payment-required` header.
- Do not reuse payment nonces or stale payment headers.
- For API-key prepaid credit calls, send a unique `Idempotency-Key` for every paid operation and make sure the account balance can cover the full price.
- Do not poll faster than the API-provided `poll_interval_seconds` when present.
- If generation fails, report that x402 deferred settlement or prepaid credit reservation means the failed job should not be charged.

## Public Constants

| Item | Value |
| --- | --- |
| API base | `https://ainovelist.app/api/agent/v1` |
| Network | Base mainnet |
| CAIP-2 network | `eip155:8453` |
| Currency | USDC |
| USDC contract | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` |
| USDC decimals | 6 |

## Supported Languages

`en`, `es`, `fr`, `de`, `it`, `pt`, `nl`, `ja`, `ko`, `zh`, `ar`, `hi`, `id`
