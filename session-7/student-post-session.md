# Session 7: Post-Session Recap & Tutorial

**Manager Dashboard + File Attachments**

---

## What We Built

Session 7 gave the CRM its first taste of **analytics** and **document management** — the two features every real sales team asks for by week two. Until now, the dashboard was a static stub and leads had no way to carry files. Tonight we fixed both, using patterns you've already built six times.

By the end of Session 7, the `whispyr-academy-crm` repo had:

- **Recharts dependency** — JIT-installed the moment the dashboard needed its first chart (`npm install recharts`). No earlier.
- **Dashboard service module** — the seventh service to follow the 5-file pattern (`schema`, `db`, `helpers`, `service`, `index`) exposing `getManagerOverview` as a single orchestration point.
- **Database aggregations with `groupBy`** — `dbGetLeadStageCounts` and `dbGetLeadsCountByStatus` use Prisma's `groupBy` with `_count: { _all: true }` instead of fetching rows and counting in JavaScript.
- **Parallel query orchestration** — `getManagerOverview` runs six aggregation queries in `Promise.all([...])` so total latency equals the slowest query, not the sum of all of them.
- **Zero-fill helpers** — `toStageCounts` and `toStatusCounts` ensure every enum value appears in the chart data even when zero leads match, so bars and pie slices never silently disappear.
- **`DashboardServiceError`** — a new domain error class wired into `handleRouteError` so dashboard failures surface as proper HTTP status codes.
- **Dashboard API route** — `GET /api/dashboard/overview` gated by `authenticateUser([Role.MANAGER, Role.ADMIN])`. Agents receive a clean 403.
- **`useDashboardOverview` hook** — a TanStack Query hook with a 60-second `staleTime`, keyed on `["dashboard", "overview"]`.
- **Dashboard page rewrite** — `/dashboard` now renders four KPI cards (Total, Open, Won, Lost), a `<BarChart>` of leads by stage, a `<PieChart>` of leads by status, and a top-agents leaderboard table. Every chart lives inside a `<ResponsiveContainer>` with an explicit height.
- **`Attachment` Prisma model** — a new table with `id`, `leadId` (cascading delete), `uploadedById`, `fileName`, `storagePath` (unique), `mimeType`, `sizeBytes`, and `createdAt`. Relations added to `Lead` and `Profile`.
- **`add_attachment_model` migration** — generated with `npx prisma migrate dev --name add_attachment_model` and the client regenerated live.
- **`lead-attachments` Supabase Storage bucket** — private, 10MB file size limit, created through the Supabase dashboard.
- **Attachments service module** — the eighth service to follow the 5-file pattern, with helpers that wrap the Supabase Admin storage client and generate signed URLs on every list call.
- **Server-side validation constants** — `ALLOWED_MIME_TYPES` (pdf, png, jpeg, webp) and `MAX_FILE_SIZE_BYTES = 10 * 1024 * 1024` exported from `src/services/attachments/schema.ts`.
- **Double-write pattern with cleanup** — `uploadForLead` uploads the file to Storage first, then creates the metadata row and an `ATTACHMENT_ADDED` activity inside a single Prisma transaction. If the transaction fails, the orphaned file is deleted from Storage automatically.
- **`AttachmentServiceError`** — registered in `handleRouteError` alongside the other service errors.
- **Attachment API routes** — `GET /api/leads/[id]/attachments` lists files with fresh 1-hour signed URLs, and `POST /api/leads/[id]/attachments` accepts `FormData` and runs the full upload pipeline.
- **`useAttachments` and `useUploadAttachment` hooks** — TanStack hooks with a 55-minute `staleTime` (deliberately shorter than the 60-minute signed URL TTL) and invalidation of both `attachments` and `activities` query keys on upload.
- **Files tab on the lead detail page** — `src/components/leads/lead-details/Files.tsx` with drag-and-drop upload, empty state, file list with icons and relative timestamps, and a per-file Download button that opens the signed URL in a new tab.
- **Activity logging on upload** — every upload writes an `ATTACHMENT_ADDED` row inside the same transaction as the metadata insert, so the activity timeline stays consistent with Storage.

> **Note:** We built the manager dashboard and file attachments end-to-end. We did **NOT** build the agent personal dashboard, the role-aware leads table, bulk reassignment, the delete-attachment flow, or the production polish pass. Those are your **assignment before Session 8**. See the "Assignment" section below.

---

## Step-by-Step Walkthrough

If you missed the session or want to review, follow these steps in order.

---

### Phase 1: Why Aggregate At The Database

**Concepts covered:** N+1 performance traps, database aggregations with Prisma `groupBy`, parallel query orchestration with `Promise.all`.

**The trap:**

Imagine you have 10,000 leads and you want to know how many are in each stage. The naive approach looks like this:

```typescript
// DO NOT DO THIS
const leads = await prisma.lead.findMany();
const prospectCount = leads.filter((l) => l.stage === "PROSPECT").length;
const negotiationCount = leads.filter((l) => l.stage === "NEGOTIATION").length;
```

This hydrates 10,000 rows over the wire, allocates 10,000 JavaScript objects, and iterates the whole array multiple times. It will work fine on your dev database. It will time out in production.

**The correct approach:**

```typescript
const rows = await prisma.lead.groupBy({
  by: ["stage"],
  _count: { _all: true },
});
// rows = [{ stage: "PROSPECT", _count: { _all: 42 } }, ...]
```

`groupBy` tells the database "give me one row per stage with the count attached." The database uses its indexes and query planner to return a tiny result set — usually in milliseconds — regardless of how many leads you have.

**Rule for the rest of your career:** never count in JavaScript what you can count in SQL.

**`Promise.all` for parallelism:**

`getManagerOverview` needs six aggregations (total leads, stage counts, status counts, won this month, overdue reminders, top agents). Running them sequentially means waiting for all six to finish one after the other. Running them with `Promise.all([...])` means the total time equals the slowest single query.

```typescript
const [totalLeads, stageCounts, statusCounts, overdueReminders, topAgents] =
  await Promise.all([
    dbGetTotalLeads(),
    dbGetLeadStageCounts(),
    dbGetLeadsCountByStatus(),
    dbGetOverdueReminderCount(),
    dbGetTopAgents(5),
  ]);
```

---

### Phase 2: Dashboard Service Module

**Reference files:** `src/services/dashboard/{schema,db,helpers,service,index}.ts`

Seventh time, same five files. You know the drill:

- **`schema.ts`** defines `StageCount`, `StatusCount`, `TopAgent`, and the top-level `DashboardOverview` return type.
- **`db.ts`** holds every query: `dbGetTotalLeads`, `dbGetLeadStageCounts`, `dbGetLeadsCountByStatus`, `dbGetOverdueReminderCount`, `dbGetTopAgents`. Each one is a single Prisma call with a tight `select` or aggregation.
- **`helpers.ts`** exports `toStageCounts` and `toStatusCounts`, which zero-fill missing enum values so Recharts always has every bar and every slice. Without zero-fill, a stage with zero leads silently disappears from the chart — a visual lie.
- **`service.ts`** exposes `getManagerOverview`, the orchestration point. All Promise.all parallelism lives here. It also exports `DashboardServiceError`, which extends `Error` with a `statusCode` property.
- **`index.ts`** re-exports the public API.

**Key design — registering the error class:**

Open `src/utils/handleRouteError.ts` and add `DashboardServiceError` to the union of instances it catches. This is how every service error becomes a proper HTTP response.

```typescript
if (
  error instanceof AuthenticationError ||
  error instanceof LeadServiceError ||
  error instanceof NotificationServiceError ||
  error instanceof AdminServiceError ||
  error instanceof DashboardServiceError
) {
  return NextResponse.json(
    { error: error.message },
    { status: error.statusCode },
  );
}
```

---

### Phase 3: Dashboard Route, Hook, and UI

**Reference files:**
`src/app/api/dashboard/overview/route.ts`
`src/lib/tanstack/useDashboard.ts`
`src/app/(protected)/dashboard/page.tsx`

**Route — role-gated from the first line:**

```typescript
export const GET = async () => {
  try {
    await authenticateUser([Role.MANAGER, Role.ADMIN]);
    const overview = await getManagerOverview();
    return NextResponse.json(overview);
  } catch (error) {
    return handleRouteError(error);
  }
};
```

Notice the allowlist `[Role.MANAGER, Role.ADMIN]`. Agents hitting this endpoint get a 403 from `authenticateUser` before any query runs. We'll add the agent branch in homework.

**Hook — 60s staleTime:**

Dashboards don't need to be real-time. The numbers change over hours, not seconds. A `staleTime` of 60 seconds means TanStack will cache the result between renders, tab switches, and brief navigations, which keeps the database calm.

```typescript
export const useDashboardOverview = () =>
  useQuery({
    queryKey: ["dashboard", "overview"],
    queryFn: async () => {
      const { data } = await api.get<DashboardOverview>("/dashboard/overview");
      return data;
    },
    staleTime: 60_000,
  });
```

**Page — Recharts lives in `ResponsiveContainer`:**

Every chart in the new dashboard is wrapped in a `<ResponsiveContainer width="100%" height="100%">` inside a parent with an explicit height (`h-[320px]`). This is non-negotiable. Recharts has no intrinsic size — if its parent is zero-height, the chart silently renders nothing.

```tsx
<div className="h-[320px]">
  <ResponsiveContainer width="100%" height="100%">
    <BarChart data={overview.stageCounts}>
      <XAxis dataKey="name" />
      <YAxis allowDecimals={false} />
      <Tooltip />
      <Bar dataKey="value" fill="hsl(var(--primary))" />
    </BarChart>
  </ResponsiveContainer>
</div>
```

The `fill="hsl(var(--primary))"` binds the bar color to your Tailwind theme token. Change the theme, the chart updates. One source of truth.

---

### Phase 4: Object Storage Concepts

**The rule:** never store binary data in Postgres. Files go in object storage. Metadata goes in the database.

**Public vs private buckets:**

- A **public bucket** serves any file at a guessable URL. Anyone who knows (or can guess) the path can download. Fine for avatars on a public profile page. Disastrous for contracts, IDs, call recordings.
- A **private bucket** refuses every request by default. To let a user download a file, the server mints a **signed URL** — a short-lived, cryptographically signed link that encodes the path, an expiry timestamp, and an HMAC signature. Any attempt to change any part of the URL invalidates the signature.

**Default TTL:** 1 hour. That's a balance between usability (the user has time to actually download) and security (a leaked URL stops working soon).

**Rule for the rest of your career:** private bucket + signed URL is the production default. Public buckets are for public files. Everything else is private.

**Adding the `Attachment` model:**

```prisma
model Attachment {
  id            String   @id @default(cuid())
  leadId        String
  lead          Lead     @relation(fields: [leadId], references: [id], onDelete: Cascade)
  uploadedById  String
  uploadedBy    Profile  @relation(fields: [uploadedById], references: [id])
  fileName      String
  storagePath   String   @unique
  mimeType      String
  sizeBytes     Int
  createdAt     DateTime @default(now())

  @@index([leadId])
}
```

`onDelete: Cascade` means deleting a lead automatically deletes its attachment rows. The file cleanup in Storage is handled by your homework delete flow. `storagePath` is UNIQUE in the database — our collision detector.

Then:

```bash
npx prisma migrate dev --name add_attachment_model
npx prisma generate
```

**Bucket creation:**

Supabase Dashboard → Storage → New bucket → name `lead-attachments`, **Public: OFF**, file size limit `10 MB`, allowed MIME types blank (we validate server-side).

---

### Phase 5: Attachments Service Module

**Reference files:** `src/services/attachments/{schema,db,helpers,service,index}.ts`

**`schema.ts` — the validation constants:**

```typescript
export const ALLOWED_MIME_TYPES = [
  "application/pdf",
  "image/png",
  "image/jpeg",
  "image/webp",
] as const;

export const MAX_FILE_SIZE_BYTES = 10 * 1024 * 1024; // 10MB
```

Both constants are exported so the route, the service, and (eventually) the client can all reference the same source of truth.

**`helpers.ts` — the Supabase admin wrapper:**

This file is the only place that imports `supabaseAdmin`. Every other file calls these helpers, which means `supabaseAdmin` never accidentally ends up in a client bundle.

- `BUCKET_NAME = "lead-attachments"`
- `SIGNED_URL_TTL_SECONDS = 3600` — exactly 1 hour
- `buildStoragePath(leadId, fileName)` — sanitizes the filename (strips unsafe characters), prepends a millisecond timestamp (`${Date.now()}-${safeName}`), and nests under `${leadId}/`. This gives us a path that is collision-proof across leads and within a lead.
- `uploadToStorage(path, file)` — calls `supabaseAdmin.storage.from(BUCKET_NAME).upload(path, file, { upsert: false, contentType: file.type })`. `upsert: false` means the upload fails if the path is taken — another safety net.
- `deleteFromStorage(path)` — used only for cleanup on DB failure.
- `createSignedDownloadUrl(path)` — calls `createSignedUrl(path, SIGNED_URL_TTL_SECONDS)`. Never cached.

**`service.ts` — the double write with cleanup:**

`uploadForLead` is the most important function in this module. Walk through it slowly:

1. Validate size and MIME type. Throw `AttachmentServiceError` with a clean message if invalid. This runs before any network I/O.
2. Build the storage path.
3. Upload the file to Storage. If the upload fails, throw — nothing to clean up.
4. Open a Prisma transaction:
   - Insert the `Attachment` row.
   - Insert the `Activity` row with type `ATTACHMENT_ADDED`.
5. If the transaction throws, catch it, call `deleteFromStorage(path)` to clean up the orphan, then re-throw so the route handler surfaces the error.
6. Return the attachment with a fresh signed URL.

**Key design — atomic activity logging:**

The activity row lives **inside the transaction** with the attachment row. Either both commit or both roll back. There is no state where the attachment exists without the activity, or vice versa. This is how you keep your audit trail honest.

**`listForLead` regenerates signed URLs on every call:**

Never store signed URLs in the database. Never cache them in localStorage. Regenerate on read. The 55-minute `staleTime` on the hook is tuned to keep the cached URLs well under the 60-minute TTL.

---

### Phase 6: Route, Hook, and Files Tab

**Reference files:**
`src/app/api/leads/[id]/attachments/route.ts`
`src/lib/tanstack/useAttachments.ts`
`src/components/leads/lead-details/Files.tsx`

**Route — FormData parsing:**

Two things to notice:

- `params` is a Promise in Next.js 16 — always `await` it.
- The request body is parsed with `req.formData()`, not `req.json()`. Browsers send multipart uploads as FormData; trying to parse JSON will throw before you even see the file.

**Hook — invalidate two keys on upload:**

Uploading a file creates two rows (attachment + activity), so we invalidate both query keys. This is what keeps the Files tab and the Activity timeline in sync without a page refresh.

**`Files.tsx` — a dumb component:**

The Files component does no business logic. It reads `useAttachments(leadId)` for the list and calls `useUploadAttachment(leadId).mutate(file)` on upload. Every rule about what's valid, what's stored, and how URLs are signed lives in the service layer. That's intentional — the component stays simple because the service is doing the hard work.

---

## Key Concepts Recap

- **Aggregate at the database.** `groupBy` + `_count` is the first tool you reach for. `findMany().filter().length` is a red flag.
- **Parallelize independent queries.** `Promise.all([...])` is free performance.
- **Zero-fill your enums.** If a stage has zero leads, the chart should still show that bar at zero. Missing bars are lies.
- **Private buckets by default.** Public buckets are for public files. Everything else is private.
- **Signed URLs, not public URLs.** Regenerate on every list. Never cache, never store.
- **Never import the admin client in a client component.** Keep it in helpers files only. One mistake and your service role key ships to the browser.
- **Double writes need cleanup.** Upload the file, then try to write the row. If the row fails, delete the file. No orphans.
- **Activity logs live inside the transaction.** Atomic audit trail. No partial state.

---

## Assignment (Due Before Session 8)

Session 7 homework has five tasks. Estimated total: ~6.25 hours. Full details in `snippets/06-homework-assignment.md`.

**1. Agent Personal Dashboard (~1 hour)** — Extend the dashboard service with `getAgentOverview(profileId)`. Update the route to branch on `profile.role`. Replace the 403 with the new branch.

**2. Role-Aware Leads Table (~1.5 hours)** — Add an `assignedTo` column and bulk-selection checkboxes to the existing `/leads` table. Conditionally render only when `role === "MANAGER" || role === "ADMIN"`. No new page file.

**3. Bulk Reassignment (~2 hours)** — `POST /api/leads/reassign` route, `useReassignLeads` hook, inline Dialog from the leads-table toolbar. One Prisma transaction, one `ASSIGNMENT_CHANGE` activity per lead.

**4. Delete Attachment (~45 minutes)** — `DELETE /api/leads/[id]/attachments/[attachmentId]`. Delete the Storage object FIRST, then the DB row. Trash icon button with `AlertDialog` confirmation.

**5. Production Polish Pass (~1 hour)** — Every page, every mutation: loading state, empty state, success toast, error toast with real messages, disabled submit buttons during mutations. Fix anything missing.

---

## Before Session 8

- Finish all five homework tasks above.
- Push your branch to the `whispyr-academy-crm` repo.
- Watch the Session 8 pre-session videos: **Vercel Deployment for Next.js** and **Prisma Migrations in Production** (links in `videos.md`).
- Skim the Vercel dashboard — we'll be deploying live in Session 8.

Session 8 is the final session: deployment, seed data, architecture review, and demo day. The CRM should feel nearly complete by the time you arrive.

See you next week.
