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
| `id` | the session id the UCP endpoint returned. A quote id only when there is no session |
| `status` | `ready_for_complete` for a session the buyer can complete. `incomplete` only when something the buyer must add, such as shipping, is still unknown: the buyer does not accept it. Never `completed` or `canceled` |
| `currency` | ISO 4217, ideally the intent's `content.currency` |
| `line_items[].item.price` | the unit price in minor units: HK$22.00 is `2200`, ¥500 is `500` |
| `line_items[].quantity` | how many |
| `line_items[].totals` and `totals` | `subtotal` and `total` in minor units. `total` is the amount the buyer will pay, including tax and any shipping already known. A later tax or shipping line makes the buyer refuse |
| `links` | at least the terms of service |
| `policies` | optional return, warranty and other terms, each with a `type` such as `dev.ucp.shopping.policy.return` and a `description`. The buyer shows them to its user |
| `messages` | optional notes. A `warning` with `presentation: "disclosure"` is shown word for word beside what its `path` names (`$.line_items[0]`, or no `path` for the whole order) and never hidden: use it for allergens, safety notices and terms the buyer must see |
| `expires_at` | optional RFC 3339 time; the offer never outlasts it |

Minor units: divide by 100 for display in two-decimal currencies (HKD, USD, EUR), not at all in zero-decimal ones (JPY, KRW).

## Session or quote

- **Live session.** This is the offer a funded buyer can pay. Create it at the UCP endpoint (`POST {endpoint}/checkout-sessions`, no payment) and send what it returns. The buyer reads `GET {endpoint}/checkout-sessions/{id}` and completes that session with the active payment instruction. Changing the price after offering makes the buyer refuse; send a new offer with `--supersedes` instead.
- **Quote.** Only when there is no UCP endpoint. `id` is your own quote id, and each `--item-url` is the URL of that one SKU. The buyer's payment enabler creates the session from those URLs. Its total must equal the quote, so the price at that URL, plus tax and known shipping, must already be in the quote. A page that lists several variants is not enough.

## Discounts and claims

- A discount code the price relies on goes in the checkout's discounts and in `--code`.
- A claim, such as a loyalty tier the price depends on, goes in the checkout's `context.eligibility` and in `--eligibility`.

## Loyalty

When the buyer is one of your members, show them what the membership gives on this order with UCP's loyalty extension. Add `"dev.ucp.common.loyalty": [{ "version": "2026-08-25", "extends": "dev.ucp.shopping.checkout" }]` to `ucp.capabilities`, and a `loyalty` object keyed by the membership's claim:

```json
"loyalty": {
  "com.example.coffee_club_member": {
    "id": "membership_1",
    "name": "Coffee Club",
    "provisional": true,
    "tiers": [{ "id": "gold", "name": "Gold", "benefits": [{ "id": "BEN_001", "description": "10% off every order" }] }],
    "rewards": [{
      "currency": { "name": "Beans", "code": "BNS" },
      "earning_forecast": { "amount": 22, "breakdown": [{ "id": "RULE_1", "amount": 22, "description": "1 bean per dollar", "benefit_id": "BEN_001" }] }
    }]
  }
}
```

- The key is the claim, in reverse-domain form, the same as in `--eligibility`.
- `provisional` is `true` until you have verified the membership in your own system. Add `display_id`, a masked member number such as `****5678`, only after that. Never put a raw member id in any field.
- Loyalty does not change `total`. A member price or free shipping goes in `totals` (`items_discount` or `discount`) and in `discounts.applied` with `eligibility` set to the claim. Points earned go in `rewards` and are never subtracted.
- `intent offer` checks `loyalty` against the UCP schema; `CHECKOUT_INVALID` names the field to fix.

## Fulfillment

Say how and when the order reaches the buyer with UCP's fulfillment extension. Add `"dev.ucp.shopping.fulfillment": [{ "version": "2026-08-25", "extends": "dev.ucp.shopping.checkout" }]` to `ucp.capabilities`, and a `fulfillment` object:

```json
"fulfillment": {
  "methods": [{
    "id": "method_1",
    "type": "shipping",
    "line_item_ids": ["li_1"],
    "selected_destination_id": "dest_area",
    "destinations": [{ "type": "shipping_address", "id": "dest_area", "address_country": "HK" }],
    "groups": [{
      "id": "package_1",
      "line_item_ids": ["li_1"],
      "selected_option_id": "morning",
      "options": [{
        "id": "morning",
        "title": "Morning delivery",
        "description": { "plain": "Delivered between 8:15 and 8:45 each weekday" },
        "earliest_fulfillment_time": "2026-09-28T08:15:00+08:00",
        "latest_fulfillment_time": "2026-10-02T08:45:00+08:00",
        "totals": [{ "type": "total", "amount": 0 }]
      }]
    }]
  }]
}
```

- **Price it for the area.** The intent never says the buyer's address, so you do not know it when you quote. Price the option for the intent's `near` geohash and `country`, and honor that price for any address in that area. The buyer's payment enabler adds the stored address at payment. A destination names the area only: `address_country`, and a region or locality if the price depends on it. Never a street, a name or a phone number.
- **Method `type`.** `shipping` for anything brought to the buyer, `pickup` or `curbside` for collection at your location (a `business_location` destination with its `name`). The buyer matches it to the intent's `fulfillment.type`: `delivery` and `shipping` to `shipping` or `delivery`, `pickup` to `pickup` or `curbside`, `digital` to `digital`, `service` to `service`.
- **Select one option per group** with `selected_option_id`. That option is the offer: the buyer refuses a checkout that later selects another one.
- **Give a delivery estimate.** `earliest_fulfillment_time` and `latest_fulfillment_time` are RFC 3339 times. The buyer checks them against the intent's `fulfillment.window`, and ranks an offer that arrives outside it after the offers that arrive inside it. An option with no estimate is not checked.
- The option's price goes in its own `totals`, and in the checkout's `totals` as a `fulfillment` line when it is not zero. The checkout `total` includes it.

## Payment terms

When part of the payment is due after checkout, such as a deposit now and the balance at check-in, installments, or Net 30, say so with UCP's payment terms extension. Add `"dev.ucp.common.payment.terms": [{ "version": "2026-08-25", "extends": ["dev.ucp.shopping.checkout", "dev.ucp.shopping.order"] }]` to `ucp.capabilities`, and the terms to `payment`:

```json
"payment": {
  "selected_term_id": "pt_deposit",
  "terms": [{
    "id": "pt_deposit",
    "title": "First night now, balance at check-in",
    "schedules": [
      { "id": "s1", "type": "immediate", "description": { "plain": "Due today when you book." }, "amount": 30000 },
      { "id": "s2", "type": "deferred", "description": { "plain": "Due at check-in on November 2, 2026." }, "due_at": "2026-11-02T15:00:00+08:00", "amount": 90000 }
    ]
  }]
}
```

- `selected_term_id` names the term in effect. The selected term's schedule amounts add up to the checkout `total` exactly.
- Each schedule is one payment. Its `description` says on its own when it is due. `immediate` is due at checkout; any other `type`, such as `deferred`, is due later. Add `due_at` when you know the date.
- A finance charge or a saving for paying now is its own line in `totals`: a `fee`, or a negative `discount`, with `display_text`.
- **The selected term is the deal.** The buyer never changes your session, so it cannot choose another term: it accepts the selected one or declines. To offer a different term, send a new offer with `--supersedes`. You may list other terms beside it, but only the selected one is binding.
- The buyer refuses a checkout whose selected term, or its schedules' number, types, due dates or amounts, changed after you offered, and one that carries a `payment_term_changed` warning.
- Terms the buyer must see, such as a deposit that is not refunded, go in a policy whose `applies_to` names the term (`$.payment.terms[0]`), with a disclosure warning whose `code` is that policy's `type`.
- On a funded intent, the whole `total` stays within `funded.cap`, not only what is due today. A buyer whose payment method cannot pay the later schedules declines.
