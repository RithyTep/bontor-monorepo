# Tasks: Admin Panel Dashboard

**Feature**: `003-admin-panel`
**Generated**: 2024-12-02

---

## Phase 1: Auth & Layout

- [ ] `[1.1]` Create admin auth utilities (`lib/auth.ts`)
- [ ] `[1.2]` Implement root layout with session check
- [ ] `[1.3]` Create admin login page
- [ ] `[1.4]` Implement protected layout with role check
- [ ] `[1.5]` Create AdminSidebar with navigation links
- [ ] `[1.6]` Create AdminHeader with user menu

**Checkpoint**: Admin auth working, can access protected routes

---

## Phase 2: Dashboard API

- [ ] `[2.1] [P]` Implement `admin.dashboard.stats` procedure
- [ ] `[2.2] [P]` Implement `admin.dashboard.topSongs` procedure
- [ ] `[2.3] [P]` Implement `admin.dashboard.activity` procedure
- [ ] `[2.4]` Write contract tests for dashboard API

**Checkpoint**: Dashboard data API complete

---

## Phase 3: Dashboard Components

- [ ] `[3.1] [P]` Implement StatsCards component (4 metric cards)
- [ ] `[3.2] [P]` Implement TopSongsTable component
- [ ] `[3.3] [P]` Implement ActivityFeed component
- [ ] `[3.4] [P]` Implement QuickActions component

**Checkpoint**: All dashboard components ready

---

## Phase 4: Dashboard Page

- [ ] `[4.1]` Create dashboard page layout
- [ ] `[4.2]` Integrate all dashboard components
- [ ] `[4.3]` Add loading states
- [ ] `[4.4]` Add error handling

**Checkpoint**: Dashboard fully functional

---

## Phase 5: Polish

- [ ] `[5.1]` Add refresh button for stale data
- [ ] `[5.2]` Responsive layout for tablet
- [ ] `[5.3]` Session timeout warning
- [ ] `[5.4]` Test with real data

**Checkpoint**: Production-ready dashboard

---

## Parallel Task Groups

**Group A** (Dashboard API):
- `[2.1]`, `[2.2]`, `[2.3]`

**Group B** (Dashboard Components):
- `[3.1]`, `[3.2]`, `[3.3]`, `[3.4]`

---

## Estimated Effort

| Phase | Tasks | Estimate |
|-------|-------|----------|
| Auth & Layout | 6 | 1 day |
| Dashboard API | 4 | 0.5 days |
| Components | 4 | 1 day |
| Page | 4 | 0.5 days |
| Polish | 4 | 0.5 days |

**Total**: ~3.5 days
