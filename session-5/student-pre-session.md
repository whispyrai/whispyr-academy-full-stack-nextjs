# Session 5: Pre-Session Guide

**AI Features — Lead Brief + Call Follow-up**

---

## Before You Arrive

### Session 4 Homework That Must Already Work

- [ ] GET /api/leads/[id]/reminders returns reminders with pagination
- [ ] GET /api/reminders returns user's reminders with status filtering
- [ ] PATCH /api/reminders/[id] can cancel + removes QStash message
- [ ] Lead detail Reminders tab shows reminders with status badges
- [ ] Create Reminder dialog with Title, Due Date, Due Time
- [ ] Overdue reminders highlighted (PENDING + dueAt past)
- [ ] /reminders page with table and filter tabs
- [ ] Click lead name navigates to lead detail
- [ ] Complete/Cancel actions work + update UI
- [ ] End-to-end flow: create → wait → notification appears → click navigates to lead
- [ ] Code committed and pushed

If didn't finish everything: prioritize reminder list views and Create Reminder dialog.

### Account Setup

- AI Gateway API key — will be provided before session. Add to .env as AI_GATEWAY_API_KEY.

### Videos To Watch

**Vercel AI SDK by Matt:** [https://www.youtube.com/watch?v=mojZpktAiYQ](https://www.youtube.com/watch?v=mojZpktAiYQ) (~15 min)

Focus on `generateText` with `Output.object()`, Zod schemas as contracts, typed AI output.

See videos.md for details.

You do NOT need to install any new packages before class.

---

## What We Will Build Live

Two AI-powered features using the Vercel AI SDK + Vercel AI Gateway:

**Lead Brief Generator:** structured brief from lead data + activity history

**Call Follow-up Assistant:** script + next step + reminder suggestion after logging a call

We are NOT building prompt history, A/B testing, or streaming. Those are deferred.

---

## What To Expect

- Duration: 3 hours (1.5 hrs → 10-min break → 1.5 hrs)
- Pattern: Same slides ↔ code alternation as Sessions 2-4
- Key concept: AI as a structured API integration. We're not building chatbots — we're using AI to transform data into actionable insights with typed, validated output schemas.
- Same service pattern: The 5-file service module (schema, db, service, helpers, index) you learned in Sessions 2-4 applies here too. AI is just another service.
