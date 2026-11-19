# Stallo Engineering Handbook

This handbook sets out how the Stallo project is organized: the repositories, the roles, who owns which areas, and how work goes from an issue to `main`. It applies to every contributor equally. Changes to it go through a pull request like any other change.

---

## 1. Product overview

Stallo is a WhatsApp commerce assistant for small sellers. A seller links a WhatsApp account by scanning a QR code. Customers order in ordinary chat. Stallo interprets those messages, creates orders, collects payment through Paystack or Hubtel, tracks delivery, and sends the seller a daily summary along with any items that need attention.

## 2. Repositories

| Repository | Purpose | Stack |
|---|---|---|
| [`stallo-api`](https://github.com/stallo-hq/stallo-api) | Webhooks, orchestration, domain services, workers, database | Python 3.12, FastAPI, PostgreSQL, RabbitMQ, Alembic |
| [`stallo-web`](https://github.com/stallo-hq/stallo-web) | Seller dashboard | React, TypeScript, Vite, TanStack Query |
| `.github` | This handbook and shared templates | Markdown |

**Separate frontend and backend repositories.** Each side ships on its own schedule and has its own CI and reviewers, which keeps pull requests focused. The **contract between them is the OpenAPI schema** that `stallo-api` publishes at `/openapi.json`. `stallo-web` generates its types from that schema.

**One backend application.** Each component in the architecture diagram is a **module** inside `stallo-api`, not a separate service. It deploys as one API process and two worker processes. A module is split out only if it later needs to scale on its own.

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
| **R1 Integrations** | `integrations` | WhatsApp linked-device integration (Evolution API / WAHA); webhook receivers; RabbitMQ topology and queue contracts; seller accounts and linked sessions; local development stack |
| **R2 AI & Orchestration** | `ai` | LLM extraction (prompts, output schemas, confidence scoring); parser/orchestrator routing; conversation manager (state, pending questions, order sessions) |
| **R3 Commerce Domain** | `commerce` | Database schema and migrations; order service and state machine; inventory; delivery |
| **R4 Payments & Notifications** | `payments` | Paystack/Hubtel integration and webhook verification; payment service (payment status tracked separately from order status); exception manager; paced outbound messaging; seller alerts; daily summary |
| **R5 Frontend Architecture** | `frontend` | Frontend structure, routing, authentication, API client; dashboard, payments and settings pages (including WhatsApp QR linking) |
| **R6 Frontend & Delivery** | `frontend` | Orders, inventory and "needs attention" pages; CI pipelines; deployment and environments; end-to-end test plan |

Role assignments are recorded in the GitHub team memberships, not in this document. To see who holds a role, open the team at `github.com/orgs/stallo-hq/teams`.

### 3.3 Backend module ownership

| `stallo-api` path | Owner |
|---|---|
| `app/modules/ingestion/`, `app/modules/sellers/`, `app/messaging/` | R1 |
| `app/modules/orchestrator/`, `app/modules/conversations/` | R2 |
| `app/modules/orders/`, `app/modules/inventory/`, `app/modules/delivery/`, `alembic/` | R3 |
| `app/modules/payments/`, `app/modules/exceptions/`, `app/modules/notifications/` | R4 |
| Shared configuration, CI and `.github/` | `leads` |

### 3.4 Interfaces between roles

| Interface | Between | Defined in |
|---|---|---|
| Normalized inbound message | R1 → R2 | `ingestion/schemas.py` |
| Queue and event names, event payloads | R1 and all consumers | `app/messaging/queues.py` |
| LLM extraction output (intent, items, quantities, confidence) | R2 → R3 | `orchestrator/schemas.py` |
| Order and payment status values | R3 ↔ R4 | `orders/state_machine.py`, `payments/schemas.py` |
| Dashboard REST API | R1–R4 → R5, R6 | OpenAPI (`/openapi.json`) |

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
- [ ] Tests cover the new logic (backend: pytest; frontend: type-safe and manually verified)
- [ ] Migration included if the schema changed
- [ ] API types regenerated in `stallo-web` if the API changed
- [ ] Documentation updated if setup or behaviour changed
- [ ] Issue closed on the board

## 5. Engineering standards

**Backend**
- Routers stay thin. Business logic lives in `service.py`.
- Modules don't import each other's `models.py`. They interact through service functions or domain events.
- Status changes go through the relevant state machine and are never set directly.
- Webhook endpoints validate the request, enqueue it and return `202`. Processing happens in workers.
- Every LLM extraction carries a confidence score. Results below `LLM_CONFIDENCE_THRESHOLD` go to the exception manager instead of creating an order.
- Code is formatted and linted with `ruff`, and all functions have type hints.

**Frontend**
- Code is organized by feature under `src/features/<area>/`.
- Server state is managed with TanStack Query hooks, using the generated OpenAPI types.
- `npm run lint` and `npm run build` pass before a pull request is opened.

**Secrets**
- `.env` files are never committed. Every new variable is added to `.env.example` with a placeholder value.
- Real credentials (Paystack, LLM, WhatsApp) are kept in a shared password manager and in GitHub Actions secrets, never in chat or code.

## 6. Environments

| Environment | Source | Deployed by | Notes |
|---|---|---|---|
| Local | any branch | the developer | `docker compose up -d` in `stallo-api` |
| Staging | `main` | CI, automatically | Test WhatsApp number and Paystack test keys |
| Production | release tag `v*` | CI, after lead approval | Live numbers and keys |

## 7. Milestones

| Week | Goal |
|---|---|
| 1 | Repositories, CI and the local stack working for every contributor. Interfaces from §3.4 drafted and merged. Database schema v1. |
| 2 | End to end with a linked test number: inbound message → queue → orchestrator → canned reply. Dashboard login and layout. |
| 3 | LLM extracts an order → order created → confirmation reply. Orders page shows live data. |
| 4 | Paystack payment link → webhook → payment confirmed → order marked paid. Payments page. |
| 5 | Exception handling and the "needs attention" page; inventory; delivery status. |
| 6 | Daily summary, paced notifications, hardening, staging demo. |

## 8. Communication

- **Written decisions.** Decisions are recorded in the issue, pull request or an ADR, not only agreed in a call.
- **Daily update** in the team channel: progress, plan, blockers.
- **Blockers are raised right away**, not held until the next meeting.

## 9. Onboarding

1. Accept the organization invitation. A lead adds you to your role's team.
2. Read this handbook and the README of the repository you'll work in.
3. Run `stallo-api` locally (`docker compose up -d`, then `uvicorn`) and `stallo-web` (`npm run dev`).
4. Get development credentials from the shared password manager.
5. Pick up an issue labelled `good first issue` and open a pull request within the first week.
