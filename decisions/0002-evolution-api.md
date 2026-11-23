# ADR 0002: Sellers link WhatsApp through a self-hosted Evolution API, not Meta's Cloud API

- **Status:** accepted
- **Date:** 2026-11-23
- **Supersedes:** [ADR 0001](0001-convex-and-cloud-api.md), point 3 (WhatsApp through Meta's Cloud API). The rest of ADR 0001 stands.

## Context

ADR 0001 chose Meta's official WhatsApp Cloud API. In practice, onboarding sellers through Embedded Signup and sending templates both need Meta business verification, and Stallo doesn't have the business registration documents for that yet. Without it we can't onboard real sellers at all.

We also want to launch with no infrastructure cost.

[Evolution API](https://github.com/evolution-foundation/evolution-api) is open source and supports many sessions on one server. It connects to a seller's own WhatsApp or WhatsApp Business number as a **linked device** (QR code, like WhatsApp Web). We pin `evoapicloud/evolution-api:v2.3.7`. Its request and webhook formats were checked against that version's source code, because parts of its published docs are out of date.

## Decision

1. **Meta's Cloud API is removed completely.** That means Embedded Signup, the Graph API client, Meta webhooks and message templates. Every seller links their own number by scanning a QR code in Settings > WhatsApp. Each seller is one Evolution "instance" (`sellers.whatsappInstance`), with its live state and QR code in `whatsappSessions`.
2. **Self-hosted.** Evolution and its Postgres run from `stallo-web/infra/evolution/docker-compose.yml` on a machine that stays on (a free cloud VM, or a spare computer behind a free Cloudflare Tunnel). Redis is not used.
3. **Webhooks are signed.** Evolution doesn't sign request bodies. Instead, for each session Stallo sets the webhook header `jwt_key`, which makes Evolution send `Authorization: Bearer <JWT>`, signed HS256 with `EVOLUTION_WEBHOOK_SECRET` and valid for 10 minutes. Convex rejects anything else, and rejects everything if the secret isn't set (fail closed).
4. **Safety rules around the linked number.** Linked devices aren't an official channel. Automated use breaks WhatsApp's terms, and WhatsApp bans numbers that look automated or spammy. The seller's own business number is at stake, so `convex/outbound.ts` enforces:
   1. **Inbound only.** Stallo sends WhatsApp messages only to someone who wrote to that seller in the last 24 hours. It never starts a chat, and never sends broadcasts, bulk messages or promotions.
   2. **Human takeover.** A message the seller types on their phone (a `fromMe` webhook that Stallo didn't send) pauses Stallo in that chat for **3 hours** (`SELLER_QUIET_MS`). Customer messages are still recorded, but not answered. Messages Stallo sends are reported by Evolution as `send.message`, not `messages.upsert`. As a second check, any `fromMe` id that matches an `outboundMessages.whatsappMessageId` is ignored.
   3. **Seen, typing, send.** Stallo marks the customer's message read, waits about a second (scheduler), then sends with a short typing indicator of 1.5 to 4 seconds depending on length. Evolution's `sendText` `delay` shows the indicator and holds the request open meanwhile.
   4. **Hourly limit per seller:** 25 messages an hour while the link is under 7 days old, then 60. Messages over the limit wait (`notBefore`); they are never dropped or sent in a burst. Messages to one person always go in order.
   5. **Session guard.** A new QR code can't be requested while a session is starting or showing a code that is under a minute old. Evolution's `QRCODE_LIMIT` caps codes per attempt.
5. **Fixed wording for facts.** Delivery codes, amounts, order numbers and payment links always come from fixed text (`convex/model/copy.ts`). Only the assistant's conversational replies (greetings, questions, answers) are written by the model. No extra AI call is made just to vary wording.
6. **SMS for everyone who hasn't written in.** Riders, the receiver of an order placed by someone else, and the seller's own alerts (order paid, needs attention, daily summary, WhatsApp unlinked) go by SMS through SasuSync. These are all queued as `outboundMessages` with `via: 'sms'`, so they are stored, retried and audited like WhatsApp messages. Messages a customer must get (delivery code, delivered, delivery failed, cancelled) fall back to SMS when the customer hasn't written in the last 24 hours.

## Consequences

- **Ban risk sits with the seller's number.** The rules above lower it but can't remove it, and WhatsApp can change its detection at any time. Sellers are told plainly what Stallo does on their number. If a number is unlinked or banned, the seller gets an SMS and a Needs attention item.
- **We run a server.** If it's down, Stallo can't read or reply. Messages that arrive while it's down are delivered by Evolution when it comes back, and duplicates are ignored. The Postgres volume needs backups; losing it means every seller scans again.
- **Personal data on our server.** Evolution stores new messages (needed to fetch pictures). Contacts, chats and history are switched off. The privacy policy names the WhatsApp linked-device setup.
- **SMS costs** replace template costs: one SMS per rider job, per seller alert and per daily summary.
- **Rider replies:** riders confirm on the rider page. A rider who writes the code to the seller's WhatsApp is still understood, because they wrote first.
- **Moving back to the Cloud API** once Stallo is verified stays possible. The transport sits in `convex/integrations/evolution.ts`, `convex/outbound.ts`, `convex/whatsappConnect.ts` and the webhook route. Order logic, the message wording and the `outboundMessages` queue don't depend on it. Record that in a new ADR.
- **Migration:** run `npx convex run admin:clearMetaFields` once per deployment. It empties the Meta fields and asks Meta-connected sellers to link again by QR code.
