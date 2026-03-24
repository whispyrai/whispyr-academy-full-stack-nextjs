# Session 3: Post-Session Recap & Tutorial

**Activity Logging & Audit Trail — Building a Service Layer That Tracks Everything**

---

## What We Built

Session 3 was all about building the next layer of the CRM: a complete activity logging and audit trail system. We took the lead management infrastructure from Session 2 and added the ability to track every change that happens to a lead — who changed what, when they changed it, and what the before/after values were.

By the end of Session 3, the `whispyr-academy-crm` repo had:

- A **5-file service module pattern** that we formalized and will use for every domain in the app (lead, activity, reminder, etc.)
- An **Activity service layer** with validation, database access, business logic, and helper functions that convert change metadata into human-readable content
- **Change detection** integrated into the lead update flow — when you change a lead's status or stage, an Activity record is automatically created
- `GET /api/leads/[id]/activities` — a paginated, role-scoped endpoint that lists all activities for a lead
- `useGetLeadActivities` hook with TanStack Query for client-side caching and pagination
- A **Timeline component** that renders the activity log with icons, timestamps, and human-readable descriptions
- **Batch activity creation** using Prisma's `createMany` for efficient multi-record writes
- **Role-based scoping** — agents only see activities for their assigned leads

> **Note:** We completed the activity logging infrastructure and the read path (displaying activities). We did NOT build the write paths for notes or call attempts — these are intentionally left for your assignment so you can practice the full-stack pattern.

---

## Step-by-Step Walkthrough

If you missed the session or want to review, follow these steps in order. Each step references the exact files in the [`whispyr-academy-crm`](https://github.com/whispyr-academy/whispyr-academy-crm) repo.

---

### Phase 0: Carryover Verification

**What we did:**

Session 3 started with a quick check that all Session 2 assignments were complete:

- [ ] Leads table displays on `/leads` with pagination
- [ ] Create Lead dialog opens and submits successfully
- [ ] Lead detail page at `/leads/[id]` shows a single lead
- [ ] PATCH endpoint at `/api/leads/[id]` can update lead status, stage, and assignment

If any of these were missing, we built them together during the first 30 minutes so we had a solid foundation for the activity system. **This is your reminder to complete Session 2 tasks if you haven't yet — Session 3 depends on them.**

---

### Phase 1: The Service Layer Architectural Pattern

**Concepts covered:** Modular architecture, separation of concerns, the domain-driven design principle.

**What we did:**

We formalized a pattern that every service module in the app will follow. Think of this as a blueprint that we'll use for activities, reminders, users, and any other domain.

**The 5-file pattern:**

```
src/services/{domain}/
├── schema.ts    ← Zod validation schemas for this domain
├── db.ts        ← Raw database query functions
├── service.ts   ← Business logic and authorization
├── helpers.ts   ← Utility functions (transformations, calculations)
└── index.ts     ← Public API (re-exports for other services)
```

**Why this pattern?**

- **schema.ts**: All validation in one place. If you need to validate activity input in multiple routes, you import from here.
- **db.ts**: Purely data access. No business logic, no authorization. Just "given these parameters, query the database."
- **service.ts**: The traffic cop. Validates input, checks authorization, decides which db functions to call, orchestrates transactions.
- **helpers.ts**: Pure functions that transform data. `buildActivityContent()` takes raw metadata and returns a human-readable string. No side effects.
- **index.ts**: The public face. Other services and routes import from `src/services/activity`, which is really importing from `index.ts`. This lets you refactor internals without breaking imports.

**Reference file:** Look at the structure in `src/services/activity/` in the repo. You'll see all five files. This is the blueprint for the entire app.

**Software engineering principle — Modular cohesion:**

Files are organized by domain, not by layer. You don't have `src/schemas/activity.ts` and `src/db/activity.ts` scattered around — they live together in `src/services/activity/` because they belong to the same domain. This makes it easy to find related code and reason about a domain in isolation.

---

### Phase 2: Activity Service Module — schema.ts

**Concepts covered:** Zod refinement, enum validation, type inference, discriminated types.

**What we did:**

Created comprehensive validation schemas for activity operations in `src/services/activity/schema.ts`.

**Reference file:** `src/services/activity/schema.ts`

The key schema is `createActivitySchema`:

```typescript
const leadStatusSchema = z.enum(LeadStatus);
const leadStageSchema = z.enum(LeadStage);
const createActivitySchema = z
  .object({
    leadId: z.uuid(),
    actorId: z.uuid(),
    type: z.enum(ActivityType),
    meta: z
      .object({
        from: z.unknown(),
        to: z.unknown(),
      })
      .optional(),
  })
  .superRefine((data) => {
    if (
      ["STATUS_CHANGE", "STAGE_CHANGE", "ASSIGNMENT_CHANGE"].includes(data.type)
    ) {
      if (!data.meta) {
        throw new Error("Meta is required for status change");
      }

      switch (data.type) {
        case ActivityType.STATUS_CHANGE:
          data.meta.from = leadStatusSchema.parse(data.meta.from);
          data.meta.to = leadStatusSchema.parse(data.meta.to);
          break;
        case ActivityType.STAGE_CHANGE:
          data.meta.from = leadStageSchema.parse(data.meta.from);
          data.meta.to = leadStageSchema.parse(data.meta.to);
          break;
        case ActivityType.ASSIGNMENT_CHANGE:
          data.meta.from = z.string().parse(data.meta.from); // agent name
          data.meta.to = z.string().parse(data.meta.to); // agent name
          break;
        default:
          break;
      }
    }
  });
```

**Key design — `.superRefine()` for conditional validation:**

This is a Zod technique that lets you run custom validation logic. If the activity type is `STATUS_CHANGE`, you MUST provide `meta.from` and `meta.to`, and they must be valid `LeadStatus` values. If the type is `NOTE`, `meta` is optional because notes don't have a "before/after."

**Key design — The meta pattern:**

Activities that represent changes (status change, stage change, assignment change) carry a `meta` object with `{ from, to }`. This is the raw data about what changed. Later, a helper function converts this into a human-readable string like "Status changed from OPEN to WON".

Also defined:

```typescript
export const getLeadActivitiesSchema = z.object({
  leadId: z.uuid(),
  page: z.coerce.number().min(1).default(1),
  pageSize: z.coerce.number().min(1).max(100).default(10),
});

export type GetLeadActivitiesRequest = z.infer<typeof getLeadActivitiesSchema>;

export type ListLeadActivitiesResponseData = {
  activities: ActivitySummaryItem[];
  pagination: PaginationMeta;
};
```

This schema validates the query parameters for the GET activities endpoint. It's separate from `createActivitySchema` because reading activities has different validation rules than creating them.

**Key concept — Type safety from schemas:**

`z.infer<typeof createActivitySchema>` extracts the TypeScript type from the Zod schema. So `CreateActivityRequest` is guaranteed to match the runtime validation. If you change the schema, the type changes automatically. One source of truth.

---

### Phase 3: Activity Service Module — db.ts

**Concepts covered:** Batch operations, parallel queries with Promise.all, transaction client pattern.

**What we did:**

Created `src/services/activity/db.ts` with two functions: one to create activities in bulk, one to fetch them with pagination.

**Reference file:** `src/services/activity/db.ts`

```typescript
export async function dbCreateActivities(
  activities: Prisma.ActivityCreateManyInput[],
  tx?: Prisma.TransactionClient,
) {
  const client = tx ?? prisma;
  const created = await client.activity.createMany({
    data: activities,
  });

  return created;
}

export async function dbGetLeadActivities(
  where: Prisma.ActivityWhereInput,
  params: {
    page: number;
    pageSize: number;
  },
) {
  const [activities, total] = await Promise.all([
    prisma.activity.findMany({
      where,
      orderBy: {
        createdAt: "desc",
      },
      select: {
        content: true,
        type: true,
        createdAt: true,
        id: true,
        actor: {
          select: {
            name: true,
          },
        },
      },
      take: params.pageSize,
      skip: (params.page - 1) * params.pageSize,
    }),
    prisma.activity.count({ where }),
  ]);

  return {
    activities,
    total,
  };
}
```

**Key design — Batch creation with `createMany`:**

Instead of looping and calling `create()` for each activity, we use `createMany()` to insert multiple records in a single database round-trip. This is much faster — if you're logging 5 changes to a lead (maybe status, stage, and assignment all changed), you send ONE query instead of FIVE.

**Key design — `tx` parameter for transactions:**

The `tx` parameter lets you run db functions inside a Prisma transaction. If called inside a transaction, use `tx.activity.createMany()`. If called standalone, use `prisma.activity.createMany()`. The function doesn't care — it just uses whichever client was passed in. This makes `dbCreateActivities` reusable in both contexts: standalone calls and transactional calls.

**Key design — Parallel queries with `Promise.all`:**

To render a paginated list, you need two queries: fetch the page of records AND count the total. Instead of:

```typescript
const activities = await prisma.activity.findMany(...);
const total = await prisma.activity.count(...);
```

We do:

```typescript
const [activities, total] = await Promise.all([
  prisma.activity.findMany(...),
  prisma.activity.count(...),
]);
```

Both queries run in parallel, cutting the latency in half.

**Key design — Selecting only needed fields:**

```typescript
select: {
  content: true,
  type: true,
  createdAt: true,
  id: true,
  actor: {
    select: {
      name: true,
    },
  },
}
```

We don't fetch the entire Activity record — just the fields the timeline needs. This reduces the data transferred and makes queries faster.

---

### Phase 4: Activity Service Module — helpers.ts

**Concepts covered:** Pure functions, deterministic output, conversion at write time.

**What we did:**

Created `src/services/activity/helpers.ts` with a single helper function that converts raw change metadata into human-readable content.

**Reference file:** `src/services/activity/helpers.ts`

```typescript
export function buildActivityContent(
  activityType: ActivityType,
  meta:
    | {
        from: unknown;
        to: unknown;
      }
    | undefined,
) {
  if (!meta) {
    return null;
  }

  switch (activityType) {
    case ActivityType.STATUS_CHANGE:
      return `Status changed from ${meta.from} to ${meta.to}`;
    case ActivityType.STAGE_CHANGE:
      return `Stage changed from ${meta.from} to ${meta.to}`;
    case ActivityType.ASSIGNMENT_CHANGE:
      return `Assignment changed from ${meta.from} to ${meta.to}`;
    default:
      return null;
  }
}
```

**Key design — Conversion at write time:**

When you create an activity, `buildActivityContent` runs ONCE and stores the result in the database as a string. The database never stores the raw `meta` object — it stores the formatted content. This is a deliberate trade-off:

**Pro:** When you query activities, the content is already formatted. No need to compute it again on read.

**Con:** If you later want to change the format of how changes are displayed, the old activities still have the old format in the database.

This is appropriate for an audit log where historical accuracy matters more than dynamic formatting.

**Key concept — Pure functions:**

`buildActivityContent` takes inputs and returns a value. It has no side effects (doesn't query the database, doesn't modify global state). This makes it testable and predictable. Given the same inputs, it always returns the same output.

---

### Phase 5: Activity Service Module — service.ts

**Concepts covered:** Validation and orchestration, discriminated unions, role-based filtering.

**What we did:**

Created `src/services/activity/service.ts` with the business logic for creating and reading activities.

**Reference file:** `src/services/activity/service.ts`

The `createActivities` function:

```typescript
export async function createActivities(
  request: CreateActivityRequest[],
  tx?: Prisma.TransactionClient,
) {
  const validated = createManyActivitiesSchema.safeParse(request);
  if (!validated.success) {
    return {
      success: false as const,
      errors: validated.error.flatten().fieldErrors,
    };
  }

  const activitiesToCreate: Prisma.ActivityCreateManyInput[] = [];
  for (const activity of validated.data) {
    const content = buildActivityContent(activity.type, activity.meta);
    activitiesToCreate.push({
      leadId: activity.leadId,
      actorId: activity.actorId,
      content,
      type: activity.type,
    });
  }

  const countCreated = await dbCreateActivities(activitiesToCreate, tx);

  return {
    success: true as const,
    count: countCreated.count,
  };
}
```

**Key design — Discriminated unions:**

The function returns either `{ success: false, errors }` or `{ success: true, count }`. In TypeScript, this is called a discriminated union — the `success` field discriminates which fields are available. If `success` is `false`, you can access `errors`. If `success` is `true`, you can access `count`. This provides compile-time safety: you can't accidentally try to use `errors` when `success` is `true`.

The `as const` ensures the literal value `false` and `true` are used as types, not widened to `boolean`.

**Key design — Validation then transformation:**

1. Validate the entire batch with `createManyActivitiesSchema.safeParse()`
2. If validation fails, return an error result immediately
3. If validation succeeds, transform each activity (call `buildActivityContent`) and build the list for the database
4. Execute the batch insert
5. Return success with the count

This layered approach is clean and testable.

The `getLeadActivities` function:

```typescript
export async function getLeadActivities(
  request: GetLeadActivitiesRequest,
  userSnapshot: UserSnapshot,
) {
  const where: Prisma.ActivityWhereInput = {
    leadId: request.leadId,
  };

  if (userSnapshot.role === Role.AGENT) {
    where.lead = {
      assignedToId: userSnapshot.id,
    };
  }

  const result = await dbGetLeadActivities(where, {
    page: request.page,
    pageSize: request.pageSize,
  });

  return {
    activities: result.activities,
    pagination: buildPagination(result.total, request.page, request.pageSize),
  };
}
```

**Key design — Role-scoped filtering:**

An AGENT can only see activities for leads assigned to them. A MANAGER or ADMIN can see activities for any lead. The scoping is enforced at the SERVICE layer (not at the database), so it's easily testable and auditable.

**Key design — UserSnapshot instead of full Profile:**

The function receives `{ id, role }` instead of the full Profile object. This is intentional — the service only needs to know the user's ID and role. Passing smaller, focused objects makes the code easier to test and understand.

---

### Phase 6: Activity Service Module — index.ts

**Concepts covered:** Public API design, namespace organization, re-exports.

**What we did:**

Created `src/services/activity/index.ts` as the public face of the activity service.

**Reference file:** `src/services/activity/index.ts`

```typescript
import { createActivities, getLeadActivities } from "./service";
import { getLeadActivitiesSchema } from "./schema";

export const ActivityService = {
  create: createActivities,
  getByLeadId: getLeadActivities,
} as const;

export const ActivitySchema = {
  getByLeadId: getLeadActivitiesSchema,
} as const;

export type { CreateActivityRequest } from "./schema";
```

**Key design — Object-based API:**

Instead of exporting individual functions like `export { createActivities, getLeadActivities }`, we export objects: `ActivityService` and `ActivitySchema`. This provides a clear namespace:

```typescript
// Instead of:
import { createActivities } from "@/services/activity/service";

// You write:
import { ActivityService } from "@/services/activity";
ActivityService.create(...);
```

The namespace makes it clear that `create` is part of the activity domain. If another service also has a `create` function, there's no naming conflict.

**Key design — `as const`:**

This tells TypeScript to treat the object as immutable and preserve literal types. Without it, TypeScript might widen the object shape, losing some type information.

---

### Phase 7: Lead Service Integration — Change Detection

**Concepts covered:** Integration between services, transaction orchestration, audit trail creation.

**What we did:**

Updated the lead service to automatically create Activity records when a lead changes.

**Reference file:** `src/services/lead/helpers.ts`

Added a new helper function:

```typescript
interface BuildLeadChangeActivitiesParams {
  leadId: string;
  actorId: string;
  existingLead: Lead;
  newLead: Partial<Lead>;
}

export function buildLeadChangeActivities({
  leadId,
  actorId,
  existingLead,
  newLead,
}: BuildLeadChangeActivitiesParams) {
  const activities: CreateActivityRequest[] = [];

  if (newLead.status && newLead.status !== existingLead.status) {
    activities.push({
      leadId,
      actorId,
      type: ActivityType.STATUS_CHANGE,
      meta: {
        from: existingLead.status,
        to: newLead.status,
      },
    });
  }

  if (newLead.stage && newLead.stage !== existingLead.stage) {
    activities.push({
      leadId,
      actorId,
      type: ActivityType.STAGE_CHANGE,
      meta: {
        from: existingLead.stage,
        to: newLead.stage,
      },
    });
  }

  return activities;
}
```

This function compares the old lead with the new lead and generates Activity records for any fields that changed. It returns an array because a single update might trigger multiple activities (e.g., changing both status and stage).

**Reference file:** `src/services/lead/service.ts`

Updated the `updateLead` function to use this helper:

```typescript
export async function updateLead(
  profile: Profile,
  id: string,
  data: EditLeadRequest,
) {
  const existingLead = await dbGetLeadById(id);

  if (!existingLead) {
    throw new LeadServiceError("Lead not found", 404);
  }

  // ... authorization checks ...

  const activities = buildLeadChangeActivities({
    leadId: id,
    actorId: profile.id,
    existingLead,
    newLead: data,
  });

  const result = await prisma.$transaction(async (tx) => {
    const updatedLead = await dbUpdateLead(id, data, tx);
    const activitiesCreated = await ActivityService.create(activities, tx);
    if (!activitiesCreated.success)
      throw new Error("Failed to create activities");

    return {
      lead: updatedLead,
      activities: activitiesCreated.count,
    };
  });

  return result;
}
```

**Key design — Transaction orchestration:**

The transaction does THREE things atomically:

1. Update the lead
2. Create activity records for what changed
3. Verify the activities were created successfully

If any step fails, all three are rolled back. You never end up with a lead update that has no corresponding activity record.

**Key design — Service-to-service calls:**

The lead service calls `ActivityService.create()` from within a transaction, passing the `tx` client. This is how services talk to each other while staying inside the same transaction. The activity service's `dbCreateActivities` function doesn't care whether it's called inside a transaction or standalone — it just uses the client it was given.

---

### Phase 8: Activities API Route

**Concepts covered:** Ultra-thin route handlers, delegating to services.

**What we did:**

Created `src/app/api/leads/[id]/activities/route.ts` to expose activities via HTTP.

**Reference file:** `src/app/api/leads/[id]/activities/route.ts`

```typescript
export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> },
) {
  try {
    const { id } = await params;
    const profile = await authenticateUser();

    const searchParams = request.nextUrl.searchParams;
    const page = searchParams.get("page");
    const pageSize = searchParams.get("pageSize");

    const validated = ActivitySchema.getByLeadId.parse({
      leadId: id,
      page: page,
      pageSize: pageSize,
    });

    const activities = await ActivityService.getByLeadId(validated, {
      id: profile.id,
      role: profile.role,
    });

    return NextResponse.json({ success: true, data: activities });
  } catch (error) {
    return handleRouteError(error);
  }
}
```

**Key design — The flow:**

```
1. Extract leadId from dynamic route parameter [id]
2. Authenticate the user
3. Extract query parameters (page, pageSize)
4. Validate with ActivitySchema.getByLeadId
5. Call ActivityService.getByLeadId
6. Return as JSON
```

Each step does ONE thing. If validation fails, the error is caught and handled. If the user isn't authenticated, `authenticateUser()` throws before the service is called.

**Key design — Thin route handlers:**

Route handlers are HTTP concerns only — parsing requests, calling services, formatting responses. All business logic lives in the service layer. This keeps routes short and testable.

---

### Phase 9: TanStack Query Hook — useActivities

**Concepts covered:** Custom query hooks, query keys for caching, async data fetching.

**What we did:**

Created `src/lib/tanstack/useActivities.ts` with a hook that fetches activities with TanStack Query's caching.

**Reference file:** `src/lib/tanstack/useActivities.ts`

```typescript
export function useGetLeadActivities(request: GetLeadActivitiesRequest) {
  return useQuery({
    queryKey: ["activities", request],
    queryFn: async (): Promise<ListLeadActivitiesResponseData> => {
      const { data } = await api.get(`/leads/${request.leadId}/activities`, {
        params: {
          page: request.page,
          pageSize: request.pageSize,
        },
      });

      return data.data;
    },
  });
}
```

**Key design — Query key includes request:**

The query key is `["activities", request]`. If the request changes (different leadId, page, or pageSize), TanStack Query treats it as a different query. This means:

- User views lead 1's activities → cache key is `["activities", { leadId: "1", page: 1, pageSize: 10 }]`
- User switches to lead 2 → new request, new cache key, new fetch

Without including the request in the key, changing pages would hit the cache incorrectly.

**Key design — Type-safe response:**

The `queryFn` returns `Promise<ListLeadActivitiesResponseData>`. TypeScript knows that `data` will be of that type. In the component, `useQuery` returns `data` typed as `ListLeadActivitiesResponseData | undefined`.

---

### Phase 10: Activity Timeline Component

**Concepts covered:** Client Components, icon libraries, relative timestamps, UI composition.

**What we did:**

Created `src/components/leads/lead-details/Timeline.tsx` that renders the activity log on the lead detail page.

**Reference file:** `src/components/leads/lead-details/Timeline.tsx`

The component is a Client Component because it uses React hooks and state:

```typescript
"use client"

export const Timeline = ({ leadId }: { leadId: string }) => {
  const [page, setPage] = useState(1);
  const pageSize = 10;
  const { data, isLoading, isError, error } = useGetLeadActivities({
    leadId,
    page,
    pageSize,
  })

  if (isLoading) {
    return <div>Loading...</div>;
  }

  if (isError) {
    return <div>Error: {getApiErrorMessage(error, "Failed to load activities")}</div>;
  }

  const activities = data?.activities ?? []
  if (activities.length === 0) {
    return <div className="text-center text-muted-foreground">No activities found</div>;
  }

  const total = data?.pagination.total ?? 0;
  const pageCount = data?.pagination.pages ?? 0;
  const startItem = total === 0 ? 0 : (page - 1) * pageSize + 1;
  const endItem = total === 0 ? 0 : Math.min(page * pageSize, total);

  return <div className="space-y-0">
    {activities.map((activity, idx) => {
      const Icon = activityIcons[activity.type]
      const label = activityLabels[activity.type]
      const isLast = idx === activities.length - 1

      return (
        <div key={activity.id} className="flex gap-3">
          <div className="flex flex-col items-center">
            <div className="flex h-8 w-8 shrink-0 items-center justify-center rounded-full bg-muted">
              <Icon className="h-4 w-4 text-muted-foreground" />
            </div>
            {!isLast && <div className="w-px flex-1 bg-border" />}
          </div>
          <div className="flex-1 pb-6">
            <div className="flex flex-wrap items-baseline gap-x-2 gap-y-0.5">
              <span className="text-sm font-medium">{label}</span>
              {activity.actor && (
                <span className="text-xs text-muted-foreground">by {activity.actor.name}</span>
              )}
              <span
                className="ml-auto text-xs text-muted-foreground"
                title={new Date(activity.createdAt).toLocaleString()}
              >
                {formatDistanceToNow(new Date(activity.createdAt), { addSuffix: true })}
              </span>
            </div>
            {activity.content && <p className="mt-1 text-sm text-muted-foreground">{activity.content}</p>}
          </div>
        </div>
      )
    })}

    <Pagination startItem={startItem} endItem={endItem} total={total} page={page} pageCount={pageCount} isLoading={isLoading} setPage={setPage} />
  </div>;
};

const activityIcons: Record<ActivityType, LucideIcon> = {
  [ActivityType.LEAD_CREATED]: PlusCircle,
  [ActivityType.NOTE]: Pencil,
  [ActivityType.CALL_ATTEMPT]: Phone,
  [ActivityType.STATUS_CHANGE]: CheckCircle,
  [ActivityType.STAGE_CHANGE]: ArrowRight,
  [ActivityType.ASSIGNMENT_CHANGE]: User,
  [ActivityType.REMINDER_CREATED]: Bell,
  [ActivityType.ATTACHMENT_ADDED]: Paperclip,
  [ActivityType.AI_LEAD_BRIEF_GENERATED]: Brain,
  [ActivityType.AI_FOLLOWUP_DRAFT_GENERATED]: Brain,
}

const activityLabels: Record<ActivityType, string> = {
  [ActivityType.LEAD_CREATED]: "Lead Created",
  [ActivityType.NOTE]: "Note",
  [ActivityType.CALL_ATTEMPT]: "Call Attempt",
  [ActivityType.STATUS_CHANGE]: "Status Change",
  [ActivityType.STAGE_CHANGE]: "Stage Change",
  [ActivityType.ASSIGNMENT_CHANGE]: "Assignment Change",
  [ActivityType.REMINDER_CREATED]: "Reminder Created",
  [ActivityType.ATTACHMENT_ADDED]: "Attachment Added",
  [ActivityType.AI_LEAD_BRIEF_GENERATED]: "AI Lead Brief Generated",
  [ActivityType.AI_FOLLOWUP_DRAFT_GENERATED]: "AI Followup Draft Generated",
}
```

**Key design — Icon and label maps:**

`activityIcons` and `activityLabels` are lookup objects that map activity types to Lucide icons and human-readable labels. This is a pattern you'll see throughout the app — data-driven UI rendering.

**Key design — Relative timestamps:**

`formatDistanceToNow(new Date(activity.createdAt), { addSuffix: true })` shows "2 minutes ago" instead of a raw timestamp. The `title` attribute shows the full timestamp on hover for precision.

**Key design — Timeline visual structure:**

Each activity is rendered as:

1. An icon in a circle on the left
2. A vertical line connecting to the next activity (unless it's the last one)
3. The activity label, actor, and timestamp on the right
4. The activity content (e.g., "Status changed from OPEN to WON") below

This creates a visual timeline that's easy to scan.

---

## What We Did NOT Build (Your Assignment)

We completed the infrastructure for reading and displaying activities, but did not build the write paths for notes or call attempts. These are intentionally left for you to practice the full-stack pattern from Session 2.

The timeline component has a placeholder for where you'll add UI to create notes and log call attempts, but the backend routes, mutation hooks, and service functions are yours to build.

---

## Supplementary Videos

### Review What We Covered

These videos reinforce the concepts from Session 3. Watch them if anything felt unclear.

**Service Layer Architecture — Why Separate Concerns?**
[Backend Architecture Patterns](https://www.youtube.com/watch?v=P8_szbL40HE) by Raw Coding (~22 min)
Covers the schema → db → service → route pattern. This is exactly what we formalized in the 5-file service module.

**TanStack Query — Advanced Query Keys**
[React Query Query Keys Explained](https://www.youtube.com/watch?v=VJppm-w-pZE) by TkDodo (~10 min)
Covers how query keys affect caching and invalidation. Important for understanding why we include the request object in the query key.

**Prisma Transactions & Batch Operations**
[Prisma Transactions](https://www.prisma.io/docs/orm/prisma-client/queries/transactions) (official docs)
Read the section on batch operations (`createMany`, `deleteMany`). This is what we used for creating multiple activities at once.

**Lucide Icons Customization**
[Lucide React Icons](https://lucide.dev/) (official docs)
Just a reference — we used Lucide for the timeline icons. The library is huge; pick icons that make sense for your use case.

---

## Assignment: Build Note & Call Attempt Features

You now have a complete infrastructure for logging activities. Your assignment is to build the two write paths that create activities: notes and call attempts. Both follow the exact same full-stack pattern from Session 2.

### Task 1: Create Note Activity (estimated: 2-2.5 hours)

Build the complete feature for users to add notes to a lead. This includes the backend route, mutation hook, and UI.

**Backend Route:** Create `POST /api/leads/[id]/activities/note` endpoint.

```typescript
// in src/app/api/leads/[id]/activities/route.ts

export async function POST(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> },
) {
  try {
    const { id } = await params;
    const profile = await authenticateUser();
    const body = await request.json();

    // Validate: { content: string, min 1 char }
    // Create activity with type NOTE
    // Call ActivityService.create
    // Return the created activity
  } catch (error) {
    return handleRouteError(error);
  }
}
```

**Schema:** Add to `src/services/activity/schema.ts`:

```typescript
export const createNoteSchema = z.object({
  content: z.string().min(1).max(5000),
});

export type CreateNoteRequest = z.infer<typeof createNoteSchema>;
```

**Key design consideration:** Notes don't have a `meta` field (no before/after). When you call `ActivityService.create`, pass:

```typescript
{
  leadId: id,
  actorId: profile.id,
  type: ActivityType.NOTE,
  // no meta — content comes directly from the note text
}
```

But wait — `buildActivityContent` returns `null` when there's no `meta`. This is a design challenge: how do you get the note text into the database?

**Hint:** Think about where to put the note's text. Should it go in the `content` field in the database? What would `buildActivityContent` need to return for notes? Consider modifying `buildActivityContent` to handle `NOTE` type differently, or passing the content through a different mechanism.

**Mutation Hook:** Create `useCreateNote` in `src/lib/tanstack/useActivities.ts`:

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
```

**UI:** Add a note form to the Timeline component or as a separate component that opens a dialog.

- Textarea for the note content
- Submit button
- Clear on success, show validation errors on failure
- Disable submit button while mutating

**Hint:** Look at how `useCreateLead` works — the mutation pattern is identical. The key difference is that you're POST-ing to `/api/leads/[id]/activities` instead of `/api/leads`.

### Task 2: Log Call Attempt (estimated: 2-2.5 hours)

Build the feature for logging call attempts with structured data: outcome and optional notes.

**Schema:** Add to `src/services/activity/schema.ts`:

```typescript
const callOutcomeEnum = z.enum([
  "NO_ANSWER",
  "ANSWERED",
  "WRONG_NUMBER",
  "BUSY",
  "CALL_BACK_LATER",
]);

export const createCallAttemptSchema = z.object({
  outcome: callOutcomeEnum,
  notes: z.string().max(5000).optional(),
});

export type CreateCallAttemptRequest = z.infer<typeof createCallAttemptSchema>;
```

**Backend Route:** Create a separate endpoint `POST /api/leads/[id]/activities/call-attempt/route.ts`:

```typescript
export async function POST(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> },
) {
  try {
    const { id } = await params;
    const profile = await authenticateUser();
    const body = await request.json();

    // Validate with createCallAttemptSchema
    // Build content string: "ANSWERED — Notes go here" or just "ANSWERED"
    // Create activity with type CALL_ATTEMPT
    // Return success
  } catch (error) {
    return handleRouteError(error);
  }
}
```

**Content encoding:** Use this pattern:

```typescript
const content = input.notes
  ? `${input.outcome} — ${input.notes}`
  : input.outcome;
```

**Mutation Hook:** Create `useLogCallAttempt` in `src/lib/tanstack/useActivities.ts`:

```typescript
export function useLogCallAttempt(leadId: string) {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: async (data: CreateCallAttemptRequest) => {
      const response = await api.post(`/leads/${leadId}/call-attempt`, data);
      return response.data.data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["activities"] });
    },
  });
}
```

**UI:** Create a `LogCallDialog` component with:

- A Select dropdown for the outcome (NO_ANSWER, ANSWERED, WRONG_NUMBER, BUSY, CALL_BACK_LATER)
- A textarea for optional notes
- Submit button disabled until outcome is selected
- Dialog opens from a "Log Call" button on the timeline

**Hint:** For the outcome select, use shadcn's Select component (`npx shadcn@latest add select`). Map each enum value to a human-readable label in the UI.

### Task 3: Integrate Into Timeline

**What to do:**

Add buttons to the Timeline component that open the note form and call dialog. After submission, the activities should appear immediately in the timeline (via cache invalidation).

**Expected outcome:**

1. User views a lead's timeline
2. User clicks "Add Note", enters text, submits
3. New note activity appears at the top of the timeline
4. User clicks "Log Call", selects outcome and notes, submits
5. New call activity appears in the timeline

Both should refresh the timeline via `queryClient.invalidateQueries({ queryKey: ["activities"] })` in the mutation's `onSuccess`.

### Task 4: Watch Pre-Session 4 Videos

#### 1. Upstash QStash — Serverless Scheduling

**What it should cover:** What QStash is — a message queue / scheduler for serverless environments. The publish → callback flow: you send a message with a target URL and a delay, QStash calls that URL when the time comes. Understand at-least-once delivery (your callback might be called more than once, so you need to handle that).

> **Why this matters:** The entire reminder system in Session 4 is built on QStash. If you understand the "publish a delayed message → receive a callback" pattern, the live coding will click immediately.

**Suggested video:** [How to avoid serverless function timeouts in NextJs](https://www.youtube.com/watch?v=4jQBKoNQhzU) by Hamed Bahram (~17 min)

---

#### 2. Redis — The Basics

**What it should cover:** What Redis is (an in-memory key-value store), basic operations (SET, GET, DEL), and TTL (time-to-live / automatic key expiration). Just the fundamentals — we'll use Redis for a single focused purpose.

> **Why this matters:** We use Redis in Session 4 for idempotency — making sure a reminder callback doesn't fire twice. You just need to understand "set a key, check if a key exists, keys expire automatically."

**Suggested video:** [How to use Redis Caching for Incredible Performance](https://www.youtube.com/watch?v=-5RTyEim384) by Josh Tried Coding (~13 min)

---

## Checkpoint Before Session 4

Before Session 4, verify all of the following:

- [ ] Activity timeline renders on the lead detail page with pagination
- [ ] Status/stage changes show in timeline (e.g., "Status changed from OPEN to WON")
- [ ] "Add Note" button opens a dialog with a textarea
- [ ] Submitting a note creates an activity that appears in the timeline immediately
- [ ] "Log Call" button opens a dialog with outcome dropdown + optional notes
- [ ] Logging a call creates an activity showing the outcome and notes
- [ ] All activities have icons and relative timestamps (e.g., "2 minutes ago")
- [ ] Service layer structure: schema / db / service / helpers / index (you can see this in the activity module)
- [ ] Route handlers are thin — they authenticate, validate, call the service, and respond
- [ ] Code committed and pushed to GitHub

---

## Resources

- [TanStack Query v5 — Mutations](https://tanstack.com/query/v5/docs/react/guides/mutations) — `useMutation`, `onSuccess`, cache invalidation
- [Zod Docs](https://zod.dev/) — Schema validation, `.superRefine()`, type inference
- [Next.js Route Handlers](https://nextjs.org/docs/app/building-your-application/routing/route-handlers) — Request/response handling
- [Prisma — Batch Operations](https://www.prisma.io/docs/orm/prisma-client/queries/crud#create-multiple-records) — `createMany`
- [Prisma — Transactions](https://www.prisma.io/docs/orm/prisma-client/queries/transactions) — Atomic multi-step operations
- [date-fns — formatDistanceToNow](https://date-fns.org/docs/formatDistanceToNow) — Relative timestamps
- [Lucide Icons](https://lucide.dev/) — Icon library
- [shadcn/ui Components](https://ui.shadcn.com/) — Dialog, Select, Textarea, Button, and more

---

_Session 3 Complete. You now understand the 5-file service module pattern and how to build an audit trail. This is the foundation for everything else in the app. See you in Session 4._
