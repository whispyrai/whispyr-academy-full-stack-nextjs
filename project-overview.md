# CRM Pro — Project Overview

**Whispyr Academy | Cohort 1: Full-Stack Web Development (Next.js) + AI**

---

## What You're Building

You're building **CRM Pro** — a production-grade operations CRM (Customer Relationship Management) application powered by AI. This is the kind of tool that real sales and operations teams use every day to track leads, manage pipelines, set reminders, and make better decisions.

By the end of this bootcamp, you'll have a deployed, working application that you can show to any employer and say: "I built this."

---

## The Big Picture

Imagine you run a sales team. Your agents call potential customers (leads), track their progress through a pipeline, set reminders to follow up, and report results to their managers. The admin manages the team, imports new leads from spreadsheets, and makes sure everything runs smoothly.

That's what your CRM does. Specifically:

- **Leads** move through a pipeline: New → Contacted → Qualified → Negotiating → Won or Lost
- **Every action is recorded** in an activity timeline — notes, calls, status changes, assignments, everything
- **Reminders** are scheduled for the future and fire automatically when they're due
- **AI assists** agents by generating lead briefs and suggesting follow-up scripts after calls
- **Managers** see dashboards, reassign leads, and oversee the full pipeline
- **Admins** manage users, import/export leads via CSV, and configure the system

---

## Roles

The app has three user roles, each with different access levels:

**Agent** — The frontline user. Agents see only their own assigned leads. They log calls, add notes, set reminders, and use AI tools to work more effectively.

**Manager** — Sees everything. Managers view the full pipeline, reassign leads between agents, and monitor team performance through dashboards and reports.

**Admin** — Full system access. Admins do everything managers can, plus manage users (create accounts, change roles, deactivate), import/export leads via CSV, and configure system settings. Admins use email `admin@crm-pro.com` in local development/demo environments.

Access is enforced on the server. It's not just about hiding UI elements — the API itself rejects unauthorized requests.

---

## Tech Stack

Every tool in this stack earns its place. Here's what you'll use and why:

| Tool                           | What It Does                        | Why You're Learning It                                                                                                 |
| ------------------------------ | ----------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| **Next.js 16 (App Router)**    | Full-stack React framework          | The industry standard for production React apps. Server components, API routes, proxy function — all in one framework. |
| **TypeScript**                 | Type-safe JavaScript                | Catches bugs before they reach production. Every serious codebase uses it.                                             |
| **shadcn/ui + Tailwind CSS**   | UI components + styling             | Enterprise-quality components out of the box. No custom CSS headaches.                                                 |
| **Prisma 7**                   | Database ORM                        | Write database queries in TypeScript instead of raw SQL. Migrations, seeding, type safety.                             |
| **Supabase**                   | Postgres database + Auth + Storage  | Managed database, authentication, and file storage. You get a production backend without managing servers.             |
| **TanStack Query**             | Client-side data fetching + caching | The standard for fetching, caching, and updating server data in React apps. You'll use it everywhere.                  |
| **Zod**                        | Input validation                    | Validates every piece of data before it touches your database. Server-side, always.                                    |
| **Upstash QStash**             | Scheduled job delivery              | Delivers reminders at a specific future time. Teaches you how real scheduling works in serverless.                     |
| **Upstash Redis**              | In-memory cache                     | Prevents duplicate job execution (idempotency). Fast key-value lookups.                                                |
| **Resend**                     | Transactional email                 | Sends invite emails and reminder notifications. Clean API, real deliverability.                                        |
| **Vercel AI SDK + AI Gateway** | AI integration                      | Calls AI models with structured outputs. You'll use it to build features that feel like magic.                         |
| **Vercel**                     | Deployment                          | Your app goes live here. Push to GitHub, it deploys automatically.                                                     |

---

## Core Features (What You Must Build)

These are required. Your project does not pass without them.

### 1. Authentication & User Management

No public signups. The admin creates all user accounts manually. When the admin creates a user, the system sends them an invite email with a link to set their password. The sidebar and available pages adapt based on your role.

### 2. Lead Management

Leads are the heart of the CRM. You'll build a table view with server-side pagination, sorting, and filtering. Each lead has a detail page with tabs for Overview, Activity, Reminders, Files, and AI. Every change to a lead (status update, stage change, reassignment) is tracked automatically.

### 3. Activity Timeline

This is the canonical history of everything that happened to a lead. When a lead is created, that's an activity. When someone adds a note, logs a call, changes the status, uploads a file, or generates an AI brief — each one writes an activity row. The timeline is append-only: nothing is ever edited or deleted.

This is a real production pattern. You're learning event-driven thinking.

### 4. Reminders with QStash

Agents create reminders from a lead's detail page. When you create a reminder, the system schedules a future delivery via QStash. When the time comes, QStash calls your API, and your handler marks the reminder as fired and creates an in-app notification.

You'll learn how scheduled jobs work in serverless environments — and why you can't just use `setTimeout`.

### 5. In-App Notifications

When a reminder fires, a notification appears. The notification bell in the top bar shows an unread count. Clicking a notification takes you to the relevant lead. You can mark notifications as read individually or all at once.

### 6. File Attachments

Upload files to a lead from the Files tab. Files are stored in Supabase Storage with signed URLs (not publicly accessible). Each upload writes an activity entry.

### 7. AI Feature: Lead Brief Generator

On any lead's AI tab, you can click "Generate Brief." The system gathers the lead's data and recent activity history, sends it to an AI model, and gets back a structured brief: a summary, key facts, risks, recommended next actions, and questions to ask. You can save the brief to the timeline.

The AI doesn't invent information. Your prompt explicitly tells it to use only the provided data.

### 8. AI Feature: Call Follow-up Assistant

After logging a call attempt, you can ask the AI for follow-up suggestions. It generates a call script (opening line, questions, objection handlers), recommends a next step, and pre-fills a reminder form. You review everything before saving — the AI never writes to the database on its own.

### 9. Manager Views

Managers get a dashboard with KPI cards and charts (leads by stage, by status, overdue reminders). They can view all leads across all agents and reassign leads through an assignment center.

### 10. Admin: User Management

Admins can list all users, create new users (which triggers an invite email), edit roles, and deactivate accounts. Deactivation is soft — the account is flagged, not deleted.

### 11. CSV Export

Managers and admins can export the currently filtered leads table to a .csv file.

### 12. Seed Data & Deployment

Your app must include a seed script that creates demo data: an admin, managers, agents, 50+ leads across different stages and assignments, activities, and reminders. The app must be deployed to Vercel with a complete README.

---

## Extended Features (For Top Marks)

If you finish Core early, these will push your project further:

- **Kanban Board** — drag-and-drop leads between pipeline stages
- **Manager Daily Digest** — a scheduled AI-generated email summarizing the pipeline each morning
- **Global Search** — search leads by name or phone from the top bar
- **Bulk Operations** — change status or tags for multiple leads at once
- **Agent Metrics** — personal performance dashboard
- **Manager Reports** — org-wide metrics with leaderboards and pipeline velocity
- **Admin Settings** — configure lead sources and tags
- **Avatar Uploads** — profile photos for users
- **CSV Import** - this allows the admin to bulk-import leads

---

## How the Bootcamp Works

### Format

8 live sessions over 4 weeks (2 per week). Each session is 3 hours: 1.5 hours → 15-minute break → 1.5 hours.

### Learning Model

Not everything is taught in the live session. Before most sessions, you'll watch assigned YouTube videos covering the tools you'll use next. The live session then focuses on applying those tools in the context of your CRM — connecting concepts, debugging issues, and building features together. Sessions 2-8 start with a short Q&A on the pre-session videos.

### Homework

After each session, you have implementation tasks due before the next session. You submit by pushing to your GitHub repo. Your code is reviewed between sessions — you'll get written feedback so you can course-correct early.

### Solo Project

This is individual work. You build your own CRM from start to finish. Everyone uses the same spec, but your implementation, code quality, and design decisions are your own.

---

## Architecture Patterns You'll Learn

Throughout the bootcamp, you'll follow professional software engineering patterns used in production codebases:

### Service Layer Pattern

Business logic lives in `src/services/`. This keeps controllers/API routes clean and makes logic testable and reusable. For example, `leadService.ts` contains functions like `createLead()`, `reassignLead()`, and `updateLeadStatus()`. Components never call the database directly — they go through services.

### Authentication Helper

Authentication logic is centralized in `src/lib/auth.ts` with an `authenticateUser()` helper. Every protected API route uses this to verify the user, check their role, and enforce access control. This prevents auth bypasses and keeps the pattern consistent across your entire backend.

### Frontend Data Access Layer

All client-side data fetching goes through custom hooks in `src/tanstack/`. Instead of spreading `fetch()` calls throughout components, you create query hooks like `useLeads()` and mutation hooks like `useCreateLead()`. This provides a single source of truth for API contracts, error handling, and caching. **Components never define fetch logic inline.**

### Database Migrations

All schema changes go through Prisma migrations. The workflow is:

```bash
# Create a migration (generates a .sql file, doesn't run it yet)
npx prisma migrate dev --name "add_lead_tags_field" --create-only

# Review the generated SQL, then deploy it
npx prisma migrate deploy
```

This pattern ensures your production database and local development stay in sync, and gives you a clear audit trail of every schema change.

### Software Engineering Best Practices

This project teaches you how real software teams work:

- **Type safety first:** TypeScript catches bugs before deployment
- **Server-side validation:** Never trust client input. Every mutation uses Zod validation on the server
- **Immutable audit trails:** Activities are append-only — you can't edit or delete them. This is how real systems maintain compliance
- **Role-based access control:** Security is checked on every request, not just in the UI
- **Separation of concerns:** Services, components, hooks, and utilities all have clear responsibilities
- **Git discipline:** Clean commits with meaningful messages; code review before merging
- **Documentation:** Seed scripts, READMEs, and inline comments explain the "why" behind design decisions

---

## What Whispyr Provides

You don't need to set up everything from scratch. Whispyr gives you:

- **Resend API key** — for sending emails (invites, reminders)
- **AI Gateway credentials** — for AI model calls via Vercel AI SDK
- **QStash signature verification utility** — a helper function you copy into your project
- **Wireframes** — UI reference for all Core pages
- **Partial reference code** — a working example of the auth + leads CRUD setup
- **Curated video playlist** — the videos assigned before each session

### You Set Up

- Supabase project
- Upstash account
- Vercel account
- GitHub repository

---

## What You Need to Know Before Starting

This bootcamp assumes you're already comfortable with:

- **TypeScript** — types, interfaces, basic generics, strict mode
- **React** — functional components, hooks (useState, useEffect, useContext), forms, component state
- **Git** — clone, branch, commit, push, pull, pull requests
- **SQL basics** — tables, rows, columns, primary keys, foreign keys, basic queries
- **Command line** — terminal navigation, npm/pnpm commands

You do **not** need prior experience with Next.js, Prisma, Supabase, TanStack Query, Zod, Upstash, Vercel AI SDK, or Resend. You'll learn all of these during the bootcamp.

---

## What's Explicitly Out of Scope

Don't build any of these. They're not expected:

- Multi-tenant organizations
- Public sign-ups
- Rate limiting
- Automated testing
- Realtime presence
- Lead scoring
- WhatsApp or calling integrations
- Custom pipeline stages
- AI that writes to the database without user confirmation
- Mobile-responsive design (desktop-first is fine)

---

## One Last Thing

This project is hard. It's designed to be. You're building a real application with a real tech stack, and by the end, you'll have something genuinely impressive to show for it.

The key to finishing is consistency. Do the homework. Watch the videos. Ask questions when you're stuck. Don't let two sessions go by without catching up.

Let's build something great.
