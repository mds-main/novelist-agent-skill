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

Call `GET /catalog` first: it lists the options that are live right now (for example whether World Class and XL are on sale) and the current price of every combination.

## What Decides the Price

| Choice | Field | Effect on price |
| --- | --- | --- |
| Bookstore publishing | `publish_to_bookstore` | `true` is cheaper: the novel may appear in the public bookstore. `false` keeps it private to you. |
| Quality tier | `quality_tier` | `world_class` costs a multiple of `pro` (premium models and configuration). |
| Length | `novel_size` | `xl` adds a surcharge. `s`, `m` and `l` cost the same. |
| Delivery format | `output_format` | `both` (EPUB + PDF) adds a small fixed surcharge. `epub` and `pdf` cost the same. |

Prices are dynamic. For x402 the `payment-required` header of the `402` response is authoritative. For prepaid credits the server checks the full price against the account balance.

## Request Body

Required fields:

| Field | Type | Limits |
| --- | --- | --- |
| `title` | string | 1-200 characters |
| `synopsis` | string | 10-5000 characters |

Optional fields:

| Field | Type | Default | Values |
| --- | --- | --- | --- |
| `language` | string | `en` | `en`, `es`, `fr`, `de`, `it`, `pt`, `nl`, `ja`, `ko`, `zh`, `ar`, `hi`, `id`, `ca`, `eu` |
| `quality_tier` | string | `pro` | `pro`, `world_class` |
| `novel_size` | string | `l` | `s`, `m`, `l`, `xl` |
| `image_style` | string | `auto` | `auto`, `oil_painting`, `cinematic_realism`, `storybook_illustration`, `watercolor`, `animated_film`, `anime`, `impressionist`, `cubist` |
| `output_format` | string | `epub` | `epub`, `pdf`, `both` |
| `publish_to_bookstore` | boolean | `false` | |
| `content_rating` | string | `PG-13` | `G`, `PG`, `PG-13`, `R` |
| `genres` | string[] | `["Fiction"]` | up to 5 |
| `themes` | string[] | none | up to 5 |
| `tone` | string[] | none | up to 5 |
| `target_audience` | string | none | up to 100 characters |
| `setting` | string | none | up to 500 characters |

Field notes:

- `novel_size` controls length. `l` is the standard full-length novel; `s` and `m` are shorter, `xl` is the longest. XL is not offered for a tier that writes the whole novel in a single pass: `GET /catalog` shows `xl_available` per tier, and requesting it anyway returns `400 novel_size_unavailable_for_tier`.
- `image_style` sets one consistent artwork style for the cover and the part-opener illustrations. `auto` lets the illustrator choose from the story.
- `output_format`: `pdf` and `both` also produce the print-ready files needed to order a physical copy later (see `PRINT.md`). An `epub`-only book cannot be printed.
- `chapter_count` and `words_per_chapter` are deprecated. They are still accepted so older clients keep working, but they are ignored: use `novel_size`.
- Unknown values are rejected with `400` and a short code (`invalid_language`, `invalid_novel_size`, `invalid_image_style`, `invalid_output_format`), including on the free `402` quote, so check the quote before signing.

Example body:

```json
{
  "title": "The Quantum Garden",
  "language": "en",
  "synopsis": "A botanist inherits a mysterious garden where plants exist in quantum superposition.",
  "genres": ["Science Fiction", "Fantasy"],
  "themes": ["Nature", "Heritage"],
  "setting": "A walled Victorian garden in present-day Cornwall",
  "content_rating": "PG",
  "quality_tier": "world_class",
  "novel_size": "l",
  "image_style": "watercolor",
  "output_format": "both",
  "publish_to_bookstore": false
}
```

## Payment and Submission Flow

### x402

1. Submit `POST /generate` with the full body and no payment header.
2. Expect `402 Payment Required`. The JSON body echoes `product_type`, `quality_tier`, `novel_size`, `output_format` and `price_usdc`; `pricing_options` lists the default-size prices of the other tier and publishing combinations.
3. Decode the `payment-required` header (base64 JSON) and pick an `accepts` option.
4. Sign an EIP-3009 authorization for exactly that option. Generation settles after completion, so the authorization must stay valid for at least 3 hours (`validBefore`).
5. Retry `POST /generate` with the same JSON body and `PAYMENT-SIGNATURE`.
6. Expect `202 Accepted` with `request_id`, `status_url` and `poll_interval_seconds`.

### API key with prepaid credits

1. The user signs up at `https://ainovelist.app`, creates an API key in the dashboard and adds prepaid credits.
2. Submit `POST /generate` with the body and:

```http
Authorization: Bearer <api_key>
Idempotency-Key: <unique-generation-id>
```

3. The API reserves credits for the full price (including the XL and EPUB + PDF surcharges).
4. Expect `202 Accepted`. Credits are captured only when the novel is delivered.
5. With too little balance, expect `402` with `code: insufficient_prepaid_credits`; the user refills in the dashboard.

Queued response shape:

```json
{
  "request_id": "REQUEST_ID",
  "status": "queued",
  "settlement_status": "pending or reserved",
  "payment_model": "deferred_settlement or prepaid_credits",
  "generation": {
    "title": "The Quantum Garden",
    "language": "en",
    "quality_tier": "world_class",
    "novel_size": "l",
    "image_style": "watercolor",
    "output_format": "both",
    "parts": 6,
    "estimated_time_minutes": 60,
    "max_time_minutes": 120
  },
  "status_url": "/agent/v1/status/REQUEST_ID",
  "poll_interval_seconds": 60
}
```

## Polling

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
| `processing` / `generating` | The novel is being written |
| `email_sent` | Final files are being stored; keep polling |
| `completed` | Files are ready |
| `failed` | Generation failed; you are not charged |

## Download

When `completed`, the status response has a `download` object:

| Key | Present when |
| --- | --- |
| `epub_url` | `output_format` is `epub` or `both` |
| `pdf_url` | `output_format` is `pdf` or `both` |
| `expires_hours` | always |

Download URLs are signed, time-limited and bound to the paying wallet or API key. Use them exactly as returned. If a link expires, call status again for a fresh one.

## Prompting Guidance

Give a specific synopsis: protagonist, goal, conflict, setting, stakes, genre and tone. Avoid one-line generic prompts when the user expects a coherent full-length novel.

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
| `ca` | Catalan |
| `eu` | Basque |
