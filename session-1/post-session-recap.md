# Session 1: Post-Session Recap & Tutorial

**Foundation — Project Setup, Database, Auth Infrastructure & Route Protection**

---

## What We Built

In Session 1, we went from zero to a fully wired Next.js application with a real database, authentication infrastructure, and route protection. We followed the **Just-In-Time (JIT)** philosophy throughout — installing tools only when we had a concrete need for them.

By the end of the session, the `whispyr-academy-crm` repo had:

- A running Next.js 16 project with TypeScript and Tailwind CSS
- Prisma 7 connected to a Supabase Postgres database
- A `Profile` model with a `Role` enum (ADMIN, MANAGER, AGENT)
- Three Supabase client files (server, client, admin)
- A seeded admin user bridging Supabase Auth and our Profile table
- A proxy that protects `/crm/`\* routes and redirects unauthenticated users to `/login`

---

## Step-by-Step Walkthrough

If you missed the session or want to review, follow these steps in order. Each step references the exact files in the `[whispyr-academy-crm](https://github.com/heshamelmahdi/whispyr-academy-crm)` repo so you can see the final state of the code.

### Phase 1: Project Scaffolding (Slides + Live Coding)

**Concepts covered:** Next.js App Router architecture, file-based routing, layouts vs pages, Server Components vs Client Components.

**What we did:**

1. Created a new Next.js 16 project:

```bash
npx create-next-app@latest whispyr-academy-crm \
  --typescript \
  --tailwind \
  --app \
  --src-dir \
  --import-alias "@/*" \
```

1. Initialized git and made the initial commit:

```bash
cd whispyr-academy-crm
git add .
git commit -m "initial commit"
```

1. Simplified the default boilerplate. We stripped out the default Next.js landing page and replaced it with a minimal home page.

**Reference files in the repo:**

- `src/app/layout.tsx` — Root layout with Roboto font configured via `next/font/google`
- `src/app/page.tsx` — Minimal home page (just renders `<h1>Home</h1>`)
- `src/app/globals.css` — Tailwind CSS imports and CSS variables
- `next.config.ts` — Next.js config with React Compiler enabled
- `tsconfig.json` — TypeScript configuration with `@/`\* path alias

**Key concept — App Router file conventions:**

```
src/app/
├── layout.tsx    → Root layout (wraps ALL pages — the "picture frame")
├── page.tsx      → Home page at /
├── login/
│   └── page.tsx  → Login page at /login
└── (protected)/
    └── crm/
        └── page.tsx  → Dashboard at /crm
```

Every `page.tsx` becomes a route automatically. Folders create URL segments. `layout.tsx` wraps all child routes and never re-renders when navigating between pages.

**Key concept — Server Components vs Client Components:**

By default, everything in App Router is a Server Component. You add `"use client"` only when you need browser APIs, state, or event handlers. Server Components can use `async/await` and call databases directly.

---

### Phase 2: Supabase Project Setup (Slides + Dashboard Walkthrough)

**Concepts covered:** What Supabase is, managed Postgres, connection strings (pooled vs direct), environment variables.

**What we did:**

1. Created a new Supabase project at [supabase.com](https://supabase.com)
2. Collected the required credentials from the Supabase dashboard:

- Project URL
- Anon (publishable) key
- Service role key
- Database connection strings (pooled + direct)

3. Created a `.env` file at the project root (never committed to git):

```env
# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_PUBLISHABLE_DEFAULT_KEY=eyJ...
SUPABASE_SERVICE_ROLE_KEY=eyJ...

# Database
DATABASE_URL=postgresql://postgres.xxx:password@aws-0-region.pooler.supabase.com:6543/postgres
DIRECT_URL=postgresql://postgres.xxx:password@aws-0-region.pooler.supabase.com:5432/postgres
```

**Key concept — Environment variable security:**

| Variable                                       | Exposure         | Why?                                                                      |
| ---------------------------------------------- | ---------------- | ------------------------------------------------------------------------- |
| `NEXT_PUBLIC_SUPABASE_URL`                     | Public (browser) | Just the project hostname                                                 |
| `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_DEFAULT_KEY` | Public (browser) | Limited permissions, safe to expose                                       |
| `SUPABASE_SERVICE_ROLE_KEY`                    | Server-only      | Bypasses ALL security — if leaked, total breach                           |
| `DATABASE_URL`                                 | Server-only      | Postgres credentials via PgBouncer (port 6543)                            |
| `DIRECT_URL`                                   | Server-only      | Direct Postgres connection (port 5432), used by Prisma CLI for migrations |

**Rule:** Anything prefixed with `NEXT_PUBLIC_` gets bundled into client JavaScript. Anyone can see it. Everything else stays server-only.

**Software engineering principle — Least Privilege:** Each tool gets the minimum credentials it needs. The browser gets only the anon key. Prisma CLI gets the direct URL. The app runtime gets the pooled URL.

---

### Phase 3: Prisma 7 Setup + Migration (Slides + Live Coding)

**Concepts covered:** What an ORM does, Prisma 7's new architecture (separate schema/config/generate), the Schema → Migrate → Generate pipeline, connection pooling, singleton pattern.

**What we did:**

1. Installed Prisma and its dependencies:

```bash
npm install prisma @prisma/client @prisma/adapter-pg pg
npm install -D tsx dotenv @types/pg
```

1. Created the Prisma schema with our first model.

**Reference file:** `prisma/schema.prisma`

```prisma
generator client {
  provider   = "prisma-client-js"
  output     = "../src/generated/prisma"
}

datasource db {
  provider  = "postgresql"
}

enum Role {
  ADMIN
  MANAGER
  AGENT
}

model Profile {
  id        String   @id @default(uuid())
  email     String   @unique
  name      String
  role      Role
  avatarUrl String?
  isActive  Boolean  @default(true)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

1. Created the Prisma config file.

**Reference file:** `prisma.config.ts` — This is new in Prisma 7. The CLI config is now a separate TypeScript file.

1. Ran the three-step migration flow:

```bash
# Step 1: Generate migration SQL (doesn't run it yet)
npx prisma migrate dev --name "add_profile_table" --create-only

# Step 2: Run the migration against the database
npx prisma migrate deploy

# Step 3: Generate the TypeScript client
npx prisma generate
```

1. Created the Prisma client singleton.

**Reference file:** `src/lib/prisma.ts`

This file creates ONE PrismaClient instance and reuses it. In development, the `globalForPrisma` pattern prevents creating a new client on every hot reload (which would leak database connections).

1. Added `src/generated/prisma` to `.gitignore` — generated code should never be committed.

**Key concept — The three-step workflow:**

In Prisma 7, migration and generation are separate explicit steps. Every time you change the schema:

1. `migrate dev --create-only` → generates the SQL
2. `migrate deploy` → runs the SQL against Postgres
3. `generate` → creates the TypeScript client

If you skip step 3, your editor won't know about new fields or models.

**Key concept — Connection pooling:**

`DATABASE_URL` (port 6543) goes through PgBouncer, which multiplexes many requests to fewer database connections. Your app might make 1000 requests/minute, but Postgres has a connection limit (~20-100). Without pooling, you'd exhaust it. `DIRECT_URL` (port 5432) is only for migrations, which need exclusive access.

**Key concept — Singleton pattern:**

```typescript
const globalForPrisma = globalThis as unknown as { prisma: PrismaClient }
export const prisma = globalForPrisma.prisma || new PrismaClient(...)
globalForPrisma.prisma = prisma
```

This ensures only ONE PrismaClient exists across your app. Without it, every hot reload in development creates a new client → new connection → eventual "too many connections" error.

---

### Phase 4: Supabase Auth Clients (Slides + Live Coding)

**Concepts covered:** Supabase Auth architecture, the auth bridge pattern (Supabase Auth ↔ Prisma Profile), three client types and when to use each.

**What we did:**

Created three Supabase client files, each for a different execution context:

**Reference file:** `src/lib/supabase/server.ts` — Server client. Used in Server Components and Route Handlers. Reads the user's session from cookies.

**Reference file:** `src/lib/supabase/client.ts` — Browser client. Used in Client Components (e.g., login forms). Uses the anon key only.

**Reference file:** `src/lib/supabase/admin.ts` — Admin client. Uses the service role key. Bypasses all security. Used only in server-side scripts (like seeding). NEVER import this in client code.

**When to use which client:**

| Context                          | Client      | Why                                       |
| -------------------------------- | ----------- | ----------------------------------------- |
| Server Component / Route Handler | `server.ts` | Reads session from cookies, respects auth |
| Client Component (login, forms)  | `client.ts` | Browser-safe, uses anon key               |
| Admin scripts, seeding           | `admin.ts`  | Service role key, bypasses security       |

**Key concept — The Auth Bridge:**

This is the single most important concept in the entire bootcamp:

```
┌─────────────────────────────┐
│       Supabase Auth         │
│   auth.users                │
│   id: "abc-123"             │
│   email, password hash,     │
│   session tokens            │
│   → Answers: WHO is this?   │
└──────────────┬──────────────┘
               │ same ID
┌──────────────▼──────────────┐
│       Prisma / Postgres     │
│   Profile                   │
│   id: "abc-123" ← linked   │
│   name, role, isActive      │
│   → Answers: WHAT can they  │
│     do?                     │
└─────────────────────────────┘
```

Supabase Auth manages passwords and sessions. Your Profile table manages roles and app-specific data. The link is the user ID — when you create a Supabase Auth user, you copy that same ID as the primary key of the Profile row.

**Software engineering principle — Separation of Concerns:** Auth is a commodity service. You could swap Supabase Auth for Auth0 or Firebase Auth without changing your Profile table. Each system has one job.

---

### Phase 5: Seed Script (Live Coding)

**Concepts covered:** Database seeding, the auth bridge in practice, admin client usage.

**What we did:**

Created a seed script that creates the first admin user in both Supabase Auth AND our Profile table.

**Reference file:** `prisma/seed/seed.ts`

The seed script:

1. Uses the **admin** Supabase client (service role key) to create a user in `auth.users`
2. Takes the returned user ID
3. Creates a Profile row in Postgres with that same ID, setting `role: "ADMIN"`

```bash
# Run the seed
npx prisma db seed
```

After running, verify in the Supabase dashboard:

- **Auth → Users**: You should see `admin@crm.com` (or whatever email was used)
- **Table Editor → Profile**: You should see a row with the same ID, role = ADMIN

**Key concept — The bridge in action:**

```typescript
// 1. Create auth user (Supabase manages password + sessions)
const { data: authData } = await supabaseAdmin.auth.admin.createUser({
  email: "admin@crm.com",
  password: "admin123",
  email_confirm: true,
});

// 2. Create profile (we manage role + app data)
await prisma.profile.create({
  data: {
    id: authData.user!.id, // ← Same ID! This is the bridge.
    email: "admin@crm.com",
    name: "System Admin",
    role: "ADMIN",
  },
});
```

---

### Phase 6: Proxy — Route Protection (Slides + Live Coding)

**Concepts covered:** Guard patterns, fail-secure design, Next.js 16 proxy (renamed from middleware), cookie-based sessions.

**What we did:**

Created a proxy that intercepts every request to protected routes and checks for a valid session.

**Reference file:** `src/proxy.ts`

The proxy:

1. Runs BEFORE every request matching `/crm/:path`\* or `/login`
2. Creates a Supabase client that reads the session from cookies
3. If no session and accessing `/crm/*` → redirect to `/login`
4. If session exists and on `/login` → redirect to `/crm`
5. Otherwise, let the request through

**Key concept — Next.js 16 breaking change:**

In Next.js 15, this was `middleware.ts` with an exported `middleware` function. In Next.js 16, it's renamed to `proxy.ts` with an exported `proxy` function. The concept is identical — it runs before every matched request — but the new name better reflects that it's a network-level proxy. It runs exclusively on the Node.js runtime (Edge runtime support was removed).

**Key concept — Fail-secure pattern:**

```typescript
if (!user && request.nextUrl.pathname.startsWith("/crm")) {
  return NextResponse.redirect(loginUrl); // No session? DENY.
}
```

The default is to deny access unless proven authorized. This is the opposite of "fail-open" (allow unless proven dangerous). If there's a bug in your auth code:

- Fail-secure: user is denied access (annoying but safe)
- Fail-open: user gets access they shouldn't have (security breach)

Always fail secure.

---

## What We Did NOT Cover (Moved to Session 2)

We ran out of time before reaching these planned items. They will be completed at the **start of Session 2**:

1. **shadcn/ui installation** — JIT component library setup
2. **Login page** — Email/password form using shadcn components
3. **App shell** — Protected layout with sidebar navigation and top bar
4. **Architecture preview** — Introduction to the service layer pattern

---

## Supplementary Videos

### Review What We Covered

These videos reinforce the concepts from Session 1. Watch them if anything felt unclear.

**Next.js App Router — How It Works**
[Learn Next.js 15 In 12 Minutes](https://www.youtube.com/watch?v=p-eASfbBXEk) by ByteGrad (~12 min)
Covers file-based routing, layouts, pages, Server vs Client Components. We used all of these during scaffolding.

**Comprehensive Prisma Tutorial**
[Prisma in Next.js](https://www.youtube.com/watch?v=QXxy8Uv1LnQ)
Covers what an ORM does, Prisma's schema file, and basic client usage. We set all of this up live.

---

## Before Session 2: What to Watch and Do

### 1. Watch These Videos (Required, ~30 min total)

**TanStack Query (React Query) — Introduction**
[TanStack Query in Next.js](https://www.youtube.com/watch?v=Ji9OvOtAWBk) by OrcDev (~11 min - required) + [TanStack Query - How to become a React Query God](https://www.youtube.com/watch?v=mPaCnwpFvZY) by Austin Davis (~29 min - optional)
Covers `useQuery`, `useMutation`, query keys, and cache invalidation. Session 2 uses TanStack Query for all frontend data fetching.

**Zod — Schema Validation in TypeScript**
[Learn Zod In 30 Minutes](https://www.youtube.com/watch?v=L6BE-U3oy80) by Web Dev Simplified (~30 min)
Covers schema definitions, `.parse()` vs `.safeParse()`, and `z.infer<typeof schema>`. Every API route we build will validate input with Zod.

### 2. Make Sure Your Project Runs

Before Session 2, verify:

- `npm run dev` starts without errors
- Your `.env` file has all required variables (Supabase URL, keys, database URLs)
- Prisma migration succeeded (check Supabase dashboard → Table Editor → you should see the Profile table)
- Seed ran successfully (check Supabase dashboard → Auth → Users → you should see the admin user)
- The admin user's Profile row exists in the Profile table with role = ADMIN

### 3. Review the Repo

Pull the latest version of `whispyr-academy-crm` and read through the files we created. Make sure you understand:

- How the three Supabase clients differ (`server.ts`, `client.ts`, `admin.ts`)
- How the seed script bridges Supabase Auth and Prisma Profile
- How the proxy protects routes

---

## Resources

- [Next.js App Router Docs](https://nextjs.org/docs/app) — File-based routing, layouts, Server vs Client Components
- [Prisma Docs](https://www.prisma.io/docs/guides/frameworks/nextjs) — Schema reference, migrations, client API
- [Supabase Auth Docs](https://supabase.com/docs/guides/auth/quickstarts/nextjs) — Session management, auth helpers
- [Tailwind CSS Docs](https://tailwindcss.com/docs) — Utility class reference

---

_Session 1 Complete. See you in Session 2!_
