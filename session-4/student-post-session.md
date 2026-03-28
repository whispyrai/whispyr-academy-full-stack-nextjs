# Session 4: Post-Session Recap & Tutorial

**Reminders, Notifications & Scheduled Jobs — Building a Time-Aware CRM**

---

## What We Built

Session 4 introduced the concept of **time** into the CRM. Up until now, every action was immediate: create a lead, change a status, add a note. In this session, we built the infrastructure for things that happen *later* — scheduled reminders, background job processing, and in-app notifications.

By the end of Session 4, the `whispyr-academy-crm` repo had:

- **Session 3 carryover completion** — the note/call-attempt write APIs, TanStack mutation hooks, and dialog components (AddNoteDialog, LogCallDialog) that were assigned as homework
- **Assignment change wiring** — the lead edit flow now supports changing the assigned agent, with full Activity trail tracking (`ASSIGNMENT_CHANGE` type with `{ from, to }` metadata)
- **Frontend assignment UI** — a Select dropdown on the lead detail Overview tab that lets managers/admins reassign a lead to a different agent
- **Seed script for agents** — a Prisma seed file that creates 5 agent accounts in Supabase Auth + Prisma Profile, so the assignment dropdown has agents to choose from
- **Reminder and Notification database models** — two new Prisma models (`Reminder` and `Notification`) with enums (`ReminderStatus`, `NotificationReadState`) and full relational links to Lead and Profile
- **Reminder service module** — the complete 5-file service pattern (schema, db, service, helpers, index) for creating and firing reminders
- **Notification service module** — a parallel 5-file service module for creating in-app notifications
- **QStash integration** — Upstash QStash client for scheduling delayed HTTP callbacks (the "publish now, call me back later" pattern)
- **Redis integration** — an ioredis client used for idempotency checks on reminder callbacks
- `**POST /api/leads/[id]/reminders`** — API route to create a reminder for a lead, which schedules a QStash callback
- `**POST /api/upstash/reminder-due**` — webhook endpoint that QStash calls when a reminder fires, which creates a Notification and marks the Reminder as FIRED
- **Idempotent callback processing** — Redis-based deduplication ensuring a reminder callback never fires twice, even if QStash delivers the message more than once
- `**useCreateLeadReminder`** — TanStack Query mutation hook for the create reminder flow
- **Notification API routes** — `GET /api/notifications` for listing notifications with pagination and unread count, and `PATCH /api/notifications/[id]/read` for marking a notification as read
- `**useGetNotifications` with polling** — TanStack Query hook that auto-refetches every 5 seconds (`refetchInterval: 1000 * 5`) so new notifications appear without a page refresh
- `**useMarkNotificationRead`** — TanStack mutation hook that marks a notification as read and invalidates the notification cache
- **NotificationBell component** — a popover in the app header that shows a bell icon with an unread count badge, paginated notification list, and click-to-mark-read behavior with lead navigation
- **A throwaway test form** on the lead detail Reminders tab — used during the session to verify the end-to-end pipeline works. This is NOT the final UI.

> **Note:** We built the full reminder creation and firing pipeline (API → QStash → webhook → notification) and verified it with a rough test form. We did NOT build the production Reminders UI — no reminder listing, no status badges, no Create Reminder dialog, and no standalone `/reminders` page. The entire frontend experience and the remaining read/update routes are your **assignment before Session 5**. See the "Assignment" section below.

---

## Step-by-Step Walkthrough

If you missed the session or want to review, follow these steps in order. Each step references the exact files in the `[whispyr-academy-crm](https://github.com/whispyr-academy/whispyr-academy-crm)` repo.

---

### Phase 0: Session 3 Carryover — Note & Call Attempt Write Paths

**What we did:**

Session 4 started by completing the Session 3 assignment: the write paths for adding notes and logging call attempts. If you completed these as homework, your code should look similar. If not, here's what was built.

**Reference files:**

- `src/app/api/leads/[id]/activities/note/route.ts` — POST endpoint for creating notes
- `src/app/api/leads/[id]/activities/call-attempt/route.ts` — POST endpoint for logging call attempts
- `src/services/activity/schema.ts` — New schemas: `createNoteSchema` and `createCallAttemptSchema`
- `src/services/activity/index.ts` — Updated exports to include new schemas and types
- `src/services/activity/helpers.ts` — Updated `buildActivityContent` to handle `NOTE` and `CALL_ATTEMPT` types
- `src/lib/tanstack/useActivities.ts` — New hooks: `useCreateNote` and `useLogCallAttempt`
- `src/components/leads/lead-details/AddNoteDialog.tsx` — Dialog component for adding notes
- `src/components/leads/lead-details/LogCallDialog.tsx` — Dialog component with outcome dropdown and notes textarea

**New validation schemas:**

```typescript
// Note — just a content string
export const createNoteSchema = z.object({
  content: z.string().trim().min(1).max(5000),
});

// Call attempt — structured outcome + optional notes
export const callOutcomeEnum = z.enum([
  "NO_ANSWER",
  "ANSWERED",
  "WRONG_NUMBER",
  "BUSY",
  "CALL_BACK_LATER",
]);

export const createCallAttemptSchema = z.object({
  outcome: callOutcomeEnum,
  notes: z.string().trim().max(5000).optional(),
});
```

**Key design — `buildActivityContent` updated for direct content:**

The original `buildActivityContent` only handled change activities (status, stage, assignment) that carry `{ from, to }` metadata. Notes and call attempts don't have metadata — they carry content directly. The function was updated to check the activity type first:

```typescript
export function buildActivityContent(
  activityType: ActivityType,
  meta: { from: unknown; to: unknown } | undefined,
  content?: string,
) {
  // Notes and call attempts pass content directly
  if (
    activityType === ActivityType.NOTE ||
    activityType === ActivityType.CALL_ATTEMPT
  ) {
    return content ?? null;
  }

  // Change activities use meta to build content
  if (!meta) return null;

  switch (activityType) {
    case ActivityType.STATUS_CHANGE:
      return `Status changed from ${meta.from} to ${meta.to}`;
    // ... other change types
  }
}
```

This is an important design pattern: **different activity types have different content strategies**. Change activities derive their content from metadata. Notes and call attempts receive content directly.

**Call attempt content encoding:**

```typescript
const content = input.notes
  ? `${input.outcome} — ${input.notes}`
  : input.outcome;
```

The outcome and notes are combined into a single string for storage. This keeps the Activity model simple (one `content` field) while encoding structured data. The trade-off is that parsing the outcome back out of the string later would require string manipulation — but for a timeline display, the combined string works well.

**New mutation hooks:**

```typescript
export function useCreateNote(leadId: string) {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: async (content: string) => {
      const { data } = await api.post(`/leads/${leadId}/activities/note`, {
        content,
      });
      return data.data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["activities"] });
    },
  });
}

export function useLogCallAttempt(leadId: string) {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: async (payload: CreateCallAttemptRequest) => {
      const { data } = await api.post(
        `/leads/${leadId}/activities/call-attempt`,
        payload,
      );
      return data.data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["activities"] });
    },
  });
}
```

Both hooks follow the exact same pattern as `useCreateLead` from Session 2: fire a POST, invalidate the cache on success, let TanStack Query refetch the timeline automatically.

---

### Phase 1: Assignment Change — Backend Wiring

**Concepts covered:** Permission functions, change detection for new field types, resolving human-readable names for metadata.

**What we did:**

The lead edit flow already supported changing status and stage. We added support for changing the assigned agent — which required updates to the helpers, permissions, and service layers.

**Reference file:** `src/services/lead/helpers.ts`

Added assignment change detection to `buildLeadChangeActivities`:

```typescript
if (
  newLead.assignedToId &&
  newLead.assignedToId !== existingLead.assignedToId
) {
  activities.push({
    leadId,
    actorId,
    type: ActivityType.ASSIGNMENT_CHANGE,
    meta: {
      from: existingLead.assignedTo?.name ?? "Unassigned",
      to: newLead.assignedTo?.name ?? "Unassigned",
    },
  });
}
```

**Key design — Human-readable metadata:**

Notice that the `meta.from` and `meta.to` for assignment changes are **names** (e.g., "Agent 1"), not IDs. This is intentional. The activity timeline will display "Assignment changed from Agent 1 to Agent 3" — which is immediately readable. If we stored UUIDs, we'd need a separate lookup to resolve names every time we render the timeline.

The trade-off: if the agent's name changes after the activity was created, the activity still shows the old name. For an audit log, this is actually desirable — it records what was true at the time of the change.

**Reference file:** `src/services/lead/permissions.ts`

Added a new permission check:

```typescript
export function canEditLeadAssignment(role: Role, data: EditLeadRequest) {
  if (role !== Role.AGENT) {
    return true;
  }
  return data.assignedToId === undefined;
}
```

Agents cannot reassign leads. Only managers and admins can. This is enforced at the service layer, not the UI — the UI hides the dropdown for agents, but the server also rejects the request if an agent tries.

**Reference file:** `src/services/lead/service.ts`

Added the permission check to the `updateLead` function:

```typescript
if (!canEditLeadAssignment(profile.role, data)) {
  throw new LeadServiceError("Unauthorized", 403);
}
```

**Software engineering principle — Defense in depth (again):**

The UI hides the assignment dropdown from agents. But the server also checks. If someone bypasses the UI (via API client, browser devtools, or a bug), the server catches it. Never rely on the client alone for authorization.

---

### Phase 2: Assignment Change — Frontend UI & Seed Script

**Concepts covered:** Server Component data fetching for dropdown options, Prisma seed scripts, Supabase Admin client for user creation.

**What we did:**

1. Created a seed script that creates 5 agent accounts in both Supabase Auth and Prisma.
2. Updated the lead detail page to fetch all agents and pass them to the client component.
3. Added a Select dropdown on the Overview tab for reassigning leads.

**Reference file:** `prisma/seed/seed-agents.ts`

```typescript
const agents = [
  { name: "Agent 1", email: "agent1@crm.com", password: "agent123" },
  { name: "Agent 2", email: "agent2@crm.com", password: "agent123" },
  // ... Agent 3, 4, 5
];

for (const agent of agents) {
  // Create user in Supabase Auth
  const { data, error } = await supabaseAdmin.auth.admin.createUser({
    email: agent.email,
    password: agent.password,
    email_confirm: true,
  });

  // Create matching Profile in Prisma
  await prisma.profile.create({
    data: {
      id: data.user.id,
      email: agent.email,
      name: agent.name,
      role: "AGENT",
    },
  });
}
```

**Key concept — The Supabase Admin client:**

Notice we use `supabaseAdmin.auth.admin.createUser()` — this is the **admin** client, not the regular client. The admin client has elevated privileges: it can create users without sending confirmation emails (`email_confirm: true`), bypass rate limits, and manage users directly. This is the same client we use in production for admin operations.

The seed script is referenced in `prisma.config.ts` so you can run it with:

```bash
npx prisma db seed
```

**Reference file:** `src/app/(protected)/leads/[id]/page.tsx`

Updated the Server Component to fetch all agents:

```typescript
const users = await prisma.profile.findMany({
  where: { role: Role.AGENT },
});

return <LeadDetailClient id={id} role={profile.role} users={users} />;
```

This query runs on the server and passes the agent list as a prop. The client component doesn't need to make a separate API call to get the agent list — it's already there.

**Reference file:** `src/components/leads/lead-details/Overview.tsx`

Added the Select dropdown for assignment:

```typescript
{isManagerOrAdmin && isEditing ? (
  <Select
    value={selectedAssignedToId ?? undefined}
    onValueChange={(value) =>
      setDraft((currentDraft) => ({
        ...currentDraft!,
        assignedToId: value,
      }))
    }
  >
    <SelectTrigger>
      <SelectValue placeholder="Select an agent" />
    </SelectTrigger>
    <SelectContent>
      {users.map((user) => (
        <SelectItem key={user.id} value={user.id}>
          {user.name}
        </SelectItem>
      ))}
    </SelectContent>
  </Select>
) : null}
```

The Select only renders when `isManagerOrAdmin && isEditing`. The value is stored in the draft state, and when the user clicks Save, the `useEditLead` mutation fires with `assignedToId` included — triggering the full change detection → activity creation pipeline.

**New shadcn component:** `src/components/ui/select.tsx` — installed with `npx shadcn@latest add select`.

---

### Phase 3: Reminder & Notification Database Schema

**Concepts covered:** Prisma schema design for scheduled entities, status enums, nullable fields for deferred values, relational links.

**What we did:**

Added two new models and their enums to the Prisma schema.

**Reference file:** `prisma/schema.prisma`

**Reminder model:**

```prisma
model Reminder {
  id           String         @id @default(uuid())
  leadId       String
  lead         Lead           @relation(fields: [leadId], references: [id])
  assignedToId String
  assignedTo   Profile        @relation(fields: [assignedToId], references: [id])
  title        String
  note         String?
  dueAt        DateTime
  status       ReminderStatus @default(PENDING)
  firedAt      DateTime?
  qstashMessageId String?
  createdAt    DateTime       @default(now())
  updatedAt    DateTime       @updatedAt
}

enum ReminderStatus {
  PENDING
  FIRED
  CANCELLED
}
```

**Notification model:**

```prisma
model Notification {
  id          String                @id @default(uuid())
  recipientId String
  recipient   Profile               @relation(fields: [recipientId], references: [id])
  leadId      String?
  lead        Lead?                 @relation(fields: [leadId], references: [id])
  title       String
  body        String
  readAt      DateTime?
  readState   NotificationReadState @default(UNREAD)
  createdAt   DateTime              @default(now())
}

enum NotificationReadState {
  UNREAD
  READ
}
```

Ran the migration:

```bash
npx prisma migrate dev --name "add_reminder_and_notification_models"
npx prisma generate
```

**Key design — The Reminder lifecycle:**

A Reminder goes through three states:

```
PENDING  →  FIRED  →  (done)
    ↓
CANCELLED  →  (done)
```

1. **PENDING**: Created by a user, scheduled in QStash. Waiting for the due time.
2. **FIRED**: QStash called the webhook at the due time. A Notification was created.
3. **CANCELLED**: The user cancelled the reminder before it fired.

**Key design — `qstashMessageId`:**

This field stores the QStash message ID returned when we schedule the reminder. It's nullable because:

1. The reminder is created first (in the database)
2. Then scheduled in QStash (which returns a message ID)
3. Then the message ID is saved back to the reminder

All three steps happen in a transaction. If QStash scheduling fails, the entire reminder creation is rolled back. The `qstashMessageId` is useful later if we need to cancel the scheduled callback.

**Key design — `firedAt` vs `dueAt`:**

- `dueAt` is when the reminder *should* fire (set by the user)
- `firedAt` is when the reminder *actually* fired (set by the webhook handler)

These might differ slightly due to QStash delivery timing. Having both lets you audit whether reminders are firing on time.

**Key design — Notification's optional `leadId`:**

Not all notifications are about leads. A notification might be a system announcement, a team message, or anything else. Making `leadId` optional keeps the model flexible. When it IS set, it creates a link so the user can click through to the lead.

**Relationships updated on Profile and Lead:**

```prisma
model Profile {
  // ... existing fields
  reminders     Reminder[]
  notifications Notification[]
}

model Lead {
  // ... existing fields
  reminders     Reminder[]
  notifications Notification[]
}
```

---

### Phase 4: QStash & Redis Clients

**Concepts covered:** Serverless job scheduling, message queue patterns, in-memory key-value stores, environment variables for external services.

**What we did:**

Set up client libraries for two external services: Upstash QStash (scheduled job delivery) and Redis (in-memory key-value store).

**Reference file:** `src/lib/qstash.ts`

```typescript
import { Client, Receiver } from "@upstash/qstash";
import { NextRequest } from "next/server";

export const qstash = new Client({
  token: process.env.QSTASH_TOKEN,
});

const qstashReceiver = new Receiver({
  currentSigningKey: process.env.QSTASH_CURRENT_SIGNING_KEY,
  nextSigningKey: process.env.QSTASH_NEXT_SIGNING_KEY,
});

export const verifyQstashSignature = async (
  request: NextRequest,
  rawBody: string,
) => {
  const signature = request.headers.get("Upstash-Signature");
  if (!signature) throw new Error("Upstash-Signature header is required");
  return await qstashReceiver.verify({ signature, body: rawBody });
};

export const reminderCallbackUrl = `${process.env.NEXT_PUBLIC_API_URL}/upstash/reminder-due`;
```

**Key concept — The QStash flow:**

QStash is a message queue designed for serverless environments. Here's the mental model:

```
1. Your app creates a reminder
2. Your app calls qstash.publishJSON({
     url: "https://your-app.com/api/upstash/reminder-due",
     body: { reminderId: "abc123" },
     notBefore: 1711612800  // Unix timestamp (seconds)
   })
3. QStash stores the message and waits
4. At the specified time, QStash sends a POST to your URL
5. Your webhook handler processes the reminder
```

You're not running a cron job. You're not keeping a connection open. You publish a message with a future delivery time, and QStash calls you back. This is perfect for serverless — your app doesn't need to be running between step 2 and step 4.

**Key concept — Signature verification:**

When QStash calls your webhook, how do you know it's really QStash and not someone spoofing the request? QStash signs every request with a cryptographic signature using your signing keys. The `verifyQstashSignature` function checks this signature. If it doesn't match, the request is rejected with a 401.

The `Receiver` uses two signing keys (`currentSigningKey` and `nextSigningKey`) for key rotation — QStash can rotate keys without downtime because your app accepts both the current and next key.

**Reference file:** `src/lib/redis.ts`

```typescript
import Redis from "ioredis";

export const redis = new Redis(process.env.REDIS_URL!);
```

Redis is an in-memory key-value store. We use it for one specific purpose in this session: **idempotency**. When QStash calls our webhook, it guarantees "at-least-once" delivery — meaning it might call us twice for the same reminder. Redis lets us check "have I already processed this reminder?" before doing anything.

**Environment variables needed:**

```env
QSTASH_TOKEN=...
QSTASH_CURRENT_SIGNING_KEY=...
QSTASH_NEXT_SIGNING_KEY=...
NEXT_PUBLIC_API_URL=https://your-app-url.com
REDIS_URL=redis://...
```

**New npm packages:**

```bash
npm install @upstash/qstash ioredis
```

---

### Phase 5: Reminder Service Module

**Concepts covered:** Service module pattern (5 files), transaction orchestration across external services, scheduling with QStash, the "create → schedule → update" pattern.

**What we did:**

Built the complete reminder service following the 5-file pattern from Session 3.

**Reference file:** `src/services/reminder/schema.ts`

```typescript
export const createReminderSchema = z.object({
  title: z.string().min(1),
  note: z.string().optional(),
  dueAt: z.coerce.date().refine((date) => {
    return (
      date.getTime() > new Date().getTime() &&
      date.getTime() < new Date(Date.now() + 1000 * 60 * 60 * 24 * 7).getTime()
    );
  }),
  assignedToId: z.uuid().optional(),
});

export const qstashReminderDueSchema = z.object({
  reminderId: z.uuid(),
});
```

**Key design — Date validation with `.refine()`:**

The `dueAt` field uses `z.coerce.date()` to parse strings into Date objects, then `.refine()` to enforce that the date is in the future and no more than 7 days out. This prevents users from creating reminders in the past or scheduling something months away (which might indicate a mistake).

**Key design — Two schemas:**

- `createReminderSchema` validates user input when creating a reminder
- `qstashReminderDueSchema` validates the payload QStash sends to our webhook

Separating these makes it clear: one schema faces the user, the other faces QStash.

**Reference file:** `src/services/reminder/db.ts`

```typescript
export const dbCreateReminder = async (
  reminder: CreateReminderRequest & { assignedToId: string },
  tx?: Prisma.TransactionClient,
) => {
  const client = tx ?? prisma;
  return client.reminder.create({
    data: {
      title: reminder.title,
      note: reminder.note,
      dueAt: reminder.dueAt,
      leadId: reminder.leadId,
      assignedToId: reminder.assignedToId,
    },
  });
};

export const dbUpdateReminderQstashMessageId = async (
  reminderId: string,
  qstashMessageId: string,
  tx?: Prisma.TransactionClient,
) => {
  const client = tx ?? prisma;
  return client.reminder.update({
    where: { id: reminderId },
    data: { qstashMessageId },
  });
};

export const dbGetReminder = async (reminderId: string, tx?) => {
  // Fetches reminder with lead and assignedTo relations
  // (needed for building the notification content)
};

export const dbUpdateReminderStatus = async (
  reminderId: string,
  status: ReminderStatus,
  tx?,
) => {
  // Updates the status field (PENDING → FIRED or CANCELLED)
};
```

**Reference file:** `src/services/reminder/helpers.ts`

```typescript
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

This helper is used in both the reminder and notification services. Admins and managers can access any lead. Agents can only access leads assigned to them.

**Reference file:** `src/services/reminder/service.ts` — The `createReminder` function

```typescript
export const createReminder = async (
  request: CreateReminderRequest,
  userSnapshot: UserSnapshot,
) => {
  const assignedToId = request.assignedToId ?? userSnapshot.id;

  // Validate the user has access to this lead
  const leadAssignedTo = await dbGetLeadAssignedTo(request.leadId);
  if (!validateLeadAccess(leadAssignedTo?.assignedToId, userSnapshot)) {
    throw new Error("You are not authorized to create a reminder for this lead");
  }

  const reminder = await prisma.$transaction(async (tx) => {
    // Step 1: Create the reminder in the database
    const reminder = await dbCreateReminder({ ...request, assignedToId }, tx);

    // Step 2: Schedule the callback in QStash
    const publishResult = await qstash.publishJSON({
      url: reminderCallbackUrl,
      body: { reminderId: reminder.id },
      notBefore: reminder.dueAt.getTime() / 1000,
    });

    // Step 3: Save the QStash message ID back to the reminder
    await dbUpdateReminderQstashMessageId(
      reminder.id,
      publishResult.messageId,
      tx,
    );

    return reminder;
  });

  return reminder;
};
```

**Key concept — The three-step transaction:**

This is the most important pattern in the session. Three things happen atomically:

1. **Create the reminder** in the database (so it has an ID)
2. **Schedule the QStash callback** using that ID (QStash now knows to call us later)
3. **Save the QStash message ID** back to the reminder (so we can cancel it later if needed)

If any step fails, the entire transaction rolls back. You never end up with a reminder in the database that has no QStash callback, or a QStash callback pointing to a reminder that doesn't exist.

**Key concept — `notBefore` uses Unix seconds:**

QStash's `notBefore` parameter expects Unix time in **seconds**, not milliseconds. JavaScript's `Date.getTime()` returns milliseconds. So we divide by 1000:

```typescript
notBefore: reminder.dueAt.getTime() / 1000
```

This is a common bug — if you pass milliseconds, QStash interprets it as a date thousands of years in the future.

**Key concept — `assignedToId` default:**

```typescript
const assignedToId = request.assignedToId ?? userSnapshot.id;
```

If the caller doesn't specify who the reminder is for, it defaults to the user creating it. This means agents can create reminders for themselves without specifying their own ID.

**Reference file:** `src/services/reminder/index.ts`

```typescript
export const ReminderService = {
  create: createReminder,
  fire: fireReminder,
} as const;

export const ReminderSchema = {
  create: createReminderSchema,
  qstash: qstashReminderDueSchema,
} as const;
```

---

### Phase 6: The Reminder Webhook — Firing Reminders

**Concepts covered:** Webhook handlers, idempotency with Redis, at-least-once delivery, service-to-service calls in transactions.

**What we did:**

Built the webhook endpoint that QStash calls when a reminder is due, and the `fireReminder` service function that processes it.

**Reference file:** `src/app/api/upstash/reminder-due/route.ts`

```typescript
export async function POST(request: NextRequest) {
  try {
    const rawBody = await request.text();

    // Step 1: Verify the request is from QStash
    const isValid = await verifyQstashSignature(request, rawBody);
    if (!isValid)
      return NextResponse.json({ error: "Invalid signature" }, { status: 401 });

    // Step 2: Parse the payload
    const body = ReminderSchema.qstash.parse(JSON.parse(rawBody));

    // Step 3: Fire the reminder
    const reminder = await ReminderService.fire(body.reminderId);

    return NextResponse.json({ ok: true });
  } catch (error) {
    console.error("Error occurred while firing reminder", error);
    return NextResponse.json({ error: "Internal server error" }, { status: 500 });
  }
}
```

**Key design — Raw body parsing:**

Notice we call `request.text()` instead of `request.json()`. This is because signature verification needs the raw string body — if you parse it to JSON and re-stringify it, whitespace or key ordering might change, invalidating the signature. We parse JSON ourselves after verification.

**Key design — Route placement:**

The webhook is at `/api/upstash/reminder-due`, not under `/api/leads/`. This is intentional — it's not a user-facing endpoint. It's a system endpoint that only QStash should call. Grouping system webhooks under `/api/upstash/` makes it clear that these routes have different authentication (signature verification vs. user sessions).

**Reference file:** `src/services/reminder/service.ts` — The `fireReminder` function

```typescript
export const fireReminder = async (reminderId: string) => {
  // Step 1: Idempotency check with Redis
  const idempotencyKey = `reminder:fired:${reminderId}`;
  const alreadyProcessed = await redis.get(idempotencyKey);
  if (alreadyProcessed) return { status: "duplicate" as const };

  // Step 2: Fetch the reminder
  const reminder = await dbGetReminder(reminderId);
  if (!reminder) {
    await redis.set(idempotencyKey, "missing");
    await redis.expire(idempotencyKey, 60 * 60 * 24);
    return { status: "missing" as const };
  }

  // Step 3: Set the idempotency key (before processing)
  await redis.set(idempotencyKey, "processed");
  await redis.expire(idempotencyKey, 60 * 60 * 24);

  // Step 4: Process the reminder in a transaction
  await prisma.$transaction(async (tx) => {
    // Mark reminder as FIRED
    await dbUpdateReminderStatus(reminderId, "FIRED", tx);

    // Create a notification for the assigned user
    await NotificationService.create(
      {
        title: reminder.title,
        body: `Reminder for lead ${reminder.lead.name}. Note: ${reminder.note}`,
        recipientId: reminder.assignedToId,
        leadId: reminder.leadId,
      },
      { id: reminder.assignedTo.id, role: reminder.assignedTo.role },
      tx,
    );
  });

  return { status: "success" as const };
};
```

**Key concept — Idempotency:**

QStash guarantees **at-least-once** delivery. "At least once" means your webhook might be called twice (or more) for the same message — if QStash doesn't receive a 200 response quickly enough, it retries. Without idempotency, a reminder could create two notifications.

The idempotency pattern:

```
1. Build a unique key: "reminder:fired:{reminderId}"
2. Check Redis: does this key exist?
   → Yes: return "duplicate" (already processed)
   → No: continue processing
3. Set the key in Redis BEFORE processing
4. Process the reminder
5. The key expires after 24 hours (cleanup)
```

Setting the key *before* processing (step 3) is intentional. If the processing crashes after step 3 but before step 4 completes, the next retry will see the key and skip. This means you might occasionally miss a reminder fire, but you'll **never** fire it twice. In most systems, processing once-or-zero is safer than processing twice.

**Key concept — Redis `expire`:**

```typescript
await redis.set(idempotencyKey, "processed");
await redis.expire(idempotencyKey, 60 * 60 * 24); // 24 hours
```

The key auto-deletes after 24 hours. Without expiration, Redis would accumulate keys forever. Since QStash retries happen within minutes (not days), a 24-hour TTL is more than sufficient.

**Key concept — Service-to-service calls:**

The `fireReminder` function calls `NotificationService.create()` — one service calling another, both inside the same transaction. This is the same pattern from Session 3 (lead service calling activity service). The notification service doesn't know it's being called from a reminder — it just creates a notification. This separation keeps both services independently testable.

---

### Phase 7: Notification Service Module

**Concepts covered:** Service module pattern, access control for non-user-initiated actions.

**What we did:**

Built the notification service following the same 5-file pattern.

**Reference file:** `src/services/notification/schema.ts`

```typescript
export const createNotificationSchema = z.object({
  title: z.string().min(1),
  body: z.string().min(1),
  recipientId: z.uuid(),
  leadId: z.uuid().optional(),
});
```

**Reference file:** `src/services/notification/service.ts`

```typescript
export const createNotification = async (
  request: CreateNotificationRequest,
  userSnapshot: UserSnapshot,
  tx?: Prisma.TransactionClient,
) => {
  if (request.leadId) {
    const lead = await dbGetLeadAssignedTo(request.leadId);
    if (!lead) throw new Error("Lead not found");

    if (!validateLeadAccess(lead.assignedToId, userSnapshot)) {
      throw new Error(
        "You are not authorized to create a notification for this lead",
      );
    }
  }

  return dbCreateNotification(request, tx);
};
```

**Key design — The notification service is intentionally simple.**

Right now, it validates access and creates a record. In the future, this is where you'd add: push notification delivery (FCM, APNs), email notification, Slack integration, batching/throttling. The service layer is the right place because those are business logic decisions — "when should we send an email vs. just an in-app notification?"

**The full notification service — listing and marking read:**

Beyond creating notifications (which the reminder webhook calls), we also built the read paths during the session.

**Reference file:** `src/services/notification/schema.ts` — Additional schemas

```typescript
export const listNotificationsQuerySchema = z.object({
  page: z.coerce.number().min(1).default(1),
  pageSize: z.coerce.number().min(1).max(100).default(10),
});

export const notificationIdParamsSchema = z.object({
  id: z.uuid(),
});
```

The `listNotificationsQuerySchema` handles pagination query params. The `notificationIdParamsSchema` validates the `[id]` route param when marking a notification as read.

**Reference file:** `src/services/notification/schema.ts` — Response types

```typescript
export interface NotificationListItem {
  id: string;
  title: string;
  body: string;
  leadId: string | null;
  readState: NotificationReadState;
  readAt: Date | null;
  createdAt: Date;
  lead: NotificationLeadSummary | null;
}

export interface ListNotificationsResponseData {
  notifications: NotificationListItem[];
  pagination: PaginationMeta;
  /** Total unread notifications for this recipient (not limited to current page). */
  unreadCount: number;
}
```

Notice the `unreadCount` field — it's returned alongside the paginated list so the UI can show the total unread badge number without a separate API call.

**Reference file:** `src/services/notification/db.ts` — List and mark-read queries

```typescript
export const dbListNotificationsForRecipient = async (
  recipientId: string,
  params: ListNotificationsParams,
): Promise<ListNotificationsResponseData> => {
  const where: Prisma.NotificationWhereInput = { recipientId };
  const whereUnread: Prisma.NotificationWhereInput = {
    recipientId,
    readState: NotificationReadState.UNREAD,
  };

  const [rows, total, unreadCount] = await Promise.all([
    prisma.notification.findMany({
      where,
      select: notificationListSelect,
      take: params.pageSize,
      skip: (params.page - 1) * params.pageSize,
      orderBy: { createdAt: "desc" },
    }),
    prisma.notification.count({ where }),
    prisma.notification.count({ where: whereUnread }),
  ]);

  return { notifications: rows, pagination: buildPagination(total, params.page, params.pageSize), unreadCount };
};
```

**Key design — Three parallel queries with `Promise.all`:**

We need three numbers: the paginated rows, the total count (for pagination), and the unread count (for the badge). Running them in parallel with `Promise.all` instead of sequentially cuts the query time by ~3x. This is a common pattern when a single API response needs data from multiple independent queries.

```typescript
export const dbMarkNotificationReadForRecipient = async (
  id: string,
  recipientId: string,
): Promise<NotificationListItem | null> => {
  const existing = await prisma.notification.findFirst({
    where: { id, recipientId },
    select: { id: true },
  });

  if (!existing) return null;

  return prisma.notification.update({
    where: { id },
    data: { readState: NotificationReadState.READ, readAt: new Date() },
    select: notificationListSelect,
  });
};
```

**Key design — Recipient scoping on mark-read:**

We don't just look up the notification by ID — we also check `recipientId`. This ensures users can only mark their own notifications as read. Without this check, any authenticated user could mark anyone's notifications as read by guessing IDs.

**Reference file:** `src/services/notification/service.ts` — List and mark-read functions

```typescript
export async function listNotifications(profile: Profile, params: ListNotificationsParams) {
  return dbListNotificationsForRecipient(profile.id, params);
}

export async function markNotificationRead(profile: Profile, id: string) {
  const updated = await dbMarkNotificationReadForRecipient(id, profile.id);
  if (!updated) throw new NotificationServiceError("Notification not found", 404);
  return updated;
}
```

The service layer is thin here — it passes the authenticated profile's ID to the DB layer. The `NotificationServiceError` with a `statusCode` lets the route handler return the appropriate HTTP status.

**Reference file:** `src/app/api/notifications/route.ts`

```typescript
// GET /api/notifications?page=1&pageSize=10
export async function GET(request: NextRequest) {
  try {
    const profile = await authenticateUser();
    const searchParams = request.nextUrl.searchParams;
    const params = listNotificationsQuerySchema.parse({
      page: searchParams.get("page"),
      pageSize: searchParams.get("pageSize"),
    });

    const data = await listNotifications(profile, params);
    return NextResponse.json({ success: true, data });
  } catch (error) {
    return handleRouteError(error);
  }
}
```

**Reference file:** `src/app/api/notifications/[id]/read/route.ts`

```typescript
// PATCH /api/notifications/[id]/read
export async function PATCH(
  _request: NextRequest,
  { params }: { params: Promise<{ id: string }> },
) {
  try {
    const profile = await authenticateUser();
    const { id } = notificationIdParamsSchema.parse(await params);
    const data = await markNotificationRead(profile, id);
    return NextResponse.json({ success: true, data });
  } catch (error) {
    return handleRouteError(error);
  }
}
```

**Key design — Route path `/notifications/[id]/read`:**

The mark-read endpoint is `PATCH /api/notifications/[id]/read`, not `PATCH /api/notifications/[id]` with a body. This makes the intent explicit in the URL. If we later add other actions (dismiss, snooze), they each get their own sub-route rather than overloading a single PATCH.

**Reference file:** `src/lib/tanstack/useNotifications.ts`

```typescript
export function useGetNotifications(params: ListNotificationsParams) {
  return useQuery({
    queryKey: ["notifications", params],
    queryFn: async (): Promise<ListNotificationsResponseData> => {
      const { data } = await api.get("/notifications", { params });
      return data.data;
    },
    refetchInterval: 1000 * 5, // every 5 seconds
  });
}

export function useMarkNotificationRead() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: async (id: string): Promise<NotificationListItem> => {
      const { data } = await api.patch(`/notifications/${id}/read`);
      return data.data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["notifications"] });
    },
  });
}
```

**Key concept — Polling with `refetchInterval`:**

```typescript
refetchInterval: 1000 * 5, // every 5 seconds
```

This is TanStack Query's built-in polling mechanism. Every 5 seconds, TanStack Query re-runs the `queryFn` silently in the background. If new notifications exist, React re-renders the component. If nothing changed, nothing happens visually.

**Why polling and not WebSockets?** For a CRM with a handful of concurrent users, polling every 5 seconds is simple, reliable, and introduces zero infrastructure complexity. WebSockets would give you instant delivery but require a persistent connection, a WebSocket server, and reconnection logic. The trade-off: polling wastes a small amount of bandwidth checking when nothing changed, but the implementation is trivial — one line of config.

**Key concept — Cache invalidation on mark-read:**

When `useMarkNotificationRead` succeeds, it calls `queryClient.invalidateQueries({ queryKey: ["notifications"] })`. This immediately triggers a refetch of the notification list, which updates the unread count badge. The user sees the badge number decrease the instant they click a notification — no waiting for the next polling cycle.

**Reference file:** `src/components/notification-icon.tsx`

```typescript
export function NotificationBell() {
  const [page, setPage] = useState(1)
  const pageSize = 10
  const { data, isLoading, isError } = useGetNotifications({ page, pageSize })
  const notifications = data?.notifications ?? []
  const markRead = useMarkNotificationRead()
  const unreadCount = data?.unreadCount ?? 0

  return (
    <Popover>
      <PopoverTrigger asChild>
        <Button variant="ghost" size="icon" className="relative">
          <Bell className="size-5" />
          {unreadCount > 0 ? (
            <span className="absolute -top-1 -right-1 rounded-full bg-rose-500 px-1.5 text-[10px] text-white">
              {unreadCount}
            </span>
          ) : null}
        </Button>
      </PopoverTrigger>

      <PopoverContent align="end" sideOffset={8} className="w-80 ...">
        <div className="border-b px-4 py-3">
          <h4>Notifications</h4>
        </div>

        {/* Loading, error, and empty states */}

        {notifications.map((notification) => (
          <Link
            key={notification.id}
            href={notification.leadId ? `/leads/${notification.leadId}` : "#"}
            onClick={() => {
              if (notification.readState === "UNREAD") {
                markRead.mutate(notification.id)
              }
            }}
          >
            <p>{notification.title}</p>
            <p>{notification.body}</p>
          </Link>
        ))}

        <Pagination ... />
      </PopoverContent>
    </Popover>
  )
}
```

**Key design — The notification bell architecture:**

The `NotificationBell` is a self-contained client component that manages its own data fetching via `useGetNotifications`. It renders in the app header (sidebar top area) and consists of:

1. **Trigger button** — a ghost button with a Bell icon and an absolute-positioned badge showing the unread count. The badge only appears when `unreadCount > 0`.
2. **Popover content** — a dropdown panel with a notification list. Each notification is a `Link` that navigates to the associated lead and marks the notification as read on click.
3. **Loading/error/empty states** — proper handling for all three states, including an empty state with an Inbox icon.
4. **Pagination** — reuses the same `Pagination` component from the leads table.

**Key design — Click-to-read pattern:**

```typescript
onClick={() => {
  if (notification.readState === "UNREAD") {
    markRead.mutate(notification.id)
  }
}}
```

Notifications are marked as read when clicked, not when seen. This is intentional — users might scroll through notifications without acting on them. Only clicking (which navigates to the lead) marks it as read. This prevents the "I saw it but forgot about it" problem.

**Software engineering principle — Polling drives the entire flow:**

The notification bell polls every 5 seconds. When a reminder fires (via QStash webhook → `fireReminder` → `NotificationService.create`), a new notification row appears in the database. Within 5 seconds, the next polling cycle picks it up and the bell badge increments. The user sees the notification without any real-time infrastructure — just a database row and a timer.

---

### Phase 8: Reminders API Route

**Concepts covered:** Thin route handlers, TanStack mutations.

**What we did:**

Created the API route for creating reminders and a basic TanStack mutation hook. We tested the full pipeline (create reminder → QStash schedules → webhook fires → notification created) using a throwaway test form. The test form in the repo is intentionally rough — building the proper Reminders UI is your assignment.

**Reference file:** `src/app/api/leads/[id]/reminders/route.ts`

```typescript
export async function POST(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> },
) {
  try {
    const profile = await authenticateUser();
    const { id } = leadIdParamsSchema.parse(await params);
    const body = await request.json();
    const data = ReminderSchema.create.parse(body);

    const reminder = await ReminderService.create(
      { ...data, leadId: id },
      { id: profile.id, role: profile.role },
    );

    return NextResponse.json({ success: true, data: reminder });
  } catch (error) {
    return handleRouteError(error);
  }
}
```

The route follows the exact same pattern: authenticate → validate → call service → respond.

**Reference file:** `src/lib/tanstack/useReminders.ts`

```typescript
export function useCreateLeadReminder(leadId: string) {
  return useMutation({
    mutationFn: async (request: CreateReminderRequest): Promise<Reminder> => {
      const { data } = await api.post(`/leads/${leadId}/reminders`, request);
      return data.data;
    },
  });
}
```

**What we tested live (not part of the final UI):**

During the session, we threw together a quick form with a Calendar, Popover, and time input to test that the full pipeline works end-to-end. We verified that:

1. Creating a reminder calls the API route
2. The API route creates the Reminder in the database and schedules a QStash callback
3. When the callback fires, the webhook handler creates a Notification and marks the Reminder as FIRED
4. Idempotency works — duplicate callbacks are ignored

The test form in the repo (`src/components/leads/lead-details/Reminders.tsx`) is a bare-bones implementation used only for this verification. **Your assignment is to replace it** with a proper Reminders UI that lists existing reminders, shows status badges, and uses a polished Create Reminder dialog matching the wireframes.

**New shadcn components installed during testing:**

```bash
npx shadcn@latest add calendar popover
```

These components are available in the repo for you to use when you build the proper UI.

---

## What We Did NOT Build (Your Assignment)

We built the complete reminder creation and firing pipeline (backend only), plus a throwaway test form to verify it works. The entire frontend experience for reminders is yours to build:

1. **The `/reminders` page** — the standalone page accessible from the sidebar that lists ALL of a user's reminders across all leads, with tabs for filtering by status
2. **Reminder list on the lead detail page** — the Reminders tab currently only has a rough test form; it needs to be replaced with a proper UI that lists existing reminders with status badges, overdue detection, and action buttons
3. `**GET /api/reminders`** — an API route that lists reminders for the current user (with filtering by status)
4. `**GET /api/leads/[id]/reminders**` — an API route that lists reminders for a specific lead
5. `**PATCH /api/reminders/[id]**` — an API route to complete or cancel a reminder
6. **Create Reminder Dialog** — a proper dialog component matching the wireframe (Title, Due Date, Due Time fields) to replace the test form

---

## Supplementary Videos

### Review What We Covered

These videos reinforce the concepts from Session 4. Watch them if anything felt unclear.

**QStash — Serverless Message Queue**
[How to avoid serverless function timeouts in NextJs](https://www.youtube.com/watch?v=4jQBKoNQhzU) by Hamed Bahram (~17 min)
Covers the publish → callback flow, delayed delivery, and retry handling. This is exactly what we used for the reminder scheduling.

**Redis — Idempotency & Key-Value Basics**
[How to use Redis Caching for Incredible Performance](https://www.youtube.com/watch?v=-5RTyEim384) by Josh Tried Coding (~13 min)
Covers SET, GET, TTL, and expiration. We used Redis for a single purpose — idempotency — but understanding the basics will help when we add caching later.

**Upstash QStash Documentation**
[QStash Docs](https://upstash.com/docs/qstash/overall/getstarted)
Read the "Getting Started" and "Publishing Messages" sections. Pay attention to `notBefore` (scheduling) and signature verification.

---

## Assignment: Build the Reminder UI & Routes Before Session 5

You have the entire backend pipeline: create reminder → schedule in QStash → fire webhook → create notification. Your assignment is to build the read paths and remaining UI that make reminders usable.

### Task 1: GET Reminders API Routes (estimated: 1.5-2 hours)

Build two read endpoints for reminders.

**Endpoint 1:** `GET /api/leads/[id]/reminders` — returns all reminders for a specific lead.

Create the route at `src/app/api/leads/[id]/reminders/route.ts` (add a GET handler alongside the existing POST).

**What to implement:**

- Authenticate the user
- Validate the lead ID
- Role-scope the query: agents only see reminders for leads assigned to them; managers/admins see all
- Return reminders sorted by `dueAt` descending
- Include pagination (use the same `buildPagination` utility from Session 3)

**Schema:** Add to `src/services/reminder/schema.ts`:

```typescript
export const listLeadRemindersSchema = z.object({
  leadId: z.uuid(),
  page: z.coerce.number().min(1).default(1),
  pageSize: z.coerce.number().min(1).max(100).default(10),
  status: z.nativeEnum(ReminderStatus).optional(),
});
```

**DB function:** Add to `src/services/reminder/db.ts`:

```typescript
export const dbGetLeadReminders = async (
  where: Prisma.ReminderWhereInput,
  params: { page: number; pageSize: number },
) => {
  const [reminders, total] = await Promise.all([
    prisma.reminder.findMany({
      where,
      orderBy: { dueAt: "desc" },
      take: params.pageSize,
      skip: (params.page - 1) * params.pageSize,
      include: {
        lead: { select: { id: true, name: true } },
      },
    }),
    prisma.reminder.count({ where }),
  ]);
  return { reminders, total };
};
```

**Endpoint 2:** `GET /api/reminders` — returns all reminders for the current user across all leads.

Create the route at `src/app/api/reminders/route.ts`.

**What to implement:**

- Authenticate the user
- For agents: filter by `assignedToId = profile.id`
- For managers/admins: return all reminders (or optionally filter by assignedToId from query params)
- Support filtering by `status` query parameter (PENDING, FIRED, CANCELLED)
- Include pagination
- Include the lead name in each reminder (so the table can show which lead it's for)

**TanStack hooks:** Create `src/lib/tanstack/useReminders.ts` with:

```typescript
export function useGetLeadReminders(leadId: string, params) { ... }
export function useGetMyReminders(params) { ... }
```

### Task 2: PATCH Reminder — Complete & Cancel (estimated: 1-1.5 hours)

Build a PATCH endpoint to update a reminder's status.

**Route:** `PATCH /api/reminders/[id]`

**Schema:**

```typescript
export const updateReminderSchema = z.object({
  status: z.enum(["CANCELLED"]),
});
```

Only CANCELLED is allowed via the API — FIRED is set by the webhook, not by users. When cancelling, also cancel the QStash message using the stored `qstashMessageId`:

```typescript
await qstash.messages.delete(reminder.qstashMessageId);
```

**What to implement:**

- Authenticate the user
- Validate the reminder exists
- Check authorization: only the assigned user, a manager, or an admin can cancel
- Update the status to CANCELLED
- Cancel the QStash scheduled message
- Return the updated reminder

**TanStack hook:** `useUpdateReminder(reminderId)` with cache invalidation for both `["reminders"]` and `["lead-reminders"]`.

### Task 3: Lead Detail Reminders Tab (estimated: 1.5-2 hours)

Replace the throwaway test form in `src/components/leads/lead-details/Reminders.tsx` with a proper UI that matches the wireframe. The current file is a rough form we used during the session to test the backend pipeline — it needs to be completely rewritten.

**Expected outcome:** The Reminders tab on the lead detail page shows:

- A list of existing reminders for this lead (title, due date, status)
- Each reminder shows its status as a colored badge (Upcoming = blue, Overdue = red, Completed = green, Cancelled = gray)
- Action buttons: "Complete" and "Cancel" for PENDING reminders
- Overdue detection: if `dueAt` is in the past and status is still PENDING, show "Overdue" in red
- A "+ Create Reminder" button that opens a dialog

**Create Reminder Dialog:** Build a proper dialog matching the wireframe at `share-with-students/session-4/wireframes/S4 - Create Reminder Dialog@2x.png`:

- Title field (required)
- Due Date picker (calendar)
- Due Time picker
- Cancel and Create Reminder buttons

**Hints:**

- Use `useGetLeadReminders(leadId)` to fetch reminders
- Use `useCreateLeadReminder(leadId)` for the create form
- Use `useUpdateReminder(reminderId)` for complete/cancel actions
- For overdue detection: `reminder.status === "PENDING" && new Date(reminder.dueAt) < new Date()`

### Task 4: `/reminders` Page (estimated: 2-2.5 hours)

Build the standalone reminders page accessible from the sidebar, matching the wireframe at `share-with-students/session-4/wireframes/S4 - Reminders List Page@2x.png`.

**Expected outcome:** The `/reminders` page shows:

- Page title: "My Reminders"
- Filter tabs: All, Upcoming, Overdue, Completed, Cancelled
- A table with columns: Due Date, Title, Lead (clickable link to lead detail), Status, Actions
- "Complete" and "Cancel" action buttons for PENDING reminders
- The "Overdue" tab shows a count badge (number of overdue items)
- Pagination at the bottom

**What to build:**

- Update `src/app/(protected)/reminders/page.tsx` — replace the placeholder with the real component
- Create a client component: `src/components/reminders/reminders-page-client.tsx`
- Use `useGetMyReminders({ page, pageSize, status })` to fetch data
- Use tabs to filter by status
- Lead name should be a link to `/leads/[id]`

**Hints:**

- The tab "Overdue" is a special case — it's not a `ReminderStatus` enum value. You'll need to handle it client-side by filtering PENDING reminders where `dueAt < now`, or add a special query parameter in your API
- Use the same `Pagination` component from `src/components/leads/reusable.tsx`
- Use shadcn's Table component for the table layout

### Task 5: Create Reminder Dialog as Reusable Component (estimated: 1 hour)

The Create Reminder dialog should work in two places:

1. On the lead detail Reminders tab (Task 3)
2. On the `/reminders` page (Task 4) — but here it needs a lead selector since you're not in a lead context

Build it as a reusable component at `src/components/reminders/CreateReminderDialog.tsx` that accepts an optional `leadId` prop. When `leadId` is provided, it's pre-filled and locked. When it's not provided, show a lead search/select field.

### Task 6: Watch Pre-Session 5 Video (estimated: 15 min)

Session 5 introduces **AI features** into the CRM. You'll build two AI-powered tools: a Lead Brief Generator and a Call Follow-up Assistant. Both use the **Vercel AI SDK** to call an AI model and get back structured, typed responses.

#### Vercel AI SDK — `generateObject`

**What it should cover:** What the Vercel AI SDK is — a unified TypeScript interface for calling AI models. Focus specifically on `generateObject`: you pass a prompt + a Zod schema, and the SDK returns a typed object matching that schema. Understand the basic flow: define schema → build prompt → call generateObject → get typed result.

> **Why this matters:** Both AI features we build in Session 5 use `generateObject`. If you walk in knowing the basic API shape, we can spend our time on the interesting parts — prompt engineering, context assembly, and building the UI around AI responses.

**Watch:** [Vercel AI SDK by Matt](https://www.youtube.com/watch?v=mojZpktAiYQ) (~15 min)

---

## Checkpoint Before Session 5

Before Session 5, verify all of the following:

- `GET /api/leads/[id]/reminders` returns reminders for a lead with pagination
- `GET /api/reminders` returns the current user's reminders with status filtering
- `PATCH /api/reminders/[id]` can cancel a reminder and removes the QStash message
- Lead detail Reminders tab shows existing reminders with status badges
- Create Reminder dialog matches the wireframe (Title, Due Date, Due Time)
- Overdue reminders are highlighted (PENDING + dueAt in the past)
- `/reminders` page shows all reminders in a table with filter tabs
- Clicking a lead name in the reminders table navigates to the lead detail page
- Complete/Cancel actions work and update the UI immediately
- Code committed and pushed to GitHub
- Pre-Session 5 video watched (Vercel AI SDK `generateObject`)
- AI Gateway API key added to `.env` as `AI_GATEWAY_API_KEY` (will be provided to you)

---

## Resources

- [Upstash QStash Docs](https://upstash.com/docs/qstash/overall/getstarted) — Publishing, scheduling, signature verification
- [ioredis Documentation](https://github.com/redis/ioredis) — Redis client for Node.js
- [TanStack Query v5 — Mutations](https://tanstack.com/query/v5/docs/react/guides/mutations) — `useMutation`, `onSuccess`, cache invalidation
- [Prisma — Relations](https://www.prisma.io/docs/orm/prisma-schema/data-model/relations) — One-to-many relations
- [Next.js Route Handlers](https://nextjs.org/docs/app/building-your-application/routing/route-handlers) — Request/response handling
- [shadcn/ui Calendar](https://ui.shadcn.com/docs/components/calendar) — Date picker component
- [shadcn/ui Popover](https://ui.shadcn.com/docs/components/popover) — Popover for date picker
- [date-fns](https://date-fns.org/) — Date formatting and manipulation

---

*Session 4 Complete. You now understand how to build time-aware features with scheduled jobs, webhooks, and idempotent processing. The reminder system is the first feature in the CRM that operates asynchronously — things happen even when no one is looking at the screen. See you in Session 5.*