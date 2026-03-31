# Session 5: Post-Session Recap & Tutorial

**AI Features — Lead Brief Generator + Call Follow-up Assistant**

---

## What We Built

Session 5 introduced **AI** into the CRM — not as magic or chatbots, but as a structured API integration. We send specific data (lead fields, activity history, reminders) and get back typed, validated objects (summaries, risks, scripts, next actions). Every AI call follows the same engineering pattern as the rest of the app: schema → service → route → UI.

By the end of Session 5, the `whispyr-academy-full-stack-nextjs` repo had:

- **Vercel AI SDK** — SDK installed (`ai`), configured to use the Vercel AI Gateway for routing requests to any model provider through simple model strings
- **AI service module** — the complete 5-file service pattern (schema, db, service, helpers, index) applied to AI, following the same architecture from Sessions 2-4
- **Lead Brief schema** — a Zod schema defining the exact shape of a brief: summary (paragraph), key facts (2-5 bullets), risks (up to 3), next actions (with reasoning and suggested dates), and questions to ask (up to 3)
- **Call Follow-up schema** — a Zod schema defining the follow-up structure: call script (opening, questions, objection handlers), recommended next step, and suggested reminder with a coerced Date
- **`AILeadBrief` Prisma model** — a new database table for persisting AI-generated briefs as JSON, linked to both the lead and the user who created it
- **`AI_LEAD_BRIEF_GENERATED` activity type** — a new activity type that gets automatically logged to the timeline whenever a brief is generated
- **`POST /api/ai/lead-brief`** — route handler that authenticates, validates, fetches lead context, calls the AI service, logs the generation as an activity, and returns a typed brief
- **`POST /api/ai/lead-brief/save`** — route handler that persists a generated brief to the `AILeadBrief` table
- **`GET /api/leads/[id]/lead-brief`** — route handler that retrieves the most recently saved brief for a lead
- **Lead Brief UI** — an AI tab on the lead detail page with three states: initial (generate button), loading (spinner), and saved (structured card with disclaimer and regenerate). Generated briefs open in a preview Dialog for review before saving.
- **`BriefContent` component** — a reusable component that renders a brief's summary, key facts, risks, next actions, and questions in structured cards
- **Prompt engineering** — grounded prompts that constrain the AI to only use provided data, with explicit rules for missing information
- **Human-in-the-loop pattern** — every AI suggestion requires explicit human action to save. Nothing auto-executes.
- **Disclaimers** — "AI suggestions can be wrong. Always verify before taking action." visible on all AI-generated content

> **Note:** We built the Lead Brief feature end-to-end during the session, including database persistence and a preview-before-save flow. We defined the Call Follow-up schema but did **NOT** build the Call Follow-up service, route, hooks, or UI. The entire Call Follow-up feature is your **assignment before Session 6**. See the "Assignment" section below.

---

## Step-by-Step Walkthrough

If you missed the session or want to review, follow these steps in order. Each step references the exact files in the [`whispyr-academy-full-stack-nextjs`](https://github.com/whispyrai/whispyr-academy-full-stack-nextjs) repo.

---

### Phase 1: AI Concepts + SDK Setup

**Concepts covered:** AI as structured API, Vercel AI SDK, Vercel AI Gateway, environment configuration.

**What we did:**

Installed the Vercel AI SDK and configured it to use the Vercel AI Gateway as the proxy between our app and the AI model.

**New packages installed (JIT):**

```bash
npm install ai
```

- `ai` — The Vercel AI SDK (provider-agnostic interface). AI SDK 6 uses the Vercel AI Gateway by default — no separate provider package needed.

**Key concept — AI is a stateless transformation service:**

The AI model has no database access, no memory, no network. It only sees the text you include in the prompt. Your app owns data assembly, context gathering, prompt construction, validation, and storage. The AI owns text transformation only.

```text
f(prompt) → structured object

Your app assembles context.
Your app builds the prompt string.
Your app sends the prompt to the AI.
The AI reads the exact text, transforms it, returns typed output.
Your app validates the output before using it.
```

**Key concept — The Vercel AI Gateway:**

The Vercel AI Gateway sits between your server and the model provider. AI SDK 6+ uses it by default. Your app calls any model using simple model strings like `"deepseek/deepseek-v3.2-thinking"` or `"anthropic/claude-sonnet-4-5"`. The gateway provides rate limiting, cost tracking, caching, and provider routing. To switch models, you change one string — no code changes.

**Key concept — API key stays on the server:**

```
Browser → Your Server (API key here) → AI Gateway → Model Provider
```

The browser never touches the API key. The client makes a request to YOUR server, which authenticates the user, builds the prompt, and calls the gateway.

**Environment variables needed:**

```env
AI_GATEWAY_API_KEY=vai_xxxxxxxxxxxxx
```

---

### Phase 2: Prisma Schema Changes

**Concepts covered:** Database modeling for AI features, activity type extension.

**What we did:**

Added a new `AILeadBrief` model to persist generated briefs, and extended the `ActivityType` enum with AI-related types.

**Reference file:** `prisma/schema.prisma`

```prisma
model AILeadBrief {
  id String @id @default(uuid())

  leadId String
  lead   Lead   @relation(fields: [leadId], references: [id])

  brief Json

  createdById String
  createdBy   Profile @relation(fields: [createdById], references: [id])

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

**Key design — Brief stored as JSON:**

The `brief` field is `Json` type — it stores the entire structured brief object (summary, keyFacts, risks, nextActions, questionsToAskNext) as a single JSON column. This is the right choice because briefs are read-only snapshots — you never query by individual fields within the brief. Storing as JSON avoids creating 5+ related tables for what is essentially a document.

**New activity types added to `ActivityType` enum:**

```prisma
enum ActivityType {
  // ... existing types
  AI_LEAD_BRIEF_GENERATED
  AI_FOLLOWUP_DRAFT_GENERATED
}
```

These types appear in the activity timeline. When someone generates a brief, the timeline shows "Lead brief generated by [name]" — so the team can see who used the AI features and when.

**New `createAIActivitySchema` in activity service:**

```typescript
// src/services/activity/schema.ts
export const createAIActivitySchema = z.object({
  type: z.enum([
    ActivityType.AI_LEAD_BRIEF_GENERATED,
    ActivityType.AI_FOLLOWUP_DRAFT_GENERATED,
  ]),
  leadId: z.uuid(),
  actorId: z.uuid(),
  content: z.string().min(1),
});

export type CreateAIActivityRequest = z.infer<typeof createAIActivitySchema>;
```

The `Lead` and `Profile` models also got new relation fields (`aileadBriefs AILeadBrief[]`) to connect to the new model.

After adding the model, run the migration:

```bash
npx prisma migrate dev --name add-ai-lead-brief
npx prisma generate
```

---

### Phase 3: AI Service Module — Schema + DB + Helpers

**Concepts covered:** Service layer pattern for AI, Zod schemas as contracts with AI, prompt building helpers, context fetching.

**What we did:**

Built the AI service module following the same 5-file pattern from Sessions 2-4. The AI service is not special — it follows the exact same architecture as the lead, activity, reminder, and notification services.

**Service folder structure:**

```text
src/services/ai/
  schema.ts      ← Zod schemas for AI output + request validation
  db.ts          ← Database queries for context + persistence
  helpers.ts     ← Prompt builders + access validation
  service.ts     ← generateLeadBrief, saveLeadBrief, getLastLeadBrief
  index.ts       ← Barrel exports
```

**Reference file:** `src/services/ai/schema.ts`

**Lead Brief schema:**

```typescript
import { z } from "zod";

export const leadBriefSchema = z.object({
  summary: z
    .string()
    .max(1000)
    .describe(
      "A brief summary of the lead's situation based on activity history.",
    ),
  keyFacts: z
    .array(z.string().max(100))
    .min(2)
    .max(5)
    .describe(
      "2-5 bullet points about the lead (stage, activity pattern, engagement level, etc.)",
    ),
  risks: z
    .array(z.string().max(100))
    .max(3)
    .describe("up to 3 risks the lead is facing or objections"),
  nextActions: z
    .array(
      z.object({
        action: z.string().describe("The specific action to take"),
        why: z.string().describe("Why this action matters"),
        suggestedDueAt: z
          .string()
          .optional()
          .describe(
            "Optional suggested due date (e.g., 'in 3 days', 'this week')",
          ),
      }),
    )
    .max(3)
    .describe("Up to 3 recommended next actions with reasoning"),
  questionsToAskNext: z
    .array(z.string())
    .max(3)
    .describe("Up to 3 specific questions to ask the lead on the next call"),
});

export type LeadBrief = z.infer<typeof leadBriefSchema>;
```

**Key design — `summary` is a single string, not an array:**

Unlike the snippets which showed `summary` as an array of bullet points, the actual implementation uses a single `z.string().max(1000)`. This lets the AI write a natural flowing paragraph instead of forcing bullets. The UI renders it as a paragraph in the Summary card.

**Key design — `.describe()` on every field:**

Each field uses `.describe()` to tell the AI what content to generate for that field. When the SDK passes this schema to the model, the descriptions become part of the instruction. This is how you guide the AI's output without writing it all in the prompt.

**Key design — Constrained array lengths with `.max()` and `.min()`:**

```typescript
keyFacts: z.array(z.string().max(100)).min(2).max(5);
```

The schema constrains the output: 2-5 key facts, up to 3 risks, up to 3 next actions. String lengths are capped at 100 characters per item. This prevents the AI from generating a 20-bullet list or single-word items.

**Call Follow-up schema:**

```typescript
export const callFollowUpSchema = z.object({
  callScript: z.object({
    opening: z
      .string()
      .describe(
        "A suggested opening for the next call referencing last conversation",
      ),
    questions: z
      .array(z.string())
      .length(3)
      .describe("3 specific questions to ask based on the lead's situation"),
    objectionHandlers: z
      .array(
        z.object({
          objection: z
            .string()
            .describe("A potential objection the lead might raise"),
          response: z
            .string()
            .describe("How to handle this objection professionally"),
        }),
      )
      .length(2)
      .describe("2 common objections and how to handle them"),
  }),
  recommendedNextStep: z
    .string()
    .describe("The single most important next step after this call"),
  suggestedReminder: z.object({
    title: z.string().describe("Short, actionable reminder title"),
    note: z.string().describe("Context for the reminder"),
    suggestedDueAt: z.coerce
      .date()
      .describe("Suggested due date in ISO format"),
  }),
});

export type CallFollowUp = z.infer<typeof callFollowUpSchema>;
```

**Key design — `z.coerce.date()` for suggested due date:**

```typescript
suggestedDueAt: z.coerce.date().describe("Suggested due date in ISO format");
```

The call follow-up schema uses `z.coerce.date()` instead of `z.string()` for the suggested reminder date. This automatically converts the AI's ISO date string into a JavaScript `Date` object. When you later pass this to `useCreateLeadReminder`, it's already the right type — no manual parsing needed.

**Key design — Nested objects in schemas:**

The `callScript` field contains a nested object with `opening`, `questions`, and `objectionHandlers`. This structure tells the AI to organize its response into logical sections. Without this structure, you'd get a blob of text that you'd have to parse yourself.

**Request validation schemas:**

```typescript
export const generateLeadBriefSchema = z.object({
  leadId: z.uuid(),
});

export const saveLeadBriefSchema = z.object({
  leadId: z.uuid(),
  brief: leadBriefSchema,
});

export type SaveLeadBriefRequest = z.infer<typeof saveLeadBriefSchema>;
```

Note: `saveLeadBriefSchema` validates that the brief matches the `leadBriefSchema` exactly before it's persisted. This prevents storing malformed or manually crafted objects.

---

**Reference file:** `src/services/ai/db.ts`

Database read operations to gather context for AI prompts, plus write operations for brief persistence.

```typescript
import { prisma } from "@/lib/prisma";
import { SaveLeadBriefRequest } from "./schema";
import { Profile } from "@/generated/prisma/client";

export async function dbGetLeadWithContext(leadId: string) {
  return await prisma.lead.findUnique({
    where: { id: leadId },
    select: {
      id: true,
      name: true,
      email: true,
      phone: true,
      stage: true,
      status: true,
      assignedTo: {
        select: {
          id: true,
          name: true,
        },
      },
      createdAt: true,
    },
  });
}

export async function dbGetRecentActivities(leadId: string, limit = 20) {
  return await prisma.activity.findMany({
    where: { leadId },
    orderBy: { createdAt: "desc" },
    take: limit,
    select: {
      id: true,
      type: true,
      content: true,
      createdAt: true,
      actor: {
        select: { name: true },
      },
    },
  });
}

export async function dbGetNextReminder(leadId: string) {
  return await prisma.reminder.findFirst({
    where: {
      leadId,
      status: "PENDING",
      dueAt: {
        gte: new Date(),
      },
    },
    orderBy: { dueAt: "asc" },
    select: {
      id: true,
      title: true,
      note: true,
      dueAt: true,
    },
  });
}

export async function dbCreateLeadBrief(
  request: SaveLeadBriefRequest,
  user: Profile,
) {
  return await prisma.aILeadBrief.create({
    data: {
      leadId: request.leadId,
      brief: request.brief,
      createdById: user.id,
    },
  });
}

export async function dbGetLastLeadBrief(leadId: string) {
  return await prisma.aILeadBrief.findFirst({
    where: { leadId },
    orderBy: { createdAt: "desc" },
    select: {
      id: true,
      leadId: true,
      brief: true,
      createdAt: true,
      updatedAt: true,
      createdById: true,
    },
  });
}
```

**Key design — Context fetching is separate from AI logic:**

The `db.ts` file fetches data. The `helpers.ts` file formats it into prompts. The `service.ts` file orchestrates them. This separation means if you need lead context for a different feature later, you reuse `dbGetLeadWithContext` — you don't duplicate the query.

**Key design — `select` limits the fields returned:**

We only fetch the fields the prompt needs. Less data means fewer tokens and faster AI responses. We don't `include` the full lead — just the specific fields that go into the prompt string.

**Key design — Brief persistence is separate from generation:**

`dbCreateLeadBrief` and `dbGetLastLeadBrief` handle the persistence layer. The user generates a brief, previews it, and then explicitly saves it. Generation and saving are two distinct operations.

---

**Reference file:** `src/services/ai/helpers.ts`

Prompt building helpers that assemble database context into structured prompts.

```typescript
import { CallOutcome } from "../activity";
import {
  dbGetLeadWithContext,
  dbGetNextReminder,
  dbGetRecentActivities,
} from "./db";
import { UserSnapshot } from "@/utils/types/user";

type LeadContext = Awaited<ReturnType<typeof dbGetLeadWithContext>>;
type RecentActivities = Awaited<ReturnType<typeof dbGetRecentActivities>>;
type NextReminder = Awaited<ReturnType<typeof dbGetNextReminder>>;

function formatLeadContext(leadContext: LeadContext) {
  if (!leadContext) return "No lead context found";

  return `
  Name: ${leadContext.name}
  Email: ${leadContext.email}
  Phone: ${leadContext.phone}
  Stage: ${leadContext.stage}
  Status: ${leadContext.status}
  Assigned to: ${leadContext.assignedTo?.name ?? "Unassigned"}
  Created: ${leadContext.createdAt.toLocaleDateString("en-US")}
  `;
}

function formatRecentActivities(recentActivities: RecentActivities) {
  if (recentActivities.length === 0) return "No recent activities found";

  return recentActivities
    .map((a) => {
      const date = a.createdAt.toLocaleDateString("en-US");
      const content = a.content || "(no content)";
      return `[${date}] ${a.actor.name} — ${a.type}: ${content}`;
    })
    .join("\n");
}

function formatNextReminder(nextReminder: NextReminder) {
  if (!nextReminder) return "No upcoming reminder found";

  const dueDate = nextReminder.dueAt.toLocaleDateString("en-US");
  return `${nextReminder.title} (due ${dueDate}): ${nextReminder.note || "No additional notes"}`;
}

export function buildLeadBriefPrompt(args: {
  leadContext: LeadContext;
  recentActivities: RecentActivities;
  nextReminder: NextReminder;
}): string {
  const { leadContext, recentActivities, nextReminder } = args;
  const leadContextText = formatLeadContext(leadContext);
  const recentActivitiesText = formatRecentActivities(recentActivities);
  const nextReminderText = formatNextReminder(nextReminder);

  return `You are a CRM operations assistant. Generate a structured brief for a sales agent based ONLY on the data provided below. Do not invent facts. If information is missing or unclear, state "Not enough data to assess."

=== LEAD DATA ===
${leadContextText}

=== RECENT ACTIVITY (Most Recent First) ===
${recentActivitiesText}

=== NEXT REMINDER ===
${nextReminderText}

=== YOUR TASK ===
Generate a brief for the agent. Include:
1. A summary of the lead and current situation
2. Key facts the agent should know (based on activity patterns and stage)
3. Potential risks or concerns
4. Recommended next actions with clear reasoning
5. Specific questions to ask on the next call

Focus on actionable insights. Be specific. Use only the data provided.`;
}

export function buildCallFollowupPrompt(args: {
  leadContext: LeadContext;
  recentActivities: RecentActivities;
  callOutcome: CallOutcome;
  agentNotes?: string | null;
}): string {
  const { leadContext, recentActivities, callOutcome, agentNotes } = args;
  const leadContextText = formatLeadContext(leadContext);
  const recentActivitiesText = formatRecentActivities(recentActivities);

  return `You are a sales coach. A sales agent just completed a call with a lead. Based ONLY on the data provided below, suggest a follow-up strategy. Do not invent facts about the lead or their needs.

=== LEAD DATA ===
${leadContextText}

=== RECENT ACTIVITY ===
${recentActivitiesText}

=== THIS CALL ===
Outcome: ${callOutcome}
Agent Notes: ${agentNotes || "None provided"}

=== YOUR TASK ===
Suggest:
1. A personalized opening for the next call (reference something from this call or history)
2. Three specific follow-up questions based on what was discussed
3. Two common objections this lead might raise, and how to handle them
4. The single most important next step for the agent
5. A suggested reminder (due date in YYYY-MM-DD format, title, and context)

Be specific. Reference details from the call and history. Adapt to the lead's stage and situation.`;
}

export const validateLeadAccess = async (
  assignedToId: string | null | undefined,
  userSnapshot: UserSnapshot,
) => {
  if (["ADMIN", "MANAGER"].includes(userSnapshot.role)) {
    return true;
  }
  if (userSnapshot.id === assignedToId) {
    return true;
  }
  return false;
};
```

**Key concept — `Awaited<ReturnType<>>` for inferred types:**

```typescript
type LeadContext = Awaited<ReturnType<typeof dbGetLeadWithContext>>;
```

Instead of writing manual interfaces that mirror the Prisma `select` shape, we infer the types directly from the database functions. If you add a field to the `select` in `dbGetLeadWithContext`, the `LeadContext` type updates automatically. No duplication, no drift.

**Key concept — Grounded prompts:**

This is the most important prompt engineering principle in the session. The prompt:

1. **Gives context** — "You are a CRM operations assistant... generate a brief..."
2. **Provides all data** — lead fields, activities, reminders, formatted as structured text
3. **Sets explicit rules** — "based ONLY on the data provided" and "if information is missing, state it"
4. **Prevents hallucination** — "Do not invent facts"

When you test this with a lead that has minimal history, the AI will say "Not enough data to assess" instead of making things up. That's the grounding working.

**Key concept — Separate formatting functions:**

Each data type has its own formatting function: `formatLeadContext`, `formatRecentActivities`, `formatNextReminder`. This keeps the prompt builders readable and makes each formatter reusable. The call follow-up prompt reuses `formatLeadContext` and `formatRecentActivities` — no copy-paste.

**Key concept — `CallOutcome` imported from activity service:**

```typescript
import { CallOutcome } from "../activity";
```

The call follow-up prompt builder takes a `CallOutcome` type (the same enum values from Session 3: `NO_ANSWER`, `ANSWERED`, `WRONG_NUMBER`, `BUSY`, `CALL_BACK_LATER`). This ensures the prompt receives the correct values — no string typos.

**Key design — `validateLeadAccess` shared across services:**

The access check (`ADMIN`/`MANAGER` can access any lead, `AGENT` only their own) follows the same pattern as the reminder service. It takes a `UserSnapshot` (`{ id, role }`) and returns a boolean.

---

**Reference file:** `src/services/ai/service.ts`

Orchestration layer that fetches context, builds prompts, and calls the AI.

```typescript
import {
  dbCreateLeadBrief,
  dbGetLastLeadBrief,
  dbGetLeadWithContext,
  dbGetNextReminder,
  dbGetRecentActivities,
} from "./db";
import { buildLeadBriefPrompt, validateLeadAccess } from "./helpers";
import { generateText, Output } from "ai";
import { leadBriefSchema, SaveLeadBriefRequest } from "./schema";
import { createAIActivity } from "../activity/service";
import { LeadServiceError } from "../lead/service";
import { ActivityType } from "@/generated/prisma/client";
import { Profile } from "@/generated/prisma/client";

export async function generateLeadBrief(leadId: string, user: Profile) {
  const lead = await dbGetLeadWithContext(leadId);
  if (!lead) {
    throw new Error("Lead not found");
  }

  if (!(await validateLeadAccess(lead.assignedTo?.id, user))) {
    throw new Error("You are not authorized to access this lead");
  }

  const [activities, nextReminder] = await Promise.all([
    dbGetRecentActivities(leadId),
    dbGetNextReminder(leadId),
  ]);

  const prompt = buildLeadBriefPrompt({
    leadContext: lead,
    recentActivities: activities,
    nextReminder: nextReminder,
  });

  const { output } = await generateText({
    model: "deepseek/deepseek-v3.2-thinking",
    output: Output.object({ schema: leadBriefSchema }),
    prompt,
  });

  await createAIActivity({
    type: ActivityType.AI_LEAD_BRIEF_GENERATED,
    leadId: leadId,
    actorId: user.id,
    content: `Lead brief generated by ${user.name}`,
  });

  return output;
}

export async function saveLeadBrief(
  request: SaveLeadBriefRequest,
  user: Profile,
) {
  const lead = await dbGetLeadWithContext(request.leadId);
  if (!lead) {
    throw new Error("Lead not found");
  }

  if (!(await validateLeadAccess(lead.assignedTo?.id, user))) {
    throw new Error("You are not authorized to access this lead");
  }

  const brief = await dbCreateLeadBrief(request, user);
  return brief;
}

export async function getLastLeadBrief(leadId: string, user: Profile) {
  const lead = await dbGetLeadWithContext(leadId);
  if (!lead) {
    throw new LeadServiceError("Lead not found", 404);
  }

  if (!(await validateLeadAccess(lead.assignedTo?.id, user))) {
    throw new LeadServiceError("Unauthorized", 403);
  }

  const row = await dbGetLastLeadBrief(leadId);
  return row ?? null;
}
```

**Key concept — `generateText` with `Output.object()` is the core API:**

```typescript
const { output } = await generateText({
  model: "deepseek/deepseek-v3.2-thinking",
  output: Output.object({ schema: leadBriefSchema }),
  prompt,
});
```

Three inputs, one output:

1. **`model`** — which AI model to call (as a simple string like `"deepseek/deepseek-v3.2-thinking"`). The Vercel AI Gateway resolves this to the right provider automatically.
2. **`output: Output.object({ schema: ... })`** — the Zod schema that defines the expected output shape
3. **`prompt`** — the grounded text with all context and rules

The SDK sends the prompt + serialized schema to the model, validates the response against the schema, and returns a fully typed object. If validation fails, it throws.

**Key design — Auto-logging the AI activity:**

```typescript
await createAIActivity({
  type: ActivityType.AI_LEAD_BRIEF_GENERATED,
  leadId: leadId,
  actorId: user.id,
  content: `Lead brief generated by ${user.name}`,
});
```

Every time a brief is generated, the service automatically logs an `AI_LEAD_BRIEF_GENERATED` activity to the lead's timeline. This happens on generation, not on save — so the team can see that someone used the AI feature even if they didn't save the result. The `createAIActivity` function is imported from the existing activity service.

**Key design — Three separate operations:**

The service exposes three functions, each with a clear responsibility:

1. `generateLeadBrief` — calls AI, logs activity, returns ephemeral brief
2. `saveLeadBrief` — persists a brief to the `AILeadBrief` table
3. `getLastLeadBrief` — retrieves the most recently saved brief for a lead

This separation means generating and saving are independent actions. The user can generate multiple briefs before deciding which to save — or choose not to save at all.

**Key design — `Profile` instead of `UserSnapshot`:**

The service functions take the full `Profile` type from Prisma, not a `UserSnapshot` subset. This gives access to `user.name` for the activity log content. The `validateLeadAccess` helper still receives only `{ id, role }` internally.

**Key design — `Promise.all` for parallel data fetching:**

```typescript
const [activities, nextReminder] = await Promise.all([
  dbGetRecentActivities(leadId),
  dbGetNextReminder(leadId),
]);
```

Two independent queries run in parallel instead of sequentially. The lead is fetched first (because we need to check access), then activities and reminders fetch in parallel.

---

**Reference file:** `src/services/ai/index.ts`

Barrel exports follow the same pattern as `ActivityService` and `ReminderService`:

```typescript
import { generateLeadBriefSchema, saveLeadBriefSchema } from "./schema";
import { generateLeadBrief, getLastLeadBrief, saveLeadBrief } from "./service";

export const AIService = {
  generateLeadBrief,
  saveLeadBrief,
  getLastLeadBrief,
} as const;

export const AISchema = {
  generateLeadBrief: generateLeadBriefSchema,
  saveLeadBrief: saveLeadBriefSchema,
} as const;
```

**Key design — `as const` on the export objects:**

Same pattern as `ActivityService`, `ReminderService`. Route handlers import `AIService` and `AISchema` from `@/services/ai`. They never reach into internal files.

---

### Phase 4: Lead Brief — Route Handlers

**Concepts covered:** Thin route handlers, three API endpoints for a complete generate-preview-save flow.

**What we did:**

Created three routes: generate a brief, save a brief, and retrieve the last saved brief.

**Reference file:** `src/app/api/ai/lead-brief/route.ts`

```typescript
import { AISchema, AIService } from "@/services/ai";
import { authenticateUser } from "@/utils/authenticateUser";
import { handleRouteError } from "@/utils/handleRouteError";
import { NextRequest, NextResponse } from "next/server";

export async function POST(req: NextRequest) {
  try {
    const profile = await authenticateUser();
    const body = await req.json();

    const { leadId } = AISchema.generateLeadBrief.parse(body);

    const brief = await AIService.generateLeadBrief(leadId, profile);

    return NextResponse.json({ success: true, data: brief });
  } catch (error) {
    return handleRouteError(error);
  }
}
```

The route is minimal: authenticate → validate → call service → respond. The `profile` (full Prisma Profile object) is passed directly to the service — the service uses `profile.id`, `profile.role`, and `profile.name` internally.

**Reference file:** `src/app/api/ai/lead-brief/save/route.ts`

```typescript
import { AISchema, AIService } from "@/services/ai";
import { authenticateUser } from "@/utils/authenticateUser";
import { handleRouteError } from "@/utils/handleRouteError";
import { NextRequest, NextResponse } from "next/server";

export async function POST(req: NextRequest) {
  try {
    const profile = await authenticateUser();
    const body = await req.json();

    const data = AISchema.saveLeadBrief.parse(body);

    const brief = await AIService.saveLeadBrief(data, profile);

    return NextResponse.json({ success: true, data: brief });
  } catch (error) {
    return handleRouteError(error);
  }
}
```

**Key design — The save route validates the brief against the schema:**

`AISchema.saveLeadBrief.parse(body)` validates that the incoming brief matches `leadBriefSchema` exactly. This prevents the client from sending a manually crafted or modified object that doesn't match the AI's output shape.

**Reference file:** `src/app/api/leads/[id]/lead-brief/route.ts`

```typescript
import { AIService } from "@/services/ai";
import { leadIdParamsSchema } from "@/services/lead/schema";
import { authenticateUser } from "@/utils/authenticateUser";
import { handleRouteError } from "@/utils/handleRouteError";
import { NextRequest, NextResponse } from "next/server";

export async function GET(
  _request: NextRequest,
  { params }: { params: Promise<{ id: string }> },
) {
  try {
    const profile = await authenticateUser();
    const { id } = leadIdParamsSchema.parse(await params);

    const leadBrief = await AIService.getLastLeadBrief(id, profile);

    return NextResponse.json({ success: true, data: { leadBrief } });
  } catch (error) {
    return handleRouteError(error);
  }
}
```

**Key design — GET route uses the lead ID from URL params:**

The get-brief route is nested under `/api/leads/[id]/lead-brief`, following the REST convention: the brief belongs to the lead. The lead ID comes from URL params (validated with `leadIdParamsSchema`), not from the request body.

---

### Phase 5: Lead Brief — TanStack Hooks

**Concepts covered:** Mutations for generation and saving, queries for fetching saved briefs, cache invalidation strategy.

**What we did:**

Created three TanStack hooks in a single `useAI.ts` file.

**Reference file:** `src/lib/tanstack/useAI.ts`

```typescript
import { LeadBrief } from "@/services/ai/schema";
import { api } from "../api";
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";

export function useGenerateLeadBrief(leadId: string) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (): Promise<LeadBrief> => {
      const { data } = await api.post("/ai/lead-brief", { leadId });
      return data.data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["activities"] });
    },
  });
}

export function useSaveBrief(leadId: string) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (brief: LeadBrief) => {
      const { data } = await api.post(`/ai/lead-brief/save`, {
        leadId,
        brief,
      });
      return data.data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["brief", leadId] });
    },
  });
}

export function useGetBrief(leadId: string) {
  return useQuery({
    queryKey: ["brief", leadId],
    queryFn: async (): Promise<{
      leadBrief: {
        id: string;
        leadId: string;
        brief: LeadBrief;
        createdById: string;
        createdAt: Date;
        updatedAt: Date;
      } | null;
    }> => {
      const { data } = await api.get(`/leads/${leadId}/lead-brief`);
      return data.data;
    },
  });
}
```

**Key design — `useGenerateLeadBrief` invalidates activities:**

```typescript
onSuccess: () => {
  queryClient.invalidateQueries({ queryKey: ["activities"] });
},
```

Since generating a brief also creates an `AI_LEAD_BRIEF_GENERATED` activity in the timeline, we invalidate the activities cache so the timeline tab refreshes automatically.

**Key design — `useSaveBrief` invalidates the brief cache:**

```typescript
onSuccess: () => {
  queryClient.invalidateQueries({ queryKey: ["brief", leadId] });
},
```

After saving, we invalidate `["brief", leadId]` so `useGetBrief` refetches and shows the newly saved brief in the main view.

**Key design — `useGetBrief` is a query, not a mutation:**

Unlike the generate and save hooks (which are mutations — they cause side effects), `useGetBrief` is a `useQuery` hook. It automatically fetches the last saved brief when the AI tab mounts and caches the result. This means if the user switches tabs and comes back, the brief is still there without re-fetching.

---

### Phase 6: Lead Brief — UI Components

**Concepts covered:** Three-state UI, preview dialog, BriefContent as reusable component, disclaimer pattern.

**What we did:**

Built the AI tab with a generate-preview-save flow and a separate `BriefContent` component for rendering the structured brief.

**Reference file:** `src/components/leads/lead-details/AI.tsx`

```tsx
import { Alert, AlertDescription } from "@/components/ui/alert";
import { Button } from "@/components/ui/button";
import {
  useGenerateLeadBrief,
  useGetBrief,
  useSaveBrief,
} from "@/lib/tanstack/useAI";
import { AlertTriangle, Loader2, RefreshCw } from "lucide-react";
import { useState } from "react";
import { BriefContent } from "./BriefContent";
import {
  Dialog,
  DialogContent,
  DialogFooter,
  DialogHeader,
  DialogTitle,
} from "@/components/ui/dialog";

export const AI = ({ leadId }: { leadId: string }) => {
  const [error, setError] = useState<string | null>(null);
  const [isGeneratedBriefOpen, setIsGeneratedBriefOpen] = useState(false);
  const generateBrief = useGenerateLeadBrief(leadId);
  const saveBrief = useSaveBrief(leadId);
  const { data: briefResponse, isPending: isLoadingBrief } =
    useGetBrief(leadId);
  const brief = briefResponse?.leadBrief?.brief;

  const generatedBrief = generateBrief.data;

  const handleGenerate = () => {
    setError(null);
    generateBrief.mutate(undefined, {
      onError: (error) => {
        setError(error.message);
      },
      onSuccess: () => {
        setIsGeneratedBriefOpen(true);
      },
    });
  };

  const handleSave = () => {
    if (!generatedBrief) return;
    saveBrief.mutate(generatedBrief, {
      onError: (error) => {
        setError(error.message);
      },
      onSuccess: () => {
        setIsGeneratedBriefOpen(false);
      },
    });
  };

  const isPending = generateBrief.isPending;

  if (isLoadingBrief) {
    return (
      <div className="flex items-center gap-2 p-4">
        <Loader2 className="h-4 w-4 animate-spin" />
        <span className="text-sm text-muted-foreground">Loading Brief...</span>
      </div>
    );
  }

  return (
    <>
      {!brief ? (
        // Initial state: Generate button
        <div className="space-y-4 p-4">
          <p className="text-sm text-muted-foreground">
            AI will analyze this lead&apos;s activity history and suggest
            actions.
          </p>
          <Button onClick={handleGenerate} disabled={generateBrief.isPending}>
            {generateBrief.isPending ? (
              <>
                <Loader2 className="mr-2 h-4 w-4 animate-spin" />
                Generating Brief...
              </>
            ) : (
              "Generate Lead Brief"
            )}
          </Button>
          {error && (
            <Alert variant="destructive">
              <AlertTriangle className="h-4 w-4" />
              <AlertDescription>{error}</AlertDescription>
            </Alert>
          )}
        </div>
      ) : (
        // Brief exists: show it with regenerate option
        <div className="space-y-4 p-4">
          {/* Header with actions */}
          <div className="flex items-center justify-between">
            <h3 className="text-lg font-semibold">AI-Generated Brief</h3>
            <div className="flex gap-2">
              <Button
                variant="outline"
                size="sm"
                onClick={handleGenerate}
                disabled={isPending}
              >
                <RefreshCw className="mr-2 h-4 w-4" />
                {isPending ? "Generating..." : "Regenerate"}
              </Button>
            </div>
          </div>

          {/* Disclaimer */}
          <Alert>
            <AlertTriangle className="h-4 w-4" />
            <AlertDescription>
              AI suggestions can be wrong. Always verify before taking action.
            </AlertDescription>
          </Alert>

          {/* Brief sections */}
          <BriefContent brief={brief} />

          {/* Error */}
          {error && (
            <Alert variant="destructive">
              <AlertTriangle className="h-4 w-4" />
              <AlertDescription>{error}</AlertDescription>
            </Alert>
          )}
        </div>
      )}

      {/* Dialog for previewing generated brief before saving */}
      <Dialog
        open={isGeneratedBriefOpen}
        onOpenChange={setIsGeneratedBriefOpen}
      >
        <DialogContent className="max-w-2xl max-h-[80vh] overflow-y-auto">
          <DialogHeader>
            <DialogTitle>AI-Generated Brief</DialogTitle>
          </DialogHeader>
          {generatedBrief ? (
            <div className="space-y-4">
              <BriefContent brief={generatedBrief} />
              <DialogFooter>
                <Button onClick={handleSave} disabled={saveBrief.isPending}>
                  {saveBrief.isPending ? "Saving..." : "Save Brief"}
                </Button>
              </DialogFooter>
            </div>
          ) : (
            <div className="flex items-center gap-2">
              <Loader2 className="h-4 w-4 animate-spin" />
              <span>Generating Brief...</span>
            </div>
          )}
        </DialogContent>
      </Dialog>
    </>
  );
};
```

**Key design — Three-state UI flow:**

1. **Loading** — `useGetBrief` is fetching the last saved brief. Shows a spinner.
2. **Initial** — no saved brief exists. Shows a "Generate Lead Brief" button with explanatory text.
3. **Saved** — a previously saved brief exists. Shows the structured brief with disclaimer and a "Regenerate" button.

**Key design — Preview Dialog before saving:**

When the user clicks "Generate Lead Brief" (or "Regenerate"), the mutation runs and on success opens a Dialog with the generated brief rendered inside `BriefContent`. The user reviews the output and clicks "Save Brief" to persist it — or closes the dialog to discard it. This is the human-in-the-loop pattern: AI generates, the user reviews and decides.

**Key design — The disclaimer pattern:**

```tsx
<Alert>
  <AlertTriangle className="h-4 w-4" />
  <AlertDescription>
    AI suggestions can be wrong. Always verify before taking action.
  </AlertDescription>
</Alert>
```

This disclaimer appears above every piece of AI-generated content. It's both a UX best practice (sets user expectations) and a product responsibility pattern.

**Reference file:** `src/components/leads/lead-details/BriefContent.tsx`

```tsx
import { LeadBrief } from "@/services/ai/schema";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { AlertTriangle } from "lucide-react";

export function BriefContent({ brief }: { brief: LeadBrief }) {
  if (!brief) return null;
  return (
    <div className="space-y-4">
      {/* Summary */}
      <Card>
        <CardHeader>
          <CardTitle className="text-sm">Summary</CardTitle>
        </CardHeader>
        <CardContent>
          <p className="text-sm text-muted-foreground">{brief.summary}</p>
        </CardContent>
      </Card>

      {/* Key Facts */}
      {brief.keyFacts.length > 0 && (
        <Card>
          <CardHeader>
            <CardTitle className="text-sm">Key Facts</CardTitle>
          </CardHeader>
          <CardContent>
            <div className="flex flex-wrap gap-2">
              {brief.keyFacts.map((fact, i) => (
                <span
                  key={i}
                  className="inline-block rounded bg-blue-100 px-2 py-1 text-xs text-blue-900"
                >
                  {fact}
                </span>
              ))}
            </div>
          </CardContent>
        </Card>
      )}

      {/* Risks */}
      {brief.risks.length > 0 && (
        <Card>
          <CardHeader>
            <CardTitle className="text-sm">Risks</CardTitle>
          </CardHeader>
          <CardContent>
            <ul className="space-y-2">
              {brief.risks.map((risk, i) => (
                <li key={i} className="flex gap-3 text-sm text-yellow-800">
                  <AlertTriangle className="mt-0.5 h-4 w-4 shrink-0 text-yellow-600" />
                  <span>{risk}</span>
                </li>
              ))}
            </ul>
          </CardContent>
        </Card>
      )}

      {/* Next Actions */}
      {brief.nextActions.length > 0 && (
        <Card>
          <CardHeader>
            <CardTitle className="text-sm">Recommended Next Actions</CardTitle>
          </CardHeader>
          <CardContent className="space-y-3">
            {brief.nextActions.map((action, i) => (
              <div key={i} className="rounded border p-3">
                <p className="text-sm font-medium">{action.action}</p>
                <p className="mt-1 text-xs text-muted-foreground">
                  {action.why}
                </p>
                {action.suggestedDueAt && (
                  <p className="mt-2 text-xs text-muted-foreground">
                    Suggested: {action.suggestedDueAt}
                  </p>
                )}
              </div>
            ))}
          </CardContent>
        </Card>
      )}

      {/* Questions to Ask */}
      {brief.questionsToAskNext.length > 0 && (
        <Card>
          <CardHeader>
            <CardTitle className="text-sm">Questions to Ask Next</CardTitle>
          </CardHeader>
          <CardContent>
            <ol className="space-y-2">
              {brief.questionsToAskNext.map((question, i) => (
                <li key={i} className="flex gap-3 text-sm">
                  <span className="shrink-0 font-semibold text-muted-foreground">
                    {i + 1}.
                  </span>
                  <span>{question}</span>
                </li>
              ))}
            </ol>
          </CardContent>
        </Card>
      )}
    </div>
  );
}
```

**Key design — `BriefContent` as a separate reusable component:**

`BriefContent` is extracted into its own file because it's used in two places: the saved brief view (inside the AI tab) and the preview dialog. Same data, same rendering — no duplication.

**Key design — Summary as paragraph, key facts as badges:**

The summary renders as a `<p>` (it's a single string). Key facts render as inline badges (`bg-blue-100`). Risks use warning icons. Next actions use bordered cards with reasoning. Questions use numbered lists. Each section has a distinct visual treatment that helps users scan quickly.

---

**Integration into lead detail:**

**Reference file:** `src/components/leads/lead-detail-client.tsx`

The AI tab was added to the existing lead detail tabs:

```tsx
<TabsList>
  <TabsTrigger value="overview">Overview</TabsTrigger>
  <TabsTrigger value="timeline">Activities</TabsTrigger>
  <TabsTrigger value="reminders">Reminders</TabsTrigger>
  <TabsTrigger value="ai">AI</TabsTrigger>
  <TabsTrigger value="files">Files</TabsTrigger>
</TabsList>

<TabsContent value="ai">
  <AI leadId={id} />
</TabsContent>
```

The `AI` component receives only the `leadId` prop and manages its own data fetching internally.

---

### Phase 7: Prompt Engineering Discussion

**Concepts covered:** Why grounding matters, structured vs free text, prompt iteration, the disclaimer pattern.

**What we discussed (not live-coded):**

**Why grounding matters:**

- Bad prompt: "What should the agent do next?" — the model invents suggestions from training data
- Good prompt: "Based ONLY on the activity history and current stage provided above, what should the agent do next?" — the model sticks to facts

**Why structured outputs beat free text:**

- Without Zod: you get markdown like "Here are 3 questions: (1) X (2) Y (3) Z" — you have to parse this string
- With Zod: you get `{ questions: ["X", "Y", "Z"] }` — typed, predictable, machine-readable

**How to iterate on prompts:**

1. Generate briefs for a lead with lots of history → expect detailed output
2. Generate for a brand-new lead → expect "Not enough data" gracefully
3. Try call follow-ups for different outcomes (ANSWERED vs NO_ANSWER) → scripts should differ
4. If output feels generic, add more context to the prompt and test again

**The disclaimer pattern:**

Every AI-generated output includes: "AI suggestions can be wrong. Always verify before taking action." This is both a UX best practice (sets expectations) and product responsibility. The AI is advisory, not authoritative.

---

## What We Did NOT Build (Your Assignment)

We built the Lead Brief feature end-to-end during the session: schema, service, three routes, three TanStack hooks, UI with preview dialog, and database persistence. The Call Follow-up schema is defined but the service, route, hooks, and UI are **not implemented**.

Your assignment is to build the complete Call Follow-up feature using the same patterns you saw in the Lead Brief implementation, plus test and refine both features.

---

## Supplementary Videos

### Review What We Covered

**Vercel AI SDK — Structured Data**
[Vercel AI SDK — Generating Structured Data](https://sdk.vercel.ai/docs/ai-sdk-core/generating-structured-data) — Documentation page
The core API we used: pass a Zod schema + prompt to `generateText()` with `Output.object()`, get back a typed object. Read through the examples if anything felt unclear.

**Prompt Engineering Guide**
[OpenAI Prompt Engineering Guide](https://platform.openai.com/docs/guides/prompt-engineering) — Best practices for structured prompting, grounding, and preventing hallucination.

---

## Assignment: Build Call Follow-up + Test & Refine

You have the Lead Brief feature working end-to-end. Your assignment is to build the Call Follow-up Assistant, stress-test both features, and prepare for Session 6.

### Task 1: Call Follow-up Service Function (estimated: 1-1.5 hours)

Build the `generateCallFollowup` function in `src/services/ai/service.ts`.

**What to implement:**

The function should follow the exact same pattern as `generateLeadBrief`:

1. Take `leadId`, `callOutcome`, `agentNotes`, and `user` (Profile) as parameters
2. Fetch the lead with `dbGetLeadWithContext`
3. Check access with `validateLeadAccess`
4. Fetch recent activities with `dbGetRecentActivities` (use 10 instead of 20 — the follow-up is more focused on recent context)
5. Build the prompt with `buildCallFollowupPrompt` (already implemented in `helpers.ts`)
6. Call `generateText` with `Output.object({ schema: callFollowUpSchema })`
7. Return the typed output

```typescript
export async function generateCallFollowup(
  leadId: string,
  callOutcome: CallOutcome,
  agentNotes: string | undefined,
  user: Profile,
) {
  // 1. Fetch lead and check access
  // 2. Fetch recent activities (limit 10)
  // 3. Build prompt
  // 4. Call generateText with Output.object({ schema: callFollowUpSchema })
  // 5. Return output
}
```

**Hints:**

- Import `CallOutcome` from `"../activity"` — this is the same enum from Session 3
- Import `callFollowUpSchema` from `"./schema"` — it's already defined
- Import `buildCallFollowupPrompt` from `"./helpers"` — it's already implemented
- Follow the exact pattern of `generateLeadBrief` — the only differences are: fewer activities (10 vs 20), no next reminder fetch, and the call outcome/notes are passed to the prompt builder

**Update the index:** Add `generateCallFollowup` to the `AIService` export in `src/services/ai/index.ts`. Also add a request validation schema:

```typescript
// In schema.ts
export const generateCallFollowUpRequestSchema = z.object({
  leadId: z.uuid(),
  callOutcome: z.enum([
    "NO_ANSWER",
    "ANSWERED",
    "WRONG_NUMBER",
    "BUSY",
    "CALL_BACK_LATER",
  ]),
  agentNotes: z.string().trim().max(5000).optional(),
});
```

### Task 2: Call Follow-up Route Handler (estimated: 30 min)

Create the route at `src/app/api/ai/call-followup/route.ts`.

**What to implement:**

Follow the exact same pattern as `POST /api/ai/lead-brief`:

```typescript
// POST /api/ai/call-followup
export async function POST(req: NextRequest) {
  try {
    const profile = await authenticateUser();
    const body = await req.json();

    // Parse with generateCallFollowUpRequestSchema
    // Call AIService.generateCallFollowup(leadId, callOutcome, agentNotes, profile)
    // Return { success: true, data: followup }
  } catch (error) {
    return handleRouteError(error);
  }
}
```

### Task 3: Call Follow-up TanStack Hook (estimated: 15 min)

Add the hook to `src/lib/tanstack/useAI.ts`:

```typescript
export function useGenerateCallFollowup(leadId: string) {
  return useMutation({
    mutationFn: async (args: {
      callOutcome: string;
      agentNotes?: string;
    }): Promise<CallFollowUp> => {
      const { data } = await api.post("/ai/call-followup", {
        leadId,
        ...args,
      });
      return data.data;
    },
  });
}
```

Import the `CallFollowUp` type from `@/services/ai/schema`.

### Task 4: Call Follow-up Dialog Integration (estimated: 2-2.5 hours)

This is the most substantial task. Modify the existing `LogCallDialog` to support a multi-step flow. After logging a call, the user can optionally get AI-generated follow-up suggestions.

**The multi-step dialog flow:**

```
Step 1: "log"     → User fills in call outcome + notes → call is saved as Activity
Step 2: "suggest" → "Call logged. Want AI to suggest a follow-up?" → loading state
Step 3: "review"  → Show script + next step + suggested reminder → Create Reminder or Discard
```

**What to implement:**

Modify `src/components/leads/lead-details/LogCallDialog.tsx`:

1. Add a `step` state: `useState<"log" | "suggest" | "review">("log")`
2. After the call is logged successfully, transition to `"suggest"` step
3. In the suggest step, show "Get AI Suggestion" button and a "Skip" button
4. When the user clicks "Get AI Suggestion", call `useGenerateCallFollowup`
5. On success, transition to `"review"` step
6. In the review step, show the AI's output in structured cards:
   - **Call Script** card: opening, questions (numbered), objection handlers (bordered cards)
   - **Recommended Next Step** card
   - **Suggested Reminder** card: title, note, due date
7. Show "Create Reminder" and "Discard" buttons
8. "Create Reminder" calls `useCreateLeadReminder` with the suggested data
9. "Discard" closes the dialog
10. Add a disclaimer at the top of the review step

**Hooks you'll need:**

- `useLogCallAttempt(leadId)` — from Session 3 (already exists)
- `useGenerateCallFollowup(leadId)` — from Task 3
- `useCreateLeadReminder(leadId)` — from Session 4 (already exists)

**UI components to use:**

- `Dialog`, `DialogContent`, `DialogHeader`, `DialogTitle`, `DialogFooter` — from shadcn
- `Card`, `CardContent`, `CardHeader`, `CardTitle` — for structured output display
- `Alert`, `AlertDescription` — for the disclaimer
- `Button` — with loading states (`Loader2` icon + `isPending` checks)
- `Sparkles` icon from lucide-react — for the "Get AI Suggestion" button

**Key design decision — Human-in-the-loop:**

The AI proposes a reminder, but the user must explicitly click "Create Reminder" to act on it. The AI doesn't auto-create anything. The call is already logged regardless of whether the user gets an AI suggestion — the suggestion is optional.

**Key design decision — `z.coerce.date()` in the schema:**

The `suggestedReminder.suggestedDueAt` field uses `z.coerce.date()`, which means it's already a JavaScript `Date` object. When passing it to `useCreateLeadReminder`, convert it to ISO string:

```typescript
dueAt: followup.suggestedReminder.suggestedDueAt.toISOString();
```

**Suggested component structure:**

Extract the review step into a separate `FollowupReview` component for readability, similar to how `BriefContent` was extracted from the AI tab.

### Task 5: Test with Multiple Lead Scenarios (estimated: 1-1.5 hours)

Test both AI features with different types of leads:

**Lead Brief Generator:**

- A lead with **rich history** (20+ activities, multiple calls, notes, stage changes) → expect detailed, specific output with actionable next steps
- A lead with **minimal data** (just created, no activities) → expect "Not enough data to assess" for some fields — this is correct behavior, not a bug
- A lead in **different stages** (QUALIFICATION vs NEGOTIATION vs CLOSED_WON) → expect stage-appropriate suggestions

**Call Follow-up Assistant:**

- Log a call with **ANSWERED** outcome + detailed notes → expect a follow-up script referencing the conversation
- Log a call with **NO_ANSWER** outcome → expect a different script (re-attempt, voicemail strategy)
- Log a call with **WRONG_NUMBER** outcome → expect the AI to suggest data verification

For each test, ask: Is the output **specific** or **generic**? Specific is good — it means the prompt is grounded. Generic means the prompt needs more context.

### Task 6: Refine Your Prompts (estimated: 30-45 min)

Based on your testing in Task 5, improve the prompts in `src/services/ai/helpers.ts`:

- If key facts are vague → add more lead fields to the prompt
- If next actions are generic → add explicit instructions like "Reference the most recent activity in your suggestions"
- If call scripts don't adapt to outcomes → add outcome-specific instructions in the prompt
- If the AI invents company details → strengthen the grounding rule: "Do NOT assume any information not explicitly provided"

### Task 7: Watch Pre-Session 6 Videos (estimated: 25 min)

Session 6 introduces **admin features**: user management via email invites and CSV import/export.

#### 1. Supabase Admin API — Server-Side User Management (~10 min)

**What it should cover:** The difference between the Supabase client API (used in the browser, limited permissions) and the Admin API (used on the server with the service role key, full permissions). How to create users programmatically, generate invite links, and manage user accounts from server-side code.

> **Why this matters:** Session 6 builds the admin user management feature. We'll use the Supabase Admin API to create users and send invite emails. Understanding the client vs admin API distinction is critical for security.

**Suggested resource:** [Supabase Auth Admin Docs](https://supabase.com/docs/reference/javascript/auth-admin-createuser) — read through `createUser` and `generateLink` (~10 min).

---

#### 2. CSV Parsing in JavaScript (~10 min)

**What it should cover:** How to parse CSV files in the browser using a library like PapaParse. The basic flow: user selects file → read file contents → parse into rows → validate each row → display results.

> **Why this matters:** Session 6 includes building a CSV import feature. Users upload a CSV of leads, we validate each row, and import the valid ones. Knowing how CSV parsing works lets us focus on the validation and import logic during the session.

**Suggested resource:** [PapaParse Tutorial](https://www.papaparse.com/) — browse the docs and "Demo" page (~10 min).

---

## Checkpoint Before Session 6

Before Session 6, verify all of the following:

- [ ] Lead Brief: open lead → AI tab → Generate Brief → preview dialog shows structured output
- [ ] Lead Brief: Save Brief → dialog closes → saved brief appears in AI tab
- [ ] Lead Brief: Regenerate produces a fresh brief and opens the preview dialog
- [ ] Lead Brief: works with minimal-data leads (no crash, shows "Not enough data" gracefully)
- [ ] Lead Brief: `AI_LEAD_BRIEF_GENERATED` activity appears in the timeline after generation
- [ ] Call Follow-up: `POST /api/ai/call-followup` route returns structured output
- [ ] Call Follow-up: Log Call dialog has the multi-step flow (log → suggest → review)
- [ ] Call Follow-up: "Get AI Suggestion" generates and displays the follow-up script
- [ ] Call Follow-up: script sections (opening, questions, objection handlers) render correctly
- [ ] Call Follow-up: "Create Reminder" pre-fills from AI suggestion and creates a reminder
- [ ] Call Follow-up: "Skip" and "Discard" close without creating a reminder
- [ ] Call Follow-up: different outcomes (ANSWERED vs NO_ANSWER) produce different scripts
- [ ] AI disclaimers visible on all AI-generated content
- [ ] Error states handled (AI service down → user sees helpful message, not a crash)
- [ ] Resend account created with API key ready
- [ ] Pre-Session 6 videos watched (Supabase Admin API, CSV Parsing)
- [ ] Code committed and pushed to GitHub

---

## Topic Reference Table

| Topic                          | File Reference                                       | Phase   |
| ------------------------------ | ---------------------------------------------------- | ------- |
| AI Gateway setup + SDK install | `snippets/01-ai-concepts-and-gateway-setup.md`       | Phase 1 |
| `AILeadBrief` Prisma model     | `prisma/schema.prisma`                               | Phase 2 |
| AI activity types              | `src/services/activity/schema.ts`                    | Phase 2 |
| Lead Brief schema              | `src/services/ai/schema.ts`                          | Phase 3 |
| Call Follow-up schema          | `src/services/ai/schema.ts`                          | Phase 3 |
| Context fetching (db.ts)       | `src/services/ai/db.ts`                              | Phase 3 |
| Prompt builders (helpers.ts)   | `src/services/ai/helpers.ts`                         | Phase 3 |
| Lead Brief service             | `src/services/ai/service.ts`                         | Phase 3 |
| Lead Brief routes              | `src/app/api/ai/lead-brief/`                         | Phase 4 |
| TanStack AI hooks              | `src/lib/tanstack/useAI.ts`                          | Phase 5 |
| AI tab component               | `src/components/leads/lead-details/AI.tsx`           | Phase 6 |
| BriefContent component         | `src/components/leads/lead-details/BriefContent.tsx` | Phase 6 |
| Prompt engineering principles  | `snippets/05-prompt-engineering-discussion.md`       | Phase 7 |

---

## Resources

- [Vercel AI SDK Docs](https://sdk.vercel.ai/docs) — `generateText`, structured output with `Output.object()`, model configuration, error handling
- [Vercel AI SDK — Structured Data](https://sdk.vercel.ai/docs/ai-sdk-core/generating-structured-data) — Zod schema integration with `Output.object()`
- [Zod Docs](https://zod.dev/) — Schema definitions for AI output shapes
- [OpenAI Prompt Engineering Guide](https://platform.openai.com/docs/guides/prompt-engineering) — Grounding, structured prompting
- [Vercel AI Gateway](https://vercel.com/docs/ai-gateway) — Gateway configuration, rate limiting, cost tracking
- [TanStack Query v5 — Mutations](https://tanstack.com/query/v5/docs/react/guides/mutations) — `useMutation`, loading states
- [TanStack Query v5 — Queries](https://tanstack.com/query/v5/docs/react/guides/queries) — `useQuery`, cache management
- [date-fns](https://date-fns.org/) — Date formatting for prompt building

---

_Session 5 Complete. You now understand how to integrate AI as a structured API with typed, validated outputs. The AI features follow the same service-layer pattern as every other feature in the CRM — schema, service, route, UI. The key principle: AI suggests, humans decide. See you in Session 6._
