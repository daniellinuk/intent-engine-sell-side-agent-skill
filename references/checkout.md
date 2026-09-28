# The checkout an offer carries

An offer carries one UCP checkout object, version `2026-08-25`. `intent offer` checks it against the UCP schema before sending, and the buyer checks it again before paying.

## Template

Five large oat flat whites at HK$22.00 each, HK$110.00 in all:

```json
{
  "ucp": {
    "version": "2026-08-25",
    "capabilities": { "dev.ucp.shopping.checkout": [{ "version": "2026-08-25" }] },
    "payment_handlers": {}
  },
  "id": "quote_2026_0928_001",
  "status": "ready_for_complete",
  "currency": "HKD",
  "line_items": [
    {
      "id": "li_1",
      "item": { "id": "sku_flat_white_oat_l", "title": "Flat white, oat milk, large", "price": 2200 },
      "quantity": 5,
      "totals": [
        { "type": "subtotal", "amount": 11000 },
        { "type": "total", "amount": 11000 }
      ]
    }
  ],
  "totals": [
    { "type": "subtotal", "amount": 11000 },
    { "type": "total", "amount": 11000 }
  ],
  "links": [{ "type": "terms_of_service", "url": "https://shop.example.com/terms" }],
  "expires_at": "2026-09-29T09:00:00Z"
}
```

Send it with an item URL for each line item:

```bash
intent offer --intent <address> --endpoint https://shop.example.com/ucp \
  --checkout checkout.json --item-url li_1=https://shop.example.com/products/flat-white
```

## Fields

| Field | What to put |
| --- | --- |
| `id` | the session id your UCP endpoint returned, or your own quote id when there is no session |
| `status` | `ready_for_complete`, or `incomplete` when the buyer must still add details such as an address. Never `completed` or `canceled`: the buyer refuses them |
| `currency` | ISO 4217, ideally the intent's `content.currency` |
| `line_items[].item.price` | the unit price in minor units: HK$22.00 is `2200`, ¥500 is `500` |
| `line_items[].quantity` | how many |
| `line_items[].totals` and `totals` | `subtotal` and `total` amounts in minor units; the checkout `total` is what the buyer will pay |
| `links` | at least the terms of service |
| `expires_at` | optional RFC 3339 time; the offer never outlasts it |

Minor units: divide by 100 for display in two-decimal currencies (HKD, USD, EUR), not at all in zero-decimal ones (JPY, KRW).

## Session or quote

- **Live session.** Create it at your UCP endpoint (`POST {endpoint}/checkout-sessions`, no payment) and send what it returns. Before paying, the buyer reads `GET {endpoint}/checkout-sessions/{id}` and pays only when it matches the offer. Changing the session's price after offering makes the buyer refuse; send a new offer with `--supersedes` instead.
- **Quote.** With no session, `id` is your own quote id, and the buyer's payment enabler creates the session from the item URLs when it pays. Its checkout must total exactly what the offer says, so the prices on the product pages must match the quote.

## Discounts and claims

- A discount code the price relies on goes in the checkout's discounts and in `--code`.
- A claim, such as a loyalty tier the price depends on, goes in the checkout's `context.eligibility` and in `--eligibility`.
