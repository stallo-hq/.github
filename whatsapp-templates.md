# WhatsApp message templates

WhatsApp only delivers free-form messages within 24 hours of the person's last message to that number. Anything Stallo may send later goes as a **pre-approved template**. Stallo sends the free-form text when the window is open and falls back to the template when it isn't.

Submit each template below in **WhatsApp Manager → Message templates → Create template**:

- **Language:** English (`en`)
- **Name:** exactly as written (lower case, underscores)
- **Body:** the text as written. `{{1}}`, `{{2}}` … are variables, and their order must not change, because the code fills them in that order (`stallo-web/convex/whatsapp/templates.ts`).
- Meta asks for a sample value for each variable. The examples column gives one.

There are two groups, submitted to different WhatsApp accounts:

- **Seller-number templates** are sent from each seller's own connected number to customers and riders. Submit them on **each seller's** WhatsApp Business Account. With Embedded Signup the seller's account sits under Stallo's business portfolio, so the Stallo team can do this in WhatsApp Manager after the seller connects.
- **Stallo-number templates** are sent from Stallo's own number (`WHATSAPP_PLATFORM_PHONE_NUMBER_ID`) to sellers. Submit them once, on Stallo's own account. If that number isn't set, these messages are skipped and sellers see everything on the dashboard instead.

## Seller-number templates (category: Utility)

| Name | Body | Example values |
|---|---|---|
| `stallo_delivery_pin` | Your order {{2}} from {{1}} is on the way with {{3}}. Your delivery code is {{4}}. Give it to the rider only when you have your order. | Ama's Kitchen · #1001 · Yaw · 4821 |
| `stallo_order_delivered` | Your order {{2}} from {{1}} has been delivered. Thank you for buying from us. | Ama's Kitchen · #1001 |
| `stallo_delivery_failed` | We couldn't deliver your order {{2}} from {{1}} today. The seller will contact you to arrange another time. | Ama's Kitchen · #1001 |
| `stallo_order_cancelled` | Your order {{2}} from {{1}} has been cancelled. If you paid, your refund is on its way. | Ama's Kitchen · #1001 |
| `stallo_rider_job` | New delivery for {{1}}: order {{2}}. Customer: {{3}}. Address: {{4}}. Open this page for details and to confirm delivery: {{5}} | Ama's Kitchen · #1001 · Kofi, 020 111 2222 · Madina, near the zongo junction · https://stallo.app/r/abc |

## Stallo-number templates (category: Utility)

| Name | Body | Example values |
|---|---|---|
| `stallo_seller_order_paid` | Order {{1}} is paid: {{2}}, {{3}}. Open Stallo to send it out for delivery. | #1001 · 2 × Jollof with chicken · GH₵ 130.00 |
| `stallo_seller_attention` | Something needs you on Stallo: {{1}} | Not sure what 020 111 2222 meant. Reply to them on WhatsApp. |
| `stallo_seller_daily_summary` | Today at {{1}}: {{2}} orders, {{3}} paid. {{4}} to deliver and {{5}} that need you. Open Stallo for details. | Ama's Kitchen · 12 · GH₵ 1,240.00 · 3 · 1 |

## Changing a template

1. Change the body here and in `convex/whatsapp/templates.ts` (parameter count and comment) in the same pull request.
2. Submit the new version in WhatsApp Manager. Until Meta approves it, sends of that template fail and are retried, then shown to the seller under Needs attention.
