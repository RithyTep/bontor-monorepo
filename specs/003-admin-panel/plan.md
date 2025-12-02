# Implementation Plan: Admin Panel Dashboard

**Branch**: `003-admin-panel`
**Created**: 2024-12-02
**Spec**: `specs/003-admin-panel/spec.md`

---

## Summary

Build the admin dashboard with platform metrics, activity feed, and navigation. Implement role-based access control using NextAuth v5 session data.

---

## Technical Context

| Aspect | Decision |
|--------|----------|
| Framework | Next.js 16 (App Router) |
| Auth | NextAuth v5 with role in session |
| State | Server components for initial load, client for interactivity |
| UI | shadcn/ui components (Card, Table, etc.) |
| Charts | recharts (lightweight) |

---

## Constitution Check

### Phase -1: Pre-Implementation Gates

#### Simplicity Gate (Article V)
- [x] Dashboard is part of existing `apps/admin`
- [x] Metrics queries in `@karaoke/trpc` admin router

#### Anti-Abstraction Gate (Article VI)
- [x] Using NextAuth getServerSession directly
- [x] Standard Prisma aggregate queries
- [x] shadcn/ui components without wrappers

#### Integration-First Gate (Article VII)
- [x] Auth tested with real session
- [x] Metrics tested against seeded database

---

## Project Structure

```
apps/admin/
├── app/
│   ├── layout.tsx              # Root layout with auth check
│   ├── page.tsx                # Redirect to /admin/dashboard
│   ├── login/page.tsx          # Admin login
│   ├── dashboard/
│   │   └── page.tsx            # Dashboard with metrics
│   └── (protected)/
│       ├── layout.tsx          # Protected layout with sidebar
│       ├── songs/...
│       ├── artists/...
│       └── users/...
├── components/
│   ├── dashboard/
│   │   ├── StatsCards.tsx      # Total counts display
│   │   ├── TopSongsTable.tsx   # Top played songs
│   │   ├── ActivityFeed.tsx    # Recent admin actions
│   │   └── QuickActions.tsx    # Action buttons
│   └── layout/
│       ├── AdminSidebar.tsx
│       └── AdminHeader.tsx
└── lib/
    ├── auth.ts                 # Admin auth utilities
    └── admin-trpc.ts
```

---

## Technical Design

### Auth Flow

```
┌──────────────────────────────────────────────────────────────────┐
│                     ADMIN AUTH FLOW                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  User visits /admin/*                                            │
│         │                                                         │
│         ▼                                                         │
│  ┌─────────────────┐                                             │
│  │ Check Session   │                                             │
│  │ (getServerSess.)│                                             │
│  └────────┬────────┘                                             │
│           │                                                       │
│     ┌─────┴─────┐                                                │
│     │           │                                                │
│  No Session   Has Session                                        │
│     │           │                                                │
│     ▼           ▼                                                │
│  Redirect   ┌─────────────┐                                      │
│  to Login   │ Check Role  │                                      │
│             └──────┬──────┘                                      │
│                    │                                              │
│              ┌─────┴─────┐                                       │
│              │           │                                       │
│         role=admin   role=user                                   │
│              │           │                                       │
│              ▼           ▼                                       │
│         Show Admin   Access Denied                               │
│         Dashboard    Page                                        │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

### Metrics Queries

```typescript
// packages/trpc/routers/admin/dashboard.ts

interface DashboardStats {
  totalSongs: number
  totalArtists: number
  totalUsers: number
  totalCategories: number
  songsThisWeek: number
  songsThisMonth: number
  activeUsers7d: number
}

// Efficient aggregate queries
const stats = await prisma.$transaction([
  prisma.song.count(),
  prisma.artist.count(),
  prisma.user.count(),
  prisma.category.count(),
  prisma.song.count({
    where: { createdAt: { gte: weekAgo } }
  }),
  prisma.song.count({
    where: { createdAt: { gte: monthAgo } }
  }),
  prisma.user.count({
    where: { lastActiveAt: { gte: weekAgo } }
  }),
])
```

### Dashboard API Contract

```typescript
dashboardRouter
├── stats       // DashboardStats aggregate
├── topSongs    // Top 5 by playCount
├── activity    // Last 10 audit log entries
```

---

## File Creation Order

### Phase 1: Auth & Layout

1. `apps/admin/lib/auth.ts` - Auth utilities
2. `apps/admin/app/layout.tsx` - Root layout
3. `apps/admin/app/login/page.tsx` - Login page
4. `apps/admin/app/(protected)/layout.tsx` - Protected layout
5. `apps/admin/components/layout/AdminSidebar.tsx`
6. `apps/admin/components/layout/AdminHeader.tsx`

### Phase 2: Dashboard API

7. `packages/trpc/routers/admin/dashboard.ts`
8. `packages/trpc/tests/admin/dashboard.test.ts`

### Phase 3: Dashboard UI

9. `apps/admin/components/dashboard/StatsCards.tsx`
10. `apps/admin/components/dashboard/TopSongsTable.tsx`
11. `apps/admin/components/dashboard/ActivityFeed.tsx`
12. `apps/admin/components/dashboard/QuickActions.tsx`
13. `apps/admin/app/dashboard/page.tsx`

---

## Complexity Tracking

| Article | Deviation | Justification | Date |
|---------|-----------|---------------|------|
| None | N/A | N/A | - |
