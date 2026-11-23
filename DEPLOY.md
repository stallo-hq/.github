# Deploying Stallo to production

Stallo runs as three hosted pieces:

| Piece | Where | What it is |
|---|---|---|
| Website and dashboard | Vercel | The `stallo-web` repo (React) |
| Backend: data, WhatsApp logic, payments, AI, SMS | Convex (production deployment) | The `convex/` folder in `stallo-web` |
| WhatsApp gateway (sellers' linked numbers) | A machine you run (free VM or spare computer) | Evolution API + Postgres, `stallo-web/infra/evolution/` ([ADR 0002](decisions/0002-evolution-api.md)) |

The FastAPI service (`stallo-api`, Render) is not used by the current product. Skip it.

Production is completely separate from development: its own database, its own settings, its own webhook addresses. Nothing from dev (sellers, products, orders) carries over. Sellers sign up again in production.

Never put a secret in a chat, a commit or a screenshot. Secrets go only into the Convex and Vercel dashboards.

---

## 1. Before you start

- [ ] **Paystack:** your business is activated for live payments (Paystack dashboard shows it).
- [ ] **Paystack live keys:** if a live secret key was ever pasted anywhere outside the dashboard, generate new ones first: Paystack → Settings → API Keys & Webhooks.
- [ ] **WhatsApp gateway:** the Evolution API server is running with HTTPS and answers `/instance/fetchInstances` (see `stallo-web/infra/evolution/README.md`). Use a separate server (or at least a separate API key and database) from development.
- [ ] **SasuSync:** SMS credit, and the sender ID `Stallo` approved. SMS now also carries rider job links, seller alerts and the daily summary, so budget for more than sign-up codes.

## 2. Deploy the website and create the production backend

1. Convex dashboard → project **stallo** → switch to **Production** → Settings → **Generate production deploy key**. Copy it.
2. Vercel → Add New → Project → import `stallo-hq/stallo-web`.
   - Framework: Vite (already set by `vercel.json`).
   - Build command, output folder and deep-link rewrites come from `vercel.json`. Don't change them.
   - Environment variable: `CONVEX_DEPLOY_KEY` = the key from step 1 (Production environment only).
3. Deploy. The build runs `npx convex deploy`, which pushes the backend to the production Convex deployment and points the website at it.
4. Note two addresses:
   - **Website:** your Vercel address, e.g. `https://stallo.vercel.app`, or your own domain once added.
   - **Backend:** Convex dashboard → Production → Settings → URL & Deploy Key → the **HTTP Actions URL**, ending in `.convex.site`.

## 3. Production settings (Convex dashboard → Production → Settings → Environment Variables)

Settings take effect immediately; no redeploy needed.

### Sign-in

| Name | Value |
|---|---|
| `SITE_URL` | Your website address, no trailing slash. Also where Paystack sends customers after paying. |
| `JWT_PRIVATE_KEY` | Run `node scripts/auth-keys.mjs > auth-keys.txt` in `stallo-web`, copy the first value, then delete `auth-keys.txt`. |
| `JWKS` | The second value from the same file. Production needs its own pair; never reuse dev's. |

### SMS (SasuSync)

| Name | Value |
|---|---|
| `SASUSYNC_API_KEY` | SasuSync portal → API keys. Consider a separate key for production. |
| `SASUSYNC_SENDER_ID` | `Stallo` |
| `SMS_TEST_MODE` | **Do not set** in production. |
| `SMS_HOURLY_LIMIT` | Optional. Codes per phone per hour. |

### Payments (Paystack, live)

| Name | Value |
|---|---|
| `PAYSTACK_SECRET_KEY` | The **live** secret key (`sk_live_…`). The public key is not needed. |
| `PAYSTACK_WEBHOOK_SECRET` | Optional. Only if Paystack gives you a separate webhook signing secret. |
| `PAYSTACK_CUSTOMER_EMAIL_DOMAIN` | Optional. Domain for the placeholder customer emails Paystack requires. |
| `PLATFORM_FEE_PERCENT` | `1.5` |
| `PLATFORM_FEE_CAP_PESEWAS` | `1500` (GH₵ 15 most per order) |
| `PLATFORM_FEE_FLAT_PESEWAS` | Optional, e.g. `0`. |

### WhatsApp (Evolution API)

| Name | Value |
|---|---|
| `EVOLUTION_API_URL` | The gateway's public HTTPS address, e.g. `https://wa.example.com` (`SERVER_URL` in its `.env`). |
| `EVOLUTION_API_KEY` | `AUTHENTICATION_API_KEY` from the gateway's `.env`. |
| `EVOLUTION_WEBHOOK_SECRET` | A new random value, 32+ characters (`openssl rand -hex 32`). Never reuse dev's. |

Remove any old `WHATSAPP_*` and `META_*` variables; nothing reads them any more.

### AI

| Name | Value |
|---|---|
| `GROQ_API_KEY` | Groq console → API keys. Used when no Anthropic key is set. |
| `ANTHROPIC_API_KEY` | Optional. If set, Claude is used instead of Groq. |
| `LLM_MODEL`, `GROQ_MODEL`, `GROQ_VISION_MODEL`, `LLM_CONFIDENCE_THRESHOLD` | Optional. Defaults are fine. |

## 4. Webhooks

Replace `<backend>` with the production `.convex.site` address from step 2.

| Service | Where | Address |
|---|---|---|
| Paystack | Settings → API Keys & Webhooks → **Live Webhook URL** | `https://<backend>/paystack/webhook` |
| Evolution API | Nothing to paste | Stallo registers `https://<backend>/evolution/webhook` on each seller's session itself, signed with `EVOLUTION_WEBHOOK_SECRET` |

Leave Paystack's **Test** Webhook URL pointing at the dev backend.

If you change `EVOLUTION_WEBHOOK_SECRET` or the backend address later, run `npx convex run whatsappConnect:refreshWebhooks --prod` in `stallo-web`.

If the production deployment ever had Meta-connected sellers, run `npx convex run admin:clearMetaFields --prod` once.

## 5. First run

1. Open the website, sign up with your phone number (a real SMS arrives), create the business.
2. Settings → WhatsApp: Show QR code, and scan it from the phone (WhatsApp → Settings → Linked devices). Settings → Payouts: add the bank or mobile money account. This creates a **live** Paystack subaccount; the ones made on dev were test-only.
3. Settings → Delivery and payment, Delivery areas. Add products.
4. From another phone, message the seller's number first, then order something cheap and pay GH₵ 1–2 for real. Check the reply shows "typing…" before it arrives. Check the order turns **Paid** and Stallo's fee shows on the order.
5. Refund it from the order page. Check the refund arrives.
6. Dispatch a test order. Check the rider gets the job by SMS, and confirm the delivery code on the rider page.
7. Reply to the test customer yourself from the seller's phone, then message again from the customer: Stallo should stay quiet for 3 hours (or until "Let Stallo reply now").

## 6. After launch

- Watch Convex → Production → Logs for errors in the first days.
- Needs attention in the dashboard shows anything the system couldn't handle, including a phone that got unlinked.
- Keep the gateway machine on and back up its Postgres volume. If it stops, Stallo can't read or reply until it's back.
- To change the fee, change `PLATFORM_FEE_PERCENT` / `PLATFORM_FEE_CAP_PESEWAS`; no deploy needed.
- New code reaches production when `main` is pushed (Vercel redeploys and pushes the backend).
