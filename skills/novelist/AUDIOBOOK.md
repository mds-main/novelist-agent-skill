# Novelist Audiobooks Guide

Use this reference when a user wants to listen to a Novelist book: an AI-narrated audiobook of the full text, delivered as an audio file.

API base:

```text
https://ainovelist.app/api/agent/v1
```

Check `GET /catalog` first: `audiobooks.available` is `false` when audiobooks are not offered, and the section lists the narration languages and the two prices.

## Which Books

- Any `completed` book the buyer can read:
  - a book you generated (the paying wallet, or the API key's account),
  - a book you bought from the bookstore,
  - any book published in the bookstore.
- The book's language must be one the narrator supports (`audiobooks.languages` in the catalogue). Otherwise: `400 language_not_supported`.
- Books listed in the bookstore use the public price, every other book the private price. `GET /audiobooks/{book_id}` shows which one applies.

## 1. Check price and status (free)

```text
GET /audiobooks/{book_id}
```

Without `wallet` or an API key the response is the public price sheet: `product_type`, `price_eur`, `price_usdc`, `language_supported`, `default_locale`, `locales` and `owned: false`.

To see your own audiobook, identify yourself:

- x402 wallet: `GET /audiobooks/{book_id}?wallet={wallet_address}` with the wallet-ownership proof headers `X-Wallet-Signature` and `X-Wallet-Timestamp` (see `PAYMENT.md`). Without a valid proof the call returns `401` with `message_to_sign`.
- API key: `Authorization: Bearer <api_key>` instead of `wallet`.

When you own the audiobook the response also has `job_status`, `progress`, `duration_seconds` and, once narration is complete, `download_url`.

## 2. Buy

```text
POST /audiobooks/{book_id}
```

Optional body:

```json
{
  "tts_locale": "es-MX",
  "wallet_address": "0xYourWallet"
}
```

`tts_locale` picks a regional voice where the language has several (for example `es-ES` or `es-MX`, `pt-BR` or `pt-PT`); otherwise the default locale is used.

### x402 (USDC, settled immediately)

1. Send the request with `wallet_address` and no payment. Expect `402` with the `payment-required` header.
2. Sign exactly the offered amount and retry the same request with `PAYMENT-SIGNATURE`. The payer wallet becomes the owner.
3. Expect `201`. The payment is settled immediately and narration starts.

### API key with prepaid credits

```http
Authorization: Bearer <api_key>
Idempotency-Key: <unique-purchase-id>
```

The price is spent from the credit balance. The audiobook also appears in the user's library on the website.

If you already own the audiobook, the call returns `200` with `payment.status: already_owned`, charges nothing, and restarts the narration if a previous run failed. With x402 this repeat call settles no payment, so it also needs the wallet-ownership proof headers of the paying wallet (otherwise `401` with `message_to_sign`).

## 3. Wait and download

Narration of a full novel takes a while. Poll `GET /audiobooks/{book_id}` (no more than every `poll_interval_seconds`):

| `job_status` | Meaning |
| --- | --- |
| `pending` / `running` | Narration in progress; `progress.chunks_done` of `progress.chunks_total` |
| `completed` | Ready: download from `download_url` |
| `failed` | Stopped; `error` explains why. Narration is retried automatically; buying again as the owner restarts it immediately |

`download_url` is signed, time-limited and bound to the owner. The file is usually M4A (`audio/mp4`).

## Common Errors

| Status | Code | Meaning |
| --- | --- | --- |
| `400` | `language_not_supported` | The narrator does not support the book's language |
| `401` | `wallet_proof_required`, `wallet_proof_expired`, `wallet_proof_invalid` | Sign `message_to_sign` with the wallet and retry with the proof headers |
| `402` | `insufficient_prepaid_credits` | Refill credits in the dashboard |
| `403` | `forbidden` | You cannot read this book |
| `403` | `audiobook_not_owned` | The download link does not belong to an owner |
| `404` | `audiobook_not_generated` | Narration is not finished yet |
| `409` | `book_not_completed` | The book is still being written |
| `503` | `audiobooks_unavailable` | Audiobooks are not offered right now |
