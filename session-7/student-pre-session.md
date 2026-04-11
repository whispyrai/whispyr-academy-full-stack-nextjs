# Session 7: Pre-Session Guide

**Manager Dashboard + File Attachments**

---

## Before You Arrive

### Session 6 Homework That Must Already Work

You should arrive with these Session 6 pieces complete:

- [ ] User management works end-to-end: create a user, receive a magic link email, log in as that user
- [ ] Changing a user's role updates the badge in the users table
- [ ] Deactivating a user flips `isActive` to `false` (soft delete — the row is still there)
- [ ] CSV import: upload a file, see the validation report with valid/invalid counts, import the valid rows
- [ ] Imported leads appear in `/crm/leads` with `LEAD_CREATED` activity entries
- [ ] CSV export (homework): download filtered leads as a CSV
- [ ] Admin layout with tabs (homework): Users / Import / Export tabs on `/admin`

### What We Will Finish Together At The Start Of Session 7

The first 15 minutes of Session 7 are a homework check and a mini-demo of the admin features. We will **not** debug the admin or CSV code live. If you're behind, we'll help during the break. Session 7 does **not** depend on CSV export, the admin layout, or the isActive middleware check.

---

## Account Setup

You should already have your Supabase project from Session 1. For Session 7 we need to add a Storage bucket. You can either:

**Option A: Create it before class (recommended, 2 minutes)**

- [ ] Open your Supabase dashboard → Storage → **New bucket**
- [ ] Name: `lead-attachments`
- [ ] Toggle **Public bucket: OFF** (this is important — it must be private)
- [ ] File size limit: `10 MB`
- [ ] Allowed MIME types: leave blank (we'll validate on the server)
- [ ] Click **Save**

**Option B: Create it live in class**

We will create the bucket together in Phase 4. Either is fine.

You do **not** need to install any new packages before class. We will install `recharts` live the moment the dashboard needs its first chart.

Verify your `.env.local` still has these from earlier sessions (no new env vars for Session 7):

- `NEXT_PUBLIC_SUPABASE_URL`
- `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_DEFAULT_KEY`
- `SUPABASE_SERVICE_ROLE_KEY` — this is the one that will upload files for us

---

## Videos To Watch

Watch these before Session 7:

1. **Recharts — Declarative Charts in React**
   - Focus on `ResponsiveContainer`, `BarChart`, `PieChart`, and the composable children (`XAxis`, `YAxis`, `Bar`, `Cell`, `Tooltip`)
   - Understand that data is an array of `{ name, value }` objects
   - Notice that every chart needs a parent container with an explicit height — charts die silently inside zero-height divs

2. **Supabase Storage — Private Buckets + Signed URLs**
   - Focus on the difference between a public bucket and a private bucket
   - Understand what a signed URL is: a short-lived, cryptographically signed link that proves you have permission to access a private file
   - `supabase.storage.from(bucket).upload(path, file)` and `createSignedUrl(path, expirySeconds)`

See `videos.md` for suggested links.

---

## What We Will Build Live

This session adds two production features to the CRM:

1. **Manager Dashboard** — KPI cards (total / open / won / lost), a bar chart of leads by stage, a pie chart of leads by status, and a top-agents leaderboard. All counts come from Prisma `groupBy` queries running in parallel.
2. **File Attachments** — a Files tab on every lead detail page. Upload a PDF or image (up to 10MB), see it listed with uploader and timestamp, click Download to open the file via a signed URL, and see `ATTACHMENT_ADDED` in the activity timeline.

We are **not** building the agent personal dashboard, the role-aware `/crm/leads` table, bulk reassignment, or the delete-attachment flow live. Those are homework — same service-layer pattern you have built seven times by now.

---

## What To Expect

- **Duration:** 3 hours
- **Pattern:** slides → code → slides → code
- **Most important concept:** aggregate at the database (`groupBy`, `_count`), never in JavaScript — the database is built for this
- **Second most important concept:** private bucket + signed URL is the production default; public buckets are for avatars, not contracts
- **Heads up:** This is the last live-coding session. Session 8 is deployment, seed data, and demos. Your CRM should feel nearly complete by the end of tonight.
