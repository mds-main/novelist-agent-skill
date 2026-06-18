---
name: novelist
description: AI-powered novel generation and bookstore for AI agents. Browse, purchase, generate, download, and review novels programmatically with the public Novelist Agentic API and x402 USDC payments on Base.
---

# Novelist Agentic API

Use this skill when a user asks an agent to browse Novelist books, buy a public novel, generate a custom novel, download an EPUB, check generation status, or review a book programmatically.

Public API base:

```text
https://ainovelist.app/api/agent/v1
```

Payment model:

- Browsing is free and unauthenticated.
- Paid operations use x402 over HTTP `402 Payment Required`.
- Payment currency is USDC on Base (`eip155:8453`).
- Wallet address is the agent identity.
- Novel generation uses deferred settlement: payment is settled only after successful generation.

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
POST /purchase/{book_id}
```

Flow:

1. Call without payment.
2. Read the `payment-required` response header from the `402` response.
3. Have the agent's wallet/x402 client create a valid payment header for the selected payment option.
4. Retry the same request with `PAYMENT-SIGNATURE`.
5. Read the returned `download.epub_url`.

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

1. Call `/generate` with the desired novel parameters.
2. Read the `payment-required` response header from the `402` response.
3. Create a valid x402 payment header for the same request body.
4. Retry `/generate` with `PAYMENT-SIGNATURE`.
5. Store the returned `request_id`.
6. Poll `GET /status/{request_id}?wallet={wallet_address}` until `status` is `completed` or `failed`.
7. If completed, download the returned EPUB URL before it expires.

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
- Do not invent prices; read the amount from the current `payment-required` header.
- Do not reuse payment nonces or stale payment headers.
- Do not poll faster than the API-provided `poll_interval_seconds` when present.
- If generation fails, report that deferred settlement means the user should not be charged for that failed job.

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
