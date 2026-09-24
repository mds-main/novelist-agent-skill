# Novelist Printed Books Guide

Use this reference when a user wants a physical, printed copy of a Novelist book shipped to an address. Books are printed on demand by Lulu and shipped worldwide (a few sanctioned or unreachable countries are excluded).

API base:

```text
https://ainovelist.app/api/agent/v1
```

Check `GET /catalog` first: `print.available` is `false` when printing is not offered, and the section lists the trims and shipping levels.

## Which Books Can Be Printed

- Only the book's owner can order a printed copy:
  - x402: the wallet that paid for the book's generation.
  - API key: any book owned by the key's account (generated through the API or on the website).
- The book must have been generated with `output_format` `pdf` or `both`, and be `completed`. An EPUB-only book returns `409 book_not_print_ready`.
- Bookstore purchases do not grant print rights.

## Options

| Field | Required | Values |
| --- | --- | --- |
| `book_id` | yes | the generation `request_id` |
| `shipping_level` | yes | `MAIL`, `PRIORITY_MAIL`, `GROUND_HD`, `GROUND_BUS`, `GROUND`, `EXPEDITED`, `EXPRESS` (not every level reaches every country) |
| `address` | yes | see below |
| `quantity` | no | 1-10, default 1 |
| `trim_size` | no | `4.25x6.875` (Pocket Book), `5.83x8.27` (A5), `6x9` (US Trade, default). Only trims rendered for the book are orderable; the quote lists them in `available_trims`. |
| `binding_type` | no | `perfect_bound` (paperback, default), `coil_bound`, `case_wrap`, `linen_wrap` |
| `interior_color` | no | `standard_bw`, `premium_bw`, `standard_color`, `premium_color` |
| `paper_type` | no | `60_uncoated_cream`, `60_uncoated_white`, `80_coated_white` |
| `cover_finish` | no | `glossy`, `matte` |
| `contact_email` | x402: yes, API key: no (defaults to the account email) | where order updates are sent |
| `wallet_address` | x402 quote and challenge | the wallet that owns the book |
| `expected_total` | no | the quote `total` you accepted; a different fresh total returns `409 price_drifted` |

Address fields:

| Field | Required | Notes |
| --- | --- | --- |
| `name` | yes | recipient, up to 200 characters |
| `street1` | yes | |
| `street2` | no | |
| `city` | yes | |
| `state_code` | for some countries | required for US, CA, AU, ES, IT, JP, IN, BR, MX and others |
| `postcode` | yes | |
| `country_code` | yes | ISO 3166-1 alpha-2 |
| `phone_number` | yes | 7-20 characters, digits and `+ - ( ) .` only |
| `organization` | no | |
| `is_business` | no | boolean |

Use only an address the user explicitly provided for this order. Order responses never echo the address back.

## 1. Quote (free)

```text
POST /print/quote
```

With an API key send `Authorization: Bearer <api_key>`. With x402 put `wallet_address` in the body.

```json
{
  "book_id": "REQUEST_ID",
  "shipping_level": "MAIL",
  "trim_size": "6x9",
  "wallet_address": "0xYourWallet",
  "address": {
    "name": "Ada Reader",
    "street1": "1 Library Lane",
    "city": "London",
    "postcode": "SW1A 1AA",
    "country_code": "GB",
    "phone_number": "+44 20 7946 0000"
  }
}
```

The response has `quote` (Lulu currency: `line_item_with_markup`, `shipping_cost`, `tax_amount`, `total`, ...), `price_usdc` (what an x402 payment costs), `price_eur_credits` (what a prepaid-credit order costs), `page_count`, `available_trims`, and sometimes `address_suggestion` (a standardised address you may offer the user before paying).

The price is Lulu's printing cost with a markup, plus shipping and tax at cost. It changes with trim, binding, page count, destination and shipping speed.

## 2. Order

```text
POST /print/orders
```

### x402 (USDC, settled immediately)

1. Send the order body (including `wallet_address` and `contact_email`) without payment.
2. Expect `402` with a fresh quote and the `payment-required` header. Only mainnet networks are accepted: test-network USDC never buys a real book.
3. Sign exactly the offered amount and retry the same body with `PAYMENT-SIGNATURE`. The payer must be the wallet that owns the book.
4. Expect `201` with the order. The payment is settled on-chain immediately because the job is sent to the printer.

If the price moved between the challenge and the paid retry, the API refuses before settling: `409 price_drifted` with the new `price_usdc`. Request a new challenge and sign again. A signature for much more than the quote is refused with `400 payment_exceeds_quote`.

### API key with prepaid credits

Send the order body with:

```http
Authorization: Bearer <api_key>
Idempotency-Key: <unique-order-id>
```

The EUR equivalent of the quote is spent from the credit balance and the order is placed. With too little balance, expect `402 insufficient_prepaid_credits`. Retrying with the same `Idempotency-Key` returns the same order and never charges twice.

### Order response

```json
{
  "order_id": "ORDER_ID",
  "status": "submitted",
  "book_title": "The Quantum Garden",
  "trim_size": "6x9",
  "quantity": 1,
  "shipping_level": "MAIL",
  "currency": "USD",
  "total_cost": "20.00",
  "tracking_id": null,
  "status_url": "/agent/v1/print/orders/ORDER_ID",
  "payment": {"model": "x402", "network": "eip155:8453", "tx_hash": "0x...", "amount_usdc": "20.00"}
}
```

## 3. Track

```text
GET /print/orders/{order_id}?wallet={wallet_address}
GET /print/orders?wallet={wallet_address}
```

With a wallet, also send the wallet-ownership proof headers `X-Wallet-Signature` and `X-Wallet-Timestamp` (see `PAYMENT.md`); without them the call returns `401` with `message_to_sign`. With an API key, send `Authorization: Bearer <api_key>` instead of `wallet`.

| Status | Meaning |
| --- | --- |
| `created` | Recorded, payment not yet applied |
| `paid` | Paid, waiting to be sent to the printer |
| `submitting` / `submitted` | Sent to the printer |
| `in_production` | Being printed |
| `shipped` | On its way: `tracking_id`, `tracking_urls`, `carrier_name`, `estimated_arrival_min/max` |
| `delivered` | Delivered |
| `cancelled` / `failed` | Stopped |
| `refunded` | Money returned |

Poll no more than every few hours once shipped; status also updates automatically from the printer.

## Refunds

- If the printer rejects the job, the order is refunded automatically:
  - prepaid credits go back to the credit balance;
  - an x402 payment is sent back in USDC to the wallet that paid, on the same network, when `GET /catalog` shows `print.x402_auto_refunds: true`. The order then shows `status: refunded`, `refund_amount`, `refund_currency: usdc` and the refund transaction in the order history.
- If an automatic x402 refund cannot be completed, the order is flagged for the Novelist team, which refunds it by hand. The user can also contact `support@ainovelist.app` with the `order_id` and the payment `tx_hash`.
- A job already in production cannot be cancelled.

## Common Errors

| Status | Code | Meaning |
| --- | --- | --- |
| `400` | `invalid_shipping_level`, `shipping_level_unavailable_for_country` | Pick another shipping level |
| `400` | `address_not_recognized`, `address_fields_rejected` | Fix the address |
| `400` | `print_render_unavailable_for_trim` | Use one of `available_trims` |
| `400` | `contact_email_required`, `wallet_address_required` | Missing field |
| `401` | `wallet_proof_required`, `wallet_proof_expired`, `wallet_proof_invalid` | Reading orders as a wallet: sign `message_to_sign` and retry with the proof headers |
| `402` | `live_print_payment_required` | Pay on a mainnet network |
| `402` | `insufficient_prepaid_credits` | Refill credits in the dashboard |
| `403` | `forbidden` | You do not own this book |
| `409` | `book_not_print_ready` | Book is not `completed` or was not generated as `pdf`/`both` |
| `409` | `price_drifted` | Re-quote and pay the new price |
| `503` | `print_orders_unavailable` | Printing is not offered right now |
