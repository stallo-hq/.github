# Messages Stallo sends, and how

Since [ADR 0002](decisions/0002-evolution-api.md) there are **no WhatsApp message templates**. Stallo sends from the seller's own linked WhatsApp, and only to people who wrote to that seller in the last 24 hours. Everyone else gets an SMS through SasuSync.

All wording lives in one place: `stallo-web/convex/model/copy.ts`. Every message goes through `queueMessage` (`convex/model/outbound.ts`), which stores it and then sends it through `convex/outbound.ts`. That file applies the inbound-only rule, the hourly limit, chat order and seen → typing → send. Failures are retried, then shown under Needs attention.

## Channels

| Who | Channel | Why |
|---|---|---|
| Customer, replying to their message | WhatsApp (seller's number) | They wrote first |
| Customer, must-arrive message (delivery code, delivered, delivery failed, cancelled) | WhatsApp if they wrote in the last 24 hours, otherwise SMS | Must arrive, and Stallo never starts a WhatsApp chat |
| Rider (job link) | SMS | Riders haven't written to the seller |
| Rider, replying with the delivery code on WhatsApp | WhatsApp | They wrote first |
| Receiver of an order placed for them | SMS | They may never have written to the seller |
| Seller (order paid, needs attention, daily summary, WhatsApp unlinked) | SMS | Stallo has no WhatsApp number of its own |

## Wording rules

- Facts are always fixed text with values from the database: delivery codes, amounts, order numbers, payment links, addresses.
- The assistant may write conversational replies (greetings, follow-up questions, answers about the shop). Those replies never contain prices or codes it made up; prices come from the product list.
- Short, plain English, sentence case, no exclamation marks, no emoji (CLAUDE.md §6.3).
- SMS to sellers start with "Stallo:" and aim to fit one SMS part (160 characters).

## Seller SMS texts

| Function in `copy.ts` | Example |
|---|---|
| `sellerOrderPaid` | Stallo: order #1001 is paid (GH₵ 130.00). 2 × Jollof with chicken. Open your dashboard to send it out. |
| `sellerAttention` | Stallo: Delivery of order #1001 failed (customer not answering). Send it again or mark it returned. |
| `sellerDailySummary` | Stallo, Ama's Kitchen today: 12 orders, GH₵ 1,240.00 paid, 3 to deliver, 1 need attention. |
| `sellerWhatsAppDisconnected` | Stallo: WhatsApp for Ama's Kitchen was unlinked, so Stallo can't read or reply to orders. Open Settings > WhatsApp to link it again. |

## Changing a message

Edit `convex/model/copy.ts` and update the flow tests that quote it. Nothing needs approval from Meta.
