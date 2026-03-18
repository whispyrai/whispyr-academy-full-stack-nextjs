# Session 2: Post-Session Recap & Tutorial

**Leads CRUD + Validation — From Login to a Working Data Pipeline**

---

## What We Built

Session 2 was a longer session where we caught up on Session 1 carryover (login page, sidebar, app shell) and then built the entire data pipeline for leads: schema validation, role-scoped API routes, atomic database writes, and client-side caching with TanStack Query.

By the end of the session, the `whispyr-academy-crm` repo had:

- A working login page connected to Supabase Auth
- A protected app shell with role-based sidebar navigation and user dropdown
- A `Lead` model and `Activity` model in the database with full migrations
- A three-tier service layer (schema → db → service) for leads
- A reusable `authenticateUser()` helper eliminating auth boilerplate
- Zod validation schemas for lead creation and listing
- `GET /api/leads` — role-scoped read endpoint (agents see only their leads, managers/admins see all)
- `POST /api/leads` — atomic create endpoint (lead + activity in one transaction)
- TanStack Query hooks (`useGetLeads`, `useCreateLead`) with cache invalidation

> **Note:** We completed all the backend data-logic and TanStack Query wiring, but did not build the frontend UI for displaying leads (table, filters, pagination) or the create lead dialog. The remaining UI work is your **assignment before Session 3**. See the "Assignment" section below.

---

## Step-by-Step Walkthrough

If you missed the session or want to review, follow these steps in order. Each step references the exact files in the [`whispyr-academy-crm`](https://github.com/whispyr-academy/whispyr-academy-crm) repo.

---

### Phase 0a: Supabase Server Client Fix

**What we did:**

Before building anything new, we fixed an issue with the Supabase server client initialization. The original code from Session 1 was creating the client at module load time, which caused issues with Next.js request-scoped cookie access.

**Reference file:** `src/lib/supabase/server.ts`

The fix: moved the Supabase server client creation into the request path so it always has access to the current request's cookies. This is a subtle but important change — Supabase's `createServerClient` needs access to Next.js cookies, and cookies are only available inside a request handler, not at module initialization time.

**Software engineering principle — Request-scoped initialization:** Any code that depends on per-request context (cookies, headers, auth tokens) must be created within the request lifecycle, not at module load time. Module-level code runs once and is shared across all requests — it can't access per-request data.

---

### Phase 0b: shadcn/ui Initialization (Session 1 Carryover)

**Concepts covered:** JIT (Just-In-Time) tool installation — installing the UI library only when we need our first component.

**What we did:**

1. Initialized shadcn/ui:

```bash
npx shadcn@latest init
```

This created `components.json` (shadcn config), `src/lib/utils.ts` (the `cn()` utility for merging Tailwind classes), and updated `globals.css` with CSS variables for the design system.

2. Added the components needed for the login page:

```bash
npx shadcn@latest add button card input label
```

**Reference files in the repo:**

- `components.json` — shadcn/ui configuration
- `src/lib/utils.ts` — `cn()` class merging utility
- `src/app/globals.css` — Updated with CSS variables for light/dark theming
- `src/components/ui/button.tsx`, `card.tsx`, `input.tsx`, `label.tsx` — shadcn component source files

**Key concept — JIT installation:**

We didn't install shadcn/ui in Session 1 because we had no UI to build yet. Now that we need a login form, we install it. Each `npx shadcn@latest add [component]` copies the component source code into your project — you own it, you can modify it. This is different from traditional component libraries where the code is hidden inside `node_modules`.

---

### Phase 1: Login Page (Session 1 Carryover)

**Concepts covered:** Client Components, `"use client"` directive, Supabase browser client for auth, controlled form inputs, `router.push()` + `router.refresh()`.

**What we did:**

Created the login page at `/login` — an email/password form that authenticates against Supabase.

**Reference file:** `src/app/login/page.tsx`

The login page:

1. Uses `"use client"` because it needs React state (`useState`) and router hooks (`useRouter`)
2. Renders a Card with email and password inputs using shadcn components
3. On submit, calls `supabase.auth.signInWithPassword()` via the **browser** Supabase client
4. On success: `router.push("/dashboard")` then `router.refresh()` to force Server Components to re-run with the new session
5. On error: displays the error message from Supabase

```typescript
const handleLogin = async (e: React.SubmitEvent) => {
  e.preventDefault();
  setError("");
  setLoading(true);

  const { error: authError } = await supabase.auth.signInWithPassword({
    email,
    password,
  });

  if (authError) {
    setError(authError.message);
    setLoading(false);
    return;
  }

  router.push("/dashboard");
  router.refresh();
  setLoading(false);
};
```

**Key concept — Why `router.refresh()`?**

After `router.push("/dashboard")`, the browser navigates to the dashboard. But Server Components might have cached their output from before the login (when there was no session). `router.refresh()` forces all Server Components on the page to re-render with the new session data. Without it, the protected layout might still think the user is unauthenticated.

**Key concept — Browser client for login:**

We use the browser Supabase client (`src/lib/supabase/client.ts`) — not the server client — because the login form runs in the browser. The browser client uses the anon key and stores the session in cookies after successful authentication.

Also updated the proxy (`src/proxy.ts`) to match the new route structure: `/dashboard`, `/leads`, `/reminders`, `/login`, `/users`.

---

### Phase 2: App Shell — Protected Layout & Sidebar (Session 1 Carryover)

**Concepts covered:** Route groups, Server Component data fetching, defense-in-depth auth, role-based navigation, shadcn Sidebar component system, React Context for sidebar state.

**What we did:**

1. Created the route group `(protected)` with a layout that checks authentication and loads the user profile.
2. Installed shadcn sidebar and related components:

```bash
npx shadcn@latest add sidebar avatar dropdown-menu separator sheet skeleton tooltip
npm install lucide-react
```

3. Built three key pieces: the protected layout, the sidebar, and the user dropdown.

**Reference file:** `src/app/(protected)/layout.tsx` — Protected Layout

The layout is a **Server Component** that:

1. Gets the current user from Supabase (reads session from cookies)
2. Redirects to `/login` if no user found
3. Loads the user's Profile from Prisma (to get their role, name, etc.)
4. If the profile doesn't exist or `isActive` is false, signs them out and redirects to `/login`
5. Wraps children with `QueryProvider` (TanStack Query) and `SidebarProvider` (shadcn sidebar state)
6. Passes the user's role and profile to `AppSidebar`

```typescript
export default async function ProtectedLayout({ children }: { children: React.ReactNode }) {
  const supabase = await createSupabaseServerClient();
  const {
    data: { user },
  } = await supabase.auth.getUser();
  if (!user) redirect("/login");

  const profile = await prisma.profile.findUnique({ where: { id: user.id } });
  if (!profile || !profile.isActive) {
    await supabase.auth.signOut();
    redirect("/login");
  }

  return (
    <QueryProvider>
      <SidebarProvider>
        <AppSidebar role={profile.role} user={profile} />
        {children}
      </SidebarProvider>
    </QueryProvider>
  );
}
```

**Key concept — Defense in depth:**

The proxy already checks auth and redirects unauthenticated users. So why does the layout also check? Because defense in depth means layering security. If someone bypasses the proxy (a misconfigured route, a bug), the layout catches them. The proxy is the first wall; the layout is the second wall. In production, you always want multiple layers.

**Reference file:** `src/components/app-sidebar.tsx` — Sidebar Navigation

The sidebar is a **Client Component** (`"use client"`) that:

- Defines navigation items: Dashboard, Leads, Reminders (for all roles) and Users (admin only)
- Highlights the active link using `usePathname()`
- Shows the Administration section only when `role === "ADMIN"`
- Renders a user dropdown (`NavUser`) in the footer

```typescript
const mainSidebarItems = [
  { label: "Dashboard", href: "/dashboard", icon: LayoutDashboard },
  { label: "Leads", href: "/leads", icon: Users },
  { label: "Reminders", href: "/reminders", icon: Calendar },
];

const adminSidebarItems = [{ label: "Users", href: "/users", icon: User }];
```

**Reference file:** `src/components/app-sidebar-footer.tsx` — User Dropdown

The footer renders the logged-in user's avatar (first initial), name, and email with a dropdown menu for Account, Billing, Notifications, and Log out. The logout handler calls `supabase.auth.signOut()` and redirects to `/login`.

**Key concept — Route groups:**

The `(protected)` folder in the `app/` directory is a route group. The parentheses tell Next.js: "this folder is for organization only — don't include it in the URL." So `src/app/(protected)/dashboard/page.tsx` maps to `/dashboard`, not `/(protected)/dashboard`. This lets us share the protected layout across all CRM pages without affecting URLs.

Also created placeholder pages for:

- `src/app/(protected)/dashboard/page.tsx`
- `src/app/(protected)/leads/page.tsx`
- `src/app/(protected)/reminders/page.tsx`

---

### Phase 3: Lead & Activity Database Schema

**Concepts covered:** Prisma schema design, enums, relations (one-to-many), optional fields, migration workflow.

**What we did:**

Extended the Prisma schema with two new models and their enums.

**Reference file:** `prisma/schema.prisma`

**Lead model:**

```prisma
model Lead {
  id           String     @id @default(uuid())
  name         String
  phone        String
  email        String
  stage        LeadStage  @default(NEW)
  status       LeadStatus @default(OPEN)
  assignedToId String?
  assignedTo   Profile?   @relation(fields: [assignedToId], references: [id])
  activities   Activity[]
  createdAt    DateTime   @default(now())
  updatedAt    DateTime   @updatedAt
}
```

**Activity model:**

```prisma
model Activity {
  id        String       @id @default(uuid())
  leadId    String
  lead      Lead         @relation(fields: [leadId], references: [id])
  actorId   String
  actor     Profile      @relation(fields: [actorId], references: [id])
  content   String?
  type      ActivityType
  createdAt DateTime     @default(now())
}
```

**Enums:**

```prisma
enum LeadStatus { OPEN, WON, LOST }
enum LeadStage  { NEW, CONTACTED, QUALIFIED, NEGOTIATING }
enum ActivityType {
  LEAD_CREATED, NOTE, CALL_ATTEMPT, STATUS_CHANGE,
  STAGE_CHANGE, ASSIGNMENT_CHANGE, REMINDER_CREATED,
  ATTACHMENT_ADDED, AI_LEAD_BRIEF_GENERATED,
  AI_FOLLOWUP_DRAFT_GENERATED
}
```

Ran two migrations (leads table first, then activity table) and regenerated the Prisma client:

```bash
npx prisma migrate dev --name "add_leads_table" --create-only
npx prisma migrate deploy
npx prisma migrate dev --name "add_activity_table" --create-only
npx prisma migrate deploy
npx prisma generate
```

**Key concept — The Activity model as an append-only log:**

The Activity table is designed as an append-only event log. You write to it; you never edit or delete. Every activity answers four questions: WHO did it (`actorId`), WHAT happened (`type`), WHEN it happened (`createdAt`), and WHY/HOW (`content`). This pattern is used in production systems everywhere — audit logs, banking transactions, Git commits.

**Key concept — Relationships:**

- A `Profile` has many `Lead` records (one-to-many via `assignedToId`)
- A `Lead` has many `Activity` records (one-to-many via `leadId`)
- A `Profile` has many `Activity` records (one-to-many via `actorId` — who performed the action)

The `assignedToId` on Lead is optional (`String?`) because a lead might not be assigned to anyone yet.

---

### Phase 4: The authenticateUser() Helper

**Concepts covered:** DRY principle, custom error classes, optional role-based authorization, single point of change.

**What we did:**

Created a reusable authentication function that every route handler calls first.

**Reference file:** `src/utils/authenticateUser.ts`

```typescript
export class AuthenticationError extends Error {
  constructor(
    message: string,
    public statusCode: number,
  ) {
    super(message);
    this.name = "AuthenticationError";
  }
}

export async function authenticateUser(allowedRoles?: Role[]) {
  // Step 1: Get session from Supabase
  const supabase = await createSupabaseServerClient();
  const {
    data: { user },
  } = await supabase.auth.getUser();
  if (!user) throw new AuthenticationError("Unauthorized", 401);

  // Step 2: Fetch profile from Prisma
  const profile = await prisma.profile.findUnique({ where: { id: user.id } });
  if (!profile) throw new AuthenticationError("User not found", 404);
  if (!profile.isActive)
    throw new AuthenticationError("User is not active", 403);

  // Step 3: Optional role check
  if (allowedRoles && !allowedRoles.includes(profile.role)) {
    throw new AuthenticationError("Unauthorized", 403);
  }

  return profile;
}
```

**How it works:**

1. Reads the Supabase session from cookies → gets the user
2. Looks up the Profile in Prisma → gets role, isActive, etc.
3. Optionally checks if the user's role is in `allowedRoles`
4. Throws `AuthenticationError` with the right status code at each failure point
5. Returns the full Profile if everything passes

**Usage in route handlers:**

```typescript
// Any authenticated user
const profile = await authenticateUser();

// Only ADMIN or MANAGER
const profile = await authenticateUser([Role.ADMIN, Role.MANAGER]);
```

**Software engineering principle — DRY (Don't Repeat Yourself):**

Without this helper, every route handler would repeat the same 15 lines of auth code: create Supabase client, get user, check null, find profile, check null, check active, check role. That's 15 lines × every route = hundreds of duplicated lines. If you ever change how auth works (say, adding a "suspended" status), you'd need to update every single route. With the helper, you change one function.

**Software engineering principle — Custom error classes:**

`AuthenticationError` carries a `statusCode`. In the route handler's catch block, we check `if (error instanceof AuthenticationError)` and return the right HTTP status. This is cleaner than returning error objects everywhere — throw the error, let the catch block handle the response formatting.

---

### Phase 5: Zod Validation Schemas

**Concepts covered:** Schema-based validation, type inference from schemas, coercion for query parameters, single source of truth for types.

**What we did:**

Defined Zod schemas for lead-related operations.

**Reference file:** `src/services/lead/schema.ts`

```typescript
import { z } from "zod";

export const listLeadsQuerySchema = z.object({
  page: z.coerce.number().min(1).default(1),
  pageSize: z.coerce.number().min(1).max(100).default(10),
});

export type ListLeadsParams = z.infer<typeof listLeadsQuerySchema>;

export const createLeadSchema = z.object({
  name: z.string().min(1),
  phone: z.string().min(8).max(15),
  email: z.email(),
  note: z.string().optional(),
});

export type CreateLeadRequest = z.infer<typeof createLeadSchema>;
```

**Key concept — `z.coerce.number()`:**

Query parameters from URLs always arrive as strings (`"1"`, `"10"`). `z.coerce.number()` automatically converts `"1"` to `1` before validating. Without coercion, `z.number().parse("1")` would fail because `"1"` is a string, not a number.

**Key concept — `z.infer<typeof schema>`:**

This extracts a TypeScript type from a Zod schema. Instead of defining both a Zod schema AND a TypeScript interface (and keeping them in sync), you define the schema once and derive the type:

```typescript
export type CreateLeadRequest = z.infer<typeof createLeadSchema>;
// Equivalent to: { name: string; phone: string; email: string; note?: string }
```

One source of truth. If you add a field to the schema, the type updates automatically.

**Key concept — `.default()` for pagination:**

`z.coerce.number().min(1).default(1)` means: if the caller doesn't provide `page`, use `1`. This makes pagination parameters optional in the API call — `GET /api/leads` works without any query params.

---

### Phase 6: GET /api/leads — Role-Scoped Read

**Concepts covered:** Route handlers in Next.js, the authenticate → validate → execute → respond pattern, role-based database scoping, the three-tier service layer.

**What we did:**

Built the read endpoint for leads using the three-tier service layer.

**Reference file:** `src/app/api/leads/route.ts` (GET handler)

The route handler follows a strict pattern:

```
1. Authenticate → authenticateUser()
2. Validate    → listLeadsQuerySchema.parse(params)
3. Execute     → listLeads(profile, params)
4. Respond     → NextResponse.json({ success, data })
```

**Reference file:** `src/services/lead/service.ts` — Business logic

```typescript
export async function listLeads(profile: Profile, params: ListLeadsParams) {
  const where: Prisma.LeadWhereInput = {};
  if (profile.role === Role.AGENT) {
    where.assignedToId = profile.id;
  }
  return dbListLeads(where, params);
}
```

This is the most important code in the session. One `if` statement implements the entire authorization model:

- **AGENT**: sees only leads assigned to them (`where.assignedToId = profile.id`)
- **MANAGER / ADMIN**: sees all leads (empty `where` = no filter)

**Reference file:** `src/services/lead/db.ts` — Database access

```typescript
export async function dbListLeads(
  where: Prisma.LeadWhereInput,
  params: ListLeadsParams,
) {
  return prisma.lead.findMany({
    where,
    take: params.pageSize,
    skip: (params.page - 1) * params.pageSize,
    orderBy: { createdAt: "desc" },
  });
}
```

The db layer is purely a data access function — it takes a `where` clause and pagination params, queries Prisma, and returns results. It doesn't know about roles, auth, or business rules. That's the service layer's job.

**Software engineering principle — Separation of concerns in layers:**

```
Route Handler (HTTP concerns)
  → authenticateUser()     → who is calling?
  → schema.parse()         → is the input valid?
  → service.listLeads()    → what data should they see?
    → db.dbListLeads()     → execute the query
```

Each layer has one job. If you need to change how pagination works, you touch `db.ts`. If you need to change who sees what, you touch `service.ts`. If you need to change the HTTP response format, you touch the route handler. No layer bleeds into another.

**Error handling:**

All errors are caught in the route handler's `try/catch`:

- `AuthenticationError` → returns the appropriate status code (401, 403, 404)
- `ZodError` → returns 400 with field-level error details (`error.flatten().fieldErrors`)
- Any other error → returns 500 ("Internal server error")

---

### Phase 7: POST /api/leads — Atomic Create with $transaction

**Concepts covered:** Atomic database transactions, the `$transaction` pattern, creating related records together, audit trail.

**What we did:**

Built the create endpoint that writes both a Lead and a LEAD_CREATED Activity in one atomic transaction.

**Reference file:** `src/app/api/leads/route.ts` (POST handler)

The POST handler:

1. Authenticates the user and requires ADMIN or MANAGER role
2. Parses and validates the request body with `createLeadSchema`
3. Calls `createLead(profile, data)` which handles the transaction
4. Returns the created lead

```typescript
const profile = await authenticateUser([Role.ADMIN, Role.MANAGER]);
const body = await request.json();
const data = createLeadSchema.parse(body);
const lead = await createLead(profile, data);
```

**Reference file:** `src/services/lead/db.ts` (dbCreateLead)

```typescript
export async function dbCreateLead(profile: Profile, data: CreateLeadRequest) {
  return await prisma.$transaction(async (tx) => {
    const lead = await tx.lead.create({
      data: {
        name: data.name,
        phone: data.phone,
        email: data.email,
      },
    });

    await tx.activity.create({
      data: {
        leadId: lead.id,
        actorId: profile.id,
        content: data.note,
        type: ActivityType.LEAD_CREATED,
      },
    });

    return lead;
  });
}
```

**Key concept — `prisma.$transaction()`:**

A transaction ensures that multiple database operations either ALL succeed or ALL fail together. In this case:

- If the Lead is created but the Activity fails → the Lead is rolled back. Neither exists.
- If both succeed → both are committed.

Without a transaction, you could end up with a Lead that has no Activity (inconsistent data). The transaction guarantees atomicity.

**Key concept — The `tx` client:**

Inside the transaction callback, you use `tx` (the transaction client) instead of `prisma` for all queries. Using `prisma` directly would bypass the transaction and execute on a separate connection — defeating the purpose. Always use the `tx` parameter for operations that need to be atomic.

**Key concept — Role-based authorization on write:**

```typescript
const profile = await authenticateUser([Role.ADMIN, Role.MANAGER]);
```

Only ADMIN and MANAGER can create leads. If an AGENT calls this endpoint, `authenticateUser` throws a 403 before any database operation runs. Authorization is enforced at the gate, not inside the business logic.

---

### Phase 8: TanStack Query Setup + Hooks

**Concepts covered:** QueryClient configuration, QueryClientProvider, Axios instance, custom hooks wrapping `useQuery` and `useMutation`, cache invalidation.

**What we did:**

1. Installed TanStack Query and Axios:

```bash
npm install @tanstack/react-query axios
```

2. Created the query provider.

**Reference file:** `src/providers/query-provider.tsx`

```typescript
"use client";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import React, { useState } from "react";

export function QueryProvider({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(() => new QueryClient());
  return <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>;
}
```

The `QueryProvider` wraps the entire protected area (added in the protected layout). `useState(() => new QueryClient())` ensures one QueryClient per session — not per render.

3. Created the Axios API instance.

**Reference file:** `src/lib/api.ts`

```typescript
import axios from "axios";

export const api = axios.create({
  baseURL: "/api",
  withCredentials: true,
});
```

`withCredentials: true` ensures cookies (including the Supabase session) are sent with every request. `baseURL: "/api"` means all calls are relative — `api.get("/leads")` calls `/api/leads`.

4. Created the TanStack Query hooks.

**Reference file:** `src/lib/tanstack/useLeads.ts`

```typescript
export function useGetLeads(params: ListLeadsParams) {
  return useQuery({
    queryKey: ["leads", params],
    queryFn: async (): Promise<Lead[]> => {
      const { data } = await api.get("/leads", { params });
      return data.data;
    },
  });
}

export function useCreateLead(lead: CreateLeadRequest) {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: async (): Promise<Lead> => {
      const { data } = await api.post("/leads", lead);
      return data.data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["leads"] });
    },
  });
}
```

**Key concept — Query keys:**

`queryKey: ["leads", params]` is the cache address for this data. If `params` is `{ page: 1, pageSize: 10 }`, TanStack Query stores the result under `["leads", { page: 1, pageSize: 10 }]`. A different params object = a different cache entry. When you revisit the same params, the data is served instantly from cache.

**Key concept — Cache invalidation cycle:**

This is the core mental model for TanStack Query mutations:

```
1. User submits create form
2. useMutation fires POST /api/leads
3. Server creates Lead + Activity (transaction)
4. onSuccess callback runs:
   queryClient.invalidateQueries({ queryKey: ["leads"] })
5. TanStack Query marks ALL cache entries starting with ["leads"] as stale
6. Any mounted useQuery with key ["leads", ...] automatically refetches
7. The leads table re-renders with the new data
```

No manual state updates. No `setLeads([...leads, newLead])`. The cache handles it. Create a lead → invalidate → the table updates itself.

---

## What We Did NOT Build (Your Assignment)

We completed all the data logic and wiring, but did not build the frontend UI that uses these hooks. The pages are currently placeholders. This is your assignment before Session 3.

---

## Supplementary Videos

### Review What We Covered

These videos reinforce the concepts from Session 2. Watch them if anything felt unclear.

**TanStack Query (React Query) — Full Introduction**
[TanStack Query in Next.js](https://www.youtube.com/watch?v=Ji9OvOtAWBk) by OrcDev (~11 min)
Covers `useQuery`, `useMutation`, query keys, and cache invalidation — exactly the patterns we used for `useGetLeads` and `useCreateLead`.

**Zod — Schema Validation in TypeScript**
[Learn Zod In 30 Minutes](https://www.youtube.com/watch?v=L6BE-U3oy80) by Web Dev Simplified (~30 min)
Covers schema definitions, `.parse()` vs `.safeParse()`, and `z.infer<typeof schema>` — the foundations of our validation layer.

**Next.js Route Handlers**
[Mastering Next.js Route Handlers: A Comprehensive Guide](https://www.youtube.com/watch?v=xThOII9T4i4) by Code Ryan (~44 min)
Covers the `GET`/`POST` export pattern in `route.ts` files — how we built `/api/leads`.

**Prisma Transactions**
[Prisma Docs — Transactions](https://www.prisma.io/docs/orm/prisma-client/queries/transactions)
Read the interactive transaction section. This is the `$transaction(async (tx) => {...})` pattern we used for atomic lead + activity creation.

---

## Assignment: Build the Leads UI Before Session 3

You have a longer gap between sessions this time. Use it to build the frontend that consumes the data layer we set up together. By the start of Session 3, your app should feel like a real CRM — you can log in, see leads, create new ones, and view individual lead details.

Everything you need already exists in the repo: the API routes, the validation schemas, the TanStack Query hooks. Your job is to build the UI that calls them.

### Task 1: Leads Table Page (estimated: 1.5-2 hours)

Build out `src/app/(protected)/leads/page.tsx` so it displays a table of leads.

**Expected outcome:** When you navigate to `/leads`, you see a table showing all leads with columns for name, email, phone, stage, status, and assigned agent. The table should have working pagination (Previous / Next buttons).

**What to use:**

- `useGetLeads()` from `src/lib/tanstack/useLeads.ts` to fetch the data
- shadcn Table component (`npx shadcn@latest add table`)
- shadcn Badge component (`npx shadcn@latest add badge`) for status/stage pills
- Loading state: show a loading indicator while data fetches
- Error state: show an error message if the fetch fails

**Hints:**

- The page must be a Client Component (`"use client"`) because it uses hooks
- The hook accepts `{ page, pageSize }` — use React state to track the current page
- Increment/decrement page on button clicks
- The hook returns `{ data, isLoading, isError }` — handle all three states
- For status/stage badges, use different colors (e.g., green for WON, red for LOST, blue for OPEN)

### Task 2: Create Lead Dialog (estimated: 1-1.5 hours)

Add a "Create Lead" button on the leads page that opens a dialog/modal form.

**Expected outcome:** Clicking "Create Lead" opens a dialog with fields for name, phone, email, and an optional note. Submitting the form creates the lead and the table updates automatically (via cache invalidation).

**What to use:**

- `useCreateLead()` from `src/lib/tanstack/useLeads.ts`
- shadcn Dialog component (`npx shadcn@latest add dialog`)
- shadcn form components (Button, Input, Label — already installed)
- Close the dialog on success
- Show validation errors if the server returns them

**Hints:**

- The mutation hook already has `onSuccess` → `invalidateQueries(["leads"])` built in. When you submit, the table will automatically refetch.
- Use controlled form inputs (`useState` for each field)
- Disable the submit button while the mutation is pending (`mutation.isPending`)

### Task 3: Lead Detail Page (estimated: 1.5-2 hours)

Build a dynamic route at `src/app/(protected)/leads/[id]/page.tsx` that shows a single lead's details.

**Expected outcome:** Clicking a lead's name in the table navigates to `/leads/[id]`. The detail page shows the lead's name, phone, email, stage, status, and assigned agent. It should have a tabbed layout (Overview, Timeline, Reminders, Files) — only Overview needs to work for now.

**What to build:**

- A new API route: `GET /api/leads/[id]` — returns a single lead by ID (follow the same authenticate → validate → execute → respond pattern)
- A new TanStack Query hook: `useGetLead(id)` with queryKey `["lead", id]`
- A new service function: `getLead(profile, id)` that fetches the lead (with role scoping — agents can only view their own leads)
- The page component that calls the hook and renders the data

**Hints:**

- Create the route handler at `src/app/api/leads/[id]/route.ts`
- Use `params.id` in the route handler and page component to get the lead ID
- For tabs, use shadcn Tabs component (`npx shadcn@latest add tabs`)
- The timeline tab will be built in Session 3 — just render a placeholder for now

### Task 4: Lead Editing (estimated: 1.5-2 hours)

Build a PATCH endpoint and wire up inline editing or an edit form on the lead detail page.

**Expected outcome:** You can change a lead's status, stage, or assignment. Each change automatically creates an Activity record with `{ from, to }` metadata so the timeline knows what changed.

**What to build:**

- A new API route: `PATCH /api/leads/[id]` that updates the lead
- A Zod schema for the edit payload (partial — all fields optional):

```typescript
export const editLeadSchema = z.object({
  name: z.string().min(1).optional(),
  phone: z.string().min(8).max(15).optional(),
  email: z.email().optional(),
  stage: z.nativeEnum(LeadStage).optional(),
  status: z.nativeEnum(LeadStatus).optional(),
  assignedToId: z.string().uuid().optional(),
});
```

- In the db layer: use `$transaction` to update the Lead AND create Activity rows for each changed field, including content like `"Status changed"` and storing `{ from: "OPEN", to: "WON" }` metadata
- A TanStack Query mutation hook: `useEditLead(id)`
- Edit UI on the detail page (dropdowns for status/stage, or an edit form)

**Hints:**

- Fetch the current lead first, then compare old values with new values to determine which fields changed
- Only create Activity records for fields that actually changed
- Use ActivityType values: `STATUS_CHANGE`, `STAGE_CHANGE`, `ASSIGNMENT_CHANGE` for the respective fields
- Invalidate both `["leads"]` (for the table) and `["lead", id]` (for the detail page) on success

### Task 5: Watch Pre-Session 3 Videos (estimated: 20 min)

**Supabase Storage — Overview**
[Save IMAGES With Supabase Storage and Next.js](https://www.youtube.com/watch?v=87JAdYPC2n0) by Cole Blender (~16 min)
Covers buckets, private/public access, and signed URLs. Light watch — just understand the mental model.

**Service Layer Architecture — Why Separate Concerns?**
Search for a 10-15 min video on "service layer pattern backend" or "separation of concerns backend architecture." You've already been doing this (schema → db → service → route handler). The video will formalize the pattern before Session 3, where we take it further.

---

## Checkpoint Before Session 3

Before Session 3, verify all of the following:

- [ ] Login page works at `/login`
- [ ] Sidebar shows role-appropriate navigation (ADMIN sees Administration section)
- [ ] Leads table displays leads with name, email, phone, stage, status columns
- [ ] Pagination works (Previous/Next buttons change the page)
- [ ] "Create Lead" button opens a dialog, form submits successfully
- [ ] New lead appears in the table immediately after creation (cache invalidation)
- [ ] Log in as Agent → see only leads assigned to that agent
- [ ] Log in as Manager/Admin → see all leads
- [ ] Lead detail page at `/leads/[id]` shows lead information
- [ ] Can edit a lead's status/stage → Activity row created with `{ from, to }`
- [ ] Code committed and pushed to GitHub

---

## Resources

- [TanStack Query v5 Docs](https://tanstack.com/query/v5) — `useQuery`, `useMutation`, query key management, cache invalidation
- [Zod Docs](https://zod.dev/) — Schema types, coercion, refinements, `.safeParse()`, `z.infer`
- [Next.js Route Handlers](https://nextjs.org/docs/app/building-your-application/routing/route-handlers) — Request/response handling in App Router
- [Prisma Client API](https://www.prisma.io/docs/orm/prisma-client) — `findMany`, `create`, `update`, `$transaction`
- [shadcn/ui Components](https://ui.shadcn.com/) — Table, Dialog, Badge, Tabs, and all components you'll need for the assignment
- [Lucide Icons](https://lucide.dev/) — Icon library used in the sidebar

---

_Session 2 Complete. Start your assignment early — you have more to build this time! See you in Session 3._
