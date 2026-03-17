# Session 2: Pre-Session Guide

**Leads CRUD + Validation**

---

## Before You Arrive

### Homework Check

You should have completed the following from Session 1:

- [ ] Next.js 16 project scaffolded and running (`npm run dev` works)
- [ ] Prisma 7 connected to Supabase Postgres with the full schema migrated (Lead, Activity, Reminder, Notification, Attachment + enums)
- [ ] Three Supabase client files created (server, client, admin)
- [ ] Seed script working — 7 total users (1 Admin, 2 Managers, 4 Agents) seeded via admin client
- [ ] `src/proxy.ts` protecting `/crm/*` routes
- [ ] `npx prisma generate` runs without errors

**Note:** We did NOT build the login page, app shell, or install shadcn/ui in Session 1. That's intentional — we'll do those at the start of Session 2 as a warm-up. Don't install them ahead of time.

### Videos to Watch

These are essential for Session 2 — don't skip them.

1. **TanStack Query — Introduction** (~20 min) — Understand `useQuery`, `useMutation`, query keys, and cache invalidation. This is how we'll fetch and update data on the frontend.
2. **Zod — Schema Validation** (~10 min) — Understand how to define schemas, `.parse()` vs `.safeParse()`, and `z.infer` for type generation.

> See `videos.md` in the session folder for specific video links.

---

## What We'll Build in Session 2

This is a longer session (~3h 45min) because we start by catching up on a few items from Session 1, then build the first real feature.

**First ~45 minutes (catching up):**
- Install **shadcn/ui** — the component library (we install it when we need our first UI component)
- Build the **login page** — email/password form connected to Supabase Auth
- Build the **app shell** — sidebar with role-based navigation, top bar, protected layout

**Next ~1h 15min (backend — API first):**
- Build a reusable **auth helper** that every route handler calls
- Define **Zod validation schemas** for lead data
- Build **GET /api/leads** — returns leads scoped by role (agents see only theirs, managers see all)
- Build **POST /api/leads** — creates a lead + activity record atomically using `$transaction`
- Test both routes with curl/Postman before touching any frontend code

**Final ~1h 30min (frontend — wiring up the UI):**
- Set up **TanStack Query** with DevTools and organize the data access layer
- Build the **leads table** — filters, pagination, role-scoped data
- Build the **create lead form** — submit → cache invalidates → table updates automatically

The pattern you learn here (authenticate → validate → execute → respond) is the pattern we'll use for every API route going forward.

---

## What to Expect

- **Duration:** ~3 hours 45 minutes (carryover + new content, with a 15-min break after the backend section)
- **Format:** We alternate between short concept slides and live coding. No long lecture blocks — you'll be coding within the first 5 minutes.
- **Approach:** We build and test the API independently first, then wire up the UI. This is how professional teams work — the backend doesn't depend on any frontend to be validated.
- **Heads up:** We'll write our first Zod schemas and route handlers. These patterns repeat in every future session, so pay close attention to the structure.
