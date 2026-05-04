# Loom — SaaS Design Spec

**Date:** 2026-05-03
**Project:** Boop Agent → Loom (rebrand)
**Approach:** Convex-as-hub, multi-tenant SaaS

---

## Overview

Loom is a personal iMessage AI agent offered as a SaaS product. Users get a dedicated iMessage number, talk to their agent naturally, set up cron automations, and connect integrations — all from their phone. A web dashboard handles settings, billing, automations, and conversation history.

Inspired by Lindy AI but designed for normal people: the onboarding happens entirely inside iMessage, the agent sells itself, and the web dashboard is for power users.

---

## Rebrand

- Product name: **Loom**
- Domain: `heyloom.ai`
- All references to "Boop" replaced throughout codebase, UI, and docs
- New README written from scratch (see README section below)

---

## Architecture

```
Sendblue (pool of numbers)
        │
        │  POST /webhook  (thin Express receiver, ~50 lines)
        ▼
   Convex HTTP Action  ──► route by assignedNumber ──► userId lookup
        │
        ▼
   Convex Action: runAgent(userId, message)
        │  checks tier limits, enforces quotas
        ▼
   Interaction Agent  ──►  Execution Agent(s)
        │                        │
        ▼                        ▼
   Convex (truth)          Composio integrations (per-user entity)
        │
        ▼
   Next.js 15 Web Dashboard  (Clerk auth, real-time via Convex)
```

### Key architectural decisions

- **Convex is the hub.** All agent execution, user state, tier enforcement, and real-time updates flow through Convex. No shared in-memory state.
- **Express shrinks to a webhook receiver.** Its only job is accepting Sendblue POST requests and forwarding them to a Convex HTTP action.
- **Agent execution moves to Convex actions.** `runAgent(userId)` is the single entry point — it enforces limits, loads memory, and spawns the interaction agent.
- **Row-level isolation.** Every Convex table has a `userId` index. All queries filter by `userId` from the authenticated Clerk session, never from the client payload.

---

## User Tables (Convex Schema)

### `users`
```
clerkId: string          // Clerk user ID (primary auth identity)
phone: string            // user's personal phone number
assignedNumber: string   // their dedicated Sendblue number
tier: "free" | "starter" | "core" | "max"
trialExpiresAt: number   // epoch ms — Core trial for 7 days
stripeCustomerId: string
stripeSubscriptionId: string
composioEntityId: string // per-user Composio entity for isolated integrations
createdAt: number
```

### `numberPool`
```
number: string           // Sendblue phone number
status: "available" | "assigned"
assignedTo: string | null  // userId
assignedAt: number | null
```

### `dailyUsage`
```
userId: string
date: string             // YYYY-MM-DD UTC
messageCount: number
```

All existing tables (`messages`, `agents`, `automations`, `memoryRecords`, `drafts`, `usageRecords`) gain a required `userId` field with an index.

---

## Tiers

| | Free | Starter ($9/mo) | Core ($19/mo) | Max ($49/mo) |
|---|---|---|---|---|
| Messages/day | 10 | 50 | 200 | Unlimited |
| Model | Haiku | Haiku | Sonnet | Opus |
| Integrations | 1 | 5 | 20 | Unlimited |
| Cron jobs | 1 | 5 | 20 | Unlimited |
| iMessage number | Yes | Yes | Yes | Yes |

New users start on a **7-day Core trial**. After expiry, they drop to Free unless subscribed.

---

## Auth & User Provisioning

### iMessage onboarding (primary flow)

1. Admin pre-provisions a pool of 10 Sendblue numbers stored in `numberPool` with status `available`.
2. A new phone texts any available number. The Convex HTTP action sees the number is unassigned → creates a `users` record with `tier: "core"`, `trialExpiresAt: now + 7 days`, assigns the Sendblue number permanently, creates a Composio entity for the user.
3. The agent starts in **sales mode**: introduces Loom, explains capabilities, pitches the 7-day Core trial, answers questions.
4. When user texts the trigger word `SIGNUP`, the `completeSignup` mutation marks the user active.
5. The assigned Sendblue number is locked to that user forever. Messages from an unknown sender to an assigned number receive: "This number belongs to someone else. Visit [domain] to get your own Loom."

### Web auth (Clerk)

- Magic link (passwordless email) + phone OTP via Clerk.
- New web user: email → magic link → dashboard → prompted to add phone → OTP to verify → Convex links Clerk ID to phone-based account (or creates new account).
- **Web login challenge:** user clicks "Verify via iMessage" → agent texts them a one-time code → they enter it on web. Built as a custom Convex mutation + Clerk session extension.
- Clerk middleware protects all `/dashboard/*` routes.
- First login redirects to `/onboarding` if phone not yet linked.

---

## Billing (Stripe)

- Stripe Checkout for new subscriptions.
- Stripe Customer Portal for plan changes and cancellations.
- Convex HTTP action handles Stripe webhooks:
  - `checkout.session.completed` → update `users.tier`
  - `customer.subscription.updated` → update tier
  - `customer.subscription.deleted` → downgrade to Free
- **Trial expiry:** Convex scheduled function runs daily, finds users where `trialExpiresAt < now` and `tier === "core"` with no subscription → downgrades to Free.
- **Upgrade via iMessage:** user texts `UPGRADE` → agent generates a Stripe payment link and sends it back via iMessage.
- **Limit hit:** agent replies "You've used all 10 messages today. Reply UPGRADE to unlock more."

---

## Tier Enforcement

All enforcement happens server-side in Convex — nothing is enforced client-side.

- `runAgent(userId)` checks `dailyUsage` before running. Over limit → reply with upgrade prompt, do not run agent.
- `createAutomation` mutation checks cron job count against tier limit before writing.
- Integration connection step checks connected count against tier limit before completing OAuth.
- Model selection: `runAgent` reads tier, maps to model: `{ free: "haiku", starter: "haiku", core: "sonnet", max: "opus" }`.

---

## Per-User Agent Isolation

- Sendblue webhook routes by `assignedNumber` → `numberPool` lookup → `userId`. All downstream operations scoped to that userId.
- Composio: each user has their own `composioEntityId`. Integration tokens are per-entity. User A's Gmail never touches user B's agent.
- Execution agent receives only the integrations the user has connected AND is within their tier limit.
- No shared in-memory state between users — every Convex action is stateless.

---

## Web Dashboard

**Stack:** Next.js 15 App Router, shadcn/ui, Clerk, Convex real-time, premium design via taste skill (https://www.tasteskill.dev/).

**Hosting:** Vercel (Next.js app) + Convex cloud + Railway or Fly.io (thin Express webhook receiver).

### Pages

| Route | Purpose |
|---|---|
| `/` | Marketing landing page (migrate from `/website`) |
| `/onboarding` | Post-signup: verify phone, pick plan, confirm number |
| `/dashboard` | Usage stats, message count, tier badge, agent status |
| `/automations` | List/create/delete cron jobs, next run, last result |
| `/integrations` | Connect/disconnect Composio integrations, tier limit indicator |
| `/conversations` | Read-only iMessage conversation history |
| `/settings` | Phone, email, preferences, danger zone |
| `/billing` | Current plan, usage bar, upgrade → Stripe Customer Portal |

---

## README (to be written during implementation)

The new README for Loom must cover:

### Setup
- Prerequisites: Node.js 20+, Convex account, Clerk account, Sendblue account (agent plan), Stripe account, Composio account
- Clone repo, `npm install`
- `cp .env.example .env.local` — all required keys listed with descriptions
- `npx convex dev` to start Convex and push schema
- Populate `numberPool` — instructions for provisioning Sendblue numbers via their dashboard + the seed script
- `npm run dev` to start everything

### Environment Variables
Full table of every env var, what it does, where to get it:
- `CONVEX_DEPLOYMENT`
- `NEXT_PUBLIC_CONVEX_URL`
- `CLERK_SECRET_KEY` + `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`
- `SENDBLUE_API_KEY` + `SENDBLUE_API_SECRET`
- `STRIPE_SECRET_KEY` + `STRIPE_WEBHOOK_SECRET` + `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY`
- `COMPOSIO_API_KEY`

### Running locally
- `npm run dev` — starts Next.js, Convex dev, and Express webhook receiver in parallel
- Ngrok tunnel for Sendblue webhook (script auto-registers URL)
- Stripe CLI for local webhook forwarding: `stripe listen --forward-to localhost:3001/stripe/webhook`

### Testing
- Unit tests: Convex functions tested with `convex-test`
- Integration test: seed a test user, send a test message via Sendblue test mode, verify agent responds
- Billing test: Stripe test mode, test card numbers
- Auth test: Clerk test mode phone numbers

### Hosting / Deployment
- **Convex:** `npx convex deploy` — auto-deploys to Convex cloud
- **Next.js:** Push to GitHub → Vercel auto-deploys. Add all env vars in Vercel dashboard.
- **Express webhook receiver:** Deploy to Railway (`railway up`) or Fly.io (`fly deploy`). Set `SENDBLUE_WEBHOOK_URL` to the deployed URL.
- **Stripe webhook:** Register the deployed URL in Stripe dashboard → Webhooks.
- **Domain:** Point custom domain to Vercel deployment.

---

## Implementation Phases

1. **Rebrand** — rename Boop → Loom throughout codebase, update assets
2. **Multi-tenant Convex schema** — add `userId` to all tables, `users` + `numberPool` + `dailyUsage` tables
3. **Auth** — Clerk integration, phone-based signup flow, web login challenge
4. **Billing** — Stripe subscriptions, webhook handler, trial expiry scheduled function
5. **Tier enforcement** — `runAgent` limits, cron/integration caps, model selection
6. **Number pool** — provisioning script, assignment logic, sales-mode agent prompt
7. **Web dashboard** — Next.js app, all pages, shadcn/ui + taste skill design
8. **README + docs** — full setup, env vars, hosting guide
9. **Migration** — migrate existing single-user data if needed
