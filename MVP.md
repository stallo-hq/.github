# Stallo MVP

Stallo lets a small seller in Ghana take orders on WhatsApp without typing every reply. Customers write the way they already do. Stallo builds the order, takes payment through Paystack, and helps the seller deliver it.

## What's in

1. Seller signs up with their Ghana mobile number (SMS code) and a password.
2. Seller links the WhatsApp or WhatsApp Business number they already sell on by scanning a QR code (Linked devices). They keep using WhatsApp on their phone as normal. See [ADR 0002](decisions/0002-evolution-api.md).
3. Seller adds products: name, price, stock.
4. Seller adds a payout account (MoMo or bank). Stallo creates a Paystack subaccount for it.
5. Customers order in plain language (English, Twi, Pidgin). Stallo asks for anything missing and asks them to confirm.
6. On confirmation the customer gets a Paystack link (MoMo or card). Stallo's fee is split off automatically; the rest is settled to the seller.
7. When Paystack confirms payment the order is paid, stock goes down, and the seller gets an SMS.
8. Seller dispatches with the rider's name and phone. The customer gets a 4-digit delivery code (WhatsApp, or SMS if they haven't written in the last 24 hours); the rider gets the order and a private page link by SMS.
9. Rider enters the code on that page, or sends it on WhatsApp. The order is delivered. Failed deliveries and returns are handled.
10. Needs attention: messages Stallo wasn't sure about, failed payments or refunds, failed deliveries, stock problems, messages that couldn't be sent, WhatsApp unlinked.
11. Daily summary to the seller by SMS at 8 pm.

Stallo only ever replies on WhatsApp. It never starts a chat, never broadcasts, and never sends promotions from a seller's number (ADR 0002). Anyone who hasn't written to the seller gets an SMS instead.

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

### When the seller replies by hand

The seller keeps using WhatsApp on their phone. When they type to a customer themselves, the linked device reports it to Stallo (a `fromMe` message Stallo didn't send). Stallo adds it to the chat history and stays quiet in that chat for 3 hours after the seller's last reply, so the customer never gets two different answers. Messages that arrive meanwhile are kept for context but not answered. The order page shows this and has a "Let Stallo reply now" button to end it early. When Stallo answers again, the assistant sees what the seller said.

## Delivery

- The PIN is 4 digits, stored hashed. The rider gets 5 tries; after that the seller is told.
- The rider page (`/r/<link>`) needs no login. The link is long and random, and it shows the customer's details only while the delivery is open.
- Failed delivery: the seller or the rider marks it, with a reason. The seller can dispatch again or mark it returned (stock goes back).

## Money

- Amounts are integer pesewas everywhere and shown as `GH₵ 12.50`.
- Stallo's fee per order: `PLATFORM_FEE_PERCENT` of the goods plus `PLATFORM_FEE_FLAT_PESEWAS`, capped at `PLATFORM_FEE_CAP_PESEWAS` (pilot: 1.5%, cap GH₵ 15). Never charged on the delivery fee. Recorded on the order.
- Delivery fee: set by the seller, added to every order total.
- Cash on delivery (seller switches it on): the customer replies CASH to the summary. Any delivery fee is paid online first; the goods are paid to the rider. The rider ticks "I collected the cash" before the code is accepted; the seller marks "Cash received from rider".
- Fees on cash orders are added to the seller's balance when the cash is collected and taken from later online payments (never more than half of one payment). Cash is not offered while a seller owes GH₵ 50 or more.
- Rider fees are agreed between seller and rider. Stallo only shows the amount, and whether the rider keeps it from the cash; it never pays riders.
- Refunds go through Paystack's Refund API from the order page.
