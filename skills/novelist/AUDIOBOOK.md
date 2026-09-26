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
- Books listed in the bookstore use the public price, every other book the private price. `GET /audiobooks/{book_id}` shows which one applies to a caller who can see the book.

## 1. Check price and status (free)

```text
GET /audiobooks/{book_id}
```

The response carries `product_type`, `price_eur`, `price_usdc`, `language_supported`, `default_locale`, `locales` and `owned`.

- A book listed in the bookstore: anyone can call it, with no `wallet` and no API key.
- Any other book (one you generated, bought, or hold the audiobook of): identify yourself as shown below. A caller who cannot read the book gets `404 book_not_found`, exactly as for a book that does not exist. For a wallet that can read the book, the `402` challenge of the purchase call (step 2) also quotes the price.

To identify yourself:

- x402 wallet: `GET /audiobooks/{book_id}?wallet={wallet_address}` with the wallet-ownership proof headers `X-Wallet-Signature` and `X-Wallet-Timestamp` (see `PAYMENT.md`). Without a valid proof the call returns `401` with `message_to_sign`, before the book is looked up.
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

For a book that is not in the bookstore (one you generated or bought), send the wallet-ownership proof headers `X-Wallet-Signature` and `X-Wallet-Timestamp` of that wallet (see `PAYMENT.md`) on both calls. Wallet addresses are public, so without the proof the call returns `401` with `message_to_sign`, exactly as it does for a book that does not exist. The challenge's `already_owned` is only `true` for a wallet that sent its proof.

Send one paid request per book at a time. If a paid call times out, do not sign a new payment: check `GET /audiobooks/{book_id}` with the wallet-ownership proof first. A second purchase of the same audiobook by the same wallet waits for the first one and is then answered as `already_owned` (see below) without charging.

If a second payment still settles for an audiobook the wallet already owns, the call returns `200` with `payment.status: duplicate_payment`, `payment.refund: manual` and the `tx_hash`. The audiobook is yours; the extra payment is refunded by hand, so report the `tx_hash` to support and do not pay again.

### API key with prepaid credits

```http
Authorization: Bearer <api_key>
Idempotency-Key: <unique-purchase-id>
```

The price is spent from the credit balance. The audiobook also appears in the user's library on the website.

If you already own the audiobook, the call returns `200` with `payment.status: already_owned`, charges nothing, and restarts the narration if a previous run failed. With x402 this repeat call settles no payment, so it also needs the wallet-ownership proof headers of the paying wallet (otherwise `401` with `message_to_sign`).

## 3. Wait and download

Narration of a full novel takes a while. Poll `GET /audiobooks/{book_id}` (no more than every `poll_interval_seconds`). The `job_status` in a purchase response is a snapshot taken before narration is queued, so read the live status from `GET`.

| `job_status` | Meaning |
| --- | --- |
| `not_started` | The purchase is recorded but narration is not queued yet. The first purchase response usually shows this. If `GET` still shows it after a poll interval, send the purchase request again as the owner (no charge) to start narration |
| `pending` / `running` | Narration in progress; `progress.chunks_done` of `progress.chunks_total` |
| `completed` | Ready: download from `download_url` |
| `failed` | Stopped; `error` is a stable code (for example `tts_api_error` or `no_renderable_text`) that says why. Temporary failures (for example a narration service outage or an interrupted run) are retried automatically in the background a limited number of times, so the status can return to `pending` or `running`; chapters already narrated are kept, so a retry continues where the run stopped. When the automatic retries run out, `error` is `recovery_attempts_exhausted`. For any failure that is not retried automatically, send the purchase request again as the owner (no charge) to restart narration, and if the same `error` comes back, report it |

`download_url` is signed, time-limited and bound to the owner. The file is usually M4A (`audio/mp4`).

## Common Errors

| Status | Code | Meaning |
| --- | --- | --- |
| `400` | `invalid_book_id` | `book_id` is not a valid book id (a UUID) |
| `400` | `idempotency_key_required` | Prepaid credits need an `Idempotency-Key` header of at most 128 characters |
| `400` | `language_not_supported` | The narrator does not support the book's language |
| `401` | `wallet_proof_required`, `wallet_proof_expired`, `wallet_proof_invalid` | Sign `message_to_sign` with the wallet and retry with the proof headers |
| `402` | `insufficient_prepaid_credits` | Refill credits in the dashboard |
| `403` | `audiobook_not_owned` | The download link does not belong to an owner |
| `404` | `book_not_found` | No such book, or a book that is not in the bookstore and that you cannot read (identify yourself first: API key, or wallet with its proof) |
| `404` | `audiobook_not_generated` | Narration is not finished yet |
| `409` | `book_not_completed` | The book is still being written |
| `409` | `audiobook_payment_in_progress` | This `Idempotency-Key` was already used for a prepaid purchase that is still being processed or was refunded. Check `GET /audiobooks/{book_id}`; if you do not own the audiobook, retry with a new key |
| `500` | `audiobook_entitlement_failed` | The purchase could not be recorded. With prepaid credits the spend is refunded: retry with a new `Idempotency-Key`, and report it if the balance was not restored. With x402 the payment has already settled and the error carries `tx_hash`: do not pay again, report the `tx_hash` |
| `503` | `audiobooks_unavailable` | Audiobooks are not offered right now |
| `503` | `price_unavailable` | The price cannot be quoted right now; retry later |
