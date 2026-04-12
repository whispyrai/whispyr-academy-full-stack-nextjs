# Session 7: Post-Session Recap & Tutorial

**Manager Dashboard + File Attachments**

---

## What We Built

Session 7 gave the CRM its first **analytics surface** and laid the groundwork for **document management**. Until now, the dashboard was a static stub and leads had no way to carry files. Tonight we fixed the first problem end-to-end and built the service layer for the second, using patterns you've now seen seven times.

By the end of Session 7, the `whispyr-academy-crm` repo had:

- **Session 6 homework landed** -- CSV export, valid-row preview, admin layout with tabs, resend-invite action, and paginated users table all committed and verified before new work began
- **ESLint cleanup** -- unused variables dropped from `seed-agents.ts` and `CsvImporter.tsx` before touching new code
- **Dashboard service module** -- the seventh service to follow the 5-file pattern (`types`, `db`, `helpers`, `service`, `index`) exposing `getDashboardData` as a single orchestration point
- **Seven parallel aggregation queries** -- `getDashboardData` runs total leads, leads-by-stage, leads-by-status, overdue reminders, new-leads-this-week, new-leads-last-week, and won/total conversion counts inside a single `Promise.all([...])`
- **Role-aware data filtering** -- agents see only their own leads (`assignedToId: user.id`), managers and admins see everything. Top agents leaderboard is hidden entirely for agents
- **Week-over-week comparison** -- `startOfUtcWeekSunday` helper computes the UTC week boundary so "new leads this week" can be compared against last week with a percentage change
- **Conversion rate calculation** -- `(won / total) * 100` with a zero-division guard, returned alongside the raw won and total counts
- **`DashboardData` type** -- derived from `ReturnType<typeof getDashboardData>` so the type always matches the service, not a hand-written interface
- **Dashboard API route** -- `GET /api/dashboard` gated by `authenticateUser()` (all authenticated roles can access; the service itself scopes the data by role)
- **`useDashboardOverview` hook** -- a TanStack Query hook with a 60-second `staleTime`, keyed on `["dashboard"]`
- **shadcn chart component** -- `src/components/ui/chart.tsx` installed as the Recharts wrapper layer (`ChartContainer`, `ChartConfig`, `ChartTooltip`, `ChartTooltipContent`) providing theme-aware, responsive chart rendering
- **KPI cards** -- a reusable `KpiCard` component (label + icon + value + optional subValue) rendered four times: Total Leads, New Leads This Week (with week-over-week subtext), Conversion Rate (with won/total subtext), and Overdue Reminders
- **Stage breakdown bar chart** -- `ByStageBreakdown` renders a Recharts `BarChart` inside a `ChartContainer` with `CartesianGrid`, `XAxis`, `ChartTooltip`, and themed `Bar` fill via `var(--chart-1)`
- **Top agents leaderboard** -- a `Card` listing agents sorted by won count, visible only to managers and admins
- **Dashboard page rewrite** -- `/dashboard` is now a server component that authenticates and passes `role` to `DashboardPageClient`, which renders the full KPI + chart + leaderboard layout with role-conditional grid styling
- **`Attachment` Prisma model** -- a new table with `id`, `leadId`, `uploadedById`, `fileName`, `storagePath` (unique), `mimeType`, `sizeBytes`, and `createdAt`. Relations added to `Lead` and `Profile`
- **`add_attachment_model` migration** -- generated with `npx prisma migrate dev --name add_attachment_model` and the client regenerated live
- **Supabase Storage helpers** -- `src/lib/supabase/storage.ts` wrapping `supabaseAdmin.storage` with three functions: `uploadLeadAttachment`, `deleteLeadAttachment`, and `getLeadAttachmentSignedUrl` (1-hour TTL)
- **Attachments service module** -- the eighth service to follow the 5-file pattern, with `listForLead` (signed URL generation in parallel via `Promise.all`) and `uploadForLead` (validate size/MIME, upload to Storage, Prisma transaction for metadata + activity, cleanup on failure)
- **`AttachmentServiceError`** -- a new domain error class wired into `handleRouteError` alongside the other service errors
- **Server-side validation constants** -- `ALLOWED_MIME_TYPES` (images, PDF, Word, Excel, video) and `MAX_FILE_SIZE_BYTES = 10 * 1024 * 1024` exported from `src/services/attachments/schema.ts`
- **`AttachmentListItem` type** -- omits `storagePath` from the DB row and adds `downloadUrl`, so the internal storage path never leaks to the client

> **Note:** We built the manager dashboard end-to-end and the attachments **service layer** (model, migration, storage helpers, service module). We did **NOT** build the attachment API routes, TanStack hooks, Files tab UI, the status pie chart, delete-attachment flow, or the production polish pass. Those are your **assignment before Session 8**. See the "Assignment" section below.

---

## Step-by-Step Walkthrough

If you missed the session or want to review, follow these steps in order. Each step references the exact files in the [`whispyr-academy-crm`](https://github.com/whispyr-academy/whispyr-academy-crm) repo.

---

### Phase 0: Session 6 Carryover -- Verify Your Assignment Work

Before starting the dashboard, confirm your Session 6 homework is complete:

- `GET /api/leads/export` returns a CSV file with role-scoped leads and a `Content-Disposition: attachment` header
- `/admin` has a shared layout with Users / Import / Export tabs using `usePathname()` for active state
- The CSV importer shows valid-row preview AND invalid-row errors
- "Resend invite" sends a fresh magic link to an existing user
- Users table paginates with page/pageSize query params

If any of that is incomplete, finish it before moving on. Session 7 builds on top of a clean admin layer.

We also ran a quick ESLint pass to drop unused variables from `seed-agents.ts` and `CsvImporter.tsx` before writing new code. Clean slate.

---

### Phase 1: Why Aggregate At The Database

**Concepts covered:** N+1 performance traps, database aggregations with Prisma `groupBy`, parallel query orchestration with `Promise.all`.

**The trap:**

Imagine you have 10,000 leads and you want to know how many are in each stage. The naive approach looks like this:

```typescript
// DO NOT DO THIS
const leads = await prisma.lead.findMany();
const newCount = leads.filter((l) => l.stage === "NEW").length;
const contactedCount = leads.filter((l) => l.stage === "CONTACTED").length;
```

This hydrates 10,000 rows over the wire, allocates 10,000 JavaScript objects, and iterates the whole array multiple times. It will work fine on your dev database. It will time out in production.

**The correct approach:**

```typescript
const rows = await prisma.lead.groupBy({
  by: ["stage"],
  _count: { _all: true },
});
// rows = [{ stage: "NEW", _count: { _all: 42 } }, ...]
```

`groupBy` tells the database "give me one row per stage with the count attached." The database uses its indexes and query planner to return a tiny result set -- usually in milliseconds -- regardless of how many leads you have.

**Rule for the rest of your career:** never count in JavaScript what you can count in SQL.

**`Promise.all` for parallelism:**

`getDashboardData` needs seven aggregations (total leads, stage counts, status counts, overdue reminders, new-this-week, new-last-week, won/total). Running them sequentially means waiting for all seven to finish one after the other. Running them with `Promise.all([...])` means the total time equals the slowest single query.

```typescript
const [
  totalLeads,
  totalLeadsByStage,
  totalLeadsByStatus,
  overdueRemindersCount,
  newLeadsThisWeekCount,
  newLeadsLastWeekCount,
  { total: conversionTotal, won: conversionWon },
] = await Promise.all([
  dbGetTotalLeads(where),
  dbGetTotalLeadsByStage(where),
  dbGetTotalLeadsByStatus(where),
  dbGetOverdueRemindersCount(where),
  dbCountLeadsCreatedInRange(where, { gte: thisWeekStartUtc, lte: now }),
  dbCountLeadsCreatedInRange(where, { gte: lastWeekStartUtc, lt: thisWeekStartUtc }),
  dbGetWonAndTotalLeads(where),
]);
```

---

### Phase 2: Dashboard Service Module -- DB Layer

**Reference file:** `src/services/dashboard/db.ts`

Seven queries, each doing one thing. Every query takes a `where` parameter so the service can scope by role upstream.

```typescript
import {
  LeadStage,
  LeadStatus,
  Prisma,
  ReminderStatus,
  Role,
} from "@/generated/prisma/client";
import { prisma } from "@/lib/prisma";

export async function dbGetTotalLeads(where: Prisma.LeadWhereInput) {
  return await prisma.lead.count({ where });
}
```

**Stage counts with `groupBy`:**

```typescript
export async function dbGetTotalLeadsByStage(where: Prisma.LeadWhereInput) {
  const result = await prisma.lead.groupBy({
    where,
    by: ["stage"],
    _count: {
      _all: true,
    },
  });

  return result
    .map((item) => ({
      stage: item.stage,
      count: item._count._all,
    }))
    .sort(
      (a, b) => sortStages().indexOf(a.stage) - sortStages().indexOf(b.stage),
    );
}
```

**Key design -- deterministic sort order:**

The `sortStages()` helper returns the enum values in pipeline order: `NEW`, `CONTACTED`, `QUALIFIED`, `NEGOTIATING`. Without this, `groupBy` returns rows in arbitrary database order, and the chart bars would shuffle on every refresh. The sort ensures the bar chart always reads left-to-right through the sales funnel.

**Status counts follow the same pattern:**

```typescript
export async function dbGetTotalLeadsByStatus(where: Prisma.LeadWhereInput) {
  const result = await prisma.lead.groupBy({
    where,
    by: ["status"],
    _count: { _all: true },
  });

  return result.map((item) => ({
    status: item.status,
    count: item._count._all,
  }));
}
```

**Date-range counting for week-over-week:**

```typescript
export async function dbCountLeadsCreatedInRange(
  baseWhere: Prisma.LeadWhereInput,
  createdAt: Prisma.DateTimeFilter,
) {
  return prisma.lead.count({
    where: {
      ...baseWhere,
      createdAt,
    },
  });
}
```

Reusable for any time window -- the service decides the boundaries, the db function just counts.

**Conversion rate -- parallel won + total:**

```typescript
export async function dbGetWonAndTotalLeads(baseWhere: Prisma.LeadWhereInput) {
  const [total, won] = await Promise.all([
    prisma.lead.count({ where: baseWhere }),
    prisma.lead.count({
      where: { ...baseWhere, status: LeadStatus.WON },
    }),
  ]);
  return { total, won };
}
```

Two counts run in parallel inside a single db function. Even within db helpers, parallelize when possible.

**Top agents leaderboard:**

```typescript
export async function dbGetTopAgents(limit = 5) {
  const agents = await prisma.profile.findMany({
    where: {
      role: Role.AGENT,
      isActive: true,
    },
    select: {
      id: true,
      name: true,
      email: true,
      _count: { select: { leads: true } },
      leads: {
        where: { status: LeadStatus.WON },
        select: { id: true },
      },
    },
  });

  return agents
    .map((agent) => ({
      id: agent.id,
      name: agent.name,
      email: agent.email,
      leadsCount: agent._count.leads,
      wonCount: agent.leads.length,
    }))
    .sort((a, b) => b.wonCount - a.wonCount)
    .slice(0, limit);
}
```

**Key design -- `_count` for total, filtered relation for won:**

Prisma's `_count` gives the total leads per agent. To get only WON leads, we include a filtered `leads` relation with `where: { status: WON }` and count the array length. This avoids a separate `groupBy` call per agent.

---

### Phase 3: Dashboard Service Module -- Helpers, Types, Service, Index

**Reference file:** `src/services/dashboard/helpers.ts`

```typescript
/**
 * Sunday 00:00:00.000 UTC for the UTC week containing `d`
 * (week runs Sunday-Saturday).
 */
export function startOfUtcWeekSunday(d: Date): Date {
  const day = d.getUTCDay();
  const x = new Date(d);
  x.setUTCDate(x.getUTCDate() - day);
  x.setUTCHours(0, 0, 0, 0);
  return x;
}
```

**Key design -- UTC everywhere:**

All date math uses `getUTCDay`, `setUTCDate`, `setUTCHours`. Local timezone arithmetic causes silent bugs when the server runs in a different timezone than the user. UTC boundaries are predictable, testable, and don't shift with daylight saving.

**Reference file:** `src/services/dashboard/types.ts`

```typescript
import { getDashboardData } from "./service";

export type DashboardData = {
  totalLeads: Awaited<ReturnType<typeof getDashboardData>>["totalLeads"];
  totalLeadsByStage: Awaited<ReturnType<typeof getDashboardData>>["totalLeadsByStage"];
  totalLeadsByStatus: Awaited<ReturnType<typeof getDashboardData>>["totalLeadsByStatus"];
  overdueRemindersCount: Awaited<ReturnType<typeof getDashboardData>>["overdueRemindersCount"];
  newLeadsThisWeek: Awaited<ReturnType<typeof getDashboardData>>["newLeadsThisWeek"];
  conversionRate: Awaited<ReturnType<typeof getDashboardData>>["conversionRate"];
  topAgents?: Awaited<ReturnType<typeof getDashboardData>>["topAgents"];
};
```

**Key design -- types derived from the service, not hand-written:**

Every field in `DashboardData` is extracted from the return type of `getDashboardData` using `Awaited<ReturnType<...>>`. This means the type automatically stays in sync with the service -- if a new field is added to the return object, the type picks it up. If a field is renamed, TypeScript catches every consumer. No hand-written interface to forget to update.

The `topAgents` field is optional (`?`) because the service only includes it for non-agent roles.

**Reference file:** `src/services/dashboard/service.ts`

```typescript
import { Role } from "@/generated/prisma/client";
import { UserSnapshot } from "@/utils/types/user";
import {
  dbCountLeadsCreatedInRange,
  dbGetOverdueRemindersCount,
  dbGetTopAgents,
  dbGetTotalLeads,
  dbGetTotalLeadsByStage,
  dbGetTotalLeadsByStatus,
  dbGetWonAndTotalLeads,
} from "./db";
import { startOfUtcWeekSunday } from "./helpers";

export async function getDashboardData(user: UserSnapshot) {
  const where = {
    ...(user.role === Role.AGENT && { assignedToId: user.id }),
  };

  const now = new Date();
  const thisWeekStartUtc = startOfUtcWeekSunday(now);
  const lastWeekStartUtc = new Date(thisWeekStartUtc);
  lastWeekStartUtc.setUTCDate(lastWeekStartUtc.getUTCDate() - 7);

  const [
    totalLeads,
    totalLeadsByStage,
    totalLeadsByStatus,
    overdueRemindersCount,
    newLeadsThisWeekCount,
    newLeadsLastWeekCount,
    { total: conversionTotal, won: conversionWon },
  ] = await Promise.all([
    dbGetTotalLeads(where),
    dbGetTotalLeadsByStage(where),
    dbGetTotalLeadsByStatus(where),
    dbGetOverdueRemindersCount(where),
    dbCountLeadsCreatedInRange(where, { gte: thisWeekStartUtc, lte: now }),
    dbCountLeadsCreatedInRange(where, { gte: lastWeekStartUtc, lt: thisWeekStartUtc }),
    dbGetWonAndTotalLeads(where),
  ]);

  const percentChangeFromLastWeek =
    newLeadsLastWeekCount === 0
      ? null
      : ((newLeadsThisWeekCount - newLeadsLastWeekCount) /
          newLeadsLastWeekCount) * 100;

  const conversionRate =
    conversionTotal === 0 ? 0 : (conversionWon / conversionTotal) * 100;

  let topAgents: Awaited<ReturnType<typeof dbGetTopAgents>> = [];

  if (user.role !== Role.AGENT) {
    topAgents = await dbGetTopAgents();
  }

  return {
    totalLeads,
    totalLeadsByStage,
    totalLeadsByStatus,
    overdueRemindersCount,
    newLeadsThisWeek: {
      count: newLeadsThisWeekCount,
      lastWeekCount: newLeadsLastWeekCount,
      percentChangeFromLastWeek,
    },
    conversionRate: {
      percentage: conversionRate,
      won: conversionWon,
      total: conversionTotal,
    },
    ...(user.role !== Role.AGENT && { topAgents }),
  };
}
```

**Key design -- role-aware WHERE clause:**

```typescript
const where = {
  ...(user.role === Role.AGENT && { assignedToId: user.id }),
};
```

If the user is an agent, every query is scoped to their assigned leads. Managers and admins get an empty `where`, which means "all leads." This single line controls the entire dashboard's data boundary. The spread-with-conditional pattern (`...(condition && { key: value })`) is the cleanest way to conditionally add a property to an object literal.

**Key design -- `percentChangeFromLastWeek` null guard:**

If last week had zero new leads, division would produce `Infinity`. We return `null` instead, and the UI renders a fallback message ("No new leads in the prior week to compare"). Never divide by zero; handle the edge case explicitly.

**Key design -- `topAgents` runs AFTER `Promise.all`:**

The top-agents query is role-gated (agents don't see it). Rather than adding a conditional branch inside `Promise.all`, the service runs it separately after the main batch, only for non-agents. This keeps the `Promise.all` clean and avoids running a wasted query for agents.

**Reference file:** `src/services/dashboard/index.ts`

```typescript
import { getDashboardData } from "./service";
export type { DashboardData } from "./types";

export const DashboardService = {
  getDashboardData,
} as const;
```

Same namespaced barrel export as every other service.

---

### Phase 4: Dashboard Route + TanStack Hook

**Reference file:** `src/app/api/dashboard/route.ts`

```typescript
import { DashboardService } from "@/services/dashboard";
import { authenticateUser } from "@/utils/authenticateUser";
import { handleRouteError } from "@/utils/handleRouteError";
import { NextResponse } from "next/server";

export async function GET() {
  try {
    const profile = await authenticateUser();
    const overview = await DashboardService.getDashboardData(profile);
    return NextResponse.json({ success: true, data: overview });
  } catch (error) {
    return handleRouteError(error);
  }
}
```

**Key design -- no role array in `authenticateUser()`:**

Unlike the admin routes which pass `[Role.ADMIN]`, the dashboard route calls `authenticateUser()` with no role restriction. Every authenticated user can hit the endpoint -- the **service** handles role-based data scoping internally. This is a deliberate architectural choice: agents need a dashboard too, they just see different data.

**Reference file:** `src/lib/tanstack/useDashboardOverview.ts`

```typescript
import { useQuery } from "@tanstack/react-query";
import { api } from "@/lib/api";
import type { DashboardData } from "@/services/dashboard";

/** Fetches dashboard overview including newLeadsThisWeek and conversionRate. */
export function useDashboardOverview() {
  return useQuery<DashboardData>({
    queryKey: ["dashboard"],
    queryFn: async () => {
      const { data } = await api.get("/dashboard");
      return data.data;
    },
    // Dashboard data is expensive to compute. Keep it fresh for
    // 60 seconds before refetching on window focus.
    staleTime: 60 * 1000,
  });
}
```

**Key design -- 60-second `staleTime`:**

Dashboards don't need to be real-time. The numbers change over hours, not seconds. A `staleTime` of 60 seconds means TanStack will cache the result between renders, tab switches, and brief navigations, which keeps the database calm. Compare this to the leads list (no staleTime -- always fresh) and the attachment list (55-minute staleTime tuned to the signed URL TTL, which you'll set up in homework).

**Key design -- `data.data` unwrapping:**

The API returns `{ success: true, data: overview }`. The Axios response wraps that in `response.data`. So `data.data` is the actual dashboard payload. This double-unwrap is consistent with every other hook in the codebase.

---

### Phase 5: Dashboard UI -- KPI Cards + Charts

**Reference file:** `src/app/(protected)/dashboard/page.tsx`

```typescript
import { authenticateUser } from "@/utils/authenticateUser";
import { DashboardPageClient } from "@/components/dashboard/dashboard-page-client";

const DashboardPage = async () => {
  const profile = await authenticateUser();
  return <DashboardPageClient role={profile.role} />;
};

export default DashboardPage;
```

Server component authenticates and passes just the `role` prop. The client component handles rendering.

**Reference file:** `src/components/dashboard/KpiCard.tsx`

```typescript
import React from "react";
import { Card, CardHeader, CardTitle, CardContent } from "../ui/card";

type KpiCardProps = {
  label: string;
  value: React.ReactNode;
  icon: React.ReactNode;
  subValue?: string;
};

const KpiCard = ({ label, value, icon, subValue }: KpiCardProps) => {
  return (
    <Card>
      <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
        <CardTitle className="text-sm font-medium text-muted-foreground">
          {label}
        </CardTitle>
        {icon}
      </CardHeader>
      <CardContent className="space-y-1">
        <div className="text-3xl font-semibold tabular-nums">{value}</div>
        {subValue ? (
          <p className="text-xs text-muted-foreground">{subValue}</p>
        ) : null}
      </CardContent>
    </Card>
  );
};

export default KpiCard;
```

**Key design -- `tabular-nums`:**

The Tailwind class `tabular-nums` forces equal-width digits so numbers don't cause layout shifts when they change. Without it, "1,234" and "5,678" take different widths because proportional fonts render "1" narrower than "5". Dashboard cards with fluctuating numbers need this.

**Key design -- `React.ReactNode` for `value`:**

The value prop accepts any renderable content, not just strings. This lets the conversion rate card pass `${percentage.toFixed(1)}%` as a formatted string while the count cards pass raw numbers. One component, flexible rendering.

**Reference file:** `src/components/dashboard/ByStageBreakdown.tsx`

```typescript
import { DashboardData } from "@/services/dashboard";
import { Card, CardContent, CardHeader, CardTitle } from "../ui/card";
import { BarChart, XAxis, Bar, CartesianGrid } from "recharts";
import {
  ChartConfig,
  ChartContainer,
  ChartTooltip,
  ChartTooltipContent,
} from "../ui/chart";

const ByStageBreakdown = ({
  data,
}: {
  data: DashboardData["totalLeadsByStage"];
}) => {
  const chartConfig = {
    stage: {
      label: "Stage",
      color: "var(--chart-1)",
    },
  } satisfies ChartConfig;

  return (
    <Card className="col-span-3">
      <CardHeader>
        <CardTitle>Leads by Stage</CardTitle>
      </CardHeader>
      <CardContent>
        <ChartContainer config={chartConfig}>
          <BarChart accessibilityLayer data={data}>
            <CartesianGrid vertical={false} />
            <XAxis
              dataKey="stage"
              tickLine={false}
              tickMargin={10}
              axisLine={false}
            />
            <ChartTooltip
              cursor={false}
              content={<ChartTooltipContent hideLabel />}
            />
            <Bar dataKey="count" fill="var(--chart-1)" radius={8} />
          </BarChart>
        </ChartContainer>
      </CardContent>
    </Card>
  );
};

export default ByStageBreakdown;
```

**Key design -- shadcn chart components instead of raw Recharts:**

Instead of importing `ResponsiveContainer` directly, we use shadcn's `ChartContainer` which wraps `ResponsiveContainer` and provides theme integration through `ChartConfig`. The config object maps data keys to labels and CSS custom property colors (`var(--chart-1)`). This means the chart automatically adapts to light/dark mode without any manual color management.

**Key design -- `accessibilityLayer` prop:**

The `accessibilityLayer` prop on `BarChart` adds ARIA attributes and keyboard navigation to the chart. Screen readers can navigate between bars. This is a one-prop accessibility win.

**Key design -- `CartesianGrid vertical={false}`:**

Removing vertical grid lines reduces visual noise. The horizontal grid helps read values; the vertical grid just clutters.

**Reference file:** `src/components/dashboard/dashboard-page-client.tsx`

```typescript
"use client";

import { useDashboardOverview } from "@/lib/tanstack/useDashboardOverview";
import { AlertCircle, Percent, UserPlus, Users } from "lucide-react";
import { useMemo } from "react";
import KpiCard from "./KpiCard";
import { Role } from "@/generated/prisma/enums";
import { cn } from "@/lib/utils";
import ByStageBreakdown from "./ByStageBreakdown";
import { Card, CardContent, CardHeader, CardTitle } from "../ui/card";

function formatWeekOverWeekSubtext(
  percentChange: number | null,
): string | undefined {
  if (percentChange === null) {
    return "No new leads in the prior week to compare";
  }
  const sign = percentChange > 0 ? "+" : "";
  return `${sign}${percentChange.toFixed(1)}% vs last week`;
}

export function DashboardPageClient({ role }: { role: Role }) {
  const { data, isLoading, error } = useDashboardOverview();

  const { subHeaderText, secondRowGridStyle } = useMemo(() => {
    if (role === "AGENT") {
      return {
        subHeaderText: "Review the current state of your leads in your pipeline.",
        secondRowGridStyle: "grid-cols-1",
      };
    }
    return {
      subHeaderText:
        "Review the current state of the pipeline in your organization.",
      secondRowGridStyle: "grid-cols-4",
    };
  }, [role]);

  if (isLoading) return <div>Loading...</div>;
  if (error || !data)
    return <div>Error: {error?.message ?? "Unknown error"}</div>;

  return (
    <div className="space-y-6 p-4 md:p-6">
      <div className="space-y-1">
        <h1 className="text-3xl font-semibold tracking-tight">Dashboard</h1>
        <p className="text-sm text-muted-foreground">{subHeaderText}</p>
      </div>

      <div className="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-4">
        <KpiCard
          label="Total leads"
          value={data.totalLeads}
          icon={<Users className="h-4 w-4 text-muted-foreground" />}
        />
        <KpiCard
          label="New leads this week"
          value={data.newLeadsThisWeek.count}
          icon={<UserPlus className="h-4 w-4 text-muted-foreground" />}
          subValue={formatWeekOverWeekSubtext(
            data.newLeadsThisWeek.percentChangeFromLastWeek,
          )}
        />
        <KpiCard
          label="Conversion rate"
          value={`${data.conversionRate.percentage.toFixed(1)}%`}
          icon={<Percent className="h-4 w-4 text-muted-foreground" />}
          subValue={`${data.conversionRate.won} won / ${data.conversionRate.total} total leads`}
        />
        <KpiCard
          label="Overdue reminders"
          value={data.overdueRemindersCount}
          icon={<AlertCircle className="h-4 w-4 text-muted-foreground" />}
        />
      </div>

      <div className={cn("grid gap-4", secondRowGridStyle)}>
        <ByStageBreakdown data={data.totalLeadsByStage} />
        {data.topAgents ? (
          <Card>
            <CardHeader>
              <CardTitle>Top Performing Agents</CardTitle>
              <CardContent>
                {data.topAgents.map((agent) => (
                  <div key={agent.id} className="flex flex-col gap-1">
                    <p className="text-sm font-medium">{agent.name}</p>
                    <p className="text-xs text-muted-foreground">
                      Won {agent.wonCount} of {agent.leadsCount} leads
                    </p>
                  </div>
                ))}
              </CardContent>
            </CardHeader>
          </Card>
        ) : (
          <></>
        )}
      </div>
    </div>
  );
}
```

**Key design -- `useMemo` for role-dependent layout:**

The `useMemo` computes sub-header text and grid styling once per role change. Agents see a single-column second row (no leaderboard), managers/admins see a four-column grid (chart spanning 3, leaderboard in 1). The `cn()` utility merges the base `"grid gap-4"` with the dynamic column class.

**Key design -- `formatWeekOverWeekSubtext` as a pure function:**

Formatting logic lives outside the component as a pure function. It handles the null case (no prior week data) and the sign prefix (`+` for positive, negative sign is automatic). This keeps the JSX clean and the formatting testable.

**Key design -- conditional leaderboard rendering:**

```typescript
{data.topAgents ? (<Card>...</Card>) : <></>}
```

The service omits `topAgents` entirely for agents (via the spread-conditional pattern). On the client, `data.topAgents` is `undefined` for agents, so the whole leaderboard card disappears. Empty fragment `<></>` avoids a React key warning.

---

### Phase 6: Object Storage Concepts

**Concepts covered:** Object storage vs. database, public vs. private buckets, signed URLs, the `Attachment` Prisma model.

**The rule:** never store binary data in Postgres. Files go in object storage. Metadata goes in the database.

**Public vs. private buckets:**

- A **public bucket** serves any file at a guessable URL. Anyone who knows (or can guess) the path can download. Fine for avatars on a public profile page. Disastrous for contracts, IDs, call recordings.
- A **private bucket** refuses every request by default. To let a user download a file, the server mints a **signed URL** -- a short-lived, cryptographically signed link that encodes the path, an expiry timestamp, and an HMAC signature. Any attempt to change any part of the URL invalidates the signature.

**Default TTL:** 1 hour. That's a balance between usability (the user has time to actually download) and security (a leaked URL stops working soon).

**Rule for the rest of your career:** private bucket + signed URL is the production default. Public buckets are for public files. Everything else is private.

**Adding the `Attachment` model:**

```prisma
model Attachment {
  id String @id @default(uuid())

  leadId String
  lead   Lead   @relation(fields: [leadId], references: [id])

  uploadedById String
  uploadedBy   Profile @relation(fields: [uploadedById], references: [id])

  fileName    String
  storagePath String @unique
  mimeType    String
  sizeBytes   Int

  createdAt DateTime @default(now())
}
```

Relations added to both `Lead` and `Profile`:

```prisma
// In Lead model:
attachments   Attachment[]

// In Profile model:
attachments   Attachment[]
```

`storagePath` is `@unique` in the database -- our collision detector. If two concurrent uploads somehow generate the same path, the second insert fails instead of silently overwriting.

Then:

```bash
npx prisma migrate dev --name add_attachment_model
npx prisma generate
```

**Bucket creation:**

Supabase Dashboard -> Storage -> New bucket -> name `lead-attachments`, **Public: OFF**, file size limit `10 MB`, allowed MIME types blank (we validate server-side).

---

### Phase 7: Supabase Storage Helpers

**Reference file:** `src/lib/supabase/storage.ts`

```typescript
import supabaseAdmin from "./admin";

const LEAD_ATTACHMENTS_BUCKET = "lead-attachments";
const SIGNED_URL_TTL_SECONDS = 60 * 60; // 1 HOUR

export async function uploadLeadAttachment(
  storagePath: string,
  file: File,
): Promise<void> {
  const { error } = await supabaseAdmin.storage
    .from(LEAD_ATTACHMENTS_BUCKET)
    .upload(storagePath, file, {
      contentType: file.type,
      upsert: false,
    });

  if (error) {
    throw new Error(`Failed to upload attachment: ${error.message}`);
  }
}

export async function deleteLeadAttachment(storagePath: string): Promise<void> {
  const { error } = await supabaseAdmin.storage
    .from(LEAD_ATTACHMENTS_BUCKET)
    .remove([storagePath]);

  if (error) {
    throw new Error(`Failed to delete attachment: ${error.message}`);
  }
}

export async function getLeadAttachmentSignedUrl(
  storagePath: string,
): Promise<string> {
  const { data, error } = await supabaseAdmin.storage
    .from(LEAD_ATTACHMENTS_BUCKET)
    .createSignedUrl(storagePath, SIGNED_URL_TTL_SECONDS);

  if (error) {
    throw new Error(`Failed to get signed URL: ${error.message}`);
  }

  return data.signedUrl;
}
```

**Key design -- `supabaseAdmin` isolation:**

This is the ONLY file that imports `supabaseAdmin` for storage operations. Every other file calls these three helpers. This means `supabaseAdmin` never accidentally ends up in a client bundle. If you see `supabaseAdmin` imported in a component file, that's a security bug -- the service role key would ship to the browser.

**Key design -- `upsert: false`:**

The upload call explicitly sets `upsert: false`. This means the upload fails if the path is already taken, rather than silently overwriting an existing file. Combined with the `@unique` constraint on `storagePath` in Prisma, we have two layers of collision protection.

**Key design -- `contentType: file.type`:**

Without setting `contentType`, Supabase defaults to `application/octet-stream`, which means browsers will download the file instead of previewing it. Setting it correctly means PDFs open in the browser, images render inline, and Word docs get the right icon.

**Key design -- error throwing, not returning:**

All three functions throw on error rather than returning `{ data, error }`. This keeps the service layer clean -- it can use try/catch instead of checking error objects after every call.

---

### Phase 8: Attachments Service Module

**Reference file:** `src/services/attachments/schema.ts`

```typescript
import { z } from "zod";
import { dbListAttachmentsForLead } from "./db";

export const ALLOWED_MIME_TYPES = [
  "image/jpeg",
  "image/png",
  "image/webp",
  "application/pdf",
  "text/plain",
  "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
  "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
  "video/mp4",
  "video/mpeg",
  "video/quicktime",
];

export const MAX_FILE_SIZE_BYTES = 10 * 1024 * 1024; // 10MB

export const uploadAttachmentSchema = z.object({
  leadId: z.uuid(),
});

export type AttachmentListItem = Omit<
  Awaited<ReturnType<typeof dbListAttachmentsForLead>>[number],
  "storagePath"
> & { downloadUrl: string };
```

**Key design -- `AttachmentListItem` omits `storagePath`:**

The internal storage path (`leads/abc123/1712345678-contract.pdf`) is an implementation detail. Clients never need it -- they get a `downloadUrl` (the signed URL) instead. The `Omit<..., "storagePath"> & { downloadUrl: string }` type enforces this at compile time: if anyone tries to return the raw path, TypeScript catches it.

**Key design -- broad MIME type allowlist:**

Unlike the session plan which listed only images and PDF, the actual implementation allows Word docs (`.docx`), Excel spreadsheets (`.xlsx`), plain text, and video formats. A real CRM needs to attach contracts, proposals, call recordings, and spreadsheets -- not just images.

**Reference file:** `src/services/attachments/helpers.ts`

```typescript
export function buildStoragePath(leadId: string, fileName: string) {
  const safeName = fileName.replace(/[^a-zA-Z0-9._-]/g, "_");
  return `${leadId}/${Date.now()}-${safeName}`;
}
```

**Key design -- filename sanitization + timestamp prefix:**

Two problems solved in one function:

1. **Sanitization:** `replace(/[^a-zA-Z0-9._-]/g, "_")` strips spaces, Unicode characters, directory traversal attempts (`../`), and anything else that could break a URL or a filesystem path.
2. **Collision prevention:** `${Date.now()}-` prefixes a millisecond timestamp. Two uploads of `contract.pdf` to the same lead get paths like `abc123/1712345678000-contract.pdf` and `abc123/1712345678500-contract.pdf`. No collision, no overwrite.

**Reference file:** `src/services/attachments/db.ts`

```typescript
import { Prisma } from "@/generated/prisma/client";
import { prisma } from "@/lib/prisma";

export async function dbListAttachmentsForLead(leadId: string) {
  return prisma.attachment.findMany({
    where: { leadId },
    orderBy: { createdAt: "desc" },
    select: {
      id: true,
      fileName: true,
      mimeType: true,
      sizeBytes: true,
      storagePath: true,
      createdAt: true,
      uploadedBy: {
        select: { id: true, name: true },
      },
    },
  });
}

export async function dbFindAttachmentById(id: string) {
  return prisma.attachment.findUnique({
    where: { id },
    select: {
      id: true,
      fileName: true,
      mimeType: true,
      sizeBytes: true,
      storagePath: true,
      createdAt: true,
      uploadedBy: {
        select: { id: true, name: true },
      },
    },
  });
}

export function dbGetLeadById(id: string) {
  return prisma.lead.findUnique({
    where: { id },
    select: { id: true },
  });
}

export function dbCreateAttachment(
  input: {
    leadId: string;
    uploadedById: string;
    fileName: string;
    storagePath: string;
    mimeType: string;
    sizeBytes: number;
  },
  tx?: Prisma.TransactionClient,
) {
  const client = tx ?? prisma;
  return client.attachment.create({
    data: input,
    select: { id: true },
  });
}
```

**Key design -- `tx?: Prisma.TransactionClient`:**

`dbCreateAttachment` accepts an optional transaction client. When called from `uploadForLead`, it receives the transaction client so the attachment row and the activity row commit atomically. When called standalone (if ever needed), it falls back to the default `prisma` client. This is the canonical pattern for making db functions transaction-aware without forcing every caller to open a transaction.

**Reference file:** `src/services/attachments/service.ts`

```typescript
import {
  deleteLeadAttachment,
  getLeadAttachmentSignedUrl,
  uploadLeadAttachment,
} from "@/lib/supabase/storage";
import { dbCreateAttachment, dbGetLeadById, dbListAttachmentsForLead } from "./db";
import { ALLOWED_MIME_TYPES, AttachmentListItem, MAX_FILE_SIZE_BYTES } from "./schema";
import { UserSnapshot } from "@/utils/types/user";
import { buildStoragePath } from "./helpers";
import { prisma } from "@/lib/prisma";
import { ActivityService } from "../activity";
import { ActivityType } from "@/generated/prisma/enums";

export class AttachmentServiceError extends Error {
  constructor(
    message: string,
    public statusCode: number,
  ) {
    super(message);
    this.name = "AttachmentServiceError";
  }
}

export async function listForLead(
  leadId: string,
): Promise<AttachmentListItem[]> {
  const rows = await dbListAttachmentsForLead(leadId);

  return Promise.all(
    rows.map(async (row) => ({
      id: row.id,
      fileName: row.fileName,
      mimeType: row.mimeType,
      sizeBytes: row.sizeBytes,
      createdAt: row.createdAt,
      uploadedBy: row.uploadedBy,
      downloadUrl: await getLeadAttachmentSignedUrl(row.storagePath),
    })),
  );
}

export async function uploadForLead(input: {
  leadId: string;
  file: File;
  userSnapshot: UserSnapshot;
}) {
  const { leadId, file, userSnapshot } = input;

  // 1. Validate file size
  if (file.size > MAX_FILE_SIZE_BYTES) {
    throw new AttachmentServiceError(
      `File size exceeds maximum allowed size of ${MAX_FILE_SIZE_BYTES} bytes`,
      400,
    );
  }

  // 2. Validate file type
  if (!ALLOWED_MIME_TYPES.includes(file.type)) {
    throw new AttachmentServiceError(
      `File type ${file.type} is not allowed.`,
      400,
    );
  }

  // 3. Validate lead exists
  const lead = await dbGetLeadById(leadId);
  if (!lead) {
    throw new AttachmentServiceError("Lead not found", 404);
  }

  // 4. Upload file to Storage
  const storagePath = buildStoragePath(lead.id, file.name);
  await uploadLeadAttachment(storagePath, file);

  try {
    // 5. Create database record + activity in one transaction
    const attachment = await prisma.$transaction(async (tx) => {
      const attachment = await dbCreateAttachment(
        {
          leadId,
          uploadedById: userSnapshot.id,
          fileName: file.name,
          storagePath,
          mimeType: file.type,
          sizeBytes: file.size,
        },
        tx,
      );

      await ActivityService.create([
        {
          actorId: userSnapshot.id,
          leadId,
          type: ActivityType.ATTACHMENT_ADDED,
          content: `Uploaded attachment: ${file.name}`,
        },
      ]);

      return attachment;
    });

    return attachment;
  } catch (error) {
    // 6. Cleanup: delete orphaned file if DB write fails
    console.error(error);
    await deleteLeadAttachment(storagePath);
    throw new AttachmentServiceError("Failed to upload attachment", 500);
  }
}
```

**Key design -- the upload orchestration sequence:**

Walk through `uploadForLead` step by step:

1. **Validate size** -- throw 400 before any network I/O if the file is too large
2. **Validate MIME type** -- throw 400 if the file type isn't in the allowlist
3. **Validate lead exists** -- throw 404 if the leadId is bogus. This runs before the upload so we don't store files for non-existent leads
4. **Upload to Storage** -- the file hits Supabase. If this fails, throw -- nothing to clean up yet
5. **Prisma transaction** -- insert the Attachment row AND the Activity row atomically. Both commit or both roll back
6. **Cleanup on failure** -- if the transaction throws, delete the orphaned file from Storage, then re-throw so the route handler surfaces the error

This is the **double-write pattern with cleanup**. The file and the metadata must stay in sync. The file is written first because it's the harder operation to undo -- Storage doesn't have transactions. If the easier operation (database insert) fails, we clean up the file.

**Key design -- atomic activity logging (third time):**

The activity row lives **inside the transaction** with the attachment row. Either both commit or both roll back. There is no state where the attachment exists without the activity, or vice versa. This is the same pattern from Session 3 (lead calls activity) and Session 4 (reminder calls notification).

**Key design -- `listForLead` regenerates signed URLs on every call:**

```typescript
return Promise.all(
  rows.map(async (row) => ({
    ...fields,
    downloadUrl: await getLeadAttachmentSignedUrl(row.storagePath),
  })),
);
```

Never store signed URLs in the database. Never cache them in localStorage. Regenerate on read. The `Promise.all` ensures all URLs are generated in parallel -- one slow URL doesn't block the others.

**Reference file:** `src/services/attachments/index.ts`

```typescript
import { uploadAttachmentSchema } from "./schema";
import { listForLead, uploadForLead } from "./service";
export type { AttachmentListItem } from "./schema";

export const AttachmentService = {
  listForLead,
  uploadForLead,
} as const;

export const AttachmentSchema = {
  uploadForLead: uploadAttachmentSchema,
} as const;
```

**Key design -- `AttachmentSchema` alongside `AttachmentService`:**

The barrel export includes both the service and the schema namespace. Routes import `AttachmentSchema.uploadForLead` for validation and `AttachmentService.uploadForLead` for business logic. Same pattern as admin and import-export services.

---

### Phase 9: Registering the Error Class

**Reference file:** `src/utils/handleRouteError.ts`

```typescript
import { LeadServiceError } from "@/services/lead/service";
import { NotificationServiceError } from "@/services/notification/service";
import { AuthenticationError } from "./authenticateUser";
import { ZodError } from "zod";
import { NextResponse } from "next/server";
import { AdminServiceError } from "@/services/admin/service";
import { AttachmentServiceError } from "@/services/attachments/service";

export const handleRouteError = (error: unknown) => {
  if (
    error instanceof AuthenticationError ||
    error instanceof LeadServiceError ||
    error instanceof NotificationServiceError ||
    error instanceof AdminServiceError ||
    error instanceof AttachmentServiceError
  ) {
    return NextResponse.json(
      { error: error.message },
      { status: error.statusCode },
    );
  }

  if (error instanceof ZodError) {
    return NextResponse.json(
      { error: error.flatten().fieldErrors },
      { status: 400 },
    );
  }

  return NextResponse.json({ error: "Internal server error" }, { status: 500 });
};
```

`AttachmentServiceError` is now the fifth domain error class in the union. Every time you add a new service with its own error class, you add one `instanceof` check here. The pattern scales linearly and keeps error handling centralized.

---

## What We Did NOT Build (Your Assignment)

We built the manager dashboard end-to-end and the attachments service layer (model, migration, storage helpers, service). The following pieces were intentionally deferred:

1. **Attachment API routes** -- `GET /api/leads/[id]/attachments` and `POST /api/leads/[id]/attachments` (FormData parsing)
2. **Attachment TanStack hooks** -- `useAttachments(leadId)` and `useUploadAttachment(leadId)` with appropriate staleTime and cache invalidation
3. **Files tab on lead detail page** -- a new tab component with upload button, file list, and download links
4. **Status pie chart** -- a `PieChart` (or second `BarChart`) visualizing leads-by-status, complementing the stage bar chart
5. **Delete attachment flow** -- `DELETE /api/leads/[id]/attachments/[attachmentId]` with Storage cleanup and AlertDialog confirmation
6. **Production polish pass** -- loading skeletons, empty states, success/error toasts, disabled buttons during mutations

---

## Supplementary Videos

### Review What We Covered

**Recharts -- Composable React Charts**
[Build Dashboards with React and Recharts](https://www.youtube.com/watch?v=uiGlfhwPSKw) (~18 min)
How Recharts composes charts from primitives (`BarChart`, `XAxis`, `Bar`, `Tooltip`). How to theme charts with CSS custom properties. How to pass data as arrays of objects.

**Prisma `groupBy` -- Database Aggregations**
[Prisma groupBy and aggregations](https://www.youtube.com/watch?v=RebA5J-rlwg) by Prisma (~10 min)
`groupBy`, `_count`, `_sum`, `_avg`, `having` clauses. Useful if you want to extend the dashboard with average deal size or sum of revenue.

**Supabase Storage -- Private Buckets + Signed URLs**
[Supabase Storage Deep Dive](https://www.youtube.com/watch?v=dLqSmxX3r7I) by Supabase (~15 min)
Bucket policies, RLS on storage, signed URLs, upload/download lifecycle. Covers the concepts we used in the attachments module.

### Prepare for Session 8

**Vercel Deployment for Next.js**
[Deploy Next.js to Vercel](https://www.youtube.com/watch?v=2HBIzEx6IZA) by Lee Robinson (~12 min)
Environment variables, build commands, preview deployments, production domains. Session 8 starts with a live deploy.

**Prisma Migrations in Production**
[Prisma Migrate in Production](https://www.youtube.com/watch?v=8qbHJVSjqh0) by Prisma (~8 min)
`prisma migrate deploy` vs. `prisma migrate dev`. Why you never run `migrate dev` against production. How to handle migration history drift.

---

## Assignment: Build the Attachment Pipeline + Dashboard Polish Before Session 8

You have a working dashboard and a complete attachments service layer. Your assignment is to wire the attachments to the API and UI, add the missing chart, and polish everything for production.

### Task 1: Attachment API Routes (estimated: 1-1.5 hours)

Build the routes that expose the attachment service to the client.

**List attachments:** create `src/app/api/leads/[id]/attachments/route.ts` with a GET handler.

```typescript
export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> },
) {
  try {
    const profile = await authenticateUser();
    const { id: leadId } = await params;
    const attachments = await AttachmentService.listForLead(leadId);
    return NextResponse.json({ success: true, data: attachments });
  } catch (error) {
    return handleRouteError(error);
  }
}
```

**Upload attachment:** add a POST handler to the same file. Two things to get right:

1. Parse the request body with `request.formData()`, NOT `request.json()`. Browsers send multipart uploads as FormData.
2. Extract the file with `formData.get("file") as File`. Validate that it exists before passing to the service.

```typescript
export async function POST(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> },
) {
  try {
    const profile = await authenticateUser();
    const { id: leadId } = await params;
    const formData = await request.formData();
    const file = formData.get("file") as File | null;

    if (!file) {
      return NextResponse.json(
        { error: "No file provided" },
        { status: 400 },
      );
    }

    const attachment = await AttachmentService.uploadForLead({
      leadId,
      file,
      userSnapshot: profile,
    });

    return NextResponse.json({ success: true, data: attachment }, { status: 201 });
  } catch (error) {
    return handleRouteError(error);
  }
}
```

Remember: `params` is a Promise in Next.js 16 -- always `await` it.

### Task 2: Attachment TanStack Hooks (estimated: 30 minutes)

Create `src/lib/tanstack/useAttachments.ts` with two hooks:

**`useAttachments(leadId)`** -- a query hook keyed on `["attachments", leadId]`. Set `staleTime` to `55 * 60 * 1000` (55 minutes) -- deliberately shorter than the 60-minute signed URL TTL so cached URLs are always fresh when the user clicks Download.

**`useUploadAttachment(leadId)`** -- a mutation hook that POSTs a `FormData` object to `/leads/${leadId}/attachments`. On success, invalidate both `["attachments", leadId]` (to refresh the file list) and `["activities", leadId]` (to refresh the activity timeline, since every upload creates an `ATTACHMENT_ADDED` activity).

```typescript
export function useAttachments(leadId: string) {
  return useQuery<AttachmentListItem[]>({
    queryKey: ["attachments", leadId],
    queryFn: async () => {
      const { data } = await api.get(`/leads/${leadId}/attachments`);
      return data.data;
    },
    staleTime: 55 * 60 * 1000, // 55 min -- under the 60-min signed URL TTL
  });
}

export function useUploadAttachment(leadId: string) {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: async (file: File) => {
      const formData = new FormData();
      formData.append("file", file);
      const { data } = await api.post(
        `/leads/${leadId}/attachments`,
        formData,
      );
      return data.data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["attachments", leadId] });
      queryClient.invalidateQueries({ queryKey: ["activities", leadId] });
    },
  });
}
```

### Task 3: Files Tab on Lead Detail Page (estimated: 1.5-2 hours)

Create `src/components/leads/lead-details/Files.tsx` and integrate it as a new tab on the existing lead detail page.

**Expected behavior:**

- Empty state when no attachments exist (icon + "No files yet" + upload button)
- File list showing: file name, uploader name, relative timestamp (e.g., "2 hours ago"), file size, and a Download button
- Upload button that opens a file picker. On selection, call `useUploadAttachment(leadId).mutate(file)`
- Download button that opens the signed URL in a new tab (`window.open(attachment.downloadUrl, "_blank")`)
- Loading state while the attachment list fetches
- Error toast if upload fails (use the error message from the API response)

**Keep the component dumb.** It reads `useAttachments(leadId)` for the list and calls `useUploadAttachment(leadId).mutate(file)` on upload. All validation, Storage interaction, and activity logging happens in the service layer.

### Task 4: Status Visualization (estimated: 45 minutes)

Add a chart showing leads by status to complement the existing stage bar chart.

**Option A: Pie chart** using Recharts `PieChart` + `Pie` + `Cell` components inside a `ChartContainer`. Map each status (OPEN, WON, LOST) to a different `var(--chart-N)` color.

**Option B: Horizontal bar chart** using the same `BarChart` pattern as `ByStageBreakdown` but with `layout="vertical"`.

Either works. The data is already available in `data.totalLeadsByStatus` -- you just need the visualization component and a slot in the dashboard grid.

### Task 5: Delete Attachment (estimated: 45 minutes)

Add `DELETE /api/leads/[id]/attachments/[attachmentId]` and wire it to the UI.

**Service:** add `deleteForLead(attachmentId, userSnapshot)` to the attachments service. Look up the attachment, verify it belongs to the lead, delete the Storage object FIRST (since that's the harder-to-undo operation), then delete the DB row.

**Route:** `src/app/api/leads/[id]/attachments/[attachmentId]/route.ts` -- DELETE handler, authenticated.

**UI:** add a trash icon button on each file row in `Files.tsx`. Wrap the delete action in an `AlertDialog` (destructive action -- same pattern as user deactivation).

**Hook:** `useDeleteAttachment(leadId)` mutation that invalidates `["attachments", leadId]` on success.

### Task 6: Production Polish Pass (estimated: 1 hour)

Go through every page and mutation in the CRM. Check for:

- Loading skeletons or spinners on every data-fetching page
- Empty states on every list (leads, reminders, attachments, users)
- Success toasts on every mutation (create, update, delete, upload, import)
- Error toasts with real messages (not just "Something went wrong")
- Disabled submit buttons during pending mutations (`isPending` from useMutation)
- Missing keyboard accessibility (can you tab through the forms?)

Fix anything missing. This is the final polish before deployment in Session 8.

---

## Checkpoint Before Session 8

Before Session 8, verify all of the following:

- `/dashboard` renders four KPI cards with real counts, a stage bar chart, and a status visualization
- Managers/admins see the top agents leaderboard; agents see only their own data
- The week-over-week percentage shows correctly (or shows the fallback message when last week had zero leads)
- `GET /api/leads/[leadId]/attachments` returns an array of attachments with signed download URLs
- `POST /api/leads/[leadId]/attachments` accepts FormData, uploads to Supabase Storage, and creates the DB row + activity
- The Files tab on lead detail shows the attachment list with working download links
- Uploading a file shows it immediately in the list and creates an `ATTACHMENT_ADDED` entry in the activity timeline
- Uploading a file over 10MB or with a disallowed MIME type returns a clean error message
- Deleting an attachment removes it from both Storage and the database
- `npm run build` passes with zero errors
- Code committed and pushed to GitHub
- Pre-Session 8 videos watched (Vercel deployment + Prisma migrations in production)

---

## Resources

- [Prisma `groupBy` API](https://www.prisma.io/docs/orm/reference/prisma-client-reference#groupby) -- aggregation syntax, `_count`, `_sum`, `having`
- [Recharts Documentation](https://recharts.org/en-US) -- `BarChart`, `PieChart`, `ResponsiveContainer`, `Tooltip`
- [shadcn/ui Charts](https://ui.shadcn.com/charts) -- `ChartContainer`, `ChartConfig`, theme integration
- [Supabase Storage](https://supabase.com/docs/guides/storage) -- buckets, upload, signed URLs, RLS
- [Supabase Admin Client](https://supabase.com/docs/reference/javascript/admin-api) -- service-role operations
- [TanStack Query -- staleTime](https://tanstack.com/query/v5/docs/react/guides/caching) -- when data is "fresh" vs. "stale"
- [Prisma Transactions](https://www.prisma.io/docs/orm/prisma-client/queries/transactions) -- `$transaction` with the callback API
- [Next.js 16 Route Handlers](https://nextjs.org/docs/app/api-reference/file-conventions/route) -- `params` is now a Promise

---

*Session 7 Complete. You now know how to build analytics dashboards with database aggregations and Recharts, and how to build a file storage system on top of Supabase Storage with signed URLs and atomic activity logging. The CRM has its first real management surface. See you in Session 8, where we deploy to Vercel, seed production data, and deliver the capstone demo.*
