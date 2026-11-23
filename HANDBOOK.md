# Stallo Engineering Handbook

This handbook sets out how the Stallo project is organized: the repositories, the roles, who owns which areas, and how work goes from an issue to `main`. It applies to every contributor equally. Changes to it go through a pull request like any other change.

---

## 1. Product overview

Stallo is a WhatsApp commerce assistant for small sellers in Ghana. A seller links their own WhatsApp number to Stallo by scanning a QR code (a linked device on Stallo's Evolution API server, [ADR 0002](decisions/0002-evolution-api.md)). Customers order in ordinary chat. Stallo reads those messages, builds the order, sends a Paystack payment link (keeping a per-order fee automatically), coordinates delivery with a 4-digit PIN, and sends the seller a daily summary along with anything that needs their attention.

The MVP scope and flows are in [`MVP.md`](MVP.md). The architecture decision is [ADR 0001](decisions/0001-convex-and-cloud-api.md).

## 2. Repositories

| Repository | Purpose | Stack |
|---|---|---|
| [`stallo-web`](https://github.com/stallo-hq/stallo-web) | Marketing site, seller dashboard, rider page **and the whole backend** (`convex/`: data, auth, webhooks, integrations, scheduled jobs) | React 19, TypeScript, Vite, React Router 7, Radix UI, Convex, Convex Auth |
| [`stallo-api`](https://github.com/stallo-hq/stallo-api) | Not used by the MVP (see ADR 0001). Kept for reference until it is archived. | Python 3.12, FastAPI |
| `.github` | This handbook, ADRs, MVP spec, deploy and message docs | Markdown |

**One backend.** Everything server-side runs as Convex functions in `stallo-web/convex/`: queries and mutations for the dashboard, HTTP actions for the WhatsApp and Paystack webhooks, actions for outside APIs, and crons for retries and the daily summary. The contract between the web app and the backend is the generated Convex API (`convex/_generated/api`), so types are always in step.

## 3. Organization

### 3.1 Leadership

| Position | Responsibilities |
|---|---|
| **Project Lead** | Final decision on cross-cutting technical questions; release approval; org and repository administration (GitHub Owner) |
| **Deputy Lead** | Shares technical leadership with the Project Lead; manages team membership; reviews cross-cutting changes; acts for the Project Lead when they're unavailable |

Both positions belong to the `leads` GitHub team. Each is held by someone who also holds one of the engineering roles below.

### 3.2 Engineering roles

Every area of the codebase has **one owner role**. The owner decides questions within their area, reviews pull requests that touch it (enforced through `CODEOWNERS`), and keeps its documentation current. Ownership doesn't restrict who may contribute: anyone may open a pull request in any area, and the owner reviews it.

| Role | GitHub team | Responsibilities |
|---|---|---|
| **R1 Integrations** | `integrations` | WhatsApp through Evolution API (QR linking, signed webhook, sending and its safety rules, the gateway in `infra/evolution/`); reliable inbound processing and retries; seller accounts |
| **R2 AI & Orchestration** | `ai` | LLM extraction (prompt, output schema, confidence); conversation handling (basket, questions, confirmation) |
| **R3 Commerce Domain** | `commerce` | Convex schema; order state machine; products and stock; delivery and PIN |
| **R4 Payments & Notifications** | `payments` | Paystack subaccounts, split payments, webhook verification, refunds; needs attention; outbound messaging; daily summary |
| **R5 Frontend Architecture** | `frontend` | Frontend structure, routing, sign-up and SMS verification, design system; overview and settings pages |
| **R6 Frontend & Delivery** | `frontend` | Orders, products and rider pages; CI; deployment and environments; end-to-end test plan |

Role assignments are recorded in the GitHub team memberships, not in this document. To see who holds a role, open the team at `github.com/orgs/stallo-hq/teams`.

### 3.3 Module ownership (`stallo-web`)

| Path | Owner |
|---|---|
| `convex/http.ts`, `convex/inbound.ts`, `convex/outbound.ts`, `convex/whatsappConnect.ts`, `convex/integrations/whatsapp.ts`, `convex/whatsapp/` | R1 |
| `convex/integrations/assistant.ts`, `convex/model/conversation.ts`, `convex/model/copy.ts` | R2 |
| `convex/schema.ts`, `convex/model/orders.ts`, `convex/products.ts`, `convex/orders.ts`, `convex/deliveries.ts`, `convex/model/delivery.ts` | R3 |
| `convex/payments.ts`, `convex/payouts.ts`, `convex/integrations/paystack.ts`, `convex/model/attention.ts`, `convex/model/fees.ts`, `convex/summary.ts` | R4 |
| `src/app/`, `src/components/`, `src/features/{landing,auth,overview,settings}/`, `src/styles/` | R5 |
| `src/features/{orders,products,rider,payment}/`, CI, `vercel.json` | R6 |
| `CLAUDE.md`, shared configuration, `.github/` | `leads` |

### 3.4 Interfaces between roles

| Interface | Between | Defined in |
|---|---|---|
| Stored inbound message and its processing states | R1 → R2 | `convex/schema.ts` (`inboundMessages`), `convex/inbound.ts` |
| LLM extraction output (intent, items, address, confidence) | R2 → R3 | `convex/integrations/assistant.ts` (`Extraction`) |
| Order and payment status values and allowed moves | R3 ↔ R4 | `convex/model/orders.ts` |
| Message wording and channels (WhatsApp or SMS) | R1, R3, R4 | `convex/model/copy.ts`, [`whatsapp-messages.md`](whatsapp-messages.md) |
| Dashboard data | R1–R4 → R5, R6 | Generated Convex API (`convex/_generated/api`) |

A change to any file in this table is a **contract change**. Its pull request must tag every affected owner, and it's merged before any code that depends on it.

### 3.5 Decision-making

1. **Within an area:** the owner decides.
2. **Across areas:** the affected owners agree in the issue or pull request. The decision is recorded as an Architecture Decision Record under `docs/decisions/` in the relevant repository.
3. **Unresolved:** escalate to the leads. The Project Lead makes the final call.

## 4. Workflow

### 4.1 Planning
- All work is tracked as **GitHub Issues** on the organization's **GitHub Project** board: `Backlog → Ready → In progress → In review → Done`.
- Every issue has a single assignee, an area label (`area:ai`, `area:payments`, …) and a type (`feat`, `bug`, `chore`).
- Work runs in **one-week sprints**, with planning at the start and a demo at the end.
- An issue estimated at more than 2–3 days of work is split into smaller ones.

### 4.2 Branching
- `main` is always deployable. All changes reach it through a pull request; nobody pushes to it directly.
  > The organization is on GitHub Free, which doesn't enforce branch protection on private repositories. This rule is a team agreement until the organization moves to GitHub Team (or a repository becomes public). At that point a ruleset on `main` will require a pull request, one approval, CODEOWNERS review and passing CI.
- Branch names follow `<type>/<issue#>-<short-name>`, for example `feat/42-order-state-machine` or `fix/57-paystack-signature`.
- Branches stay short-lived. A branch open for more than three days is split up.

### 4.3 Commits
Commits follow [Conventional Commits](https://www.conventionalcommits.org/):

```
feat(orders): add order state machine
fix(payments): verify Paystack signature on raw body
chore(ci): cache pip dependencies
```

### 4.4 Pull requests
- Each pull request links its issue (`Closes #42`) and fills in the template.
- **One approval is required.** CODEOWNERS requests the right reviewer automatically. Contract changes need approval from every affected owner.
- CI (lint, tests and build) must pass before merge.
- Pull requests are **squash-merged**, so the title becomes the commit message and must follow Conventional Commits.
- Reviews are completed within **one working day**.
- Pull requests stay under ~400 changed lines where possible.

### 4.5 Definition of done
- [ ] Merged to `main` with CI passing
- [ ] Tests cover the new logic (`vitest` for `src/lib`, `convex-test` for Convex functions)
- [ ] Documentation updated if setup or behaviour changed
- [ ] Issue closed on the board

## 5. Engineering standards

**Backend (Convex)**
- Public functions stay thin. Rules live in `convex/model/`. Every seller-facing function starts with `requireSeller` / `currentSeller` and checks the caller owns what it touches.
- Every function validates its arguments with `v.*` validators.
- Status changes go through `transition()` / `setPaymentStatus()` in `convex/model/orders.ts` and are never set directly.
- Webhooks verify the signature on the raw body, fail closed when the secret isn't set, store the event, and return quickly. Processing happens in scheduled functions.
- Anything sent to WhatsApp or by SMS goes through `queueMessage` (stored first, retried, failures recorded). Never call Evolution or SasuSync to send a message directly: `convex/outbound.ts` enforces the inbound-only rule, the hourly limit and chat order (ADR 0002).
- Stallo never starts a WhatsApp chat and never sends broadcasts or promotions from a seller's number.
- Every LLM extraction carries a confidence score. Results below `LLM_CONFIDENCE_THRESHOLD` go to Needs attention instead of changing an order. Prices always come from the product list, never from the model.
- Money is integer pesewas, phones are E.164, time zone is Africa/Accra.

**Frontend**
- Code is organized by feature under `src/features/<area>/`. Pages use their feature's hooks, not Convex directly.
- Styling is CSS Modules plus the tokens in `src/styles/tokens.css`. No UI kits.
- `npm run lint`, `npm test` and `npm run build` pass before a pull request is opened.

**Secrets**
- `.env` files are never committed. Every new variable is added to `.env.example` and to [`DEPLOY.md`](DEPLOY.md).
- Real credentials (Paystack, Anthropic, WhatsApp, SasuSync) live only in the Convex and Vercel dashboards and a shared password manager, never in chat or code.

## 6. Environments

| Environment | Source | Deployed by | Notes |
|---|---|---|---|
| Local | any branch | the developer | `npx convex dev` + `npm run dev` in `stallo-web`, each developer on their own Convex dev deployment |
| Production | `main` | Vercel (runs `npx convex deploy`); the Evolution gateway is run by hand (`infra/evolution/`) | Sellers' real numbers, Paystack live key once approved |

## 7. Milestones

| # | Goal |
|---|---|
| M1 | Convex schema, phone + password sign-up with SMS code, marketing site, dashboard shell |
| M2 | Products |
| M3 | WhatsApp webhook → stored message → reply, no duplicates on retries |
| M4 | AI extraction → basket → customer confirms; low confidence goes to Needs attention |
| M5 | Paystack link with Stallo's fee split off → webhook → paid |
| M6 | Dispatch, PIN, rider page, failed delivery and returns |
| M7 | Needs attention, daily summary, retries |
| M8 | Deploy checklists ([`DEPLOY.md`](DEPLOY.md)) |

## 8. Communication

- **Written decisions.** Decisions are recorded in the issue, pull request or an ADR, not only agreed in a call.
- **Daily update** in the team channel: progress, plan, blockers.
- **Blockers are raised right away**, not held until the next meeting.

## 9. Onboarding

1. Accept the organization invitation. A lead adds you to your role's team.
2. Read this handbook and the README of the repository you'll work in.
3. Run `stallo-web` locally: `npx convex dev` in one terminal, `npm run dev` in another (see its README).
4. Get development credentials from the shared password manager.
5. Pick up an issue labelled `good first issue` and open a pull request within the first week.
