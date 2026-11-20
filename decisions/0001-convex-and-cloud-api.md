# ADR 0001: Convex for data and integrations, WhatsApp Cloud API, Paystack only

- **Status:** accepted
- **Date:** 2026-11-20

## Context

The first design had a FastAPI service (`stallo-api`) with PostgreSQL, SQLAlchemy and Alembic, RabbitMQ with worker processes, an unofficial WhatsApp linked-device bridge (Evolution API / WAHA, connected by QR code), and both Paystack and Hubtel for payments.

For the MVP that is a lot to run, deploy and keep secure for a small team. The linked-device bridge also breaks WhatsApp's terms and can get a seller's number banned, which is the one thing a seller can't afford.

## Decision

1. **Convex holds all data and runs all server code.** Schema, auth (Convex Auth, phone number + password, verified by SMS through SasuSync), dashboard queries and mutations, the WhatsApp and Paystack webhooks (HTTP actions), calls to outside APIs (actions), retries and the daily summary (scheduler and crons) all live in `stallo-web/convex/`.
2. **The FastAPI service is not used by the MVP.** A planned split (FastAPI for integrations, Convex for data) was dropped during the build: every integration is a few HTTP calls, Convex actions handle them well, and one deployment is simpler than two plus a shared secret between them. `stallo-api` stays as reference and will be archived in its own pull request.
3. **WhatsApp through Meta's official WhatsApp Cloud API.** Sellers connect with Embedded Signup. They can keep their WhatsApp Business app number (coexistence) or use a new number. Messages outside the 24-hour window use approved templates (`whatsapp-templates.md`).
4. **Paystack only.** Each seller gets a Paystack subaccount (MoMo or bank). Each order is one transaction split between the seller's subaccount and Stallo's fee (`transaction_charge`), with Paystack's own fee borne by the subaccount. Hubtel is dropped for now.
5. **No message queue.** RabbitMQ is replaced by storing each inbound WhatsApp message in Convex first (unique by message id, so Meta's retries are ignored), processing it in a scheduled function, and a cron every 2 minutes that retries anything stuck (3 attempts, then Needs attention).
6. **Anthropic Claude for order extraction**, with structured output and a confidence threshold. Prices always come from the seller's product list.

## Consequences

- One repository (`stallo-web`) and one deployment (Vercel running `npx convex deploy`) for the whole product.
- Types are shared end to end through the generated Convex API. There is no OpenAPI contract to keep in step.
- Python skills on the team move to TypeScript for backend work.
- We depend on Convex as a platform. Data can be exported, and the integration code (`convex/integrations/`) is plain `fetch`, so moving it later is contained.
- Holding a seller's payout until the delivery PIN is confirmed (Paystack manual settlement) is **not** built. It needs confirmation from Paystack Ghana and legal advice first.
