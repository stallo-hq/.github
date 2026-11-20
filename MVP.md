# Stallo MVP

Stallo lets a small seller in Ghana take orders on WhatsApp without typing every reply. Customers write the way they already do. Stallo builds the order, takes payment through Paystack, and helps the seller deliver it.

## What's in

1. Seller signs up with their Ghana mobile number (SMS code) and a password.
2. Seller connects WhatsApp through Meta (official WhatsApp Cloud API): their existing WhatsApp Business number, or a new number.
3. Seller adds products: name, price, stock.
4. Seller adds a payout account (MoMo or bank). Stallo creates a Paystack subaccount for it.
5. Customers order in plain language (English, Twi, Pidgin). Stallo asks for anything missing and asks them to confirm.
6. On confirmation the customer gets a Paystack link (MoMo or card). Stallo's fee is split off automatically; the rest is settled to the seller.
7. When Paystack confirms payment the order is paid, stock goes down, and the seller is told.
8. Seller dispatches with the rider's name and phone. The customer gets a 4-digit delivery code; the rider gets the order and a private page link.
9. Rider enters the code on that page, or sends it on WhatsApp. The order is delivered. Failed deliveries and returns are handled.
10. Needs attention: messages Stallo wasn't sure about, failed payments or refunds, failed deliveries, stock problems, messages that couldn't be sent.
11. Daily summary to the seller at 8 pm.

Not in the MVP: point of sale, rider app, native apps, multiple staff or branches, discounts, marketplace, Hubtel, analytics beyond today's numbers, Instagram or TikTok, holding payouts until delivery.

## Order flow

```
customer message ──► draft ──► awaiting_confirmation ──► awaiting_payment ──► paid ──► dispatched ──► delivered
                                     ▲        │                                              │
                                     └────────┘ (changes)                                    ▼
                                                                                      delivery_failed ──► returned
                                                                                             │
                                                                                             └──► dispatched (try again)

cancelled: from any state before dispatched
```

Payment status is tracked separately: `none → pending → paid | failed`, and `paid → refunded`.

### In chat

| Customer | Stallo |
|---|---|
| "Abeg 2 jollof, bring am Madina" | Lists the order with prices from the product list, the total and the address, and asks them to reply YES |
| "Make it 3" | Updates the order and asks again |
| "Yes" | Sends the Paystack link |
| pays | "Payment received…" and tells the seller |
| (seller dispatches) | "On the way with Yaw. Your delivery code is 4821." |
| (rider enters 4821) | "Delivered. Thank you." |

If Stallo isn't confident it understood, it doesn't change the order. It replies that the seller will get back to them, and puts the message under Needs attention.

## Delivery

- The PIN is 4 digits, stored hashed. The rider gets 5 tries; after that the seller is told.
- The rider page (`/r/<link>`) needs no login. The link is long and random, and it shows the customer's details only while the delivery is open.
- Failed delivery: the seller or the rider marks it, with a reason. The seller can dispatch again or mark it returned (stock goes back).

## Money

- Amounts are integer pesewas everywhere and shown as `GH₵ 12.50`.
- Fee per paid order: `PLATFORM_FEE_PERCENT` of the subtotal plus `PLATFORM_FEE_FLAT_PESEWAS`, never more than the order. It is recorded on the order.
- Refunds go through Paystack's Refund API from the order page.
