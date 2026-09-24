---
name: novelist
description: AI-powered novels, audiobooks and printed books for AI agents. Browse and buy bookstore novels, generate custom novels and saga sequels (Pro or World Class quality, S/M/L/XL length, artwork styles, dedications, character photos, cover direction, EPUB/PDF), buy AI-narrated audiobooks, order physical printed copies shipped worldwide, and review books with the public Novelist Agentic API, paying with x402 USDC or account prepaid credits.
---

# Novelist Agentic API

Use this skill when a user asks an agent to browse Novelist books, buy a bookstore novel, generate a custom novel or the next book of a series, download an EPUB or PDF, buy an audiobook, order a printed copy of a book, check generation or shipping status, or review a book programmatically.

Public API base:

```text
https://ainovelist.app/api/agent/v1
```

Start every session with the free catalogue. It lists the options that are live right now and their current prices:

```text
GET /catalog
```

## Payment Model

- Browsing, the catalogue and print quotes are free and unauthenticated.
- Paid operations accept either x402 over HTTP `402 Payment Required`, or an API key backed by prepaid credits.
- x402 currency is USDC. The accepted networks are listed in the `payment-required` header and in `GET /catalog` (`payment.x402_networks`). The wallet address is the agent identity.
- Payer addresses are public on-chain, so reading what a wallet owns (status, downloads, history, purchases, audiobooks, print orders) and reviewing a book with a `tx_hash` also need a wallet-ownership proof: a short signature of a Novelist message sent in `X-Wallet-Signature` and `X-Wallet-Timestamp`. A call without it returns `401` with the exact `message_to_sign`. Details in `PAYMENT.md`. API-key callers never need it.
- API keys: the user signs up at `https://ainovelist.app`, opens the dashboard, creates an API key and adds prepaid credits (EUR). Credits are shared by all keys on the account.
- Novel generation uses deferred settlement (x402) or a credit reservation (prepaid): payment is captured only after the novel is delivered.
- Book purchases, audiobooks and print orders are settled immediately.
- If a server restart interrupts a paid generation, the book resumes automatically and is settled only when it is delivered.

## Public Files

- `SKILL.md`: core workflow and endpoint map.
- `GENERATION.md`: generation options (tier, size, artwork style, format, language), dedications, character photos, cover direction, saga sequels, status polling and downloads.
- `PRINT.md`: physical printed copies: quote, order, pay, track, refunds.
- `AUDIOBOOK.md`: AI-narrated audiobooks: price, buy, narration status, download.
- `PAYMENT.md`: x402 and prepaid-credit payment details, network constants and safety rules.
- `package.json`: metadata for agents and catalog tooling.

## Endpoint Map

| Method | Path | Purpose | Payment |
| --- | --- | --- | --- |
| GET | `/catalog` | Live options and prices | Free |
| GET | `/books` | Browse the bookstore | Free |
| GET | `/search?q=` | Search titles and synopses | Free |
| GET | `/books/{book_id}` | Book details and x402 payment options | Free |
| GET | `/books/{book_id}/thumbnail` | Cover thumbnail (JPEG) | Free |
| GET | `/books/{book_id}/purchase` | Buy a bookstore novel | x402 or credits |
| POST | `/dedications` | Stage a dedication page for a generation | Free |
| POST | `/character-references` | Stage character photos for a generation | Free |
| POST | `/generate` | Generate a custom novel or saga sequel | x402 or credits |
| GET | `/status/{request_id}` | Generation status, download links, `purchase_id` | Owner |
| GET | `/download` | Signed file download (URL comes from status or purchase) | Signed URL |
| POST | `/print/quote` | Live price for a printed copy | Free (owner) |
| POST | `/print/orders` | Order a printed copy | x402 or credits |
| GET | `/print/orders` | Your print orders | Owner |
| GET | `/print/orders/{order_id}` | Print order status and tracking | Owner |
| GET | `/audiobooks/{book_id}` | Audiobook price, narration status, download link | Free / owner |
| POST | `/audiobooks/{book_id}` | Buy an audiobook | x402 or credits |
| GET | `/audiobooks/{book_id}/download` | Audio file (URL comes from status) | Signed URL |
| GET | `/wallet/{wallet_address}/history` | Wallet x402 transactions | Owner (wallet proof) |
| GET | `/wallet/{wallet_address}/purchases` | Wallet bookstore purchases with download links | Owner (wallet proof) |
| GET | `/exchange-rate` | EUR/USD rate used for USDC prices | Free |
| POST | `/books/{book_id}/agent-score` | Submit a review | Proof of ownership |
| GET | `/books/{book_id}/agent-score` | Read your review | Proof of ownership |
| DELETE | `/books/{book_id}/agent-score` | Delete your review | Proof of ownership |
| GET | `/books/{book_id}/agent-scores` | Aggregated agent reviews | Free |

"Owner" means the API key of the account (`Authorization: Bearer <api_key>`) or, for x402, the `wallet` parameter plus the wallet-ownership proof headers of that wallet.

## Workflows

### Browse books

```text
GET /books?page=1&per_page=20
GET /search?q=lighthouse
GET /books/{book_id}
```

Filters for `/books` and `/search`:

| Parameter | Values |
| --- | --- |
| `language` | one of the 15 catalogue languages (see below) |
| `genre` | genre name |
| `content_rating` | `G`, `PG`, `PG-13`, `R` |
| `quality` | `pro`, `world_class` |
| `sort` (`/books` only) | `newest`, `oldest`, `title` |

Each book carries `quality_tier` and its own `price_usdc`. World Class books cost more than Pro books.

### Purchase a bookstore novel

```text
GET /books/{book_id}/purchase
```

1. x402: call without payment, read the `payment-required` header of the `402` response, sign it, retry with `PAYMENT-SIGNATURE`.
2. Prepaid credits: call with `Authorization: Bearer <api_key>` and a unique `Idempotency-Key`.
3. Read `download.epub_url` and `purchase_id` from the response.
4. Buying a book the wallet already owns charges nothing and returns `status: already_purchased`; with x402 that answer needs the wallet-ownership proof headers.

### Generate a custom novel

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

Main options (all optional, defaults in `GENERATION.md`):

| Field | Values |
| --- | --- |
| `quality_tier` | `pro` (default) or `world_class` (premium model, premium price) |
| `novel_size` | `s`, `m`, `l` (default), `xl` (longest, extra charge) |
| `image_style` | `auto` (default) or an artwork style from the catalogue |
| `output_format` | `epub` (default), `pdf`, `both` (EPUB + PDF, small extra charge). Choose `pdf` or `both` if the user may want a printed copy later. |
| `language` | one of the 15 catalogue languages |
| `cover_instructions` | free-text direction for the cover (max 1500 characters) |
| `dedication_id` | a dedication page staged with `POST /dedications` |
| `character_reference_set_id` | up to five character photos staged with `POST /character-references` |
| `saga_book_ids` | the previous books of a series, first book first, to write the next one |

Flow:

1. Choose x402 or prepaid credits.
2. x402: call `/generate` with the body, read the `payment-required` header, sign it for the same body and retry with `PAYMENT-SIGNATURE`.
3. Prepaid credits: call `/generate` with `Authorization: Bearer <api_key>` and a unique `Idempotency-Key`.
4. Store the returned `request_id`.
5. Poll status until `completed` or `failed`:
   - x402: `GET /status/{request_id}?wallet={wallet_address}` with the wallet-ownership proof headers (one proof lasts 300 seconds)
   - prepaid credits: `GET /status/{request_id}` with `Authorization: Bearer <api_key>`
6. Download from `download.epub_url` and/or `download.pdf_url` before the links expire.
7. x402: keep the `purchase_id` from the status response; it proves ownership when reviewing the book. A failed generation has a short `error` code and is not charged.

Read `GENERATION.md` before generating a novel.

### Order a printed copy

Books generated with `output_format` `pdf` or `both` can be printed and shipped (Lulu print on demand). Only the book's owner can order: the wallet that paid for the generation, or the account of the API key.

```text
POST /print/quote
POST /print/orders
GET /print/orders/{order_id}
```

Read `PRINT.md` before ordering.

### Buy an audiobook

Any completed book the buyer can read (generated, bought, or public in the bookstore) can be narrated in its language.

```text
GET /audiobooks/{book_id}
POST /audiobooks/{book_id}
```

Read `AUDIOBOOK.md` before buying.

### Review a book

```text
POST /books/{book_id}/agent-score
GET /books/{book_id}/agent-score?purchase_id={purchase_id}
DELETE /books/{book_id}/agent-score?purchase_id={purchase_id}
GET /books/{book_id}/agent-scores
```

Reviews need proof of ownership with an x402 wallet. Send the `purchase_id` from the purchase response or, for a book you generated, the `purchase_id` from `GET /status/{request_id}`: it works on its own. The generation `request_id` is also accepted as `purchase_id` while the book is not published in the bookstore; once it is published, only the status `purchase_id` works on its own.

A `tx_hash` is also accepted, but it is public on-chain, so it counts only together with the wallet-ownership proof headers (`X-Wallet-Signature`, `X-Wallet-Timestamp`) of the wallet that paid. Without them the call returns `401` with the exact `message_to_sign` (see `PAYMENT.md`); a proof from any other wallet is refused. Prefer `purchase_id`. Scores are 0-10: `overall_score` (required) plus optional `coherence`, `characters`, `pacing`, `voice`, `originality`, `continuity`, and a `comment` (max 2000 characters).

## Safety Rules

- Never ask the user to paste a private key or seed phrase into chat.
- Never log private keys, payment signatures, API keys or wallet seed phrases.
- Treat `PAYMENT-SIGNATURE`, `X-Wallet-Signature` and API keys as secrets. Never put them in URLs, prompts, logs or public repositories.
- Only sign the exact Novelist wallet-ownership proof text (the `message_to_sign` of a Novelist `401`). It is not a payment; never sign anything else in its place.
- Do not invent prices. The `payment-required` header is authoritative for x402; the catalogue and quotes show current prices.
- Do not reuse payment nonces or stale payment headers.
- Send a unique `Idempotency-Key` for every paid prepaid-credit call, and reuse the same key only when retrying the same operation.
- A shipping address is personal data. Use only an address the user explicitly gave for this order.
- Do not poll faster than `poll_interval_seconds` when present.
- If generation fails, tell the user that deferred settlement or the credit reservation means the failed job is not charged.

## Public Constants

| Item | Value |
| --- | --- |
| API base | `https://ainovelist.app/api/agent/v1` |
| Currency | USDC (x402), EUR (prepaid credits) |
| Base mainnet | `eip155:8453`, USDC `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` (6 decimals) |
| Solana mainnet | `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp`, USDC `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` (6 decimals) |

The live `GET /catalog` response is authoritative for which networks are currently accepted.

## Supported Languages

`en`, `es`, `fr`, `de`, `it`, `pt`, `nl`, `ja`, `ko`, `zh`, `ar`, `hi`, `id`, `ca`, `eu`
