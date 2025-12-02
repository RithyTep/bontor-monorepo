# F005: Admin Panel

## Problem Statement

Product and content teams need a centralized interface to manage the Karaoke Cambodia platform: monitoring usage, managing content, and handling user issues. Without an admin panel, operations require direct database access and lack visibility.

## Solution

Build a dedicated admin panel with:
- Dashboard for key metrics
- Content management (songs, artists, categories)
- User management
- System health monitoring

## User Stories

### US1: View Dashboard
As an admin, I want to see key metrics at a glance so that I can monitor platform health.

**Acceptance Criteria:**
- [ ] Total songs, artists, users counts
- [ ] Songs added this week/month
- [ ] Active users (7-day)
- [ ] Top played songs
- [ ] Recent activity feed

### US2: Manage Content
As an admin, I want to access content management features so that I can maintain the catalog.

**Acceptance Criteria:**
- [ ] Navigate to songs, artists, categories
- [ ] Quick actions (publish, edit, delete)
- [ ] Search and filter capabilities
- [ ] Bulk operations (publish multiple)

### US3: Manage Users
As an admin, I want to view and manage user accounts so that I can handle support issues.

**Acceptance Criteria:**
- [ ] User list with search
- [ ] View user profile details
- [ ] View user's favorites and scores
- [ ] Disable/enable user accounts
- [ ] Cannot delete users (soft disable only)

### US4: Admin Authentication
As an admin, I want secure access to the admin panel so that unauthorized users cannot access it.

**Acceptance Criteria:**
- [ ] Separate admin login (or role-based)
- [ ] Admin role required for access
- [ ] Session timeout (30 min inactivity)
- [ ] Audit log of admin actions

## Implementation

### Admin App Structure

```
apps/admin/
├── app/
│   ├── layout.tsx           # Admin layout with sidebar
│   ├── page.tsx             # Redirect to dashboard
│   ├── dashboard/
│   │   └── page.tsx         # Dashboard with stats
│   ├── songs/
│   │   ├── page.tsx         # Song list
│   │   ├── new/page.tsx     # Create song
│   │   └── [id]/
│   │       ├── page.tsx     # Song detail
│   │       └── edit/page.tsx
│   ├── artists/
│   │   ├── page.tsx
│   │   ├── new/page.tsx
│   │   └── [id]/edit/page.tsx
│   ├── categories/
│   │   └── page.tsx
│   ├── users/
│   │   ├── page.tsx
│   │   └── [id]/page.tsx
│   └── settings/
│       └── page.tsx
├── components/
│   ├── layout/
│   │   ├── Sidebar.tsx
│   │   ├── Header.tsx
│   │   └── Breadcrumb.tsx
│   ├── dashboard/
│   │   ├── StatsCard.tsx
│   │   ├── RecentActivity.tsx
│   │   └── TopSongsChart.tsx
│   └── forms/
│       ├── SongForm.tsx
│       ├── ArtistForm.tsx
│       └── CategoryForm.tsx
└── lib/
    ├── admin-trpc.ts
    └── auth.ts
```

### Role-Based Access

```typescript
// User roles
enum Role {
  USER = 'user',
  ADMIN = 'admin',
  SUPER_ADMIN = 'super_admin'
}

// Prisma schema addition
model User {
  // ... existing fields
  role Role @default(USER)
}

// Admin middleware
const adminProcedure = protectedProcedure.use(async ({ ctx, next }) => {
  if (ctx.user.role !== 'admin' && ctx.user.role !== 'super_admin') {
    throw new TRPCError({ code: 'FORBIDDEN' })
  }
  return next()
})
```

### Dashboard Metrics API

```typescript
dashboardRouter
├── stats        // Total counts
├── recent       // Recent songs added
├── topSongs     // Most played
├── activity     // Admin action log
└── users        // User growth stats
```

### Audit Logging

```prisma
model AdminAuditLog {
  id          String   @id @default(uuid())
  adminId     String
  admin       User     @relation(...)
  action      String   // "song.create", "user.disable", etc.
  targetType  String   // "song", "user", etc.
  targetId    String
  details     Json?    // Additional context
  createdAt   DateTime @default(now())

  @@index([adminId])
  @@index([targetType, targetId])
}
```

## Metrics

### Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Admin task completion time | < 5 min avg | Time tracking |
| Dashboard load time | < 2s | Performance monitoring |
| Admin error rate | < 1% | Error logs |

### Security Metrics

- Failed admin login attempts
- Unusual admin activity patterns
- Session duration monitoring

## Related

- Features: F002_SongCMS, F006_Authentication
- Decisions: D007_Authentication
- Implementation: I003_CMSWorkflow
