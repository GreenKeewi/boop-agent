# Loom SaaS — Plan 1: Foundation

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rebrand Boop → Loom, convert the Convex schema to multi-tenant (userId on every table), add users/numberPool/dailyUsage tables, and route Sendblue webhooks by assigned number rather than a single global number.

**Architecture:** Every Convex table gains a required `userId` field so all data is user-scoped at the database layer. Three new tables are added: `users` (identity + tier), `numberPool` (Sendblue number inventory), and `dailyUsage` (per-user daily message counts). The Express webhook handler is updated to look up the recipient Sendblue number in `numberPool` to resolve a `userId` before calling `handleUserMessage`.

**Tech Stack:** TypeScript, Convex, Express, Sendblue API, tsx (scripts)

---

## File Map

| File | Action | Purpose |
|---|---|---|
| `convex/schema.ts` | Modify | Add `userId` to all tables; add `users`, `numberPool`, `dailyUsage` |
| `convex/users.ts` | Create | User CRUD mutations/queries |
| `convex/numberPool.ts` | Create | Number assignment + lookup |
| `convex/dailyUsage.ts` | Create | Increment + read daily message counts |
| `convex/messages.ts` | Modify | Scope all queries by `userId` |
| `convex/conversations.ts` | Modify | Scope by `userId` |
| `convex/automations.ts` | Modify | Scope by `userId` |
| `convex/agents.ts` | Modify | Scope by `userId` |
| `convex/drafts.ts` | Modify | Scope by `userId` |
| `server/sendblue.ts` | Modify | Route by `to_number` → `numberPool` → `userId` |
| `server/interaction-agent.ts` | Modify | Accept + thread `userId`; rename Boop → Loom in system prompt |
| `server/index.ts` | Modify | Pass `userId` through to agent; update service name |
| `scripts/seed-number-pool.ts` | Create | Seed `numberPool` with Sendblue numbers from env |
| `package.json` | Modify | Rename package to `loom` |
| `README.md` | Modify | Update branding references |

---

## Task 1: Rebrand package.json and README

**Files:**
- Modify: `package.json`
- Modify: `README.md`

- [ ] **Step 1: Update package name**

In `package.json`, change:
```json
"name": "boop-agent",
```
to:
```json
"name": "loom",
```

- [ ] **Step 2: Update health endpoint service name in server/index.ts**

In `server/index.ts`, change:
```typescript
res.json({ ok: true, service: "boop-agent" });
```
to:
```typescript
res.json({ ok: true, service: "loom" });
```

- [ ] **Step 3: Update README branding**

Replace the top of `README.md` (lines 1–10) with:
```markdown
<p align="center">
  <img src="assets/boop.gif" alt="Loom" width="220" />
</p>

# Loom

An iMessage-based personal AI agent. Text it like a person — it remembers you, runs automations, and connects to your tools.
```

- [ ] **Step 4: Commit**

```bash
git add package.json server/index.ts README.md
git commit -m "chore: rebrand Boop → Loom"
```

---

## Task 2: Update interaction agent system prompt

**Files:**
- Modify: `server/interaction-agent.ts`

- [ ] **Step 1: Replace agent persona name**

In `server/interaction-agent.ts`, change the `INTERACTION_SYSTEM` constant — replace the first line:
```typescript
const INTERACTION_SYSTEM = `You are Boop, a personal agent the user texts from iMessage.
```
with:
```typescript
const INTERACTION_SYSTEM = `You are Loom, a personal agent the user texts from iMessage.
```

- [ ] **Step 2: Commit**

```bash
git add server/interaction-agent.ts
git commit -m "chore: rename agent persona Boop → Loom"
```

---

## Task 3: Extend Convex schema — add userId to all tables + new tables

**Files:**
- Modify: `convex/schema.ts`

This is the core schema change. Every existing table gains `userId: v.string()` and a `by_user` index. Three new tables are added.

- [ ] **Step 1: Replace convex/schema.ts entirely**

```typescript
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";

export default defineSchema({
  // ─── New SaaS tables ───────────────────────────────────────────────────────

  users: defineTable({
    clerkId: v.optional(v.string()),
    phone: v.string(),
    assignedNumber: v.string(),
    tier: v.union(
      v.literal("free"),
      v.literal("starter"),
      v.literal("core"),
      v.literal("max"),
    ),
    trialExpiresAt: v.optional(v.number()),
    stripeCustomerId: v.optional(v.string()),
    stripeSubscriptionId: v.optional(v.string()),
    composioEntityId: v.optional(v.string()),
    status: v.union(v.literal("pending"), v.literal("active")),
    createdAt: v.number(),
  })
    .index("by_phone", ["phone"])
    .index("by_assigned_number", ["assignedNumber"])
    .index("by_clerk_id", ["clerkId"]),

  numberPool: defineTable({
    number: v.string(),
    status: v.union(v.literal("available"), v.literal("assigned")),
    assignedTo: v.optional(v.string()),
    assignedAt: v.optional(v.number()),
  })
    .index("by_number", ["number"])
    .index("by_status", ["status"]),

  dailyUsage: defineTable({
    userId: v.string(),
    date: v.string(),
    messageCount: v.number(),
  })
    .index("by_user_date", ["userId", "date"]),

  // ─── Existing tables (now user-scoped) ─────────────────────────────────────

  messages: defineTable({
    userId: v.string(),
    conversationId: v.string(),
    role: v.union(v.literal("user"), v.literal("assistant"), v.literal("system")),
    content: v.string(),
    agentId: v.optional(v.string()),
    turnId: v.optional(v.string()),
    createdAt: v.number(),
  })
    .index("by_user", ["userId"])
    .index("by_conversation", ["conversationId"])
    .index("by_conversation_turn", ["conversationId", "turnId"]),

  conversations: defineTable({
    userId: v.string(),
    conversationId: v.string(),
    title: v.optional(v.string()),
    summary: v.optional(v.string()),
    messageCount: v.number(),
    lastActivityAt: v.number(),
  })
    .index("by_user", ["userId"])
    .index("by_conversation", ["conversationId"]),

  memoryRecords: defineTable({
    userId: v.string(),
    memoryId: v.string(),
    content: v.string(),
    tier: v.union(v.literal("short"), v.literal("long"), v.literal("permanent")),
    segment: v.union(
      v.literal("identity"),
      v.literal("preference"),
      v.literal("correction"),
      v.literal("relationship"),
      v.literal("project"),
      v.literal("knowledge"),
      v.literal("context"),
    ),
    importance: v.number(),
    decayRate: v.number(),
    accessCount: v.number(),
    lastAccessedAt: v.number(),
    sourceTurn: v.optional(v.string()),
    lifecycle: v.union(v.literal("active"), v.literal("archived"), v.literal("pruned")),
    supersedes: v.optional(v.array(v.string())),
    embedding: v.optional(v.array(v.float64())),
    metadata: v.optional(v.string()),
    createdAt: v.number(),
  })
    .index("by_user", ["userId"])
    .index("by_memory_id", ["memoryId"])
    .index("by_tier", ["tier"])
    .index("by_segment", ["segment"])
    .index("by_lifecycle", ["lifecycle"])
    .vectorIndex("by_embedding", {
      vectorField: "embedding",
      dimensions: 1024,
      filterFields: ["lifecycle"],
    }),

  executionAgents: defineTable({
    userId: v.string(),
    agentId: v.string(),
    conversationId: v.optional(v.string()),
    name: v.string(),
    task: v.string(),
    status: v.union(
      v.literal("spawned"),
      v.literal("running"),
      v.literal("completed"),
      v.literal("failed"),
      v.literal("cancelled"),
    ),
    result: v.optional(v.string()),
    error: v.optional(v.string()),
    mcpServers: v.array(v.string()),
    inputTokens: v.number(),
    outputTokens: v.number(),
    cacheReadTokens: v.optional(v.number()),
    cacheCreationTokens: v.optional(v.number()),
    costUsd: v.number(),
    startedAt: v.number(),
    completedAt: v.optional(v.number()),
  })
    .index("by_user", ["userId"])
    .index("by_agent_id", ["agentId"])
    .index("by_status", ["status"])
    .index("by_conversation", ["conversationId"]),

  usageRecords: defineTable({
    userId: v.string(),
    source: v.union(
      v.literal("dispatcher"),
      v.literal("execution"),
      v.literal("extract"),
      v.literal("consolidation-proposer"),
      v.literal("consolidation-adversary"),
      v.literal("consolidation-judge"),
    ),
    conversationId: v.optional(v.string()),
    turnId: v.optional(v.string()),
    agentId: v.optional(v.string()),
    runId: v.optional(v.string()),
    model: v.string(),
    inputTokens: v.number(),
    outputTokens: v.number(),
    cacheReadTokens: v.number(),
    cacheCreationTokens: v.number(),
    costUsd: v.number(),
    durationMs: v.number(),
    createdAt: v.number(),
  })
    .index("by_user", ["userId"])
    .index("by_conversation", ["conversationId"])
    .index("by_agent", ["agentId"])
    .index("by_source", ["source"]),

  agentLogs: defineTable({
    userId: v.string(),
    agentId: v.string(),
    logType: v.union(
      v.literal("thinking"),
      v.literal("tool_use"),
      v.literal("tool_result"),
      v.literal("text"),
      v.literal("error"),
    ),
    toolName: v.optional(v.string()),
    content: v.string(),
    createdAt: v.number(),
  })
    .index("by_user", ["userId"])
    .index("by_agent", ["agentId"]),

  memoryEvents: defineTable({
    userId: v.string(),
    eventType: v.string(),
    conversationId: v.optional(v.string()),
    memoryId: v.optional(v.string()),
    agentId: v.optional(v.string()),
    data: v.string(),
    createdAt: v.number(),
  })
    .index("by_user", ["userId"])
    .index("by_conversation", ["conversationId"])
    .index("by_type", ["eventType"]),

  automations: defineTable({
    userId: v.string(),
    automationId: v.string(),
    name: v.string(),
    task: v.string(),
    integrations: v.array(v.string()),
    schedule: v.string(),
    enabled: v.boolean(),
    conversationId: v.optional(v.string()),
    notifyConversationId: v.optional(v.string()),
    lastRunAt: v.optional(v.number()),
    nextRunAt: v.optional(v.number()),
    createdAt: v.number(),
  })
    .index("by_user", ["userId"])
    .index("by_automation_id", ["automationId"])
    .index("by_enabled", ["enabled"]),

  sendblueDedup: defineTable({
    handle: v.string(),
    claimedAt: v.number(),
  }).index("by_handle", ["handle"]),

  drafts: defineTable({
    userId: v.string(),
    draftId: v.string(),
    conversationId: v.string(),
    kind: v.string(),
    summary: v.string(),
    payload: v.string(),
    status: v.union(
      v.literal("pending"),
      v.literal("sent"),
      v.literal("rejected"),
      v.literal("expired"),
    ),
    createdAt: v.number(),
    decidedAt: v.optional(v.number()),
  })
    .index("by_user", ["userId"])
    .index("by_draft_id", ["draftId"])
    .index("by_conversation_status", ["conversationId", "status"]),

  consolidationRuns: defineTable({
    userId: v.string(),
    runId: v.string(),
    trigger: v.string(),
    status: v.union(
      v.literal("running"),
      v.literal("completed"),
      v.literal("failed"),
    ),
    proposalsCount: v.number(),
    mergedCount: v.number(),
    prunedCount: v.number(),
    notes: v.optional(v.string()),
    details: v.optional(v.string()),
    startedAt: v.number(),
    completedAt: v.optional(v.number()),
  })
    .index("by_user", ["userId"])
    .index("by_run_id", ["runId"])
    .index("by_status", ["status"]),

  automationRuns: defineTable({
    userId: v.string(),
    runId: v.string(),
    automationId: v.string(),
    status: v.union(
      v.literal("running"),
      v.literal("completed"),
      v.literal("failed"),
    ),
    result: v.optional(v.string()),
    error: v.optional(v.string()),
    agentId: v.optional(v.string()),
    startedAt: v.number(),
    completedAt: v.optional(v.number()),
  })
    .index("by_user", ["userId"])
    .index("by_automation", ["automationId"])
    .index("by_run_id", ["runId"]),
});
```

- [ ] **Step 2: Push schema to Convex dev**

```bash
npx convex dev --once
```

Expected: schema pushed, no type errors. Note: existing data will be incompatible with the new required `userId` field — this is expected. The dev database will need to be reset (see Step 3).

- [ ] **Step 3: Reset dev database**

In the Convex dashboard (https://dashboard.convex.dev), navigate to your project → Settings → "Clear all data". This wipes the single-user dev data so the new schema can be used cleanly.

- [ ] **Step 4: Commit**

```bash
git add convex/schema.ts
git commit -m "feat(schema): add userId to all tables, add users/numberPool/dailyUsage"
```

---

## Task 4: Create convex/users.ts

**Files:**
- Create: `convex/users.ts`

- [ ] **Step 1: Create the file**

```typescript
import { mutation, query } from "./_generated/server";
import { v } from "convex/values";

export const getByPhone = query({
  args: { phone: v.string() },
  handler: async (ctx, { phone }) => {
    return ctx.db
      .query("users")
      .withIndex("by_phone", (q) => q.eq("phone", phone))
      .unique();
  },
});

export const getByAssignedNumber = query({
  args: { assignedNumber: v.string() },
  handler: async (ctx, { assignedNumber }) => {
    return ctx.db
      .query("users")
      .withIndex("by_assigned_number", (q) => q.eq("assignedNumber", assignedNumber))
      .unique();
  },
});

export const getById = query({
  args: { userId: v.id("users") },
  handler: async (ctx, { userId }) => {
    return ctx.db.get(userId);
  },
});

export const create = mutation({
  args: {
    phone: v.string(),
    assignedNumber: v.string(),
    composioEntityId: v.optional(v.string()),
  },
  handler: async (ctx, { phone, assignedNumber, composioEntityId }) => {
    const trialExpiresAt = Date.now() + 7 * 24 * 60 * 60 * 1000;
    return ctx.db.insert("users", {
      phone,
      assignedNumber,
      tier: "core",
      trialExpiresAt,
      status: "pending",
      composioEntityId,
      createdAt: Date.now(),
    });
  },
});

export const activate = mutation({
  args: { userId: v.id("users") },
  handler: async (ctx, { userId }) => {
    await ctx.db.patch(userId, { status: "active" });
  },
});

export const updateTier = mutation({
  args: {
    userId: v.id("users"),
    tier: v.union(v.literal("free"), v.literal("starter"), v.literal("core"), v.literal("max")),
    stripeCustomerId: v.optional(v.string()),
    stripeSubscriptionId: v.optional(v.string()),
  },
  handler: async (ctx, { userId, tier, stripeCustomerId, stripeSubscriptionId }) => {
    await ctx.db.patch(userId, {
      tier,
      ...(stripeCustomerId && { stripeCustomerId }),
      ...(stripeSubscriptionId && { stripeSubscriptionId }),
    });
  },
});

export const expireTrials = mutation({
  args: {},
  handler: async (ctx) => {
    const now = Date.now();
    const coreUsers = await ctx.db
      .query("users")
      .filter((q) => q.eq(q.field("tier"), "core"))
      .collect();
    let expired = 0;
    for (const user of coreUsers) {
      if (
        user.trialExpiresAt &&
        user.trialExpiresAt < now &&
        !user.stripeSubscriptionId
      ) {
        await ctx.db.patch(user._id, { tier: "free" });
        expired++;
      }
    }
    return { expired };
  },
});
```

- [ ] **Step 2: Push to Convex**

```bash
npx convex dev --once
```

Expected: no TypeScript errors, functions registered.

- [ ] **Step 3: Commit**

```bash
git add convex/users.ts
git commit -m "feat(convex): add users table mutations and queries"
```

---

## Task 5: Create convex/numberPool.ts

**Files:**
- Create: `convex/numberPool.ts`

- [ ] **Step 1: Create the file**

```typescript
import { mutation, query } from "./_generated/server";
import { v } from "convex/values";

export const getByNumber = query({
  args: { number: v.string() },
  handler: async (ctx, { number }) => {
    return ctx.db
      .query("numberPool")
      .withIndex("by_number", (q) => q.eq("number", number))
      .unique();
  },
});

export const getAvailable = query({
  args: {},
  handler: async (ctx) => {
    return ctx.db
      .query("numberPool")
      .withIndex("by_status", (q) => q.eq("status", "available"))
      .first();
  },
});

export const seed = mutation({
  args: { numbers: v.array(v.string()) },
  handler: async (ctx, { numbers }) => {
    let added = 0;
    for (const number of numbers) {
      const existing = await ctx.db
        .query("numberPool")
        .withIndex("by_number", (q) => q.eq("number", number))
        .unique();
      if (!existing) {
        await ctx.db.insert("numberPool", {
          number,
          status: "available",
        });
        added++;
      }
    }
    return { added };
  },
});

export const assign = mutation({
  args: { number: v.string(), userId: v.string() },
  handler: async (ctx, { number, userId }) => {
    const entry = await ctx.db
      .query("numberPool")
      .withIndex("by_number", (q) => q.eq("number", number))
      .unique();
    if (!entry) throw new Error(`Number ${number} not in pool`);
    if (entry.status === "assigned") throw new Error(`Number ${number} already assigned`);
    await ctx.db.patch(entry._id, {
      status: "assigned",
      assignedTo: userId,
      assignedAt: Date.now(),
    });
  },
});

export const getUserIdForNumber = query({
  args: { number: v.string() },
  handler: async (ctx, { number }) => {
    const entry = await ctx.db
      .query("numberPool")
      .withIndex("by_number", (q) => q.eq("number", number))
      .unique();
    return entry?.assignedTo ?? null;
  },
});
```

- [ ] **Step 2: Push to Convex**

```bash
npx convex dev --once
```

Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add convex/numberPool.ts
git commit -m "feat(convex): add numberPool mutations and queries"
```

---

## Task 6: Create convex/dailyUsage.ts

**Files:**
- Create: `convex/dailyUsage.ts`

- [ ] **Step 1: Create the file**

```typescript
import { mutation, query } from "./_generated/server";
import { v } from "convex/values";

function todayUtc(): string {
  return new Date().toISOString().slice(0, 10);
}

export const getToday = query({
  args: { userId: v.string() },
  handler: async (ctx, { userId }) => {
    const date = todayUtc();
    const row = await ctx.db
      .query("dailyUsage")
      .withIndex("by_user_date", (q) => q.eq("userId", userId).eq("date", date))
      .unique();
    return row?.messageCount ?? 0;
  },
});

export const increment = mutation({
  args: { userId: v.string() },
  handler: async (ctx, { userId }) => {
    const date = todayUtc();
    const existing = await ctx.db
      .query("dailyUsage")
      .withIndex("by_user_date", (q) => q.eq("userId", userId).eq("date", date))
      .unique();
    if (existing) {
      await ctx.db.patch(existing._id, { messageCount: existing.messageCount + 1 });
      return existing.messageCount + 1;
    } else {
      await ctx.db.insert("dailyUsage", { userId, date, messageCount: 1 });
      return 1;
    }
  },
});
```

- [ ] **Step 2: Push to Convex**

```bash
npx convex dev --once
```

Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add convex/dailyUsage.ts
git commit -m "feat(convex): add dailyUsage tracking"
```

---

## Task 7: Create scripts/seed-number-pool.ts

**Files:**
- Create: `scripts/seed-number-pool.ts`

This script reads a comma-separated list of Sendblue numbers from `LOOM_NUMBER_POOL` env var and seeds the `numberPool` table.

- [ ] **Step 1: Create the script**

```typescript
import { ConvexHttpClient } from "convex/browser";
import { api } from "../convex/_generated/api.js";
import { config } from "dotenv";

config({ path: ".env.local" });

const url = process.env.CONVEX_URL;
if (!url) {
  console.error("CONVEX_URL not set in .env.local");
  process.exit(1);
}

const raw = process.env.LOOM_NUMBER_POOL;
if (!raw) {
  console.error(
    "LOOM_NUMBER_POOL not set. Set it to a comma-separated list of Sendblue numbers, e.g.:\n  LOOM_NUMBER_POOL=+12025550101,+12025550102",
  );
  process.exit(1);
}

const numbers = raw
  .split(",")
  .map((n) => n.trim())
  .filter(Boolean);

const client = new ConvexHttpClient(url);
const result = await client.mutation(api.numberPool.seed, { numbers });
console.log(`Seeded ${result.added} numbers (${numbers.length - result.added} already existed).`);
```

- [ ] **Step 2: Add seed script to package.json scripts**

In `package.json`, add inside `"scripts"`:
```json
"seed:numbers": "tsx scripts/seed-number-pool.ts"
```

- [ ] **Step 3: Add LOOM_NUMBER_POOL to .env.local**

Add a line to `.env.local`:
```
LOOM_NUMBER_POOL=+1XXXXXXXXXX,+1XXXXXXXXXX
```
Replace with your 10 actual Sendblue numbers (comma-separated, E.164 format).

- [ ] **Step 4: Run the seed script**

```bash
npm run seed:numbers
```

Expected output:
```
Seeded 10 numbers (0 already existed).
```

- [ ] **Step 5: Commit**

```bash
git add scripts/seed-number-pool.ts package.json
git commit -m "feat(scripts): add number pool seed script"
```

---

## Task 8: Update Sendblue webhook to route by assigned number

**Files:**
- Modify: `server/sendblue.ts`

Currently the webhook identifies the conversation by `from_number` (sender). In multi-tenant mode, the webhook also receives `to_number` (which Sendblue number was texted). We look up `to_number` in `numberPool` to get the `userId`, then create/route the conversation under that user.

- [ ] **Step 1: Read current sendblue.ts webhook handler (lines 122–182) — already read above**

- [ ] **Step 2: Replace createSendblueRouter in server/sendblue.ts**

Replace the entire `createSendblueRouter` function:

```typescript
export function createSendblueRouter(): express.Router {
  const router = express.Router();

  router.post("/webhook", async (req, res) => {
    const { content, from_number, to_number, is_outbound, message_handle } = req.body ?? {};
    if (is_outbound || !content || !from_number) {
      res.json({ ok: true, skipped: true });
      return;
    }

    if (message_handle) {
      const { claimed } = await convex.mutation(api.sendblueDedup.claim, {
        handle: message_handle,
      });
      if (!claimed) {
        res.json({ ok: true, deduped: true });
        return;
      }
    }

    // Resolve which user owns the number that was texted.
    const normalizedTo = normalizeE164(to_number);
    if (!normalizedTo) {
      console.warn("[sendblue] webhook missing to_number — skipping");
      res.json({ ok: true, skipped: true });
      return;
    }

    const poolEntry = await convex.query(api.numberPool.getByNumber, {
      number: normalizedTo,
    });

    // Number not in pool — unknown number, ignore.
    if (!poolEntry) {
      console.warn(`[sendblue] to_number ${normalizedTo} not in pool — ignoring`);
      res.json({ ok: true, skipped: true });
      return;
    }

    // Number is available (not yet assigned) — new user signup flow.
    if (poolEntry.status === "available") {
      // Create or retrieve user by their sender phone.
      let user = await convex.query(api.users.getByPhone, { phone: from_number });
      if (!user) {
        const userId = await convex.mutation(api.users.create, {
          phone: from_number,
          assignedNumber: normalizedTo,
        });
        await convex.mutation(api.numberPool.assign, {
          number: normalizedTo,
          userId: userId as string,
        });
        user = await convex.query(api.users.getById, { userId: userId as any });
      }
      if (!user) {
        console.error("[sendblue] failed to create user");
        res.json({ ok: true });
        return;
      }
      res.json({ ok: true });
      // Route to interaction agent in sales mode.
      await routeToAgent({ user, content, from_number, message_handle, salesMode: true });
      return;
    }

    // Number is assigned — find the owning user.
    const user = poolEntry.assignedTo
      ? await convex.query(api.users.getById, { userId: poolEntry.assignedTo as any })
      : null;

    if (!user) {
      console.error(`[sendblue] numberPool entry ${normalizedTo} has no valid user`);
      res.json({ ok: true });
      return;
    }

    // Sender is not the assigned owner — reject.
    if (user.phone !== from_number) {
      await sendImessage(
        from_number,
        "This number belongs to someone else. Visit heyloom.ai to get your own Loom.",
        normalizedTo,
      );
      res.json({ ok: true });
      return;
    }

    res.json({ ok: true });
    await routeToAgent({ user, content, from_number, message_handle, salesMode: false });
  });

  return router;
}

async function routeToAgent({
  user,
  content,
  from_number,
  message_handle,
  salesMode,
}: {
  user: { _id: string; phone: string; assignedNumber: string; status: string };
  content: string;
  from_number: string;
  message_handle?: string;
  salesMode: boolean;
}): Promise<void> {
  const userId = user._id;
  const conversationId = `sms:${from_number}`;
  const turnTag = Math.random().toString(36).slice(2, 8);
  const preview = content.length > 100 ? content.slice(0, 100) + "…" : content;
  console.log(`[turn ${turnTag}][user ${userId}] ← ${from_number}: ${JSON.stringify(preview)}`);
  const start = Date.now();

  broadcast("message_in", { conversationId, userId, content, from_number, handle: message_handle });

  const stopTyping = startTypingLoop(from_number, user.assignedNumber);
  try {
    const reply = await handleUserMessage({
      userId,
      conversationId,
      content,
      turnTag,
      salesMode,
      onThinking: (t) => broadcast("thinking", { conversationId, t }),
    });
    if (reply) {
      const elapsed = ((Date.now() - start) / 1000).toFixed(1);
      console.log(`[turn ${turnTag}] → reply (${elapsed}s, ${reply.length} chars)`);
      await sendImessage(from_number, reply, user.assignedNumber);
      await convex.mutation(api.messages.send, {
        userId,
        conversationId,
        role: "assistant",
        content: reply,
      });
    }
  } catch (err) {
    console.error(`[turn ${turnTag}] handler error`, err);
  } finally {
    stopTyping();
  }
}
```

- [ ] **Step 3: Update sendImessage signature to accept fromNumber**

Replace the `sendImessage` function signature and `from` resolution:

```typescript
export async function sendImessage(
  toNumber: string,
  text: string,
  fromNumber?: string,
): Promise<void> {
  const h = headers();
  if (!h) {
    console.warn("[sendblue] missing credentials — not sending");
    return;
  }
  const from = normalizeE164(fromNumber ?? process.env.SENDBLUE_FROM_NUMBER);
  if (!from) {
    console.error("[sendblue] no from_number available");
    return;
  }
  const plain = stripMarkdown(text);
  for (const part of chunk(plain)) {
    const res = await fetch(`${API_BASE}/send-message`, {
      method: "POST",
      headers: h,
      body: JSON.stringify({ number: toNumber, content: part, from_number: from }),
    });
    if (!res.ok) {
      const body = await res.text().catch(() => "");
      console.error(`[sendblue] send failed ${res.status}: ${body}`);
    } else {
      console.log(`[sendblue] → sent ${part.length} chars to ${toNumber} from ${from}`);
    }
  }
}
```

- [ ] **Step 4: Update sendTypingIndicator and startTypingLoop signatures**

```typescript
export async function sendTypingIndicator(toNumber: string, fromNumber?: string): Promise<void> {
  const h = headers();
  if (!h) return;
  const from = fromNumber ?? process.env.SENDBLUE_FROM_NUMBER;
  try {
    await fetch(`${API_BASE}/send-typing-indicator`, {
      method: "POST",
      headers: h,
      body: JSON.stringify({ number: toNumber, from_number: from }),
    });
  } catch {
    /* non-fatal */
  }
}

export function startTypingLoop(toNumber: string, fromNumber?: string): () => void {
  sendTypingIndicator(toNumber, fromNumber);
  const timer = setInterval(() => sendTypingIndicator(toNumber, fromNumber), 5000);
  return () => clearInterval(timer);
}
```

- [ ] **Step 5: Verify TypeScript compiles**

```bash
npm run typecheck
```

Expected: no errors (some errors in interaction-agent.ts from the new `userId` param are expected — fixed in Task 9).

- [ ] **Step 6: Commit**

```bash
git add server/sendblue.ts
git commit -m "feat(sendblue): route webhook by assigned number → userId"
```

---

## Task 9: Update interaction-agent.ts to accept userId

**Files:**
- Modify: `server/interaction-agent.ts`

The `handleUserMessage` function needs to accept `userId` and `salesMode` parameters and thread them through.

- [ ] **Step 1: Update the HandleUserMessageOptions type**

Find the options type/interface for `handleUserMessage` and add the new fields. Look for the call signature near the top of the function — it likely takes an object. Update it to:

```typescript
interface HandleUserMessageOptions {
  userId: string;
  conversationId: string;
  content: string;
  turnTag: string;
  salesMode?: boolean;
  onThinking?: (t: string) => void;
}
```

- [ ] **Step 2: Add SALES_MODE_ADDITION constant after INTERACTION_SYSTEM**

```typescript
const SALES_MODE_ADDITION = `

IMPORTANT: This user is NEW and has NOT signed up yet. You are in SALES MODE.

Your goals in order:
1. Greet them warmly. Introduce yourself as Loom — their personal iMessage AI agent.
2. Briefly explain what you can do: schedule things, manage their calendar, search the web, send emails, set reminders, and more — all from iMessage.
3. Mention they get a FREE 7-day trial of the full Core plan.
4. When they seem interested or ask how to sign up, tell them to reply SIGNUP to activate their free trial.
5. Once they reply SIGNUP, confirm their account is active and transition to normal assistant mode.

Keep it conversational. Two or three short messages max before the pitch. Don't be pushy.`;
```

- [ ] **Step 3: Pass salesMode into the system prompt**

In the `handleUserMessage` function body, change where the system prompt is constructed. Find where `INTERACTION_SYSTEM` is used as the system prompt and update it:

```typescript
const systemPrompt = salesMode
  ? INTERACTION_SYSTEM + SALES_MODE_ADDITION
  : INTERACTION_SYSTEM;
```

- [ ] **Step 4: Handle SIGNUP trigger**

At the top of `handleUserMessage`, before running the agent, check for the SIGNUP keyword:

```typescript
if (salesMode && content.trim().toUpperCase() === "SIGNUP") {
  await convex.mutation(api.users.activate, { userId: userId as any });
  return "You're in! 🎉 Your 7-day Loom Core trial is now active. Text me anything — I'm ready to help.";
}
```

- [ ] **Step 5: Pass userId to all Convex calls inside handleUserMessage**

Find all `convex.mutation(api.messages.send, {...})` and similar calls inside `handleUserMessage` and add `userId` to each payload. For example:

```typescript
await convex.mutation(api.messages.send, {
  userId,
  conversationId,
  role: "user",
  content,
});
```

Do the same for any `api.conversations.*`, `api.usageRecords.*`, `api.agentLogs.*`, `api.memoryEvents.*` calls.

- [ ] **Step 6: Verify TypeScript**

```bash
npm run typecheck
```

Expected: no errors.

- [ ] **Step 7: Commit**

```bash
git add server/interaction-agent.ts
git commit -m "feat(agent): thread userId through interaction agent, add sales mode"
```

---

## Task 10: Update Convex function files to require userId

**Files:**
- Modify: `convex/messages.ts`
- Modify: `convex/conversations.ts`
- Modify: `convex/automations.ts`
- Modify: `convex/agents.ts`
- Modify: `convex/drafts.ts`

Each of these files has mutations and queries that write or read data. They all need to accept and use `userId`.

- [ ] **Step 1: Read convex/messages.ts**

```bash
cat convex/messages.ts
```

- [ ] **Step 2: Add userId to the send mutation args and insert payload**

In every `insert` call in `convex/messages.ts`, add `userId: v.string()` to args and include it in the document. In every `query` that fetches messages for a conversation, add a `userId` filter. Example pattern:

```typescript
export const send = mutation({
  args: {
    userId: v.string(),
    conversationId: v.string(),
    role: v.union(v.literal("user"), v.literal("assistant"), v.literal("system")),
    content: v.string(),
    agentId: v.optional(v.string()),
    turnId: v.optional(v.string()),
  },
  handler: async (ctx, args) => {
    return ctx.db.insert("messages", {
      userId: args.userId,
      conversationId: args.conversationId,
      role: args.role,
      content: args.content,
      agentId: args.agentId,
      turnId: args.turnId,
      createdAt: Date.now(),
    });
  },
});
```

Apply the same pattern to all mutations in `conversations.ts`, `automations.ts`, `agents.ts`, `drafts.ts`.

- [ ] **Step 3: Push updated functions**

```bash
npx convex dev --once
```

Expected: no errors.

- [ ] **Step 4: Run typecheck**

```bash
npm run typecheck
```

Expected: no errors.

- [ ] **Step 5: Commit**

```bash
git add convex/messages.ts convex/conversations.ts convex/automations.ts convex/agents.ts convex/drafts.ts
git commit -m "feat(convex): scope all mutations and queries by userId"
```

---

## Task 11: Smoke test end-to-end

- [ ] **Step 1: Start the dev environment**

```bash
npm run dev
```

Expected: server starts, Convex dev running, no crash.

- [ ] **Step 2: Verify health endpoint**

```bash
curl http://localhost:3001/health
```

Expected:
```json
{"ok":true,"service":"loom"}
```

- [ ] **Step 3: Verify number pool is seeded**

In the Convex dashboard, open the `numberPool` table. Confirm 10 rows exist with `status: "available"`.

- [ ] **Step 4: Simulate a new-user webhook**

```bash
curl -X POST http://localhost:3001/sendblue/webhook \
  -H "Content-Type: application/json" \
  -d '{
    "content": "Hello",
    "from_number": "+15005550001",
    "to_number": "<one of your pool numbers>",
    "is_outbound": false,
    "message_handle": "test-handle-001"
  }'
```

Expected: HTTP 200 `{"ok":true}`. Check Convex dashboard — a new `users` row should appear with `status: "pending"`, and the `numberPool` entry should change to `status: "assigned"`.

- [ ] **Step 5: Simulate SIGNUP trigger**

```bash
curl -X POST http://localhost:3001/sendblue/webhook \
  -H "Content-Type: application/json" \
  -d '{
    "content": "SIGNUP",
    "from_number": "+15005550001",
    "to_number": "<same pool number>",
    "is_outbound": false,
    "message_handle": "test-handle-002"
  }'
```

Expected: user `status` in Convex changes to `"active"`.

- [ ] **Step 6: Commit smoke test passing**

```bash
git commit --allow-empty -m "chore: plan-1 foundation smoke test passing"
```

---

## Self-Review Checklist

- [x] Rebrand: Boop → Loom in package.json, server/index.ts, interaction-agent.ts, README
- [x] Schema: `userId` added to all 12 existing tables + 3 new tables
- [x] `users.ts`: create, activate, updateTier, expireTrials, getByPhone, getByAssignedNumber
- [x] `numberPool.ts`: seed, assign, getByNumber, getAvailable, getUserIdForNumber
- [x] `dailyUsage.ts`: increment, getToday
- [x] Webhook routing: to_number → numberPool → userId → agent
- [x] New user flow: available number → create user → sales mode agent
- [x] SIGNUP trigger: activates user, transitions out of sales mode
- [x] Stranger rejection: sends "number belongs to someone else" reply
- [x] sendImessage updated to accept per-user fromNumber
- [x] All Convex mutations updated to accept + store userId
- [x] Seed script: reads LOOM_NUMBER_POOL env var, idempotent
