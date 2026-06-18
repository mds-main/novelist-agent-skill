# Novelist Generation Guide

Use this reference when creating custom novels through the Novelist Agentic API.

API base:

```text
https://ainovelist.app/api/agent/v1
```

Endpoint:

```text
POST /generate
```

## Generation Types

| Type | Field | Result |
| --- | --- | --- |
| Public | `"publish_to_bookstore": true` | Lower price; generated novel may appear in the bookstore. |
| Private | `"publish_to_bookstore": false` | Higher price; only the paying wallet can access it. |

Prices are dynamic. For x402, always read the current amount from the `payment-required` header returned by the API. For API-key prepaid credits, the server validates the full price against the account credit balance.

## Request Body

Required fields:

| Field | Type | Limits |
| --- | --- | --- |
| `title` | string | 1-200 characters |
| `synopsis` | string | 10-5000 characters |

Optional fields:

| Field | Type | Default |
| --- | --- | --- |
| `language` | string | `en` |
| `genres` | string[] | `["Fiction"]` |
| `themes` | string[] | none |
| `target_audience` | string | none |
| `content_rating` | string | `PG-13` |
| `tone` | string[] | none |
| `setting` | string | none |
| `chapter_count` | integer | `8` |
| `words_per_chapter` | integer | `2000` |
| `publish_to_bookstore` | boolean | `false` |

Allowed ranges:

| Field | Range |
| --- | --- |
| `chapter_count` | 3-20 |
| `words_per_chapter` | 1000-5000 |
| `content_rating` | `G`, `PG`, `PG-13`, `R` |

Example body:

```json
{
  "title": "The Quantum Garden",
  "language": "en",
  "synopsis": "A botanist inherits a mysterious garden where plants exist in quantum superposition.",
  "genres": ["Science Fiction", "Fantasy"],
  "themes": ["Nature", "Heritage", "Quantum Physics"],
  "content_rating": "PG",
  "chapter_count": 8,
  "words_per_chapter": 2000,
  "publish_to_bookstore": true
}
```

## Payment and Submission Flow

Choose one payment mode:

### x402

1. Submit `POST /generate` with the desired request body and no payment header.
2. Expect `402 Payment Required`.
3. Decode/read the payment requirements from the `payment-required` header.
4. Use a wallet-side x402/EIP-3009 implementation to create the `PAYMENT-SIGNATURE` header for the exact same resource and body.
5. Retry `POST /generate` with the same JSON body and the payment header.
6. Expect `202 Accepted` with `request_id`, `status_url`, and `poll_interval_seconds`.

### API key with prepaid credits

1. The user signs up first at `https://ainovelist.app`.
2. In the dashboard, the user creates an API key and purchases prepaid credits.
3. Submit `POST /generate` with the desired request body and:

```http
Authorization: Bearer <api_key>
Idempotency-Key: <unique-generation-id>
```

4. The API reserves prepaid credits for the full server-side price.
5. Expect `202 Accepted` with `request_id`, `status_url`, and `poll_interval_seconds`.
6. If the account does not have enough funds, expect HTTP `402` with `code: insufficient_prepaid_credits` and refill in the dashboard.

Successful queued response shape:

```json
{
  "request_id": "REQUEST_ID",
  "status": "queued",
  "settlement_status": "pending or reserved",
  "payment_model": "deferred_settlement or prepaid_credits",
  "status_url": "/agent/v1/status/REQUEST_ID",
  "poll_interval_seconds": 60
}
```

## Polling

Use:

```text
GET /status/{request_id}?wallet={wallet_address}
```

For prepaid API-key generations:

```text
GET /status/{request_id}
Authorization: Bearer <api_key>
```

Status values:

| Status | Meaning |
| --- | --- |
| `queued` | Waiting to start |
| `processing` | Novel is being generated |
| `completed` | EPUB is ready |
| `failed` | Generation failed; deferred payment should not be charged |

When completed, the status response includes a `download.epub_url` value.

## Download

Download URLs are time-limited and wallet-bound. If a URL expires, request status again with the same wallet to get a fresh URL if available.

The final artifact is an EPUB file containing the generated novel.

## Prompting Guidance

Give the API a specific synopsis. Include concrete protagonist, conflict, setting, stakes, genre, and tone. Avoid one-sentence generic prompts when the user expects a coherent long-form novel.

Good synopsis pattern:

```text
{Protagonist} wants {goal}, but {conflict}. The story is set in {setting}, combines {genres}, and should emphasize {themes/tone}.
```

## Supported Languages

| Code | Language |
| --- | --- |
| `en` | English |
| `es` | Spanish |
| `fr` | French |
| `de` | German |
| `it` | Italian |
| `pt` | Portuguese |
| `nl` | Dutch |
| `ja` | Japanese |
| `ko` | Korean |
| `zh` | Chinese |
| `ar` | Arabic |
| `hi` | Hindi |
| `id` | Indonesian |
